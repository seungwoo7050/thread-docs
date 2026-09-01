# Redis Lua 정밀도 경계와 안전 정수 모델

## `feat(numeric): bound Redis integer arithmetic`

diff --git a/src/main/java/com/sportsbook/risk/policy/SafeRedisNumber.java b/src/main/java/com/sportsbook/risk/policy/SafeRedisNumber.java
new file mode 100644
index 0000000..1f65de6
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/policy/SafeRedisNumber.java
@@ -0,0 +1,36 @@
+package com.sportsbook.risk.policy;
+
+/** Integer operations that remain exact when Redis Lua represents numbers as doubles. */
+public final class SafeRedisNumber {
+  public static final long MAX_VALUE = 9_007_199_254_740_991L;
+
+  private SafeRedisNumber() {}
+
+  public static long requireNonNegative(long value, String name) {
+    if (value < 0L || value > MAX_VALUE) {
+      throw new IllegalArgumentException(name + " must be between 0 and " + MAX_VALUE);
+    }
+    return value;
+  }
+
+  public static long requirePositive(long value, String name) {
+    if (value == 0L) {
+      throw new IllegalArgumentException(name + " must be positive");
+    }
+    return requireNonNegative(value, name);
+  }
+
+  public static long add(long left, long right, String name) {
+    requireNonNegative(left, name);
+    requireNonNegative(right, name);
+    long result = Math.addExact(left, right);
+    return requireNonNegative(result, name);
+  }
+
+  public static long multiply(long value, long multiplier, String name) {
+    requireNonNegative(value, name);
+    requireNonNegative(multiplier, name);
+    long result = Math.multiplyExact(value, multiplier);
+    return requireNonNegative(result, name);
+  }
+}


## `test(numeric): verify Redis integer boundaries`

diff --git a/src/test/java/com/sportsbook/risk/policy/SafeRedisNumberTest.java b/src/test/java/com/sportsbook/risk/policy/SafeRedisNumberTest.java
new file mode 100644
index 0000000..eda3283
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/policy/SafeRedisNumberTest.java
@@ -0,0 +1,41 @@
+package com.sportsbook.risk.policy;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import org.junit.jupiter.api.Test;
+
+class SafeRedisNumberTest {
+  @Test
+  void acceptsTheLargestExactlyRepresentableInteger() {
+    assertThat(SafeRedisNumber.requireNonNegative(SafeRedisNumber.MAX_VALUE, "amount"))
+        .isEqualTo(SafeRedisNumber.MAX_VALUE);
+  }
+
+  @Test
+  void rejectsNegativeAndInexactValues() {
+    assertThatThrownBy(() -> SafeRedisNumber.requireNonNegative(-1L, "amount"))
+        .isInstanceOf(IllegalArgumentException.class);
+    assertThatThrownBy(
+            () -> SafeRedisNumber.requireNonNegative(SafeRedisNumber.MAX_VALUE + 1L, "amount"))
+        .isInstanceOf(IllegalArgumentException.class);
+  }
+
+  @Test
+  void checksAdditionAndMultiplicationResults() {
+    assertThat(SafeRedisNumber.add(40L, 2L, "amount")).isEqualTo(42L);
+    assertThat(SafeRedisNumber.multiply(21L, 2L, "amount")).isEqualTo(42L);
+
+    assertThatThrownBy(() -> SafeRedisNumber.add(SafeRedisNumber.MAX_VALUE, 1L, "amount"))
+        .isInstanceOf(IllegalArgumentException.class);
+    assertThatThrownBy(() -> SafeRedisNumber.multiply(SafeRedisNumber.MAX_VALUE, 2L, "amount"))
+        .isInstanceOf(IllegalArgumentException.class);
+  }
+
+  @Test
+  void requiresPositiveInputsWhenRequested() {
+    assertThatThrownBy(() -> SafeRedisNumber.requirePositive(0L, "stake"))
+        .isInstanceOf(IllegalArgumentException.class)
+        .hasMessageContaining("positive");
+  }
+}


## `feat(command): define typed risk candidates`

