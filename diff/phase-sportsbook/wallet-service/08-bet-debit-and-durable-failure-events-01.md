# 베팅 차감과 영속 실패 이벤트

## `feat(command): define positive withdrawal and debit commands`

diff --git a/src/main/java/com/sportsbook/wallet/service/command/DebitCommand.java b/src/main/java/com/sportsbook/wallet/service/command/DebitCommand.java
new file mode 100644
index 0000000..b187e96
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/service/command/DebitCommand.java
@@ -0,0 +1,19 @@
+package com.sportsbook.wallet.service.command;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import java.util.Objects;
+import java.util.UUID;
+
+/** Stages available funds in the locked balance while a bet remains open. */
+public record DebitCommand(UUID userId, Money amount, IdempotencyKey idempotencyKey) {
+
+  public DebitCommand {
+    Objects.requireNonNull(userId, "userId");
+    Objects.requireNonNull(amount, "amount");
+    Objects.requireNonNull(idempotencyKey, "idempotencyKey");
+    if (!amount.isPositive()) {
+      throw new IllegalArgumentException("Debit amount must be strictly positive");
+    }
+  }
+}
diff --git a/src/main/java/com/sportsbook/wallet/service/command/WithdrawCommand.java b/src/main/java/com/sportsbook/wallet/service/command/WithdrawCommand.java
new file mode 100644
index 0000000..97ac8df
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/service/command/WithdrawCommand.java
@@ -0,0 +1,19 @@
+package com.sportsbook.wallet.service.command;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import java.util.Objects;
+import java.util.UUID;
+
+/** Moves available funds from a user wallet to the external payment account. */
+public record WithdrawCommand(UUID userId, Money amount, IdempotencyKey idempotencyKey) {
+
+  public WithdrawCommand {
+    Objects.requireNonNull(userId, "userId");
+    Objects.requireNonNull(amount, "amount");
+    Objects.requireNonNull(idempotencyKey, "idempotencyKey");
+    if (!amount.isPositive()) {
+      throw new IllegalArgumentException("Withdrawal amount must be strictly positive");
+    }
+  }
+}


## `feat(service): stage betting debits`

diff --git a/src/main/java/com/sportsbook/wallet/service/WalletService.java b/src/main/java/com/sportsbook/wallet/service/WalletService.java
index ae93131..4050c70 100644
--- a/src/main/java/com/sportsbook/wallet/service/WalletService.java
+++ b/src/main/java/com/sportsbook/wallet/service/WalletService.java
@@ -10,6 +10,7 @@ import com.sportsbook.wallet.domain.WalletOperationKind;
 import com.sportsbook.wallet.domain.error.AccountNotFoundException;
 import com.sportsbook.wallet.domain.error.CurrencyMismatchException;
 import com.sportsbook.wallet.persistence.AccountRepository;
+import com.sportsbook.wallet.service.command.DebitCommand;
 import com.sportsbook.wallet.service.command.DepositCommand;
 import com.sportsbook.wallet.service.command.OpenAccountCommand;
 import com.sportsbook.wallet.service.command.WithdrawCommand;
@@ -101,6 +102,22 @@ public class WalletService {
         });
   }
 
