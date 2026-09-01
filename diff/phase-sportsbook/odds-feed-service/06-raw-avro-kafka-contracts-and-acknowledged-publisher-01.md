# Raw Avro Kafka 계약과 승인 기반 발행기

## `feat(kafka): bind publishing contracts`

diff --git a/src/main/java/com/sportsbook/oddsfeed/config/KafkaTopicsProperties.java b/src/main/java/com/sportsbook/oddsfeed/config/KafkaTopicsProperties.java
new file mode 100644
index 0000000..9c9ddb3
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/config/KafkaTopicsProperties.java
@@ -0,0 +1,7 @@
+package com.sportsbook.oddsfeed.config;
+
+import org.springframework.boot.context.properties.ConfigurationProperties;
+
+@ConfigurationProperties(prefix = "oddsfeed.kafka.topics")
+public record KafkaTopicsProperties(
+    String oddsChanged, String marketStatusChanged, String eventLifecycle, String matchResult) {}
diff --git a/src/main/java/com/sportsbook/oddsfeed/config/PublishProperties.java b/src/main/java/com/sportsbook/oddsfeed/config/PublishProperties.java
new file mode 100644
index 0000000..332dd68
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/config/PublishProperties.java
@@ -0,0 +1,8 @@
+package com.sportsbook.oddsfeed.config;
+
+import java.math.BigDecimal;
+import java.time.Duration;
+import org.springframework.boot.context.properties.ConfigurationProperties;
+
+@ConfigurationProperties(prefix = "oddsfeed.publish")
+public record PublishProperties(BigDecimal oddsChangeThreshold, Duration brokerAckTimeout) {}


## `test(kafka): verify publishing contracts`

diff --git a/src/test/java/com/sportsbook/oddsfeed/config/PublishingPropertiesTest.java b/src/test/java/com/sportsbook/oddsfeed/config/PublishingPropertiesTest.java
new file mode 100644
index 0000000..2a145b5
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/config/PublishingPropertiesTest.java
@@ -0,0 +1,44 @@
+package com.sportsbook.oddsfeed.config;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.math.BigDecimal;
+import java.time.Duration;
+import java.util.Map;
+import org.junit.jupiter.api.Test;
+import org.springframework.boot.context.properties.bind.Bindable;
+import org.springframework.boot.context.properties.bind.Binder;
+import org.springframework.boot.context.properties.source.MapConfigurationPropertySource;
+
+class PublishingPropertiesTest {
+
+  @Test
+  void bindsTopicsAndPublishingLimits() {
+    var source =
+        new MapConfigurationPropertySource(
+            Map.of(
+                "oddsfeed.kafka.topics.odds-changed", "odds.changed.v1",
+                "oddsfeed.kafka.topics.market-status-changed", "market.status.changed.v1",
+                "oddsfeed.kafka.topics.event-lifecycle", "event.lifecycle.v1",
+                "oddsfeed.kafka.topics.match-result", "match.result.v1",
+                "oddsfeed.publish.odds-change-threshold", "0.01",
+                "oddsfeed.publish.broker-ack-timeout", "3s"));
+    Binder binder = new Binder(source);
+
+    KafkaTopicsProperties topics =
+        binder
+            .bind("oddsfeed.kafka.topics", Bindable.of(KafkaTopicsProperties.class))
+            .orElseThrow(IllegalStateException::new);
+    PublishProperties publishing =
+        binder
+            .bind("oddsfeed.publish", Bindable.of(PublishProperties.class))
+            .orElseThrow(IllegalStateException::new);
+
+    assertThat(topics.oddsChanged()).isEqualTo("odds.changed.v1");
+    assertThat(topics.marketStatusChanged()).isEqualTo("market.status.changed.v1");
+    assertThat(topics.eventLifecycle()).isEqualTo("event.lifecycle.v1");
+    assertThat(topics.matchResult()).isEqualTo("match.result.v1");
+    assertThat(publishing.oddsChangeThreshold()).isEqualByComparingTo(new BigDecimal("0.01"));
+    assertThat(publishing.brokerAckTimeout()).isEqualTo(Duration.ofSeconds(3));
+  }
+}


## `feat(kafka): define broker and topic defaults`

