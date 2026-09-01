# 원시 Kafka 레코드 소비와 재시도·DLT 복구

## `chore(kafka): define event delivery defaults`

diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index 61fe403..730568d 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -16,6 +16,24 @@ spring:
       database: ${GATEWAY_REDIS_DATABASE:0}
       ssl:
         enabled: ${GATEWAY_REDIS_SSL:false}
+  kafka:
+    bootstrap-servers: ${GATEWAY_KAFKA_BOOTSTRAP:localhost:9092}
+    consumer:
+      auto-offset-reset: latest
+      enable-auto-commit: false
+      key-deserializer: org.apache.kafka.common.serialization.ByteArrayDeserializer
+      value-deserializer: org.apache.kafka.common.serialization.ByteArrayDeserializer
+    producer:
+      acks: all
+      key-serializer: org.apache.kafka.common.serialization.ByteArraySerializer
+      value-serializer: org.apache.kafka.common.serialization.ByteArraySerializer
+      properties:
+        enable.idempotence: true
+        delivery.timeout.ms: ${GATEWAY_KAFKA_DELIVERY_TIMEOUT_MS:10000}
+        request.timeout.ms: ${GATEWAY_KAFKA_REQUEST_TIMEOUT_MS:5000}
+        max.block.ms: ${GATEWAY_KAFKA_MAX_BLOCK_MS:5000}
+    listener:
+      ack-mode: record
   lifecycle:
     timeout-per-shutdown-phase: 20s
 
@@ -42,5 +60,15 @@ gateway:
       capacity: ${GATEWAY_RATELIMIT_IP_CAPACITY:60}
       refill-period: ${GATEWAY_RATELIMIT_IP_PERIOD:1m}
     trusted-proxy-cidrs: ${GATEWAY_TRUSTED_PROXY_CIDRS:}
+  topics:
+    odds-changed: ${GATEWAY_TOPIC_ODDS_CHANGED:odds.changed}
+    bet-settled: ${GATEWAY_TOPIC_BET_SETTLED:bet.settled.v1}
+    bet-voided: ${GATEWAY_TOPIC_BET_VOIDED:bet.voided.v1}
+    bet-resolution-revised: ${GATEWAY_TOPIC_BET_RESOLUTION_REVISED:bet.resolution.revised.v1}
+  kafka:
+    retry-interval: ${GATEWAY_KAFKA_RETRY_INTERVAL:1s}
+    retry-attempts: ${GATEWAY_KAFKA_RETRY_ATTEMPTS:2}
+    dlt-wait-timeout: ${GATEWAY_KAFKA_DLT_WAIT_TIMEOUT:11s}
+    dlt-timeout-buffer: ${GATEWAY_KAFKA_DLT_TIMEOUT_BUFFER:1s}
   ws:
     allowed-origins: ${GATEWAY_WS_ALLOWED_ORIGINS:http://localhost:[*],http://127.0.0.1:[*]}


## `feat(kafka): consume raw event records`

diff --git a/src/main/java/com/sportsbook/gateway/kafka/GatewayKafkaConsumerConfiguration.java b/src/main/java/com/sportsbook/gateway/kafka/GatewayKafkaConsumerConfiguration.java
new file mode 100644
index 0000000..799b7c3
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/kafka/GatewayKafkaConsumerConfiguration.java
@@ -0,0 +1,47 @@
+package com.sportsbook.gateway.kafka;
+
+import java.util.HashMap;
+import java.util.Map;
+import org.apache.kafka.clients.consumer.ConsumerConfig;
+import org.apache.kafka.common.serialization.ByteArrayDeserializer;
+import org.springframework.beans.factory.ObjectProvider;
+import org.springframework.boot.autoconfigure.kafka.KafkaProperties;
+import org.springframework.boot.ssl.SslBundles;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.kafka.annotation.EnableKafka;
+import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
+import org.springframework.kafka.core.ConsumerFactory;
+import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
+import org.springframework.kafka.listener.ContainerProperties;
+import org.springframework.kafka.listener.DefaultErrorHandler;
+
+/** Configures event consumers to retain Kafka keys and values as their original bytes. */
+@EnableKafka
+@Configuration(proxyBeanMethods = false)
+public class GatewayKafkaConsumerConfiguration {
+
+  @Bean
+  ConsumerFactory<byte[], byte[]> gatewayConsumerFactory(
+      KafkaProperties properties, SslBundles sslBundles) {
+    Map<String, Object> configuration =
+        new HashMap<>(properties.buildConsumerProperties(sslBundles));
+    configuration.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
+    configuration.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, ByteArrayDeserializer.class);
+    configuration.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, ByteArrayDeserializer.class);
+    return new DefaultKafkaConsumerFactory<>(configuration);
+  }
+
+  @Bean
+  ConcurrentKafkaListenerContainerFactory<byte[], byte[]> kafkaListenerContainerFactory(
+      ConsumerFactory<byte[], byte[]> gatewayConsumerFactory,
+      ObjectProvider<DefaultErrorHandler> errorHandler) {
+    ConcurrentKafkaListenerContainerFactory<byte[], byte[]> factory =
+        new ConcurrentKafkaListenerContainerFactory<>();
+    factory.setConsumerFactory(gatewayConsumerFactory);
+    factory.setBatchListener(false);
+    errorHandler.ifAvailable(factory::setCommonErrorHandler);
+    factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.RECORD);
+    return factory;
+  }
+}


