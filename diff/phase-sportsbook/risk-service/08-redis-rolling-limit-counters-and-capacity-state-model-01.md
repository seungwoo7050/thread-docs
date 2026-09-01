# Redis 롤링 한도 카운터와 용량 상태 모델

## `feat(counter): namespace monetary counters by currency`

diff --git a/src/main/java/com/sportsbook/risk/counter/LimitKeys.java b/src/main/java/com/sportsbook/risk/counter/LimitKeys.java
new file mode 100644
index 0000000..680dc45
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/counter/LimitKeys.java
@@ -0,0 +1,46 @@
+package com.sportsbook.risk.counter;
+
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.policy.SafeRedisNumber;
+import java.util.Objects;
+
+/** Redis keys for one user's committed rolling-limit dimension. */
+public final class LimitKeys {
+  private static final String PREFIX = "risk:limit:";
+
+  private LimitKeys() {}
+
+  public static Keys monetary(UserId userId, LimitType type, Currency currency) {
+    Objects.requireNonNull(type, "type");
+    Objects.requireNonNull(currency, "currency");
+    if (!type.currencyScoped()) {
+      throw new IllegalArgumentException("monetary keys require a currency-scoped limit");
+    }
+    return keys(userId, type.suffix() + ":" + currency.name().toLowerCase());
+  }
+
+  public static Keys selections(UserId userId) {
+    return keys(userId, LimitType.SELECTIONS_PER_MINUTE.suffix());
+  }
+
+  public static String member(BetId betId, long amount) {
+    Objects.requireNonNull(betId, "betId");
+    SafeRedisNumber.requirePositive(amount, "amount");
+    return betId.value() + "|" + amount;
+  }
+
+  private static Keys keys(UserId userId, String dimension) {
+    Objects.requireNonNull(userId, "userId");
+    String base = PREFIX + "{" + userId.value() + "}:" + dimension;
+    return new Keys(base + ":entries", base + ":sum");
+  }
+
+  public record Keys(String entries, String sum) {
+    public Keys {
+      Objects.requireNonNull(entries, "entries");
+      Objects.requireNonNull(sum, "sum");
+    }
+  }
+}


## `test(counter): verify currency-aware counter keys`

diff --git a/src/test/java/com/sportsbook/risk/counter/LimitKeysTest.java b/src/test/java/com/sportsbook/risk/counter/LimitKeysTest.java
new file mode 100644
index 0000000..3d86275
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/counter/LimitKeysTest.java
@@ -0,0 +1,45 @@
+package com.sportsbook.risk.counter;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.UserId;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class LimitKeysTest {
+  private static final UserId USER =
+      UserId.of(UUID.fromString("00000000-0000-0000-0000-000000000001"));
+  private static final BetId BET =
+      BetId.of(UUID.fromString("00000000-0000-0000-0000-000000000002"));
+
+  @Test
+  void isolatesMonetaryWindowsByCurrency() {
+    LimitKeys.Keys krw = LimitKeys.monetary(USER, LimitType.STAKE_DAILY, Currency.KRW);
+    LimitKeys.Keys usd = LimitKeys.monetary(USER, LimitType.STAKE_DAILY, Currency.USD);
+
+    assertThat(krw.entries()).contains("{00000000-0000-0000-0000-000000000001}");
+    assertThat(krw.entries()).endsWith(":stake-daily:krw:entries");
+    assertThat(usd.entries()).endsWith(":stake-daily:usd:entries");
+    assertThat(krw).isNotEqualTo(usd);
+  }
+
+  @Test
+  void keepsSelectionCapacityCurrencyNeutral() {
+    LimitKeys.Keys selections = LimitKeys.selections(USER);
+
+    assertThat(selections.entries()).endsWith(":selections-per-minute:entries");
+    assertThat(selections.sum()).endsWith(":selections-per-minute:sum");
+    assertThatThrownBy(
+            () -> LimitKeys.monetary(USER, LimitType.SELECTIONS_PER_MINUTE, Currency.KRW))
+        .isInstanceOf(IllegalArgumentException.class);
+  }
+
+  @Test
+  void encodesUuidMembersWithoutAmbiguousIdentityDelimiters() {
+    assertThat(LimitKeys.member(BET, 1500)).isEqualTo("00000000-0000-0000-0000-000000000002|1500");
+    assertThatThrownBy(() -> LimitKeys.member(BET, 0)).isInstanceOf(IllegalArgumentException.class);
+  }
+}


## `feat(counter): implement idempotent sliding windows`

