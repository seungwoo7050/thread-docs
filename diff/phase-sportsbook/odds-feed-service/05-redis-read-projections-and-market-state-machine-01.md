# Redis 조회 프로젝션과 마켓 상태 머신

## `feat(cache): define projection key contracts`

diff --git a/src/main/java/com/sportsbook/oddsfeed/cache/CacheKeys.java b/src/main/java/com/sportsbook/oddsfeed/cache/CacheKeys.java
new file mode 100644
index 0000000..61c32a5
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/cache/CacheKeys.java
@@ -0,0 +1,46 @@
+package com.sportsbook.oddsfeed.cache;
+
+import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.MarketId;
+import com.sportsbook.protocol.value.SelectionId;
+
+public final class CacheKeys {
+
+  private CacheKeys() {}
+
+  public static String odds(EventId eventId, MarketId marketId, SelectionId selectionId) {
+    return "odds:" + eventId.value() + ":" + marketId.value() + ":" + selectionId.value();
+  }
+
+  public static String event(EventId eventId) {
+    return "event:" + eventId.value();
+  }
+
+  public static String market(EventId eventId, MarketId marketId) {
+    return "market:" + eventId.value() + ":" + marketId.value();
+  }
+
+  public static String providerMarket(EventId eventId, MarketId marketId) {
+    return "market:provider:" + eventId.value() + ":" + marketId.value();
+  }
+
+  public static String marketOverride(EventId eventId, MarketId marketId) {
+    return "market:override:" + eventId.value() + ":" + marketId.value();
+  }
+
+  public static String eventMarkets(EventId eventId) {
+    return "event:markets:" + eventId.value();
+  }
+
+  public static String eventTerminal(EventId eventId) {
+    return "event:terminal:" + eventId.value();
+  }
+
+  public static String marketTerminal(EventId eventId, MarketId marketId) {
+    return "market:terminal:" + eventId.value() + ":" + marketId.value();
+  }
+
+  public static String marketFeedHold(EventId eventId, MarketId marketId) {
+    return "market:feed-hold:" + eventId.value() + ":" + marketId.value();
+  }
+}


## `test(cache): verify projection keys`

diff --git a/src/test/java/com/sportsbook/oddsfeed/cache/CacheKeysTest.java b/src/test/java/com/sportsbook/oddsfeed/cache/CacheKeysTest.java
new file mode 100644
index 0000000..d5933f0
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/cache/CacheKeysTest.java
@@ -0,0 +1,38 @@
+package com.sportsbook.oddsfeed.cache;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.MarketId;
+import com.sportsbook.protocol.value.SelectionId;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class CacheKeysTest {
+
+  private static final EventId EVENT =
+      new EventId(UUID.fromString("00000000-0000-0000-0000-000000000001"));
+  private static final MarketId MARKET =
+      new MarketId(UUID.fromString("00000000-0000-0000-0000-000000000002"));
+  private static final SelectionId SELECTION =
+      new SelectionId(UUID.fromString("00000000-0000-0000-0000-000000000003"));
+
+  @Test
+  void createsStableProjectionKeys() {
+    assertThat(CacheKeys.odds(EVENT, MARKET, SELECTION))
+        .isEqualTo("odds:" + EVENT.value() + ":" + MARKET.value() + ":" + SELECTION.value());
+    assertThat(CacheKeys.event(EVENT)).isEqualTo("event:" + EVENT.value());
+    assertThat(CacheKeys.market(EVENT, MARKET))
+        .isEqualTo("market:" + EVENT.value() + ":" + MARKET.value());
+    assertThat(CacheKeys.providerMarket(EVENT, MARKET))
+        .isEqualTo("market:provider:" + EVENT.value() + ":" + MARKET.value());
+    assertThat(CacheKeys.marketOverride(EVENT, MARKET))
+        .isEqualTo("market:override:" + EVENT.value() + ":" + MARKET.value());
+    assertThat(CacheKeys.eventMarkets(EVENT)).isEqualTo("event:markets:" + EVENT.value());
+    assertThat(CacheKeys.eventTerminal(EVENT)).isEqualTo("event:terminal:" + EVENT.value());
+    assertThat(CacheKeys.marketTerminal(EVENT, MARKET))
+        .isEqualTo("market:terminal:" + EVENT.value() + ":" + MARKET.value());
+    assertThat(CacheKeys.marketFeedHold(EVENT, MARKET))
+        .isEqualTo("market:feed-hold:" + EVENT.value() + ":" + MARKET.value());
+  }
+}


