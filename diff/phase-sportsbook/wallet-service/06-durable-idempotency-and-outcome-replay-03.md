## `test(service): recover after eviction outage and post-commit faults`

diff --git a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
index f3cef70..cdac3a8 100644
--- a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
+++ b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
@@ -3,6 +3,9 @@ package com.sportsbook.wallet.persistence;
 import static org.assertj.core.api.Assertions.assertThat;
 import static org.assertj.core.api.Assertions.assertThatThrownBy;
 import static org.assertj.core.api.Assertions.catchThrowableOfType;
+import static org.mockito.Mockito.doThrow;
+import static org.mockito.Mockito.reset;
+import static org.mockito.Mockito.when;
 
 import com.sportsbook.protocol.value.IdempotencyKey;
 import com.sportsbook.protocol.value.Money;
@@ -458,6 +461,29 @@ class WalletPersistenceTest {
     assertThat(ledger.findByIdempotencyKey(success.idempotencyKey().value())).hasSize(2);
   }
 
+  @Test
+  void recoversAfterCacheEvictionOutageAndPostCommitFaults() {
+    UUID userId = UUID.fromString("019b76da-a000-7000-8000-000000000017");
+    wallet.openAccount(new OpenAccountCommand(userId, com.sportsbook.protocol.value.Currency.KRW));
+    DepositCommand command =
+        new DepositCommand(userId, Money.krw(90L), IdempotencyKey.of("deposit:cache-fault"));
+    doThrow(new org.springframework.data.redis.RedisConnectionFailureException("post-commit"))
+        .when(cache)
+        .mark(command.idempotencyKey());
+
+    var committed = wallet.deposit(command);
+    reset(cache);
+    var afterEviction = wallet.deposit(command);
+    when(cache.mightContain(command.idempotencyKey()))
+        .thenThrow(new org.springframework.data.redis.RedisConnectionFailureException("lookup"));
+    var duringOutage = wallet.deposit(command);
+
+    assertThat(afterEviction).isEqualTo(committed);
+    assertThat(duringOutage).isEqualTo(committed);
+    assertThat(wallet.requireAccount(userId).available()).isEqualTo(Money.krw(90L));
+    assertThat(ledger.findByIdempotencyKey(command.idempotencyKey().value())).hasSize(2);
+  }
+
   private static void await(java.util.concurrent.CountDownLatch latch) {
     try {
       latch.await();


## `test(concurrency): converge one hundred requests under one key`

diff --git a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
index 59fb448..85b957b 100644
--- a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
+++ b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
@@ -28,6 +28,7 @@ import com.sportsbook.wallet.domain.error.WalletRejectedException;
 import com.sportsbook.wallet.integrity.OperationCommitted;
 import com.sportsbook.wallet.service.IdempotencyCache;
 import com.sportsbook.wallet.service.WalletOperationExecutor;
+import com.sportsbook.wallet.service.WalletOperationResult;
 import com.sportsbook.wallet.service.WalletOutcomeResolver;
 import com.sportsbook.wallet.service.WalletService;
 import com.sportsbook.wallet.service.WalletTransferExecutor;
@@ -553,6 +554,50 @@ class WalletPersistenceTest {
     }
   }
 
+  @Test
+  void convergesOneHundredConcurrentRequestsForOneKey() {
+    UUID userId = UUID.fromString("019b76da-a000-7000-8000-000000000019");
+    wallet.openAccount(new OpenAccountCommand(userId, com.sportsbook.protocol.value.Currency.KRW));
+    DepositCommand command =
+        new DepositCommand(userId, Money.krw(77L), IdempotencyKey.of("deposit:hundred-race"));
+    var start = new java.util.concurrent.CountDownLatch(1);
+    var pool = java.util.concurrent.Executors.newFixedThreadPool(20);
+    try {
+      var attempts =
+          java.util.stream.IntStream.range(0, 100)
+              .mapToObj(
+                  ignored ->
+                      CompletableFuture.supplyAsync(
+                          () -> {
+                            await(start);
+                            return retryableAttempt(() -> wallet.deposit(command));
+                          },
+                          pool))
+              .toList();
+      start.countDown();
+      CompletableFuture.allOf(attempts.toArray(CompletableFuture[]::new)).join();
+
+      var initial = attempts.stream().map(CompletableFuture::join).toList();
+      var converged =
+          initial.stream()
+              .map(outcome -> outcome.orElseGet(() -> wallet.deposit(command)))
+              .toList();
+      WalletOperationResult winner = converged.get(0);
+      assertThat(converged).hasSize(100).containsOnly(winner);
+      assertThat(operations.findById(command.idempotencyKey().value()))
+          .get()
+          .satisfies(
+              operation -> {
+                assertThat(operation.status()).isEqualTo(WalletOperationStatus.SUCCEEDED);
+                assertThat(operation.operationGroupId()).isEqualTo(winner.operationGroupId());
+              });
+    } finally {
+      pool.shutdownNow();
+    }
+    assertThat(wallet.requireAccount(userId).available()).isEqualTo(Money.krw(77L));
+    assertThat(ledger.findByIdempotencyKey(command.idempotencyKey().value())).hasSize(2);
+  }
+
   private static void await(java.util.concurrent.CountDownLatch latch) {
     try {
       latch.await();
