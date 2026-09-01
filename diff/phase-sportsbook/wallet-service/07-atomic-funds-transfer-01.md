# 원자적 자금 이체

## `feat(repository): lock accounts for balance writes`

diff --git a/src/main/java/com/sportsbook/wallet/persistence/AccountRepository.java b/src/main/java/com/sportsbook/wallet/persistence/AccountRepository.java
new file mode 100644
index 0000000..f22b353
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/persistence/AccountRepository.java
@@ -0,0 +1,18 @@
+package com.sportsbook.wallet.persistence;
+
+import com.sportsbook.wallet.domain.Account;
+import jakarta.persistence.LockModeType;
+import java.util.Optional;
+import java.util.UUID;
+import org.springframework.data.jpa.repository.JpaRepository;
+import org.springframework.data.jpa.repository.Lock;
+import org.springframework.data.jpa.repository.Query;
+import org.springframework.data.repository.query.Param;
+
+/** Account storage. Every balance write enters through the pessimistic lock query. */
+public interface AccountRepository extends JpaRepository<Account, UUID> {
+
+  @Lock(LockModeType.PESSIMISTIC_WRITE)
+  @Query("select account from Account account where account.userId = :userId")
+  Optional<Account> findByUserIdForUpdate(@Param("userId") UUID userId);
+}


## `test(persistence): verify pessimistic account row locks`

diff --git a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
index 9f25867..2c8b191 100644
--- a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
+++ b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
@@ -3,8 +3,11 @@ package com.sportsbook.wallet.persistence;
 import static org.assertj.core.api.Assertions.assertThat;
 import static org.assertj.core.api.Assertions.assertThatThrownBy;
 
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.Account;
 import java.math.BigDecimal;
 import java.math.BigInteger;
+import java.time.Instant;
 import java.util.UUID;
 import org.junit.jupiter.api.Test;
 import org.springframework.beans.factory.annotation.Autowired;
@@ -14,6 +17,7 @@ import org.springframework.test.context.DynamicPropertyRegistry;
 import org.springframework.test.context.DynamicPropertySource;
 import org.springframework.transaction.annotation.Propagation;
 import org.springframework.transaction.annotation.Transactional;
+import org.springframework.transaction.support.DefaultTransactionDefinition;
 import org.testcontainers.containers.PostgreSQLContainer;
 import org.testcontainers.junit.jupiter.Container;
 import org.testcontainers.junit.jupiter.Testcontainers;
@@ -26,6 +30,9 @@ class WalletPersistenceTest {
   static final PostgreSQLContainer<?> POSTGRES = new PostgreSQLContainer<>("postgres:16-alpine");
 
   @Autowired JdbcTemplate jdbc;
+  @Autowired AccountRepository accounts;
+  @Autowired javax.sql.DataSource dataSource;
+  @Autowired org.springframework.transaction.PlatformTransactionManager transactions;
 
   @DynamicPropertySource
   static void databaseProperties(DynamicPropertyRegistry registry) {
@@ -92,4 +99,27 @@ class WalletPersistenceTest {
                     userId))
         .isInstanceOf(org.springframework.dao.DataIntegrityViolationException.class);
   }
+
+  @Test
+  void preventsAConcurrentWriterFromTakingTheAccountLock() throws Exception {
+    UUID userId = UUID.fromString("019b76da-a000-7000-8000-000000000011");
+    accounts.saveAndFlush(Account.openFor(userId, Money.krw(0L).currency(), Instant.now()));
+    var status = transactions.getTransaction(new DefaultTransactionDefinition());
+    try {
+      accounts.findByUserIdForUpdate(userId).orElseThrow();
+      try (var connection = dataSource.getConnection()) {
+        connection.setAutoCommit(false);
+        try (var timeout = connection.createStatement();
+            var lock =
+                connection.prepareStatement(
+                    "SELECT user_id FROM account WHERE user_id = ? FOR UPDATE")) {
+          timeout.execute("SET LOCAL lock_timeout = '100ms'");
+          lock.setObject(1, userId);
+          assertThatThrownBy(lock::executeQuery).isInstanceOf(java.sql.SQLException.class);
+        }
+      }
+    } finally {
+      transactions.rollback(status);
+    }
+  }
 }


## `feat(service): wire transaction and clock boundaries`

