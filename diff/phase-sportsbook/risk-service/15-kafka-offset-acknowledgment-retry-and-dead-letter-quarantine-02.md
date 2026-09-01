## `test(events): route reservation reconciliation outcomes`

diff --git a/src/test/java/com/sportsbook/risk/event/BetPlacedConsumerReconciliationTest.java b/src/test/java/com/sportsbook/risk/event/BetPlacedConsumerReconciliationTest.java
new file mode 100644
index 0000000..bdd2926
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/event/BetPlacedConsumerReconciliationTest.java
@@ -0,0 +1,89 @@
+package com.sportsbook.risk.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+import static org.mockito.ArgumentMatchers.any;
+import static org.mockito.Mockito.inOrder;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.never;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+
+import java.time.Clock;
+import java.time.ZoneOffset;
+import java.util.stream.Stream;
+import org.junit.jupiter.api.BeforeEach;
+import org.junit.jupiter.api.Test;
+import org.junit.jupiter.params.ParameterizedTest;
+import org.junit.jupiter.params.provider.Arguments;
+import org.junit.jupiter.params.provider.MethodSource;
+import org.mockito.ArgumentCaptor;
+import org.mockito.InOrder;
+import org.springframework.kafka.support.Acknowledgment;
+
+class BetPlacedConsumerReconciliationTest {
+  private final AcceptedBetReconciler reconciler = mock(AcceptedBetReconciler.class);
+  private final BetPlacedDeadLetterPublisher deadLetters = mock(BetPlacedDeadLetterPublisher.class);
+  private final Acknowledgment acknowledgment = mock(Acknowledgment.class);
+  private BetPlacedConsumer consumer;
+
+  @BeforeEach
+  void setUp() {
+    Clock clock = Clock.fixed(BetPlacedEventFixture.OBSERVED_AT, ZoneOffset.UTC);
+    consumer = new BetPlacedConsumer(reconciler, deadLetters, clock);
+  }
+
+  @Test
+  void successfulReconciliationAcknowledgesTheSourceEvent() {
+    when(reconciler.reconcile(any())).thenReturn(AcceptedBetReconciliation.CONFIRMED);
+
+    consumer.onBetPlaced(
+        BetPlacedEventFixture.payload(), BetPlacedEventFixture.USER_ID, acknowledgment);
+
+    ArgumentCaptor<AcceptedBetEnvelope> envelope =
+        ArgumentCaptor.forClass(AcceptedBetEnvelope.class);
+    verify(reconciler).reconcile(envelope.capture());
+    assertThat(envelope.getValue().command().now()).isEqualTo(BetPlacedEventFixture.OBSERVED_AT);
+    verify(deadLetters, never()).publishAndAwait(any(), any(), any());
+    verify(acknowledgment).acknowledge();
+  }
+
+  @ParameterizedTest
+  @MethodSource("permanentResults")
+  void permanentReservationResultsAreDeadLetteredBeforeAcknowledgment(
+      AcceptedBetReconciliation result, BetPlacedFailureReason reason) {
+    byte[] payload = BetPlacedEventFixture.payload();
+    when(reconciler.reconcile(any())).thenReturn(result);
+
+    consumer.onBetPlaced(payload, BetPlacedEventFixture.USER_ID, acknowledgment);
+
+    InOrder order = inOrder(deadLetters, acknowledgment);
+    order.verify(deadLetters).publishAndAwait(BetPlacedEventFixture.USER_ID, payload, reason);
+    order.verify(acknowledgment).acknowledge();
+  }
+
+  @Test
+  void transientReconciliationFailureIsRetriedWithoutDeadLetterOrAcknowledgment() {
+    when(reconciler.reconcile(any())).thenThrow(new IllegalStateException("redis unavailable"));
+
+    assertThatThrownBy(
+            () ->
+                consumer.onBetPlaced(
+                    BetPlacedEventFixture.payload(), BetPlacedEventFixture.USER_ID, acknowledgment))
+        .isInstanceOf(IllegalStateException.class)
+        .hasMessage("redis unavailable");
+
+    verify(deadLetters, never()).publishAndAwait(any(), any(), any());
+    verify(acknowledgment, never()).acknowledge();
+  }
+
+  private static Stream<Arguments> permanentResults() {
+    return Stream.of(
+        Arguments.of(
+            AcceptedBetReconciliation.FINGERPRINT_MISMATCH,
+            BetPlacedFailureReason.FINGERPRINT_MISMATCH),
+        Arguments.of(
+            AcceptedBetReconciliation.TERMINAL_RESERVATION,
+            BetPlacedFailureReason.TERMINAL_RESERVATION));
+  }
+}


