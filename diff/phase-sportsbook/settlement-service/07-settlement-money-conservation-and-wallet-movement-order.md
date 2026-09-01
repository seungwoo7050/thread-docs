# 정산 금액 보존과 Wallet 자금 이동 순서

## `feat(resolver): split wallet settlement movements`

diff --git a/src/main/java/com/sportsbook/settlement/resolver/WalletMovementPlan.java b/src/main/java/com/sportsbook/settlement/resolver/WalletMovementPlan.java
new file mode 100644
index 0000000..3c61142
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/resolver/WalletMovementPlan.java
@@ -0,0 +1,22 @@
+package com.sportsbook.settlement.resolver;
+
+import com.sportsbook.protocol.value.Money;
+import java.util.Objects;
+
+/** Split between locked stake, house pool, and forfeited exposure for one resolution. */
+public record WalletMovementPlan(
+    Money returnedStake, Money housePayout, Money forfeitedStake, Money totalExposure) {
+
+  public WalletMovementPlan {
+    Objects.requireNonNull(returnedStake, "returnedStake");
+    Objects.requireNonNull(housePayout, "housePayout");
+    Objects.requireNonNull(forfeitedStake, "forfeitedStake");
+    Objects.requireNonNull(totalExposure, "totalExposure");
+    if (returnedStake.isNegative()
+        || housePayout.isNegative()
+        || forfeitedStake.isNegative()
+        || !returnedStake.add(forfeitedStake).equals(totalExposure)) {
+      throw new IllegalArgumentException("Invalid wallet movement split");
+    }
+  }
+}
diff --git a/src/main/java/com/sportsbook/settlement/resolver/WalletMovementPlanner.java b/src/main/java/com/sportsbook/settlement/resolver/WalletMovementPlanner.java
new file mode 100644
index 0000000..ada9ee2
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/resolver/WalletMovementPlanner.java
@@ -0,0 +1,21 @@
+package com.sportsbook.settlement.resolver;
+
+import com.sportsbook.protocol.value.Money;
+
+/** Separates unit-stake exposure from returned stake and house-funded profit. */
+public final class WalletMovementPlanner {
+
+  public WalletMovementPlan plan(Money unitStake, SettlementOutcome outcome) {
+    if (unitStake == null || unitStake.amount() <= 0 || outcome == null) {
+      throw new IllegalArgumentException("Wallet movement planning requires a positive unit stake");
+    }
+    Money totalExposure = unitStake.multiply(outcome.totalLines());
+    Money returnedStake = unitStake.multiply(outcome.survivingLines());
+    Money forfeitedStake = totalExposure.subtract(returnedStake);
+    Money housePayout = outcome.payout().subtract(returnedStake);
+    if (housePayout.isNegative()) {
+      throw new IllegalArgumentException("Payout cannot be lower than surviving returned stake");
+    }
+    return new WalletMovementPlan(returnedStake, housePayout, forfeitedStake, totalExposure);
+  }
+}


## `test(resolver): verify unit stake exposure split`

diff --git a/src/test/java/com/sportsbook/settlement/resolver/WalletMovementPlannerTest.java b/src/test/java/com/sportsbook/settlement/resolver/WalletMovementPlannerTest.java
new file mode 100644
index 0000000..1c1a6af
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/resolver/WalletMovementPlannerTest.java
@@ -0,0 +1,38 @@
+package com.sportsbook.settlement.resolver;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.protocol.value.Money;
+import org.junit.jupiter.api.Test;
+
+class WalletMovementPlannerTest {
+
+  private final WalletMovementPlanner planner = new WalletMovementPlanner();
+
+  @Test
+  void keepsUnitStakeSeparateFromSystemTotalExposure() {
+    Money unitStake = Money.krw(1_000);
+    SettlementOutcome outcome = new SettlementOutcome(SettlementResult.WON, Money.krw(6_000), 1, 3);
+
+    WalletMovementPlan plan = planner.plan(unitStake, outcome);
+
+    assertThat(unitStake).isEqualTo(Money.krw(1_000));
+    assertThat(plan.totalExposure()).isEqualTo(Money.krw(3_000));
+    assertThat(plan.returnedStake()).isEqualTo(Money.krw(1_000));
+    assertThat(plan.housePayout()).isEqualTo(Money.krw(5_000));
+    assertThat(plan.forfeitedStake()).isEqualTo(Money.krw(2_000));
+  }
+
+  @Test
+  void returnsEveryLineWithoutHouseProfitForAFullVoidResult() {
+    SettlementOutcome outcome =
+        new SettlementOutcome(SettlementResult.VOID, Money.krw(3_000), 3, 3);
+
+    WalletMovementPlan plan = planner.plan(Money.krw(1_000), outcome);
+
+    assertThat(plan.returnedStake()).isEqualTo(Money.krw(3_000));
+    assertThat(plan.housePayout()).isEqualTo(Money.krw(0));
+    assertThat(plan.forfeitedStake()).isEqualTo(Money.krw(0));
+  }
+}


## `feat(settlement): enforce money plan conservation`

