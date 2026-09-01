## `feat(delivery): reclaim interrupted deliveries`

diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEventQueue.java b/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEventQueue.java
index 6947530..bbeadaf 100644
--- a/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEventQueue.java
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEventQueue.java
@@ -6,13 +6,17 @@ import com.sportsbook.oddsfeed.config.CriticalDeliveryProperties;
 import io.micrometer.core.instrument.Counter;
 import io.micrometer.core.instrument.MeterRegistry;
 import java.nio.charset.StandardCharsets;
+import java.util.ArrayList;
 import java.util.List;
 import java.util.Map;
 import java.util.concurrent.atomic.AtomicBoolean;
 import java.util.concurrent.atomic.AtomicLong;
 import org.springframework.dao.DataAccessException;
+import org.springframework.data.domain.Range;
 import org.springframework.data.redis.connection.stream.Consumer;
 import org.springframework.data.redis.connection.stream.MapRecord;
+import org.springframework.data.redis.connection.stream.PendingMessage;
+import org.springframework.data.redis.connection.stream.PendingMessages;
 import org.springframework.data.redis.connection.stream.ReadOffset;
 import org.springframework.data.redis.connection.stream.RecordId;
 import org.springframework.data.redis.connection.stream.StreamOffset;
@@ -31,6 +35,7 @@ public class CriticalEventQueue {
   private final ObjectMapper objectMapper;
   private final CriticalDeliveryProperties properties;
   private final Counter enqueued;
+  private final Counter reclaimed;
   private final Counter failures;
   private final AtomicBoolean healthy = new AtomicBoolean(true);
   private final AtomicLong pendingCount = new AtomicLong();
@@ -45,6 +50,7 @@ public class CriticalEventQueue {
     this.objectMapper = objectMapper;
     this.properties = properties;
     this.enqueued = meterRegistry.counter("oddsfeed.critical.delivery.enqueued");
+    this.reclaimed = meterRegistry.counter("oddsfeed.critical.delivery.reclaimed");
     this.failures = meterRegistry.counter("oddsfeed.critical.delivery.failure");
     meterRegistry.gauge("oddsfeed.critical.delivery.pending", pendingCount);
   }
@@ -72,18 +78,30 @@ public class CriticalEventQueue {
   public List<QueuedCriticalEvent> poll() {
     try {
       ensureGroup();
-      List<MapRecord<String, String, String>> records =
-          streamOperations()
-              .read(
-                  Consumer.from(properties.consumerGroup(), properties.consumerName()),
-                  StreamReadOptions.empty().count(properties.batchSize()),
-                  StreamOffset.create(properties.streamKey(), ReadOffset.lastConsumed()));
+      PendingMessages pending = pendingMessages();
+      List<MapRecord<String, String, String>> records = claimExpired(pending);
+      if (records == null) {
+        records = List.of();
+      }
+      boolean wereReclaimed = !records.isEmpty();
+      if (records.isEmpty() && pending.isEmpty()) {
+        records =
+            streamOperations()
+                .read(
+                    Consumer.from(properties.consumerGroup(), properties.consumerName()),
+                    StreamReadOptions.empty().count(properties.batchSize()),
+                    StreamOffset.create(properties.streamKey(), ReadOffset.lastConsumed()));
+      }
+      updatePendingCount();
       healthy.set(true);
       if (records == null || records.isEmpty()) {
         return List.of();
       }
-      pendingCount.addAndGet(records.size());
-      return records.stream().map(record -> decode(record, false)).toList();
+      if (wereReclaimed) {
+        reclaimed.increment(records.size());
+      }
+      boolean reclaimedRecord = wereReclaimed;
+      return records.stream().map(record -> decode(record, reclaimedRecord)).toList();
     } catch (RuntimeException error) {
       groupReady.set(false);
       healthy.set(false);
@@ -104,6 +122,42 @@ public class CriticalEventQueue {
     return redis.<String, String>opsForStream();
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
+    return streamOperations()
+        .claim(
+            properties.streamKey(),
+            properties.consumerGroup(),
+            properties.consumerName(),
+            properties.claimIdle(),
+            claimable.toArray(RecordId[]::new));
+  }
+
+  private void updatePendingCount() {
+    pendingCount.set(
+        streamOperations()
+            .pending(properties.streamKey(), properties.consumerGroup())
+            .getTotalPendingMessages());
+  }
+
   private QueuedCriticalEvent decode(MapRecord<String, String, String> record, boolean reclaimed) {
     String payload = record.getValue().get(PAYLOAD_FIELD);
     if (payload == null) {


## `test(delivery): verify pending event recovery`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventQueueTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventQueueTest.java
index 8e4b9f2..ef8731e 100644
--- a/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventQueueTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventQueueTest.java
@@ -78,6 +78,21 @@ class CriticalEventQueueTest {
     assertThat(queue.poll()).isEmpty();
   }
 
+  @Test
+  void reclaimsPendingEventsForAReplacementConsumer() {
+    queue.enqueue(lifecycle(EventLifecycleStatus.SCHEDULED));
+    QueuedCriticalEvent firstDelivery = queue.poll().get(0);
+
+    CriticalEventQueue replacement = queue("publisher-2", Duration.ZERO);
+    QueuedCriticalEvent recovered = replacement.poll().get(0);
+
+    assertThat(firstDelivery.reclaimed()).isFalse();
+    assertThat(recovered.reclaimed()).isTrue();
+    assertThat(recovered.recordId()).isEqualTo(firstDelivery.recordId());
+    assertThat(recovered.event()).isEqualTo(firstDelivery.event());
+    assertThat(replacement.pendingCount()).isEqualTo(1);
+  }
+
   private CriticalEventQueue queue(String consumerName, Duration claimIdle) {
     return new CriticalEventQueue(
         redis,


## `feat(delivery): acknowledge completed deliveries`

diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEventQueue.java b/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEventQueue.java
index bbeadaf..762d7bc 100644
--- a/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEventQueue.java
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEventQueue.java
@@ -24,17 +24,27 @@ import org.springframework.data.redis.connection.stream.StreamReadOptions;
 import org.springframework.data.redis.core.RedisCallback;
 import org.springframework.data.redis.core.StreamOperations;
 import org.springframework.data.redis.core.StringRedisTemplate;
+import org.springframework.data.redis.core.script.RedisScript;
 import org.springframework.stereotype.Component;
 
 @Component
 public class CriticalEventQueue {
 
   private static final String PAYLOAD_FIELD = "payload";
+  private static final RedisScript<Long> ACKNOWLEDGE =
+      RedisScript.of(
+          """
+          local acknowledged = redis.call('XACK', KEYS[1], ARGV[1], ARGV[2])
+          redis.call('XDEL', KEYS[1], ARGV[2])
+          return acknowledged
+          """,
+          Long.class);
 
   private final StringRedisTemplate redis;
   private final ObjectMapper objectMapper;
   private final CriticalDeliveryProperties properties;
   private final Counter enqueued;
+  private final Counter acknowledged;
   private final Counter reclaimed;
   private final Counter failures;
   private final AtomicBoolean healthy = new AtomicBoolean(true);
@@ -50,6 +60,7 @@ public class CriticalEventQueue {
     this.objectMapper = objectMapper;
     this.properties = properties;
     this.enqueued = meterRegistry.counter("oddsfeed.critical.delivery.enqueued");
+    this.acknowledged = meterRegistry.counter("oddsfeed.critical.delivery.acknowledged");
     this.reclaimed = meterRegistry.counter("oddsfeed.critical.delivery.reclaimed");
     this.failures = meterRegistry.counter("oddsfeed.critical.delivery.failure");
     meterRegistry.gauge("oddsfeed.critical.delivery.pending", pendingCount);
@@ -110,6 +121,26 @@ public class CriticalEventQueue {
     }
   }
 
+  public void acknowledge(QueuedCriticalEvent queued) {
+    try {
+      Long count =
+          redis.execute(
+              ACKNOWLEDGE,
+              List.of(properties.streamKey()),
+              properties.consumerGroup(),
+              queued.recordId().getValue());
+      if (count != null && count > 0) {
+        acknowledged.increment();
+        pendingCount.updateAndGet(current -> Math.max(0, current - 1));
+      }
+      healthy.set(true);
+    } catch (DataAccessException error) {
+      healthy.set(false);
+      failures.increment();
+      throw error;
+    }
+  }
+
   public boolean isHealthy() {
     return healthy.get();
   }


## `test(delivery): verify acknowledgement cleanup`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventQueueTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventQueueTest.java
index ef8731e..38ff1e8 100644
--- a/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventQueueTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventQueueTest.java
@@ -93,6 +93,19 @@ class CriticalEventQueueTest {
     assertThat(replacement.pendingCount()).isEqualTo(1);
   }
 
+  @Test
+  void acknowledgesAndDeletesCompletedRecords() {
+    queue.enqueue(lifecycle(EventLifecycleStatus.SCHEDULED));
+    QueuedCriticalEvent queued = queue.poll().get(0);
+
+    queue.acknowledge(queued);
+
+    assertThat(queue.pendingCount()).isZero();
+    assertThat(redis.opsForStream().pending(streamKey, "publisher").getTotalPendingMessages())
+        .isZero();
+    assertThat(redis.opsForStream().size(streamKey)).isZero();
+  }
+
   private CriticalEventQueue queue(String consumerName, Duration claimIdle) {
     return new CriticalEventQueue(
         redis,


## `feat(delivery): drain acknowledged events`

diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessor.java b/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessor.java
new file mode 100644
index 0000000..2f2c747
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessor.java
@@ -0,0 +1,55 @@
+package com.sportsbook.oddsfeed.delivery;
+
+import com.sportsbook.oddsfeed.api.EventCatalog;
+import com.sportsbook.oddsfeed.cache.RedisOddsCache;
+import com.sportsbook.oddsfeed.publisher.OddsFeedPublisher;
+import java.util.concurrent.atomic.AtomicBoolean;
+import org.springframework.scheduling.annotation.Scheduled;
+import org.springframework.stereotype.Component;
+
+@Component
+public class CriticalEventProcessor {
+
+  private final CriticalEventQueue queue;
+  protected final OddsFeedPublisher publisher;
+  protected final RedisOddsCache cache;
+  protected final EventCatalog catalog;
+  private final AtomicBoolean healthy = new AtomicBoolean(true);
+
+  public CriticalEventProcessor(
+      CriticalEventQueue queue,
+      OddsFeedPublisher publisher,
+      RedisOddsCache cache,
+      EventCatalog catalog) {
+    this.queue = queue;
+    this.publisher = publisher;
+    this.cache = cache;
+    this.catalog = catalog;
+  }
+
+  @Scheduled(fixedDelayString = "${oddsfeed.delivery.poll-interval-ms:250}")
+  public void drain() {
+    try {
+      for (QueuedCriticalEvent queued : queue.poll()) {
+        try {
+          apply(queued.event());
+          queue.acknowledge(queued);
+          healthy.set(true);
+        } catch (RuntimeException error) {
+          healthy.set(false);
+          break;
+        }
+      }
+    } catch (RuntimeException error) {
+      healthy.set(false);
+    }
+  }
+
+  void apply(CriticalEvent event) {
+    throw new IllegalStateException("Unsupported critical event type: " + event.type());
+  }
+
+  public boolean isHealthy() {
+    return healthy.get();
+  }
+}


## `test(delivery): verify processor recovery`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessorTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessorTest.java
new file mode 100644
index 0000000..d83db43
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventProcessorTest.java
@@ -0,0 +1,79 @@
+package com.sportsbook.oddsfeed.delivery;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.oddsfeed.config.CriticalDeliveryProperties;
+import com.sportsbook.protocol.event.EventLifecycleStatus;
+import com.sportsbook.protocol.value.EventId;
+import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
+import java.time.Duration;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.data.redis.connection.stream.RecordId;
+import org.springframework.data.redis.core.StringRedisTemplate;
+
+class CriticalEventProcessorTest {
+
+  @Test
+  void retainsFailedEventsAndAcknowledgesTheirRetry() {
+    CriticalEvent event =
+        CriticalEvent.lifecycle(
+            new EventId(UUID.randomUUID()),
+            EventLifecycleStatus.SCHEDULED,
+            Instant.EPOCH,
+            Instant.EPOCH);
+    StubQueue queue = new StubQueue(event);
+    RecoveringProcessor processor = new RecoveringProcessor(queue);
+
+    processor.drain();
+    assertThat(processor.isHealthy()).isFalse();
+    assertThat(queue.acknowledgements).isZero();
+
+    processor.drain();
+    assertThat(processor.isHealthy()).isTrue();
+    assertThat(queue.acknowledgements).isEqualTo(1);
+    assertThat(queue.poll()).isEmpty();
+  }
+
+  private static final class RecoveringProcessor extends CriticalEventProcessor {
+    private int attempts;
+
+    private RecoveringProcessor(CriticalEventQueue queue) {
+      super(queue, null, null, null);
+    }
+
+    @Override
+    void apply(CriticalEvent event) {
+      if (++attempts == 1) {
+        throw new IllegalStateException("temporary failure");
+      }
+    }
+  }
+
+  private static final class StubQueue extends CriticalEventQueue {
+    private final QueuedCriticalEvent queued;
+    private int acknowledgements;
+
+    private StubQueue(CriticalEvent event) {
+      super(
+          new StringRedisTemplate(),
+          new ObjectMapper(),
+          new CriticalDeliveryProperties("stream", "group", "consumer", 1, Duration.ZERO),
+          new SimpleMeterRegistry());
+      queued = new QueuedCriticalEvent(RecordId.of("1-0"), event, false);
+    }
+
+    @Override
+    public List<QueuedCriticalEvent> poll() {
+      return acknowledgements == 0 ? List.of(queued) : List.of();
+    }
+
+    @Override
+    public void acknowledge(QueuedCriticalEvent event) {
+      acknowledgements++;
+    }
+  }
+}
