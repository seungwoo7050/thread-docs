# 피드 발견·상태 수화·구독 소유권과 재접속

## `feat(cache): store event summaries`

diff --git a/src/main/java/com/sportsbook/oddsfeed/cache/RedisOddsCache.java b/src/main/java/com/sportsbook/oddsfeed/cache/RedisOddsCache.java
index 5c68f1a..b11ba69 100644
--- a/src/main/java/com/sportsbook/oddsfeed/cache/RedisOddsCache.java
+++ b/src/main/java/com/sportsbook/oddsfeed/cache/RedisOddsCache.java
@@ -1,7 +1,9 @@
 package com.sportsbook.oddsfeed.cache;
 
+import com.fasterxml.jackson.core.JsonProcessingException;
 import com.fasterxml.jackson.databind.ObjectMapper;
 import com.sportsbook.oddsfeed.config.CacheProperties;
+import com.sportsbook.oddsfeed.provider.EventSummary;
 import com.sportsbook.protocol.value.EventId;
 import com.sportsbook.protocol.value.MarketId;
 import com.sportsbook.protocol.value.Odds;
@@ -70,6 +72,28 @@ public class RedisOddsCache {
     return value == null ? Optional.empty() : Optional.of(Odds.ofDecimal(value));
   }
 
+  public void storeEvent(EventSummary summary) {
+    try {
+      redis
+          .opsForValue()
+          .set(CacheKeys.event(summary.eventId()), objectMapper.writeValueAsString(summary), ttl);
+    } catch (JsonProcessingException error) {
+      throw new IllegalStateException("Failed to serialize event summary", error);
+    }
+  }
+
+  public Optional<EventSummary> getEvent(EventId eventId) {
+    String json = redis.opsForValue().get(CacheKeys.event(eventId));
+    if (json == null) {
+      return Optional.empty();
+    }
+    try {
+      return Optional.of(objectMapper.readValue(json, EventSummary.class));
+    } catch (JsonProcessingException error) {
+      throw new IllegalStateException("Failed to deserialize event summary", error);
+    }
+  }
+
   private String ttlMillis() {
     return Long.toString(ttl.toMillis());
   }


## `feat(catalog): index current event summaries`

diff --git a/src/main/java/com/sportsbook/oddsfeed/api/EventCatalog.java b/src/main/java/com/sportsbook/oddsfeed/api/EventCatalog.java
new file mode 100644
index 0000000..d6e65d9
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/api/EventCatalog.java
@@ -0,0 +1,40 @@
+package com.sportsbook.oddsfeed.api;
+
+import com.sportsbook.oddsfeed.provider.EventSummary;
+import com.sportsbook.protocol.value.EventId;
+import java.util.Comparator;
+import java.util.List;
+import java.util.Map;
+import java.util.Optional;
+import java.util.concurrent.ConcurrentHashMap;
+import org.springframework.stereotype.Component;
+
+@Component
+public class EventCatalog {
+
+  private final Map<EventId, EventSummary> events = new ConcurrentHashMap<>();
+
+  public void put(EventSummary summary) {
+    events.put(summary.eventId(), summary);
+  }
+
+  public boolean putIfAbsent(EventSummary summary) {
+    return events.putIfAbsent(summary.eventId(), summary) == null;
+  }
+
+  public Optional<EventSummary> get(EventId eventId) {
+    return Optional.ofNullable(events.get(eventId));
+  }
+
+  public List<EventSummary> orderedByKickoff() {
+    return events.values().stream()
+        .sorted(
+            Comparator.comparing(EventSummary::scheduledStartAt)
+                .thenComparing(event -> event.eventId().value()))
+        .toList();
+  }
+
+  public int size() {
+    return events.size();
+  }
+}


## `feat(feed): discover and seed provider events`

