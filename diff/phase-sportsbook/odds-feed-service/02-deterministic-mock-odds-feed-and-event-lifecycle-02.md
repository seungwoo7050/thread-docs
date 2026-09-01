## `feat(mock): expose replayable provider streams`

diff --git a/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProvider.java b/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProvider.java
index 5693c87..f8bd706 100644
--- a/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProvider.java
+++ b/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProvider.java
@@ -28,6 +28,7 @@ import java.util.concurrent.ConcurrentHashMap;
 import org.springframework.context.annotation.Profile;
 import org.springframework.stereotype.Component;
 import reactor.core.publisher.Flux;
+import reactor.core.publisher.Sinks;
 
 @Component
 @Profile("mock")
@@ -36,6 +37,7 @@ public class MockOddsProvider implements OddsProvider {
   static final int INITIAL_EVENT_COUNT = 3;
   static final Duration MATCH_DURATION = Duration.ofMinutes(90);
   static final Duration KICKOFF_SPACING = Duration.ofMinutes(1);
+  private static final int REPLAY_HISTORY = 256;
   private static final double SECONDS_PER_MINUTE = 60.0;
   private static final String[] FOOTBALL_TEAMS = {
     "Manchester United",
@@ -51,6 +53,7 @@ public class MockOddsProvider implements OddsProvider {
   private final MockProperties properties;
   private final Clock clock;
   private final Map<EventId, MockEvent> events = new ConcurrentHashMap<>();
+  private final Map<EventId, Sinks.Many<ProviderEvent>> streams = new ConcurrentHashMap<>();
   private long runSeed;
   private Random structureRandom;
 
@@ -69,6 +72,7 @@ public class MockOddsProvider implements OddsProvider {
       Instant end = kickoff.plus(toRealDuration(MATCH_DURATION));
       MockEvent event = buildEvent(kickoff, end, index);
       events.put(event.summary.eventId(), event);
+      streams.put(event.summary.eventId(), Sinks.many().replay().limit(REPLAY_HISTORY));
     }
   }
 
@@ -127,6 +131,17 @@ public class MockOddsProvider implements OddsProvider {
     return clock;
   }
 
+  void emit(EventId eventId, ProviderEvent event) {
+    Sinks.Many<ProviderEvent> sink = streams.get(eventId);
+    if (sink == null) {
+      return;
+    }
+    Sinks.EmitResult result = sink.tryEmitNext(event);
+    if (result.isFailure()) {
+      throw new IllegalStateException("Could not emit mock provider event: " + result);
+    }
+  }
+
   @Override
   public List<EventSummary> listEvents(Sport sport) {
     List<EventSummary> result = new ArrayList<>();
@@ -140,7 +155,8 @@ public class MockOddsProvider implements OddsProvider {
 
   @Override
   public Flux<ProviderEvent> streamEvents(EventId eventId) {
-    return Flux.empty();
+    Sinks.Many<ProviderEvent> sink = streams.get(eventId);
+    return sink == null ? Flux.empty() : sink.asFlux();
   }
 
   @Override


## `test(mock): verify provider stream replay`

diff --git a/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProviderTest.java b/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProviderTest.java
index 77db25f..3dc5fe9 100644
--- a/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProviderTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProviderTest.java
@@ -3,6 +3,7 @@ package com.sportsbook.oddsfeed.provider.mock;
 import static org.assertj.core.api.Assertions.assertThat;
 
 import com.sportsbook.oddsfeed.config.MockProperties;
+import com.sportsbook.oddsfeed.provider.ProviderEvent;
 import com.sportsbook.oddsfeed.provider.Sport;
 import com.sportsbook.protocol.domain.MarketType;
 import com.sportsbook.protocol.event.EventLifecycleStatus;
@@ -17,6 +18,7 @@ import java.util.LinkedHashMap;
 import java.util.Map;
 import java.util.UUID;
 import org.junit.jupiter.api.Test;
+import reactor.test.StepVerifier;
 
 class MockOddsProviderTest {
 
@@ -57,6 +59,28 @@ class MockOddsProviderTest {
         .allMatch(summary -> summary.status() == EventLifecycleStatus.SCHEDULED);
   }
 
+  @Test
+  void streamReplaysEventsEmittedBeforeSubscription() {
+    MockOddsProvider provider = newProvider(424242L);
+    provider.seed();
+    var event = provider.listEvents(Sport.FOOTBALL).get(0);
+    ProviderEvent update =
+        new ProviderEvent.MarketStatusUpdated(
+            event.eventId(),
+            new MarketId(UUID.randomUUID()),
+            MarketStatus.OPEN,
+            MarketStatus.SUSPENDED,
+            "manual review",
+            NOW);
+
+    provider.emit(event.eventId(), update);
+
+    StepVerifier.create(provider.streamEvents(event.eventId()))
+        .expectNext(update)
+        .thenCancel()
+        .verify();
+  }
+
   private static MockOddsProvider newProvider(long seed) {
     MockProperties properties =
         new MockProperties(


## `feat(mock): advance event lifecycles`

diff --git a/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProvider.java b/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProvider.java
index f8bd706..67dde5f 100644
--- a/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProvider.java
+++ b/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProvider.java
@@ -26,6 +26,7 @@ import java.util.Random;
 import java.util.UUID;
 import java.util.concurrent.ConcurrentHashMap;
 import org.springframework.context.annotation.Profile;
+import org.springframework.scheduling.annotation.Scheduled;
 import org.springframework.stereotype.Component;
 import reactor.core.publisher.Flux;
 import reactor.core.publisher.Sinks;
@@ -142,6 +143,52 @@ public class MockOddsProvider implements OddsProvider {
     }
   }
 
+  @Scheduled(fixedRateString = "${oddsfeed.mock.tick-interval-ms:500}")
+  void scheduledTick() {
+    tick(clock.instant());
+  }
+
+  void tick(Instant now) {
+    for (MockEvent event : events.values()) {
+      advance(event, now);
+    }
+  }
+
+  private void advance(MockEvent event, Instant now) {
+    if (event.status == EventLifecycleStatus.FINISHED
+        || event.status == EventLifecycleStatus.CANCELLED
+        || event.status == EventLifecycleStatus.POSTPONED) {
+      return;
+    }
+    if (event.status == EventLifecycleStatus.SCHEDULED && !now.isBefore(event.kickoffAt)) {
+      transitionTo(event, EventLifecycleStatus.IN_PLAY, now);
+    }
+    if (event.status == EventLifecycleStatus.IN_PLAY && !now.isBefore(event.endAt)) {
+      transitionTo(event, EventLifecycleStatus.FINISHED, now);
+    }
+  }
+
+  Iterable<MockEvent> activeEvents() {
+    return events.values();
+  }
+
+  void transitionTo(MockEvent event, EventLifecycleStatus next, Instant now) {
+    event.status = next;
+    event.summary =
+        new EventSummary(
+            event.summary.eventId(),
+            event.summary.sport(),
+            event.summary.competition(),
+            event.summary.homeTeam(),
+            event.summary.awayTeam(),
+            event.summary.scheduledStartAt(),
+            next);
+    emit(
+        event.summary.eventId(),
+        new ProviderEvent.LifecycleUpdated(
+            event.summary.eventId(), next, event.summary.scheduledStartAt(), now));
+  }
+
   @Override
   public List<EventSummary> listEvents(Sport sport) {
     List<EventSummary> result = new ArrayList<>();


## `test(mock): verify lifecycle progression`

diff --git a/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProviderTest.java b/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProviderTest.java
index 3dc5fe9..c8ee01e 100644
--- a/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProviderTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProviderTest.java
@@ -81,6 +81,25 @@ class MockOddsProviderTest {
         .verify();
   }
 
+  @Test
+  void tickAdvancesScheduledEventsThroughFullTime() {
+    MockOddsProvider provider = newProvider(424242L);
+    provider.seed();
+    var event = provider.listEvents(Sport.FOOTBALL).get(0);
+
+    provider.tick(event.scheduledStartAt());
+    assertThat(provider.listEvents(Sport.FOOTBALL))
+        .filteredOn(summary -> summary.eventId().equals(event.eventId()))
+        .extracting(summary -> summary.status())
+        .containsExactly(EventLifecycleStatus.IN_PLAY);
+
+    provider.tick(event.scheduledStartAt().plusSeconds(90));
+    assertThat(provider.listEvents(Sport.FOOTBALL))
+        .filteredOn(summary -> summary.eventId().equals(event.eventId()))
+        .extracting(summary -> summary.status())
+        .containsExactly(EventLifecycleStatus.FINISHED);
+  }
+
   private static MockOddsProvider newProvider(long seed) {
     MockProperties properties =
         new MockProperties(


## `feat(mock): publish deterministic odds ticks`

diff --git a/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProvider.java b/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProvider.java
index 67dde5f..cfdc42d 100644
--- a/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProvider.java
+++ b/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProvider.java
@@ -40,6 +40,7 @@ public class MockOddsProvider implements OddsProvider {
   static final Duration KICKOFF_SPACING = Duration.ofMinutes(1);
   private static final int REPLAY_HISTORY = 256;
   private static final double SECONDS_PER_MINUTE = 60.0;
+  private static final long ODDS_STREAM_SALT = 0x9E3779B97F4A7C15L;
   private static final String[] FOOTBALL_TEAMS = {
     "Manchester United",
     "Chelsea",
@@ -57,6 +58,7 @@ public class MockOddsProvider implements OddsProvider {
   private final Map<EventId, Sinks.Many<ProviderEvent>> streams = new ConcurrentHashMap<>();
   private long runSeed;
   private Random structureRandom;
+  private Random oddsRandom;
 
   public MockOddsProvider(MockProperties properties, Clock clock) {
     this.properties = properties;
@@ -67,6 +69,7 @@ public class MockOddsProvider implements OddsProvider {
   void seed() {
     runSeed = properties.randomSeed() == 0 ? new Random().nextLong() : properties.randomSeed();
     structureRandom = new Random(runSeed);
+    oddsRandom = new Random(runSeed ^ ODDS_STREAM_SALT);
     Instant now = clock.instant();
     for (int index = 0; index < INITIAL_EVENT_COUNT; index++) {
       Instant kickoff = now.plus(toRealDuration(KICKOFF_SPACING.multipliedBy(index)));
@@ -165,6 +168,28 @@ public class MockOddsProvider implements OddsProvider {
     }
     if (event.status == EventLifecycleStatus.IN_PLAY && !now.isBefore(event.endAt)) {
       transitionTo(event, EventLifecycleStatus.FINISHED, now);
+      return;
+    }
+    for (MockMarket market : event.markets.values()) {
+      if (market.status != MarketStatus.OPEN) {
+        continue;
+      }
+      for (MockSelection selection : market.selections.values()) {
+        Odds previous = selection.currentOdds;
+        Odds next = OddsSimulator.nextOdds(previous, selection.impliedProbability, oddsRandom);
+        if (!previous.equals(next)) {
+          selection.currentOdds = next;
+          emit(
+              event.summary.eventId(),
+              new ProviderEvent.OddsUpdated(
+                  event.summary.eventId(),
+                  market.marketId,
+                  selection.selectionId,
+                  previous,
+                  next,
+                  now));
+        }
+      }
     }
   }
 


## `test(mock): verify odds tick cadence`

diff --git a/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProviderTest.java b/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProviderTest.java
index c8ee01e..287a6ce 100644
--- a/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProviderTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProviderTest.java
@@ -100,6 +100,42 @@ class MockOddsProviderTest {
         .containsExactly(EventLifecycleStatus.FINISHED);
   }
 
+  @Test
+  void oddsTicksAreDeterministicForTheConfiguredSeed() {
+    MockOddsProvider first = newProvider(99L);
+    MockOddsProvider second = newProvider(99L);
+    first.seed();
+    second.seed();
+    var firstEvent = first.listEvents(Sport.FOOTBALL).get(0);
+    var secondEvent =
+        second.listEvents(Sport.FOOTBALL).stream()
+            .filter(summary -> summary.eventId().equals(firstEvent.eventId()))
+            .findFirst()
+            .orElseThrow();
+
+    first.tick(firstEvent.scheduledStartAt());
+    second.tick(secondEvent.scheduledStartAt());
+
+    var firstOdds =
+        first
+            .streamEvents(firstEvent.eventId())
+            .ofType(ProviderEvent.OddsUpdated.class)
+            .map(ProviderEvent.OddsUpdated::newOdds)
+            .take(3)
+            .collectList()
+            .block();
+    var secondOdds =
+        second
+            .streamEvents(secondEvent.eventId())
+            .ofType(ProviderEvent.OddsUpdated.class)
+            .map(ProviderEvent.OddsUpdated::newOdds)
+            .take(3)
+            .collectList()
+            .block();
+
+    assertThat(firstOdds).containsExactlyElementsOf(secondOdds);
+  }
+
   private static MockOddsProvider newProvider(long seed) {
     MockProperties properties =
         new MockProperties(


## `feat(mock): settle terminal event fixtures`

diff --git a/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProvider.java b/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProvider.java
index cfdc42d..fa8738c 100644
--- a/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProvider.java
+++ b/src/main/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProvider.java
@@ -7,8 +7,10 @@ import com.sportsbook.oddsfeed.provider.OddsProvider;
 import com.sportsbook.oddsfeed.provider.ProviderEvent;
 import com.sportsbook.oddsfeed.provider.Sport;
 import com.sportsbook.protocol.domain.MarketType;
+import com.sportsbook.protocol.domain.SettlementResult;
 import com.sportsbook.protocol.event.EventLifecycleStatus;
 import com.sportsbook.protocol.event.MarketStatus;
+import com.sportsbook.protocol.event.MatchFinalStatus;
 import com.sportsbook.protocol.value.EventId;
 import com.sportsbook.protocol.value.MarketId;
 import com.sportsbook.protocol.value.Odds;
@@ -41,6 +43,8 @@ public class MockOddsProvider implements OddsProvider {
   private static final int REPLAY_HISTORY = 256;
   private static final double SECONDS_PER_MINUTE = 60.0;
   private static final long ODDS_STREAM_SALT = 0x9E3779B97F4A7C15L;
+  private static final long RESULT_STREAM_SALT = 0xD1B54A32D192ED03L;
+  private static final int EVENT_ID_ROTATION = 31;
   private static final String[] FOOTBALL_TEAMS = {
     "Manchester United",
     "Chelsea",
@@ -56,6 +60,7 @@ public class MockOddsProvider implements OddsProvider {
   private final Clock clock;
   private final Map<EventId, MockEvent> events = new ConcurrentHashMap<>();
   private final Map<EventId, Sinks.Many<ProviderEvent>> streams = new ConcurrentHashMap<>();
+  private final Map<EventId, MatchOutcome> outcomes = new ConcurrentHashMap<>();
   private long runSeed;
   private Random structureRandom;
   private Random oddsRandom;
@@ -167,6 +172,7 @@ public class MockOddsProvider implements OddsProvider {
       transitionTo(event, EventLifecycleStatus.IN_PLAY, now);
     }
     if (event.status == EventLifecycleStatus.IN_PLAY && !now.isBefore(event.endAt)) {
+      outcomes.put(event.summary.eventId(), synthesizeOutcome(event));
       transitionTo(event, EventLifecycleStatus.FINISHED, now);
       return;
     }
@@ -214,6 +220,49 @@ public class MockOddsProvider implements OddsProvider {
             event.summary.eventId(), next, event.summary.scheduledStartAt(), now));
   }
 
+  private MatchOutcome synthesizeOutcome(MockEvent event) {
+    double roll = resultRandom(event.summary.eventId()).nextDouble();
+    String score;
+    String winningSelection;
+    if (roll < properties.baseHomeWinProbability()) {
+      score = "2-1";
+      winningSelection = "HOME";
+    } else if (roll < properties.baseHomeWinProbability() + properties.baseDrawProbability()) {
+      score = "1-1";
+      winningSelection = "DRAW";
+    } else {
+      score = "0-1";
+      winningSelection = "AWAY";
+    }
+    return new MatchOutcome(
+        event.summary.eventId(),
+        score,
+        MatchFinalStatus.COMPLETED,
+        gradeSelections(event, winningSelection),
+        event.endAt);
+  }
+
+  private Map<String, String> gradeSelections(MockEvent event, String winningSelection) {
+    Map<String, String> detail = new LinkedHashMap<>();
+    for (MockMarket market : event.markets.values()) {
+      for (MockSelection selection : market.selections.values()) {
+        SettlementResult result =
+            selection.name.equals(winningSelection) ? SettlementResult.WON : SettlementResult.LOST;
+        detail.put(selection.selectionId.value().toString(), result.name());
+      }
+    }
+    return detail;
+  }
+
+  private Random resultRandom(EventId eventId) {
+    UUID value = eventId.value();
+    return new Random(
+        runSeed
+            ^ value.getMostSignificantBits()
+            ^ Long.rotateLeft(value.getLeastSignificantBits(), EVENT_ID_ROTATION)
+            ^ RESULT_STREAM_SALT);
+  }
+
   @Override
   public List<EventSummary> listEvents(Sport sport) {
     List<EventSummary> result = new ArrayList<>();
@@ -233,7 +282,7 @@ public class MockOddsProvider implements OddsProvider {
 
   @Override
   public Optional<MatchOutcome> getMatchResult(EventId eventId) {
-    return Optional.empty();
+    return Optional.ofNullable(outcomes.get(eventId));
   }
 
   static final class MockEvent {


## `test(mock): verify terminal outcomes`

diff --git a/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProviderTest.java b/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProviderTest.java
index 287a6ce..243b931 100644
--- a/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProviderTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProviderTest.java
@@ -6,8 +6,10 @@ import com.sportsbook.oddsfeed.config.MockProperties;
 import com.sportsbook.oddsfeed.provider.ProviderEvent;
 import com.sportsbook.oddsfeed.provider.Sport;
 import com.sportsbook.protocol.domain.MarketType;
+import com.sportsbook.protocol.domain.SettlementResult;
 import com.sportsbook.protocol.event.EventLifecycleStatus;
 import com.sportsbook.protocol.event.MarketStatus;
+import com.sportsbook.protocol.event.MatchFinalStatus;
 import com.sportsbook.protocol.value.MarketId;
 import com.sportsbook.protocol.value.Odds;
 import com.sportsbook.protocol.value.SelectionId;
@@ -17,6 +19,7 @@ import java.time.ZoneOffset;
 import java.util.LinkedHashMap;
 import java.util.Map;
 import java.util.UUID;
+import java.util.concurrent.atomic.AtomicReference;
 import org.junit.jupiter.api.Test;
 import reactor.test.StepVerifier;
 
@@ -136,6 +139,34 @@ class MockOddsProviderTest {
     assertThat(firstOdds).containsExactlyElementsOf(secondOdds);
   }
 
+  @Test
+  void finishedLifecycleExposesGradedOutcomeImmediately() {
+    MockOddsProvider provider = newProvider(424242L);
+    provider.seed();
+    var event = provider.listEvents(Sport.FOOTBALL).get(0);
+    AtomicReference<MatchFinalStatus> observedStatus = new AtomicReference<>();
+    provider
+        .streamEvents(event.eventId())
+        .ofType(ProviderEvent.LifecycleUpdated.class)
+        .filter(update -> update.status() == EventLifecycleStatus.FINISHED)
+        .subscribe(
+            ignored ->
+                observedStatus.set(
+                    provider.getMatchResult(event.eventId()).orElseThrow().finalStatus()));
+
+    provider.tick(event.scheduledStartAt());
+    provider.tick(event.scheduledStartAt().plusSeconds(90));
+
+    var outcome = provider.getMatchResult(event.eventId()).orElseThrow();
+    assertThat(observedStatus).hasValue(MatchFinalStatus.COMPLETED);
+    assertThat(outcome.detail()).hasSize(3);
+    assertThat(outcome.detail().values())
+        .containsExactlyInAnyOrder(
+            SettlementResult.WON.name(),
+            SettlementResult.LOST.name(),
+            SettlementResult.LOST.name());
+  }
+
   private static MockOddsProvider newProvider(long seed) {
     MockProperties properties =
         new MockProperties(