diff --git a/src/main/java/com/sportsbook/wallet/config/WalletInfrastructureConfig.java b/src/main/java/com/sportsbook/wallet/config/WalletInfrastructureConfig.java
new file mode 100644
index 0000000..422356f
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/config/WalletInfrastructureConfig.java
@@ -0,0 +1,22 @@
+package com.sportsbook.wallet.config;
+
+import java.time.Clock;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.transaction.PlatformTransactionManager;
+import org.springframework.transaction.support.TransactionTemplate;
+
+/** Explicit clock and transaction boundaries shared by wallet application services. */
+@Configuration
+public class WalletInfrastructureConfig {
+
+  @Bean
+  public Clock systemClock() {
+    return Clock.systemUTC();
+  }
+
+  @Bean
+  public TransactionTemplate writeTransactionTemplate(PlatformTransactionManager manager) {
+    return new TransactionTemplate(manager);
+  }
+}


## `feat(service): describe wallet transfer plans`

diff --git a/src/main/java/com/sportsbook/wallet/service/WalletTransferPlan.java b/src/main/java/com/sportsbook/wallet/service/WalletTransferPlan.java
new file mode 100644
index 0000000..dc2d801
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/service/WalletTransferPlan.java
@@ -0,0 +1,16 @@
+package com.sportsbook.wallet.service;
+
+import com.sportsbook.wallet.domain.LedgerEntry;
+import com.sportsbook.wallet.domain.LedgerReason;
+import java.util.Objects;
+
+/** Balance mutation's matching journal topology. */
+record WalletTransferPlan(
+    LedgerEntry.TransferLeg destination, LedgerEntry.TransferLeg source, LedgerReason reason) {
+
+  WalletTransferPlan {
+    Objects.requireNonNull(destination, "destination");
+    Objects.requireNonNull(source, "source");
+    Objects.requireNonNull(reason, "reason");
+  }
+}


## `feat(service): persist matched transfers and commit notifications`

diff --git a/src/main/java/com/sportsbook/wallet/integrity/OperationCommitted.java b/src/main/java/com/sportsbook/wallet/integrity/OperationCommitted.java
new file mode 100644
index 0000000..33a3986
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/integrity/OperationCommitted.java
@@ -0,0 +1,12 @@
+package com.sportsbook.wallet.integrity;
+
+import java.util.Objects;
+import java.util.UUID;
+
+/** Transaction-bound notification used by post-commit integrity checks. */
+public record OperationCommitted(UUID operationGroupId) {
+
+  public OperationCommitted {
+    Objects.requireNonNull(operationGroupId, "operationGroupId");
+  }
+}
diff --git a/src/main/java/com/sportsbook/wallet/service/WalletTransferWriter.java b/src/main/java/com/sportsbook/wallet/service/WalletTransferWriter.java
new file mode 100644
index 0000000..531ae70
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/service/WalletTransferWriter.java
@@ -0,0 +1,44 @@
+package com.sportsbook.wallet.service;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.LedgerEntry;
+import com.sportsbook.wallet.domain.LedgerReason;
+import com.sportsbook.wallet.infrastructure.id.UuidV7;
+import com.sportsbook.wallet.integrity.OperationCommitted;
+import com.sportsbook.wallet.persistence.LedgerEntryRepository;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.springframework.context.ApplicationEventPublisher;
+import org.springframework.stereotype.Component;
+
+/** Appends one complete double-entry transfer and announces its transaction-bound group. */
+@Component
+public class WalletTransferWriter {
+
+  private final LedgerEntryRepository ledger;
+  private final ApplicationEventPublisher events;
+
+  public WalletTransferWriter(LedgerEntryRepository ledger, ApplicationEventPublisher events) {
+    this.ledger = ledger;
+    this.events = events;
+  }
+
+  @SuppressWarnings("checkstyle:ParameterNumber")
+  public WalletOperationResult write(
+      LedgerEntry.TransferLeg destination,
+      LedgerEntry.TransferLeg source,
+      Money amount,
+      LedgerReason reason,
+      IdempotencyKey key,
+      UUID userId,
+      Instant now) {
+    UUID groupId = UuidV7.generate();
+    LedgerEntry.Pair pair =
+        LedgerEntry.pair(destination, source, amount, reason, key, groupId, now);
+    ledger.saveAll(List.of(pair.debit(), pair.credit()));
+    events.publishEvent(new OperationCommitted(groupId));
+    return new WalletOperationResult(groupId, userId, amount, reason, now);
+  }
+}


