## `feat(history): bound owner history with stable cursors`

diff --git a/backend/src/main/java/dev/evolution/monitor/HistoryQuery.java b/backend/src/main/java/dev/evolution/monitor/HistoryQuery.java
new file mode 100644
index 0000000..8339e0b
--- /dev/null
+++ b/backend/src/main/java/dev/evolution/monitor/HistoryQuery.java
@@ -0,0 +1,39 @@
+package dev.evolution.monitor;
+
+import java.nio.charset.StandardCharsets;
+import java.time.Instant;
+import java.util.Base64;
+import java.util.UUID;
+import org.springframework.http.HttpStatus;
+import org.springframework.web.server.ResponseStatusException;
+
+// Pagination state is not an authorization credential. The store always scopes
+// the query to the authenticated owner, independently of these cursor fields.
+record HistoryQuery(int limit, String state, Instant beforeTime, UUID beforeId) {
+    static HistoryQuery parse(UUID monitor, String rawLimit, String state, String cursor) {
+        try {
+            int limit = rawLimit == null ? 20 : Integer.parseInt(rawLimit);
+            if (limit < 1 || limit > 100 || (state != null && !state.equals("SUCCEEDED") && !state.equals("FAILED"))) {
+                throw new IllegalArgumentException();
+            }
+            if (cursor == null) return new HistoryQuery(limit, state, null, null);
+            String[] fields = new String(Base64.getUrlDecoder().decode(cursor), StandardCharsets.UTF_8).split("\\|", -1);
+            if (fields.length != 6 || !fields[0].equals("1") || !fields[1].equals(monitor.toString())
+                    || !fields[2].equals(state == null ? "" : state) || !fields[3].equals(Integer.toString(limit))) {
+                throw new IllegalArgumentException();
+            }
+            Instant time = Instant.parse(fields[4]);
+            UUID id = UUID.fromString(fields[5]);
+            if (!fields[5].equals(id.toString())) throw new IllegalArgumentException();
+            return new HistoryQuery(limit, state, time, id);
+        } catch (IllegalArgumentException | java.time.DateTimeException error) {
+            throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "Invalid history limit, state, or cursor.");
+        }
+    }
+
+    String nextCursor(UUID monitor, CheckRunner.CheckRun last) {
+        String fields = String.join("|", "1", monitor.toString(), state == null ? "" : state,
+                Integer.toString(limit), last.finishedAt().toString(), last.id().toString());
+        return Base64.getUrlEncoder().withoutPadding().encodeToString(fields.getBytes(StandardCharsets.UTF_8));
+    }
+}
diff --git a/backend/src/main/java/dev/evolution/monitor/MonitorController.java b/backend/src/main/java/dev/evolution/monitor/MonitorController.java
index 2eaad18..6614e9d 100644
--- a/backend/src/main/java/dev/evolution/monitor/MonitorController.java
+++ b/backend/src/main/java/dev/evolution/monitor/MonitorController.java
@@ -5,6 +5,7 @@ import java.net.URI;
 import java.util.List;
 import java.util.UUID;
 import org.springframework.http.HttpStatus;
+import org.springframework.http.ResponseEntity;
 import org.springframework.security.core.annotation.AuthenticationPrincipal;
 import org.springframework.web.bind.annotation.DeleteMapping;
 import org.springframework.web.bind.annotation.GetMapping;
@@ -13,6 +14,7 @@ import org.springframework.web.bind.annotation.PostMapping;
 import org.springframework.web.bind.annotation.PutMapping;
 import org.springframework.web.bind.annotation.RequestBody;
 import org.springframework.web.bind.annotation.RequestMapping;
+import org.springframework.web.bind.annotation.RequestParam;
 import org.springframework.web.bind.annotation.ResponseStatus;
 import org.springframework.web.bind.annotation.RestController;
 import org.springframework.web.server.ResponseStatusException;
@@ -69,8 +71,13 @@ public class MonitorController {
     }
 
     @GetMapping("/{id}/checks")
