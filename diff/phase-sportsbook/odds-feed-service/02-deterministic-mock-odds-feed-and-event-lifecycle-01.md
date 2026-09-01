# 결정적 모의 배당 피드와 경기 수명 주기

## `feat(mock): bind deterministic provider settings`

diff --git a/src/main/java/com/sportsbook/oddsfeed/config/ApplicationConfig.java b/src/main/java/com/sportsbook/oddsfeed/config/ApplicationConfig.java
index 15b7d4d..5c26ca9 100644
--- a/src/main/java/com/sportsbook/oddsfeed/config/ApplicationConfig.java
+++ b/src/main/java/com/sportsbook/oddsfeed/config/ApplicationConfig.java
@@ -1,12 +1,14 @@
 package com.sportsbook.oddsfeed.config;
 
 import java.time.Clock;
+import org.springframework.boot.context.properties.EnableConfigurationProperties;
 import org.springframework.context.annotation.Bean;
 import org.springframework.context.annotation.Configuration;
 import org.springframework.scheduling.annotation.EnableScheduling;
 
 @Configuration
 @EnableScheduling
+@EnableConfigurationProperties(MockProperties.class)
 public class ApplicationConfig {
 
   @Bean
diff --git a/src/main/java/com/sportsbook/oddsfeed/config/MockProperties.java b/src/main/java/com/sportsbook/oddsfeed/config/MockProperties.java
new file mode 100644
index 0000000..0f3b970
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/config/MockProperties.java
@@ -0,0 +1,16 @@
+package com.sportsbook.oddsfeed.config;
+
+import org.springframework.boot.context.properties.ConfigurationProperties;
+
+@ConfigurationProperties(prefix = "oddsfeed.mock")
+public record MockProperties(
+    double minutesPerSecond,
+    Scenarios scenarios,
+    double baseHomeWinProbability,
+    double baseDrawProbability,
+    double baseAwayWinProbability,
+    long randomSeed,
+    int tickIntervalMs) {
+
+  public record Scenarios(boolean autoRotate, int rotationIntervalSeconds) {}
+}


## `test(mock): verify provider settings`

diff --git a/src/test/java/com/sportsbook/oddsfeed/config/MockPropertiesTest.java b/src/test/java/com/sportsbook/oddsfeed/config/MockPropertiesTest.java
new file mode 100644
index 0000000..eb47f8e
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/config/MockPropertiesTest.java
@@ -0,0 +1,36 @@
+package com.sportsbook.oddsfeed.config;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.util.Map;
+import org.junit.jupiter.api.Test;
+import org.springframework.boot.context.properties.bind.Bindable;
+import org.springframework.boot.context.properties.bind.Binder;
+import org.springframework.boot.context.properties.source.MapConfigurationPropertySource;
+
+class MockPropertiesTest {
+
+  @Test
+  void bindsDeterministicSimulationSettings() {
+    Map<String, String> values =
+        Map.of(
+            "oddsfeed.mock.minutes-per-second", "2",
+            "oddsfeed.mock.scenarios.auto-rotate", "true",
+            "oddsfeed.mock.scenarios.rotation-interval-seconds", "30",
+            "oddsfeed.mock.base-home-win-probability", "0.45",
+            "oddsfeed.mock.base-draw-probability", "0.25",
+            "oddsfeed.mock.base-away-win-probability", "0.30",
+            "oddsfeed.mock.random-seed", "424242",
+            "oddsfeed.mock.tick-interval-ms", "500");
+
+    MockProperties properties =
+        new Binder(new MapConfigurationPropertySource(values))
+            .bind("oddsfeed.mock", Bindable.of(MockProperties.class))
+            .orElseThrow(IllegalStateException::new);
+
+    assertThat(properties.minutesPerSecond()).isEqualTo(2);
+    assertThat(properties.randomSeed()).isEqualTo(424242);
+    assertThat(properties.scenarios().autoRotate()).isTrue();
+    assertThat(properties.scenarios().rotationIntervalSeconds()).isEqualTo(30);
+  }
+}


## `feat(mock): define deterministic profile defaults`

diff --git a/src/main/resources/application-mock.yml b/src/main/resources/application-mock.yml
new file mode 100644
index 0000000..7f7bf1d
--- /dev/null
+++ b/src/main/resources/application-mock.yml
@@ -0,0 +1,13 @@
+oddsfeed:
+  provider:
+    mode: mock
+  mock:
+    minutes-per-second: 1
+    random-seed: ${ODDSFEED_MOCK_RANDOM_SEED:424242}
+    tick-interval-ms: ${ODDSFEED_MOCK_TICK_INTERVAL_MS:500}
+    scenarios:
+      auto-rotate: true
+      rotation-interval-seconds: 60
+    base-home-win-probability: 0.45
+    base-draw-probability: 0.25
+    base-away-win-probability: 0.30


## `feat(mock): simulate bounded decimal odds`

diff --git a/src/main/java/com/sportsbook/oddsfeed/provider/mock/OddsSimulator.java b/src/main/java/com/sportsbook/oddsfeed/provider/mock/OddsSimulator.java
new file mode 100644
index 0000000..54749d8
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/provider/mock/OddsSimulator.java
@@ -0,0 +1,30 @@
+package com.sportsbook.oddsfeed.provider.mock;
+
+import com.sportsbook.protocol.value.Odds;
+import java.math.BigDecimal;
+import java.math.RoundingMode;
+import java.util.Random;
+
+public final class OddsSimulator {
+
+  static final double NOISE_STDDEV = 0.02;
+  static final double MEAN_REVERSION = 0.10;
+  static final double MIN_ODDS = 1.01;
+  static final double MAX_ODDS = 100.0;
+
+  private OddsSimulator() {}
+
+  public static Odds initialOdds(double impliedProbability) {
+    double fair = 1.0 / impliedProbability;
+    return Odds.ofDecimal(BigDecimal.valueOf(fair).setScale(Odds.SCALE, RoundingMode.HALF_EVEN));
+  }
+
+  public static Odds nextOdds(Odds current, double impliedProbability, Random random) {
+    double fair = 1.0 / impliedProbability;
+    double currentValue = current.decimal().doubleValue();
+    double noisy = currentValue * (1.0 + random.nextGaussian() * NOISE_STDDEV);
+    double next = noisy * (1.0 - MEAN_REVERSION) + fair * MEAN_REVERSION;
+    double clamped = Math.max(MIN_ODDS, Math.min(MAX_ODDS, next));
+    return Odds.ofDecimal(BigDecimal.valueOf(clamped).setScale(Odds.SCALE, RoundingMode.HALF_EVEN));
+  }
+}


## `test(mock): verify bounded odds simulation`

diff --git a/src/test/java/com/sportsbook/oddsfeed/provider/mock/OddsSimulatorTest.java b/src/test/java/com/sportsbook/oddsfeed/provider/mock/OddsSimulatorTest.java
new file mode 100644
index 0000000..52c6966
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/provider/mock/OddsSimulatorTest.java
@@ -0,0 +1,54 @@
+package com.sportsbook.oddsfeed.provider.mock;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.value.Odds;
+import java.util.Random;
+import org.junit.jupiter.api.Test;
+
+class OddsSimulatorTest {
+
+  @Test
+  void initialOddsAreFairValueForImpliedProbability() {
+    Odds homeWin = OddsSimulator.initialOdds(0.5);
+    assertThat(homeWin.decimal().doubleValue())
+        .isCloseTo(2.0, org.assertj.core.data.Offset.offset(1e-4));
+  }
+
+  @Test
+  void nextOddsStaysWithinClampedRange() {
+    Random rng = new Random(42L);
+    Odds current = OddsSimulator.initialOdds(0.45);
+    for (int i = 0; i < 1000; i++) {
+      current = OddsSimulator.nextOdds(current, 0.45, rng);
+      assertThat(current.decimal().doubleValue())
+          .isBetween(OddsSimulator.MIN_ODDS, OddsSimulator.MAX_ODDS);
+    }
+  }
+
+  @Test
+  void nextOddsRevertsTowardFairOverTime() {
+    Random rng = new Random(7L);
+    double implied = 0.40;
+    double fair = 1.0 / implied;
+    Odds current = Odds.ofDecimal("5.0000");
+    for (int i = 0; i < 5000; i++) {
+      current = OddsSimulator.nextOdds(current, implied, rng);
+    }
+    assertThat(current.decimal().doubleValue())
+        .isCloseTo(fair, org.assertj.core.data.Offset.offset(0.5));
+  }
+
+  @Test
+  void deterministicGivenSameSeed() {
+    Random rng1 = new Random(123L);
+    Random rng2 = new Random(123L);
+    Odds o1 = Odds.ofDecimal("2.0000");
+    Odds o2 = Odds.ofDecimal("2.0000");
+    for (int i = 0; i < 50; i++) {
+      o1 = OddsSimulator.nextOdds(o1, 0.5, rng1);
+      o2 = OddsSimulator.nextOdds(o2, 0.5, rng2);
+    }
+    assertThat(o1).isEqualTo(o2);
+  }
+}


## `feat(mock): expose deterministic provider shell`

diff --git a/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProvider.java b/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProvider.java
new file mode 100644
index 0000000..1ae3235
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProvider.java
@@ -0,0 +1,51 @@
+package com.sportsbook.oddsfeed.provider.mock;
+
+import com.sportsbook.oddsfeed.config.MockProperties;
+import com.sportsbook.oddsfeed.provider.EventSummary;
+import com.sportsbook.oddsfeed.provider.MatchOutcome;
+import com.sportsbook.oddsfeed.provider.OddsProvider;
+import com.sportsbook.oddsfeed.provider.ProviderEvent;
+import com.sportsbook.oddsfeed.provider.Sport;
+import com.sportsbook.protocol.value.EventId;
+import java.time.Clock;
+import java.util.List;
+import java.util.Optional;
+import org.springframework.context.annotation.Profile;
+import org.springframework.stereotype.Component;
+import reactor.core.publisher.Flux;
+
+@Component
+@Profile("mock")
+public class MockOddsProvider implements OddsProvider {
+
+  private final MockProperties properties;
+  private final Clock clock;
+
+  public MockOddsProvider(MockProperties properties, Clock clock) {
+    this.properties = properties;
+    this.clock = clock;
+  }
+
+  MockProperties properties() {
+    return properties;
+  }
+
+  Clock clock() {
+    return clock;
+  }
+
+  @Override
+  public List<EventSummary> listEvents(Sport sport) {
+    return List.of();
+  }
+
+  @Override
+  public Flux<ProviderEvent> streamEvents(EventId eventId) {
+    return Flux.empty();
+  }
+
+  @Override
+  public Optional<MatchOutcome> getMatchResult(EventId eventId) {
+    return Optional.empty();
+  }
+}


## `feat(mock): model one-by-two market state`

diff --git a/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProvider.java b/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProvider.java
index 1ae3235..890da00 100644
--- a/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProvider.java
+++ b/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProvider.java
@@ -6,10 +6,20 @@ import com.sportsbook.oddsfeed.provider.MatchOutcome;
 import com.sportsbook.oddsfeed.provider.OddsProvider;
 import com.sportsbook.oddsfeed.provider.ProviderEvent;
 import com.sportsbook.oddsfeed.provider.Sport;
+import com.sportsbook.protocol.domain.MarketType;
+import com.sportsbook.protocol.event.EventLifecycleStatus;
+import com.sportsbook.protocol.event.MarketStatus;
 import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.MarketId;
+import com.sportsbook.protocol.value.Odds;
+import com.sportsbook.protocol.value.SelectionId;
 import java.time.Clock;
+import java.time.Instant;
+import java.util.ArrayList;
 import java.util.List;
+import java.util.Map;
 import java.util.Optional;
+import java.util.concurrent.ConcurrentHashMap;
 import org.springframework.context.annotation.Profile;
 import org.springframework.stereotype.Component;
 import reactor.core.publisher.Flux;
@@ -20,6 +30,7 @@ public class MockOddsProvider implements OddsProvider {
 
   private final MockProperties properties;
   private final Clock clock;
+  private final Map<EventId, MockEvent> events = new ConcurrentHashMap<>();
 
   public MockOddsProvider(MockProperties properties, Clock clock) {
     this.properties = properties;
@@ -36,7 +47,13 @@ public class MockOddsProvider implements OddsProvider {
 
   @Override
   public List<EventSummary> listEvents(Sport sport) {
-    return List.of();
+    List<EventSummary> result = new ArrayList<>();
+    for (MockEvent event : events.values()) {
+      if (event.summary.sport() == sport) {
+        result.add(event.summary);
+      }
+    }
+    return List.copyOf(result);
   }
 
   @Override
@@ -48,4 +65,39 @@ public class MockOddsProvider implements OddsProvider {
   public Optional<MatchOutcome> getMatchResult(EventId eventId) {
     return Optional.empty();
   }
+
+  static final class MockEvent {
+    EventSummary summary;
+    final Map<MarketId, MockMarket> markets = new ConcurrentHashMap<>();
+    Instant kickoffAt;
+    Instant endAt;
+    EventLifecycleStatus status;
+  }
+
+  static final class MockMarket {
+    final MarketId marketId;
+    final MarketType type;
+    MarketStatus status = MarketStatus.OPEN;
+    final Map<SelectionId, MockSelection> selections;
+
+    MockMarket(MarketId marketId, MarketType type, Map<SelectionId, MockSelection> selections) {
+      this.marketId = marketId;
+      this.type = type;
+      this.selections = selections;
+    }
+  }
+
+  static final class MockSelection {
+    final SelectionId selectionId;
+    final String name;
+    final double impliedProbability;
+    Odds currentOdds;
+
+    MockSelection(SelectionId id, String name, double impliedProbability, Odds initial) {
+      this.selectionId = id;
+      this.name = name;
+      this.impliedProbability = impliedProbability;
+      this.currentOdds = initial;
+    }
+  }
 }


## `test(mock): verify market state transitions`

diff --git a/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProviderTest.java b/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProviderTest.java
new file mode 100644
index 0000000..862417a
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProviderTest.java
@@ -0,0 +1,33 @@
+package com.sportsbook.oddsfeed.provider.mock;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.domain.MarketType;
+import com.sportsbook.protocol.event.MarketStatus;
+import com.sportsbook.protocol.value.MarketId;
+import com.sportsbook.protocol.value.Odds;
+import com.sportsbook.protocol.value.SelectionId;
+import java.util.LinkedHashMap;
+import java.util.Map;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class MockOddsProviderTest {
+
+  @Test
+  void marketStartsOpenWithNamedSelections() {
+    SelectionId selectionId = new SelectionId(UUID.randomUUID());
+    MockOddsProvider.MockSelection selection =
+        new MockOddsProvider.MockSelection(selectionId, "HOME", 0.45, Odds.ofDecimal("2.2222"));
+    Map<SelectionId, MockOddsProvider.MockSelection> selections = new LinkedHashMap<>();
+    selections.put(selectionId, selection);
+
+    MockOddsProvider.MockMarket market =
+        new MockOddsProvider.MockMarket(
+            new MarketId(UUID.randomUUID()), MarketType.MATCH_RESULT_1X2, selections);
+
+    assertThat(market.status).isEqualTo(MarketStatus.OPEN);
+    assertThat(market.type).isEqualTo(MarketType.MATCH_RESULT_1X2);
+    assertThat(market.selections.get(selectionId).name).isEqualTo("HOME");
+  }
+}


## `feat(mock): seed deterministic event fixtures`

diff --git a/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProvider.java b/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProvider.java
index 890da00..5693c87 100644
--- a/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProvider.java
+++ b/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProvider.java
@@ -13,12 +13,17 @@ import com.sportsbook.protocol.value.EventId;
 import com.sportsbook.protocol.value.MarketId;
 import com.sportsbook.protocol.value.Odds;
 import com.sportsbook.protocol.value.SelectionId;
+import jakarta.annotation.PostConstruct;
 import java.time.Clock;
+import java.time.Duration;
 import java.time.Instant;
 import java.util.ArrayList;
+import java.util.LinkedHashMap;
 import java.util.List;
 import java.util.Map;
 import java.util.Optional;
+import java.util.Random;
+import java.util.UUID;
 import java.util.concurrent.ConcurrentHashMap;
 import org.springframework.context.annotation.Profile;
 import org.springframework.stereotype.Component;
@@ -28,15 +33,92 @@ import reactor.core.publisher.Flux;
 @Profile("mock")
 public class MockOddsProvider implements OddsProvider {
 
+  static final int INITIAL_EVENT_COUNT = 3;
+  static final Duration MATCH_DURATION = Duration.ofMinutes(90);
+  static final Duration KICKOFF_SPACING = Duration.ofMinutes(1);
+  private static final double SECONDS_PER_MINUTE = 60.0;
+  private static final String[] FOOTBALL_TEAMS = {
+    "Manchester United",
+    "Chelsea",
+    "Liverpool",
+    "Arsenal",
+    "Tottenham",
+    "Manchester City",
+    "Newcastle",
+    "Brighton"
+  };
+
   private final MockProperties properties;
   private final Clock clock;
   private final Map<EventId, MockEvent> events = new ConcurrentHashMap<>();
+  private long runSeed;
+  private Random structureRandom;
 
   public MockOddsProvider(MockProperties properties, Clock clock) {
     this.properties = properties;
     this.clock = clock;
   }
 
+  @PostConstruct
+  void seed() {
+    runSeed = properties.randomSeed() == 0 ? new Random().nextLong() : properties.randomSeed();
+    structureRandom = new Random(runSeed);
+    Instant now = clock.instant();
+    for (int index = 0; index < INITIAL_EVENT_COUNT; index++) {
+      Instant kickoff = now.plus(toRealDuration(KICKOFF_SPACING.multipliedBy(index)));
+      Instant end = kickoff.plus(toRealDuration(MATCH_DURATION));
+      MockEvent event = buildEvent(kickoff, end, index);
+      events.put(event.summary.eventId(), event);
+    }
+  }
+
+  private MockEvent buildEvent(Instant kickoff, Instant end, int index) {
+    EventId eventId = new EventId(nextUuid());
+    MarketId marketId = new MarketId(nextUuid());
+    Map<SelectionId, MockSelection> selections = new LinkedHashMap<>();
+    addSelection(selections, "HOME", properties.baseHomeWinProbability());
+    addSelection(selections, "DRAW", properties.baseDrawProbability());
+    addSelection(selections, "AWAY", properties.baseAwayWinProbability());
+
+    MockMarket market = new MockMarket(marketId, MarketType.MATCH_RESULT_1X2, selections);
+    int homeIndex = (index * 2) % FOOTBALL_TEAMS.length;
+    EventSummary summary =
+        new EventSummary(
+            eventId,
+            Sport.FOOTBALL,
+            "Premier League",
+            FOOTBALL_TEAMS[homeIndex],
+            FOOTBALL_TEAMS[(homeIndex + 1) % FOOTBALL_TEAMS.length],
+            kickoff,
+            EventLifecycleStatus.SCHEDULED);
+
+    MockEvent event = new MockEvent();
+    event.summary = summary;
+    event.markets.put(marketId, market);
+    event.kickoffAt = kickoff;
+    event.endAt = end;
+    event.status = EventLifecycleStatus.SCHEDULED;
+    return event;
+  }
+
+  private void addSelection(
+      Map<SelectionId, MockSelection> selections, String name, double probability) {
+    SelectionId selectionId = new SelectionId(nextUuid());
+    selections.put(
+        selectionId,
+        new MockSelection(selectionId, name, probability, OddsSimulator.initialOdds(probability)));
+  }
+
+  private UUID nextUuid() {
+    return new UUID(structureRandom.nextLong(), structureRandom.nextLong());
+  }
+
+  private Duration toRealDuration(Duration mockDuration) {
+    double mockMinutes = mockDuration.toSeconds() / SECONDS_PER_MINUTE;
+    long realSeconds = Math.max(1L, (long) (mockMinutes / properties.minutesPerSecond()));
+    return Duration.ofSeconds(realSeconds);
+  }
+
   MockProperties properties() {
     return properties;
   }


## `test(mock): verify deterministic event catalog`

diff --git a/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProviderTest.java b/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProviderTest.java
index 862417a..77db25f 100644
--- a/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProviderTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProviderTest.java
@@ -2,11 +2,17 @@ package com.sportsbook.oddsfeed.provider.mock;
 
 import static org.assertj.core.api.Assertions.assertThat;
 
+import com.sportsbook.oddsfeed.config.MockProperties;
+import com.sportsbook.oddsfeed.provider.Sport;
 import com.sportsbook.protocol.domain.MarketType;
+import com.sportsbook.protocol.event.EventLifecycleStatus;
 import com.sportsbook.protocol.event.MarketStatus;
 import com.sportsbook.protocol.value.MarketId;
 import com.sportsbook.protocol.value.Odds;
 import com.sportsbook.protocol.value.SelectionId;
+import java.time.Clock;
+import java.time.Instant;
+import java.time.ZoneOffset;
 import java.util.LinkedHashMap;
 import java.util.Map;
 import java.util.UUID;
@@ -14,6 +20,8 @@ import org.junit.jupiter.api.Test;
 
 class MockOddsProviderTest {
 
+  private static final Instant NOW = Instant.parse("2026-05-28T10:00:00Z");
+
   @Test
   void marketStartsOpenWithNamedSelections() {
     SelectionId selectionId = new SelectionId(UUID.randomUUID());
@@ -30,4 +38,29 @@ class MockOddsProviderTest {
     assertThat(market.type).isEqualTo(MarketType.MATCH_RESULT_1X2);
     assertThat(market.selections.get(selectionId).name).isEqualTo("HOME");
   }
+
+  @Test
+  void seedCreatesStableScheduledCatalog() {
+    MockOddsProvider first = newProvider(424242L);
+    MockOddsProvider second = newProvider(424242L);
+    first.seed();
+    second.seed();
+
+    assertThat(first.listEvents(Sport.FOOTBALL)).hasSize(MockOddsProvider.INITIAL_EVENT_COUNT);
+    assertThat(first.listEvents(Sport.FOOTBALL))
+        .extracting(summary -> summary.eventId().value())
+        .containsExactlyInAnyOrderElementsOf(
+            second.listEvents(Sport.FOOTBALL).stream()
+                .map(summary -> summary.eventId().value())
+                .toList());
+    assertThat(first.listEvents(Sport.FOOTBALL))
+        .allMatch(summary -> summary.status() == EventLifecycleStatus.SCHEDULED);
+  }
+
+  private static MockOddsProvider newProvider(long seed) {
+    MockProperties properties =
+        new MockProperties(
+            1.0, new MockProperties.Scenarios(false, 60), 0.45, 0.25, 0.30, seed, 500);
+    return new MockOddsProvider(properties, Clock.fixed(NOW, ZoneOffset.UTC));
+  }
 }