## `feat(cache): store current selection odds`

diff --git a/src/main/java/com/sportsbook/oddsfeed/cache/RedisOddsCache.java b/src/main/java/com/sportsbook/oddsfeed/cache/RedisOddsCache.java
new file mode 100644
index 0000000..5c68f1a
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/cache/RedisOddsCache.java
@@ -0,0 +1,76 @@
+package com.sportsbook.oddsfeed.cache;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.oddsfeed.config.CacheProperties;
+import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.MarketId;
+import com.sportsbook.protocol.value.Odds;
+import com.sportsbook.protocol.value.SelectionId;
+import java.time.Duration;
+import java.util.List;
+import java.util.Optional;
+import org.springframework.data.redis.core.StringRedisTemplate;
+import org.springframework.data.redis.core.script.DefaultRedisScript;
+import org.springframework.data.redis.core.script.RedisScript;
+import org.springframework.stereotype.Component;
+
+@Component
+public class RedisOddsCache {
+
+  private static final RedisScript<String> STORE_ODDS_AND_REGISTER =
+      new DefaultRedisScript<>(
+          """
+          local eventTerminal = redis.call('EXISTS', KEYS[4]) == 1
+          if eventTerminal then
+            redis.call('SET', KEYS[5], 'EVENT_' .. redis.call('GET', KEYS[4]), 'NX')
+          end
+          local terminal = eventTerminal or redis.call('EXISTS', KEYS[5]) == 1
+          local effective
+          if terminal then
+            effective = 'CLOSED'
+            redis.call('PSETEX', KEYS[2], ARGV[2], effective)
+            redis.call('HSETNX', KEYS[3], ARGV[3], 'OPEN')
+          else
+            redis.call('PSETEX', KEYS[1], ARGV[2], ARGV[1])
+            effective = redis.call('GET', KEYS[2]) or 'OPEN'
+            redis.call('HSET', KEYS[3], ARGV[3], effective)
+          end
+          redis.call('PEXPIRE', KEYS[3], ARGV[2])
+          return effective
+          """,
+          String.class);
+
+  private final StringRedisTemplate redis;
+  private final ObjectMapper objectMapper;
+  private final Duration ttl;
+
+  public RedisOddsCache(
+      StringRedisTemplate redis, ObjectMapper objectMapper, CacheProperties properties) {
+    this.redis = redis;
+    this.objectMapper = objectMapper;
+    this.ttl = properties.ttl();
+  }
+
+  public void storeOdds(EventId eventId, MarketId marketId, SelectionId selectionId, Odds odds) {
+    redis.execute(
+        STORE_ODDS_AND_REGISTER,
+        List.of(
+            CacheKeys.odds(eventId, marketId, selectionId),
+            CacheKeys.market(eventId, marketId),
+            CacheKeys.eventMarkets(eventId),
+            CacheKeys.eventTerminal(eventId),
+            CacheKeys.marketTerminal(eventId, marketId)),
+        odds.decimal().toPlainString(),
+        ttlMillis(),
+        marketId.value().toString());
+  }
+
+  public Optional<Odds> getOdds(EventId eventId, MarketId marketId, SelectionId selectionId) {
+    String value = redis.opsForValue().get(CacheKeys.odds(eventId, marketId, selectionId));
+    return value == null ? Optional.empty() : Optional.of(Odds.ofDecimal(value));
+  }
+
+  private String ttlMillis() {
+    return Long.toString(ttl.toMillis());
+  }
+}