+  public WalletOperationResult debit(DebitCommand command) {
+    return transferExecutor.execute(
+        command.idempotencyKey(),
+        WalletCaller.BETTING,
+        WalletOperationKind.BET_DEBIT,
+        command.userId(),
+        command.amount(),
+        (account, now) -> {
+          account.moveAvailableToLocked(command.amount(), now);
+          return new WalletTransferPlan(
+              new LedgerEntry.TransferLeg(command.userId(), BalanceBucket.LOCKED),
+              new LedgerEntry.TransferLeg(command.userId(), BalanceBucket.AVAILABLE),
+              LedgerReason.BET_DEBIT);
+        });
+  }
+
   private static Account requireCurrency(Account account, OpenAccountCommand command) {
     if (account.currency() != command.currency()) {
       throw new CurrencyMismatchException(account.currency(), command.currency());


## `feat(events): build debit success and failure messages`

diff --git a/src/main/java/com/sportsbook/wallet/outbox/WalletEventFactory.java b/src/main/java/com/sportsbook/wallet/outbox/WalletEventFactory.java
new file mode 100644
index 0000000..e8ef3fa
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/outbox/WalletEventFactory.java
@@ -0,0 +1,85 @@
+package com.sportsbook.wallet.outbox;
+
+import com.sportsbook.protocol.event.WalletDebitFailed;
+import com.sportsbook.protocol.event.WalletDebitFailureReason;
+import com.sportsbook.protocol.event.WalletDebited;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.WalletFailureCode;
+import com.sportsbook.wallet.domain.WalletFailureSnapshot;
+import com.sportsbook.wallet.service.command.DebitCommand;
+import java.time.Instant;
+import java.time.temporal.ChronoUnit;
+import java.util.UUID;
+import org.springframework.stereotype.Component;
+
+@Component
+public class WalletEventFactory {
+
+  public static final String DEBITED_TOPIC = "wallet.debited.v1";
+  public static final String DEBIT_FAILED_TOPIC = "wallet.debit-failed.v1";
+
+  public PendingOutboxMessage debited(
+      DebitCommand command, UUID destinationLedgerEntryId, Instant occurredAt) {
+    WalletDebited event =
+        new WalletDebited(
+            command.userId().toString(),
+            eventMoney(command.amount()),
+            command.idempotencyKey().value(),
+            destinationLedgerEntryId.toString(),
+            wireTime(occurredAt));
+    return pending(
+        command,
+        DEBITED_TOPIC,
+        WalletDebited.getClassSchema().getName(),
+        AvroSerializer.serialize(event),
+        occurredAt);
+  }
+
+  public PendingOutboxMessage debitFailed(
+      DebitCommand command, WalletFailureSnapshot failure, Instant occurredAt) {
+    WalletDebitFailed event =
+        new WalletDebitFailed(
+            command.userId().toString(),
+            eventMoney(command.amount()),
+            command.idempotencyKey().value(),
+            failureReason(failure.code()),
+            wireTime(occurredAt));
+    return pending(
+        command,
+        DEBIT_FAILED_TOPIC,
+        WalletDebitFailed.getClassSchema().getName(),
+        AvroSerializer.serialize(event),
+        occurredAt);
+  }
+
+  private PendingOutboxMessage pending(
+      DebitCommand command, String topic, String schema, byte[] payload, Instant occurredAt) {
+    return PendingOutboxMessage.create(
+        command.idempotencyKey().value(),
+        topic,
+        command.userId().toString(),
+        schema,
+        command.idempotencyKey().value(),
+        payload,
+        occurredAt);
+  }
+
+  private com.sportsbook.protocol.event.Money eventMoney(Money money) {
+    return new com.sportsbook.protocol.event.Money(money.amount(), money.currency().name());
+  }
+
+  private WalletDebitFailureReason failureReason(WalletFailureCode code) {
+    return switch (code) {
+      case INSUFFICIENT_BALANCE -> WalletDebitFailureReason.INSUFFICIENT_BALANCE;
+      case ACCOUNT_SUSPENDED -> WalletDebitFailureReason.ACCOUNT_SUSPENDED;
+      case ACCOUNT_NOT_FOUND -> WalletDebitFailureReason.ACCOUNT_NOT_FOUND;
+      case CURRENCY_MISMATCH -> WalletDebitFailureReason.CURRENCY_MISMATCH;
+      case AMOUNT_OUT_OF_RANGE ->
+          throw new IllegalArgumentException("Debit failure has no shared reason: " + code);
+    };
+  }
+
+  private Instant wireTime(Instant occurredAt) {
+    return occurredAt.truncatedTo(ChronoUnit.MILLIS);
+  }
+}


## `test(events): verify debit topics keys reasons and payloads`

diff --git a/src/test/java/com/sportsbook/wallet/outbox/WalletEventFactoryTest.java b/src/test/java/com/sportsbook/wallet/outbox/WalletEventFactoryTest.java
new file mode 100644
index 0000000..41ff08b
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/outbox/WalletEventFactoryTest.java
@@ -0,0 +1,79 @@
+package com.sportsbook.wallet.outbox;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.event.WalletDebitFailed;
+import com.sportsbook.protocol.event.WalletDebitFailureReason;
+import com.sportsbook.protocol.event.WalletDebited;
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.WalletFailureCode;
+import com.sportsbook.wallet.domain.WalletFailureSnapshot;
+import com.sportsbook.wallet.service.command.DebitCommand;
+import java.time.Instant;
+import java.util.UUID;
+import java.util.stream.Stream;
+import org.junit.jupiter.api.Test;
+import org.junit.jupiter.params.ParameterizedTest;
+import org.junit.jupiter.params.provider.Arguments;
+import org.junit.jupiter.params.provider.MethodSource;
+
+class WalletEventFactoryTest {
+
+  private static final UUID USER_ID = UUID.fromString("0198ca71-8000-7000-8000-000000000001");
+  private static final UUID BET_ID = UUID.fromString("0198ca71-8000-7000-8000-000000000002");
+  private static final UUID LEDGER_ENTRY_ID =
+      UUID.fromString("0198ca71-8000-7000-8000-000000000003");
+  private static final Instant NOW = Instant.parse("2026-08-21T00:00:00.123456Z");
+
+  private final WalletEventFactory factory = new WalletEventFactory();
+  private final DebitCommand command =
+      new DebitCommand(USER_ID, Money.krw(1_000L), IdempotencyKey.of(BET_ID.toString()));
+
+  @Test
+  void buildsADebitedMessageWithTheDestinationLedgerRow() {
+    PendingOutboxMessage message = factory.debited(command, LEDGER_ENTRY_ID, NOW);
+
+    WalletDebited event = AvroSerializer.deserialize(message.payload(), WalletDebited.class);
+    assertCommon(message, WalletEventFactory.DEBITED_TOPIC, "WalletDebited");
+    assertThat(event.getUserId()).isEqualTo(USER_ID.toString());
+    assertThat(event.getAmount().getAmount()).isEqualTo(1_000L);
+    assertThat(event.getAmount().getCurrency()).isEqualTo("KRW");
+    assertThat(event.getLedgerTxId()).isEqualTo(LEDGER_ENTRY_ID.toString());
+    assertThat(event.getOccurredAt()).isEqualTo(Instant.parse("2026-08-21T00:00:00.123Z"));
+  }
+
+  @ParameterizedTest
+  @MethodSource("failureReasons")
+  void mapsEveryDurableDebitFailure(
+      WalletFailureCode code, WalletDebitFailureReason expectedReason) {
+    PendingOutboxMessage message =
+        factory.debitFailed(command, WalletFailureSnapshot.of(code, "failed"), NOW);
+
+    WalletDebitFailed event =
+        AvroSerializer.deserialize(message.payload(), WalletDebitFailed.class);
+    assertCommon(message, WalletEventFactory.DEBIT_FAILED_TOPIC, "WalletDebitFailed");
+    assertThat(event.getReason()).isEqualTo(expectedReason);
+    assertThat(event.getRequestedAmount().getAmount()).isEqualTo(1_000L);
+  }
+
+  private void assertCommon(PendingOutboxMessage message, String topic, String schema) {
+    assertThat(message.operationKey()).isEqualTo(BET_ID.toString());
+    assertThat(message.deduplicationKey()).isEqualTo(BET_ID.toString());
+    assertThat(message.partitionKey()).isEqualTo(USER_ID.toString());
+    assertThat(message.topic()).isEqualTo(topic);
+    assertThat(message.schemaName()).isEqualTo(schema);
+  }
+
+  private static Stream<Arguments> failureReasons() {
+    return Stream.of(
+        Arguments.of(
+            WalletFailureCode.INSUFFICIENT_BALANCE, WalletDebitFailureReason.INSUFFICIENT_BALANCE),
+        Arguments.of(
+            WalletFailureCode.ACCOUNT_SUSPENDED, WalletDebitFailureReason.ACCOUNT_SUSPENDED),
+        Arguments.of(
+            WalletFailureCode.ACCOUNT_NOT_FOUND, WalletDebitFailureReason.ACCOUNT_NOT_FOUND),
+        Arguments.of(
+            WalletFailureCode.CURRENCY_MISMATCH, WalletDebitFailureReason.CURRENCY_MISMATCH));
+  }
+}


## `feat(service): preserve user ledger receipts`

diff --git a/src/main/java/com/sportsbook/wallet/service/WalletTransferReceipt.java b/src/main/java/com/sportsbook/wallet/service/WalletTransferReceipt.java
new file mode 100644
index 0000000..e09c235
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/service/WalletTransferReceipt.java
@@ -0,0 +1,14 @@
+package com.sportsbook.wallet.service;
+
+import java.util.Objects;
+import java.util.UUID;
+
+record WalletTransferReceipt(
+    WalletOperationResult result, UUID destinationEntryId, UUID sourceEntryId) {
+
+  WalletTransferReceipt {
+    Objects.requireNonNull(result, "result");
+    Objects.requireNonNull(destinationEntryId, "destinationEntryId");
+    Objects.requireNonNull(sourceEntryId, "sourceEntryId");
+  }
+}
diff --git a/src/main/java/com/sportsbook/wallet/service/WalletTransferWriter.java b/src/main/java/com/sportsbook/wallet/service/WalletTransferWriter.java
index 531ae70..4ffa119 100644
--- a/src/main/java/com/sportsbook/wallet/service/WalletTransferWriter.java
+++ b/src/main/java/com/sportsbook/wallet/service/WalletTransferWriter.java
@@ -34,11 +34,24 @@ public class WalletTransferWriter {
       IdempotencyKey key,
       UUID userId,
       Instant now) {
+    return writeReceipt(destination, source, amount, reason, key, userId, now).result();
+  }
+
+  @SuppressWarnings("checkstyle:ParameterNumber")
+  public WalletTransferReceipt writeReceipt(
+      LedgerEntry.TransferLeg destination,
+      LedgerEntry.TransferLeg source,
+      Money amount,
+      LedgerReason reason,
+      IdempotencyKey key,
+      UUID userId,
+      Instant now) {
     UUID groupId = UuidV7.generate();
     LedgerEntry.Pair pair =
         LedgerEntry.pair(destination, source, amount, reason, key, groupId, now);
     ledger.saveAll(List.of(pair.debit(), pair.credit()));
     events.publishEvent(new OperationCommitted(groupId));
-    return new WalletOperationResult(groupId, userId, amount, reason, now);
+    WalletOperationResult result = new WalletOperationResult(groupId, userId, amount, reason, now);
+    return new WalletTransferReceipt(result, pair.debit().entryId(), pair.credit().entryId());
   }
 }


## `feat(debit): commit debit success or durable rejection atomically`

diff --git a/src/main/java/com/sportsbook/wallet/service/WalletService.java b/src/main/java/com/sportsbook/wallet/service/WalletService.java
index 4050c70..afb6b8f 100644
--- a/src/main/java/com/sportsbook/wallet/service/WalletService.java
+++ b/src/main/java/com/sportsbook/wallet/service/WalletService.java
@@ -103,12 +103,8 @@ public class WalletService {
   }
 
   public WalletOperationResult debit(DebitCommand command) {
-    return transferExecutor.execute(
-        command.idempotencyKey(),
-        WalletCaller.BETTING,
-        WalletOperationKind.BET_DEBIT,
-        command.userId(),
-        command.amount(),
+    return transferExecutor.executeDebit(
+        command,
         (account, now) -> {
           account.moveAvailableToLocked(command.amount(), now);
           return new WalletTransferPlan(
diff --git a/src/main/java/com/sportsbook/wallet/service/WalletTransferExecutor.java b/src/main/java/com/sportsbook/wallet/service/WalletTransferExecutor.java
index 56d1a6e..e0883f8 100644
--- a/src/main/java/com/sportsbook/wallet/service/WalletTransferExecutor.java
+++ b/src/main/java/com/sportsbook/wallet/service/WalletTransferExecutor.java
@@ -9,7 +9,10 @@ import com.sportsbook.wallet.domain.WalletOperation;
 import com.sportsbook.wallet.domain.WalletOperationKind;
 import com.sportsbook.wallet.domain.error.AccountNotFoundException;
 import com.sportsbook.wallet.domain.error.CurrencyMismatchException;
+import com.sportsbook.wallet.outbox.OutboxAppender;
+import com.sportsbook.wallet.outbox.WalletEventFactory;
 import com.sportsbook.wallet.persistence.AccountRepository;
+import com.sportsbook.wallet.service.command.DebitCommand;
 import java.time.Clock;
 import java.time.Instant;
 import java.time.temporal.ChronoUnit;
@@ -25,18 +28,22 @@ public class WalletTransferExecutor {
   private final WalletOperationExecutor operations;
   private final WalletTransferWriter transfers;
   private final WalletOutcomeResolver outcomes;
+  private final OutboxAppender outboxAppender;
   private final Clock clock;
+  private final WalletEventFactory eventFactory = new WalletEventFactory();
 
   public WalletTransferExecutor(
       AccountRepository accounts,
       WalletOperationExecutor operations,
       WalletTransferWriter transfers,
       WalletOutcomeResolver outcomes,
+      OutboxAppender outboxAppender,
       Clock clock) {
     this.accounts = accounts;
     this.operations = operations;
     this.transfers = transfers;
     this.outcomes = outcomes;
+    this.outboxAppender = outboxAppender;
     this.clock = clock;
   }
 
@@ -48,6 +55,30 @@ public class WalletTransferExecutor {
       UUID userId,
       Money amount,
       BiFunction<Account, Instant, WalletTransferPlan> mutation) {
+    return execute(key, caller, kind, userId, amount, null, mutation);
+  }
+
+  public WalletOperationResult executeDebit(
+      DebitCommand command, BiFunction<Account, Instant, WalletTransferPlan> mutation) {
+    return execute(
+        command.idempotencyKey(),
+        WalletCaller.BETTING,
+        WalletOperationKind.BET_DEBIT,
+        command.userId(),
+        command.amount(),
+        command,
+        mutation);
+  }
+
+  @SuppressWarnings("checkstyle:ParameterNumber")
+  private WalletOperationResult execute(
+      IdempotencyKey key,
+      WalletCaller caller,
+      WalletOperationKind kind,
+      UUID userId,
+      Money amount,
+      DebitCommand debit,
+      BiFunction<Account, Instant, WalletTransferPlan> mutation) {
     WalletOperation operation =
         operations.execute(
             key,
@@ -55,7 +86,8 @@ public class WalletTransferExecutor {
             kind,
             userId,
             amount,
-            fingerprint -> firstWrite(key, caller, kind, userId, amount, fingerprint, mutation));
+            fingerprint ->
+                firstWrite(key, caller, kind, userId, amount, fingerprint, debit, mutation));
     return outcomes.resolve(operation);
   }
 
@@ -67,21 +99,29 @@ public class WalletTransferExecutor {
       UUID userId,
       Money amount,
       String fingerprint,
+      DebitCommand debit,
       BiFunction<Account, Instant, WalletTransferPlan> mutation) {
     Instant now = clock.instant().truncatedTo(ChronoUnit.MICROS);
+    WalletTransferReceipt receipt;
     try {
       Account account = lockAccount(userId, amount);
       WalletTransferPlan plan = mutation.apply(account, now);
-      WalletOperationResult result =
-          transfers.write(
+      receipt =
+          transfers.writeReceipt(
               plan.destination(), plan.source(), amount, plan.reason(), key, userId, now);
-      return WalletOperation.succeeded(
-          key, caller, kind, userId, amount, fingerprint, result.operationGroupId(), now);
     } catch (RuntimeException businessOrInfrastructure) {
       WalletFailureSnapshot failure =
           WalletFailureMapper.snapshot(businessOrInfrastructure, amount);
+      if (debit != null) {
+        outboxAppender.append(eventFactory.debitFailed(debit, failure, now));
+      }
       return WalletOperation.rejected(key, caller, kind, userId, amount, fingerprint, failure, now);
     }
+    if (debit != null) {
+      outboxAppender.append(eventFactory.debited(debit, receipt.destinationEntryId(), now));
+    }
+    return WalletOperation.succeeded(
+        key, caller, kind, userId, amount, fingerprint, receipt.result().operationGroupId(), now);
   }
 
   private Account lockAccount(UUID userId, Money amount) {


## `test(debit): replay the same success or terminal failure`

diff --git a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
index 4a3800f..0d18139 100644
--- a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
+++ b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
@@ -86,6 +86,7 @@ class WalletPersistenceTest {
   @Autowired AccountRepository accounts;
   @Autowired LedgerEntryRepository ledger;
   @Autowired WalletOperationRepository operations;
+  @Autowired OutboxEventRepository outboxEvents;
   @Autowired IdempotencyKeyLock idempotencyLocks;
   @Autowired WalletService wallet;
   @Autowired CommitFault commitFault;
@@ -557,6 +558,53 @@ class WalletPersistenceTest {
     }
   }
 
+  @Test
+  void exactlyReplaysDebitSuccessAndItsSingleOutboxMessage() {
+    UUID userId = UUID.fromString("019b76da-a000-7000-8000-00000000001b");
+    wallet.openAccount(new OpenAccountCommand(userId, com.sportsbook.protocol.value.Currency.KRW));
+    wallet.deposit(
+        new DepositCommand(userId, Money.krw(100L), IdempotencyKey.of("deposit:debit-success")));
+    DebitCommand command =
+        new DebitCommand(userId, Money.krw(60L), IdempotencyKey.of("debit:success"));
+
+    var first = wallet.debit(command);
+    var replay = wallet.debit(command);
+
+    assertThat(replay).isEqualTo(first);
+    assertThat(wallet.requireAccount(userId).available()).isEqualTo(Money.krw(40L));
+    assertThat(wallet.requireAccount(userId).locked()).isEqualTo(Money.krw(60L));
+    assertThat(ledger.findByIdempotencyKey(command.idempotencyKey().value())).hasSize(2);
+    assertThat(outboxFor(command.idempotencyKey()))
+        .singleElement()
+        .extracting(com.sportsbook.wallet.outbox.OutboxEvent::topic)
+        .isEqualTo(com.sportsbook.wallet.outbox.WalletEventFactory.DEBITED_TOPIC);
+  }
+
+  @Test
+  void exactlyReplaysATerminalDebitFailureAfterFundsArrive() {
+    UUID userId = UUID.fromString("019b76da-a000-7000-8000-00000000001c");
+    wallet.openAccount(new OpenAccountCommand(userId, com.sportsbook.protocol.value.Currency.KRW));
+    DebitCommand command =
+        new DebitCommand(userId, Money.krw(60L), IdempotencyKey.of("debit:terminal-failure"));
+
+    WalletRejectedException first =
+        catchThrowableOfType(() -> wallet.debit(command), WalletRejectedException.class);
+    wallet.deposit(
+        new DepositCommand(userId, Money.krw(100L), IdempotencyKey.of("deposit:after-rejection")));
+    WalletRejectedException replay =
+        catchThrowableOfType(() -> wallet.debit(command), WalletRejectedException.class);
+
+    assertThat(replay.failure().code()).isEqualTo(WalletFailureCode.INSUFFICIENT_BALANCE);
+    assertThat(replay.failure().detail()).isEqualTo(first.failure().detail());
+    assertThat(wallet.requireAccount(userId).available()).isEqualTo(Money.krw(100L));
+    assertThat(wallet.requireAccount(userId).locked()).isEqualTo(Money.krw(0L));
+    assertThat(ledger.findByIdempotencyKey(command.idempotencyKey().value())).isEmpty();
+    assertThat(outboxFor(command.idempotencyKey()))
+        .singleElement()
+        .extracting(com.sportsbook.wallet.outbox.OutboxEvent::topic)
+        .isEqualTo(com.sportsbook.wallet.outbox.WalletEventFactory.DEBIT_FAILED_TOPIC);
+  }
+
   @Test
   void convergesOneHundredConcurrentRequestsForOneKey() {
     UUID userId = UUID.fromString("019b76da-a000-7000-8000-000000000019");
@@ -609,4 +657,10 @@ class WalletPersistenceTest {
       throw new IllegalStateException(interrupted);
     }
   }
+
+  private java.util.List<com.sportsbook.wallet.outbox.OutboxEvent> outboxFor(IdempotencyKey key) {
+    return outboxEvents.findAll().stream()
+        .filter(event -> event.operationKey().equals(key.value()))
+        .toList();
+  }
 }


