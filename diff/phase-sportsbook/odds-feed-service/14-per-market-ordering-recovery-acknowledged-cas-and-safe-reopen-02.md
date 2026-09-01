## `test(commands): verify pending action recovery`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
index 156bfb0..79d5999 100644
--- a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
@@ -291,10 +291,35 @@ class OperatorActionQueueTest {
     assertThat(redis.opsForStream().pending(STREAM, "group").getTotalPendingMessages()).isZero();
   }
 
+  @Test
+  void replacementConsumerReclaimsAnInterruptedDelivery() {
+    OperatorActionQueue original = queue("before-crash", Duration.ZERO);
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+    original.submit(
+        IdempotencyKey.of("pending-key"),
+        UUID.randomUUID(),
+        eventId,
+        marketId,
+        MarketStatus.SUSPENDED,
+        "incident",
+        NOW);
+    QueuedOperatorMarketAction delivered = original.poll().get(0);
+
+    QueuedOperatorMarketAction reclaimed = queue("after-restart", Duration.ZERO).poll().get(0);
+
+    assertThat(reclaimed.action()).isEqualTo(delivered.action());
+    assertThat(reclaimed.reclaimed()).isTrue();
+  }
+
   private OperatorActionQueue queue() {
+    return queue("consumer", Duration.ZERO);
+  }
+
+  private OperatorActionQueue queue(String consumer, Duration claimIdle) {
     return new OperatorActionQueue(
         redis,
-        new OperatorDeliveryProperties(STREAM, "group", "consumer", 20, Duration.ZERO, 10),
+        new OperatorDeliveryProperties(STREAM, "group", consumer, 20, claimIdle, 10),
         new CacheProperties(Duration.ofHours(24)),
         new SimpleMeterRegistry());
   }


## `feat(commands): define acknowledged completion CAS`

diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorCompletionScript.java b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorCompletionScript.java
new file mode 100644
index 0000000..f6697d8
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorCompletionScript.java
@@ -0,0 +1,68 @@
+package com.sportsbook.oddsfeed.delivery;
+
+import org.springframework.data.redis.core.script.DefaultRedisScript;
+import org.springframework.data.redis.core.script.RedisScript;
+
+/** Atomic Redis transition applied after Kafka acknowledges an operator action. */
+final class OperatorCompletionScript {
+
+  static final RedisScript<String> INSTANCE =
+      new DefaultRedisScript<>(
+          """
+          local sequence = tonumber(ARGV[1])
+          local predecessor = tonumber(ARGV[2])
+          local committed = tonumber(redis.call('GET', KEYS[1]) or '0')
+          local tail = tonumber(redis.call('GET', KEYS[2]) or '0')
+          local provider = redis.call('GET', KEYS[4]) or 'OPEN'
+          local terminal = redis.call('EXISTS', KEYS[7]) == 1
+            or redis.call('EXISTS', KEYS[8]) == 1
+          if provider == 'CLOSED' then
+            redis.call('SET', KEYS[8], 'MARKET_CLOSED', 'NX')
+            terminal = true
+          end
+          if committed >= sequence then
+            local idempotency = redis.call('GET', KEYS[6])
+            redis.call('PEXPIRE', KEYS[6], ARGV[5])
+            if idempotency then
+              redis.call('PEXPIRE', idempotency, ARGV[5])
+            end
+            if tail == committed then
+              redis.call('PEXPIRE', KEYS[1], ARGV[5])
+              redis.call('PEXPIRE', KEYS[2], ARGV[5])
+            end
+            return 'COMPLETED'
+          end
+          if committed ~= predecessor then
+            return 'BLOCKED'
+          end
+
+          redis.call('SET', KEYS[1], tostring(sequence))
+          local idempotency = redis.call('GET', KEYS[6])
+          redis.call('PEXPIRE', KEYS[6], ARGV[5])
+          if idempotency then
+            redis.call('PEXPIRE', idempotency, ARGV[5])
+          end
+          if tail ~= sequence then
+            return 'SUPERSEDED'
+          end
+
+          local requested = ARGV[3]
+          local effective = requested
+          if terminal then
+            effective = 'CLOSED'
+          elseif requested == 'OPEN' then
+            effective = redis.call('GET', KEYS[3]) or provider
+          else
+            redis.call('SET', KEYS[5], requested)
+          end
+          redis.call('PSETEX', KEYS[3], ARGV[4], effective)
+          redis.call('HSET', KEYS[10], ARGV[6], effective)
+          redis.call('PEXPIRE', KEYS[10], ARGV[4])
+          redis.call('PEXPIRE', KEYS[1], ARGV[5])
+          redis.call('PEXPIRE', KEYS[2], ARGV[5])
+          return 'APPLIED'
+          """,
+          String.class);
+
+  private OperatorCompletionScript() {}
+}


