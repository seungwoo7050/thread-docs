# 실패 폐쇄형 베팅 접수 검증

## `feat(policy): bind guarded betting limits`

diff --git a/src/main/java/com/sportsbook/betting/policy/BettingPolicyProperties.java b/src/main/java/com/sportsbook/betting/policy/BettingPolicyProperties.java
new file mode 100644
index 0000000..b1d3531
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/policy/BettingPolicyProperties.java
@@ -0,0 +1,55 @@
+package com.sportsbook.betting.policy;
+
+import com.sportsbook.protocol.value.Currency;
+import java.math.BigDecimal;
+import java.util.EnumSet;
+import java.util.Map;
+import org.springframework.boot.context.properties.ConfigurationProperties;
+
+@ConfigurationProperties(prefix = "betting.policy")
+public record BettingPolicyProperties(
+    int maxSelections,
+    BigDecimal maxTotalOdds,
+    Map<Currency, Long> minStake,
+    Map<Currency, Long> maxStake,
+    BigDecimal slippageTolerancePercent) {
+
+  private static final int DEFAULT_MAX_SELECTIONS = 15;
+  private static final BigDecimal DEFAULT_MAX_TOTAL_ODDS = new BigDecimal("10000");
+  private static final BigDecimal DEFAULT_SLIPPAGE_PERCENT = new BigDecimal("3");
+
+  public BettingPolicyProperties {
+    if (maxSelections < 0 || maxSelections > DEFAULT_MAX_SELECTIONS) {
+      throw new IllegalArgumentException("maxSelections must be in 1..15 when configured");
+    }
+    maxSelections = maxSelections > 0 ? maxSelections : DEFAULT_MAX_SELECTIONS;
+    maxTotalOdds =
+        maxTotalOdds == null ? DEFAULT_MAX_TOTAL_ODDS : maxTotalOdds.stripTrailingZeros();
+    slippageTolerancePercent =
+        slippageTolerancePercent == null
+            ? DEFAULT_SLIPPAGE_PERCENT
+            : slippageTolerancePercent.stripTrailingZeros();
+    minStake =
+        minStake == null || minStake.isEmpty()
+            ? Map.of(Currency.KRW, 1_000L, Currency.USD, 100L)
+            : Map.copyOf(minStake);
+    maxStake =
+        maxStake == null || maxStake.isEmpty()
+            ? Map.of(Currency.KRW, 1_000_000L, Currency.USD, 1_000L)
+            : Map.copyOf(maxStake);
+    if (maxTotalOdds.signum() <= 0 || slippageTolerancePercent.signum() <= 0) {
+      throw new IllegalArgumentException("Odds and slippage policy values must be positive");
+    }
+    if (!minStake.keySet().equals(EnumSet.allOf(Currency.class))
+        || !maxStake.keySet().equals(EnumSet.allOf(Currency.class))) {
+      throw new IllegalArgumentException("Stake bounds must cover every currency");
+    }
+    for (Currency currency : Currency.values()) {
+      long minimum = minStake.get(currency);
+      long maximum = maxStake.get(currency);
+      if (minimum <= 0 || maximum <= 0 || minimum > maximum) {
+        throw new IllegalArgumentException("Stake bounds must be positive with minimum <= maximum");
+      }
+    }
+  }
+}


## `test(policy): verify guarded defaults`

