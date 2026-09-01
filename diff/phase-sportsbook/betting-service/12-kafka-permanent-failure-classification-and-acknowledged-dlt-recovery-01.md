# Kafka 영구 오류 분류와 승인 기반 DLT 복구

## `feat(messaging): classify permanent consumer records`

diff --git a/src/main/java/com/sportsbook/betting/config/KafkaMessageValidator.java b/src/main/java/com/sportsbook/betting/config/KafkaMessageValidator.java
new file mode 100644
index 0000000..5f09285
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/config/KafkaMessageValidator.java
@@ -0,0 +1,42 @@
+package com.sportsbook.betting.config;
+
+import com.sportsbook.betting.outbox.AvroSerializer;
+import java.nio.charset.StandardCharsets;
+import java.util.UUID;
+import org.apache.avro.specific.SpecificRecord;
+
+public final class KafkaMessageValidator {
+
+  public static <T extends SpecificRecord> T decode(byte[] payload, Class<T> type) {
+    try {
+      return AvroSerializer.deserialize(payload, type);
+    } catch (RuntimeException failure) {
+      throw new PermanentKafkaException("Invalid " + type.getSimpleName() + " payload", failure);
+    }
+  }
+
+  public static UUID canonical(String value, String name) {
+    try {
+      UUID parsed = UUID.fromString(value);
+      if (!parsed.toString().equals(value)) {
+        throw new IllegalArgumentException("not canonical");
+      }
+      return parsed;
+    } catch (RuntimeException failure) {
+      throw new PermanentKafkaException(name + " must be a canonical lowercase UUID", failure);
+    }
+  }
+
+  public static void requireKey(byte[] rawKey, String expected, String name) {
+    if (rawKey == null) {
+      throw new PermanentKafkaException(name + " Kafka key is required");
+    }
+    String actual = new String(rawKey, StandardCharsets.US_ASCII);
+    UUID actualId = canonical(actual, name + " Kafka key");
+    if (!actualId.equals(canonical(expected, name)) || !actual.equals(expected)) {
+      throw new PermanentKafkaException(name + " Kafka key mismatch");
+    }
+  }
+
+  private KafkaMessageValidator() {}
+}
diff --git a/src/main/java/com/sportsbook/betting/config/PermanentKafkaException.java b/src/main/java/com/sportsbook/betting/config/PermanentKafkaException.java
new file mode 100644
index 0000000..b975ad2
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/config/PermanentKafkaException.java
@@ -0,0 +1,14 @@
+package com.sportsbook.betting.config;
+
+public final class PermanentKafkaException extends RuntimeException {
+
+  private static final long serialVersionUID = 1L;
+
+  public PermanentKafkaException(String message) {
+    super(message);
+  }
+
+  public PermanentKafkaException(String message, Throwable cause) {
+    super(message, cause);
+  }
+}


## `test(messaging): verify permanent record validation`

diff --git a/src/test/java/com/sportsbook/betting/config/KafkaMessageValidatorTest.java b/src/test/java/com/sportsbook/betting/config/KafkaMessageValidatorTest.java
new file mode 100644
index 0000000..4551c33
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/config/KafkaMessageValidatorTest.java
@@ -0,0 +1,51 @@
+package com.sportsbook.betting.config;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.betting.outbox.AvroSerializer;
+import com.sportsbook.protocol.event.Money;
+import java.nio.charset.StandardCharsets;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class KafkaMessageValidatorTest {
+
+  @Test
+  void acceptsExactRawUuidKeyAndStrictAvroBytes() {
+    UUID id = UUID.randomUUID();
+    Money money = Money.newBuilder().setAmount(1_000).setCurrency("KRW").build();
+
+    KafkaMessageValidator.requireKey(
+        id.toString().getBytes(StandardCharsets.US_ASCII), id.toString(), "betId");
+
+    assertThat(KafkaMessageValidator.decode(AvroSerializer.serialize(money), Money.class))
+        .isEqualTo(money);
+  }
+
+  @Test
+  void classifiesMalformedOrMismatchedKeysAsPermanent() {
+    UUID id = UUID.randomUUID();
+
+    assertThatThrownBy(
+            () ->
+                KafkaMessageValidator.requireKey(
+                    UUID.randomUUID().toString().getBytes(StandardCharsets.US_ASCII),
+                    id.toString(),
+                    "betId"))
+        .isInstanceOf(PermanentKafkaException.class)
+        .hasMessageContaining("mismatch");
+    assertThatThrownBy(
+            () ->
+                KafkaMessageValidator.requireKey(new byte[] {(byte) 0xff}, id.toString(), "betId"))
+        .isInstanceOf(PermanentKafkaException.class)
+        .hasMessageContaining("canonical");
+  }
+
+  @Test
+  void classifiesMalformedAvroAsPermanent() {
+    assertThatThrownBy(() -> KafkaMessageValidator.decode(new byte[] {1}, Money.class))
+        .isInstanceOf(PermanentKafkaException.class)
+        .hasMessageContaining("Invalid Money payload");
+  }
+}


