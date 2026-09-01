# 위험 정책 기본값과 사용자 한도 오버라이드

## `feat(policy): define sliding limit dimensions`

diff --git a/src/main/java/com/sportsbook/risk/counter/LimitType.java b/src/main/java/com/sportsbook/risk/counter/LimitType.java
new file mode 100644
index 0000000..d804291
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/counter/LimitType.java
@@ -0,0 +1,42 @@
+package com.sportsbook.risk.counter;
+
+import java.time.Duration;
+
+/** Sliding policy dimensions and their Redis retention windows. */
+public enum LimitType {
+  STAKE_DAILY("stake-daily", Duration.ofDays(1), Measure.SUM),
+  STAKE_WEEKLY("stake-weekly", Duration.ofDays(7), Measure.SUM),
+  STAKE_MONTHLY("stake-monthly", Duration.ofDays(30), Measure.SUM),
+  SELECTIONS_PER_MINUTE("selections-per-minute", Duration.ofMinutes(1), Measure.COUNT);
+
+  private final String suffix;
+  private final Duration window;
+  private final Measure measure;
+
+  LimitType(String suffix, Duration window, Measure measure) {
+    this.suffix = suffix;
+    this.window = window;
+    this.measure = measure;
+  }
+
+  public String suffix() {
+    return suffix;
+  }
+
+  public Duration window() {
+    return window;
+  }
+
+  public Measure measure() {
+    return measure;
+  }
+
+  public boolean currencyScoped() {
+    return measure == Measure.SUM;
+  }
+
+  public enum Measure {
+    SUM,
+    COUNT
+  }
+}


## `test(policy): verify sliding limit dimensions`

diff --git a/src/test/java/com/sportsbook/risk/counter/LimitTypeTest.java b/src/test/java/com/sportsbook/risk/counter/LimitTypeTest.java
new file mode 100644
index 0000000..f6520fe
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/counter/LimitTypeTest.java
@@ -0,0 +1,24 @@
+package com.sportsbook.risk.counter;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.time.Duration;
+import org.junit.jupiter.api.Test;
+
+class LimitTypeTest {
+  @Test
+  void separatesCurrencyAmountsFromGlobalCounts() {
+    assertThat(LimitType.STAKE_DAILY.currencyScoped()).isTrue();
+    assertThat(LimitType.STAKE_WEEKLY.currencyScoped()).isTrue();
+    assertThat(LimitType.STAKE_MONTHLY.currencyScoped()).isTrue();
+    assertThat(LimitType.SELECTIONS_PER_MINUTE.currencyScoped()).isFalse();
+  }
+
+  @Test
+  void exposesStableSlidingWindows() {
+    assertThat(LimitType.STAKE_DAILY.window()).isEqualTo(Duration.ofDays(1));
+    assertThat(LimitType.STAKE_WEEKLY.window()).isEqualTo(Duration.ofDays(7));
+    assertThat(LimitType.STAKE_MONTHLY.window()).isEqualTo(Duration.ofDays(30));
+    assertThat(LimitType.SELECTIONS_PER_MINUTE.window()).isEqualTo(Duration.ofMinutes(1));
+  }
+}


## `feat(policy): configure currency risk limits`