diff --git a/src/main/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestrator.java b/src/main/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestrator.java
new file mode 100644
index 0000000..9df7a62
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestrator.java
@@ -0,0 +1,53 @@
+package com.sportsbook.oddsfeed.orchestrator;
+
+import com.sportsbook.oddsfeed.api.EventCatalog;
+import com.sportsbook.oddsfeed.cache.RedisOddsCache;
+import com.sportsbook.oddsfeed.provider.EventSummary;
+import com.sportsbook.oddsfeed.provider.OddsProvider;
+import com.sportsbook.oddsfeed.provider.Sport;
+import jakarta.annotation.PostConstruct;
+import java.util.Optional;
+import java.util.concurrent.TimeUnit;
+import org.springframework.scheduling.annotation.Scheduled;
+import org.springframework.stereotype.Component;
+
+@Component
+public class FeedOrchestrator {
+
+  private final OddsProvider provider;
+  private final RedisOddsCache cache;
+  private final EventCatalog catalog;
+
+  public FeedOrchestrator(OddsProvider provider, RedisOddsCache cache, EventCatalog catalog) {
+    this.provider = provider;
+    this.cache = cache;
+    this.catalog = catalog;
+  }
+
+  @PostConstruct
+  void start() {
+    refresh();
+  }
+
+  @Scheduled(
+      fixedRateString = "${oddsfeed.orchestrator.refresh-interval-seconds:30}",
+      timeUnit = TimeUnit.SECONDS)
+  void refresh() {
+    for (Sport sport : Sport.values()) {
+      for (EventSummary summary : provider.listEvents(sport)) {
+        seedProjection(summary);
+      }
+    }
+  }
+
+  private void seedProjection(EventSummary providerSummary) {
+    if (catalog.get(providerSummary.eventId()).isPresent()) {
+      return;
+    }
+    Optional<EventSummary> cached = cache.getEvent(providerSummary.eventId());
+    EventSummary initial = cached.orElse(providerSummary);
+    if (catalog.putIfAbsent(initial) && cached.isEmpty()) {
+      cache.storeEvent(initial);
+    }
+  }
+}


## `test(feed): verify discovery projections`

