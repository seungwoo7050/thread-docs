## `test(cache): verify market registry recovery`

diff --git a/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheIntegrationTest.java b/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheIntegrationTest.java
index c12b9e9..4cbfcc2 100644
--- a/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheIntegrationTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheIntegrationTest.java
@@ -56,6 +56,16 @@ class RedisOddsCacheIntegrationTest {
     assertThat(cache.getRegisteredMarkets(eventId)).containsEntry(marketId, MarketStatus.SUSPENDED);
   }
 
+  @Test
+  void registrySurvivesCacheRestart() {
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+    RedisOddsCache cache = cache();
+    cache.storeProviderMarketStatus(eventId, marketId, MarketStatus.OPEN);
+
+    assertThat(cache().getRegisteredMarkets(eventId)).containsEntry(marketId, MarketStatus.OPEN);
+  }
+
   private RedisOddsCache cache() {
     return new RedisOddsCache(
         redis,


## `feat(cache): preserve terminal market closures`

diff --git a/src/main/java/com/sportsbook/oddsfeed/cache/RedisOddsCache.java b/src/main/java/com/sportsbook/oddsfeed/cache/RedisOddsCache.java
index 8e742ff..9ddb3b5 100644
--- a/src/main/java/com/sportsbook/oddsfeed/cache/RedisOddsCache.java
+++ b/src/main/java/com/sportsbook/oddsfeed/cache/RedisOddsCache.java
@@ -4,14 +4,21 @@ import com.fasterxml.jackson.core.JsonProcessingException;
 import com.fasterxml.jackson.databind.ObjectMapper;
 import com.sportsbook.oddsfeed.config.CacheProperties;
 import com.sportsbook.oddsfeed.provider.EventSummary;
+import com.sportsbook.protocol.event.EventLifecycleStatus;
 import com.sportsbook.protocol.event.MarketStatus;
 import com.sportsbook.protocol.value.EventId;
 import com.sportsbook.protocol.value.MarketId;
 import com.sportsbook.protocol.value.Odds;
 import com.sportsbook.protocol.value.SelectionId;
 import java.time.Duration;
+import java.util.Collections;
+import java.util.Comparator;
+import java.util.LinkedHashMap;
 import java.util.List;
+import java.util.Map;
 import java.util.Optional;
+import java.util.TreeMap;
+import java.util.UUID;
 import org.springframework.data.redis.core.StringRedisTemplate;
 import org.springframework.data.redis.core.script.DefaultRedisScript;
 import org.springframework.data.redis.core.script.RedisScript;
@@ -144,6 +151,27 @@ public class RedisOddsCache {
     return value == null ? Optional.empty() : Optional.of(MarketStatus.valueOf(value));
   }
 
+  public boolean isMarketTerminal(EventId eventId, MarketId marketId) {
+    return Boolean.TRUE.equals(redis.hasKey(CacheKeys.marketTerminal(eventId, marketId)));
+  }
+
+  public Optional<EventLifecycleStatus> getEventTerminal(EventId eventId) {
+    String value = redis.opsForValue().get(CacheKeys.eventTerminal(eventId));
+    return value == null ? Optional.empty() : Optional.of(EventLifecycleStatus.valueOf(value));
+  }
+
+  public Map<MarketId, MarketStatus> getRegisteredMarkets(EventId eventId) {
+    Map<Object, Object> entries = redis.opsForHash().entries(CacheKeys.eventMarkets(eventId));
+    Map<MarketId, MarketStatus> markets =
+        new TreeMap<>(Comparator.comparing(id -> id.value().toString()));
+    entries.forEach(
+        (id, status) ->
+            markets.put(
+                new MarketId(UUID.fromString(id.toString())),
+                MarketStatus.valueOf(status.toString())));
+    return Collections.unmodifiableMap(new LinkedHashMap<>(markets));
+  }
+
   private static List<String> marketKeys(EventId eventId, MarketId marketId) {
     return List.of(
         CacheKeys.market(eventId, marketId),


## `test(cache): reject late terminal reopen attempts`

diff --git a/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheIntegrationTest.java b/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheIntegrationTest.java
index 4cbfcc2..487baa1 100644
--- a/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheIntegrationTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheIntegrationTest.java
@@ -7,6 +7,8 @@ import com.sportsbook.oddsfeed.config.CacheProperties;
 import com.sportsbook.protocol.event.MarketStatus;
 import com.sportsbook.protocol.value.EventId;
 import com.sportsbook.protocol.value.MarketId;
+import com.sportsbook.protocol.value.Odds;
+import com.sportsbook.protocol.value.SelectionId;
 import java.time.Duration;
 import java.util.UUID;
 import org.junit.jupiter.api.AfterEach;
@@ -66,6 +68,25 @@ class RedisOddsCacheIntegrationTest {
     assertThat(cache().getRegisteredMarkets(eventId)).containsEntry(marketId, MarketStatus.OPEN);
   }
 
+  @Test
+  void terminalMarketRejectsLateProviderAndOddsUpdates() {
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+    SelectionId selectionId = new SelectionId(UUID.randomUUID());
+    RedisOddsCache cache = cache();
+    cache.storeOdds(eventId, marketId, selectionId, Odds.ofDecimal("1.85"));
+
+    assertThat(cache.storeProviderMarketStatus(eventId, marketId, MarketStatus.CLOSED))
+        .isEqualTo(MarketStatus.CLOSED);
+    assertThat(cache.storeProviderMarketStatus(eventId, marketId, MarketStatus.OPEN))
+        .isEqualTo(MarketStatus.CLOSED);
+    cache.storeOdds(eventId, marketId, selectionId, Odds.ofDecimal("2.20"));
+
+    assertThat(cache.getOdds(eventId, marketId, selectionId)).contains(Odds.ofDecimal("1.85"));
+    assertThat(cache.getMarketStatus(eventId, marketId)).contains(MarketStatus.CLOSED);
+    assertThat(redis.getExpire(CacheKeys.marketTerminal(eventId, marketId))).isEqualTo(-1);
+  }
+
   private RedisOddsCache cache() {
     return new RedisOddsCache(
         redis,


## `feat(cache): project operator market overrides`

diff --git a/src/main/java/com/sportsbook/oddsfeed/cache/RedisOddsCache.java b/src/main/java/com/sportsbook/oddsfeed/cache/RedisOddsCache.java
index 9ddb3b5..66d81cd 100644
--- a/src/main/java/com/sportsbook/oddsfeed/cache/RedisOddsCache.java
+++ b/src/main/java/com/sportsbook/oddsfeed/cache/RedisOddsCache.java
@@ -83,6 +83,45 @@ public class RedisOddsCache {
           """,
           String.class);
 
+  private static final RedisScript<String> STORE_OPERATOR_MARKET_STATUS =
+      new DefaultRedisScript<>(
+          """
+          local requested = ARGV[1]
+          local eventTerminal = redis.call('EXISTS', KEYS[6]) == 1
+          if eventTerminal then
+            redis.call('SET', KEYS[5], 'EVENT_' .. redis.call('GET', KEYS[6]), 'NX')
+          end
+          if requested == 'OPEN' then
+            redis.call('DEL', KEYS[3])
+          else
+            redis.call('SET', KEYS[3], requested)
+          end
+          local provider = redis.call('GET', KEYS[2]) or 'OPEN'
+          if provider == 'CLOSED' and not eventTerminal then
+            redis.call('SET', KEYS[5], 'MARKET_CLOSED')
+          end
+          local terminal = eventTerminal or redis.call('EXISTS', KEYS[5]) == 1
+          local effective
+          if terminal then
+            effective = 'CLOSED'
+          elseif requested ~= 'OPEN' then
+            effective = requested
+          elseif redis.call('EXISTS', KEYS[4]) == 1 then
+            effective = 'SUSPENDED'
+          else
+            effective = provider
+          end
+          redis.call('PSETEX', KEYS[1], ARGV[2], effective)
+          if eventTerminal then
+            redis.call('HSETNX', KEYS[7], ARGV[3], 'OPEN')
+          else
+            redis.call('HSET', KEYS[7], ARGV[3], effective)
+          end
+          redis.call('PEXPIRE', KEYS[7], ARGV[2])
+          return effective
+          """,
+          String.class);
+
   private final StringRedisTemplate redis;
   private final ObjectMapper objectMapper;
   private final Duration ttl;
@@ -146,6 +185,22 @@ public class RedisOddsCache {
             marketId.value().toString()));
   }
 
+  public MarketStatus storeOperatorMarketStatus(
+      EventId eventId, MarketId marketId, MarketStatus status) {
+    return requireStatus(
+        redis.execute(
+            STORE_OPERATOR_MARKET_STATUS,
+            marketKeys(eventId, marketId),
+            status.name(),
+            ttlMillis(),
+            marketId.value().toString()));
+  }
+
+  public Optional<MarketStatus> getMarketOverride(EventId eventId, MarketId marketId) {
+    String value = redis.opsForValue().get(CacheKeys.marketOverride(eventId, marketId));
+    return value == null ? Optional.empty() : Optional.of(MarketStatus.valueOf(value));
+  }
+
   public Optional<MarketStatus> getMarketStatus(EventId eventId, MarketId marketId) {
     String value = redis.opsForValue().get(CacheKeys.market(eventId, marketId));
     return value == null ? Optional.empty() : Optional.of(MarketStatus.valueOf(value));


## `test(cache): verify operator precedence`

diff --git a/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheIntegrationTest.java b/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheIntegrationTest.java
index 487baa1..e9432c9 100644
--- a/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheIntegrationTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheIntegrationTest.java
@@ -87,6 +87,28 @@ class RedisOddsCacheIntegrationTest {
     assertThat(redis.getExpire(CacheKeys.marketTerminal(eventId, marketId))).isEqualTo(-1);
   }
 
+  @Test
+  void operatorOverridePrecedesProviderUntilCleared() {
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+    RedisOddsCache cache = cache();
+    cache.storeProviderMarketStatus(eventId, marketId, MarketStatus.OPEN);
+
+    assertThat(cache.storeOperatorMarketStatus(eventId, marketId, MarketStatus.SUSPENDED))
+        .isEqualTo(MarketStatus.SUSPENDED);
+    assertThat(cache.storeProviderMarketStatus(eventId, marketId, MarketStatus.OPEN))
+        .isEqualTo(MarketStatus.SUSPENDED);
+    assertThat(cache.getMarketOverride(eventId, marketId)).contains(MarketStatus.SUSPENDED);
+
+    assertThat(cache.storeOperatorMarketStatus(eventId, marketId, MarketStatus.OPEN))
+        .isEqualTo(MarketStatus.OPEN);
+    assertThat(cache.getMarketOverride(eventId, marketId)).isEmpty();
+
+    cache.storeProviderMarketStatus(eventId, marketId, MarketStatus.CLOSED);
+    assertThat(cache.storeOperatorMarketStatus(eventId, marketId, MarketStatus.OPEN))
+        .isEqualTo(MarketStatus.CLOSED);
+  }
+
   private RedisOddsCache cache() {
     return new RedisOddsCache(
         redis,


## `feat(cache): project feed availability holds`

diff --git a/src/main/java/com/sportsbook/oddsfeed/cache/RedisOddsCache.java b/src/main/java/com/sportsbook/oddsfeed/cache/RedisOddsCache.java
index 66d81cd..6a320b2 100644
--- a/src/main/java/com/sportsbook/oddsfeed/cache/RedisOddsCache.java
+++ b/src/main/java/com/sportsbook/oddsfeed/cache/RedisOddsCache.java
@@ -11,6 +11,7 @@ import com.sportsbook.protocol.value.MarketId;
 import com.sportsbook.protocol.value.Odds;
 import com.sportsbook.protocol.value.SelectionId;
 import java.time.Duration;
+import java.time.Instant;
 import java.util.Collections;
 import java.util.Comparator;
 import java.util.LinkedHashMap;
@@ -122,6 +123,43 @@ public class RedisOddsCache {
           """,
           String.class);
 
+  private static final RedisScript<String> PROJECT_LATEST_ODDS =
+      new DefaultRedisScript<>(
+          """
+          local eventTerminal = redis.call('EXISTS', KEYS[7]) == 1
+          if eventTerminal then
+            redis.call('SET', KEYS[6], 'EVENT_' .. redis.call('GET', KEYS[7]), 'NX')
+          end
+          if eventTerminal or redis.call('EXISTS', KEYS[6]) == 1 then
+            redis.call('PSETEX', KEYS[2], ARGV[2], 'CLOSED')
+            redis.call('HSETNX', KEYS[8], ARGV[3], 'OPEN')
+            redis.call('PEXPIRE', KEYS[8], ARGV[2])
+            return 'CLOSED'
+          end
+          local heldAt = redis.call('GET', KEYS[5])
+          if not heldAt or tonumber(ARGV[4]) >= tonumber(heldAt) then
+            redis.call('PSETEX', KEYS[1], ARGV[2], ARGV[1])
+            if redis.call('EXISTS', KEYS[3]) == 0 then
+              redis.call('PSETEX', KEYS[3], ARGV[2], 'OPEN')
+            end
+            if ARGV[5] == 'HOLD' then
+              redis.call('PSETEX', KEYS[5], ARGV[2], ARGV[4])
+            else
+              redis.call('DEL', KEYS[5])
+            end
+          end
+          local effective = redis.call('GET', KEYS[4])
+          if not effective then
+            effective = redis.call('EXISTS', KEYS[5]) == 1
+              and 'SUSPENDED' or (redis.call('GET', KEYS[3]) or 'OPEN')
+          end
+          redis.call('PSETEX', KEYS[2], ARGV[2], effective)
+          redis.call('HSET', KEYS[8], ARGV[3], effective)
+          redis.call('PEXPIRE', KEYS[8], ARGV[2])
+          return effective
+          """,
+          String.class);
+
   private final StringRedisTemplate redis;
   private final ObjectMapper objectMapper;
   private final Duration ttl;
@@ -152,6 +190,20 @@ public class RedisOddsCache {
     return value == null ? Optional.empty() : Optional.of(Odds.ofDecimal(value));
   }
 
+  public MarketStatus holdLatestOdds(
+      EventId eventId, MarketId marketId, SelectionId selectionId, Odds odds, Instant observedAt) {
+    return executeOddsProjection(eventId, marketId, selectionId, odds, observedAt, "HOLD");
+  }
+
+  public MarketStatus projectLatestOdds(
+      EventId eventId, MarketId marketId, SelectionId selectionId, Odds odds, Instant observedAt) {
+    return executeOddsProjection(eventId, marketId, selectionId, odds, observedAt, "RELEASE");
+  }
+
+  public boolean isFeedHeld(EventId eventId, MarketId marketId) {
+    return Boolean.TRUE.equals(redis.hasKey(CacheKeys.marketFeedHold(eventId, marketId)));
+  }
+
   public void storeEvent(EventSummary summary) {
     try {
       redis
@@ -227,6 +279,32 @@ public class RedisOddsCache {
     return Collections.unmodifiableMap(new LinkedHashMap<>(markets));
   }
 
+  private MarketStatus executeOddsProjection(
+      EventId eventId,
+      MarketId marketId,
+      SelectionId selectionId,
+      Odds odds,
+      Instant observedAt,
+      String mode) {
+    return requireStatus(
+        redis.execute(
+            PROJECT_LATEST_ODDS,
+            List.of(
+                CacheKeys.odds(eventId, marketId, selectionId),
+                CacheKeys.market(eventId, marketId),
+                CacheKeys.providerMarket(eventId, marketId),
+                CacheKeys.marketOverride(eventId, marketId),
+                CacheKeys.marketFeedHold(eventId, marketId),
+                CacheKeys.marketTerminal(eventId, marketId),
+                CacheKeys.eventTerminal(eventId),
+                CacheKeys.eventMarkets(eventId)),
+            odds.decimal().toPlainString(),
+            ttlMillis(),
+            marketId.value().toString(),
+            Long.toString(observedAt.toEpochMilli()),
+            mode));
+  }
+
   private static List<String> marketKeys(EventId eventId, MarketId marketId) {
     return List.of(
         CacheKeys.market(eventId, marketId),


## `test(cache): verify availability precedence`

diff --git a/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheIntegrationTest.java b/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheIntegrationTest.java
index e9432c9..13a1148 100644
--- a/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheIntegrationTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheIntegrationTest.java
@@ -10,6 +10,7 @@ import com.sportsbook.protocol.value.MarketId;
 import com.sportsbook.protocol.value.Odds;
 import com.sportsbook.protocol.value.SelectionId;
 import java.time.Duration;
+import java.time.Instant;
 import java.util.UUID;
 import org.junit.jupiter.api.AfterEach;
 import org.junit.jupiter.api.BeforeEach;
@@ -109,6 +110,66 @@ class RedisOddsCacheIntegrationTest {
         .isEqualTo(MarketStatus.CLOSED);
   }
 
+  @Test
+  void feedHoldRejectsStaleAvailabilityUpdates() {
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+    SelectionId selectionId = new SelectionId(UUID.randomUUID());
+    RedisOddsCache cache = cache();
+    Instant heldAt = Instant.parse("2026-06-01T18:00:02Z");
+
+    assertThat(cache.holdLatestOdds(eventId, marketId, selectionId, Odds.ofDecimal("2.20"), heldAt))
+        .isEqualTo(MarketStatus.SUSPENDED);
+    assertThat(
+            cache.projectLatestOdds(
+                eventId, marketId, selectionId, Odds.ofDecimal("1.90"), heldAt.minusSeconds(1)))
+        .isEqualTo(MarketStatus.SUSPENDED);
+    assertThat(cache.getOdds(eventId, marketId, selectionId)).contains(Odds.ofDecimal("2.20"));
+    assertThat(cache.isFeedHeld(eventId, marketId)).isTrue();
+
+    assertThat(
+            cache.projectLatestOdds(
+                eventId, marketId, selectionId, Odds.ofDecimal("2.30"), heldAt.plusSeconds(1)))
+        .isEqualTo(MarketStatus.OPEN);
+    assertThat(cache.getOdds(eventId, marketId, selectionId)).contains(Odds.ofDecimal("2.30"));
+    assertThat(cache.isFeedHeld(eventId, marketId)).isFalse();
+  }
+
+  @Test
+  void operatorOverridePrecedesFeedHoldAndProvider() {
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+    SelectionId selectionId = new SelectionId(UUID.randomUUID());
+    RedisOddsCache cache = cache();
+    Instant heldAt = Instant.parse("2026-06-01T18:00:02Z");
+    cache.storeProviderMarketStatus(eventId, marketId, MarketStatus.OPEN);
+    cache.holdLatestOdds(eventId, marketId, selectionId, Odds.ofDecimal("2.20"), heldAt);
+
+    assertThat(cache.storeOperatorMarketStatus(eventId, marketId, MarketStatus.SUSPENDED))
+        .isEqualTo(MarketStatus.SUSPENDED);
+    assertThat(cache.storeProviderMarketStatus(eventId, marketId, MarketStatus.OPEN))
+        .isEqualTo(MarketStatus.SUSPENDED);
+    assertThat(cache.getMarketOverride(eventId, marketId)).contains(MarketStatus.SUSPENDED);
+    assertThat(cache.isFeedHeld(eventId, marketId)).isTrue();
+    assertThat(cache.storeOperatorMarketStatus(eventId, marketId, MarketStatus.OPEN))
+        .isEqualTo(MarketStatus.SUSPENDED);
+
+    assertThat(
+            cache.projectLatestOdds(
+                eventId, marketId, selectionId, Odds.ofDecimal("2.30"), heldAt.plusSeconds(1)))
+        .isEqualTo(MarketStatus.OPEN);
+
+    MarketId suspendedMarket = new MarketId(UUID.randomUUID());
+    SelectionId suspendedSelection = new SelectionId(UUID.randomUUID());
+    cache.storeProviderMarketStatus(eventId, suspendedMarket, MarketStatus.SUSPENDED);
+    assertThat(
+            cache.projectLatestOdds(
+                eventId, suspendedMarket, suspendedSelection, Odds.ofDecimal("1.90"), heldAt))
+        .isEqualTo(MarketStatus.SUSPENDED);
+    assertThat(redis.opsForValue().get(CacheKeys.providerMarket(eventId, suspendedMarket)))
+        .isEqualTo(MarketStatus.SUSPENDED.name());
+  }
+
   private RedisOddsCache cache() {
     return new RedisOddsCache(
         redis,


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


