# 배당 정규화와 표시 변환

## `feat(odds): define normalized decimal odds`

diff --git a/src/main/java/com/sportsbook/protocol/value/Odds.java b/src/main/java/com/sportsbook/protocol/value/Odds.java
new file mode 100644
index 0000000..bf392b8
--- /dev/null
+++ b/src/main/java/com/sportsbook/protocol/value/Odds.java
@@ -0,0 +1,39 @@
+package com.sportsbook.protocol.value;
+
+import java.math.BigDecimal;
+import java.math.RoundingMode;
+import java.util.Objects;
+
+/** Decimal odds with scale-normalized equality. */
+public record Odds(BigDecimal decimal) {
+
+  public static final int SCALE = 4;
+
+  public Odds {
+    Objects.requireNonNull(decimal, "decimal");
+    if (decimal.compareTo(BigDecimal.ONE) < 0) {
+      throw new IllegalArgumentException("Odds must be at least 1.00: " + decimal);
+    }
+  }
+
+  @Override
+  public boolean equals(Object other) {
+    if (this == other) {
+      return true;
+    }
+    return other instanceof Odds odds && decimal.compareTo(odds.decimal) == 0;
+  }
+
+  @Override
+  public int hashCode() {
+    return decimal.setScale(SCALE, RoundingMode.HALF_EVEN).hashCode();
+  }
+
+  public static Odds ofDecimal(BigDecimal value) {
+    return new Odds(value.setScale(SCALE, RoundingMode.HALF_EVEN));
+  }
+
+  public static Odds ofDecimal(String value) {
+    return ofDecimal(new BigDecimal(value));
+  }
+}


## `test(odds): verify decimal odds invariants`

diff --git a/src/test/java/com/sportsbook/protocol/value/OddsTest.java b/src/test/java/com/sportsbook/protocol/value/OddsTest.java
new file mode 100644
index 0000000..199102f
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/value/OddsTest.java
@@ -0,0 +1,39 @@
+package com.sportsbook.protocol.value;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatIllegalArgumentException;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import java.math.BigDecimal;
+import org.junit.jupiter.api.Test;
+
+class OddsTest {
+
+  private final ObjectMapper mapper = new ObjectMapper();
+
+  @Test
+  void decimalFactoryNormalizesScale() {
+    assertThat(Odds.ofDecimal("1.85").decimal()).isEqualTo(new BigDecimal("1.8500"));
+    assertThat(Odds.ofDecimal(new BigDecimal("3")).decimal()).isEqualTo(new BigDecimal("3.0000"));
+  }
+
+  @Test
+  void decimalOddsCannotFallBelowOne() {
+    assertThatIllegalArgumentException().isThrownBy(() -> Odds.ofDecimal("0.99"));
+    assertThat(Odds.ofDecimal("1.00").decimal()).isEqualTo(new BigDecimal("1.0000"));
+  }
+
+  @Test
+  void equalityIgnoresBigDecimalScale() {
+    Odds compact = new Odds(new BigDecimal("1.85"));
+    Odds padded = new Odds(new BigDecimal("1.8500"));
+    assertThat(compact).isEqualTo(padded).hasSameHashCodeAs(padded);
+  }
+
+  @Test
+  void jsonRoundTripSurvivesScaleChanges() throws Exception {
+    Odds original = Odds.ofDecimal("1.85");
+    assertThat(mapper.readValue(mapper.writeValueAsString(original), Odds.class))
+        .isEqualTo(original);
+  }
+}


## `feat(odds): convert display formats`

diff --git a/src/main/java/com/sportsbook/protocol/value/Odds.java b/src/main/java/com/sportsbook/protocol/value/Odds.java
index bf392b8..7334fbf 100644
--- a/src/main/java/com/sportsbook/protocol/value/Odds.java
+++ b/src/main/java/com/sportsbook/protocol/value/Odds.java
@@ -1,6 +1,7 @@
 package com.sportsbook.protocol.value;
 
 import java.math.BigDecimal;
+import java.math.BigInteger;
 import java.math.RoundingMode;
 import java.util.Objects;
 
