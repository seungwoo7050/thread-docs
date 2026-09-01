## `test(commands): retain actions after broker failure`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionProcessorTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionProcessorTest.java
index da7aee7..67851a3 100644
--- a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionProcessorTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionProcessorTest.java
@@ -9,6 +9,7 @@ import static org.mockito.Mockito.times;
 import static org.mockito.Mockito.verify;
 import static org.mockito.Mockito.when;
 
+import com.sportsbook.oddsfeed.publisher.KafkaPublishException;
 import com.sportsbook.oddsfeed.publisher.OddsFeedPublisher;
 import com.sportsbook.protocol.event.MarketStatus;
 import com.sportsbook.protocol.value.EventId;
@@ -170,6 +171,38 @@ class OperatorActionProcessorTest {
     return new OperatorDeliveryDecision(outcome, announcedStatus);
   }
 
+  @Test
+  void brokerFailureLeavesRecordPendingForRetry() {
+    OperatorActionQueue queue = mock(OperatorActionQueue.class);
+    OddsFeedPublisher publisher = mock(OddsFeedPublisher.class);
+    QueuedOperatorMarketAction queued = queuedAction();
+    when(queue.poll()).thenReturn(List.of(queued), List.of());
+    when(queue.deliveryDecision(queued.action()))
+        .thenReturn(decision(OperatorDeliveryDecision.Outcome.PUBLISH, MarketStatus.OPEN));
+    doThrow(new KafkaPublishException("broker unavailable", new IllegalStateException("down")))
+        .when(publisher)
+        .publishMarketStatusChanged(
+            queued.action().eventId(),
+            queued.action().marketId(),
+            queued.action().previousStatus(),
+            MarketStatus.OPEN,
+            queued.action().reason(),
+            queued.action().occurredAt());
+    OperatorActionProcessor processor =
+        new OperatorActionProcessor(queue, publisher, new SimpleMeterRegistry());
+
+    processor.drain();
+
+    verify(queue, never()).complete(queued.action());
+    verify(queue, never()).cleanup(queued);
+    assertThat(processor.isHealthy()).isFalse();
+
+    when(queue.pendingCount()).thenReturn(1L);
+    processor.drain();
+
+    assertThat(processor.isHealthy()).isFalse();
+  }
+
   private static QueuedOperatorMarketAction queuedAction() {
     OperatorMarketAction action =
         new OperatorMarketAction(