## `feat(service): execute durable transfer outcomes`

diff --git a/src/main/java/com/sportsbook/wallet/service/WalletTransferExecutor.java b/src/main/java/com/sportsbook/wallet/service/WalletTransferExecutor.java
new file mode 100644
index 0000000..56d1a6e
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/service/WalletTransferExecutor.java
@@ -0,0 +1,97 @@
+package com.sportsbook.wallet.service;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.Account;
+import com.sportsbook.wallet.domain.WalletCaller;
+import com.sportsbook.wallet.domain.WalletFailureSnapshot;
+import com.sportsbook.wallet.domain.WalletOperation;
+import com.sportsbook.wallet.domain.WalletOperationKind;
+import com.sportsbook.wallet.domain.error.AccountNotFoundException;
+import com.sportsbook.wallet.domain.error.CurrencyMismatchException;
+import com.sportsbook.wallet.persistence.AccountRepository;
+import java.time.Clock;
+import java.time.Instant;
+import java.time.temporal.ChronoUnit;
+import java.util.UUID;
+import java.util.function.BiFunction;
+import org.springframework.stereotype.Component;
+
+/** Runs account mutation, matched ledger pair, and authoritative outcome in one transaction. */
+@Component
+public class WalletTransferExecutor {
+
+  private final AccountRepository accounts;
+  private final WalletOperationExecutor operations;
+  private final WalletTransferWriter transfers;
+  private final WalletOutcomeResolver outcomes;
+  private final Clock clock;
+
+  public WalletTransferExecutor(
+      AccountRepository accounts,
+      WalletOperationExecutor operations,
+      WalletTransferWriter transfers,
+      WalletOutcomeResolver outcomes,
+      Clock clock) {
+    this.accounts = accounts;
+    this.operations = operations;
+    this.transfers = transfers;
+    this.outcomes = outcomes;
+    this.clock = clock;
+  }
+
+  @SuppressWarnings("checkstyle:ParameterNumber")
+  public WalletOperationResult execute(
+      IdempotencyKey key,
+      WalletCaller caller,
+      WalletOperationKind kind,
+      UUID userId,
+      Money amount,
+      BiFunction<Account, Instant, WalletTransferPlan> mutation) {
+    WalletOperation operation =
+        operations.execute(
+            key,
+            caller,
+            kind,
+            userId,
+            amount,
+            fingerprint -> firstWrite(key, caller, kind, userId, amount, fingerprint, mutation));
+    return outcomes.resolve(operation);
+  }
+
+  @SuppressWarnings("checkstyle:ParameterNumber")
+  private WalletOperation firstWrite(
+      IdempotencyKey key,
+      WalletCaller caller,
+      WalletOperationKind kind,
+      UUID userId,
+      Money amount,
+      String fingerprint,
+      BiFunction<Account, Instant, WalletTransferPlan> mutation) {
+    Instant now = clock.instant().truncatedTo(ChronoUnit.MICROS);
+    try {
+      Account account = lockAccount(userId, amount);
+      WalletTransferPlan plan = mutation.apply(account, now);
+      WalletOperationResult result =
+          transfers.write(
+              plan.destination(), plan.source(), amount, plan.reason(), key, userId, now);
+      return WalletOperation.succeeded(
+          key, caller, kind, userId, amount, fingerprint, result.operationGroupId(), now);
+    } catch (RuntimeException businessOrInfrastructure) {
+      WalletFailureSnapshot failure =
+          WalletFailureMapper.snapshot(businessOrInfrastructure, amount);
+      return WalletOperation.rejected(key, caller, kind, userId, amount, fingerprint, failure, now);
+    }
+  }
+
+  private Account lockAccount(UUID userId, Money amount) {
+    Account account =
+        accounts
+            .findByUserIdForUpdate(userId)
+            .orElseThrow(() -> new AccountNotFoundException(userId));
+    if (account.currency() != amount.currency()) {
+      throw new CurrencyMismatchException(account.currency(), amount.currency());
+    }
+    return account;
+  }
+}


## `feat(service): accept external deposits`