diff --git a/src/test/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestratorTest.java b/src/test/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestratorTest.java
new file mode 100644
index 0000000..1f8d5a8
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestratorTest.java
@@ -0,0 +1,96 @@
+package com.sportsbook.oddsfeed.orchestrator;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.oddsfeed.api.EventCatalog;
+import com.sportsbook.oddsfeed.cache.RedisOddsCache;
+import com.sportsbook.oddsfeed.config.CacheProperties;
+import com.sportsbook.oddsfeed.provider.EventSummary;
+import com.sportsbook.oddsfeed.provider.MatchOutcome;
+import com.sportsbook.oddsfeed.provider.OddsProvider;
+import com.sportsbook.oddsfeed.provider.ProviderEvent;
+import com.sportsbook.oddsfeed.provider.Sport;
+import com.sportsbook.protocol.event.EventLifecycleStatus;
+import com.sportsbook.protocol.value.EventId;
+import java.time.Duration;
+import java.time.Instant;
+import java.util.HashMap;
+import java.util.List;
+import java.util.Map;
+import java.util.Optional;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.data.redis.core.StringRedisTemplate;
+import reactor.core.publisher.Flux;
+
+class FeedOrchestratorTest {
+
+  @Test
+  void seedsNewEventsAndPreservesCachedLifecycleState() {
+    EventSummary fresh = event(UUID.randomUUID(), EventLifecycleStatus.SCHEDULED);
+    EventSummary providerCopy = event(UUID.randomUUID(), EventLifecycleStatus.SCHEDULED);
+    EventSummary cached = event(providerCopy.eventId().value(), EventLifecycleStatus.FINISHED);
+    RecordingCache cache = new RecordingCache(Map.of(cached.eventId(), cached));
+    EventCatalog catalog = new EventCatalog();
+    FeedOrchestrator orchestrator =
+        new FeedOrchestrator(new StubProvider(List.of(fresh, providerCopy)), cache, catalog);
+
+    orchestrator.refresh();
+    orchestrator.refresh();
+
+    assertThat(catalog.get(fresh.eventId())).contains(fresh);
+    assertThat(catalog.get(cached.eventId())).contains(cached);
+    assertThat(cache.stores).isEqualTo(1);
+  }
+
+  private static EventSummary event(UUID id, EventLifecycleStatus status) {
+    return new EventSummary(
+        new EventId(id),
+        Sport.FOOTBALL,
+        "Premier League",
+        "Home",
+        "Away",
+        Instant.parse("2026-06-01T18:00:00Z"),
+        status);
+  }
+
+  private record StubProvider(List<EventSummary> events) implements OddsProvider {
+    @Override
+    public List<EventSummary> listEvents(Sport sport) {
+      return sport == Sport.FOOTBALL ? events : List.of();
+    }
+
+    @Override
+    public Flux<ProviderEvent> streamEvents(EventId eventId) {
+      return Flux.empty();
+    }
+
+    @Override
+    public Optional<MatchOutcome> getMatchResult(EventId eventId) {
+      return Optional.empty();
+    }
+  }
+
+  private static final class RecordingCache extends RedisOddsCache {
+    private final Map<EventId, EventSummary> events;
+    private int stores;
+
+    private RecordingCache(Map<EventId, EventSummary> events) {
+      super(
+          new StringRedisTemplate(), new ObjectMapper(), new CacheProperties(Duration.ofHours(1)));
+      this.events = new HashMap<>(events);
+    }
+
+    @Override
+    public Optional<EventSummary> getEvent(EventId eventId) {
+      return Optional.ofNullable(events.get(eventId));
+    }
+
+    @Override
+    public void storeEvent(EventSummary summary) {
+      stores++;
+      events.put(summary.eventId(), summary);
+    }
+  }
+}


## `feat(feed): manage provider subscriptions`

diff --git a/src/main/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestrator.java b/src/main/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestrator.java
index 41369d7..bca61be 100644
--- a/src/main/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestrator.java
+++ b/src/main/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestrator.java
@@ -15,14 +15,20 @@ import com.sportsbook.protocol.event.EventLifecycleStatus;
 import com.sportsbook.protocol.event.MarketStatus;
 import com.sportsbook.protocol.value.EventId;
 import jakarta.annotation.PostConstruct;
+import jakarta.annotation.PreDestroy;
 import java.util.LinkedHashMap;
 import java.util.Map;
 import java.util.Optional;
 import java.util.UUID;
+import java.util.concurrent.ConcurrentHashMap;
 import java.util.concurrent.TimeUnit;
+import java.util.concurrent.atomic.AtomicBoolean;
+import java.util.concurrent.atomic.AtomicReference;
 import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.scheduling.annotation.Scheduled;
 import org.springframework.stereotype.Component;
