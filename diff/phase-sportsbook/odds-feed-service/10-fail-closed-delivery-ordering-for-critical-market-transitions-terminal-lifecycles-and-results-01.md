# 중요 마켓 전이와 경기 종료·결과의 fail-close 전달 순서

## `feat(cache): fail close registered event markets`

diff --git a/src/main/java/com/sportsbook/oddsfeed/cache/RedisOddsCache.java b/src/main/java/com/sportsbook/oddsfeed/cache/RedisOddsCache.java
index 6a320b2..1ee02e6 100644
--- a/src/main/java/com/sportsbook/oddsfeed/cache/RedisOddsCache.java
+++ b/src/main/java/com/sportsbook/oddsfeed/cache/RedisOddsCache.java
@@ -160,6 +160,31 @@ public class RedisOddsCache {
           """,
           String.class);
 
+  private static final RedisScript<String> CLOSE_EVENT_MARKETS =
+      new DefaultRedisScript<>(
+          """
+          redis.call('SET', KEYS[1], ARGV[1], 'NX')
+          local terminalStatus = redis.call('GET', KEYS[1])
+          local inventory = redis.call('HGETALL', KEYS[2])
+          local closed = {}
+          for index = 1, #inventory, 2 do
+            local marketId = inventory[index]
+            local previous = inventory[index + 1]
+            local effectiveKey = ARGV[3] .. marketId
+            local providerKey = ARGV[4] .. marketId
+            local terminalKey = ARGV[5] .. marketId
+            redis.call('SET', terminalKey, 'EVENT_' .. terminalStatus, 'NX')
+            redis.call('PSETEX', providerKey, ARGV[2], 'CLOSED')
+            redis.call('PSETEX', effectiveKey, ARGV[2], 'CLOSED')
+            if previous ~= 'CLOSED' then
+              table.insert(closed, marketId .. '|' .. previous)
+            end
+          end
+          redis.call('PEXPIRE', KEYS[2], ARGV[2])
+          return table.concat(closed, '\\n')
+          """,
+          String.class);
+
   private final StringRedisTemplate redis;
   private final ObjectMapper objectMapper;
   private final Duration ttl;
@@ -279,6 +304,28 @@ public class RedisOddsCache {
     return Collections.unmodifiableMap(new LinkedHashMap<>(markets));
   }
 
+  public Map<UUID, MarketStatus> closeEventMarkets(
+      EventId eventId, EventLifecycleStatus terminalStatus) {
+    String encoded =
+        redis.execute(
+            CLOSE_EVENT_MARKETS,
+            List.of(CacheKeys.eventTerminal(eventId), CacheKeys.eventMarkets(eventId)),
+            terminalStatus.name(),
+            ttlMillis(),
+            "market:" + eventId.value() + ":",
+            "market:provider:" + eventId.value() + ":",
+            "market:terminal:" + eventId.value() + ":");
+    if (encoded == null || encoded.isBlank()) {
+      return Map.of();
+    }
+    Map<UUID, MarketStatus> closed = new TreeMap<>();
+    for (String entry : encoded.split("\\n")) {
+      String[] parts = entry.split("\\|", 2);
+      closed.put(UUID.fromString(parts[0]), MarketStatus.valueOf(parts[1]));
+    }
+    return Collections.unmodifiableMap(new LinkedHashMap<>(closed));
+  }
+
   private MarketStatus executeOddsProjection(
       EventId eventId,
       MarketId marketId,


## `test(cache): verify atomic terminal closure`

diff --git a/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheIntegrationTest.java b/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheIntegrationTest.java
index 13a1148..b40bcd5 100644
--- a/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheIntegrationTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheIntegrationTest.java
@@ -4,6 +4,7 @@ import static org.assertj.core.api.Assertions.assertThat;
 
 import com.fasterxml.jackson.databind.ObjectMapper;
 import com.sportsbook.oddsfeed.config.CacheProperties;
+import com.sportsbook.protocol.event.EventLifecycleStatus;
 import com.sportsbook.protocol.event.MarketStatus;
 import com.sportsbook.protocol.value.EventId;
 import com.sportsbook.protocol.value.MarketId;
@@ -11,6 +12,7 @@ import com.sportsbook.protocol.value.Odds;
 import com.sportsbook.protocol.value.SelectionId;
 import java.time.Duration;
 import java.time.Instant;
+import java.util.Map;
 import java.util.UUID;
 import org.junit.jupiter.api.AfterEach;
 import org.junit.jupiter.api.BeforeEach;
@@ -170,6 +172,29 @@ class RedisOddsCacheIntegrationTest {
         .isEqualTo(MarketStatus.SUSPENDED.name());
   }
 
+  @Test
+  void closesRecoveredMarketsWithPermanentTerminalLatches() {
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+    SelectionId selectionId = new SelectionId(UUID.randomUUID());
+    RedisOddsCache cache = cache();
+    cache.storeProviderMarketStatus(eventId, marketId, MarketStatus.OPEN);
+
+    RedisOddsCache restarted = cache();
+    assertThat(restarted.getRegisteredMarkets(eventId)).containsEntry(marketId, MarketStatus.OPEN);
+    assertThat(restarted.closeEventMarkets(eventId, EventLifecycleStatus.FINISHED))
+        .isEqualTo(Map.of(marketId.value(), MarketStatus.OPEN));
+    assertThat(restarted.closeEventMarkets(eventId, EventLifecycleStatus.FINISHED))
+        .isEqualTo(Map.of(marketId.value(), MarketStatus.OPEN));
+
+    restarted.storeProviderMarketStatus(eventId, marketId, MarketStatus.OPEN);
+    restarted.storeOperatorMarketStatus(eventId, marketId, MarketStatus.OPEN);
+    restarted.storeOdds(eventId, marketId, selectionId, Odds.ofDecimal("2.20"));
+    assertThat(restarted.getMarketStatus(eventId, marketId)).contains(MarketStatus.CLOSED);
+    assertThat(redis.getExpire(CacheKeys.eventTerminal(eventId))).isEqualTo(-1);
+    assertThat(redis.getExpire(CacheKeys.marketTerminal(eventId, marketId))).isEqualTo(-1);
+  }
+
   private RedisOddsCache cache() {
     return new RedisOddsCache(
         redis,


## `feat(delivery): snapshot terminal results`

diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEvent.java b/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEvent.java
index d6fc12e..d9a5e94 100644
--- a/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEvent.java
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEvent.java
@@ -2,9 +2,13 @@ package com.sportsbook.oddsfeed.delivery;
 
 import com.sportsbook.protocol.event.EventLifecycleStatus;
 import com.sportsbook.protocol.event.MarketStatus;
+import com.sportsbook.protocol.event.MatchFinalStatus;
 import com.sportsbook.protocol.value.EventId;
 import com.sportsbook.protocol.value.MarketId;
 import java.time.Instant;
+import java.util.Collections;
+import java.util.LinkedHashMap;
+import java.util.Map;
 import java.util.UUID;
 
 public record CriticalEvent(
@@ -16,6 +20,11 @@ public record CriticalEvent(
     String reason,
     EventLifecycleStatus lifecycleStatus,
     Instant scheduledStartAt,
+    Map<UUID, MarketStatus> terminalMarkets,
+    String score,
+    MatchFinalStatus matchFinalStatus,
+    Map<String, String> resultDetail,
+    Instant resultSettledAt,
     Instant occurredAt) {
 
   public enum Type {
@@ -39,6 +48,11 @@ public record CriticalEvent(
         reason,
         null,
         null,
+        Map.of(),
+        null,
+        null,
+        Map.of(),
+        null,
         occurredAt);
   }
 
@@ -53,6 +67,38 @@ public record CriticalEvent(
         null,
         status,
         scheduledStartAt,
+        Map.of(),
+        null,
+        null,
+        Map.of(),
+        null,
+        occurredAt);
+  }
+
+  public static CriticalEvent terminalLifecycle(
+      EventId eventId,
+      EventLifecycleStatus status,
+      Instant scheduledStartAt,
+      Instant occurredAt,
+      Map<UUID, MarketStatus> terminalMarkets,
+      String score,
+      MatchFinalStatus finalStatus,
+      Map<String, String> resultDetail,
+      Instant settledAt) {
+    return new CriticalEvent(
+        Type.EVENT_LIFECYCLE,
+        eventId.value(),
+        null,
+        null,
+        null,
+        null,
+        status,
+        scheduledStartAt,
+        Collections.unmodifiableMap(new LinkedHashMap<>(terminalMarkets)),
+        score,
+        finalStatus,
+        Map.copyOf(resultDetail),
+        settledAt,
         occurredAt);
   }
 }


## `test(delivery): verify terminal snapshot isolation`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventTest.java
index adf9b02..460f55c 100644
--- a/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventTest.java
@@ -1,12 +1,16 @@
 package com.sportsbook.oddsfeed.delivery;
 
 import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
 
 import com.sportsbook.protocol.event.EventLifecycleStatus;
 import com.sportsbook.protocol.event.MarketStatus;
+import com.sportsbook.protocol.event.MatchFinalStatus;
 import com.sportsbook.protocol.value.EventId;
 import com.sportsbook.protocol.value.MarketId;
 import java.time.Instant;
+import java.util.HashMap;
+import java.util.Map;
 import java.util.UUID;
 import org.junit.jupiter.api.Test;
 
@@ -53,4 +57,34 @@ class CriticalEventTest {
     assertThat(event.occurredAt()).isEqualTo(occurredAt);
     assertThat(event.marketId()).isNull();
   }
+
+  @Test
+  void isolatesTerminalMarketAndResultSnapshots() {
+    Map<UUID, MarketStatus> markets = new HashMap<>();
+    markets.put(marketId.value(), MarketStatus.OPEN);
+    Map<String, String> detail = new HashMap<>();
+    detail.put("winner", "home");
+    Instant settledAt = Instant.parse("2026-06-01T20:00:00Z");
+
+    CriticalEvent event =
+        CriticalEvent.terminalLifecycle(
+            eventId,
+            EventLifecycleStatus.FINISHED,
+            Instant.EPOCH,
+            settledAt,
+            markets,
+            "2-1",
+            MatchFinalStatus.COMPLETED,
+            detail,
+            settledAt);
+    markets.clear();
+    detail.clear();
+
+    assertThat(event.terminalMarkets()).containsEntry(marketId.value(), MarketStatus.OPEN);
+    assertThat(event.resultDetail()).containsEntry("winner", "home");
+    assertThat(event.score()).isEqualTo("2-1");
+    assertThat(event.resultSettledAt()).isEqualTo(settledAt);
+    assertThatThrownBy(() -> event.terminalMarkets().clear())
+        .isInstanceOf(UnsupportedOperationException.class);
+  }
 }


## `feat(delivery): apply safe market transitions`

diff --git a/src/main/java/com/sportsbook/oddsfeed/cache/RedisOddsCache.java b/src/main/java/com/sportsbook/oddsfeed/cache/RedisOddsCache.java
index 1ee02e6..d2c98bc 100644
--- a/src/main/java/com/sportsbook/oddsfeed/cache/RedisOddsCache.java
+++ b/src/main/java/com/sportsbook/oddsfeed/cache/RedisOddsCache.java
@@ -84,6 +84,40 @@ public class RedisOddsCache {
           """,
           String.class);
 