## `feat(commands): complete acknowledged operator actions`

diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java
index 824f32c..d8ad1a5 100644
--- a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java
@@ -9,6 +9,7 @@ import com.sportsbook.protocol.value.IdempotencyKey;
 import com.sportsbook.protocol.value.MarketId;
 import io.micrometer.core.instrument.MeterRegistry;
 import java.nio.charset.StandardCharsets;
+import java.time.Duration;
 import java.time.Instant;
 import java.util.ArrayList;
 import java.util.List;
@@ -37,6 +38,9 @@ public class OperatorActionQueue {
   private static final String ACTION_PREFIX = "oddsfeed:operator:action:";
   private static final String SEQUENCE_PREFIX = "oddsfeed:operator:sequence:";
   private static final String COMMITTED_PREFIX = "oddsfeed:operator:committed:";
+  private static final int MAPPING_RETENTION_DAYS = 7;
+  private static final long COMPLETED_MAPPING_TTL_MILLIS =
+      Duration.ofDays(MAPPING_RETENTION_DAYS).toMillis();
   private static final int MAX_REASON_LENGTH = 256;
 
   private final StringRedisTemplate redis;
@@ -132,6 +136,42 @@ public class OperatorActionQueue {
     }
   }
 
+  public DeliveryState deliveryState(OperatorMarketAction action) {
+    String raw = redis.opsForValue().get(committedKey(action.eventId(), action.marketId()));
+    long committed = raw == null ? 0 : Long.parseLong(raw);
+    if (committed >= action.sequence()) {
+      return DeliveryState.COMPLETED;
+    }
+    return committed == action.predecessor() ? DeliveryState.READY : DeliveryState.BLOCKED;
+  }
+
+  public Completion complete(OperatorMarketAction action) {
+    String result =
+        redis.execute(
+            OperatorCompletionScript.INSTANCE,
+            List.of(
+                committedKey(action.eventId(), action.marketId()),
+                sequenceKey(action.eventId(), action.marketId()),
+                CacheKeys.market(action.eventId(), action.marketId()),
+                CacheKeys.providerMarket(action.eventId(), action.marketId()),
+                CacheKeys.marketOverride(action.eventId(), action.marketId()),
+                actionKey(action.actionId()),
+                CacheKeys.eventTerminal(action.eventId()),
+                CacheKeys.marketTerminal(action.eventId(), action.marketId()),
+                CacheKeys.marketFeedHold(action.eventId(), action.marketId()),
+                CacheKeys.eventMarkets(action.eventId())),
+            Long.toString(action.sequence()),
+            Long.toString(action.predecessor()),
+            action.requestedStatus().name(),
+            Long.toString(marketTtlMillis),
+            Long.toString(COMPLETED_MAPPING_TTL_MILLIS),
+            action.marketId().value().toString());
+    if (result == null) {
+      throw new IllegalStateException("Operator action completion returned no result");
+    }
+    return Completion.valueOf(result);
+  }
+
   static String idempotencyRedisKey(IdempotencyKey key) {
     return IDEMPOTENCY_PREFIX + MarketActionFingerprint.idempotencyKey(key);
   }
@@ -227,4 +267,17 @@ public class OperatorActionQueue {
     }
     return false;
   }
+
+  public enum DeliveryState {
+    READY,
+    BLOCKED,
+    COMPLETED
+  }
+
+  public enum Completion {
+    APPLIED,
+    SUPERSEDED,
+    COMPLETED,
+    BLOCKED
+  }
 }


## `test(commands): verify durable acknowledged completion`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
index 79d5999..041eedd 100644
--- a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
@@ -13,6 +13,7 @@ import com.sportsbook.protocol.value.MarketId;
 import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
 import java.time.Duration;
 import java.time.Instant;
+import java.util.List;
 import java.util.UUID;
 import org.junit.jupiter.api.AfterEach;
 import org.junit.jupiter.api.BeforeEach;
@@ -312,6 +313,71 @@ class OperatorActionQueueTest {
     assertThat(reclaimed.reclaimed()).isTrue();
   }
 