diff --git a/src/main/java/com/sportsbook/oddsfeed/config/ApplicationConfig.java b/src/main/java/com/sportsbook/oddsfeed/config/ApplicationConfig.java
index c2095de..1e36b15 100644
--- a/src/main/java/com/sportsbook/oddsfeed/config/ApplicationConfig.java
+++ b/src/main/java/com/sportsbook/oddsfeed/config/ApplicationConfig.java
@@ -8,7 +8,13 @@ import org.springframework.scheduling.annotation.EnableScheduling;
 
 @Configuration
 @EnableScheduling
-@EnableConfigurationProperties({MockProperties.class, RealProperties.class, CacheProperties.class})
+@EnableConfigurationProperties({
+  MockProperties.class,
+  RealProperties.class,
+  CacheProperties.class,
+  KafkaTopicsProperties.class,
+  PublishProperties.class
+})
 public class ApplicationConfig {
 
   @Bean
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index 24e18a5..5f67477 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -43,7 +43,25 @@ spring:
           max-active: 16
           max-idle: 8
           min-idle: 2
+  kafka:
+    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
+    producer:
+      acks: all
+      properties:
+        enable.idempotence: true
+        compression.type: lz4
+      key-serializer: org.apache.kafka.common.serialization.StringSerializer
+      value-serializer: org.apache.kafka.common.serialization.ByteArraySerializer
 
 oddsfeed:
+  publish:
+    odds-change-threshold: 0.01
+    broker-ack-timeout: ${KAFKA_BROKER_ACK_TIMEOUT:5s}
+  kafka:
+    topics:
+      odds-changed: odds.changed
+      market-status-changed: market.status.changed
+      event-lifecycle: event.lifecycle
+      match-result: match.result
   cache:
     ttl: 24h


## `feat(kafka): encode Avro records`

diff --git a/src/main/java/com/sportsbook/oddsfeed/kafka/AvroDeserializer.java b/src/main/java/com/sportsbook/oddsfeed/kafka/AvroDeserializer.java
new file mode 100644
index 0000000..e376606
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/kafka/AvroDeserializer.java
@@ -0,0 +1,39 @@
+package com.sportsbook.oddsfeed.kafka;
+
+import java.io.IOException;
+import org.apache.avro.Schema;
+import org.apache.avro.data.TimeConversions;
+import org.apache.avro.io.BinaryDecoder;
+import org.apache.avro.io.DecoderFactory;
+import org.apache.avro.specific.SpecificData;
+import org.apache.avro.specific.SpecificDatumReader;
+import org.apache.avro.specific.SpecificRecord;
+import org.apache.kafka.common.errors.SerializationException;
+import org.apache.kafka.common.serialization.Deserializer;
+
+public class AvroDeserializer<T extends SpecificRecord> implements Deserializer<T> {
+
+  static {
+    SpecificData.get().addLogicalTypeConversion(new TimeConversions.TimestampMillisConversion());
+  }
+
+  private final Schema schema;
+
+  public AvroDeserializer(Schema schema) {
+    this.schema = schema;
+  }
+
+  @Override
+  public T deserialize(String topic, byte[] data) {
+    if (data == null) {
+      return null;
+    }
+    try {
+      SpecificDatumReader<T> reader = new SpecificDatumReader<>(schema);
+      BinaryDecoder decoder = DecoderFactory.get().binaryDecoder(data, null);
+      return reader.read(null, decoder);
+    } catch (IOException error) {
+      throw new SerializationException("Failed to deserialize Avro record on " + topic, error);
+    }
+  }
+}
diff --git a/src/main/java/com/sportsbook/oddsfeed/kafka/AvroSerializer.java b/src/main/java/com/sportsbook/oddsfeed/kafka/AvroSerializer.java
new file mode 100644
index 0000000..f021ab5
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/kafka/AvroSerializer.java
@@ -0,0 +1,36 @@
+package com.sportsbook.oddsfeed.kafka;
+
+import java.io.ByteArrayOutputStream;
+import java.io.IOException;
+import org.apache.avro.data.TimeConversions;
+import org.apache.avro.io.BinaryEncoder;
+import org.apache.avro.io.EncoderFactory;
+import org.apache.avro.specific.SpecificData;
+import org.apache.avro.specific.SpecificDatumWriter;
+import org.apache.avro.specific.SpecificRecord;
+import org.apache.kafka.common.errors.SerializationException;
+import org.apache.kafka.common.serialization.Serializer;
+
+public class AvroSerializer<T extends SpecificRecord> implements Serializer<T> {
+
+  static {
+    SpecificData.get().addLogicalTypeConversion(new TimeConversions.TimestampMillisConversion());
+  }
+
+  @Override
+  public byte[] serialize(String topic, T data) {
+    if (data == null) {
+      return null;
+    }
+    ByteArrayOutputStream output = new ByteArrayOutputStream();
+    try {
+      SpecificDatumWriter<T> writer = new SpecificDatumWriter<>(data.getSchema());
+      BinaryEncoder encoder = EncoderFactory.get().binaryEncoder(output, null);
+      writer.write(data, encoder);
+      encoder.flush();
+      return output.toByteArray();
+    } catch (IOException error) {
+      throw new SerializationException("Failed to serialize Avro record on " + topic, error);
+    }
+  }
+}


## `test(kafka): verify Avro binary round trips`

diff --git a/src/test/java/com/sportsbook/oddsfeed/kafka/AvroSerializerTest.java b/src/test/java/com/sportsbook/oddsfeed/kafka/AvroSerializerTest.java
new file mode 100644
index 0000000..80e705c
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/kafka/AvroSerializerTest.java
@@ -0,0 +1,66 @@
+package com.sportsbook.oddsfeed.kafka;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.event.EventLifecycle;
+import com.sportsbook.protocol.event.EventLifecycleStatus;
+import com.sportsbook.protocol.event.OddsChanged;
+import java.time.Instant;
+import org.junit.jupiter.api.Test;
+
+class AvroSerializerTest {
+
+  @Test
+  void roundTripsOddsChangesWithDecimalText() {
+    OddsChanged original =
+        new OddsChanged(
+            "00000000-0000-0000-0000-000000000001",
+            "00000000-0000-0000-0000-000000000002",
+            "00000000-0000-0000-0000-000000000003",
+            "1.8500",
+            "1.9000",
+            Instant.parse("2026-05-28T10:00:00Z"));
+
+    byte[] encoded = new AvroSerializer<OddsChanged>().serialize("odds.changed", original);
+    OddsChanged decoded =
+        new AvroDeserializer<OddsChanged>(OddsChanged.getClassSchema())
+            .deserialize("odds.changed", encoded);
+
+    assertThat(decoded.getEventId()).isEqualTo(original.getEventId());
+    assertThat(decoded.getMarketId()).isEqualTo(original.getMarketId());
+    assertThat(decoded.getSelectionId()).isEqualTo(original.getSelectionId());
+    assertThat(decoded.getPreviousOdds()).isEqualTo(original.getPreviousOdds());
+    assertThat(decoded.getNewOdds()).isEqualTo(original.getNewOdds());
+    assertThat(decoded.getChangedAt()).isEqualTo(original.getChangedAt());
+  }
+
+  @Test
+  void roundTripsLifecycleTimestampsAndEnums() {
+    Instant kickoff = Instant.parse("2026-06-01T18:00:00Z");
+    Instant occurredAt = Instant.parse("2026-05-28T10:00:00Z");
+    EventLifecycle original =
+        new EventLifecycle(
+            "00000000-0000-0000-0000-000000000001",
+            EventLifecycleStatus.SCHEDULED,
+            occurredAt,
+            kickoff);
+
+    byte[] encoded = new AvroSerializer<EventLifecycle>().serialize("event.lifecycle", original);
+    EventLifecycle decoded =
+        new AvroDeserializer<EventLifecycle>(EventLifecycle.getClassSchema())
+            .deserialize("event.lifecycle", encoded);
+
+    assertThat(decoded.getStatus()).isEqualTo(EventLifecycleStatus.SCHEDULED);
+    assertThat(decoded.getOccurredAt()).isEqualTo(occurredAt);
+    assertThat(decoded.getScheduledStartAt()).isEqualTo(kickoff);
+  }
+
+  @Test
+  void preservesKafkaNullValues() {
+    assertThat(new AvroSerializer<OddsChanged>().serialize("odds.changed", null)).isNull();
+    assertThat(
+            new AvroDeserializer<OddsChanged>(OddsChanged.getClassSchema())
+                .deserialize("odds.changed", null))
+        .isNull();
+  }
+}


## `feat(kafka): configure typed producers`

diff --git a/src/main/java/com/sportsbook/oddsfeed/kafka/KafkaConfig.java b/src/main/java/com/sportsbook/oddsfeed/kafka/KafkaConfig.java
new file mode 100644
index 0000000..7cc7fda
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/kafka/KafkaConfig.java
@@ -0,0 +1,35 @@
+package com.sportsbook.oddsfeed.kafka;
+
+import com.sportsbook.oddsfeed.config.PublishProperties;
+import java.util.HashMap;
+import java.util.Map;
+import org.apache.avro.specific.SpecificRecord;
+import org.apache.kafka.clients.producer.ProducerConfig;
+import org.apache.kafka.common.serialization.StringSerializer;
+import org.springframework.boot.autoconfigure.kafka.KafkaProperties;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.kafka.core.DefaultKafkaProducerFactory;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.kafka.core.ProducerFactory;
+
+@Configuration
+public class KafkaConfig {
+
+  @Bean
+  public ProducerFactory<String, SpecificRecord> avroProducerFactory(
+      KafkaProperties properties, PublishProperties publishProperties) {
+    Map<String, Object> settings = new HashMap<>(properties.buildProducerProperties());
+    settings.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
+    settings.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, AvroSerializer.class);
+    settings.put(
+        ProducerConfig.MAX_BLOCK_MS_CONFIG, publishProperties.brokerAckTimeout().toMillis());
+    return new DefaultKafkaProducerFactory<>(settings);
+  }
+
+  @Bean
+  public KafkaTemplate<String, SpecificRecord> avroKafkaTemplate(
+      ProducerFactory<String, SpecificRecord> producerFactory) {
+    return new KafkaTemplate<>(producerFactory);
+  }
+}


