# Redis Stream 기반 중요 이벤트 내구 큐와 중단 복구

## `feat(delivery): bind durable delivery settings`

diff --git a/src/main/java/com/sportsbook/oddsfeed/config/ApplicationConfig.java b/src/main/java/com/sportsbook/oddsfeed/config/ApplicationConfig.java
index 1e36b15..80d261c 100644
--- a/src/main/java/com/sportsbook/oddsfeed/config/ApplicationConfig.java
+++ b/src/main/java/com/sportsbook/oddsfeed/config/ApplicationConfig.java
@@ -13,7 +13,8 @@ import org.springframework.scheduling.annotation.EnableScheduling;
   RealProperties.class,
   CacheProperties.class,
   KafkaTopicsProperties.class,
-  PublishProperties.class
+  PublishProperties.class,
+  CriticalDeliveryProperties.class
 })
 public class ApplicationConfig {
 
diff --git a/src/main/java/com/sportsbook/oddsfeed/config/CriticalDeliveryProperties.java b/src/main/java/com/sportsbook/oddsfeed/config/CriticalDeliveryProperties.java
new file mode 100644
index 0000000..0ad0f59
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/config/CriticalDeliveryProperties.java
@@ -0,0 +1,12 @@
+package com.sportsbook.oddsfeed.config;
+
+import java.time.Duration;
+import org.springframework.boot.context.properties.ConfigurationProperties;
+
+@ConfigurationProperties(prefix = "oddsfeed.delivery")
+public record CriticalDeliveryProperties(
+    String streamKey,
+    String consumerGroup,
+    String consumerName,
+    int batchSize,
+    Duration claimIdle) {}


## `test(delivery): verify durable delivery settings`

diff --git a/src/test/java/com/sportsbook/oddsfeed/config/CriticalDeliveryPropertiesTest.java b/src/test/java/com/sportsbook/oddsfeed/config/CriticalDeliveryPropertiesTest.java
new file mode 100644
index 0000000..2e542b9
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/config/CriticalDeliveryPropertiesTest.java
@@ -0,0 +1,36 @@
+package com.sportsbook.oddsfeed.config;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.time.Duration;
+import java.util.Map;
+import org.junit.jupiter.api.Test;
+import org.springframework.boot.context.properties.bind.Bindable;
+import org.springframework.boot.context.properties.bind.Binder;
+import org.springframework.boot.context.properties.source.MapConfigurationPropertySource;
+
+class CriticalDeliveryPropertiesTest {
+
+  @Test
+  void bindsDurableStreamSettings() {
+    var source =
+        new MapConfigurationPropertySource(
+            Map.of(
+                "oddsfeed.delivery.stream-key", "critical-events",
+                "oddsfeed.delivery.consumer-group", "publisher",
+                "oddsfeed.delivery.consumer-name", "publisher-1",
+                "oddsfeed.delivery.batch-size", "25",
+                "oddsfeed.delivery.claim-idle", "10s"));
+
+    CriticalDeliveryProperties properties =
+        new Binder(source)
+            .bind("oddsfeed.delivery", Bindable.of(CriticalDeliveryProperties.class))
+            .orElseThrow(IllegalStateException::new);
+
+    assertThat(properties.streamKey()).isEqualTo("critical-events");
+    assertThat(properties.consumerGroup()).isEqualTo("publisher");
+    assertThat(properties.consumerName()).isEqualTo("publisher-1");
+    assertThat(properties.batchSize()).isEqualTo(25);
+    assertThat(properties.claimIdle()).isEqualTo(Duration.ofSeconds(10));
+  }
+}


## `feat(delivery): define stream delivery defaults`

diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index 5f67477..df854ab 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -63,5 +63,12 @@ oddsfeed:
       market-status-changed: market.status.changed
       event-lifecycle: event.lifecycle
       match-result: match.result
+  delivery:
+    stream-key: ${CRITICAL_EVENT_STREAM:oddsfeed:critical-events}
+    consumer-group: ${CRITICAL_EVENT_GROUP:oddsfeed-publisher}
+    consumer-name: ${HOSTNAME:oddsfeed-local}
+    batch-size: 50
+    claim-idle: ${CRITICAL_EVENT_CLAIM_IDLE:5s}
+    poll-interval-ms: ${CRITICAL_EVENT_POLL_INTERVAL_MS:250}
   cache:
     ttl: 24h


## `feat(delivery): define market and lifecycle envelopes`

diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEvent.java b/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEvent.java
new file mode 100644
index 0000000..d6fc12e
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEvent.java
@@ -0,0 +1,58 @@
+package com.sportsbook.oddsfeed.delivery;
+
+import com.sportsbook.protocol.event.EventLifecycleStatus;
+import com.sportsbook.protocol.event.MarketStatus;
+import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.MarketId;
+import java.time.Instant;
+import java.util.UUID;
+
+public record CriticalEvent(
+    Type type,
+    UUID eventId,
+    UUID marketId,
+    MarketStatus previousMarketStatus,
+    MarketStatus nextMarketStatus,
+    String reason,
+    EventLifecycleStatus lifecycleStatus,
+    Instant scheduledStartAt,
+    Instant occurredAt) {
+
+  public enum Type {
+    MARKET_STATUS,
+    EVENT_LIFECYCLE
+  }
+
+  public static CriticalEvent marketStatus(
+      EventId eventId,
+      MarketId marketId,
+      MarketStatus previous,
+      MarketStatus next,
+      String reason,
+      Instant occurredAt) {
+    return new CriticalEvent(
+        Type.MARKET_STATUS,
+        eventId.value(),
+        marketId.value(),
+        previous,
+        next,
+        reason,
+        null,
+        null,
+        occurredAt);
+  }
+
+  public static CriticalEvent lifecycle(
+      EventId eventId, EventLifecycleStatus status, Instant scheduledStartAt, Instant occurredAt) {
+    return new CriticalEvent(
+        Type.EVENT_LIFECYCLE,
+        eventId.value(),
+        null,
+        null,
+        null,
+        null,
+        status,
+        scheduledStartAt,
+        occurredAt);
+  }
+}


## `test(delivery): verify critical event envelopes`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventTest.java
new file mode 100644
index 0000000..adf9b02
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventTest.java
@@ -0,0 +1,56 @@
+package com.sportsbook.oddsfeed.delivery;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.event.EventLifecycleStatus;
+import com.sportsbook.protocol.event.MarketStatus;
+import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.MarketId;
+import java.time.Instant;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class CriticalEventTest {
+
+  private final EventId eventId = new EventId(UUID.randomUUID());
+  private final MarketId marketId = new MarketId(UUID.randomUUID());
+
+  @Test
+  void capturesMarketTransitionValues() {
+    Instant occurredAt = Instant.parse("2026-06-01T18:00:00Z");
+
+    CriticalEvent event =
+        CriticalEvent.marketStatus(
+            eventId,
+            marketId,
+            MarketStatus.OPEN,
+            MarketStatus.SUSPENDED,
+            "feed unavailable",
+            occurredAt);
+
+    assertThat(event.type()).isEqualTo(CriticalEvent.Type.MARKET_STATUS);
+    assertThat(event.eventId()).isEqualTo(eventId.value());
+    assertThat(event.marketId()).isEqualTo(marketId.value());
+    assertThat(event.previousMarketStatus()).isEqualTo(MarketStatus.OPEN);
+    assertThat(event.nextMarketStatus()).isEqualTo(MarketStatus.SUSPENDED);
+    assertThat(event.reason()).isEqualTo("feed unavailable");
+    assertThat(event.occurredAt()).isEqualTo(occurredAt);
+    assertThat(event.lifecycleStatus()).isNull();
+  }
+
+  @Test
+  void capturesLifecycleValues() {
+    Instant kickoff = Instant.parse("2026-06-01T18:00:00Z");
+    Instant occurredAt = kickoff.plusSeconds(60);
+
+    CriticalEvent event =
+        CriticalEvent.lifecycle(eventId, EventLifecycleStatus.IN_PLAY, kickoff, occurredAt);
+
+    assertThat(event.type()).isEqualTo(CriticalEvent.Type.EVENT_LIFECYCLE);
+    assertThat(event.eventId()).isEqualTo(eventId.value());
+    assertThat(event.lifecycleStatus()).isEqualTo(EventLifecycleStatus.IN_PLAY);
+    assertThat(event.scheduledStartAt()).isEqualTo(kickoff);
+    assertThat(event.occurredAt()).isEqualTo(occurredAt);
+    assertThat(event.marketId()).isNull();
+  }
+}


## `feat(delivery): persist critical events`

diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEventQueue.java b/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEventQueue.java
new file mode 100644
index 0000000..7d94a76
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEventQueue.java
@@ -0,0 +1,74 @@
+package com.sportsbook.oddsfeed.delivery;
+
+import com.fasterxml.jackson.core.JsonProcessingException;
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.oddsfeed.config.CriticalDeliveryProperties;
+import io.micrometer.core.instrument.Counter;
+import io.micrometer.core.instrument.MeterRegistry;
+import java.util.Map;
+import java.util.concurrent.atomic.AtomicBoolean;
+import java.util.concurrent.atomic.AtomicLong;
+import org.springframework.dao.DataAccessException;
+import org.springframework.data.redis.connection.stream.RecordId;
+import org.springframework.data.redis.core.StreamOperations;
+import org.springframework.data.redis.core.StringRedisTemplate;
+import org.springframework.stereotype.Component;
+
+@Component
+public class CriticalEventQueue {
+
+  private static final String PAYLOAD_FIELD = "payload";
+
+  private final StringRedisTemplate redis;
+  private final ObjectMapper objectMapper;
+  private final CriticalDeliveryProperties properties;
+  private final Counter enqueued;
+  private final Counter failures;
+  private final AtomicBoolean healthy = new AtomicBoolean(true);
+  private final AtomicLong pendingCount = new AtomicLong();
+
+  public CriticalEventQueue(
+      StringRedisTemplate redis,
+      ObjectMapper objectMapper,
+      CriticalDeliveryProperties properties,
+      MeterRegistry meterRegistry) {
+    this.redis = redis;
+    this.objectMapper = objectMapper;
+    this.properties = properties;
+    this.enqueued = meterRegistry.counter("oddsfeed.critical.delivery.enqueued");
+    this.failures = meterRegistry.counter("oddsfeed.critical.delivery.failure");
+    meterRegistry.gauge("oddsfeed.critical.delivery.pending", pendingCount);
+  }
+
+  public RecordId enqueue(CriticalEvent event) {
+    try {
+      String payload = objectMapper.writeValueAsString(event);
+      RecordId recordId =
+          streamOperations().add(properties.streamKey(), Map.of(PAYLOAD_FIELD, payload));
+      if (recordId == null) {
+        throw new IllegalStateException("Redis did not return a critical event record id");
+      }
+      healthy.set(true);
+      enqueued.increment();
+      return recordId;
+    } catch (JsonProcessingException error) {
+      throw new IllegalArgumentException("Failed to serialize critical event", error);
+    } catch (DataAccessException error) {
+      healthy.set(false);
+      failures.increment();
+      throw error;
+    }
+  }
+
+  public boolean isHealthy() {
+    return healthy.get();
+  }
+
+  public long pendingCount() {
+    return pendingCount.get();
+  }
+
+  private StreamOperations<String, String, String> streamOperations() {
+    return redis.<String, String>opsForStream();
+  }
+}


## `test(delivery): verify durable event enqueue`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventQueueTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventQueueTest.java
new file mode 100644
index 0000000..d7b9494
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventQueueTest.java
@@ -0,0 +1,82 @@
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
+import org.junit.jupiter.api.AfterEach;
+import org.junit.jupiter.api.BeforeEach;
+import org.junit.jupiter.api.Test;
+import org.springframework.data.domain.Range;
+import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
+import org.springframework.data.redis.connection.stream.MapRecord;
+import org.springframework.data.redis.connection.stream.RecordId;
+import org.springframework.data.redis.core.StringRedisTemplate;
+import org.testcontainers.containers.GenericContainer;
+import org.testcontainers.junit.jupiter.Container;
+import org.testcontainers.junit.jupiter.Testcontainers;
+
+@Testcontainers(disabledWithoutDocker = true)
+class CriticalEventQueueTest {
+
+  @Container
+  private static final GenericContainer<?> REDIS =
+      new GenericContainer<>("redis:7-alpine").withExposedPorts(6379);
+
+  private final ObjectMapper objectMapper = new ObjectMapper().findAndRegisterModules();
+  private LettuceConnectionFactory connectionFactory;
+  private StringRedisTemplate redis;
+  private String streamKey;
+  private CriticalEventQueue queue;
+
+  @BeforeEach
+  void setUp() {
+    connectionFactory = new LettuceConnectionFactory(REDIS.getHost(), REDIS.getFirstMappedPort());
+    connectionFactory.afterPropertiesSet();
+    redis = new StringRedisTemplate(connectionFactory);
+    redis.afterPropertiesSet();
+    streamKey = "critical-events:" + UUID.randomUUID();
+    queue = queue("publisher-1", Duration.ofSeconds(5));
+  }
+
+  @AfterEach
+  void tearDown() {
+    connectionFactory.destroy();
+  }
+
+  @Test
+  void appendsSerializedEventsToTheConfiguredStream() throws Exception {
+    CriticalEvent event = lifecycle(EventLifecycleStatus.SCHEDULED);
+
+    RecordId recordId = queue.enqueue(event);
+
+    List<MapRecord<String, String, String>> records =
+        redis.<String, String>opsForStream().range(streamKey, Range.unbounded());
+    assertThat(records).hasSize(1);
+    assertThat(records.get(0).getId()).isEqualTo(recordId);
+    assertThat(
+            objectMapper.readValue(records.get(0).getValue().get("payload"), CriticalEvent.class))
+        .isEqualTo(event);
+    assertThat(queue.isHealthy()).isTrue();
+  }
+
+  private CriticalEventQueue queue(String consumerName, Duration claimIdle) {
+    return new CriticalEventQueue(
+        redis,
+        objectMapper,
+        new CriticalDeliveryProperties(streamKey, "publisher", consumerName, 25, claimIdle),
+        new SimpleMeterRegistry());
+  }
+
+  private static CriticalEvent lifecycle(EventLifecycleStatus status) {
+    return CriticalEvent.lifecycle(
+        new EventId(UUID.randomUUID()), status, Instant.EPOCH, Instant.EPOCH);
+  }
+}


## `feat(delivery): consume unread critical events`

diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEventQueue.java b/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEventQueue.java
index 7d94a76..6947530 100644
--- a/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEventQueue.java
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalEventQueue.java
@@ -5,11 +5,19 @@ import com.fasterxml.jackson.databind.ObjectMapper;
 import com.sportsbook.oddsfeed.config.CriticalDeliveryProperties;
 import io.micrometer.core.instrument.Counter;
 import io.micrometer.core.instrument.MeterRegistry;
+import java.nio.charset.StandardCharsets;
+import java.util.List;
 import java.util.Map;
 import java.util.concurrent.atomic.AtomicBoolean;
 import java.util.concurrent.atomic.AtomicLong;
 import org.springframework.dao.DataAccessException;
+import org.springframework.data.redis.connection.stream.Consumer;
+import org.springframework.data.redis.connection.stream.MapRecord;
+import org.springframework.data.redis.connection.stream.ReadOffset;
 import org.springframework.data.redis.connection.stream.RecordId;
+import org.springframework.data.redis.connection.stream.StreamOffset;
+import org.springframework.data.redis.connection.stream.StreamReadOptions;
+import org.springframework.data.redis.core.RedisCallback;
 import org.springframework.data.redis.core.StreamOperations;
 import org.springframework.data.redis.core.StringRedisTemplate;
 import org.springframework.stereotype.Component;
@@ -26,6 +34,7 @@ public class CriticalEventQueue {
   private final Counter failures;
   private final AtomicBoolean healthy = new AtomicBoolean(true);
   private final AtomicLong pendingCount = new AtomicLong();
+  private final AtomicBoolean groupReady = new AtomicBoolean();
 
   public CriticalEventQueue(
       StringRedisTemplate redis,
@@ -60,6 +69,29 @@ public class CriticalEventQueue {
     }
   }
 
+  public List<QueuedCriticalEvent> poll() {
+    try {
+      ensureGroup();
+      List<MapRecord<String, String, String>> records =
+          streamOperations()
+              .read(
+                  Consumer.from(properties.consumerGroup(), properties.consumerName()),
+                  StreamReadOptions.empty().count(properties.batchSize()),
+                  StreamOffset.create(properties.streamKey(), ReadOffset.lastConsumed()));
+      healthy.set(true);
+      if (records == null || records.isEmpty()) {
+        return List.of();
+      }
+      pendingCount.addAndGet(records.size());
+      return records.stream().map(record -> decode(record, false)).toList();
+    } catch (RuntimeException error) {
+      groupReady.set(false);
+      healthy.set(false);
+      failures.increment();
+      throw error;
+    }
+  }
+
   public boolean isHealthy() {
     return healthy.get();
   }
@@ -71,4 +103,44 @@ public class CriticalEventQueue {
   private StreamOperations<String, String, String> streamOperations() {
     return redis.<String, String>opsForStream();
   }
+
+  private QueuedCriticalEvent decode(MapRecord<String, String, String> record, boolean reclaimed) {
+    String payload = record.getValue().get(PAYLOAD_FIELD);
+    if (payload == null) {
+      throw new IllegalStateException("Critical stream record has no payload: " + record.getId());
+    }
+    try {
+      return new QueuedCriticalEvent(
+          record.getId(), objectMapper.readValue(payload, CriticalEvent.class), reclaimed);
+    } catch (JsonProcessingException error) {
+      throw new IllegalStateException("Failed to deserialize critical event", error);
+    }
+  }
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
+    } catch (DataAccessException error) {
+      Throwable cause = error;
+      while (cause != null && !String.valueOf(cause.getMessage()).contains("BUSYGROUP")) {
+        cause = cause.getCause();
+      }
+      if (cause == null) {
+        throw error;
+      }
+    }
+    groupReady.set(true);
+  }
 }
diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/QueuedCriticalEvent.java b/src/main/java/com/sportsbook/oddsfeed/delivery/QueuedCriticalEvent.java
new file mode 100644
index 0000000..bc48b8f
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/QueuedCriticalEvent.java
@@ -0,0 +1,5 @@
+package com.sportsbook.oddsfeed.delivery;
+
+import org.springframework.data.redis.connection.stream.RecordId;
+
+public record QueuedCriticalEvent(RecordId recordId, CriticalEvent event, boolean reclaimed) {}


## `test(delivery): verify ordered stream consumption`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventQueueTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventQueueTest.java
index d7b9494..8e4b9f2 100644
--- a/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventQueueTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalEventQueueTest.java
@@ -67,6 +67,17 @@ class CriticalEventQueueTest {
     assertThat(queue.isHealthy()).isTrue();
   }
 
+  @Test
+  void consumesUnreadEventsInStreamOrder() {
+    CriticalEvent first = lifecycle(EventLifecycleStatus.SCHEDULED);
+    CriticalEvent second = lifecycle(EventLifecycleStatus.IN_PLAY);
+    queue.enqueue(first);
+    queue.enqueue(second);
+
+    assertThat(queue.poll()).extracting(QueuedCriticalEvent::event).containsExactly(first, second);
+    assertThat(queue.poll()).isEmpty();
+  }
+
   private CriticalEventQueue queue(String consumerName, Duration claimIdle) {
     return new CriticalEventQueue(
         redis,


