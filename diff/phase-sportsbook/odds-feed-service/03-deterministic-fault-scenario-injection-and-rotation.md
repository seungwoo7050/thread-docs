# 결정적 장애 시나리오 주입과 순환

## `feat(scenarios): postpone scheduled matches`

diff --git a/src/main/java/com/sportsbook/oddsfeed/provider/mock/MatchPostponed.java b/src/main/java/com/sportsbook/oddsfeed/provider/mock/MatchPostponed.java
new file mode 100644
index 0000000..4ea7a31
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/provider/mock/MatchPostponed.java
@@ -0,0 +1,24 @@
+package com.sportsbook.oddsfeed.provider.mock;
+
+import com.sportsbook.protocol.event.EventLifecycleStatus;
+import java.time.Instant;
+import java.util.Random;
+
+public final class MatchPostponed implements MockScenario {
+
+  @Override
+  public String id() {
+    return "MatchPostponed";
+  }
+
+  @Override
+  public boolean canApply(MockOddsProvider.MockEvent event, Instant now) {
+    return event.status == EventLifecycleStatus.SCHEDULED;
+  }
+
+  @Override
+  public void apply(
+      MockOddsProvider.MockEvent event, Instant now, Random rng, MockOddsProvider provider) {
+    provider.transitionTo(event, EventLifecycleStatus.POSTPONED, now);
+  }
+}
diff --git a/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockScenario.java b/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockScenario.java
new file mode 100644
index 0000000..0278705
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockScenario.java
@@ -0,0 +1,13 @@
+package com.sportsbook.oddsfeed.provider.mock;
+
+import java.time.Instant;
+import java.util.Random;
+
+public interface MockScenario {
+
+  String id();
+
+  boolean canApply(MockOddsProvider.MockEvent event, Instant now);
+
+  void apply(MockOddsProvider.MockEvent event, Instant now, Random rng, MockOddsProvider provider);
+}


## `test(scenarios): verify postponed matches`

diff --git a/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockScenarioTest.java b/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockScenarioTest.java
new file mode 100644
index 0000000..478c597
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockScenarioTest.java
@@ -0,0 +1,56 @@
+package com.sportsbook.oddsfeed.provider.mock;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.oddsfeed.config.MockProperties;
+import com.sportsbook.oddsfeed.provider.EventSummary;
+import com.sportsbook.protocol.event.EventLifecycleStatus;
+import java.time.Clock;
+import java.time.Instant;
+import java.time.ZoneOffset;
+import java.util.Random;
+import org.junit.jupiter.api.BeforeEach;
+import org.junit.jupiter.api.Test;
+
+class MockScenarioTest {
+
+  private static final Instant NOW = Instant.parse("2026-05-28T10:00:00Z");
+  private static final MockProperties PROPERTIES =
+      new MockProperties(1.0, new MockProperties.Scenarios(false, 60), 0.45, 0.25, 0.30, 42L, 500);
+
+  private MockOddsProvider provider;
+  private MockOddsProvider.MockEvent event;
+
+  @BeforeEach
+  void setUp() {
+    provider = new MockOddsProvider(PROPERTIES, Clock.fixed(NOW, ZoneOffset.UTC));
+    provider.seed();
+    event = provider.activeEvents().iterator().next();
+  }
+
+  @Test
+  void postponedScenarioTransitionsScheduledEvent() {
+    EventSummary before = event.summary;
+
+    new MatchPostponed().apply(event, NOW, new Random(0), provider);
+
+    assertThat(event.status).isEqualTo(EventLifecycleStatus.POSTPONED);
+    assertThat(event.summary.eventId()).isEqualTo(before.eventId());
+  }
+
+  @Test
+  void postponedScenarioRejectsInPlayEvent() {
+    provider.tick(event.kickoffAt);
+
+    assertThat(new MatchPostponed().canApply(event, event.kickoffAt)).isFalse();
+  }
+
+  @Test
+  void postponedEventStopsAdvancing() {
+    new MatchPostponed().apply(event, NOW, new Random(0), provider);
+
+    provider.tick(event.endAt.plusSeconds(1));
+
+    assertThat(event.status).isEqualTo(EventLifecycleStatus.POSTPONED);
+  }
+}