## `test(events): fail closed without accepted reconciliation`

diff --git a/src/test/java/com/sportsbook/risk/event/BetPlacedConsumerAvailabilityTest.java b/src/test/java/com/sportsbook/risk/event/BetPlacedConsumerAvailabilityTest.java
new file mode 100644
index 0000000..dbc3484
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/event/BetPlacedConsumerAvailabilityTest.java
@@ -0,0 +1,31 @@
+package com.sportsbook.risk.event;
+
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verifyNoInteractions;
+import static org.mockito.Mockito.when;
+
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.ObjectProvider;
+import org.springframework.kafka.support.Acknowledgment;
+
+class BetPlacedConsumerAvailabilityTest {
+  @Test
+  void missingReconcilerFailsClosedWithoutAcknowledgment() {
+    @SuppressWarnings("unchecked")
+    ObjectProvider<AcceptedBetReconciler> provider = mock(ObjectProvider.class);
+    BetPlacedDeadLetterPublisher deadLetters = mock(BetPlacedDeadLetterPublisher.class);
+    Acknowledgment acknowledgment = mock(Acknowledgment.class);
+    when(provider.getIfUnique()).thenReturn(null);
+    BetPlacedConsumer consumer = new BetPlacedConsumer(provider, deadLetters);
+
+    assertThatThrownBy(
+            () ->
+                consumer.onBetPlaced(
+                    BetPlacedEventFixture.payload(), BetPlacedEventFixture.USER_ID, acknowledgment))
+        .isInstanceOf(IllegalStateException.class)
+        .hasMessage("exactly one accepted-bet reconciler is required");
+
+    verifyNoInteractions(deadLetters, acknowledgment);
+  }
+}


## `test(events): provide embedded Kafka fixtures`

diff --git a/src/test/java/com/sportsbook/risk/event/BetPlacedKafkaIntegrationSupport.java b/src/test/java/com/sportsbook/risk/event/BetPlacedKafkaIntegrationSupport.java
new file mode 100644
index 0000000..a401207
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/event/BetPlacedKafkaIntegrationSupport.java
@@ -0,0 +1,66 @@
+package com.sportsbook.risk.event;
+
+import static org.mockito.Mockito.reset;
+
+import java.time.Duration;
+import java.util.Map;
+import java.util.UUID;
+import java.util.concurrent.TimeUnit;
+import org.apache.kafka.clients.consumer.Consumer;
+import org.apache.kafka.clients.consumer.ConsumerConfig;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
+import org.apache.kafka.common.serialization.ByteArrayDeserializer;
+import org.apache.kafka.common.serialization.StringDeserializer;
+import org.junit.jupiter.api.BeforeEach;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.boot.test.mock.mockito.MockBean;
+import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.kafka.test.EmbeddedKafkaBroker;
+import org.springframework.kafka.test.context.EmbeddedKafka;
+import org.springframework.kafka.test.utils.KafkaTestUtils;
+import org.springframework.test.annotation.DirtiesContext;
+
+@SpringBootTest(
+    properties = {
+      "risk.auth.betting-service-api-key=bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
+      "risk.auth.admin-api-key=aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
+      "risk.auth.platform-api-key=pppppppppppppppppppppppppppppppp",
+      "management.health.redis.enabled=false",
+      "management.endpoint.health.validate-group-membership=false"
+    })
+@EmbeddedKafka(
+    partitions = 1,
+    topics = {"bet.placed.v1", "bet.placed.v1.DLT"},
+    bootstrapServersProperty = "spring.kafka.bootstrap-servers")
+@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_CLASS)
+abstract class BetPlacedKafkaIntegrationSupport {
+  static final String DLT_TOPIC = "bet.placed.v1.DLT";
+
+  @Autowired private KafkaTemplate<String, byte[]> kafka;
+  @Autowired private EmbeddedKafkaBroker broker;
+  @MockBean protected AcceptedBetReconciler reconciler;
+
+  @BeforeEach
+  void resetReconciler() {
+    reset(reconciler);
+  }
+
+  void publish(String key, byte[] payload) throws Exception {
+    kafka.send("bet.placed.v1", key, payload).get(10, TimeUnit.SECONDS);
+  }
+
+  ConsumerRecord<String, byte[]> consumeDeadLetter() {
+    Map<String, Object> properties =
+        KafkaTestUtils.consumerProps("risk-dlt-test-" + UUID.randomUUID(), "false", broker);
+    properties.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
+    try (Consumer<String, byte[]> consumer =
+        new DefaultKafkaConsumerFactory<>(
+                properties, new StringDeserializer(), new ByteArrayDeserializer())
+            .createConsumer()) {
+      broker.consumeFromAnEmbeddedTopic(consumer, DLT_TOPIC);
+      return KafkaTestUtils.getSingleRecord(consumer, DLT_TOPIC, Duration.ofSeconds(10));
+    }
+  }
+}


