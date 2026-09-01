## `feat(policy): compose diagnostic evaluation`

diff --git a/src/main/java/com/sportsbook/risk/service/RiskPolicyConfiguration.java b/src/main/java/com/sportsbook/risk/service/RiskPolicyConfiguration.java
new file mode 100644
index 0000000..936a2f5
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/service/RiskPolicyConfiguration.java
@@ -0,0 +1,37 @@
+package com.sportsbook.risk.service;
+
+import com.sportsbook.risk.event.RiskSignalPublisher;
+import com.sportsbook.risk.pattern.RuleEngine;
+import com.sportsbook.risk.pattern.rule.RapidBettingRule;
+import com.sportsbook.risk.pattern.rule.RepeatedSameSelectionRule;
+import com.sportsbook.risk.pattern.rule.SuddenStakeIncreaseRule;
+import com.sportsbook.risk.policy.RiskLimitProperties;
+import com.sportsbook.risk.policy.RiskPatternProperties;
+import com.sportsbook.risk.snapshot.RiskSnapshotReader;
+import io.micrometer.core.instrument.MeterRegistry;
+import java.util.List;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+
+/** Composes deterministic diagnostic policy collaborators. */
+@Configuration
+public class RiskPolicyConfiguration {
+  @Bean
+  RuleEngine riskRuleEngine(RiskPatternProperties patterns) {
+    return new RuleEngine(
+        List.of(
+            new RapidBettingRule(patterns.rapidBetting()),
+            new SuddenStakeIncreaseRule(patterns.suddenStake()),
+            new RepeatedSameSelectionRule(patterns.repeatedSelection())));
+  }
+
+  @Bean
+  RiskCheckService riskCheckService(
+      RiskLimitProperties limits,
+      RiskSnapshotReader snapshots,
+      RuleEngine rules,
+      RiskSignalPublisher signals,
+      MeterRegistry meters) {
+    return new RiskCheckService(limits, snapshots, rules, signals, meters);
+  }
+}


## `test(policy): verify diagnostic composition`

diff --git a/src/test/java/com/sportsbook/risk/service/RiskPolicyConfigurationTest.java b/src/test/java/com/sportsbook/risk/service/RiskPolicyConfigurationTest.java
new file mode 100644
index 0000000..2eb78c1
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/service/RiskPolicyConfigurationTest.java
@@ -0,0 +1,30 @@
+package com.sportsbook.risk.service;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.mock;
+
+import com.sportsbook.risk.event.RiskSignalPublisher;
+import com.sportsbook.risk.policy.RiskLimitProperties;
+import com.sportsbook.risk.policy.RiskPatternProperties;
+import com.sportsbook.risk.snapshot.RiskSnapshotReader;
+import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
+import org.junit.jupiter.api.Test;
+
+class RiskPolicyConfigurationTest {
+  @Test
+  void composesRulesInTheirStablePriorityOrder() {
+    RiskPolicyConfiguration configuration = new RiskPolicyConfiguration();
+    var rules = configuration.riskRuleEngine(new RiskPatternProperties(null, null, null));
+
+    assertThat(rules.ruleOrder())
+        .containsExactly("RAPID_BETTING", "SUDDEN_STAKE_INCREASE", "REPEATED_SAME_SELECTION");
+    assertThat(
+            configuration.riskCheckService(
+                new RiskLimitProperties(null, null, null, null, 0),
+                mock(RiskSnapshotReader.class),
+                rules,
+                mock(RiskSignalPublisher.class),
+                new SimpleMeterRegistry()))
+        .isNotNull();
+  }
+}


## `feat(events): publish best-effort limit signals`