diff --git a/src/test/java/com/sportsbook/betting/policy/BettingPolicyPropertiesTest.java b/src/test/java/com/sportsbook/betting/policy/BettingPolicyPropertiesTest.java
new file mode 100644
index 0000000..ccd47d1
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/policy/BettingPolicyPropertiesTest.java
@@ -0,0 +1,50 @@
+package com.sportsbook.betting.policy;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.protocol.value.Currency;
+import java.math.BigDecimal;
+import java.util.Map;
+import org.junit.jupiter.api.Test;
+
+class BettingPolicyPropertiesTest {
+
+  @Test
+  void suppliesSafeDefaults() {
+    BettingPolicyProperties policy = new BettingPolicyProperties(0, null, null, null, null);
+
+    assertThat(policy.maxSelections()).isEqualTo(15);
+    assertThat(policy.maxTotalOdds()).isEqualByComparingTo("10000");
+    assertThat(policy.minStake()).containsEntry(Currency.KRW, 1_000L);
+    assertThat(policy.maxStake()).containsEntry(Currency.USD, 1_000L);
+    assertThat(policy.slippageTolerancePercent()).isEqualByComparingTo("3");
+  }
+
+  @Test
+  void rejectsUnsafeConfiguredLimits() {
+    assertThatThrownBy(() -> new BettingPolicyProperties(16, null, null, null, null))
+        .hasMessageContaining("1..15");
+    assertThatThrownBy(
+            () -> new BettingPolicyProperties(15, BigDecimal.ZERO, null, null, new BigDecimal("3")))
+        .hasMessageContaining("positive");
+    assertThatThrownBy(
+            () ->
+                new BettingPolicyProperties(
+                    15,
+                    new BigDecimal("100"),
+                    Map.of(Currency.KRW, 2_000L),
+                    Map.of(Currency.KRW, 1_000L, Currency.USD, 100L),
+                    new BigDecimal("3")))
+        .hasMessageContaining("every currency");
+    assertThatThrownBy(
+            () ->
+                new BettingPolicyProperties(
+                    15,
+                    new BigDecimal("100"),
+                    Map.of(Currency.KRW, 2_000L, Currency.USD, 100L),
+                    Map.of(Currency.KRW, 1_000L, Currency.USD, 1_000L),
+                    new BigDecimal("3")))
+        .hasMessageContaining("minimum <= maximum");
+  }
+}


## `feat(error): classify validation verdicts`

diff --git a/src/main/java/com/sportsbook/betting/error/BetPlacementException.java b/src/main/java/com/sportsbook/betting/error/BetPlacementException.java
new file mode 100644
index 0000000..baa450c
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/error/BetPlacementException.java
@@ -0,0 +1,18 @@
+package com.sportsbook.betting.error;
+
+import com.sportsbook.protocol.error.ErrorCode;
+import java.util.Objects;
+
+public abstract class BetPlacementException extends RuntimeException {
+
+  private final transient ErrorCode errorCode;
+
+  protected BetPlacementException(ErrorCode errorCode, String message) {
+    super(message);
+    this.errorCode = Objects.requireNonNull(errorCode, "errorCode");
+  }
+
+  public ErrorCode errorCode() {
+    return errorCode;
+  }
+}
diff --git a/src/main/java/com/sportsbook/betting/error/ValidationFailedException.java b/src/main/java/com/sportsbook/betting/error/ValidationFailedException.java
new file mode 100644
index 0000000..77d4606
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/error/ValidationFailedException.java
@@ -0,0 +1,10 @@
+package com.sportsbook.betting.error;
+
+import com.sportsbook.protocol.error.ErrorCode;
+
+public class ValidationFailedException extends BetPlacementException {
+
+  public ValidationFailedException(String message) {
+    super(ErrorCode.VALIDATION_FAILED, message);
+  }
+}


## `feat(validation): enforce slip structure policy`

diff --git a/src/main/java/com/sportsbook/betting/validation/BetSlipValidator.java b/src/main/java/com/sportsbook/betting/validation/BetSlipValidator.java
new file mode 100644
index 0000000..13d1648
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/validation/BetSlipValidator.java
@@ -0,0 +1,56 @@
+package com.sportsbook.betting.validation;
+
+import com.sportsbook.betting.domain.BetLeg;
+import com.sportsbook.betting.error.ValidationFailedException;
+import com.sportsbook.betting.policy.BettingPolicyProperties;
+import com.sportsbook.protocol.domain.BetSlipType;
+import java.util.HashSet;
+import java.util.List;
+import java.util.Set;
+import java.util.UUID;
+import org.springframework.stereotype.Component;
+
+@Component
+public class BetSlipValidator {
+
+  private final BettingPolicyProperties policy;
+
+  public BetSlipValidator(BettingPolicyProperties policy) {
+    this.policy = policy;
+  }
+
+  public void validate(BetSlipType type, List<BetLeg> legs) {
+    if (legs.isEmpty()) {
+      throw new ValidationFailedException("Slip must contain at least one selection");
+    }
+    if (legs.size() > policy.maxSelections()) {
+      throw new ValidationFailedException("Maximum selections exceeded");
+    }
+    requireDistinctSelections(legs);
+    if (!(type instanceof BetSlipType.Single)) {
+      requireDistinctMarketsAndEvents(legs);
+    }
+  }
+
+  private static void requireDistinctMarketsAndEvents(List<BetLeg> legs) {
+    Set<UUID> markets = new HashSet<>();
+    Set<UUID> events = new HashSet<>();
+    for (BetLeg leg : legs) {
+      if (!markets.add(leg.marketId())) {
+        throw new ValidationFailedException("Same market is not allowed in a multi");
+      }
+      if (!events.add(leg.eventId())) {
+        throw new ValidationFailedException("Same event is not allowed in a multi");
+      }
+    }
+  }
+
+  private static void requireDistinctSelections(List<BetLeg> legs) {
+    Set<UUID> selections = new HashSet<>();
+    for (BetLeg leg : legs) {
+      if (!selections.add(leg.selectionId())) {
+        throw new ValidationFailedException("Duplicate selection is not allowed");
+      }
+    }
+  }
+}