diff --git a/src/main/resources/scripts/sliding-window.lua b/src/main/resources/scripts/sliding-window.lua
new file mode 100644
index 0000000..c13a2fc
--- /dev/null
+++ b/src/main/resources/scripts/sliding-window.lua
@@ -0,0 +1,64 @@
+local entriesKey = KEYS[1]
+local sumKey = KEYS[2]
+local mode = ARGV[1]
+local now = tonumber(ARGV[2])
+local window = tonumber(ARGV[3])
+local ttl = tonumber(ARGV[4])
+local maxExact = 9007199254740991
+
+if not now or not window or not ttl or window <= 0 or ttl < window then
+  return redis.error_reply("invalid sliding-window timing")
+end
+if mode ~= "READ" and mode ~= "RECORD" then
+  return redis.error_reply("invalid sliding-window mode")
+end
+if redis.call("EXISTS", entriesKey) == 0 then
+  redis.call("DEL", sumKey)
+  if mode == "READ" then
+    return {"0", "0"}
+  end
+end
+
+local stored = redis.call("GET", sumKey)
+local total = stored and tonumber(stored) or 0
+if not total or total < 0 or total > maxExact then
+  return redis.error_reply("corrupt sliding-window sum")
+end
+
+local expired = redis.call("ZRANGEBYSCORE", entriesKey, "-inf", now - window)
+for _, member in ipairs(expired) do
+  local encoded = string.match(member, "|([0-9]+)$")
+  local amount = encoded and tonumber(encoded) or nil
+  if not amount or amount <= 0 or amount > maxExact or total < amount then
+    return redis.error_reply("corrupt sliding-window member")
+  end
+  total = total - amount
+end
+if #expired > 0 then
+  redis.call("ZREM", entriesKey, unpack(expired))
+end
+
+local added = 0
+if mode == "RECORD" then
+  local member = ARGV[5]
+  local amount = tonumber(ARGV[6])
+  if not member or member == "" or not amount or amount <= 0 or amount > maxExact then
+    return redis.error_reply("invalid sliding-window entry")
+  end
+  if total > maxExact - amount then
+    return redis.error_reply("sliding-window sum exceeds exact range")
+  end
+  added = redis.call("ZADD", entriesKey, "NX", now, member)
+  if added == 1 then
+    total = total + amount
+  end
+end
+
+if redis.call("ZCARD", entriesKey) == 0 then
+  redis.call("DEL", entriesKey, sumKey)
+  total = 0
+else
+  redis.call("SET", sumKey, string.format("%.0f", total), "PX", ttl)
+  redis.call("PEXPIRE", entriesKey, ttl)
+end
+return {string.format("%.0f", total), tostring(added)}


## `test(counter): verify sliding window script semantics`

