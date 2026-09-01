## `test(delivery): publish effective provider restrictions`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessorTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessorTest.java
index 497a554..0404c0e 100644
--- a/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessorTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessorTest.java
@@ -121,6 +121,16 @@ class CriticalEventProcessorTest {
     assertRestrictivePreviewSuppressesOpen("feed hold");
   }
 
+  @Test
+  void terminalStatePublishesAnEffectiveCloseForProviderSuspension() {
+    assertProviderSuspensionPublishesEffectiveClose("terminal");
+  }
+
+  @Test
+  void operatorClosePublishesAnEffectiveCloseForProviderSuspension() {
+    assertProviderSuspensionPublishesEffectiveClose("operator close");
+  }
+
   private static void assertRestrictivePreviewSuppressesOpen(String reason) {
     CriticalEventQueue queue = mock(CriticalEventQueue.class);
     OddsFeedPublisher publisher = mock(OddsFeedPublisher.class);
@@ -142,6 +152,31 @@ class CriticalEventProcessorTest {
     verify(queue).acknowledge(queued);
   }
 
+  private static void assertProviderSuspensionPublishesEffectiveClose(String reason) {
+    CriticalEventQueue queue = mock(CriticalEventQueue.class);
+    OddsFeedPublisher publisher = mock(OddsFeedPublisher.class);
+    RedisOddsCache cache = mock(RedisOddsCache.class);
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+    CriticalEvent event =
+        CriticalEvent.marketStatus(
+            eventId, marketId, MarketStatus.OPEN, MarketStatus.SUSPENDED, reason, Instant.EPOCH);
+    QueuedCriticalEvent queued = new QueuedCriticalEvent(RecordId.of("2-3"), event, false);
+    when(queue.poll()).thenReturn(List.of(queued));
+    when(cache.storeProviderMarketStatus(eventId, marketId, MarketStatus.SUSPENDED))
+        .thenReturn(MarketStatus.CLOSED);
+
+    new CriticalEventProcessor(queue, publisher, cache, new EventCatalog()).drain();
+
+    InOrder order = inOrder(cache, publisher, queue);
+    order.verify(cache).storeProviderMarketStatus(eventId, marketId, MarketStatus.SUSPENDED);
+    order
+        .verify(publisher)
+        .publishMarketStatusChanged(
+            eventId, marketId, MarketStatus.OPEN, MarketStatus.CLOSED, reason, Instant.EPOCH);
+    order.verify(queue).acknowledge(queued);
+  }
+
   private static final class RecoveringProcessor extends CriticalEventProcessor {
     private int attempts;
 


## `test(cache): verify provider open previews`

diff --git a/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheIntegrationTest.java b/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheIntegrationTest.java
index b40bcd5..73c1696 100644
--- a/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheIntegrationTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheIntegrationTest.java
@@ -195,6 +195,51 @@ class RedisOddsCacheIntegrationTest {
     assertThat(redis.getExpire(CacheKeys.marketTerminal(eventId, marketId))).isEqualTo(-1);
   }
 
+  @Test
+  void restrictiveOpenPreviewsRegisterUnknownMarketsWithoutPublicOpening() {
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId overriddenMarket = new MarketId(UUID.randomUUID());
+    MarketId heldMarket = new MarketId(UUID.randomUUID());
+    RedisOddsCache cache = cache();
+    redis
+        .opsForValue()
+        .set(CacheKeys.marketOverride(eventId, overriddenMarket), MarketStatus.CLOSED.name());
+    redis.opsForValue().set(CacheKeys.marketFeedHold(eventId, heldMarket), "1750000000000");
+
+    assertThat(cache.prepareProviderOpen(eventId, overriddenMarket)).isEqualTo(MarketStatus.CLOSED);
+    assertThat(cache.prepareProviderOpen(eventId, heldMarket)).isEqualTo(MarketStatus.SUSPENDED);
+
+    assertThat(redis.opsForValue().get(CacheKeys.providerMarket(eventId, overriddenMarket)))
+        .isEqualTo(MarketStatus.OPEN.name());
+    assertThat(redis.opsForValue().get(CacheKeys.providerMarket(eventId, heldMarket)))
+        .isEqualTo(MarketStatus.OPEN.name());
+    assertThat(cache.getRegisteredMarkets(eventId))
+        .containsEntry(overriddenMarket, MarketStatus.CLOSED)
+        .containsEntry(heldMarket, MarketStatus.SUSPENDED);
+    assertThat(redis.getExpire(CacheKeys.eventMarkets(eventId))).isPositive();
+  }
+
+  @Test
+  void providerClosedPreviewCreatesAPermanentLatchWithoutReopening() {
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+    RedisOddsCache cache = cache();
+    redis
+        .opsForValue()
+        .set(
+            CacheKeys.providerMarket(eventId, marketId),
+            MarketStatus.CLOSED.name(),
+            Duration.ofHours(1));
+
+    assertThat(cache.isMarketTerminal(eventId, marketId)).isFalse();
+    assertThat(cache.prepareProviderOpen(eventId, marketId)).isEqualTo(MarketStatus.CLOSED);
+
+    assertThat(cache.isMarketTerminal(eventId, marketId)).isTrue();
+    assertThat(redis.getExpire(CacheKeys.marketTerminal(eventId, marketId))).isEqualTo(-1);
+    assertThat(redis.opsForValue().get(CacheKeys.providerMarket(eventId, marketId)))
+        .isEqualTo(MarketStatus.CLOSED.name());
+  }
+
   private RedisOddsCache cache() {
     return new RedisOddsCache(
         redis,


## `feat(delivery): deliver lifecycle transitions`

diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessor.java b/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessor.java
index 1ba5cfd..7cb536d 100644
--- a/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessor.java
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessor.java
@@ -2,10 +2,15 @@ package com.sportsbook.oddsfeed.delivery;
 
 import com.sportsbook.oddsfeed.api.EventCatalog;
 import com.sportsbook.oddsfeed.cache.RedisOddsCache;
+import com.sportsbook.oddsfeed.provider.EventSummary;
 import com.sportsbook.oddsfeed.publisher.OddsFeedPublisher;
+import com.sportsbook.protocol.event.EventLifecycleStatus;
 import com.sportsbook.protocol.event.MarketStatus;
 import com.sportsbook.protocol.value.EventId;
 import com.sportsbook.protocol.value.MarketId;
+import java.util.LinkedHashMap;
+import java.util.Map;
+import java.util.UUID;
 import java.util.concurrent.atomic.AtomicBoolean;
 import org.springframework.scheduling.annotation.Scheduled;
 import org.springframework.stereotype.Component;
@@ -49,9 +54,18 @@ public class CriticalEventProcessor {
   }
 
   void apply(CriticalEvent event) {
-    if (event.type() != CriticalEvent.Type.MARKET_STATUS) {
-      throw new IllegalStateException("Unsupported critical event type: " + event.type());
+    if (event.type() == CriticalEvent.Type.EVENT_LIFECYCLE) {
+      applyLifecycle(event);
+      return;
+    }
+    if (event.type() == CriticalEvent.Type.MARKET_STATUS) {
+      applyMarketTransition(event);
+      return;
     }
+    throw new IllegalStateException("Unsupported critical event type: " + event.type());
+  }
+
+  private void applyMarketTransition(CriticalEvent event) {
     EventId eventId = new EventId(event.eventId());
     MarketId marketId = new MarketId(event.marketId());
     if (event.nextMarketStatus() == MarketStatus.OPEN) {
@@ -67,6 +81,54 @@ public class CriticalEventProcessor {
     publishMarketTransition(event, eventId, marketId, effective);
   }
 
+  private void applyLifecycle(CriticalEvent event) {
+    EventId eventId = new EventId(event.eventId());
+    Map<UUID, MarketStatus> terminalMarkets = new LinkedHashMap<>();
+    event
+        .terminalMarkets()
+        .forEach(
+            (marketId, status) -> {
+              if (status != MarketStatus.CLOSED) {
+                terminalMarkets.put(marketId, status);
+              }
+            });
+    if (isTerminal(event.lifecycleStatus())) {
+      cache
+          .closeEventMarkets(eventId, event.lifecycleStatus())
+          .forEach(terminalMarkets::putIfAbsent);
+    }
+    if (event.matchFinalStatus() != null) {
+      throw new IllegalStateException("Embedded match results are not deliverable yet");
+    }
+    publisher.publishEventLifecycle(
+        eventId, event.lifecycleStatus(), event.scheduledStartAt(), event.occurredAt());
+    cache
+        .getEvent(eventId)
+        .ifPresent(
+            current -> {
+              EventSummary updated =
+                  new EventSummary(
+                      current.eventId(),
+                      current.sport(),
+                      current.competition(),
+                      current.homeTeam(),
+                      current.awayTeam(),
+                      current.scheduledStartAt(),
+                      event.lifecycleStatus());
+              cache.storeEvent(updated);
+              catalog.put(updated);
+            });
+    terminalMarkets.forEach(
+        (market, previous) ->
+            publisher.publishMarketStatusChanged(
+                eventId,
+                new MarketId(market),
+                previous,
+                MarketStatus.CLOSED,
+                "EVENT_" + event.lifecycleStatus(),
+                event.occurredAt()));
+  }
+
   private void publishMarketTransition(
       CriticalEvent event, EventId eventId, MarketId marketId, MarketStatus effectiveStatus) {
     publisher.publishMarketStatusChanged(
@@ -78,6 +140,12 @@ public class CriticalEventProcessor {
         event.occurredAt());
   }
 
+  private static boolean isTerminal(EventLifecycleStatus status) {
+    return status == EventLifecycleStatus.FINISHED
+        || status == EventLifecycleStatus.CANCELLED
+        || status == EventLifecycleStatus.POSTPONED;
+  }
+
   public boolean isHealthy() {
     return healthy.get();
   }


## `test(delivery): verify terminal delivery ordering`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessorTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessorTest.java
index 0404c0e..2a84ba1 100644
--- a/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessorTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessorTest.java
@@ -21,6 +21,8 @@ import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
 import java.time.Duration;
 import java.time.Instant;
 import java.util.List;
+import java.util.Map;
+import java.util.Optional;
 import java.util.UUID;
 import org.junit.jupiter.api.Test;
 import org.mockito.InOrder;
@@ -131,6 +133,61 @@ class CriticalEventProcessorTest {
     assertProviderSuspensionPublishesEffectiveClose("operator close");
   }
 
+  @Test
+  void closesTerminalMarketsBeforePublishingLifecycleAndClosures() {
+    CriticalEventQueue queue = mock(CriticalEventQueue.class);
+    OddsFeedPublisher publisher = mock(OddsFeedPublisher.class);
+    RedisOddsCache cache = mock(RedisOddsCache.class);
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId embeddedMarket = new MarketId(UUID.randomUUID());
+    MarketId recoveredMarket = new MarketId(UUID.randomUUID());
+    CriticalEvent event =
+        CriticalEvent.terminalLifecycle(
+            eventId,
+            EventLifecycleStatus.FINISHED,
+            Instant.EPOCH,
+            Instant.EPOCH,
+            Map.of(embeddedMarket.value(), MarketStatus.SUSPENDED),
+            null,
+            null,
+            Map.of(),
+            null);
+    QueuedCriticalEvent queued = new QueuedCriticalEvent(RecordId.of("3-0"), event, false);
+    when(queue.poll()).thenReturn(List.of(queued));
+    when(cache.closeEventMarkets(eventId, EventLifecycleStatus.FINISHED))
+        .thenReturn(Map.of(recoveredMarket.value(), MarketStatus.OPEN));
+    when(cache.getEvent(eventId)).thenReturn(Optional.empty());
+
+    new CriticalEventProcessor(queue, publisher, cache, new EventCatalog()).drain();
+
+    InOrder order = inOrder(cache, publisher, queue);
+    order.verify(cache).closeEventMarkets(eventId, EventLifecycleStatus.FINISHED);
+    order
+        .verify(publisher)
+        .publishEventLifecycle(
+            eventId, EventLifecycleStatus.FINISHED, Instant.EPOCH, Instant.EPOCH);
+    order.verify(cache).getEvent(eventId);
+    order
+        .verify(publisher)
+        .publishMarketStatusChanged(
+            eventId,
+            embeddedMarket,
+            MarketStatus.SUSPENDED,
+            MarketStatus.CLOSED,
+            "EVENT_FINISHED",
+            Instant.EPOCH);
+    order
+        .verify(publisher)
+        .publishMarketStatusChanged(
+            eventId,
+            recoveredMarket,
+            MarketStatus.OPEN,
+            MarketStatus.CLOSED,
+            "EVENT_FINISHED",
+            Instant.EPOCH);
+    order.verify(queue).acknowledge(queued);
+  }
+
   private static void assertRestrictivePreviewSuppressesOpen(String reason) {
     CriticalEventQueue queue = mock(CriticalEventQueue.class);
     OddsFeedPublisher publisher = mock(OddsFeedPublisher.class);


## `feat(delivery): deliver match results`

diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEvent.java b/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEvent.java
index d9a5e94..c3a7653 100644
--- a/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEvent.java
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEvent.java
@@ -29,7 +29,8 @@ public record CriticalEvent(
 
   public enum Type {
     MARKET_STATUS,
-    EVENT_LIFECYCLE
+    EVENT_LIFECYCLE,
+    MATCH_RESULT
   }
 
   public static CriticalEvent marketStatus(
@@ -101,4 +102,27 @@ public record CriticalEvent(
         settledAt,
         occurredAt);
   }
+
+  public static CriticalEvent matchResult(
+      EventId eventId,
+      String score,
+      MatchFinalStatus finalStatus,
+      Map<String, String> resultDetail,
+      Instant settledAt) {
+    return new CriticalEvent(
+        Type.MATCH_RESULT,
+        eventId.value(),
+        null,
+        null,
+        null,
+        null,
+        null,
+        null,
+        Map.of(),
+        score,
+        finalStatus,
+        Map.copyOf(resultDetail),
+        settledAt,
+        settledAt);
+  }
 }
diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessor.java b/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessor.java
index 7cb536d..98b8bab 100644
--- a/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessor.java
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessor.java
@@ -62,6 +62,10 @@ public class CriticalEventProcessor {
       applyMarketTransition(event);
       return;
     }
+    if (event.type() == CriticalEvent.Type.MATCH_RESULT) {
+      publishMatchResult(event, new EventId(event.eventId()));
+      return;
+    }
     throw new IllegalStateException("Unsupported critical event type: " + event.type());
   }
 
@@ -97,9 +101,6 @@ public class CriticalEventProcessor {
           .closeEventMarkets(eventId, event.lifecycleStatus())
           .forEach(terminalMarkets::putIfAbsent);
     }
-    if (event.matchFinalStatus() != null) {
-      throw new IllegalStateException("Embedded match results are not deliverable yet");
-    }
     publisher.publishEventLifecycle(
         eventId, event.lifecycleStatus(), event.scheduledStartAt(), event.occurredAt());
     cache
@@ -127,6 +128,18 @@ public class CriticalEventProcessor {
                 MarketStatus.CLOSED,
                 "EVENT_" + event.lifecycleStatus(),
                 event.occurredAt()));
+    if (event.matchFinalStatus() != null) {
+      publishMatchResult(event, eventId);
+    }
+  }
+
+  private void publishMatchResult(CriticalEvent event, EventId eventId) {
+    publisher.publishMatchResult(
+        eventId,
+        event.score(),
+        event.matchFinalStatus(),
+        event.resultDetail(),
+        event.resultSettledAt());
   }
 
   private void publishMarketTransition(


## `test(delivery): verify result delivery`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessorTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessorTest.java
index 2a84ba1..a4e0206 100644
--- a/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessorTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessorTest.java
@@ -15,6 +15,7 @@ import com.sportsbook.oddsfeed.config.CriticalDeliveryProperties;
 import com.sportsbook.oddsfeed.publisher.OddsFeedPublisher;
 import com.sportsbook.protocol.event.EventLifecycleStatus;
 import com.sportsbook.protocol.event.MarketStatus;
+import com.sportsbook.protocol.event.MatchFinalStatus;
 import com.sportsbook.protocol.value.EventId;
 import com.sportsbook.protocol.value.MarketId;
 import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
@@ -188,6 +189,64 @@ class CriticalEventProcessorTest {
     order.verify(queue).acknowledge(queued);
   }
 
+  @Test
+  void publishesDirectMatchResultBeforeAcknowledgement() {
+    CriticalEventQueue queue = mock(CriticalEventQueue.class);
+    OddsFeedPublisher publisher = mock(OddsFeedPublisher.class);
+    EventId eventId = new EventId(UUID.randomUUID());
+    CriticalEvent event =
+        CriticalEvent.matchResult(
+            eventId, "2-1", MatchFinalStatus.COMPLETED, Map.of("winner", "home"), Instant.EPOCH);
+    QueuedCriticalEvent queued = new QueuedCriticalEvent(RecordId.of("4-0"), event, false);
+    when(queue.poll()).thenReturn(List.of(queued));
+
+    new CriticalEventProcessor(queue, publisher, mock(RedisOddsCache.class), new EventCatalog())
+        .drain();
+
+    InOrder order = inOrder(publisher, queue);
+    order
+        .verify(publisher)
+        .publishMatchResult(
+            eventId, "2-1", MatchFinalStatus.COMPLETED, Map.of("winner", "home"), Instant.EPOCH);
+    order.verify(queue).acknowledge(queued);
+  }
+
+  @Test
+  void publishesEmbeddedResultAfterItsTerminalLifecycle() {
+    CriticalEventQueue queue = mock(CriticalEventQueue.class);
+    OddsFeedPublisher publisher = mock(OddsFeedPublisher.class);
+    RedisOddsCache cache = mock(RedisOddsCache.class);
+    EventId eventId = new EventId(UUID.randomUUID());
+    CriticalEvent event =
+        CriticalEvent.terminalLifecycle(
+            eventId,
+            EventLifecycleStatus.FINISHED,
+            Instant.EPOCH,
+            Instant.EPOCH,
+            Map.of(),
+            "2-1",
+            MatchFinalStatus.COMPLETED,
+            Map.of("winner", "home"),
+            Instant.EPOCH);
+    QueuedCriticalEvent queued = new QueuedCriticalEvent(RecordId.of("4-1"), event, false);
+    when(queue.poll()).thenReturn(List.of(queued));
+    when(cache.closeEventMarkets(eventId, EventLifecycleStatus.FINISHED)).thenReturn(Map.of());
+    when(cache.getEvent(eventId)).thenReturn(Optional.empty());
+
+    new CriticalEventProcessor(queue, publisher, cache, new EventCatalog()).drain();
+
+    InOrder order = inOrder(publisher, queue);
+    order
+        .verify(publisher)
+        .publishEventLifecycle(
+            eventId, EventLifecycleStatus.FINISHED, Instant.EPOCH, Instant.EPOCH);
+    order
+        .verify(publisher)
+        .publishMatchResult(
+            eventId, "2-1", MatchFinalStatus.COMPLETED, Map.of("winner", "home"), Instant.EPOCH);
+    order.verify(queue).acknowledge(queued);
+  }
+
   private static void assertRestrictivePreviewSuppressesOpen(String reason) {
     CriticalEventQueue queue = mock(CriticalEventQueue.class);
     OddsFeedPublisher publisher = mock(OddsFeedPublisher.class);


## `feat(feed): enqueue safe market transitions`

diff --git a/src/main/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestrator.java b/src/main/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestrator.java
index 39586cb..f546978 100644
--- a/src/main/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestrator.java
+++ b/src/main/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestrator.java
@@ -2,12 +2,15 @@ package com.sportsbook.oddsfeed.orchestrator;
 
 import com.sportsbook.oddsfeed.api.EventCatalog;
 import com.sportsbook.oddsfeed.cache.RedisOddsCache;
+import com.sportsbook.oddsfeed.delivery.CriticalEvent;
+import com.sportsbook.oddsfeed.delivery.CriticalEventQueue;
 import com.sportsbook.oddsfeed.provider.EventSummary;
 import com.sportsbook.oddsfeed.provider.OddsProvider;
 import com.sportsbook.oddsfeed.provider.ProviderEvent;
 import com.sportsbook.oddsfeed.provider.Sport;
 import com.sportsbook.oddsfeed.publisher.KafkaPublishException;
 import com.sportsbook.oddsfeed.publisher.OddsFeedPublisher;
+import com.sportsbook.protocol.event.MarketStatus;
 import com.sportsbook.protocol.value.EventId;
 import jakarta.annotation.PostConstruct;
 import java.util.Optional;
@@ -23,21 +26,32 @@ public class FeedOrchestrator {
   private final RedisOddsCache cache;
   private final OddsFeedPublisher publisher;
   private final EventCatalog catalog;
+  private final CriticalEventQueue criticalQueue;
 
   public FeedOrchestrator(OddsProvider provider, RedisOddsCache cache, EventCatalog catalog) {
-    this(provider, cache, null, catalog);
+    this(provider, cache, null, catalog, null);
   }
 
-  @Autowired
   public FeedOrchestrator(
       OddsProvider provider,
       RedisOddsCache cache,
       OddsFeedPublisher publisher,
       EventCatalog catalog) {
+    this(provider, cache, publisher, catalog, null);
+  }
+
+  @Autowired
+  public FeedOrchestrator(
+      OddsProvider provider,
+      RedisOddsCache cache,
+      OddsFeedPublisher publisher,
+      EventCatalog catalog,
+      CriticalEventQueue criticalQueue) {
     this.provider = provider;
     this.cache = cache;
     this.publisher = publisher;
     this.catalog = catalog;
+    this.criticalQueue = criticalQueue;
   }
 
   @PostConstruct
@@ -70,6 +84,8 @@ public class FeedOrchestrator {
   void dispatch(EventId eventId, ProviderEvent event) {
     if (event instanceof ProviderEvent.OddsUpdated odds) {
       handleOdds(odds);
+    } else if (event instanceof ProviderEvent.MarketStatusUpdated status) {
+      handleMarketStatus(status);
     }
   }
 
@@ -101,4 +117,18 @@ public class FeedOrchestrator {
     cache.projectLatestOdds(
         odds.eventId(), odds.marketId(), odds.selectionId(), odds.newOdds(), odds.occurredAt());
   }
+
+  private void handleMarketStatus(ProviderEvent.MarketStatusUpdated status) {
+    criticalQueue.enqueue(
+        CriticalEvent.marketStatus(
+            status.eventId(),
+            status.marketId(),
+            status.previousStatus(),
+            status.newStatus(),
+            status.reason(),
+            status.occurredAt()));
+    if (status.newStatus() != MarketStatus.OPEN) {
+      cache.storeProviderMarketStatus(status.eventId(), status.marketId(), status.newStatus());
+    }
+  }
 }


