## `feat(events): type accepted bet key mismatches`

diff --git a/src/main/java/com/sportsbook/risk/event/AcceptedBetEnvelope.java b/src/main/java/com/sportsbook/risk/event/AcceptedBetEnvelope.java
index b99a55e..f798b11 100644
--- a/src/main/java/com/sportsbook/risk/event/AcceptedBetEnvelope.java
+++ b/src/main/java/com/sportsbook/risk/event/AcceptedBetEnvelope.java
@@ -38,7 +38,7 @@ public record AcceptedBetEnvelope(RiskCheckCommand command, Instant requestedAt)
 
     UserId userId = UserId.of(uuid(event.getUserId(), "userId"));
     if (!userId.value().toString().equals(kafkaKey)) {
-      throw new IllegalArgumentException("Kafka key must equal userId");
+      throw new BetPlacedKeyMismatchException();
     }
     BetId betId = BetId.of(uuid(event.getBetId(), "betId"));
     List<SelectionId> selectionIds =
diff --git a/src/main/java/com/sportsbook/risk/event/BetPlacedKeyMismatchException.java b/src/main/java/com/sportsbook/risk/event/BetPlacedKeyMismatchException.java
new file mode 100644
index 0000000..5f5ec67
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/event/BetPlacedKeyMismatchException.java
@@ -0,0 +1,8 @@
+package com.sportsbook.risk.event;
+
+/** Identifies a Kafka partition-key contract violation independently of malformed payloads. */
+final class BetPlacedKeyMismatchException extends IllegalArgumentException {
+  BetPlacedKeyMismatchException() {
+    super("Kafka key must equal userId");
+  }
+}


## `test(events): verify accepted bet key mismatch typing`

diff --git a/src/test/java/com/sportsbook/risk/event/AcceptedBetEnvelopeTest.java b/src/test/java/com/sportsbook/risk/event/AcceptedBetEnvelopeTest.java
index 851a30e..db955a4 100644
--- a/src/test/java/com/sportsbook/risk/event/AcceptedBetEnvelopeTest.java
+++ b/src/test/java/com/sportsbook/risk/event/AcceptedBetEnvelopeTest.java
@@ -38,6 +38,7 @@ class AcceptedBetEnvelopeTest {
     assertThatThrownBy(() -> decode(malformed, USER)).isInstanceOf(IllegalArgumentException.class);
 
     assertThatThrownBy(() -> decode(validEvent(), "10000000-0000-4000-8000-000000000002"))
+        .isInstanceOf(BetPlacedKeyMismatchException.class)
         .hasMessageContaining("Kafka key");
   }
 


## `feat(events): classify permanent accepted bet failures`

diff --git a/src/main/java/com/sportsbook/risk/event/BetPlacedFailureReason.java b/src/main/java/com/sportsbook/risk/event/BetPlacedFailureReason.java
new file mode 100644
index 0000000..01d1621
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/event/BetPlacedFailureReason.java
@@ -0,0 +1,16 @@
+package com.sportsbook.risk.event;
+
+/** Stable DLT classifications for permanently unprocessable accepted-bet events. */
+public enum BetPlacedFailureReason {
+  MALFORMED_EVENT,
+  KEY_MISMATCH,
+  FINGERPRINT_MISMATCH,
+  TERMINAL_RESERVATION;
+
+  static BetPlacedFailureReason fromDecodeFailure(RuntimeException failure) {
+    if (failure instanceof BetPlacedKeyMismatchException) {
+      return KEY_MISMATCH;
+    }
+    return MALFORMED_EVENT;
+  }
+}


## `test(events): verify permanent event classifications`

diff --git a/src/test/java/com/sportsbook/risk/event/BetPlacedFailureReasonTest.java b/src/test/java/com/sportsbook/risk/event/BetPlacedFailureReasonTest.java
new file mode 100644
index 0000000..5a0de3f
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/event/BetPlacedFailureReasonTest.java
@@ -0,0 +1,17 @@
+package com.sportsbook.risk.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import org.junit.jupiter.api.Test;
+
+class BetPlacedFailureReasonTest {
+  @Test
+  void distinguishesPartitionKeyViolationsFromOtherMalformedEvents() {
+    assertThat(BetPlacedFailureReason.fromDecodeFailure(new BetPlacedKeyMismatchException()))
+        .isEqualTo(BetPlacedFailureReason.KEY_MISMATCH);
+    assertThat(
+            BetPlacedFailureReason.fromDecodeFailure(
+                new IllegalArgumentException("selectionIds must be unique")))
+        .isEqualTo(BetPlacedFailureReason.MALFORMED_EVENT);
+  }
+}


## `fix(kafka): preserve binary event payloads`

diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index dcfd4b3..31a5c6c 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -18,8 +18,12 @@ spring:
       group-id: risk.bet-placed-consumer
       auto-offset-reset: earliest
       enable-auto-commit: false
+      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
+      value-deserializer: org.apache.kafka.common.serialization.ByteArrayDeserializer
     producer:
       acks: all
+      key-serializer: org.apache.kafka.common.serialization.StringSerializer
+      value-serializer: org.apache.kafka.common.serialization.ByteArraySerializer
       properties:
         max.block.ms: 1000
     listener:


## `test(kafka): verify binary serde configuration`

diff --git a/src/test/java/com/sportsbook/risk/DeployedRiskConfigurationTest.java b/src/test/java/com/sportsbook/risk/DeployedRiskConfigurationTest.java
index fea206a..f1a1bf9 100644
--- a/src/test/java/com/sportsbook/risk/DeployedRiskConfigurationTest.java
+++ b/src/test/java/com/sportsbook/risk/DeployedRiskConfigurationTest.java
@@ -39,6 +39,14 @@ class DeployedRiskConfigurationTest {
     assertThat(patterns.repeatedSelection().action()).isEqualTo(PatternAction.REVIEW);
     assertThat(reservations.retention()).isEqualTo(Duration.ofDays(32));
     assertThat(history.maxStakeSamples()).isEqualTo(100);
+    assertThat(source.getProperty("spring.kafka.consumer.key-deserializer"))
+        .isEqualTo("org.apache.kafka.common.serialization.StringDeserializer");
+    assertThat(source.getProperty("spring.kafka.consumer.value-deserializer"))
+        .isEqualTo("org.apache.kafka.common.serialization.ByteArrayDeserializer");
+    assertThat(source.getProperty("spring.kafka.producer.key-serializer"))
+        .isEqualTo("org.apache.kafka.common.serialization.StringSerializer");
+    assertThat(source.getProperty("spring.kafka.producer.value-serializer"))
+        .isEqualTo("org.apache.kafka.common.serialization.ByteArraySerializer");
     assertThat(source.getProperty("management.endpoint.health.group.readiness.include"))
         .isEqualTo("readinessState,redis,kafka");
   }