diff --git a/src/test/java/com/sportsbook/risk/counter/SlidingWindowScriptTest.java b/src/test/java/com/sportsbook/risk/counter/SlidingWindowScriptTest.java
new file mode 100644
index 0000000..a2c5748
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/counter/SlidingWindowScriptTest.java
@@ -0,0 +1,56 @@
+package com.sportsbook.risk.counter;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.risk.support.RedisTestSupport;
+import java.util.List;
+import org.junit.jupiter.api.Test;
+import org.springframework.data.redis.RedisSystemException;
+import org.springframework.data.redis.core.script.RedisScript;
+
+class SlidingWindowScriptTest extends RedisTestSupport {
+  private static final List<String> KEYS = List.of("test:entries", "test:sum");
+
+  @Test
+  void recordsOnceAndExpiresEntriesAtomically() {
+    RedisScript<List> script = script("sliding-window.lua");
+
+    assertThat(execute(script, "RECORD", 1000, 1000, "bet-a|100", 100)).containsExactly("100", "1");
+    assertThat(execute(script, "RECORD", 1200, 1000, "bet-a|100", 100)).containsExactly("100", "0");
+    assertThat(execute(script, "RECORD", 1500, 1000, "bet-b|50", 50)).containsExactly("150", "1");
+    assertThat(execute(script, "READ", 2001, 1000, "", 0)).containsExactly("50", "0");
+    assertThat(execute(script, "READ", 2501, 1000, "", 0)).containsExactly("0", "0");
+    assertThat(redis.hasKey(KEYS.get(0))).isFalse();
+  }
+
+  @Test
+  void repairsOrphanSumsAndRejectsCorruptMembers() {
+    RedisScript<List> script = script("sliding-window.lua");
+    redis.opsForValue().set(KEYS.get(1), "99");
+
+    assertThat(execute(script, "READ", 1000, 1000, "", 0)).containsExactly("0", "0");
+    redis.opsForZSet().add(KEYS.get(0), "invalid", 1);
+    redis.opsForValue().set(KEYS.get(1), "1");
+
+    assertThatThrownBy(() -> execute(script, "READ", 2000, 1000, "", 0))
+        .isInstanceOf(RedisSystemException.class)
+        .hasRootCauseMessage("corrupt sliding-window member");
+  }
+
+  @SuppressWarnings("unchecked")
+  private List<String> execute(
+      RedisScript<List> script, String mode, long now, long window, String member, long amount) {
+    return (List<String>)
+        (List<?>)
+            redis.execute(
+                script,
+                KEYS,
+                mode,
+                Long.toString(now),
+                Long.toString(window),
+                "5000",
+                member,
+                Long.toString(amount));
+  }
+}
diff --git a/src/test/java/com/sportsbook/risk/support/RedisTestSupport.java b/src/test/java/com/sportsbook/risk/support/RedisTestSupport.java
new file mode 100644
index 0000000..6a94fa1
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/support/RedisTestSupport.java
@@ -0,0 +1,44 @@
+package com.sportsbook.risk.support;
+
+import java.util.List;
+import org.junit.jupiter.api.AfterEach;
+import org.junit.jupiter.api.BeforeEach;
+import org.springframework.core.io.ClassPathResource;
+import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
+import org.springframework.data.redis.core.StringRedisTemplate;
+import org.springframework.data.redis.core.script.DefaultRedisScript;
+import org.testcontainers.containers.GenericContainer;
+import org.testcontainers.junit.jupiter.Container;
+import org.testcontainers.junit.jupiter.Testcontainers;
+import org.testcontainers.utility.DockerImageName;
+
+@Testcontainers(disabledWithoutDocker = true)
+public abstract class RedisTestSupport {
+  @Container
+  protected static final GenericContainer<?> REDIS =
+      new GenericContainer<>(DockerImageName.parse("redis:7.4-alpine")).withExposedPorts(6379);
+
+  protected StringRedisTemplate redis;
+  private LettuceConnectionFactory connectionFactory;
+
+  @BeforeEach
+  void connectRedis() {
+    connectionFactory = new LettuceConnectionFactory(REDIS.getHost(), REDIS.getFirstMappedPort());
+    connectionFactory.afterPropertiesSet();
+    redis = new StringRedisTemplate(connectionFactory);
+    redis.afterPropertiesSet();
+    redis.getConnectionFactory().getConnection().serverCommands().flushDb();
+  }
+
+  @AfterEach
+  void disconnectRedis() {
+    connectionFactory.destroy();
+  }
+
+  protected DefaultRedisScript<List> script(String name) {
+    DefaultRedisScript<List> script = new DefaultRedisScript<>();
+    script.setLocation(new ClassPathResource("scripts/" + name));
+    script.setResultType(List.class);
+    return script;
+  }
+}


## `feat(counter): expose sliding window counters`