## `test(cache): verify current odds projection`

diff --git a/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheTest.java b/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheTest.java
new file mode 100644
index 0000000..9c5a656
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheTest.java
@@ -0,0 +1,80 @@
+package com.sportsbook.oddsfeed.cache;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.oddsfeed.config.CacheProperties;
+import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.MarketId;
+import com.sportsbook.protocol.value.Odds;
+import com.sportsbook.protocol.value.SelectionId;
+import java.lang.reflect.Proxy;
+import java.time.Duration;
+import java.util.HashMap;
+import java.util.List;
+import java.util.Map;
+import java.util.UUID;
+import org.junit.jupiter.api.BeforeEach;
+import org.junit.jupiter.api.Test;
+import org.springframework.data.redis.core.StringRedisTemplate;
+import org.springframework.data.redis.core.ValueOperations;
+import org.springframework.data.redis.core.script.RedisScript;
+
+class RedisOddsCacheTest {
+
+  private final EventId eventId = new EventId(UUID.randomUUID());
+  private final MarketId marketId = new MarketId(UUID.randomUUID());
+  private final SelectionId selectionId = new SelectionId(UUID.randomUUID());
+  private RecordingRedis redis;
+  private RedisOddsCache cache;
+
+  @BeforeEach
+  void setUp() {
+    redis = new RecordingRedis();
+    cache =
+        new RedisOddsCache(
+            redis,
+            new ObjectMapper().findAndRegisterModules(),
+            new CacheProperties(Duration.ofHours(24)));
+  }
+
+  @Test
+  void storesOddsWithRegistryAndTerminalGuards() {
+    Odds odds = Odds.ofDecimal("1.85");
+
+    cache.storeOdds(eventId, marketId, selectionId, odds);
+
+    assertThat(cache.getOdds(eventId, marketId, selectionId)).contains(odds);
+    assertThat(redis.keys)
+        .containsExactly(
+            CacheKeys.odds(eventId, marketId, selectionId),
+            CacheKeys.market(eventId, marketId),
+            CacheKeys.eventMarkets(eventId),
+            CacheKeys.eventTerminal(eventId),
+            CacheKeys.marketTerminal(eventId, marketId));
+  }
+
+  private static final class RecordingRedis extends StringRedisTemplate {
+    private final Map<String, String> values = new HashMap<>();
+    private List<String> keys = List.of();
+
+    @Override
+    @SuppressWarnings("unchecked")
+    public <T> T execute(RedisScript<T> script, List<String> keys, Object... args) {
+      this.keys = List.copyOf(keys);
+      values.put(keys.get(0), args[0].toString());
+      return (T) "OPEN";
+    }
+
+    @Override
+    @SuppressWarnings("unchecked")
+    public ValueOperations<String, String> opsForValue() {
+      return (ValueOperations<String, String>)
+          Proxy.newProxyInstance(
+              ValueOperations.class.getClassLoader(),
+              new Class<?>[] {ValueOperations.class},
+              (proxy, method, args) ->
+                  "get".equals(method.getName()) ? values.get(args[0].toString()) : null);
+    }
+  }
+}


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


## `test(cache): verify event summary projection`

diff --git a/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheTest.java b/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheTest.java
index 9c5a656..12f839e 100644
--- a/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheTest.java
@@ -4,12 +4,16 @@ import static org.assertj.core.api.Assertions.assertThat;
 
 import com.fasterxml.jackson.databind.ObjectMapper;
 import com.sportsbook.oddsfeed.config.CacheProperties;
+import com.sportsbook.oddsfeed.provider.EventSummary;
+import com.sportsbook.oddsfeed.provider.Sport;
+import com.sportsbook.protocol.event.EventLifecycleStatus;
 import com.sportsbook.protocol.value.EventId;
 import com.sportsbook.protocol.value.MarketId;
 import com.sportsbook.protocol.value.Odds;
 import com.sportsbook.protocol.value.SelectionId;
 import java.lang.reflect.Proxy;
 import java.time.Duration;