diff --git a/src/main/java/com/sportsbook/risk/service/RiskCheckCommand.java b/src/main/java/com/sportsbook/risk/service/RiskCheckCommand.java
new file mode 100644
index 0000000..756b955
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/service/RiskCheckCommand.java
@@ -0,0 +1,38 @@
+package com.sportsbook.risk.service;
+
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.SelectionId;
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.policy.SafeRedisNumber;
+import java.time.Instant;
+import java.util.HashSet;
+import java.util.List;
+import java.util.Objects;
+
+/** Fully validated candidate shared by diagnostic, reservation, and event paths. */
+public record RiskCheckCommand(
+    UserId userId, BetId betId, Money stake, List<SelectionId> selectionIds, Instant now) {
+
+  public static final int MAX_SELECTIONS = 15;
+
+  public RiskCheckCommand {
+    Objects.requireNonNull(userId, "userId");
+    Objects.requireNonNull(betId, "betId");
+    Objects.requireNonNull(stake, "stake");
+    Objects.requireNonNull(selectionIds, "selectionIds");
+    Objects.requireNonNull(now, "now");
+    SafeRedisNumber.requirePositive(stake.amount(), "stake.amount");
+    if (selectionIds.isEmpty() || selectionIds.size() > MAX_SELECTIONS) {
+      throw new IllegalArgumentException(
+          "selectionIds size must be between 1 and " + MAX_SELECTIONS);
+    }
+    if (selectionIds.stream().anyMatch(Objects::isNull)) {
+      throw new IllegalArgumentException("selectionIds must not contain null");
+    }
+    if (new HashSet<>(selectionIds).size() != selectionIds.size()) {
+      throw new IllegalArgumentException("selectionIds must be unique");
+    }
+    selectionIds = List.copyOf(selectionIds);
+  }
+}


## `test(command): verify typed risk candidates`

diff --git a/src/test/java/com/sportsbook/risk/service/RiskCheckCommandTest.java b/src/test/java/com/sportsbook/risk/service/RiskCheckCommandTest.java
new file mode 100644
index 0000000..c264b65
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/service/RiskCheckCommandTest.java
@@ -0,0 +1,46 @@
+package com.sportsbook.risk.service;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.SelectionId;
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.policy.SafeRedisNumber;
+import java.time.Instant;
+import java.util.ArrayList;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class RiskCheckCommandTest {
+  private static final UserId USER_ID = UserId.of(UUID.randomUUID());
+  private static final BetId BET_ID = BetId.of(UUID.randomUUID());
+  private static final SelectionId SELECTION_ID = SelectionId.of(UUID.randomUUID());
+
+  @Test
+  void retainsTypedCandidateValues() {
+    var command = command(new Money(100L, Currency.KRW), new ArrayList<>(List.of(SELECTION_ID)));
+
+    assertThat(command.userId()).isEqualTo(USER_ID);
+    assertThat(command.betId()).isEqualTo(BET_ID);
+    assertThat(command.selectionIds()).containsExactly(SELECTION_ID);
+  }
+
+  @Test
+  void rejectsAmountsOutsideTheRedisIntegerDomain() {
+    assertThatThrownBy(() -> command(new Money(0L, Currency.KRW), List.of(SELECTION_ID)))
+        .isInstanceOf(IllegalArgumentException.class);
+    assertThatThrownBy(
+            () ->
+                command(
+                    new Money(SafeRedisNumber.MAX_VALUE + 1L, Currency.KRW), List.of(SELECTION_ID)))
+        .isInstanceOf(IllegalArgumentException.class);
+  }
+
+  private static RiskCheckCommand command(Money stake, List<SelectionId> selections) {
+    return new RiskCheckCommand(USER_ID, BET_ID, stake, selections, Instant.EPOCH);
+  }
+}


## `test(command): enforce selection boundaries`

diff --git a/src/test/java/com/sportsbook/risk/service/RiskCheckSelectionTest.java b/src/test/java/com/sportsbook/risk/service/RiskCheckSelectionTest.java
new file mode 100644
index 0000000..1ed6a76
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/service/RiskCheckSelectionTest.java
@@ -0,0 +1,58 @@
+package com.sportsbook.risk.service;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.SelectionId;
+import com.sportsbook.protocol.value.UserId;
+import java.time.Instant;
+import java.util.ArrayList;
+import java.util.List;
+import java.util.UUID;
+import java.util.stream.IntStream;
+import org.junit.jupiter.api.Test;
+
+class RiskCheckSelectionTest {
+  @Test
+  void requiresOneToFifteenUniqueSelections() {
+    SelectionId selection = selection(1);
+
+    assertThatThrownBy(() -> command(List.of())).isInstanceOf(IllegalArgumentException.class);
+    assertThatThrownBy(() -> command(List.of(selection, selection)))
+        .isInstanceOf(IllegalArgumentException.class)
+        .hasMessageContaining("unique");
+    assertThatThrownBy(
+            () ->
+                command(
+                    IntStream.rangeClosed(1, RiskCheckCommand.MAX_SELECTIONS + 1)
+                        .mapToObj(RiskCheckSelectionTest::selection)
+                        .toList()))
+        .isInstanceOf(IllegalArgumentException.class);
+  }
+
+  @Test
+  void isolatesSelectionsFromCallerMutation() {
+    var mutable = new ArrayList<>(List.of(selection(1)));
+    RiskCheckCommand command = command(mutable);
+    mutable.add(selection(2));
+
+    assertThat(command.selectionIds()).hasSize(1);
+    assertThatThrownBy(() -> command.selectionIds().add(selection(3)))
+        .isInstanceOf(UnsupportedOperationException.class);
+  }
+
+  private static RiskCheckCommand command(List<SelectionId> selections) {
+    return new RiskCheckCommand(
+        UserId.of(UUID.randomUUID()),
+        BetId.of(UUID.randomUUID()),
+        Money.krw(100L),
+        selections,
+        Instant.EPOCH);
+  }
+
+  private static SelectionId selection(int suffix) {
+    return SelectionId.of(new UUID(0L, suffix));
+  }
+}


