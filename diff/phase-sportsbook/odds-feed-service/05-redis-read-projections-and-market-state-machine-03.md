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