diff --git a/src/main/java/com/sportsbook/risk/policy/RiskLimitProperties.java b/src/main/java/com/sportsbook/risk/policy/RiskLimitProperties.java
new file mode 100644
index 0000000..278ec57
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/policy/RiskLimitProperties.java
@@ -0,0 +1,71 @@
+package com.sportsbook.risk.policy;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.risk.counter.LimitType;
+import java.util.EnumMap;
+import java.util.Map;
+import java.util.Objects;
+import org.springframework.boot.context.properties.ConfigurationProperties;
+
+/** Default user limits used when no administrative override exists. */
+@ConfigurationProperties(prefix = "risk.limits")
+public record RiskLimitProperties(
+    Map<Currency, Long> stakeDaily,
+    Map<Currency, Long> stakeWeekly,
+    Map<Currency, Long> stakeMonthly,
+    Map<Currency, Long> singleBetMax,
+    int selectionsPerMinute) {
+
+  private static final Map<Currency, Long> DAILY = defaults(1_000_000L, 100_000L);
+  private static final Map<Currency, Long> WEEKLY = defaults(5_000_000L, 500_000L);
+  private static final Map<Currency, Long> MONTHLY = defaults(20_000_000L, 2_000_000L);
+  private static final Map<Currency, Long> SINGLE = defaults(500_000L, 50_000L);
+  private static final int DEFAULT_SELECTIONS_PER_MINUTE = 30;
+
+  public RiskLimitProperties {
+    stakeDaily = normalize(stakeDaily, DAILY, "stake-daily");
+    stakeWeekly = normalize(stakeWeekly, WEEKLY, "stake-weekly");
+    stakeMonthly = normalize(stakeMonthly, MONTHLY, "stake-monthly");
+    singleBetMax = normalize(singleBetMax, SINGLE, "single-bet-max");
+    if (selectionsPerMinute == 0) {
+      selectionsPerMinute = DEFAULT_SELECTIONS_PER_MINUTE;
+    }
+    SafeRedisNumber.requirePositive(selectionsPerMinute, "selections-per-minute");
+  }
+
+  public long singleBetMax(Currency currency) {
+    return required(singleBetMax, currency);
+  }
+
+  public long limit(LimitType type, Currency currency) {
+    return switch (type) {
+      case STAKE_DAILY -> required(stakeDaily, currency);
+      case STAKE_WEEKLY -> required(stakeWeekly, currency);
+      case STAKE_MONTHLY -> required(stakeMonthly, currency);
+      case SELECTIONS_PER_MINUTE -> selectionsPerMinute;
+    };
+  }
+
+  private static Map<Currency, Long> normalize(
+      Map<Currency, Long> source, Map<Currency, Long> fallback, String name) {
+    EnumMap<Currency, Long> result = new EnumMap<>(Currency.class);
+    result.putAll(source == null || source.isEmpty() ? fallback : source);
+    for (Currency currency : Currency.values()) {
+      Long value = result.get(currency);
+      if (value == null) {
+        value = fallback.get(currency);
+        result.put(currency, value);
+      }
+      SafeRedisNumber.requireNonNegative(value, name + "." + currency.name());
+    }
+    return Map.copyOf(result);
+  }
+
+  private static Map<Currency, Long> defaults(long krw, long usd) {
+    return Map.of(Currency.KRW, krw, Currency.USD, usd);
+  }
+
+  private static long required(Map<Currency, Long> values, Currency currency) {
+    return Objects.requireNonNull(values.get(currency), "missing limit for " + currency);
+  }
+}


## `test(policy): verify currency limit defaults`