diff --git a/src/main/java/com/sportsbook/risk/counter/RedisLuaScriptLoader.java b/src/main/java/com/sportsbook/risk/counter/RedisLuaScriptLoader.java
new file mode 100644
index 0000000..c810620
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/counter/RedisLuaScriptLoader.java
@@ -0,0 +1,17 @@
+package com.sportsbook.risk.counter;
+
+import java.util.List;
+import org.springframework.core.io.ClassPathResource;
+import org.springframework.data.redis.core.script.DefaultRedisScript;
+
+/** Loads immutable classpath Lua scripts with an explicit Redis result shape. */
+public final class RedisLuaScriptLoader {
+  private RedisLuaScriptLoader() {}
+
+  public static DefaultRedisScript<List> listScript(String name) {
+    DefaultRedisScript<List> script = new DefaultRedisScript<>();
+    script.setLocation(new ClassPathResource("scripts/" + name));
+    script.setResultType(List.class);
+    return script;
+  }
+}
diff --git a/src/main/java/com/sportsbook/risk/counter/SlidingWindowCounter.java b/src/main/java/com/sportsbook/risk/counter/SlidingWindowCounter.java
new file mode 100644
index 0000000..7fe35f8
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/counter/SlidingWindowCounter.java
@@ -0,0 +1,79 @@
+package com.sportsbook.risk.counter;
+
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.risk.policy.SafeRedisNumber;
+import java.time.Clock;
+import java.time.Duration;
+import java.time.Instant;
+import java.util.List;
+import java.util.Objects;
+import org.springframework.data.redis.core.StringRedisTemplate;
+import org.springframework.data.redis.core.script.RedisScript;
+
+/** Strict Java boundary for the atomic committed sliding-window script. */
+public final class SlidingWindowCounter {
+  private static final RedisScript<List> SCRIPT =
+      RedisLuaScriptLoader.listScript("sliding-window.lua");
+  private static final Duration TTL_MARGIN = Duration.ofMinutes(5);
+
+  private final StringRedisTemplate redis;
+  private final Clock clock;
+
+  public SlidingWindowCounter(StringRedisTemplate redis, Clock clock) {
+    this.redis = Objects.requireNonNull(redis, "redis");
+    this.clock = Objects.requireNonNull(clock, "clock");
+  }
+
+  public WindowResult record(
+      LimitKeys.Keys keys, BetId betId, long amount, Duration window, Instant at) {
+    SafeRedisNumber.requirePositive(amount, "amount");
+    return execute(keys, "RECORD", window, at, LimitKeys.member(betId, amount), amount);
+  }
+
+  public long read(LimitKeys.Keys keys, Duration window) {
+    return read(keys, window, clock.instant());
+  }
+
+  public long read(LimitKeys.Keys keys, Duration window, Instant at) {
+    return execute(keys, "READ", window, at, "", 0).total();
+  }
+
+  @SuppressWarnings("unchecked")
+  private WindowResult execute(
+      LimitKeys.Keys keys, String mode, Duration window, Instant at, String member, long amount) {
+    Objects.requireNonNull(keys, "keys");
+    Objects.requireNonNull(window, "window");
+    Objects.requireNonNull(at, "at");
+    if (window.isZero() || window.isNegative()) {
+      throw new IllegalArgumentException("window must be positive");
+    }
+    List<String> result =
+        (List<String>)
+            (List<?>)
+                redis.execute(
+                    SCRIPT,
+                    List.of(keys.entries(), keys.sum()),
+                    mode,
+                    Long.toString(at.toEpochMilli()),
+                    Long.toString(window.toMillis()),
+                    Long.toString(window.plus(TTL_MARGIN).toMillis()),
+                    member,
+                    Long.toString(amount));
+    if (result == null || result.size() != 2) {
+      throw new IllegalStateException("unexpected sliding-window result");
+    }
+    try {
+      long total = Long.parseLong(result.get(0));
+      int added = Integer.parseInt(result.get(1));
+      SafeRedisNumber.requireNonNegative(total, "total");
+      if (added != 0 && added != 1) {
+        throw new IllegalStateException("unexpected sliding-window insertion result");
+      }
+      return new WindowResult(total, added == 1);
+    } catch (NumberFormatException exception) {
+      throw new IllegalStateException("malformed sliding-window result", exception);
+    }
+  }
+
+  public record WindowResult(long total, boolean added) {}
+}


## `test(counter): verify sliding counter validation`