## `feat(events): calculate accepted bet exposure`

diff --git a/src/main/java/com/sportsbook/risk/event/AcceptedBetExposure.java b/src/main/java/com/sportsbook/risk/event/AcceptedBetExposure.java
new file mode 100644
index 0000000..5bf6b90
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/event/AcceptedBetExposure.java
@@ -0,0 +1,82 @@
+package com.sportsbook.risk.event;
+
+import com.sportsbook.protocol.event.BetPlacedRequested;
+import com.sportsbook.protocol.event.BetSlipTypeTag;
+import com.sportsbook.risk.policy.SafeRedisNumber;
+import com.sportsbook.risk.service.RiskCheckCommand;
+import java.util.Objects;
+
+/** Exact monetary exposure represented by an accepted bet event. */
+public record AcceptedBetExposure(long totalAmount) {
+
+  public AcceptedBetExposure {
+    SafeRedisNumber.requirePositive(totalAmount, "totalAmount");
+  }
+
+  public static AcceptedBetExposure from(BetPlacedRequested event) {
+    Objects.requireNonNull(event, "event");
+    Objects.requireNonNull(event.getSelections(), "selections");
+    int selectionCount = event.getSelections().size();
+    if (selectionCount < 1 || selectionCount > RiskCheckCommand.MAX_SELECTIONS) {
+      throw new IllegalArgumentException("selection count must be between 1 and 15");
+    }
+
+    BetSlipTypeTag slipType = Objects.requireNonNull(event.getSlipType(), "slipType");
+    long unitAmount =
+        SafeRedisNumber.requirePositive(
+            Objects.requireNonNull(event.getStake(), "stake").getAmount(), "stake.amount");
+    long total =
+        switch (slipType) {
+          case SINGLE -> unitAmount(selectionCount, event);
+          case MULTIPLE -> multipleAmount(selectionCount, unitAmount, event);
+          case SYSTEM -> systemAmount(selectionCount, unitAmount, event);
+        };
+    return new AcceptedBetExposure(total);
+  }
+
+  private static long unitAmount(int actual, BetPlacedRequested event) {
+    requireNoSystemShape(event);
+    if (actual != 1) {
+      throw new IllegalArgumentException("SINGLE must contain exactly one selection");
+    }
+    return event.getStake().getAmount();
+  }
+
+  private static long multipleAmount(int count, long unitAmount, BetPlacedRequested event) {
+    requireNoSystemShape(event);
+    if (count < 2) {
+      throw new IllegalArgumentException("MULTIPLE must contain at least two selections");
+    }
+    return unitAmount;
+  }
+
+  private static long systemAmount(int count, long unitAmount, BetPlacedRequested event) {
+    Integer minimum = event.getSystemMinWins();
+    Integer total = event.getSystemTotalSelections();
+    if (minimum == null || total == null) {
+      throw new IllegalArgumentException("SYSTEM requires minWins and totalSelections");
+    }
+    if (total != count || total < 2 || total > RiskCheckCommand.MAX_SELECTIONS) {
+      throw new IllegalArgumentException("SYSTEM totalSelections must match 2..15 selections");
+    }
+    if (minimum < 1 || minimum > total) {
+      throw new IllegalArgumentException("SYSTEM minWins must be between 1 and totalSelections");
+    }
+    return SafeRedisNumber.multiply(unitAmount, combinations(total, minimum), "totalAmount");
+  }
+
+  private static void requireNoSystemShape(BetPlacedRequested event) {
+    if (event.getSystemMinWins() != null || event.getSystemTotalSelections() != null) {
+      throw new IllegalArgumentException("non-SYSTEM slip must not contain SYSTEM fields");
+    }
+  }
+
+  private static long combinations(int total, int choose) {
+    int smaller = Math.min(choose, total - choose);
+    long result = 1L;
+    for (int index = 1; index <= smaller; index++) {
+      result = Math.multiplyExact(result, total - smaller + index) / index;
+    }
+    return result;
+  }
+}


