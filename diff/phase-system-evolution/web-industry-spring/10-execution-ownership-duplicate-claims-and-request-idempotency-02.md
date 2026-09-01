## `feat(execution): adopt preserved E10 ownership implementation`

diff --git a/backend/src/main/java/dev/evolution/monitor/ApiErrors.java b/backend/src/main/java/dev/evolution/monitor/ApiErrors.java
index 9cb3d07..4614d7b 100644
--- a/backend/src/main/java/dev/evolution/monitor/ApiErrors.java
+++ b/backend/src/main/java/dev/evolution/monitor/ApiErrors.java
@@ -17,7 +17,7 @@ import org.springframework.web.servlet.resource.NoResourceFoundException;
 
 @RestControllerAdvice
 public class ApiErrors {
-    public enum Code { INVALID_INPUT, NOT_FOUND, UNAUTHENTICATED, FORBIDDEN, INTERNAL_ERROR }
+    public enum Code { INVALID_INPUT, NOT_FOUND, UNAUTHENTICATED, FORBIDDEN, CONFLICT, INTERNAL_ERROR }
     public record Detail(Code code, String message) {}
     public record Failure(Detail error) {}
 
@@ -37,6 +37,7 @@ public class ApiErrors {
             case 400 -> response(HttpStatus.BAD_REQUEST, Code.INVALID_INPUT,
                     error.getReason() == null ? "Request input is invalid." : error.getReason());
             case 404 -> notFound();
+            case 409 -> response(HttpStatus.CONFLICT, Code.CONFLICT, "Idempotency-Key already identifies another Monitor.");
             default -> internalError();
         };
     }
diff --git a/backend/src/main/java/dev/evolution/monitor/CheckQueue.java b/backend/src/main/java/dev/evolution/monitor/CheckQueue.java
index d6d0410..602ef09 100644
--- a/backend/src/main/java/dev/evolution/monitor/CheckQueue.java
+++ b/backend/src/main/java/dev/evolution/monitor/CheckQueue.java
@@ -5,6 +5,7 @@ import jakarta.persistence.PersistenceContext;
 import java.time.Duration;
 import java.time.Instant;
 import java.time.temporal.ChronoUnit;
+import java.util.UUID;
 import org.springframework.stereotype.Repository;
 import org.springframework.transaction.annotation.Transactional;
 
