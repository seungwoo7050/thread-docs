## `test(service): verify durable deposit replay and rejection`

diff --git a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
index ffe7293..1f3cc5b 100644
--- a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
+++ b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
@@ -2,6 +2,7 @@ package com.sportsbook.wallet.persistence;
 
 import static org.assertj.core.api.Assertions.assertThat;
 import static org.assertj.core.api.Assertions.assertThatThrownBy;
+import static org.assertj.core.api.Assertions.catchThrowableOfType;
 
 import com.sportsbook.protocol.value.IdempotencyKey;
 import com.sportsbook.protocol.value.Money;
@@ -18,12 +19,15 @@ import com.sportsbook.wallet.domain.WalletOperationKind;
 import com.sportsbook.wallet.domain.WalletOperationStatus;
 import com.sportsbook.wallet.domain.error.AccountNotFoundException;
 import com.sportsbook.wallet.domain.error.CurrencyMismatchException;
+import com.sportsbook.wallet.domain.error.IdempotencyConflictException;
 import com.sportsbook.wallet.domain.error.WalletBusyException;
+import com.sportsbook.wallet.domain.error.WalletRejectedException;
 import com.sportsbook.wallet.service.WalletOperationExecutor;
 import com.sportsbook.wallet.service.WalletOutcomeResolver;
 import com.sportsbook.wallet.service.WalletService;
 import com.sportsbook.wallet.service.WalletTransferExecutor;
 import com.sportsbook.wallet.service.WalletTransferWriter;
+import com.sportsbook.wallet.service.command.DepositCommand;
 import com.sportsbook.wallet.service.command.OpenAccountCommand;
 import java.math.BigDecimal;
 import java.math.BigInteger;
@@ -343,6 +347,48 @@ class WalletPersistenceTest {
         .hasSize(1);
   }
 