diff --git a/src/main/java/com/sportsbook/settlement/execution/SettlementMoneyPlan.java b/src/main/java/com/sportsbook/settlement/execution/SettlementMoneyPlan.java
new file mode 100644
index 0000000..24b62d6
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/execution/SettlementMoneyPlan.java
@@ -0,0 +1,25 @@
+package com.sportsbook.settlement.execution;
+
+import com.sportsbook.protocol.value.Money;
+import java.util.List;
+import java.util.Objects;
+
+public record SettlementMoneyPlan(
+    Money committed, Money payout, Money lockedRelease, Money lockedForfeit, Money houseProfit) {
+
+  public SettlementMoneyPlan {
+    List<Money> values =
+        List.of(
+            Objects.requireNonNull(committed, "committed"),
+            Objects.requireNonNull(payout, "payout"),
+            Objects.requireNonNull(lockedRelease, "lockedRelease"),
+            Objects.requireNonNull(lockedForfeit, "lockedForfeit"),
+            Objects.requireNonNull(houseProfit, "houseProfit"));
+    if (values.stream().anyMatch(Money::isNegative)
+        || values.stream().map(Money::currency).distinct().count() != 1
+        || !lockedRelease.add(lockedForfeit).equals(committed)
+        || !lockedRelease.add(houseProfit).equals(payout)) {
+      throw new IllegalArgumentException("Settlement money plan violates conservation");
+    }
+  }
+}


## `test(settlement): verify money plan conservation`

diff --git a/src/test/java/com/sportsbook/settlement/execution/SettlementMoneyPlanTest.java b/src/test/java/com/sportsbook/settlement/execution/SettlementMoneyPlanTest.java
new file mode 100644
index 0000000..961e279
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/execution/SettlementMoneyPlanTest.java
@@ -0,0 +1,33 @@
+package com.sportsbook.settlement.execution;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.protocol.value.Money;
+import org.junit.jupiter.api.Test;
+
+class SettlementMoneyPlanTest {
+
+  @Test
+  void preservesSystemExposureSeparatelyFromUnitStake() {
+    SettlementMoneyPlan plan =
+        new SettlementMoneyPlan(
+            Money.krw(3000), Money.krw(26000), Money.krw(2000), Money.krw(1000), Money.krw(24000));
+
+    assertThat(plan.committed()).isEqualTo(Money.krw(3000));
+    assertThat(plan.payout()).isEqualTo(Money.krw(26000));
+  }
+
+  @Test
+  void rejectsEitherConservationViolation() {
+    assertThatThrownBy(
+            () ->
+                new SettlementMoneyPlan(
+                    Money.krw(3000),
+                    Money.krw(26000),
+                    Money.krw(1000),
+                    Money.krw(1000),
+                    Money.krw(25000)))
+        .isInstanceOf(IllegalArgumentException.class);
+  }
+}


## `feat(settlement): release locked stake first`

diff --git a/src/main/java/com/sportsbook/settlement/execution/SettlementWalletExecutor.java b/src/main/java/com/sportsbook/settlement/execution/SettlementWalletExecutor.java
new file mode 100644
index 0000000..149aef0
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/execution/SettlementWalletExecutor.java
@@ -0,0 +1,28 @@
+package com.sportsbook.settlement.execution;
+
+import com.sportsbook.settlement.client.WalletClient;
+import com.sportsbook.settlement.client.WalletCreditPurpose;
+import java.util.Optional;
+import java.util.UUID;
+import org.springframework.stereotype.Component;
+
+@Component
+public class SettlementWalletExecutor {
+
+  private final WalletClient wallet;
+
+  public SettlementWalletExecutor(WalletClient wallet) {
+    this.wallet = wallet;
+  }
+
+  public Optional<UUID> releaseLocked(SettlementAttempt attempt, UUID userId) {
+    if (attempt.money().lockedRelease().isZero()) {
+      return Optional.empty();
+    }
+    boolean wholeSlipVoid = attempt.action() == SettlementAttempt.Action.VOID;
+    String key = (wholeSlipVoid ? "void:refund:" : "settle:refund:") + attempt.betId();
+    WalletCreditPurpose purpose =
+        wholeSlipVoid ? WalletCreditPurpose.WHOLE_SLIP_VOID : WalletCreditPurpose.RETURNED_STAKE;
+    return Optional.of(wallet.credit(key, userId, attempt.money().lockedRelease(), purpose));
+  }
+}


## `feat(settlement): forfeit lost locked stake`

diff --git a/src/main/java/com/sportsbook/settlement/execution/SettlementWalletExecutor.java b/src/main/java/com/sportsbook/settlement/execution/SettlementWalletExecutor.java
index 149aef0..68b590f 100644
--- a/src/main/java/com/sportsbook/settlement/execution/SettlementWalletExecutor.java
+++ b/src/main/java/com/sportsbook/settlement/execution/SettlementWalletExecutor.java
@@ -25,4 +25,13 @@ public class SettlementWalletExecutor {
         wholeSlipVoid ? WalletCreditPurpose.WHOLE_SLIP_VOID : WalletCreditPurpose.RETURNED_STAKE;
     return Optional.of(wallet.credit(key, userId, attempt.money().lockedRelease(), purpose));
   }