diff --git a/src/main/java/com/sportsbook/risk/event/KafkaRiskSignalPublisher.java b/src/main/java/com/sportsbook/risk/event/KafkaRiskSignalPublisher.java
new file mode 100644
index 0000000..b521710
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/event/KafkaRiskSignalPublisher.java
@@ -0,0 +1,79 @@
+package com.sportsbook.risk.event;
+
+import com.sportsbook.protocol.event.RiskLimitType;
+import com.sportsbook.protocol.event.RiskLimitViolated;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.counter.LimitType;
+import com.sportsbook.risk.pattern.PatternMatch;
+import java.time.Instant;
+import java.util.Objects;
+import org.slf4j.Logger;
+import org.slf4j.LoggerFactory;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.stereotype.Component;
+
+/** Best-effort Kafka publisher for shared risk signal records. */
+@Component
+public final class KafkaRiskSignalPublisher implements RiskSignalPublisher {
+  private static final Logger LOG = LoggerFactory.getLogger(KafkaRiskSignalPublisher.class);
+
+  private final KafkaTemplate<String, byte[]> kafka;
+  private final EventTopics topics;
+
+  public KafkaRiskSignalPublisher(KafkaTemplate<String, byte[]> kafka, EventTopics topics) {
+    this.kafka = Objects.requireNonNull(kafka, "kafka");
+    this.topics = Objects.requireNonNull(topics, "topics");
+  }
+
+  @Override
+  public void publishLimit(
+      UserId userId,
+      LimitType type,
+      long current,
+      long limit,
+      Money candidate,
+      Instant occurredAt) {
+    Objects.requireNonNull(userId, "userId");
+    Objects.requireNonNull(type, "type");
+    Objects.requireNonNull(candidate, "candidate");
+    Objects.requireNonNull(occurredAt, "occurredAt");
+    RiskLimitType wireType = wireType(type);
+    if (wireType == null) {
+      return;
+    }
+    RiskLimitViolated event =
+        new RiskLimitViolated(
+            userId.value().toString(),
+            wireType,
+            current,
+            limit,
+            new com.sportsbook.protocol.event.Money(
+                candidate.amount(), candidate.currency().name()),
+            occurredAt);
+    send(topics.limitViolated(), userId.value().toString(), AvroCodec.encode(event));
+  }
+
+  @Override
+  public void publishPattern(UserId userId, PatternMatch match, Instant occurredAt) {
+    Objects.requireNonNull(userId, "userId");
+    Objects.requireNonNull(match, "match");
+    Objects.requireNonNull(occurredAt, "occurredAt");
+  }
+
+  private void send(String topic, String key, byte[] payload) {
+    try {
+      kafka.send(topic, key, payload);
+    } catch (RuntimeException exception) {
+      LOG.warn("risk signal submission failed topic={}", topic, exception);
+    }
+  }
+
+  private static RiskLimitType wireType(LimitType type) {
+    return switch (type) {
+      case STAKE_DAILY -> RiskLimitType.STAKE_DAILY;
+      case SELECTIONS_PER_MINUTE -> RiskLimitType.SELECTIONS_PER_MINUTE;
+      case STAKE_WEEKLY, STAKE_MONTHLY -> null;
+    };
+  }
+}
diff --git a/src/main/java/com/sportsbook/risk/event/RiskSignalPublisher.java b/src/main/java/com/sportsbook/risk/event/RiskSignalPublisher.java
new file mode 100644
index 0000000..6a52ed3
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/event/RiskSignalPublisher.java
@@ -0,0 +1,15 @@
+package com.sportsbook.risk.event;
+
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.counter.LimitType;
+import com.sportsbook.risk.pattern.PatternMatch;
+import java.time.Instant;
+
+/** Non-authoritative risk signal boundary; publication never decides admission. */
+public interface RiskSignalPublisher {
+  void publishLimit(
+      UserId userId, LimitType type, long current, long limit, Money candidate, Instant occurredAt);
+
+  void publishPattern(UserId userId, PatternMatch match, Instant occurredAt);
+}


## `feat(events): observe signal delivery failures`

diff --git a/src/main/java/com/sportsbook/risk/event/KafkaRiskSignalPublisher.java b/src/main/java/com/sportsbook/risk/event/KafkaRiskSignalPublisher.java
index b521710..f7a8d2c 100644
--- a/src/main/java/com/sportsbook/risk/event/KafkaRiskSignalPublisher.java
+++ b/src/main/java/com/sportsbook/risk/event/KafkaRiskSignalPublisher.java
@@ -6,10 +6,14 @@ import com.sportsbook.protocol.value.Money;
 import com.sportsbook.protocol.value.UserId;
 import com.sportsbook.risk.counter.LimitType;
 import com.sportsbook.risk.pattern.PatternMatch;
+import io.micrometer.core.instrument.Counter;
+import io.micrometer.core.instrument.MeterRegistry;
 import java.time.Instant;
 import java.util.Objects;
+import org.apache.avro.specific.SpecificRecordBase;
 import org.slf4j.Logger;
 import org.slf4j.LoggerFactory;
+import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.kafka.core.KafkaTemplate;
 import org.springframework.stereotype.Component;
 
