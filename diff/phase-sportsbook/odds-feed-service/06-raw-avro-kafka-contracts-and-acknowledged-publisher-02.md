## `test(publisher): verify acknowledged delivery health`

diff --git a/src/test/java/com/sportsbook/oddsfeed/publisher/OddsFeedPublisherTest.java b/src/test/java/com/sportsbook/oddsfeed/publisher/OddsFeedPublisherTest.java
index 61fbd33..e7cc97d 100644
--- a/src/test/java/com/sportsbook/oddsfeed/publisher/OddsFeedPublisherTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/publisher/OddsFeedPublisherTest.java
@@ -1,6 +1,7 @@
 package com.sportsbook.oddsfeed.publisher;
 
 import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
 
 import com.sportsbook.oddsfeed.config.KafkaTopicsProperties;
 import com.sportsbook.oddsfeed.config.PublishProperties;
@@ -63,6 +64,41 @@ class OddsFeedPublisherTest {
     assertThat(((OddsChanged) kafka.payload).getNewOdds()).isEqualTo("2.0100");
   }
 
+  @Test
+  void changesHealthOnlyAfterAcknowledgedDelivery() {
+    RecordingKafkaTemplate kafka = new RecordingKafkaTemplate();
+    BrokerAvailability availability = new BrokerAvailability();
+    OddsFeedPublisher publisher = publisher(kafka, availability);
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+    SelectionId selectionId = new SelectionId(UUID.randomUUID());
+
+    assertThat(publisher.isHealthy()).isFalse();
+    publisher.publishOddsChanged(
+        eventId,
+        marketId,
+        selectionId,
+        Odds.ofDecimal("2.00"),
+        Odds.ofDecimal("2.10"),
+        Instant.EPOCH,
+        false);
+    assertThat(publisher.isHealthy()).isTrue();
+
+    kafka.fail = true;
+    assertThatThrownBy(
+            () ->
+                publisher.publishOddsChanged(
+                    eventId,
+                    marketId,
+                    selectionId,
+                    Odds.ofDecimal("2.10"),
+                    Odds.ofDecimal("2.20"),
+                    Instant.EPOCH,
+                    false))
+        .isInstanceOf(KafkaPublishException.class);
+    assertThat(publisher.isHealthy()).isFalse();
+  }
+
   private static OddsFeedPublisher publisher(
       RecordingKafkaTemplate kafka, BrokerAvailability availability) {
     return new OddsFeedPublisher(
@@ -76,6 +112,7 @@ class OddsFeedPublisherTest {
     private String topic;
     private String key;
     private SpecificRecord payload;
+    private boolean fail;
 
     private RecordingKafkaTemplate() {
       super(new DefaultKafkaProducerFactory<>(Map.of()));
@@ -87,6 +124,9 @@ class OddsFeedPublisherTest {
       this.topic = topic;
       this.key = key;
       this.payload = payload;
+      if (fail) {
+        return CompletableFuture.failedFuture(new IllegalStateException("broker unavailable"));
+      }
       return CompletableFuture.completedFuture(null);
     }
   }


## `feat(publisher): publish critical feed events`

diff --git a/src/main/java/com/sportsbook/oddsfeed/publisher/OddsFeedPublisher.java b/src/main/java/com/sportsbook/oddsfeed/publisher/OddsFeedPublisher.java
index b2700f5..fcb1d5d 100644
--- a/src/main/java/com/sportsbook/oddsfeed/publisher/OddsFeedPublisher.java
+++ b/src/main/java/com/sportsbook/oddsfeed/publisher/OddsFeedPublisher.java
@@ -3,6 +3,12 @@ package com.sportsbook.oddsfeed.publisher;
 import com.sportsbook.oddsfeed.config.KafkaTopicsProperties;
 import com.sportsbook.oddsfeed.config.PublishProperties;
 import com.sportsbook.oddsfeed.kafka.BrokerAvailability;
+import com.sportsbook.protocol.event.EventLifecycle;
+import com.sportsbook.protocol.event.EventLifecycleStatus;
+import com.sportsbook.protocol.event.MarketStatus;
+import com.sportsbook.protocol.event.MarketStatusChanged;
+import com.sportsbook.protocol.event.MatchFinalStatus;
+import com.sportsbook.protocol.event.MatchResult;
 import com.sportsbook.protocol.event.OddsChanged;
 import com.sportsbook.protocol.value.EventId;
 import com.sportsbook.protocol.value.MarketId;
@@ -11,6 +17,7 @@ import com.sportsbook.protocol.value.SelectionId;
 import java.math.BigDecimal;
 import java.math.RoundingMode;
 import java.time.Instant;
+import java.util.Map;
 import java.util.concurrent.ExecutionException;
 import java.util.concurrent.TimeUnit;
 import java.util.concurrent.TimeoutException;
@@ -63,6 +70,45 @@ public class OddsFeedPublisher {
     return true;
   }
 
+  public void publishMarketStatusChanged(
+      EventId eventId,
+      MarketId marketId,
+      MarketStatus previous,
+      MarketStatus next,
+      String reason,
+      Instant occurredAt) {
+    send(
+        topics.marketStatusChanged(),
+        eventId,
+        new MarketStatusChanged(
+            eventId.value().toString(),
+            marketId.value().toString(),
+            previous,
+            next,
+            reason,
+            occurredAt));
+  }
+
+  public void publishEventLifecycle(
+      EventId eventId, EventLifecycleStatus status, Instant scheduledStartAt, Instant occurredAt) {
+    send(
+        topics.eventLifecycle(),
+        eventId,
+        new EventLifecycle(eventId.value().toString(), status, occurredAt, scheduledStartAt));
+  }
+
+  public void publishMatchResult(
+      EventId eventId,
+      String score,
+      MatchFinalStatus finalStatus,
+      Map<String, String> detail,
+      Instant settledAt) {
+    send(
+        topics.matchResult(),
+        eventId,
+        new MatchResult(eventId.value().toString(), score, finalStatus, detail, settledAt));
+  }
+
   boolean isSignificantChange(Odds previous, Odds next) {
     BigDecimal difference = next.decimal().subtract(previous.decimal()).abs();
     BigDecimal relative =


## `test(publisher): verify critical event payloads`

diff --git a/src/test/java/com/sportsbook/oddsfeed/publisher/OddsFeedPublisherTest.java b/src/test/java/com/sportsbook/oddsfeed/publisher/OddsFeedPublisherTest.java
index e7cc97d..df71666 100644
--- a/src/test/java/com/sportsbook/oddsfeed/publisher/OddsFeedPublisherTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/publisher/OddsFeedPublisherTest.java
@@ -6,6 +6,9 @@ import static org.assertj.core.api.Assertions.assertThatThrownBy;
 import com.sportsbook.oddsfeed.config.KafkaTopicsProperties;
 import com.sportsbook.oddsfeed.config.PublishProperties;
 import com.sportsbook.oddsfeed.kafka.BrokerAvailability;
+import com.sportsbook.protocol.event.EventLifecycleStatus;
+import com.sportsbook.protocol.event.MarketStatus;
+import com.sportsbook.protocol.event.MatchFinalStatus;
 import com.sportsbook.protocol.event.OddsChanged;
 import com.sportsbook.protocol.value.EventId;
 import com.sportsbook.protocol.value.MarketId;
@@ -14,6 +17,8 @@ import com.sportsbook.protocol.value.SelectionId;
 import java.math.BigDecimal;
 import java.time.Duration;
 import java.time.Instant;
+import java.util.ArrayList;
+import java.util.List;
 import java.util.Map;
 import java.util.UUID;
 import java.util.concurrent.CompletableFuture;
@@ -99,6 +104,32 @@ class OddsFeedPublisherTest {
     assertThat(publisher.isHealthy()).isFalse();
   }
 
+  @Test
+  void publishesCriticalEventsWithTheirContractPayloads() {
+    RecordingKafkaTemplate kafka = new RecordingKafkaTemplate();
+    OddsFeedPublisher publisher = publisher(kafka, new BrokerAvailability());
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+
+    publisher.publishMarketStatusChanged(
+        eventId,
+        marketId,
+        MarketStatus.OPEN,
+        MarketStatus.SUSPENDED,
+        "feed unavailable",
+        Instant.EPOCH);
+    publisher.publishEventLifecycle(
+        eventId, EventLifecycleStatus.FINISHED, Instant.EPOCH, Instant.EPOCH.plusSeconds(10));
+    publisher.publishMatchResult(
+        eventId, "2-1", MatchFinalStatus.COMPLETED, Map.of("winner", "home"), Instant.EPOCH);
+
+    assertThat(kafka.payloads)
+        .extracting(value -> value.getClass().getSimpleName())
+        .containsExactly("MarketStatusChanged", "EventLifecycle", "MatchResult");
+    assertThat(kafka.key).isEqualTo(eventId.value().toString());
+    assertThat(kafka.topic).isEqualTo("result");
+  }
+
   private static OddsFeedPublisher publisher(
       RecordingKafkaTemplate kafka, BrokerAvailability availability) {
     return new OddsFeedPublisher(
@@ -112,6 +143,7 @@ class OddsFeedPublisherTest {
     private String topic;
     private String key;
     private SpecificRecord payload;
+    private final List<SpecificRecord> payloads = new ArrayList<>();
     private boolean fail;
 
     private RecordingKafkaTemplate() {
@@ -124,6 +156,7 @@ class OddsFeedPublisherTest {
       this.topic = topic;
       this.key = key;
       this.payload = payload;
+      this.payloads.add(payload);
       if (fail) {
         return CompletableFuture.failedFuture(new IllegalStateException("broker unavailable"));
       }