## `feat(messaging): configure raw DLT publication`

diff --git a/src/main/java/com/sportsbook/betting/config/KafkaRecoveryConfig.java b/src/main/java/com/sportsbook/betting/config/KafkaRecoveryConfig.java
new file mode 100644
index 0000000..5910373
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/config/KafkaRecoveryConfig.java
@@ -0,0 +1,45 @@
+package com.sportsbook.betting.config;
+
+import java.util.HashMap;
+import java.util.Map;
+import org.apache.kafka.clients.producer.ProducerConfig;
+import org.apache.kafka.common.serialization.ByteArraySerializer;
+import org.springframework.beans.factory.annotation.Value;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.kafka.core.DefaultKafkaProducerFactory;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.kafka.core.ProducerFactory;
+
+@Configuration
+public class KafkaRecoveryConfig {
+
+  private final String bootstrapServers;
+
+  public KafkaRecoveryConfig(@Value("${spring.kafka.bootstrap-servers}") String bootstrapServers) {
+    this.bootstrapServers = bootstrapServers;
+  }
+
+  @Bean
+  ProducerFactory<byte[], byte[]> rawDltProducerFactory() {
+    return new DefaultKafkaProducerFactory<>(rawProducerProperties(bootstrapServers));
+  }
+
+  @Bean
+  KafkaTemplate<byte[], byte[]> rawDltKafkaTemplate(
+      ProducerFactory<byte[], byte[]> rawDltProducerFactory) {
+    return new KafkaTemplate<>(rawDltProducerFactory);
+  }
+
+  static Map<String, Object> rawProducerProperties(String bootstrapServers) {
+    Map<String, Object> properties =
+        new HashMap<>(KafkaConfig.producerProperties(bootstrapServers));
+    properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, ByteArraySerializer.class);
+    properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, ByteArraySerializer.class);
+    properties.put(ProducerConfig.CLIENT_ID_CONFIG, "betting-service-dlt");
+    properties.put(ProducerConfig.MAX_BLOCK_MS_CONFIG, 5_000);
+    properties.put(ProducerConfig.REQUEST_TIMEOUT_MS_CONFIG, 5_000);
+    properties.put(ProducerConfig.DELIVERY_TIMEOUT_MS_CONFIG, 10_000);
+    return Map.copyOf(properties);
+  }
+}


## `test(messaging): verify raw DLT serializers`

diff --git a/src/test/java/com/sportsbook/betting/config/KafkaRecoveryConfigTest.java b/src/test/java/com/sportsbook/betting/config/KafkaRecoveryConfigTest.java
new file mode 100644
index 0000000..f9d199e
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/config/KafkaRecoveryConfigTest.java
@@ -0,0 +1,26 @@
+package com.sportsbook.betting.config;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.util.Map;
+import org.apache.kafka.clients.producer.ProducerConfig;
+import org.apache.kafka.common.serialization.ByteArraySerializer;
+import org.junit.jupiter.api.Test;
+
+class KafkaRecoveryConfigTest {
+
+  @Test
+  void preservesRawKeysAndValuesForDltPublication() {
+    Map<String, Object> properties = KafkaRecoveryConfig.rawProducerProperties("broker:9092");
+
+    assertThat(properties.get(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG))
+        .isEqualTo(ByteArraySerializer.class);
+    assertThat(properties.get(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG))
+        .isEqualTo(ByteArraySerializer.class);
+    assertThat(properties.get(ProducerConfig.ACKS_CONFIG)).isEqualTo("all");
+    assertThat(properties.get(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG)).isEqualTo(true);
+    assertThat(properties.get(ProducerConfig.MAX_BLOCK_MS_CONFIG)).isEqualTo(5_000);
+    assertThat(properties.get(ProducerConfig.REQUEST_TIMEOUT_MS_CONFIG)).isEqualTo(5_000);
+    assertThat(properties.get(ProducerConfig.DELIVERY_TIMEOUT_MS_CONFIG)).isEqualTo(10_000);
+  }
+}


