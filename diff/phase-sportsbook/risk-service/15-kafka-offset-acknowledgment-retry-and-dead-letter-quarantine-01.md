# Kafka 오프셋 승인, 재시도, 데드레터 격리

## `feat(events): define accepted bet reconciliation outcomes`

diff --git a/src/main/java/com/sportsbook/risk/event/AcceptedBetReconciler.java b/src/main/java/com/sportsbook/risk/event/AcceptedBetReconciler.java
new file mode 100644
index 0000000..b058b25
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/event/AcceptedBetReconciler.java
@@ -0,0 +1,7 @@
+package com.sportsbook.risk.event;
+
+/** Atomic boundary for reservation confirmation or first-seen accepted-bet projection. */
+@FunctionalInterface
+public interface AcceptedBetReconciler {
+  AcceptedBetReconciliation reconcile(AcceptedBetEnvelope envelope);
+}
diff --git a/src/main/java/com/sportsbook/risk/event/AcceptedBetReconciliation.java b/src/main/java/com/sportsbook/risk/event/AcceptedBetReconciliation.java
new file mode 100644
index 0000000..9106969
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/event/AcceptedBetReconciliation.java
@@ -0,0 +1,27 @@
+package com.sportsbook.risk.event;
+
+/** Atomic result of reconciling an accepted event with reservation and committed state. */
+public enum AcceptedBetReconciliation {
+  CONFIRMED(null),
+  PROJECTED(null),
+  REPLAYED(null),
+  FINGERPRINT_MISMATCH(BetPlacedFailureReason.FINGERPRINT_MISMATCH),
+  TERMINAL_RESERVATION(BetPlacedFailureReason.TERMINAL_RESERVATION);
+
+  private final BetPlacedFailureReason failureReason;
+
+  AcceptedBetReconciliation(BetPlacedFailureReason failureReason) {
+    this.failureReason = failureReason;
+  }
+
+  public boolean permanentFailure() {
+    return failureReason != null;
+  }
+
+  public BetPlacedFailureReason failureReason() {
+    if (failureReason == null) {
+      throw new IllegalStateException("successful reconciliation has no failure reason");
+    }
+    return failureReason;
+  }
+}


## `test(events): verify reconciliation outcome semantics`

diff --git a/src/test/java/com/sportsbook/risk/event/AcceptedBetReconciliationTest.java b/src/test/java/com/sportsbook/risk/event/AcceptedBetReconciliationTest.java
new file mode 100644
index 0000000..b122182
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/event/AcceptedBetReconciliationTest.java
@@ -0,0 +1,21 @@
+package com.sportsbook.risk.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import org.junit.jupiter.api.Test;
+
+class AcceptedBetReconciliationTest {
+  @Test
+  void exposesOnlyPermanentReservationFailuresAsDeadLetterReasons() {
+    assertThat(AcceptedBetReconciliation.CONFIRMED.permanentFailure()).isFalse();
+    assertThat(AcceptedBetReconciliation.PROJECTED.permanentFailure()).isFalse();
+    assertThat(AcceptedBetReconciliation.REPLAYED.permanentFailure()).isFalse();
+    assertThat(AcceptedBetReconciliation.FINGERPRINT_MISMATCH.failureReason())
+        .isEqualTo(BetPlacedFailureReason.FINGERPRINT_MISMATCH);
+    assertThat(AcceptedBetReconciliation.TERMINAL_RESERVATION.failureReason())
+        .isEqualTo(BetPlacedFailureReason.TERMINAL_RESERVATION);
+    assertThatThrownBy(AcceptedBetReconciliation.CONFIRMED::failureReason)
+        .isInstanceOf(IllegalStateException.class);
+  }
+}


## `feat(events): publish accepted bet dead letters`

