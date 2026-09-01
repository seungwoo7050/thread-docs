## `test(commands): resolve current reopen projections`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
index e3d699e..6c1674c 100644
--- a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
@@ -477,6 +477,50 @@ class OperatorActionQueueTest {
         .isEqualTo(MarketStatus.CLOSED);
   }
 
+  @Test
+  void deliveryDecisionPublishesANormalReopenAsOpen() {
+    assertThat(reopenDeliveryStatus(MarketStatus.OPEN, false)).isEqualTo(MarketStatus.OPEN);
+  }
+
+  @Test
+  void deliveryDecisionReevaluatesProviderSuspension() {
+    assertThat(reopenDeliveryStatus(MarketStatus.SUSPENDED, false))
+        .isEqualTo(MarketStatus.SUSPENDED);
+  }
+
+  @Test
+  void deliveryDecisionReevaluatesANewFeedHold() {
+    assertThat(reopenDeliveryStatus(MarketStatus.OPEN, true)).isEqualTo(MarketStatus.SUSPENDED);
+  }
+
+  private MarketStatus reopenDeliveryStatus(MarketStatus providerAfterSubmit, boolean hold) {
+    OperatorActionQueue queue = queue();
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+    redis.opsForValue().set(CacheKeys.providerMarket(eventId, marketId), MarketStatus.OPEN.name());
+    redis
+        .opsForValue()
+        .set(CacheKeys.marketOverride(eventId, marketId), MarketStatus.SUSPENDED.name());
+    queue.submit(
+        IdempotencyKey.of("delivery-" + UUID.randomUUID()),
+        UUID.randomUUID(),
+        eventId,
+        marketId,
+        MarketStatus.OPEN,
+        "review complete",
+        NOW);
+    QueuedOperatorMarketAction reopen = queue.poll().get(0);
+    redis
+        .opsForValue()
+        .set(CacheKeys.providerMarket(eventId, marketId), providerAfterSubmit.name());
+    if (hold) {
+      redis
+          .opsForValue()
+          .set(CacheKeys.marketFeedHold(eventId, marketId), Long.toString(NOW.toEpochMilli()));
+    }
+    return queue.deliveryDecision(reopen.action()).announcedStatus();
+  }
+
   private OperatorActionQueue queue() {
     return queue("consumer", Duration.ZERO);
   }


## `feat(commands): publish queued operator actions`

diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionProcessor.java b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionProcessor.java
new file mode 100644
index 0000000..c91ad4a
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionProcessor.java
@@ -0,0 +1,83 @@
+package com.sportsbook.oddsfeed.delivery;
+
+import com.sportsbook.oddsfeed.publisher.OddsFeedPublisher;
+import io.micrometer.core.instrument.Counter;
+import io.micrometer.core.instrument.MeterRegistry;
+import java.util.concurrent.atomic.AtomicBoolean;
+import org.slf4j.Logger;
+import org.slf4j.LoggerFactory;
+import org.springframework.scheduling.annotation.Scheduled;
+import org.springframework.stereotype.Component;
+
+/** Publishes durable operator actions and advances each market with an acknowledged CAS. */
+@Component
+public class OperatorActionProcessor {
+
+  private static final Logger log = LoggerFactory.getLogger(OperatorActionProcessor.class);
+
+  private final OperatorActionQueue queue;
+  private final OddsFeedPublisher publisher;
+  private final Counter processed;
+  private final Counter failures;
+  private final AtomicBoolean healthy = new AtomicBoolean(true);
+
+  public OperatorActionProcessor(
+      OperatorActionQueue queue, OddsFeedPublisher publisher, MeterRegistry meterRegistry) {
+    this.queue = queue;
+    this.publisher = publisher;
+    this.processed = meterRegistry.counter("oddsfeed.operator.action.processed");
+    this.failures = meterRegistry.counter("oddsfeed.operator.action.processing.failure");
+  }
+
+  @Scheduled(fixedDelayString = "${oddsfeed.operator.delivery.poll-interval-ms:250}")
+  void drain() {
+    var queuedActions = queue.poll();
+    if (queuedActions.isEmpty() && queue.pendingCount() == 0) {
+      healthy.set(true);
+    }
+    for (QueuedOperatorMarketAction queued : queuedActions) {
+      OperatorMarketAction action = queued.action();
+      try {
+        OperatorDeliveryDecision decision = queue.deliveryDecision(action);
+        if (decision.outcome() == OperatorDeliveryDecision.Outcome.COMPLETED) {
+          queue.cleanup(queued);
+          processed.increment();
+          healthy.set(true);
+          continue;
+        }
+        if (decision.outcome() == OperatorDeliveryDecision.Outcome.BLOCKED) {
+          break;
+        }
+
+        if (decision.outcome() == OperatorDeliveryDecision.Outcome.PUBLISH) {
+          publisher.publishMarketStatusChanged(
+              action.eventId(),
+              action.marketId(),
+              action.previousStatus(),
+              decision.announcedStatus(),
+              action.reason(),
+              action.occurredAt());
+        }
+        OperatorActionQueue.Completion completion = queue.complete(action);
+        if (completion == OperatorActionQueue.Completion.BLOCKED) {
+          break;
+        }
+        queue.cleanup(queued);
+        processed.increment();
+        healthy.set(true);
+      } catch (RuntimeException exception) {
+        failures.increment();
+        healthy.set(false);
+        log.warn(
+            "Operator action {} remains pending after delivery failure: {}",
+            action.actionId(),
+            exception.toString());
+        break;
+      }
+    }
+  }
+
+  public boolean isHealthy() {
+    return healthy.get();
+  }
+}
diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java
index d22e1e6..7fb2f05 100644
--- a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java
@@ -15,6 +15,7 @@ import java.util.ArrayList;
 import java.util.List;
 import java.util.UUID;
 import java.util.concurrent.atomic.AtomicBoolean;
+import java.util.concurrent.atomic.AtomicLong;
 import org.springframework.dao.DataAccessException;
 import org.springframework.data.domain.Range;
 import org.springframework.data.redis.connection.stream.Consumer;
