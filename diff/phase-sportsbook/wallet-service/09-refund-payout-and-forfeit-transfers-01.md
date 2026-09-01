# 환급·지급·몰수 자금 이동

## `feat(command): define positive credit and forfeit commands`

diff --git a/src/main/java/com/sportsbook/wallet/service/command/CreditCommand.java b/src/main/java/com/sportsbook/wallet/service/command/CreditCommand.java
new file mode 100644
index 0000000..68ea1f9
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/service/command/CreditCommand.java
@@ -0,0 +1,27 @@
+package com.sportsbook.wallet.service.command;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import java.util.Objects;
+import java.util.UUID;
+
+/** Credits available funds from locked stake or the house pool. */
+public record CreditCommand(
+    UUID userId, Money amount, Source source, CreditReason reason, IdempotencyKey idempotencyKey) {
+
+  public CreditCommand {
+    Objects.requireNonNull(userId, "userId");
+    Objects.requireNonNull(amount, "amount");
+    Objects.requireNonNull(source, "source");
+    Objects.requireNonNull(reason, "reason");
+    Objects.requireNonNull(idempotencyKey, "idempotencyKey");
+    if (!amount.isPositive()) {
+      throw new IllegalArgumentException("Credit amount must be strictly positive");
+    }
+  }
+
+  public enum Source {
+    USER_LOCKED,
+    HOUSE_POOL
+  }
+}
diff --git a/src/main/java/com/sportsbook/wallet/service/command/ForfeitCommand.java b/src/main/java/com/sportsbook/wallet/service/command/ForfeitCommand.java
new file mode 100644
index 0000000..f0b7596
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/service/command/ForfeitCommand.java
@@ -0,0 +1,19 @@
+package com.sportsbook.wallet.service.command;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import java.util.Objects;
+import java.util.UUID;
+
+/** Transfers a losing stake from the user's locked balance to the house. */
+public record ForfeitCommand(UUID userId, Money amount, IdempotencyKey idempotencyKey) {
+
+  public ForfeitCommand {
+    Objects.requireNonNull(userId, "userId");
+    Objects.requireNonNull(amount, "amount");
+    Objects.requireNonNull(idempotencyKey, "idempotencyKey");
+    if (!amount.isPositive()) {
+      throw new IllegalArgumentException("Forfeit amount must be strictly positive");
+    }
+  }
+}


## `feat(command): classify credit reasons`

diff --git a/src/main/java/com/sportsbook/wallet/service/command/CreditReason.java b/src/main/java/com/sportsbook/wallet/service/command/CreditReason.java
new file mode 100644
index 0000000..5718fec
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/service/command/CreditReason.java
@@ -0,0 +1,8 @@
+package com.sportsbook.wallet.service.command;
+
+/** Explicit business meaning carried by every user credit. */
+public enum CreditReason {
+  PAYOUT,
+  VOID,
+  REFUND
+}


## `feat(service): prepare semantic credit transfers`

diff --git a/src/main/java/com/sportsbook/wallet/service/WalletTransferExecutor.java b/src/main/java/com/sportsbook/wallet/service/WalletTransferExecutor.java
index e0883f8..28de4e3 100644
--- a/src/main/java/com/sportsbook/wallet/service/WalletTransferExecutor.java
+++ b/src/main/java/com/sportsbook/wallet/service/WalletTransferExecutor.java
@@ -12,6 +12,8 @@ import com.sportsbook.wallet.domain.error.CurrencyMismatchException;
 import com.sportsbook.wallet.outbox.OutboxAppender;
 import com.sportsbook.wallet.outbox.WalletEventFactory;
 import com.sportsbook.wallet.persistence.AccountRepository;
+import com.sportsbook.wallet.service.command.CreditCommand;
+import com.sportsbook.wallet.service.command.CreditReason;
 import com.sportsbook.wallet.service.command.DebitCommand;
 import java.time.Clock;
 import java.time.Instant;