+  @Test
+  void completionAdvancesPredecessorsAndExpiresFinishedMappings() {
+    OperatorActionQueue queue = queue();
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+    UUID firstActionId = UUID.randomUUID();
+    IdempotencyKey firstKey = IdempotencyKey.of("completed-first");
+    queue.submit(firstKey, firstActionId, eventId, marketId, MarketStatus.SUSPENDED, "first", NOW);
+    queue.submit(
+        IdempotencyKey.of("completed-second"),
+        UUID.randomUUID(),
+        eventId,
+        marketId,
+        MarketStatus.CLOSED,
+        "second",
+        NOW);
+    List<QueuedOperatorMarketAction> actions = queue.poll();
+
+    assertThat(queue.deliveryState(actions.get(1).action()))
+        .isEqualTo(OperatorActionQueue.DeliveryState.BLOCKED);
+    assertThat(queue.complete(actions.get(0).action()))
+        .isEqualTo(OperatorActionQueue.Completion.SUPERSEDED);
+    assertThat(redis.getExpire(OperatorActionQueue.sequenceKey(eventId, marketId))).isEqualTo(-1);
+    assertThat(redis.getExpire(OperatorActionQueue.committedKey(eventId, marketId))).isEqualTo(-1);
+    assertThat(queue.deliveryState(actions.get(1).action()))
+        .isEqualTo(OperatorActionQueue.DeliveryState.READY);
+    assertThat(queue.complete(actions.get(1).action()))
+        .isEqualTo(OperatorActionQueue.Completion.APPLIED);
+
+    assertThat(redis.getExpire(OperatorActionQueue.idempotencyRedisKey(firstKey)))
+        .isBetween(Duration.ofDays(6).toSeconds(), Duration.ofDays(7).toSeconds());
+    assertThat(redis.getExpire("oddsfeed:operator:action:" + firstActionId))
+        .isBetween(Duration.ofDays(6).toSeconds(), Duration.ofDays(7).toSeconds());
+    assertThat(redis.getExpire(OperatorActionQueue.sequenceKey(eventId, marketId)))
+        .isBetween(Duration.ofDays(6).toSeconds(), Duration.ofDays(7).toSeconds());
+    assertThat(redis.getExpire(OperatorActionQueue.committedKey(eventId, marketId)))
+        .isBetween(Duration.ofDays(6).toSeconds(), Duration.ofDays(7).toSeconds());
+    assertThat(redis.opsForValue().get(CacheKeys.market(eventId, marketId)))
+        .isEqualTo(MarketStatus.CLOSED.name());
+  }
+
+  @Test
+  void completionSurvivesACrashBeforeStreamAcknowledgement() {
+    OperatorActionQueue original = queue("before-crash", Duration.ZERO);
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+    original.submit(
+        IdempotencyKey.of("crash-key"),
+        UUID.randomUUID(),
+        eventId,
+        marketId,
+        MarketStatus.SUSPENDED,
+        "incident",
+        NOW);
+    QueuedOperatorMarketAction delivered = original.poll().get(0);
+    original.complete(delivered.action());
+
+    OperatorActionQueue replacement = queue("after-restart", Duration.ZERO);
+    QueuedOperatorMarketAction reclaimed = replacement.poll().get(0);
+    assertThat(replacement.deliveryState(reclaimed.action()))
+        .isEqualTo(OperatorActionQueue.DeliveryState.COMPLETED);
+    replacement.cleanup(reclaimed);
+    assertThat(redis.opsForStream().size(STREAM)).isZero();
+  }
+
   private OperatorActionQueue queue() {
     return queue("consumer", Duration.ZERO);
   }


## `test(commands): retain pending sequence state`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
index 041eedd..b19cb6a 100644
--- a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
@@ -378,6 +378,41 @@ class OperatorActionQueueTest {
     assertThat(redis.opsForStream().size(STREAM)).isZero();
   }
 