+  private static final RedisScript<String> PREPARE_PROVIDER_OPEN =
+      new DefaultRedisScript<>(
+          """
+          local eventTerminal = redis.call('EXISTS', KEYS[6]) == 1
+          local marketTerminal = redis.call('EXISTS', KEYS[5]) == 1
+          local provider = redis.call('GET', KEYS[2])
+          if eventTerminal then
+            redis.call('HSETNX', KEYS[7], ARGV[2], 'OPEN')
+            redis.call('PEXPIRE', KEYS[7], ARGV[1])
+            return 'CLOSED'
+          end
+          if provider == 'CLOSED' then
+            redis.call('SET', KEYS[5], 'MARKET_CLOSED', 'NX')
+            marketTerminal = true
+          end
+          if marketTerminal then
+            redis.call('HSET', KEYS[7], ARGV[2], 'CLOSED')
+            redis.call('PEXPIRE', KEYS[7], ARGV[1])
+            return 'CLOSED'
+          end
+          local effective = redis.call('GET', KEYS[3])
+          if not effective and redis.call('EXISTS', KEYS[4]) == 1 then
+            effective = 'SUSPENDED'
+          end
+          if effective then
+            redis.call('PSETEX', KEYS[2], ARGV[1], 'OPEN')
+            redis.call('HSET', KEYS[7], ARGV[2], effective)
+            redis.call('PEXPIRE', KEYS[7], ARGV[1])
+            return effective
+          end
+          return 'OPEN'
+          """,
+          String.class);
+
   private static final RedisScript<String> STORE_OPERATOR_MARKET_STATUS =
       new DefaultRedisScript<>(
           """
@@ -262,6 +296,15 @@ public class RedisOddsCache {
             marketId.value().toString()));
   }
 
+  public MarketStatus prepareProviderOpen(EventId eventId, MarketId marketId) {
+    return requireStatus(
+        redis.execute(
+            PREPARE_PROVIDER_OPEN,
+            marketKeys(eventId, marketId),
+            ttlMillis(),
+            marketId.value().toString()));
+  }
+
   public MarketStatus storeOperatorMarketStatus(
       EventId eventId, MarketId marketId, MarketStatus status) {
     return requireStatus(
diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessor.java b/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessor.java
index 2f2c747..1ba5cfd 100644
--- a/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessor.java
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessor.java
@@ -3,6 +3,9 @@ package com.sportsbook.oddsfeed.delivery;
 import com.sportsbook.oddsfeed.api.EventCatalog;
 import com.sportsbook.oddsfeed.cache.RedisOddsCache;
 import com.sportsbook.oddsfeed.publisher.OddsFeedPublisher;
+import com.sportsbook.protocol.event.MarketStatus;
+import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.MarketId;
 import java.util.concurrent.atomic.AtomicBoolean;
 import org.springframework.scheduling.annotation.Scheduled;
 import org.springframework.stereotype.Component;
@@ -46,7 +49,33 @@ public class CriticalEventProcessor {
   }
 
   void apply(CriticalEvent event) {
-    throw new IllegalStateException("Unsupported critical event type: " + event.type());
+    if (event.type() != CriticalEvent.Type.MARKET_STATUS) {
+      throw new IllegalStateException("Unsupported critical event type: " + event.type());
+    }
+    EventId eventId = new EventId(event.eventId());
+    MarketId marketId = new MarketId(event.marketId());
+    if (event.nextMarketStatus() == MarketStatus.OPEN) {
+      if (cache.prepareProviderOpen(eventId, marketId) != MarketStatus.OPEN) {
+        return;
+      }
+      publishMarketTransition(event, eventId, marketId, MarketStatus.OPEN);
+      cache.storeProviderMarketStatus(eventId, marketId, MarketStatus.OPEN);
+      return;
+    }
+    MarketStatus effective =
+        cache.storeProviderMarketStatus(eventId, marketId, event.nextMarketStatus());
+    publishMarketTransition(event, eventId, marketId, effective);
+  }
+
+  private void publishMarketTransition(
+      CriticalEvent event, EventId eventId, MarketId marketId, MarketStatus effectiveStatus) {
+    publisher.publishMarketStatusChanged(
+        eventId,
+        marketId,
+        event.previousMarketStatus(),
+        effectiveStatus,
+        event.reason(),
+        event.occurredAt());
   }
 
   public boolean isHealthy() {


## `test(delivery): verify restrictive-first ordering`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessorTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessorTest.java
index d83db43..b7a60ad 100644
--- a/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessorTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessorTest.java
@@ -1,17 +1,26 @@
 package com.sportsbook.oddsfeed.delivery;
 
 import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.inOrder;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.when;
 
 import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.oddsfeed.api.EventCatalog;
+import com.sportsbook.oddsfeed.cache.RedisOddsCache;
 import com.sportsbook.oddsfeed.config.CriticalDeliveryProperties;
+import com.sportsbook.oddsfeed.publisher.OddsFeedPublisher;
 import com.sportsbook.protocol.event.EventLifecycleStatus;
+import com.sportsbook.protocol.event.MarketStatus;
 import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.MarketId;
 import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
 import java.time.Duration;
 import java.time.Instant;
 import java.util.List;
 import java.util.UUID;
 import org.junit.jupiter.api.Test;
+import org.mockito.InOrder;
 import org.springframework.data.redis.connection.stream.RecordId;
 import org.springframework.data.redis.core.StringRedisTemplate;
 
@@ -38,6 +47,67 @@ class CriticalEventProcessorTest {
     assertThat(queue.poll()).isEmpty();
   }
 
+  @Test
+  void restrictiveProjectionPrecedesKafkaAndStreamAcknowledgement() {
+    CriticalEventQueue queue = mock(CriticalEventQueue.class);
+    OddsFeedPublisher publisher = mock(OddsFeedPublisher.class);
+    RedisOddsCache cache = mock(RedisOddsCache.class);
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+    CriticalEvent event =
+        CriticalEvent.marketStatus(
+            eventId,
+            marketId,
+            MarketStatus.OPEN,
+            MarketStatus.SUSPENDED,
+            "incident",
+            Instant.EPOCH);
+    QueuedCriticalEvent queued = new QueuedCriticalEvent(RecordId.of("2-0"), event, false);
+    when(queue.poll()).thenReturn(List.of(queued));
+    when(cache.storeProviderMarketStatus(eventId, marketId, MarketStatus.SUSPENDED))
+        .thenReturn(MarketStatus.SUSPENDED);
+
+    new CriticalEventProcessor(queue, publisher, cache, new EventCatalog()).drain();
+
+    InOrder order = inOrder(cache, publisher, queue);
+    order.verify(cache).storeProviderMarketStatus(eventId, marketId, MarketStatus.SUSPENDED);
+    order
+        .verify(publisher)
+        .publishMarketStatusChanged(
+            eventId,
+            marketId,
+            MarketStatus.OPEN,
+            MarketStatus.SUSPENDED,
+            "incident",
+            Instant.EPOCH);
+    order.verify(queue).acknowledge(queued);
+  }
+
+  @Test
+  void providerOpenProjectsOnlyAfterKafkaAcknowledgement() {
+    CriticalEventQueue queue = mock(CriticalEventQueue.class);
+    OddsFeedPublisher publisher = mock(OddsFeedPublisher.class);
+    RedisOddsCache cache = mock(RedisOddsCache.class);
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+    CriticalEvent event =
+        CriticalEvent.marketStatus(
+            eventId, marketId, MarketStatus.SUSPENDED, MarketStatus.OPEN, "resumed", Instant.EPOCH);
+    QueuedCriticalEvent queued = new QueuedCriticalEvent(RecordId.of("2-1"), event, false);
+    when(queue.poll()).thenReturn(List.of(queued));
+    when(cache.prepareProviderOpen(eventId, marketId)).thenReturn(MarketStatus.OPEN);
+
+    new CriticalEventProcessor(queue, publisher, cache, new EventCatalog()).drain();
+
+    InOrder order = inOrder(publisher, cache, queue);
+    order
+        .verify(publisher)
+        .publishMarketStatusChanged(
+            eventId, marketId, MarketStatus.SUSPENDED, MarketStatus.OPEN, "resumed", Instant.EPOCH);
+    order.verify(cache).storeProviderMarketStatus(eventId, marketId, MarketStatus.OPEN);
+    order.verify(queue).acknowledge(queued);
+  }
+
   private static final class RecoveringProcessor extends CriticalEventProcessor {
     private int attempts;
 


## `test(delivery): suppress unsafe provider openings`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessorTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessorTest.java
index b7a60ad..497a554 100644
--- a/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessorTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessorTest.java
@@ -3,6 +3,9 @@ package com.sportsbook.oddsfeed.delivery;
 import static org.assertj.core.api.Assertions.assertThat;
 import static org.mockito.Mockito.inOrder;
 import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.never;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.verifyNoInteractions;
 import static org.mockito.Mockito.when;
 
 import com.fasterxml.jackson.databind.ObjectMapper;
@@ -108,6 +111,37 @@ class CriticalEventProcessorTest {
     order.verify(queue).acknowledge(queued);
   }
 
+  @Test
+  void operatorOverrideSuppressesProviderOpenPublication() {
+    assertRestrictivePreviewSuppressesOpen("operator override");
+  }
+
+  @Test
+  void feedHoldSuppressesProviderOpenPublication() {
+    assertRestrictivePreviewSuppressesOpen("feed hold");
+  }
+
+  private static void assertRestrictivePreviewSuppressesOpen(String reason) {
+    CriticalEventQueue queue = mock(CriticalEventQueue.class);
+    OddsFeedPublisher publisher = mock(OddsFeedPublisher.class);
+    RedisOddsCache cache = mock(RedisOddsCache.class);
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+    CriticalEvent event =
+        CriticalEvent.marketStatus(
+            eventId, marketId, MarketStatus.SUSPENDED, MarketStatus.OPEN, reason, Instant.EPOCH);
+    QueuedCriticalEvent queued = new QueuedCriticalEvent(RecordId.of("2-2"), event, false);
+    when(queue.poll()).thenReturn(List.of(queued));
+    when(cache.prepareProviderOpen(eventId, marketId)).thenReturn(MarketStatus.SUSPENDED);
+
+    new CriticalEventProcessor(queue, publisher, cache, new EventCatalog()).drain();
+
+    verify(cache).prepareProviderOpen(eventId, marketId);
+    verify(cache, never()).storeProviderMarketStatus(eventId, marketId, MarketStatus.OPEN);
+    verifyNoInteractions(publisher);
+    verify(queue).acknowledge(queued);
+  }
+
   private static final class RecoveringProcessor extends CriticalEventProcessor {
     private int attempts;
 


