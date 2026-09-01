# 운영 가시성과 준비 상태

## `build(observability): enable Prometheus registry`

diff --git a/pom.xml b/pom.xml
index c897244..4dc2e1d 100644
--- a/pom.xml
+++ b/pom.xml
@@ -56,6 +56,10 @@
             <groupId>org.springframework.boot</groupId>
             <artifactId>spring-boot-starter-actuator</artifactId>
         </dependency>
+        <dependency>
+            <groupId>io.micrometer</groupId>
+            <artifactId>micrometer-registry-prometheus</artifactId>
+        </dependency>
         <dependency>
             <groupId>org.springframework.boot</groupId>
             <artifactId>spring-boot-starter-web</artifactId>
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index c95830a..21e00e6 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -22,6 +22,10 @@ spring:
       ack-mode: manual_immediate
 
 management:
+  prometheus:
+    metrics:
+      export:
+        enabled: true
   endpoint:
     health:
       probes:


## `feat(observability): define bounded operation metrics`

diff --git a/src/main/java/com/sportsbook/settlement/observability/SettlementMetrics.java b/src/main/java/com/sportsbook/settlement/observability/SettlementMetrics.java
new file mode 100644
index 0000000..cc4b52e
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/observability/SettlementMetrics.java
@@ -0,0 +1,47 @@
+package com.sportsbook.settlement.observability;
+
+import io.micrometer.core.instrument.MeterRegistry;
+import io.micrometer.core.instrument.Timer;
+import java.util.Objects;
+import java.util.regex.Pattern;
+import org.springframework.stereotype.Component;
+
+@Component
+public final class SettlementMetrics {
+
+  public static final String OPERATIONS = "settlement.operations";
+  public static final String DURATION = "settlement.operation.duration";
+  private static final Pattern LABEL = Pattern.compile("[a-z0-9_]{1,32}");
+
+  private final MeterRegistry registry;
+
+  public SettlementMetrics(MeterRegistry registry) {
+    this.registry = registry;
+  }
+
+  public Timer.Sample start() {
+    return Timer.start(registry);
+  }
+
+  public void count(String flow, String outcome) {
+    count(flow, outcome, 1);
+  }
+
+  public void count(String flow, String outcome, long amount) {
+    if (amount < 0) {
+      throw new IllegalArgumentException("Metric amount must be nonnegative");
+    }
+    registry.counter(OPERATIONS, "flow", label(flow), "outcome", label(outcome)).increment(amount);
+  }
+
+  public void stop(Timer.Sample sample, String flow) {
+    Objects.requireNonNull(sample, "sample").stop(registry.timer(DURATION, "flow", label(flow)));
+  }
+
+  private static String label(String value) {
+    if (value == null || !LABEL.matcher(value).matches()) {
+      throw new IllegalArgumentException("Metric labels must be bounded constants");
+    }
+    return value;
+  }
+}


## `feat(observability): measure base result fanout`

diff --git a/src/main/java/com/sportsbook/settlement/result/ResultFanout.java b/src/main/java/com/sportsbook/settlement/result/ResultFanout.java
index 7b3d5f6..227acf0 100644
--- a/src/main/java/com/sportsbook/settlement/result/ResultFanout.java
+++ b/src/main/java/com/sportsbook/settlement/result/ResultFanout.java
@@ -2,6 +2,7 @@ package com.sportsbook.settlement.result;
 
 import com.sportsbook.settlement.execution.SettlementExecution;
 import com.sportsbook.settlement.execution.SettlementExecutionRunner;
+import com.sportsbook.settlement.observability.SettlementMetrics;
 import com.sportsbook.settlement.persistence.BetRepository;
 import java.util.ArrayList;
 import java.util.List;