+  @Test
+  void newPendingActionPersistsSequenceStateBeforeItsExpiry() {
+    OperatorActionQueue queue = queue();
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+    queue.submit(
+        IdempotencyKey.of("completed-before-new"),
+        UUID.randomUUID(),
+        eventId,
+        marketId,
+        MarketStatus.SUSPENDED,
+        "incident",
+        NOW);
+    QueuedOperatorMarketAction first = queue.poll().get(0);
+    queue.complete(first.action());
+    queue.cleanup(first);
+
+    assertThat(redis.getExpire(OperatorActionQueue.sequenceKey(eventId, marketId))).isPositive();
+    assertThat(redis.getExpire(OperatorActionQueue.committedKey(eventId, marketId))).isPositive();
+
+    OperatorActionSubmission second =
+        queue.submit(
+            IdempotencyKey.of("new-before-expiry"),
+            UUID.randomUUID(),
+            eventId,
+            marketId,
+            MarketStatus.CLOSED,
+            "escalated incident",
+            NOW);
+
+    assertThat(second.sequence()).isEqualTo(2);
+    assertThat(redis.getExpire(OperatorActionQueue.sequenceKey(eventId, marketId))).isEqualTo(-1);
+    assertThat(redis.getExpire(OperatorActionQueue.committedKey(eventId, marketId))).isEqualTo(-1);
+  }
+
   private OperatorActionQueue queue() {
     return queue("consumer", Duration.ZERO);
   }


## `feat(commands): define delivery decisions`

diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorDeliveryDecision.java b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorDeliveryDecision.java
new file mode 100644
index 0000000..94351cf
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorDeliveryDecision.java
@@ -0,0 +1,22 @@
+package com.sportsbook.oddsfeed.delivery;
+
+import com.sportsbook.protocol.event.MarketStatus;
+import java.util.Objects;
+
+/** Current Redis verdict immediately before an operator action is published. */
+record OperatorDeliveryDecision(Outcome outcome, MarketStatus announcedStatus) {
+
+  OperatorDeliveryDecision {
+    Objects.requireNonNull(outcome, "outcome");
+    if ((outcome == Outcome.PUBLISH) != (announcedStatus != null)) {
+      throw new IllegalArgumentException("Only publish decisions contain a market status");
+    }
+  }
+
+  enum Outcome {
+    PUBLISH,
+    SKIP,
+    BLOCKED,
+    COMPLETED
+  }
+}
diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorDeliveryDecisionScript.java b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorDeliveryDecisionScript.java
new file mode 100644
index 0000000..1485778
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorDeliveryDecisionScript.java
@@ -0,0 +1,49 @@
+package com.sportsbook.oddsfeed.delivery;
+
+import org.springframework.data.redis.core.script.DefaultRedisScript;
+import org.springframework.data.redis.core.script.RedisScript;
+
+/** Atomically resolves whether and how a queued operator action may be published. */
+final class OperatorDeliveryDecisionScript {
+
+  static final RedisScript<String> INSTANCE =
+      new DefaultRedisScript<>(
+          """
+          local sequence = tonumber(ARGV[1])
+          local predecessor = tonumber(ARGV[2])
+          local committed = tonumber(redis.call('GET', KEYS[1]) or '0')
+          if committed >= sequence then
+            return 'COMPLETED'
+          end
+          if committed ~= predecessor then
+            return 'BLOCKED'
+          end
+
+          local tail = tonumber(redis.call('GET', KEYS[2]) or '0')
+          local provider = redis.call('GET', KEYS[3]) or 'OPEN'
+          local terminal = redis.call('EXISTS', KEYS[4]) == 1
+            or redis.call('EXISTS', KEYS[5]) == 1
+          if provider == 'CLOSED' then
+            redis.call('SET', KEYS[5], 'MARKET_CLOSED', 'NX')
+            terminal = true
+          end
+          if tail ~= sequence or (ARGV[3] == 'OPEN' and terminal) then
+            return 'SKIP'
+          end
+
+          local announced = ARGV[3]
+          if terminal then
+            announced = 'CLOSED'
+          elseif announced == 'OPEN' then
+            if redis.call('EXISTS', KEYS[6]) == 1 then
+              announced = 'SUSPENDED'
+            else
+              announced = provider
+            end
+          end
+          return 'PUBLISH|' .. announced
+          """,
+          String.class);
+
+  private OperatorDeliveryDecisionScript() {}
+}


## `feat(commands): evaluate queued operator actions`

diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java
index d8ad1a5..d22e1e6 100644
--- a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java
@@ -145,6 +145,23 @@ public class OperatorActionQueue {
     return committed == action.predecessor() ? DeliveryState.READY : DeliveryState.BLOCKED;
   }
 
