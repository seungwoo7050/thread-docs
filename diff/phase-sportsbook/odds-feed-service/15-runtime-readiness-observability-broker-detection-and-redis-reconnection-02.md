## `test(redis): verify Redis address refresh`

diff --git a/src/test/java/com/sportsbook/oddsfeed/config/RedisClientConfigTest.java b/src/test/java/com/sportsbook/oddsfeed/config/RedisClientConfigTest.java
new file mode 100644
index 0000000..678d79a
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/config/RedisClientConfigTest.java
@@ -0,0 +1,66 @@
+package com.sportsbook.oddsfeed.config;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import io.lettuce.core.RedisURI;
+import io.lettuce.core.resource.DefaultClientResources;
+import io.lettuce.core.resource.DnsResolver;
+import java.net.InetAddress;
+import java.net.InetSocketAddress;
+import java.util.concurrent.atomic.AtomicInteger;
+import org.junit.jupiter.api.Test;
+import org.springframework.boot.autoconfigure.data.redis.ClientResourcesBuilderCustomizer;
+
+class RedisClientConfigTest {
+
+  @Test
+  void resolvesLocalhostToAConcreteAddress() {
+    DefaultClientResources resources =
+        clientResources(new RedisClientConfig().redisDnsResolverCustomizer());
+
+    try {
+      InetSocketAddress resolved =
+          (InetSocketAddress)
+              resources.socketAddressResolver().resolve(RedisURI.create("localhost", 6379));
+
+      assertThat(resolved.isUnresolved()).isFalse();
+      assertThat(resolved.getAddress().isLoopbackAddress()).isTrue();
+    } finally {
+      resources.shutdown().syncUninterruptibly();
+    }
+  }
+
+  @Test
+  void resolvesHostnamesForEveryConnectionAttempt() {
+    AtomicInteger resolutions = new AtomicInteger();
+    DnsResolver changingResolver =
+        hostname -> {
+          assertThat(hostname).isEqualTo("redis.internal");
+          String endpoint = resolutions.getAndIncrement() == 0 ? "192.0.2.10" : "192.0.2.11";
+          return new InetAddress[] {InetAddress.getByName(endpoint)};
+        };
+    DefaultClientResources resources =
+        clientResources(RedisClientConfig.dnsResolverCustomizer(changingResolver));
+    RedisURI redisUri = RedisURI.create("redis.internal", 6379);
+
+    try {
+      InetSocketAddress first =
+          (InetSocketAddress) resources.socketAddressResolver().resolve(redisUri);
+      InetSocketAddress second =
+          (InetSocketAddress) resources.socketAddressResolver().resolve(redisUri);
+
+      assertThat(first.getAddress().getHostAddress()).isEqualTo("192.0.2.10");
+      assertThat(second.getAddress().getHostAddress()).isEqualTo("192.0.2.11");
+      assertThat(resolutions).hasValue(2);
+    } finally {
+      resources.shutdown().syncUninterruptibly();
+    }
+  }
+
+  private static DefaultClientResources clientResources(
+      ClientResourcesBuilderCustomizer customizer) {
+    DefaultClientResources.Builder builder = DefaultClientResources.builder();
+    customizer.customize(builder);
+    return builder.build();
+  }
+}


## `feat(health): report operator delivery readiness`

diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalDeliveryHealthIndicator.java b/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalDeliveryHealthIndicator.java
index a394b02..158cf99 100644
--- a/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalDeliveryHealthIndicator.java
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalDeliveryHealthIndicator.java
@@ -1,6 +1,7 @@
 package com.sportsbook.oddsfeed.delivery;
 
 import com.sportsbook.oddsfeed.publisher.OddsFeedPublisher;
+import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.boot.actuate.health.Health;
 import org.springframework.boot.actuate.health.HealthIndicator;
 import org.springframework.stereotype.Component;
