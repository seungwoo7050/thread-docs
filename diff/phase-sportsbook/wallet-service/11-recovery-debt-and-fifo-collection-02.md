## `feat(recovery): validate locked FIFO claims`

diff --git a/src/main/java/com/sportsbook/wallet/service/RecoveryClaim.java b/src/main/java/com/sportsbook/wallet/service/RecoveryClaim.java
new file mode 100644
index 0000000..94bf25b
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/service/RecoveryClaim.java
@@ -0,0 +1,38 @@
+package com.sportsbook.wallet.service;
+
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.Account;
+import com.sportsbook.wallet.domain.AdjustmentStatus;
+import com.sportsbook.wallet.domain.WalletAdjustment;
+import com.sportsbook.wallet.domain.WalletCaller;
+import com.sportsbook.wallet.domain.WalletOperation;
+import com.sportsbook.wallet.domain.WalletOperationKind;
+import com.sportsbook.wallet.domain.WalletOperationStatus;
+import java.math.BigInteger;
+
+/** One account-locked FIFO head and its mutable operation outcome. */
+record RecoveryClaim(
+    Account account, WalletAdjustment proof, WalletOperation operation, Money amount) {
+
+  static RecoveryClaim locked(Account account, WalletAdjustment proof, WalletOperation operation) {
+    if (!account.userId().equals(proof.userId())
+        || !proof.userId().equals(operation.userId())
+        || !proof.idempotencyKey().equals(operation.idempotencyKey())) {
+      throw new IllegalStateException("Recovery claim identity is inconsistent");
+    }
+    if (proof.status() != AdjustmentStatus.BLOCKED
+        || proof.deltaAmount() >= 0L
+        || operation.status() != WalletOperationStatus.BLOCKED_FUNDS
+        || operation.kind() != WalletOperationKind.BET_ADJUSTMENT
+        || operation.caller() != WalletCaller.SETTLEMENT) {
+      throw new IllegalStateException("Recovery claim state is inconsistent");
+    }
+    Money amount = new Money(-proof.deltaAmount(), proof.currency());
+    if (!operation.requestAmount().equals(amount)
+        || account.currency() != amount.currency()
+        || account.recoveryDebtAmount().compareTo(BigInteger.valueOf(amount.amount())) < 0) {
+      throw new IllegalStateException("Recovery claim amount is inconsistent");
+    }
+    return new RecoveryClaim(account, proof, operation, amount);
+  }
+}


## `feat(recovery): process locked FIFO heads`

diff --git a/src/main/java/com/sportsbook/wallet/service/RecoveryHeadProcessor.java b/src/main/java/com/sportsbook/wallet/service/RecoveryHeadProcessor.java
new file mode 100644
index 0000000..3f1da27
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/service/RecoveryHeadProcessor.java
@@ -0,0 +1,70 @@
+package com.sportsbook.wallet.service;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.wallet.domain.WalletAdjustment;
+import com.sportsbook.wallet.persistence.WalletAdjustmentRepository;
+import com.sportsbook.wallet.persistence.WalletOperationRepository;
+import java.time.Instant;
+import org.springframework.stereotype.Component;
+
+/** Applies or durably backs off one already locked FIFO recovery claim. */
+@Component
+class RecoveryHeadProcessor {
+  enum Result {
+    APPLIED,
+    DEFERRED_FUNDS
+  }
+
+  private final RecoveryRetryPolicy retries;
+  private final WalletTransferWriter transfers;
+  private final WalletAdjustmentRepository adjustments;
+  private final WalletOperationRepository operations;
+
+  RecoveryHeadProcessor(
+      RecoveryRetryPolicy retries,
+      WalletTransferWriter transfers,
+      WalletAdjustmentRepository adjustments,
+      WalletOperationRepository operations) {
+    this.retries = retries;
+    this.transfers = transfers;
+    this.adjustments = adjustments;
+    this.operations = operations;
+  }
+
+  Result process(RecoveryClaim claim, Instant now) {
+    if (claim.account().available().amount() < claim.amount().amount()) {
+      claim.proof().deferUntil(now, retries.retryAt(now, claim.proof().retryCount()));
+      return Result.DEFERRED_FUNDS;
+    }
+    WalletTransferPlan plan = AdjustmentTransfers.recover(claim.account(), claim.proof(), now);
+    WalletTransferReceipt receipt =
+        transfers.writeReceipt(
+            plan.destination(),
+            plan.source(),
+            claim.amount(),
+            plan.reason(),
+            IdempotencyKey.of(claim.proof().idempotencyKey()),
+            claim.proof().userId(),
+            now);
+    claim.proof().completeRecovery(receipt.result().operationGroupId(), now);
+    claim.operation().completeBlocked(receipt.result().operationGroupId(), now);
+    wakeOrVerifyEmpty(claim, now);
+    return Result.APPLIED;
+  }
+
+  private void wakeOrVerifyEmpty(RecoveryClaim claim, Instant now) {
+    adjustments.flush();
+    operations.flush();
+    WalletAdjustment next =
+        adjustments.findOldestBlockedForUpdate(claim.account().userId()).orElse(null);
+    if (claim.account().isOutboundFrozen() && next == null) {
+      throw new IllegalStateException("Recovery debt has no remaining FIFO head");
+    }
+    if (!claim.account().isOutboundFrozen() && next != null) {
+      throw new IllegalStateException("Recovery FIFO head has no remaining debt");
+    }
+    if (next != null) {
+      next.wake(now);
+    }
+  }
+}


