# DLT 수동 재처리를 위한 레코드 복원과 헤더 정화

## `feat(kafka): define event topic inventory`

diff --git a/src/main/java/com/sportsbook/gateway/kafka/GatewayTopicProperties.java b/src/main/java/com/sportsbook/gateway/kafka/GatewayTopicProperties.java
new file mode 100644
index 0000000..9d27880
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/kafka/GatewayTopicProperties.java
@@ -0,0 +1,39 @@
+package com.sportsbook.gateway.kafka;
+
+import java.util.Set;
+import org.springframework.boot.context.properties.ConfigurationProperties;
+import org.springframework.util.StringUtils;
+
+/** Names the four event streams consumed by the gateway. */
+@ConfigurationProperties(prefix = "gateway.topics")
+public record GatewayTopicProperties(
+    String oddsChanged, String betSettled, String betVoided, String betResolutionRevised) {
+
+  private static final int INPUT_TOPIC_COUNT = 4;
+
+  public GatewayTopicProperties {
+    requireTopic(oddsChanged, "odds-changed");
+    requireTopic(betSettled, "bet-settled");
+    requireTopic(betVoided, "bet-voided");
+    requireTopic(betResolutionRevised, "bet-resolution-revised");
+    if (inputTopics(oddsChanged, betSettled, betVoided, betResolutionRevised).size()
+        != INPUT_TOPIC_COUNT) {
+      throw new IllegalArgumentException("gateway input topics must be distinct");
+    }
+  }
+
+  public Set<String> inputTopics() {
+    return inputTopics(oddsChanged, betSettled, betVoided, betResolutionRevised);
+  }
+
+  private static Set<String> inputTopics(
+      String odds, String settled, String voided, String revised) {
+    return Set.of(odds, settled, voided, revised);
+  }
+
+  private static void requireTopic(String value, String property) {
+    if (!StringUtils.hasText(value)) {
+      throw new IllegalArgumentException("gateway.topics." + property + " must not be blank");
+    }
+  }
+}


## `feat(kafka): route failures by source topic`

diff --git a/src/main/java/com/sportsbook/gateway/kafka/GatewayTopicProperties.java b/src/main/java/com/sportsbook/gateway/kafka/GatewayTopicProperties.java
index 9d27880..cdcb9ad 100644
--- a/src/main/java/com/sportsbook/gateway/kafka/GatewayTopicProperties.java
+++ b/src/main/java/com/sportsbook/gateway/kafka/GatewayTopicProperties.java
@@ -1,6 +1,9 @@
 package com.sportsbook.gateway.kafka;
 
+import java.util.LinkedHashMap;
+import java.util.Map;
 import java.util.Set;
+import org.apache.kafka.common.TopicPartition;
 import org.springframework.boot.context.properties.ConfigurationProperties;
 import org.springframework.util.StringUtils;
 