## `feat(scenarios): suspend markets abruptly`

diff --git a/src/main/java/com/sportsbook/oddsfeed/provider/mock/SuddenMarketSuspend.java b/src/main/java/com/sportsbook/oddsfeed/provider/mock/SuddenMarketSuspend.java
new file mode 100644
index 0000000..b751adb
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/provider/mock/SuddenMarketSuspend.java
@@ -0,0 +1,47 @@
+package com.sportsbook.oddsfeed.provider.mock;
+
+import com.sportsbook.oddsfeed.provider.ProviderEvent;
+import com.sportsbook.protocol.event.EventLifecycleStatus;
+import com.sportsbook.protocol.event.MarketStatus;
+import java.time.Instant;
+import java.util.Random;
+
+public final class SuddenMarketSuspend implements MockScenario {
+
+  private static final String REASON = "simulated in-play pause";
+
+  @Override
+  public String id() {
+    return "SuddenMarketSuspend";
+  }
+
+  @Override
+  public boolean canApply(MockOddsProvider.MockEvent event, Instant now) {
+    if (event.status != EventLifecycleStatus.IN_PLAY
+        && event.status != EventLifecycleStatus.SCHEDULED) {
+      return false;
+    }
+    return event.markets.values().stream().anyMatch(market -> market.status == MarketStatus.OPEN);
+  }
+
+  @Override
+  public void apply(
+      MockOddsProvider.MockEvent event, Instant now, Random rng, MockOddsProvider provider) {
+    MockOddsProvider.MockMarket target =
+        event.markets.values().stream()
+            .filter(market -> market.status == MarketStatus.OPEN)
+            .findFirst()
+            .orElseThrow();
+    MarketStatus previous = target.status;
+    target.status = MarketStatus.SUSPENDED;
+    provider.emit(
+        event.summary.eventId(),
+        new ProviderEvent.MarketStatusUpdated(
+            event.summary.eventId(),
+            target.marketId,
+            previous,
+            MarketStatus.SUSPENDED,
+            REASON,
+            now));
+  }
+}


## `test(scenarios): verify sudden suspension`

diff --git a/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockScenarioTest.java b/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockScenarioTest.java
index 478c597..d8ab8bc 100644
--- a/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockScenarioTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockScenarioTest.java
@@ -4,10 +4,14 @@ import static org.assertj.core.api.Assertions.assertThat;
 
 import com.sportsbook.oddsfeed.config.MockProperties;
 import com.sportsbook.oddsfeed.provider.EventSummary;
+import com.sportsbook.oddsfeed.provider.ProviderEvent;
 import com.sportsbook.protocol.event.EventLifecycleStatus;
+import com.sportsbook.protocol.event.MarketStatus;
 import java.time.Clock;
 import java.time.Instant;
 import java.time.ZoneOffset;
+import java.util.ArrayList;
+import java.util.List;
 import java.util.Random;
 import org.junit.jupiter.api.BeforeEach;
 import org.junit.jupiter.api.Test;
@@ -53,4 +57,29 @@ class MockScenarioTest {
 
     assertThat(event.status).isEqualTo(EventLifecycleStatus.POSTPONED);
   }
+
+  @Test
+  void suddenSuspensionChangesOneOpenMarket() {
+    List<ProviderEvent.MarketStatusUpdated> updates = new ArrayList<>();
+    var subscription =
+        provider
+            .streamEvents(event.summary.eventId())
+            .ofType(ProviderEvent.MarketStatusUpdated.class)
+            .subscribe(updates::add);
+
+    new SuddenMarketSuspend().apply(event, NOW, new Random(0), provider);
+    subscription.dispose();
+
+    assertThat(event.markets.values())
+        .extracting(market -> market.status)
+        .containsExactly(MarketStatus.SUSPENDED);
+    assertThat(updates)
+        .singleElement()
+        .satisfies(
+            update -> {
+              assertThat(update.previousStatus()).isEqualTo(MarketStatus.OPEN);
+              assertThat(update.newStatus()).isEqualTo(MarketStatus.SUSPENDED);
+              assertThat(update.reason()).isNotBlank();
+            });
+  }
 }