## `feat(recovery): claim one FIFO head transactionally`

diff --git a/src/main/java/com/sportsbook/wallet/service/RecoveryWorker.java b/src/main/java/com/sportsbook/wallet/service/RecoveryWorker.java
new file mode 100644
index 0000000..838bec7
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/service/RecoveryWorker.java
@@ -0,0 +1,75 @@
+package com.sportsbook.wallet.service;
+
+import com.sportsbook.wallet.domain.Account;
+import com.sportsbook.wallet.domain.WalletAdjustment;
+import com.sportsbook.wallet.domain.WalletOperation;
+import com.sportsbook.wallet.persistence.AccountRepository;
+import com.sportsbook.wallet.persistence.DatabaseClock;
+import com.sportsbook.wallet.persistence.WalletAdjustmentRepository;
+import com.sportsbook.wallet.persistence.WalletOperationRepository;
+import java.util.Optional;
+import org.springframework.stereotype.Component;
+import org.springframework.transaction.PlatformTransactionManager;
+import org.springframework.transaction.TransactionDefinition;
+import org.springframework.transaction.support.TransactionTemplate;
+
+/** Claims and processes at most one due account in its own database transaction. */
+@Component
+public class RecoveryWorker {
+  private static final int TRANSACTION_TIMEOUT_SECONDS = 5;
+
+  public enum Result {
+    NO_WORK,
+    APPLIED,
+    DEFERRED_FUNDS
+  }
+
+  private final AccountRepository accounts;
+  private final WalletAdjustmentRepository adjustments;
+  private final WalletOperationRepository operations;
+  private final DatabaseClock databaseClock;
+  private final RecoveryHeadProcessor processor;
+  private final TransactionTemplate transaction;
+
+  public RecoveryWorker(
+      AccountRepository accounts,
+      WalletAdjustmentRepository adjustments,
+      WalletOperationRepository operations,
+      DatabaseClock databaseClock,
+      RecoveryHeadProcessor processor,
+      PlatformTransactionManager transactionManager) {
+    this.accounts = accounts;
+    this.adjustments = adjustments;
+    this.operations = operations;
+    this.databaseClock = databaseClock;
+    this.processor = processor;
+    this.transaction = new TransactionTemplate(transactionManager);
+    this.transaction.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
+    this.transaction.setTimeout(TRANSACTION_TIMEOUT_SECONDS);
+  }
+
+  public Result recoverOne() {
+    return transaction.execute(status -> recoverLocked());
+  }
+
+  private Result recoverLocked() {
+    Optional<Account> candidate = accounts.lockNextDueRecoveryAccount();
+    if (candidate.isEmpty()) {
+      return Result.NO_WORK;
+    }
+    Account account = candidate.orElseThrow();
+    WalletAdjustment proof =
+        adjustments
+            .findOldestBlockedForUpdate(account.userId())
+            .orElseThrow(() -> new IllegalStateException("Recovery debt has no FIFO head"));
+    WalletOperation operation =
+        operations
+            .findByIdForUpdate(proof.idempotencyKey())
+            .orElseThrow(() -> new IllegalStateException("Recovery proof has no operation"));
+    RecoveryClaim claim = RecoveryClaim.locked(account, proof, operation);
+    return switch (processor.process(claim, databaseClock.now())) {
+      case APPLIED -> Result.APPLIED;
+      case DEFERRED_FUNDS -> Result.DEFERRED_FUNDS;
+    };
+  }
+}


