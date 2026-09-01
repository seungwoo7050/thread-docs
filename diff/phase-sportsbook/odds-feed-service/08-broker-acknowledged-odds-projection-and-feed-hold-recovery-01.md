# 브로커 승인 기반 배당 투영과 feed hold 복구

## `feat(kafka): track broker availability`

diff --git a/src/main/java/com/sportsbook/oddsfeed/kafka/BrokerAvailability.java b/src/main/java/com/sportsbook/oddsfeed/kafka/BrokerAvailability.java
new file mode 100644
index 0000000..5936b26
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/kafka/BrokerAvailability.java
@@ -0,0 +1,22 @@
+package com.sportsbook.oddsfeed.kafka;
+
+import java.util.concurrent.atomic.AtomicBoolean;
+import org.springframework.stereotype.Component;
+
+@Component
+public final class BrokerAvailability {
+
+  private final AtomicBoolean available = new AtomicBoolean();
+
+  public boolean isAvailable() {
+    return available.get();
+  }
+
+  public void markAvailable() {
+    available.set(true);
+  }
+
+  public void markUnavailable() {
+    available.set(false);
+  }
+}


## `test(kafka): verify availability transitions`

diff --git a/src/test/java/com/sportsbook/oddsfeed/kafka/BrokerAvailabilityTest.java b/src/test/java/com/sportsbook/oddsfeed/kafka/BrokerAvailabilityTest.java
new file mode 100644
index 0000000..37e3109
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/kafka/BrokerAvailabilityTest.java
@@ -0,0 +1,21 @@
+package com.sportsbook.oddsfeed.kafka;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import org.junit.jupiter.api.Test;
+
+class BrokerAvailabilityTest {
+
+  @Test
+  void startsUnavailableAndTracksAcknowledgements() {
+    BrokerAvailability availability = new BrokerAvailability();
+
+    assertThat(availability.isAvailable()).isFalse();
+
+    availability.markAvailable();
+    assertThat(availability.isAvailable()).isTrue();
+
+    availability.markUnavailable();
+    assertThat(availability.isAvailable()).isFalse();
+  }
+}


## `feat(kafka): probe broker connectivity`

diff --git a/src/main/java/com/sportsbook/oddsfeed/kafka/KafkaBrokerProbe.java b/src/main/java/com/sportsbook/oddsfeed/kafka/KafkaBrokerProbe.java
new file mode 100644
index 0000000..11b74d5
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/kafka/KafkaBrokerProbe.java
@@ -0,0 +1,37 @@
+package com.sportsbook.oddsfeed.kafka;
+
+import com.sportsbook.oddsfeed.config.KafkaTopicsProperties;
+import org.apache.avro.specific.SpecificRecord;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.scheduling.annotation.Scheduled;
+import org.springframework.stereotype.Component;
+
+@Component
+public final class KafkaBrokerProbe {
+
+  private final KafkaTemplate<String, SpecificRecord> kafka;
+  private final KafkaTopicsProperties topics;
+  private final BrokerAvailability availability;
+
+  public KafkaBrokerProbe(
+      KafkaTemplate<String, SpecificRecord> kafka,
+      KafkaTopicsProperties topics,
+      BrokerAvailability availability) {
+    this.kafka = kafka;
+    this.topics = topics;
+    this.availability = availability;
+  }
+
+  @Scheduled(fixedDelayString = "${oddsfeed.publish.probe-interval:5000}")
+  public void probe() {
+    try {
+      if (kafka.partitionsFor(topics.oddsChanged()).isEmpty()) {
+        availability.markUnavailable();
+      } else {
+        availability.markAvailable();
+      }
+    } catch (RuntimeException error) {
+      availability.markUnavailable();
+    }
+  }
+}


## `test(kafka): verify independent broker probes`