@@ -13,19 +14,35 @@ public class ResultFanout {
   private final BetRepository bets;
   private final ResultSettlementPreparer preparer;
   private final SettlementExecutionRunner runner;
+  private final SettlementMetrics metrics;
 
   public ResultFanout(
-      BetRepository bets, ResultSettlementPreparer preparer, SettlementExecutionRunner runner) {
+      BetRepository bets,
+      ResultSettlementPreparer preparer,
+      SettlementExecutionRunner runner,
+      SettlementMetrics metrics) {
     this.bets = bets;
     this.preparer = preparer;
     this.runner = runner;
+    this.metrics = metrics;
   }
 
   public SettlementExecutionRunner.BatchResult fanOut(AcceptedResult accepted) {
-    List<SettlementExecution> executions = new ArrayList<>();
-    for (var betId : bets.findResultActionableIdsByEvent(accepted.eventId())) {
-      preparer.prepare(betId, accepted).ifPresent(executions::add);
+    var sample = metrics.start();
+    try {
+      List<SettlementExecution> executions = new ArrayList<>();
+      for (var betId : bets.findResultActionableIdsByEvent(accepted.eventId())) {
+        preparer.prepare(betId, accepted).ifPresent(executions::add);
+      }
+      var result = runner.fanOut(List.copyOf(executions));
+      metrics.count("base_result", "succeeded", result.succeeded());
+      metrics.count("base_result", "failed", result.failed());
+      return result;
+    } catch (RuntimeException failure) {
+      metrics.count("base_result", "failed");
+      throw failure;
+    } finally {
+      metrics.stop(sample, "base_result");
     }
-    return runner.fanOut(List.copyOf(executions));
   }
 }


## `feat(observability): measure lifecycle recovery`

diff --git a/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleTombstoneScanner.java b/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleTombstoneScanner.java
index e845b48..ca10f58 100644
--- a/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleTombstoneScanner.java
+++ b/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleTombstoneScanner.java
@@ -2,6 +2,7 @@ package com.sportsbook.settlement.lifecycle;
 
 import com.sportsbook.settlement.config.SettlementRuntimeProperties;
 import com.sportsbook.settlement.config.SettlementWorkerConfiguration;
+import com.sportsbook.settlement.observability.SettlementMetrics;
 import org.springframework.scheduling.annotation.Scheduled;
 import org.springframework.stereotype.Component;
 
@@ -11,12 +12,17 @@ public class LifecycleTombstoneScanner {
   private final LifecycleStore store;
   private final LifecycleFanout fanout;
   private final SettlementRuntimeProperties runtime;
+  private final SettlementMetrics metrics;
 
   public LifecycleTombstoneScanner(
-      LifecycleStore store, LifecycleFanout fanout, SettlementRuntimeProperties runtime) {
+      LifecycleStore store,
+      LifecycleFanout fanout,
+      SettlementRuntimeProperties runtime,
+      SettlementMetrics metrics) {
     this.store = store;
     this.fanout = fanout;
     this.runtime = runtime;
+    this.metrics = metrics;
   }
 
   @Scheduled(
@@ -24,8 +30,17 @@ public class LifecycleTombstoneScanner {
       initialDelayString = "${settlement.runtime.recovery-interval:PT1S}",
       scheduler = SettlementWorkerConfiguration.LIFECYCLE)
   public int scan() {
-    var tombstones = store.findActionableTombstones(runtime.batchSize());
-    tombstones.forEach(fanout::fanOut);
-    return tombstones.size();
+    var sample = metrics.start();
+    try {
+      var tombstones = store.findActionableTombstones(runtime.batchSize());
+      tombstones.forEach(fanout::fanOut);
+      metrics.count("lifecycle", "processed", tombstones.size());
+      return tombstones.size();
+    } catch (RuntimeException failure) {
+      metrics.count("lifecycle", "failed");
+      throw failure;
+    } finally {
+      metrics.stop(sample, "lifecycle");
+    }
   }
 }


## `feat(observability): measure revision recovery`

diff --git a/src/main/java/com/sportsbook/settlement/correction/RevisionRecoveryScanner.java b/src/main/java/com/sportsbook/settlement/correction/RevisionRecoveryScanner.java
index 52deda6..c995841 100644
--- a/src/main/java/com/sportsbook/settlement/correction/RevisionRecoveryScanner.java
+++ b/src/main/java/com/sportsbook/settlement/correction/RevisionRecoveryScanner.java
@@ -2,6 +2,7 @@ package com.sportsbook.settlement.correction;
 
 import com.sportsbook.settlement.config.SettlementRuntimeProperties;
 import com.sportsbook.settlement.config.SettlementWorkerConfiguration;
+import com.sportsbook.settlement.observability.SettlementMetrics;
 import java.util.ArrayList;
 import java.util.List;
 import org.springframework.scheduling.annotation.Scheduled;
@@ -14,16 +15,19 @@ public class RevisionRecoveryScanner {
   private final RevisionPlanReader plans;
   private final RevisionExecutionRunner runner;
   private final SettlementRuntimeProperties runtime;
+  private final SettlementMetrics metrics;
 
   public RevisionRecoveryScanner(
       RevisionRecoveryRepository recovery,
       RevisionPlanReader plans,
       RevisionExecutionRunner runner,
-      SettlementRuntimeProperties runtime) {
+      SettlementRuntimeProperties runtime,
+      SettlementMetrics metrics) {
     this.recovery = recovery;
     this.plans = plans;
     this.runner = runner;
     this.runtime = runtime;
+    this.metrics = metrics;
   }
 
   @Scheduled(
@@ -31,15 +35,25 @@ public class RevisionRecoveryScanner {
       initialDelayString = "${settlement.runtime.recovery-interval:PT1S}",
       scheduler = SettlementWorkerConfiguration.RECOVERY)
   public List<RevisionExecutionRunner.Result> recover() {
-    var claims = recovery.claimDue(runtime.leaseDuration(), runtime.batchSize());
-    List<RevisionExecutionRunner.Result> results = new ArrayList<>(claims.size());
-    for (var claim : claims) {
-      RevisionPlan plan =
-          plans
-              .find(claim.revisionId())
-              .orElseThrow(() -> new IllegalStateException("Claimed revision plan is missing"));
-      results.add(runner.execute(plan, claim.lease(), true, !claim.blockedProof()));
+    var sample = metrics.start();
+    try {
+      var claims = recovery.claimDue(runtime.leaseDuration(), runtime.batchSize());
+      List<RevisionExecutionRunner.Result> results = new ArrayList<>(claims.size());
+      for (var claim : claims) {
+        RevisionPlan plan =
+            plans
+                .find(claim.revisionId())
+                .orElseThrow(() -> new IllegalStateException("Claimed revision plan is missing"));
+        var result = runner.execute(plan, claim.lease(), true, !claim.blockedProof());
+        results.add(result);
+        metrics.count("revision", result.name().toLowerCase(java.util.Locale.ROOT));
+      }
+      return List.copyOf(results);
+    } catch (RuntimeException failure) {
+      metrics.count("revision", "failed");
+      throw failure;
+    } finally {
+      metrics.stop(sample, "revision");
     }
-    return List.copyOf(results);
   }
 }


## `feat(observability): meter outbox publication`

diff --git a/pom.xml b/pom.xml
index 04f8c98..50166a8 100644
--- a/pom.xml
+++ b/pom.xml
@@ -52,6 +52,10 @@
             <groupId>org.springframework.boot</groupId>
             <artifactId>spring-boot-starter-data-jpa</artifactId>
         </dependency>
+        <dependency>
+            <groupId>org.springframework.boot</groupId>
+            <artifactId>spring-boot-starter-actuator</artifactId>
+        </dependency>
         <dependency>
             <groupId>org.flywaydb</groupId>
             <artifactId>flyway-core</artifactId>
diff --git a/src/main/java/com/sportsbook/settlement/outbox/OutboxPublisher.java b/src/main/java/com/sportsbook/settlement/outbox/OutboxPublisher.java
index 1eceb19..5ca1409 100644
--- a/src/main/java/com/sportsbook/settlement/outbox/OutboxPublisher.java
+++ b/src/main/java/com/sportsbook/settlement/outbox/OutboxPublisher.java
@@ -3,6 +3,7 @@ package com.sportsbook.settlement.outbox;
 import com.sportsbook.settlement.config.RawKafkaProducerConfiguration;
 import com.sportsbook.settlement.config.SettlementRuntimeProperties;
 import com.sportsbook.settlement.config.SettlementWorkerConfiguration;
+import io.micrometer.core.instrument.MeterRegistry;
 import java.nio.charset.StandardCharsets;
 import java.time.Clock;
 import java.util.concurrent.ExecutionException;
@@ -21,21 +22,26 @@ import org.springframework.transaction.annotation.Transactional;
 public class OutboxPublisher {
 
   private static final long SEND_TIMEOUT_SECONDS = 11;
+  public static final String PUBLISHED_METRIC = "settlement.outbox.published";
+  public static final String FAILURE_METRIC = "settlement.outbox.publish.failures";
 
   private final OutboxEventRepository repository;
   private final KafkaOperations<byte[], byte[]> kafka;
   private final SettlementRuntimeProperties runtime;
   private final Clock clock;
+  private final MeterRegistry meters;
 
   public OutboxPublisher(
       OutboxEventRepository repository,
       @Qualifier(RawKafkaProducerConfiguration.OPERATIONS) KafkaOperations<byte[], byte[]> kafka,
       SettlementRuntimeProperties runtime,
-      Clock clock) {
+      Clock clock,
+      MeterRegistry meters) {
     this.repository = repository;
     this.kafka = kafka;
     this.runtime = runtime;
     this.clock = clock;
+    this.meters = meters;
   }
 
   @Transactional
@@ -47,6 +53,7 @@ public class OutboxPublisher {
     for (OutboxEvent event : pending) {
       publish(event);
       event.markPublished(clock.instant());
+      meters.counter(PUBLISHED_METRIC, "topic", event.topic()).increment();
     }
     return pending.size();
   }
@@ -59,8 +66,10 @@ public class OutboxPublisher {
       kafka.send(record).get(SEND_TIMEOUT_SECONDS, TimeUnit.SECONDS);
     } catch (InterruptedException exception) {
       Thread.currentThread().interrupt();
+      meters.counter(FAILURE_METRIC, "topic", event.topic()).increment();
       throw new KafkaException("Interrupted while publishing outbox event", exception);
     } catch (ExecutionException | TimeoutException exception) {
+      meters.counter(FAILURE_METRIC, "topic", event.topic()).increment();
       throw new KafkaException("Failed to publish outbox event", exception);
     }
   }


## `feat(observability): count admin outcomes`

diff --git a/src/main/java/com/sportsbook/settlement/admin/AdminCandidateCommands.java b/src/main/java/com/sportsbook/settlement/admin/AdminCandidateCommands.java
index 7690831..0f03088 100644
--- a/src/main/java/com/sportsbook/settlement/admin/AdminCandidateCommands.java
+++ b/src/main/java/com/sportsbook/settlement/admin/AdminCandidateCommands.java
@@ -1,6 +1,7 @@
 package com.sportsbook.settlement.admin;
 
 import com.sportsbook.settlement.correction.CorrectionFanout;
+import com.sportsbook.settlement.observability.SettlementMetrics;
 import com.sportsbook.settlement.result.AcceptedResultRepository;
 import com.sportsbook.settlement.result.ResultFanout;
 import java.util.UUID;
@@ -14,18 +15,21 @@ public class AdminCandidateCommands {
   private final AcceptedResultRepository acceptedResults;
   private final ResultFanout baseFanout;
   private final CorrectionFanout correctionFanout;
+  private final SettlementMetrics metrics;
 
   public AdminCandidateCommands(
       AdminCandidateApproval approvals,
       AdminCandidateRejection rejections,
       AcceptedResultRepository acceptedResults,
       ResultFanout baseFanout,
-      CorrectionFanout correctionFanout) {
+      CorrectionFanout correctionFanout,
+      SettlementMetrics metrics) {
     this.approvals = approvals;
     this.rejections = rejections;
     this.acceptedResults = acceptedResults;
     this.baseFanout = baseFanout;
     this.correctionFanout = correctionFanout;
+    this.metrics = metrics;
   }
 
   public Receipt approve(UUID idempotencyKey, UUID candidateId) {
@@ -36,6 +40,7 @@ public class AdminCandidateCommands {
             .orElseThrow(() -> new IllegalStateException("Approved result projection is missing"));
     baseFanout.fanOut(accepted);
     correctionFanout.fanOut(accepted);
+    metrics.count("admin_action", decision.replay() ? "replay" : "approved");
     return new Receipt(
         decision.action().idempotencyKey(), decision.action().outcome().name(), decision.replay());
   }
@@ -43,6 +48,7 @@ public class AdminCandidateCommands {
   public Receipt reject(UUID idempotencyKey, UUID candidateId, String reason) {
     AdminCandidateRejection.Decision decision =
         rejections.decide(idempotencyKey, candidateId, reason);
+    metrics.count("admin_action", decision.replay() ? "replay" : "rejected");
     return new Receipt(
         decision.action().idempotencyKey(), decision.action().outcome().name(), decision.replay());
   }
diff --git a/src/main/java/com/sportsbook/settlement/admin/AdminRevisionCommands.java b/src/main/java/com/sportsbook/settlement/admin/AdminRevisionCommands.java
index 2175b6f..0fb65ab 100644
--- a/src/main/java/com/sportsbook/settlement/admin/AdminRevisionCommands.java
+++ b/src/main/java/com/sportsbook/settlement/admin/AdminRevisionCommands.java
@@ -1,5 +1,6 @@
 package com.sportsbook.settlement.admin;
 
+import com.sportsbook.settlement.observability.SettlementMetrics;
 import java.time.Instant;
 import java.util.UUID;
 import org.springframework.stereotype.Service;
@@ -9,10 +10,13 @@ public class AdminRevisionCommands {
 
   private final AdminRevisionRetry retry;
   private final AdminRevisionQueryRepository revisions;
+  private final SettlementMetrics metrics;
 
-  public AdminRevisionCommands(AdminRevisionRetry retry, AdminRevisionQueryRepository revisions) {
+  public AdminRevisionCommands(
+      AdminRevisionRetry retry, AdminRevisionQueryRepository revisions, SettlementMetrics metrics) {
     this.retry = retry;
     this.revisions = revisions;
+    this.metrics = metrics;
   }
 
   public Receipt retry(UUID idempotencyKey, UUID revisionId) {
@@ -21,6 +25,7 @@ public class AdminRevisionCommands {
         revisions
             .find(revisionId)
             .orElseThrow(() -> new IllegalStateException("Queued revision is missing"));
+    metrics.count("admin_retry", decision.replay() ? "replay" : "queued");
     return new Receipt(
         decision.action().idempotencyKey(),
         decision.replay() ? "REPLAY" : "QUEUED",


## `feat(observability): expose settlement backlog gauges`

diff --git a/src/main/java/com/sportsbook/settlement/observability/SettlementBacklogMetrics.java b/src/main/java/com/sportsbook/settlement/observability/SettlementBacklogMetrics.java
new file mode 100644
index 0000000..3aa6075
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/observability/SettlementBacklogMetrics.java
@@ -0,0 +1,45 @@
+package com.sportsbook.settlement.observability;
+
+import io.micrometer.core.instrument.Gauge;
+import io.micrometer.core.instrument.MeterRegistry;
+import io.micrometer.core.instrument.binder.MeterBinder;
+import java.util.Objects;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.stereotype.Component;
+
+@Component
+public final class SettlementBacklogMetrics implements MeterBinder {
+
+  public static final String BACKLOG = "settlement.backlog";
+
+  private final JdbcTemplate jdbc;
+
+  public SettlementBacklogMetrics(JdbcTemplate jdbc) {
+    this.jdbc = jdbc;
+  }
+
+  @Override
+  public void bindTo(MeterRegistry registry) {
+    bind(registry, "pending_bets", "select count(*) from bet where status = 'PENDING'");
+    bind(
+        registry,
+        "blocked_revisions",
+        "select count(*) from settlement_revision where state = 'BLOCKED'");
+    bind(
+        registry,
+        "exhausted_revisions",
+        "select count(*) from settlement_revision where state = 'EXHAUSTED'");
+    bind(registry, "outbox", "select count(*) from outbox_event where published_at is null");
+  }
+
+  private void bind(MeterRegistry registry, String kind, String sql) {
+    Gauge.builder(BACKLOG, jdbc, template -> count(template, sql))
+        .tag("kind", kind)
+        .description("Durable settlement work awaiting completion")
+        .register(registry);
+  }
+
+  private static double count(JdbcTemplate jdbc, String sql) {
+    return Objects.requireNonNull(jdbc.queryForObject(sql, Long.class));
+  }
+}