## `feat(scenarios): crash market odds`

diff --git a/src/main/java/com/sportsbook/oddsfeed/provider/mock/OddsCrash.java b/src/main/java/com/sportsbook/oddsfeed/provider/mock/OddsCrash.java
new file mode 100644
index 0000000..ce65c62
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/provider/mock/OddsCrash.java
@@ -0,0 +1,47 @@
+package com.sportsbook.oddsfeed.provider.mock;
+
+import com.sportsbook.oddsfeed.provider.ProviderEvent;
+import com.sportsbook.protocol.event.EventLifecycleStatus;
+import com.sportsbook.protocol.value.Odds;
+import java.math.BigDecimal;
+import java.math.RoundingMode;
+import java.time.Instant;
+import java.util.List;
+import java.util.Random;
+
+public final class OddsCrash implements MockScenario {
+
+  private static final double MULTIPLIER = 0.3;
+
+  @Override
+  public String id() {
+    return "OddsCrash";
+  }
+
+  @Override
+  public boolean canApply(MockOddsProvider.MockEvent event, Instant now) {
+    return event.status == EventLifecycleStatus.IN_PLAY
+        || event.status == EventLifecycleStatus.SCHEDULED;
+  }
+
+  @Override
+  public void apply(
+      MockOddsProvider.MockEvent event, Instant now, Random rng, MockOddsProvider provider) {
+    MockOddsProvider.MockMarket market = event.markets.values().iterator().next();
+    List<MockOddsProvider.MockSelection> selections = List.copyOf(market.selections.values());
+    MockOddsProvider.MockSelection target = selections.get(rng.nextInt(selections.size()));
+    Odds previous = target.currentOdds;
+    BigDecimal crashed =
+        previous
+            .decimal()
+            .multiply(BigDecimal.valueOf(MULTIPLIER))
+            .max(BigDecimal.valueOf(OddsSimulator.MIN_ODDS))
+            .setScale(Odds.SCALE, RoundingMode.HALF_EVEN);
+    Odds next = Odds.ofDecimal(crashed);
+    target.currentOdds = next;
+    provider.emit(
+        event.summary.eventId(),
+        new ProviderEvent.OddsUpdated(
+            event.summary.eventId(), market.marketId, target.selectionId, previous, next, now));
+  }
+}


## `test(scenarios): verify odds crashes`

diff --git a/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockScenarioTest.java b/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockScenarioTest.java
index d8ab8bc..1b59893 100644
--- a/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockScenarioTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockScenarioTest.java
@@ -7,6 +7,8 @@ import com.sportsbook.oddsfeed.provider.EventSummary;
 import com.sportsbook.oddsfeed.provider.ProviderEvent;
 import com.sportsbook.protocol.event.EventLifecycleStatus;
 import com.sportsbook.protocol.event.MarketStatus;
+import java.math.BigDecimal;
+import java.math.RoundingMode;
 import java.time.Clock;
 import java.time.Instant;
 import java.time.ZoneOffset;
@@ -82,4 +84,30 @@ class MockScenarioTest {
               assertThat(update.reason()).isNotBlank();
             });
   }
+
+  @Test
+  void oddsCrashPublishesOneSharpPriceChange() {
+    List<ProviderEvent.OddsUpdated> updates = new ArrayList<>();
+    var subscription =
+        provider
+            .streamEvents(event.summary.eventId())
+            .ofType(ProviderEvent.OddsUpdated.class)
+            .subscribe(updates::add);
+
+    new OddsCrash().apply(event, NOW, new Random(0), provider);
+    subscription.dispose();
+
+    assertThat(updates)
+        .singleElement()
+        .satisfies(
+            update -> {
+              var ratio =
+                  update
+                      .newOdds()
+                      .decimal()
+                      .divide(update.previousOdds().decimal(), 4, RoundingMode.HALF_EVEN);
+              assertThat(ratio).isLessThan(new BigDecimal("0.5"));
+              assertThat(update.occurredAt()).isEqualTo(NOW);
+            });
+  }
 }


## `feat(scenarios): model late goals`