@@ -20,10 +24,22 @@ public final class KafkaRiskSignalPublisher implements RiskSignalPublisher {
 
   private final KafkaTemplate<String, byte[]> kafka;
   private final EventTopics topics;
+  private final Counter delivered;
+  private final Counter failed;
 
-  public KafkaRiskSignalPublisher(KafkaTemplate<String, byte[]> kafka, EventTopics topics) {
+  @Autowired
+  public KafkaRiskSignalPublisher(
+      KafkaTemplate<String, byte[]> kafka, EventTopics topics, MeterRegistry meters) {
     this.kafka = Objects.requireNonNull(kafka, "kafka");
     this.topics = Objects.requireNonNull(topics, "topics");
+    Objects.requireNonNull(meters, "meters");
+    this.delivered =
+        Counter.builder("risk.signal.delivery").tag("outcome", "delivered").register(meters);
+    this.failed = Counter.builder("risk.signal.delivery").tag("outcome", "failed").register(meters);
+  }
+
+  KafkaRiskSignalPublisher(KafkaTemplate<String, byte[]> kafka, EventTopics topics) {
+    this(kafka, topics, io.micrometer.core.instrument.Metrics.globalRegistry);
   }
 
   @Override
@@ -51,7 +67,7 @@ public final class KafkaRiskSignalPublisher implements RiskSignalPublisher {
             new com.sportsbook.protocol.event.Money(
                 candidate.amount(), candidate.currency().name()),
             occurredAt);
-    send(topics.limitViolated(), userId.value().toString(), AvroCodec.encode(event));
+    send(topics.limitViolated(), userId.value().toString(), event);
   }
 
   @Override
@@ -61,10 +77,21 @@ public final class KafkaRiskSignalPublisher implements RiskSignalPublisher {
     Objects.requireNonNull(occurredAt, "occurredAt");
   }
 
-  private void send(String topic, String key, byte[] payload) {
+  private void send(String topic, String key, SpecificRecordBase record) {
     try {
-      kafka.send(topic, key, payload);
+      kafka
+          .send(topic, key, AvroCodec.encode(record))
+          .whenComplete(
+              (result, exception) -> {
+                if (exception == null) {
+                  delivered.increment();
+                } else {
+                  failed.increment();
+                  LOG.warn("risk signal delivery failed topic={}", topic, exception);
+                }
+              });
     } catch (RuntimeException exception) {
+      failed.increment();
       LOG.warn("risk signal submission failed topic={}", topic, exception);
     }
   }


## `feat(events): publish best-effort pattern signals`

diff --git a/src/main/java/com/sportsbook/risk/event/KafkaRiskSignalPublisher.java b/src/main/java/com/sportsbook/risk/event/KafkaRiskSignalPublisher.java
index f7a8d2c..0c7fbda 100644
--- a/src/main/java/com/sportsbook/risk/event/KafkaRiskSignalPublisher.java
+++ b/src/main/java/com/sportsbook/risk/event/KafkaRiskSignalPublisher.java
@@ -2,13 +2,19 @@ package com.sportsbook.risk.event;
 
 import com.sportsbook.protocol.event.RiskLimitType;
 import com.sportsbook.protocol.event.RiskLimitViolated;
+import com.sportsbook.protocol.event.RiskPatternSuspected;
+import com.sportsbook.protocol.event.RiskPatternType;
 import com.sportsbook.protocol.value.Money;
 import com.sportsbook.protocol.value.UserId;
 import com.sportsbook.risk.counter.LimitType;
 import com.sportsbook.risk.pattern.PatternMatch;
+import com.sportsbook.risk.pattern.rule.RapidBettingRule;
+import com.sportsbook.risk.pattern.rule.RepeatedSameSelectionRule;
+import com.sportsbook.risk.pattern.rule.SuddenStakeIncreaseRule;
 import io.micrometer.core.instrument.Counter;
 import io.micrometer.core.instrument.MeterRegistry;
 import java.time.Instant;
+import java.util.Map;
 import java.util.Objects;
 import org.apache.avro.specific.SpecificRecordBase;
 import org.slf4j.Logger;
@@ -75,6 +81,17 @@ public final class KafkaRiskSignalPublisher implements RiskSignalPublisher {
     Objects.requireNonNull(userId, "userId");
     Objects.requireNonNull(match, "match");
     Objects.requireNonNull(occurredAt, "occurredAt");
+    RiskPatternType wireType = patternType(match.rule());
+    if (wireType == null) {
+      return;
+    }
+    RiskPatternSuspected event =
+        new RiskPatternSuspected(
+            userId.value().toString(),
+            wireType,
+            Map.of("action", match.action().name(), "reason", match.reason()),
+            occurredAt);
+    send(topics.patternSuspected(), userId.value().toString(), event);
   }
 
   private void send(String topic, String key, SpecificRecordBase record) {
@@ -103,4 +120,13 @@ public final class KafkaRiskSignalPublisher implements RiskSignalPublisher {
       case STAKE_WEEKLY, STAKE_MONTHLY -> null;
     };
   }
+
+  private static RiskPatternType patternType(String rule) {
+    return switch (rule) {
+      case RapidBettingRule.NAME -> RiskPatternType.RAPID_BETTING;
+      case SuddenStakeIncreaseRule.NAME -> RiskPatternType.SUDDEN_STAKE_INCREASE;
+      case RepeatedSameSelectionRule.NAME -> RiskPatternType.REPEATED_SAME_SELECTION;
+      default -> null;
+    };
+  }
 }


## `test(events): verify non-authoritative delivery failures`

diff --git a/src/test/java/com/sportsbook/risk/event/KafkaRiskSignalPublisherTest.java b/src/test/java/com/sportsbook/risk/event/KafkaRiskSignalPublisherTest.java
index 5c8adcb..1c16970 100644
--- a/src/test/java/com/sportsbook/risk/event/KafkaRiskSignalPublisherTest.java
+++ b/src/test/java/com/sportsbook/risk/event/KafkaRiskSignalPublisherTest.java
@@ -1,19 +1,25 @@
 package com.sportsbook.risk.event;
 
 import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatCode;
 import static org.mockito.ArgumentMatchers.any;
 import static org.mockito.ArgumentMatchers.eq;
+import static org.mockito.Mockito.doReturn;
+import static org.mockito.Mockito.doThrow;
 import static org.mockito.Mockito.mock;
 import static org.mockito.Mockito.times;
 import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
 
 import com.sportsbook.protocol.event.RiskLimitType;
 import com.sportsbook.protocol.event.RiskLimitViolated;
 import com.sportsbook.protocol.value.Money;
 import com.sportsbook.protocol.value.UserId;
 import com.sportsbook.risk.counter.LimitType;
+import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
 import java.time.Instant;
 import java.util.UUID;
+import java.util.concurrent.CompletableFuture;
 import org.junit.jupiter.api.Test;
 import org.mockito.ArgumentCaptor;
 import org.springframework.kafka.core.KafkaTemplate;
@@ -56,8 +62,39 @@ class KafkaRiskSignalPublisherTest {
     verify(kafka, times(1)).send(any(), any(), any());
   }
 
+  @Test
+  void containsSynchronousAndAsynchronousDeliveryFailures() {
+    KafkaTemplate<String, byte[]> synchronousKafka = template();
+    SimpleMeterRegistry meters = new SimpleMeterRegistry();
+    KafkaRiskSignalPublisher synchronousPublisher =
+        new KafkaRiskSignalPublisher(synchronousKafka, TOPICS, meters);
+    doThrow(new IllegalStateException("unavailable"))
+        .when(synchronousKafka)
+        .send(any(), any(), any());
+
+    assertThatCode(
+            () ->
+                synchronousPublisher.publishLimit(
+                    USER, LimitType.STAKE_DAILY, 1, 2, Money.krw(1), Instant.EPOCH))
+        .doesNotThrowAnyException();
+    CompletableFuture<org.springframework.kafka.support.SendResult<String, byte[]>> failed =
+        new CompletableFuture<>();
+    failed.completeExceptionally(new IllegalStateException("broker rejected"));
+    KafkaTemplate<String, byte[]> asynchronousKafka = template();
+    doReturn(failed).when(asynchronousKafka).send(any(), any(), any());
+    KafkaRiskSignalPublisher asynchronousPublisher =
+        new KafkaRiskSignalPublisher(asynchronousKafka, TOPICS, meters);
+    asynchronousPublisher.publishLimit(
+        USER, LimitType.STAKE_DAILY, 1, 2, Money.krw(1), Instant.EPOCH);
+
+    assertThat(meters.get("risk.signal.delivery").tag("outcome", "failed").counter().count())
+        .isEqualTo(2);
+  }
+
   @SuppressWarnings("unchecked")
   private static KafkaTemplate<String, byte[]> template() {
-    return mock(KafkaTemplate.class);
+    KafkaTemplate<String, byte[]> kafka = mock(KafkaTemplate.class);
+    when(kafka.send(any(), any(), any())).thenReturn(CompletableFuture.completedFuture(null));
+    return kafka;
   }
 }
