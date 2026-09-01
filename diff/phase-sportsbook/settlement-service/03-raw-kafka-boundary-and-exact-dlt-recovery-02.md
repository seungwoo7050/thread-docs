## `fix(kafka): isolate raw producer factory`

diff --git a/src/main/java/com/sportsbook/settlement/config/RawKafkaProducerConfiguration.java b/src/main/java/com/sportsbook/settlement/config/RawKafkaProducerConfiguration.java
index 6a35dc0..411884a 100644
--- a/src/main/java/com/sportsbook/settlement/config/RawKafkaProducerConfiguration.java
+++ b/src/main/java/com/sportsbook/settlement/config/RawKafkaProducerConfiguration.java
@@ -4,31 +4,22 @@ import java.util.HashMap;
 import java.util.Map;
 import org.apache.kafka.clients.producer.ProducerConfig;
 import org.apache.kafka.common.serialization.ByteArraySerializer;
-import org.springframework.beans.factory.annotation.Qualifier;
 import org.springframework.boot.autoconfigure.kafka.KafkaProperties;
 import org.springframework.context.annotation.Bean;
 import org.springframework.context.annotation.Configuration;
 import org.springframework.kafka.core.DefaultKafkaProducerFactory;
 import org.springframework.kafka.core.KafkaOperations;
 import org.springframework.kafka.core.KafkaTemplate;
-import org.springframework.kafka.core.ProducerFactory;
 
 /** Dedicated bounded raw-byte producer for exact dead-letter publication. */
 @Configuration
 public class RawKafkaProducerConfiguration {
 
-  public static final String PRODUCER_FACTORY = "settlementRawProducerFactory";
   public static final String OPERATIONS = "settlementRawKafkaOperations";
 
-  @Bean(PRODUCER_FACTORY)
-  ProducerFactory<byte[], byte[]> settlementRawProducerFactory(KafkaProperties properties) {
-    return new DefaultKafkaProducerFactory<>(producerProperties(properties));
-  }
-
   @Bean(OPERATIONS)
-  KafkaOperations<byte[], byte[]> settlementRawKafkaOperations(
-      @Qualifier(PRODUCER_FACTORY) ProducerFactory<byte[], byte[]> producerFactory) {
-    return new KafkaTemplate<>(producerFactory);
+  KafkaOperations<byte[], byte[]> settlementRawKafkaOperations(KafkaProperties properties) {
+    return new KafkaTemplate<>(new DefaultKafkaProducerFactory<>(producerProperties(properties)));
   }
 
   static Map<String, Object> producerProperties(KafkaProperties properties) {


## `test(kafka): verify exact raw DLT recovery`

diff --git a/src/test/java/com/sportsbook/settlement/event/ExactDeadLetterRecovererTest.java b/src/test/java/com/sportsbook/settlement/event/ExactDeadLetterRecovererTest.java
new file mode 100644
index 0000000..c30b17e
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/event/ExactDeadLetterRecovererTest.java
@@ -0,0 +1,91 @@
+package com.sportsbook.settlement.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+import static org.mockito.ArgumentMatchers.any;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+
+import java.nio.ByteBuffer;
+import java.nio.charset.StandardCharsets;
+import java.time.Duration;
+import java.util.concurrent.CompletableFuture;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
+import org.apache.kafka.clients.producer.ProducerRecord;
+import org.junit.jupiter.api.Test;
+import org.mockito.ArgumentCaptor;
+import org.springframework.kafka.KafkaException;
+import org.springframework.kafka.core.KafkaOperations;
+
+class ExactDeadLetterRecovererTest {
+
+  @SuppressWarnings("unchecked")
+  private final KafkaOperations<byte[], byte[]> kafka = mock(KafkaOperations.class);
+
+  @Test
+  void preservesPartitionRawBytesAndApplicationHeaders() {
+    when(kafka.send(any(ProducerRecord.class))).thenReturn(CompletableFuture.completedFuture(null));
+    byte[] key = "key".getBytes(StandardCharsets.UTF_8);
+    byte[] value = {1, 2, 3};
+    ConsumerRecord<byte[], byte[]> source =
+        new ConsumerRecord<>("bet.placed.v1", 2, 19, key, value);
+    source.headers().add("traceparent", "trace".getBytes(StandardCharsets.UTF_8));
+
+    new ExactDeadLetterRecoverer(kafka, "settlement-placement", Duration.ofSeconds(11))
+        .recover(source, new IllegalArgumentException("poison"));
+
+    ArgumentCaptor<ProducerRecord<byte[], byte[]>> sent =
+        ArgumentCaptor.forClass(ProducerRecord.class);
+    verify(kafka).send(sent.capture());
+    ProducerRecord<byte[], byte[]> record = sent.getValue();
+    assertThat(record.topic()).isEqualTo("bet.placed.v1.DLT");
+    assertThat(record.partition()).isEqualTo(2);
+    assertThat(record.key()).isSameAs(key);
+    assertThat(record.value()).isSameAs(value);
+    assertThat(text(record, "traceparent")).isEqualTo("trace");
+    assertThat(text(record, ExactDeadLetterRecoverer.ORIGINAL_TOPIC)).isEqualTo("bet.placed.v1");
+    assertThat(number(record, ExactDeadLetterRecoverer.ORIGINAL_OFFSET)).isEqualTo(19L);
+  }
+
+  @Test
+  void propagatesAnExactPartitionSendFailure() {
+    CompletableFuture<org.springframework.kafka.support.SendResult<byte[], byte[]>> failed =
+        new CompletableFuture<>();
+    failed.completeExceptionally(new IllegalStateException("missing DLT partition"));
+    when(kafka.send(any(ProducerRecord.class))).thenReturn(failed);
+    ConsumerRecord<byte[], byte[]> source =
+        new ConsumerRecord<>("match.result", 4, 1, new byte[0], new byte[0]);
+
+    assertThatThrownBy(
+            () ->
+                new ExactDeadLetterRecoverer(kafka, "settlement-result", Duration.ofSeconds(1))
+                    .recover(source, new IllegalArgumentException("poison")))
+        .isInstanceOf(KafkaException.class)
+        .hasMessageContaining("Failed to publish exact DLT");
+  }
+
+  @Test
+  void republishesNullPayloadsToTheSameDeadLetterPartition() {
+    when(kafka.send(any(ProducerRecord.class))).thenReturn(CompletableFuture.completedFuture(null));
+    ConsumerRecord<byte[], byte[]> tombstone =
+        new ConsumerRecord<>("match.result.v1", 3, 7, new byte[0], null);
+
+    new ExactDeadLetterRecoverer(kafka, "settlement-result", Duration.ofSeconds(1))
+        .recover(tombstone, new StrictAvroDecoder.DecodeException("null payload"));
+
+    ArgumentCaptor<ProducerRecord<byte[], byte[]>> sent =
+        ArgumentCaptor.forClass(ProducerRecord.class);
+    verify(kafka).send(sent.capture());
+    assertThat(sent.getValue().partition()).isEqualTo(3);
+    assertThat(sent.getValue().value()).isNull();
+  }
+
+  private static String text(ProducerRecord<byte[], byte[]> record, String name) {
+    return new String(record.headers().lastHeader(name).value(), StandardCharsets.UTF_8);
+  }
+
+  private static long number(ProducerRecord<byte[], byte[]> record, String name) {
+    return ByteBuffer.wrap(record.headers().lastHeader(name).value()).getLong();
+  }
+}