+
+  public Optional<UUID> forfeitLocked(SettlementAttempt attempt, UUID userId) {
+    if (attempt.money().lockedForfeit().isZero()) {
+      return Optional.empty();
+    }
+    return Optional.of(
+        wallet.forfeit(
+            "settle:forfeit:" + attempt.betId(), userId, attempt.money().lockedForfeit()));
+  }
 }


## `feat(settlement): pay house funded profit`

diff --git a/src/main/java/com/sportsbook/settlement/execution/SettlementWalletExecutor.java b/src/main/java/com/sportsbook/settlement/execution/SettlementWalletExecutor.java
index 68b590f..21c5d57 100644
--- a/src/main/java/com/sportsbook/settlement/execution/SettlementWalletExecutor.java
+++ b/src/main/java/com/sportsbook/settlement/execution/SettlementWalletExecutor.java
@@ -34,4 +34,16 @@ public class SettlementWalletExecutor {
         wallet.forfeit(
             "settle:forfeit:" + attempt.betId(), userId, attempt.money().lockedForfeit()));
   }
+
+  public Optional<UUID> payHouseProfit(SettlementAttempt attempt, UUID userId) {
+    if (attempt.money().houseProfit().isZero()) {
+      return Optional.empty();
+    }
+    return Optional.of(
+        wallet.credit(
+            "settle:payout:" + attempt.betId(),
+            userId,
+            attempt.money().houseProfit(),
+            WalletCreditPurpose.PROFIT_PAYOUT));
+  }
 }


## `feat(settlement): define executable attempt context`

diff --git a/src/main/java/com/sportsbook/settlement/execution/SettlementExecution.java b/src/main/java/com/sportsbook/settlement/execution/SettlementExecution.java
new file mode 100644
index 0000000..311d810
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/execution/SettlementExecution.java
@@ -0,0 +1,12 @@
+package com.sportsbook.settlement.execution;
+
+import java.util.Objects;
+import java.util.UUID;
+
+public record SettlementExecution(SettlementAttempt attempt, UUID userId) {
+
+  public SettlementExecution {
+    Objects.requireNonNull(attempt, "attempt");
+    Objects.requireNonNull(userId, "userId");
+  }
+}


## `test(outbox): verify whole slip void refunds`

diff --git a/src/test/java/com/sportsbook/settlement/outbox/SettlementEventFactoryTest.java b/src/test/java/com/sportsbook/settlement/outbox/SettlementEventFactoryTest.java
index 5783dae..8840c0b 100644
--- a/src/test/java/com/sportsbook/settlement/outbox/SettlementEventFactoryTest.java
+++ b/src/test/java/com/sportsbook/settlement/outbox/SettlementEventFactoryTest.java
@@ -4,7 +4,9 @@ import static org.assertj.core.api.Assertions.assertThat;
 
 import com.sportsbook.protocol.domain.SettlementResult;
 import com.sportsbook.protocol.event.BetSettled;
+import com.sportsbook.protocol.event.BetVoided;
 import com.sportsbook.protocol.event.SettlementResultAvro;
+import com.sportsbook.protocol.event.VoidReason;
 import com.sportsbook.protocol.value.Currency;
 import com.sportsbook.protocol.value.Money;
 import com.sportsbook.protocol.value.Odds;
@@ -59,6 +61,33 @@ class SettlementEventFactoryTest {
     assertThat(event.getResultDetail()).hasSize(3);
   }
 
+  @Test
+  void emitsWholeSlipExposureForLifecycleVoidRefunds() {
+    UUID eventId = UUID.randomUUID();
+    Bet bet =
+        Bet.pending(
+            UUID.randomUUID(),
+            UUID.randomUUID(),
+            SlipKind.SYSTEM,
+            2,
+            3,
+            new EmbeddedMoney(1_000, Currency.KRW),
+            Instant.EPOCH,
+            List.of(selection(eventId), selection(eventId), selection(eventId)),
+            Instant.EPOCH);
+    bet.recordVoided(Money.krw(3_000), Instant.EPOCH);
+    SettlementEventFactory factory =
+        new SettlementEventFactory(
+            new SettlementTopics(null, null, null, null, null, null), new StrictAvroEncoder());
+
+    OutboxEvent outbox = factory.voided(bet, eventId, VoidReason.EVENT_CANCELLED);
+    BetVoided event = new StrictAvroDecoder().decode(outbox.payload(), BetVoided.class);
+
+    assertThat(outbox.topic()).isEqualTo("bet.voided.v1");
+    assertThat(event.getReason()).isEqualTo(VoidReason.EVENT_CANCELLED);
+    assertThat(event.getRefund().getAmount()).isEqualTo(3_000);
+  }
+
   private static BetSelection selection(UUID eventId) {
     return new BetSelection(
         eventId, UUID.randomUUID(), UUID.randomUUID(), Odds.ofDecimal("2.0000"));
