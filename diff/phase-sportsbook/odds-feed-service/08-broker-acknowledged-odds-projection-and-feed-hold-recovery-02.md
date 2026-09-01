## `feat(feed): project acknowledged odds`

diff --git a/src/main/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestrator.java b/src/main/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestrator.java
index 9df7a62..02c4247 100644
--- a/src/main/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestrator.java
+++ b/src/main/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestrator.java
@@ -4,10 +4,14 @@ import com.sportsbook.oddsfeed.api.EventCatalog;
 import com.sportsbook.oddsfeed.cache.RedisOddsCache;
 import com.sportsbook.oddsfeed.provider.EventSummary;
 import com.sportsbook.oddsfeed.provider.OddsProvider;
+import com.sportsbook.oddsfeed.provider.ProviderEvent;
 import com.sportsbook.oddsfeed.provider.Sport;
+import com.sportsbook.oddsfeed.publisher.OddsFeedPublisher;
+import com.sportsbook.protocol.value.EventId;
 import jakarta.annotation.PostConstruct;
 import java.util.Optional;
 import java.util.concurrent.TimeUnit;
+import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.scheduling.annotation.Scheduled;
 import org.springframework.stereotype.Component;
 
@@ -16,11 +20,22 @@ public class FeedOrchestrator {
 
   private final OddsProvider provider;
   private final RedisOddsCache cache;
+  private final OddsFeedPublisher publisher;
   private final EventCatalog catalog;
 
   public FeedOrchestrator(OddsProvider provider, RedisOddsCache cache, EventCatalog catalog) {
+    this(provider, cache, null, catalog);
+  }
+
+  @Autowired
+  public FeedOrchestrator(
+      OddsProvider provider,
+      RedisOddsCache cache,
+      OddsFeedPublisher publisher,
+      EventCatalog catalog) {
     this.provider = provider;
     this.cache = cache;
+    this.publisher = publisher;
     this.catalog = catalog;
   }
 
@@ -50,4 +65,23 @@ public class FeedOrchestrator {
       cache.storeEvent(initial);
     }
   }
+
+  void dispatch(EventId eventId, ProviderEvent event) {
+    if (event instanceof ProviderEvent.OddsUpdated odds) {
+      boolean held = cache.isFeedHeld(odds.eventId(), odds.marketId());
+      boolean published =
+          publisher.publishOddsChanged(
+              odds.eventId(),
+              odds.marketId(),
+              odds.selectionId(),
+              odds.previousOdds(),
+              odds.newOdds(),
+              odds.occurredAt(),
+              held);
+      if (!held || published) {
+        cache.projectLatestOdds(
+            odds.eventId(), odds.marketId(), odds.selectionId(), odds.newOdds(), odds.occurredAt());
+      }
+    }
+  }
 }


## `test(feed): verify acknowledged odds ordering`

diff --git a/src/test/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestratorTest.java b/src/test/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestratorTest.java
index 1f8d5a8..ff3c41e 100644
--- a/src/test/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestratorTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestratorTest.java
@@ -11,10 +11,16 @@ import com.sportsbook.oddsfeed.provider.MatchOutcome;
 import com.sportsbook.oddsfeed.provider.OddsProvider;
 import com.sportsbook.oddsfeed.provider.ProviderEvent;
 import com.sportsbook.oddsfeed.provider.Sport;
+import com.sportsbook.oddsfeed.publisher.OddsFeedPublisher;
 import com.sportsbook.protocol.event.EventLifecycleStatus;
+import com.sportsbook.protocol.event.MarketStatus;
 import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.MarketId;
+import com.sportsbook.protocol.value.Odds;
+import com.sportsbook.protocol.value.SelectionId;
 import java.time.Duration;
 import java.time.Instant;
+import java.util.ArrayList;
 import java.util.HashMap;
 import java.util.List;
 import java.util.Map;
@@ -44,6 +50,34 @@ class FeedOrchestratorTest {
     assertThat(cache.stores).isEqualTo(1);
   }
 