## `test(kafka): verify record acknowledgment semantics`

diff --git a/src/test/java/com/sportsbook/gateway/kafka/GatewayKafkaConsumerConfigurationTest.java b/src/test/java/com/sportsbook/gateway/kafka/GatewayKafkaConsumerConfigurationTest.java
new file mode 100644
index 0000000..be260d8
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/kafka/GatewayKafkaConsumerConfigurationTest.java
@@ -0,0 +1,35 @@
+package com.sportsbook.gateway.kafka;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import org.apache.kafka.clients.consumer.ConsumerConfig;
+import org.apache.kafka.common.serialization.ByteArrayDeserializer;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
+import org.springframework.kafka.core.ConsumerFactory;
+import org.springframework.kafka.listener.ContainerProperties;
+
+@SpringBootTest(
+    properties = {"spring.main.web-application-type=none", "management.tracing.enabled=false"})
+class GatewayKafkaConsumerConfigurationTest {
+
+  @Autowired private ConsumerFactory<byte[], byte[]> consumers;
+  @Autowired private ConcurrentKafkaListenerContainerFactory<byte[], byte[]> containers;
+
+  @Test
+  void retainsRawKeysAndValuesWithoutAutoCommit() {
+    assertThat(consumers.getConfigurationProperties())
+        .containsEntry(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false)
+        .containsEntry(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, ByteArrayDeserializer.class)
+        .containsEntry(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, ByteArrayDeserializer.class);
+  }
+
+  @Test
+  void acknowledgesOneRecordAtATime() {
+    assertThat(containers.isBatchListener()).isFalse();
+    assertThat(containers.getContainerProperties().getAckMode())
+        .isEqualTo(ContainerProperties.AckMode.RECORD);
+  }
+}
diff --git a/src/test/resources/application.properties b/src/test/resources/application.properties
new file mode 100644
index 0000000..baebbb8
--- /dev/null
+++ b/src/test/resources/application.properties
@@ -0,0 +1 @@
+spring.kafka.listener.auto-startup=false


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


## `test(kafka): verify dead-letter topic mapping`