## `test(recovery): restart after atomic rollback`

diff --git a/src/test/java/com/sportsbook/wallet/service/RecoveryWorkerPersistenceTest.java b/src/test/java/com/sportsbook/wallet/service/RecoveryWorkerPersistenceTest.java
index c9c46c9..ca5d679 100644
--- a/src/test/java/com/sportsbook/wallet/service/RecoveryWorkerPersistenceTest.java
+++ b/src/test/java/com/sportsbook/wallet/service/RecoveryWorkerPersistenceTest.java
@@ -1,12 +1,14 @@
 package com.sportsbook.wallet.service;
 
 import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
 
 import com.sportsbook.protocol.value.IdempotencyKey;
 import com.sportsbook.protocol.value.Money;
 import com.sportsbook.wallet.domain.AdjustmentStatus;
 import com.sportsbook.wallet.domain.WalletOperation;
 import com.sportsbook.wallet.domain.WalletOperationStatus;
+import com.sportsbook.wallet.integrity.OperationCommitted;
 import com.sportsbook.wallet.persistence.AccountRepository;
 import com.sportsbook.wallet.persistence.LedgerEntryRepository;
 import com.sportsbook.wallet.persistence.WalletAdjustmentRepository;
@@ -18,9 +20,12 @@ import java.math.BigInteger;
 import java.util.UUID;
 import java.util.concurrent.CompletableFuture;
 import java.util.concurrent.CountDownLatch;
+import java.util.concurrent.atomic.AtomicBoolean;
 import org.junit.jupiter.api.Test;
 import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.context.annotation.Import;
+import org.springframework.context.event.EventListener;
 import org.springframework.test.context.DynamicPropertyRegistry;
 import org.springframework.test.context.DynamicPropertySource;
 import org.testcontainers.containers.PostgreSQLContainer;
@@ -29,6 +34,7 @@ import org.testcontainers.junit.jupiter.Testcontainers;
 
 @SpringBootTest(properties = "wallet.outbox.scheduling-enabled=false")
 @Testcontainers
