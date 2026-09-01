# 베팅 라인 전개와 지급액 계산

## `feat(resolver): model resolved selection snapshots`

diff --git a/src/main/java/com/sportsbook/settlement/resolver/ResolvedSelection.java b/src/main/java/com/sportsbook/settlement/resolver/ResolvedSelection.java
new file mode 100644
index 0000000..9fb5aa3
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/resolver/ResolvedSelection.java
@@ -0,0 +1,24 @@
+package com.sportsbook.settlement.resolver;
+
+import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.protocol.value.Odds;
+import java.util.Objects;
+import java.util.UUID;
+
+/** Immutable settle-time leg snapshot used by payout resolution. */
+public record ResolvedSelection(UUID selectionId, Odds odds, SettlementResult outcome) {
+
+  public ResolvedSelection {
+    Objects.requireNonNull(selectionId, "selectionId");
+    Objects.requireNonNull(odds, "odds");
+    Objects.requireNonNull(outcome, "outcome");
+  }
+
+  public boolean wins() {
+    return outcome == SettlementResult.WON;
+  }
+
+  public boolean returnsStake() {
+    return outcome == SettlementResult.PUSH || outcome == SettlementResult.VOID;
+  }
+}


## `feat(resolver): expand single and multiple lines`

diff --git a/src/main/java/com/sportsbook/settlement/resolver/SettlementLine.java b/src/main/java/com/sportsbook/settlement/resolver/SettlementLine.java
new file mode 100644
index 0000000..64b2749
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/resolver/SettlementLine.java
@@ -0,0 +1,14 @@
+package com.sportsbook.settlement.resolver;
+
+import java.util.List;
+
+/** One independently staked line in deterministic combination order. */
+public record SettlementLine(int ordinal, List<ResolvedSelection> selections) {
+
+  public SettlementLine {
+    if (ordinal < 0 || selections == null || selections.isEmpty()) {
+      throw new IllegalArgumentException("Settlement line requires an ordinal and selections");
+    }
+    selections = List.copyOf(selections);
+  }
+}
diff --git a/src/main/java/com/sportsbook/settlement/resolver/SettlementLineFactory.java b/src/main/java/com/sportsbook/settlement/resolver/SettlementLineFactory.java
new file mode 100644
index 0000000..afa2255
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/resolver/SettlementLineFactory.java
@@ -0,0 +1,24 @@
+package com.sportsbook.settlement.resolver;
+
+import com.sportsbook.protocol.domain.BetSlipType;
+import java.util.List;
+import java.util.Objects;
+
+/** Expands a slip into independently staked payout lines. */
+public final class SettlementLineFactory {
+
+  public List<SettlementLine> lines(BetSlipType slipType, List<ResolvedSelection> selections) {
+    Objects.requireNonNull(slipType, "slipType");
+    List<ResolvedSelection> snapshot = List.copyOf(selections);
+    if (slipType instanceof BetSlipType.Single && snapshot.size() != 1) {
+      throw new IllegalArgumentException("Single slip requires one resolved selection");
+    }
+    if (slipType instanceof BetSlipType.Multiple && snapshot.size() < 2) {
+      throw new IllegalArgumentException("Multiple slip requires at least two selections");
+    }
+    if (slipType instanceof BetSlipType.System) {
+      throw new IllegalArgumentException("System line expansion requires combination support");
+    }
+    return List.of(new SettlementLine(0, snapshot));
+  }
+}


## `feat(resolver): expand deterministic system combinations`

diff --git a/src/main/java/com/sportsbook/settlement/resolver/SettlementLineFactory.java b/src/main/java/com/sportsbook/settlement/resolver/SettlementLineFactory.java
index afa2255..4420f45 100644
--- a/src/main/java/com/sportsbook/settlement/resolver/SettlementLineFactory.java
+++ b/src/main/java/com/sportsbook/settlement/resolver/SettlementLineFactory.java
@@ -1,6 +1,7 @@
 package com.sportsbook.settlement.resolver;
 
 import com.sportsbook.protocol.domain.BetSlipType;
+import java.util.ArrayList;
 import java.util.List;
 import java.util.Objects;
 