@@ -48,6 +49,7 @@ public class OperatorActionQueue {
   private final OperatorActionCodec codec = new OperatorActionCodec();
   private final long marketTtlMillis;
   private final AtomicBoolean groupReady = new AtomicBoolean();
+  private final AtomicLong pendingCount = new AtomicLong();
 
   public OperatorActionQueue(
       StringRedisTemplate redis,
@@ -117,6 +119,10 @@ public class OperatorActionQueue {
                   StreamReadOptions.empty().count(properties.batchSize()),
                   StreamOffset.create(properties.streamKey(), ReadOffset.lastConsumed()));
     }
+    pendingCount.set(
+        streamOperations()
+            .pending(properties.streamKey(), properties.consumerGroup())
+            .getTotalPendingMessages());
     if (records == null || records.isEmpty()) {
       return List.of();
     }
@@ -134,6 +140,13 @@ public class OperatorActionQueue {
     if (result == null || !result.matches("[01]\\|[01]")) {
       throw new IllegalStateException("Malformed operator Stream cleanup result");
     }
+    if (result.charAt(0) == '1') {
+      pendingCount.updateAndGet(current -> Math.max(0, current - 1));
+    }
+  }
+
+  public long pendingCount() {
+    return pendingCount.get();
   }
 
   public DeliveryState deliveryState(OperatorMarketAction action) {


## `test(commands): verify restrictive delivery order`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionProcessorTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionProcessorTest.java
new file mode 100644
index 0000000..ff5a511
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionProcessorTest.java
@@ -0,0 +1,97 @@
+package com.sportsbook.oddsfeed.delivery;
+
+import static org.mockito.Mockito.inOrder;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.never;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.oddsfeed.publisher.OddsFeedPublisher;
+import com.sportsbook.protocol.event.MarketStatus;
+import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.MarketId;
+import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.mockito.InOrder;
+import org.springframework.data.redis.connection.stream.RecordId;
+
+class OperatorActionProcessorTest {
+
+  @Test
+  void publishesResolvedReopenBeforeCompletionAndStreamCleanup() {
+    OperatorActionQueue queue = mock(OperatorActionQueue.class);
+    OddsFeedPublisher publisher = mock(OddsFeedPublisher.class);
+    QueuedOperatorMarketAction queued = queuedAction();
+    when(queue.poll()).thenReturn(List.of(queued));
+    when(queue.deliveryDecision(queued.action()))
+        .thenReturn(decision(OperatorDeliveryDecision.Outcome.PUBLISH, MarketStatus.OPEN));
+    when(queue.complete(queued.action())).thenReturn(OperatorActionQueue.Completion.APPLIED);
+
+    new OperatorActionProcessor(queue, publisher, new SimpleMeterRegistry()).drain();
+
+    InOrder order = inOrder(queue, publisher);
+    order.verify(queue).poll();
+    order.verify(queue).deliveryDecision(queued.action());
+    order
+        .verify(publisher)
+        .publishMarketStatusChanged(
+            queued.action().eventId(),
+            queued.action().marketId(),
+            queued.action().previousStatus(),
+            MarketStatus.OPEN,
+            queued.action().reason(),
+            queued.action().occurredAt());
+    order.verify(queue).complete(queued.action());
+    order.verify(queue).cleanup(queued);
+  }
+
+  @Test
+  void terminalOrSupersededReopenCompletesWithoutPublishing() {
+    OperatorActionQueue queue = mock(OperatorActionQueue.class);
+    OddsFeedPublisher publisher = mock(OddsFeedPublisher.class);
+    QueuedOperatorMarketAction queued = queuedAction();
+    when(queue.poll()).thenReturn(List.of(queued));
+    when(queue.deliveryDecision(queued.action()))
+        .thenReturn(decision(OperatorDeliveryDecision.Outcome.SKIP, null));
+    when(queue.complete(queued.action())).thenReturn(OperatorActionQueue.Completion.APPLIED);
+
+    new OperatorActionProcessor(queue, publisher, new SimpleMeterRegistry()).drain();
+
+    InOrder order = inOrder(queue, publisher);
+    order.verify(queue).deliveryDecision(queued.action());
+    order.verify(queue).complete(queued.action());
+    order.verify(queue).cleanup(queued);
+    verify(publisher, never())
+        .publishMarketStatusChanged(
+            queued.action().eventId(),
+            queued.action().marketId(),
+            queued.action().previousStatus(),
+            queued.action().announcedStatus(),
+            queued.action().reason(),
+            queued.action().occurredAt());
+  }
+
+  private static OperatorDeliveryDecision decision(
+      OperatorDeliveryDecision.Outcome outcome, MarketStatus announcedStatus) {
+    return new OperatorDeliveryDecision(outcome, announcedStatus);
+  }
+
+  private static QueuedOperatorMarketAction queuedAction() {
+    OperatorMarketAction action =
+        new OperatorMarketAction(
+            UUID.randomUUID(),
+            new EventId(UUID.randomUUID()),
+            new MarketId(UUID.randomUUID()),
+            MarketStatus.SUSPENDED,
+            MarketStatus.OPEN,
+            MarketStatus.OPEN,
+            "incident",
+            1,
+            0,
+            Instant.parse("2026-08-21T05:00:00Z"));
+    return new QueuedOperatorMarketAction(RecordId.of("1-0"), action, false);
+  }
+}


## `test(commands): publish current action projections`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionProcessorTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionProcessorTest.java
index ff5a511..0df358c 100644
--- a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionProcessorTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionProcessorTest.java
@@ -74,6 +74,29 @@ class OperatorActionProcessorTest {
             queued.action().occurredAt());
   }
 
+  @Test
+  void publishesReopenUsingTheLatestRestrictiveProjection() {
+    OperatorActionQueue queue = mock(OperatorActionQueue.class);
+    OddsFeedPublisher publisher = mock(OddsFeedPublisher.class);
+    QueuedOperatorMarketAction queued = queuedAction();
+    when(queue.poll()).thenReturn(List.of(queued));
+    when(queue.deliveryDecision(queued.action()))
+        .thenReturn(decision(OperatorDeliveryDecision.Outcome.PUBLISH, MarketStatus.SUSPENDED));
+    when(queue.complete(queued.action())).thenReturn(OperatorActionQueue.Completion.APPLIED);
+
+    new OperatorActionProcessor(queue, publisher, new SimpleMeterRegistry()).drain();
+
+    verify(publisher)
+        .publishMarketStatusChanged(
+            queued.action().eventId(),
+            queued.action().marketId(),
+            queued.action().previousStatus(),
+            MarketStatus.SUSPENDED,
+            queued.action().reason(),
+            queued.action().occurredAt());
+    verify(queue).cleanup(queued);
+  }
+
   private static OperatorDeliveryDecision decision(
       OperatorDeliveryDecision.Outcome outcome, MarketStatus announcedStatus) {
     return new OperatorDeliveryDecision(outcome, announcedStatus);


## `test(commands): recover completed stream cleanup`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionProcessorTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionProcessorTest.java
index 0df358c..da7aee7 100644
--- a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionProcessorTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionProcessorTest.java
@@ -1,8 +1,11 @@
 package com.sportsbook.oddsfeed.delivery;
 
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.doThrow;
 import static org.mockito.Mockito.inOrder;
 import static org.mockito.Mockito.mock;
 import static org.mockito.Mockito.never;
+import static org.mockito.Mockito.times;
 import static org.mockito.Mockito.verify;
 import static org.mockito.Mockito.when;
 
@@ -97,6 +100,71 @@ class OperatorActionProcessorTest {
     verify(queue).cleanup(queued);
   }
 
+  @Test
+  void completedReclaimCleansUpAndRestoresProcessorHealth() {
+    OperatorActionQueue queue = mock(OperatorActionQueue.class);
+    OddsFeedPublisher publisher = mock(OddsFeedPublisher.class);
+    QueuedOperatorMarketAction queued = queuedAction();
+    when(queue.poll()).thenReturn(List.of(queued));
+    when(queue.deliveryDecision(queued.action()))
+        .thenReturn(
+            decision(OperatorDeliveryDecision.Outcome.PUBLISH, MarketStatus.OPEN),
+            decision(OperatorDeliveryDecision.Outcome.COMPLETED, null));
+    when(queue.complete(queued.action())).thenReturn(OperatorActionQueue.Completion.APPLIED);
+    doThrow(new IllegalStateException("cleanup unavailable"))
+        .doNothing()
+        .when(queue)
+        .cleanup(queued);
+    OperatorActionProcessor processor =
+        new OperatorActionProcessor(queue, publisher, new SimpleMeterRegistry());
+
+    processor.drain();
+    assertThat(processor.isHealthy()).isFalse();
+
+    processor.drain();
+
+    assertThat(processor.isHealthy()).isTrue();
+    verify(publisher, times(1))
+        .publishMarketStatusChanged(
+            queued.action().eventId(),
+            queued.action().marketId(),
+            queued.action().previousStatus(),
+            MarketStatus.OPEN,
+            queued.action().reason(),
+            queued.action().occurredAt());
+    verify(queue, times(2)).cleanup(queued);
+  }
+
+  @Test
+  void emptyQueueRecoversAfterAnAmbiguousCleanupResponse() {
+    OperatorActionQueue queue = mock(OperatorActionQueue.class);
+    OddsFeedPublisher publisher = mock(OddsFeedPublisher.class);
+    QueuedOperatorMarketAction queued = queuedAction();
+    when(queue.poll()).thenReturn(List.of(queued), List.of());
+    when(queue.deliveryDecision(queued.action()))
+        .thenReturn(decision(OperatorDeliveryDecision.Outcome.PUBLISH, MarketStatus.OPEN));
+    when(queue.complete(queued.action())).thenReturn(OperatorActionQueue.Completion.APPLIED);
+    doThrow(new IllegalStateException("response lost")).when(queue).cleanup(queued);
+    OperatorActionProcessor processor =
+        new OperatorActionProcessor(queue, publisher, new SimpleMeterRegistry());
+
+    processor.drain();
+    assertThat(processor.isHealthy()).isFalse();
+
+    when(queue.pendingCount()).thenReturn(0L);
+    processor.drain();
+
+    assertThat(processor.isHealthy()).isTrue();
+    verify(publisher, times(1))
+        .publishMarketStatusChanged(
+            queued.action().eventId(),
+            queued.action().marketId(),
+            queued.action().previousStatus(),
+            MarketStatus.OPEN,
+            queued.action().reason(),
+            queued.action().occurredAt());
+  }
+
   private static OperatorDeliveryDecision decision(
       OperatorDeliveryDecision.Outcome outcome, MarketStatus announcedStatus) {
     return new OperatorDeliveryDecision(outcome, announcedStatus);


## `feat(commands): finalize acknowledged reopens`

diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorCompletionScript.java b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorCompletionScript.java
index f6697d8..9d96ede 100644
--- a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorCompletionScript.java
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorCompletionScript.java
@@ -21,6 +21,11 @@ final class OperatorCompletionScript {
             terminal = true
           end
           if committed >= sequence then
+            if terminal then
+              redis.call('PSETEX', KEYS[3], ARGV[4], 'CLOSED')
+              redis.call('HSET', KEYS[10], ARGV[6], 'CLOSED')
+              redis.call('PEXPIRE', KEYS[10], ARGV[4])
+            end
             local idempotency = redis.call('GET', KEYS[6])
             redis.call('PEXPIRE', KEYS[6], ARGV[5])
             if idempotency then
@@ -43,6 +48,11 @@ final class OperatorCompletionScript {
             redis.call('PEXPIRE', idempotency, ARGV[5])
           end
           if tail ~= sequence then
+            if terminal then
+              redis.call('PSETEX', KEYS[3], ARGV[4], 'CLOSED')
+              redis.call('HSET', KEYS[10], ARGV[6], 'CLOSED')
+              redis.call('PEXPIRE', KEYS[10], ARGV[4])
+            end
             return 'SUPERSEDED'
           end
 
@@ -51,7 +61,12 @@ final class OperatorCompletionScript {
           if terminal then
             effective = 'CLOSED'
           elseif requested == 'OPEN' then
-            effective = redis.call('GET', KEYS[3]) or provider
+            redis.call('DEL', KEYS[5])
+            if redis.call('EXISTS', KEYS[9]) == 1 then
+              effective = 'SUSPENDED'
+            else
+              effective = provider
+            end
           else
             redis.call('SET', KEYS[5], requested)
           end


## `test(commands): verify reopen completion safety`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
index 6c1674c..02ff458 100644
--- a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
@@ -521,6 +521,73 @@ class OperatorActionQueueTest {
     return queue.deliveryDecision(reopen.action()).announcedStatus();
   }
 
+  @Test
+  void terminalLatchAfterReopenQueueingKeepsTheOverrideClosed() {
+    OperatorActionQueue queue = queue();
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+    redis
+        .opsForValue()
+        .set(CacheKeys.marketOverride(eventId, marketId), MarketStatus.SUSPENDED.name());
+    redis.opsForValue().set(CacheKeys.market(eventId, marketId), MarketStatus.SUSPENDED.name());
+    submit(queue, "terminal-race", eventId, marketId, MarketStatus.OPEN);
+    QueuedOperatorMarketAction reopen = queue.poll().get(0);
+
+    redis.opsForValue().set(CacheKeys.marketTerminal(eventId, marketId), "MARKET_CLOSED");
+    assertThat(queue.complete(reopen.action())).isEqualTo(OperatorActionQueue.Completion.APPLIED);
+
+    assertThat(redis.opsForValue().get(CacheKeys.market(eventId, marketId)))
+        .isEqualTo(MarketStatus.CLOSED.name());
+    assertThat(redis.opsForValue().get(CacheKeys.marketOverride(eventId, marketId)))
+        .isEqualTo(MarketStatus.SUSPENDED.name());
+  }
+
+  @Test
+  void acknowledgedReopenPreservesAnActiveFeedHold() {
+    OperatorActionQueue queue = queue();
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+    redis.opsForValue().set(CacheKeys.providerMarket(eventId, marketId), MarketStatus.OPEN.name());
+    redis
+        .opsForValue()
+        .set(CacheKeys.marketOverride(eventId, marketId), MarketStatus.SUSPENDED.name());
+    redis.opsForValue().set(CacheKeys.market(eventId, marketId), MarketStatus.SUSPENDED.name());
+    redis
+        .opsForValue()
+        .set(CacheKeys.marketFeedHold(eventId, marketId), Long.toString(NOW.toEpochMilli()));
+    submit(queue, "held-reopen", eventId, marketId, MarketStatus.OPEN);
+    QueuedOperatorMarketAction reopen = queue.poll().get(0);
+
+    assertThat(reopen.action().announcedStatus()).isEqualTo(MarketStatus.SUSPENDED);
+    assertThat(queue.complete(reopen.action())).isEqualTo(OperatorActionQueue.Completion.APPLIED);
+    assertThat(redis.hasKey(CacheKeys.marketOverride(eventId, marketId))).isFalse();
+    assertThat(redis.hasKey(CacheKeys.marketFeedHold(eventId, marketId))).isTrue();
+    assertThat(redis.opsForValue().get(CacheKeys.market(eventId, marketId)))
+        .isEqualTo(MarketStatus.SUSPENDED.name());
+    assertThat(redis.opsForHash().get(CacheKeys.eventMarkets(eventId), marketId.value().toString()))
+        .isEqualTo(MarketStatus.SUSPENDED.name());
+  }
+
+  @Test
+  void supersededReopenCannotEraseANewerRestriction() {
+    OperatorActionQueue queue = queue();
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+    redis
+        .opsForValue()
+        .set(CacheKeys.marketOverride(eventId, marketId), MarketStatus.SUSPENDED.name());
+    submit(queue, "older-reopen", eventId, marketId, MarketStatus.OPEN);
+    submit(queue, "newer-close", eventId, marketId, MarketStatus.CLOSED);
+    List<QueuedOperatorMarketAction> actions = queue.poll();
+
+    assertThat(queue.complete(actions.get(0).action()))
+        .isEqualTo(OperatorActionQueue.Completion.SUPERSEDED);
+    assertThat(redis.opsForValue().get(CacheKeys.marketOverride(eventId, marketId)))
+        .isEqualTo(MarketStatus.CLOSED.name());
+    assertThat(queue.deliveryState(actions.get(1).action()))
+        .isEqualTo(OperatorActionQueue.DeliveryState.READY);
+  }
+
   private OperatorActionQueue queue() {
     return queue("consumer", Duration.ZERO);
   }
@@ -532,4 +599,14 @@ class OperatorActionQueueTest {
         new CacheProperties(Duration.ofHours(24)),
         new SimpleMeterRegistry());
   }
+
+  private static void submit(
+      OperatorActionQueue queue,
+      String key,
+      EventId eventId,
+      MarketId marketId,
+      MarketStatus status) {
+    queue.submit(
+        IdempotencyKey.of(key), UUID.randomUUID(), eventId, marketId, status, "review", NOW);
+  }
 }