@@ -55,7 +57,16 @@ public class WalletTransferExecutor {
       UUID userId,
       Money amount,
       BiFunction<Account, Instant, WalletTransferPlan> mutation) {
-    return execute(key, caller, kind, userId, amount, null, mutation);
+    return execute(
+        key,
+        caller,
+        kind,
+        userId,
+        amount,
+        OperationFingerprint.transfer(caller, kind, userId, amount),
+        null,
+        null,
+        mutation);
   }
 
   public WalletOperationResult executeDebit(
@@ -66,6 +77,34 @@ public class WalletTransferExecutor {
         WalletOperationKind.BET_DEBIT,
         command.userId(),
         command.amount(),
+        OperationFingerprint.transfer(
+            WalletCaller.BETTING,
+            WalletOperationKind.BET_DEBIT,
+            command.userId(),
+            command.amount()),
+        command,
+        null,
+        mutation);
+  }
+
+  public WalletOperationResult executeCredit(
+      WalletCaller caller,
+      CreditCommand command,
+      BiFunction<Account, Instant, WalletTransferPlan> mutation) {
+    requireAllowedCredit(caller, command);
+    WalletOperationKind kind =
+        command.reason() == CreditReason.PAYOUT
+            ? WalletOperationKind.BET_PAYOUT
+            : WalletOperationKind.BET_REFUND;
+    return execute(
+        command.idempotencyKey(),
+        caller,
+        kind,
+        command.userId(),
+        command.amount(),
+        OperationFingerprint.credit(
+            caller, kind, command.userId(), command.amount(), command.source(), command.reason()),
+        null,
         command,
         mutation);
   }
@@ -77,7 +116,9 @@ public class WalletTransferExecutor {
       WalletOperationKind kind,
       UUID userId,
       Money amount,
+      OperationFingerprint requestFingerprint,
       DebitCommand debit,
+      CreditCommand credit,
       BiFunction<Account, Instant, WalletTransferPlan> mutation) {
     WalletOperation operation =
         operations.execute(
@@ -86,8 +127,10 @@ public class WalletTransferExecutor {
             kind,
             userId,
             amount,
+            requestFingerprint,
             fingerprint ->
-                firstWrite(key, caller, kind, userId, amount, fingerprint, debit, mutation));
+                firstWrite(
+                    key, caller, kind, userId, amount, fingerprint, debit, credit, mutation));
     return outcomes.resolve(operation);
   }
 