@@ -16,9 +17,32 @@ public final class SettlementLineFactory {
     if (slipType instanceof BetSlipType.Multiple && snapshot.size() < 2) {
       throw new IllegalArgumentException("Multiple slip requires at least two selections");
     }
-    if (slipType instanceof BetSlipType.System) {
-      throw new IllegalArgumentException("System line expansion requires combination support");
+    if (slipType instanceof BetSlipType.System system) {
+      if (system.totalSelections() != snapshot.size()) {
+        throw new IllegalArgumentException("System N must match resolved selections");
+      }
+      List<SettlementLine> combinations = new ArrayList<>();
+      choose(snapshot, system.minWins(), 0, new ArrayList<>(), combinations);
+      return List.copyOf(combinations);
     }
     return List.of(new SettlementLine(0, snapshot));
   }
+
+  private static void choose(
+      List<ResolvedSelection> selections,
+      int remaining,
+      int next,
+      List<ResolvedSelection> chosen,
+      List<SettlementLine> lines) {
+    if (remaining == 0) {
+      lines.add(new SettlementLine(lines.size(), chosen));
+      return;
+    }
+    int lastStart = selections.size() - remaining;
+    for (int index = next; index <= lastStart; index++) {
+      chosen.add(selections.get(index));
+      choose(selections, remaining - 1, index + 1, chosen, lines);
+      chosen.remove(chosen.size() - 1);
+    }
+  }
 }


## `test(resolver): verify system combination ordering`

diff --git a/src/test/java/com/sportsbook/settlement/resolver/SystemCombinationTest.java b/src/test/java/com/sportsbook/settlement/resolver/SystemCombinationTest.java
new file mode 100644
index 0000000..544deec
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/resolver/SystemCombinationTest.java
@@ -0,0 +1,41 @@
+package com.sportsbook.settlement.resolver;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.protocol.value.Odds;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class SystemCombinationTest {
+
+  @Test
+  void expandsKOfNInStableLexicographicOrder() {
+    ResolvedSelection a = selection("00000000-0000-0000-0000-000000000001");
+    ResolvedSelection b = selection("00000000-0000-0000-0000-000000000002");
+    ResolvedSelection c = selection("00000000-0000-0000-0000-000000000003");
+    ResolvedSelection d = selection("00000000-0000-0000-0000-000000000004");
+
+    List<SettlementLine> lines =
+        new SettlementLineFactory().lines(new BetSlipType.System(2, 4), List.of(a, b, c, d));
+
+    assertThat(lines).hasSize(6);
+    assertThat(lines).extracting(SettlementLine::ordinal).containsExactly(0, 1, 2, 3, 4, 5);
+    assertThat(lines)
+        .extracting(line -> line.selections().stream().map(ResolvedSelection::selectionId).toList())
+        .containsExactly(
+            List.of(a.selectionId(), b.selectionId()),
+            List.of(a.selectionId(), c.selectionId()),
+            List.of(a.selectionId(), d.selectionId()),
+            List.of(b.selectionId(), c.selectionId()),
+            List.of(b.selectionId(), d.selectionId()),
+            List.of(c.selectionId(), d.selectionId()));
+  }
+
+  private static ResolvedSelection selection(String id) {
+    return new ResolvedSelection(
+        UUID.fromString(id), Odds.ofDecimal("2.0000"), SettlementResult.WON);
+  }
+}


## `feat(resolver): calculate unit stake line payouts`