@@ -11,23 +12,43 @@ public class CriticalDeliveryHealthIndicator implements HealthIndicator {
   private final CriticalEventQueue queue;
   private final OddsFeedPublisher publisher;
   private final CriticalEventProcessor processor;
+  private final OperatorActionQueue operatorQueue;
+  private final OperatorActionProcessor operatorProcessor;
 
+  @Autowired
   public CriticalDeliveryHealthIndicator(
-      CriticalEventQueue queue, OddsFeedPublisher publisher, CriticalEventProcessor processor) {
+      CriticalEventQueue queue,
+      OddsFeedPublisher publisher,
+      CriticalEventProcessor processor,
+      OperatorActionQueue operatorQueue,
+      OperatorActionProcessor operatorProcessor) {
     this.queue = queue;
     this.publisher = publisher;
     this.processor = processor;
+    this.operatorQueue = operatorQueue;
+    this.operatorProcessor = operatorProcessor;
+  }
+
+  CriticalDeliveryHealthIndicator(
+      CriticalEventQueue queue, OddsFeedPublisher publisher, CriticalEventProcessor processor) {
+    this(queue, publisher, processor, null, null);
   }
 
   @Override
   public Health health() {
-    boolean available = queue.isHealthy() && publisher.isHealthy() && processor.isHealthy();
+    boolean operatorAvailable =
+        operatorQueue == null || operatorQueue.isHealthy() && operatorProcessor.isHealthy();
+    boolean available =
+        queue.isHealthy() && publisher.isHealthy() && processor.isHealthy() && operatorAvailable;
     Health.Builder health = available ? Health.up() : Health.down();
     return health
         .withDetail("redisStream", queue.isHealthy() ? "UP" : "DOWN")
         .withDetail("kafkaPublisher", publisher.isHealthy() ? "UP" : "DOWN")
         .withDetail("criticalProcessor", processor.isHealthy() ? "UP" : "DOWN")
         .withDetail("pendingRecords", queue.pendingCount())
+        .withDetail("operatorDelivery", operatorAvailable ? "UP" : "DOWN")
+        .withDetail(
+            "operatorPendingRecords", operatorQueue == null ? 0 : operatorQueue.pendingCount())
         .build();
   }
 }
diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java
index 7fb2f05..c944318 100644
--- a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java
@@ -49,6 +49,7 @@ public class OperatorActionQueue {
   private final OperatorActionCodec codec = new OperatorActionCodec();
   private final long marketTtlMillis;
   private final AtomicBoolean groupReady = new AtomicBoolean();
+  private final AtomicBoolean healthy = new AtomicBoolean(true);
   private final AtomicLong pendingCount = new AtomicLong();
 
   public OperatorActionQueue(
@@ -59,6 +60,7 @@ public class OperatorActionQueue {
     this.redis = redis;
     this.properties = properties;
     this.marketTtlMillis = cacheProperties.ttl().toMillis();
+    meterRegistry.gauge("oddsfeed.operator.action.pending", pendingCount);
   }
 
   public OperatorActionSubmission submit(
@@ -69,6 +71,26 @@ public class OperatorActionQueue {
       MarketStatus requestedStatus,
       String reason,
       Instant occurredAt) {
+    try {
+      OperatorActionSubmission submission =
+          submitOnce(
+              idempotencyKey, actionId, eventId, marketId, requestedStatus, reason, occurredAt);
+      healthy.set(true);
+      return submission;
+    } catch (DataAccessException exception) {
+      healthy.set(false);
+      throw exception;
+    }
+  }
+
+  private OperatorActionSubmission submitOnce(
+      IdempotencyKey idempotencyKey,
+      UUID actionId,
+      EventId eventId,
+      MarketId marketId,
+      MarketStatus requestedStatus,
+      String reason,
+      Instant occurredAt) {
     String normalizedReason = normalizeReason(reason);
     String fingerprint =
         MarketActionFingerprint.request(
@@ -107,6 +129,19 @@ public class OperatorActionQueue {
   }
 
   public List<QueuedOperatorMarketAction> poll() {
+    try {
+      List<QueuedOperatorMarketAction> actions = pollOnce();
+      updatePendingCount();
+      healthy.set(true);
+      return actions;
+    } catch (DataAccessException exception) {
+      groupReady.set(false);
+      healthy.set(false);
+      throw exception;
+    }
+  }
+
+  private List<QueuedOperatorMarketAction> pollOnce() {
     ensureGroup();
     PendingMessages pending = pendingMessages();
     List<MapRecord<String, String, String>> records = claimExpired(pending);
@@ -119,10 +154,6 @@ public class OperatorActionQueue {
                   StreamReadOptions.empty().count(properties.batchSize()),
                   StreamOffset.create(properties.streamKey(), ReadOffset.lastConsumed()));
     }
-    pendingCount.set(
-        streamOperations()
-            .pending(properties.streamKey(), properties.consumerGroup())
-            .getTotalPendingMessages());
     if (records == null || records.isEmpty()) {
       return List.of();
     }
@@ -131,20 +162,30 @@ public class OperatorActionQueue {
   }
 
   public void cleanup(QueuedOperatorMarketAction queued) {
-    String result =
-        redis.execute(
-            OperatorStreamCleanupScript.INSTANCE,
-            List.of(properties.streamKey()),
-            properties.consumerGroup(),
-            queued.recordId().getValue());
-    if (result == null || !result.matches("[01]\\|[01]")) {
-      throw new IllegalStateException("Malformed operator Stream cleanup result");
-    }
-    if (result.charAt(0) == '1') {
-      pendingCount.updateAndGet(current -> Math.max(0, current - 1));
+    try {
+      String result =
+          redis.execute(
+              OperatorStreamCleanupScript.INSTANCE,
+              List.of(properties.streamKey()),
+              properties.consumerGroup(),
+              queued.recordId().getValue());
+      if (result == null || !result.matches("[01]\\|[01]")) {
+        throw new IllegalStateException("Malformed operator Stream cleanup result");
+      }
+      if (result.charAt(0) == '1') {
+        pendingCount.updateAndGet(current -> Math.max(0, current - 1));
+      }
+      healthy.set(true);
+    } catch (DataAccessException exception) {
+      healthy.set(false);
+      throw exception;
     }
   }
 
+  public boolean isHealthy() {
+    return healthy.get();
+  }
+
   public long pendingCount() {
     return pendingCount.get();
   }
@@ -287,6 +328,13 @@ public class OperatorActionQueue {
     return redis.<String, String>opsForStream();
   }
 
+  private void updatePendingCount() {
+    pendingCount.set(
+        streamOperations()
+            .pending(properties.streamKey(), properties.consumerGroup())
+            .getTotalPendingMessages());
+  }
+
   private static boolean containsBusyGroup(Throwable error) {
     Throwable current = error;
     while (current != null) {


## `test(health): verify operator readiness details`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalDeliveryHealthIndicatorTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalDeliveryHealthIndicatorTest.java
index 8b3414e..77da93e 100644
--- a/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalDeliveryHealthIndicatorTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalDeliveryHealthIndicatorTest.java
@@ -1,6 +1,8 @@
 package com.sportsbook.oddsfeed.delivery;
 
 import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.when;
 
 import com.fasterxml.jackson.databind.ObjectMapper;
 import com.sportsbook.oddsfeed.config.CriticalDeliveryProperties;
@@ -44,6 +46,31 @@ class CriticalDeliveryHealthIndicatorTest {
         .isEqualTo("readinessState,redis,criticalDelivery");
   }
 
+  @Test
+  void operatorDeliveryFailureMakesReadinessUnavailable() {
+    CriticalEventQueue criticalQueue = mock(CriticalEventQueue.class);
+    OddsFeedPublisher publisher = mock(OddsFeedPublisher.class);
+    CriticalEventProcessor criticalProcessor = mock(CriticalEventProcessor.class);
+    OperatorActionQueue operatorQueue = mock(OperatorActionQueue.class);
+    OperatorActionProcessor operatorProcessor = mock(OperatorActionProcessor.class);
+    when(criticalQueue.isHealthy()).thenReturn(true);
+    when(publisher.isHealthy()).thenReturn(true);
+    when(criticalProcessor.isHealthy()).thenReturn(true);
+    when(operatorQueue.isHealthy()).thenReturn(true);
+    when(operatorQueue.pendingCount()).thenReturn(3L);
+    when(operatorProcessor.isHealthy()).thenReturn(false);
+
+    var health =
+        new CriticalDeliveryHealthIndicator(
+                criticalQueue, publisher, criticalProcessor, operatorQueue, operatorProcessor)
+            .health();
+
+    assertThat(health.getStatus()).isEqualTo(Status.DOWN);
+    assertThat(health.getDetails())
+        .containsEntry("operatorDelivery", "DOWN")
+        .containsEntry("operatorPendingRecords", 3L);
+  }
+
   private static final class StubQueue extends CriticalEventQueue {
     private final boolean healthy;
     private final long pending;