diff --git a/src/main/java/com/sportsbook/wallet/service/WalletService.java b/src/main/java/com/sportsbook/wallet/service/WalletService.java
index 2caaf74..f10b1fe 100644
--- a/src/main/java/com/sportsbook/wallet/service/WalletService.java
+++ b/src/main/java/com/sportsbook/wallet/service/WalletService.java
@@ -1,9 +1,16 @@
 package com.sportsbook.wallet.service;
 
 import com.sportsbook.wallet.domain.Account;
+import com.sportsbook.wallet.domain.BalanceBucket;
+import com.sportsbook.wallet.domain.LedgerEntry;
+import com.sportsbook.wallet.domain.LedgerReason;
+import com.sportsbook.wallet.domain.SystemAccountIds;
+import com.sportsbook.wallet.domain.WalletCaller;
+import com.sportsbook.wallet.domain.WalletOperationKind;
 import com.sportsbook.wallet.domain.error.AccountNotFoundException;
 import com.sportsbook.wallet.domain.error.CurrencyMismatchException;
 import com.sportsbook.wallet.persistence.AccountRepository;
+import com.sportsbook.wallet.service.command.DepositCommand;
 import com.sportsbook.wallet.service.command.OpenAccountCommand;
 import java.time.Clock;
 import java.time.Instant;