## `test(events): verify accepted bet exposure`

diff --git a/src/test/java/com/sportsbook/risk/event/AcceptedBetExposureTest.java b/src/test/java/com/sportsbook/risk/event/AcceptedBetExposureTest.java
new file mode 100644
index 0000000..249e320
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/event/AcceptedBetExposureTest.java
@@ -0,0 +1,90 @@
+package com.sportsbook.risk.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.protocol.event.BetPlacedRequested;
+import com.sportsbook.protocol.event.BetSlipTypeTag;
+import com.sportsbook.protocol.event.Money;
+import com.sportsbook.protocol.event.RequestedSelection;
+import com.sportsbook.risk.policy.SafeRedisNumber;
+import java.util.List;
+import java.util.stream.IntStream;
+import org.junit.jupiter.api.Test;
+
+class AcceptedBetExposureTest {
+  @Test
+  void singleAndMultipleKeepTheSubmittedAmount() {
+    assertThat(AcceptedBetExposure.from(event(BetSlipTypeTag.SINGLE, null, null, 1, 75L)))
+        .extracting(AcceptedBetExposure::totalAmount)
+        .isEqualTo(75L);
+    assertThat(AcceptedBetExposure.from(event(BetSlipTypeTag.MULTIPLE, null, null, 3, 90L)))
+        .extracting(AcceptedBetExposure::totalAmount)
+        .isEqualTo(90L);
+  }
+
+  @Test
+  void systemMultipliesTheUnitAmountByExactCombinations() {
+    assertThat(AcceptedBetExposure.from(event(BetSlipTypeTag.SYSTEM, 2, 5, 5, 100L)))
+        .extracting(AcceptedBetExposure::totalAmount)
+        .isEqualTo(1_000L);
+    assertThat(AcceptedBetExposure.from(event(BetSlipTypeTag.SYSTEM, 3, 5, 5, 100L)))
+        .extracting(AcceptedBetExposure::totalAmount)
+        .isEqualTo(1_000L);
+  }
+
+  @Test
+  void rejectsSlipShapeMismatches() {
+    assertThatThrownBy(
+            () -> AcceptedBetExposure.from(event(BetSlipTypeTag.SINGLE, null, null, 2, 1L)))
+        .hasMessageContaining("exactly one");
+    assertThatThrownBy(
+            () -> AcceptedBetExposure.from(event(BetSlipTypeTag.MULTIPLE, null, null, 1, 1L)))
+        .hasMessageContaining("at least two");
+    assertThatThrownBy(() -> AcceptedBetExposure.from(event(BetSlipTypeTag.SYSTEM, 2, 4, 3, 1L)))
+        .hasMessageContaining("totalSelections");
+    assertThatThrownBy(() -> AcceptedBetExposure.from(event(BetSlipTypeTag.SYSTEM, 4, 3, 3, 1L)))
+        .hasMessageContaining("minWins");
+  }
+
+  @Test
+  void rejectsSystemFieldsOnUnitTotalSlips() {
+    assertThatThrownBy(() -> AcceptedBetExposure.from(event(BetSlipTypeTag.MULTIPLE, 1, 2, 2, 1L)))
+        .hasMessageContaining("non-SYSTEM");
+  }
+
+  @Test
+  void rejectsUnsafeUnitAndCalculatedAmounts() {
+    assertThatThrownBy(
+            () -> AcceptedBetExposure.from(event(BetSlipTypeTag.SINGLE, null, null, 1, 0L)))
+        .hasMessageContaining("positive");
+    long unsafeUnit = SafeRedisNumber.MAX_VALUE / 3L + 1L;
+    assertThatThrownBy(
+            () -> AcceptedBetExposure.from(event(BetSlipTypeTag.SYSTEM, 2, 3, 3, unsafeUnit)))
+        .hasMessageContaining("totalAmount");
+  }
+
+  @Test
+  void capsEverySlipAtFifteenSelections() {
+    assertThatThrownBy(
+            () -> AcceptedBetExposure.from(event(BetSlipTypeTag.MULTIPLE, null, null, 16, 1L)))
+        .hasMessageContaining("between 1 and 15");
+  }
+
+  private static BetPlacedRequested event(
+      BetSlipTypeTag type, Integer minimum, Integer total, int selectionCount, long amount) {
+    List<RequestedSelection> selections =
+        IntStream.range(0, selectionCount).mapToObj(ignored -> new RequestedSelection()).toList();
+    return BetPlacedRequested.newBuilder()
+        .setBetId("10000000-0000-4000-8000-000000000001")
+        .setUserId("20000000-0000-4000-8000-000000000001")
+        .setSlipType(type)
+        .setSystemMinWins(minimum)
+        .setSystemTotalSelections(total)
+        .setSelections(selections)
+        .setStake(new Money(amount, "KRW"))
+        .setIdempotencyKey("accepted-exposure-test")
+        .setRequestedAt(java.time.Instant.EPOCH)
+        .build();
+  }
+}


