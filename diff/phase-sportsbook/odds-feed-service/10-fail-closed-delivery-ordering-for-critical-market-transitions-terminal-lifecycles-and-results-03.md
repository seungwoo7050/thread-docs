## `test(feed): verify critical market enqueue`

diff --git a/src/test/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestratorTest.java b/src/test/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestratorTest.java
index 7cd29ed..1adadc9 100644
--- a/src/test/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestratorTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestratorTest.java
@@ -1,11 +1,15 @@
 package com.sportsbook.oddsfeed.orchestrator;
 
 import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
 
 import com.fasterxml.jackson.databind.ObjectMapper;
 import com.sportsbook.oddsfeed.api.EventCatalog;
 import com.sportsbook.oddsfeed.cache.RedisOddsCache;
 import com.sportsbook.oddsfeed.config.CacheProperties;
+import com.sportsbook.oddsfeed.config.CriticalDeliveryProperties;
+import com.sportsbook.oddsfeed.delivery.CriticalEvent;
+import com.sportsbook.oddsfeed.delivery.CriticalEventQueue;
 import com.sportsbook.oddsfeed.provider.EventSummary;
 import com.sportsbook.oddsfeed.provider.MatchOutcome;
 import com.sportsbook.oddsfeed.provider.OddsProvider;
@@ -19,6 +23,7 @@ import com.sportsbook.protocol.value.EventId;
 import com.sportsbook.protocol.value.MarketId;
 import com.sportsbook.protocol.value.Odds;
 import com.sportsbook.protocol.value.SelectionId;
+import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
 import java.time.Duration;
 import java.time.Instant;
 import java.util.ArrayList;
@@ -28,6 +33,7 @@ import java.util.Map;
 import java.util.Optional;
 import java.util.UUID;
 import org.junit.jupiter.api.Test;
+import org.springframework.data.redis.connection.stream.RecordId;
 import org.springframework.data.redis.core.StringRedisTemplate;
 import reactor.core.publisher.Flux;
 
@@ -85,6 +91,42 @@ class FeedOrchestratorTest {
     assertOutageOrder(new RecordingPublisher(new ArrayList<>(), true, true), "publish", "hold");
   }
 