diff --git a/src/test/java/com/sportsbook/risk/policy/RiskLimitPropertiesTest.java b/src/test/java/com/sportsbook/risk/policy/RiskLimitPropertiesTest.java
new file mode 100644
index 0000000..3a5c419
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/policy/RiskLimitPropertiesTest.java
@@ -0,0 +1,43 @@
+package com.sportsbook.risk.policy;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.risk.counter.LimitType;
+import java.util.Map;
+import org.junit.jupiter.api.Test;
+
+class RiskLimitPropertiesTest {
+  @Test
+  void suppliesBothSupportedCurrencyDefaults() {
+    RiskLimitProperties properties = properties(null, null, null, null, 0);
+
+    assertThat(properties.limit(LimitType.STAKE_DAILY, Currency.KRW)).isEqualTo(1_000_000L);
+    assertThat(properties.limit(LimitType.STAKE_DAILY, Currency.USD)).isEqualTo(100_000L);
+    assertThat(properties.limit(LimitType.STAKE_WEEKLY, Currency.KRW)).isEqualTo(5_000_000L);
+    assertThat(properties.limit(LimitType.STAKE_MONTHLY, Currency.USD)).isEqualTo(2_000_000L);
+    assertThat(properties.singleBetMax(Currency.KRW)).isEqualTo(500_000L);
+    assertThat(properties.selectionsPerMinute()).isEqualTo(30);
+  }
+
+  @Test
+  void fillsMissingCurrenciesWithoutAliasingInputMaps() {
+    Map<Currency, Long> daily = new java.util.EnumMap<>(Currency.class);
+    daily.put(Currency.KRW, 42L);
+
+    RiskLimitProperties properties = properties(daily, null, null, null, 10);
+    daily.put(Currency.KRW, 100L);
+
+    assertThat(properties.limit(LimitType.STAKE_DAILY, Currency.KRW)).isEqualTo(42L);
+    assertThat(properties.limit(LimitType.STAKE_DAILY, Currency.USD)).isEqualTo(100_000L);
+  }
+
+  private static RiskLimitProperties properties(
+      Map<Currency, Long> daily,
+      Map<Currency, Long> weekly,
+      Map<Currency, Long> monthly,
+      Map<Currency, Long> single,
+      int selections) {
+    return new RiskLimitProperties(daily, weekly, monthly, single, selections);
+  }
+}


## `test(policy): reject unsafe limit settings`

diff --git a/src/test/java/com/sportsbook/risk/policy/RiskLimitPropertiesTest.java b/src/test/java/com/sportsbook/risk/policy/RiskLimitPropertiesTest.java
index 3a5c419..cfd2390 100644
--- a/src/test/java/com/sportsbook/risk/policy/RiskLimitPropertiesTest.java
+++ b/src/test/java/com/sportsbook/risk/policy/RiskLimitPropertiesTest.java
@@ -1,6 +1,7 @@
 package com.sportsbook.risk.policy;
 
 import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
 
 import com.sportsbook.protocol.value.Currency;
 import com.sportsbook.risk.counter.LimitType;
@@ -32,6 +33,19 @@ class RiskLimitPropertiesTest {
     assertThat(properties.limit(LimitType.STAKE_DAILY, Currency.USD)).isEqualTo(100_000L);
   }
 
