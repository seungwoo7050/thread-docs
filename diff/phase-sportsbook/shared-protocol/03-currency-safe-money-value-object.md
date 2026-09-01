# 통화 안전 금액 값 객체

## `feat(money): define supported currencies`

diff --git a/src/main/java/com/sportsbook/protocol/value/Currency.java b/src/main/java/com/sportsbook/protocol/value/Currency.java
new file mode 100644
index 0000000..41f19de
--- /dev/null
+++ b/src/main/java/com/sportsbook/protocol/value/Currency.java
@@ -0,0 +1,7 @@
+package com.sportsbook.protocol.value;
+
+/** Supported currencies (ADR-0003: KRW + USD multi-currency). */
+public enum Currency {
+  KRW,
+  USD,
+}


## `test(money): verify supported currencies`

diff --git a/src/test/java/com/sportsbook/protocol/value/CurrencyTest.java b/src/test/java/com/sportsbook/protocol/value/CurrencyTest.java
new file mode 100644
index 0000000..5ac67be
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/value/CurrencyTest.java
@@ -0,0 +1,19 @@
+package com.sportsbook.protocol.value;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import org.junit.jupiter.api.Test;
+
+class CurrencyTest {
+
+  @Test
+  void exactlyTwoCurrenciesDefinedInV1() {
+    assertThat(Currency.values()).containsExactlyInAnyOrder(Currency.KRW, Currency.USD);
+  }
+
+  @Test
+  void valueOfRoundTripsByName() {
+    assertThat(Currency.valueOf("KRW")).isEqualTo(Currency.KRW);
+    assertThat(Currency.valueOf("USD")).isEqualTo(Currency.USD);
+  }
+}


## `feat(money): define monetary amounts`

diff --git a/src/main/java/com/sportsbook/protocol/value/Money.java b/src/main/java/com/sportsbook/protocol/value/Money.java
new file mode 100644
index 0000000..ec7caeb
--- /dev/null
+++ b/src/main/java/com/sportsbook/protocol/value/Money.java
@@ -0,0 +1,82 @@
+package com.sportsbook.protocol.value;
+
+import com.fasterxml.jackson.annotation.JsonAutoDetect;
+import com.fasterxml.jackson.annotation.JsonAutoDetect.Visibility;
+import java.util.Objects;
+
+/**
+ * Money as {@code long} minor units + a {@link Currency} discriminator (ADR-0003).
+ *
+ * <p>Negative amounts are allowed because ledger entries (debit/credit) need both signs;
+ * domain-level "balance must be non-negative" rules live in wallet-service, not here. All
+ * arithmetic uses {@link Math#addExact} / {@link Math#multiplyExact} / {@link Math#subtractExact} /
+ * {@link Math#negateExact} so silent overflow is impossible — an overflowing operation throws
+ * {@link ArithmeticException}.
+ *
+ * <p>{@code @JsonAutoDetect(isGetterVisibility = NONE)}: Jackson otherwise treats {@code isZero()}
+ * / {@code isPositive()} / {@code isNegative()} as boolean bean properties and leaks them into the
+ * JSON payload alongside the record components. Disabling is-getter discovery limits the wire shape
+ * to {@code amount} + {@code currency}.
+ */
+@JsonAutoDetect(isGetterVisibility = Visibility.NONE)
+public record Money(long amount, Currency currency) implements Comparable<Money> {
+
+  public Money {
+    Objects.requireNonNull(currency, "currency");
+  }
+
+  public Money add(Money other) {
+    requireSameCurrency(other);
+    return new Money(Math.addExact(amount, other.amount), currency);
+  }
+
+  public Money subtract(Money other) {
+    requireSameCurrency(other);
+    return new Money(Math.subtractExact(amount, other.amount), currency);
+  }
+
+  public Money multiply(long multiplier) {
+    return new Money(Math.multiplyExact(amount, multiplier), currency);
+  }
+
+  public Money negate() {
+    return new Money(Math.negateExact(amount), currency);
+  }
+
+  @Override
+  public int compareTo(Money other) {
+    requireSameCurrency(other);
+    return Long.compare(amount, other.amount);
+  }
+
+  public boolean isZero() {
+    return amount == 0L;
+  }
+
+  public boolean isPositive() {
+    return amount > 0L;
+  }
+
+  public boolean isNegative() {
+    return amount < 0L;
+  }
+
+  private void requireSameCurrency(Money other) {
+    if (currency != other.currency) {
+      throw new IllegalArgumentException(
+          "Currency mismatch: " + currency + " vs " + other.currency);
+    }
+  }
+
+  public static Money krw(long amount) {
+    return new Money(amount, Currency.KRW);
+  }
+
+  public static Money usd(long amount) {
+    return new Money(amount, Currency.USD);
+  }
+
+  public static Money zero(Currency currency) {
+    return new Money(0L, currency);
+  }
+}