@@ -8,6 +9,9 @@ import java.util.Objects;
 public record Odds(BigDecimal decimal) {
 
   public static final int SCALE = 4;
+  private static final BigDecimal TWO = new BigDecimal("2.0000");
+  private static final BigDecimal HUNDRED = BigDecimal.valueOf(100);
+  private static final int MIN_AMERICAN_MAGNITUDE = 100;
 
   public Odds {
     Objects.requireNonNull(decimal, "decimal");
@@ -36,4 +40,37 @@ public record Odds(BigDecimal decimal) {
   public static Odds ofDecimal(String value) {
     return ofDecimal(new BigDecimal(value));
   }
+
+  public String toAmerican() {
+    BigDecimal net = decimal.subtract(BigDecimal.ONE);
+    if (decimal.compareTo(TWO) >= 0) {
+      int value = net.multiply(HUNDRED).setScale(0, RoundingMode.HALF_EVEN).intValueExact();
+      return "+" + value;
+    }
+    return "-" + HUNDRED.divide(net, 0, RoundingMode.HALF_EVEN).intValueExact();
+  }
+
+  public String toFractional() {
+    BigDecimal net = decimal.subtract(BigDecimal.ONE).setScale(SCALE, RoundingMode.HALF_EVEN);
+    BigInteger numerator = net.movePointRight(SCALE).toBigInteger();
+    BigInteger denominator = BigDecimal.ONE.movePointRight(SCALE).toBigInteger();
+    BigInteger divisor = numerator.gcd(denominator);
+    return numerator.divide(divisor) + "/" + denominator.divide(divisor);
+  }
+
+  public static Odds ofAmerican(int american) {
+    if (Math.abs(american) < MIN_AMERICAN_MAGNITUDE) {
+      throw new IllegalArgumentException(
+          "American odds magnitude must be at least 100: " + american);
+    }
+    BigDecimal decimal =
+        american > 0
+            ? BigDecimal.valueOf(american)
+                .divide(HUNDRED, SCALE, RoundingMode.HALF_EVEN)
+                .add(BigDecimal.ONE)
+            : HUNDRED
+                .divide(BigDecimal.valueOf(-american), SCALE, RoundingMode.HALF_EVEN)
+                .add(BigDecimal.ONE);
+    return ofDecimal(decimal);
+  }
 }


## `test(odds): verify American and fractional conversions`

diff --git a/src/test/java/com/sportsbook/protocol/value/OddsConversionTest.java b/src/test/java/com/sportsbook/protocol/value/OddsConversionTest.java
new file mode 100644
index 0000000..d782965
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/value/OddsConversionTest.java
@@ -0,0 +1,40 @@
+package com.sportsbook.protocol.value;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatIllegalArgumentException;
+
+import java.math.BigDecimal;
+import org.junit.jupiter.api.Test;
+
+class OddsConversionTest {
+
+  @Test
+  void favoritesConvertToNegativeAmericanOdds() {
+    assertThat(Odds.ofDecimal("1.5").toAmerican()).isEqualTo("-200");
+    assertThat(Odds.ofDecimal("1.85").toAmerican()).isEqualTo("-118");
+  }
+
+  @Test
+  void underdogsConvertToPositiveAmericanOdds() {
+    assertThat(Odds.ofDecimal("2.0").toAmerican()).isEqualTo("+100");
+    assertThat(Odds.ofDecimal("2.5").toAmerican()).isEqualTo("+150");
+  }
+
+  @Test
+  void fractionalOddsAreReduced() {
+    assertThat(Odds.ofDecimal("1.85").toFractional()).isEqualTo("17/20");
+    assertThat(Odds.ofDecimal("3.0").toFractional()).isEqualTo("2/1");
+  }
+
+  @Test
+  void AmericanOddsConvertToNormalizedDecimals() {
+    assertThat(Odds.ofAmerican(150).decimal()).isEqualTo(new BigDecimal("2.5000"));
+    assertThat(Odds.ofAmerican(-200).decimal()).isEqualTo(new BigDecimal("1.5000"));
+  }
+
+  @Test
+  void AmericanOddsRejectInvalidMagnitude() {
+    assertThatIllegalArgumentException().isThrownBy(() -> Odds.ofAmerican(99));
+    assertThatIllegalArgumentException().isThrownBy(() -> Odds.ofAmerican(-99));
+  }
+}