diff --git a/src/main/java/com/sportsbook/risk/event/BetPlacedDeadLetterPublisher.java b/src/main/java/com/sportsbook/risk/event/BetPlacedDeadLetterPublisher.java
new file mode 100644
index 0000000..3028f22
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/event/BetPlacedDeadLetterPublisher.java
@@ -0,0 +1,47 @@
+package com.sportsbook.risk.event;
+
+import io.micrometer.core.instrument.MeterRegistry;
+import java.nio.charset.StandardCharsets;
+import java.time.Duration;
+import java.util.Objects;
+import java.util.concurrent.TimeUnit;
+import org.apache.kafka.clients.producer.ProducerRecord;
+import org.apache.kafka.common.header.internals.RecordHeader;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.stereotype.Component;
+
+/** Publishes permanent input failures and waits for the DLT broker acknowledgment. */
+@Component
+public class BetPlacedDeadLetterPublisher {
+  private static final Duration PUBLISH_TIMEOUT = Duration.ofSeconds(10);
+
+  private final KafkaTemplate<String, byte[]> kafka;
+  private final EventTopics topics;
+  private final MeterRegistry meters;
+
+  public BetPlacedDeadLetterPublisher(
+      KafkaTemplate<String, byte[]> kafka, EventTopics topics, MeterRegistry meters) {
+    this.kafka = Objects.requireNonNull(kafka, "kafka");
+    this.topics = Objects.requireNonNull(topics, "topics");
+    this.meters = Objects.requireNonNull(meters, "meters");
+  }
+
+  public void publishAndAwait(String key, byte[] payload, BetPlacedFailureReason reason) {
+    Objects.requireNonNull(reason, "reason");
+    ProducerRecord<String, byte[]> record =
+        new ProducerRecord<>(topics.betPlacedDlt(), key, payload);
+    record
+        .headers()
+        .add(
+            new RecordHeader("risk-dlt-reason", reason.name().getBytes(StandardCharsets.US_ASCII)));
+    try {
+      kafka.send(record).get(PUBLISH_TIMEOUT.toMillis(), TimeUnit.MILLISECONDS);
+      meters.counter("risk_bet_placed_dlt_total", "reason", reason.name()).increment();
+    } catch (InterruptedException failure) {
+      Thread.currentThread().interrupt();
+      throw new IllegalStateException("interrupted while publishing bet.placed DLT", failure);
+    } catch (Exception failure) {
+      throw new IllegalStateException("failed to publish bet.placed DLT", failure);
+    }
+  }
+}


## `test(events): verify dead letter broker confirmation`

diff --git a/src/test/java/com/sportsbook/risk/event/BetPlacedDeadLetterPublisherTest.java b/src/test/java/com/sportsbook/risk/event/BetPlacedDeadLetterPublisherTest.java
new file mode 100644
index 0000000..37e6892
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/event/BetPlacedDeadLetterPublisherTest.java
@@ -0,0 +1,74 @@
+package com.sportsbook.risk.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+import static org.mockito.ArgumentMatchers.any;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+
+import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
+import java.nio.charset.StandardCharsets;
+import java.util.concurrent.CompletableFuture;
+import java.util.concurrent.ExecutionException;
+import java.util.concurrent.TimeUnit;
+import org.apache.kafka.clients.producer.ProducerRecord;
+import org.junit.jupiter.api.BeforeEach;
+import org.junit.jupiter.api.Test;
+import org.mockito.ArgumentCaptor;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.kafka.support.SendResult;
+
+class BetPlacedDeadLetterPublisherTest {
+  private final KafkaTemplate<String, byte[]> kafka = mock();
+  private final SimpleMeterRegistry meters = new SimpleMeterRegistry();
+  private BetPlacedDeadLetterPublisher publisher;
+
+  @BeforeEach
+  void setUp() {
+    publisher =
+        new BetPlacedDeadLetterPublisher(kafka, new EventTopics(null, null, null, null), meters);
+  }
+
+  @Test
+  void waitsForBrokerAcknowledgmentAndPreservesTheRejectedRecord() throws Exception {
+    CompletableFuture<SendResult<String, byte[]>> future = mock();
+    when(kafka.send(any(ProducerRecord.class))).thenReturn(future);
+    byte[] payload = {4, 2};
+
+    publisher.publishAndAwait("user-key", payload, BetPlacedFailureReason.KEY_MISMATCH);
+
+    verify(future).get(10_000L, TimeUnit.MILLISECONDS);
+    ArgumentCaptor<ProducerRecord<String, byte[]>> record = ArgumentCaptor.captor();
+    verify(kafka).send(record.capture());
+    assertThat(record.getValue().topic()).isEqualTo("bet.placed.v1.DLT");
+    assertThat(record.getValue().key()).isEqualTo("user-key");
+    assertThat(record.getValue().value()).isSameAs(payload);
+    assertThat(reason(record.getValue())).isEqualTo("KEY_MISMATCH");
+    assertThat(meters.counter("risk_bet_placed_dlt_total", "reason", "KEY_MISMATCH").count())
+        .isEqualTo(1.0);
+  }
+
+  @Test
+  void brokerFailurePropagatesWithoutRecordingACompletedPublication() throws Exception {
+    CompletableFuture<SendResult<String, byte[]>> future = mock();
+    when(kafka.send(any(ProducerRecord.class))).thenReturn(future);
+    when(future.get(10_000L, TimeUnit.MILLISECONDS))
+        .thenThrow(new ExecutionException(new IllegalStateException("broker unavailable")));
+
+    assertThatThrownBy(
+            () ->
+                publisher.publishAndAwait(
+                    "user-key", new byte[] {1}, BetPlacedFailureReason.MALFORMED_EVENT))
+        .isInstanceOf(IllegalStateException.class)
+        .hasMessage("failed to publish bet.placed DLT")
+        .hasRootCauseMessage("broker unavailable");
+    assertThat(meters.counter("risk_bet_placed_dlt_total", "reason", "MALFORMED_EVENT").count())
+        .isZero();
+  }
+
+  private static String reason(ProducerRecord<String, byte[]> record) {
+    return new String(
+        record.headers().lastHeader("risk-dlt-reason").value(), StandardCharsets.US_ASCII);
+  }
+}