## `test(money): verify arithmetic and currency safety`

diff --git a/src/test/java/com/sportsbook/protocol/value/MoneyArithmeticTest.java b/src/test/java/com/sportsbook/protocol/value/MoneyArithmeticTest.java
new file mode 100644
index 0000000..2a13a58
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/value/MoneyArithmeticTest.java
@@ -0,0 +1,49 @@
+package com.sportsbook.protocol.value;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatExceptionOfType;
+import static org.assertj.core.api.Assertions.assertThatIllegalArgumentException;
+
+import org.junit.jupiter.api.Test;
+
+class MoneyArithmeticTest {
+
+  @Test
+  void factoriesPreserveAmountAndCurrency() {
+    assertThat(Money.krw(1_000)).isEqualTo(new Money(1_000, Currency.KRW));
+    assertThat(Money.usd(250)).isEqualTo(new Money(250, Currency.USD));
+    assertThat(Money.zero(Currency.KRW)).isEqualTo(Money.krw(0));
+  }
+
+  @Test
+  void arithmeticPreservesCurrency() {
+    assertThat(Money.krw(10_000).add(Money.krw(2_500))).isEqualTo(Money.krw(12_500));
+    assertThat(Money.krw(10_000).subtract(Money.krw(2_500))).isEqualTo(Money.krw(7_500));
+    assertThat(Money.usd(250).multiply(4)).isEqualTo(Money.usd(1_000));
+    assertThat(Money.krw(500).negate()).isEqualTo(Money.krw(-500));
+  }
+
+  @Test
+  void crossCurrencyOperationsAreRejected() {
+    assertThatIllegalArgumentException()
+        .isThrownBy(() -> Money.krw(100).add(Money.usd(100)))
+        .withMessageContaining("Currency mismatch");
+    assertThatIllegalArgumentException().isThrownBy(() -> Money.krw(100).compareTo(Money.usd(100)));
+  }
+
+  @Test
+  void overflowIsNeverSilent() {
+    assertThatExceptionOfType(ArithmeticException.class)
+        .isThrownBy(() -> Money.krw(Long.MAX_VALUE).add(Money.krw(1)));
+    assertThatExceptionOfType(ArithmeticException.class)
+        .isThrownBy(() -> Money.krw(Long.MAX_VALUE).multiply(2));
+  }
+
+  @Test
+  void comparisonAndSignHelpersReflectAmount() {
+    assertThat(Money.krw(100)).isLessThan(Money.krw(200));
+    assertThat(Money.krw(0).isZero()).isTrue();
+    assertThat(Money.krw(1).isPositive()).isTrue();
+    assertThat(Money.krw(-1).isNegative()).isTrue();
+  }
+}


## `test(money): verify monetary JSON shape`

diff --git a/src/test/java/com/sportsbook/protocol/value/MoneyJsonTest.java b/src/test/java/com/sportsbook/protocol/value/MoneyJsonTest.java
new file mode 100644
index 0000000..ce9b9a3
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/value/MoneyJsonTest.java
@@ -0,0 +1,24 @@
+package com.sportsbook.protocol.value;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import org.junit.jupiter.api.Test;
+
+class MoneyJsonTest {
+
+  private final ObjectMapper mapper = new ObjectMapper();
+
+  @Test
+  void serializationContainsOnlyRecordComponents() throws Exception {
+    assertThat(mapper.writeValueAsString(Money.krw(123_456)))
+        .isEqualTo("{\"amount\":123456,\"currency\":\"KRW\"}");
+  }
+
+  @Test
+  void jsonRoundTripPreservesMoney() throws Exception {
+    Money original = Money.usd(12_345);
+    assertThat(mapper.readValue(mapper.writeValueAsString(original), Money.class))
+        .isEqualTo(original);
+  }
+}