@@ -100,6 +143,7 @@ public class WalletTransferExecutor {
       Money amount,
       String fingerprint,
       DebitCommand debit,
+      CreditCommand credit,
       BiFunction<Account, Instant, WalletTransferPlan> mutation) {
     Instant now = clock.instant().truncatedTo(ChronoUnit.MICROS);
     WalletTransferReceipt receipt;
@@ -120,10 +164,31 @@ public class WalletTransferExecutor {
     if (debit != null) {
       outboxAppender.append(eventFactory.debited(debit, receipt.destinationEntryId(), now));
     }
+    if (credit != null) {
+      outboxAppender.append(eventFactory.credited(credit, receipt.destinationEntryId(), now));
+    }
     return WalletOperation.succeeded(
         key, caller, kind, userId, amount, fingerprint, receipt.result().operationGroupId(), now);
   }
 
+  static void requireAllowedCredit(WalletCaller caller, CreditCommand command) {
+    boolean allowed =
+        (caller == WalletCaller.BETTING
+                && command.source() == CreditCommand.Source.USER_LOCKED
+                && command.reason() == CreditReason.REFUND)
+            || (caller == WalletCaller.SETTLEMENT
+                && ((command.source() == CreditCommand.Source.USER_LOCKED
+                        && command.reason() != CreditReason.PAYOUT)
+                    || (command.source() == CreditCommand.Source.HOUSE_POOL
+                        && command.reason() == CreditReason.PAYOUT)))
+            || (caller == WalletCaller.ADMIN
+                && command.source() == CreditCommand.Source.HOUSE_POOL
+                && command.reason() == CreditReason.REFUND);
+    if (!allowed) {
+      throw new IllegalArgumentException("Caller is not allowed for credit source and reason");
+    }
+  }
+
   private Account lockAccount(UUID userId, Money amount) {
     Account account =
         accounts


## `test(service): accept house-funded refunds`

diff --git a/src/test/java/com/sportsbook/wallet/service/WalletTransferTopologyTest.java b/src/test/java/com/sportsbook/wallet/service/WalletTransferTopologyTest.java
index 201ec39..e0874d7 100644
--- a/src/test/java/com/sportsbook/wallet/service/WalletTransferTopologyTest.java
+++ b/src/test/java/com/sportsbook/wallet/service/WalletTransferTopologyTest.java
@@ -44,6 +44,12 @@ class WalletTransferTopologyTest {
                 BalanceBucket.AVAILABLE),
             new Topology(
                 LedgerReason.BET_REFUND, USER, BalanceBucket.AVAILABLE, USER, BalanceBucket.LOCKED),
+            new Topology(
+                LedgerReason.BET_REFUND,
+                USER,
+                BalanceBucket.AVAILABLE,
+                SystemAccountIds.HOUSE,
+                BalanceBucket.AVAILABLE),
             new Topology(
                 LedgerReason.BET_FORFEIT,
                 SystemAccountIds.HOUSE,


## `feat(credit): refund stakes and pay winnings`

diff --git a/src/main/java/com/sportsbook/wallet/service/WalletOperationResult.java b/src/main/java/com/sportsbook/wallet/service/WalletOperationResult.java
index b680a5a..670ccc4 100644
--- a/src/main/java/com/sportsbook/wallet/service/WalletOperationResult.java
+++ b/src/main/java/com/sportsbook/wallet/service/WalletOperationResult.java
@@ -81,7 +81,8 @@ public record WalletOperationResult(
                   && matches(credit, SystemAccountIds.HOUSE, BalanceBucket.AVAILABLE);
           case BET_REFUND ->
               matches(debit, userId, BalanceBucket.AVAILABLE)
-                  && matches(credit, userId, BalanceBucket.LOCKED);
+                  && (matches(credit, userId, BalanceBucket.LOCKED)
+                      || matches(credit, SystemAccountIds.HOUSE, BalanceBucket.AVAILABLE));
           case BET_FORFEIT ->
               matches(debit, SystemAccountIds.HOUSE, BalanceBucket.AVAILABLE)
                   && matches(credit, userId, BalanceBucket.LOCKED);
diff --git a/src/main/java/com/sportsbook/wallet/service/WalletService.java b/src/main/java/com/sportsbook/wallet/service/WalletService.java
index afb6b8f..44637fd 100644
--- a/src/main/java/com/sportsbook/wallet/service/WalletService.java
+++ b/src/main/java/com/sportsbook/wallet/service/WalletService.java
@@ -10,6 +10,8 @@ import com.sportsbook.wallet.domain.WalletOperationKind;
 import com.sportsbook.wallet.domain.error.AccountNotFoundException;
 import com.sportsbook.wallet.domain.error.CurrencyMismatchException;
 import com.sportsbook.wallet.persistence.AccountRepository;
+import com.sportsbook.wallet.service.command.CreditCommand;
+import com.sportsbook.wallet.service.command.CreditReason;
 import com.sportsbook.wallet.service.command.DebitCommand;
 import com.sportsbook.wallet.service.command.DepositCommand;
 import com.sportsbook.wallet.service.command.OpenAccountCommand;
@@ -114,6 +116,30 @@ public class WalletService {
         });
   }
 
+  public WalletOperationResult credit(WalletCaller caller, CreditCommand command) {
+    return transferExecutor.executeCredit(
+        caller,
+        command,
+        (account, now) -> {
+          LedgerEntry.TransferLeg source;
+          if (command.source() == CreditCommand.Source.USER_LOCKED) {
+            account.moveLockedToAvailable(command.amount(), now);
+            source = new LedgerEntry.TransferLeg(command.userId(), BalanceBucket.LOCKED);
+          } else {
+            account.increaseAvailable(command.amount(), now);
+            source = new LedgerEntry.TransferLeg(SystemAccountIds.HOUSE, BalanceBucket.AVAILABLE);
+          }
+          LedgerReason reason =
+              command.reason() == CreditReason.PAYOUT
+                  ? LedgerReason.BET_PAYOUT
+                  : LedgerReason.BET_REFUND;
+          return new WalletTransferPlan(
+              new LedgerEntry.TransferLeg(command.userId(), BalanceBucket.AVAILABLE),
+              source,
+              reason);
+        });
+  }
+
   private static Account requireCurrency(Account account, OpenAccountCommand command) {
     if (account.currency() != command.currency()) {
       throw new CurrencyMismatchException(account.currency(), command.currency());


## `test(credit): allow settlement and admin refunds`

diff --git a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
index 41157bd..a439896 100644
--- a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
+++ b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
@@ -713,6 +713,51 @@ class WalletPersistenceTest {
         .isInstanceOf(RuntimeException.class);
   }
 
+  @Test
+  void allowsSettlementAndAdminRefundsFromTheirAuthorizedSources() {
+    UUID settlementUser = UUID.fromString("019b76da-a000-7000-8000-000000000024");
+    wallet.openAccount(
+        new OpenAccountCommand(settlementUser, com.sportsbook.protocol.value.Currency.KRW));
+    wallet.deposit(
+        new DepositCommand(
+            settlementUser, Money.krw(80L), IdempotencyKey.of("deposit:settlement-refund")));
+    wallet.debit(
+        new DebitCommand(
+            settlementUser, Money.krw(80L), IdempotencyKey.of("debit:settlement-refund")));
+
+    var settlementRefund =
+        wallet.credit(
+            WalletCaller.SETTLEMENT,
+            new CreditCommand(
+                settlementUser,
+                Money.krw(30L),
+                CreditCommand.Source.USER_LOCKED,
+                CreditReason.REFUND,
+                IdempotencyKey.of("credit:settlement-refund")));
+
+    assertThat(wallet.requireAccount(settlementUser).available()).isEqualTo(Money.krw(30L));
+    assertThat(wallet.requireAccount(settlementUser).locked()).isEqualTo(Money.krw(50L));
+    assertThat(settlementRefund.reason())
+        .isEqualTo(com.sportsbook.wallet.domain.LedgerReason.BET_REFUND);
+
+    UUID adminUser = UUID.fromString("019b76da-a000-7000-8000-000000000025");
+    wallet.openAccount(
+        new OpenAccountCommand(adminUser, com.sportsbook.protocol.value.Currency.KRW));
+    var adminRefund =
+        wallet.credit(
+            WalletCaller.ADMIN,
+            new CreditCommand(
+                adminUser,
+                Money.krw(45L),
+                CreditCommand.Source.HOUSE_POOL,
+                CreditReason.REFUND,
+                IdempotencyKey.of("credit:admin-refund")));
+
+    assertThat(wallet.requireAccount(adminUser).available()).isEqualTo(Money.krw(45L));
+    assertThat(adminRefund.reason())
+        .isEqualTo(com.sportsbook.wallet.domain.LedgerReason.BET_REFUND);
+  }
+
   @Test
   void convergesOneHundredConcurrentRequestsForOneKey() {
     UUID userId = UUID.fromString("019b76da-a000-7000-8000-000000000019");


## `test(credit): verify refund and payout transfers`

diff --git a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
index 7415cb1..41157bd 100644
--- a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
+++ b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
@@ -35,6 +35,8 @@ import com.sportsbook.wallet.service.WalletOutcomeResolver;
 import com.sportsbook.wallet.service.WalletService;
 import com.sportsbook.wallet.service.WalletTransferExecutor;
 import com.sportsbook.wallet.service.WalletTransferWriter;
+import com.sportsbook.wallet.service.command.CreditCommand;
+import com.sportsbook.wallet.service.command.CreditReason;
 import com.sportsbook.wallet.service.command.DebitCommand;
 import com.sportsbook.wallet.service.command.DepositCommand;
 import com.sportsbook.wallet.service.command.OpenAccountCommand;
@@ -633,6 +635,84 @@ class WalletPersistenceTest {
         .isEqualTo(com.sportsbook.wallet.outbox.WalletEventFactory.DEBIT_FAILED_TOPIC);
   }
 
+  @Test
+  void refundsLockedStakesAndPaysHouseFundedWinnings() {
+    UUID refundUser = UUID.fromString("019b76da-a000-7000-8000-00000000001e");
+    wallet.openAccount(
+        new OpenAccountCommand(refundUser, com.sportsbook.protocol.value.Currency.KRW));
+    wallet.deposit(
+        new DepositCommand(refundUser, Money.krw(100L), IdempotencyKey.of("deposit:credit-seed")));
+    wallet.debit(
+        new DebitCommand(refundUser, Money.krw(100L), IdempotencyKey.of("debit:credit-seed")));
+    CreditCommand bettingRefund =
+        new CreditCommand(
+            refundUser,
+            Money.krw(40L),
+            CreditCommand.Source.USER_LOCKED,
+            CreditReason.REFUND,
+            IdempotencyKey.of("credit:betting-refund"));
+    CreditCommand settlementVoid =
+        new CreditCommand(
+            refundUser,
+            Money.krw(10L),
+            CreditCommand.Source.USER_LOCKED,
+            CreditReason.VOID,
+            IdempotencyKey.of("credit:settlement-void"));
+
+    var refund = wallet.credit(com.sportsbook.wallet.domain.WalletCaller.BETTING, bettingRefund);
+    var voided =
+        wallet.credit(com.sportsbook.wallet.domain.WalletCaller.SETTLEMENT, settlementVoid);
+
+    assertThat(wallet.credit(com.sportsbook.wallet.domain.WalletCaller.BETTING, bettingRefund))
+        .isEqualTo(refund);
+    assertThat(wallet.credit(com.sportsbook.wallet.domain.WalletCaller.SETTLEMENT, settlementVoid))
+        .isEqualTo(voided);
+    assertThat(wallet.requireAccount(refundUser).available()).isEqualTo(Money.krw(50L));
+    assertThat(wallet.requireAccount(refundUser).locked()).isEqualTo(Money.krw(50L));
+    assertThat(refund.reason()).isEqualTo(com.sportsbook.wallet.domain.LedgerReason.BET_REFUND);
+
+    UUID payoutUser = UUID.fromString("019b76da-a000-7000-8000-00000000001f");
+    wallet.openAccount(
+        new OpenAccountCommand(payoutUser, com.sportsbook.protocol.value.Currency.KRW));
+    CreditCommand payout =
+        new CreditCommand(
+            payoutUser,
+            Money.krw(250L),
+            CreditCommand.Source.HOUSE_POOL,
+            CreditReason.PAYOUT,
+            IdempotencyKey.of("credit:settlement-payout"));
+    var paid = wallet.credit(com.sportsbook.wallet.domain.WalletCaller.SETTLEMENT, payout);
+
+    assertThat(wallet.credit(com.sportsbook.wallet.domain.WalletCaller.SETTLEMENT, payout))
+        .isEqualTo(paid);
+    assertThat(wallet.requireAccount(payoutUser).available()).isEqualTo(Money.krw(250L));
+    assertThat(paid.reason()).isEqualTo(com.sportsbook.wallet.domain.LedgerReason.BET_PAYOUT);
+    assertThat(outboxFor(payout.idempotencyKey())).hasSize(1);
+
+    CreditCommand changedReason =
+        new CreditCommand(
+            refundUser,
+            settlementVoid.amount(),
+            settlementVoid.source(),
+            CreditReason.REFUND,
+            settlementVoid.idempotencyKey());
+    assertThatThrownBy(
+            () ->
+                wallet.credit(com.sportsbook.wallet.domain.WalletCaller.SETTLEMENT, changedReason))
+        .isInstanceOf(IdempotencyConflictException.class);
+    assertThatThrownBy(
+            () ->
+                wallet.credit(
+                    com.sportsbook.wallet.domain.WalletCaller.BETTING,
+                    new CreditCommand(
+                        refundUser,
+                        Money.krw(1L),
+                        CreditCommand.Source.HOUSE_POOL,
+                        CreditReason.PAYOUT,
+                        IdempotencyKey.of("credit:forbidden"))))
+        .isInstanceOf(RuntimeException.class);
+  }
+
   @Test
   void convergesOneHundredConcurrentRequestsForOneKey() {
     UUID userId = UUID.fromString("019b76da-a000-7000-8000-000000000019");


## `feat(forfeit): transfer lost stakes to the house`

diff --git a/src/main/java/com/sportsbook/wallet/service/WalletService.java b/src/main/java/com/sportsbook/wallet/service/WalletService.java
index 44637fd..8a98cc2 100644
--- a/src/main/java/com/sportsbook/wallet/service/WalletService.java
+++ b/src/main/java/com/sportsbook/wallet/service/WalletService.java
@@ -14,6 +14,7 @@ import com.sportsbook.wallet.service.command.CreditCommand;
 import com.sportsbook.wallet.service.command.CreditReason;
 import com.sportsbook.wallet.service.command.DebitCommand;
 import com.sportsbook.wallet.service.command.DepositCommand;
+import com.sportsbook.wallet.service.command.ForfeitCommand;
 import com.sportsbook.wallet.service.command.OpenAccountCommand;
 import com.sportsbook.wallet.service.command.WithdrawCommand;
 import java.time.Clock;
@@ -140,6 +141,22 @@ public class WalletService {
         });
   }
 
+  public WalletOperationResult forfeit(ForfeitCommand command) {
+    return transferExecutor.execute(
+        command.idempotencyKey(),
+        WalletCaller.SETTLEMENT,
+        WalletOperationKind.BET_FORFEIT,
+        command.userId(),
+        command.amount(),
+        (account, now) -> {
+          account.forfeitLocked(command.amount(), now);
+          return new WalletTransferPlan(
+              new LedgerEntry.TransferLeg(SystemAccountIds.HOUSE, BalanceBucket.AVAILABLE),
+              new LedgerEntry.TransferLeg(command.userId(), BalanceBucket.LOCKED),
+              LedgerReason.BET_FORFEIT);
+        });
+  }
+
   private static Account requireCurrency(Account account, OpenAccountCommand command) {
     if (account.currency() != command.currency()) {
       throw new CurrencyMismatchException(account.currency(), command.currency());


