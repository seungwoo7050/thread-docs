# 운영 메트릭과 Redis·Kafka 준비 상태

## `build(observability): add metrics and tracing`

diff --git a/pom.xml b/pom.xml
index 46c854a..b1b994a 100644
--- a/pom.xml
+++ b/pom.xml
@@ -18,6 +18,7 @@
         <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
         <spring-boot.version>3.2.11</spring-boot.version>
         <shared-protocol.version>1.0.0</shared-protocol.version>
+        <logstash-encoder.version>7.4</logstash-encoder.version>
         <maven-compiler.version>3.13.0</maven-compiler.version>
         <maven-surefire.version>3.5.1</maven-surefire.version>
     </properties>
@@ -64,6 +65,23 @@
             <groupId>org.springframework.kafka</groupId>
             <artifactId>spring-kafka</artifactId>
         </dependency>
+        <dependency>
+            <groupId>io.micrometer</groupId>
+            <artifactId>micrometer-registry-prometheus</artifactId>
+        </dependency>
+        <dependency>
+            <groupId>io.micrometer</groupId>
+            <artifactId>micrometer-tracing-bridge-otel</artifactId>
+        </dependency>
+        <dependency>
+            <groupId>io.opentelemetry</groupId>
+            <artifactId>opentelemetry-exporter-otlp</artifactId>
+        </dependency>
+        <dependency>
+            <groupId>net.logstash.logback</groupId>
+            <artifactId>logstash-logback-encoder</artifactId>
+            <version>${logstash-encoder.version}</version>
+        </dependency>
     </dependencies>
 
     <build>


## `feat(metrics): count reservation decisions`

diff --git a/src/main/java/com/sportsbook/risk/reservation/RiskReservationService.java b/src/main/java/com/sportsbook/risk/reservation/RiskReservationService.java
index a4858e6..b8fa433 100644
--- a/src/main/java/com/sportsbook/risk/reservation/RiskReservationService.java
+++ b/src/main/java/com/sportsbook/risk/reservation/RiskReservationService.java
@@ -2,21 +2,33 @@ package com.sportsbook.risk.reservation;
 
 import com.sportsbook.protocol.value.BetId;
 import com.sportsbook.risk.service.RiskCheckCommand;
+import io.micrometer.core.instrument.MeterRegistry;
+import io.micrometer.core.instrument.Metrics;
 import java.time.Instant;
 import java.util.Objects;