+@Import(RecoveryWorkerPersistenceTest.CommitFault.class)
 class RecoveryWorkerPersistenceTest {
   @Container
   static final PostgreSQLContainer<?> POSTGRES = new PostgreSQLContainer<>("postgres:16-alpine");
@@ -40,6 +46,7 @@ class RecoveryWorkerPersistenceTest {
   @Autowired WalletAdjustmentRepository adjustments;
   @Autowired WalletOperationRepository operations;
   @Autowired LedgerEntryRepository ledger;
+  @Autowired CommitFault commitFault;
 
   @DynamicPropertySource
   static void databaseProperties(DynamicPropertyRegistry registry) {
@@ -126,6 +133,46 @@ class RecoveryWorkerPersistenceTest {
         .isEqualTo(BigInteger.ZERO);
   }
 
+  @Test
+  void rollbackLeavesBlockedStateForRestartedRecovery() {
+    UUID userId = UUID.fromString("019b76da-a000-7000-8000-000000000196");
+    UUID revisionId = UUID.fromString("019b76da-a000-7000-8000-000000000197");
+    wallet.openAccount(new OpenAccountCommand(userId, Money.krw(0L).currency()));
+    wallet.deposit(
+        new DepositCommand(userId, Money.krw(200L), IdempotencyKey.of("deposit:restart-seed")));
+    AdjustmentCommand command =
+        new AdjustmentCommand(
+            revisionId,
+            UUID.fromString("019b76da-a000-7000-8000-000000000198"),
+            1L,
+            userId,
+            Money.krw(500L),
+            Money.krw(200L),
+            IdempotencyKey.of("settlement:revision:" + revisionId));
+    adjustmentService.adjust(command);
+    wallet.deposit(
+        new DepositCommand(userId, Money.krw(100L), IdempotencyKey.of("deposit:restart-wake")));
+    commitFault.failNext();
+
+    assertThatThrownBy(worker::recoverOne).isInstanceOf(IllegalStateException.class);
+
+    assertThat(adjustments.findById(revisionId).orElseThrow().status())
+        .isEqualTo(AdjustmentStatus.BLOCKED);
+    assertThat(operations.findById(command.idempotencyKey().value()).orElseThrow().status())
+        .isEqualTo(WalletOperationStatus.BLOCKED_FUNDS);
+    assertThat(ledger.findByIdempotencyKey(command.idempotencyKey().value())).isEmpty();
+    assertThat(accounts.findById(userId).orElseThrow())
+        .satisfies(
+            account -> {
+              assertThat(account.available()).isEqualTo(Money.krw(300L));
+              assertThat(account.recoveryDebtAmount()).isEqualTo(BigInteger.valueOf(300L));
+              assertThat(account.isOutboundFrozen()).isTrue();
+            });
+    assertThat(worker.recoverOne()).isEqualTo(RecoveryWorker.Result.APPLIED);
+    assertThat(worker.recoverOne()).isEqualTo(RecoveryWorker.Result.NO_WORK);
+    assertThat(ledger.findByIdempotencyKey(command.idempotencyKey().value())).hasSize(2);
+  }
+
   private static void await(CountDownLatch latch) {
     try {
       latch.await();
@@ -134,4 +181,19 @@ class RecoveryWorkerPersistenceTest {
       throw new IllegalStateException(interrupted);
     }
   }
+
+  static final class CommitFault {
+    private final AtomicBoolean armed = new AtomicBoolean();
+
+    void failNext() {
+      armed.set(true);
+    }
+
+    @EventListener
+    void afterLedgerWrite(OperationCommitted ignored) {
+      if (armed.compareAndSet(true, false)) {
+        throw new IllegalStateException("injected commit fault");
+      }
+    }
+  }
 }


## `test(recovery): serialize competing collectors`

diff --git a/src/test/java/com/sportsbook/wallet/service/RecoveryWorkerPersistenceTest.java b/src/test/java/com/sportsbook/wallet/service/RecoveryWorkerPersistenceTest.java
index 1a5a30a..c9c46c9 100644
--- a/src/test/java/com/sportsbook/wallet/service/RecoveryWorkerPersistenceTest.java
+++ b/src/test/java/com/sportsbook/wallet/service/RecoveryWorkerPersistenceTest.java
@@ -16,6 +16,8 @@ import com.sportsbook.wallet.service.command.DepositCommand;
 import com.sportsbook.wallet.service.command.OpenAccountCommand;
 import java.math.BigInteger;
 import java.util.UUID;
+import java.util.concurrent.CompletableFuture;
+import java.util.concurrent.CountDownLatch;
 import org.junit.jupiter.api.Test;
 import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.boot.test.context.SpringBootTest;
@@ -84,4 +86,52 @@ class RecoveryWorkerPersistenceTest {
               assertThat(account.isOutboundFrozen()).isFalse();
             });
   }
+
+  @Test
+  void twoWorkersCannotCollectTheSameHead() {
+    UUID userId = UUID.fromString("019b76da-a000-7000-8000-000000000193");
+    UUID revisionId = UUID.fromString("019b76da-a000-7000-8000-000000000194");
+    wallet.openAccount(new OpenAccountCommand(userId, Money.krw(0L).currency()));
+    wallet.deposit(
+        new DepositCommand(userId, Money.krw(200L), IdempotencyKey.of("deposit:replica-seed")));
+    AdjustmentCommand command =
+        new AdjustmentCommand(
+            revisionId,
+            UUID.fromString("019b76da-a000-7000-8000-000000000195"),
+            1L,
+            userId,
+            Money.krw(600L),
+            Money.krw(300L),
+            IdempotencyKey.of("settlement:revision:" + revisionId));
+    adjustmentService.adjust(command);
+    wallet.deposit(
+        new DepositCommand(userId, Money.krw(100L), IdempotencyKey.of("deposit:replica-wake")));
+    CountDownLatch start = new CountDownLatch(1);
+    java.util.function.Supplier<RecoveryWorker.Result> recover =
+        () -> {
+          await(start);
+          return worker.recoverOne();
+        };
+    CompletableFuture<RecoveryWorker.Result> first = CompletableFuture.supplyAsync(recover);
+    CompletableFuture<RecoveryWorker.Result> second = CompletableFuture.supplyAsync(recover);
+
+    start.countDown();
+
+    assertThat(java.util.List.of(first.join(), second.join()))
+        .containsExactlyInAnyOrder(RecoveryWorker.Result.APPLIED, RecoveryWorker.Result.NO_WORK);
+    assertThat(ledger.findByIdempotencyKey(command.idempotencyKey().value())).hasSize(2);
+    assertThat(adjustments.findById(revisionId).orElseThrow().status())
+        .isEqualTo(AdjustmentStatus.APPLIED);
+    assertThat(accounts.findById(userId).orElseThrow().recoveryDebtAmount())
+        .isEqualTo(BigInteger.ZERO);
+  }
+
+  private static void await(CountDownLatch latch) {
+    try {
+      latch.await();
+    } catch (InterruptedException interrupted) {
+      Thread.currentThread().interrupt();
+      throw new IllegalStateException(interrupted);
+    }
+  }
 }