diff --git a/src/main/java/com/sportsbook/settlement/resolver/PayoutCalculation.java b/src/main/java/com/sportsbook/settlement/resolver/PayoutCalculation.java
new file mode 100644
index 0000000..645b37d
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/resolver/PayoutCalculation.java
@@ -0,0 +1,18 @@
+package com.sportsbook.settlement.resolver;
+
+import com.sportsbook.protocol.value.Money;
+import java.util.Objects;
+
+/** Monetary result of line evaluation before slip-level classification. */
+public record PayoutCalculation(Money payout, int survivingLines, int totalLines) {
+
+  public PayoutCalculation {
+    Objects.requireNonNull(payout, "payout");
+    if (payout.isNegative()
+        || totalLines < 1
+        || survivingLines < 0
+        || survivingLines > totalLines) {
+      throw new IllegalArgumentException("Invalid payout line calculation");
+    }
+  }
+}
diff --git a/src/main/java/com/sportsbook/settlement/resolver/SettlementPayoutCalculator.java b/src/main/java/com/sportsbook/settlement/resolver/SettlementPayoutCalculator.java
new file mode 100644
index 0000000..e93f780
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/resolver/SettlementPayoutCalculator.java
@@ -0,0 +1,45 @@
+package com.sportsbook.settlement.resolver;
+
+import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.protocol.value.Money;
+import java.math.BigDecimal;
+import java.math.RoundingMode;
+import java.util.List;
+
+/** Calculates unit-stake line payouts with one final minor-unit floor. */
+public final class SettlementPayoutCalculator {
+
+  public PayoutCalculation calculate(List<SettlementLine> lines, Money unitStake) {
+    if (lines == null || lines.isEmpty() || unitStake == null || unitStake.amount() <= 0) {
+      throw new IllegalArgumentException("Payout requires lines and a positive unit stake");
+    }
+    BigDecimal summedProducts = BigDecimal.ZERO;
+    int surviving = 0;
+    for (SettlementLine line : lines) {
+      BigDecimal product = product(line);
+      summedProducts = summedProducts.add(product);
+      if (product.signum() > 0) {
+        surviving++;
+      }
+    }
+    long amount =
+        BigDecimal.valueOf(unitStake.amount())
+            .multiply(summedProducts)
+            .setScale(0, RoundingMode.FLOOR)
+            .longValueExact();
+    return new PayoutCalculation(new Money(amount, unitStake.currency()), surviving, lines.size());
+  }
+
+  private static BigDecimal product(SettlementLine line) {
+    BigDecimal product = BigDecimal.ONE;
+    for (ResolvedSelection selection : line.selections()) {
+      if (selection.outcome() == SettlementResult.LOST) {
+        return BigDecimal.ZERO;
+      }
+      if (selection.outcome() == SettlementResult.WON) {
+        product = product.multiply(selection.odds().decimal());
+      }
+    }
+    return product;
+  }
+}


## `test(resolver): verify line sums and final rounding`

diff --git a/src/test/java/com/sportsbook/settlement/resolver/SettlementPayoutCalculatorTest.java b/src/test/java/com/sportsbook/settlement/resolver/SettlementPayoutCalculatorTest.java
new file mode 100644
index 0000000..f120afb
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/resolver/SettlementPayoutCalculatorTest.java
@@ -0,0 +1,50 @@
+package com.sportsbook.settlement.resolver;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.Odds;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class SettlementPayoutCalculatorTest {
+
+  private final SettlementLineFactory lines = new SettlementLineFactory();
+  private final SettlementPayoutCalculator calculator = new SettlementPayoutCalculator();
+
+  @Test
+  void sumsSystemLineProductsBeforeOneFinalFloor() {
+    List<ResolvedSelection> selections =
+        List.of(
+            selection("1.5000", SettlementResult.WON), selection("1.5000", SettlementResult.WON));
+
+    PayoutCalculation payout =
+        calculator.calculate(lines.lines(new BetSlipType.System(1, 2), selections), Money.krw(1));
+
+    assertThat(payout.payout()).isEqualTo(Money.krw(3));
+    assertThat(payout.survivingLines()).isEqualTo(2);
+    assertThat(payout.totalLines()).isEqualTo(2);
+  }
+
+  @Test
+  void killsLostLinesAndTreatsPushOrVoidAsNeutral() {
+    List<ResolvedSelection> selections =
+        List.of(
+            selection("2.0000", SettlementResult.WON),
+            selection("3.0000", SettlementResult.LOST),
+            selection("4.0000", SettlementResult.VOID));
+    List<SettlementLine> system = lines.lines(new BetSlipType.System(2, 3), selections);
+
+    PayoutCalculation payout = calculator.calculate(system, Money.krw(1_000));
+
+    assertThat(payout.payout()).isEqualTo(Money.krw(2_000));
+    assertThat(payout.survivingLines()).isOne();
+  }
+
+  private static ResolvedSelection selection(String odds, SettlementResult result) {
+    return new ResolvedSelection(UUID.randomUUID(), Odds.ofDecimal(odds), result);
+  }
+}


## `feat(resolver): classify base settlement outcomes`