-    public ApiData<List<CheckRunner.CheckRun>> history(@AuthenticationPrincipal UserAccounts.AccountUser user, @PathVariable UUID id) {
-        return new ApiData<>(store.history(user.userId(), id));
+    public ResponseEntity<ApiData<List<CheckRunner.CheckRun>>> history(@AuthenticationPrincipal UserAccounts.AccountUser user,
+            @PathVariable UUID id, @RequestParam(required = false) String limit,
+            @RequestParam(required = false) String state, @RequestParam(required = false) String cursor) {
+        var page = store.historyPage(user.userId(), id, limit, state, cursor);
+        var response = ResponseEntity.ok();
+        if (page.nextCursor() != null) response.header("X-Next-Cursor", page.nextCursor());
+        return response.body(new ApiData<>(page.items()));
     }
 
     @GetMapping("/{id}/checks/{checkId}")
diff --git a/backend/src/main/java/dev/evolution/monitor/MonitorStore.java b/backend/src/main/java/dev/evolution/monitor/MonitorStore.java
index f0c274e..5ba82fd 100644
--- a/backend/src/main/java/dev/evolution/monitor/MonitorStore.java
+++ b/backend/src/main/java/dev/evolution/monitor/MonitorStore.java
@@ -71,14 +71,31 @@ public class MonitorStore {
     }
 
     public List<CheckRunner.CheckRun> history(UUID owner, UUID id) {
+        return historyPage(owner, id, null, null, null).items();
+    }
+
+    public HistoryPage historyPage(UUID owner, UUID id, String limit, String state, String cursor) {
         requireMonitor(owner, id);
-        return entities.createQuery("select c from CheckRunEntity c, MonitorEntity m "
-                        + "where c.monitorId = m.id and m.id = :id and m.ownerUserId = :owner "
+        var window = HistoryQuery.parse(id, limit, state, cursor);
+        String conditions = "where c.monitorId = m.id and m.id = :id and m.ownerUserId = :owner ";
+        if (window.state() != null) conditions += "and c.state = :state ";
+        if (window.beforeTime() != null) conditions += "and (c.finishedAt < :beforeTime "
+                + "or (c.finishedAt = :beforeTime and c.id < :beforeId)) ";
+        var query = entities.createQuery("select c from CheckRunEntity c, MonitorEntity m " + conditions
                         + "order by c.finishedAt desc, c.id desc", CheckRunEntity.class)
-                .setParameter("id", id).setParameter("owner", owner).getResultList().stream()
-                .map(CheckRunEntity::toDomain).toList();
+                .setParameter("id", id).setParameter("owner", owner).setMaxResults(window.limit() + 1);
+        if (window.state() != null) query.setParameter("state", window.state());
+        if (window.beforeTime() != null) {
+            query.setParameter("beforeTime", window.beforeTime()).setParameter("beforeId", window.beforeId());
+        }
+        var rows = query.getResultList();
+        var items = rows.stream().limit(window.limit()).map(CheckRunEntity::toDomain).toList();
+        String next = rows.size() > window.limit() ? window.nextCursor(id, items.getLast()) : null;
+        return new HistoryPage(items, next);
     }
 
+    public record HistoryPage(List<CheckRunner.CheckRun> items, String nextCursor) {}
+
     public CheckRunner.CheckRun check(UUID owner, UUID monitorId, UUID checkId) {
         return entities.createQuery("select c from CheckRunEntity c, MonitorEntity m where c.id = :checkId "
                         + "and c.monitorId = :monitorId and c.monitorId = m.id and m.ownerUserId = :owner",
diff --git a/backend/src/main/resources/db/migration/V5__index_check_history.sql b/backend/src/main/resources/db/migration/V5__index_check_history.sql
new file mode 100644
index 0000000..54c4aaa
--- /dev/null
+++ b/backend/src/main/resources/db/migration/V5__index_check_history.sql
@@ -0,0 +1,2 @@
+CREATE INDEX check_runs_history_order_idx ON check_runs (monitor_id, finished_at DESC, id DESC);
+CREATE INDEX check_runs_history_state_order_idx ON check_runs (monitor_id, state, finished_at DESC, id DESC);
diff --git a/backend/src/test/java/dev/evolution/monitor/HistoryIndexMigrationTest.java b/backend/src/test/java/dev/evolution/monitor/HistoryIndexMigrationTest.java
new file mode 100644
index 0000000..66a9de0
--- /dev/null
+++ b/backend/src/test/java/dev/evolution/monitor/HistoryIndexMigrationTest.java
@@ -0,0 +1,73 @@
+package dev.evolution.monitor;
+
+import static org.junit.jupiter.api.Assertions.*;
+
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
+
+class HistoryIndexMigrationTest {
+    @Test
+    void v5AddsIndexesWithoutChangingExistingRowsOrMigrationChecksums() throws Exception {
+        String schema = "e07_index_upgrade";
+        TestDatabase.reset(schema);
+        try {
+            assertEquals(4, TestDatabase.migration(schema).target("4").load().migrate().migrationsExecuted);
+            UUID owner = UUID.randomUUID();
+            UUID monitor = UUID.randomUUID();
+            try (var connection = TestDatabase.connect(); var user = connection.prepareStatement(
+                    "INSERT INTO e07_index_upgrade.users (id,username,password_hash) VALUES (?, 'alice-e04', ?)")) {
+                user.setObject(1, owner);
+                user.setString(2, new BCryptPasswordEncoder(10).encode(SessionClient.password()));
+                assertEquals(1, user.executeUpdate());
+            }
+            TestDatabase.execute("INSERT INTO e07_index_upgrade.monitors VALUES ('" + monitor
+                    + "','History fixture','http://127.0.0.1:4321/ok',60,true,"
+                    + "'2026-08-28T00:00:00.000Z','2026-08-28T00:00:00.000Z','" + owner + "')");
+            TestDatabase.execute("""
+                    INSERT INTO e07_index_upgrade.check_runs
+                    SELECT ('00000000-0000-4000-8000-' || lpad(n::text,12,'0'))::uuid,
+                    '%s'::uuid, 'MANUAL', CASE WHEN n%%2=1 THEN 'SUCCEEDED' ELSE 'FAILED' END,
+                    CASE WHEN n%%2=1 THEN 200 ELSE 503 END, 1,
+                    CASE WHEN n%%2=1 THEN NULL ELSE 'HTTP_STATUS' END,
+                    '2026-08-28T00:00:00.000Z'::timestamptz, '2026-08-28T00:00:00.000Z'::timestamptz
+                    FROM generate_series(1,7) n
+                    """.formatted(monitor));
+            String rowsBefore = rows();
+            String migrationsBefore = oldMigrations();
+            var upgrade = TestDatabase.migration(schema).load();
+            assertEquals(1, upgrade.migrate().migrationsExecuted);
+            upgrade.validate();
+            assertEquals(5, upgrade.info().applied().length);
+            assertEquals(0, upgrade.migrate().migrationsExecuted);
+            assertEquals(rowsBefore, rows());
+            assertEquals(migrationsBefore, oldMigrations());
+            assertEquals("2", scalar("SELECT count(*) FROM pg_indexes WHERE schemaname='e07_index_upgrade' "
+                    + "AND indexname IN ('check_runs_history_order_idx','check_runs_history_state_order_idx')"));
+            Files.writeString(Path.of("target/e07-index-migration.json"), """
+                    {"result":"PASS","upgradeFrom":4,"upgradeTo":5,"migrationsExecuted":1,
+                     "repeatMigrations":0,"sevenHistoricalRowsUnchanged":true,
+                     "monitorUnchanged":true,"priorMigrationChecksumsUnchanged":true,"newIndexes":2}
+                    """);
+        } finally { TestDatabase.drop(schema); }
+    }
+
+    private static String rows() throws Exception {
+        return scalar("SELECT json_agg(m)::text FROM (SELECT * FROM e07_index_upgrade.monitors ORDER BY id) m")
+                + scalar("SELECT json_agg(c)::text FROM (SELECT * FROM e07_index_upgrade.check_runs ORDER BY id) c");
+    }
+
+    private static String oldMigrations() throws Exception {
+        return scalar("SELECT json_agg(m)::text FROM (SELECT version,checksum FROM e07_index_upgrade.flyway_schema_history "
+                + "WHERE version IN ('1','2','3','4') ORDER BY installed_rank) m");
+    }
+
+    private static String scalar(String sql) throws Exception {
+        try (var connection = TestDatabase.connect(); var query = connection.createStatement(); var rows = query.executeQuery(sql)) {
+            assertTrue(rows.next());
+            return rows.getString(1);
+        }
+    }
+}
diff --git a/backend/src/test/java/dev/evolution/monitor/HistoryPaginationTest.java b/backend/src/test/java/dev/evolution/monitor/HistoryPaginationTest.java
index 4966b9b..42e6e41 100644
--- a/backend/src/test/java/dev/evolution/monitor/HistoryPaginationTest.java
+++ b/backend/src/test/java/dev/evolution/monitor/HistoryPaginationTest.java
@@ -6,9 +6,15 @@ import com.fasterxml.jackson.databind.JsonNode;
 import com.fasterxml.jackson.databind.ObjectMapper;
 import java.nio.file.Files;
 import java.nio.file.Path;
+import java.nio.charset.StandardCharsets;
+import java.util.ArrayList;
+import java.util.Base64;
 import java.util.List;
 import java.util.Map;
 import java.util.UUID;
+import java.util.concurrent.CopyOnWriteArrayList;
+import org.hibernate.resource.jdbc.spi.StatementInspector;
+import org.springframework.aop.support.AopUtils;
 import org.junit.jupiter.api.AfterAll;
 import org.junit.jupiter.api.BeforeAll;
 import org.junit.jupiter.api.Test;
@@ -25,17 +31,20 @@ import org.springframework.test.context.DynamicPropertySource;
 @DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_CLASS)
 class HistoryPaginationTest {
     private static final String PASSWORD = SessionClient.password();
+    private static final String BOB_PASSWORD = SessionClient.password();
     @Autowired TestRestTemplate api;
     @Autowired ObjectMapper json;
+    @Autowired MonitorStore store;
 
     @DynamicPropertySource
     static void database(DynamicPropertyRegistry properties) {
         TestDatabase.configure(properties, "e07_history");
+        properties.add("spring.jpa.properties.hibernate.session_factory.statement_inspector", () -> SqlEvidence.class.getName());
     }
 
     @BeforeAll
     static void accounts(@Autowired UserAccounts accounts) {
-        accounts.bootstrap(PASSWORD, SessionClient.password());
+        accounts.bootstrap(PASSWORD, BOB_PASSWORD);
     }
 
     @AfterAll
@@ -49,6 +58,7 @@ class HistoryPaginationTest {
                 "name", "History fixture", "url", "http://127.0.0.1:4321/ok", "interval", 60, "enabled", true)), 201)
                 .at("/monitor/id").textValue();
         seed(monitor, 1, 7);
+        SqlEvidence.statements.clear();
         var first = alice.get(path(monitor) + "?limit=3");
         JsonNode rows = data(first, 200);
         if (Boolean.getBoolean("e07.baseline")) {
@@ -60,7 +70,86 @@ class HistoryPaginationTest {
         }
         assertEquals(3, rows.size(), "The explicit first page must be bounded to three rows");
         assertEquals(List.of(id(7), id(6), id(5)), ids(rows));
-        assertNotNull(first.getHeaders().getFirst("X-Next-Cursor"));
+        String cursor = next(first);
+        assertNotNull(cursor);
+        if (Boolean.getBoolean("e07.baseline")) return;
+
+        var second = alice.get(path(monitor) + "?limit=3&cursor=" + cursor);
+        assertEquals(List.of(id(4), id(3), id(2)), ids(data(second, 200)));
+        var third = alice.get(path(monitor) + "?limit=3&cursor=" + next(second));
+        assertEquals(List.of(id(1)), ids(data(third, 200)));
+        assertNull(next(third));
+        var original = new ArrayList<>(ids(rows));
+        original.addAll(ids(data(second, 200)));
+        original.addAll(ids(data(third, 200)));
+        assertEquals(7, original.stream().distinct().count());
+
+        var failed = alice.get(path(monitor) + "?limit=3&state=FAILED");
+        assertEquals(List.of(id(6), id(4), id(2)), ids(data(failed, 200)));
+        assertNull(next(failed));
+        data(failed, 200).forEach(row -> assertEquals("FAILED", row.get("state").textValue()));
+        for (String query : List.of("", "?limit=100")) {
+            var accepted = alice.get(path(monitor) + query);
+            assertEquals(original, ids(data(accepted, 200)));
+            assertNull(next(accepted));
+        }
+        String decoded = new String(Base64.getUrlDecoder().decode(cursor), StandardCharsets.UTF_8);
+        String badVersion = Base64.getUrlEncoder().withoutPadding()
+                .encodeToString(("2" + decoded.substring(1)).getBytes(StandardCharsets.UTF_8));
+        var invalid = List.of("limit=0", "limit=-1", "limit=1.5", "limit=101", "limit=invalid", "limit=",
+                "state=UNKNOWN", "state=", "limit=3&cursor=not-a-cursor", "limit=3&cursor=",
+                "limit=3&cursor=" + badVersion, "limit=3&state=FAILED&cursor=" + cursor, "limit=2&cursor=" + cursor);
+        for (String query : invalid) error(alice.get(path(monitor) + "?" + query), 400, "INVALID_INPUT");
+        String other = data(alice.mutate("/api/monitors", HttpMethod.POST, Map.of(
+                "name", "Cursor binding fixture", "url", "http://127.0.0.1:4321/ok", "interval", 60, "enabled", true)), 201)
+                .at("/monitor/id").textValue();
+        error(alice.get(path(other) + "?limit=3&cursor=" + cursor), 400, "INVALID_INPUT");
+        data(alice.mutate("/api/monitors/" + other, HttpMethod.DELETE, null), 200);
+        var bob = new SessionClient(api);
+        assertEquals(200, bob.login("bob-e04", BOB_PASSWORD).getStatusCode().value());
+        error(bob.get(path(monitor) + "?limit=3&cursor=" + cursor), 404, "NOT_FOUND");
+        error(alice.get(path("00000000-0000-0000-0000-000000000000") + "?limit=3"), 404, "NOT_FOUND");
+        error(new SessionClient(api).get(path(monitor) + "?limit=3"), 401, "UNAUTHENTICATED");
+
+        TestDatabase.execute("DELETE FROM e07_history.check_runs");
+        seed(monitor, 1, 7);
+        var beforeInsert = alice.get(path(monitor) + "?limit=3");
+        assertEquals(List.of(id(7), id(6), id(5)), ids(data(beforeInsert, 200)));
+        seed(monitor, 8, 8);
+        var afterInsert = alice.get(path(monitor) + "?limit=3&cursor=" + next(beforeInsert));
+        assertEquals(List.of(id(4), id(3), id(2)), ids(data(afterInsert, 200)));
+        var last = alice.get(path(monitor) + "?limit=3&cursor=" + next(afterInsert));
+        assertEquals(List.of(id(1)), ids(data(last, 200)));
+        assertNull(next(last));
+        var continued = new ArrayList<>(ids(data(beforeInsert, 200)));
+        continued.addAll(ids(data(afterInsert, 200)));
+        continued.addAll(ids(data(last, 200)));
+        assertEquals(original, continued);
+
+        assertTrue(AopUtils.isAopProxy(store));
+        var sql = SqlEvidence.statements.stream().filter(statement -> statement.contains("from e07_history.check_runs")).distinct().toList();
+        assertFalse(sql.isEmpty());
+        assertTrue(sql.stream().allMatch(statement -> statement.contains("owner_user_id=?")
+                && statement.contains("order by") && statement.contains("finished_at desc,")
+                && statement.contains(".id desc") && statement.contains("fetch first ? rows only")
+                && !statement.contains("offset")), "Every history SQL read must be owner-scoped, ordered and capped");
+        assertTrue(sql.stream().anyMatch(statement -> statement.contains(".state=?")));
+        assertTrue(sql.stream().anyMatch(statement -> statement.contains(".finished_at<?")
+                && statement.contains(".finished_at=?") && statement.contains(".id<?")));
+        var indexes = new ArrayList<String>();
+        try (var connection = TestDatabase.connect(); var statement = connection.createStatement();
+                var definitions = statement.executeQuery("SELECT indexdef FROM pg_indexes WHERE schemaname='e07_history' "
+                        + "AND indexname IN ('check_runs_history_order_idx','check_runs_history_state_order_idx') ORDER BY indexname")) {
+            while (definitions.next()) indexes.add(definitions.getString(1));
+        }
+        assertEquals(2, indexes.size());
+        assertTrue(indexes.stream().anyMatch(index -> index.contains("(monitor_id, finished_at DESC, id DESC)")));
+        assertTrue(indexes.stream().anyMatch(index -> index.contains("(monitor_id, state, finished_at DESC, id DESC)")));
+        Files.writeString(Path.of("target/e07-history.json"), json.writerWithDefaultPrettyPrinter().writeValueAsString(Map.of(
+                "result", "PASS", "firstPage", ids(rows), "originalContinuation", original,
+                "failedFilter", ids(data(failed, 200)), "afterNewerInsertContinuation", continued,
+                "invalidInputsRejected", invalid.size() + 1, "defaultAndMaxAccepted", true,
+                "ownerAndAnonymousBoundaries", true, "generatedSql", sql, "indexes", indexes)) + "\n");
     }
 
     private static void seed(String monitor, int from, int through) throws Exception {
@@ -86,6 +175,7 @@ class HistoryPaginationTest {
 
     private static String id(int number) { return "00000000-0000-4000-8000-%012d".formatted(number); }
     private static String path(String monitor) { return "/api/monitors/" + monitor + "/checks"; }
+    private static String next(ResponseEntity<JsonNode> response) { return response.getHeaders().getFirst("X-Next-Cursor"); }
     private static List<String> ids(JsonNode rows) {
         var ids = new java.util.ArrayList<String>();
         rows.forEach(row -> ids.add(row.get("id").textValue()));
@@ -96,4 +186,12 @@ class HistoryPaginationTest {
         assertTrue(response.getBody().has("data"));
         return response.getBody().get("data");
     }
+    private static void error(ResponseEntity<JsonNode> response, int status, String code) {
+        assertEquals(status, response.getStatusCode().value());
+        assertEquals(code, response.getBody().at("/error/code").textValue());
+    }
+    public static class SqlEvidence implements StatementInspector {
+        static final List<String> statements = new CopyOnWriteArrayList<>();
+        @Override public String inspect(String sql) { statements.add(sql); return sql; }
+    }
 }
diff --git a/backend/src/test/java/dev/evolution/monitor/OwnershipMigrationTest.java b/backend/src/test/java/dev/evolution/monitor/OwnershipMigrationTest.java
index 4a2cb66..efbea55 100644
--- a/backend/src/test/java/dev/evolution/monitor/OwnershipMigrationTest.java
+++ b/backend/src/test/java/dev/evolution/monitor/OwnershipMigrationTest.java
@@ -29,7 +29,7 @@ class OwnershipMigrationTest {
         String schema = "e05_fresh";
         TestDatabase.reset(schema);
         try {
-            var flyway = TestDatabase.migration(schema).load();
+            var flyway = TestDatabase.migration(schema).target("4").load();
             assertEquals(4, flyway.migrate().migrationsExecuted);
             assertEquals(List.of("1", "2", "3", "4"), java.util.Arrays.stream(flyway.info().applied())
                     .map(migration -> migration.getVersion().toString()).toList());
@@ -62,7 +62,7 @@ class OwnershipMigrationTest {
             // An explicitly staged but unknown UUID is refused too; nothing is auto-assigned.
             TestDatabase.execute("ALTER TABLE e05_upgrade.monitors ADD COLUMN owner_user_id uuid; "
                     + "UPDATE e05_upgrade.monitors SET owner_user_id = '00000000-0000-0000-0000-000000000000'");
-            var unknown = assertThrows(RuntimeException.class, () -> TestDatabase.migration(schema).load().migrate());
+            var unknown = assertThrows(RuntimeException.class, () -> TestDatabase.migration(schema).target("4").load().migrate());
             assertTrue(messages(unknown).contains("assignment must reference an existing verified user"));
             assertEquals(before, canonicalRows());
             assertEquals(3, TestDatabase.migration(schema).load().info().applied().length);
@@ -75,7 +75,7 @@ class OwnershipMigrationTest {
                 update.setString(2, "bob-e04");
                 assertEquals(1, update.executeUpdate());
             }
-            var upgrade = TestDatabase.migration(schema).load();
+            var upgrade = TestDatabase.migration(schema).target("4").load();
             assertEquals(1, upgrade.migrate().migrationsExecuted);
             upgrade.validate();
             assertEquals(4, upgrade.info().applied().length);
@@ -112,7 +112,7 @@ class OwnershipMigrationTest {
             String schema = fixture[0];
             TestDatabase.reset(schema);
             try {
-                assertEquals(4, TestDatabase.migration(schema).load().migrate().migrationsExecuted);
+                assertEquals(4, TestDatabase.migration(schema).target("4").load().migrate().migrationsExecuted);
                 TestDatabase.execute("ALTER TABLE " + schema + ".monitors " + fixture[1]);
                 assertStartupFailure(schema, fixture[2]);
             } finally { TestDatabase.drop(schema); }
@@ -189,6 +189,7 @@ class OwnershipMigrationTest {
 
     private static String[] arguments(String schema) {
         return new String[]{"--spring.flyway.schemas=" + schema, "--spring.flyway.default-schema=" + schema,
+                "--spring.flyway.target=4",
                 "--spring.jpa.properties.hibernate.default_schema=" + schema,
                 "--spring.main.banner-mode=off", "--logging.level.root=OFF", "--server.port=4322"};
     }
diff --git a/backend/src/test/java/dev/evolution/monitor/TestDatabase.java b/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
index 093f3e8..0d971b5 100644
--- a/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
+++ b/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
@@ -12,7 +12,7 @@ final class TestDatabase {
     private static final Set<String> SCHEMAS = Set.of("e04_functional", "e04_migrations", "e04_mapping",
             "e04_missing_column", "e04_extra_required", "e04_session", "e04_users",
             "e04_missing_user_column", "e04_extra_user_required", "e05_ownership", "e05_fresh",
-            "e05_upgrade", "e05_missing_owner", "e05_nullable_owner", "e05_missing_owner_fk", "e07_history");
+            "e05_upgrade", "e05_missing_owner", "e05_nullable_owner", "e05_missing_owner_fk", "e07_history", "e07_index_upgrade");
 
     static Connection connect() throws SQLException {
         return DriverManager.getConnection(System.getenv().getOrDefault("DB_URL", "jdbc:postgresql://127.0.0.1:15432/monitor"),
diff --git a/evidence/E07/backend.json b/evidence/E07/backend.json
new file mode 100644
index 0000000..45410d1
--- /dev/null
+++ b/evidence/E07/backend.json
@@ -0,0 +1,12 @@
+{
+  "originalContinuation" : [ "00000000-0000-4000-8000-000000000007", "00000000-0000-4000-8000-000000000006", "00000000-0000-4000-8000-000000000005", "00000000-0000-4000-8000-000000000004", "00000000-0000-4000-8000-000000000003", "00000000-0000-4000-8000-000000000002", "00000000-0000-4000-8000-000000000001" ],
+  "invalidInputsRejected" : 14,
+  "result" : "PASS",
+  "generatedSql" : [ "select cre1_0.id,cre1_0.failure_reason,cre1_0.finished_at,cre1_0.http_status,cre1_0.latency_ms,cre1_0.monitor_id,cre1_0.started_at,cre1_0.state,cre1_0.trigger_kind from e07_history.check_runs cre1_0,e07_history.monitors me1_0 where cre1_0.monitor_id=me1_0.id and me1_0.id=? and me1_0.owner_user_id=? order by cre1_0.finished_at desc,cre1_0.id desc fetch first ? rows only", "select cre1_0.id,cre1_0.failure_reason,cre1_0.finished_at,cre1_0.http_status,cre1_0.latency_ms,cre1_0.monitor_id,cre1_0.started_at,cre1_0.state,cre1_0.trigger_kind from e07_history.check_runs cre1_0,e07_history.monitors me1_0 where cre1_0.monitor_id=me1_0.id and me1_0.id=? and me1_0.owner_user_id=? and (cre1_0.finished_at<? or (cre1_0.finished_at=? and cre1_0.id<?)) order by cre1_0.finished_at desc,cre1_0.id desc fetch first ? rows only", "select cre1_0.id,cre1_0.failure_reason,cre1_0.finished_at,cre1_0.http_status,cre1_0.latency_ms,cre1_0.monitor_id,cre1_0.started_at,cre1_0.state,cre1_0.trigger_kind from e07_history.check_runs cre1_0,e07_history.monitors me1_0 where cre1_0.monitor_id=me1_0.id and me1_0.id=? and me1_0.owner_user_id=? and cre1_0.state=? order by cre1_0.finished_at desc,cre1_0.id desc fetch first ? rows only" ],
+  "ownerAndAnonymousBoundaries" : true,
+  "afterNewerInsertContinuation" : [ "00000000-0000-4000-8000-000000000007", "00000000-0000-4000-8000-000000000006", "00000000-0000-4000-8000-000000000005", "00000000-0000-4000-8000-000000000004", "00000000-0000-4000-8000-000000000003", "00000000-0000-4000-8000-000000000002", "00000000-0000-4000-8000-000000000001" ],
+  "failedFilter" : [ "00000000-0000-4000-8000-000000000006", "00000000-0000-4000-8000-000000000004", "00000000-0000-4000-8000-000000000002" ],
+  "indexes" : [ "CREATE INDEX check_runs_history_order_idx ON e07_history.check_runs USING btree (monitor_id, finished_at DESC, id DESC)", "CREATE INDEX check_runs_history_state_order_idx ON e07_history.check_runs USING btree (monitor_id, state, finished_at DESC, id DESC)" ],
+  "defaultAndMaxAccepted" : true,
+  "firstPage" : [ "00000000-0000-4000-8000-000000000007", "00000000-0000-4000-8000-000000000006", "00000000-0000-4000-8000-000000000005" ]
+}
diff --git a/evidence/E07/index-migration.json b/evidence/E07/index-migration.json
new file mode 100644
index 0000000..63bc07c
--- /dev/null
+++ b/evidence/E07/index-migration.json
@@ -0,0 +1,3 @@
+{"result":"PASS","upgradeFrom":4,"upgradeTo":5,"migrationsExecuted":1,
+ "repeatMigrations":0,"sevenHistoricalRowsUnchanged":true,
+ "monitorUnchanged":true,"priorMigrationChecksumsUnchanged":true,"newIndexes":2}


