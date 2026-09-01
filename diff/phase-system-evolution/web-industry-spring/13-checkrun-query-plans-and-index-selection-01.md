# E20 CheckRun Query Plan과 Index 선택

## `test(e20): verify the fixed skewed history plan`

diff --git a/backend/src/test/java/dev/evolution/monitor/HistoryQueryPlanTest.java b/backend/src/test/java/dev/evolution/monitor/HistoryQueryPlanTest.java
new file mode 100644
index 0000000..d81045e
--- /dev/null
+++ b/backend/src/test/java/dev/evolution/monitor/HistoryQueryPlanTest.java
@@ -0,0 +1,245 @@
+package dev.evolution.monitor;
+
+import static org.junit.jupiter.api.Assertions.*;
+
+import ch.qos.logback.classic.Level;
+import ch.qos.logback.classic.Logger;
+import ch.qos.logback.classic.spi.ILoggingEvent;
+import ch.qos.logback.core.read.ListAppender;
+import com.fasterxml.jackson.databind.JsonNode;
+import com.fasterxml.jackson.databind.ObjectMapper;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.security.MessageDigest;
+import java.time.Instant;
+import java.util.ArrayList;
+import java.util.HexFormat;
+import java.util.LinkedHashMap;
+import java.util.List;
+import java.util.Map;
+import java.util.UUID;
+import java.util.stream.IntStream;
+import org.junit.jupiter.api.AfterAll;
+import org.junit.jupiter.api.BeforeAll;
+import org.junit.jupiter.api.Test;
+import org.junit.jupiter.api.condition.EnabledIfSystemProperty;
+import org.slf4j.LoggerFactory;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
+import org.springframework.test.annotation.DirtiesContext;
+import org.springframework.test.context.DynamicPropertyRegistry;
+import org.springframework.test.context.DynamicPropertySource;
+import org.springframework.web.server.ResponseStatusException;
+
+// The fixed 99k dataset is opt-in; default CI must not repeat this capped plan proof.
+@EnabledIfSystemProperty(named = "e20.plan-proof", matches = "baseline|final")
+@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
+@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_CLASS)
+class HistoryQueryPlanTest {
+    private static final String SCHEMA = "e20_plan";
+    private static final Path FIXTURE = Path.of("../evidence/phase-1/E20");
+    private static final Path OUTPUT = Path.of("../output/phase-1/e20");
+    private static final UUID ALICE = UUID.fromString("00000000-0000-4000-a000-000000000001");
+    private static final UUID BOB = UUID.fromString("00000000-0000-4000-a000-000000000002");
+    private static final UUID A = UUID.fromString("00000000-0000-4000-9000-000000000001");
+    private static final Instant T0 = Instant.parse("2026-08-28T00:00:00Z");
+    private static final ObjectMapper JSON = new ObjectMapper().findAndRegisterModules();
+    private static final Map<String, Object> evidence = new LinkedHashMap<>();
+    @Autowired MonitorStore store;
+
+    @DynamicPropertySource
+    static void database(DynamicPropertyRegistry properties) {
+        TestDatabase.configure(properties, SCHEMA);
+        properties.add("spring.jpa.properties.hibernate.session_factory.statement_inspector",
+                () -> HistoryPaginationTest.SqlEvidence.class.getName());
+    }
+
+    @BeforeAll
+    static void seed(@Autowired MonitorStore initializedStore) throws Exception {
+        // Parameter injection initializes the real context/Flyway before this static seed.
+        Files.createDirectories(OUTPUT);
+        var fixture = JSON.readTree(Files.readString(FIXTURE.resolve("fixture.json")));
+        assertEquals(fixture.at("/query/sourceSha256").textValue(), hash(Path.of("src/main/java/dev/evolution/monitor/MonitorStore.java")));
+        assertEquals(fixture.at("/query/entitySha256").textValue(), hash(Path.of("src/main/java/dev/evolution/monitor/CheckRunEntity.java")));
+        assertEquals(fixture.at("/query/v5Sha256").textValue(), hash(Path.of("src/main/resources/db/migration/V5__index_check_history.sql")));
+        evidence.put("phase", System.getProperty("e20.plan-proof"));
+        evidence.put("fixtureSha256", hash(FIXTURE.resolve("fixture.json")));
+        evidence.put("seedSha256", hash(FIXTURE.resolve("seed.sql")));
+        evidence.put("result", "INCOMPLETE");
+        try (var connection = TestDatabase.connect()) {
+            connection.setAutoCommit(false);
+            try (var insert = connection.prepareStatement("INSERT INTO e20_plan.users (id,username,password_hash) VALUES (?,?,?)")) {
+                for (var user : Map.of(ALICE, "alice-e04", BOB, "bob-e04").entrySet()) {
+                    insert.setObject(1, user.getKey());
+                    insert.setString(2, user.getValue());
+                    insert.setString(3, new BCryptPasswordEncoder(10).encode(SessionClient.password()));
+                    assertEquals(1, insert.executeUpdate());
+                }
+            }
+            try (var statement = connection.createStatement()) {
+                // Two bulk INSERTs and one ANALYZE command; no per-row API calls or extra dataset.
+                statement.execute(Files.readString(FIXTURE.resolve("seed.sql")));
+            }
+            connection.commit();
+        }
+        evidence.put("datasetMaterializations", 1);
+        evidence.put("analyzeCommands", 1);
+    }
+
+    @AfterAll
+    static void cleanup() throws Exception {
+        try { TestDatabase.drop(SCHEMA); evidence.put("ownedSchemaDropped", true); }
+        finally {
+            Files.createDirectories(OUTPUT);
+            Files.writeString(OUTPUT.resolve(System.getProperty("e20.plan-proof") + "-result.json"),
+                    JSON.writerWithDefaultPrettyPrinter().writeValueAsString(evidence) + "\n");
+        }
+    }
+
+    @Test
+    void fixedSkewedFailedHistoryUsesTheActualOwnerScopedQuery() throws Exception {
+        var fixture = JSON.readTree(Files.readString(FIXTURE.resolve("fixture.json")));
+        String sql = Files.readString(FIXTURE.resolve("history.sql")).strip();
+        assertEquals(fixture.at("/query/sqlSha256").textValue(), hash(FIXTURE.resolve("history.sql")));
+        List<Map<String, Object>> distribution = distribution();
+        evidence.put("distribution", distribution);
+        assertEquals(10, distribution.size());
+        for (int i = 0; i < distribution.size(); i++) {
+            assertEquals(i == 0 ? 90000L : 1000L, distribution.get(i).get("rows"));
+            assertEquals(i == 0 ? 900L : 10L, distribution.get(i).get("failed"));
+        }
+        List<Map<String, Object>> indexesBefore = indexes();
+        evidence.put("indexes", indexesBefore);
+        assertEquals(2, indexesBefore.size());
+
+        // Reuse the existing SQL inspector; bind logging is enabled only after secret setup,
+        // for this single known owner/Monitor/state read, and never for account/session work.
+        HistoryPaginationTest.SqlEvidence.statements.clear();
+        Logger logger = (Logger) LoggerFactory.getLogger("org.hibernate.orm.jdbc.bind");
+        Level originalLevel = logger.getLevel();
+        boolean originalAdditive = logger.isAdditive();
+        var binds = new ListAppender<ILoggingEvent>();
+        binds.setContext(logger.getLoggerContext());
+        binds.start();
+        logger.addAppender(binds);
+        logger.setAdditive(false);
+        logger.setLevel(Level.TRACE);
+        MonitorStore.HistoryPage first;
+        try { first = store.historyPage(ALICE, A, "20", "FAILED", null); }
+        finally {
+            logger.setLevel(originalLevel);
+            logger.setAdditive(originalAdditive);
+            logger.detachAppender(binds);
+            binds.stop();
+        }
+        var actualSql = HistoryPaginationTest.SqlEvidence.statements.stream()
+                .filter(value -> value.contains("from e20_plan.check_runs ")).toList();
+        var actualBinds = binds.list.stream().map(ILoggingEvent::getFormattedMessage).toList();
+        evidence.put("actualHistorySql", actualSql);
+        evidence.put("scopedBindTrace", actualBinds);
+        evidence.put("explainBindings", fixture.at("/query/bindings"));
+        evidence.put("publicLimit", 20);
+        evidence.put("nativeLookaheadLimit", 21);
+        evidence.put("limitBindTraceObserved", actualBinds.stream().anyMatch(value -> value.endsWith("[21]")));
+        assertEquals(List.of(sql), actualSql);
+        for (String value : List.of(A.toString(), ALICE.toString(), "FAILED")) {
+            assertTrue(actualBinds.stream().anyMatch(line -> line.endsWith("[" + value + "]")));
+        }
+        assertEquals(expected(90000), first.items());
+        assertNotNull(first.nextCursor());
+        evidence.put("firstPage", first.items());
+        evidence.put("firstCursor", first.nextCursor());
+
+        // Exactly one plan for this invocation, using the SQL just produced by the application.
+        JsonNode plan;
+        try (var connection = TestDatabase.connect();
+                var query = connection.prepareStatement("EXPLAIN(ANALYZE,BUFFERS,FORMAT JSON) " + sql)) {
+            query.setObject(1, A);
+            query.setObject(2, ALICE);
+            query.setString(3, "FAILED");
+            query.setInt(4, 21);
+            try (var rows = query.executeQuery()) { assertTrue(rows.next()); plan = JSON.readTree(rows.getString(1)); }
+        }
+        Files.writeString(OUTPUT.resolve(System.getProperty("e20.plan-proof") + "-plan.json"),
+                JSON.writerWithDefaultPrettyPrinter().writeValueAsString(plan) + "\n");
+        evidence.put("explainInvocations", 1);
+        List<JsonNode> nodes = new ArrayList<>();
+        collect(plan.get(0).get("Plan"), nodes);
+        var checkNodes = nodes.stream().filter(node -> node.path("Relation Name").asText().equals("check_runs")).toList();
+        boolean fit = checkNodes.size() == 1 && checkNodes.getFirst().path("Index Name").asText().equals("check_runs_history_state_order_idx")
+                && checkNodes.getFirst().path("Actual Rows").asInt() == 21
+                && checkNodes.getFirst().path("Rows Removed by Filter").asInt() == 0
+                && checkNodes.getFirst().path("Index Cond").asText().contains("monitor_id")
+                && checkNodes.getFirst().path("Index Cond").asText().contains("state")
+                && nodes.stream().noneMatch(node -> node.path("Node Type").asText().contains("Sort"))
+                && plan.get(0).path("Plan").path("Actual Rows").asInt() == 21;
+        evidence.put("existingPlanMeetsFixedCriteria", fit);
+        evidence.put("checkRunPlanNodes", checkNodes);
+
+        HistoryPaginationTest.SqlEvidence.statements.clear();
+        var second = store.historyPage(ALICE, A, "20", "FAILED", first.nextCursor());
+        assertEquals(expected(88000), second.items());
+        assertEquals(List.of(fixture.at("/continuation/sql").textValue()), HistoryPaginationTest.SqlEvidence.statements.stream()
+                .filter(value -> value.contains("from e20_plan.check_runs ")).toList());
+        var first40 = new ArrayList<>(first.items());
+        first40.addAll(second.items());
+        assertEquals(40, first40.stream().map(CheckRunner.CheckRun::id).distinct().count());
+        assertEquals(404, assertThrows(ResponseStatusException.class,
+                () -> store.historyPage(BOB, A, "20", "FAILED", null)).getStatusCode().value());
+        evidence.put("secondPage", second.items());
+        evidence.put("fortyExactDistinctRows", true);
+        evidence.put("foreignOwnerStatus", 404);
+
+        var migration = TestDatabase.migration(SCHEMA).load();
+        migration.validate();
+        int repeated = migration.migrate().migrationsExecuted;
+        assertEquals(0, repeated);
+        evidence.put("repeatMigrations", repeated);
+        evidence.put("appliedMigrationVersions", java.util.Arrays.stream(migration.info().applied())
+                .map(info -> Map.of("version", info.getVersion().toString(), "checksum", info.getChecksum())).toList());
+        assertEquals(indexesBefore, indexes());
+        assertEquals(distribution, distribution());
+        evidence.put("datasetAndIndexesUnchanged", true);
+        evidence.put("result", fit ? "PASS: existing V5 index meets fixed plan criteria" : "BASELINE: index assessment needed");
+        if (System.getProperty("e20.plan-proof").equals("final")) assertTrue(fit, "Final plan must meet the frozen structural criteria");
+    }
+
+    private static List<CheckRunner.CheckRun> expected(int firstOrdinal) {
+        return IntStream.range(0, 20).mapToObj(offset -> {
+            int ordinal = firstOrdinal - offset * 100;
+            Instant time = T0.plusMillis(ordinal);
+            return new CheckRunner.CheckRun(UUID.fromString("00000000-0000-4000-8000-%012d".formatted(ordinal)),
+                    A, "MANUAL", "FAILED", 503, 0L, "HTTP_STATUS", time, time);
+        }).toList();
+    }
+
+    private static List<Map<String, Object>> distribution() throws Exception {
+        List<Map<String, Object>> rows = new ArrayList<>();
+        try (var connection = TestDatabase.connect(); var query = connection.createStatement(); var result = query.executeQuery(
+                "SELECT m.name,count(*) AS runs,count(*) FILTER(WHERE c.state='FAILED') AS failed "
+                + "FROM e20_plan.monitors m JOIN e20_plan.check_runs c ON c.monitor_id=m.id GROUP BY m.name ORDER BY m.name")) {
+            while (result.next()) rows.add(Map.of("monitor", result.getString(1), "rows", result.getLong(2), "failed", result.getLong(3)));
+        }
+        return rows;
+    }
+
+    private static List<Map<String, Object>> indexes() throws Exception {
+        List<Map<String, Object>> rows = new ArrayList<>();
+        try (var connection = TestDatabase.connect(); var query = connection.createStatement(); var result = query.executeQuery(
+                "SELECT indexname,indexdef,pg_relation_size((schemaname||'.'||indexname)::regclass) FROM pg_indexes "
+                + "WHERE schemaname='e20_plan' AND indexname IN ('check_runs_history_order_idx','check_runs_history_state_order_idx') ORDER BY indexname")) {
+            while (result.next()) rows.add(Map.of("name", result.getString(1), "definition", result.getString(2), "bytes", result.getLong(3)));
+        }
+        return rows;
+    }
+
+    private static void collect(JsonNode node, List<JsonNode> nodes) {
+        nodes.add(node);
+        node.path("Plans").forEach(child -> collect(child, nodes));
+    }
+
+    private static String hash(Path path) throws Exception {
+        return HexFormat.of().formatHex(MessageDigest.getInstance("SHA-256").digest(Files.readAllBytes(path)));
+    }
+}
diff --git a/backend/src/test/java/dev/evolution/monitor/TestDatabase.java b/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
index 94845a6..26b2afa 100644
--- a/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
+++ b/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
@@ -13,7 +13,7 @@ final class TestDatabase {
             "e04_missing_column", "e04_extra_required", "e04_session", "e04_users",
             "e04_missing_user_column", "e04_extra_user_required", "e05_ownership", "e05_fresh",
             "e05_upgrade", "e05_missing_owner", "e05_nullable_owner", "e05_missing_owner_fk", "e07_history", "e07_index_upgrade",
-            "e09_scheduler", "e10_ownership", "e11_recovery");
+            "e09_scheduler", "e10_ownership", "e11_recovery", "e20_plan");
 
     static Connection connect() throws SQLException {
         return DriverManager.getConnection(System.getenv().getOrDefault("DB_URL", "jdbc:postgresql://127.0.0.1:15432/monitor"),
diff --git a/evidence/phase-1/E20/author-output.json b/evidence/phase-1/E20/author-output.json
new file mode 100644
index 0000000..8198ad7
--- /dev/null
+++ b/evidence/phase-1/E20/author-output.json
@@ -0,0 +1,15 @@
+{
+  "format": "Lossless UTF-8 original console/JUnit text; no whitespace stripping",
+  "files": [
+    {
+      "path": "output/phase-1/e20/baseline-maven.log",
+      "sha256": "86efff626bcf5023013bd23ff27b529c959c4625a245e8905b3a1914eb67cd48",
+      "utf8": "[INFO] Scanning for projects...\n[INFO] \n[INFO] ---------------------< dev.evolution:monitor-api >----------------------\n[INFO] Building monitor-api 0.0.1\n[INFO]   from pom.xml\n[INFO] --------------------------------[ jar ]---------------------------------\n[INFO] \n[INFO] --- enforcer:3.6.2:enforce (pinned-runtimes) @ monitor-api ---\n[INFO] Rule 0: org.apache.maven.enforcer.rules.version.RequireJavaVersion passed\n[INFO] Rule 1: org.apache.maven.enforcer.rules.version.RequireMavenVersion passed\n[INFO] \n[INFO] --- resources:3.3.1:resources (default-resources) @ monitor-api ---\n[INFO] Copying 2 resources from src/main/resources to target/classes\n[INFO] Copying 8 resources from src/main/resources to target/classes\n[INFO] \n[INFO] --- compiler:3.14.1:compile (default-compile) @ monitor-api ---\n[INFO] Nothing to compile - all classes are up to date.\n[INFO] \n[INFO] --- resources:3.3.1:testResources (default-testResources) @ monitor-api ---\n[INFO] skip non existing resourceDirectory /private/tmp/web-systems-evolution-0a006589-industry-spring/backend/src/test/resources\n[INFO] \n[INFO] --- compiler:3.14.1:testCompile (default-testCompile) @ monitor-api ---\n[INFO] Recompiling the module because of changed source code.\n[INFO] Compiling 18 source files with javac [debug parameters release 21] to target/test-classes\n[INFO] \n[INFO] --- surefire:3.5.6:test (default-test) @ monitor-api ---\n[INFO] Using auto detected provider org.apache.maven.surefire.junitplatform.JUnitPlatformProvider\n[INFO] \n[INFO] -------------------------------------------------------\n[INFO]  T E S T S\n[INFO] -------------------------------------------------------\n[INFO] Running dev.evolution.monitor.HistoryQueryPlanTest\n17:13:22.002 [main] INFO org.springframework.test.context.support.AnnotationConfigContextLoaderUtils -- Could not detect default configuration classes for test class [dev.evolution.monitor.HistoryQueryPlanTest]: HistoryQueryPlanTest does not declare any static, non-private, non-final, nested classes annotated with @Configuration.\n17:13:22.089 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper -- Found @SpringBootConfiguration dev.evolution.monitor.MonitorApplication for test class dev.evolution.monitor.HistoryQueryPlanTest\n\n  .   ____          _            __ _ _\n /\\\\ / ___'_ __ _ _(_)_ __  __ _ \\ \\ \\ \\\n( ( )\\___ | '_ | '_| | '_ \\/ _` | \\ \\ \\ \\\n \\\\/  ___)| |_)| | | | | || (_| |  ) ) ) )\n  '  |____| .__|_| |_|_| |_\\__, | / / / /\n =========|_|==============|___/=/_/_/_/\n\n :: Spring Boot ::               (v3.5.16)\n\n2026-08-28T17:13:22.430+09:00  INFO 55145 --- [monitor-api] [           main] d.e.monitor.HistoryQueryPlanTest         : Starting HistoryQueryPlanTest using Java 21.0.7 with PID 55145 (started by woopinbell in /private/tmp/web-systems-evolution-0a006589-industry-spring/backend)\n2026-08-28T17:13:22.430+09:00  INFO 55145 --- [monitor-api] [           main] d.e.monitor.HistoryQueryPlanTest         : No active profile set, falling back to 1 default profile: \"default\"\n2026-08-28T17:13:22.659+09:00  INFO 55145 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT mode.\n2026-08-28T17:13:22.672+09:00  INFO 55145 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 9 ms. Found 0 JPA repository interfaces.\n2026-08-28T17:13:22.897+09:00  INFO 55145 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...\n2026-08-28T17:13:22.919+09:00  INFO 55145 --- [monitor-api] [           main] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Added connection org.postgresql.jdbc.PgConnection@415795f3\n2026-08-28T17:13:22.920+09:00  INFO 55145 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.\n2026-08-28T17:13:22.939+09:00  INFO 55145 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)\n2026-08-28T17:13:22.971+09:00  INFO 55145 --- [monitor-api] [           main] o.f.c.i.s.JdbcTableSchemaHistory         : Schema history table \"e20_plan\".\"flyway_schema_history\" does not exist yet\n2026-08-28T17:13:22.973+09:00  INFO 55145 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 8 migrations (execution time 00:00.016s)\n2026-08-28T17:13:22.987+09:00  INFO 55145 --- [monitor-api] [           main] o.f.c.i.s.JdbcTableSchemaHistory         : Creating Schema History table \"e20_plan\".\"flyway_schema_history\" ...\n2026-08-28T17:13:23.027+09:00  INFO 55145 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema \"e20_plan\": << Empty Schema >>\n2026-08-28T17:13:23.033+09:00  INFO 55145 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema \"e20_plan\" to version \"1 - create monitors\"\n2026-08-28T17:13:23.057+09:00  INFO 55145 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema \"e20_plan\" to version \"2 - create check runs\"\n2026-08-28T17:13:23.073+09:00  INFO 55145 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema \"e20_plan\" to version \"3 - create users\"\n2026-08-28T17:13:23.085+09:00  INFO 55145 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema \"e20_plan\" to version \"4 - require monitor ownership\"\n2026-08-28T17:13:23.103+09:00  INFO 55145 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema \"e20_plan\" to version \"5 - index check history\"\n2026-08-28T17:13:23.118+09:00  INFO 55145 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema \"e20_plan\" to version \"6 - queue check execution\"\n2026-08-28T17:13:23.133+09:00  INFO 55145 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema \"e20_plan\" to version \"7 - execution ownership and manual identity\"\n2026-08-28T17:13:23.142+09:00  INFO 55145 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema \"e20_plan\" to version \"8 - recover expired executions\"\n2026-08-28T17:13:23.153+09:00  INFO 55145 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Successfully applied 8 migrations to schema \"e20_plan\", now at version v8 (execution time 00:00.045s)\n2026-08-28T17:13:23.209+09:00  INFO 55145 --- [monitor-api] [           main] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]\n2026-08-28T17:13:23.241+09:00  INFO 55145 --- [monitor-api] [           main] org.hibernate.Version                    : HHH000412: Hibernate ORM core version 6.6.53.Final\n2026-08-28T17:13:23.257+09:00  INFO 55145 --- [monitor-api] [           main] o.h.c.internal.RegionFactoryInitiator    : HHH000026: Second-level cache disabled\n2026-08-28T17:13:23.382+09:00  INFO 55145 --- [monitor-api] [           main] o.s.o.j.p.SpringPersistenceUnitInfo      : No LoadTimeWeaver setup: ignoring JPA class transformer\n2026-08-28T17:13:23.431+09:00  INFO 55145 --- [monitor-api] [           main] org.hibernate.orm.connections.pooling    : HHH10001005: Database info:\n\tDatabase JDBC URL [Connecting through datasource 'HikariDataSource (HikariPool-1)']\n\tDatabase driver: undefined/unknown\n\tDatabase version: 17.11\n\tAutocommit mode: undefined/unknown\n\tIsolation level: undefined/unknown\n\tMinimum pool size: undefined/unknown\n\tMaximum pool size: undefined/unknown\n2026-08-28T17:13:23.807+09:00  INFO 55145 --- [monitor-api] [           main] o.h.e.t.j.p.i.JtaPlatformInitiator       : HHH000489: No JTA platform available (set 'hibernate.transaction.jta.platform' to enable JTA platform integration)\n2026-08-28T17:13:23.823+09:00  INFO 55145 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'\n2026-08-28T17:13:23.975+09:00  INFO 55145 --- [monitor-api] [           main] d.e.monitor.HistoryQueryPlanTest         : Started HistoryQueryPlanTest in 1.819 seconds (process running for 2.278)\nMockito is currently self-attaching to enable the inline-mock-maker. This will no longer work in future releases of the JDK. Please add Mockito as an agent to your build as described in Mockito's documentation: https://javadoc.io/doc/org.mockito/mockito-core/latest/org.mockito/org/mockito/Mockito.html#0.3\nOpenJDK 64-Bit Server VM warning: Sharing is only supported for boot loader classes because bootstrap classpath has been appended\nWARNING: A Java agent has been loaded dynamically (/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy-agent/1.17.8/byte-buddy-agent-1.17.8.jar)\nWARNING: If a serviceability tool is in use, please run with -XX:+EnableDynamicAgentLoading to hide this warning\nWARNING: If a serviceability tool is not in use, please run with -Djdk.instrument.traceUsage for more information\nWARNING: Dynamic loading of agents will be disallowed by default in a future release\n2026-08-28T17:13:26.031+09:00  INFO 55145 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)\n2026-08-28T17:13:26.042+09:00  INFO 55145 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 8 migrations (execution time 00:00.005s)\n2026-08-28T17:13:26.067+09:00  INFO 55145 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 8 migrations (execution time 00:00.004s)\n2026-08-28T17:13:26.077+09:00  INFO 55145 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema \"e20_plan\": 8\n2026-08-28T17:13:26.078+09:00  INFO 55145 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Schema \"e20_plan\" is up to date. No migration necessary.\n2026-08-28T17:13:26.174+09:00  INFO 55145 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Closing JPA EntityManagerFactory for persistence unit 'default'\n2026-08-28T17:13:26.175+09:00  INFO 55145 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown initiated...\n2026-08-28T17:13:26.176+09:00  INFO 55145 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown completed.\n[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 4.245 s -- in dev.evolution.monitor.HistoryQueryPlanTest\n[INFO] \n[INFO] Results:\n[INFO] \n[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0\n[INFO] \n[INFO] ------------------------------------------------------------------------\n[INFO] BUILD SUCCESS\n[INFO] ------------------------------------------------------------------------\n[INFO] Total time:  6.564 s\n[INFO] Finished at: 2026-08-28T17:13:26+09:00\n[INFO] ------------------------------------------------------------------------\n"
+    },
+    {
+      "path": "backend/target/surefire-reports/dev.evolution.monitor.HistoryQueryPlanTest.txt",
+      "sha256": "a0757cd5a0875749505d4ad4d6e7a41b421d4da44a67cac47756d8a34914e562",
+      "utf8": "-------------------------------------------------------------------------------\nTest set: dev.evolution.monitor.HistoryQueryPlanTest\n-------------------------------------------------------------------------------\nTests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 4.245 s -- in dev.evolution.monitor.HistoryQueryPlanTest\n"
+    }
+  ]
+}
diff --git a/evidence/phase-1/E20/baseline-plan.json b/evidence/phase-1/E20/baseline-plan.json
new file mode 100644
index 0000000..734fb00
--- /dev/null
+++ b/evidence/phase-1/E20/baseline-plan.json
@@ -0,0 +1,147 @@
+[ {
+  "Plan" : {
+    "Node Type" : "Limit",
+    "Parallel Aware" : false,
+    "Async Capable" : false,
+    "Startup Cost" : 0.42,
+    "Total Cost" : 30.96,
+    "Plan Rows" : 21,
+    "Plan Width" : 418,
+    "Actual Startup Time" : 0.032,
+    "Actual Total Time" : 0.068,
+    "Actual Rows" : 21,
+    "Actual Loops" : 1,
+    "Shared Hit Blocks" : 25,
+    "Shared Read Blocks" : 0,
+    "Shared Dirtied Blocks" : 0,
+    "Shared Written Blocks" : 0,
+    "Local Hit Blocks" : 0,
+    "Local Read Blocks" : 0,
+    "Local Dirtied Blocks" : 0,
+    "Local Written Blocks" : 0,
+    "Temp Read Blocks" : 0,
+    "Temp Written Blocks" : 0,
+    "Plans" : [ {
+      "Node Type" : "Nested Loop",
+      "Parent Relationship" : "Outer",
+      "Parallel Aware" : false,
+      "Async Capable" : false,
+      "Join Type" : "Inner",
+      "Startup Cost" : 0.42,
+      "Total Cost" : 1363.24,
+      "Plan Rows" : 937,
+      "Plan Width" : 418,
+      "Actual Startup Time" : 0.031,
+      "Actual Total Time" : 0.066,
+      "Actual Rows" : 21,
+      "Actual Loops" : 1,
+      "Inner Unique" : false,
+      "Shared Hit Blocks" : 25,
+      "Shared Read Blocks" : 0,
+      "Shared Dirtied Blocks" : 0,
+      "Shared Written Blocks" : 0,
+      "Local Hit Blocks" : 0,
+      "Local Read Blocks" : 0,
+      "Local Dirtied Blocks" : 0,
+      "Local Written Blocks" : 0,
+      "Temp Read Blocks" : 0,
+      "Temp Written Blocks" : 0,
+      "Plans" : [ {
+        "Node Type" : "Index Scan",
+        "Parent Relationship" : "Outer",
+        "Parallel Aware" : false,
+        "Async Capable" : false,
+        "Scan Direction" : "Forward",
+        "Index Name" : "check_runs_history_state_order_idx",
+        "Relation Name" : "check_runs",
+        "Alias" : "cre1_0",
+        "Startup Cost" : 0.42,
+        "Total Cost" : 1350.38,
+        "Plan Rows" : 937,
+        "Plan Width" : 418,
+        "Actual Startup Time" : 0.023,
+        "Actual Total Time" : 0.051,
+        "Actual Rows" : 21,
+        "Actual Loops" : 1,
+        "Index Cond" : "((monitor_id = '00000000-0000-4000-9000-000000000001'::uuid) AND ((state)::text = 'FAILED'::text) AND (finished_at IS NOT NULL))",
+        "Rows Removed by Index Recheck" : 0,
+        "Shared Hit Blocks" : 24,
+        "Shared Read Blocks" : 0,
+        "Shared Dirtied Blocks" : 0,
+        "Shared Written Blocks" : 0,
+        "Local Hit Blocks" : 0,
+        "Local Read Blocks" : 0,
+        "Local Dirtied Blocks" : 0,
+        "Local Written Blocks" : 0,
+        "Temp Read Blocks" : 0,
+        "Temp Written Blocks" : 0
+      }, {
+        "Node Type" : "Materialize",
+        "Parent Relationship" : "Inner",
+        "Parallel Aware" : false,
+        "Async Capable" : false,
+        "Startup Cost" : 0.0,
+        "Total Cost" : 1.15,
+        "Plan Rows" : 1,
+        "Plan Width" : 16,
+        "Actual Startup Time" : 0.0,
+        "Actual Total Time" : 0.0,
+        "Actual Rows" : 1,
+        "Actual Loops" : 21,
+        "Shared Hit Blocks" : 1,
+        "Shared Read Blocks" : 0,
+        "Shared Dirtied Blocks" : 0,
+        "Shared Written Blocks" : 0,
+        "Local Hit Blocks" : 0,
+        "Local Read Blocks" : 0,
+        "Local Dirtied Blocks" : 0,
+        "Local Written Blocks" : 0,
+        "Temp Read Blocks" : 0,
+        "Temp Written Blocks" : 0,
+        "Plans" : [ {
+          "Node Type" : "Seq Scan",
+          "Parent Relationship" : "Outer",
+          "Parallel Aware" : false,
+          "Async Capable" : false,
+          "Relation Name" : "monitors",
+          "Alias" : "me1_0",
+          "Startup Cost" : 0.0,
+          "Total Cost" : 1.15,
+          "Plan Rows" : 1,
+          "Plan Width" : 16,
+          "Actual Startup Time" : 0.005,
+          "Actual Total Time" : 0.006,
+          "Actual Rows" : 1,
+          "Actual Loops" : 1,
+          "Filter" : "((id = '00000000-0000-4000-9000-000000000001'::uuid) AND (owner_user_id = '00000000-0000-4000-a000-000000000001'::uuid))",
+          "Rows Removed by Filter" : 9,
+          "Shared Hit Blocks" : 1,
+          "Shared Read Blocks" : 0,
+          "Shared Dirtied Blocks" : 0,
+          "Shared Written Blocks" : 0,
+          "Local Hit Blocks" : 0,
+          "Local Read Blocks" : 0,
+          "Local Dirtied Blocks" : 0,
+          "Local Written Blocks" : 0,
+          "Temp Read Blocks" : 0,
+          "Temp Written Blocks" : 0
+        } ]
+      } ]
+    } ]
+  },
+  "Planning" : {
+    "Shared Hit Blocks" : 267,
+    "Shared Read Blocks" : 0,
+    "Shared Dirtied Blocks" : 0,
+    "Shared Written Blocks" : 0,
+    "Local Hit Blocks" : 0,
+    "Local Read Blocks" : 0,
+    "Local Dirtied Blocks" : 0,
+    "Local Written Blocks" : 0,
+    "Temp Read Blocks" : 0,
+    "Temp Written Blocks" : 0
+  },
+  "Planning Time" : 0.348,
+  "Triggers" : [ ],
+  "Execution Time" : 0.087
+} ]
diff --git a/evidence/phase-1/E20/baseline-result.json b/evidence/phase-1/E20/baseline-result.json
new file mode 100644
index 0000000..68106de
--- /dev/null
+++ b/evidence/phase-1/E20/baseline-result.json
@@ -0,0 +1,545 @@
+{
+  "phase" : "baseline",
+  "fixtureSha256" : "7ec00fd2734f5a21c739d10cc93634d6f9d621a4c225328bd5d73be3ab61ee4b",
+  "seedSha256" : "76e01303ea456e8db0b6a241bd429f21dceac52ccca06d78853fd5689977ccf4",
+  "result" : "PASS: existing V5 index meets fixed plan criteria",
+  "datasetMaterializations" : 1,
+  "analyzeCommands" : 1,
+  "distribution" : [ {
+    "rows" : 90000,
+    "failed" : 900,
+    "monitor" : "A"
+  }, {
+    "rows" : 1000,
+    "failed" : 10,
+    "monitor" : "B"
+  }, {
+    "rows" : 1000,
+    "failed" : 10,
+    "monitor" : "C"
+  }, {
+    "rows" : 1000,
+    "failed" : 10,
+    "monitor" : "D"
+  }, {
+    "rows" : 1000,
+    "failed" : 10,
+    "monitor" : "E"
+  }, {
+    "rows" : 1000,
+    "failed" : 10,
+    "monitor" : "F"
+  }, {
+    "rows" : 1000,
+    "failed" : 10,
+    "monitor" : "G"
+  }, {
+    "rows" : 1000,
+    "failed" : 10,
+    "monitor" : "H"
+  }, {
+    "rows" : 1000,
+    "failed" : 10,
+    "monitor" : "I"
+  }, {
+    "rows" : 1000,
+    "failed" : 10,
+    "monitor" : "J"
+  } ],
+  "indexes" : [ {
+    "name" : "check_runs_history_order_idx",
+    "definition" : "CREATE INDEX check_runs_history_order_idx ON e20_plan.check_runs USING btree (monitor_id, finished_at DESC, id DESC)",
+    "bytes" : 10321920
+  }, {
+    "name" : "check_runs_history_state_order_idx",
+    "definition" : "CREATE INDEX check_runs_history_state_order_idx ON e20_plan.check_runs USING btree (monitor_id, state, finished_at DESC, id DESC)",
+    "bytes" : 13631488
+  } ],
+  "actualHistorySql" : [ "select cre1_0.id,cre1_0.claim_owner,cre1_0.failure_reason,cre1_0.finished_at,cre1_0.http_status,cre1_0.idempotency_key,cre1_0.latency_ms,cre1_0.lease_expires_at,cre1_0.manual_owner_user_id,cre1_0.monitor_id,cre1_0.queued_at,cre1_0.scheduled_for,cre1_0.started_at,cre1_0.state,cre1_0.trigger_kind from e20_plan.check_runs cre1_0,e20_plan.monitors me1_0 where cre1_0.monitor_id=me1_0.id and me1_0.id=? and me1_0.owner_user_id=? and cre1_0.finished_at is not null and cre1_0.state=? order by cre1_0.finished_at desc,cre1_0.id desc fetch first ? rows only" ],
+  "scopedBindTrace" : [ "binding parameter (1:UUID) <- [00000000-0000-4000-9000-000000000001]", "binding parameter (2:UUID) <- [00000000-0000-4000-a000-000000000001]", "binding parameter (1:UUID) <- [00000000-0000-4000-9000-000000000001]", "binding parameter (2:UUID) <- [00000000-0000-4000-a000-000000000001]", "binding parameter (3:VARCHAR) <- [FAILED]", "binding parameter (4:INTEGER) <- [21]" ],
+  "explainBindings" : [ {
+    "position" : 1,
+    "type" : "uuid",
+    "value" : "00000000-0000-4000-9000-000000000001"
+  }, {
+    "position" : 2,
+    "type" : "uuid",
+    "value" : "00000000-0000-4000-a000-000000000001"
+  }, {
+    "position" : 3,
+    "type" : "varchar",
+    "value" : "FAILED"
+  }, {
+    "position" : 4,
+    "type" : "integer",
+    "value" : 21
+  } ],
+  "publicLimit" : 20,
+  "nativeLookaheadLimit" : 21,
+  "limitBindTraceObserved" : true,
+  "firstPage" : [ {
+    "id" : "00000000-0000-4000-8000-000000090000",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875290.000000000,
+    "finishedAt" : 1787875290.000000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000089900",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875289.900000000,
+    "finishedAt" : 1787875289.900000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000089800",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875289.800000000,
+    "finishedAt" : 1787875289.800000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000089700",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875289.700000000,
+    "finishedAt" : 1787875289.700000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000089600",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875289.600000000,
+    "finishedAt" : 1787875289.600000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000089500",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875289.500000000,
+    "finishedAt" : 1787875289.500000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000089400",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875289.400000000,
+    "finishedAt" : 1787875289.400000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000089300",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875289.300000000,
+    "finishedAt" : 1787875289.300000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000089200",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875289.200000000,
+    "finishedAt" : 1787875289.200000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000089100",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875289.100000000,
+    "finishedAt" : 1787875289.100000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000089000",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875289.000000000,
+    "finishedAt" : 1787875289.000000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000088900",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875288.900000000,
+    "finishedAt" : 1787875288.900000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000088800",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875288.800000000,
+    "finishedAt" : 1787875288.800000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000088700",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875288.700000000,
+    "finishedAt" : 1787875288.700000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000088600",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875288.600000000,
+    "finishedAt" : 1787875288.600000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000088500",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875288.500000000,
+    "finishedAt" : 1787875288.500000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000088400",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875288.400000000,
+    "finishedAt" : 1787875288.400000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000088300",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875288.300000000,
+    "finishedAt" : 1787875288.300000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000088200",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875288.200000000,
+    "finishedAt" : 1787875288.200000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000088100",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875288.100000000,
+    "finishedAt" : 1787875288.100000000
+  } ],
+  "firstCursor" : "MXwwMDAwMDAwMC0wMDAwLTQwMDAtOTAwMC0wMDAwMDAwMDAwMDF8RkFJTEVEfDIwfDIwMjYtMDgtMjhUMDA6MDE6MjguMTAwWnwwMDAwMDAwMC0wMDAwLTQwMDAtODAwMC0wMDAwMDAwODgxMDA",
+  "explainInvocations" : 1,
+  "existingPlanMeetsFixedCriteria" : true,
+  "checkRunPlanNodes" : [ {
+    "Node Type" : "Index Scan",
+    "Parent Relationship" : "Outer",
+    "Parallel Aware" : false,
+    "Async Capable" : false,
+    "Scan Direction" : "Forward",
+    "Index Name" : "check_runs_history_state_order_idx",
+    "Relation Name" : "check_runs",
+    "Alias" : "cre1_0",
+    "Startup Cost" : 0.42,
+    "Total Cost" : 1350.38,
+    "Plan Rows" : 937,
+    "Plan Width" : 418,
+    "Actual Startup Time" : 0.023,
+    "Actual Total Time" : 0.051,
+    "Actual Rows" : 21,
+    "Actual Loops" : 1,
+    "Index Cond" : "((monitor_id = '00000000-0000-4000-9000-000000000001'::uuid) AND ((state)::text = 'FAILED'::text) AND (finished_at IS NOT NULL))",
+    "Rows Removed by Index Recheck" : 0,
+    "Shared Hit Blocks" : 24,
+    "Shared Read Blocks" : 0,
+    "Shared Dirtied Blocks" : 0,
+    "Shared Written Blocks" : 0,
+    "Local Hit Blocks" : 0,
+    "Local Read Blocks" : 0,
+    "Local Dirtied Blocks" : 0,
+    "Local Written Blocks" : 0,
+    "Temp Read Blocks" : 0,
+    "Temp Written Blocks" : 0
+  } ],
+  "secondPage" : [ {
+    "id" : "00000000-0000-4000-8000-000000088000",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875288.000000000,
+    "finishedAt" : 1787875288.000000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000087900",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875287.900000000,
+    "finishedAt" : 1787875287.900000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000087800",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875287.800000000,
+    "finishedAt" : 1787875287.800000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000087700",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875287.700000000,
+    "finishedAt" : 1787875287.700000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000087600",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875287.600000000,
+    "finishedAt" : 1787875287.600000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000087500",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875287.500000000,
+    "finishedAt" : 1787875287.500000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000087400",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875287.400000000,
+    "finishedAt" : 1787875287.400000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000087300",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875287.300000000,
+    "finishedAt" : 1787875287.300000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000087200",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875287.200000000,
+    "finishedAt" : 1787875287.200000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000087100",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875287.100000000,
+    "finishedAt" : 1787875287.100000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000087000",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875287.000000000,
+    "finishedAt" : 1787875287.000000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000086900",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875286.900000000,
+    "finishedAt" : 1787875286.900000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000086800",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875286.800000000,
+    "finishedAt" : 1787875286.800000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000086700",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875286.700000000,
+    "finishedAt" : 1787875286.700000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000086600",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875286.600000000,
+    "finishedAt" : 1787875286.600000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000086500",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875286.500000000,
+    "finishedAt" : 1787875286.500000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000086400",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875286.400000000,
+    "finishedAt" : 1787875286.400000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000086300",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875286.300000000,
+    "finishedAt" : 1787875286.300000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000086200",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875286.200000000,
+    "finishedAt" : 1787875286.200000000
+  }, {
+    "id" : "00000000-0000-4000-8000-000000086100",
+    "monitorId" : "00000000-0000-4000-9000-000000000001",
+    "trigger" : "MANUAL",
+    "state" : "FAILED",
+    "httpStatus" : 503,
+    "latencyMs" : 0,
+    "failureReason" : "HTTP_STATUS",
+    "startedAt" : 1787875286.100000000,
+    "finishedAt" : 1787875286.100000000
+  } ],
+  "fortyExactDistinctRows" : true,
+  "foreignOwnerStatus" : 404,
+  "repeatMigrations" : 0,
+  "appliedMigrationVersions" : [ {
+    "checksum" : 626767143,
+    "version" : "1"
+  }, {
+    "checksum" : 1432752866,
+    "version" : "2"
+  }, {
+    "checksum" : -1657218365,
+    "version" : "3"
+  }, {
+    "checksum" : -1004961846,
+    "version" : "4"
+  }, {
+    "checksum" : -527015721,
+    "version" : "5"
+  }, {
+    "checksum" : -1822634019,
+    "version" : "6"
+  }, {
+    "checksum" : 173943298,
+    "version" : "7"
+  }, {
+    "checksum" : 749586815,
+    "version" : "8"
+  } ],
+  "datasetAndIndexesUnchanged" : true,
+  "ownedSchemaDropped" : true
+}
diff --git a/evidence/phase-1/E20/fixture.json b/evidence/phase-1/E20/fixture.json
new file mode 100644
index 0000000..8fa72e7
--- /dev/null
+++ b/evidence/phase-1/E20/fixture.json
@@ -0,0 +1,100 @@
+{
+  "thread": "E20",
+  "profile": "phase-1",
+  "start": "b309d2f8b6de8b81c5936906e296f314441646bc",
+  "specRevision": "2ada57a71cd34fa2fae9809415c362a8bbfcdf02",
+  "schema": "e20_plan",
+  "dataset": {
+    "monitors": 10,
+    "checkRuns": 99000,
+    "uniqueDatasetVariants": 1,
+    "globalOrdinals": [
+      1,
+      99000
+    ],
+    "distribution": "A:90000; B through J:1000 each",
+    "failed": "globalOrdinal % 100 == 0; 990 total; A900",
+    "timestamp": "2026-08-28T00:00:00.000Z + globalOrdinal milliseconds; startedAt=finishedAt=queuedAt; latency0",
+    "monitorId": "00000000-0000-4000-9000-12-digit decimal ordinal1..10",
+    "checkId": "00000000-0000-4000-8000-12-digit decimal globalOrdinal",
+    "aliceId": "00000000-0000-4000-a000-000000000001",
+    "bobId": "00000000-0000-4000-a000-000000000002",
+    "ownership": "Alice:A-I; Bob:J",
+    "analyzeCommandsPerSeed": 1
+  },
+  "query": {
+    "method": "MonitorStore.historyPage",
+    "owner": "00000000-0000-4000-a000-000000000001",
+    "monitor": "00000000-0000-4000-9000-000000000001",
+    "state": "FAILED",
+    "publicLimit": 20,
+    "sqlLookaheadLimit": 21,
+    "cursor": null,
+    "sqlFile": "history.sql",
+    "sqlSha256": "28a76a06b4c6725dd412a9762423edfcebb635c48ababf7a981ba2ad4e1020d2",
+    "bindings": [
+      {
+        "position": 1,
+        "type": "uuid",
+        "value": "00000000-0000-4000-9000-000000000001"
+      },
+      {
+        "position": 2,
+        "type": "uuid",
+        "value": "00000000-0000-4000-a000-000000000001"
+      },
+      {
+        "position": 3,
+        "type": "varchar",
+        "value": "FAILED"
+      },
+      {
+        "position": 4,
+        "type": "integer",
+        "value": 21
+      }
+    ],
+    "source": "own current generated E07 history SQL, schema-only substitution; exact runtime SQL assertion before EXPLAIN",
+    "sourceSha256": "5dc1845f37fad4fdafe21b0aebb465c3bd51c19640f9dc4440e4d3fade135122",
+    "entitySha256": "d960343ecbb9d95a0e45c5427792f6d596d3f709e49a7dea4b4a557ad7ac0fae",
+    "v5Sha256": "1a2115ff8ddae50bc1f75b5ef13e1e06c9ebfd2f0cb988cc577e7c4de9c14dcc"
+  },
+  "continuation": {
+    "sql": "select cre1_0.id,cre1_0.claim_owner,cre1_0.failure_reason,cre1_0.finished_at,cre1_0.http_status,cre1_0.idempotency_key,cre1_0.latency_ms,cre1_0.lease_expires_at,cre1_0.manual_owner_user_id,cre1_0.monitor_id,cre1_0.queued_at,cre1_0.scheduled_for,cre1_0.started_at,cre1_0.state,cre1_0.trigger_kind from e20_plan.check_runs cre1_0,e20_plan.monitors me1_0 where cre1_0.monitor_id=me1_0.id and me1_0.id=? and me1_0.owner_user_id=? and cre1_0.finished_at is not null and cre1_0.state=? and (cre1_0.finished_at<? or (cre1_0.finished_at=? and cre1_0.id<?)) order by cre1_0.finished_at desc,cre1_0.id desc fetch first ? rows only",
+    "afterOrdinal": 88100,
+    "beforeTime": "2026-08-28T00:01:28.100Z",
+    "beforeId": "00000000-0000-4000-8000-000000088100",
+    "expectedOrdinals": [
+      88000,
+      86100
+    ],
+    "count": 20
+  },
+  "sequence": [
+    "migrate current V1-V8 into one owned schema; seed once; ANALYZE both tables in one command",
+    "actual first FAILED history page20 with existing read-only transaction proxy; scope SQL/bind observation to this call",
+    "one EXPLAIN(ANALYZE,BUFFERS,FORMAT JSON) of identical SQL and frozen native bindings",
+    "actual next page from returned cursor: exact20 rows, no gap/duplicate in first40; Bob access to A rejected",
+    "validate/repeat current migrations: zero new migrations; preserve counts/index definitions; drop owned schema"
+  ],
+  "structuralCriteria": [
+    "21 native rows ->20 public rows plus continuation",
+    "ordered state composite index scan",
+    "no check_runs sequential scan or explicit Sort",
+    "check_runs scan produces21 rows and removes0 by filter",
+    "record buffer hits/reads and complete raw plan; execution time is observational, not a target"
+  ],
+  "observer": "reuse HistoryPaginationTest.SqlEvidence; bind TRACE only around nonsecret history read; restore logger in finally",
+  "budget": {
+    "authorBaselinePlans": 1,
+    "authorDuplicateFinalPlans": 0,
+    "rootIndependentFinalPlans": 1,
+    "maxCandidateIndexes": 3,
+    "preferredNewIndexes": 0,
+    "loadRuns": 0,
+    "parameterSweeps": 0,
+    "automaticRetries": 0,
+    "rootMayRematerializeIdenticalDataset": true,
+    "defaultCiPlanProof": false
+  }
+}
diff --git a/evidence/phase-1/E20/fixtures.md b/evidence/phase-1/E20/fixtures.md
new file mode 100644
index 0000000..5a237b3
--- /dev/null
+++ b/evidence/phase-1/E20/fixtures.md
@@ -0,0 +1,54 @@
+# E20 frozen query-plan contract
+
+Profile phase-1; START b309d2f8b6de8b81c5936906e296f314441646bc;
+Spec-Revision 2ada57a71cd34fa2fae9809415c362a8bbfcdf02.
+Frozen before the one unchanged-product baseline. `fixture.json`, `history.sql`
+and `seed.sql` contain the exact identifiers, timestamps, data, query and bindings.
+
+The isolated schema is e20_plan. The fixed dataset has10 Monitors and99,000
+terminal CheckRuns: A90,000, B–J1,000 each. Every100th global ordinal is
+FAILED/503/HTTP_STATUS; the rest is SUCCEEDED/200/null. Global ordinal1..99,000
+determines the UUID suffix and started_at=finished_at=queued_at at
+2026-08-28T00:00:00.000Z plus that many milliseconds; latency0. Alice owns A–I;
+Bob owns J. User UUIDs are fixed, while password hashes are generated only at
+runtime and excluded from logs and evidence. No worker, web server or outbound
+connection is started. The seed runs once and has one ANALYZE command for the
+two query tables. Root may materialize this same dataset again in isolation;
+that is another materialization, not another dataset variant.
+
+The primary query is the real MonitorStore.historyPage owner-scoped FAILED
+history of A. Public limit20 is preserved; the native SQL fetches21 to decide
+whether to emit the existing cursor. Exact SQL is taken from this track's
+current E07 generated-query evidence with only the schema name substituted,
+and must equal the SQL emitted by the actual production call before EXPLAIN.
+Bindings are MonitorA UUID, Alice UUID, FAILED, and21. This retains the owner
+join, finished_at-not-null guard and finished_at/id descending ordering.
+
+Reuse the existing StatementInspector. Temporarily capture Hibernate bind TRACE
+only for the nonsecret first history call, after user/seed setup. Restore the
+logger in finally. The source-pinned limit+1 and native EXPLAIN bindings are
+recorded explicitly; do not invent a bind message if a Hibernate limit adapter
+does not emit one. No generic JDBC proxy or additional observer framework.
+
+Execute one EXPLAIN(ANALYZE,BUFFERS,FORMAT JSON) with the identical SQL/bindings.
+Check structure, actual rows/filters and buffers, not an absolute latency target.
+The existing state composite index meets the fixed structural criterion if it
+provides the ordered21 rows with0 CheckRun rows removed by filter, no CheckRun
+sequential scan and no explicit Sort. A small owner-table scan is allowed.
+If this already holds, choose0 new indexes and make no product/schema change.
+An inefficient baseline is evidence to assess, not a fabricated assertion failure.
+
+The same test reads the actual next page using the returned cursor. First-page
+ordinals are90000,89900,...,88100; second-page ordinals88000,...,86100. Compare
+every returned domain field with the fixed seed and require40 distinct records
+in their exact order. Bob's attempt to read A remains404. Reuse unchanged E07
+tie/newer-insertion evidence; root may run its existing history regression once.
+Validate/repeat current migrations with0 changes and record the existing index
+definitions/sizes. The owned schema is dropped in finally/AfterAll.
+
+Author budget: one baseline plan, no duplicate author final EXPLAIN. Root owns
+one independent final EXPLAIN as same-condition confirmation. At author
+submission that final confirmation is explicitly pending. Candidate indexes
+at most3, preferably0; no alternate data, parameter sweep, load run, planner
+settings, E11 kills, E12 outbound suite or full browser gate. The new test is
+opt-in and does not materialize the dataset during default CI/regression runs.
diff --git a/evidence/phase-1/E20/history.sql b/evidence/phase-1/E20/history.sql
new file mode 100644
index 0000000..c2b3117
--- /dev/null
+++ b/evidence/phase-1/E20/history.sql
@@ -0,0 +1 @@
+select cre1_0.id,cre1_0.claim_owner,cre1_0.failure_reason,cre1_0.finished_at,cre1_0.http_status,cre1_0.idempotency_key,cre1_0.latency_ms,cre1_0.lease_expires_at,cre1_0.manual_owner_user_id,cre1_0.monitor_id,cre1_0.queued_at,cre1_0.scheduled_for,cre1_0.started_at,cre1_0.state,cre1_0.trigger_kind from e20_plan.check_runs cre1_0,e20_plan.monitors me1_0 where cre1_0.monitor_id=me1_0.id and me1_0.id=? and me1_0.owner_user_id=? and cre1_0.finished_at is not null and cre1_0.state=? order by cre1_0.finished_at desc,cre1_0.id desc fetch first ? rows only
diff --git a/evidence/phase-1/E20/invocations.jsonl b/evidence/phase-1/E20/invocations.jsonl
new file mode 100644
index 0000000..7c4c276
--- /dev/null
+++ b/evidence/phase-1/E20/invocations.jsonl
@@ -0,0 +1 @@
+{"command":"mvn -B -ntp -f backend/pom.xml -Dtest=HistoryQueryPlanTest -De20.plan-proof=baseline test","startedAt":"2026-08-28T08:13:18.578Z","elapsedSeconds":7.662,"exitCode":0,"signal":null}
diff --git a/evidence/phase-1/E20/seed.sql b/evidence/phase-1/E20/seed.sql
new file mode 100644
index 0000000..fdba335
--- /dev/null
+++ b/evidence/phase-1/E20/seed.sql
@@ -0,0 +1,24 @@
+-- Exactly one fixed logical dataset. Users are inserted with runtime-only password hashes by the test.
+INSERT INTO e20_plan.monitors
+    (id, name, url, interval_seconds, enabled, created_at, updated_at, owner_user_id)
+SELECT ('00000000-0000-4000-9000-' || lpad(n::text, 12, '0'))::uuid,
+    chr(64 + n), 'http://public.e20.test/ok', 60, false,
+    '2026-08-28T00:00:00.000Z'::timestamptz, '2026-08-28T00:00:00.000Z'::timestamptz,
+    CASE WHEN n < 10 THEN '00000000-0000-4000-a000-000000000001'::uuid
+         ELSE '00000000-0000-4000-a000-000000000002'::uuid END
+FROM generate_series(1, 10) n;
+
+INSERT INTO e20_plan.check_runs
+    (id, monitor_id, trigger_kind, state, http_status, latency_ms, failure_reason,
+     started_at, finished_at, queued_at)
+SELECT ('00000000-0000-4000-8000-' || lpad(n::text, 12, '0'))::uuid,
+    ('00000000-0000-4000-9000-' || lpad((CASE WHEN n <= 90000 THEN 1 ELSE 2 + (n - 90001) / 1000 END)::text, 12, '0'))::uuid,
+    'MANUAL', CASE WHEN n % 100 = 0 THEN 'FAILED' ELSE 'SUCCEEDED' END,
+    CASE WHEN n % 100 = 0 THEN 503 ELSE 200 END, 0,
+    CASE WHEN n % 100 = 0 THEN 'HTTP_STATUS' ELSE NULL END,
+    '2026-08-28T00:00:00.000Z'::timestamptz + n * interval '1 millisecond',
+    '2026-08-28T00:00:00.000Z'::timestamptz + n * interval '1 millisecond',
+    '2026-08-28T00:00:00.000Z'::timestamptz + n * interval '1 millisecond'
+FROM generate_series(1, 99000) n;
+
+ANALYZE e20_plan.monitors, e20_plan.check_runs;