## `feat(snapshot): define precision-safe snapshot wire`

diff --git a/src/main/java/com/sportsbook/risk/snapshot/RiskSnapshotWire.java b/src/main/java/com/sportsbook/risk/snapshot/RiskSnapshotWire.java
new file mode 100644
index 0000000..a49c532
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/snapshot/RiskSnapshotWire.java
@@ -0,0 +1,16 @@
+package com.sportsbook.risk.snapshot;
+
+import java.util.List;
+import java.util.Map;
+
+/** Precision-safe JSON contract returned by the combined Redis snapshot script. */
+record RiskSnapshotWire(
+    String version, String expired, Map<String, LimitSlot> limits, PatternFacts patterns) {
+  record LimitSlot(Boolean ok, String committed, String active, String override, String error) {}
+
+  record FactSlot(Boolean ok, String value, List<String> values, String error) {}
+
+  record PatternFacts(FactSlot rapid, FactSlot stakes, List<SelectionFact> selections) {}
+
+  record SelectionFact(String selectionId, FactSlot slot) {}
+}
diff --git a/src/main/java/com/sportsbook/risk/snapshot/RiskWireNumbers.java b/src/main/java/com/sportsbook/risk/snapshot/RiskWireNumbers.java
new file mode 100644
index 0000000..7561aff
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/snapshot/RiskWireNumbers.java
@@ -0,0 +1,23 @@
+package com.sportsbook.risk.snapshot;
+
+import com.sportsbook.risk.policy.SafeRedisNumber;
+
+/** Canonical integer parser for values kept as strings across Redis JSON. */
+final class RiskWireNumbers {
+  private RiskWireNumbers() {}
+
+  static long exact(String raw, String name) {
+    if (raw == null || !raw.matches("0|[1-9][0-9]*")) {
+      throw malformed(name);
+    }
+    try {
+      return SafeRedisNumber.requireNonNegative(Long.parseLong(raw), name);
+    } catch (IllegalArgumentException failure) {
+      throw new IllegalStateException("malformed snapshot integer: " + name, failure);
+    }
+  }
+
+  static IllegalStateException malformed(String name) {
+    return new IllegalStateException("malformed snapshot field: " + name);
+  }
+}


## `test(snapshot): verify precision-safe snapshot numbers`

diff --git a/src/test/java/com/sportsbook/risk/snapshot/RiskWireNumbersTest.java b/src/test/java/com/sportsbook/risk/snapshot/RiskWireNumbersTest.java
new file mode 100644
index 0000000..a0e5611
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/snapshot/RiskWireNumbersTest.java
@@ -0,0 +1,36 @@
+package com.sportsbook.risk.snapshot;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.risk.policy.SafeRedisNumber;
+import org.junit.jupiter.api.Test;
+
+class RiskWireNumbersTest {
+  @Test
+  void parsesCanonicalExactIntegers() {
+    assertThat(RiskWireNumbers.exact("0", "value")).isZero();
+    assertThat(RiskWireNumbers.exact(Long.toString(SafeRedisNumber.MAX_VALUE), "value"))
+        .isEqualTo(SafeRedisNumber.MAX_VALUE);
+  }
+
+  @Test
+  void rejectsNoncanonicalOrInexactValues() {
+    for (String value : new String[] {"", "01", "-1", "+1", "1.0", " 1"}) {
+      assertThatThrownBy(() -> RiskWireNumbers.exact(value, "value"))
+          .isInstanceOf(IllegalStateException.class)
+          .hasMessageContaining("value");
+    }
+    assertThatThrownBy(
+            () -> RiskWireNumbers.exact(Long.toString(SafeRedisNumber.MAX_VALUE + 1), "value"))
+        .isInstanceOf(IllegalStateException.class)
+        .hasMessageContaining("value");
+  }
+
+  @Test
+  void rejectsMissingValues() {
+    assertThatThrownBy(() -> RiskWireNumbers.exact(null, "value"))
+        .isInstanceOf(IllegalStateException.class)
+        .hasMessageContaining("value");
+  }
+}
