## `feat(kafka): define permanent contract failures`

diff --git a/src/main/java/com/sportsbook/gateway/kafka/GatewayEventContractException.java b/src/main/java/com/sportsbook/gateway/kafka/GatewayEventContractException.java
new file mode 100644
index 0000000..73c3332
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/kafka/GatewayEventContractException.java
@@ -0,0 +1,13 @@
+package com.sportsbook.gateway.kafka;
+
+/** A permanent event contract violation that must bypass transient delivery retries. */
+public final class GatewayEventContractException extends RuntimeException {
+
+  public GatewayEventContractException(String message) {
+    super(message);
+  }
+
+  public GatewayEventContractException(String message, Throwable cause) {
+    super(message, cause);
+  }
+}


## `feat(kafka): classify event failures`

diff --git a/src/main/java/com/sportsbook/gateway/kafka/GatewayKafkaErrorConfiguration.java b/src/main/java/com/sportsbook/gateway/kafka/GatewayKafkaErrorConfiguration.java
new file mode 100644
index 0000000..00fb66f
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/kafka/GatewayKafkaErrorConfiguration.java
@@ -0,0 +1,25 @@
+package com.sportsbook.gateway.kafka;
+
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.kafka.listener.DeadLetterPublishingRecoverer;
+import org.springframework.kafka.listener.DefaultErrorHandler;
+import org.springframework.util.backoff.FixedBackOff;
+
+/** Classifies permanent contract failures separately from transient delivery failures. */
+@Configuration(proxyBeanMethods = false)
+public class GatewayKafkaErrorConfiguration {
+
+  @Bean
+  DefaultErrorHandler gatewayKafkaErrorHandler(
+      DeadLetterPublishingRecoverer recoverer, GatewayKafkaProperties properties) {
+    DefaultErrorHandler handler =
+        new DefaultErrorHandler(
+            recoverer,
+            new FixedBackOff(properties.retryInterval().toMillis(), properties.retryAttempts()));
+    handler.addNotRetryableExceptions(GatewayEventContractException.class);
+    handler.setAckAfterHandle(true);
+    handler.setResetStateOnRecoveryFailure(true);
+    return handler;
+  }
+}
diff --git a/src/main/java/com/sportsbook/gateway/kafka/GatewayKafkaProperties.java b/src/main/java/com/sportsbook/gateway/kafka/GatewayKafkaProperties.java
index 7cf8a37..bf297ba 100644
--- a/src/main/java/com/sportsbook/gateway/kafka/GatewayKafkaProperties.java
+++ b/src/main/java/com/sportsbook/gateway/kafka/GatewayKafkaProperties.java
@@ -7,10 +7,16 @@ import org.springframework.boot.context.properties.bind.DefaultValue;
 /** Bounded publication settings for failed raw event records. */
 @ConfigurationProperties(prefix = "gateway.kafka")
 public record GatewayKafkaProperties(
+    @DefaultValue("1s") Duration retryInterval,
+    @DefaultValue("2") long retryAttempts,
     @DefaultValue("11s") Duration dltWaitTimeout,
     @DefaultValue("1s") Duration dltTimeoutBuffer) {
 
   public GatewayKafkaProperties {
+    requirePositive(retryInterval, "retry-interval");
+    if (retryAttempts < 0) {
+      throw new IllegalArgumentException("gateway.kafka.retry-attempts must not be negative");
+    }
     requirePositive(dltWaitTimeout, "dlt-wait-timeout");
     requirePositive(dltTimeoutBuffer, "dlt-timeout-buffer");
   }


## `test(kafka): bypass retries for contract failures`

diff --git a/src/test/java/com/sportsbook/gateway/kafka/GatewayKafkaErrorConfigurationTest.java b/src/test/java/com/sportsbook/gateway/kafka/GatewayKafkaErrorConfigurationTest.java
new file mode 100644
index 0000000..df07283
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/kafka/GatewayKafkaErrorConfigurationTest.java
@@ -0,0 +1,25 @@
+package com.sportsbook.gateway.kafka;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.mock;
+
+import java.time.Duration;
+import org.junit.jupiter.api.Test;
+import org.springframework.kafka.listener.DeadLetterPublishingRecoverer;
+import org.springframework.kafka.listener.DefaultErrorHandler;
+
+class GatewayKafkaErrorConfigurationTest {
+
+  @Test
+  void sendsPermanentContractFailuresDirectlyToRecovery() {
+    GatewayKafkaProperties properties =
+        new GatewayKafkaProperties(
+            Duration.ofSeconds(1), 2, Duration.ofSeconds(11), Duration.ofSeconds(1));
+    DefaultErrorHandler handler =
+        new GatewayKafkaErrorConfiguration()
+            .gatewayKafkaErrorHandler(mock(DeadLetterPublishingRecoverer.class), properties);
+
+    assertThat(handler.removeClassification(GatewayEventContractException.class)).isFalse();
+    assertThat(handler.isAckAfterHandle()).isTrue();
+  }
+}


## `test(kafka): retry transient delivery failures`

diff --git a/src/test/java/com/sportsbook/gateway/kafka/GatewayTransientRetryIntegrationTest.java b/src/test/java/com/sportsbook/gateway/kafka/GatewayTransientRetryIntegrationTest.java
new file mode 100644
index 0000000..e373bfe
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/kafka/GatewayTransientRetryIntegrationTest.java
@@ -0,0 +1,97 @@
+package com.sportsbook.gateway.kafka;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.nio.charset.StandardCharsets;
+import java.time.Duration;
+import java.util.List;
+import java.util.concurrent.CopyOnWriteArrayList;
+import java.util.concurrent.CountDownLatch;
+import java.util.concurrent.TimeUnit;
+import org.junit.jupiter.api.BeforeEach;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.boot.test.context.TestConfiguration;
+import org.springframework.context.annotation.Bean;
+import org.springframework.kafka.annotation.KafkaListener;
+import org.springframework.kafka.config.KafkaListenerEndpointRegistry;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.kafka.test.context.EmbeddedKafka;
+import org.springframework.kafka.test.utils.ContainerTestUtils;
+
+@SpringBootTest(
+    properties = {
+      "spring.main.web-application-type=none",
+      "management.tracing.enabled=false",
+      "spring.kafka.consumer.auto-offset-reset=earliest",
+      "logging.level.kafka=ERROR",
+      "logging.level.org.apache.kafka=ERROR",
+      "logging.level.org.springframework.kafka=ERROR"
+    })
+@EmbeddedKafka(
+    partitions = 1,
+    topics = "gateway.retry.test",
+    bootstrapServersProperty = "spring.kafka.bootstrap-servers")
+class GatewayTransientRetryIntegrationTest {
+
+  @Autowired private KafkaTemplate<byte[], byte[]> kafka;
+  @Autowired private KafkaListenerEndpointRegistry listeners;
+  @Autowired private FailureProbe probe;
+
+  @BeforeEach
+  void awaitProbe() {
+    probe.reset();
+    ContainerTestUtils.waitForAssignment(
+        listeners.getListenerContainer("transient-retry-probe"), 1);
+  }
+
+  @Test
+  void retriesTwiceAtOneSecondBeforeTheThirdAttemptSucceeds() throws Exception {
+    kafka
+        .send("gateway.retry.test", "key".getBytes(StandardCharsets.UTF_8), new byte[] {1, 2, 3})
+        .get(5, TimeUnit.SECONDS);
+
+    assertThat(probe.completed.await(10, TimeUnit.SECONDS)).isTrue();
+    assertThat(probe.attempts).hasSize(3);
+    assertThat(elapsed(probe.attempts, 0)).isGreaterThanOrEqualTo(Duration.ofMillis(900));
+    assertThat(elapsed(probe.attempts, 1)).isGreaterThanOrEqualTo(Duration.ofMillis(900));
+  }
+
+  private static Duration elapsed(List<Long> attempts, int first) {
+    return Duration.ofNanos(attempts.get(first + 1) - attempts.get(first));
+  }
+
+  @TestConfiguration
+  static class ProbeConfiguration {
+
+    @Bean
+    FailureProbe failureProbe() {
+      return new FailureProbe();
+    }
+  }
+
+  static final class FailureProbe {
+
+    private final List<Long> attempts = new CopyOnWriteArrayList<>();
+    private CountDownLatch completed = new CountDownLatch(1);
+
+    @KafkaListener(
+        id = "transient-retry-probe",
+        topics = "gateway.retry.test",
+        groupId = "gateway-retry-test",
+        autoStartup = "true")
+    void receive(byte[] ignored) {
+      attempts.add(System.nanoTime());
+      if (attempts.size() < 3) {
+        throw new IllegalStateException("temporary delivery failure");
+      }
+      completed.countDown();
+    }
+
+    void reset() {
+      attempts.clear();
+      completed = new CountDownLatch(1);
+    }
+  }
+}


## `test(kafka): retain offsets when recovery fails`

diff --git a/src/test/java/com/sportsbook/gateway/kafka/GatewayRecoveryFailureIntegrationTest.java b/src/test/java/com/sportsbook/gateway/kafka/GatewayRecoveryFailureIntegrationTest.java
new file mode 100644
index 0000000..bcbf5ae
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/kafka/GatewayRecoveryFailureIntegrationTest.java
@@ -0,0 +1,99 @@
+package com.sportsbook.gateway.kafka;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.nio.charset.StandardCharsets;
+import java.util.List;
+import java.util.Map;
+import java.util.concurrent.CopyOnWriteArrayList;
+import java.util.concurrent.CountDownLatch;
+import java.util.concurrent.TimeUnit;
+import org.apache.kafka.clients.admin.Admin;
+import org.apache.kafka.clients.admin.AdminClientConfig;
+import org.apache.kafka.clients.admin.NewTopic;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
+import org.apache.kafka.common.TopicPartition;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.context.annotation.Import;
+import org.springframework.kafka.annotation.KafkaListener;
+import org.springframework.kafka.config.KafkaListenerEndpointRegistry;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.kafka.test.EmbeddedKafkaBroker;
+import org.springframework.kafka.test.context.EmbeddedKafka;
+import org.springframework.kafka.test.utils.ContainerTestUtils;
+
+@SpringBootTest(
+    properties = {
+      "spring.main.web-application-type=none",
+      "gateway.topics.odds-changed=gateway.recovery.test",
+      "logging.level.kafka=ERROR"
+    })
+@EmbeddedKafka(
+    partitions = 2,
+    topics = "gateway.recovery.test",
+    brokerProperties = "auto.create.topics.enable=false",
+    bootstrapServersProperty = "spring.kafka.bootstrap-servers")
+@Import(GatewayRecoveryFailureIntegrationTest.RecoveryProbe.class)
+class GatewayRecoveryFailureIntegrationTest {
+
+  @Autowired private KafkaTemplate<byte[], byte[]> kafka;
+  @Autowired private KafkaListenerEndpointRegistry listeners;
+  @Autowired private EmbeddedKafkaBroker broker;
+  @Autowired private RecoveryProbe probe;
+
+  @Test
+  void keepsTheFailedOffsetWhenTheDltLacksTheSourcePartition() throws Exception {
+    broker.addTopics(new NewTopic("gateway.recovery.test.DLT", 1, (short) 1));
+    probe.reset();
+    ContainerTestUtils.waitForAssignment(listeners.getListenerContainer("recovery-probe"), 2);
+    send("first");
+    assertThat(probe.redelivered.await(35, TimeUnit.SECONDS)).isTrue();
+    assertThat(probe.keys).hasSizeGreaterThanOrEqualTo(4).allMatch("first"::equals);
+    listeners.getListenerContainer("recovery-probe").stop();
+    assertFailedOffsetRemainsNext();
+  }
+
+  private void send(String key) throws Exception {
+    kafka
+        .send("gateway.recovery.test", 1, key.getBytes(StandardCharsets.UTF_8), new byte[] {1})
+        .get(5, TimeUnit.SECONDS);
+  }
+
+  private void assertFailedOffsetRemainsNext() throws Exception {
+    try (Admin admin =
+        Admin.create(
+            Map.of(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, broker.getBrokersAsString()))) {
+      var committed =
+          admin
+              .listConsumerGroupOffsets("gateway-recovery-test")
+              .partitionsToOffsetAndMetadata()
+              .get(5, TimeUnit.SECONDS);
+      assertThat(committed.get(new TopicPartition("gateway.recovery.test", 1)).offset()).isZero();
+    }
+  }
+
+  static final class RecoveryProbe {
+    private final List<String> keys = new CopyOnWriteArrayList<>();
+    private CountDownLatch redelivered = new CountDownLatch(1);
+
+    @KafkaListener(
+        id = "recovery-probe",
+        topics = "gateway.recovery.test",
+        groupId = "gateway-recovery-test",
+        autoStartup = "true")
+    void receive(ConsumerRecord<byte[], byte[]> record) {
+      keys.add(new String(record.key(), StandardCharsets.UTF_8));
+      if (keys.size() == 4) {
+        redelivered.countDown();
+      }
+      throw new IllegalStateException("delivery remains unavailable");
+    }
+
+    void reset() {
+      keys.clear();
+      redelivered = new CountDownLatch(1);
+    }
+  }
+}


## `test(kafka): resume partition after recovery succeeds`

diff --git a/src/test/java/com/sportsbook/gateway/kafka/GatewayRecoveryFailureIntegrationTest.java b/src/test/java/com/sportsbook/gateway/kafka/GatewayRecoveryFailureIntegrationTest.java
index bcbf5ae..53b9160 100644
--- a/src/test/java/com/sportsbook/gateway/kafka/GatewayRecoveryFailureIntegrationTest.java
+++ b/src/test/java/com/sportsbook/gateway/kafka/GatewayRecoveryFailureIntegrationTest.java
@@ -3,26 +3,37 @@ package com.sportsbook.gateway.kafka;
 import static org.assertj.core.api.Assertions.assertThat;
 
 import java.nio.charset.StandardCharsets;
+import java.time.Duration;
 import java.util.List;
 import java.util.Map;
+import java.util.UUID;
 import java.util.concurrent.CopyOnWriteArrayList;
 import java.util.concurrent.CountDownLatch;
 import java.util.concurrent.TimeUnit;
 import org.apache.kafka.clients.admin.Admin;
 import org.apache.kafka.clients.admin.AdminClientConfig;
+import org.apache.kafka.clients.admin.NewPartitions;
 import org.apache.kafka.clients.admin.NewTopic;
+import org.apache.kafka.clients.consumer.Consumer;
 import org.apache.kafka.clients.consumer.ConsumerRecord;
 import org.apache.kafka.common.TopicPartition;
+import org.apache.kafka.common.serialization.ByteArrayDeserializer;
+import org.awaitility.Awaitility;
+import org.junit.jupiter.api.MethodOrderer.OrderAnnotation;
+import org.junit.jupiter.api.Order;
 import org.junit.jupiter.api.Test;
+import org.junit.jupiter.api.TestMethodOrder;
 import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.boot.test.context.SpringBootTest;
 import org.springframework.context.annotation.Import;
 import org.springframework.kafka.annotation.KafkaListener;
 import org.springframework.kafka.config.KafkaListenerEndpointRegistry;
+import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
 import org.springframework.kafka.core.KafkaTemplate;
 import org.springframework.kafka.test.EmbeddedKafkaBroker;
 import org.springframework.kafka.test.context.EmbeddedKafka;
 import org.springframework.kafka.test.utils.ContainerTestUtils;
+import org.springframework.kafka.test.utils.KafkaTestUtils;
 
 @SpringBootTest(
     properties = {
@@ -36,6 +47,7 @@ import org.springframework.kafka.test.utils.ContainerTestUtils;
     brokerProperties = "auto.create.topics.enable=false",
     bootstrapServersProperty = "spring.kafka.bootstrap-servers")
 @Import(GatewayRecoveryFailureIntegrationTest.RecoveryProbe.class)
+@TestMethodOrder(OrderAnnotation.class)
 class GatewayRecoveryFailureIntegrationTest {
 
   @Autowired private KafkaTemplate<byte[], byte[]> kafka;
@@ -44,15 +56,44 @@ class GatewayRecoveryFailureIntegrationTest {
   @Autowired private RecoveryProbe probe;
 
   @Test
+  @Order(1)
   void keepsTheFailedOffsetWhenTheDltLacksTheSourcePartition() throws Exception {
     broker.addTopics(new NewTopic("gateway.recovery.test.DLT", 1, (short) 1));
-    probe.reset();
-    ContainerTestUtils.waitForAssignment(listeners.getListenerContainer("recovery-probe"), 2);
+    startProbe();
     send("first");
     assertThat(probe.redelivered.await(35, TimeUnit.SECONDS)).isTrue();
     assertThat(probe.keys).hasSizeGreaterThanOrEqualTo(4).allMatch("first"::equals);
     listeners.getListenerContainer("recovery-probe").stop();
-    assertFailedOffsetRemainsNext();
+    assertThat(committedOffset()).isZero();
+  }
+
+  @Test
+  @Order(2)
+  void resumesThePartitionAfterDeadLetterRecoverySucceeds() throws Exception {
+    addDeadLetterPartition();
+    Map<String, Object> consumerProperties =
+        KafkaTestUtils.consumerProps("recovered-" + UUID.randomUUID(), "false", broker);
+    try (Consumer<byte[], byte[]> deadLetters =
+        new DefaultKafkaConsumerFactory<>(
+                consumerProperties, new ByteArrayDeserializer(), new ByteArrayDeserializer())
+            .createConsumer()) {
+      deadLetters.subscribe(List.of("gateway.recovery.test.DLT"));
+      startProbe();
+      send("second");
+
+      ConsumerRecord<byte[], byte[]> failed =
+          KafkaTestUtils.getSingleRecord(
+              deadLetters, "gateway.recovery.test.DLT", Duration.ofSeconds(15));
+      assertThat(failed.key()).containsExactly("first".getBytes(StandardCharsets.UTF_8));
+      assertThat(failed.value()).containsExactly(1);
+      assertThat(failed.partition()).isEqualTo(1);
+      assertThat(probe.secondDelivered.await(10, TimeUnit.SECONDS)).isTrue();
+      Awaitility.await()
+          .atMost(Duration.ofSeconds(10))
+          .untilAsserted(() -> assertThat(committedOffset()).isGreaterThanOrEqualTo(2));
+    } finally {
+      listeners.getListenerContainer("recovery-probe").stop();
+    }
   }
 
   private void send(String key) throws Exception {
@@ -61,7 +102,24 @@ class GatewayRecoveryFailureIntegrationTest {
         .get(5, TimeUnit.SECONDS);
   }
 
-  private void assertFailedOffsetRemainsNext() throws Exception {
+  private void startProbe() {
+    probe.reset();
+    listeners.getListenerContainer("recovery-probe").start();
+    ContainerTestUtils.waitForAssignment(listeners.getListenerContainer("recovery-probe"), 2);
+  }
+
+  private void addDeadLetterPartition() throws Exception {
+    try (Admin admin =
+        Admin.create(
+            Map.of(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, broker.getBrokersAsString()))) {
+      admin
+          .createPartitions(Map.of("gateway.recovery.test.DLT", NewPartitions.increaseTo(2)))
+          .all()
+          .get(5, TimeUnit.SECONDS);
+    }
+  }
+
+  private long committedOffset() throws Exception {
     try (Admin admin =
         Admin.create(
             Map.of(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, broker.getBrokersAsString()))) {
@@ -70,13 +128,14 @@ class GatewayRecoveryFailureIntegrationTest {
               .listConsumerGroupOffsets("gateway-recovery-test")
               .partitionsToOffsetAndMetadata()
               .get(5, TimeUnit.SECONDS);
-      assertThat(committed.get(new TopicPartition("gateway.recovery.test", 1)).offset()).isZero();
+      return committed.get(new TopicPartition("gateway.recovery.test", 1)).offset();
     }
   }
 
   static final class RecoveryProbe {
     private final List<String> keys = new CopyOnWriteArrayList<>();
     private CountDownLatch redelivered = new CountDownLatch(1);
+    private CountDownLatch secondDelivered = new CountDownLatch(1);
 
     @KafkaListener(
         id = "recovery-probe",
@@ -84,8 +143,13 @@ class GatewayRecoveryFailureIntegrationTest {
         groupId = "gateway-recovery-test",
         autoStartup = "true")
     void receive(ConsumerRecord<byte[], byte[]> record) {
-      keys.add(new String(record.key(), StandardCharsets.UTF_8));
-      if (keys.size() == 4) {
+      String key = new String(record.key(), StandardCharsets.UTF_8);
+      keys.add(key);
+      if ("second".equals(key)) {
+        secondDelivered.countDown();
+        return;
+      }
+      if (keys.stream().filter("first"::equals).count() == 4) {
         redelivered.countDown();
       }
       throw new IllegalStateException("delivery remains unavailable");
@@ -94,6 +158,7 @@ class GatewayRecoveryFailureIntegrationTest {
     void reset() {
       keys.clear();
       redelivered = new CountDownLatch(1);
+      secondDelivered = new CountDownLatch(1);
     }
   }
 }