+  public OperatorDeliveryDecision deliveryDecision(OperatorMarketAction action) {
+    String result =
+        redis.execute(
+            OperatorDeliveryDecisionScript.INSTANCE,
+            List.of(
+                committedKey(action.eventId(), action.marketId()),
+                sequenceKey(action.eventId(), action.marketId()),
+                CacheKeys.providerMarket(action.eventId(), action.marketId()),
+                CacheKeys.eventTerminal(action.eventId()),
+                CacheKeys.marketTerminal(action.eventId(), action.marketId()),
+                CacheKeys.marketFeedHold(action.eventId(), action.marketId())),
+            Long.toString(action.sequence()),
+            Long.toString(action.predecessor()),
+            action.requestedStatus().name());
+    return parseDeliveryDecision(result);
+  }
+
   public Completion complete(OperatorMarketAction action) {
     String result =
         redis.execute(
@@ -268,6 +285,18 @@ public class OperatorActionQueue {
     return false;
   }
 
+  private static OperatorDeliveryDecision parseDeliveryDecision(String result) {
+    if (result == null) {
+      throw new IllegalStateException("Operator delivery decision returned no result");
+    }
+    if (result.startsWith("PUBLISH|")) {
+      return new OperatorDeliveryDecision(
+          OperatorDeliveryDecision.Outcome.PUBLISH,
+          MarketStatus.valueOf(result.substring("PUBLISH|".length())));
+    }
+    return new OperatorDeliveryDecision(OperatorDeliveryDecision.Outcome.valueOf(result), null);
+  }
+
   public enum DeliveryState {
     READY,
     BLOCKED,


## `test(commands): skip invalid reopen deliveries`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
index b19cb6a..e3d699e 100644
--- a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
@@ -413,6 +413,70 @@ class OperatorActionQueueTest {
     assertThat(redis.getExpire(OperatorActionQueue.committedKey(eventId, marketId))).isEqualTo(-1);
   }
 
+  @Test
+  void deliveryDecisionSkipsAReopenAfterATerminalLatch() {
+    OperatorActionQueue queue = queue();
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+    redis.opsForValue().set(CacheKeys.providerMarket(eventId, marketId), MarketStatus.OPEN.name());
+    redis
+        .opsForValue()
+        .set(CacheKeys.marketOverride(eventId, marketId), MarketStatus.SUSPENDED.name());
+    queue.submit(
+        IdempotencyKey.of("terminal-after-submit"),
+        UUID.randomUUID(),
+        eventId,
+        marketId,
+        MarketStatus.OPEN,
+        "review complete",
+        NOW);
+    QueuedOperatorMarketAction reopen = queue.poll().get(0);
+
+    redis.opsForValue().set(CacheKeys.marketTerminal(eventId, marketId), "MARKET_CLOSED");
+
+    assertThat(queue.deliveryDecision(reopen.action()).outcome())
+        .isEqualTo(OperatorDeliveryDecision.Outcome.SKIP);
+    assertThat(queue.complete(reopen.action())).isEqualTo(OperatorActionQueue.Completion.APPLIED);
+    assertThat(redis.opsForValue().get(CacheKeys.market(eventId, marketId)))
+        .isEqualTo(MarketStatus.CLOSED.name());
+    assertThat(redis.hasKey(CacheKeys.marketOverride(eventId, marketId))).isTrue();
+  }
+
+  @Test
+  void deliveryDecisionSkipsAReopenSupersededByANewerClose() {
+    OperatorActionQueue queue = queue();
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+    redis.opsForValue().set(CacheKeys.providerMarket(eventId, marketId), MarketStatus.OPEN.name());
+    redis
+        .opsForValue()
+        .set(CacheKeys.marketOverride(eventId, marketId), MarketStatus.SUSPENDED.name());
+    queue.submit(
+        IdempotencyKey.of("superseded-reopen"),
+        UUID.randomUUID(),
+        eventId,
+        marketId,
+        MarketStatus.OPEN,
+        "review complete",
+        NOW);
+    queue.submit(
+        IdempotencyKey.of("newer-close"),
+        UUID.randomUUID(),
+        eventId,
+        marketId,
+        MarketStatus.CLOSED,
+        "incident returned",
+        NOW);
+    List<QueuedOperatorMarketAction> actions = queue.poll();
+
+    assertThat(queue.deliveryDecision(actions.get(0).action()).outcome())
+        .isEqualTo(OperatorDeliveryDecision.Outcome.SKIP);
+    assertThat(queue.complete(actions.get(0).action()))
+        .isEqualTo(OperatorActionQueue.Completion.SUPERSEDED);
+    assertThat(queue.deliveryDecision(actions.get(1).action()).announcedStatus())
+        .isEqualTo(MarketStatus.CLOSED);
+  }
+
   private OperatorActionQueue queue() {
     return queue("consumer", Duration.ZERO);
   }


