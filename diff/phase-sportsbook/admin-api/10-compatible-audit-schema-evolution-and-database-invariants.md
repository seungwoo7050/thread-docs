# 감사 스키마의 호환 진화와 데이터베이스 불변 조건

## `feat(audit): preserve the V1 audit migration`

diff --git a/src/main/resources/db/migration/V1__audit_log.sql b/src/main/resources/db/migration/V1__audit_log.sql
new file mode 100644
index 0000000..bbb105d
--- /dev/null
+++ b/src/main/resources/db/migration/V1__audit_log.sql
@@ -0,0 +1,19 @@
+-- admin-api audit trail (ADR-0011). The one table admin-api owns and the
+-- authoritative record of every operator action. Kafka admin.action receives
+-- a best-effort copy after this row is persisted.
+CREATE TABLE audit_log (
+    action_id   UUID         PRIMARY KEY,
+    actor_id    VARCHAR(128) NOT NULL,
+    actor_role  VARCHAR(32)  NOT NULL,
+    action      VARCHAR(64)  NOT NULL,
+    target      VARCHAR(256),
+    outcome     VARCHAR(16)  NOT NULL,
+    http_status INTEGER      NOT NULL,
+    reason      VARCHAR(512),
+    trace_id    VARCHAR(64),
+    occurred_at TIMESTAMPTZ  NOT NULL
+);
+
+-- The audit-log query filters by time window and optional actor, newest first.
+CREATE INDEX idx_audit_log_occurred_at ON audit_log (occurred_at DESC);
+CREATE INDEX idx_audit_log_actor_time ON audit_log (actor_id, occurred_at DESC);


## `test(audit): lock the V1 migration checksum`

diff --git a/src/test/java/com/sportsbook/admin/audit/AuditV1ChecksumTest.java b/src/test/java/com/sportsbook/admin/audit/AuditV1ChecksumTest.java
new file mode 100644
index 0000000..6ae6c46
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/audit/AuditV1ChecksumTest.java
@@ -0,0 +1,24 @@
+package com.sportsbook.admin.audit;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.security.MessageDigest;
+import java.util.HexFormat;
+import org.junit.jupiter.api.Test;
+
+class AuditV1ChecksumTest {
+
+  private static final String APPROVED_SHA256 =
+      "42dcf9a83473dfe3ba74efce702e7ce5014dea45f25ff2b7699134e1aac360a7";
+
+  @Test
+  void preservesTheReleasedV1MigrationByteForByte() throws Exception {
+    byte[] migration =
+        Files.readAllBytes(Path.of("src/main/resources/db/migration/V1__audit_log.sql"));
+    byte[] digest = MessageDigest.getInstance("SHA-256").digest(migration);
+
+    assertThat(HexFormat.of().formatHex(digest)).isEqualTo(APPROVED_SHA256);
+  }
+}


## `feat(audit): migrate to fail-closed lifecycle states`

diff --git a/src/main/resources/db/migration/V2__audit_lifecycle.sql b/src/main/resources/db/migration/V2__audit_lifecycle.sql
new file mode 100644
index 0000000..9388a35
--- /dev/null
+++ b/src/main/resources/db/migration/V2__audit_lifecycle.sql
@@ -0,0 +1,34 @@
+ALTER TABLE audit_log RENAME COLUMN occurred_at TO started_at;
+ALTER TABLE audit_log ALTER COLUMN http_status DROP NOT NULL;
+ALTER TABLE audit_log ADD COLUMN completed_at TIMESTAMPTZ;
+
+UPDATE audit_log
+SET completed_at = started_at;
+
+ALTER TABLE audit_log
+    ADD CONSTRAINT chk_audit_log_outcome
+    CHECK (
+        (outcome = 'STARTED'
+            AND completed_at IS NULL
+            AND http_status IS NULL)
+        OR
+        (outcome IN ('SUCCESS', 'FAILED')
+            AND completed_at IS NOT NULL
+            AND http_status IS NOT NULL)
+        OR
+        (outcome = 'UNKNOWN'
+            AND completed_at IS NOT NULL)
+    ),
+    ADD CONSTRAINT chk_audit_log_http_status
+    CHECK (http_status IS NULL OR http_status BETWEEN 100 AND 599),
+    ADD CONSTRAINT chk_audit_log_completion_time
+    CHECK (completed_at IS NULL OR completed_at >= started_at);
+
+ALTER INDEX idx_audit_log_occurred_at
+    RENAME TO idx_audit_log_started_at;
+ALTER INDEX idx_audit_log_actor_time
+    RENAME TO idx_audit_log_actor_started_at;
+
+CREATE INDEX idx_audit_log_stale_started
+    ON audit_log (started_at)
+    WHERE outcome = 'STARTED';


