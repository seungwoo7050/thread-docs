# 마켓별 순서 보장·중단 복구·승인 CAS와 안전한 재개

## `feat(commands): decode operator stream records`

diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionCodec.java b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionCodec.java
new file mode 100644
index 0000000..db086b5
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionCodec.java
@@ -0,0 +1,38 @@
+package com.sportsbook.oddsfeed.delivery;
+
+import com.sportsbook.protocol.event.MarketStatus;
+import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.MarketId;
+import java.time.Instant;
+import java.util.Map;
+import java.util.UUID;
+import org.springframework.data.redis.connection.stream.MapRecord;
+
+/** Decodes durable operator Stream records into domain actions. */
+final class OperatorActionCodec {
+
+  QueuedOperatorMarketAction decode(MapRecord<String, String, String> record, boolean reclaimed) {
+    Map<String, String> values = record.getValue();
+    OperatorMarketAction action =
+        new OperatorMarketAction(
+            UUID.fromString(require(values, "actionId")),
+            new EventId(UUID.fromString(require(values, "eventId"))),
+            new MarketId(UUID.fromString(require(values, "marketId"))),
+            MarketStatus.valueOf(require(values, "previousStatus")),
+            MarketStatus.valueOf(require(values, "announcedStatus")),
+            MarketStatus.valueOf(require(values, "requestedStatus")),
+            require(values, "reason"),
+            Long.parseLong(require(values, "sequence")),
+            Long.parseLong(require(values, "predecessor")),
+            Instant.ofEpochMilli(Long.parseLong(require(values, "occurredAt"))));
+    return new QueuedOperatorMarketAction(record.getId(), action, reclaimed);
+  }
+
+  private static String require(Map<String, String> values, String field) {
+    String value = values.get(field);
+    if (value == null) {
+      throw new IllegalStateException("Operator action is missing " + field);
+    }
+    return value;
+  }
+}
diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/QueuedOperatorMarketAction.java b/src/main/java/com/sportsbook/oddsfeed/delivery/QueuedOperatorMarketAction.java
new file mode 100644
index 0000000..41ca339
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/QueuedOperatorMarketAction.java
@@ -0,0 +1,7 @@
+package com.sportsbook.oddsfeed.delivery;
+
+import org.springframework.data.redis.connection.stream.RecordId;
+
+/** A decoded operator action and its Stream delivery identity. */
+public record QueuedOperatorMarketAction(
+    RecordId recordId, OperatorMarketAction action, boolean reclaimed) {}


## `test(commands): verify operator stream decoding`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionCodecTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionCodecTest.java
new file mode 100644
index 0000000..ada43fe
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionCodecTest.java
@@ -0,0 +1,73 @@
+package com.sportsbook.oddsfeed.delivery;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.protocol.event.MarketStatus;
+import java.time.Instant;
+import java.util.HashMap;
+import java.util.Map;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.data.redis.connection.stream.MapRecord;
+import org.springframework.data.redis.connection.stream.RecordId;
+import org.springframework.data.redis.connection.stream.StreamRecords;
+
+class OperatorActionCodecTest {
+
+  private static final UUID ACTION_ID = UUID.randomUUID();
+  private static final UUID EVENT_ID = UUID.randomUUID();
+  private static final UUID MARKET_ID = UUID.randomUUID();
+  private static final Instant OCCURRED_AT = Instant.parse("2026-08-21T05:00:00Z");
+
+  private final OperatorActionCodec codec = new OperatorActionCodec();
+
+  @Test
+  void decodesEveryPersistedField() {
+    QueuedOperatorMarketAction queued = codec.decode(record(fields()), true);
+
+    assertThat(queued.recordId()).isEqualTo(RecordId.of("1-0"));
+    assertThat(queued.reclaimed()).isTrue();
+    assertThat(queued.action().actionId()).isEqualTo(ACTION_ID);
+    assertThat(queued.action().eventId().value()).isEqualTo(EVENT_ID);
+    assertThat(queued.action().marketId().value()).isEqualTo(MARKET_ID);
+    assertThat(queued.action().previousStatus()).isEqualTo(MarketStatus.OPEN);
+    assertThat(queued.action().announcedStatus()).isEqualTo(MarketStatus.SUSPENDED);
+    assertThat(queued.action().requestedStatus()).isEqualTo(MarketStatus.SUSPENDED);
+    assertThat(queued.action().reason()).isEqualTo("incident");
+    assertThat(queued.action().sequence()).isEqualTo(2);
+    assertThat(queued.action().predecessor()).isEqualTo(1);
+    assertThat(queued.action().occurredAt()).isEqualTo(OCCURRED_AT);
+  }
+
+  @Test
+  void rejectsMissingRequiredFields() {
+    Map<String, String> fields = new HashMap<>(fields());
+    fields.remove("reason");
+
+    assertThatThrownBy(() -> codec.decode(record(fields), false))
+        .isInstanceOf(IllegalStateException.class)
+        .hasMessageContaining("reason");
+  }
+
+  private static Map<String, String> fields() {
+    return Map.of(
+        "actionId", ACTION_ID.toString(),
+        "eventId", EVENT_ID.toString(),
+        "marketId", MARKET_ID.toString(),
+        "previousStatus", "OPEN",
+        "announcedStatus", "SUSPENDED",
+        "requestedStatus", "SUSPENDED",
+        "reason", "incident",
+        "sequence", "2",
+        "predecessor", "1",
+        "occurredAt", Long.toString(OCCURRED_AT.toEpochMilli()));
+  }
+
+  private static MapRecord<String, String, String> record(Map<String, String> fields) {
+    return StreamRecords.newRecord()
+        .in("operator-actions")
+        .withId(RecordId.of("1-0"))
+        .ofMap(fields);
+  }
+}