## `feat(events): retain transient accepted bet failures`

diff --git a/src/main/java/com/sportsbook/risk/event/KafkaConsumerConfiguration.java b/src/main/java/com/sportsbook/risk/event/KafkaConsumerConfiguration.java
new file mode 100644
index 0000000..55b6934
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/event/KafkaConsumerConfiguration.java
@@ -0,0 +1,26 @@
+package com.sportsbook.risk.event;
+
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.kafka.listener.CommonErrorHandler;
+import org.springframework.kafka.listener.DefaultErrorHandler;
+import org.springframework.util.backoff.FixedBackOff;
+
+/** Retains transient Redis and Kafka failures on the source partition for retry. */
+@Configuration
+public class KafkaConsumerConfiguration {
+  static final long RETRY_INTERVAL_MILLIS = 1_000L;
+
+  @Bean
+  CommonErrorHandler riskKafkaErrorHandler() {
+    DefaultErrorHandler handler = new DefaultErrorHandler(retryBackOff());
+    handler.addRetryableExceptions(Exception.class);
+    handler.setAckAfterHandle(false);
+    handler.setCommitRecovered(false);
+    return handler;
+  }
+
+  static FixedBackOff retryBackOff() {
+    return new FixedBackOff(RETRY_INTERVAL_MILLIS, FixedBackOff.UNLIMITED_ATTEMPTS);
+  }
+}


## `test(events): verify unbounded accepted bet retries`

diff --git a/src/test/java/com/sportsbook/risk/event/KafkaConsumerConfigurationTest.java b/src/test/java/com/sportsbook/risk/event/KafkaConsumerConfigurationTest.java
new file mode 100644
index 0000000..735983d
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/event/KafkaConsumerConfigurationTest.java
@@ -0,0 +1,21 @@
+package com.sportsbook.risk.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import org.junit.jupiter.api.Test;
+import org.springframework.kafka.listener.CommonErrorHandler;
+import org.springframework.kafka.listener.DefaultErrorHandler;
+import org.springframework.util.backoff.FixedBackOff;
+
+class KafkaConsumerConfigurationTest {
+  @Test
+  void transientFailuresRemainUnacknowledgedForRetry() {
+    CommonErrorHandler handler = new KafkaConsumerConfiguration().riskKafkaErrorHandler();
+
+    assertThat(handler).isInstanceOf(DefaultErrorHandler.class);
+    assertThat(handler.isAckAfterHandle()).isFalse();
+    FixedBackOff backOff = KafkaConsumerConfiguration.retryBackOff();
+    assertThat(backOff.getInterval()).isEqualTo(1_000L);
+    assertThat(backOff.getMaxAttempts()).isEqualTo(FixedBackOff.UNLIMITED_ATTEMPTS);
+  }
+}


## `feat(events): consume accepted bet events`

