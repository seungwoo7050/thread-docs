## `test(debit): serialize concurrent business rejections`

diff --git a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
index 98cec16..7415cb1 100644
--- a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
+++ b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
@@ -713,6 +713,77 @@ class WalletPersistenceTest {
         .isEqualTo(1L);
   }
 
+  @Test
+  void serializesOneHundredConcurrentBusinessRejections() {
+    UUID userId = UUID.fromString("019b76da-a000-7000-8000-00000000001d");
+    wallet.openAccount(new OpenAccountCommand(userId, com.sportsbook.protocol.value.Currency.KRW));
+    DebitCommand command =
+        new DebitCommand(userId, Money.krw(77L), IdempotencyKey.of("debit:hundred-rejections"));
+    var start = new java.util.concurrent.CountDownLatch(1);
+    var pool = java.util.concurrent.Executors.newFixedThreadPool(20);
+    Supplier<WalletRejectedException> reject =
+        () -> {
+          try {
+            wallet.debit(command);
+            throw new AssertionError("Debit unexpectedly succeeded");
+          } catch (WalletRejectedException rejected) {
+            return rejected;
+          }
+        };
+    try {
+      var attempts =
+          java.util.stream.IntStream.range(0, 100)
+              .mapToObj(
+                  ignored ->
+                      CompletableFuture.supplyAsync(
+                          () -> {
+                            await(start);
+                            return retryableAttempt(reject);
+                          },
+                          pool))
+              .toList();
+      start.countDown();
+      CompletableFuture.allOf(attempts.toArray(CompletableFuture[]::new)).join();
+
+      var initial = attempts.stream().map(CompletableFuture::join).toList();
+      var converged = initial.stream().map(outcome -> outcome.orElseGet(reject)).toList();
+      WalletRejectedException winner = converged.get(0);
+      assertThat(converged)
+          .hasSize(100)
+          .allSatisfy(
+              rejection ->
+                  assertThat(rejection.failure())
+                      .usingRecursiveComparison()
+                      .isEqualTo(winner.failure()));
+      assertThat(operations.findById(command.idempotencyKey().value()))
+          .get()
+          .satisfies(
+              operation -> {
+                assertThat(operation.status()).isEqualTo(WalletOperationStatus.REJECTED);
+                assertThat(operation.failure())
+                    .usingRecursiveComparison()
+                    .isEqualTo(winner.failure());
+              });
+    } finally {
+      pool.shutdownNow();
+    }
+    assertThat(wallet.requireAccount(userId).available()).isEqualTo(Money.krw(0L));
+    assertThat(wallet.requireAccount(userId).locked()).isEqualTo(Money.krw(0L));
+    assertThat(ledger.findByIdempotencyKey(command.idempotencyKey().value())).isEmpty();
+    assertThat(outboxFor(command.idempotencyKey())).hasSize(1);
+    assertThat(
+            jdbc.queryForObject(
+                "SELECT last_sequence FROM outbox_stream WHERE topic=? AND partition_key=?",
+                Long.class,
+                com.sportsbook.wallet.outbox.WalletEventFactory.DEBIT_FAILED_TOPIC,
+                userId.toString()))
+        .isEqualTo(1L);
+    assertThat(operations.findById(command.idempotencyKey().value()))
+        .get()
+        .extracting(WalletOperation::status)
+        .isEqualTo(WalletOperationStatus.REJECTED);
+  }
+
   private static void await(java.util.concurrent.CountDownLatch latch) {
     try {
       latch.await();


## `feat(debit): recover committed debits by bet id`

diff --git a/src/main/java/com/sportsbook/wallet/service/WalletService.java b/src/main/java/com/sportsbook/wallet/service/WalletService.java
index 8a98cc2..35580be 100644
--- a/src/main/java/com/sportsbook/wallet/service/WalletService.java
+++ b/src/main/java/com/sportsbook/wallet/service/WalletService.java
@@ -10,6 +10,7 @@ import com.sportsbook.wallet.domain.WalletOperationKind;
 import com.sportsbook.wallet.domain.error.AccountNotFoundException;
 import com.sportsbook.wallet.domain.error.CurrencyMismatchException;
 import com.sportsbook.wallet.persistence.AccountRepository;
+import com.sportsbook.wallet.persistence.WalletOperationRepository;
 import com.sportsbook.wallet.service.command.CreditCommand;
 import com.sportsbook.wallet.service.command.CreditReason;
 import com.sportsbook.wallet.service.command.DebitCommand;
@@ -20,6 +21,7 @@ import com.sportsbook.wallet.service.command.WithdrawCommand;
 import java.time.Clock;
 import java.time.Instant;
 import java.time.temporal.ChronoUnit;
+import java.util.Objects;
 import java.util.Optional;
 import java.util.UUID;
 import org.springframework.dao.DataIntegrityViolationException;
@@ -34,16 +36,22 @@ public class WalletService {
   private final TransactionTemplate writeTransaction;
   private final Clock clock;
   private final WalletTransferExecutor transferExecutor;
+  private final WalletOperationRepository operations;
+  private final WalletOutcomeResolver outcomes;
 
   public WalletService(
       AccountRepository accounts,
       TransactionTemplate writeTransaction,
       Clock clock,
-      WalletTransferExecutor transferExecutor) {
+      WalletTransferExecutor transferExecutor,
+      WalletOperationRepository operations,
+      WalletOutcomeResolver outcomes) {
     this.accounts = accounts;
     this.writeTransaction = writeTransaction;
     this.clock = clock;
     this.transferExecutor = transferExecutor;
+    this.operations = operations;
+    this.outcomes = outcomes;
   }
 
   public Account openAccount(OpenAccountCommand command) {
@@ -117,6 +125,18 @@ public class WalletService {
         });
   }
 
+  /** Returns the durable debit outcome for the canonical bet-id idempotency key. */
+  public Optional<WalletOperationResult> findDebit(UUID betId) {
+    String operationKey = Objects.requireNonNull(betId, "betId").toString();
+    return operations
+        .findById(operationKey)
+        .filter(
+            operation ->
+                operation.caller() == WalletCaller.BETTING
+                    && operation.kind() == WalletOperationKind.BET_DEBIT)
+        .map(outcomes::resolve);
+  }
+
   public WalletOperationResult credit(WalletCaller caller, CreditCommand command) {
     return transferExecutor.executeCredit(
         caller,


## `test(events): prove debited ledger row identity`

diff --git a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
index 0cb2419..98cec16 100644
--- a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
+++ b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
@@ -582,6 +582,32 @@ class WalletPersistenceTest {
         .isEqualTo(com.sportsbook.wallet.outbox.WalletEventFactory.DEBITED_TOPIC);
   }
 
+  @Test
+  void debitsReferenceTheUserLockedLedgerRowRatherThanTheOperationGroup() {
+    UUID userId = UUID.fromString("019b76da-a000-7000-8000-000000000023");
+    wallet.openAccount(new OpenAccountCommand(userId, com.sportsbook.protocol.value.Currency.KRW));
+    wallet.deposit(
+        new DepositCommand(userId, Money.krw(100L), IdempotencyKey.of("deposit:debit-proof")));
+    DebitCommand command =
+        new DebitCommand(userId, Money.krw(60L), IdempotencyKey.of("debit:ledger-proof"));
+
+    var result = wallet.debit(command);
+    var message = outboxFor(command.idempotencyKey()).get(0);
+    var event =
+        com.sportsbook.wallet.outbox.AvroSerializer.deserialize(
+            message.payload(), com.sportsbook.protocol.event.WalletDebited.class);
+    LedgerEntry userSide =
+        ledger.findByIdempotencyKey(command.idempotencyKey().value()).stream()
+            .filter(entry -> entry.side() == com.sportsbook.wallet.domain.LedgerSide.DEBIT)
+            .findFirst()
+            .orElseThrow();
+
+    assertThat(userSide.accountId()).isEqualTo(userId);
+    assertThat(userSide.bucket()).isEqualTo(BalanceBucket.LOCKED);
+    assertThat(event.getLedgerTxId()).isEqualTo(userSide.entryId().toString());
+    assertThat(event.getLedgerTxId()).isNotEqualTo(result.operationGroupId().toString());
+  }
+
   @Test
   void exactlyReplaysATerminalDebitFailureAfterFundsArrive() {
     UUID userId = UUID.fromString("019b76da-a000-7000-8000-00000000001c");


## `test(debit): roll back failed outbox persistence`

diff --git a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
index 0d18139..0cb2419 100644
--- a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
+++ b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
@@ -3,6 +3,7 @@ package com.sportsbook.wallet.persistence;
 import static org.assertj.core.api.Assertions.assertThat;
 import static org.assertj.core.api.Assertions.assertThatThrownBy;
 import static org.assertj.core.api.Assertions.catchThrowableOfType;
+import static org.mockito.ArgumentMatchers.any;
 import static org.mockito.Mockito.doThrow;
 import static org.mockito.Mockito.reset;
 import static org.mockito.Mockito.when;
@@ -51,6 +52,7 @@ import org.junit.jupiter.api.Test;
 import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
 import org.springframework.boot.test.mock.mockito.MockBean;
+import org.springframework.boot.test.mock.mockito.SpyBean;
 import org.springframework.context.annotation.Import;
 import org.springframework.context.event.EventListener;
 import org.springframework.jdbc.core.JdbcTemplate;
@@ -86,7 +88,7 @@ class WalletPersistenceTest {
   @Autowired AccountRepository accounts;
   @Autowired LedgerEntryRepository ledger;
   @Autowired WalletOperationRepository operations;
-  @Autowired OutboxEventRepository outboxEvents;
+  @SpyBean OutboxEventRepository outboxEvents;
   @Autowired IdempotencyKeyLock idempotencyLocks;
   @Autowired WalletService wallet;
   @Autowired CommitFault commitFault;
@@ -649,6 +651,42 @@ class WalletPersistenceTest {
     assertThat(ledger.findByIdempotencyKey(command.idempotencyKey().value())).hasSize(2);
   }
 
+  @Test
+  void rollsBackDebitWhenOutboxPersistenceFails() {
+    UUID userId = UUID.fromString("019b76da-a000-7000-8000-00000000007f");
+    wallet.openAccount(new OpenAccountCommand(userId, com.sportsbook.protocol.value.Currency.KRW));
+    wallet.deposit(
+        new DepositCommand(userId, Money.krw(100L), IdempotencyKey.of("deposit:outbox-failure")));
+    DebitCommand command =
+        new DebitCommand(userId, Money.krw(60L), IdempotencyKey.of("debit:outbox-failure"));
+    doThrow(new IllegalStateException("outbox persistence unavailable"))
+        .when(outboxEvents)
+        .save(any(com.sportsbook.wallet.outbox.OutboxEvent.class));
+
+    assertThatThrownBy(() -> wallet.debit(command))
+        .isInstanceOf(IllegalStateException.class)
+        .hasMessage("outbox persistence unavailable");
+
+    assertThat(wallet.requireAccount(userId).available()).isEqualTo(Money.krw(100L));
+    assertThat(wallet.requireAccount(userId).locked()).isEqualTo(Money.krw(0L));
+    assertThat(ledger.findByIdempotencyKey(command.idempotencyKey().value())).isEmpty();
+    assertThat(operations.findById(command.idempotencyKey().value())).isEmpty();
+    assertThat(outboxFor(command.idempotencyKey())).isEmpty();
+    assertThat(
+            jdbc.queryForObject(
+                "SELECT count(*) FROM outbox_stream WHERE topic=? AND partition_key=?",
+                Integer.class,
+                com.sportsbook.wallet.outbox.WalletEventFactory.DEBITED_TOPIC,
+                userId.toString()))
+        .isZero();
+    reset(outboxEvents);
+    wallet.debit(command);
+    assertThat(outboxFor(command.idempotencyKey()))
+        .singleElement()
+        .extracting(com.sportsbook.wallet.outbox.OutboxEvent::streamSequence)
+        .isEqualTo(1L);
+  }
+
   private static void await(java.util.concurrent.CountDownLatch latch) {
     try {
       latch.await();


## `test(debit): reject missing and wrong-kind results`

diff --git a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
index ec2eafc..beb026a 100644
--- a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
+++ b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
@@ -867,6 +867,44 @@ class WalletPersistenceTest {
     assertThat(outboxFor(rejected.idempotencyKey())).isEmpty();
   }
 
+  @Test
+  void findsOnlyCanonicalBetDebitOutcomesWithoutRepeatingTransfers() {
+    UUID userId = UUID.fromString("019b76da-a000-7000-8000-000000000022");
+    UUID committedBetId = UUID.fromString("019b76da-b000-7000-8000-000000000001");
+    wallet.openAccount(new OpenAccountCommand(userId, com.sportsbook.protocol.value.Currency.KRW));
+    wallet.deposit(
+        new DepositCommand(userId, Money.krw(100L), IdempotencyKey.of("deposit:debit-lookup")));
+    DebitCommand committedCommand =
+        new DebitCommand(userId, Money.krw(60L), IdempotencyKey.of(committedBetId.toString()));
+    var committed = wallet.debit(committedCommand);
+    long ledgerCount = ledger.count();
+    long outboxCount = outboxEvents.count();
+
+    assertThat(wallet.findDebit(committedBetId)).contains(committed);
+    assertThat(ledger.count()).isEqualTo(ledgerCount);
+    assertThat(outboxEvents.count()).isEqualTo(outboxCount);
+
+    UUID rejectedBetId = UUID.fromString("019b76da-b000-7000-8000-000000000002");
+    DebitCommand rejectedCommand =
+        new DebitCommand(userId, Money.krw(100L), IdempotencyKey.of(rejectedBetId.toString()));
+    WalletRejectedException rejected =
+        catchThrowableOfType(() -> wallet.debit(rejectedCommand), WalletRejectedException.class);
+
+    assertThatThrownBy(() -> wallet.findDebit(rejectedBetId))
+        .isInstanceOfSatisfying(
+            WalletRejectedException.class,
+            replay -> {
+              assertThat(replay.failure().code()).isEqualTo(rejected.failure().code());
+              assertThat(replay.failure().detail()).isEqualTo(rejected.failure().detail());
+            });
+    assertThat(wallet.findDebit(UUID.fromString("019b76da-b000-7000-8000-000000000003"))).isEmpty();
+
+    UUID depositKey = UUID.fromString("019b76da-b000-7000-8000-000000000004");
+    wallet.deposit(
+        new DepositCommand(userId, Money.krw(1L), IdempotencyKey.of(depositKey.toString())));
+    assertThat(wallet.findDebit(depositKey)).isEmpty();
+  }
+
   @Test
   void convergesOneHundredConcurrentRequestsForOneKey() {
     UUID userId = UUID.fromString("019b76da-a000-7000-8000-000000000019");