diff --git a/src/test/java/com/sportsbook/gateway/kafka/GatewayTopicPropertiesTest.java b/src/test/java/com/sportsbook/gateway/kafka/GatewayTopicPropertiesTest.java
new file mode 100644
index 0000000..e2ef77a
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/kafka/GatewayTopicPropertiesTest.java
@@ -0,0 +1,52 @@
+package com.sportsbook.gateway.kafka;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import java.util.Map;
+import org.junit.jupiter.api.Test;
+
+class GatewayTopicPropertiesTest {
+
+  private final GatewayTopicProperties topics =
+      new GatewayTopicProperties(
+          "odds.changed", "bet.settled.v1", "bet.voided.v1", "bet.resolution.revised.v1");
+
+  @Test
+  void mapsOnlyTheFourExactUppercaseDeadLetterTopics() {
+    assertThat(topics.sourceToDeadLetter())
+        .containsExactlyInAnyOrderEntriesOf(
+            Map.of(
+                "odds.changed", "odds.changed.DLT",
+                "bet.settled.v1", "bet.settled.v1.DLT",
+                "bet.voided.v1", "bet.voided.v1.DLT",
+                "bet.resolution.revised.v1", "bet.resolution.revised.v1.DLT"));
+  }
+
+  @Test
+  void retainsTheSourcePartitionAndRejectsUnknownTopics() {
+    assertThat(topics.deadLetterDestination("bet.settled.v1", 3).topic())
+        .isEqualTo("bet.settled.v1.DLT");
+    assertThat(topics.deadLetterDestination("bet.settled.v1", 3).partition()).isEqualTo(3);
+    assertThatThrownBy(() -> topics.deadLetterDestination("unknown", 0))
+        .isInstanceOf(IllegalArgumentException.class);
+  }
+
+  @Test
+  void rejectsAmbiguousInputInventories() {
+    assertThatThrownBy(
+            () ->
+                new GatewayTopicProperties(
+                    "odds.changed", "odds.changed", "bet.voided.v1", "bet.resolution.revised.v1"))
+        .isInstanceOf(IllegalArgumentException.class);
+    assertThatThrownBy(
+            () ->
+                new GatewayTopicProperties(
+                    "odds.changed.DLT",
+                    "bet.settled.v1",
+                    "bet.voided.v1",
+                    "bet.resolution.revised.v1"))
+        .isInstanceOf(IllegalArgumentException.class)
+        .hasMessageContaining("must not itself be a DLT");
+  }
+}


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


## `test(kafka): verify bounded dead-letter producer`