## `test(audit): verify clean lifecycle migration`

diff --git a/src/test/java/com/sportsbook/admin/audit/AuditMigrationTest.java b/src/test/java/com/sportsbook/admin/audit/AuditMigrationTest.java
new file mode 100644
index 0000000..ec47df7
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/audit/AuditMigrationTest.java
@@ -0,0 +1,74 @@
+package com.sportsbook.admin.audit;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.sql.Connection;
+import java.sql.DriverManager;
+import java.sql.ResultSet;
+import java.sql.Statement;
+import org.flywaydb.core.Flyway;
+import org.junit.jupiter.api.BeforeEach;
+import org.junit.jupiter.api.Test;
+import org.testcontainers.containers.PostgreSQLContainer;
+import org.testcontainers.junit.jupiter.Container;
+import org.testcontainers.junit.jupiter.Testcontainers;
+
+@Testcontainers
+class AuditMigrationTest {
+
+  @Container
+  private static final PostgreSQLContainer<?> POSTGRES =
+      new PostgreSQLContainer<>("postgres:16-alpine");
+
+  @BeforeEach
+  void cleanDatabase() {
+    flyway().clean();
+  }
+
+  @Test
+  void migratesACleanDatabaseThroughV1AndV2() throws Exception {
+    assertThat(flyway().migrate().migrationsExecuted).isEqualTo(2);
+
+    try (Connection connection = connection(); Statement statement = connection.createStatement()) {
+      assertThat(
+              scalar(
+                  statement,
+                  "SELECT count(*) FROM information_schema.columns "
+                      + "WHERE table_name='audit_log' "
+                      + "AND column_name IN ('started_at','completed_at')"))
+          .isEqualTo(2);
+      assertThat(
+              scalar(
+                  statement,
+                  "SELECT count(*) FROM pg_constraint "
+                      + "WHERE conname LIKE 'chk_audit_log_%'"))
+          .isEqualTo(3);
+      assertThat(
+              scalar(
+                  statement,
+                  "SELECT count(*) FROM pg_indexes "
+                      + "WHERE indexname='idx_audit_log_stale_started'"))
+          .isEqualTo(1);
+    }
+  }
+
+  private static Flyway flyway() {
+    return Flyway.configure()
+        .dataSource(POSTGRES.getJdbcUrl(), POSTGRES.getUsername(), POSTGRES.getPassword())
+        .locations("classpath:db/migration")
+        .cleanDisabled(false)
+        .load();
+  }
+
+  private static Connection connection() throws Exception {
+    return DriverManager.getConnection(
+        POSTGRES.getJdbcUrl(), POSTGRES.getUsername(), POSTGRES.getPassword());
+  }
+
+  private static int scalar(Statement statement, String sql) throws Exception {
+    try (ResultSet result = statement.executeQuery(sql)) {
+      assertThat(result.next()).isTrue();
+      return result.getInt(1);
+    }
+  }
+}


## `test(audit): verify legacy lifecycle upgrade`