diff --git a/src/test/java/com/sportsbook/risk/counter/SlidingWindowCounterTest.java b/src/test/java/com/sportsbook/risk/counter/SlidingWindowCounterTest.java
new file mode 100644
index 0000000..3c4bc56
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/counter/SlidingWindowCounterTest.java
@@ -0,0 +1,68 @@
+package com.sportsbook.risk.counter;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+import static org.mockito.ArgumentMatchers.any;
+import static org.mockito.ArgumentMatchers.anyList;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.protocol.value.BetId;
+import java.time.Clock;
+import java.time.Duration;
+import java.time.Instant;
+import java.time.ZoneOffset;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.data.redis.core.StringRedisTemplate;
+import org.springframework.data.redis.core.script.RedisScript;
+
+class SlidingWindowCounterTest {
+  private static final Instant NOW = Instant.parse("2026-01-02T03:04:05Z");
+  private static final LimitKeys.Keys KEYS = new LimitKeys.Keys("entries", "sum");
+  private static final BetId BET =
+      BetId.of(UUID.fromString("00000000-0000-0000-0000-000000000002"));
+
+  @Test
+  @SuppressWarnings({"rawtypes", "unchecked"})
+  void recordsTypedEntriesAndReadsAtTheInjectedClock() {
+    StringRedisTemplate redis = mock(StringRedisTemplate.class);
+    when(redis.execute(any(RedisScript.class), anyList(), any(Object[].class)))
+        .thenReturn(List.of("75", "1"), List.of("75", "0"));
+    SlidingWindowCounter counter =
+        new SlidingWindowCounter(redis, Clock.fixed(NOW, ZoneOffset.UTC));
+
+    assertThat(counter.record(KEYS, BET, 75, Duration.ofMinutes(1), NOW))
+        .isEqualTo(new SlidingWindowCounter.WindowResult(75, true));
+    assertThat(counter.read(KEYS, Duration.ofMinutes(1))).isEqualTo(75);
+    verify(redis)
+        .execute(
+            any(RedisScript.class),
+            org.mockito.ArgumentMatchers.eq(List.of("entries", "sum")),
+            org.mockito.ArgumentMatchers.eq("RECORD"),
+            org.mockito.ArgumentMatchers.eq(Long.toString(NOW.toEpochMilli())),
+            org.mockito.ArgumentMatchers.eq("60000"),
+            org.mockito.ArgumentMatchers.eq("360000"),
+            org.mockito.ArgumentMatchers.eq(BET.value() + "|75"),
+            org.mockito.ArgumentMatchers.eq("75"));
+  }
+
+  @Test
+  @SuppressWarnings({"rawtypes", "unchecked"})
+  void rejectsInvalidInputsAndMalformedScriptResults() {
+    StringRedisTemplate redis = mock(StringRedisTemplate.class);
+    SlidingWindowCounter counter = new SlidingWindowCounter(redis, Clock.systemUTC());
+
+    assertThatThrownBy(() -> counter.read(KEYS, Duration.ZERO))
+        .isInstanceOf(IllegalArgumentException.class);
+    assertThatThrownBy(() -> counter.record(KEYS, BET, 0, Duration.ofMinutes(1), NOW))
+        .isInstanceOf(IllegalArgumentException.class);
+    when(redis.execute(any(RedisScript.class), anyList(), any(Object[].class)))
+        .thenReturn(List.of("not-a-number", "1"));
+    assertThatThrownBy(() -> counter.read(KEYS, Duration.ofMinutes(1), NOW))
+        .isInstanceOf(IllegalStateException.class)
+        .hasMessage("malformed sliding-window result");
+  }
+}


## `test(counter): verify concurrent counter convergence`

diff --git a/src/test/java/com/sportsbook/risk/counter/SlidingWindowScriptTest.java b/src/test/java/com/sportsbook/risk/counter/SlidingWindowScriptTest.java
index a2c5748..f366d90 100644
--- a/src/test/java/com/sportsbook/risk/counter/SlidingWindowScriptTest.java
+++ b/src/test/java/com/sportsbook/risk/counter/SlidingWindowScriptTest.java
@@ -5,6 +5,9 @@ import static org.assertj.core.api.Assertions.assertThatThrownBy;
 
 import com.sportsbook.risk.support.RedisTestSupport;
 import java.util.List;
+import java.util.concurrent.CompletableFuture;
+import java.util.concurrent.Executors;
+import java.util.stream.IntStream;
 import org.junit.jupiter.api.Test;
 import org.springframework.data.redis.RedisSystemException;
 import org.springframework.data.redis.core.script.RedisScript;
@@ -38,6 +41,29 @@ class SlidingWindowScriptTest extends RedisTestSupport {
         .hasRootCauseMessage("corrupt sliding-window member");
   }
 
+  @Test
+  void convergesUnderConcurrentSameTimestampWrites() {
+    RedisScript<List> script = script("sliding-window.lua");
+    var executor = Executors.newFixedThreadPool(8);
+    try {
+      List<CompletableFuture<List<String>>> writes =
+          IntStream.range(0, 20)
+              .mapToObj(
+                  index ->
+                      CompletableFuture.supplyAsync(
+                          () -> execute(script, "RECORD", 1000, 1000, "bet-" + index + "|1", 1),
+                          executor))
+              .toList();
+      CompletableFuture.allOf(writes.toArray(CompletableFuture[]::new)).join();
+
+      assertThat(writes).allSatisfy(write -> assertThat(write.join().get(1)).isEqualTo("1"));
+      assertThat(execute(script, "READ", 1000, 1000, "", 0)).containsExactly("20", "0");
+      assertThat(redis.opsForZSet().size(KEYS.get(0))).isEqualTo(20);
+    } finally {
+      executor.shutdownNow();
+    }
+  }
+
   @SuppressWarnings("unchecked")
   private List<String> execute(
       RedisScript<List> script, String mode, long now, long window, String member, long amount) {