## `feat(commands): chain per-market operator actions`

diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java
index 55badf6..333dc61 100644
--- a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java
@@ -20,6 +20,8 @@ public class OperatorActionQueue {
 
   private static final String IDEMPOTENCY_PREFIX = "oddsfeed:operator:idempotency:";
   private static final String ACTION_PREFIX = "oddsfeed:operator:action:";
+  private static final String SEQUENCE_PREFIX = "oddsfeed:operator:sequence:";
+  private static final String COMMITTED_PREFIX = "oddsfeed:operator:committed:";
   private static final int MAX_REASON_LENGTH = 256;
 
   private final StringRedisTemplate redis;
@@ -61,6 +63,8 @@ public class OperatorActionQueue {
                 CacheKeys.eventTerminal(eventId),
                 CacheKeys.marketTerminal(eventId, marketId),
                 CacheKeys.eventMarkets(eventId),
+                sequenceKey(eventId, marketId),
+                committedKey(eventId, marketId),
                 properties.streamKey()),
             fingerprint,
             actionId.toString(),
@@ -83,6 +87,14 @@ public class OperatorActionQueue {
     return IDEMPOTENCY_PREFIX + MarketActionFingerprint.idempotencyKey(key);
   }
 
+  static String sequenceKey(EventId eventId, MarketId marketId) {
+    return SEQUENCE_PREFIX + eventId.value() + ":" + marketId.value();
+  }
+
+  static String committedKey(EventId eventId, MarketId marketId) {
+    return COMMITTED_PREFIX + eventId.value() + ":" + marketId.value();
+  }
+
   private static String actionKey(UUID actionId) {
     return ACTION_PREFIX + actionId;
   }
diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorSubmissionScript.java b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorSubmissionScript.java
index 0663815..2ec0864 100644
--- a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorSubmissionScript.java
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorSubmissionScript.java
@@ -54,8 +54,12 @@ final class OperatorSubmissionScript {
           end
           redis.call('PEXPIRE', KEYS[9], ARGV[8])
 
+          local sequence = redis.call('INCR', KEYS[10])
+          redis.call('PERSIST', KEYS[10])
+          redis.call('PERSIST', KEYS[11])
+          local predecessor = sequence - 1
           local record = redis.call(
-            'XADD', KEYS[10], '*',
+            'XADD', KEYS[12], '*',
             'fingerprint', ARGV[1],
             'actionId', ARGV[2],
             'eventId', ARGV[3],
@@ -65,12 +69,13 @@ final class OperatorSubmissionScript {
             'announcedStatus', announced,
             'reason', ARGV[6],
             'occurredAt', ARGV[7],
-            'sequence', '0',
-            'predecessor', '-1')
-          local metadata = ARGV[1] .. '|' .. ARGV[2] .. '|0|-1|' .. record
+            'sequence', tostring(sequence),
+            'predecessor', tostring(predecessor))
+          local metadata = ARGV[1] .. '|' .. ARGV[2] .. '|' .. sequence
+            .. '|' .. predecessor .. '|' .. record
           redis.call('SET', KEYS[1], metadata)
           redis.call('SET', KEYS[2], KEYS[1])
-          return 'CREATED|' .. ARGV[2] .. '|0|-1|' .. record
+          return 'CREATED|' .. ARGV[2] .. '|' .. sequence .. '|' .. predecessor .. '|' .. record
           """,
           String.class);
 


## `test(commands): verify market action order`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
index 0e125fb..b5c5895 100644
--- a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
@@ -228,6 +228,40 @@ class OperatorActionQueueTest {
     assertThat(record.getValue()).containsEntry("announcedStatus", MarketStatus.SUSPENDED.name());
   }
 
+  @Test
+  void actionsReceiveMonotonicPerMarketPredecessors() {
+    OperatorActionQueue queue = queue();
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+
+    OperatorActionSubmission first =
+        queue.submit(
+            IdempotencyKey.of("first-key"),
+            UUID.randomUUID(),
+            eventId,
+            marketId,
+            MarketStatus.SUSPENDED,
+            "first",
+            NOW);
+    OperatorActionSubmission second =
+        queue.submit(
+            IdempotencyKey.of("second-key"),
+            UUID.randomUUID(),
+            eventId,
+            marketId,
+            MarketStatus.CLOSED,
+            "second",
+            NOW);
+
+    assertThat(first.sequence()).isEqualTo(1);
+    assertThat(first.predecessor()).isZero();
+    assertThat(second.sequence()).isEqualTo(2);
+    assertThat(second.predecessor()).isEqualTo(first.sequence());
+    assertThat(redis.opsForValue().get(OperatorActionQueue.sequenceKey(eventId, marketId)))
+        .isEqualTo("2");
+    assertThat(redis.getExpire(OperatorActionQueue.sequenceKey(eventId, marketId))).isEqualTo(-1);
+  }
+
   private OperatorActionQueue queue() {
     return new OperatorActionQueue(
         redis,


## `feat(commands): define atomic stream cleanup`

diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorStreamCleanupScript.java b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorStreamCleanupScript.java
new file mode 100644
index 0000000..78104d7
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorStreamCleanupScript.java
@@ -0,0 +1,19 @@
+package com.sportsbook.oddsfeed.delivery;
+
+import org.springframework.data.redis.core.script.DefaultRedisScript;
+import org.springframework.data.redis.core.script.RedisScript;
+
+/** Atomically removes a delivered action from its consumer group and Stream. */
+final class OperatorStreamCleanupScript {
+
+  static final RedisScript<String> INSTANCE =
+      new DefaultRedisScript<>(
+          """
+          local acknowledged = redis.call('XACK', KEYS[1], ARGV[1], ARGV[2])
+          local deleted = redis.call('XDEL', KEYS[1], ARGV[2])
+          return tostring(acknowledged) .. '|' .. tostring(deleted)
+          """,
+          String.class);
+
+  private OperatorStreamCleanupScript() {}
+}


## `feat(commands): consume queued operator actions`

diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java
index 333dc61..2d58b52 100644
--- a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java
@@ -8,9 +8,19 @@ import com.sportsbook.protocol.value.EventId;
 import com.sportsbook.protocol.value.IdempotencyKey;
 import com.sportsbook.protocol.value.MarketId;
 import io.micrometer.core.instrument.MeterRegistry;
+import java.nio.charset.StandardCharsets;
 import java.time.Instant;
 import java.util.List;
 import java.util.UUID;
+import java.util.concurrent.atomic.AtomicBoolean;
+import org.springframework.dao.DataAccessException;
+import org.springframework.data.redis.connection.stream.Consumer;
+import org.springframework.data.redis.connection.stream.MapRecord;
+import org.springframework.data.redis.connection.stream.ReadOffset;
+import org.springframework.data.redis.connection.stream.StreamOffset;
+import org.springframework.data.redis.connection.stream.StreamReadOptions;
+import org.springframework.data.redis.core.RedisCallback;
+import org.springframework.data.redis.core.StreamOperations;
 import org.springframework.data.redis.core.StringRedisTemplate;
 import org.springframework.stereotype.Component;
 
@@ -26,7 +36,9 @@ public class OperatorActionQueue {
 
   private final StringRedisTemplate redis;
   private final OperatorDeliveryProperties properties;
+  private final OperatorActionCodec codec = new OperatorActionCodec();
   private final long marketTtlMillis;
+  private final AtomicBoolean groupReady = new AtomicBoolean();
 
   public OperatorActionQueue(
       StringRedisTemplate redis,
@@ -83,6 +95,32 @@ public class OperatorActionQueue {
     return OperatorActionSubmission.fromRedis(result);
   }
 
+  public List<QueuedOperatorMarketAction> poll() {
+    ensureGroup();
+    List<MapRecord<String, String, String>> records =
+        streamOperations()
+            .read(
+                Consumer.from(properties.consumerGroup(), properties.consumerName()),
+                StreamReadOptions.empty().count(properties.batchSize()),
+                StreamOffset.create(properties.streamKey(), ReadOffset.lastConsumed()));
+    if (records == null || records.isEmpty()) {
+      return List.of();
+    }
+    return records.stream().map(record -> codec.decode(record, false)).toList();
+  }
+
+  public void cleanup(QueuedOperatorMarketAction queued) {
+    String result =
+        redis.execute(
+            OperatorStreamCleanupScript.INSTANCE,
+            List.of(properties.streamKey()),
+            properties.consumerGroup(),
+            queued.recordId().getValue());
+    if (result == null || !result.matches("[01]\\|[01]")) {
+      throw new IllegalStateException("Malformed operator Stream cleanup result");
+    }
+  }
+
   static String idempotencyRedisKey(IdempotencyKey key) {
     return IDEMPOTENCY_PREFIX + MarketActionFingerprint.idempotencyKey(key);
   }
@@ -109,4 +147,42 @@ public class OperatorActionQueue {
     }
     return normalized;
   }
+
+  private void ensureGroup() {
+    if (groupReady.get()) {
+      return;
+    }
+    try {
+      redis.execute(
+          (RedisCallback<String>)
+              connection ->
+                  connection
+                      .streamCommands()
+                      .xGroupCreate(
+                          properties.streamKey().getBytes(StandardCharsets.UTF_8),
+                          properties.consumerGroup(),
+                          ReadOffset.from("0-0"),
+                          true));
+    } catch (DataAccessException exception) {
+      if (!containsBusyGroup(exception)) {
+        throw exception;
+      }
+    }
+    groupReady.set(true);
+  }
+
+  private StreamOperations<String, String, String> streamOperations() {
+    return redis.<String, String>opsForStream();
+  }
+
+  private static boolean containsBusyGroup(Throwable error) {
+    Throwable current = error;
+    while (current != null) {
+      if (current.getMessage() != null && current.getMessage().contains("BUSYGROUP")) {
+        return true;
+      }
+      current = current.getCause();
+    }
+    return false;
+  }
 }


## `test(commands): verify operator stream consumption`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
index b5c5895..156bfb0 100644
--- a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
@@ -262,6 +262,35 @@ class OperatorActionQueueTest {
     assertThat(redis.getExpire(OperatorActionQueue.sequenceKey(eventId, marketId))).isEqualTo(-1);
   }
 
+  @Test
+  void pollDecodesAndAtomicallyCleansUpNewActions() {
+    OperatorActionQueue queue = queue();
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+    UUID actionId = UUID.randomUUID();
+    queue.submit(
+        IdempotencyKey.of("poll-key"),
+        actionId,
+        eventId,
+        marketId,
+        MarketStatus.SUSPENDED,
+        "incident",
+        NOW);
+
+    QueuedOperatorMarketAction queued = queue.poll().get(0);
+
+    assertThat(queued.reclaimed()).isFalse();
+    assertThat(queued.action().actionId()).isEqualTo(actionId);
+    assertThat(queued.action().eventId()).isEqualTo(eventId);
+    assertThat(queued.action().marketId()).isEqualTo(marketId);
+    assertThat(queued.action().announcedStatus()).isEqualTo(MarketStatus.SUSPENDED);
+    assertThat(queued.action().sequence()).isEqualTo(1);
+    assertThat(queued.action().predecessor()).isZero();
+    queue.cleanup(queued);
+    assertThat(redis.opsForStream().size(STREAM)).isZero();
+    assertThat(redis.opsForStream().pending(STREAM, "group").getTotalPendingMessages()).isZero();
+  }
+
   private OperatorActionQueue queue() {
     return new OperatorActionQueue(
         redis,


## `feat(commands): reclaim interrupted operator deliveries`

diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java
index 2d58b52..824f32c 100644
--- a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java
@@ -10,13 +10,18 @@ import com.sportsbook.protocol.value.MarketId;
 import io.micrometer.core.instrument.MeterRegistry;
 import java.nio.charset.StandardCharsets;
 import java.time.Instant;
+import java.util.ArrayList;
 import java.util.List;
 import java.util.UUID;
 import java.util.concurrent.atomic.AtomicBoolean;
 import org.springframework.dao.DataAccessException;
+import org.springframework.data.domain.Range;
 import org.springframework.data.redis.connection.stream.Consumer;
 import org.springframework.data.redis.connection.stream.MapRecord;
+import org.springframework.data.redis.connection.stream.PendingMessage;
+import org.springframework.data.redis.connection.stream.PendingMessages;
 import org.springframework.data.redis.connection.stream.ReadOffset;
+import org.springframework.data.redis.connection.stream.RecordId;
 import org.springframework.data.redis.connection.stream.StreamOffset;
 import org.springframework.data.redis.connection.stream.StreamReadOptions;
 import org.springframework.data.redis.core.RedisCallback;
@@ -97,16 +102,22 @@ public class OperatorActionQueue {
 
   public List<QueuedOperatorMarketAction> poll() {
     ensureGroup();
-    List<MapRecord<String, String, String>> records =
-        streamOperations()
-            .read(
-                Consumer.from(properties.consumerGroup(), properties.consumerName()),
-                StreamReadOptions.empty().count(properties.batchSize()),
-                StreamOffset.create(properties.streamKey(), ReadOffset.lastConsumed()));
+    PendingMessages pending = pendingMessages();
+    List<MapRecord<String, String, String>> records = claimExpired(pending);
+    boolean reclaimed = !records.isEmpty();
+    if (records.isEmpty() && pending.isEmpty()) {
+      records =
+          streamOperations()
+              .read(
+                  Consumer.from(properties.consumerGroup(), properties.consumerName()),
+                  StreamReadOptions.empty().count(properties.batchSize()),
+                  StreamOffset.create(properties.streamKey(), ReadOffset.lastConsumed()));
+    }
     if (records == null || records.isEmpty()) {
       return List.of();
     }
-    return records.stream().map(record -> codec.decode(record, false)).toList();
+    boolean reclaimedRecord = reclaimed;
+    return records.stream().map(record -> codec.decode(record, reclaimedRecord)).toList();
   }
 
   public void cleanup(QueuedOperatorMarketAction queued) {
@@ -171,6 +182,37 @@ public class OperatorActionQueue {
     groupReady.set(true);
   }
 
+  private PendingMessages pendingMessages() {
+    return streamOperations()
+        .pending(
+            properties.streamKey(),
+            properties.consumerGroup(),
+            Range.unbounded(),
+            properties.batchSize());
+  }
+
+  private List<MapRecord<String, String, String>> claimExpired(PendingMessages pending) {
+    List<RecordId> claimable = new ArrayList<>();
+    for (PendingMessage message : pending) {
+      if (message.getElapsedTimeSinceLastDelivery().compareTo(properties.claimIdle()) < 0) {
+        break;
+      }
+      claimable.add(message.getId());
+    }
+    if (claimable.isEmpty()) {
+      return List.of();
+    }
+    List<MapRecord<String, String, String>> claimed =
+        streamOperations()
+            .claim(
+                properties.streamKey(),
+                properties.consumerGroup(),
+                properties.consumerName(),
+                properties.claimIdle(),
+                claimable.toArray(RecordId[]::new));
+    return claimed == null ? List.of() : claimed;
+  }
+
   private StreamOperations<String, String, String> streamOperations() {
     return redis.<String, String>opsForStream();
   }


