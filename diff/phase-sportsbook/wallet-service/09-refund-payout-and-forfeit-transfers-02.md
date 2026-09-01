## `test(forfeit): verify balance ledger and retry invariants`

diff --git a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
index 363896b..ec2eafc 100644
--- a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
+++ b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
@@ -39,6 +39,7 @@ import com.sportsbook.wallet.service.command.CreditCommand;
 import com.sportsbook.wallet.service.command.CreditReason;
 import com.sportsbook.wallet.service.command.DebitCommand;
 import com.sportsbook.wallet.service.command.DepositCommand;
+import com.sportsbook.wallet.service.command.ForfeitCommand;
 import com.sportsbook.wallet.service.command.OpenAccountCommand;
 import com.sportsbook.wallet.service.command.WithdrawCommand;
 import java.math.BigDecimal;
@@ -830,6 +831,42 @@ class WalletPersistenceTest {
     assertThat(outboxFor(payout.idempotencyKey())).hasSize(1);
   }
 
+  @Test
+  void forfeitsLockedFundsWithDurableSuccessAndFailure() {
+    UUID userId = UUID.fromString("019b76da-a000-7000-8000-000000000021");
+    wallet.openAccount(new OpenAccountCommand(userId, com.sportsbook.protocol.value.Currency.KRW));
+    wallet.deposit(
+        new DepositCommand(userId, Money.krw(100L), IdempotencyKey.of("deposit:forfeit-seed")));
+    wallet.debit(
+        new DebitCommand(userId, Money.krw(100L), IdempotencyKey.of("debit:forfeit-seed")));
+    ForfeitCommand success =
+        new ForfeitCommand(userId, Money.krw(60L), IdempotencyKey.of("forfeit:success"));
+
+    var committed = wallet.forfeit(success);
+
+    assertThat(wallet.forfeit(success)).isEqualTo(committed);
+    assertThat(wallet.requireAccount(userId).locked()).isEqualTo(Money.krw(40L));
+    assertThat(committed.reason()).isEqualTo(com.sportsbook.wallet.domain.LedgerReason.BET_FORFEIT);
+    assertThat(ledger.findByIdempotencyKey(success.idempotencyKey().value())).hasSize(2);
+    assertThat(outboxFor(success.idempotencyKey())).isEmpty();
+
+    ForfeitCommand rejected =
+        new ForfeitCommand(userId, Money.krw(100L), IdempotencyKey.of("forfeit:rejected"));
+    WalletRejectedException first =
+        catchThrowableOfType(() -> wallet.forfeit(rejected), WalletRejectedException.class);
+    wallet.deposit(
+        new DepositCommand(userId, Money.krw(100L), IdempotencyKey.of("deposit:forfeit-more")));
+    wallet.debit(
+        new DebitCommand(userId, Money.krw(100L), IdempotencyKey.of("debit:forfeit-more")));
+    WalletRejectedException replay =
+        catchThrowableOfType(() -> wallet.forfeit(rejected), WalletRejectedException.class);
+
+    assertThat(replay.failure().detail()).isEqualTo(first.failure().detail());
+    assertThat(wallet.requireAccount(userId).locked()).isEqualTo(Money.krw(140L));
+    assertThat(ledger.findByIdempotencyKey(rejected.idempotencyKey().value())).isEmpty();
+    assertThat(outboxFor(rejected.idempotencyKey())).isEmpty();
+  }
+
   @Test
   void convergesOneHundredConcurrentRequestsForOneKey() {
     UUID userId = UUID.fromString("019b76da-a000-7000-8000-000000000019");


## `feat(events): build credited wallet messages`

diff --git a/src/main/java/com/sportsbook/wallet/outbox/WalletEventFactory.java b/src/main/java/com/sportsbook/wallet/outbox/WalletEventFactory.java
index e8ef3fa..18abb86 100644
--- a/src/main/java/com/sportsbook/wallet/outbox/WalletEventFactory.java
+++ b/src/main/java/com/sportsbook/wallet/outbox/WalletEventFactory.java
@@ -1,11 +1,13 @@
 package com.sportsbook.wallet.outbox;
 
+import com.sportsbook.protocol.event.WalletCredited;
 import com.sportsbook.protocol.event.WalletDebitFailed;
 import com.sportsbook.protocol.event.WalletDebitFailureReason;
 import com.sportsbook.protocol.event.WalletDebited;
 import com.sportsbook.protocol.value.Money;
 import com.sportsbook.wallet.domain.WalletFailureCode;
 import com.sportsbook.wallet.domain.WalletFailureSnapshot;
+import com.sportsbook.wallet.service.command.CreditCommand;
 import com.sportsbook.wallet.service.command.DebitCommand;
 import java.time.Instant;
 import java.time.temporal.ChronoUnit;
@@ -17,6 +19,27 @@ public class WalletEventFactory {
 
   public static final String DEBITED_TOPIC = "wallet.debited.v1";
   public static final String DEBIT_FAILED_TOPIC = "wallet.debit-failed.v1";
+  public static final String CREDITED_TOPIC = "wallet.credited.v1";
+
+  public PendingOutboxMessage credited(
+      CreditCommand command, UUID destinationLedgerEntryId, Instant occurredAt) {
+    WalletCredited event =
+        new WalletCredited(
+            command.userId().toString(),
+            eventMoney(command.amount()),
+            command.idempotencyKey().value(),
+            destinationLedgerEntryId.toString(),
+            com.sportsbook.protocol.event.WalletCreditReason.valueOf(command.reason().name()),
+            wireTime(occurredAt));
+    return PendingOutboxMessage.create(
+        command.idempotencyKey().value(),
+        CREDITED_TOPIC,
+        command.userId().toString(),
+        WalletCredited.getClassSchema().getName(),
+        command.idempotencyKey().value(),
+        AvroSerializer.serialize(event),
+        occurredAt);
+  }
 
   public PendingOutboxMessage debited(
       DebitCommand command, UUID destinationLedgerEntryId, Instant occurredAt) {


## `test(events): verify credited reasons and payloads`

diff --git a/src/test/java/com/sportsbook/wallet/outbox/WalletEventFactoryTest.java b/src/test/java/com/sportsbook/wallet/outbox/WalletEventFactoryTest.java
index 41ff08b..39713d7 100644
--- a/src/test/java/com/sportsbook/wallet/outbox/WalletEventFactoryTest.java
+++ b/src/test/java/com/sportsbook/wallet/outbox/WalletEventFactoryTest.java
@@ -2,6 +2,7 @@ package com.sportsbook.wallet.outbox;
 
 import static org.assertj.core.api.Assertions.assertThat;
 
+import com.sportsbook.protocol.event.WalletCredited;
 import com.sportsbook.protocol.event.WalletDebitFailed;
 import com.sportsbook.protocol.event.WalletDebitFailureReason;
 import com.sportsbook.protocol.event.WalletDebited;
@@ -9,6 +10,8 @@ import com.sportsbook.protocol.value.IdempotencyKey;
 import com.sportsbook.protocol.value.Money;
 import com.sportsbook.wallet.domain.WalletFailureCode;
 import com.sportsbook.wallet.domain.WalletFailureSnapshot;
+import com.sportsbook.wallet.service.command.CreditCommand;
+import com.sportsbook.wallet.service.command.CreditReason;
 import com.sportsbook.wallet.service.command.DebitCommand;
 import java.time.Instant;
 import java.util.UUID;
@@ -57,6 +60,26 @@ class WalletEventFactoryTest {
     assertThat(event.getRequestedAmount().getAmount()).isEqualTo(1_000L);
   }
 
+  @ParameterizedTest
+  @MethodSource("creditReasons")
+  void mapsEveryCreditedReason(CreditReason reason) {
+    CreditCommand credit =
+        new CreditCommand(
+            USER_ID,
+            Money.krw(1_000L),
+            CreditCommand.Source.HOUSE_POOL,
+            reason,
+            IdempotencyKey.of("credit:" + reason.name().toLowerCase()));
+
+    PendingOutboxMessage message = factory.credited(credit, LEDGER_ENTRY_ID, NOW);
+
+    WalletCredited event = AvroSerializer.deserialize(message.payload(), WalletCredited.class);
+    assertThat(message.topic()).isEqualTo(WalletEventFactory.CREDITED_TOPIC);
+    assertThat(message.partitionKey()).isEqualTo(USER_ID.toString());
+    assertThat(event.getReason().name()).isEqualTo(reason.name());
+    assertThat(event.getLedgerTxId()).isEqualTo(LEDGER_ENTRY_ID.toString());
+  }
+
   private void assertCommon(PendingOutboxMessage message, String topic, String schema) {
     assertThat(message.operationKey()).isEqualTo(BET_ID.toString());
     assertThat(message.deduplicationKey()).isEqualTo(BET_ID.toString());
@@ -76,4 +99,8 @@ class WalletEventFactoryTest {
         Arguments.of(
             WalletFailureCode.CURRENCY_MISMATCH, WalletDebitFailureReason.CURRENCY_MISMATCH));
   }
+
+  private static Stream<CreditReason> creditReasons() {
+    return Stream.of(CreditReason.values());
+  }
 }


## `test(events): prove credited ledger row identity`

diff --git a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
index cf11e00..363896b 100644
--- a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
+++ b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
@@ -800,6 +800,36 @@ class WalletPersistenceTest {
         userId, Money.krw(1L), source, reason, IdempotencyKey.of("credit:forbidden:" + suffix));
   }
 
+  @Test
+  void creditsReferenceTheUserSideLedgerRowRatherThanTheOperationGroup() {
+    UUID userId = UUID.fromString("019b76da-a000-7000-8000-000000000020");
+    wallet.openAccount(new OpenAccountCommand(userId, com.sportsbook.protocol.value.Currency.KRW));
+    CreditCommand payout =
+        new CreditCommand(
+            userId,
+            Money.krw(250L),
+            CreditCommand.Source.HOUSE_POOL,
+            CreditReason.PAYOUT,
+            IdempotencyKey.of("credit:ledger-proof"));
+
+    var result = wallet.credit(com.sportsbook.wallet.domain.WalletCaller.SETTLEMENT, payout);
+    var message = outboxFor(payout.idempotencyKey()).get(0);
+    var event =
+        com.sportsbook.wallet.outbox.AvroSerializer.deserialize(
+            message.payload(), com.sportsbook.protocol.event.WalletCredited.class);
+    LedgerEntry userSide =
+        ledger.findByIdempotencyKey(payout.idempotencyKey().value()).stream()
+            .filter(entry -> entry.side() == com.sportsbook.wallet.domain.LedgerSide.DEBIT)
+            .findFirst()
+            .orElseThrow();
+
+    assertThat(userSide.accountId()).isEqualTo(userId);
+    assertThat(userSide.bucket()).isEqualTo(BalanceBucket.AVAILABLE);
+    assertThat(event.getLedgerTxId()).isEqualTo(userSide.entryId().toString());
+    assertThat(event.getLedgerTxId()).isNotEqualTo(result.operationGroupId().toString());
+    assertThat(outboxFor(payout.idempotencyKey())).hasSize(1);
+  }
+
   @Test
   void convergesOneHundredConcurrentRequestsForOneKey() {
     UUID userId = UUID.fromString("019b76da-a000-7000-8000-000000000019");