## `feat(messaging): require acknowledged permanent recovery`

diff --git a/src/main/java/com/sportsbook/betting/config/KafkaConfig.java b/src/main/java/com/sportsbook/betting/config/KafkaConfig.java
index 5c4bc5e..61d3eb5 100644
--- a/src/main/java/com/sportsbook/betting/config/KafkaConfig.java
+++ b/src/main/java/com/sportsbook/betting/config/KafkaConfig.java
@@ -3,7 +3,6 @@ package com.sportsbook.betting.config;
 import java.util.HashMap;
 import java.util.Map;
 import org.apache.kafka.clients.producer.ProducerConfig;
-import org.apache.kafka.common.TopicPartition;
 import org.apache.kafka.common.serialization.ByteArraySerializer;
 import org.apache.kafka.common.serialization.StringSerializer;
 import org.springframework.beans.factory.annotation.Value;
@@ -12,10 +11,6 @@ import org.springframework.context.annotation.Configuration;
 import org.springframework.kafka.core.DefaultKafkaProducerFactory;
 import org.springframework.kafka.core.KafkaTemplate;
 import org.springframework.kafka.core.ProducerFactory;
-import org.springframework.kafka.listener.CommonErrorHandler;
-import org.springframework.kafka.listener.DeadLetterPublishingRecoverer;
-import org.springframework.kafka.listener.DefaultErrorHandler;
-import org.springframework.util.backoff.FixedBackOff;
 
 @Configuration
 public class KafkaConfig {
@@ -37,15 +32,6 @@ public class KafkaConfig {
     return new KafkaTemplate<>(bettingProducerFactory);
   }
 
-  @Bean
-  CommonErrorHandler walletConsumerErrorHandler(KafkaTemplate<String, byte[]> kafka) {
-    DeadLetterPublishingRecoverer recoverer =
-        new DeadLetterPublishingRecoverer(
-            kafka,
-            (record, failure) -> new TopicPartition(record.topic() + ".dlt", record.partition()));
-    return new DefaultErrorHandler(recoverer, new FixedBackOff(500, 3));
-  }
-
   static Map<String, Object> producerProperties(String bootstrapServers) {
     Map<String, Object> properties = new HashMap<>();
     properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
diff --git a/src/main/java/com/sportsbook/betting/config/KafkaRecoveryConfig.java b/src/main/java/com/sportsbook/betting/config/KafkaRecoveryConfig.java
index 5910373..38f5656 100644
--- a/src/main/java/com/sportsbook/betting/config/KafkaRecoveryConfig.java
+++ b/src/main/java/com/sportsbook/betting/config/KafkaRecoveryConfig.java
@@ -1,15 +1,22 @@
 package com.sportsbook.betting.config;
 
+import java.time.Duration;
 import java.util.HashMap;
 import java.util.Map;
 import org.apache.kafka.clients.producer.ProducerConfig;
+import org.apache.kafka.common.TopicPartition;
 import org.apache.kafka.common.serialization.ByteArraySerializer;
+import org.springframework.beans.factory.annotation.Qualifier;
 import org.springframework.beans.factory.annotation.Value;
 import org.springframework.context.annotation.Bean;
 import org.springframework.context.annotation.Configuration;
 import org.springframework.kafka.core.DefaultKafkaProducerFactory;
 import org.springframework.kafka.core.KafkaTemplate;
 import org.springframework.kafka.core.ProducerFactory;
+import org.springframework.kafka.listener.CommonErrorHandler;
+import org.springframework.kafka.listener.DeadLetterPublishingRecoverer;
+import org.springframework.kafka.listener.DefaultErrorHandler;
+import org.springframework.util.backoff.FixedBackOff;
 
 @Configuration
 public class KafkaRecoveryConfig {
@@ -31,6 +38,32 @@ public class KafkaRecoveryConfig {
     return new KafkaTemplate<>(rawDltProducerFactory);
   }
 
+  @Bean
+  CommonErrorHandler kafkaConsumerErrorHandler(
+      @Qualifier("rawDltKafkaTemplate") KafkaTemplate<byte[], byte[]> kafka) {
+    return errorHandler(kafka, 1_000L);
+  }
+
+  static DefaultErrorHandler errorHandler(
+      KafkaTemplate<byte[], byte[]> kafka, long retryDelayMillis) {
+    DefaultErrorHandler handler =
+        new DefaultErrorHandler(
+            recoverer(kafka), new FixedBackOff(retryDelayMillis, FixedBackOff.UNLIMITED_ATTEMPTS));
+    handler.setClassifications(Map.of(PermanentKafkaException.class, false), true);
+    return handler;
+  }
+
+  static DeadLetterPublishingRecoverer recoverer(KafkaTemplate<byte[], byte[]> kafka) {
+    DeadLetterPublishingRecoverer recoverer =
+        new DeadLetterPublishingRecoverer(
+            kafka,
+            (record, failure) -> new TopicPartition(record.topic() + ".DLT", record.partition()));
+    recoverer.setVerifyPartition(false);
+    recoverer.setFailIfSendResultIsError(true);
+    recoverer.setWaitForSendResultTimeout(Duration.ofSeconds(11));
+    return recoverer;
+  }
+
   static Map<String, Object> rawProducerProperties(String bootstrapServers) {
     Map<String, Object> properties =
         new HashMap<>(KafkaConfig.producerProperties(bootstrapServers));


## `test(messaging): verify acknowledged permanent recovery`

diff --git a/src/test/java/com/sportsbook/betting/config/KafkaRecoveryConfigTest.java b/src/test/java/com/sportsbook/betting/config/KafkaRecoveryConfigTest.java
index f9d199e..5ba2aa1 100644
--- a/src/test/java/com/sportsbook/betting/config/KafkaRecoveryConfigTest.java
+++ b/src/test/java/com/sportsbook/betting/config/KafkaRecoveryConfigTest.java
@@ -1,12 +1,30 @@
 package com.sportsbook.betting.config;
 
 import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.ArgumentMatchers.any;
+import static org.mockito.ArgumentMatchers.anyString;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.never;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.verifyNoInteractions;
+import static org.mockito.Mockito.when;
 
 import java.util.Map;
+import java.util.concurrent.CompletableFuture;
+import org.apache.kafka.clients.consumer.Consumer;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
 import org.apache.kafka.clients.producer.ProducerConfig;
+import org.apache.kafka.clients.producer.ProducerRecord;
+import org.apache.kafka.common.errors.UnknownTopicOrPartitionException;
 import org.apache.kafka.common.serialization.ByteArraySerializer;
 import org.junit.jupiter.api.Test;
+import org.mockito.ArgumentCaptor;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.kafka.core.ProducerFactory;
+import org.springframework.kafka.listener.MessageListenerContainer;
+import org.springframework.kafka.support.SendResult;
 
+@SuppressWarnings("unchecked")
 class KafkaRecoveryConfigTest {
 
   @Test
@@ -23,4 +41,85 @@ class KafkaRecoveryConfigTest {
     assertThat(properties.get(ProducerConfig.REQUEST_TIMEOUT_MS_CONFIG)).isEqualTo(5_000);
     assertThat(properties.get(ProducerConfig.DELIVERY_TIMEOUT_MS_CONFIG)).isEqualTo(10_000);
   }
+
+  @Test
+  void publishesPermanentRecordsToTheSamePartitionWithRawBytes() {
+    KafkaTemplate<byte[], byte[]> kafka = successfulKafka();
+    byte[] key = {1, 2};
+    byte[] value = {3, 4};
+    ConsumerRecord<byte[], byte[]> source =
+        new ConsumerRecord<>("wallet.debited.v1", 3, 7, key, value);
+
+    boolean recovered =
+        KafkaRecoveryConfig.errorHandler(kafka, 0L)
+            .handleOne(
+                new PermanentKafkaException("bad record"),
+                source,
+                mock(Consumer.class),
+                mock(MessageListenerContainer.class));
+
+    ArgumentCaptor<ProducerRecord<byte[], byte[]>> sent =
+        ArgumentCaptor.forClass(ProducerRecord.class);
+    verify(kafka).send(sent.capture());
+    assertThat(recovered).isTrue();
+    assertThat(sent.getValue().topic()).isEqualTo("wallet.debited.v1.DLT");
+    assertThat(sent.getValue().partition()).isEqualTo(3);
+    assertThat(sent.getValue().key()).containsExactly(key);
+    assertThat(sent.getValue().value()).containsExactly(value);
+    verify(kafka, never()).partitionsFor(anyString());
+  }
+
+  @Test
+  void leavesTransientAndFailedDltPublicationUnrecovered() {
+    ConsumerRecord<byte[], byte[]> source =
+        new ConsumerRecord<>("bet.settled.v1", 1, 9, new byte[] {1}, new byte[] {2});
+    KafkaTemplate<byte[], byte[]> transientKafka = successfulKafka();
+    Consumer<byte[], byte[]> transientConsumer = mock(Consumer.class);
+
+    boolean transientRecovered =
+        KafkaRecoveryConfig.errorHandler(transientKafka, 0L)
+            .handleOne(
+                new RuntimeException("database unavailable"),
+                source,
+                transientConsumer,
+                mock(MessageListenerContainer.class));
+
+    assertThat(transientRecovered).isFalse();
+    verify(transientKafka, never()).send(any(ProducerRecord.class));
+    verifyNoInteractions(transientConsumer);
+
+    KafkaTemplate<byte[], byte[]> failedKafka =
+        kafka(
+            CompletableFuture.failedFuture(
+                new UnknownTopicOrPartitionException("missing DLT partition")));
+    Consumer<byte[], byte[]> failedConsumer = mock(Consumer.class);
+    org.springframework.kafka.listener.DefaultErrorHandler handler =
+        KafkaRecoveryConfig.errorHandler(failedKafka, 0L);
+    boolean failedRecovered =
+        handler.handleOne(
+            new PermanentKafkaException("bad record"),
+            source,
+            failedConsumer,
+            mock(MessageListenerContainer.class));
+    assertThat(failedRecovered).isFalse();
+    assertThat(handler.isAckAfterHandle()).isTrue();
+    verify(failedKafka).send(any(ProducerRecord.class));
+    verifyNoInteractions(failedConsumer);
+  }
+
+  private static KafkaTemplate<byte[], byte[]> successfulKafka() {
+    SendResult<byte[], byte[]> result = mock(SendResult.class);
+    return kafka(CompletableFuture.completedFuture(result));
+  }
+
+  private static KafkaTemplate<byte[], byte[]> kafka(
+      CompletableFuture<SendResult<byte[], byte[]>> result) {
+    KafkaTemplate<byte[], byte[]> kafka = mock(KafkaTemplate.class);
+    ProducerFactory<byte[], byte[]> factory = mock(ProducerFactory.class);
+    when(kafka.getProducerFactory()).thenReturn(factory);
+    when(factory.getConfigurationProperties())
+        .thenReturn(Map.of(ProducerConfig.DELIVERY_TIMEOUT_MS_CONFIG, 5_000));
+    when(kafka.send(any(ProducerRecord.class))).thenReturn(result);
+    return kafka;
+  }
 }


## `feat(messaging): require preprovisioned topics`

diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index 67e8c3d..8dc90dc 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -28,6 +28,8 @@ spring:
       auto-offset-reset: earliest
       key-deserializer: org.apache.kafka.common.serialization.ByteArrayDeserializer
       value-deserializer: org.apache.kafka.common.serialization.ByteArrayDeserializer
+      properties:
+        allow.auto.create.topics: false
     listener:
       ack-mode: record
   data:


## `test(messaging): require preprovisioned topics`

diff --git a/src/test/java/com/sportsbook/betting/config/RuntimeConfigurationTest.java b/src/test/java/com/sportsbook/betting/config/RuntimeConfigurationTest.java
index 6c99810..dbeec21 100644
--- a/src/test/java/com/sportsbook/betting/config/RuntimeConfigurationTest.java
+++ b/src/test/java/com/sportsbook/betting/config/RuntimeConfigurationTest.java
@@ -22,6 +22,8 @@ class RuntimeConfigurationTest {
         .isEqualTo("org.apache.kafka.common.serialization.ByteArrayDeserializer");
     assertThat(value(sources, "spring.kafka.consumer.value-deserializer"))
         .isEqualTo("org.apache.kafka.common.serialization.ByteArrayDeserializer");
+    assertThat(value(sources, "spring.kafka.consumer.properties.allow.auto.create.topics"))
+        .isEqualTo(false);
     assertThat(value(sources, "spring.kafka.listener.ack-mode")).isEqualTo("record");
   }
 