@@ -15,30 +16,30 @@ public class CheckQueue {
     public record Execution(CheckRunner.CheckRun check, String url) {}
 
     @Transactional
-    public Execution startNext() {
-        // E09 has one worker. Concurrent claims/leases are deliberately not introduced here.
-        var next = entities.createQuery("select c from CheckRunEntity c where c.state = 'QUEUED' "
-                        + "order by c.queuedAt, c.id", CheckRunEntity.class)
-                .setMaxResults(1).getResultStream().findFirst();
-        if (next.isEmpty()) return null;
-        var row = next.get();
+    public Execution startNext(UUID owner) {
+        // The selected row stays locked until RUNNING and its owner commit together.
+        var rows = entities.createNativeQuery("select * from {h-schema}check_runs where state = 'QUEUED' "
+                        + "order by queued_at, id limit 1 for update skip locked", CheckRunEntity.class).getResultList();
+        if (rows.isEmpty()) return null;
+        var row = (CheckRunEntity) rows.getFirst();
         var parent = entities.find(MonitorEntity.class, row.toDomain().monitorId());
         if (parent == null) return null;
         var monitor = parent.toDomain();
-        row.start(Instant.now());
+        row.start(Instant.now(), owner);
         return new Execution(row.toDomain(), monitor.url());
     }
 
     @Transactional
-    public void finish(CheckRunner.CheckRun result) {
+    public boolean finish(CheckRunner.CheckRun result, UUID owner) {
         // A deleted parent stays deleted; a terminal execution is never reopened or appended again.
-        entities.createQuery("update CheckRunEntity c set c.state = :state, c.httpStatus = :status, "
+        return entities.createQuery("update CheckRunEntity c set c.state = :state, c.httpStatus = :status, "
                         + "c.latencyMs = :latency, c.failureReason = :reason, c.finishedAt = :finished "
-                        + "where c.id = :id and c.monitorId = :monitor and c.state = 'RUNNING'")
+                        + "where c.id = :id and c.monitorId = :monitor and c.state = 'RUNNING' and c.claimOwner = :owner")
                 .setParameter("state", result.state()).setParameter("status", result.httpStatus())
                 .setParameter("latency", result.latencyMs()).setParameter("reason", result.failureReason())
                 .setParameter("finished", result.finishedAt().truncatedTo(ChronoUnit.MICROS))
-                .setParameter("id", result.id()).setParameter("monitor", result.monitorId()).executeUpdate();
+                .setParameter("id", result.id()).setParameter("monitor", result.monitorId())
+                .setParameter("owner", owner).executeUpdate() == 1;
     }
 
     @Transactional
@@ -51,14 +52,12 @@ public class CheckQueue {
             if (elapsed < monitor.interval()) continue;
             // One current interval slot per tick; no historical catch-up queue is invented.
             Instant due = monitor.createdAt().plusSeconds((elapsed / monitor.interval()) * monitor.interval());
-            boolean exists = entities.createQuery("select c.id from CheckRunEntity c where c.monitorId = :monitor "
-                            + "and c.scheduledFor = :due", java.util.UUID.class)
-                    .setParameter("monitor", monitor.id()).setParameter("due", due).setMaxResults(1)
-                    .getResultStream().findAny().isPresent();
-            if (!exists) {
-                entities.persist(CheckRunEntity.queued(monitor.id(), "SCHEDULED", now, due));
-                created++;
-            }
+            created += entities.createNativeQuery("insert into {h-schema}check_runs "
+                            + "(id, monitor_id, trigger_kind, state, queued_at, scheduled_for) "
+                            + "values (:id, :monitor, 'SCHEDULED', 'QUEUED', :queued, :due) "
+                            + "on conflict (monitor_id, scheduled_for) where scheduled_for is not null do nothing")
+                    .setParameter("id", UUID.randomUUID()).setParameter("monitor", monitor.id())
+                    .setParameter("queued", now.truncatedTo(ChronoUnit.MICROS)).setParameter("due", due).executeUpdate();
         }
         return created;
     }
diff --git a/backend/src/main/java/dev/evolution/monitor/CheckRunEntity.java b/backend/src/main/java/dev/evolution/monitor/CheckRunEntity.java
index 8a7528b..a5b205a 100644
--- a/backend/src/main/java/dev/evolution/monitor/CheckRunEntity.java
+++ b/backend/src/main/java/dev/evolution/monitor/CheckRunEntity.java
@@ -23,6 +23,9 @@ public class CheckRunEntity {
     private Instant queuedAt;
     @Column(name = "scheduled_for", columnDefinition = "timestamp(6) with time zone")
     private Instant scheduledFor;
+    @Column(name = "manual_owner_user_id") private UUID manualOwnerUserId;
+    @Column(name = "idempotency_key", length = 128) private String idempotencyKey;
+    @Column(name = "claim_owner") private UUID claimOwner;
     @Column(name = "started_at", columnDefinition = "timestamp(6) with time zone")
     private Instant startedAt;
     @Column(name = "finished_at", columnDefinition = "timestamp(6) with time zone")
@@ -41,9 +44,10 @@ public class CheckRunEntity {
         return row;
     }
 
-    void start(Instant now) {
+    void start(Instant now, UUID owner) {
         state = "RUNNING";
         startedAt = now.truncatedTo(ChronoUnit.MICROS);
+        claimOwner = owner;
     }
 
     static CheckRunEntity fromDomain(CheckRunner.CheckRun check) {
diff --git a/backend/src/main/java/dev/evolution/monitor/CheckWorker.java b/backend/src/main/java/dev/evolution/monitor/CheckWorker.java
index 36c4dee..9f798d8 100644
--- a/backend/src/main/java/dev/evolution/monitor/CheckWorker.java
+++ b/backend/src/main/java/dev/evolution/monitor/CheckWorker.java
@@ -1,11 +1,13 @@
 package dev.evolution.monitor;
 
+import java.util.UUID;
 import org.springframework.stereotype.Component;
 
 @Component
 public class CheckWorker {
     private final CheckQueue queue;
     private final CheckRunner runner;
+    private final UUID owner = UUID.randomUUID();
 
     CheckWorker(CheckQueue queue, CheckRunner runner) {
         this.queue = queue;
@@ -13,10 +15,12 @@ public class CheckWorker {
     }
 
     public boolean executeNext() {
-        var execution = queue.startNext(); // The RUNNING transaction commits before outbound I/O.
+        var execution = queue.startNext(owner); // The RUNNING transaction commits before outbound I/O.
         if (execution == null) return false;
         var result = runner.run(execution.check(), execution.url());
-        queue.finish(result); // A separate short transaction records only the observed outcome.
+        queue.finish(result, owner); // Only this execution owner may record the observed outcome.
         return true;
     }
+
+    UUID ownerId() { return owner; }
 }
diff --git a/backend/src/main/java/dev/evolution/monitor/MonitorController.java b/backend/src/main/java/dev/evolution/monitor/MonitorController.java
index 45f89fc..c7c4773 100644
--- a/backend/src/main/java/dev/evolution/monitor/MonitorController.java
+++ b/backend/src/main/java/dev/evolution/monitor/MonitorController.java
@@ -13,6 +13,7 @@ import org.springframework.web.bind.annotation.PathVariable;
 import org.springframework.web.bind.annotation.PostMapping;
 import org.springframework.web.bind.annotation.PutMapping;
 import org.springframework.web.bind.annotation.RequestBody;
+import org.springframework.web.bind.annotation.RequestHeader;
 import org.springframework.web.bind.annotation.RequestMapping;
 import org.springframework.web.bind.annotation.RequestParam;
 import org.springframework.web.bind.annotation.ResponseStatus;
@@ -64,9 +65,10 @@ public class MonitorController {
 
     @PostMapping("/{id}/checks")
     @ResponseStatus(HttpStatus.ACCEPTED)
-    public ApiData<CheckRunner.CheckRun> check(@AuthenticationPrincipal UserAccounts.AccountUser user, @PathVariable UUID id) {
+    public ApiData<CheckRunner.CheckRun> check(@AuthenticationPrincipal UserAccounts.AccountUser user,
+            @PathVariable UUID id, @RequestHeader(value = "Idempotency-Key", required = false) String key) {
         // The transaction commits before202; this API process performs no Check I/O.
-        return new ApiData<>(store.enqueueCheck(user.userId(), id));
+        return new ApiData<>(store.enqueueCheck(user.userId(), id, key));
     }
 
     @GetMapping("/{id}/checks")
diff --git a/backend/src/main/java/dev/evolution/monitor/MonitorStore.java b/backend/src/main/java/dev/evolution/monitor/MonitorStore.java
index 28a6902..a333395 100644
--- a/backend/src/main/java/dev/evolution/monitor/MonitorStore.java
+++ b/backend/src/main/java/dev/evolution/monitor/MonitorStore.java
@@ -63,11 +63,29 @@ public class MonitorStore {
     }
 
     @Transactional
-    public CheckRunner.CheckRun enqueueCheck(UUID owner, UUID monitorId) {
+    public CheckRunner.CheckRun enqueueCheck(UUID owner, UUID monitorId, String key) {
+        if (key == null || key.isEmpty() || key.length() > 128
+                || key.chars().anyMatch(character -> character < 33 || character > 126)) {
+            throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "A valid Idempotency-Key is required.");
+        }
         requireMonitor(owner, monitorId);
-        var row = CheckRunEntity.queued(monitorId, "MANUAL", Instant.now(), null);
-        entities.persist(row);
-        return row.toDomain();
+        // The unique identity waits for a concurrent creator's transaction before the fresh read.
+        entities.createNativeQuery("insert into {h-schema}check_runs "
+                        + "(id, monitor_id, trigger_kind, state, queued_at, manual_owner_user_id, idempotency_key) "
+                        + "select :id, m.id, 'MANUAL', 'QUEUED', :queued, :owner, :key from {h-schema}monitors m "
+                        + "where m.id = :monitor and m.owner_user_id = :owner "
+                        + "on conflict (manual_owner_user_id, idempotency_key) do nothing")
+                .setParameter("id", UUID.randomUUID()).setParameter("monitor", monitorId)
+                .setParameter("queued", Instant.now().truncatedTo(ChronoUnit.MICROS))
+                .setParameter("owner", owner).setParameter("key", key).executeUpdate();
+        var check = entities.createQuery("select c from CheckRunEntity c where c.manualOwnerUserId = :owner "
+                        + "and c.idempotencyKey = :key", CheckRunEntity.class)
+                .setParameter("owner", owner).setParameter("key", key)
+                .getResultStream().findFirst().map(CheckRunEntity::toDomain).orElseThrow(MonitorStore::notFound);
+        if (!check.monitorId().equals(monitorId)) {
+            throw new ResponseStatusException(HttpStatus.CONFLICT, "Idempotency-Key already identifies another Monitor.");
+        }
+        return check;
     }
 
     @Transactional
diff --git a/backend/src/main/resources/db/migration/V7__execution_ownership_and_manual_identity.sql b/backend/src/main/resources/db/migration/V7__execution_ownership_and_manual_identity.sql
new file mode 100644
index 0000000..9c04834
--- /dev/null
+++ b/backend/src/main/resources/db/migration/V7__execution_ownership_and_manual_identity.sql
@@ -0,0 +1,11 @@
+-- Historical runs have no recoverable request key or worker identity; preserve their data.
+ALTER TABLE check_runs
+    ADD COLUMN manual_owner_user_id uuid REFERENCES users(id),
+    ADD COLUMN idempotency_key varchar(128),
+    ADD COLUMN claim_owner uuid,
+    ADD CONSTRAINT check_runs_manual_identity UNIQUE (manual_owner_user_id, idempotency_key),
+    ADD CONSTRAINT check_runs_manual_identity_shape CHECK (
+        (manual_owner_user_id IS NULL AND idempotency_key IS NULL)
+        OR (trigger_kind = 'MANUAL' AND manual_owner_user_id IS NOT NULL
+            AND idempotency_key IS NOT NULL AND idempotency_key COLLATE "C" ~ '^[!-~]{1,128}$')
+    );
diff --git a/backend/src/test/java/dev/evolution/monitor/E10WorkerProcess.java b/backend/src/test/java/dev/evolution/monitor/E10WorkerProcess.java
new file mode 100644
index 0000000..cbefe9b
--- /dev/null
+++ b/backend/src/test/java/dev/evolution/monitor/E10WorkerProcess.java
@@ -0,0 +1,61 @@
+package dev.evolution.monitor;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.nio.file.StandardCopyOption;
+import java.time.Instant;
+import java.util.LinkedHashMap;
+import java.util.Map;
+import java.util.UUID;
+import org.springframework.boot.WebApplicationType;
+import org.springframework.boot.builder.SpringApplicationBuilder;
+
+// Test-only startup adapter: a real JVM runs the production worker once after a database gate.
+public class E10WorkerProcess {
+    static final int START_GATE = 101001;
+    static final int LOSER_GATE = 101002;
+
+    public static void main(String[] args) throws Exception {
+        UUID checkId = UUID.fromString(args[0]);
+        UUID monitorId = UUID.fromString(args[1]);
+        Path directory = Path.of(args[2]);
+        String label = args[3];
+        if (!label.equals("one") && !label.equals("two")) throw new IllegalArgumentException("Unknown worker label");
+        try (var application = new SpringApplicationBuilder(MonitorApplication.class)
+                .web(WebApplicationType.NONE).run("--spring.main.banner-mode=off");
+                var barrier = TestDatabase.connect()) {
+            var worker = application.getBean(CheckWorker.class);
+            Map<String, Object> result = new LinkedHashMap<>();
+            result.put("processId", ProcessHandle.current().pid());
+            result.put("ownerId", worker.ownerId().toString());
+            result.put("checkId", checkId.toString());
+            write(directory.resolve(label + "-ready.json"), result);
+            gate(barrier, START_GATE);
+            boolean won = worker.executeNext();
+            result.put("wonClaim", won);
+            if (!won) {
+                gate(barrier, LOSER_GATE);
+                var nonOwner = new UUID(0, 0);
+                var attempted = new CheckRunner.CheckRun(checkId, monitorId, "MANUAL", "FAILED", null,
+                        1L, "TIMEOUT", Instant.now(), Instant.now());
+                result.put("attemptedOwnerId", nonOwner.toString());
+                result.put("terminalWriteChangedRow", application.getBean(CheckQueue.class).finish(attempted, nonOwner));
+            }
+            write(directory.resolve(label + "-result.json"), result);
+        }
+    }
+
+    private static void gate(java.sql.Connection connection, int gate) throws Exception {
+        try (var statement = connection.createStatement()) {
+            statement.execute("SELECT pg_advisory_lock_shared(" + gate + ")");
+            statement.execute("SELECT pg_advisory_unlock_shared(" + gate + ")");
+        }
+    }
+
+    private static void write(Path path, Map<String, Object> value) throws Exception {
+        Path temporary = path.resolveSibling(path.getFileName() + ".tmp");
+        Files.writeString(temporary, new ObjectMapper().writeValueAsString(value) + "\n");
+        Files.move(temporary, path, StandardCopyOption.ATOMIC_MOVE, StandardCopyOption.REPLACE_EXISTING);
+    }
+}
diff --git a/backend/src/test/java/dev/evolution/monitor/ExecutionOwnershipTest.java b/backend/src/test/java/dev/evolution/monitor/ExecutionOwnershipTest.java
new file mode 100644
index 0000000..eb08683
--- /dev/null
+++ b/backend/src/test/java/dev/evolution/monitor/ExecutionOwnershipTest.java
@@ -0,0 +1,293 @@
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
+class ExecutionOwnershipTest {
+    private static final String KEY = "manual-intent-e10-1";
+    private static final String PASSWORD = SessionClient.password();
+    private static final AtomicInteger okCalls = new AtomicInteger();
+    private static final AtomicInteger failCalls = new AtomicInteger();
+    private static final CountDownLatch held = new CountDownLatch(1);
+    private static final CountDownLatch release = new CountDownLatch(1);
+    private static HttpServer fixture;
+    private static ExecutorService fixtureThreads;
+    @Autowired TestRestTemplate api;
+    @Autowired ObjectMapper json;
+
+    @DynamicPropertySource
+    static void database(DynamicPropertyRegistry properties) { TestDatabase.configure(properties, "e10_ownership"); }
+
+    @BeforeAll
+    static void fixture(@Autowired UserAccounts accounts) throws Exception {
+        accounts.bootstrap(PASSWORD, SessionClient.password());
+        fixture = HttpServer.create(new InetSocketAddress("127.0.0.1", 4321), 0);
+        fixtureThreads = Executors.newVirtualThreadPerTaskExecutor();
+        fixture.setExecutor(fixtureThreads);
+        fixture.createContext("/ok", exchange -> {
+            okCalls.incrementAndGet();
+            held.countDown();
+            try {
+                if (!release.await(30, TimeUnit.SECONDS)) throw new IllegalStateException("Fixture release did not arrive");
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
+        release.countDown();
+        if (fixture != null) fixture.stop(0);
+        if (fixtureThreads != null) fixtureThreads.close();
+        TestDatabase.drop("e10_ownership");
+    }
+
+    @Test
+    void parallelIdentityAndTwoRealWorkersRetainOneIntentAndOneExecutionOwner() throws Exception {
+        Map<String, Object> evidence = new LinkedHashMap<>();
+        var completed = new ArrayList<String>();
+        var processes = new ArrayList<Process>();
+        Path directory = Path.of("target/e10-workers").toAbsolutePath();
+        Files.createDirectories(directory);
+        for (String label : List.of("one", "two")) {
+            Files.deleteIfExists(directory.resolve(label + "-ready.json"));
+            Files.deleteIfExists(directory.resolve(label + "-result.json"));
+        }
+        evidence.put("fixtureSha256", HexFormat.of().formatHex(MessageDigest.getInstance("SHA-256")
+                .digest(Files.readAllBytes(Path.of("../evidence/phase-1/E10/fixtures.md")))));
+        evidence.put("completed", completed);
+        evidence.put("workerEntry", "two non-web JVMs, test-only startup gates, production CheckWorker.executeNext once");
+        evidence.put("result", "INCOMPLETE");
+        try {
+            var alice = new SessionClient(api);
+            data(alice.login("alice-e04", PASSWORD), 200);
+            String a = create(alice, "A", "/ok", true);
+            String b = create(alice, "B", "/fail", false);
+            alice.csrf();
+            var firstClient = alice.copyCredential();
+            var secondClient = alice.copyCredential();
+            JsonNode first;
+            JsonNode second;
+            try (var requests = Executors.newVirtualThreadPerTaskExecutor(); var lock = TestDatabase.connect()) {
+                lock.setAutoCommit(false);
+                lock.createStatement().execute("LOCK TABLE e10_ownership.check_runs IN SHARE MODE");
+                var ready = new CountDownLatch(2);
+                var start = new CountDownLatch(1);
+                var firstRequest = requests.submit(() -> { ready.countDown(); start.await(); return firstClient.sendCheck(a, KEY); });
+                var secondRequest = requests.submit(() -> { ready.countDown(); start.await(); return secondClient.sendCheck(a, KEY); });
+                assertTrue(ready.await(30, TimeUnit.SECONDS));
+                start.countDown();
+                await(() -> integer("SELECT count(*) FROM pg_locks WHERE relation='e10_ownership.check_runs'::regclass "
+                        + "AND mode='RowExclusiveLock' AND NOT granted") == 2);
+                lock.commit();
+                first = data(firstRequest.get(30, TimeUnit.SECONDS), 202);
+                second = data(secondRequest.get(30, TimeUnit.SECONDS), 202);
+            }
+            String id = first.get("id").textValue();
+            assertEquals(id, second.get("id").textValue());
+            assertEquals("QUEUED", first.get("state").textValue());
+            assertEquals("QUEUED", second.get("state").textValue());
+            assertEquals(1, count());
+            assertEquals(0, okCalls.get() + failCalls.get());
+            evidence.put("parallel", Map.of("blockedInsertTransactions", 2, "statuses", List.of(202, 202),
+                    "sameId", true, "persistedRows", count(), "outboundRequests", okCalls.get() + failCalls.get()));
+            completed.add("same-owner/key parallel requests deduplicated by PostgreSQL after both inserts reached the lock barrier");
+
+            JsonNode beforeRejected = row(id);
+            var conflict = alice.sendCheck(b, KEY);
+            assertEquals(409, conflict.getStatusCode().value());
+            assertEquals("CONFLICT", conflict.getBody().at("/error/code").textValue());
+            assertEquals(beforeRejected, row(id));
+            data(alice.mutate("/api/monitors/" + a, HttpMethod.PUT,
+                    new MonitorController.CreateMonitor("A edited", "http://127.0.0.1:4321/ok", 120, false)), 200);
+            assertEquals(id, data(alice.sendCheck(a, KEY), 202).get("id").textValue());
+            int rejected = 0;
+            for (String invalid : new String[]{null, "", "has space", "é", "x".repeat(129)}) {
+                var response = alice.sendCheck(a, invalid);
+                assertEquals(400, response.getStatusCode().value());
+                assertEquals("INVALID_INPUT", response.getBody().at("/error/code").textValue());
+                assertEquals(beforeRejected, row(id));
+                rejected++;
+            }
+            assertEquals(1, count());
+            assertEquals(0, integer("SELECT count(*) FROM e10_ownership.check_runs WHERE monitor_id='" + b + "'"));
+            assertEquals(0, okCalls.get() + failCalls.get());
+            evidence.put("identityMeaning", Map.of("otherTargetStatus", 409, "otherTargetRows", 0,
+                    "mutableMonitorReplaySameId", true, "invalidInputsRejected", rejected, "persistedRows", count()));
+            completed.add("changed target conflicts; mutable Monitor fields do not redefine identity; invalid keys cause no write");
+
+            try (var gates = TestDatabase.connect(); var statement = gates.createStatement()) {
+                statement.execute("SELECT pg_advisory_lock(" + E10WorkerProcess.START_GATE + ")");
+                statement.execute("SELECT pg_advisory_lock(" + E10WorkerProcess.LOSER_GATE + ")");
+                for (String label : List.of("one", "two")) {
+                    var builder = new ProcessBuilder(Path.of(System.getProperty("java.home"), "bin/java").toString(),
+                            "-cp", System.getProperty("java.class.path"), E10WorkerProcess.class.getName(),
+                            id, a, directory.toString(), label);
+                    builder.environment().put("DB_SCHEMA", "e10_ownership");
+                    builder.environment().remove("E04_ALICE_PASSWORD");
+                    builder.environment().remove("E04_BOB_PASSWORD");
+                    builder.redirectErrorStream(true).redirectOutput(directory.resolve(label + ".log").toFile());
+                    processes.add(builder.start());
+                }
+                await(() -> {
+                    assertTrue(processes.stream().allMatch(Process::isAlive), "Both owned JVMs must reach their startup gate");
+                    return integer("SELECT count(*) FROM pg_locks WHERE locktype='advisory' AND classid=0 AND objid="
+                            + E10WorkerProcess.START_GATE + " AND mode='ShareLock' AND NOT granted") == 2;
+                });
+                JsonNode readyOne = json.readTree(directory.resolve("one-ready.json").toFile());
+                JsonNode readyTwo = json.readTree(directory.resolve("two-ready.json").toFile());
+                assertEquals(processes.get(0).pid(), readyOne.get("processId").longValue());
+                assertEquals(processes.get(1).pid(), readyTwo.get("processId").longValue());
+                assertNotEquals(readyOne.get("processId"), readyTwo.get("processId"));
+                assertNotEquals(readyOne.get("ownerId"), readyTwo.get("ownerId"));
+                assertTrue(processes.stream().noneMatch(process -> process.pid() == ProcessHandle.current().pid()));
+                evidence.put("workersReady", List.of(readyOne, readyTwo));
+                evidence.put("blockedReadyWorkers", 2);
+                statement.execute("SELECT pg_advisory_unlock(" + E10WorkerProcess.START_GATE + ")");
+                assertTrue(held.await(30, TimeUnit.SECONDS));
+                JsonNode running = row(id);
+                assertEquals("RUNNING", running.get("state").textValue());
+                assertFalse(running.get("claimOwner").isNull());
+                assertTrue(running.get("finishedAt").isNull());
+                assertEquals(1, okCalls.get());
+                assertEquals(0, failCalls.get());
+                JsonNode replayRunning = data(alice.sendCheck(a, KEY), 202);
+                assertEquals(id, replayRunning.get("id").textValue());
+                assertEquals("RUNNING", replayRunning.get("state").textValue());
+                statement.execute("SELECT pg_advisory_unlock(" + E10WorkerProcess.LOSER_GATE + ")");
+                await(() -> Files.exists(directory.resolve("one-result.json")) || Files.exists(directory.resolve("two-result.json")));
+                String loserLabel = Files.exists(directory.resolve("one-result.json")) ? "one" : "two";
+                JsonNode loser = json.readTree(directory.resolve(loserLabel + "-result.json").toFile());
+                assertFalse(loser.get("wonClaim").booleanValue());
+                assertFalse(loser.get("terminalWriteChangedRow").booleanValue());
+                assertNotEquals(loser.get("ownerId"), running.get("claimOwner"));
+                assertEquals(new UUID(0, 0).toString(), loser.get("attemptedOwnerId").textValue());
+                assertEquals(running, row(id), "The losing process cannot change the held owner's terminal outcome");
+                evidence.put("whileHeld", Map.of("sameId", true, "state", "RUNNING", "persistedRows", count(),
+                        "fixtureRequests", okCalls.get(), "currentReplayState", replayRunning.get("state").textValue(),
+                        "loser", loser, "ownerAndOutcomeUnchanged", true));
+                release.countDown();
+                for (Process process : processes) {
+                    assertTrue(process.waitFor(30, TimeUnit.SECONDS), "Owned worker must exit after its single attempt");
+                    assertEquals(0, process.exitValue());
+                }
+                List<JsonNode> workers = List.of(json.readTree(directory.resolve("one-result.json").toFile()),
+                        json.readTree(directory.resolve("two-result.json").toFile()));
+                assertEquals(1, workers.stream().filter(worker -> worker.get("wonClaim").booleanValue()).count());
+                JsonNode winner = workers.stream().filter(worker -> worker.get("wonClaim").booleanValue()).findFirst().orElseThrow();
+                assertEquals(winner.get("ownerId"), row(id).get("claimOwner"));
+                evidence.put("workersCompleted", workers);
+            }
+            JsonNode terminal = data(alice.sendCheck(a, KEY), 202);
+            assertEquals(id, terminal.get("id").textValue());
+            assertEquals("SUCCEEDED", terminal.get("state").textValue());
+            assertEquals(200, terminal.get("httpStatus").intValue());
+            assertEquals(1, count());
+            assertEquals(1, okCalls.get());
+            assertEquals(0, failCalls.get());
+            completed.add("two actual workers produced one owner/outbound/result; losing process completion was rejected while RUNNING");
+            evidence.put("terminal", Map.of("sameId", true, "state", terminal.get("state").textValue(),
+                    "httpStatus", terminal.get("httpStatus").intValue(), "rows", count(), "outboundRequests", okCalls.get()));
+
+            String newKey = "n".repeat(128);
+            JsonNode fresh = data(alice.sendCheck(a, newKey), 202);
+            assertNotEquals(id, fresh.get("id").textValue());
+            assertEquals("QUEUED", fresh.get("state").textValue());
+            assertEquals(fresh.get("id"), data(alice.sendCheck(a, newKey), 202).get("id"));
+            assertEquals(id, data(alice.sendCheck(a, KEY), 202).get("id").textValue());
+            assertEquals(2, count());
+            assertEquals(1, okCalls.get());
+            evidence.put("nextIntent", Map.of("keyLength", 128, "newId", true, "replaySameId", true,
+                    "originalIdentityRetained", true, "persistedRows", count(), "outboundRequests", okCalls.get()));
+            completed.add("terminal retransmission retains current identity; a different valid key adds one new queued intent");
+            evidence.put("result", "PASS");
+        } finally {
+            release.countDown();
+            for (Process process : processes) {
+                if (process.isAlive()) process.destroy();
+                if (!process.waitFor(5, TimeUnit.SECONDS)) {
+                    process.destroyForcibly();
+                    assertTrue(process.waitFor(5, TimeUnit.SECONDS));
+                }
+            }
+            evidence.put("allOwnedWorkerExitsAwaited", processes.stream().noneMatch(Process::isAlive));
+            Files.writeString(Path.of("target/e10-ownership.json"), json.writerWithDefaultPrettyPrinter().writeValueAsString(evidence) + "\n");
+        }
+    }
+
+    private String create(SessionClient client, String name, String path, boolean enabled) {
+        return data(client.mutate("/api/monitors", HttpMethod.POST,
+                new MonitorController.CreateMonitor(name, "http://127.0.0.1:4321" + path, 60, enabled)), 201)
+                .at("/monitor/id").textValue();
+    }
+
+    private static JsonNode data(ResponseEntity<JsonNode> response, int status) {
+        assertEquals(status, response.getStatusCode().value());
+        return response.getBody().get("data");
+    }
+
+    private int count() throws Exception { return integer("SELECT count(*) FROM e10_ownership.check_runs"); }
+
+    private JsonNode row(String id) throws Exception {
+        try (var connection = TestDatabase.connect(); var statement = connection.prepareStatement(
+                "SELECT json_build_object('id',id,'state',state,'claimOwner',claim_owner,'startedAt',started_at,"
+                + "'finishedAt',finished_at,'httpStatus',http_status,'latencyMs',latency_ms,'failureReason',failure_reason)::text "
+                + "FROM e10_ownership.check_runs WHERE id=?")) {
+            statement.setObject(1, UUID.fromString(id));
+            try (var rows = statement.executeQuery()) { assertTrue(rows.next()); return json.readTree(rows.getString(1)); }
+        }
+    }
+
+    private static int integer(String sql) throws Exception {
+        try (var connection = TestDatabase.connect(); var statement = connection.createStatement(); var rows = statement.executeQuery(sql)) {
+            assertTrue(rows.next());
+            return rows.getInt(1);
+        }
+    }
+
+    @FunctionalInterface private interface Checkpoint { boolean reached() throws Exception; }
+    private static void await(Checkpoint checkpoint) throws Exception {
+        long deadline = System.nanoTime() + TimeUnit.SECONDS.toNanos(30);
+        while (System.nanoTime() < deadline) {
+            if (checkpoint.reached()) return;
+            Thread.sleep(25);
+        }
+        fail("Frozen E10 checkpoint did not complete");
+    }
+}
diff --git a/backend/src/test/java/dev/evolution/monitor/HistoryIndexMigrationTest.java b/backend/src/test/java/dev/evolution/monitor/HistoryIndexMigrationTest.java
index 5581a08..6c68c05 100644
--- a/backend/src/test/java/dev/evolution/monitor/HistoryIndexMigrationTest.java
+++ b/backend/src/test/java/dev/evolution/monitor/HistoryIndexMigrationTest.java
@@ -70,6 +70,24 @@ class HistoryIndexMigrationTest {
                      "repeatMigrations":0,"sevenTerminalRowsAndTimestampsUnchanged":true,
                      "priorMigrationChecksumsUnchanged":true,"queuedAtBackfilledFromFinishedAt":true}
                     """);
+            String beforeOwnership = scalar("SELECT json_agg(c ORDER BY c.id)::text FROM e07_index_upgrade.check_runs c");
+            String checksumsBeforeOwnership = scalar("SELECT json_agg(c ORDER BY c.version)::text FROM "
+                    + "(SELECT version,checksum FROM e07_index_upgrade.flyway_schema_history WHERE success) c");
+            var ownershipUpgrade = TestDatabase.migration(schema).target("7").load();
+            assertEquals(1, ownershipUpgrade.migrate().migrationsExecuted);
+            assertEquals(0, ownershipUpgrade.migrate().migrationsExecuted);
+            String afterOwnership = scalar("SELECT json_agg(to_jsonb(c)-'manual_owner_user_id'-'idempotency_key'-'claim_owner' "
+                    + "ORDER BY c.id)::text FROM e07_index_upgrade.check_runs c");
+            assertEquals(json.readTree(beforeOwnership), json.readTree(afterOwnership));
+            assertEquals(checksumsBeforeOwnership, scalar("SELECT json_agg(c ORDER BY c.version)::text FROM "
+                    + "(SELECT version,checksum FROM e07_index_upgrade.flyway_schema_history WHERE success AND version <> '7') c"));
+            assertEquals("7", scalar("SELECT count(*) FROM e07_index_upgrade.check_runs "
+                    + "WHERE manual_owner_user_id IS NULL AND idempotency_key IS NULL AND claim_owner IS NULL"));
+            Files.writeString(Path.of("target/e10-ownership-migration.json"), """
+                    {"result":"PASS","upgradeFrom":6,"upgradeTo":7,"migrationsExecuted":1,"repeatMigrations":0,
+                     "sevenExistingRowsIncludingQueueMetadataUnchanged":true,"priorMigrationChecksumsUnchanged":true,
+                     "noHistoricalRequestOrWorkerIdentityInvented":true}
+                    """);
         } finally { TestDatabase.drop(schema); }
     }
 
diff --git a/backend/src/test/java/dev/evolution/monitor/OwnershipAuthorizationTest.java b/backend/src/test/java/dev/evolution/monitor/OwnershipAuthorizationTest.java
index a823612..a9d11e6 100644
--- a/backend/src/test/java/dev/evolution/monitor/OwnershipAuthorizationTest.java
+++ b/backend/src/test/java/dev/evolution/monitor/OwnershipAuthorizationTest.java
@@ -109,8 +109,8 @@ class OwnershipAuthorizationTest {
                 .at("/monitor/id").textValue();
         bobId = data(bob.mutate("/api/monitors", HttpMethod.POST, input("Monitor B", "/fail", 120, true)), 201)
                 .at("/monitor/id").textValue();
-        JsonNode a = completedCheck(alice, aliceId);
-        JsonNode b = completedCheck(bob, bobId);
+        JsonNode a = completedCheck(alice, aliceId, "owner-scope-e10-1");
+        JsonNode b = completedCheck(bob, bobId, "owner-scope-e10-1");
         aliceCheck = a.get("id").textValue();
         bobCheck = b.get("id").textValue();
         assertEquals("SUCCEEDED", a.get("state").textValue());
@@ -374,7 +374,12 @@ class OwnershipAuthorizationTest {
     }
 
     private JsonNode completedCheck(SessionClient client, String monitorId) {
-        JsonNode accepted = data(client.mutate(path(monitorId) + "/checks", HttpMethod.POST, null), 202);
+        return completedCheck(client, monitorId, UUID.randomUUID().toString());
+    }
+
+    private JsonNode completedCheck(SessionClient client, String monitorId, String key) {
+        client.csrf();
+        JsonNode accepted = data(client.sendCheck(monitorId, key), 202);
         assertEquals("QUEUED", accepted.get("state").textValue());
         // Only trusted worker setup is classified separately; all user/store SQL stays owner-audited.
         SqlEvidence.workerScope.set(true);
diff --git a/backend/src/test/java/dev/evolution/monitor/SessionClient.java b/backend/src/test/java/dev/evolution/monitor/SessionClient.java
index 0bfd5f5..ba6ce86 100644
--- a/backend/src/test/java/dev/evolution/monitor/SessionClient.java
+++ b/backend/src/test/java/dev/evolution/monitor/SessionClient.java
@@ -6,6 +6,7 @@ import com.fasterxml.jackson.databind.JsonNode;
 import java.security.SecureRandom;
 import java.util.Base64;
 import java.util.Map;
+import java.util.UUID;
 import org.springframework.boot.test.web.client.TestRestTemplate;
 import org.springframework.http.HttpEntity;
 import org.springframework.http.HttpHeaders;
@@ -75,11 +76,22 @@ final class SessionClient {
     }
 
     ResponseEntity<JsonNode> sendWithEvidence(String path, HttpMethod method, Object body, String origin, SessionClient proof) {
+        String key = method == HttpMethod.POST && path.endsWith("/checks") ? UUID.randomUUID().toString() : null;
+        return sendWithEvidence(path, method, body, origin, proof, key);
+    }
+
+    ResponseEntity<JsonNode> sendCheck(String monitorId, String key) {
+        return sendWithEvidence("/api/monitors/" + monitorId + "/checks", HttpMethod.POST, null, TRUSTED_ORIGIN, this, key);
+    }
+
+    private ResponseEntity<JsonNode> sendWithEvidence(String path, HttpMethod method, Object body,
+            String origin, SessionClient proof, String key) {
         var headers = new HttpHeaders();
         if (cookie != null) headers.set(HttpHeaders.COOKIE, cookie);
         if (origin != null) headers.setOrigin(origin);
         if (proof != null) headers.set(proof.csrfHeader, proof.csrfToken);
         if (body != null) headers.setContentType(MediaType.APPLICATION_JSON);
+        if (key != null) headers.set("Idempotency-Key", key);
         var response = api.exchange(path, method, new HttpEntity<>(body, headers), JsonNode.class);
         for (String value : response.getHeaders().getOrEmpty(HttpHeaders.SET_COOKIE)) {
             if (value.startsWith(AuthenticationConfig.COOKIE_NAME + "=")) cookie = value.split(";", 2)[0];
@@ -95,6 +107,9 @@ final class SessionClient {
                 request.getHeaders().setOrigin(TRUSTED_ORIGIN);
                 request.getHeaders().set(csrfHeader, csrfToken);
             }
+            if (request.getMethod() == HttpMethod.POST && request.getURI().getPath().endsWith("/checks")) {
+                request.getHeaders().set("Idempotency-Key", UUID.randomUUID().toString());
+            }
             return execution.execute(request, body);
         });
     }
diff --git a/backend/src/test/java/dev/evolution/monitor/TestDatabase.java b/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
index 50e70c0..e27b8e6 100644
--- a/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
+++ b/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
@@ -13,7 +13,7 @@ final class TestDatabase {
             "e04_missing_column", "e04_extra_required", "e04_session", "e04_users",
             "e04_missing_user_column", "e04_extra_user_required", "e05_ownership", "e05_fresh",
             "e05_upgrade", "e05_missing_owner", "e05_nullable_owner", "e05_missing_owner_fk", "e07_history", "e07_index_upgrade",
-            "e09_scheduler");
+            "e09_scheduler", "e10_ownership");
 
     static Connection connect() throws SQLException {
         return DriverManager.getConnection(System.getenv().getOrDefault("DB_URL", "jdbc:postgresql://127.0.0.1:15432/monitor"),