## `feat(recovery): schedule automatic collection`

diff --git a/src/main/java/com/sportsbook/wallet/service/RecoveryScheduler.java b/src/main/java/com/sportsbook/wallet/service/RecoveryScheduler.java
new file mode 100644
index 0000000..3982055
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/service/RecoveryScheduler.java
@@ -0,0 +1,21 @@
+package com.sportsbook.wallet.service;
+
+import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
+import org.springframework.scheduling.annotation.Scheduled;
+import org.springframework.stereotype.Component;
+
+/** Periodically gives one due FIFO head its own bounded recovery transaction. */
+@Component
+@ConditionalOnProperty(name = "wallet.recovery.scheduling-enabled", havingValue = "true")
+public class RecoveryScheduler {
+  private final RecoveryWorker worker;
+
+  public RecoveryScheduler(RecoveryWorker worker) {
+    this.worker = worker;
+  }
+
+  @Scheduled(fixedDelayString = "${wallet.recovery.poll-interval:PT1S}")
+  public void poll() {
+    worker.recoverOne();
+  }
+}


## `test(gate): recover blocked adjustments end to end`

diff --git a/src/test/java/com/sportsbook/wallet/smoke/WalletSmokeTest.java b/src/test/java/com/sportsbook/wallet/smoke/WalletSmokeTest.java
index 0c6cf98..1b4ad9b 100644
--- a/src/test/java/com/sportsbook/wallet/smoke/WalletSmokeTest.java
+++ b/src/test/java/com/sportsbook/wallet/smoke/WalletSmokeTest.java
@@ -7,6 +7,7 @@ import com.sportsbook.wallet.domain.WalletCaller;
 import com.sportsbook.wallet.outbox.KafkaOutboxDispatcher;
 import com.sportsbook.wallet.outbox.WalletEventFactory;
 import com.sportsbook.wallet.persistence.OutboxDeliveryRepository;
+import com.sportsbook.wallet.service.RecoveryWorker;
 import java.nio.charset.StandardCharsets;
 import java.time.Duration;
 import java.util.List;
