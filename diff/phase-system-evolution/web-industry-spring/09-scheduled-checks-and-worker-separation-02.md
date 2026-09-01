## `feat(worker): persist queued checks and execute interval intents`

diff --git a/backend/src/main/java/dev/evolution/monitor/CheckQueue.java b/backend/src/main/java/dev/evolution/monitor/CheckQueue.java
new file mode 100644
index 0000000..d6d0410
--- /dev/null
+++ b/backend/src/main/java/dev/evolution/monitor/CheckQueue.java
@@ -0,0 +1,65 @@
+package dev.evolution.monitor;
+
+import jakarta.persistence.EntityManager;
+import jakarta.persistence.PersistenceContext;
+import java.time.Duration;
+import java.time.Instant;
+import java.time.temporal.ChronoUnit;
+import org.springframework.stereotype.Repository;
+import org.springframework.transaction.annotation.Transactional;
+
+@Repository
+public class CheckQueue {
+    @PersistenceContext private EntityManager entities;
+
+    public record Execution(CheckRunner.CheckRun check, String url) {}
+
+    @Transactional
+    public Execution startNext() {
+        // E09 has one worker. Concurrent claims/leases are deliberately not introduced here.
+        var next = entities.createQuery("select c from CheckRunEntity c where c.state = 'QUEUED' "
+                        + "order by c.queuedAt, c.id", CheckRunEntity.class)
+                .setMaxResults(1).getResultStream().findFirst();
+        if (next.isEmpty()) return null;
+        var row = next.get();
+        var parent = entities.find(MonitorEntity.class, row.toDomain().monitorId());
+        if (parent == null) return null;
+        var monitor = parent.toDomain();
+        row.start(Instant.now());
+        return new Execution(row.toDomain(), monitor.url());
+    }
+
+    @Transactional
+    public void finish(CheckRunner.CheckRun result) {
+        // A deleted parent stays deleted; a terminal execution is never reopened or appended again.
+        entities.createQuery("update CheckRunEntity c set c.state = :state, c.httpStatus = :status, "
+                        + "c.latencyMs = :latency, c.failureReason = :reason, c.finishedAt = :finished "
+                        + "where c.id = :id and c.monitorId = :monitor and c.state = 'RUNNING'")
+                .setParameter("state", result.state()).setParameter("status", result.httpStatus())
+                .setParameter("latency", result.latencyMs()).setParameter("reason", result.failureReason())
+                .setParameter("finished", result.finishedAt().truncatedTo(ChronoUnit.MICROS))
+                .setParameter("id", result.id()).setParameter("monitor", result.monitorId()).executeUpdate();
+    }
+
+    @Transactional
+    public int scheduleDue(Instant now) {
+        int created = 0;
+        for (var row : entities.createQuery("select m from MonitorEntity m where m.enabled = true", MonitorEntity.class)
+                .getResultList()) {
+            var monitor = row.toDomain();
+            long elapsed = Duration.between(monitor.createdAt(), now).getSeconds();
+            if (elapsed < monitor.interval()) continue;
+            // One current interval slot per tick; no historical catch-up queue is invented.
+            Instant due = monitor.createdAt().plusSeconds((elapsed / monitor.interval()) * monitor.interval());
+            boolean exists = entities.createQuery("select c.id from CheckRunEntity c where c.monitorId = :monitor "
+                            + "and c.scheduledFor = :due", java.util.UUID.class)
+                    .setParameter("monitor", monitor.id()).setParameter("due", due).setMaxResults(1)
+                    .getResultStream().findAny().isPresent();
+            if (!exists) {
+                entities.persist(CheckRunEntity.queued(monitor.id(), "SCHEDULED", now, due));
+                created++;
+            }
+        }
+        return created;
+    }
+}
diff --git a/backend/src/main/java/dev/evolution/monitor/CheckRunEntity.java b/backend/src/main/java/dev/evolution/monitor/CheckRunEntity.java
index 539a141..8a7528b 100644
--- a/backend/src/main/java/dev/evolution/monitor/CheckRunEntity.java
+++ b/backend/src/main/java/dev/evolution/monitor/CheckRunEntity.java
@@ -17,15 +17,35 @@ public class CheckRunEntity {
     @Column(name = "trigger_kind", nullable = false, length = 16) private String trigger;
     @Column(name = "state", nullable = false, length = 16) private String state;
     @Column(name = "http_status") private Integer httpStatus;
-    @Column(name = "latency_ms", nullable = false) private long latencyMs;
+    @Column(name = "latency_ms") private Long latencyMs;
     @Column(name = "failure_reason", length = 32) private String failureReason;
-    @Column(name = "started_at", nullable = false, columnDefinition = "timestamp(6) with time zone")
+    @Column(name = "queued_at", nullable = false, columnDefinition = "timestamp(6) with time zone")
+    private Instant queuedAt;
+    @Column(name = "scheduled_for", columnDefinition = "timestamp(6) with time zone")
+    private Instant scheduledFor;
+    @Column(name = "started_at", columnDefinition = "timestamp(6) with time zone")
     private Instant startedAt;
-    @Column(name = "finished_at", nullable = false, columnDefinition = "timestamp(6) with time zone")
+    @Column(name = "finished_at", columnDefinition = "timestamp(6) with time zone")
     private Instant finishedAt;
 
     protected CheckRunEntity() {}
 
+    static CheckRunEntity queued(UUID monitorId, String trigger, Instant now, Instant scheduledFor) {
+        var row = new CheckRunEntity();
+        row.id = UUID.randomUUID();
+        row.monitorId = monitorId;
+        row.trigger = trigger;
+        row.state = "QUEUED";
+        row.queuedAt = now.truncatedTo(ChronoUnit.MICROS);
+        row.scheduledFor = scheduledFor;
+        return row;
+    }
+
+    void start(Instant now) {
+        state = "RUNNING";
+        startedAt = now.truncatedTo(ChronoUnit.MICROS);
+    }
+
     static CheckRunEntity fromDomain(CheckRunner.CheckRun check) {
         var row = new CheckRunEntity();
         row.id = check.id();
@@ -37,6 +57,7 @@ public class CheckRunEntity {
         row.failureReason = check.failureReason();
         row.startedAt = check.startedAt().truncatedTo(ChronoUnit.MICROS);
         row.finishedAt = check.finishedAt().truncatedTo(ChronoUnit.MICROS);
+        row.queuedAt = row.finishedAt;
         return row;
     }
 
diff --git a/backend/src/main/java/dev/evolution/monitor/CheckRunner.java b/backend/src/main/java/dev/evolution/monitor/CheckRunner.java
index d231767..59abb52 100644
--- a/backend/src/main/java/dev/evolution/monitor/CheckRunner.java
+++ b/backend/src/main/java/dev/evolution/monitor/CheckRunner.java
@@ -38,9 +38,8 @@ public class CheckRunner {
         }
     }
 
-    public CheckRun run(UUID monitorId, String value) {
+    public CheckRun run(CheckRun execution, String value) {
         URI url = requireFixtureUrl(value); // Recheck at the actual outbound boundary.
-        Instant startedAt = Instant.now();
         long startedNanos = System.nanoTime();
         Integer status = null;
         String failureReason = null;
@@ -61,9 +60,9 @@ public class CheckRunner {
         } finally {
             if (connection != null) connection.disconnect();
         }
-        return new CheckRun(UUID.randomUUID(), monitorId, "MANUAL",
+        return new CheckRun(execution.id(), execution.monitorId(), execution.trigger(),
                 failureReason == null ? "SUCCEEDED" : "FAILED", status,
-                (System.nanoTime() - startedNanos) / 1_000_000, failureReason, startedAt, Instant.now());
+                (System.nanoTime() - startedNanos) / 1_000_000, failureReason, execution.startedAt(), Instant.now());
     }
 
     static boolean isSuccess(int status) {
@@ -71,5 +70,5 @@ public class CheckRunner {
     }
 
     public record CheckRun(UUID id, UUID monitorId, String trigger, String state, Integer httpStatus,
-                           long latencyMs, String failureReason, Instant startedAt, Instant finishedAt) {}
+                           Long latencyMs, String failureReason, Instant startedAt, Instant finishedAt) {}
 }
diff --git a/backend/src/main/java/dev/evolution/monitor/CheckWorker.java b/backend/src/main/java/dev/evolution/monitor/CheckWorker.java
new file mode 100644
index 0000000..36c4dee
--- /dev/null
+++ b/backend/src/main/java/dev/evolution/monitor/CheckWorker.java
@@ -0,0 +1,22 @@
+package dev.evolution.monitor;
+
+import org.springframework.stereotype.Component;
+
+@Component
+public class CheckWorker {
+    private final CheckQueue queue;
+    private final CheckRunner runner;
+
+    CheckWorker(CheckQueue queue, CheckRunner runner) {
+        this.queue = queue;
+        this.runner = runner;
+    }
+
+    public boolean executeNext() {
+        var execution = queue.startNext(); // The RUNNING transaction commits before outbound I/O.
+        if (execution == null) return false;
+        var result = runner.run(execution.check(), execution.url());
+        queue.finish(result); // A separate short transaction records only the observed outcome.
+        return true;
+    }
+}
diff --git a/backend/src/main/java/dev/evolution/monitor/MonitorController.java b/backend/src/main/java/dev/evolution/monitor/MonitorController.java
index 6614e9d..45f89fc 100644
--- a/backend/src/main/java/dev/evolution/monitor/MonitorController.java
+++ b/backend/src/main/java/dev/evolution/monitor/MonitorController.java
@@ -63,11 +63,10 @@ public class MonitorController {
     }
 
     @PostMapping("/{id}/checks")
+    @ResponseStatus(HttpStatus.ACCEPTED)
     public ApiData<CheckRunner.CheckRun> check(@AuthenticationPrincipal UserAccounts.AccountUser user, @PathVariable UUID id) {
-        // Each store call crosses the Spring transaction proxy. Outbound I/O holds no transaction.
-        Monitor monitor = store.monitor(user.userId(), id);
-        CheckRunner.CheckRun result = checks.run(monitor.id(), monitor.url());
-        return new ApiData<>(store.saveCheck(user.userId(), result));
+        // The transaction commits before202; this API process performs no Check I/O.
+        return new ApiData<>(store.enqueueCheck(user.userId(), id));
     }
 
     @GetMapping("/{id}/checks")
diff --git a/backend/src/main/java/dev/evolution/monitor/MonitorStore.java b/backend/src/main/java/dev/evolution/monitor/MonitorStore.java
index 5ba82fd..28a6902 100644
--- a/backend/src/main/java/dev/evolution/monitor/MonitorStore.java
+++ b/backend/src/main/java/dev/evolution/monitor/MonitorStore.java
@@ -62,6 +62,14 @@ public class MonitorStore {
         entities.clear(); // PostgreSQL cascades only this authorized parent's history.
     }
 
+    @Transactional
+    public CheckRunner.CheckRun enqueueCheck(UUID owner, UUID monitorId) {
+        requireMonitor(owner, monitorId);
+        var row = CheckRunEntity.queued(monitorId, "MANUAL", Instant.now(), null);
+        entities.persist(row);
+        return row.toDomain();
+    }
+
     @Transactional
     public CheckRunner.CheckRun saveCheck(UUID owner, CheckRunner.CheckRun check) {
         requireMonitor(owner, check.monitorId());
@@ -77,7 +85,8 @@ public class MonitorStore {
     public HistoryPage historyPage(UUID owner, UUID id, String limit, String state, String cursor) {
         requireMonitor(owner, id);
         var window = HistoryQuery.parse(id, limit, state, cursor);
-        String conditions = "where c.monitorId = m.id and m.id = :id and m.ownerUserId = :owner ";
+        String conditions = "where c.monitorId = m.id and m.id = :id and m.ownerUserId = :owner "
+                + "and c.finishedAt is not null ";
         if (window.state() != null) conditions += "and c.state = :state ";
         if (window.beforeTime() != null) conditions += "and (c.finishedAt < :beforeTime "
                 + "or (c.finishedAt = :beforeTime and c.id < :beforeId)) ";
@@ -114,7 +123,7 @@ public class MonitorStore {
         Monitor monitor = row.toDomain();
         var latest = entities.createQuery("select c from CheckRunEntity c, MonitorEntity m "
                         + "where c.monitorId = m.id and m.id = :id and m.ownerUserId = :owner "
-                        + "order by c.finishedAt desc, c.id desc", CheckRunEntity.class)
+                        + "order by coalesce(c.finishedAt, c.queuedAt) desc, c.id desc", CheckRunEntity.class)
                 .setParameter("id", monitor.id()).setParameter("owner", row.ownerUserId()).setMaxResults(1).getResultStream()
                 .findFirst().map(CheckRunEntity::toDomain).orElse(null);
         return new MonitorController.MonitorView(monitor, latest);
diff --git a/backend/src/main/java/dev/evolution/monitor/WorkerProcess.java b/backend/src/main/java/dev/evolution/monitor/WorkerProcess.java
new file mode 100644
index 0000000..295a718
--- /dev/null
+++ b/backend/src/main/java/dev/evolution/monitor/WorkerProcess.java
@@ -0,0 +1,35 @@
+package dev.evolution.monitor;
+
+import java.time.Instant;
+import org.springframework.beans.factory.annotation.Value;
+import org.springframework.context.ConfigurableApplicationContext;
+import org.springframework.context.annotation.Profile;
+import org.springframework.scheduling.annotation.EnableScheduling;
+import org.springframework.scheduling.annotation.Scheduled;
+import org.springframework.stereotype.Component;
+import org.springframework.web.context.WebApplicationContext;
+
+@Component
+@Profile("worker")
+@EnableScheduling
+class WorkerProcess {
+    private final CheckQueue queue;
+    private final CheckWorker worker;
+    private final boolean scheduling;
+
+    WorkerProcess(CheckQueue queue, CheckWorker worker, ConfigurableApplicationContext context,
+            @Value("${monitor.scheduler-enabled:true}") boolean scheduling) {
+        if (context instanceof WebApplicationContext) {
+            throw new IllegalStateException("Worker requires --spring.main.web-application-type=none.");
+        }
+        this.queue = queue;
+        this.worker = worker;
+        this.scheduling = scheduling;
+    }
+
+    @Scheduled(fixedDelay = 250)
+    void tick() {
+        if (scheduling) queue.scheduleDue(Instant.now());
+        worker.executeNext();
+    }
+}
diff --git a/backend/src/main/resources/db/migration/V6__queue_check_execution.sql b/backend/src/main/resources/db/migration/V6__queue_check_execution.sql
new file mode 100644
index 0000000..19840bc
--- /dev/null
+++ b/backend/src/main/resources/db/migration/V6__queue_check_execution.sql
@@ -0,0 +1,32 @@
+ALTER TABLE check_runs
+    ALTER COLUMN latency_ms DROP NOT NULL,
+    ALTER COLUMN started_at DROP NOT NULL,
+    ALTER COLUMN finished_at DROP NOT NULL,
+    DROP CONSTRAINT check_runs_trigger_kind_check,
+    DROP CONSTRAINT check_runs_observed_outcome,
+    ADD COLUMN queued_at timestamp(6) with time zone,
+    ADD COLUMN scheduled_for timestamp(6) with time zone;
+
+-- Existing terminal outcomes and their history ordering are unchanged.
+UPDATE check_runs SET queued_at = finished_at;
+ALTER TABLE check_runs
+    ALTER COLUMN queued_at SET NOT NULL,
+    ALTER COLUMN queued_at SET DEFAULT CURRENT_TIMESTAMP,
+    ADD CONSTRAINT check_runs_trigger_slot CHECK (
+        (trigger_kind = 'MANUAL' AND scheduled_for IS NULL)
+        OR (trigger_kind = 'SCHEDULED' AND scheduled_for IS NOT NULL)
+    ),
+    ADD CONSTRAINT check_runs_observed_outcome CHECK ((
+        (state = 'QUEUED' AND started_at IS NULL AND finished_at IS NULL
+            AND http_status IS NULL AND latency_ms IS NULL AND failure_reason IS NULL)
+        OR (state = 'RUNNING' AND started_at IS NOT NULL AND finished_at IS NULL
+            AND http_status IS NULL AND latency_ms IS NULL AND failure_reason IS NULL)
+        OR (started_at IS NOT NULL AND finished_at IS NOT NULL AND latency_ms IS NOT NULL AND (
+            (state = 'SUCCEEDED' AND http_status BETWEEN 200 AND 299 AND failure_reason IS NULL)
+            OR (state = 'FAILED' AND http_status NOT BETWEEN 200 AND 299 AND failure_reason = 'HTTP_STATUS')
+            OR (state = 'FAILED' AND http_status IS NULL AND failure_reason IN ('TIMEOUT', 'CONNECTION_FAILURE'))
+        ))
+    ) IS TRUE);
+
+CREATE UNIQUE INDEX check_runs_scheduled_slot ON check_runs (monitor_id, scheduled_for)
+    WHERE scheduled_for IS NOT NULL;
diff --git a/backend/src/test/java/dev/evolution/monitor/CheckQueueTest.java b/backend/src/test/java/dev/evolution/monitor/CheckQueueTest.java
new file mode 100644
index 0000000..7a4d7d9
--- /dev/null
+++ b/backend/src/test/java/dev/evolution/monitor/CheckQueueTest.java
@@ -0,0 +1,88 @@
+package dev.evolution.monitor;
+
+import static org.junit.jupiter.api.Assertions.*;
+import static org.mockito.ArgumentMatchers.*;
+import static org.mockito.Mockito.*;
+
+import com.fasterxml.jackson.databind.JsonNode;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.time.Instant;
+import java.util.HashMap;
+import org.junit.jupiter.api.AfterAll;
+import org.junit.jupiter.api.BeforeAll;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.boot.test.web.client.TestRestTemplate;
+import org.springframework.test.annotation.DirtiesContext;
+import org.springframework.test.context.DynamicPropertyRegistry;
+import org.springframework.test.context.DynamicPropertySource;
+import org.springframework.test.context.bean.override.mockito.MockitoSpyBean;
+
+@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
+@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_CLASS)
+class CheckQueueTest {
+    @Autowired TestRestTemplate api;
+    @Autowired CheckQueue queue;
+    @MockitoSpyBean CheckRunner checks;
+
+    @DynamicPropertySource
+    static void database(DynamicPropertyRegistry properties) { TestDatabase.configure(properties, "e09_scheduler"); }
+
+    @BeforeAll
+    static void authenticate(@Autowired TestRestTemplate api, @Autowired UserAccounts accounts) {
+        String password = SessionClient.password();
+        accounts.bootstrap(password, SessionClient.password());
+        var session = new SessionClient(api);
+        assertEquals(200, session.login("alice-e04", password).getStatusCode().value());
+        session.authorizeExistingProductRequests();
+    }
+
+    @AfterAll
+    static void cleanup() { TestDatabase.drop("e09_scheduler"); }
+
+    @Test
+    void fixedRepeatedSlotsCreateOneEnabledIntentAndNeverScheduleDisabledMonitor() throws Exception {
+        String a = create("A", "/hold", true);
+        String b = create("B", "/ok", false);
+        TestDatabase.execute("UPDATE e09_scheduler.monitors SET created_at='2026-08-27T23:59:00.000Z'");
+        Instant first = Instant.parse("2026-08-28T00:00:00.000Z");
+        int[] expected = {1, 1, 2};
+        Instant[] ticks = {first, first, first.plusSeconds(60)};
+        var ids = new HashMap<Instant, String>();
+        for (int i = 0; i < ticks.length; i++) {
+            assertEquals(i == 1 ? 0 : 1, queue.scheduleDue(ticks[i]));
+            try (var connection = TestDatabase.connect(); var query = connection.createStatement(); var rows = query.executeQuery(
+                    "SELECT id,monitor_id,trigger_kind,state,scheduled_for,started_at,finished_at,http_status,latency_ms "
+                    + "FROM e09_scheduler.check_runs ORDER BY scheduled_for")) {
+                int count = 0;
+                while (rows.next()) {
+                    assertEquals(a, rows.getString("monitor_id"));
+                    assertNotEquals(b, rows.getString("monitor_id"));
+                    assertEquals("SCHEDULED", rows.getString("trigger_kind"));
+                    assertEquals("QUEUED", rows.getString("state"));
+                    Instant due = rows.getTimestamp("scheduled_for").toInstant();
+                    assertEquals(first.plusSeconds(60L * count), due);
+                    String previous = ids.putIfAbsent(due, rows.getString("id"));
+                    if (previous != null) assertEquals(previous, rows.getString("id"));
+                    for (String field : new String[]{"started_at", "finished_at", "http_status", "latency_ms"}) assertNull(rows.getObject(field));
+                    count++;
+                }
+                assertEquals(expected[i], count);
+            }
+        }
+        verify(checks, never()).run(any(CheckRunner.CheckRun.class), anyString());
+        Files.writeString(Path.of("target/e09-scheduler.json"), """
+                {"result":"PASS","ticks":["2026-08-28T00:00:00Z","2026-08-28T00:00:00Z","2026-08-28T00:01:00Z"],
+                 "enabledCounts":[1,1,2],"disabledCounts":[0,0,0],"states":"QUEUED","sameSlotIdentityRetained":true,"outboundCalls":0}
+                """);
+    }
+
+    private String create(String name, String path, boolean enabled) {
+        var response = api.postForEntity("/api/monitors", new MonitorController.CreateMonitor(name,
+                "http://127.0.0.1:4321" + path, 60, enabled), JsonNode.class);
+        assertEquals(201, response.getStatusCode().value());
+        return response.getBody().at("/data/monitor/id").textValue();
+    }
+}
diff --git a/backend/src/test/java/dev/evolution/monitor/CheckRunnerTest.java b/backend/src/test/java/dev/evolution/monitor/CheckRunnerTest.java
index 5fbd6b6..1c409ef 100644
--- a/backend/src/test/java/dev/evolution/monitor/CheckRunnerTest.java
+++ b/backend/src/test/java/dev/evolution/monitor/CheckRunnerTest.java
@@ -7,6 +7,7 @@ import com.fasterxml.jackson.databind.SerializationFeature;
 import com.sun.net.httpserver.HttpServer;
 import java.net.InetSocketAddress;
 import java.net.ServerSocket;
+import java.time.Instant;
 import java.util.UUID;
 import java.util.concurrent.CountDownLatch;
 import org.junit.jupiter.api.Test;
@@ -41,7 +42,7 @@ class CheckRunnerTest {
             closedFixture.bind(new InetSocketAddress("127.0.0.1", 4325));
         }
         var result = new CheckRunner("http://127.0.0.1:4325").run(
-                UUID.fromString("00000000-0000-4000-8000-000000000000"), "http://127.0.0.1:4325/ok");
+                execution(), "http://127.0.0.1:4325/ok");
         assertNoResponse(result, "CONNECTION_FAILURE");
     }
 
@@ -61,7 +62,7 @@ class CheckRunnerTest {
         fixture.start();
         try {
             var result = new CheckRunner("http://127.0.0.1:4325").run(
-                    UUID.fromString("00000000-0000-4000-8000-000000000000"), "http://127.0.0.1:4325/stall");
+                    execution(), "http://127.0.0.1:4325/stall");
             assertNoResponse(result, "TIMEOUT");
         } finally {
             releaseHeaders.countDown();
@@ -69,6 +70,11 @@ class CheckRunnerTest {
         }
     }
 
+    private static CheckRunner.CheckRun execution() {
+        return new CheckRunner.CheckRun(UUID.randomUUID(), UUID.fromString("00000000-0000-4000-8000-000000000000"),
+                "MANUAL", "RUNNING", null, null, null, Instant.now(), null);
+    }
+
     private static void assertNoResponse(CheckRunner.CheckRun result, String reason) {
         assertEquals("FAILED", result.state());
         assertNull(result.httpStatus());
diff --git a/backend/src/test/java/dev/evolution/monitor/HistoryIndexMigrationTest.java b/backend/src/test/java/dev/evolution/monitor/HistoryIndexMigrationTest.java
index 66a9de0..5581a08 100644
--- a/backend/src/test/java/dev/evolution/monitor/HistoryIndexMigrationTest.java
+++ b/backend/src/test/java/dev/evolution/monitor/HistoryIndexMigrationTest.java
@@ -37,7 +37,7 @@ class HistoryIndexMigrationTest {
                     """.formatted(monitor));
             String rowsBefore = rows();
             String migrationsBefore = oldMigrations();
-            var upgrade = TestDatabase.migration(schema).load();
+            var upgrade = TestDatabase.migration(schema).target("5").load();
             assertEquals(1, upgrade.migrate().migrationsExecuted);
             upgrade.validate();
             assertEquals(5, upgrade.info().applied().length);
@@ -51,6 +51,25 @@ class HistoryIndexMigrationTest {
                      "repeatMigrations":0,"sevenHistoricalRowsUnchanged":true,
                      "monitorUnchanged":true,"priorMigrationChecksumsUnchanged":true,"newIndexes":2}
                     """);
+            String beforeQueue = scalar("SELECT json_agg(c ORDER BY c.id)::text FROM e07_index_upgrade.check_runs c");
+            String migrationChecksums = scalar("SELECT json_agg(c ORDER BY c.version)::text FROM "
+                    + "(SELECT version,checksum FROM e07_index_upgrade.flyway_schema_history WHERE success) c");
+            var queueUpgrade = TestDatabase.migration(schema).target("6").load();
+            assertEquals(1, queueUpgrade.migrate().migrationsExecuted);
+            assertEquals(0, queueUpgrade.migrate().migrationsExecuted);
+            String afterQueue = scalar("SELECT json_agg(to_jsonb(c)-'queued_at'-'scheduled_for' ORDER BY c.id)::text "
+                    + "FROM e07_index_upgrade.check_runs c");
+            var json = new com.fasterxml.jackson.databind.ObjectMapper();
+            assertEquals(json.readTree(beforeQueue), json.readTree(afterQueue));
+            assertEquals(migrationChecksums, scalar("SELECT json_agg(c ORDER BY c.version)::text FROM "
+                    + "(SELECT version,checksum FROM e07_index_upgrade.flyway_schema_history WHERE success AND version <> '6') c"));
+            assertEquals("7", scalar("SELECT count(*) FROM e07_index_upgrade.check_runs "
+                    + "WHERE queued_at=finished_at AND scheduled_for IS NULL"));
+            Files.writeString(Path.of("target/e09-queue-migration.json"), """
+                    {"result":"PASS","upgradeFrom":5,"upgradeTo":6,"migrationsExecuted":1,
+                     "repeatMigrations":0,"sevenTerminalRowsAndTimestampsUnchanged":true,
+                     "priorMigrationChecksumsUnchanged":true,"queuedAtBackfilledFromFinishedAt":true}
+                    """);
         } finally { TestDatabase.drop(schema); }
     }
 
