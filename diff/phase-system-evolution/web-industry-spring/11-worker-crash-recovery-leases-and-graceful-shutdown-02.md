## `feat: recover expired worker runs and drain shutdown`

diff --git a/backend/src/main/java/dev/evolution/monitor/CheckQueue.java b/backend/src/main/java/dev/evolution/monitor/CheckQueue.java
index 602ef09..edb0dc8 100644
--- a/backend/src/main/java/dev/evolution/monitor/CheckQueue.java
+++ b/backend/src/main/java/dev/evolution/monitor/CheckQueue.java
@@ -32,9 +32,16 @@ public class CheckQueue {
     @Transactional
     public boolean finish(CheckRunner.CheckRun result, UUID owner) {
         // A deleted parent stays deleted; a terminal execution is never reopened or appended again.
-        return entities.createQuery("update CheckRunEntity c set c.state = :state, c.httpStatus = :status, "
-                        + "c.latencyMs = :latency, c.failureReason = :reason, c.finishedAt = :finished "
-                        + "where c.id = :id and c.monitorId = :monitor and c.state = 'RUNNING' and c.claimOwner = :owner")
+        var locked = entities.createNativeQuery("select id from {h-schema}check_runs where id = :id "
+                        + "and monitor_id = :monitor and state = 'RUNNING' and claim_owner = :owner for update")
+                .setParameter("id", result.id()).setParameter("monitor", result.monitorId())
+                .setParameter("owner", owner).getResultList();
+        if (locked.isEmpty()) return false;
+        // Read the authority's wall clock after acquiring the row, not before a possible lock wait.
+        return entities.createNativeQuery("update {h-schema}check_runs set state = :state, http_status = :status, "
+                        + "latency_ms = :latency, failure_reason = :reason, finished_at = :finished "
+                        + "where id = :id and monitor_id = :monitor and state = 'RUNNING' and claim_owner = :owner "
+                        + "and lease_expires_at > clock_timestamp()")
                 .setParameter("state", result.state()).setParameter("status", result.httpStatus())
                 .setParameter("latency", result.latencyMs()).setParameter("reason", result.failureReason())
                 .setParameter("finished", result.finishedAt().truncatedTo(ChronoUnit.MICROS))
@@ -42,6 +49,15 @@ public class CheckQueue {
                 .setParameter("owner", owner).executeUpdate() == 1;
     }
 
+    @Transactional
+    public int recoverExpired(Instant now) {
+        // An expired owner cannot establish an endpoint outcome. Never replay its I/O.
+        return entities.createQuery("update CheckRunEntity c set c.state = 'ABORTED', c.httpStatus = null, "
+                        + "c.latencyMs = null, c.failureReason = null, c.finishedAt = :now "
+                        + "where c.state = 'RUNNING' and c.leaseExpiresAt <= :now")
+                .setParameter("now", now.truncatedTo(ChronoUnit.MICROS)).executeUpdate();
+    }
+
     @Transactional
     public int scheduleDue(Instant now) {
         int created = 0;
diff --git a/backend/src/main/java/dev/evolution/monitor/CheckRunEntity.java b/backend/src/main/java/dev/evolution/monitor/CheckRunEntity.java
index a5b205a..83a1eea 100644
--- a/backend/src/main/java/dev/evolution/monitor/CheckRunEntity.java
+++ b/backend/src/main/java/dev/evolution/monitor/CheckRunEntity.java
@@ -26,6 +26,8 @@ public class CheckRunEntity {
     @Column(name = "manual_owner_user_id") private UUID manualOwnerUserId;
     @Column(name = "idempotency_key", length = 128) private String idempotencyKey;
     @Column(name = "claim_owner") private UUID claimOwner;
+    @Column(name = "lease_expires_at", columnDefinition = "timestamp(6) with time zone")
+    private Instant leaseExpiresAt;
     @Column(name = "started_at", columnDefinition = "timestamp(6) with time zone")
     private Instant startedAt;
     @Column(name = "finished_at", columnDefinition = "timestamp(6) with time zone")
@@ -48,6 +50,7 @@ public class CheckRunEntity {
         state = "RUNNING";
         startedAt = now.truncatedTo(ChronoUnit.MICROS);
         claimOwner = owner;
+        leaseExpiresAt = startedAt.plusMillis(5000);
     }
 
     static CheckRunEntity fromDomain(CheckRunner.CheckRun check) {
diff --git a/backend/src/main/java/dev/evolution/monitor/CheckWorker.java b/backend/src/main/java/dev/evolution/monitor/CheckWorker.java
index 9f798d8..f8948b1 100644
--- a/backend/src/main/java/dev/evolution/monitor/CheckWorker.java
+++ b/backend/src/main/java/dev/evolution/monitor/CheckWorker.java
@@ -1,6 +1,10 @@
 package dev.evolution.monitor;
 
 import java.util.UUID;
+import org.springframework.context.event.ContextClosedEvent;
+import org.springframework.context.event.EventListener;
+import org.springframework.core.Ordered;
+import org.springframework.core.annotation.Order;
 import org.springframework.stereotype.Component;
 
 @Component
@@ -8,6 +12,7 @@ public class CheckWorker {
     private final CheckQueue queue;
     private final CheckRunner runner;
     private final UUID owner = UUID.randomUUID();
+    private volatile boolean acceptingClaims = true;
 
     CheckWorker(CheckQueue queue, CheckRunner runner) {
         this.queue = queue;
@@ -15,6 +20,7 @@ public class CheckWorker {
     }
 
     public boolean executeNext() {
+        if (!acceptingClaims) return false;
         var execution = queue.startNext(owner); // The RUNNING transaction commits before outbound I/O.
         if (execution == null) return false;
         var result = runner.run(execution.check(), execution.url());
@@ -23,4 +29,10 @@ public class CheckWorker {
     }
 
     UUID ownerId() { return owner; }
+
+    boolean acceptingClaims() { return acceptingClaims; }
+
+    @EventListener(ContextClosedEvent.class)
+    @Order(Ordered.HIGHEST_PRECEDENCE)
+    void stopClaims() { acceptingClaims = false; }
 }
diff --git a/backend/src/main/java/dev/evolution/monitor/WorkerProcess.java b/backend/src/main/java/dev/evolution/monitor/WorkerProcess.java
index 295a718..bca2033 100644
--- a/backend/src/main/java/dev/evolution/monitor/WorkerProcess.java
+++ b/backend/src/main/java/dev/evolution/monitor/WorkerProcess.java
@@ -29,6 +29,8 @@ class WorkerProcess {
 
     @Scheduled(fixedDelay = 250)
     void tick() {
+        if (!worker.acceptingClaims()) return;
+        queue.recoverExpired(Instant.now());
         if (scheduling) queue.scheduleDue(Instant.now());
         worker.executeNext();
     }
diff --git a/backend/src/main/resources/application-worker.properties b/backend/src/main/resources/application-worker.properties
new file mode 100644
index 0000000..acaf764
--- /dev/null
+++ b/backend/src/main/resources/application-worker.properties
@@ -0,0 +1,3 @@
+# Spring stops periodic scheduling, then drains the current task before destroying JPA.
+# If this bound expires, the durable lease recovers its unknown outcome as ABORTED.
+spring.lifecycle.timeout-per-shutdown-phase=3s
diff --git a/backend/src/main/resources/db/migration/V8__recover_expired_executions.sql b/backend/src/main/resources/db/migration/V8__recover_expired_executions.sql
new file mode 100644
index 0000000..7855f31
--- /dev/null
+++ b/backend/src/main/resources/db/migration/V8__recover_expired_executions.sql
@@ -0,0 +1,27 @@
+ALTER TABLE check_runs
+    ADD COLUMN lease_expires_at timestamp(6) with time zone,
+    DROP CONSTRAINT check_runs_observed_outcome;
+
+-- Preserve existing outcomes/identities; old in-flight work gains a finite deadline.
+UPDATE check_runs SET lease_expires_at = started_at + interval '5 seconds'
+    WHERE state = 'RUNNING';
+
+ALTER TABLE check_runs
+    ADD CONSTRAINT check_runs_running_lease CHECK (
+        state <> 'RUNNING' OR (lease_expires_at IS NOT NULL AND lease_expires_at > started_at)
+    ),
+    ADD CONSTRAINT check_runs_observed_outcome CHECK ((
+        (state = 'QUEUED' AND started_at IS NULL AND finished_at IS NULL
+            AND http_status IS NULL AND latency_ms IS NULL AND failure_reason IS NULL)
+        OR (state = 'RUNNING' AND started_at IS NOT NULL AND finished_at IS NULL
+            AND http_status IS NULL AND latency_ms IS NULL AND failure_reason IS NULL)
+        OR (state = 'ABORTED' AND started_at IS NOT NULL AND finished_at IS NOT NULL
+            AND http_status IS NULL AND latency_ms IS NULL AND failure_reason IS NULL)
+        OR (started_at IS NOT NULL AND finished_at IS NOT NULL AND latency_ms IS NOT NULL AND (
+            (state = 'SUCCEEDED' AND http_status BETWEEN 200 AND 299 AND failure_reason IS NULL)
+            OR (state = 'FAILED' AND http_status NOT BETWEEN 200 AND 299 AND failure_reason = 'HTTP_STATUS')
+            OR (state = 'FAILED' AND http_status IS NULL AND failure_reason IN ('TIMEOUT', 'CONNECTION_FAILURE'))
+        ))
+    ) IS TRUE);
+
+CREATE INDEX check_runs_expired_lease ON check_runs (lease_expires_at) WHERE state = 'RUNNING';
diff --git a/backend/src/test/java/dev/evolution/monitor/E11WorkerProcess.java b/backend/src/test/java/dev/evolution/monitor/E11WorkerProcess.java
new file mode 100644
index 0000000..5e778ed
--- /dev/null
+++ b/backend/src/test/java/dev/evolution/monitor/E11WorkerProcess.java
@@ -0,0 +1,139 @@
+package dev.evolution.monitor;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import jakarta.persistence.EntityManagerFactory;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.nio.file.StandardCopyOption;
+import java.util.LinkedHashMap;
+import java.util.Map;
+import java.util.Set;
+import java.util.concurrent.CountDownLatch;
+import java.util.concurrent.TimeUnit;
+import org.aopalliance.intercept.MethodInterceptor;
+import org.hibernate.Session;
+import org.springframework.aop.framework.Advised;
+import org.springframework.aop.framework.ProxyFactory;
+import org.springframework.beans.factory.ObjectProvider;
+import org.springframework.beans.factory.config.BeanPostProcessor;
+import org.springframework.boot.WebApplicationType;
+import org.springframework.boot.builder.SpringApplicationBuilder;
+import org.springframework.boot.test.context.TestConfiguration;
+import org.springframework.context.ApplicationListener;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.event.ContextClosedEvent;
+import org.springframework.core.Ordered;
+import org.springframework.orm.jpa.EntityManagerFactoryUtils;
+import org.springframework.transaction.support.TransactionSynchronizationManager;
+
+// Test-only checkpoint adapter. All claims, HTTP I/O, completion and shutdown are production beans.
+public class E11WorkerProcess {
+    private static String checkpoint;
+    private static Path directory;
+
+    public static void main(String[] args) throws Exception {
+        checkpoint = args[0];
+        directory = Path.of(args[1]);
+        if (!Set.of("before-io", "during-io", "before-commit", "after-commit", "shutdown").contains(checkpoint)) {
+            throw new IllegalArgumentException("Unknown checkpoint");
+        }
+        var builder = new SpringApplicationBuilder(MonitorApplication.class, Gates.class).web(WebApplicationType.NONE);
+        if (checkpoint.equals("shutdown")) builder.profiles("worker");
+        String applicationName = "e11-" + checkpoint + "-" + ProcessHandle.current().pid();
+        try (var application = builder.run("--spring.main.banner-mode=off", "--monitor.scheduler-enabled=false",
+                "--spring.datasource.hikari.data-source-properties.ApplicationName=" + applicationName)) {
+            var worker = application.getBean(CheckWorker.class);
+            application.addApplicationListener(new ApplicationListener<ContextClosedEvent>() {
+                @Override public void onApplicationEvent(ContextClosedEvent event) {
+                    if (event.getApplicationContext() != application) return;
+                    write("closing", Map.of("claimsStopped", !worker.acceptingClaims(),
+                            "shutdownPhaseTimeout", application.getEnvironment()
+                                    .getProperty("spring.lifecycle.timeout-per-shutdown-phase", "default")));
+                }
+            });
+            write("ready", Map.of("processId", ProcessHandle.current().pid(), "ownerId", worker.ownerId().toString(),
+                    "applicationName", applicationName, "entry", checkpoint.equals("shutdown") ? "normal scheduled worker profile"
+                            : "test-only startup adapter; production CheckWorker once"));
+            if (checkpoint.equals("shutdown")) { hold(); return; }
+            long deadline = System.nanoTime() + TimeUnit.SECONDS.toNanos(30);
+            while (!Files.exists(directory.resolve("start")) && System.nanoTime() < deadline) Thread.sleep(25);
+            if (!Files.exists(directory.resolve("start"))) throw new IllegalStateException("Start gate was not released");
+            if (!worker.executeNext()) throw new IllegalStateException("Expected one owned execution");
+            write("checkpoint", Map.of("checkpoint", "after-commit", "executeNextReturned", true));
+            hold();
+        }
+    }
+
+    private static void hold() throws InterruptedException {
+        if (!new CountDownLatch(1).await(30, TimeUnit.SECONDS)) throw new IllegalStateException("Checkpoint was not terminated");
+    }
+
+    private static void write(String name, Map<String, ?> value) {
+        try {
+            Path path = directory.resolve(name + ".json");
+            Path temporary = directory.resolve(name + ".tmp");
+            Files.writeString(temporary, new ObjectMapper().writeValueAsString(value) + "\n");
+            Files.move(temporary, path, StandardCopyOption.ATOMIC_MOVE, StandardCopyOption.REPLACE_EXISTING);
+        } catch (Exception error) { throw new IllegalStateException("Checkpoint observation could not be written", error); }
+    }
+
+    @TestConfiguration(proxyBeanMethods = false)
+    static class Gates {
+        @Bean static BeanPostProcessor checkpoints(ObjectProvider<EntityManagerFactory> factories) {
+            return new Checkpoints(factories);
+        }
+    }
+
+    static class Checkpoints implements BeanPostProcessor, Ordered {
+        private final ObjectProvider<EntityManagerFactory> factories;
+        Checkpoints(ObjectProvider<EntityManagerFactory> factories) { this.factories = factories; }
+        @Override public int getOrder() { return Ordered.LOWEST_PRECEDENCE; }
+
+        @Override public Object postProcessAfterInitialization(Object bean, String name) {
+            if (bean instanceof CheckRunner && checkpoint.equals("before-io")) {
+                var proxy = new ProxyFactory(bean);
+                proxy.setProxyTargetClass(true);
+                proxy.addAdvice((MethodInterceptor) invocation -> {
+                    if (invocation.getMethod().getName().equals("run")) {
+                        write("checkpoint", Map.of("checkpoint", "before-io", "claimProxyReturned", true,
+                                "transactionActive", TransactionSynchronizationManager.isActualTransactionActive()));
+                        hold();
+                    }
+                    return invocation.proceed();
+                });
+                return proxy.getProxy();
+            }
+            if (bean instanceof CheckQueue && checkpoint.equals("before-commit")) {
+                // Appended after the existing transaction advisor: the real UPDATE has run,
+                // but its explicit transaction has not committed when this gate is reached.
+                if (!(bean instanceof Advised advised)) throw new IllegalStateException("Expected transactional queue proxy");
+                advised.addAdvice((MethodInterceptor) invocation -> {
+                    Object result = invocation.proceed();
+                    if (invocation.getMethod().getName().equals("finish")) {
+                        var observed = (CheckRunner.CheckRun) invocation.getArguments()[0];
+                        var entityManager = EntityManagerFactoryUtils.getTransactionalEntityManager(factories.getObject());
+                        if (entityManager == null) throw new IllegalStateException("Completion must still own its transaction");
+                        Map<String, Object> evidence = new LinkedHashMap<>();
+                        evidence.put("checkpoint", "before-commit");
+                        evidence.put("observedHttpStatus", observed.httpStatus());
+                        evidence.put("updateChangedRow", result);
+                        evidence.put("transactionActive", TransactionSynchronizationManager.isActualTransactionActive());
+                        entityManager.unwrap(Session.class).doWork(connection -> {
+                            evidence.put("jdbcAutoCommit", connection.getAutoCommit());
+                            try (var statement = connection.createStatement(); var rows = statement.executeQuery(
+                                    "SELECT pg_backend_pid(), txid_current()")) {
+                                if (!rows.next()) throw new IllegalStateException("Missing active database transaction");
+                                evidence.put("databasePid", rows.getInt(1));
+                                evidence.put("transactionId", rows.getLong(2));
+                            }
+                        });
+                        write("checkpoint", evidence);
+                        hold();
+                    }
+                    return result;
+                });
+            }
+            return bean;
+        }
+    }
+}
diff --git a/backend/src/test/java/dev/evolution/monitor/HistoryIndexMigrationTest.java b/backend/src/test/java/dev/evolution/monitor/HistoryIndexMigrationTest.java
index 6c68c05..1158447 100644
--- a/backend/src/test/java/dev/evolution/monitor/HistoryIndexMigrationTest.java
+++ b/backend/src/test/java/dev/evolution/monitor/HistoryIndexMigrationTest.java
@@ -88,6 +88,31 @@ class HistoryIndexMigrationTest {
                      "sevenExistingRowsIncludingQueueMetadataUnchanged":true,"priorMigrationChecksumsUnchanged":true,
                      "noHistoricalRequestOrWorkerIdentityInvented":true}
                     """);
+            TestDatabase.execute("INSERT INTO e07_index_upgrade.check_runs "
+                    + "(id,monitor_id,trigger_kind,state,queued_at,started_at,manual_owner_user_id,idempotency_key,claim_owner) VALUES "
+                    + "('00000000-0000-4000-8000-000000000011','" + monitor + "','MANUAL','RUNNING',"
+                    + "'2026-08-28T00:00:01Z','2026-08-28T00:00:01Z','" + owner + "','e11-upgrade-running',"
+                    + "'00000000-0000-4000-8000-000000000111'),"
+                    + "('00000000-0000-4000-8000-000000000012','" + monitor + "','MANUAL','QUEUED',"
+                    + "'2026-08-28T00:00:01Z',NULL,'" + owner + "','e11-upgrade-queued',NULL)");
+            String beforeLease = scalar("SELECT json_agg(c ORDER BY c.id)::text FROM e07_index_upgrade.check_runs c");
+            String checksumsBeforeLease = scalar("SELECT json_agg(c ORDER BY c.version)::text FROM "
+                    + "(SELECT version,checksum FROM e07_index_upgrade.flyway_schema_history WHERE success) c");
+            var leaseUpgrade = TestDatabase.migration(schema).target("8").load();
+            assertEquals(1, leaseUpgrade.migrate().migrationsExecuted);
+            assertEquals(0, leaseUpgrade.migrate().migrationsExecuted);
+            assertEquals(json.readTree(beforeLease), json.readTree(scalar("SELECT json_agg(to_jsonb(c)-'lease_expires_at' "
+                    + "ORDER BY c.id)::text FROM e07_index_upgrade.check_runs c")));
+            assertEquals(checksumsBeforeLease, scalar("SELECT json_agg(c ORDER BY c.version)::text FROM "
+                    + "(SELECT version,checksum FROM e07_index_upgrade.flyway_schema_history WHERE success AND version <> '8') c"));
+            assertEquals("1", scalar("SELECT count(*) FROM e07_index_upgrade.check_runs WHERE state='RUNNING' "
+                    + "AND lease_expires_at=started_at+interval '5 seconds'"));
+            assertEquals("8", scalar("SELECT count(*) FROM e07_index_upgrade.check_runs WHERE state<>'RUNNING' AND lease_expires_at IS NULL"));
+            Files.writeString(Path.of("target/e11-lease-migration.json"), """
+                    {"result":"PASS","upgradeFrom":7,"upgradeTo":8,"migrationsExecuted":1,"repeatMigrations":0,
+                     "sevenHistoricalRowsPlusRunningAndQueuedUnchanged":true,"priorMigrationChecksumsUnchanged":true,
+                     "onlyExistingRunningRowReceives5000msLease":true,"requestAndWorkerIdentitiesUnchanged":true}
+                    """);
         } finally { TestDatabase.drop(schema); }
     }
 
diff --git a/backend/src/test/java/dev/evolution/monitor/TestDatabase.java b/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
index e27b8e6..74bce71 100644
--- a/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
+++ b/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
@@ -13,7 +13,7 @@ final class TestDatabase {
             "e04_missing_column", "e04_extra_required", "e04_session", "e04_users",
             "e04_missing_user_column", "e04_extra_user_required", "e05_ownership", "e05_fresh",
             "e05_upgrade", "e05_missing_owner", "e05_nullable_owner", "e05_missing_owner_fk", "e07_history", "e07_index_upgrade",
-            "e09_scheduler", "e10_ownership");
+            "e09_scheduler", "e10_ownership", "e11_recovery");
 
     static Connection connect() throws SQLException {
         return DriverManager.getConnection(System.getenv().getOrDefault("DB_URL", "jdbc:postgresql://127.0.0.1:15432/monitor"),
diff --git a/backend/src/test/java/dev/evolution/monitor/WorkerRecoveryTest.java b/backend/src/test/java/dev/evolution/monitor/WorkerRecoveryTest.java
new file mode 100644
index 0000000..ef96c5d
--- /dev/null
+++ b/backend/src/test/java/dev/evolution/monitor/WorkerRecoveryTest.java
@@ -0,0 +1,315 @@
+package dev.evolution.monitor;
+
+import static org.junit.jupiter.api.Assertions.*;
+
+import com.fasterxml.jackson.databind.JsonNode;
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sun.net.httpserver.HttpServer;
+import java.net.InetSocketAddress;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.security.MessageDigest;
+import java.time.Instant;
+import java.time.temporal.ChronoUnit;
+import java.util.ArrayList;
+import java.util.HexFormat;
+import java.util.LinkedHashMap;
+import java.util.List;
+import java.util.Map;
+import java.util.UUID;
+import java.util.concurrent.CountDownLatch;
+import java.util.concurrent.ExecutorService;
+import java.util.concurrent.Executors;
+import java.util.concurrent.TimeUnit;
+import java.util.concurrent.atomic.AtomicInteger;
+import org.junit.jupiter.api.AfterAll;
+import org.junit.jupiter.api.BeforeAll;
+import org.junit.jupiter.api.Test;
+import org.junit.jupiter.api.condition.EnabledIfSystemProperty;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.boot.test.web.client.TestRestTemplate;
+import org.springframework.http.HttpMethod;
+import org.springframework.http.ResponseEntity;
+import org.springframework.test.annotation.DirtiesContext;
+import org.springframework.test.context.DynamicPropertyRegistry;
+import org.springframework.test.context.DynamicPropertySource;
+
+@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
+@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_CLASS)
+@EnabledIfSystemProperty(named = "e11.process-proof", matches = "true")
+class WorkerRecoveryTest {
+    private static final String PASSWORD = SessionClient.password();
+    private static final AtomicInteger okCalls = new AtomicInteger();
+    private static final AtomicInteger failCalls = new AtomicInteger();
+    private static volatile CountDownLatch responseRelease = new CountDownLatch(1);
+    private static HttpServer fixture;
+    private static ExecutorService fixtureThreads;
+    @Autowired TestRestTemplate api;
+    @Autowired CheckQueue queue;
+    @Autowired ObjectMapper json;
+
+    @DynamicPropertySource
+    static void database(DynamicPropertyRegistry properties) { TestDatabase.configure(properties, "e11_recovery"); }
+
+    @BeforeAll
+    static void fixture(@Autowired UserAccounts accounts) throws Exception {
+        accounts.bootstrap(PASSWORD, SessionClient.password());
+        fixture = HttpServer.create(new InetSocketAddress("127.0.0.1", 4321), 0);
+        fixtureThreads = Executors.newVirtualThreadPerTaskExecutor();
+        fixture.setExecutor(fixtureThreads);
+        fixture.createContext("/ok", exchange -> {
+            CountDownLatch release = responseRelease;
+            okCalls.incrementAndGet();
+            try {
+                if (!release.await(30, TimeUnit.SECONDS)) throw new IllegalStateException("Response gate not released");
+                exchange.sendResponseHeaders(200, -1);
+            } catch (InterruptedException error) { Thread.currentThread().interrupt(); }
+            finally { exchange.close(); }
+        });
+        fixture.createContext("/fail", exchange -> {
+            failCalls.incrementAndGet(); exchange.sendResponseHeaders(503, -1); exchange.close();
+        });
+        fixture.start();
+    }
+
+    @AfterAll
+    static void cleanup() {
+        responseRelease.countDown();
+        if (fixture != null) fixture.stop(0);
+        if (fixtureThreads != null) fixtureThreads.close();
+        TestDatabase.drop("e11_recovery");
+    }
+
+    @Test
+    void fourRealKillsRecoverUnknownOutcomesAndRealSigtermDrainsWithoutAnotherClaim() throws Exception {
+        Map<String, Object> evidence = new LinkedHashMap<>();
+        var completed = new ArrayList<Map<String, Object>>();
+        var children = new ArrayList<Process>();
+        var processEvidence = new ArrayList<Map<String, Object>>();
+        evidence.put("fixtureSha256", HexFormat.of().formatHex(MessageDigest.getInstance("SHA-256")
+                .digest(Files.readAllBytes(Path.of("../evidence/phase-1/E11/fixtures.md")))));
+        evidence.put("checkpoints", completed);
+        evidence.put("processes", processEvidence);
+        evidence.put("result", "INCOMPLETE");
+        try {
+            var alice = new SessionClient(api);
+            data(alice.login("alice-e04", PASSWORD), 200);
+            String a = create(alice, "A", "/ok");
+            String b = create(alice, "B", "/fail");
+            String firstId = null;
+            for (String mode : List.of("before-io", "during-io", "before-commit", "after-commit")) {
+                Map<String, Object> observed = new LinkedHashMap<>();
+                observed.put("checkpoint", mode);
+                completed.add(observed);
+                responseRelease = new CountDownLatch(mode.equals("before-commit") || mode.equals("after-commit") ? 0 : 1);
+                int callsBefore = okCalls.get();
+                alice.csrf();
+                String id = data(alice.sendCheck(a, "e11-" + mode), 202).get("id").textValue();
+                if (firstId == null) firstId = id;
+                Path directory = directory(mode);
+                Process child = launch(mode, directory);
+                children.add(child);
+                JsonNode ready = awaitFile(child, directory.resolve("ready.json"));
+                Map<String, Object> process = processObservation(child, ready);
+                processEvidence.add(process);
+                Files.writeString(directory.resolve("start"), "release\n");
+                JsonNode checkpoint;
+                if (mode.equals("during-io")) {
+                    await(() -> okCalls.get() == callsBefore + 1, "held HTTP request");
+                    checkpoint = json.valueToTree(Map.of("checkpoint", mode, "responseWithheld", true));
+                } else checkpoint = awaitFile(child, directory.resolve("checkpoint.json"));
+                observed.put("barrier", checkpoint);
+                JsonNode beforeKill = row(id);
+                boolean committed = mode.equals("after-commit");
+                assertEquals(committed ? "SUCCEEDED" : "RUNNING", beforeKill.get("state").textValue(), mode);
+                assertEquals(ready.get("ownerId").textValue(), beforeKill.get("claim_owner").textValue());
+                Instant started = Instant.parse(beforeKill.get("started_at").textValue());
+                Instant expiry = Instant.parse(beforeKill.get("lease_expires_at").textValue());
+                assertEquals(started.plusMillis(5000), expiry);
+                assertEquals(mode.equals("before-io") ? 0 : 1, okCalls.get() - callsBefore);
+                if (mode.equals("before-io")) assertFalse(checkpoint.get("transactionActive").booleanValue());
+                if (mode.equals("before-commit")) {
+                    assertEquals(200, checkpoint.get("observedHttpStatus").intValue());
+                    assertTrue(checkpoint.get("updateChangedRow").booleanValue());
+                    assertTrue(checkpoint.get("transactionActive").booleanValue());
+                    assertFalse(checkpoint.get("jdbcAutoCommit").booleanValue());
+                    assertTrue(checkpoint.get("transactionId").longValue() > 0);
+                    assertEquals("idle in transaction", scalar("SELECT state FROM pg_stat_activity WHERE pid="
+                            + checkpoint.get("databasePid").intValue()));
+                }
+                if (!committed) assertTrue(beforeKill.get("http_status").isNull());
+                child.destroyForcibly();
+                assertTrue(child.waitFor(30, TimeUnit.SECONDS), "SIGKILL exit must be awaited");
+                process.put("signal", "SIGKILL");
+                process.put("exitCode", child.exitValue());
+                process.put("exitAwaited", true);
+                assertEquals(137, child.exitValue());
+                String applicationName = ready.get("applicationName").textValue();
+                await(() -> "0".equals(scalar("SELECT count(*) FROM pg_stat_activity WHERE application_name='"
+                        + applicationName + "'")), "dead worker database sessions to disappear");
+                observed.put("databaseSessionsGoneBeforeRecovery", true);
+                assertEquals(beforeKill, row(id), "Uncommitted terminal UPDATE must roll back on worker death");
+                responseRelease.countDown();
+                assertEquals(0, queue.recoverExpired(expiry.minus(1, ChronoUnit.MICROS)));
+                assertEquals(committed ? 0 : 1, queue.recoverExpired(expiry));
+                JsonNode terminal = row(id);
+                if (committed) assertEquals(beforeKill, terminal);
+                else {
+                    assertEquals("ABORTED", terminal.get("state").textValue());
+                    assertEquals(expiry, Instant.parse(terminal.get("finished_at").textValue()));
+                    for (String field : List.of("http_status", "latency_ms", "failure_reason")) assertTrue(terminal.get(field).isNull());
+                }
+                assertEquals(0, queue.recoverExpired(expiry));
+                var stale = new CheckRunner.CheckRun(UUID.fromString(id), UUID.fromString(a), "MANUAL", "SUCCEEDED", 200,
+                        1L, null, started, started.plusMillis(1));
+                assertFalse(queue.finish(stale, UUID.fromString(ready.get("ownerId").textValue())));
+                assertEquals(terminal, row(id));
+                JsonNode replay = data(alice.sendCheck(a, "e11-" + mode), 202);
+                assertEquals(id, replay.get("id").textValue());
+                assertEquals(committed ? "SUCCEEDED" : "ABORTED", replay.get("state").textValue());
+                assertEquals(mode.equals("before-io") ? 0 : 1, okCalls.get() - callsBefore);
+                observed.put("checkId", id);
+                observed.put("leaseMilliseconds", 5000);
+                observed.put("terminalState", terminal.get("state").textValue());
+                observed.put("outboundRequests", okCalls.get() - callsBefore);
+                observed.put("preExpiryChanges", 0);
+                observed.put("expiryChanges", committed ? 0 : 1);
+                observed.put("repeatRecoveryChanges", 0);
+                observed.put("deadOwnerWriteChangedRow", false);
+                observed.put("originalIdentitySameTerminal", true);
+            }
+
+            responseRelease = new CountDownLatch(1);
+            Path shutdownDirectory = directory("shutdown");
+            Process shutdown = launch("shutdown", shutdownDirectory);
+            children.add(shutdown);
+            JsonNode ready = awaitFile(shutdown, shutdownDirectory.resolve("ready.json"));
+            Map<String, Object> process = processObservation(shutdown, ready);
+            processEvidence.add(process);
+            int callsBefore = okCalls.get();
+            alice.csrf();
+            String drainId = data(alice.sendCheck(a, "e11-drain"), 202).get("id").textValue();
+            await(() -> okCalls.get() == callsBefore + 1, "shutdown held request");
+            assertEquals("RUNNING", row(drainId).get("state").textValue());
+            String queuedId = data(alice.sendCheck(b, "e11-still-queued"), 202).get("id").textValue();
+            JsonNode queued = row(queuedId);
+            assertEquals("QUEUED", queued.get("state").textValue());
+            long signalled = System.nanoTime();
+            shutdown.destroy();
+            JsonNode closing = awaitFile(shutdown, shutdownDirectory.resolve("closing.json"));
+            assertTrue(closing.get("claimsStopped").booleanValue());
+            assertEquals("3s", closing.get("shutdownPhaseTimeout").textValue());
+            responseRelease.countDown();
+            assertTrue(shutdown.waitFor(3, TimeUnit.SECONDS), "SIGTERM drain deadline");
+            long elapsed = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - signalled);
+            assertTrue(elapsed <= 3000, "Actual SIGTERM-to-exit must fit the frozen deadline");
+            process.put("signal", "SIGTERM");
+            process.put("exitCode", shutdown.exitValue());
+            process.put("exitAwaited", true);
+            assertEquals(143, shutdown.exitValue());
+            JsonNode drained = row(drainId);
+            assertEquals("SUCCEEDED", drained.get("state").textValue());
+            assertEquals(200, drained.get("http_status").intValue());
+            assertEquals(queued, row(queuedId));
+            assertEquals(0, failCalls.get());
+            assertEquals(1, okCalls.get() - callsBefore);
+            evidence.put("shutdown", Map.of("closingObservation", closing, "signalCount", 1, "elapsedMilliseconds", elapsed,
+                    "deadlineMilliseconds", 3000, "inFlightState", "SUCCEEDED", "httpStatus", 200,
+                    "secondRowUnchangedQueued", true, "secondOutboundRequests", failCalls.get(), "actualExitAwaited", true));
+
+            JsonNode original = row(firstId);
+            JsonNode retry = data(alice.sendCheck(a, "e11-new-intent"), 202);
+            assertNotEquals(firstId, retry.get("id").textValue());
+            assertEquals("QUEUED", retry.get("state").textValue());
+            assertEquals(firstId, data(alice.sendCheck(a, "e11-before-io"), 202).get("id").textValue());
+            assertEquals(original, row(firstId));
+            JsonNode history = data(alice.get("/api/monitors/" + a + "/checks?limit=20"), 200);
+            assertEquals(5, history.size());
+            assertEquals(3, history.findValuesAsText("state").stream().filter("ABORTED"::equals).count());
+            evidence.put("identityAndHistory", Map.of("newIntentNewQueuedId", true, "originalTerminalUnchanged", true,
+                    "terminalHistoryRows", 5, "abortedHistoryRows", 3));
+            evidence.put("result", "PASS");
+        } catch (Throwable failure) {
+            evidence.put("failure", failure.getClass().getSimpleName() + ": " + failure.getMessage());
+            throw failure;
+        } finally {
+            responseRelease.countDown();
+            int cleanupSignals = 0;
+            for (Process child : children) {
+                if (child.isAlive()) { child.destroyForcibly(); cleanupSignals++; }
+                if (!child.waitFor(30, TimeUnit.SECONDS)) throw new IllegalStateException("Owned worker did not exit");
+            }
+            evidence.put("cleanup", Map.of("allChildExitsAwaited", true, "cleanupSigkills", cleanupSignals,
+                    "fixtureReleaseSettled", responseRelease.getCount() == 0));
+            Files.writeString(Path.of("target/e11-recovery.json"), json.writerWithDefaultPrettyPrinter().writeValueAsString(evidence) + "\n");
+        }
+    }
+
+    private String create(SessionClient alice, String name, String path) {
+        return data(alice.mutate("/api/monitors", HttpMethod.POST, new MonitorController.CreateMonitor(name,
+                "http://127.0.0.1:4321" + path, 60, false)), 201).at("/monitor/id").textValue();
+    }
+
+    private static JsonNode data(ResponseEntity<JsonNode> response, int status) {
+        assertEquals(status, response.getStatusCode().value());
+        assertNotNull(response.getBody());
+        return response.getBody().get("data");
+    }
+
+    private JsonNode row(String id) throws Exception {
+        return json.readTree(scalar("SELECT row_to_json(c)::text FROM e11_recovery.check_runs c WHERE id='" + UUID.fromString(id) + "'"));
+    }
+
+    private static String scalar(String sql) throws Exception {
+        try (var connection = TestDatabase.connect(); var query = connection.createStatement(); var rows = query.executeQuery(sql)) {
+            assertTrue(rows.next());
+            return rows.getString(1);
+        }
+    }
+
+    private static Path directory(String name) throws Exception {
+        Path path = Path.of("target/e11-workers", name).toAbsolutePath();
+        Files.createDirectories(path);
+        for (String file : List.of("start", "ready.json", "checkpoint.json", "closing.json")) Files.deleteIfExists(path.resolve(file));
+        return path;
+    }
+
+    private static Process launch(String mode, Path directory) throws Exception {
+        var builder = new ProcessBuilder(Path.of(System.getProperty("java.home"), "bin", "java").toString(),
+                "-cp", System.getProperty("java.class.path"), E11WorkerProcess.class.getName(), mode, directory.toString());
+        builder.environment().put("DB_SCHEMA", "e11_recovery");
+        builder.environment().remove("E04_ALICE_PASSWORD");
+        builder.environment().remove("E04_BOB_PASSWORD");
+        return builder.redirectErrorStream(true).redirectOutput(directory.resolve("process.log").toFile()).start();
+    }
+
+    private JsonNode awaitFile(Process child, Path file) throws Exception {
+        await(() -> {
+            if (Files.exists(file)) return true;
+            assertTrue(child.isAlive(), "Owned worker exited before its observation");
+            return false;
+        }, file.getFileName().toString());
+        return json.readTree(Files.readString(file));
+    }
+
+    private static Map<String, Object> processObservation(Process process, JsonNode ready) {
+        assertEquals(process.pid(), ready.get("processId").longValue());
+        Map<String, Object> value = new LinkedHashMap<>();
+        value.put("pid", process.pid());
+        value.put("ownerId", ready.get("ownerId").textValue());
+        value.put("entry", ready.get("entry").textValue());
+        return value;
+    }
+
+    private static void await(Observation observation, String label) throws Exception {
+        long deadline = System.nanoTime() + TimeUnit.SECONDS.toNanos(30);
+        while (System.nanoTime() < deadline) {
+            if (observation.ready()) return;
+            Thread.sleep(25);
+        }
+        fail("Timed out observing " + label);
+    }
+
+    @FunctionalInterface private interface Observation { boolean ready() throws Exception; }
+}