@@ -35,6 +36,7 @@ class WalletSmokeTest extends WalletSmokeFixture {
   @Autowired StringRedisTemplate redis;
   @Autowired OutboxDeliveryRepository outbox;
   @Autowired KafkaOutboxDispatcher dispatcher;
+  @Autowired RecoveryWorker recovery;
 
   @Test
   void servesAuthenticatedDurableReplayAcrossPostgresAndRedis() throws Exception {
@@ -145,6 +147,76 @@ class WalletSmokeTest extends WalletSmokeFixture {
     }
   }
 
+  @Test
+  void wakesAndRecoversBlockedAdjustments() throws Exception {
+    UUID userId = UUID.fromString("019b783d-1000-7000-8000-000000000004");
+    UUID revisionId = UUID.fromString("019b783d-1000-7000-8000-000000000005");
+    UUID betId = UUID.fromString("019b783d-1000-7000-8000-000000000006");
+    String key = "settlement:revision:" + revisionId;
+    String proofPath = "/internal/v1/wallet/transactions/adjustment/" + revisionId;
+    String account = "{\"userId\":\"" + userId + "\",\"currency\":\"KRW\"}";
+    String adjustment =
+        """
+        {"revisionId":"%s","betId":"%s","revisionNumber":1,"userId":"%s",
+        "previousPayout":{"amount":500,"currency":"KRW"},
+        "newPayout":{"amount":200,"currency":"KRW"}}
+        """
+            .formatted(revisionId, betId, userId);
+    assertThat(
+            request(HttpMethod.POST, ACCOUNT_PATH, WalletCaller.PLATFORM, null, account)
+                .getStatusCode())
+        .isEqualTo(HttpStatus.OK);
+
+    var blocked =
+        request(
+            HttpMethod.POST,
+            "/internal/v1/wallet/transactions/adjustment",
+            WalletCaller.SETTLEMENT,
+            key,
+            adjustment);
+    assertThat(blocked.getStatusCode()).isEqualTo(HttpStatus.ACCEPTED);
+    assertThat(blocked.getHeaders().getLocation()).hasToString(proofPath);
+    assertThat(json.readTree(blocked.getBody()).path("status").textValue()).isEqualTo("BLOCKED");
+    assertThat(count("ledger_entry", key)).isZero();
+
+    var funded =
+        request(
+            HttpMethod.POST,
+            "/internal/v1/wallet/transactions/deposit",
+            WalletCaller.PLATFORM,
+            "smoke:recovery:deposit",
+            transaction(userId, 300L));
+    assertThat(funded.getStatusCode()).isEqualTo(HttpStatus.OK);
+    assertThat(recovery.recoverOne()).isEqualTo(RecoveryWorker.Result.APPLIED);
+
+    var proof = request(HttpMethod.GET, proofPath, WalletCaller.SETTLEMENT, null, null);
+    var recovered = json.readTree(proof.getBody());
+    assertThat(proof.getStatusCode()).isEqualTo(HttpStatus.OK);
+    assertThat(recovered.path("status").textValue()).isEqualTo("APPLIED");
+    assertThat(recovered.path("deltaAmount").longValue()).isEqualTo(-300L);
+    assertThat(recovered.path("operationGroupId").isTextual()).isTrue();
+    assertThat(recovered.path("appliedAt").isTextual()).isTrue();
+    assertThat(recovered.path("nextAttemptAt").isNull()).isTrue();
+    assertThat(
+            jdbc.queryForObject(
+                """
+                SELECT recovery_debt_amount=0 AND recovery_frozen_at IS NULL
+                FROM account WHERE user_id=?
+                """,
+                Boolean.class,
+                userId))
+        .isTrue();
+    assertThat(
+            jdbc.queryForObject(
+                """
+                SELECT COUNT(*) FROM ledger_entry
+                WHERE idempotency_key=? AND reason='BET_ADJUSTMENT'
+                """,
+                Integer.class,
+                key))
+        .isEqualTo(2);
+  }
+
   private int count(String table, String key) {
     return jdbc.queryForObject(
         "SELECT COUNT(*) FROM " + table + " WHERE idempotency_key=?", Integer.class, key);