+import reactor.core.Disposable;
+import reactor.core.publisher.Flux;
 
 @Component
 public class FeedOrchestrator {
@@ -32,6 +38,7 @@ public class FeedOrchestrator {
   private final OddsFeedPublisher publisher;
   private final EventCatalog catalog;
   private final CriticalEventQueue criticalQueue;
+  private final Map<EventId, Disposable> subscriptions = new ConcurrentHashMap<>();
 
   public FeedOrchestrator(OddsProvider provider, RedisOddsCache cache, EventCatalog catalog) {
     this(provider, cache, null, catalog, null);
@@ -64,6 +71,12 @@ public class FeedOrchestrator {
     refresh();
   }
 
+  @PreDestroy
+  void stop() {
+    subscriptions.values().forEach(Disposable::dispose);
+    subscriptions.clear();
+  }
+
   @Scheduled(
       fixedRateString = "${oddsfeed.orchestrator.refresh-interval-seconds:30}",
       timeUnit = TimeUnit.SECONDS)
@@ -71,6 +84,7 @@ public class FeedOrchestrator {
     for (Sport sport : Sport.values()) {
       for (EventSummary summary : provider.listEvents(sport)) {
         seedProjection(summary);
+        ensureSubscribed(summary.eventId());
       }
     }
   }
@@ -86,6 +100,40 @@ public class FeedOrchestrator {
     }
   }
 
+  private void ensureSubscribed(EventId eventId) {
+    Disposable existing = subscriptions.get(eventId);
+    if (existing != null && !existing.isDisposed()) {
+      return;
+    }
+    if (existing != null) {
+      subscriptions.remove(eventId, existing);
+    }
+    AtomicBoolean terminated = new AtomicBoolean();
+    AtomicReference<Disposable> self = new AtomicReference<>();
+    Flux<ProviderEvent> stream =
+        Flux.defer(() -> provider.streamEvents(eventId))
+            .doOnNext(event -> dispatch(eventId, event))
+            .doFinally(
+                signal -> {
+                  terminated.set(true);
+                  Disposable registered = self.get();
+                  if (registered != null) {
+                    subscriptions.remove(eventId, registered);
+                  }
+                });
+    Disposable created = stream.subscribe(ignored -> {}, error -> {});
+    self.set(created);
+    if (created.isDisposed() || terminated.get()) {
+      return;
+    }
+    Disposable raced = subscriptions.putIfAbsent(eventId, created);
+    if (raced != null) {
+      created.dispose();
+    } else if (terminated.get() || created.isDisposed()) {
+      subscriptions.remove(eventId, created);
+    }
+  }
+
   void dispatch(EventId eventId, ProviderEvent event) {
     if (event instanceof ProviderEvent.OddsUpdated odds) {
       handleOdds(odds);


## `test(feed): verify subscription lifecycle`

diff --git a/src/test/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestratorTest.java b/src/test/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestratorTest.java
index 2a25083..163cc9e 100644
--- a/src/test/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestratorTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestratorTest.java
@@ -33,10 +33,12 @@ import java.util.List;
 import java.util.Map;
 import java.util.Optional;
 import java.util.UUID;
+import java.util.concurrent.atomic.AtomicInteger;
 import org.junit.jupiter.api.Test;
 import org.springframework.data.redis.connection.stream.RecordId;
 import org.springframework.data.redis.core.StringRedisTemplate;
 import reactor.core.publisher.Flux;
+import reactor.core.publisher.Sinks;
 
 class FeedOrchestratorTest {
 
@@ -147,6 +149,39 @@ class FeedOrchestratorTest {
     assertThat(failedOrder).containsExactly("snapshot", "enqueue");
   }
 
+  @Test
+  void keepsOneActiveSubscriptionAndReplacesCompletedStreams() {
+    List<String> order = new ArrayList<>();
+    EventSummary summary = event(UUID.randomUUID(), EventLifecycleStatus.SCHEDULED);
+    SubscriptionProvider provider = new SubscriptionProvider(summary);
+    FeedOrchestrator orchestrator =
+        new FeedOrchestrator(
+            provider,
+            new RecordingCache(Map.of(), order),
+            new RecordingPublisher(order),
+            new EventCatalog(),
+            null);
+
+    orchestrator.refresh();
+    orchestrator.refresh();
+    assertThat(provider.subscriptions).hasValue(1);
+
+    provider.events.tryEmitNext(
+        new ProviderEvent.OddsUpdated(
+            summary.eventId(),
+            new MarketId(UUID.randomUUID()),
+            new SelectionId(UUID.randomUUID()),
+            Odds.ofDecimal("2.00"),
+            Odds.ofDecimal("2.10"),
+            Instant.EPOCH));
+    assertThat(order).containsExactly("publish", "cache");
+
+    provider.events.tryEmitComplete();
+    orchestrator.refresh();
+    assertThat(provider.subscriptions).hasValue(2);
+    orchestrator.stop();
+  }
+
   private static void assertMarketOrder(
       MarketStatus next, boolean failEnqueue, String... expected) {
     List<String> order = new ArrayList<>();
@@ -226,6 +261,35 @@ class FeedOrchestratorTest {
     }
   }
 
+  private static final class SubscriptionProvider implements OddsProvider {
+    private final EventSummary summary;
+    private final AtomicInteger subscriptions = new AtomicInteger();
+    private final Sinks.Many<ProviderEvent> events = Sinks.many().replay().limit(1);
+
+    private SubscriptionProvider(EventSummary summary) {
+      this.summary = summary;
+    }
+
+    @Override
+    public List<EventSummary> listEvents(Sport sport) {
+      return sport == summary.sport() ? List.of(summary) : List.of();
+    }
+
+    @Override
+    public Flux<ProviderEvent> streamEvents(EventId eventId) {
+      return Flux.defer(
+          () -> {
+            subscriptions.incrementAndGet();
+            return events.asFlux();
+          });
+    }
+
+    @Override
+    public Optional<MatchOutcome> getMatchResult(EventId eventId) {
+      return Optional.empty();
+    }
+  }
+
   private static final class RecordingCache extends RedisOddsCache {
     private final Map<EventId, EventSummary> events;
     private final List<String> order;


## `feat(feed): retry failed provider streams`

diff --git a/src/main/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestrator.java b/src/main/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestrator.java
index bca61be..7480f67 100644
--- a/src/main/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestrator.java
+++ b/src/main/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestrator.java
@@ -16,6 +16,7 @@ import com.sportsbook.protocol.event.MarketStatus;
 import com.sportsbook.protocol.value.EventId;
 import jakarta.annotation.PostConstruct;
 import jakarta.annotation.PreDestroy;
+import java.time.Duration;
 import java.util.LinkedHashMap;
 import java.util.Map;
 import java.util.Optional;
@@ -29,10 +30,15 @@ import org.springframework.scheduling.annotation.Scheduled;
 import org.springframework.stereotype.Component;
 import reactor.core.Disposable;
 import reactor.core.publisher.Flux;
+import reactor.util.retry.Retry;
 
 @Component
 public class FeedOrchestrator {
 
+  private static final Duration RECONNECT_INITIAL_BACKOFF = Duration.ofSeconds(1);
+  private static final Duration RECONNECT_MAX_BACKOFF = Duration.ofSeconds(30);
+  private static final double RECONNECT_JITTER = 0.2;
+
   private final OddsProvider provider;
   private final RedisOddsCache cache;
   private final OddsFeedPublisher publisher;
@@ -113,6 +119,10 @@ public class FeedOrchestrator {
     Flux<ProviderEvent> stream =
         Flux.defer(() -> provider.streamEvents(eventId))
             .doOnNext(event -> dispatch(eventId, event))
+            .retryWhen(
+                Retry.backoff(Long.MAX_VALUE, RECONNECT_INITIAL_BACKOFF)
+                    .maxBackoff(RECONNECT_MAX_BACKOFF)
+                    .jitter(RECONNECT_JITTER))
             .doFinally(
                 signal -> {
                   terminated.set(true);


## `test(feed): verify bounded retry backoff`

diff --git a/src/test/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestratorTest.java b/src/test/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestratorTest.java
index 163cc9e..de573f1 100644
--- a/src/test/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestratorTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestratorTest.java
@@ -33,6 +33,8 @@ import java.util.List;
 import java.util.Map;
 import java.util.Optional;
 import java.util.UUID;
+import java.util.concurrent.CountDownLatch;
+import java.util.concurrent.TimeUnit;
 import java.util.concurrent.atomic.AtomicInteger;
 import org.junit.jupiter.api.Test;
 import org.springframework.data.redis.connection.stream.RecordId;
@@ -182,6 +184,37 @@ class FeedOrchestratorTest {
     orchestrator.stop();
   }
 
+  @Test
+  void retriesFailedStreamsAfterABoundedBackoff() throws InterruptedException {
+    List<String> order = new ArrayList<>();
+    EventSummary summary = event(UUID.randomUUID(), EventLifecycleStatus.SCHEDULED);
+    ProviderEvent lifecycle =
+        new ProviderEvent.LifecycleUpdated(
+            summary.eventId(),
+            EventLifecycleStatus.CANCELLED,
+            summary.scheduledStartAt(),
+            Instant.EPOCH);
+    RetryProvider provider = new RetryProvider(summary, lifecycle);
+    RetryQueue queue = new RetryQueue(order);
+    FeedOrchestrator orchestrator =
+        new FeedOrchestrator(
+            provider,
+            new RecordingCache(Map.of(), order, Map.of()),
+            new RecordingPublisher(order),
+            new EventCatalog(),
+            queue);
+    long startedAt = System.nanoTime();
+
+    orchestrator.refresh();
+
+    assertThat(queue.delivered.await(3, TimeUnit.SECONDS)).isTrue();
+    long elapsedMillis = Duration.ofNanos(System.nanoTime() - startedAt).toMillis();
+    assertThat(elapsedMillis).isBetween(500L, 3000L);
+    assertThat(provider.subscriptions).hasValue(2);
+    assertThat(order).containsExactly("snapshot", "enqueue", "snapshot", "enqueue", "close");
+    orchestrator.stop();
+  }
+
   private static void assertMarketOrder(
       MarketStatus next, boolean failEnqueue, String... expected) {
     List<String> order = new ArrayList<>();
@@ -290,6 +323,36 @@ class FeedOrchestratorTest {
     }
   }
 
+  private static final class RetryProvider implements OddsProvider {
+    private final EventSummary summary;
+    private final ProviderEvent event;
+    private final AtomicInteger subscriptions = new AtomicInteger();
+
+    private RetryProvider(EventSummary summary, ProviderEvent event) {
+      this.summary = summary;
+      this.event = event;
+    }
+
+    @Override
+    public List<EventSummary> listEvents(Sport sport) {
+      return sport == summary.sport() ? List.of(summary) : List.of();
+    }
+
+    @Override
+    public Flux<ProviderEvent> streamEvents(EventId eventId) {
+      return Flux.defer(
+          () -> {
+            subscriptions.incrementAndGet();
+            return Flux.just(event);
+          });
+    }
+
+    @Override
+    public Optional<MatchOutcome> getMatchResult(EventId eventId) {
+      return Optional.empty();
+    }
+  }
+
   private static final class RecordingCache extends RedisOddsCache {
     private final Map<EventId, EventSummary> events;
     private final List<String> order;
@@ -401,6 +464,31 @@ class FeedOrchestratorTest {
     }
   }
 
+  private static final class RetryQueue extends CriticalEventQueue {
+    private final List<String> order;
+    private final CountDownLatch delivered = new CountDownLatch(1);
+    private int attempts;
+
+    private RetryQueue(List<String> order) {
+      super(
+          new StringRedisTemplate(),
+          new ObjectMapper(),
+          new CriticalDeliveryProperties("stream", "group", "consumer", 1, Duration.ZERO),
+          new SimpleMeterRegistry());
+      this.order = order;
+    }
+
+    @Override
+    public RecordId enqueue(CriticalEvent event) {
+      order.add("enqueue");
+      if (++attempts == 1) {
+        throw new IllegalStateException("Redis unavailable");
+      }
+      delivered.countDown();
+      return RecordId.of("2-0");
+    }
+  }
+
   private static final class RecordingPublisher extends OddsFeedPublisher {
     private final List<String> order;
     private boolean published = true;