+  @Test
+  void enqueuesBeforeRestrictingMarketsAndDefersOpening() {
+    assertMarketOrder(MarketStatus.SUSPENDED, false, "enqueue", "cache");
+    assertMarketOrder(MarketStatus.OPEN, false, "enqueue");
+    assertMarketOrder(MarketStatus.SUSPENDED, true, "enqueue");
+  }
+
+  private static void assertMarketOrder(
+      MarketStatus next, boolean failEnqueue, String... expected) {
+    List<String> order = new ArrayList<>();
+    EventId eventId = new EventId(UUID.randomUUID());
+    FeedOrchestrator orchestrator =
+        new FeedOrchestrator(
+            new StubProvider(List.of()),
+            new RecordingCache(Map.of(), order),
+            new RecordingPublisher(order),
+            new EventCatalog(),
+            new RecordingQueue(order, failEnqueue));
+    ProviderEvent event =
+        new ProviderEvent.MarketStatusUpdated(
+            eventId,
+            new MarketId(UUID.randomUUID()),
+            MarketStatus.OPEN,
+            next,
+            "provider update",
+            Instant.EPOCH);
+
+    if (failEnqueue) {
+      assertThatThrownBy(() -> orchestrator.dispatch(eventId, event))
+          .isInstanceOf(IllegalStateException.class);
+    } else {
+      orchestrator.dispatch(eventId, event);
+    }
+    assertThat(order).containsExactly(expected);
+  }
+
   private static void assertOutageOrder(RecordingPublisher publisher, String... expected) {
     RecordingCache cache = new RecordingCache(Map.of(), publisher.order);
     FeedOrchestrator orchestrator =
@@ -184,6 +226,37 @@ class FeedOrchestratorTest {
       order.add("hold");
       return MarketStatus.SUSPENDED;
     }
+
+    @Override
+    public MarketStatus storeProviderMarketStatus(
+        EventId eventId, MarketId marketId, MarketStatus status) {
+      order.add("cache");
+      return status;
+    }
+  }
+
+  private static final class RecordingQueue extends CriticalEventQueue {
+    private final List<String> order;
+    private final boolean fail;
+
+    private RecordingQueue(List<String> order, boolean fail) {
+      super(
+          new StringRedisTemplate(),
+          new ObjectMapper(),
+          new CriticalDeliveryProperties("stream", "group", "consumer", 1, Duration.ZERO),
+          new SimpleMeterRegistry());
+      this.order = order;
+      this.fail = fail;
+    }
+
+    @Override
+    public RecordId enqueue(CriticalEvent event) {
+      order.add("enqueue");
+      if (fail) {
+        throw new IllegalStateException("Redis unavailable");
+      }
+      return RecordId.of("1-0");
+    }
   }
 
   private static final class RecordingPublisher extends OddsFeedPublisher {


## `feat(feed): snapshot registered terminal markets`

diff --git a/src/main/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestrator.java b/src/main/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestrator.java
index f546978..41369d7 100644
--- a/src/main/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestrator.java
+++ b/src/main/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestrator.java
@@ -5,15 +5,20 @@ import com.sportsbook.oddsfeed.cache.RedisOddsCache;
 import com.sportsbook.oddsfeed.delivery.CriticalEvent;
 import com.sportsbook.oddsfeed.delivery.CriticalEventQueue;
 import com.sportsbook.oddsfeed.provider.EventSummary;
+import com.sportsbook.oddsfeed.provider.MatchOutcome;
 import com.sportsbook.oddsfeed.provider.OddsProvider;
 import com.sportsbook.oddsfeed.provider.ProviderEvent;
 import com.sportsbook.oddsfeed.provider.Sport;
 import com.sportsbook.oddsfeed.publisher.KafkaPublishException;
 import com.sportsbook.oddsfeed.publisher.OddsFeedPublisher;
+import com.sportsbook.protocol.event.EventLifecycleStatus;
 import com.sportsbook.protocol.event.MarketStatus;
 import com.sportsbook.protocol.value.EventId;
 import jakarta.annotation.PostConstruct;
+import java.util.LinkedHashMap;
+import java.util.Map;
 import java.util.Optional;
+import java.util.UUID;
 import java.util.concurrent.TimeUnit;
 import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.scheduling.annotation.Scheduled;
@@ -86,6 +91,8 @@ public class FeedOrchestrator {
       handleOdds(odds);
     } else if (event instanceof ProviderEvent.MarketStatusUpdated status) {
       handleMarketStatus(status);
+    } else if (event instanceof ProviderEvent.LifecycleUpdated lifecycle) {
+      handleLifecycle(lifecycle);
     }
   }
 
@@ -131,4 +138,48 @@ public class FeedOrchestrator {
       cache.storeProviderMarketStatus(status.eventId(), status.marketId(), status.newStatus());
     }
   }
+
+  private void handleLifecycle(ProviderEvent.LifecycleUpdated lifecycle) {
+    if (!isTerminal(lifecycle.status())) {
+      criticalQueue.enqueue(
+          CriticalEvent.lifecycle(
+              lifecycle.eventId(),
+              lifecycle.status(),
+              lifecycle.scheduledStartAt(),
+              lifecycle.occurredAt()));
+      return;
+    }
+    Map<UUID, MarketStatus> terminalMarkets = new LinkedHashMap<>();
+    cache
+        .getRegisteredMarkets(lifecycle.eventId())
+        .forEach(
+            (marketId, status) -> {
+              if (status != MarketStatus.CLOSED) {
+                terminalMarkets.put(marketId.value(), status);
+              }
+            });
+    Optional<MatchOutcome> outcome =
+        lifecycle.status() == EventLifecycleStatus.FINISHED
+            ? provider.getMatchResult(lifecycle.eventId())
+            : Optional.empty();
+    MatchOutcome result = outcome.orElse(null);
+    criticalQueue.enqueue(
+        CriticalEvent.terminalLifecycle(
+            lifecycle.eventId(),
+            lifecycle.status(),
+            lifecycle.scheduledStartAt(),
+            lifecycle.occurredAt(),
+            terminalMarkets,
+            result == null ? null : result.score(),
+            result == null ? null : result.finalStatus(),
+            result == null ? Map.of() : result.detail(),
+            result == null ? null : result.settledAt()));
+    cache.closeEventMarkets(lifecycle.eventId(), lifecycle.status());
+  }
+
+  private static boolean isTerminal(EventLifecycleStatus status) {
+    return status == EventLifecycleStatus.FINISHED
+        || status == EventLifecycleStatus.CANCELLED
+        || status == EventLifecycleStatus.POSTPONED;
+  }
 }