diff --git a/src/main/java/com/sportsbook/risk/event/BetPlacedConsumer.java b/src/main/java/com/sportsbook/risk/event/BetPlacedConsumer.java
new file mode 100644
index 0000000..3ef0783
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/event/BetPlacedConsumer.java
@@ -0,0 +1,80 @@
+package com.sportsbook.risk.event;
+
+import java.time.Clock;
+import java.util.Objects;
+import java.util.function.Supplier;
+import org.springframework.beans.factory.ObjectProvider;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.kafka.annotation.KafkaListener;
+import org.springframework.kafka.support.Acknowledgment;
+import org.springframework.kafka.support.KafkaHeaders;
+import org.springframework.messaging.handler.annotation.Header;
+import org.springframework.messaging.handler.annotation.Payload;
+import org.springframework.stereotype.Component;
+
+/** Reconciles accepted bets and durably quarantines only permanent input failures. */
+@Component
+public final class BetPlacedConsumer {
+  private final Supplier<AcceptedBetReconciler> reconciler;
+  private final BetPlacedDeadLetterPublisher deadLetters;
+  private final Clock clock;
+
+  @Autowired
+  public BetPlacedConsumer(
+      ObjectProvider<AcceptedBetReconciler> reconciler, BetPlacedDeadLetterPublisher deadLetters) {
+    this((Supplier<AcceptedBetReconciler>) reconciler::getIfUnique, deadLetters, Clock.systemUTC());
+  }
+
+  BetPlacedConsumer(
+      AcceptedBetReconciler reconciler, BetPlacedDeadLetterPublisher deadLetters, Clock clock) {
+    this(() -> Objects.requireNonNull(reconciler, "reconciler"), deadLetters, clock);
+  }
+
+  private BetPlacedConsumer(
+      Supplier<AcceptedBetReconciler> reconciler,
+      BetPlacedDeadLetterPublisher deadLetters,
+      Clock clock) {
+    this.reconciler = Objects.requireNonNull(reconciler, "reconciler");
+    this.deadLetters = Objects.requireNonNull(deadLetters, "deadLetters");
+    this.clock = Objects.requireNonNull(clock, "clock");
+  }
+
+  @KafkaListener(
+      topics = "${risk.topics.bet-placed:bet.placed.v1}",
+      groupId = "${spring.kafka.consumer.group-id:risk.bet-placed-consumer}")
+  public void onBetPlaced(
+      @Payload(required = false) byte[] payload,
+      @Header(value = KafkaHeaders.RECEIVED_KEY, required = false) String key,
+      Acknowledgment acknowledgment) {
+    Objects.requireNonNull(acknowledgment, "acknowledgment");
+    AcceptedBetEnvelope envelope;
+    try {
+      envelope = AcceptedBetEnvelope.decode(key, payload, clock.instant());
+    } catch (RuntimeException failure) {
+      deadLetter(key, payload, BetPlacedFailureReason.fromDecodeFailure(failure), acknowledgment);
+      return;
+    }
+
+    AcceptedBetReconciliation result =
+        Objects.requireNonNull(requiredReconciler().reconcile(envelope), "reconciliation result");
+    if (result.permanentFailure()) {
+      deadLetter(key, payload, result.failureReason(), acknowledgment);
+      return;
+    }
+    acknowledgment.acknowledge();
+  }
+
+  private AcceptedBetReconciler requiredReconciler() {
+    AcceptedBetReconciler value = reconciler.get();
+    if (value == null) {
+      throw new IllegalStateException("exactly one accepted-bet reconciler is required");
+    }
+    return value;
+  }
+
+  private void deadLetter(
+      String key, byte[] payload, BetPlacedFailureReason reason, Acknowledgment acknowledgment) {
+    deadLetters.publishAndAwait(key, payload, reason);
+    acknowledgment.acknowledge();
+  }
+}


## `test(events): quarantine malformed accepted bets`