## `test(validation): reject correlated slip structure`

diff --git a/src/test/java/com/sportsbook/betting/validation/BetSlipValidatorTest.java b/src/test/java/com/sportsbook/betting/validation/BetSlipValidatorTest.java
new file mode 100644
index 0000000..7eb06f1
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/validation/BetSlipValidatorTest.java
@@ -0,0 +1,46 @@
+package com.sportsbook.betting.validation;
+
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.betting.domain.BetLeg;
+import com.sportsbook.betting.error.ValidationFailedException;
+import com.sportsbook.betting.policy.BettingPolicyProperties;
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.value.Odds;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class BetSlipValidatorTest {
+
+  private final BetSlipValidator validator =
+      new BetSlipValidator(new BettingPolicyProperties(2, null, null, null, null));
+
+  @Test
+  void rejectsRepeatedMarketBeforeExternalCalls() {
+    UUID event = UUID.randomUUID();
+    UUID market = UUID.randomUUID();
+    List<BetLeg> legs =
+        List.of(leg(event, market), leg(event, market));
+
+    assertThatThrownBy(() -> validator.validate(new BetSlipType.Multiple(), legs))
+        .isInstanceOf(ValidationFailedException.class)
+        .hasMessageContaining("Same market");
+  }
+
+  @Test
+  void rejectsDuplicateSelectionIdentity() {
+    UUID selection = UUID.randomUUID();
+    List<BetLeg> legs =
+        List.of(
+            BetLeg.create(UUID.randomUUID(), UUID.randomUUID(), selection, Odds.ofDecimal("2.0")),
+            BetLeg.create(UUID.randomUUID(), UUID.randomUUID(), selection, Odds.ofDecimal("3.0")));
+
+    assertThatThrownBy(() -> validator.validate(new BetSlipType.Multiple(), legs))
+        .isInstanceOf(ValidationFailedException.class)
+        .hasMessageContaining("Duplicate selection");
+  }
+  private BetLeg leg(UUID event, UUID market) {
+    return BetLeg.create(event, market, UUID.randomUUID(), Odds.ofDecimal("2.0"));
+  }
+}


## `feat(validation): enforce stake and odds limits`

diff --git a/src/main/java/com/sportsbook/betting/validation/BetSlipValidator.java b/src/main/java/com/sportsbook/betting/validation/BetSlipValidator.java
index 13d1648..080ba96 100644
--- a/src/main/java/com/sportsbook/betting/validation/BetSlipValidator.java
+++ b/src/main/java/com/sportsbook/betting/validation/BetSlipValidator.java
@@ -4,6 +4,8 @@ import com.sportsbook.betting.domain.BetLeg;
 import com.sportsbook.betting.error.ValidationFailedException;
 import com.sportsbook.betting.policy.BettingPolicyProperties;
 import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.value.Money;
+import java.math.BigDecimal;
 import java.util.HashSet;
 import java.util.List;
 import java.util.Set;