@@ -10,6 +13,7 @@ public record GatewayTopicProperties(
     String oddsChanged, String betSettled, String betVoided, String betResolutionRevised) {
 
   private static final int INPUT_TOPIC_COUNT = 4;
+  private static final String DEAD_LETTER_SUFFIX = ".DLT";
 
   public GatewayTopicProperties {
     requireTopic(oddsChanged, "odds-changed");
@@ -26,6 +30,20 @@ public record GatewayTopicProperties(
     return inputTopics(oddsChanged, betSettled, betVoided, betResolutionRevised);
   }
 
+  public TopicPartition deadLetterDestination(String sourceTopic, int sourcePartition) {
+    String destination = sourceToDeadLetter().get(sourceTopic);
+    if (destination == null) {
+      throw new IllegalArgumentException("No gateway DLT is defined for topic " + sourceTopic);
+    }
+    return new TopicPartition(destination, sourcePartition);
+  }
+
+  public Map<String, String> sourceToDeadLetter() {
+    Map<String, String> destinations = new LinkedHashMap<>();
+    inputTopics().forEach(topic -> destinations.put(topic, topic + DEAD_LETTER_SUFFIX));
+    return Map.copyOf(destinations);
+  }
+
   private static Set<String> inputTopics(
       String odds, String settled, String voided, String revised) {
     return Set.of(odds, settled, voided, revised);
@@ -35,5 +53,8 @@ public record GatewayTopicProperties(
     if (!StringUtils.hasText(value)) {
       throw new IllegalArgumentException("gateway.topics." + property + " must not be blank");
     }
+    if (value.endsWith(DEAD_LETTER_SUFFIX)) {
+      throw new IllegalArgumentException("gateway input topic must not itself be a DLT: " + value);
+    }
   }
 }


## `feat(kafka): bound dead-letter publication`

diff --git a/src/main/java/com/sportsbook/gateway/kafka/GatewayDeadLetterConfiguration.java b/src/main/java/com/sportsbook/gateway/kafka/GatewayDeadLetterConfiguration.java
new file mode 100644
index 0000000..a05ce59
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/kafka/GatewayDeadLetterConfiguration.java
@@ -0,0 +1,55 @@
+package com.sportsbook.gateway.kafka;
+
+import java.util.HashMap;
+import java.util.Map;
+import org.apache.kafka.clients.producer.ProducerConfig;
+import org.apache.kafka.common.serialization.ByteArraySerializer;
+import org.springframework.boot.autoconfigure.kafka.KafkaProperties;
+import org.springframework.boot.ssl.SslBundles;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.kafka.core.DefaultKafkaProducerFactory;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.kafka.core.ProducerFactory;
+import org.springframework.kafka.listener.DeadLetterPublishingRecoverer;
+
+/** Publishes failed source records to a bounded, same-partition dead-letter destination. */
+@Configuration(proxyBeanMethods = false)
+public class GatewayDeadLetterConfiguration {
+
+  @Bean
+  ProducerFactory<byte[], byte[]> gatewayProducerFactory(
+      KafkaProperties properties, SslBundles sslBundles) {
+    Map<String, Object> configuration =
+        new HashMap<>(properties.buildProducerProperties(sslBundles));
+    configuration.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, ByteArraySerializer.class);
+    configuration.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, ByteArraySerializer.class);
+    configuration.put(ProducerConfig.ACKS_CONFIG, "all");
+    configuration.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
+    return new DefaultKafkaProducerFactory<>(configuration);
+  }
+
+  @Bean
+  KafkaTemplate<byte[], byte[]> gatewayKafkaTemplate(
+      ProducerFactory<byte[], byte[]> gatewayProducerFactory) {
+    return new KafkaTemplate<>(gatewayProducerFactory);
+  }
+
+  @Bean
+  DeadLetterPublishingRecoverer gatewayDeadLetterRecoverer(
+      KafkaTemplate<byte[], byte[]> gatewayKafkaTemplate,
+      GatewayTopicProperties topics,
+      GatewayKafkaProperties properties) {
+    DeadLetterPublishingRecoverer recoverer =
+        new DeadLetterPublishingRecoverer(
+            gatewayKafkaTemplate,
+            (record, exception) ->
+                topics.deadLetterDestination(record.topic(), record.partition()));
+    recoverer.setFailIfSendResultIsError(true);
+    recoverer.setWaitForSendResultTimeout(properties.dltWaitTimeout());
+    recoverer.setTimeoutBuffer(properties.dltTimeoutBuffer().toMillis());
+    recoverer.setVerifyPartition(false);
+    recoverer.setStripPreviousExceptionHeaders(true);
+    return recoverer;
+  }
+}
diff --git a/src/main/java/com/sportsbook/gateway/kafka/GatewayKafkaProperties.java b/src/main/java/com/sportsbook/gateway/kafka/GatewayKafkaProperties.java
new file mode 100644
index 0000000..7cf8a37
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/kafka/GatewayKafkaProperties.java
@@ -0,0 +1,23 @@
+package com.sportsbook.gateway.kafka;
+
+import java.time.Duration;
+import org.springframework.boot.context.properties.ConfigurationProperties;
+import org.springframework.boot.context.properties.bind.DefaultValue;
+
+/** Bounded publication settings for failed raw event records. */
+@ConfigurationProperties(prefix = "gateway.kafka")
+public record GatewayKafkaProperties(
+    @DefaultValue("11s") Duration dltWaitTimeout,
+    @DefaultValue("1s") Duration dltTimeoutBuffer) {
+
+  public GatewayKafkaProperties {
+    requirePositive(dltWaitTimeout, "dlt-wait-timeout");
+    requirePositive(dltTimeoutBuffer, "dlt-timeout-buffer");
+  }
+
+  private static void requirePositive(Duration value, String property) {
+    if (value == null || value.isZero() || value.isNegative()) {
+      throw new IllegalArgumentException("gateway.kafka." + property + " must be positive");
+    }
+  }
+}


## `feat(kafka): sanitize manual replay records`