diff --git a/src/main/java/com/sportsbook/oddsfeed/provider/mock/LateGoal.java b/src/main/java/com/sportsbook/oddsfeed/provider/mock/LateGoal.java
new file mode 100644
index 0000000..e10b1f0
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/provider/mock/LateGoal.java
@@ -0,0 +1,52 @@
+package com.sportsbook.oddsfeed.provider.mock;
+
+import com.sportsbook.oddsfeed.provider.ProviderEvent;
+import com.sportsbook.protocol.event.EventLifecycleStatus;
+import com.sportsbook.protocol.value.Odds;
+import java.math.BigDecimal;
+import java.math.RoundingMode;
+import java.time.Duration;
+import java.time.Instant;
+import java.util.List;
+import java.util.Random;
+
+public final class LateGoal implements MockScenario {
+
+  static final Duration TRIGGER_WINDOW = Duration.ofMinutes(10);
+  private static final double MULTIPLIER = 0.5;
+
+  @Override
+  public String id() {
+    return "LateGoal";
+  }
+
+  @Override
+  public boolean canApply(MockOddsProvider.MockEvent event, Instant now) {
+    if (event.status != EventLifecycleStatus.IN_PLAY) {
+      return false;
+    }
+    Duration remaining = Duration.between(now, event.endAt);
+    return !remaining.isNegative() && remaining.compareTo(TRIGGER_WINDOW) <= 0;
+  }
+
+  @Override
+  public void apply(
+      MockOddsProvider.MockEvent event, Instant now, Random rng, MockOddsProvider provider) {
+    MockOddsProvider.MockMarket market = event.markets.values().iterator().next();
+    List<MockOddsProvider.MockSelection> selections = List.copyOf(market.selections.values());
+    MockOddsProvider.MockSelection target = selections.get(rng.nextInt(selections.size()));
+    Odds previous = target.currentOdds;
+    BigDecimal shortened =
+        previous
+            .decimal()
+            .multiply(BigDecimal.valueOf(MULTIPLIER))
+            .max(BigDecimal.valueOf(OddsSimulator.MIN_ODDS))
+            .setScale(Odds.SCALE, RoundingMode.HALF_EVEN);
+    Odds next = Odds.ofDecimal(shortened);
+    target.currentOdds = next;
+    provider.emit(
+        event.summary.eventId(),
+        new ProviderEvent.OddsUpdated(
+            event.summary.eventId(), market.marketId, target.selectionId, previous, next, now));
+  }
+}
diff --git a/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockScenario.java b/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockScenario.java
index 0278705..9197ad5 100644
--- a/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockScenario.java
+++ b/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockScenario.java
@@ -3,7 +3,8 @@ package com.sportsbook.oddsfeed.provider.mock;
 import java.time.Instant;
 import java.util.Random;
 
-public interface MockScenario {
+public sealed interface MockScenario
+    permits LateGoal, MatchPostponed, OddsCrash, SuddenMarketSuspend {
 
   String id();
 


## `test(scenarios): verify late goals`

diff --git a/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockScenarioTest.java b/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockScenarioTest.java
index 1b59893..784f13b 100644
--- a/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockScenarioTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockScenarioTest.java
@@ -110,4 +110,36 @@ class MockScenarioTest {
               assertThat(update.occurredAt()).isEqualTo(NOW);
             });
   }
+
+  @Test
+  void lateGoalOnlyAppliesNearTheEndOfPlay() {
+    LateGoal scenario = new LateGoal();
+    assertThat(scenario.canApply(event, NOW)).isFalse();
+
+    provider.tick(event.kickoffAt);
+    Instant nearEnd = event.endAt.minusSeconds(1);
+    List<ProviderEvent.OddsUpdated> updates = new ArrayList<>();
+    var subscription =
+        provider
+            .streamEvents(event.summary.eventId())
+            .ofType(ProviderEvent.OddsUpdated.class)
+            .subscribe(updates::add);
+    updates.clear();
+
+    assertThat(scenario.canApply(event, nearEnd)).isTrue();
+    scenario.apply(event, nearEnd, new Random(1), provider);
+    subscription.dispose();
+
+    assertThat(updates)
+        .singleElement()
+        .satisfies(update -> assertThat(update.newOdds()).isNotEqualTo(update.previousOdds()));
+  }
+
+  @Test
+  void scenarioSetIsClosedToFourDisruptions() {
+    assertThat(MockScenario.class.getPermittedSubclasses())
+        .extracting(Class::getSimpleName)
+        .containsExactlyInAnyOrder(
+            "LateGoal", "MatchPostponed", "OddsCrash", "SuddenMarketSuspend");
+  }
 }