## `test(feed): verify terminal market snapshots`

diff --git a/src/test/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestratorTest.java b/src/test/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestratorTest.java
index 1adadc9..2a25083 100644
--- a/src/test/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestratorTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestratorTest.java
@@ -19,6 +19,7 @@ import com.sportsbook.oddsfeed.publisher.KafkaPublishException;
 import com.sportsbook.oddsfeed.publisher.OddsFeedPublisher;
 import com.sportsbook.protocol.event.EventLifecycleStatus;
 import com.sportsbook.protocol.event.MarketStatus;
+import com.sportsbook.protocol.event.MatchFinalStatus;
 import com.sportsbook.protocol.value.EventId;
 import com.sportsbook.protocol.value.MarketId;
 import com.sportsbook.protocol.value.Odds;
@@ -98,6 +99,54 @@ class FeedOrchestratorTest {
     assertMarketOrder(MarketStatus.SUSPENDED, true, "enqueue");
   }
 
+  @Test
+  void snapshotsTerminalMarketsBeforeEnqueueAndClosure() {
+    List<String> order = new ArrayList<>();
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId open = new MarketId(UUID.randomUUID());
+    MarketId closed = new MarketId(UUID.randomUUID());
+    MatchOutcome outcome =
+        new MatchOutcome(
+            eventId, "2-1", MatchFinalStatus.COMPLETED, Map.of("winner", "home"), Instant.EPOCH);
+    RecordingCache cache =
+        new RecordingCache(
+            Map.of(), order, Map.of(open, MarketStatus.OPEN, closed, MarketStatus.CLOSED));
+    RecordingQueue queue = new RecordingQueue(order, false);
+    FeedOrchestrator orchestrator =
+        new FeedOrchestrator(
+            new StubProvider(List.of(), Optional.of(outcome)),
+            cache,
+            new RecordingPublisher(order),
+            new EventCatalog(),
+            queue);
+
+    orchestrator.dispatch(
+        eventId,
+        new ProviderEvent.LifecycleUpdated(
+            eventId, EventLifecycleStatus.FINISHED, Instant.EPOCH, Instant.EPOCH));
+
+    assertThat(order).containsExactly("snapshot", "enqueue", "close");
+    assertThat(queue.enqueued.terminalMarkets()).containsOnlyKeys(open.value());
+    assertThat(queue.enqueued.score()).isEqualTo("2-1");
+
+    List<String> failedOrder = new ArrayList<>();
+    FeedOrchestrator failing =
+        new FeedOrchestrator(
+            new StubProvider(List.of()),
+            new RecordingCache(Map.of(), failedOrder, Map.of(open, MarketStatus.OPEN)),
+            new RecordingPublisher(failedOrder),
+            new EventCatalog(),
+            new RecordingQueue(failedOrder, true));
+    assertThatThrownBy(
+            () ->
+                failing.dispatch(
+                    eventId,
+                    new ProviderEvent.LifecycleUpdated(
+                        eventId, EventLifecycleStatus.CANCELLED, Instant.EPOCH, Instant.EPOCH)))
+        .isInstanceOf(IllegalStateException.class);
+    assertThat(failedOrder).containsExactly("snapshot", "enqueue");
+  }
+
   private static void assertMarketOrder(
       MarketStatus next, boolean failEnqueue, String... expected) {
     List<String> order = new ArrayList<>();
@@ -155,7 +204,12 @@ class FeedOrchestratorTest {
         status);
   }
 