## `test(events): verify accepted-bet broker delivery`

diff --git a/src/test/java/com/sportsbook/risk/event/BetPlacedKafkaProjectionIntegrationTest.java b/src/test/java/com/sportsbook/risk/event/BetPlacedKafkaProjectionIntegrationTest.java
new file mode 100644
index 0000000..4b5e810
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/event/BetPlacedKafkaProjectionIntegrationTest.java
@@ -0,0 +1,37 @@
+package com.sportsbook.risk.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.ArgumentMatchers.any;
+import static org.mockito.Mockito.timeout;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+
+import org.junit.jupiter.api.Test;
+import org.mockito.ArgumentCaptor;
+
+class BetPlacedKafkaProjectionIntegrationTest extends BetPlacedKafkaIntegrationSupport {
+  @Test
+  void binaryAcceptedBetReachesTheReconciler() throws Exception {
+    when(reconciler.reconcile(any())).thenReturn(AcceptedBetReconciliation.PROJECTED);
+
+    publish(BetPlacedEventFixture.USER_ID, BetPlacedEventFixture.payload());
+
+    ArgumentCaptor<AcceptedBetEnvelope> envelope =
+        ArgumentCaptor.forClass(AcceptedBetEnvelope.class);
+    verify(reconciler, timeout(10_000)).reconcile(envelope.capture());
+    assertThat(envelope.getValue().command().userId().value().toString())
+        .isEqualTo(BetPlacedEventFixture.USER_ID);
+    assertThat(envelope.getValue().command().stake().amount()).isEqualTo(10_000L);
+  }
+
+  @Test
+  void transientReconciliationFailureRedeliversTheSameEvent() throws Exception {
+    when(reconciler.reconcile(any()))
+        .thenThrow(new IllegalStateException("redis unavailable"))
+        .thenReturn(AcceptedBetReconciliation.PROJECTED);
+
+    publish(BetPlacedEventFixture.USER_ID, BetPlacedEventFixture.payload());
+
+    verify(reconciler, timeout(10_000).times(2)).reconcile(any());
+  }
+}


## `test(events): verify dead-letter broker delivery`

diff --git a/src/test/java/com/sportsbook/risk/event/BetPlacedKafkaDeadLetterIntegrationTest.java b/src/test/java/com/sportsbook/risk/event/BetPlacedKafkaDeadLetterIntegrationTest.java
new file mode 100644
index 0000000..0eaad0d
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/event/BetPlacedKafkaDeadLetterIntegrationTest.java
@@ -0,0 +1,27 @@
+package com.sportsbook.risk.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.verifyNoInteractions;
+
+import java.nio.charset.StandardCharsets;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
+import org.junit.jupiter.api.Test;
+
+class BetPlacedKafkaDeadLetterIntegrationTest extends BetPlacedKafkaIntegrationSupport {
+  @Test
+  void malformedBinaryEventIsPublishedToTheDeadLetterTopic() throws Exception {
+    byte[] malformed = {1, 2, 3};
+
+    publish(BetPlacedEventFixture.USER_ID, malformed);
+
+    ConsumerRecord<String, byte[]> deadLetter = consumeDeadLetter();
+    assertThat(deadLetter.key()).isEqualTo(BetPlacedEventFixture.USER_ID);
+    assertThat(deadLetter.value()).containsExactly(malformed);
+    assertThat(
+            new String(
+                deadLetter.headers().lastHeader("risk-dlt-reason").value(),
+                StandardCharsets.US_ASCII))
+        .isEqualTo(BetPlacedFailureReason.MALFORMED_EVENT.name());
+    verifyNoInteractions(reconciler);
+  }
+}