diff --git a/src/main/java/com/sportsbook/gateway/kafka/DltReplayRecordFactory.java b/src/main/java/com/sportsbook/gateway/kafka/DltReplayRecordFactory.java
new file mode 100644
index 0000000..1f29e60
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/kafka/DltReplayRecordFactory.java
@@ -0,0 +1,82 @@
+package com.sportsbook.gateway.kafka;
+
+import static org.springframework.kafka.support.KafkaHeaders.DELIVERY_ATTEMPT;
+
+import java.util.Map;
+import java.util.Set;
+import java.util.stream.Collectors;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
+import org.apache.kafka.clients.producer.ProducerRecord;
+import org.apache.kafka.common.header.Header;
+import org.apache.kafka.common.header.internals.RecordHeaders;
+import org.springframework.kafka.support.KafkaHeaders;
+import org.springframework.kafka.support.serializer.SerializationUtils;
+import org.springframework.stereotype.Component;
+
+/** Creates raw source records for controlled replay of gateway dead letters. */
+@Component
+public final class DltReplayRecordFactory {
+
+  private static final Set<String> FRAMEWORK_HEADERS =
+      Set.of(
+          KafkaHeaders.DLT_EXCEPTION_FQCN,
+          KafkaHeaders.DLT_EXCEPTION_CAUSE_FQCN,
+          KafkaHeaders.DLT_EXCEPTION_STACKTRACE,
+          KafkaHeaders.DLT_EXCEPTION_MESSAGE,
+          KafkaHeaders.DLT_KEY_EXCEPTION_FQCN,
+          KafkaHeaders.DLT_KEY_EXCEPTION_STACKTRACE,
+          KafkaHeaders.DLT_KEY_EXCEPTION_MESSAGE,
+          KafkaHeaders.DLT_ORIGINAL_TOPIC,
+          KafkaHeaders.DLT_ORIGINAL_PARTITION,
+          KafkaHeaders.DLT_ORIGINAL_OFFSET,
+          KafkaHeaders.DLT_ORIGINAL_CONSUMER_GROUP,
+          KafkaHeaders.DLT_ORIGINAL_TIMESTAMP,
+          KafkaHeaders.DLT_ORIGINAL_TIMESTAMP_TYPE,
+          KafkaHeaders.EXCEPTION_FQCN,
+          KafkaHeaders.EXCEPTION_CAUSE_FQCN,
+          KafkaHeaders.EXCEPTION_STACKTRACE,
+          KafkaHeaders.EXCEPTION_MESSAGE,
+          KafkaHeaders.KEY_EXCEPTION_FQCN,
+          KafkaHeaders.KEY_EXCEPTION_STACKTRACE,
+          KafkaHeaders.KEY_EXCEPTION_MESSAGE,
+          KafkaHeaders.ORIGINAL_TOPIC,
+          KafkaHeaders.ORIGINAL_PARTITION,
+          KafkaHeaders.ORIGINAL_OFFSET,
+          KafkaHeaders.ORIGINAL_TIMESTAMP,
+          KafkaHeaders.ORIGINAL_TIMESTAMP_TYPE,
+          DELIVERY_ATTEMPT,
+          SerializationUtils.KEY_DESERIALIZER_EXCEPTION_HEADER,
+          SerializationUtils.VALUE_DESERIALIZER_EXCEPTION_HEADER);
+
+  private final Map<String, String> deadLetterToSource;
+
+  public DltReplayRecordFactory(GatewayTopicProperties topics) {
+    deadLetterToSource =
+        topics.sourceToDeadLetter().entrySet().stream()
+            .collect(Collectors.toUnmodifiableMap(Map.Entry::getValue, Map.Entry::getKey));
+  }
+
+  public ProducerRecord<byte[], byte[]> replay(ConsumerRecord<byte[], byte[]> deadLetterRecord) {
+    String sourceTopic = deadLetterToSource.get(deadLetterRecord.topic());
+    if (sourceTopic == null) {
+      throw new IllegalArgumentException(
+          "Record is not from an exact gateway DLT: " + deadLetterRecord.topic());
+    }
+    RecordHeaders sanitized = new RecordHeaders();
+    for (Header header : deadLetterRecord.headers()) {
+      if (!FRAMEWORK_HEADERS.contains(header.key())) {
+        sanitized.add(header.key(), clone(header.value()));
+      }
+    }
+    return new ProducerRecord<>(
+        sourceTopic,
+        deadLetterRecord.partition(),
+        clone(deadLetterRecord.key()),
+        clone(deadLetterRecord.value()),
+        sanitized);
+  }
+
+  private static byte[] clone(byte[] bytes) {
+    return bytes == null ? null : bytes.clone();
+  }
+}


## `test(kafka): verify manual replay contract`