@@ -21,12 +28,17 @@ public class WalletService {
   private final AccountRepository accounts;
   private final TransactionTemplate writeTransaction;
   private final Clock clock;
+  private final WalletTransferExecutor transferExecutor;
 
   public WalletService(
-      AccountRepository accounts, TransactionTemplate writeTransaction, Clock clock) {
+      AccountRepository accounts,
+      TransactionTemplate writeTransaction,
+      Clock clock,
+      WalletTransferExecutor transferExecutor) {
     this.accounts = accounts;
     this.writeTransaction = writeTransaction;
     this.clock = clock;
+    this.transferExecutor = transferExecutor;
   }
 
   public Account openAccount(OpenAccountCommand command) {
@@ -54,6 +66,23 @@ public class WalletService {
     return accounts.findById(userId).orElseThrow(() -> new AccountNotFoundException(userId));
   }
 
+  public WalletOperationResult deposit(DepositCommand command) {
+    return transferExecutor.execute(
+        command.idempotencyKey(),
+        WalletCaller.PLATFORM,
+        WalletOperationKind.DEPOSIT,
+        command.userId(),
+        command.amount(),
+        (account, now) -> {
+          account.increaseAvailable(command.amount(), now);
+          return new WalletTransferPlan(
+              new LedgerEntry.TransferLeg(command.userId(), BalanceBucket.AVAILABLE),
+              new LedgerEntry.TransferLeg(
+                  SystemAccountIds.EXTERNAL_PAYMENT, BalanceBucket.AVAILABLE),
+              LedgerReason.DEPOSIT);
+        });
+  }
+
   private static Account requireCurrency(Account account, OpenAccountCommand command) {
     if (account.currency() != command.currency()) {
       throw new CurrencyMismatchException(account.currency(), command.currency());


## `feat(service): withdraw available funds`

diff --git a/src/main/java/com/sportsbook/wallet/service/WalletService.java b/src/main/java/com/sportsbook/wallet/service/WalletService.java
index f10b1fe..ae93131 100644
--- a/src/main/java/com/sportsbook/wallet/service/WalletService.java
+++ b/src/main/java/com/sportsbook/wallet/service/WalletService.java
@@ -12,6 +12,7 @@ import com.sportsbook.wallet.domain.error.CurrencyMismatchException;
 import com.sportsbook.wallet.persistence.AccountRepository;
 import com.sportsbook.wallet.service.command.DepositCommand;
 import com.sportsbook.wallet.service.command.OpenAccountCommand;
+import com.sportsbook.wallet.service.command.WithdrawCommand;
 import java.time.Clock;
 import java.time.Instant;
 import java.time.temporal.ChronoUnit;
@@ -83,6 +84,23 @@ public class WalletService {
         });
   }
 
+  public WalletOperationResult withdraw(WithdrawCommand command) {
+    return transferExecutor.execute(
+        command.idempotencyKey(),
+        WalletCaller.PLATFORM,
+        WalletOperationKind.WITHDRAW,
+        command.userId(),
+        command.amount(),
+        (account, now) -> {
+          account.decreaseAvailable(command.amount(), now);
+          return new WalletTransferPlan(
+              new LedgerEntry.TransferLeg(
+                  SystemAccountIds.EXTERNAL_PAYMENT, BalanceBucket.AVAILABLE),
+              new LedgerEntry.TransferLeg(command.userId(), BalanceBucket.AVAILABLE),
+              LedgerReason.WITHDRAW);
+        });
+  }
+
   private static Account requireCurrency(Account account, OpenAccountCommand command) {
     if (account.currency() != command.currency()) {
       throw new CurrencyMismatchException(account.currency(), command.currency());


## `test(operation): roll back outcomes with failed money transactions`

diff --git a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
index 1f3cc5b..5d39311 100644
--- a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
+++ b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
@@ -22,6 +22,7 @@ import com.sportsbook.wallet.domain.error.CurrencyMismatchException;
 import com.sportsbook.wallet.domain.error.IdempotencyConflictException;
 import com.sportsbook.wallet.domain.error.WalletBusyException;
 import com.sportsbook.wallet.domain.error.WalletRejectedException;
+import com.sportsbook.wallet.integrity.OperationCommitted;
 import com.sportsbook.wallet.service.WalletOperationExecutor;
 import com.sportsbook.wallet.service.WalletOutcomeResolver;
 import com.sportsbook.wallet.service.WalletService;
@@ -34,10 +35,12 @@ import java.math.BigInteger;
 import java.time.Instant;
 import java.util.UUID;
 import java.util.concurrent.CompletableFuture;
+import java.util.concurrent.atomic.AtomicBoolean;
 import org.junit.jupiter.api.Test;
 import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
 import org.springframework.context.annotation.Import;
+import org.springframework.context.event.EventListener;
 import org.springframework.jdbc.core.JdbcTemplate;
 import org.springframework.test.context.DynamicPropertyRegistry;
 import org.springframework.test.context.DynamicPropertySource;
@@ -58,7 +61,8 @@ import org.testcontainers.junit.jupiter.Testcontainers;
   WalletOutcomeResolver.class,
   WalletService.class,
   WalletTransferExecutor.class,
-  WalletTransferWriter.class
+  WalletTransferWriter.class,
+  WalletPersistenceTest.CommitFault.class
 })
 class WalletPersistenceTest {
   @Container
@@ -70,6 +74,7 @@ class WalletPersistenceTest {
   @Autowired WalletOperationRepository operations;
   @Autowired IdempotencyKeyLock idempotencyLocks;
   @Autowired WalletService wallet;
+  @Autowired CommitFault commitFault;
   @Autowired javax.sql.DataSource dataSource;
   @Autowired org.springframework.transaction.PlatformTransactionManager transactions;
 
@@ -389,6 +394,40 @@ class WalletPersistenceTest {
         .isEqualTo(WalletOperationStatus.REJECTED);
   }
 
+  @Test
+  void rollsBackOutcomeLedgerAndBalanceWhenTransferFails() {
+    UUID userId = UUID.fromString("019b76da-a000-7000-8000-00000000001a");
+    wallet.openAccount(new OpenAccountCommand(userId, com.sportsbook.protocol.value.Currency.KRW));
+    DepositCommand command =
+        new DepositCommand(userId, Money.krw(60L), IdempotencyKey.of("deposit:rollback"));
+    commitFault.failNext();
+
+    assertThatThrownBy(() -> wallet.deposit(command))
+        .isInstanceOf(IllegalStateException.class)
+        .hasMessage("injected commit notification failure");
+
+    assertThat(wallet.requireAccount(userId).available()).isEqualTo(Money.krw(0L));
+    assertThat(ledger.findByIdempotencyKey(command.idempotencyKey().value())).isEmpty();
+    assertThat(operations.findById(command.idempotencyKey().value())).isEmpty();
+    wallet.deposit(command);
+    assertThat(wallet.requireAccount(userId).available()).isEqualTo(Money.krw(60L));
+  }
+
+  static final class CommitFault {
+    private final AtomicBoolean failNext = new AtomicBoolean();
+
+    void failNext() {
+      failNext.set(true);
+    }
+
+    @EventListener
+    public void afterTransfer(OperationCommitted ignored) {
+      if (failNext.compareAndSet(true, false)) {
+        throw new IllegalStateException("injected commit notification failure");
+      }
+    }
+  }
+
   private static void await(java.util.concurrent.CountDownLatch latch) {
     try {
       latch.await();