-  private record StubProvider(List<EventSummary> events) implements OddsProvider {
+  private record StubProvider(List<EventSummary> events, Optional<MatchOutcome> result)
+      implements OddsProvider {
+    private StubProvider(List<EventSummary> events) {
+      this(events, Optional.empty());
+    }
+
     @Override
     public List<EventSummary> listEvents(Sport sport) {
       return sport == Sport.FOOTBALL ? events : List.of();
@@ -168,7 +222,7 @@ class FeedOrchestratorTest {
 
     @Override
     public Optional<MatchOutcome> getMatchResult(EventId eventId) {
-      return Optional.empty();
+      return result;
     }
   }
 
@@ -176,6 +230,7 @@ class FeedOrchestratorTest {
     private final Map<EventId, EventSummary> events;
     private final List<String> order;
     private boolean held;
+    private final Map<MarketId, MarketStatus> markets;
     private int stores;
 
     private RecordingCache(Map<EventId, EventSummary> events) {
@@ -183,10 +238,18 @@ class FeedOrchestratorTest {
     }
 
     private RecordingCache(Map<EventId, EventSummary> events, List<String> order) {
+      this(events, order, Map.of());
+    }
+
+    private RecordingCache(
+        Map<EventId, EventSummary> events,
+        List<String> order,
+        Map<MarketId, MarketStatus> markets) {
       super(
           new StringRedisTemplate(), new ObjectMapper(), new CacheProperties(Duration.ofHours(1)));
       this.events = new HashMap<>(events);
       this.order = order;
+      this.markets = markets;
     }
 
     @Override
@@ -233,11 +296,25 @@ class FeedOrchestratorTest {
       order.add("cache");
       return status;
     }
+
+    @Override
+    public Map<MarketId, MarketStatus> getRegisteredMarkets(EventId eventId) {
+      order.add("snapshot");
+      return markets;
+    }
+
+    @Override
+    public Map<UUID, MarketStatus> closeEventMarkets(
+        EventId eventId, EventLifecycleStatus terminalStatus) {
+      order.add("close");
+      return Map.of();
+    }
   }
 
   private static final class RecordingQueue extends CriticalEventQueue {
     private final List<String> order;
     private final boolean fail;
+    private CriticalEvent enqueued;
 
     private RecordingQueue(List<String> order, boolean fail) {
       super(
@@ -252,6 +329,7 @@ class FeedOrchestratorTest {
     @Override
     public RecordId enqueue(CriticalEvent event) {
       order.add("enqueue");
+      enqueued = event;
       if (fail) {
         throw new IllegalStateException("Redis unavailable");
       }


## `test(feed): verify restart terminal closure`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessorTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessorTest.java
index a4e0206..3681073 100644
--- a/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessorTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessorTest.java
@@ -189,6 +189,50 @@ class CriticalEventProcessorTest {
     order.verify(queue).acknowledge(queued);
   }
 
+  @Test
+  void reclaimedTerminalRecordClosesMarketsFromTheDurableRegistry() {
+    CriticalEventQueue queue = mock(CriticalEventQueue.class);
+    OddsFeedPublisher publisher = mock(OddsFeedPublisher.class);
+    RedisOddsCache cache = mock(RedisOddsCache.class);
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId recoveredMarket = new MarketId(UUID.randomUUID());
+    CriticalEvent event =
+        CriticalEvent.terminalLifecycle(
+            eventId,
+            EventLifecycleStatus.CANCELLED,
+            Instant.EPOCH,
+            Instant.EPOCH,
+            Map.of(),
+            null,
+            null,
+            Map.of(),
+            null);
+    QueuedCriticalEvent queued = new QueuedCriticalEvent(RecordId.of("3-1"), event, true);
+    when(queue.poll()).thenReturn(List.of(queued));
+    when(cache.closeEventMarkets(eventId, EventLifecycleStatus.CANCELLED))
+        .thenReturn(Map.of(recoveredMarket.value(), MarketStatus.OPEN));
+    when(cache.getEvent(eventId)).thenReturn(Optional.empty());
+
+    new CriticalEventProcessor(queue, publisher, cache, new EventCatalog()).drain();
+
+    InOrder order = inOrder(cache, publisher, queue);
+    order.verify(cache).closeEventMarkets(eventId, EventLifecycleStatus.CANCELLED);
+    order
+        .verify(publisher)
+        .publishEventLifecycle(
+            eventId, EventLifecycleStatus.CANCELLED, Instant.EPOCH, Instant.EPOCH);
+    order
+        .verify(publisher)
+        .publishMarketStatusChanged(
+            eventId,
+            recoveredMarket,
+            MarketStatus.OPEN,
+            MarketStatus.CLOSED,
+            "EVENT_CANCELLED",
+            Instant.EPOCH);
+    order.verify(queue).acknowledge(queued);
+  }
+
   @Test
   void publishesDirectMatchResultBeforeAcknowledgement() {
     CriticalEventQueue queue = mock(CriticalEventQueue.class);