+  @Test
+  void projectsOddsOnlyAfterPublisherAcknowledgement() {
+    List<String> order = new ArrayList<>();
+    RecordingCache cache = new RecordingCache(Map.of(), order);
+    RecordingPublisher publisher = new RecordingPublisher(order);
+    FeedOrchestrator orchestrator =
+        new FeedOrchestrator(new StubProvider(List.of()), cache, publisher, new EventCatalog());
+    EventId eventId = new EventId(UUID.randomUUID());
+    ProviderEvent odds =
+        new ProviderEvent.OddsUpdated(
+            eventId,
+            new MarketId(UUID.randomUUID()),
+            new SelectionId(UUID.randomUUID()),
+            Odds.ofDecimal("2.00"),
+            Odds.ofDecimal("2.10"),
+            Instant.EPOCH);
+
+    cache.held = true;
+    publisher.published = false;
+    orchestrator.dispatch(eventId, odds);
+    assertThat(order).containsExactly("publish");
+
+    publisher.published = true;
+    orchestrator.dispatch(eventId, odds);
+    assertThat(order).containsExactly("publish", "publish", "cache");
+    assertThat(publisher.forceCurrentSnapshot).isTrue();
+  }
+
   private static EventSummary event(UUID id, EventLifecycleStatus status) {
     return new EventSummary(
         new EventId(id),
@@ -74,12 +108,19 @@ class FeedOrchestratorTest {
 
   private static final class RecordingCache extends RedisOddsCache {
     private final Map<EventId, EventSummary> events;
+    private final List<String> order;
+    private boolean held;
     private int stores;
 
     private RecordingCache(Map<EventId, EventSummary> events) {
+      this(events, new ArrayList<>());
+    }
+
+    private RecordingCache(Map<EventId, EventSummary> events, List<String> order) {
       super(
           new StringRedisTemplate(), new ObjectMapper(), new CacheProperties(Duration.ofHours(1)));
       this.events = new HashMap<>(events);
+      this.order = order;
     }
 
     @Override
@@ -92,5 +133,51 @@ class FeedOrchestratorTest {
       stores++;
       events.put(summary.eventId(), summary);
     }
+
+    @Override
+    public MarketStatus projectLatestOdds(
+        EventId eventId,
+        MarketId marketId,
+        SelectionId selectionId,
+        Odds odds,
+        Instant observedAt) {
+      order.add("cache");
+      return MarketStatus.SUSPENDED;
+    }
+
+    @Override
+    public boolean isFeedHeld(EventId eventId, MarketId marketId) {
+      return held;
+    }
+  }
+
+  private static final class RecordingPublisher extends OddsFeedPublisher {
+    private final List<String> order;
+    private boolean published = true;
+    private boolean forceCurrentSnapshot;
+
+    private RecordingPublisher(List<String> order) {
+      super(null, null, null, null);
+      this.order = order;
+    }
+
+    @Override
+    public boolean isHealthy() {
+      return true;
+    }
+
+    @Override
+    public boolean publishOddsChanged(
+        EventId eventId,
+        MarketId marketId,
+        SelectionId selectionId,
+        Odds previous,
+        Odds next,
+        Instant changedAt,
+        boolean forceCurrentSnapshot) {
+      order.add("publish");
+      this.forceCurrentSnapshot = forceCurrentSnapshot;
+      return published;
+    }
   }
 }


## `feat(feed): suspend markets during broker outages`

diff --git a/src/main/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestrator.java b/src/main/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestrator.java
index 02c4247..39586cb 100644
--- a/src/main/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestrator.java
+++ b/src/main/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestrator.java
@@ -6,6 +6,7 @@ import com.sportsbook.oddsfeed.provider.EventSummary;
 import com.sportsbook.oddsfeed.provider.OddsProvider;
 import com.sportsbook.oddsfeed.provider.ProviderEvent;
 import com.sportsbook.oddsfeed.provider.Sport;
+import com.sportsbook.oddsfeed.publisher.KafkaPublishException;
 import com.sportsbook.oddsfeed.publisher.OddsFeedPublisher;
 import com.sportsbook.protocol.value.EventId;
 import jakarta.annotation.PostConstruct;
@@ -68,6 +69,17 @@ public class FeedOrchestrator {
 
   void dispatch(EventId eventId, ProviderEvent event) {
     if (event instanceof ProviderEvent.OddsUpdated odds) {
+      handleOdds(odds);
+    }
+  }
+
+  private void handleOdds(ProviderEvent.OddsUpdated odds) {
+    if (!publisher.isHealthy()) {
+      cache.holdLatestOdds(
+          odds.eventId(), odds.marketId(), odds.selectionId(), odds.newOdds(), odds.occurredAt());
+      return;
+    }
+    try {
       boolean held = cache.isFeedHeld(odds.eventId(), odds.marketId());
       boolean published =
           publisher.publishOddsChanged(
@@ -78,10 +90,15 @@ public class FeedOrchestrator {
               odds.newOdds(),
               odds.occurredAt(),
               held);
-      if (!held || published) {
-        cache.projectLatestOdds(
-            odds.eventId(), odds.marketId(), odds.selectionId(), odds.newOdds(), odds.occurredAt());
+      if (held && !published) {
+        return;
       }
+    } catch (KafkaPublishException error) {
+      cache.holdLatestOdds(
+          odds.eventId(), odds.marketId(), odds.selectionId(), odds.newOdds(), odds.occurredAt());
+      return;
     }
+    cache.projectLatestOdds(
+        odds.eventId(), odds.marketId(), odds.selectionId(), odds.newOdds(), odds.occurredAt());
   }
 }


## `test(feed): verify broker outage safety`

diff --git a/src/test/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestratorTest.java b/src/test/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestratorTest.java
index ff3c41e..7cd29ed 100644
--- a/src/test/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestratorTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/orchestrator/FeedOrchestratorTest.java
@@ -11,6 +11,7 @@ import com.sportsbook.oddsfeed.provider.MatchOutcome;
 import com.sportsbook.oddsfeed.provider.OddsProvider;
 import com.sportsbook.oddsfeed.provider.ProviderEvent;
 import com.sportsbook.oddsfeed.provider.Sport;
+import com.sportsbook.oddsfeed.publisher.KafkaPublishException;
 import com.sportsbook.oddsfeed.publisher.OddsFeedPublisher;
 import com.sportsbook.protocol.event.EventLifecycleStatus;
 import com.sportsbook.protocol.event.MarketStatus;
@@ -78,6 +79,29 @@ class FeedOrchestratorTest {
     assertThat(publisher.forceCurrentSnapshot).isTrue();
   }
 
+  @Test
+  void holdsLatestOddsWhileTheBrokerIsUnavailable() {
+    assertOutageOrder(new RecordingPublisher(new ArrayList<>(), false, false), "hold");
+    assertOutageOrder(new RecordingPublisher(new ArrayList<>(), true, true), "publish", "hold");
+  }
+
+  private static void assertOutageOrder(RecordingPublisher publisher, String... expected) {
+    RecordingCache cache = new RecordingCache(Map.of(), publisher.order);
+    FeedOrchestrator orchestrator =
+        new FeedOrchestrator(new StubProvider(List.of()), cache, publisher, new EventCatalog());
+    EventId eventId = new EventId(UUID.randomUUID());
+    orchestrator.dispatch(
+        eventId,
+        new ProviderEvent.OddsUpdated(
+            eventId,
+            new MarketId(UUID.randomUUID()),
+            new SelectionId(UUID.randomUUID()),
+            Odds.ofDecimal("2.00"),
+            Odds.ofDecimal("2.10"),
+            Instant.EPOCH));
+    assertThat(publisher.order).containsExactly(expected);
+  }
+
   private static EventSummary event(UUID id, EventLifecycleStatus status) {
     return new EventSummary(
         new EventId(id),
@@ -149,21 +173,40 @@ class FeedOrchestratorTest {
     public boolean isFeedHeld(EventId eventId, MarketId marketId) {
       return held;
     }
+
+    @Override
+    public MarketStatus holdLatestOdds(
+        EventId eventId,
+        MarketId marketId,
+        SelectionId selectionId,
+        Odds odds,
+        Instant observedAt) {
+      order.add("hold");
+      return MarketStatus.SUSPENDED;
+    }
   }
 
   private static final class RecordingPublisher extends OddsFeedPublisher {
     private final List<String> order;
     private boolean published = true;
     private boolean forceCurrentSnapshot;
+    private final boolean healthy;
+    private final boolean fail;
 
     private RecordingPublisher(List<String> order) {
+      this(order, true, false);
+    }
+
+    private RecordingPublisher(List<String> order, boolean healthy, boolean fail) {
       super(null, null, null, null);
       this.order = order;
+      this.healthy = healthy;
+      this.fail = fail;
     }
 
     @Override
     public boolean isHealthy() {
-      return true;
+      return healthy;
     }
 
     @Override
@@ -177,6 +220,9 @@ class FeedOrchestratorTest {
         boolean forceCurrentSnapshot) {
       order.add("publish");
       this.forceCurrentSnapshot = forceCurrentSnapshot;
+      if (fail) {
+        throw new KafkaPublishException("broker unavailable", new IllegalStateException());
+      }
       return published;
     }
   }