diff --git a/src/test/java/com/sportsbook/oddsfeed/kafka/KafkaBrokerProbeTest.java b/src/test/java/com/sportsbook/oddsfeed/kafka/KafkaBrokerProbeTest.java
new file mode 100644
index 0000000..9d3c74d
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/kafka/KafkaBrokerProbeTest.java
@@ -0,0 +1,93 @@
+package com.sportsbook.oddsfeed.kafka;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatCode;
+
+import com.sportsbook.oddsfeed.config.KafkaTopicsProperties;
+import java.util.List;
+import java.util.Map;
+import org.apache.avro.specific.SpecificRecord;
+import org.apache.kafka.common.KafkaException;
+import org.apache.kafka.common.Node;
+import org.apache.kafka.common.PartitionInfo;
+import org.junit.jupiter.api.Test;
+import org.springframework.context.annotation.AnnotationConfigApplicationContext;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.kafka.core.DefaultKafkaProducerFactory;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.scheduling.annotation.EnableScheduling;
+
+class KafkaBrokerProbeTest {
+
+  @Test
+  void updatesAvailabilityWithoutWaitingForARecordSend() {
+    StubKafkaTemplate kafka = new StubKafkaTemplate();
+    BrokerAvailability availability = new BrokerAvailability();
+    KafkaBrokerProbe probe =
+        new KafkaBrokerProbe(
+            kafka,
+            new KafkaTopicsProperties("odds", "market", "lifecycle", "result"),
+            availability);
+
+    probe.probe();
+    assertThat(availability.isAvailable()).isTrue();
+
+    kafka.fail = true;
+    probe.probe();
+    assertThat(availability.isAvailable()).isFalse();
+  }
+
+  @Test
+  void initializesTheDefaultProbeSchedule() {
+    assertThatCode(
+            () -> {
+              try (var context = new AnnotationConfigApplicationContext(ProbeConfiguration.class)) {
+                assertThat(context.getBean(KafkaBrokerProbe.class)).isNotNull();
+              }
+            })
+        .doesNotThrowAnyException();
+  }
+
+  @Configuration(proxyBeanMethods = false)
+  @EnableScheduling
+  static class ProbeConfiguration {
+
+    @Bean
+    StubKafkaTemplate kafkaTemplate() {
+      return new StubKafkaTemplate();
+    }
+
+    @Bean
+    KafkaTopicsProperties kafkaTopicsProperties() {
+      return new KafkaTopicsProperties("odds", "market", "lifecycle", "result");
+    }
+
+    @Bean
+    BrokerAvailability brokerAvailability() {
+      return new BrokerAvailability();
+    }
+
+    @Bean
+    KafkaBrokerProbe kafkaBrokerProbe(
+        StubKafkaTemplate kafka, KafkaTopicsProperties topics, BrokerAvailability availability) {
+      return new KafkaBrokerProbe(kafka, topics, availability);
+    }
+  }
+
+  private static final class StubKafkaTemplate extends KafkaTemplate<String, SpecificRecord> {
+    private boolean fail;
+
+    private StubKafkaTemplate() {
+      super(new DefaultKafkaProducerFactory<>(Map.of()));
+    }
+
+    @Override
+    public List<PartitionInfo> partitionsFor(String topic) {
+      if (fail) {
+        throw new KafkaException("broker unavailable");
+      }
+      return List.of(new PartitionInfo(topic, 0, Node.noNode(), null, null));
+    }
+  }
+}


## `feat(publisher): publish thresholded odds changes`

diff --git a/src/main/java/com/sportsbook/oddsfeed/publisher/OddsFeedPublisher.java b/src/main/java/com/sportsbook/oddsfeed/publisher/OddsFeedPublisher.java
new file mode 100644
index 0000000..b2700f5
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/publisher/OddsFeedPublisher.java
@@ -0,0 +1,92 @@
+package com.sportsbook.oddsfeed.publisher;
+
+import com.sportsbook.oddsfeed.config.KafkaTopicsProperties;
+import com.sportsbook.oddsfeed.config.PublishProperties;
+import com.sportsbook.oddsfeed.kafka.BrokerAvailability;
+import com.sportsbook.protocol.event.OddsChanged;
+import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.MarketId;
+import com.sportsbook.protocol.value.Odds;
+import com.sportsbook.protocol.value.SelectionId;
+import java.math.BigDecimal;
+import java.math.RoundingMode;
+import java.time.Instant;
+import java.util.concurrent.ExecutionException;
+import java.util.concurrent.TimeUnit;
+import java.util.concurrent.TimeoutException;
+import org.apache.avro.specific.SpecificRecord;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.stereotype.Component;
+
+@Component
+public class OddsFeedPublisher {
+
+  private static final int RELATIVE_SCALE = 6;
+
+  private final KafkaTemplate<String, SpecificRecord> kafka;
+  private final KafkaTopicsProperties topics;
+  private final PublishProperties properties;
+  private final BrokerAvailability availability;
+
+  public OddsFeedPublisher(
+      KafkaTemplate<String, SpecificRecord> kafka,
+      KafkaTopicsProperties topics,
+      PublishProperties properties,
+      BrokerAvailability availability) {
+    this.kafka = kafka;
+    this.topics = topics;
+    this.properties = properties;
+    this.availability = availability;
+  }
+
+  public boolean publishOddsChanged(
+      EventId eventId,
+      MarketId marketId,
+      SelectionId selectionId,
+      Odds previous,
+      Odds next,
+      Instant changedAt,
+      boolean forceCurrentSnapshot) {
+    if (!forceCurrentSnapshot && !isSignificantChange(previous, next)) {
+      return false;
+    }
+    send(
+        topics.oddsChanged(),
+        eventId,
+        new OddsChanged(
+            eventId.value().toString(),
+            marketId.value().toString(),
+            selectionId.value().toString(),
+            previous.decimal().toPlainString(),
+            next.decimal().toPlainString(),
+            changedAt));
+    return true;
+  }
+
+  boolean isSignificantChange(Odds previous, Odds next) {
+    BigDecimal difference = next.decimal().subtract(previous.decimal()).abs();
+    BigDecimal relative =
+        difference.divide(previous.decimal(), RELATIVE_SCALE, RoundingMode.HALF_EVEN);
+    return relative.compareTo(properties.oddsChangeThreshold()) >= 0;
+  }
+
+  public boolean isHealthy() {
+    return availability.isAvailable();
+  }
+
+  private void send(String topic, EventId eventId, SpecificRecord event) {
+    try {
+      kafka
+          .send(topic, eventId.value().toString(), event)
+          .get(properties.brokerAckTimeout().toMillis(), TimeUnit.MILLISECONDS);
+      availability.markAvailable();
+    } catch (InterruptedException error) {
+      Thread.currentThread().interrupt();
+      availability.markUnavailable();
+      throw new KafkaPublishException("Interrupted while awaiting Kafka acknowledgement", error);
+    } catch (ExecutionException | TimeoutException | RuntimeException error) {
+      availability.markUnavailable();
+      throw new KafkaPublishException("Kafka did not acknowledge " + topic, error);
+    }
+  }
+}