+import java.time.Instant;
 import java.util.HashMap;
 import java.util.List;
 import java.util.Map;
@@ -54,6 +58,23 @@ class RedisOddsCacheTest {
             CacheKeys.marketTerminal(eventId, marketId));
   }
 
+  @Test
+  void roundTripsEventSummaryProjection() {
+    EventSummary summary =
+        new EventSummary(
+            eventId,
+            Sport.FOOTBALL,
+            "Premier League",
+            "Manchester United",
+            "Chelsea",
+            Instant.parse("2026-06-01T18:00:00Z"),
+            EventLifecycleStatus.SCHEDULED);
+
+    cache.storeEvent(summary);
+
+    assertThat(cache.getEvent(eventId)).contains(summary);
+  }
+
   private static final class RecordingRedis extends StringRedisTemplate {
     private final Map<String, String> values = new HashMap<>();
     private List<String> keys = List.of();
@@ -73,8 +94,15 @@ class RedisOddsCacheTest {
           Proxy.newProxyInstance(
               ValueOperations.class.getClassLoader(),
               new Class<?>[] {ValueOperations.class},
-              (proxy, method, args) ->
-                  "get".equals(method.getName()) ? values.get(args[0].toString()) : null);
+              (proxy, method, args) -> {
+                if ("get".equals(method.getName())) {
+                  return values.get(args[0].toString());
+                }
+                if ("set".equals(method.getName())) {
+                  values.put(args[0].toString(), args[1].toString());
+                }
+                return null;
+              });
     }
   }
 }


## `feat(cache): register markets with provider state`

diff --git a/src/main/java/com/sportsbook/oddsfeed/cache/RedisOddsCache.java b/src/main/java/com/sportsbook/oddsfeed/cache/RedisOddsCache.java
index b11ba69..8e742ff 100644
--- a/src/main/java/com/sportsbook/oddsfeed/cache/RedisOddsCache.java
+++ b/src/main/java/com/sportsbook/oddsfeed/cache/RedisOddsCache.java
@@ -4,6 +4,7 @@ import com.fasterxml.jackson.core.JsonProcessingException;
 import com.fasterxml.jackson.databind.ObjectMapper;
 import com.sportsbook.oddsfeed.config.CacheProperties;
 import com.sportsbook.oddsfeed.provider.EventSummary;
+import com.sportsbook.protocol.event.MarketStatus;
 import com.sportsbook.protocol.value.EventId;
 import com.sportsbook.protocol.value.MarketId;
 import com.sportsbook.protocol.value.Odds;
@@ -42,6 +43,39 @@ public class RedisOddsCache {
           """,
           String.class);
 
+  private static final RedisScript<String> STORE_PROVIDER_MARKET_STATUS =
+      new DefaultRedisScript<>(
+          """
+          local requested = ARGV[1]
+          local eventTerminal = redis.call('EXISTS', KEYS[6]) == 1
+          if eventTerminal then
+            redis.call('SET', KEYS[5], 'EVENT_' .. redis.call('GET', KEYS[6]), 'NX')
+          elseif requested == 'CLOSED' then
+            redis.call('SET', KEYS[5], 'MARKET_CLOSED')
+            redis.call('PSETEX', KEYS[2], ARGV[2], 'CLOSED')
+          end
+          local terminal = eventTerminal or redis.call('EXISTS', KEYS[5]) == 1
+          local effective
+          if terminal then
+            effective = 'CLOSED'
+          else
+            redis.call('PSETEX', KEYS[2], ARGV[2], requested)
+            effective = redis.call('GET', KEYS[3])
+            if not effective then
+              effective = redis.call('EXISTS', KEYS[4]) == 1 and 'SUSPENDED' or requested
+            end
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
@@ -94,7 +128,41 @@ public class RedisOddsCache {
     }
   }
 