diff --git a/src/test/java/com/sportsbook/admin/audit/AuditMigrationTest.java b/src/test/java/com/sportsbook/admin/audit/AuditMigrationTest.java
index ec47df7..aed391d 100644
--- a/src/test/java/com/sportsbook/admin/audit/AuditMigrationTest.java
+++ b/src/test/java/com/sportsbook/admin/audit/AuditMigrationTest.java
@@ -29,7 +29,8 @@ class AuditMigrationTest {
   void migratesACleanDatabaseThroughV1AndV2() throws Exception {
     assertThat(flyway().migrate().migrationsExecuted).isEqualTo(2);
 
-    try (Connection connection = connection(); Statement statement = connection.createStatement()) {
+    try (Connection connection = connection();
+        Statement statement = connection.createStatement()) {
       assertThat(
               scalar(
                   statement,
@@ -40,8 +41,7 @@ class AuditMigrationTest {
       assertThat(
               scalar(
                   statement,
-                  "SELECT count(*) FROM pg_constraint "
-                      + "WHERE conname LIKE 'chk_audit_log_%'"))
+                  "SELECT count(*) FROM pg_constraint " + "WHERE conname LIKE 'chk_audit_log_%'"))
           .isEqualTo(3);
       assertThat(
               scalar(
@@ -52,6 +52,39 @@ class AuditMigrationTest {
     }
   }
 
+  @Test
+  void upgradesLegacyV1RowsWithoutLosingTheirCompletionEvidence() throws Exception {
+    Flyway.configure()
+        .dataSource(POSTGRES.getJdbcUrl(), POSTGRES.getUsername(), POSTGRES.getPassword())
+        .locations("classpath:db/migration")
+        .target("1")
+        .load()
+        .migrate();
+    try (Connection connection = connection();
+        Statement statement = connection.createStatement()) {
+      statement.executeUpdate(
+          "INSERT INTO audit_log "
+              + "(action_id,actor_id,actor_role,action,target,outcome,http_status,reason,trace_id,occurred_at) "
+              + "VALUES ('018f0000-0000-7000-8000-000000000001','operator-1','ADMIN',"
+              + "'WALLET_REFUND','user-1','SUCCESS',200,'approved','trace-1',"
+              + "TIMESTAMPTZ '2026-08-23 00:00:00+00')");
+    }
+
+    assertThat(flyway().migrate().migrationsExecuted).isEqualTo(1);
+
+    try (Connection connection = connection();
+        Statement statement = connection.createStatement();
+        ResultSet row =
+            statement.executeQuery(
+                "SELECT outcome,http_status,started_at=completed_at AS backfilled "
+                    + "FROM audit_log WHERE actor_id='operator-1'")) {
+      assertThat(row.next()).isTrue();
+      assertThat(row.getString("outcome")).isEqualTo("SUCCESS");
+      assertThat(row.getInt("http_status")).isEqualTo(200);
+      assertThat(row.getBoolean("backfilled")).isTrue();
+    }
+  }
+
   private static Flyway flyway() {
     return Flyway.configure()
         .dataSource(POSTGRES.getJdbcUrl(), POSTGRES.getUsername(), POSTGRES.getPassword())


## `feat(audit): model lifecycle audit records`

diff --git a/src/main/java/com/sportsbook/admin/audit/AuditLogEntity.java b/src/main/java/com/sportsbook/admin/audit/AuditLogEntity.java
new file mode 100644
index 0000000..cb18c95
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/audit/AuditLogEntity.java
@@ -0,0 +1,59 @@
+package com.sportsbook.admin.audit;
+
+import com.sportsbook.admin.security.AdminRole;
+import jakarta.persistence.Column;
+import jakarta.persistence.Entity;
+import jakarta.persistence.EnumType;
+import jakarta.persistence.Enumerated;
+import jakarta.persistence.Id;
+import jakarta.persistence.Table;
+import java.time.Instant;
+import java.util.UUID;
+import lombok.AccessLevel;
+import lombok.AllArgsConstructor;
+import lombok.Getter;
+import lombok.NoArgsConstructor;
+
+@Entity
+@Table(name = "audit_log")
+@Getter
+@NoArgsConstructor(access = AccessLevel.PROTECTED)
+@AllArgsConstructor(access = AccessLevel.PACKAGE)
+public class AuditLogEntity {
+
+  @Id
+  @Column(name = "action_id", nullable = false)
+  private UUID actionId;
+
+  @Column(name = "actor_id", nullable = false, length = 128)
+  private String actorId;
+
+  @Enumerated(EnumType.STRING)
+  @Column(name = "actor_role", nullable = false, length = 32)
+  private AdminRole actorRole;
+
+  @Column(name = "action", nullable = false, length = 64)
+  private String action;
+
+  @Column(name = "target", length = 256)
+  private String target;
+
+  @Enumerated(EnumType.STRING)
+  @Column(name = "outcome", nullable = false, length = 16)
+  private AuditOutcome outcome;
+
+  @Column(name = "http_status")
+  private Integer httpStatus;
+
+  @Column(name = "reason", length = 512)
+  private String reason;
+
+  @Column(name = "trace_id", length = 64)
+  private String traceId;
+
+  @Column(name = "started_at", nullable = false)
+  private Instant startedAt;
+
+  @Column(name = "completed_at")
+  private Instant completedAt;
+}
diff --git a/src/main/java/com/sportsbook/admin/audit/AuditOutcome.java b/src/main/java/com/sportsbook/admin/audit/AuditOutcome.java
new file mode 100644
index 0000000..9bf1e02
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/audit/AuditOutcome.java
@@ -0,0 +1,12 @@
+package com.sportsbook.admin.audit;
+
+public enum AuditOutcome {
+  STARTED,
+  SUCCESS,
+  FAILED,
+  UNKNOWN;
+
+  public boolean isTerminal() {
+    return this != STARTED;
+  }
+}


## `test(audit): verify lifecycle entity mapping`

diff --git a/src/test/java/com/sportsbook/admin/audit/AuditLogEntityMappingTest.java b/src/test/java/com/sportsbook/admin/audit/AuditLogEntityMappingTest.java
new file mode 100644
index 0000000..75ba125
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/audit/AuditLogEntityMappingTest.java
@@ -0,0 +1,59 @@
+package com.sportsbook.admin.audit;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.admin.security.AdminRole;
+import jakarta.persistence.Column;
+import jakarta.persistence.EnumType;
+import jakarta.persistence.Enumerated;
+import java.lang.reflect.Field;
+import java.time.Instant;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class AuditLogEntityMappingTest {
+
+  @Test
+  void mapsAllLifecycleEvidenceWithoutCollapsingNullableFields() {
+    UUID actionId = UUID.fromString("018f0000-0000-7000-8000-000000000001");
+    Instant startedAt = Instant.parse("2026-08-23T00:00:00Z");
+    Instant completedAt = startedAt.plusSeconds(1);
+    AuditLogEntity entity =
+        new AuditLogEntity(
+            actionId,
+            "operator-1",
+            AdminRole.ADMIN,
+            "WALLET_REFUND",
+            "user-1",
+            AuditOutcome.SUCCESS,
+            200,
+            "approved",
+            "trace-1",
+            startedAt,
+            completedAt);
+
+    assertThat(entity.getActionId()).isEqualTo(actionId);
+    assertThat(entity.getActorRole()).isEqualTo(AdminRole.ADMIN);
+    assertThat(entity.getOutcome()).isEqualTo(AuditOutcome.SUCCESS);
+    assertThat(entity.getHttpStatus()).isEqualTo(200);
+    assertThat(entity.getStartedAt()).isEqualTo(startedAt);
+    assertThat(entity.getCompletedAt()).isEqualTo(completedAt);
+    assertThat(AuditOutcome.STARTED.isTerminal()).isFalse();
+    assertThat(AuditOutcome.UNKNOWN.isTerminal()).isTrue();
+  }
+
+  @Test
+  void storesEnumsAsStringsAndUsesTheV2ColumnNames() throws Exception {
+    assertThat(field("actorRole").getAnnotation(Enumerated.class).value())
+        .isEqualTo(EnumType.STRING);
+    assertThat(field("outcome").getAnnotation(Enumerated.class).value()).isEqualTo(EnumType.STRING);
+    assertThat(field("startedAt").getAnnotation(Column.class).name()).isEqualTo("started_at");
+    assertThat(field("completedAt").getAnnotation(Column.class).name()).isEqualTo("completed_at");
+    assertThat(field("httpStatus").getAnnotation(Column.class).nullable()).isTrue();
+    assertThat(field("completedAt").getAnnotation(Column.class).nullable()).isTrue();
+  }
+
+  private static Field field(String name) throws NoSuchFieldException {
+    return AuditLogEntity.class.getDeclaredField(name);
+  }
+}
