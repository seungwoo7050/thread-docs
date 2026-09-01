## `feat(health): report settlement dependencies`

diff --git a/src/main/java/com/sportsbook/settlement/observability/SettlementDependenciesHealthIndicator.java b/src/main/java/com/sportsbook/settlement/observability/SettlementDependenciesHealthIndicator.java
new file mode 100644
index 0000000..cc660f9
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/observability/SettlementDependenciesHealthIndicator.java
@@ -0,0 +1,48 @@
+package com.sportsbook.settlement.observability;
+
+import java.util.Map;
+import org.springframework.boot.actuate.health.Health;
+import org.springframework.boot.actuate.health.HealthIndicator;
+import org.springframework.dao.DataAccessException;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.stereotype.Component;
+
+@Component
+public final class SettlementDependenciesHealthIndicator implements HealthIndicator {
+
+  private static final String WORK_QUERY =
+      """
+      select
+        (select count(*) from settlement_revision
+          where state = 'BLOCKED' and next_retry_at is null
+            and last_error_code is not null) paused,
+        (select count(*) from settlement_revision where state = 'EXHAUSTED') exhausted,
+        (select count(*) from outbox_event where published_at is null) outbox
+      """;
+
+  private final JdbcTemplate jdbc;
+
+  public SettlementDependenciesHealthIndicator(JdbcTemplate jdbc) {
+    this.jdbc = jdbc;
+  }
+
+  @Override
+  public Health health() {
+    try {
+      Map<String, Object> work = jdbc.queryForMap(WORK_QUERY);
+      return Health.up()
+          .withDetail("database", "reachable")
+          .withDetail("pausedRevisions", count(work, "paused"))
+          .withDetail("exhaustedRevisions", count(work, "exhausted"))
+          .withDetail("outboxBacklog", count(work, "outbox"))
+          .build();
+    } catch (DataAccessException ignored) {
+      return Health.down().withDetail("database", "unreachable").build();
+    }
+  }
+
+  private static long count(Map<String, Object> work, String key) {
+    Object value = work.get(key);
+    return value instanceof Number number ? number.longValue() : 0;
+  }
+}


## `feat(health): include settlement readiness details`

diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index 21e00e6..b562619 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -30,6 +30,9 @@ management:
     health:
       probes:
         enabled: true
+      group:
+        readiness:
+          include: readinessState,settlementDependencies
   endpoints:
     web:
       exposure:


## `test(observability): execute PostgreSQL health queries`

diff --git a/src/test/java/com/sportsbook/settlement/persistence/PostgresObservabilityIntegrationTest.java b/src/test/java/com/sportsbook/settlement/persistence/PostgresObservabilityIntegrationTest.java
new file mode 100644
index 0000000..38b94aa
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/persistence/PostgresObservabilityIntegrationTest.java
@@ -0,0 +1,38 @@
+package com.sportsbook.settlement.persistence;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.settlement.observability.SettlementBacklogMetrics;
+import com.sportsbook.settlement.observability.SettlementDependenciesHealthIndicator;
+import io.micrometer.prometheus.PrometheusMeterRegistry;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.actuate.health.Status;
+
+class PostgresObservabilityIntegrationTest extends PostgresIntegrationSupport {
+
+  @Autowired private SettlementDependenciesHealthIndicator health;
+  @Autowired private PrometheusMeterRegistry metrics;
+
+  @Test
+  void executesHealthAndBacklogQueriesAgainstPostgres() {
+    insertPendingBet(UUID.randomUUID());
+
+    var snapshot = health.health();
+
+    assertThat(snapshot.getStatus()).isEqualTo(Status.UP);
+    assertThat(snapshot.getDetails())
+        .containsEntry("pausedRevisions", 0L)
+        .containsEntry("exhaustedRevisions", 0L)
+        .containsEntry("outboxBacklog", 0L);
+    assertThat(backlog("pending_bets")).isEqualTo(1);
+    assertThat(backlog("blocked_revisions")).isZero();
+    assertThat(backlog("exhausted_revisions")).isZero();
+    assertThat(backlog("outbox")).isZero();
+  }
+
+  private double backlog(String kind) {
+    return metrics.get(SettlementBacklogMetrics.BACKLOG).tag("kind", kind).gauge().value();
+  }
+}