+  @Test
+  void commitsAndExactlyReplaysOneDeposit() {
+    UUID userId = UUID.fromString("019b76da-a000-7000-8000-000000000014");
+    wallet.openAccount(new OpenAccountCommand(userId, com.sportsbook.protocol.value.Currency.KRW));
+    DepositCommand command =
+        new DepositCommand(userId, Money.krw(500L), IdempotencyKey.of("deposit:durable"));
+
+    var first = wallet.deposit(command);
+    var replay = wallet.deposit(command);
+
+    assertThat(replay).isEqualTo(first);
+    assertThat(wallet.requireAccount(userId).available()).isEqualTo(Money.krw(500L));
+    assertThat(ledger.findByIdempotencyKey(command.idempotencyKey().value())).hasSize(2);
+    assertThat(operations.findById(command.idempotencyKey().value()).orElseThrow().status())
+        .isEqualTo(WalletOperationStatus.SUCCEEDED);
+    assertThatThrownBy(
+            () ->
+                wallet.deposit(
+                    new DepositCommand(userId, Money.krw(501L), command.idempotencyKey())))
+        .isInstanceOf(IdempotencyConflictException.class);
+  }
+
+  @Test
+  void replaysACommittedRejectionAfterFactsChange() {
+    UUID userId = UUID.fromString("019b76da-a000-7000-8000-000000000015");
+    DepositCommand command =
+        new DepositCommand(userId, Money.krw(100L), IdempotencyKey.of("deposit:missing"));
+
+    WalletRejectedException first =
+        catchThrowableOfType(() -> wallet.deposit(command), WalletRejectedException.class);
+    wallet.openAccount(new OpenAccountCommand(userId, com.sportsbook.protocol.value.Currency.KRW));
+    WalletRejectedException replay =
+        catchThrowableOfType(() -> wallet.deposit(command), WalletRejectedException.class);
+
+    assertThat(replay.failure().code()).isEqualTo(WalletFailureCode.ACCOUNT_NOT_FOUND);
+    assertThat(replay.failure().detail()).isEqualTo(first.failure().detail());
+    assertThat(wallet.requireAccount(userId).available()).isEqualTo(Money.krw(0L));
+    assertThat(ledger.findByIdempotencyKey(command.idempotencyKey().value())).isEmpty();
+    assertThat(operations.findById(command.idempotencyKey().value()).orElseThrow().status())
+        .isEqualTo(WalletOperationStatus.REJECTED);
+  }
+
   private static void await(java.util.concurrent.CountDownLatch latch) {
     try {
       latch.await();


## `test(service): verify durable withdrawal replay and rejection`

diff --git a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
index 5d39311..6fbc7cd 100644
--- a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
+++ b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
@@ -30,6 +30,7 @@ import com.sportsbook.wallet.service.WalletTransferExecutor;
 import com.sportsbook.wallet.service.WalletTransferWriter;
 import com.sportsbook.wallet.service.command.DepositCommand;
 import com.sportsbook.wallet.service.command.OpenAccountCommand;
+import com.sportsbook.wallet.service.command.WithdrawCommand;
 import java.math.BigDecimal;
 import java.math.BigInteger;
 import java.time.Instant;
@@ -428,6 +429,32 @@ class WalletPersistenceTest {
     }
   }
 
+  @Test
+  void exactlyReplaysWithdrawalSuccessAndInsufficientFunds() {
+    UUID userId = UUID.fromString("019b76da-a000-7000-8000-000000000016");
+    wallet.openAccount(new OpenAccountCommand(userId, com.sportsbook.protocol.value.Currency.KRW));
+    wallet.deposit(
+        new DepositCommand(userId, Money.krw(100L), IdempotencyKey.of("deposit:withdraw-seed")));
+    WithdrawCommand rejected =
+        new WithdrawCommand(userId, Money.krw(200L), IdempotencyKey.of("withdraw:rejected"));
+
+    WalletRejectedException first =
+        catchThrowableOfType(() -> wallet.withdraw(rejected), WalletRejectedException.class);
+    wallet.deposit(
+        new DepositCommand(userId, Money.krw(300L), IdempotencyKey.of("deposit:withdraw-more")));
+    WalletRejectedException replay =
+        catchThrowableOfType(() -> wallet.withdraw(rejected), WalletRejectedException.class);
+
+    assertThat(first.failure().code()).isEqualTo(WalletFailureCode.INSUFFICIENT_BALANCE);
+    assertThat(replay.failure().detail()).isEqualTo(first.failure().detail());
+    WithdrawCommand success =
+        new WithdrawCommand(userId, Money.krw(150L), IdempotencyKey.of("withdraw:success"));
+    var result = wallet.withdraw(success);
+    assertThat(wallet.withdraw(success)).isEqualTo(result);
+    assertThat(wallet.requireAccount(userId).available()).isEqualTo(Money.krw(250L));
+    assertThat(ledger.findByIdempotencyKey(success.idempotencyKey().value())).hasSize(2);
+  }
+
   private static void await(java.util.concurrent.CountDownLatch latch) {
     try {
       latch.await();


## `test(concurrency): converge exact-balance debit races`

diff --git a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
index cdac3a8..59fb448 100644
--- a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
+++ b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
@@ -32,15 +32,19 @@ import com.sportsbook.wallet.service.WalletOutcomeResolver;
 import com.sportsbook.wallet.service.WalletService;
 import com.sportsbook.wallet.service.WalletTransferExecutor;
 import com.sportsbook.wallet.service.WalletTransferWriter;
+import com.sportsbook.wallet.service.command.DebitCommand;
 import com.sportsbook.wallet.service.command.DepositCommand;
 import com.sportsbook.wallet.service.command.OpenAccountCommand;
 import com.sportsbook.wallet.service.command.WithdrawCommand;
 import java.math.BigDecimal;
 import java.math.BigInteger;
 import java.time.Instant;
+import java.util.List;
+import java.util.Optional;
 import java.util.UUID;
 import java.util.concurrent.CompletableFuture;
 import java.util.concurrent.atomic.AtomicBoolean;
+import java.util.function.Supplier;
 import org.junit.jupiter.api.Test;
 import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
@@ -484,6 +488,71 @@ class WalletPersistenceTest {
     assertThat(ledger.findByIdempotencyKey(command.idempotencyKey().value())).hasSize(2);
   }
 
+  @Test
+  void serializesTwoExactBalanceBetDebits() {
+    UUID userId = UUID.fromString("019b76da-a000-7000-8000-000000000018");
+    wallet.openAccount(new OpenAccountCommand(userId, com.sportsbook.protocol.value.Currency.KRW));
+    wallet.deposit(
+        new DepositCommand(userId, Money.krw(100L), IdempotencyKey.of("deposit:race-seed")));
+    DebitCommand firstCommand =
+        new DebitCommand(userId, Money.krw(100L), IdempotencyKey.of("debit:race:first"));
+    DebitCommand secondCommand =
+        new DebitCommand(userId, Money.krw(100L), IdempotencyKey.of("debit:race:second"));
+    java.util.concurrent.CountDownLatch start = new java.util.concurrent.CountDownLatch(1);
+    java.util.function.Function<DebitCommand, WalletOperationStatus> attempt =
+        command -> {
+          try {
+            wallet.debit(command);
+            return WalletOperationStatus.SUCCEEDED;
+          } catch (WalletRejectedException rejected) {
+            return WalletOperationStatus.REJECTED;
+          }
+        };
+    java.util.function.Function<DebitCommand, CompletableFuture<Optional<WalletOperationStatus>>>
+        submit =
+            command ->
+                CompletableFuture.supplyAsync(
+                    () -> {
+                      await(start);
+                      return retryableAttempt(() -> attempt.apply(command));
+                    });
+    var commands = List.of(firstCommand, secondCommand);
+    var attempts = commands.stream().map(submit).toList();
+
+    start.countDown();
+    CompletableFuture.allOf(attempts.toArray(CompletableFuture[]::new)).join();
+    var statuses =
+        java.util.stream.IntStream.range(0, commands.size())
+            .mapToObj(
+                index ->
+                    attempts.get(index).join().orElseGet(() -> attempt.apply(commands.get(index))))
+            .toList();
+
+    assertThat(statuses)
+        .containsExactlyInAnyOrder(WalletOperationStatus.SUCCEEDED, WalletOperationStatus.REJECTED);
+    assertThat(wallet.requireAccount(userId).available()).isEqualTo(Money.krw(0L));
+    assertThat(wallet.requireAccount(userId).locked()).isEqualTo(Money.krw(100L));
+    assertThat(
+            commands.stream()
+                .map(command -> operations.findById(command.idempotencyKey().value()).orElseThrow())
+                .map(WalletOperation::status))
+        .containsExactlyInAnyOrder(WalletOperationStatus.SUCCEEDED, WalletOperationStatus.REJECTED);
+    assertThat(
+            commands.stream()
+                .mapToInt(
+                    command -> ledger.findByIdempotencyKey(command.idempotencyKey().value()).size())
+                .sum())
+        .isEqualTo(2);
+  }
+
+  private static <T> Optional<T> retryableAttempt(Supplier<T> attempt) {
+    try {
+      return Optional.of(attempt.get());
+    } catch (WalletBusyException busy) {
+      return Optional.empty();
+    }
+  }
+
   private static void await(java.util.concurrent.CountDownLatch latch) {
     try {
       latch.await();