@@ -30,6 +32,28 @@ public class BetSlipValidator {
     if (!(type instanceof BetSlipType.Single)) {
       requireDistinctMarketsAndEvents(legs);
     }
+    requireTotalOdds(legs);
+  }
+
+  public void validateStake(Money stake) {
+    Long minimum = policy.minStake().get(stake.currency());
+    Long maximum = policy.maxStake().get(stake.currency());
+    if (minimum == null || maximum == null) {
+      throw new ValidationFailedException("Unsupported stake currency");
+    }
+    if (stake.amount() < minimum || stake.amount() > maximum) {
+      throw new ValidationFailedException("Stake is outside configured bounds");
+    }
+  }
+
+  private void requireTotalOdds(List<BetLeg> legs) {
+    BigDecimal product = BigDecimal.ONE;
+    for (BetLeg leg : legs) {
+      product = product.multiply(leg.oddsAtSubmission().decimal());
+    }
+    if (product.compareTo(policy.maxTotalOdds()) > 0) {
+      throw new ValidationFailedException("Maximum total odds exceeded");
+    }
   }
 
   private static void requireDistinctMarketsAndEvents(List<BetLeg> legs) {


## `test(validation): reject stake and odds policy breaches`

diff --git a/src/test/java/com/sportsbook/betting/validation/BetSlipValidatorTest.java b/src/test/java/com/sportsbook/betting/validation/BetSlipValidatorTest.java
index 7eb06f1..ed30382 100644
--- a/src/test/java/com/sportsbook/betting/validation/BetSlipValidatorTest.java
+++ b/src/test/java/com/sportsbook/betting/validation/BetSlipValidatorTest.java
@@ -6,6 +6,7 @@ import com.sportsbook.betting.domain.BetLeg;
 import com.sportsbook.betting.error.ValidationFailedException;
 import com.sportsbook.betting.policy.BettingPolicyProperties;
 import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.value.Money;
 import com.sportsbook.protocol.value.Odds;
 import java.util.List;
 import java.util.UUID;
@@ -20,8 +21,7 @@ class BetSlipValidatorTest {
   void rejectsRepeatedMarketBeforeExternalCalls() {
     UUID event = UUID.randomUUID();
     UUID market = UUID.randomUUID();
-    List<BetLeg> legs =
-        List.of(leg(event, market), leg(event, market));
+    List<BetLeg> legs = List.of(leg(event, market), leg(event, market));
 
     assertThatThrownBy(() -> validator.validate(new BetSlipType.Multiple(), legs))
         .isInstanceOf(ValidationFailedException.class)
@@ -40,6 +40,34 @@ class BetSlipValidatorTest {
         .isInstanceOf(ValidationFailedException.class)
         .hasMessageContaining("Duplicate selection");
   }
+
+  @Test
+  void rejectsStakeOutsideCurrencyBounds() {
+    assertThatThrownBy(() -> validator.validateStake(Money.krw(999)))
+        .isInstanceOf(ValidationFailedException.class)
+        .hasMessageContaining("bounds");
+  }
+
+  @Test
+  void rejectsOddsProductAboveConfiguredMaximum() {
+    BetSlipValidator strict =
+        new BetSlipValidator(
+            new BettingPolicyProperties(15, new java.math.BigDecimal("3"), null, null, null));
+
+    assertThatThrownBy(
+            () ->
+                strict.validate(
+                    new BetSlipType.Single(),
+                    List.of(
+                        BetLeg.create(
+                            UUID.randomUUID(),
+                            UUID.randomUUID(),
+                            UUID.randomUUID(),
+                            Odds.ofDecimal("4.0")))))
+        .isInstanceOf(ValidationFailedException.class)
+        .hasMessageContaining("odds");
+  }
+
   private BetLeg leg(UUID event, UUID market) {
     return BetLeg.create(event, market, UUID.randomUUID(), Odds.ofDecimal("2.0"));
   }


## `feat(error): classify odds verdicts`

diff --git a/src/main/java/com/sportsbook/betting/error/MarketClosedException.java b/src/main/java/com/sportsbook/betting/error/MarketClosedException.java
new file mode 100644
index 0000000..25c6070
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/error/MarketClosedException.java
@@ -0,0 +1,10 @@
+package com.sportsbook.betting.error;
+
+import com.sportsbook.protocol.error.ErrorCode;
+
+public class MarketClosedException extends BetPlacementException {
+
+  public MarketClosedException(String message) {
+    super(ErrorCode.EVENT_CLOSED, message);
+  }
+}
diff --git a/src/main/java/com/sportsbook/betting/error/OddsDriftException.java b/src/main/java/com/sportsbook/betting/error/OddsDriftException.java
new file mode 100644
index 0000000..6459861
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/error/OddsDriftException.java
@@ -0,0 +1,10 @@
+package com.sportsbook.betting.error;
+
+import com.sportsbook.protocol.error.ErrorCode;
+
+public class OddsDriftException extends BetPlacementException {
+
+  public OddsDriftException(String message) {
+    super(ErrorCode.ODDS_DRIFT, message);
+  }
+}


## `feat(odds): read effective market snapshots`

diff --git a/src/main/java/com/sportsbook/betting/validation/OddsSnapshotReader.java b/src/main/java/com/sportsbook/betting/validation/OddsSnapshotReader.java
new file mode 100644
index 0000000..b564512
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/validation/OddsSnapshotReader.java
@@ -0,0 +1,36 @@
+package com.sportsbook.betting.validation;
+
+import com.sportsbook.betting.domain.BetLeg;
+import com.sportsbook.betting.error.MarketClosedException;
+import java.math.BigDecimal;
+import org.springframework.data.redis.core.StringRedisTemplate;
+import org.springframework.stereotype.Component;
+
+@Component
+public class OddsSnapshotReader {
+
+  private final StringRedisTemplate redis;
+
+  public OddsSnapshotReader(StringRedisTemplate redis) {
+    this.redis = redis;
+  }
+
+  public BigDecimal currentOdds(BetLeg leg) {
+    String marketKey = "market:" + leg.eventId() + ":" + leg.marketId();
+    String status = redis.opsForValue().get(marketKey);
+    if (!"OPEN".equals(status)) {
+      throw new MarketClosedException("Market is not effectively OPEN");
+    }
+
+    String oddsKey = "odds:" + leg.eventId() + ":" + leg.marketId() + ":" + leg.selectionId();
+    String value = redis.opsForValue().get(oddsKey);
+    if (value == null) {
+      throw new MarketClosedException("Selection is no longer priced");
+    }
+    try {
+      return new BigDecimal(value);
+    } catch (NumberFormatException exception) {
+      throw new MarketClosedException("Selection price is invalid");
+    }
+  }
+}


## `test(odds): verify canonical snapshot keys`

diff --git a/src/test/java/com/sportsbook/betting/validation/OddsSnapshotReaderTest.java b/src/test/java/com/sportsbook/betting/validation/OddsSnapshotReaderTest.java
new file mode 100644
index 0000000..b4543d5
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/validation/OddsSnapshotReaderTest.java
@@ -0,0 +1,32 @@
+package com.sportsbook.betting.validation;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.betting.domain.BetLeg;
+import com.sportsbook.protocol.value.Odds;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.data.redis.core.StringRedisTemplate;
+import org.springframework.data.redis.core.ValueOperations;
+
+class OddsSnapshotReaderTest {
+
+  @Test
+  void readsCanonicalEffectiveStatusAndPriceKeys() {
+    UUID event = UUID.randomUUID();
+    UUID market = UUID.randomUUID();
+    UUID selection = UUID.randomUUID();
+    StringRedisTemplate redis = mock(StringRedisTemplate.class);
+    @SuppressWarnings("unchecked")
+    ValueOperations<String, String> values = mock(ValueOperations.class);
+    when(redis.opsForValue()).thenReturn(values);
+    when(values.get("market:" + event + ":" + market)).thenReturn("OPEN");
+    when(values.get("odds:" + event + ":" + market + ":" + selection)).thenReturn("2.25");
+
+    BetLeg leg = BetLeg.create(event, market, selection, Odds.ofDecimal("2.0"));
+
+    assertThat(new OddsSnapshotReader(redis).currentOdds(leg)).isEqualByComparingTo("2.25");
+  }
+}


## `feat(odds): enforce user-protective slippage`

diff --git a/src/main/java/com/sportsbook/betting/validation/OddsSlippageChecker.java b/src/main/java/com/sportsbook/betting/validation/OddsSlippageChecker.java
new file mode 100644
index 0000000..6303b37
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/validation/OddsSlippageChecker.java
@@ -0,0 +1,33 @@
+package com.sportsbook.betting.validation;
+
+import com.sportsbook.betting.domain.BetLeg;
+import com.sportsbook.betting.error.OddsDriftException;
+import com.sportsbook.betting.policy.BettingPolicyProperties;
+import java.math.BigDecimal;
+import java.util.List;
+import org.springframework.stereotype.Component;
+
+@Component
+public class OddsSlippageChecker {
+
+  private static final BigDecimal HUNDRED = BigDecimal.valueOf(100);
+
+  private final OddsSnapshotReader snapshots;
+  private final BettingPolicyProperties policy;
+
+  public OddsSlippageChecker(OddsSnapshotReader snapshots, BettingPolicyProperties policy) {
+    this.snapshots = snapshots;
+    this.policy = policy;
+  }
+
+  public void check(List<BetLeg> legs) {
+    BigDecimal tolerance = HUNDRED.subtract(policy.slippageTolerancePercent());
+    for (BetLeg leg : legs) {
+      BigDecimal current = snapshots.currentOdds(leg);
+      BigDecimal quoted = leg.oddsAtSubmission().decimal();
+      if (current.multiply(HUNDRED).compareTo(quoted.multiply(tolerance)) < 0) {
+        throw new OddsDriftException("Odds drifted beyond configured tolerance");
+      }
+    }
+  }
+}