+  public MarketStatus storeProviderMarketStatus(
+      EventId eventId, MarketId marketId, MarketStatus status) {
+    return requireStatus(
+        redis.execute(
+            STORE_PROVIDER_MARKET_STATUS,
+            marketKeys(eventId, marketId),
+            status.name(),
+            ttlMillis(),
+            marketId.value().toString()));
+  }
+
+  public Optional<MarketStatus> getMarketStatus(EventId eventId, MarketId marketId) {
+    String value = redis.opsForValue().get(CacheKeys.market(eventId, marketId));
+    return value == null ? Optional.empty() : Optional.of(MarketStatus.valueOf(value));
+  }
+
+  private static List<String> marketKeys(EventId eventId, MarketId marketId) {
+    return List.of(
+        CacheKeys.market(eventId, marketId),
+        CacheKeys.providerMarket(eventId, marketId),
+        CacheKeys.marketOverride(eventId, marketId),
+        CacheKeys.marketFeedHold(eventId, marketId),
+        CacheKeys.marketTerminal(eventId, marketId),
+        CacheKeys.eventTerminal(eventId),
+        CacheKeys.eventMarkets(eventId));
+  }
+
   private String ttlMillis() {
     return Long.toString(ttl.toMillis());
   }
+
+  private static MarketStatus requireStatus(String value) {
+    if (value == null) {
+      throw new IllegalStateException("Redis market projection returned no result");
+    }
+    return MarketStatus.valueOf(value);
+  }
 }


## `test(cache): verify atomic market projection`

diff --git a/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheIntegrationTest.java b/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheIntegrationTest.java
new file mode 100644
index 0000000..c12b9e9
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/cache/RedisOddsCacheIntegrationTest.java
@@ -0,0 +1,65 @@
+package com.sportsbook.oddsfeed.cache;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.oddsfeed.config.CacheProperties;
+import com.sportsbook.protocol.event.MarketStatus;
+import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.MarketId;
+import java.time.Duration;
+import java.util.UUID;
+import org.junit.jupiter.api.AfterEach;
+import org.junit.jupiter.api.BeforeEach;
+import org.junit.jupiter.api.Test;
+import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
+import org.springframework.data.redis.core.StringRedisTemplate;
+import org.testcontainers.containers.GenericContainer;
+import org.testcontainers.junit.jupiter.Container;
+import org.testcontainers.junit.jupiter.Testcontainers;
+
+@Testcontainers(disabledWithoutDocker = true)
+class RedisOddsCacheIntegrationTest {
+
+  @Container
+  private static final GenericContainer<?> REDIS =
+      new GenericContainer<>("redis:7-alpine").withExposedPorts(6379);
+
+  private LettuceConnectionFactory connectionFactory;
+  private StringRedisTemplate redis;
+
+  @BeforeEach
+  void setUp() {
+    connectionFactory = new LettuceConnectionFactory(REDIS.getHost(), REDIS.getFirstMappedPort());
+    connectionFactory.afterPropertiesSet();
+    redis = new StringRedisTemplate(connectionFactory);
+    redis.afterPropertiesSet();
+  }
+
+  @AfterEach
+  void tearDown() {
+    connectionFactory.destroy();
+  }
+
+  @Test
+  void projectsProviderStatusWithAtomicRegistryKeys() {
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+    RedisOddsCache cache = cache();
+
+    assertThat(cache.storeProviderMarketStatus(eventId, marketId, MarketStatus.SUSPENDED))
+        .isEqualTo(MarketStatus.SUSPENDED);
+
+    assertThat(cache.getMarketStatus(eventId, marketId)).contains(MarketStatus.SUSPENDED);
+    assertThat(redis.opsForValue().get(CacheKeys.providerMarket(eventId, marketId)))
+        .isEqualTo(MarketStatus.SUSPENDED.name());
+    assertThat(cache.getRegisteredMarkets(eventId)).containsEntry(marketId, MarketStatus.SUSPENDED);
+  }
+
+  private RedisOddsCache cache() {
+    return new RedisOddsCache(
+        redis,
+        new ObjectMapper().findAndRegisterModules(),
+        new CacheProperties(Duration.ofHours(24)));
+  }
+}