diff --git a/src/test/java/com/sportsbook/gateway/kafka/GatewayDeadLetterProducerTest.java b/src/test/java/com/sportsbook/gateway/kafka/GatewayDeadLetterProducerTest.java
new file mode 100644
index 0000000..1b19f5c
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/kafka/GatewayDeadLetterProducerTest.java
@@ -0,0 +1,37 @@
+package com.sportsbook.gateway.kafka;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.time.Duration;
+import org.apache.kafka.clients.producer.ProducerConfig;
+import org.apache.kafka.common.serialization.ByteArraySerializer;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.kafka.core.ProducerFactory;
+
+@SpringBootTest(
+    properties = {"spring.main.web-application-type=none", "management.tracing.enabled=false"})
+class GatewayDeadLetterProducerTest {
+
+  @Autowired private ProducerFactory<byte[], byte[]> producers;
+  @Autowired private GatewayKafkaProperties properties;
+
+  @Test
+  void configuresRawIdempotentPublicationWithBoundedStages() {
+    assertThat(producers.getConfigurationProperties())
+        .containsEntry(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, ByteArraySerializer.class)
+        .containsEntry(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, ByteArraySerializer.class)
+        .containsEntry(ProducerConfig.ACKS_CONFIG, "all")
+        .containsEntry(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
+    assertThat(producers.getConfigurationProperties().get(ProducerConfig.MAX_BLOCK_MS_CONFIG))
+        .hasToString("5000");
+    assertThat(producers.getConfigurationProperties().get(ProducerConfig.REQUEST_TIMEOUT_MS_CONFIG))
+        .hasToString("5000");
+    assertThat(
+            producers.getConfigurationProperties().get(ProducerConfig.DELIVERY_TIMEOUT_MS_CONFIG))
+        .hasToString("10000");
+    assertThat(properties.dltWaitTimeout()).isEqualTo(Duration.ofSeconds(11));
+    assertThat(properties.dltTimeoutBuffer()).isEqualTo(Duration.ofSeconds(1));
+  }
+}


## `test(kafka): preserve raw failed records`

diff --git a/src/test/java/com/sportsbook/gateway/kafka/GatewayDeadLetterRawRecordTest.java b/src/test/java/com/sportsbook/gateway/kafka/GatewayDeadLetterRawRecordTest.java
new file mode 100644
index 0000000..55ee88f
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/kafka/GatewayDeadLetterRawRecordTest.java
@@ -0,0 +1,87 @@
+package com.sportsbook.gateway.kafka;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.nio.charset.StandardCharsets;
+import java.time.Duration;
+import java.util.Map;
+import java.util.UUID;
+import org.apache.kafka.clients.consumer.Consumer;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
+import org.apache.kafka.common.header.Header;
+import org.apache.kafka.common.serialization.ByteArrayDeserializer;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
+import org.springframework.kafka.listener.DeadLetterPublishingRecoverer;
+import org.springframework.kafka.listener.ListenerExecutionFailedException;
+import org.springframework.kafka.support.KafkaHeaders;
+import org.springframework.kafka.test.EmbeddedKafkaBroker;
+import org.springframework.kafka.test.context.EmbeddedKafka;
+import org.springframework.kafka.test.utils.KafkaTestUtils;
+
+@SpringBootTest(
+    properties = {
+      "spring.main.web-application-type=none",
+      "management.tracing.enabled=false",
+      "logging.level.kafka=ERROR",
+      "logging.level.org.apache.kafka=ERROR",
+      "logging.level.org.springframework.kafka=ERROR"
+    })
+@EmbeddedKafka(
+    partitions = 1,
+    topics = {"odds.changed", "odds.changed.DLT"},
+    bootstrapServersProperty = "spring.kafka.bootstrap-servers")
+class GatewayDeadLetterRawRecordTest {
+
+  @Autowired private EmbeddedKafkaBroker broker;
+  @Autowired private DeadLetterPublishingRecoverer recoverer;
+
+  @Test
+  void retainsRawEvidenceAndAddsRecoveryMetadata() {
+    byte[] malformedUtf8Key = {(byte) 0xc3, (byte) 0x28};
+    byte[] payload = {0, (byte) 0xff, 3, 7};
+    String sourceGroup = "gateway-odds";
+    Map<String, Object> properties =
+        KafkaTestUtils.consumerProps("raw-dlt-" + UUID.randomUUID(), "false", broker);
+
+    try (Consumer<byte[], byte[]> consumer =
+        new DefaultKafkaConsumerFactory<>(
+                properties, new ByteArrayDeserializer(), new ByteArrayDeserializer())
+            .createConsumer()) {
+      consumer.subscribe(java.util.List.of("odds.changed.DLT"));
+      ConsumerRecord<byte[], byte[]> source =
+          new ConsumerRecord<>("odds.changed", 0, 7L, malformedUtf8Key, payload);
+      source.headers().add("kafka_application", new byte[] {1, 2});
+      source.headers().add("kafka_application", null);
+
+      recoverer.accept(
+          source,
+          new ListenerExecutionFailedException(
+              "failed event", sourceGroup, new IllegalStateException("contract failure")));
+
+      ConsumerRecord<byte[], byte[]> failed =
+          KafkaTestUtils.getSingleRecord(consumer, "odds.changed.DLT", Duration.ofSeconds(10));
+      assertThat(failed.key()).containsExactly(malformedUtf8Key);
+      assertThat(failed.value()).containsExactly(payload);
+      var applicationHeaders = failed.headers().headers("kafka_application").iterator();
+      assertThat(applicationHeaders.next().value()).containsExactly(1, 2);
+      assertThat(applicationHeaders.next().value()).isNull();
+      assertThat(applicationHeaders.hasNext()).isFalse();
+      assertThat(failed.headers().toArray())
+          .extracting(Header::key)
+          .contains(
+              KafkaHeaders.DLT_ORIGINAL_TOPIC,
+              KafkaHeaders.DLT_ORIGINAL_PARTITION,
+              KafkaHeaders.DLT_ORIGINAL_OFFSET,
+              KafkaHeaders.DLT_ORIGINAL_CONSUMER_GROUP,
+              KafkaHeaders.DLT_EXCEPTION_FQCN);
+      assertThat(
+              new String(
+                  failed.headers().lastHeader(KafkaHeaders.DLT_ORIGINAL_CONSUMER_GROUP).value(),
+                  StandardCharsets.UTF_8))
+          .isEqualTo(sourceGroup);
+    }
+  }
+}


