## `test(metrics): verify reservation script timers`

diff --git a/src/test/java/com/sportsbook/risk/reservation/RedisRiskReservationStoreTest.java b/src/test/java/com/sportsbook/risk/reservation/RedisRiskReservationStoreTest.java
index 5cbf4e0..2cc071f 100644
--- a/src/test/java/com/sportsbook/risk/reservation/RedisRiskReservationStoreTest.java
+++ b/src/test/java/com/sportsbook/risk/reservation/RedisRiskReservationStoreTest.java
@@ -13,6 +13,7 @@ import com.sportsbook.risk.policy.RiskLimitProperties;
 import com.sportsbook.risk.policy.RiskPatternProperties;
 import com.sportsbook.risk.service.RiskCheckCommand;
 import com.sportsbook.risk.support.RedisTestSupport;
+import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
 import java.time.Instant;
 import java.util.List;
 import java.util.UUID;
@@ -23,7 +24,8 @@ class RedisRiskReservationStoreTest extends RedisTestSupport {
 
   @Test
   void executesAdmissionCommitAndReleaseThroughTheTypedPort() {
-    RiskReservationStore store = store();
+    SimpleMeterRegistry meters = new SimpleMeterRegistry();
+    RiskReservationStore store = store(meters);
     RiskCheckCommand committed = command(1);
     ReservationDecision reserved = store.reserve(committed);
 
@@ -35,11 +37,15 @@ class RedisRiskReservationStoreTest extends RedisTestSupport {
     assertThat(store.reserve(released).approved()).isTrue();
     assertThat(store.release(released.betId(), NOW.plusMillis(1)))
         .isEqualTo(ReservationTransition.APPLIED);
+    assertThat(timerCount(meters, "reserve")).isEqualTo(2L);
+    assertThat(timerCount(meters, "commit")).isEqualTo(1L);
+    assertThat(timerCount(meters, "release")).isEqualTo(1L);
   }
 
   @Test
   void executesAcceptedProjectionThroughTheTypedPort() {
-    RiskReservationStore store = store();
+    SimpleMeterRegistry meters = new SimpleMeterRegistry();
+    RiskReservationStore store = store(meters);
     RiskCheckCommand accepted = command(3);
     String fingerprint = ReservationFingerprint.of(accepted);
 
@@ -47,16 +53,22 @@ class RedisRiskReservationStoreTest extends RedisTestSupport {
         .isEqualTo(ReservationTransition.APPLIED);
     assertThat(store.projectAccepted(accepted, fingerprint))
         .isEqualTo(ReservationTransition.REPLAYED);
+    assertThat(timerCount(meters, "project-accepted")).isEqualTo(2L);
   }
 
-  private RiskReservationStore store() {
+  private RiskReservationStore store(SimpleMeterRegistry meters) {
     return new RedisRiskReservationStore(
         redis,
         new RiskLimitProperties(null, null, null, null, 100),
         new RiskPatternProperties(null, null, null),
         new RiskReservationProperties(null, null),
         new RiskHistoryProperties(null, 0),
-        new ReservationWireMapper(new ObjectMapper()));
+        new ReservationWireMapper(new ObjectMapper()),
+        meters);
+  }
+
+  private static long timerCount(SimpleMeterRegistry meters, String operation) {
+    return meters.timer("risk.reservation.lua.latency", "operation", operation).count();
   }
 
   private static RiskCheckCommand command(long value) {


## `feat(metrics): count expired reservation cleanup`

diff --git a/src/main/java/com/sportsbook/risk/reservation/RedisRiskReservationStore.java b/src/main/java/com/sportsbook/risk/reservation/RedisRiskReservationStore.java
index 961fde6..d562ea9 100644
--- a/src/main/java/com/sportsbook/risk/reservation/RedisRiskReservationStore.java
+++ b/src/main/java/com/sportsbook/risk/reservation/RedisRiskReservationStore.java
@@ -6,6 +6,7 @@ import com.sportsbook.risk.pattern.RiskHistoryProperties;
 import com.sportsbook.risk.policy.RiskLimitProperties;
 import com.sportsbook.risk.policy.RiskPatternProperties;
 import com.sportsbook.risk.service.RiskCheckCommand;
+import io.micrometer.core.instrument.Counter;
 import io.micrometer.core.instrument.MeterRegistry;
 import io.micrometer.core.instrument.Metrics;
 import io.micrometer.core.instrument.Timer;
@@ -39,6 +40,7 @@ public final class RedisRiskReservationStore implements RiskReservationStore {
   private final Timer commitLatency;
   private final Timer acceptedProjectionLatency;
   private final Timer releaseLatency;
+  private final Counter expirations;
 
   @Autowired
   public RedisRiskReservationStore(
@@ -59,6 +61,7 @@ public final class RedisRiskReservationStore implements RiskReservationStore {
     this.commitLatency = timer(meters, "commit");
     this.acceptedProjectionLatency = timer(meters, "project-accepted");
     this.releaseLatency = timer(meters, "release");
+    this.expirations = Counter.builder("risk.reservation.expirations").register(meters);
   }
 
   RedisRiskReservationStore(
@@ -75,10 +78,12 @@ public final class RedisRiskReservationStore implements RiskReservationStore {
   public ReservationDecision reserve(RiskCheckCommand command) {
     ReservationScriptRequest request =
         ReservationScriptRequest.from(command, limits, patterns, reservations, history);
-    return reserveLatency
-        .record(
-            () -> mapper.map(redis.execute(RESERVE, request.keys(), request.arguments().toArray())))
-        .decision();
+    ReservationWireMapper.Decoded decoded =
+        reserveLatency.record(
+            () ->
+                mapper.map(redis.execute(RESERVE, request.keys(), request.arguments().toArray())));
+    expirations.increment(decoded.expired());
+    return decoded.decision();
   }
 
   @Override


## `test(metrics): verify reservation expiry metrics`

diff --git a/src/test/java/com/sportsbook/risk/reservation/RedisRiskReservationStoreTest.java b/src/test/java/com/sportsbook/risk/reservation/RedisRiskReservationStoreTest.java
index 2cc071f..fa93f88 100644
--- a/src/test/java/com/sportsbook/risk/reservation/RedisRiskReservationStoreTest.java
+++ b/src/test/java/com/sportsbook/risk/reservation/RedisRiskReservationStoreTest.java
@@ -56,6 +56,17 @@ class RedisRiskReservationStoreTest extends RedisTestSupport {
     assertThat(timerCount(meters, "project-accepted")).isEqualTo(2L);
   }
 
+  @Test
+  void countsLeasesRemovedDuringAdmission() {
+    SimpleMeterRegistry meters = new SimpleMeterRegistry();
+    RiskReservationStore store = store(meters);
+    store.reserve(command(1));
+
+    store.reserve(command(2, NOW.plus(RiskReservationProperties.DEFAULT_LEASE).plusMillis(1)));
+
+    assertThat(meters.counter("risk.reservation.expirations").count()).isEqualTo(1.0);
+  }
+
   private RiskReservationStore store(SimpleMeterRegistry meters) {
     return new RedisRiskReservationStore(
         redis,
@@ -72,11 +83,15 @@ class RedisRiskReservationStoreTest extends RedisTestSupport {
   }
 
   private static RiskCheckCommand command(long value) {
+    return command(value, NOW);
+  }
+
+  private static RiskCheckCommand command(long value, Instant now) {
     return new RiskCheckCommand(
         UserId.of(new UUID(0, 1)),
         BetId.of(new UUID(0, value)),
         new Money(10, Currency.KRW),
         List.of(SelectionId.of(new UUID(0, value + 10))),
-        NOW);
+        now);
   }
 }


## `feat(readiness): require Redis and Kafka`

diff --git a/src/main/java/com/sportsbook/risk/event/KafkaHealthIndicator.java b/src/main/java/com/sportsbook/risk/event/KafkaHealthIndicator.java
new file mode 100644
index 0000000..670e78b
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/event/KafkaHealthIndicator.java
@@ -0,0 +1,43 @@
+package com.sportsbook.risk.event;
+
+import java.time.Duration;
+import java.util.Objects;
+import java.util.concurrent.TimeUnit;
+import java.util.function.Supplier;
+import org.apache.kafka.clients.admin.AdminClient;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.actuate.health.Health;
+import org.springframework.boot.actuate.health.HealthIndicator;
+import org.springframework.kafka.core.KafkaAdmin;
+import org.springframework.stereotype.Component;
+
+/** Reports whether Kafka can answer cluster metadata within the readiness budget. */
+@Component
+public final class KafkaHealthIndicator implements HealthIndicator {
+  private static final Duration TIMEOUT = Duration.ofSeconds(2);
+
+  private final Supplier<AdminClient> adminClients;
+
+  @Autowired
+  public KafkaHealthIndicator(KafkaAdmin kafkaAdmin) {
+    Objects.requireNonNull(kafkaAdmin, "kafkaAdmin");
+    this.adminClients = () -> AdminClient.create(kafkaAdmin.getConfigurationProperties());
+  }
+
+  KafkaHealthIndicator(Supplier<AdminClient> adminClients) {
+    this.adminClients = Objects.requireNonNull(adminClients, "adminClients");
+  }
+
+  @Override
+  public Health health() {
+    try (AdminClient admin = adminClients.get()) {
+      admin.describeCluster().clusterId().get(TIMEOUT.toMillis(), TimeUnit.MILLISECONDS);
+      return Health.up().build();
+    } catch (InterruptedException failure) {
+      Thread.currentThread().interrupt();
+      return Health.down(failure).build();
+    } catch (Exception failure) {
+      return Health.down(failure).build();
+    }
+  }
+}
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index aa4e9af..dcfd4b3 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -77,6 +77,9 @@ management:
       probes:
         enabled: true
       show-details: never
+      group:
+        readiness:
+          include: readinessState,redis,kafka
   health:
     redis:
       enabled: true


## `test(readiness): verify dependency health contracts`

diff --git a/src/test/java/com/sportsbook/risk/DeployedRiskConfigurationTest.java b/src/test/java/com/sportsbook/risk/DeployedRiskConfigurationTest.java
index 1370f38..fea206a 100644
--- a/src/test/java/com/sportsbook/risk/DeployedRiskConfigurationTest.java
+++ b/src/test/java/com/sportsbook/risk/DeployedRiskConfigurationTest.java
@@ -39,6 +39,8 @@ class DeployedRiskConfigurationTest {
     assertThat(patterns.repeatedSelection().action()).isEqualTo(PatternAction.REVIEW);
     assertThat(reservations.retention()).isEqualTo(Duration.ofDays(32));
     assertThat(history.maxStakeSamples()).isEqualTo(100);
+    assertThat(source.getProperty("management.endpoint.health.group.readiness.include"))
+        .isEqualTo("readinessState,redis,kafka");
   }
 
   private static <T> T bind(Binder binder, String prefix, Class<T> type) {
diff --git a/src/test/java/com/sportsbook/risk/auth/InternalSecurityIntegrationTest.java b/src/test/java/com/sportsbook/risk/auth/InternalSecurityIntegrationTest.java
index 28b3cc1..cacaa85 100644
--- a/src/test/java/com/sportsbook/risk/auth/InternalSecurityIntegrationTest.java
+++ b/src/test/java/com/sportsbook/risk/auth/InternalSecurityIntegrationTest.java
@@ -21,7 +21,8 @@ import org.springframework.test.web.servlet.request.MockHttpServletRequestBuilde
       "risk.auth.betting-service-api-key=bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
       "risk.auth.admin-api-key=aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
       "risk.auth.platform-api-key=pppppppppppppppppppppppppppppppp",
-      "management.health.redis.enabled=false"
+      "management.health.redis.enabled=false",
+      "management.endpoint.health.validate-group-membership=false"
     })
 @AutoConfigureMockMvc
 class InternalSecurityIntegrationTest {
diff --git a/src/test/java/com/sportsbook/risk/event/KafkaHealthIndicatorTest.java b/src/test/java/com/sportsbook/risk/event/KafkaHealthIndicatorTest.java
new file mode 100644
index 0000000..6b9cd0e
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/event/KafkaHealthIndicatorTest.java
@@ -0,0 +1,64 @@
+package com.sportsbook.risk.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+
+import java.util.concurrent.ExecutionException;
+import java.util.concurrent.TimeUnit;
+import org.apache.kafka.clients.admin.AdminClient;
+import org.apache.kafka.clients.admin.DescribeClusterResult;
+import org.apache.kafka.common.KafkaFuture;
+import org.junit.jupiter.api.AfterEach;
+import org.junit.jupiter.api.Test;
+import org.springframework.boot.actuate.health.Status;
+
+class KafkaHealthIndicatorTest {
+  @AfterEach
+  void clearInterrupt() {
+    Thread.interrupted();
+  }
+
+  @Test
+  void reportsUpAfterBrokerMetadataConfirmation() throws Exception {
+    Fixture fixture = fixture();
+    when(fixture.clusterId().get(2_000, TimeUnit.MILLISECONDS)).thenReturn("cluster");
+
+    assertThat(fixture.indicator().health().getStatus()).isEqualTo(Status.UP);
+    verify(fixture.admin()).close();
+  }
+
+  @Test
+  void reportsDownWhenBrokerMetadataFails() throws Exception {
+    Fixture fixture = fixture();
+    when(fixture.clusterId().get(2_000, TimeUnit.MILLISECONDS))
+        .thenThrow(new ExecutionException(new IllegalStateException("unavailable")));
+
+    assertThat(fixture.indicator().health().getStatus()).isEqualTo(Status.DOWN);
+    verify(fixture.admin()).close();
+  }
+
+  @Test
+  void restoresInterruptionBeforeReportingDown() throws Exception {
+    Fixture fixture = fixture();
+    when(fixture.clusterId().get(2_000, TimeUnit.MILLISECONDS))
+        .thenThrow(new InterruptedException());
+
+    assertThat(fixture.indicator().health().getStatus()).isEqualTo(Status.DOWN);
+    assertThat(Thread.currentThread().isInterrupted()).isTrue();
+  }
+
+  private static Fixture fixture() {
+    AdminClient admin = mock(AdminClient.class);
+    DescribeClusterResult cluster = mock(DescribeClusterResult.class);
+    @SuppressWarnings("unchecked")
+    KafkaFuture<String> clusterId = mock(KafkaFuture.class);
+    when(admin.describeCluster()).thenReturn(cluster);
+    when(cluster.clusterId()).thenReturn(clusterId);
+    return new Fixture(new KafkaHealthIndicator(() -> admin), admin, clusterId);
+  }
+
+  private record Fixture(
+      KafkaHealthIndicator indicator, AdminClient admin, KafkaFuture<String> clusterId) {}
+}