## `feat(scenarios): rotate deterministic scenarios`

diff --git a/src/main/java/com/sportsbook/oddsfeed/provider/mock/ScenarioRotator.java b/src/main/java/com/sportsbook/oddsfeed/provider/mock/ScenarioRotator.java
new file mode 100644
index 0000000..9d112c4
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/provider/mock/ScenarioRotator.java
@@ -0,0 +1,55 @@
+package com.sportsbook.oddsfeed.provider.mock;
+
+import com.sportsbook.oddsfeed.config.MockProperties;
+import java.time.Clock;
+import java.time.Instant;
+import java.util.List;
+import java.util.Random;
+import java.util.concurrent.TimeUnit;
+import org.springframework.context.annotation.Profile;
+import org.springframework.scheduling.annotation.Scheduled;
+import org.springframework.stereotype.Component;
+
+@Component
+@Profile("mock")
+public class ScenarioRotator {
+
+  private final MockProperties properties;
+  private final MockOddsProvider provider;
+  private final Clock clock;
+  private final List<MockScenario> scenarios;
+  private final Random random;
+
+  public ScenarioRotator(MockProperties properties, MockOddsProvider provider, Clock clock) {
+    this.properties = properties;
+    this.provider = provider;
+    this.clock = clock;
+    this.scenarios =
+        List.of(new LateGoal(), new MatchPostponed(), new SuddenMarketSuspend(), new OddsCrash());
+    this.random =
+        properties.randomSeed() == 0 ? new Random() : new Random(properties.randomSeed() + 1);
+  }
+
+  @Scheduled(
+      fixedRateString = "${oddsfeed.mock.scenarios.rotation-interval-seconds:60}",
+      timeUnit = TimeUnit.SECONDS)
+  void scheduledRotate() {
+    if (properties.scenarios().autoRotate()) {
+      rotateOnce(clock.instant());
+    }
+  }
+
+  void rotateOnce(Instant now) {
+    MockScenario scenario = scenarios.get(random.nextInt(scenarios.size()));
+    for (MockOddsProvider.MockEvent event : provider.activeEvents()) {
+      if (scenario.canApply(event, now)) {
+        scenario.apply(event, now, random, provider);
+        return;
+      }
+    }
+  }
+
+  List<MockScenario> scenarios() {
+    return scenarios;
+  }
+}


## `test(scenarios): verify scenario rotation`

diff --git a/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockScenarioTest.java b/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockScenarioTest.java
index 784f13b..f7ffa95 100644
--- a/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockScenarioTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockScenarioTest.java
@@ -142,4 +142,24 @@ class MockScenarioTest {
         .containsExactlyInAnyOrder(
             "LateGoal", "MatchPostponed", "OddsCrash", "SuddenMarketSuspend");
   }
+
+  @Test
+  void rotatorAppliesOneEligibleScenario() {
+    ScenarioRotator rotator =
+        new ScenarioRotator(PROPERTIES, provider, Clock.fixed(NOW, ZoneOffset.UTC));
+
+    rotator.rotateOnce(NOW);
+
+    assertThat(rotator.scenarios())
+        .extracting(MockScenario::id)
+        .containsExactly("LateGoal", "MatchPostponed", "SuddenMarketSuspend", "OddsCrash");
+    boolean changed = false;
+    for (MockOddsProvider.MockEvent candidate : provider.activeEvents()) {
+      changed |=
+          candidate.status != EventLifecycleStatus.SCHEDULED
+              || candidate.markets.values().stream()
+                  .anyMatch(market -> market.status != MarketStatus.OPEN);
+    }
+    assertThat(changed).isTrue();
+  }
 }
