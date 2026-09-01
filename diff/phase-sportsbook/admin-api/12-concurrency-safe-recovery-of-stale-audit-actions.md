# 동시성 안전한 중단 감사 작업 복구

## `feat(audit): claim stale STARTED rows safely`

diff --git a/src/main/java/com/sportsbook/admin/audit/AuditWriteRepository.java b/src/main/java/com/sportsbook/admin/audit/AuditWriteRepository.java
index 99e746d..d0cb8c3 100644
--- a/src/main/java/com/sportsbook/admin/audit/AuditWriteRepository.java
+++ b/src/main/java/com/sportsbook/admin/audit/AuditWriteRepository.java
@@ -3,6 +3,7 @@ package com.sportsbook.admin.audit;
 import com.sportsbook.admin.context.AdminContext;
 import com.sportsbook.admin.security.AdminRole;
 import java.sql.Timestamp;
+import java.time.Duration;
 import java.util.List;
 import java.util.UUID;
 import org.springframework.jdbc.core.JdbcTemplate;
@@ -30,6 +31,26 @@ public class AuditWriteRepository {
                 http_status, reason, trace_id, started_at, completed_at
       """;
 
+  private static final String CLAIM_STALE =
+      """
+      WITH stale AS (
+          SELECT action_id
+          FROM audit_log
+          WHERE outcome = 'STARTED'
+            AND started_at <= CURRENT_TIMESTAMP - (? * INTERVAL '1 millisecond')
+          ORDER BY started_at
+          FOR UPDATE SKIP LOCKED
+          LIMIT ?
+      )
+      UPDATE audit_log AS audit
+      SET outcome = 'UNKNOWN', http_status = NULL, completed_at = CURRENT_TIMESTAMP
+      FROM stale
+      WHERE audit.action_id = stale.action_id AND audit.outcome = 'STARTED'
+      RETURNING audit.action_id, audit.actor_id, audit.actor_role, audit.action,
+                audit.target, audit.outcome, audit.http_status, audit.reason,
+                audit.trace_id, audit.started_at, audit.completed_at
+      """;
+
   private final JdbcTemplate jdbc;
 
   public AuditWriteRepository(JdbcTemplate jdbc) {
@@ -54,36 +75,49 @@ public class AuditWriteRepository {
   }
 
   @Transactional(propagation = Propagation.REQUIRES_NEW, timeout = 5)
-  public AuditTerminalRecord complete(
-      UUID actionId, AuditOutcome outcome, Integer httpStatus) {
+  public AuditTerminalRecord complete(UUID actionId, AuditOutcome outcome, Integer httpStatus) {
     if (!outcome.isTerminal()) {
       throw new IllegalArgumentException("Cannot complete an audit row as STARTED");
     }
     List<AuditTerminalRecord> updated =
         jdbc.query(
             COMPLETE_STARTED,
-            (result, rowNumber) ->
-                new AuditTerminalRecord(
-                    result.getObject("action_id", UUID.class),
-                    result.getString("actor_id"),
-                    AdminRole.valueOf(result.getString("actor_role")),
-                    result.getString("action"),
-                    result.getString("target"),
-                    AuditOutcome.valueOf(result.getString("outcome")),
-                    result.getObject("http_status", Integer.class),
-                    result.getString("reason"),
-                    result.getString("trace_id"),
-                    timestamp(result.getTimestamp("started_at")),
-                    timestamp(result.getTimestamp("completed_at"))),
+            AuditWriteRepository::terminalRecord,
             outcome.name(),
             httpStatus,
             actionId);
     if (updated.size() != 1) {
-      throw new IllegalStateException("Audit terminal update did not claim exactly one STARTED row");
+      throw new IllegalStateException(
+          "Audit terminal update did not claim exactly one STARTED row");
     }
     return updated.get(0);
   }
 
+  @Transactional(propagation = Propagation.REQUIRES_NEW, timeout = 5)
+  public List<AuditTerminalRecord> claimStale(Duration staleAfter, int batchSize) {
+    if (staleAfter.isNegative() || staleAfter.isZero() || batchSize < 1) {
+      throw new IllegalArgumentException("Stale claim settings must be positive");
+    }
+    return jdbc.query(
+        CLAIM_STALE, AuditWriteRepository::terminalRecord, staleAfter.toMillis(), batchSize);
+  }
+
+  private static AuditTerminalRecord terminalRecord(java.sql.ResultSet result, int rowNumber)
+      throws java.sql.SQLException {
+    return new AuditTerminalRecord(
+        result.getObject("action_id", UUID.class),
+        result.getString("actor_id"),
+        AdminRole.valueOf(result.getString("actor_role")),
+        result.getString("action"),
+        result.getString("target"),
+        AuditOutcome.valueOf(result.getString("outcome")),
+        result.getObject("http_status", Integer.class),
+        result.getString("reason"),
+        result.getString("trace_id"),
+        timestamp(result.getTimestamp("started_at")),
+        timestamp(result.getTimestamp("completed_at")));
+  }
+
   private static java.time.Instant timestamp(Timestamp timestamp) {
     return timestamp.toInstant();
   }


## `test(audit): transition only stale STARTED rows`

diff --git a/src/test/java/com/sportsbook/admin/audit/AuditStaleClaimTest.java b/src/test/java/com/sportsbook/admin/audit/AuditStaleClaimTest.java
new file mode 100644
index 0000000..6b14a81
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/audit/AuditStaleClaimTest.java
@@ -0,0 +1,87 @@
+package com.sportsbook.admin.audit;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.admin.context.AdminContext;
+import com.sportsbook.admin.security.AdminRole;
+import java.time.Duration;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
+import org.springframework.boot.test.autoconfigure.jdbc.JdbcTest;
+import org.springframework.context.annotation.Import;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.test.context.DynamicPropertyRegistry;
+import org.springframework.test.context.DynamicPropertySource;
+import org.springframework.transaction.annotation.Propagation;
+import org.springframework.transaction.annotation.Transactional;
+import org.testcontainers.containers.PostgreSQLContainer;
+import org.testcontainers.junit.jupiter.Container;
+import org.testcontainers.junit.jupiter.Testcontainers;
+
+@JdbcTest(properties = "spring.flyway.enabled=true")
+@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
+@Import(AuditWriteRepository.class)
+@Transactional(propagation = Propagation.NOT_SUPPORTED)
+@Testcontainers
+class AuditStaleClaimTest {
+
+  @Container
+  private static final PostgreSQLContainer<?> POSTGRES =
+      new PostgreSQLContainer<>("postgres:16-alpine");
+
+  @DynamicPropertySource
+  static void database(DynamicPropertyRegistry registry) {
+    registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
+    registry.add("spring.datasource.username", POSTGRES::getUsername);
+    registry.add("spring.datasource.password", POSTGRES::getPassword);
+  }
+
+  @Autowired private AuditWriteRepository auditWrites;
+  @Autowired private JdbcTemplate jdbc;
+
+  @Test
+  void transitionsOnlyOldStartedRowsToUnknown() {
+    UUID stale = UUID.fromString("018f0000-0000-7000-8000-000000000081");
+    UUID fresh = UUID.fromString("018f0000-0000-7000-8000-000000000082");
+    UUID terminal = UUID.fromString("018f0000-0000-7000-8000-000000000083");
+    begin(stale);
+    begin(fresh);
+    begin(terminal);
+    jdbc.update(
+        "UPDATE audit_log SET started_at=CURRENT_TIMESTAMP-INTERVAL '10 minutes' "
+            + "WHERE action_id IN (?, ?)",
+        stale,
+        terminal);
+    auditWrites.complete(terminal, AuditOutcome.SUCCESS, 202);
+
+    List<AuditTerminalRecord> claimed = auditWrites.claimStale(Duration.ofMinutes(5), 100);
+
+    assertThat(claimed)
+        .singleElement()
+        .satisfies(
+            record -> {
+              assertThat(record.actionId()).isEqualTo(stale);
+              assertThat(record.outcome()).isEqualTo(AuditOutcome.UNKNOWN);
+              assertThat(record.httpStatus()).isNull();
+              assertThat(record.completedAt()).isAfterOrEqualTo(record.startedAt());
+            });
+    assertThat(outcome(fresh)).isEqualTo("STARTED");
+    assertThat(outcome(terminal)).isEqualTo("SUCCESS");
+  }
+
+  private void begin(UUID actionId) {
+    auditWrites.begin(
+        new AdminContext("operator-1", AdminRole.ADMIN, actionId, "trace-1"),
+        "MARKET_CLOSE",
+        "market-1",
+        "close");
+  }
+
+  private String outcome(UUID actionId) {
+    return jdbc.queryForObject(
+        "SELECT outcome FROM audit_log WHERE action_id = ?", String.class, actionId);
+  }
+}


## `feat(audit): recover stale actions on schedule`

diff --git a/src/main/java/com/sportsbook/admin/audit/AuditStaleScheduler.java b/src/main/java/com/sportsbook/admin/audit/AuditStaleScheduler.java
new file mode 100644
index 0000000..5f84209
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/audit/AuditStaleScheduler.java
@@ -0,0 +1,49 @@
+package com.sportsbook.admin.audit;
+
+import io.micrometer.core.instrument.MeterRegistry;
+import java.time.Duration;
+import java.util.List;
+import org.slf4j.Logger;
+import org.slf4j.LoggerFactory;
+import org.springframework.beans.factory.annotation.Value;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.scheduling.annotation.EnableScheduling;
+import org.springframework.scheduling.annotation.Scheduled;
+
+@Configuration(proxyBeanMethods = false)
+@EnableScheduling
+public class AuditStaleScheduler {
+
+  private static final Logger log = LoggerFactory.getLogger(AuditStaleScheduler.class);
+  private static final String CLAIMED_METRIC = "admin.audit.stale.claimed";
+  private static final String FAILURE_METRIC = "admin.audit.stale.scan.failure";
+
+  private final AuditWriteRepository auditWrites;
+  private final MeterRegistry meters;
+  private final Duration staleAfter;
+  private final int batchSize;
+
+  public AuditStaleScheduler(
+      AuditWriteRepository auditWrites,
+      MeterRegistry meters,
+      @Value("${admin.audit.stale-after:PT5M}") Duration staleAfter,
+      @Value("${admin.audit.stale-batch-size:100}") int batchSize) {
+    this.auditWrites = auditWrites;
+    this.meters = meters;
+    this.staleAfter = staleAfter;
+    this.batchSize = batchSize;
+  }
+
+  @Scheduled(
+      initialDelayString = "${admin.audit.stale-scan-interval:PT30S}",
+      fixedDelayString = "${admin.audit.stale-scan-interval:PT30S}")
+  void scan() {
+    try {
+      List<AuditTerminalRecord> claimed = auditWrites.claimStale(staleAfter, batchSize);
+      meters.counter(CLAIMED_METRIC).increment(claimed.size());
+    } catch (RuntimeException failure) {
+      meters.counter(FAILURE_METRIC).increment();
+      log.error("Failed to claim stale audit actions", failure);
+    }
+  }
+}
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index b7eec1e..cee451c 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -73,3 +73,7 @@ admin:
       risk-api-key: ${ADMIN_RISK_API_KEY:}
       odds-feed-api-key: ${ADMIN_ODDS_FEED_API_KEY:}
       settlement-api-key: ${ADMIN_SETTLEMENT_API_KEY:}
+  audit:
+    stale-after: ${ADMIN_AUDIT_STALE_AFTER:5m}
+    stale-scan-interval: ${ADMIN_AUDIT_STALE_SCAN_INTERVAL:PT30S}
+    stale-batch-size: ${ADMIN_AUDIT_STALE_BATCH_SIZE:100}


## `test(audit): keep stale recovery observable`

diff --git a/src/test/java/com/sportsbook/admin/audit/AuditStaleSchedulerTest.java b/src/test/java/com/sportsbook/admin/audit/AuditStaleSchedulerTest.java
new file mode 100644
index 0000000..ee743f1
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/audit/AuditStaleSchedulerTest.java
@@ -0,0 +1,68 @@
+package com.sportsbook.admin.audit;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.admin.security.AdminRole;
+import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
+import java.time.Duration;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.junit.jupiter.api.extension.ExtendWith;
+import org.mockito.Mock;
+import org.mockito.junit.jupiter.MockitoExtension;
+
+@ExtendWith(MockitoExtension.class)
+class AuditStaleSchedulerTest {
+
+  @Mock private AuditWriteRepository auditWrites;
+
+  private final SimpleMeterRegistry meters = new SimpleMeterRegistry();
+
+  @Test
+  void claimsTheConfiguredBatchAndCountsEveryTransition() {
+    when(auditWrites.claimStale(Duration.ofMinutes(5), 37))
+        .thenReturn(List.of(terminal(81), terminal(82)));
+    AuditStaleScheduler scheduler =
+        new AuditStaleScheduler(auditWrites, meters, Duration.ofMinutes(5), 37);
+
+    scheduler.scan();
+
+    verify(auditWrites).claimStale(Duration.ofMinutes(5), 37);
+    assertThat(meters.counter("admin.audit.stale.claimed").count()).isEqualTo(2);
+  }
+
+  @Test
+  void recordsAFailureWithoutStoppingTheNextScan() {
+    when(auditWrites.claimStale(Duration.ofMinutes(5), 100))
+        .thenThrow(new IllegalStateException("database unavailable"))
+        .thenReturn(List.of(terminal(83)));
+    AuditStaleScheduler scheduler =
+        new AuditStaleScheduler(auditWrites, meters, Duration.ofMinutes(5), 100);
+
+    scheduler.scan();
+    scheduler.scan();
+
+    assertThat(meters.counter("admin.audit.stale.scan.failure").count()).isEqualTo(1);
+    assertThat(meters.counter("admin.audit.stale.claimed").count()).isEqualTo(1);
+  }
+
+  private static AuditTerminalRecord terminal(int suffix) {
+    Instant startedAt = Instant.parse("2026-08-22T00:00:00Z");
+    return new AuditTerminalRecord(
+        UUID.fromString("018f0000-0000-7000-8000-0000000000" + suffix),
+        "operator-1",
+        AdminRole.ADMIN,
+        "MARKET_CLOSE",
+        "market-1",
+        AuditOutcome.UNKNOWN,
+        null,
+        "stale",
+        "trace-1",
+        startedAt,
+        startedAt.plus(Duration.ofMinutes(6)));
+  }
+}


## `feat(audit): stream recovered stale actions`

diff --git a/src/main/java/com/sportsbook/admin/audit/AuditStaleScheduler.java b/src/main/java/com/sportsbook/admin/audit/AuditStaleScheduler.java
index 5f84209..c29bccc 100644
--- a/src/main/java/com/sportsbook/admin/audit/AuditStaleScheduler.java
+++ b/src/main/java/com/sportsbook/admin/audit/AuditStaleScheduler.java
@@ -19,16 +19,19 @@ public class AuditStaleScheduler {
   private static final String FAILURE_METRIC = "admin.audit.stale.scan.failure";
 
   private final AuditWriteRepository auditWrites;
+  private final AdminActionPublisher publisher;
   private final MeterRegistry meters;
   private final Duration staleAfter;
   private final int batchSize;
 
   public AuditStaleScheduler(
       AuditWriteRepository auditWrites,
+      AdminActionPublisher publisher,
       MeterRegistry meters,
       @Value("${admin.audit.stale-after:PT5M}") Duration staleAfter,
       @Value("${admin.audit.stale-batch-size:100}") int batchSize) {
     this.auditWrites = auditWrites;
+    this.publisher = publisher;
     this.meters = meters;
     this.staleAfter = staleAfter;
     this.batchSize = batchSize;
@@ -40,6 +43,7 @@ public class AuditStaleScheduler {
   void scan() {
     try {
       List<AuditTerminalRecord> claimed = auditWrites.claimStale(staleAfter, batchSize);
+      claimed.forEach(publisher::publish);
       meters.counter(CLAIMED_METRIC).increment(claimed.size());
     } catch (RuntimeException failure) {
       meters.counter(FAILURE_METRIC).increment();


## `test(audit): stream every stale transition once`

diff --git a/src/test/java/com/sportsbook/admin/audit/AuditStaleSchedulerTest.java b/src/test/java/com/sportsbook/admin/audit/AuditStaleSchedulerTest.java
index ee743f1..7b750f4 100644
--- a/src/test/java/com/sportsbook/admin/audit/AuditStaleSchedulerTest.java
+++ b/src/test/java/com/sportsbook/admin/audit/AuditStaleSchedulerTest.java
@@ -19,6 +19,7 @@ import org.mockito.junit.jupiter.MockitoExtension;
 class AuditStaleSchedulerTest {
 
   @Mock private AuditWriteRepository auditWrites;
+  @Mock private AdminActionPublisher publisher;
 
   private final SimpleMeterRegistry meters = new SimpleMeterRegistry();
 
@@ -27,11 +28,13 @@ class AuditStaleSchedulerTest {
     when(auditWrites.claimStale(Duration.ofMinutes(5), 37))
         .thenReturn(List.of(terminal(81), terminal(82)));
     AuditStaleScheduler scheduler =
-        new AuditStaleScheduler(auditWrites, meters, Duration.ofMinutes(5), 37);
+        new AuditStaleScheduler(auditWrites, publisher, meters, Duration.ofMinutes(5), 37);
 
     scheduler.scan();
 
     verify(auditWrites).claimStale(Duration.ofMinutes(5), 37);
+    verify(publisher).publish(terminal(81));
+    verify(publisher).publish(terminal(82));
     assertThat(meters.counter("admin.audit.stale.claimed").count()).isEqualTo(2);
   }
 
@@ -41,13 +44,14 @@ class AuditStaleSchedulerTest {
         .thenThrow(new IllegalStateException("database unavailable"))
         .thenReturn(List.of(terminal(83)));
     AuditStaleScheduler scheduler =
-        new AuditStaleScheduler(auditWrites, meters, Duration.ofMinutes(5), 100);
+        new AuditStaleScheduler(auditWrites, publisher, meters, Duration.ofMinutes(5), 100);
 
     scheduler.scan();
     scheduler.scan();
 
     assertThat(meters.counter("admin.audit.stale.scan.failure").count()).isEqualTo(1);
     assertThat(meters.counter("admin.audit.stale.claimed").count()).isEqualTo(1);
+    verify(publisher).publish(terminal(83));
   }
 
   private static AuditTerminalRecord terminal(int suffix) {