+  @Test
+  void rejectsInvalidPolicyAmounts() {
+    assertThatThrownBy(() -> properties(Map.of(Currency.KRW, -1L), null, null, null, 10))
+        .isInstanceOf(IllegalArgumentException.class);
+    assertThatThrownBy(
+            () ->
+                properties(
+                    Map.of(Currency.USD, SafeRedisNumber.MAX_VALUE + 1L), null, null, null, 10))
+        .isInstanceOf(IllegalArgumentException.class);
+    assertThatThrownBy(() -> properties(null, null, null, null, -1))
+        .isInstanceOf(IllegalArgumentException.class);
+  }
+
   private static RiskLimitProperties properties(
       Map<Currency, Long> daily,
       Map<Currency, Long> weekly,


## `feat(config): supply deployed risk defaults`

diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
new file mode 100644
index 0000000..618b68d
--- /dev/null
+++ b/src/main/resources/application.yml
@@ -0,0 +1,81 @@
+spring:
+  application:
+    name: risk-service
+  data:
+    redis:
+      host: ${REDIS_HOST:localhost}
+      port: ${REDIS_PORT:6379}
+      connect-timeout: 2s
+      timeout: 2s
+      lettuce:
+        pool:
+          max-active: 16
+          max-idle: 8
+          min-idle: 2
+  kafka:
+    bootstrap-servers: ${KAFKA_BOOTSTRAP:localhost:9092}
+    consumer:
+      group-id: risk.bet-placed-consumer
+      auto-offset-reset: earliest
+      enable-auto-commit: false
+    producer:
+      acks: all
+      properties:
+        max.block.ms: 1000
+    listener:
+      ack-mode: manual_immediate
+
+server:
+  port: ${SERVER_PORT:8083}
+
+risk:
+  limits:
+    stake-daily: { KRW: 1000000, USD: 100000 }
+    stake-weekly: { KRW: 5000000, USD: 500000 }
+    stake-monthly: { KRW: 20000000, USD: 2000000 }
+    single-bet-max: { KRW: 500000, USD: 50000 }
+    selections-per-minute: 30
+  patterns:
+    rapid-betting:
+      enabled: true
+      window: 60s
+      max-bets: 30
+      action: SUSPECT
+    sudden-stake:
+      enabled: true
+      multiplier: 10
+      lookback-bets: 10
+      action: SUSPECT
+    repeated-selection:
+      enabled: true
+      window: 24h
+      max-count: 5
+      action: REVIEW
+  reservations:
+    lease: 2m
+    retention: 32d
+  history:
+    idle-retention: 7d
+    max-stake-samples: 100
+  topics:
+    bet-placed: bet.placed.v1
+    bet-placed-dlt: bet.placed.v1.DLT
+    limit-violated: risk.limit.violated
+    pattern-suspected: risk.pattern.suspected
+
+management:
+  endpoints:
+    web:
+      exposure:
+        include: health,prometheus
+  endpoint:
+    health:
+      probes:
+        enabled: true
+      show-details: never
+  health:
+    redis:
+      enabled: true
+  tracing:
+    sampling:
+      probability: ${OTEL_TRACES_SAMPLER_ARG:0.1}


## `test(config): bind deployed risk defaults`

diff --git a/src/test/java/com/sportsbook/risk/DeployedRiskConfigurationTest.java b/src/test/java/com/sportsbook/risk/DeployedRiskConfigurationTest.java
new file mode 100644
index 0000000..1370f38
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/DeployedRiskConfigurationTest.java
@@ -0,0 +1,49 @@
+package com.sportsbook.risk;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.risk.counter.LimitType;
+import com.sportsbook.risk.pattern.RiskHistoryProperties;
+import com.sportsbook.risk.policy.PatternAction;
+import com.sportsbook.risk.policy.RiskLimitProperties;
+import com.sportsbook.risk.policy.RiskPatternProperties;
+import com.sportsbook.risk.reservation.RiskReservationProperties;
+import java.io.IOException;
+import java.time.Duration;
+import org.junit.jupiter.api.Test;
+import org.springframework.boot.context.properties.bind.Bindable;
+import org.springframework.boot.context.properties.bind.Binder;
+import org.springframework.boot.context.properties.source.ConfigurationPropertySources;
+import org.springframework.boot.env.YamlPropertySourceLoader;
+import org.springframework.core.env.PropertySource;
+import org.springframework.core.io.ClassPathResource;
+
+class DeployedRiskConfigurationTest {
+  @Test
+  void bindsEveryDeployedRiskPolicy() throws IOException {
+    PropertySource<?> source =
+        new YamlPropertySourceLoader()
+            .load("risk-defaults", new ClassPathResource("application.yml"))
+            .get(0);
+    Binder binder = new Binder(ConfigurationPropertySources.from(source));
+
+    RiskLimitProperties limits = bind(binder, "risk.limits", RiskLimitProperties.class);
+    RiskPatternProperties patterns = bind(binder, "risk.patterns", RiskPatternProperties.class);
+    RiskReservationProperties reservations =
+        bind(binder, "risk.reservations", RiskReservationProperties.class);
+    RiskHistoryProperties history = bind(binder, "risk.history", RiskHistoryProperties.class);
+
+    assertThat(limits.limit(LimitType.STAKE_DAILY, Currency.USD)).isEqualTo(100_000L);
+    assertThat(patterns.rapidBetting().action()).isEqualTo(PatternAction.SUSPECT);
+    assertThat(patterns.repeatedSelection().action()).isEqualTo(PatternAction.REVIEW);
+    assertThat(reservations.retention()).isEqualTo(Duration.ofDays(32));
+    assertThat(history.maxStakeSamples()).isEqualTo(100);
+  }
+
+  private static <T> T bind(Binder binder, String prefix, Class<T> type) {
+    return binder
+        .bind(prefix, Bindable.of(type))
+        .orElseThrow(() -> new AssertionError("Missing deployed configuration: " + prefix));
+  }
+}


## `feat(limits): encode user override fields`

diff --git a/src/main/java/com/sportsbook/risk/limit/LimitOverrideField.java b/src/main/java/com/sportsbook/risk/limit/LimitOverrideField.java
new file mode 100644
index 0000000..4cc307f
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/limit/LimitOverrideField.java
@@ -0,0 +1,29 @@
+package com.sportsbook.risk.limit;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.risk.counter.LimitType;
+import java.util.Objects;
+
+/** Canonical Redis hash field for an administrator-set user limit. */
+public record LimitOverrideField(LimitType type, Currency currency) {
+  public LimitOverrideField {
+    Objects.requireNonNull(type, "type");
+    if (type.currencyScoped()) {
+      Objects.requireNonNull(currency, "currency");
+    } else if (currency != null) {
+      throw new IllegalArgumentException("selection overrides must be currency-neutral");
+    }
+  }
+
+  public static LimitOverrideField monetary(LimitType type, Currency currency) {
+    return new LimitOverrideField(type, currency);
+  }
+
+  public static LimitOverrideField selections() {
+    return new LimitOverrideField(LimitType.SELECTIONS_PER_MINUTE, null);
+  }
+
+  public String redisField() {
+    return currency == null ? type.name() : type.name() + ":" + currency.name();
+  }
+}


## `test(limits): verify neutral selection overrides`

diff --git a/src/test/java/com/sportsbook/risk/limit/LimitOverrideFieldTest.java b/src/test/java/com/sportsbook/risk/limit/LimitOverrideFieldTest.java
new file mode 100644
index 0000000..b56854a
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/limit/LimitOverrideFieldTest.java
@@ -0,0 +1,28 @@
+package com.sportsbook.risk.limit;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.risk.counter.LimitType;
+import org.junit.jupiter.api.Test;
+
+class LimitOverrideFieldTest {
+  @Test
+  void qualifiesOnlyMonetaryFieldsByCurrency() {
+    assertThat(LimitOverrideField.monetary(LimitType.STAKE_DAILY, Currency.KRW).redisField())
+        .isEqualTo("STAKE_DAILY:KRW");
+    assertThat(LimitOverrideField.monetary(LimitType.STAKE_DAILY, Currency.USD).redisField())
+        .isEqualTo("STAKE_DAILY:USD");
+    assertThat(LimitOverrideField.selections().redisField()).isEqualTo("SELECTIONS_PER_MINUTE");
+  }
+
+  @Test
+  void rejectsMismatchedDimensions() {
+    assertThatThrownBy(
+            () -> LimitOverrideField.monetary(LimitType.SELECTIONS_PER_MINUTE, Currency.KRW))
+        .isInstanceOf(IllegalArgumentException.class);
+    assertThatThrownBy(() -> new LimitOverrideField(LimitType.STAKE_MONTHLY, null))
+        .isInstanceOf(NullPointerException.class);
+  }
+}


## `feat(limits): resolve user limit overrides`

diff --git a/src/main/java/com/sportsbook/risk/limit/LimitOverrideStore.java b/src/main/java/com/sportsbook/risk/limit/LimitOverrideStore.java
new file mode 100644
index 0000000..0e7ec41
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/limit/LimitOverrideStore.java
@@ -0,0 +1,13 @@
+package com.sportsbook.risk.limit;
+
+import com.sportsbook.protocol.value.UserId;
+import java.util.OptionalLong;
+
+/** Exact administrative overrides for one user's supported risk dimensions. */
+public interface LimitOverrideStore {
+  OptionalLong find(UserId userId, LimitOverrideField field);
+
+  void set(UserId userId, LimitOverrideField field, long value);
+
+  void clear(UserId userId, LimitOverrideField field);
+}
diff --git a/src/main/java/com/sportsbook/risk/limit/LimitResolver.java b/src/main/java/com/sportsbook/risk/limit/LimitResolver.java
new file mode 100644
index 0000000..b9cb521
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/limit/LimitResolver.java
@@ -0,0 +1,28 @@
+package com.sportsbook.risk.limit;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.counter.LimitType;
+import com.sportsbook.risk.policy.RiskLimitProperties;
+import java.util.Objects;
+
+/** Resolves a user override before the deployed default policy. */
+public final class LimitResolver {
+  private final RiskLimitProperties defaults;
+  private final LimitOverrideStore overrides;
+
+  public LimitResolver(RiskLimitProperties defaults, LimitOverrideStore overrides) {
+    this.defaults = Objects.requireNonNull(defaults, "defaults");
+    this.overrides = Objects.requireNonNull(overrides, "overrides");
+  }
+
+  public long resolve(UserId userId, LimitType type, Currency currency) {
+    Objects.requireNonNull(userId, "userId");
+    Objects.requireNonNull(type, "type");
+    LimitOverrideField field =
+        type.currencyScoped()
+            ? LimitOverrideField.monetary(type, Objects.requireNonNull(currency, "currency"))
+            : LimitOverrideField.selections();
+    return overrides.find(userId, field).orElseGet(() -> defaults.limit(type, currency));
+  }
+}


## `test(limits): verify override precedence`

diff --git a/src/test/java/com/sportsbook/risk/limit/LimitResolverTest.java b/src/test/java/com/sportsbook/risk/limit/LimitResolverTest.java
new file mode 100644
index 0000000..bd62566
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/limit/LimitResolverTest.java
@@ -0,0 +1,45 @@
+package com.sportsbook.risk.limit;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.counter.LimitType;
+import com.sportsbook.risk.policy.RiskLimitProperties;
+import java.util.OptionalLong;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class LimitResolverTest {
+  private static final UserId USER =
+      UserId.of(UUID.fromString("00000000-0000-0000-0000-000000000001"));
+
+  @Test
+  void prefersAUserOverrideAndFallsBackToPolicy() {
+    LimitOverrideStore store = mock(LimitOverrideStore.class);
+    LimitOverrideField krwDaily = LimitOverrideField.monetary(LimitType.STAKE_DAILY, Currency.KRW);
+    when(store.find(USER, krwDaily)).thenReturn(OptionalLong.of(1234));
+    LimitResolver resolver = new LimitResolver(defaults(), store);
+
+    assertThat(resolver.resolve(USER, LimitType.STAKE_DAILY, Currency.KRW)).isEqualTo(1234);
+    assertThat(resolver.resolve(USER, LimitType.STAKE_WEEKLY, Currency.USD)).isEqualTo(500_000);
+  }
+
+  @Test
+  void usesOneCurrencyNeutralSelectionOverride() {
+    LimitOverrideStore store = mock(LimitOverrideStore.class);
+    when(store.find(USER, LimitOverrideField.selections())).thenReturn(OptionalLong.of(12));
+    LimitResolver resolver = new LimitResolver(defaults(), store);
+
+    assertThat(resolver.resolve(USER, LimitType.SELECTIONS_PER_MINUTE, Currency.KRW)).isEqualTo(12);
+    assertThat(resolver.resolve(USER, LimitType.SELECTIONS_PER_MINUTE, Currency.USD)).isEqualTo(12);
+    verify(store, org.mockito.Mockito.times(2)).find(USER, LimitOverrideField.selections());
+  }
+
+  private static RiskLimitProperties defaults() {
+    return new RiskLimitProperties(null, null, null, null, 0);
+  }
+}