diff --git a/backend/src/test/java/dev/evolution/monitor/MonitorFunctionalTest.java b/backend/src/test/java/dev/evolution/monitor/MonitorFunctionalTest.java
index d5d7fdc..87cebbd 100644
--- a/backend/src/test/java/dev/evolution/monitor/MonitorFunctionalTest.java
+++ b/backend/src/test/java/dev/evolution/monitor/MonitorFunctionalTest.java
@@ -41,6 +41,7 @@ class MonitorFunctionalTest {
     static final AtomicInteger forbiddenRequests = new AtomicInteger();
     @Autowired TestRestTemplate api;
     @Autowired ObjectMapper json;
+    @Autowired CheckWorker worker;
     @MockitoSpyBean CheckRunner checks;
 
     @BeforeAll
@@ -205,8 +206,15 @@ class MonitorFunctionalTest {
         assertTrue(monitor.get("url").isTextual());
         assertTrue(monitor.get("interval").isIntegralNumber());
         assertTrue(monitor.get("enabled").isBoolean());
-        JsonNode check = assertDataEnvelope(api.postForEntity("/api/monitors/" + monitor.get("id").textValue()
-                + "/checks", null, JsonNode.class), HttpStatus.OK);
+        String path = "/api/monitors/" + monitor.get("id").textValue() + "/checks";
+        JsonNode queued = assertDataEnvelope(api.postForEntity(path, null, JsonNode.class), HttpStatus.ACCEPTED);
+        assertEquals("QUEUED", queued.get("state").textValue());
+        for (String field : new String[]{"httpStatus", "latencyMs", "failureReason", "startedAt", "finishedAt"}) {
+            assertTrue(queued.get(field).isNull());
+        }
+        assertTrue(worker.executeNext());
+        JsonNode check = assertDataEnvelope(api.getForEntity(path + "/" + queued.get("id").textValue(), JsonNode.class), HttpStatus.OK);
+        assertEquals(queued.get("id"), check.get("id"));
         assertEquals(Set.of("id", "monitorId", "trigger", "state", "httpStatus", "latencyMs", "failureReason",
                 "startedAt", "finishedAt"), keys(check));
         UUID.fromString(check.get("id").textValue());
@@ -290,11 +298,21 @@ class MonitorFunctionalTest {
     }
 
     @Test
-    void synchronousOutboundRequestDoesNotHoldAStoreTransaction() {
+    void workerOutboundRequestSeesCommittedRunningWithoutHoldingAStoreTransaction() {
         doAnswer(invocation -> {
             assertFalse(TransactionSynchronizationManager.isActualTransactionActive());
+            CheckRunner.CheckRun execution = invocation.getArgument(0);
+            try (var connection = TestDatabase.connect(); var query = connection.prepareStatement(
+                    "SELECT state,finished_at FROM e04_functional.check_runs WHERE id=?")) {
+                query.setObject(1, execution.id());
+                try (var rows = query.executeQuery()) {
+                    assertTrue(rows.next());
+                    assertEquals("RUNNING", rows.getString(1));
+                    assertNull(rows.getObject(2));
+                }
+            }
             return invocation.callRealMethod();
-        }).when(checks).run(any(UUID.class), anyString());
+        }).when(checks).run(any(CheckRunner.CheckRun.class), anyString());
         var view = create("/ok");
         var observed = check(view);
         JsonNode history = assertDataEnvelope(api.getForEntity("/api/monitors/" + view.monitor().id() + "/checks",
@@ -355,6 +373,10 @@ class MonitorFunctionalTest {
     private CheckRunner.CheckRun check(MonitorController.MonitorView monitor) {
         var response = api.postForEntity("/api/monitors/" + monitor.monitor().id() + "/checks", null,
                 JsonNode.class);
-        return json.convertValue(assertDataEnvelope(response, HttpStatus.OK), CheckRunner.CheckRun.class);
+        var accepted = json.convertValue(assertDataEnvelope(response, HttpStatus.ACCEPTED), CheckRunner.CheckRun.class);
+        assertEquals("QUEUED", accepted.state());
+        assertTrue(worker.executeNext());
+        return json.convertValue(assertDataEnvelope(api.getForEntity("/api/monitors/" + monitor.monitor().id()
+                + "/checks/" + accepted.id(), JsonNode.class), HttpStatus.OK), CheckRunner.CheckRun.class);
     }
 }
diff --git a/backend/src/test/java/dev/evolution/monitor/OwnershipAuthorizationTest.java b/backend/src/test/java/dev/evolution/monitor/OwnershipAuthorizationTest.java
index f56c51d..a823612 100644
--- a/backend/src/test/java/dev/evolution/monitor/OwnershipAuthorizationTest.java
+++ b/backend/src/test/java/dev/evolution/monitor/OwnershipAuthorizationTest.java
@@ -54,6 +54,7 @@ class OwnershipAuthorizationTest {
     @Autowired TestRestTemplate api;
     @Autowired ObjectMapper json;
     @Autowired MonitorStore store;
+    @Autowired CheckWorker worker;
     @Autowired UserAccounts accounts;
     @Autowired SecurityFilterChain security;
     @Autowired CsrfTokenRepository csrfTokens;
@@ -98,7 +99,7 @@ class OwnershipAuthorizationTest {
         doAnswer(invocation -> {
             assertFalse(TransactionSynchronizationManager.isActualTransactionActive(), "Outbound must be outside the store transaction");
             return invocation.callRealMethod();
-        }).when(checks).run(any(UUID.class), anyString());
+        }).when(checks).run(any(CheckRunner.CheckRun.class), anyString());
         alice = new SessionClient(api);
         bob = new SessionClient(api);
         assertEquals(200, alice.login("alice-e04", ALICE_PASSWORD).getStatusCode().value());
@@ -108,8 +109,8 @@ class OwnershipAuthorizationTest {
                 .at("/monitor/id").textValue();
         bobId = data(bob.mutate("/api/monitors", HttpMethod.POST, input("Monitor B", "/fail", 120, true)), 201)
                 .at("/monitor/id").textValue();
-        JsonNode a = data(alice.mutate(path(aliceId) + "/checks", HttpMethod.POST, null), 200);
-        JsonNode b = data(bob.mutate(path(bobId) + "/checks", HttpMethod.POST, null), 200);
+        JsonNode a = completedCheck(alice, aliceId);
+        JsonNode b = completedCheck(bob, bobId);
         aliceCheck = a.get("id").textValue();
         bobCheck = b.get("id").textValue();
         assertEquals("SUCCEEDED", a.get("state").textValue());
@@ -162,7 +163,7 @@ class OwnershipAuthorizationTest {
             assertEquals(enabled, changed.at("/monitor/enabled").booleanValue());
             assertEquals(changed, data(alice.get(path(aliceId)), 200));
         }
-        data(alice.mutate(path(aliceId) + "/checks", HttpMethod.POST, null), 200);
+        completedCheck(alice, aliceId);
         assertEquals(2, data(alice.get(path(aliceId) + "/checks"), 200).size());
         Map<String, Object> creation = new LinkedHashMap<>(input("Extra owner creation", "/ok", 60, true));
         creation.put("ownerUserId", bobOwner.toString());
@@ -233,7 +234,8 @@ class OwnershipAuthorizationTest {
                 revoked = alice.copyCredential();
                 csrfAlone = alice.csrfOnly();
             }
-            data(alice.mutate(write.path, write.method, write.body), write.name.equals("create") ? 201 : 200);
+            if (write.name.equals("check")) completedCheck(alice, aliceId);
+            else data(alice.mutate(write.path, write.method, write.body), write.name.equals("create") ? 201 : 200);
         }
         assertEquals(bobBefore, data(bob.get(path(bobId)), 200));
         assertEquals(3, outbound.get());
@@ -356,7 +358,7 @@ class OwnershipAuthorizationTest {
         assertTrue(SqlEvidence.events.stream().allMatch(SqlEvent::transaction));
         for (var event : SqlEvidence.events) {
             String sql = event.sql;
-            if (sql.startsWith("select") || sql.startsWith("update") || sql.startsWith("delete")) {
+            if (!event.worker && (sql.startsWith("select") || sql.startsWith("update") || sql.startsWith("delete"))) {
                 int where = sql.indexOf(" where ");
                 assertTrue(where >= 0 && sql.substring(where).contains("owner_user_id"), "Every resource query includes the owner predicate");
             }
@@ -371,17 +373,28 @@ class OwnershipAuthorizationTest {
                 SqlEvidence.events.stream().map(Object::toString).distinct().toList()) + "\n");
     }
 
+    private JsonNode completedCheck(SessionClient client, String monitorId) {
+        JsonNode accepted = data(client.mutate(path(monitorId) + "/checks", HttpMethod.POST, null), 202);
+        assertEquals("QUEUED", accepted.get("state").textValue());
+        // Only trusted worker setup is classified separately; all user/store SQL stays owner-audited.
+        SqlEvidence.workerScope.set(true);
+        try { assertTrue(worker.executeNext()); }
+        finally { SqlEvidence.workerScope.remove(); }
+        return data(client.get(path(monitorId) + "/checks/" + accepted.get("id").textValue()), 200);
+    }
+
     private record Actor(String name, SessionClient client, String ownId, String ownCheck, String foreignId, String foreignCheck) {}
     private record Mutation(String name, HttpMethod method, String path, Object body) {}
     private record Proof(String name, String origin, SessionClient session) {}
-    record SqlEvent(String sql, boolean transaction, boolean readOnly) {}
+    record SqlEvent(String sql, boolean transaction, boolean readOnly, boolean worker) {}
 
     public static class SqlEvidence implements StatementInspector {
         static final List<SqlEvent> events = new CopyOnWriteArrayList<>();
+        static final ThreadLocal<Boolean> workerScope = ThreadLocal.withInitial(() -> false);
         @Override public String inspect(String sql) {
             if (sql.contains("e05_ownership.monitors") || sql.contains("e05_ownership.check_runs")) {
                 events.add(new SqlEvent(sql, TransactionSynchronizationManager.isActualTransactionActive(),
-                        TransactionSynchronizationManager.isCurrentTransactionReadOnly()));
+                        TransactionSynchronizationManager.isCurrentTransactionReadOnly(), workerScope.get()));
             }
             return sql;
         }
diff --git a/backend/src/test/java/dev/evolution/monitor/OwnershipMigrationTest.java b/backend/src/test/java/dev/evolution/monitor/OwnershipMigrationTest.java
index efbea55..dbc9887 100644
--- a/backend/src/test/java/dev/evolution/monitor/OwnershipMigrationTest.java
+++ b/backend/src/test/java/dev/evolution/monitor/OwnershipMigrationTest.java
@@ -143,7 +143,8 @@ class OwnershipMigrationTest {
     private static String canonicalRows() throws Exception {
         return scalar("SELECT json_agg(m)::text FROM (SELECT id,name,url,interval_seconds,enabled,created_at,updated_at "
                 + "FROM e05_upgrade.monitors ORDER BY id) m") + scalar("SELECT json_agg(c)::text FROM "
-                + "(SELECT * FROM e05_upgrade.check_runs ORDER BY id) c");
+                + "(SELECT id,monitor_id,trigger_kind,state,http_status,latency_ms,failure_reason,started_at,finished_at "
+                + "FROM e05_upgrade.check_runs ORDER BY id) c");
     }
 
     private static String scalar(String sql) throws Exception {
@@ -189,7 +190,6 @@ class OwnershipMigrationTest {
 
     private static String[] arguments(String schema) {
         return new String[]{"--spring.flyway.schemas=" + schema, "--spring.flyway.default-schema=" + schema,
-                "--spring.flyway.target=4",
                 "--spring.jpa.properties.hibernate.default_schema=" + schema,
                 "--spring.main.banner-mode=off", "--logging.level.root=OFF", "--server.port=4322"};
     }
diff --git a/backend/src/test/java/dev/evolution/monitor/PostgresPersistenceTest.java b/backend/src/test/java/dev/evolution/monitor/PostgresPersistenceTest.java
index ad51a5f..b68c2db 100644
--- a/backend/src/test/java/dev/evolution/monitor/PostgresPersistenceTest.java
+++ b/backend/src/test/java/dev/evolution/monitor/PostgresPersistenceTest.java
@@ -67,9 +67,9 @@ class PostgresPersistenceTest {
         var monitor = new Monitor(MONITOR_ID, "Mapping fixture", "http://127.0.0.1:4321/ok", 1, false,
                 NANO_TIME, NANO_TIME);
         var successInput = new CheckRunner.CheckRun(UUID.fromString("00000000-0000-4000-8000-000000000032"),
-                MONITOR_ID, "MANUAL", "SUCCEEDED", 200, 0, null, NANO_TIME, NANO_TIME);
+                MONITOR_ID, "MANUAL", "SUCCEEDED", 200, 0L, null, NANO_TIME, NANO_TIME);
         var timeoutInput = new CheckRunner.CheckRun(UUID.fromString("00000000-0000-4000-8000-000000000033"),
-                MONITOR_ID, "MANUAL", "FAILED", null, 2000, "TIMEOUT", NANO_TIME, NANO_TIME.plusSeconds(2));
+                MONITOR_ID, "MANUAL", "FAILED", null, 2000L, "TIMEOUT", NANO_TIME, NANO_TIME.plusSeconds(2));
         var success = canonical(successInput);
         var timeout = canonical(timeoutInput);
         try {
diff --git a/backend/src/test/java/dev/evolution/monitor/TestDatabase.java b/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
index 0d971b5..50e70c0 100644
--- a/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
+++ b/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
@@ -12,7 +12,8 @@ final class TestDatabase {
     private static final Set<String> SCHEMAS = Set.of("e04_functional", "e04_migrations", "e04_mapping",
             "e04_missing_column", "e04_extra_required", "e04_session", "e04_users",
             "e04_missing_user_column", "e04_extra_user_required", "e05_ownership", "e05_fresh",
-            "e05_upgrade", "e05_missing_owner", "e05_nullable_owner", "e05_missing_owner_fk", "e07_history", "e07_index_upgrade");
+            "e05_upgrade", "e05_missing_owner", "e05_nullable_owner", "e05_missing_owner_fk", "e07_history", "e07_index_upgrade",
+            "e09_scheduler");
 
     static Connection connect() throws SQLException {
         return DriverManager.getConnection(System.getenv().getOrDefault("DB_URL", "jdbc:postgresql://127.0.0.1:15432/monitor"),