## `test(kafka): verify typed producer configuration`

diff --git a/src/test/java/com/sportsbook/oddsfeed/kafka/KafkaConfigTest.java b/src/test/java/com/sportsbook/oddsfeed/kafka/KafkaConfigTest.java
new file mode 100644
index 0000000..46b2221
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/kafka/KafkaConfigTest.java
@@ -0,0 +1,30 @@
+package com.sportsbook.oddsfeed.kafka;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.oddsfeed.config.PublishProperties;
+import java.math.BigDecimal;
+import java.time.Duration;
+import org.apache.kafka.clients.producer.ProducerConfig;
+import org.apache.kafka.common.serialization.StringSerializer;
+import org.junit.jupiter.api.Test;
+import org.springframework.boot.autoconfigure.kafka.KafkaProperties;
+import org.springframework.kafka.core.DefaultKafkaProducerFactory;
+
+class KafkaConfigTest {
+
+  @Test
+  void usesTypedSerializersAndBoundedMetadataWaits() {
+    Duration timeout = Duration.ofMillis(1234);
+    var factory =
+        new KafkaConfig()
+            .avroProducerFactory(
+                new KafkaProperties(), new PublishProperties(new BigDecimal("0.01"), timeout));
+
+    assertThat(factory).isInstanceOf(DefaultKafkaProducerFactory.class);
+    assertThat(((DefaultKafkaProducerFactory<?, ?>) factory).getConfigurationProperties())
+        .containsEntry(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class)
+        .containsEntry(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, AvroSerializer.class)
+        .containsEntry(ProducerConfig.MAX_BLOCK_MS_CONFIG, timeout.toMillis());
+  }
+}


## `feat(publisher): define Kafka publish failures`

diff --git a/src/main/java/com/sportsbook/oddsfeed/publisher/KafkaPublishException.java b/src/main/java/com/sportsbook/oddsfeed/publisher/KafkaPublishException.java
new file mode 100644
index 0000000..fb7db70
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/publisher/KafkaPublishException.java
@@ -0,0 +1,8 @@
+package com.sportsbook.oddsfeed.publisher;
+
+public class KafkaPublishException extends RuntimeException {
+
+  public KafkaPublishException(String message, Throwable cause) {
+    super(message, cause);
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