+import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.stereotype.Service;
 
 /** Application boundary for atomic betting admission and lifecycle transitions. */
 @Service
 public final class RiskReservationService {
   private final RiskReservationStore store;
+  private final MeterRegistry meters;
 
-  public RiskReservationService(RiskReservationStore store) {
+  @Autowired
+  public RiskReservationService(RiskReservationStore store, MeterRegistry meters) {
     this.store = Objects.requireNonNull(store, "store");
+    this.meters = Objects.requireNonNull(meters, "meters");
+  }
+
+  RiskReservationService(RiskReservationStore store) {
+    this(store, Metrics.globalRegistry);
   }
 
   public ReservationDecision reserve(RiskCheckCommand command) {
-    return store.reserve(Objects.requireNonNull(command, "command"));
+    ReservationDecision decision = store.reserve(Objects.requireNonNull(command, "command"));
+    meters.counter("risk.reservation.requests", "result", decisionResult(decision)).increment();
+    return decision;
   }
 
   public ReservationTransition commit(BetId betId, String token, Instant now) {
@@ -30,4 +42,15 @@ public final class RiskReservationService {
     return store.release(
         Objects.requireNonNull(betId, "betId"), Objects.requireNonNull(now, "now"));
   }
+
+  private static String decisionResult(ReservationDecision decision) {
+    if (decision.replayed()) {
+      return "replayed";
+    }
+    return switch (decision.status()) {
+      case APPROVED -> "created";
+      case REJECTED -> "rejected";
+      case CONFLICT -> "conflict";
+    };
+  }
 }


## `test(metrics): verify reservation decision metrics`

diff --git a/src/test/java/com/sportsbook/risk/reservation/RiskReservationServiceTest.java b/src/test/java/com/sportsbook/risk/reservation/RiskReservationServiceTest.java
index 59fd513..95b79a8 100644
--- a/src/test/java/com/sportsbook/risk/reservation/RiskReservationServiceTest.java
+++ b/src/test/java/com/sportsbook/risk/reservation/RiskReservationServiceTest.java
@@ -10,6 +10,7 @@ import com.sportsbook.protocol.value.Money;
 import com.sportsbook.protocol.value.SelectionId;
 import com.sportsbook.protocol.value.UserId;
 import com.sportsbook.risk.service.RiskCheckCommand;
+import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
 import java.time.Instant;
 import java.util.List;
 import java.util.UUID;
@@ -22,7 +23,8 @@ class RiskReservationServiceTest {
   @Test
   void delegatesTypedReservationOperations() {
     RiskReservationStore store = mock(RiskReservationStore.class);
-    RiskReservationService service = new RiskReservationService(store);
+    SimpleMeterRegistry meters = new SimpleMeterRegistry();
+    RiskReservationService service = new RiskReservationService(store, meters);
     RiskCheckCommand command = command();
     ReservationDecision decision =
         ReservationDecision.approved(
@@ -34,6 +36,31 @@ class RiskReservationServiceTest {
     assertThat(service.reserve(command)).isSameAs(decision);
     assertThat(service.commit(BET, "a".repeat(64), NOW)).isEqualTo(ReservationTransition.APPLIED);
     assertThat(service.release(BET, NOW)).isEqualTo(ReservationTransition.REPLAYED);
+    assertThat(meters.counter("risk.reservation.requests", "result", "created").count())
+        .isEqualTo(1.0);
+  }
+
+  @Test
+  void recordsOnlyBoundedDecisionResults() {
+    RiskReservationStore store = mock(RiskReservationStore.class);
+    SimpleMeterRegistry meters = new SimpleMeterRegistry();
+    RiskReservationService service = new RiskReservationService(store, meters);
+    RiskCheckCommand command = command();
+    when(store.reserve(command))
+        .thenReturn(
+            ReservationDecision.rejected("STAKE_DAILY_LIMIT_EXCEEDED", false, List.of()),
+            ReservationDecision.approved(
+                ReservationState.RESERVED, NOW.plusSeconds(60), "a".repeat(64), true, List.of()),
+            ReservationDecision.conflict());
+
+    service.reserve(command);
+    service.reserve(command);
+    service.reserve(command);
+
+    for (String result : List.of("rejected", "replayed", "conflict")) {
+      assertThat(meters.counter("risk.reservation.requests", "result", result).count())
+          .isEqualTo(1.0);
+    }
   }
 
   private static RiskCheckCommand command() {


## `feat(metrics): count reservation transitions`

diff --git a/src/main/java/com/sportsbook/risk/reservation/RiskReservationService.java b/src/main/java/com/sportsbook/risk/reservation/RiskReservationService.java
index b8fa433..ea57175 100644
--- a/src/main/java/com/sportsbook/risk/reservation/RiskReservationService.java
+++ b/src/main/java/com/sportsbook/risk/reservation/RiskReservationService.java
@@ -5,6 +5,7 @@ import com.sportsbook.risk.service.RiskCheckCommand;
 import io.micrometer.core.instrument.MeterRegistry;
 import io.micrometer.core.instrument.Metrics;
 import java.time.Instant;
+import java.util.Locale;
 import java.util.Objects;
 import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.stereotype.Service;
@@ -32,15 +33,20 @@ public final class RiskReservationService {
   }
 
   public ReservationTransition commit(BetId betId, String token, Instant now) {
-    return store.commit(
-        Objects.requireNonNull(betId, "betId"),
-        Objects.requireNonNull(token, "token"),
-        Objects.requireNonNull(now, "now"));
+    ReservationTransition transition =
+        store.commit(
+            Objects.requireNonNull(betId, "betId"),
+            Objects.requireNonNull(token, "token"),
+            Objects.requireNonNull(now, "now"));
+    recordTransition("commit", transition);
+    return transition;
   }
 
   public ReservationTransition release(BetId betId, Instant now) {
-    return store.release(
-        Objects.requireNonNull(betId, "betId"), Objects.requireNonNull(now, "now"));
+    ReservationTransition transition =
+        store.release(Objects.requireNonNull(betId, "betId"), Objects.requireNonNull(now, "now"));
+    recordTransition("release", transition);
+    return transition;
   }
 
   private static String decisionResult(ReservationDecision decision) {
@@ -53,4 +59,15 @@ public final class RiskReservationService {
       case CONFLICT -> "conflict";
     };
   }
+
+  private void recordTransition(String operation, ReservationTransition transition) {
+    meters
+        .counter(
+            "risk.reservation.transitions",
+            "operation",
+            operation,
+            "result",
+            transition.name().toLowerCase(Locale.ROOT))
+        .increment();
+  }
 }


## `test(metrics): verify reservation transition metrics`

diff --git a/src/test/java/com/sportsbook/risk/reservation/RiskReservationServiceTest.java b/src/test/java/com/sportsbook/risk/reservation/RiskReservationServiceTest.java
index 95b79a8..9e80b3d 100644
--- a/src/test/java/com/sportsbook/risk/reservation/RiskReservationServiceTest.java
+++ b/src/test/java/com/sportsbook/risk/reservation/RiskReservationServiceTest.java
@@ -13,6 +13,7 @@ import com.sportsbook.risk.service.RiskCheckCommand;
 import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
 import java.time.Instant;
 import java.util.List;
+import java.util.Locale;
 import java.util.UUID;
 import org.junit.jupiter.api.Test;
 
@@ -63,6 +64,34 @@ class RiskReservationServiceTest {
     }
   }
 
+  @Test
+  void recordsOnlyBoundedTransitionResults() {
+    RiskReservationStore store = mock(RiskReservationStore.class);
+    SimpleMeterRegistry meters = new SimpleMeterRegistry();
+    RiskReservationService service = new RiskReservationService(store, meters);
+    String token = "a".repeat(64);
+
+    for (ReservationTransition transition : ReservationTransition.values()) {
+      when(store.commit(BET, token, NOW)).thenReturn(transition);
+      when(store.release(BET, NOW)).thenReturn(transition);
+
+      assertThat(service.commit(BET, token, NOW)).isEqualTo(transition);
+      assertThat(service.release(BET, NOW)).isEqualTo(transition);
+
+      String result = transition.name().toLowerCase(Locale.ROOT);
+      assertThat(
+              meters
+                  .counter("risk.reservation.transitions", "operation", "commit", "result", result)
+                  .count())
+          .isEqualTo(1.0);
+      assertThat(
+              meters
+                  .counter("risk.reservation.transitions", "operation", "release", "result", result)
+                  .count())
+          .isEqualTo(1.0);
+    }
+  }
+
   private static RiskCheckCommand command() {
     return new RiskCheckCommand(
         UserId.of(new UUID(0, 1)),


## `feat(metrics): count accepted bet reconciliation`

diff --git a/src/main/java/com/sportsbook/risk/event/BetPlacedConsumer.java b/src/main/java/com/sportsbook/risk/event/BetPlacedConsumer.java
index 3ef0783..6f9f78c 100644
--- a/src/main/java/com/sportsbook/risk/event/BetPlacedConsumer.java
+++ b/src/main/java/com/sportsbook/risk/event/BetPlacedConsumer.java
@@ -1,6 +1,9 @@
 package com.sportsbook.risk.event;
 
+import io.micrometer.core.instrument.MeterRegistry;
+import io.micrometer.core.instrument.Metrics;
 import java.time.Clock;
+import java.util.Locale;
 import java.util.Objects;
 import java.util.function.Supplier;
 import org.springframework.beans.factory.ObjectProvider;
@@ -18,25 +21,47 @@ public final class BetPlacedConsumer {
   private final Supplier<AcceptedBetReconciler> reconciler;
   private final BetPlacedDeadLetterPublisher deadLetters;
   private final Clock clock;
+  private final MeterRegistry meters;
 
   @Autowired
   public BetPlacedConsumer(
+      ObjectProvider<AcceptedBetReconciler> reconciler,
+      BetPlacedDeadLetterPublisher deadLetters,
+      MeterRegistry meters) {
+    this(
+        (Supplier<AcceptedBetReconciler>) reconciler::getIfUnique,
+        deadLetters,
+        Clock.systemUTC(),
+        meters);
+  }
+
+  BetPlacedConsumer(
       ObjectProvider<AcceptedBetReconciler> reconciler, BetPlacedDeadLetterPublisher deadLetters) {
-    this((Supplier<AcceptedBetReconciler>) reconciler::getIfUnique, deadLetters, Clock.systemUTC());
+    this(reconciler, deadLetters, Metrics.globalRegistry);
   }
 
   BetPlacedConsumer(
       AcceptedBetReconciler reconciler, BetPlacedDeadLetterPublisher deadLetters, Clock clock) {
-    this(() -> Objects.requireNonNull(reconciler, "reconciler"), deadLetters, clock);
+    this(reconciler, deadLetters, clock, Metrics.globalRegistry);
+  }
+
+  BetPlacedConsumer(
+      AcceptedBetReconciler reconciler,
+      BetPlacedDeadLetterPublisher deadLetters,
+      Clock clock,
+      MeterRegistry meters) {
+    this(() -> Objects.requireNonNull(reconciler, "reconciler"), deadLetters, clock, meters);
   }
 
   private BetPlacedConsumer(
       Supplier<AcceptedBetReconciler> reconciler,
       BetPlacedDeadLetterPublisher deadLetters,
-      Clock clock) {
+      Clock clock,
+      MeterRegistry meters) {
     this.reconciler = Objects.requireNonNull(reconciler, "reconciler");
     this.deadLetters = Objects.requireNonNull(deadLetters, "deadLetters");
     this.clock = Objects.requireNonNull(clock, "clock");
+    this.meters = Objects.requireNonNull(meters, "meters");
   }
 
   @KafkaListener(
@@ -59,9 +84,11 @@ public final class BetPlacedConsumer {
         Objects.requireNonNull(requiredReconciler().reconcile(envelope), "reconciliation result");
     if (result.permanentFailure()) {
       deadLetter(key, payload, result.failureReason(), acknowledgment);
+      record(result);
       return;
     }
     acknowledgment.acknowledge();
+    record(result);
   }
 
   private AcceptedBetReconciler requiredReconciler() {
@@ -77,4 +104,10 @@ public final class BetPlacedConsumer {
     deadLetters.publishAndAwait(key, payload, reason);
     acknowledgment.acknowledge();
   }
+
+  private void record(AcceptedBetReconciliation result) {
+    meters
+        .counter("risk.bet.placed.reconciliation", "result", result.name().toLowerCase(Locale.ROOT))
+        .increment();
+  }
 }


## `test(metrics): verify accepted reconciliation metrics`

diff --git a/src/test/java/com/sportsbook/risk/event/BetPlacedConsumerReconciliationTest.java b/src/test/java/com/sportsbook/risk/event/BetPlacedConsumerReconciliationTest.java
index bdd2926..ec6b22b 100644
--- a/src/test/java/com/sportsbook/risk/event/BetPlacedConsumerReconciliationTest.java
+++ b/src/test/java/com/sportsbook/risk/event/BetPlacedConsumerReconciliationTest.java
@@ -9,13 +9,16 @@ import static org.mockito.Mockito.never;
 import static org.mockito.Mockito.verify;
 import static org.mockito.Mockito.when;
 
+import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
 import java.time.Clock;
 import java.time.ZoneOffset;
+import java.util.Locale;
 import java.util.stream.Stream;
 import org.junit.jupiter.api.BeforeEach;
 import org.junit.jupiter.api.Test;
 import org.junit.jupiter.params.ParameterizedTest;
 import org.junit.jupiter.params.provider.Arguments;
+import org.junit.jupiter.params.provider.EnumSource;
 import org.junit.jupiter.params.provider.MethodSource;
 import org.mockito.ArgumentCaptor;
 import org.mockito.InOrder;
@@ -25,17 +28,21 @@ class BetPlacedConsumerReconciliationTest {
   private final AcceptedBetReconciler reconciler = mock(AcceptedBetReconciler.class);
   private final BetPlacedDeadLetterPublisher deadLetters = mock(BetPlacedDeadLetterPublisher.class);
   private final Acknowledgment acknowledgment = mock(Acknowledgment.class);
+  private final SimpleMeterRegistry meters = new SimpleMeterRegistry();
   private BetPlacedConsumer consumer;
 
   @BeforeEach
   void setUp() {
     Clock clock = Clock.fixed(BetPlacedEventFixture.OBSERVED_AT, ZoneOffset.UTC);
-    consumer = new BetPlacedConsumer(reconciler, deadLetters, clock);
+    consumer = new BetPlacedConsumer(reconciler, deadLetters, clock, meters);
   }
 
-  @Test
-  void successfulReconciliationAcknowledgesTheSourceEvent() {
-    when(reconciler.reconcile(any())).thenReturn(AcceptedBetReconciliation.CONFIRMED);
+  @ParameterizedTest
+  @EnumSource(
+      value = AcceptedBetReconciliation.class,
+      names = {"CONFIRMED", "PROJECTED", "REPLAYED"})
+  void successfulReconciliationAcknowledgesTheSourceEvent(AcceptedBetReconciliation result) {
+    when(reconciler.reconcile(any())).thenReturn(result);
 
     consumer.onBetPlaced(
         BetPlacedEventFixture.payload(), BetPlacedEventFixture.USER_ID, acknowledgment);
@@ -46,6 +53,7 @@ class BetPlacedConsumerReconciliationTest {
     assertThat(envelope.getValue().command().now()).isEqualTo(BetPlacedEventFixture.OBSERVED_AT);
     verify(deadLetters, never()).publishAndAwait(any(), any(), any());
     verify(acknowledgment).acknowledge();
+    assertThat(reconciliationCount(result)).isEqualTo(1.0);
   }
 
   @ParameterizedTest
@@ -60,6 +68,7 @@ class BetPlacedConsumerReconciliationTest {
     InOrder order = inOrder(deadLetters, acknowledgment);
     order.verify(deadLetters).publishAndAwait(BetPlacedEventFixture.USER_ID, payload, reason);
     order.verify(acknowledgment).acknowledge();
+    assertThat(reconciliationCount(result)).isEqualTo(1.0);
   }
 
   @Test
@@ -75,6 +84,13 @@ class BetPlacedConsumerReconciliationTest {
 
     verify(deadLetters, never()).publishAndAwait(any(), any(), any());
     verify(acknowledgment, never()).acknowledge();
+    assertThat(meters.find("risk.bet.placed.reconciliation").counters()).isEmpty();
+  }
+
+  private double reconciliationCount(AcceptedBetReconciliation result) {
+    return meters
+        .counter("risk.bet.placed.reconciliation", "result", result.name().toLowerCase(Locale.ROOT))
+        .count();
   }
 
   private static Stream<Arguments> permanentResults() {


## `feat(metrics): measure reservation script latency`

diff --git a/src/main/java/com/sportsbook/risk/reservation/RedisRiskReservationStore.java b/src/main/java/com/sportsbook/risk/reservation/RedisRiskReservationStore.java
index eec0d7b..961fde6 100644
--- a/src/main/java/com/sportsbook/risk/reservation/RedisRiskReservationStore.java
+++ b/src/main/java/com/sportsbook/risk/reservation/RedisRiskReservationStore.java
@@ -6,9 +6,13 @@ import com.sportsbook.risk.pattern.RiskHistoryProperties;
 import com.sportsbook.risk.policy.RiskLimitProperties;
 import com.sportsbook.risk.policy.RiskPatternProperties;
 import com.sportsbook.risk.service.RiskCheckCommand;
+import io.micrometer.core.instrument.MeterRegistry;
+import io.micrometer.core.instrument.Metrics;
+import io.micrometer.core.instrument.Timer;
 import java.time.Instant;
 import java.util.List;
 import java.util.Objects;
+import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.data.redis.core.StringRedisTemplate;
 import org.springframework.data.redis.core.script.RedisScript;
 import org.springframework.stereotype.Component;
@@ -31,50 +35,76 @@ public final class RedisRiskReservationStore implements RiskReservationStore {
   private final RiskReservationProperties reservations;
   private final RiskHistoryProperties history;
   private final ReservationWireMapper mapper;
+  private final Timer reserveLatency;
+  private final Timer commitLatency;
+  private final Timer acceptedProjectionLatency;
+  private final Timer releaseLatency;
 
+  @Autowired
   public RedisRiskReservationStore(
       StringRedisTemplate redis,
       RiskLimitProperties limits,
       RiskPatternProperties patterns,
       RiskReservationProperties reservations,
       RiskHistoryProperties history,
-      ReservationWireMapper mapper) {
+      ReservationWireMapper mapper,
+      MeterRegistry meters) {
     this.redis = Objects.requireNonNull(redis, "redis");
     this.limits = Objects.requireNonNull(limits, "limits");
     this.patterns = Objects.requireNonNull(patterns, "patterns");
     this.reservations = Objects.requireNonNull(reservations, "reservations");
     this.history = Objects.requireNonNull(history, "history");
     this.mapper = Objects.requireNonNull(mapper, "mapper");
+    this.reserveLatency = timer(meters, "reserve");
+    this.commitLatency = timer(meters, "commit");
+    this.acceptedProjectionLatency = timer(meters, "project-accepted");
+    this.releaseLatency = timer(meters, "release");
+  }
+
+  RedisRiskReservationStore(
+      StringRedisTemplate redis,
+      RiskLimitProperties limits,
+      RiskPatternProperties patterns,
+      RiskReservationProperties reservations,
+      RiskHistoryProperties history,
+      ReservationWireMapper mapper) {
+    this(redis, limits, patterns, reservations, history, mapper, Metrics.globalRegistry);
   }
 
   @Override
   public ReservationDecision reserve(RiskCheckCommand command) {
     ReservationScriptRequest request =
         ReservationScriptRequest.from(command, limits, patterns, reservations, history);
-    String raw = redis.execute(RESERVE, request.keys(), request.arguments().toArray());
-    return mapper.map(raw).decision();
+    return reserveLatency
+        .record(
+            () -> mapper.map(redis.execute(RESERVE, request.keys(), request.arguments().toArray())))
+        .decision();
   }
 
   @Override
   public ReservationTransition commit(BetId betId, String token, Instant now) {
     ReservationTransitionRequest request =
         ReservationTransitionRequest.commit(betId, token, now, reservations, patterns, history);
-    return executeTransition(COMMIT, request.keys(), request.arguments(), "commit");
+    return commitLatency.record(
+        () -> executeTransition(COMMIT, request.keys(), request.arguments(), "commit"));
   }
 
   @Override
   public ReservationTransition projectAccepted(RiskCheckCommand command, String fingerprint) {
     AcceptedProjectionRequest request =
         AcceptedProjectionRequest.from(command, fingerprint, reservations, patterns, history);
-    return executeTransition(
-        PROJECT_ACCEPTED, request.keys(), request.arguments(), "accepted projection");
+    return acceptedProjectionLatency.record(
+        () ->
+            executeTransition(
+                PROJECT_ACCEPTED, request.keys(), request.arguments(), "accepted projection"));
   }
 
   @Override
   public ReservationTransition release(BetId betId, Instant now) {
     ReservationTransitionRequest request =
         ReservationTransitionRequest.release(betId, now, reservations);
-    return executeTransition(RELEASE, request.keys(), request.arguments(), "release");
+    return releaseLatency.record(
+        () -> executeTransition(RELEASE, request.keys(), request.arguments(), "release"));
   }
 
   private ReservationTransition executeTransition(
@@ -90,4 +120,10 @@ public final class RedisRiskReservationStore implements RiskReservationStore {
           "Redis " + operation + " script returned unknown result", failure);
     }
   }
+
+  private static Timer timer(MeterRegistry meters, String operation) {
+    return Timer.builder("risk.reservation.lua.latency")
+        .tag("operation", operation)
+        .register(Objects.requireNonNull(meters, "meters"));
+  }
 }