diff --git a/src/test/java/com/sportsbook/risk/event/BetPlacedConsumerInvalidInputTest.java b/src/test/java/com/sportsbook/risk/event/BetPlacedConsumerInvalidInputTest.java
new file mode 100644
index 0000000..85a5906
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/event/BetPlacedConsumerInvalidInputTest.java
@@ -0,0 +1,59 @@
+package com.sportsbook.risk.event;
+
+import static org.mockito.ArgumentMatchers.any;
+import static org.mockito.ArgumentMatchers.eq;
+import static org.mockito.ArgumentMatchers.same;
+import static org.mockito.Mockito.inOrder;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.never;
+import static org.mockito.Mockito.verify;
+
+import java.time.Clock;
+import java.time.ZoneOffset;
+import org.junit.jupiter.api.BeforeEach;
+import org.junit.jupiter.api.Test;
+import org.mockito.InOrder;
+import org.springframework.kafka.support.Acknowledgment;
+
+class BetPlacedConsumerInvalidInputTest {
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
+  void malformedPayloadIsPublishedBeforeItsSourceOffsetIsAcknowledged() {
+    byte[] malformed = {1, 2, 3};
+
+    consumer.onBetPlaced(malformed, BetPlacedEventFixture.USER_ID, acknowledgment);
+
+    InOrder order = inOrder(deadLetters, acknowledgment);
+    order
+        .verify(deadLetters)
+        .publishAndAwait(
+            eq(BetPlacedEventFixture.USER_ID),
+            same(malformed),
+            eq(BetPlacedFailureReason.MALFORMED_EVENT));
+    order.verify(acknowledgment).acknowledge();
+    verify(reconciler, never()).reconcile(any());
+  }
+
+  @Test
+  void mismatchedKafkaKeyHasItsOwnPermanentClassification() {
+    byte[] payload = BetPlacedEventFixture.payload();
+
+    consumer.onBetPlaced(payload, BetPlacedEventFixture.OTHER_USER_ID, acknowledgment);
+
+    verify(deadLetters)
+        .publishAndAwait(
+            BetPlacedEventFixture.OTHER_USER_ID, payload, BetPlacedFailureReason.KEY_MISMATCH);
+    verify(acknowledgment).acknowledge();
+    verify(reconciler, never()).reconcile(any());
+  }
+}
diff --git a/src/test/java/com/sportsbook/risk/event/BetPlacedEventFixture.java b/src/test/java/com/sportsbook/risk/event/BetPlacedEventFixture.java
new file mode 100644
index 0000000..095a949
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/event/BetPlacedEventFixture.java
@@ -0,0 +1,38 @@
+package com.sportsbook.risk.event;
+
+import com.sportsbook.protocol.event.BetPlacedRequested;
+import com.sportsbook.protocol.event.BetSlipTypeTag;
+import com.sportsbook.protocol.event.Money;
+import com.sportsbook.protocol.event.RequestedSelection;
+import java.time.Instant;
+import java.util.List;
+
+final class BetPlacedEventFixture {
+  static final String USER_ID = "10000000-0000-4000-8000-000000000001";
+  static final String OTHER_USER_ID = "10000000-0000-4000-8000-000000000002";
+  static final Instant REQUESTED_AT = Instant.parse("2026-08-21T06:00:00Z");
+  static final Instant OBSERVED_AT = REQUESTED_AT.plusSeconds(1);
+
+  private BetPlacedEventFixture() {}
+
+  static byte[] payload() {
+    RequestedSelection selection =
+        new RequestedSelection(
+            "30000000-0000-4000-8000-000000000001",
+            "40000000-0000-4000-8000-000000000001",
+            "50000000-0000-4000-8000-000000000001",
+            "1.85");
+    BetPlacedRequested event =
+        new BetPlacedRequested(
+            "20000000-0000-4000-8000-000000000001",
+            USER_ID,
+            BetSlipTypeTag.SINGLE,
+            null,
+            null,
+            List.of(selection),
+            new Money(10_000L, "KRW"),
+            "accepted-bet-fixture",
+            REQUESTED_AT);
+    return AvroCodec.encode(event);
+  }
+}


## `test(events): preserve source offsets on dead letter failure`

diff --git a/src/test/java/com/sportsbook/risk/event/BetPlacedConsumerInvalidInputTest.java b/src/test/java/com/sportsbook/risk/event/BetPlacedConsumerInvalidInputTest.java
index 85a5906..c6f7630 100644
--- a/src/test/java/com/sportsbook/risk/event/BetPlacedConsumerInvalidInputTest.java
+++ b/src/test/java/com/sportsbook/risk/event/BetPlacedConsumerInvalidInputTest.java
@@ -1,8 +1,10 @@
 package com.sportsbook.risk.event;
 
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
 import static org.mockito.ArgumentMatchers.any;
 import static org.mockito.ArgumentMatchers.eq;
 import static org.mockito.ArgumentMatchers.same;
+import static org.mockito.Mockito.doThrow;
 import static org.mockito.Mockito.inOrder;
 import static org.mockito.Mockito.mock;
 import static org.mockito.Mockito.never;
@@ -56,4 +58,20 @@ class BetPlacedConsumerInvalidInputTest {
     verify(acknowledgment).acknowledge();
     verify(reconciler, never()).reconcile(any());
   }
+
+  @Test
+  void failedDeadLetterPublishLeavesTheSourceOffsetUnacknowledged() {
+    byte[] payload = BetPlacedEventFixture.payload();
+    doThrow(new IllegalStateException("broker unavailable"))
+        .when(deadLetters)
+        .publishAndAwait(any(), any(), any());
+
+    assertThatThrownBy(
+            () ->
+                consumer.onBetPlaced(payload, BetPlacedEventFixture.OTHER_USER_ID, acknowledgment))
+        .isInstanceOf(IllegalStateException.class)
+        .hasMessage("broker unavailable");
+
+    verify(acknowledgment, never()).acknowledge();
+  }
 }


