## `test(real): verify event listing and quota use`

diff --git a/src/test/java/com/sportsbook/oddsfeed/provider/real/TheOddsApiProviderTest.java b/src/test/java/com/sportsbook/oddsfeed/provider/real/TheOddsApiProviderTest.java
new file mode 100644
index 0000000..cfd9b10
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/provider/real/TheOddsApiProviderTest.java
@@ -0,0 +1,80 @@
+package com.sportsbook.oddsfeed.provider.real;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.oddsfeed.config.RealProperties;
+import com.sportsbook.oddsfeed.provider.Sport;
+import com.sportsbook.protocol.event.EventLifecycleStatus;
+import java.time.Clock;
+import java.time.Instant;
+import java.time.ZoneOffset;
+import java.util.List;
+import org.junit.jupiter.api.Test;
+import org.springframework.http.HttpHeaders;
+import org.springframework.http.HttpStatus;
+import org.springframework.http.MediaType;
+import org.springframework.web.reactive.function.client.ClientResponse;
+import org.springframework.web.reactive.function.client.WebClient;
+import reactor.core.publisher.Mono;
+
+class TheOddsApiProviderTest {
+
+  private static final String EVENTS =
+      """
+      [{
+        "id": "abc123",
+        "sport_key": "soccer_epl",
+        "sport_title": "EPL",
+        "commence_time": "2026-06-01T18:00:00Z",
+        "home_team": "Manchester United",
+        "away_team": "Chelsea",
+        "bookmakers": []
+      }]
+      """;
+
+  @Test
+  void listsConfiguredSportAndConsumesOneQuotaUnit() {
+    RecordingQuota quota = new RecordingQuota();
+    WebClient client =
+        WebClient.builder()
+            .exchangeFunction(
+                request ->
+                    Mono.just(
+                        ClientResponse.create(HttpStatus.OK)
+                            .header(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
+                            .body(EVENTS)
+                            .build()))
+            .build();
+    var properties =
+        new RealProperties(
+            "key",
+            "https://odds.example",
+            List.of("soccer_epl"),
+            new RealProperties.RateLimit(5),
+            500,
+            60);
+    var clock = Clock.fixed(Instant.parse("2026-05-28T10:00:00Z"), ZoneOffset.UTC);
+    var provider = new TheOddsApiProvider(client, properties, new RateLimiter(5, clock), quota);
+
+    var events = provider.listEvents(Sport.FOOTBALL);
+
+    assertThat(events).hasSize(1);
+    assertThat(events.get(0).competition()).isEqualTo("EPL");
+    assertThat(events.get(0).status()).isEqualTo(EventLifecycleStatus.SCHEDULED);
+    assertThat(quota.current()).isEqualTo(1);
+  }
+
+  private static final class RecordingQuota implements QuotaCounter {
+    private long used;
+
+    @Override
+    public long increment() {
+      return ++used;
+    }
+
+    @Override
+    public long current() {
+      return used;
+    }
+  }
+}


## `feat(real): derive stable market identities`

diff --git a/src/main/java/com/sportsbook/oddsfeed/provider/real/TheOddsApiProvider.java b/src/main/java/com/sportsbook/oddsfeed/provider/real/TheOddsApiProvider.java
index 86d6f5c..03a9e3a 100644
--- a/src/main/java/com/sportsbook/oddsfeed/provider/real/TheOddsApiProvider.java
+++ b/src/main/java/com/sportsbook/oddsfeed/provider/real/TheOddsApiProvider.java
@@ -5,6 +5,8 @@ import com.sportsbook.oddsfeed.provider.EventSummary;
 import com.sportsbook.oddsfeed.provider.Sport;
 import com.sportsbook.protocol.event.EventLifecycleStatus;
 import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.MarketId;
+import com.sportsbook.protocol.value.SelectionId;
 import java.nio.charset.StandardCharsets;
 import java.time.Duration;
 import java.util.List;
@@ -75,9 +77,8 @@ public class TheOddsApiProvider {
   }
 
   private EventSummary toSummary(TheOddsApiDtos.Event event, Sport sport) {
-    EventId id = new EventId(UUID.nameUUIDFromBytes(event.id().getBytes(StandardCharsets.UTF_8)));
     return new EventSummary(
-        id,
+        deriveEventId(event.id()),
         sport,
         event.sportTitle(),
         event.homeTeam(),
@@ -85,4 +86,23 @@ public class TheOddsApiProvider {
         event.commenceTime(),
         EventLifecycleStatus.SCHEDULED);
   }
+
+  static EventId deriveEventId(String upstreamId) {
+    return new EventId(named(upstreamId));
+  }
+
+  static MarketId deriveMarketId(EventId eventId, String marketKey) {
+    return new MarketId(named(eventId.value() + ":" + marketKey));
+  }
+
+  static SelectionId deriveSelectionId(EventId eventId, SelectionKey selection) {
+    return new SelectionId(
+        named(eventId.value() + ":" + selection.marketKey() + ":" + selection.outcomeName()));
+  }
+
+  private static UUID named(String value) {
+    return UUID.nameUUIDFromBytes(value.getBytes(StandardCharsets.UTF_8));
+  }
+
+  record SelectionKey(String marketKey, String outcomeName) {}
 }


## `test(real): verify stable provider identities`

diff --git a/src/test/java/com/sportsbook/oddsfeed/provider/real/TheOddsApiProviderTest.java b/src/test/java/com/sportsbook/oddsfeed/provider/real/TheOddsApiProviderTest.java
index cfd9b10..8cea282 100644
--- a/src/test/java/com/sportsbook/oddsfeed/provider/real/TheOddsApiProviderTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/provider/real/TheOddsApiProviderTest.java
@@ -64,6 +64,21 @@ class TheOddsApiProviderTest {
     assertThat(quota.current()).isEqualTo(1);
   }
 
+  @Test
+  void identitiesRemainStableForTheSameProviderKeys() {
+    var eventId = TheOddsApiProvider.deriveEventId("abc123");
+    var repeatedEventId = TheOddsApiProvider.deriveEventId("abc123");
+    var marketId = TheOddsApiProvider.deriveMarketId(eventId, "h2h");
+    var selection = new TheOddsApiProvider.SelectionKey("h2h", "Chelsea");
+
+    assertThat(eventId).isEqualTo(repeatedEventId);
+    assertThat(marketId).isEqualTo(TheOddsApiProvider.deriveMarketId(eventId, "h2h"));
+    assertThat(TheOddsApiProvider.deriveSelectionId(eventId, selection))
+        .isEqualTo(TheOddsApiProvider.deriveSelectionId(repeatedEventId, selection));
+    assertThat(TheOddsApiProvider.deriveSelectionId(eventId, selection).value())
+        .isNotEqualTo(marketId.value());
+  }
+
   private static final class RecordingQuota implements QuotaCounter {
     private long used;
 


## `feat(real): stream changed market snapshots`

diff --git a/src/main/java/com/sportsbook/oddsfeed/provider/real/TheOddsApiProvider.java b/src/main/java/com/sportsbook/oddsfeed/provider/real/TheOddsApiProvider.java
index 03a9e3a..8752332 100644
--- a/src/main/java/com/sportsbook/oddsfeed/provider/real/TheOddsApiProvider.java
+++ b/src/main/java/com/sportsbook/oddsfeed/provider/real/TheOddsApiProvider.java
@@ -2,22 +2,34 @@ package com.sportsbook.oddsfeed.provider.real;
 
 import com.sportsbook.oddsfeed.config.RealProperties;
 import com.sportsbook.oddsfeed.provider.EventSummary;
+import com.sportsbook.oddsfeed.provider.MatchOutcome;
+import com.sportsbook.oddsfeed.provider.OddsProvider;
+import com.sportsbook.oddsfeed.provider.ProviderEvent;
 import com.sportsbook.oddsfeed.provider.Sport;
 import com.sportsbook.protocol.event.EventLifecycleStatus;
 import com.sportsbook.protocol.value.EventId;
 import com.sportsbook.protocol.value.MarketId;
+import com.sportsbook.protocol.value.Odds;
 import com.sportsbook.protocol.value.SelectionId;
 import java.nio.charset.StandardCharsets;
 import java.time.Duration;
+import java.time.Instant;
+import java.util.LinkedHashMap;
 import java.util.List;
+import java.util.Map;
+import java.util.Optional;
 import java.util.UUID;
+import java.util.concurrent.ConcurrentHashMap;
 import org.springframework.context.annotation.Profile;
+import org.springframework.scheduling.annotation.Scheduled;
 import org.springframework.stereotype.Component;
 import org.springframework.web.reactive.function.client.WebClient;
+import reactor.core.publisher.Flux;
+import reactor.core.publisher.Sinks;
 
 @Component
 @Profile("real")
-public class TheOddsApiProvider {
+public class TheOddsApiProvider implements OddsProvider {
 
   private static final Duration FETCH_TIMEOUT = Duration.ofSeconds(10);
 
@@ -25,6 +37,8 @@ public class TheOddsApiProvider {
   private final RealProperties properties;
   private final RateLimiter rateLimiter;
   private final QuotaCounter quotaCounter;
+  private final Map<EventId, Sinks.Many<ProviderEvent>> streams = new ConcurrentHashMap<>();
+  private final Map<EventId, Map<SelectionKey, Odds>> lastSeen = new ConcurrentHashMap<>();
 
   public TheOddsApiProvider(
       WebClient theOddsWebClient,
@@ -37,6 +51,7 @@ public class TheOddsApiProvider {
     this.quotaCounter = quotaCounter;
   }
 
+  @Override
   public List<EventSummary> listEvents(Sport sport) {
     String sportKey = sportKey(sport);
     if (sportKey == null) {
@@ -45,6 +60,42 @@ public class TheOddsApiProvider {
     return fetch(sportKey).stream().map(event -> toSummary(event, sport)).toList();
   }
 
+  @Override
+  public Flux<ProviderEvent> streamEvents(EventId eventId) {
+    return streams
+        .computeIfAbsent(eventId, ignored -> Sinks.many().multicast().onBackpressureBuffer())
+        .asFlux();
+  }
+
+  @Override
+  public Optional<MatchOutcome> getMatchResult(EventId eventId) {
+    return Optional.empty();
+  }
+
+  @Scheduled(
+      fixedRateString = "${oddsfeed.real.poll-interval-seconds:60}",
+      timeUnit = java.util.concurrent.TimeUnit.SECONDS)
+  void scheduledPoll() {
+    for (Sport sport : Sport.values()) {
+      pollSport(sport);
+    }
+  }
+
+  void pollSport(Sport sport) {
+    String sportKey = sportKey(sport);
+    if (sportKey == null) {
+      return;
+    }
+    for (TheOddsApiDtos.Event event : fetch(sportKey)) {
+      EventId eventId = deriveEventId(event.id());
+      Map<SelectionKey, Odds> next = prices(event);
+      Map<SelectionKey, Odds> previous = lastSeen.put(eventId, next);
+      if (previous != null) {
+        emitChanges(eventId, previous, next, observedAt(event));
+      }
+    }
+  }
+
   private List<TheOddsApiDtos.Event> fetch(String sportKey) {
     if (!rateLimiter.tryAcquire() || quotaCounter.increment() > properties.monthlyQuota()) {
       return List.of();
@@ -87,6 +138,53 @@ public class TheOddsApiProvider {
         EventLifecycleStatus.SCHEDULED);
   }
 
+  private Map<SelectionKey, Odds> prices(TheOddsApiDtos.Event event) {
+    Map<SelectionKey, Odds> prices = new LinkedHashMap<>();
+    if (event.bookmakers() == null || event.bookmakers().isEmpty()) {
+      return prices;
+    }
+    for (TheOddsApiDtos.Market market : event.bookmakers().get(0).markets()) {
+      if ("h2h".equals(market.key())) {
+        for (TheOddsApiDtos.Outcome outcome : market.outcomes()) {
+          prices.put(
+              new SelectionKey(market.key(), outcome.name()), Odds.ofDecimal(outcome.price()));
+        }
+      }
+    }
+    return prices;
+  }
+
+  private Instant observedAt(TheOddsApiDtos.Event event) {
+    return event.bookmakers().isEmpty()
+        ? event.commenceTime()
+        : event.bookmakers().get(0).lastUpdate();
+  }
+
+  private void emitChanges(
+      EventId eventId,
+      Map<SelectionKey, Odds> previous,
+      Map<SelectionKey, Odds> next,
+      Instant observedAt) {
+    Sinks.Many<ProviderEvent> sink = streams.get(eventId);
+    if (sink == null) {
+      return;
+    }
+    next.forEach(
+        (key, odds) -> {
+          Odds prior = previous.get(key);
+          if (!odds.equals(prior)) {
+            sink.tryEmitNext(
+                new ProviderEvent.OddsUpdated(
+                    eventId,
+                    deriveMarketId(eventId, key.marketKey()),
+                    deriveSelectionId(eventId, key),
+                    prior == null ? odds : prior,
+                    odds,
+                    observedAt));
+          }
+        });
+  }
+
   static EventId deriveEventId(String upstreamId) {
     return new EventId(named(upstreamId));
   }


## `test(real): verify changed-only polling`

diff --git a/src/test/java/com/sportsbook/oddsfeed/provider/real/TheOddsApiProviderTest.java b/src/test/java/com/sportsbook/oddsfeed/provider/real/TheOddsApiProviderTest.java
index 8cea282..a641505 100644
--- a/src/test/java/com/sportsbook/oddsfeed/provider/real/TheOddsApiProviderTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/provider/real/TheOddsApiProviderTest.java
@@ -8,6 +8,7 @@ import com.sportsbook.protocol.event.EventLifecycleStatus;
 import java.time.Clock;
 import java.time.Instant;
 import java.time.ZoneOffset;
+import java.util.ArrayList;
 import java.util.List;
 import org.junit.jupiter.api.Test;
 import org.springframework.http.HttpHeaders;
@@ -79,6 +80,66 @@ class TheOddsApiProviderTest {
         .isNotEqualTo(marketId.value());
   }
 
+  @Test
+  void pollingPublishesOnlyChangedSelectionPrices() {
+    var body =
+        new java.util.concurrent.atomic.AtomicReference<>(odds("1.85", "3.60", "4.20", "10:00"));
+    WebClient client =
+        WebClient.builder()
+            .exchangeFunction(
+                request ->
+                    Mono.just(
+                        ClientResponse.create(HttpStatus.OK)
+                            .header(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
+                            .body(body.get())
+                            .build()))
+            .build();
+    var properties =
+        new RealProperties(
+            "key",
+            "https://odds.example",
+            List.of("soccer_epl"),
+            new RealProperties.RateLimit(10),
+            500,
+            60);
+    var clock = Clock.fixed(Instant.parse("2026-05-28T10:00:00Z"), ZoneOffset.UTC);
+    var provider =
+        new TheOddsApiProvider(
+            client, properties, new RateLimiter(10, clock), new RecordingQuota());
+    provider.pollSport(Sport.FOOTBALL);
+    var eventId = TheOddsApiProvider.deriveEventId("abc123");
+    var updates = new ArrayList<com.sportsbook.oddsfeed.provider.ProviderEvent.OddsUpdated>();
+    var subscription =
+        provider
+            .streamEvents(eventId)
+            .subscribe(
+                event ->
+                    updates.add(
+                        (com.sportsbook.oddsfeed.provider.ProviderEvent.OddsUpdated) event));
+
+    body.set(odds("1.90", "3.60", "4.00", "10:05"));
+    provider.pollSport(Sport.FOOTBALL);
+    provider.pollSport(Sport.FOOTBALL);
+    subscription.dispose();
+
+    assertThat(updates).hasSize(2);
+    assertThat(updates)
+        .extracting(update -> update.newOdds().decimal().toPlainString())
+        .containsExactlyInAnyOrder("1.9000", "4.0000");
+  }
+
+  private static String odds(String home, String draw, String away, String minute) {
+    return """
+        [{"id":"abc123","sport_key":"soccer_epl","sport_title":"EPL",
+          "commence_time":"2026-06-01T18:00:00Z","home_team":"Home","away_team":"Away",
+          "bookmakers":[{"key":"book","title":"Book","last_update":"2026-05-28T%s:00Z",
+          "markets":[{"key":"h2h","last_update":"2026-05-28T%s:00Z","outcomes":[
+          {"name":"Home","price":%s},{"name":"Draw","price":%s},
+          {"name":"Away","price":%s}]}]}]}]
+        """
+        .formatted(minute, minute, home, draw, away);
+  }
+
   private static final class RecordingQuota implements QuotaCounter {
     private long used;
 