## `test(publisher): verify odds thresholds and keys`

diff --git a/src/test/java/com/sportsbook/oddsfeed/publisher/OddsFeedPublisherTest.java b/src/test/java/com/sportsbook/oddsfeed/publisher/OddsFeedPublisherTest.java
new file mode 100644
index 0000000..61fbd33
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/publisher/OddsFeedPublisherTest.java
@@ -0,0 +1,93 @@
+package com.sportsbook.oddsfeed.publisher;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.oddsfeed.config.KafkaTopicsProperties;
+import com.sportsbook.oddsfeed.config.PublishProperties;
+import com.sportsbook.oddsfeed.kafka.BrokerAvailability;
+import com.sportsbook.protocol.event.OddsChanged;
+import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.MarketId;
+import com.sportsbook.protocol.value.Odds;
+import com.sportsbook.protocol.value.SelectionId;
+import java.math.BigDecimal;
+import java.time.Duration;
+import java.time.Instant;
+import java.util.Map;
+import java.util.UUID;
+import java.util.concurrent.CompletableFuture;
+import org.apache.avro.specific.SpecificRecord;
+import org.junit.jupiter.api.Test;
+import org.springframework.kafka.core.DefaultKafkaProducerFactory;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.kafka.support.SendResult;
+
+class OddsFeedPublisherTest {
+
+  @Test
+  void publishesThresholdedChangesWithEventKeys() {
+    RecordingKafkaTemplate kafka = new RecordingKafkaTemplate();
+    OddsFeedPublisher publisher = publisher(kafka, new BrokerAvailability());
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+    SelectionId selectionId = new SelectionId(UUID.randomUUID());
+    Instant changedAt = Instant.parse("2026-06-01T18:00:00Z");
+
+    assertThat(
+            publisher.publishOddsChanged(
+                eventId,
+                marketId,
+                selectionId,
+                Odds.ofDecimal("2.00"),
+                Odds.ofDecimal("2.01"),
+                changedAt,
+                false))
+        .isFalse();
+    assertThat(kafka.payload).isNull();
+    assertThat(publisher.isSignificantChange(Odds.ofDecimal("2.00"), Odds.ofDecimal("2.03")))
+        .isTrue();
+
+    assertThat(
+            publisher.publishOddsChanged(
+                eventId,
+                marketId,
+                selectionId,
+                Odds.ofDecimal("2.00"),
+                Odds.ofDecimal("2.01"),
+                changedAt,
+                true))
+        .isTrue();
+    assertThat(kafka.topic).isEqualTo("odds.changed");
+    assertThat(kafka.key).isEqualTo(eventId.value().toString());
+    assertThat(kafka.payload).isInstanceOf(OddsChanged.class);
+    assertThat(((OddsChanged) kafka.payload).getNewOdds()).isEqualTo("2.0100");
+  }
+
+  private static OddsFeedPublisher publisher(
+      RecordingKafkaTemplate kafka, BrokerAvailability availability) {
+    return new OddsFeedPublisher(
+        kafka,
+        new KafkaTopicsProperties("odds.changed", "market", "lifecycle", "result"),
+        new PublishProperties(new BigDecimal("0.01"), Duration.ofSeconds(1)),
+        availability);
+  }
+
+  private static final class RecordingKafkaTemplate extends KafkaTemplate<String, SpecificRecord> {
+    private String topic;
+    private String key;
+    private SpecificRecord payload;
+
+    private RecordingKafkaTemplate() {
+      super(new DefaultKafkaProducerFactory<>(Map.of()));
+    }
+
+    @Override
+    public CompletableFuture<SendResult<String, SpecificRecord>> send(
+        String topic, String key, SpecificRecord payload) {
+      this.topic = topic;
+      this.key = key;
+      this.payload = payload;
+      return CompletableFuture.completedFuture(null);
+    }
+  }
+}


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