diff --git a/src/main/java/com/sportsbook/settlement/resolver/SettlementOutcome.java b/src/main/java/com/sportsbook/settlement/resolver/SettlementOutcome.java
new file mode 100644
index 0000000..6bb8eb7
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/resolver/SettlementOutcome.java
@@ -0,0 +1,18 @@
+package com.sportsbook.settlement.resolver;
+
+import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.protocol.value.Money;
+import java.util.Objects;
+
+/** Full base resolution snapshot stored and emitted for a bet. */
+public record SettlementOutcome(
+    SettlementResult result, Money payout, int survivingLines, int totalLines) {
+
+  public SettlementOutcome {
+    Objects.requireNonNull(result, "result");
+    Objects.requireNonNull(payout, "payout");
+    if (survivingLines < 0 || totalLines < 1 || survivingLines > totalLines) {
+      throw new IllegalArgumentException("Invalid settlement outcome line counts");
+    }
+  }
+}
diff --git a/src/main/java/com/sportsbook/settlement/resolver/SettlementResolver.java b/src/main/java/com/sportsbook/settlement/resolver/SettlementResolver.java
new file mode 100644
index 0000000..08c743e
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/resolver/SettlementResolver.java
@@ -0,0 +1,36 @@
+package com.sportsbook.settlement.resolver;
+
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.protocol.value.Money;
+import java.util.List;
+
+/** Resolves base slip result and payout from authoritative selection outcomes. */
+public final class SettlementResolver {
+
+  private final SettlementLineFactory lines = new SettlementLineFactory();
+  private final SettlementPayoutCalculator payouts = new SettlementPayoutCalculator();
+
+  public SettlementOutcome resolve(
+      BetSlipType slipType, List<ResolvedSelection> selections, Money unitStake) {
+    PayoutCalculation payout = payouts.calculate(lines.lines(slipType, selections), unitStake);
+    return new SettlementOutcome(
+        classify(selections, payout),
+        payout.payout(),
+        payout.survivingLines(),
+        payout.totalLines());
+  }
+
+  private static SettlementResult classify(
+      List<ResolvedSelection> selections, PayoutCalculation payout) {
+    if (payout.payout().isZero()) {
+      return SettlementResult.LOST;
+    }
+    if (selections.stream().anyMatch(ResolvedSelection::wins)) {
+      return SettlementResult.WON;
+    }
+    boolean allVoid =
+        selections.stream().allMatch(selection -> selection.outcome() == SettlementResult.VOID);
+    return allVoid ? SettlementResult.VOID : SettlementResult.PUSH;
+  }
+}


## `test(resolver): distinguish result void from lifecycle void`

diff --git a/src/test/java/com/sportsbook/settlement/resolver/SettlementResolverTest.java b/src/test/java/com/sportsbook/settlement/resolver/SettlementResolverTest.java
new file mode 100644
index 0000000..9638d90
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/resolver/SettlementResolverTest.java
@@ -0,0 +1,57 @@
+package com.sportsbook.settlement.resolver;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.Odds;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class SettlementResolverTest {
+
+  private final SettlementResolver resolver = new SettlementResolver();
+
+  @Test
+  void classifiesStraightWinLossAndPush() {
+    assertThat(resolve(SettlementResult.WON).result()).isEqualTo(SettlementResult.WON);
+    assertThat(resolve(SettlementResult.LOST).result()).isEqualTo(SettlementResult.LOST);
+    assertThat(resolve(SettlementResult.PUSH).result()).isEqualTo(SettlementResult.PUSH);
+    assertThat(resolve(SettlementResult.PUSH).payout()).isEqualTo(Money.krw(100));
+  }
+
+  @Test
+  void allVoidIsASettledResultVoidNotAWholeSlipLifecycleVoid() {
+    SettlementOutcome outcome = resolve(SettlementResult.VOID);
+
+    assertThat(outcome.result()).isEqualTo(SettlementResult.VOID);
+    assertThat(outcome.payout()).isEqualTo(Money.krw(100));
+  }
+
+  @Test
+  void classifiesAPartiallyWinningSystemFromItsSurvivingLine() {
+    List<ResolvedSelection> selections =
+        List.of(
+            selection("2.0000", SettlementResult.WON),
+            selection("3.0000", SettlementResult.WON),
+            selection("4.0000", SettlementResult.LOST));
+
+    SettlementOutcome outcome =
+        resolver.resolve(new BetSlipType.System(2, 3), selections, Money.krw(1_000));
+
+    assertThat(outcome.result()).isEqualTo(SettlementResult.WON);
+    assertThat(outcome.payout()).isEqualTo(Money.krw(6_000));
+    assertThat(outcome.survivingLines()).isOne();
+  }
+
+  private SettlementOutcome resolve(SettlementResult result) {
+    return resolver.resolve(
+        new BetSlipType.Single(), List.of(selection("2.0000", result)), Money.krw(100));
+  }
+
+  private static ResolvedSelection selection(String odds, SettlementResult result) {
+    return new ResolvedSelection(UUID.randomUUID(), Odds.ofDecimal(odds), result);
+  }
+}