diff --git a/src/test/java/com/sportsbook/gateway/kafka/DltReplayRecordFactoryTest.java b/src/test/java/com/sportsbook/gateway/kafka/DltReplayRecordFactoryTest.java
new file mode 100644
index 0000000..42c15dc
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/kafka/DltReplayRecordFactoryTest.java
@@ -0,0 +1,80 @@
+package com.sportsbook.gateway.kafka;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import java.util.List;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
+import org.apache.kafka.common.header.Header;
+import org.junit.jupiter.api.Test;
+import org.springframework.kafka.support.KafkaHeaders;
+import org.springframework.kafka.support.serializer.SerializationUtils;
+
+class DltReplayRecordFactoryTest {
+
+  private static final List<String> FRAMEWORK_HEADERS =
+      List.of(
+          KafkaHeaders.DLT_EXCEPTION_FQCN,
+          KafkaHeaders.DLT_EXCEPTION_CAUSE_FQCN,
+          KafkaHeaders.DLT_EXCEPTION_STACKTRACE,
+          KafkaHeaders.DLT_EXCEPTION_MESSAGE,
+          KafkaHeaders.DLT_KEY_EXCEPTION_FQCN,
+          KafkaHeaders.DLT_KEY_EXCEPTION_STACKTRACE,
+          KafkaHeaders.DLT_KEY_EXCEPTION_MESSAGE,
+          KafkaHeaders.DLT_ORIGINAL_TOPIC,
+          KafkaHeaders.DLT_ORIGINAL_PARTITION,
+          KafkaHeaders.DLT_ORIGINAL_OFFSET,
+          KafkaHeaders.DLT_ORIGINAL_CONSUMER_GROUP,
+          KafkaHeaders.DLT_ORIGINAL_TIMESTAMP,
+          KafkaHeaders.DLT_ORIGINAL_TIMESTAMP_TYPE,
+          KafkaHeaders.EXCEPTION_FQCN,
+          KafkaHeaders.EXCEPTION_CAUSE_FQCN,
+          KafkaHeaders.EXCEPTION_STACKTRACE,
+          KafkaHeaders.EXCEPTION_MESSAGE,
+          KafkaHeaders.KEY_EXCEPTION_FQCN,
+          KafkaHeaders.KEY_EXCEPTION_STACKTRACE,
+          KafkaHeaders.KEY_EXCEPTION_MESSAGE,
+          KafkaHeaders.ORIGINAL_TOPIC,
+          KafkaHeaders.ORIGINAL_PARTITION,
+          KafkaHeaders.ORIGINAL_OFFSET,
+          KafkaHeaders.ORIGINAL_TIMESTAMP,
+          KafkaHeaders.ORIGINAL_TIMESTAMP_TYPE,
+          KafkaHeaders.DELIVERY_ATTEMPT,
+          SerializationUtils.KEY_DESERIALIZER_EXCEPTION_HEADER,
+          SerializationUtils.VALUE_DESERIALIZER_EXCEPTION_HEADER);
+
+  @Test
+  void preservesRawApplicationEvidenceAndRemovesFrameworkMetadata() {
+    byte[] key = {(byte) 0xc3, 0x28};
+    byte[] value = {0, (byte) 0xff, 7};
+    byte[] first = {1, 2};
+    byte[] prefixed = {3, 4};
+    ConsumerRecord<byte[], byte[]> deadLetter =
+        new ConsumerRecord<>("odds.changed.DLT", 3, 9L, key, value);
+    deadLetter.headers().add("application", first);
+    deadLetter.headers().add("application", null);
+    deadLetter.headers().add("kafka_application", prefixed);
+    FRAMEWORK_HEADERS.forEach(name -> deadLetter.headers().add(name, new byte[] {8}));
+
+    DltReplayRecordFactory factory =
+        new DltReplayRecordFactory(
+            new GatewayTopicProperties(
+                "odds.changed", "bet.settled.v1", "bet.voided.v1", "bet.resolution.revised.v1"));
+    var replay = factory.replay(deadLetter);
+
+    assertThat(replay.topic()).isEqualTo("odds.changed");
+    assertThat(replay.partition()).isEqualTo(3);
+    assertThat(replay.key()).containsExactly(key).isNotSameAs(key);
+    assertThat(replay.value()).containsExactly(value).isNotSameAs(value);
+    var application = replay.headers().headers("application").iterator();
+    assertThat(application.next().value()).containsExactly(first).isNotSameAs(first);
+    assertThat(application.next().value()).isNull();
+    assertThat(application.hasNext()).isFalse();
+    Header kafkaApplication = replay.headers().lastHeader("kafka_application");
+    assertThat(kafkaApplication.value()).containsExactly(prefixed).isNotSameAs(prefixed);
+    FRAMEWORK_HEADERS.forEach(name -> assertThat(replay.headers().lastHeader(name)).isNull());
+    assertThatThrownBy(
+            () -> factory.replay(new ConsumerRecord<>("odds.changed.DLT.DLT", 3, 9L, key, value)))
+        .isInstanceOf(IllegalArgumentException.class);
+  }
+}
