## `test(integration): verify Kafka raw DLT recovery`

diff --git a/src/test/java/com/sportsbook/betting/integration/KafkaRecoveryIntegrationTest.java b/src/test/java/com/sportsbook/betting/integration/KafkaRecoveryIntegrationTest.java
new file mode 100644
index 0000000..63b2219
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/integration/KafkaRecoveryIntegrationTest.java
@@ -0,0 +1,62 @@
+package com.sportsbook.betting.integration;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.betting.support.PostgresIntegrationSupport;
+import java.time.Duration;
+import java.util.Map;
+import java.util.concurrent.TimeUnit;
+import org.apache.kafka.clients.consumer.Consumer;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
+import org.apache.kafka.clients.consumer.KafkaConsumer;
+import org.apache.kafka.clients.producer.ProducerRecord;
+import org.apache.kafka.common.serialization.ByteArrayDeserializer;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.beans.factory.annotation.Qualifier;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.kafka.test.EmbeddedKafkaBroker;
+import org.springframework.kafka.test.context.EmbeddedKafka;
+import org.springframework.kafka.test.utils.KafkaTestUtils;
+import org.springframework.test.context.ActiveProfiles;
+import org.testcontainers.junit.jupiter.Testcontainers;
+
+@Testcontainers
+@ActiveProfiles("test")
+@EmbeddedKafka(
+    partitions = 4,
+    topics = {"wallet.debited.v1", "wallet.debited.v1.DLT"},
+    bootstrapServersProperty = "spring.kafka.bootstrap-servers")
+@SpringBootTest(
+    webEnvironment = SpringBootTest.WebEnvironment.NONE,
+    properties = "spring.kafka.listener.auto-startup=true")
+class KafkaRecoveryIntegrationTest extends PostgresIntegrationSupport {
+
+  @Autowired EmbeddedKafkaBroker broker;
+
+  @Autowired
+  @Qualifier("rawDltKafkaTemplate")
+  KafkaTemplate<byte[], byte[]> kafka;
+
+  @Test
+  void recoversPermanentBytesToTheSameDltPartition() throws Exception {
+    byte[] key = {1, 2, 3};
+    byte[] value = {4, 5, 6};
+    Map<String, Object> properties = KafkaTestUtils.consumerProps("dlt-probe", "false", broker);
+    try (Consumer<byte[], byte[]> consumer =
+        new KafkaConsumer<>(properties, new ByteArrayDeserializer(), new ByteArrayDeserializer())) {
+      broker.consumeFromAnEmbeddedTopic(consumer, "wallet.debited.v1.DLT");
+      kafka
+          .send(new ProducerRecord<>("wallet.debited.v1", 2, key, value))
+          .get(10, TimeUnit.SECONDS);
+
+      ConsumerRecord<byte[], byte[]> recovered =
+          KafkaTestUtils.getSingleRecord(consumer, "wallet.debited.v1.DLT", Duration.ofSeconds(15));
+
+      assertThat(recovered.partition()).isEqualTo(2);
+      assertThat(recovered.key()).containsExactly(key);
+      assertThat(recovered.value()).containsExactly(value);
+    }
+  }
+}
