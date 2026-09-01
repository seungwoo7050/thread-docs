## `Monitor와 모든 CheckRun의 권위 저장소를 PostgreSQL로 전환`

diff --git a/.github/workflows/ci.yml b/.github/workflows/ci.yml
index 010b39a..97eb08a 100644
--- a/.github/workflows/ci.yml
+++ b/.github/workflows/ci.yml
@@ -1,4 +1,4 @@
-name: E01 verification
+name: E03 verification
 on: [push, pull_request]
 permissions:
   contents: read
@@ -28,5 +28,8 @@ jobs:
       - run: npm install --global npm@11.17.0
       - run: npm ci
       - run: npx playwright install --with-deps chromium
-      - name: Unit, functional, type, build and minimum browser gates
+      - name: Unit, PostgreSQL, functional, restart, type, build and browser gates
         run: npm run verify
+      - name: Remove only the isolated CI database project
+        if: always()
+        run: npm run db:destroy
diff --git a/backend/pom.xml b/backend/pom.xml
index 26ef890..9d6a9b1 100644
--- a/backend/pom.xml
+++ b/backend/pom.xml
@@ -19,6 +19,19 @@
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-web</artifactId>
     </dependency>
+    <dependency>
+      <groupId>org.springframework.boot</groupId>
+      <artifactId>spring-boot-starter-data-jpa</artifactId>
+    </dependency>
+    <dependency>
+      <groupId>org.flywaydb</groupId>
+      <artifactId>flyway-database-postgresql</artifactId>
+    </dependency>
+    <dependency>
+      <groupId>org.postgresql</groupId>
+      <artifactId>postgresql</artifactId>
+      <scope>runtime</scope>
+    </dependency>
     <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-test</artifactId>
diff --git a/backend/src/main/java/dev/evolution/monitor/CheckRunEntity.java b/backend/src/main/java/dev/evolution/monitor/CheckRunEntity.java
new file mode 100644
index 0000000..539a141
--- /dev/null
+++ b/backend/src/main/java/dev/evolution/monitor/CheckRunEntity.java
@@ -0,0 +1,47 @@
+package dev.evolution.monitor;
+
+import jakarta.persistence.Column;
+import jakarta.persistence.Entity;
+import jakarta.persistence.Id;
+import jakarta.persistence.Table;
+import java.time.Instant;
+import java.time.temporal.ChronoUnit;
+import java.util.UUID;
+
+@Entity
+@Table(name = "check_runs")
+public class CheckRunEntity {
+    @Id @Column(name = "id", nullable = false) private UUID id;
+    // The database foreign key owns deletion; no lazy entity graph escapes the store.
+    @Column(name = "monitor_id", nullable = false) private UUID monitorId;
+    @Column(name = "trigger_kind", nullable = false, length = 16) private String trigger;
+    @Column(name = "state", nullable = false, length = 16) private String state;
+    @Column(name = "http_status") private Integer httpStatus;
+    @Column(name = "latency_ms", nullable = false) private long latencyMs;
+    @Column(name = "failure_reason", length = 32) private String failureReason;
+    @Column(name = "started_at", nullable = false, columnDefinition = "timestamp(6) with time zone")
+    private Instant startedAt;
+    @Column(name = "finished_at", nullable = false, columnDefinition = "timestamp(6) with time zone")
+    private Instant finishedAt;
+
+    protected CheckRunEntity() {}
+
+    static CheckRunEntity fromDomain(CheckRunner.CheckRun check) {
+        var row = new CheckRunEntity();
+        row.id = check.id();
+        row.monitorId = check.monitorId();
+        row.trigger = check.trigger();
+        row.state = check.state();
+        row.httpStatus = check.httpStatus();
+        row.latencyMs = check.latencyMs();
+        row.failureReason = check.failureReason();
+        row.startedAt = check.startedAt().truncatedTo(ChronoUnit.MICROS);
+        row.finishedAt = check.finishedAt().truncatedTo(ChronoUnit.MICROS);
+        return row;
+    }
+
+    CheckRunner.CheckRun toDomain() {
+        return new CheckRunner.CheckRun(id, monitorId, trigger, state, httpStatus, latencyMs,
+                failureReason, startedAt, finishedAt);
+    }
+}
diff --git a/backend/src/main/java/dev/evolution/monitor/Monitor.java b/backend/src/main/java/dev/evolution/monitor/Monitor.java
new file mode 100644
index 0000000..467a972
--- /dev/null
+++ b/backend/src/main/java/dev/evolution/monitor/Monitor.java
@@ -0,0 +1,8 @@
+package dev.evolution.monitor;
+
+import java.time.Instant;
+import java.util.UUID;
+
+// Canonical application/wire value; never a managed persistence entity.
+public record Monitor(UUID id, String name, String url, int interval, boolean enabled,
+                      Instant createdAt, Instant updatedAt) {}
diff --git a/backend/src/main/java/dev/evolution/monitor/MonitorController.java b/backend/src/main/java/dev/evolution/monitor/MonitorController.java
index 18d963c..e066e10 100644
--- a/backend/src/main/java/dev/evolution/monitor/MonitorController.java
+++ b/backend/src/main/java/dev/evolution/monitor/MonitorController.java
@@ -2,16 +2,14 @@ package dev.evolution.monitor;
 
 import com.fasterxml.jackson.databind.JsonNode;
 import java.net.URI;
-import java.time.Instant;
-import java.util.Comparator;
 import java.util.List;
-import java.util.Map;
 import java.util.UUID;
-import java.util.concurrent.ConcurrentHashMap;
 import org.springframework.http.HttpStatus;
+import org.springframework.web.bind.annotation.DeleteMapping;
 import org.springframework.web.bind.annotation.GetMapping;
 import org.springframework.web.bind.annotation.PathVariable;
 import org.springframework.web.bind.annotation.PostMapping;
+import org.springframework.web.bind.annotation.PutMapping;
 import org.springframework.web.bind.annotation.RequestBody;
 import org.springframework.web.bind.annotation.RequestMapping;
 import org.springframework.web.bind.annotation.ResponseStatus;
@@ -21,18 +19,17 @@ import org.springframework.web.server.ResponseStatusException;
 @RestController
 @RequestMapping("/api/monitors")
 public class MonitorController {
-    private final Map<UUID, Monitor> monitors = new ConcurrentHashMap<>();
-    private final Map<UUID, CheckRunner.CheckRun> latestChecks = new ConcurrentHashMap<>();
     private final CheckRunner checks;
+    private final MonitorStore store;
 
-    public MonitorController(CheckRunner checks) {
+    public MonitorController(CheckRunner checks, MonitorStore store) {
         this.checks = checks;
+        this.store = store;
     }
 
     @GetMapping
     public ApiData<List<MonitorView>> list() {
-        return new ApiData<>(monitors.values().stream().sorted(Comparator.comparing(Monitor::createdAt))
-                .map(monitor -> new MonitorView(monitor, latestChecks.get(monitor.id()))).toList());
+        return new ApiData<>(store.list());
     }
 
     @PostMapping
@@ -40,20 +37,43 @@ public class MonitorController {
     public ApiData<MonitorView> create(@RequestBody JsonNode body) {
         CreateMonitor input = CreateMonitor.fromJson(body);
         checks.requireFixtureUrl(input.url());
-        Instant now = Instant.now();
-        Monitor monitor = new Monitor(UUID.randomUUID(), input.name(), input.url(), input.interval(),
-                input.enabled(), now, now);
-        monitors.put(monitor.id(), monitor);
-        return new ApiData<>(new MonitorView(monitor, null));
+        return new ApiData<>(store.create(input));
+    }
+
+    @GetMapping("/{id}")
+    public ApiData<MonitorView> get(@PathVariable UUID id) {
+        return new ApiData<>(store.get(id));
+    }
+
+    @PutMapping("/{id}")
+    public ApiData<MonitorView> replace(@PathVariable UUID id, @RequestBody JsonNode body) {
+        CreateMonitor input = CreateMonitor.fromJson(body);
+        checks.requireFixtureUrl(input.url());
+        return new ApiData<>(store.replace(id, input));
+    }
+
+    @DeleteMapping("/{id}")
+    public ApiData<Void> delete(@PathVariable UUID id) {
+        store.delete(id);
+        return new ApiData<>(null);
     }
 
     @PostMapping("/{id}/checks")
     public ApiData<CheckRunner.CheckRun> check(@PathVariable UUID id) {
-        Monitor monitor = monitors.get(id);
-        if (monitor == null) throw new ResponseStatusException(HttpStatus.NOT_FOUND, "Monitor not found");
+        // Each store call crosses the Spring transaction proxy. Outbound I/O holds no transaction.
+        Monitor monitor = store.monitor(id);
         CheckRunner.CheckRun result = checks.run(monitor.id(), monitor.url());
-        latestChecks.put(id, result);
-        return new ApiData<>(result);
+        return new ApiData<>(store.saveCheck(result));
+    }
+
+    @GetMapping("/{id}/checks")
+    public ApiData<List<CheckRunner.CheckRun>> history(@PathVariable UUID id) {
+        return new ApiData<>(store.history(id));
+    }
+
+    @GetMapping("/{id}/checks/{checkId}")
+    public ApiData<CheckRunner.CheckRun> check(@PathVariable UUID id, @PathVariable UUID checkId) {
+        return new ApiData<>(store.check(id, checkId));
     }
 
     public record CreateMonitor(String name, String url, int interval, boolean enabled) {
@@ -68,6 +88,9 @@ public class MonitorController {
             if (name.isEmpty() || name.length() > 100) {
                 throw invalid("Name must contain 1 to 100 UTF-16 code units after trimming.");
             }
+            if (name.indexOf('\0') >= 0) {
+                throw invalid("Name must not contain a NUL character.");
+            }
             int interval;
             try {
                 // 60.0 has the integer value 60; strings and fractional values are not coerced.
@@ -96,7 +119,5 @@ public class MonitorController {
             return new ResponseStatusException(HttpStatus.BAD_REQUEST, message);
         }
     }
-    public record Monitor(UUID id, String name, String url, int interval, boolean enabled,
-                          Instant createdAt, Instant updatedAt) {}
     public record MonitorView(Monitor monitor, CheckRunner.CheckRun latestCheck) {}
 }
diff --git a/backend/src/main/java/dev/evolution/monitor/MonitorEntity.java b/backend/src/main/java/dev/evolution/monitor/MonitorEntity.java
new file mode 100644
index 0000000..d8afb35
--- /dev/null
+++ b/backend/src/main/java/dev/evolution/monitor/MonitorEntity.java
@@ -0,0 +1,50 @@
+package dev.evolution.monitor;
+
+import jakarta.persistence.Column;
+import jakarta.persistence.Entity;
+import jakarta.persistence.Id;
+import jakarta.persistence.Table;
+import java.time.Instant;
+import java.time.temporal.ChronoUnit;
+import java.util.UUID;
+
+@Entity
+@Table(name = "monitors")
+public class MonitorEntity {
+    @Id @Column(name = "id", nullable = false) private UUID id;
+    @Column(name = "name", nullable = false, length = 100) private String name;
+    @Column(name = "url", nullable = false, columnDefinition = "text") private String url;
+    @Column(name = "interval_seconds", nullable = false) private int intervalSeconds;
+    @Column(name = "enabled", nullable = false) private boolean enabled;
+    @Column(name = "created_at", nullable = false, columnDefinition = "timestamp(6) with time zone")
+    private Instant createdAt;
+    @Column(name = "updated_at", nullable = false, columnDefinition = "timestamp(6) with time zone")
+    private Instant updatedAt;
+
+    protected MonitorEntity() {}
+
+    static MonitorEntity fromDomain(Monitor monitor) {
+        var row = new MonitorEntity();
+        row.id = monitor.id();
+        row.name = monitor.name();
+        row.url = monitor.url();
+        row.intervalSeconds = monitor.interval();
+        row.enabled = monitor.enabled();
+        // Canonical precision is applied before the initial response, not only on a later DB read.
+        row.createdAt = monitor.createdAt().truncatedTo(ChronoUnit.MICROS);
+        row.updatedAt = monitor.updatedAt().truncatedTo(ChronoUnit.MICROS);
+        return row;
+    }
+
+    void replace(MonitorController.CreateMonitor input) {
+        name = input.name();
+        url = input.url();
+        intervalSeconds = input.interval();
+        enabled = input.enabled();
+        updatedAt = Instant.now().truncatedTo(ChronoUnit.MICROS);
+    }
+
+    Monitor toDomain() {
+        return new Monitor(id, name, url, intervalSeconds, enabled, createdAt, updatedAt);
+    }
+}
diff --git a/backend/src/main/java/dev/evolution/monitor/MonitorStore.java b/backend/src/main/java/dev/evolution/monitor/MonitorStore.java
new file mode 100644
index 0000000..5b52e5d
--- /dev/null
+++ b/backend/src/main/java/dev/evolution/monitor/MonitorStore.java
@@ -0,0 +1,92 @@
+package dev.evolution.monitor;
+
+import jakarta.persistence.EntityManager;
+import jakarta.persistence.PersistenceContext;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.springframework.http.HttpStatus;
+import org.springframework.stereotype.Repository;
+import org.springframework.transaction.annotation.Transactional;
+import org.springframework.web.server.ResponseStatusException;
+
+@Repository
+@Transactional(readOnly = true)
+public class MonitorStore {
+    @PersistenceContext private EntityManager entities;
+
+    public List<MonitorController.MonitorView> list() {
+        return entities.createQuery("select m from MonitorEntity m order by m.createdAt, m.id", MonitorEntity.class)
+                .getResultList().stream().map(this::view).toList();
+    }
+
+    public MonitorController.MonitorView get(UUID id) {
+        return view(requireMonitor(id));
+    }
+
+    public Monitor monitor(UUID id) {
+        return requireMonitor(id).toDomain();
+    }
+
+    @Transactional
+    public MonitorController.MonitorView create(MonitorController.CreateMonitor input) {
+        Instant now = Instant.now();
+        var row = MonitorEntity.fromDomain(new Monitor(UUID.randomUUID(), input.name(), input.url(),
+                input.interval(), input.enabled(), now, now));
+        entities.persist(row);
+        return new MonitorController.MonitorView(row.toDomain(), null);
+    }
+
+    @Transactional
+    public MonitorController.MonitorView replace(UUID id, MonitorController.CreateMonitor input) {
+        var row = requireMonitor(id);
+        row.replace(input); // Managed entity dirty checking is flushed before the proxy returns.
+        return view(row);
+    }
+
+    @Transactional
+    public void delete(UUID id) {
+        entities.remove(requireMonitor(id)); // PostgreSQL ON DELETE CASCADE removes every historical child.
+    }
+
+    @Transactional
+    public CheckRunner.CheckRun saveCheck(CheckRunner.CheckRun check) {
+        requireMonitor(check.monitorId());
+        var row = CheckRunEntity.fromDomain(check);
+        entities.persist(row);
+        return row.toDomain();
+    }
+
+    public List<CheckRunner.CheckRun> history(UUID id) {
+        requireMonitor(id);
+        return entities.createQuery("select c from CheckRunEntity c where c.monitorId = :id "
+                        + "order by c.finishedAt desc, c.id desc", CheckRunEntity.class)
+                .setParameter("id", id).getResultList().stream().map(CheckRunEntity::toDomain).toList();
+    }
+
+    public CheckRunner.CheckRun check(UUID monitorId, UUID checkId) {
+        requireMonitor(monitorId);
+        return entities.createQuery("select c from CheckRunEntity c where c.id = :checkId and c.monitorId = :monitorId",
+                        CheckRunEntity.class).setParameter("checkId", checkId).setParameter("monitorId", monitorId)
+                .getResultStream().findFirst().map(CheckRunEntity::toDomain).orElseThrow(MonitorStore::notFound);
+    }
+
+    private MonitorEntity requireMonitor(UUID id) {
+        var row = entities.find(MonitorEntity.class, id);
+        if (row == null) throw notFound();
+        return row;
+    }
+
+    private MonitorController.MonitorView view(MonitorEntity row) {
+        Monitor monitor = row.toDomain();
+        var latest = entities.createQuery("select c from CheckRunEntity c where c.monitorId = :id "
+                        + "order by c.finishedAt desc, c.id desc", CheckRunEntity.class)
+                .setParameter("id", monitor.id()).setMaxResults(1).getResultStream()
+                .findFirst().map(CheckRunEntity::toDomain).orElse(null);
+        return new MonitorController.MonitorView(monitor, latest);
+    }
+
+    private static ResponseStatusException notFound() {
+        return new ResponseStatusException(HttpStatus.NOT_FOUND, "Resource not found");
+    }
+}
diff --git a/backend/src/main/java/dev/evolution/monitor/SchemaCompatibility.java b/backend/src/main/java/dev/evolution/monitor/SchemaCompatibility.java
new file mode 100644
index 0000000..70270a2
--- /dev/null
+++ b/backend/src/main/java/dev/evolution/monitor/SchemaCompatibility.java
@@ -0,0 +1,56 @@
+package dev.evolution.monitor;
+
+import jakarta.persistence.Column;
+import jakarta.persistence.EntityManagerFactory;
+import jakarta.persistence.Table;
+import java.sql.SQLException;
+import java.util.Arrays;
+import java.util.Map;
+import java.util.Set;
+import java.util.stream.Collectors;
+import javax.sql.DataSource;
+import org.springframework.beans.factory.InitializingBean;
+import org.springframework.beans.factory.annotation.Value;
+import org.springframework.stereotype.Component;
+
+@Component
+public class SchemaCompatibility implements InitializingBean {
+    private final DataSource database;
+    private final String schema;
+
+    public SchemaCompatibility(DataSource database, EntityManagerFactory validatedMappings,
+            @Value("${spring.jpa.properties.hibernate.default_schema}") String schema) {
+        // The factory dependency ensures Flyway and Hibernate's required-column/type validation finished first.
+        this.database = database;
+        this.schema = schema;
+    }
+
+    @Override
+    public void afterPropertiesSet() throws SQLException {
+        Map<String, Set<String>> mapped = Map.of(
+                MonitorEntity.class.getAnnotation(Table.class).name(), columns(MonitorEntity.class),
+                CheckRunEntity.class.getAnnotation(Table.class).name(), columns(CheckRunEntity.class));
+        // Hibernate validate does not reject extra NOT NULL columns which make its INSERTs impossible.
+        try (var connection = database.getConnection(); var query = connection.prepareStatement("""
+                SELECT table_name, column_name FROM information_schema.columns
+                WHERE table_schema = ? AND is_nullable = 'NO' AND column_default IS NULL
+                  AND is_identity = 'NO' AND is_generated = 'NEVER'
+                """)) {
+            query.setString(1, schema);
+            try (var rows = query.executeQuery()) {
+                while (rows.next()) {
+                    String table = rows.getString("table_name");
+                    String column = rows.getString("column_name");
+                    if (mapped.containsKey(table) && !mapped.get(table).contains(column)) {
+                        throw new IllegalStateException("Unmapped required column: " + table + "." + column);
+                    }
+                }
+            }
+        }
+    }
+
+    private static Set<String> columns(Class<?> entity) {
+        return Arrays.stream(entity.getDeclaredFields()).map(field -> field.getAnnotation(Column.class))
+                .filter(column -> column != null).map(Column::name).collect(Collectors.toUnmodifiableSet());
+    }
+}
diff --git a/backend/src/main/resources/application.properties b/backend/src/main/resources/application.properties
index ab830cb..3c72628 100644
--- a/backend/src/main/resources/application.properties
+++ b/backend/src/main/resources/application.properties
@@ -3,3 +3,13 @@ server.address=127.0.0.1
 server.port=${API_PORT:4322}
 monitor.fixture-origin=${FIXTURE_ORIGIN:http://127.0.0.1:4321}
 spring.jackson.deserialization.fail-on-trailing-tokens=true
+spring.datasource.url=${DB_URL:jdbc:postgresql://127.0.0.1:15432/monitor}
+spring.datasource.username=${DB_USER:wse_industry}
+spring.datasource.password=${DB_PASSWORD:}
+spring.flyway.schemas=${DB_SCHEMA:public}
+spring.flyway.default-schema=${DB_SCHEMA:public}
+spring.flyway.clean-disabled=true
+spring.jpa.hibernate.ddl-auto=validate
+spring.jpa.open-in-view=false
+spring.jpa.properties.hibernate.default_schema=${DB_SCHEMA:public}
+spring.jpa.properties.hibernate.jdbc.time_zone=UTC
diff --git a/backend/src/main/resources/db/migration/V1__create_monitors.sql b/backend/src/main/resources/db/migration/V1__create_monitors.sql
new file mode 100644
index 0000000..fee7045
--- /dev/null
+++ b/backend/src/main/resources/db/migration/V1__create_monitors.sql
@@ -0,0 +1,9 @@
+CREATE TABLE monitors (
+    id uuid PRIMARY KEY,
+    name varchar(100) NOT NULL CHECK (length(name) > 0),
+    url text NOT NULL,
+    interval_seconds integer NOT NULL CHECK (interval_seconds BETWEEN 1 AND 86400),
+    enabled boolean NOT NULL,
+    created_at timestamp(6) with time zone NOT NULL,
+    updated_at timestamp(6) with time zone NOT NULL
+);
diff --git a/backend/src/main/resources/db/migration/V2__create_check_runs.sql b/backend/src/main/resources/db/migration/V2__create_check_runs.sql
new file mode 100644
index 0000000..6429759
--- /dev/null
+++ b/backend/src/main/resources/db/migration/V2__create_check_runs.sql
@@ -0,0 +1,16 @@
+CREATE TABLE check_runs (
+    id uuid PRIMARY KEY,
+    monitor_id uuid NOT NULL REFERENCES monitors(id) ON DELETE CASCADE,
+    trigger_kind varchar(16) NOT NULL CHECK (trigger_kind = 'MANUAL'),
+    state varchar(16) NOT NULL,
+    http_status integer CHECK (http_status BETWEEN 100 AND 599),
+    latency_ms bigint NOT NULL CHECK (latency_ms >= 0),
+    failure_reason varchar(32),
+    started_at timestamp(6) with time zone NOT NULL,
+    finished_at timestamp(6) with time zone NOT NULL CHECK (finished_at >= started_at),
+    CONSTRAINT check_runs_observed_outcome CHECK ((
+        (state = 'SUCCEEDED' AND http_status BETWEEN 200 AND 299 AND failure_reason IS NULL)
+        OR (state = 'FAILED' AND http_status NOT BETWEEN 200 AND 299 AND failure_reason = 'HTTP_STATUS')
+        OR (state = 'FAILED' AND http_status IS NULL AND failure_reason IN ('TIMEOUT', 'CONNECTION_FAILURE'))
+    ) IS TRUE)
+);
diff --git a/backend/src/test/java/dev/evolution/monitor/ApiErrorBoundaryTest.java b/backend/src/test/java/dev/evolution/monitor/ApiErrorBoundaryTest.java
index d018ba8..8bfba92 100644
--- a/backend/src/test/java/dev/evolution/monitor/ApiErrorBoundaryTest.java
+++ b/backend/src/test/java/dev/evolution/monitor/ApiErrorBoundaryTest.java
@@ -16,7 +16,7 @@ class ApiErrorBoundaryTest {
         CheckRunner checks = mock(CheckRunner.class);
         when(checks.requireFixtureUrl("http://127.0.0.1:4321/ok"))
                 .thenThrow(new IllegalStateException("Private implementation detail"));
-        var mvc = MockMvcBuilders.standaloneSetup(new MonitorController(checks))
+        var mvc = MockMvcBuilders.standaloneSetup(new MonitorController(checks, mock(MonitorStore.class)))
                 .setControllerAdvice(new ApiErrors()).build();
         String body = mvc.perform(post("/api/monitors").contentType(MediaType.APPLICATION_JSON).content("""
                 {"name":"Fixture monitor","url":"http://127.0.0.1:4321/ok","interval":60,"enabled":true}
diff --git a/backend/src/test/java/dev/evolution/monitor/MonitorFunctionalTest.java b/backend/src/test/java/dev/evolution/monitor/MonitorFunctionalTest.java
index 1ee7395..349fd94 100644
--- a/backend/src/test/java/dev/evolution/monitor/MonitorFunctionalTest.java
+++ b/backend/src/test/java/dev/evolution/monitor/MonitorFunctionalTest.java
@@ -1,6 +1,8 @@
 package dev.evolution.monitor;
 
 import static org.junit.jupiter.api.Assertions.*;
+import static org.mockito.ArgumentMatchers.*;
+import static org.mockito.Mockito.doAnswer;
 import com.fasterxml.jackson.databind.JsonNode;
 import com.fasterxml.jackson.databind.ObjectMapper;
 import com.fasterxml.jackson.databind.node.ObjectNode;
@@ -23,8 +25,15 @@ import org.springframework.http.HttpEntity;
 import org.springframework.http.HttpHeaders;
 import org.springframework.http.MediaType;
 import org.springframework.http.ResponseEntity;
+import org.springframework.http.HttpMethod;
+import org.springframework.test.annotation.DirtiesContext;
+import org.springframework.test.context.DynamicPropertyRegistry;
+import org.springframework.test.context.DynamicPropertySource;
+import org.springframework.test.context.bean.override.mockito.MockitoSpyBean;
+import org.springframework.transaction.support.TransactionSynchronizationManager;
 
 @SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
+@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_CLASS)
 class MonitorFunctionalTest {
     static HttpServer fixture;
     static HttpServer forbidden;
@@ -32,6 +41,12 @@ class MonitorFunctionalTest {
     static final AtomicInteger forbiddenRequests = new AtomicInteger();
     @Autowired TestRestTemplate api;
     @Autowired ObjectMapper json;
+    @MockitoSpyBean CheckRunner checks;
+
+    @DynamicPropertySource
+    static void database(DynamicPropertyRegistry properties) {
+        TestDatabase.configure(properties, "e03_functional");
+    }
 
     @BeforeAll
     static void startFixture() throws IOException {
@@ -74,6 +89,7 @@ class MonitorFunctionalTest {
     static void stopFixture() {
         if (fixture != null) fixture.stop(0);
         if (forbidden != null) forbidden.stop(0);
+        TestDatabase.drop("e03_functional");
     }
 
     @Test
@@ -249,6 +265,35 @@ class MonitorFunctionalTest {
         assertEquals(0, forbiddenRequests.get());
     }
 
+    @Test
+    void postgresTextBoundaryRejectsNulWithoutCreationOrReplacement() {
+        JsonNode original = assertDataEnvelope(postJson(validInput().toString()), HttpStatus.CREATED);
+        String id = original.at("/monitor/id").textValue();
+        JsonNode before = assertDataEnvelope(api.getForEntity("/api/monitors", JsonNode.class), HttpStatus.OK);
+        String body = validInput().put("name", "A\u0000B").toString();
+        assertApiError(postJson(body), HttpStatus.BAD_REQUEST, "INVALID_INPUT");
+        var headers = new HttpHeaders();
+        headers.setContentType(MediaType.APPLICATION_JSON);
+        assertApiError(api.exchange("/api/monitors/" + id, HttpMethod.PUT, new HttpEntity<>(body, headers), JsonNode.class),
+                HttpStatus.BAD_REQUEST, "INVALID_INPUT");
+        assertEquals(original, assertDataEnvelope(api.getForEntity("/api/monitors/" + id, JsonNode.class), HttpStatus.OK));
+        assertEquals(before, assertDataEnvelope(api.getForEntity("/api/monitors", JsonNode.class), HttpStatus.OK));
+    }
+
+    @Test
+    void synchronousOutboundRequestDoesNotHoldAStoreTransaction() {
+        doAnswer(invocation -> {
+            assertFalse(TransactionSynchronizationManager.isActualTransactionActive());
+            return invocation.callRealMethod();
+        }).when(checks).run(any(UUID.class), anyString());
+        var view = create("/ok");
+        var observed = check(view);
+        JsonNode history = assertDataEnvelope(api.getForEntity("/api/monitors/" + view.monitor().id() + "/checks",
+                JsonNode.class), HttpStatus.OK);
+        assertEquals(1, history.size());
+        assertEquals(observed, json.convertValue(history.get(0), CheckRunner.CheckRun.class));
+    }
+
     private MonitorController.MonitorView create(String path) {
         var response = api.postForEntity("/api/monitors", new MonitorController.CreateMonitor(
                 "Fixture monitor", "http://127.0.0.1:4321" + path, 60, true), JsonNode.class);
diff --git a/backend/src/test/java/dev/evolution/monitor/PostgresPersistenceTest.java b/backend/src/test/java/dev/evolution/monitor/PostgresPersistenceTest.java
new file mode 100644
index 0000000..da64061
--- /dev/null
+++ b/backend/src/test/java/dev/evolution/monitor/PostgresPersistenceTest.java
@@ -0,0 +1,230 @@
+package dev.evolution.monitor;
+
+import static org.junit.jupiter.api.Assertions.*;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import jakarta.persistence.EntityManagerFactory;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.time.Instant;
+import java.time.OffsetDateTime;
+import java.time.temporal.ChronoUnit;
+import java.util.List;
+import java.util.UUID;
+import java.util.concurrent.CopyOnWriteArrayList;
+import java.util.concurrent.atomic.AtomicBoolean;
+import java.util.concurrent.atomic.AtomicReference;
+import org.hibernate.resource.jdbc.spi.StatementInspector;
+import org.junit.jupiter.api.Test;
+import org.springframework.aop.support.AopUtils;
+import org.springframework.boot.WebApplicationType;
+import org.springframework.boot.builder.SpringApplicationBuilder;
+import org.springframework.boot.web.context.WebServerInitializedEvent;
+import org.springframework.context.ApplicationListener;
+import org.springframework.context.ConfigurableApplicationContext;
+import org.springframework.orm.jpa.SharedEntityManagerCreator;
+import org.springframework.transaction.PlatformTransactionManager;
+import org.springframework.transaction.support.TransactionSynchronizationManager;
+import org.springframework.transaction.support.TransactionTemplate;
+import org.springframework.web.server.ResponseStatusException;
+
+class PostgresPersistenceTest {
+    private static final UUID MONITOR_ID = UUID.fromString("00000000-0000-4000-8000-000000000031");
+    private static final Instant NANO_TIME = Instant.parse("2026-08-28T00:00:00.123456789Z");
+    private static final Instant DATABASE_TIME = Instant.parse("2026-08-28T00:00:00.123456Z");
+
+    @Test
+    void freshMigrationChainRepeatsWithoutDdlAndUpgradesAnIndependentV1Schema() {
+        String schema = "e03_migrations";
+        TestDatabase.reset(schema);
+        try {
+            var fresh = TestDatabase.migration(schema).load();
+            assertEquals(2, fresh.migrate().migrationsExecuted);
+            assertEquals(List.of("1", "2"), java.util.Arrays.stream(fresh.info().applied())
+                    .map(migration -> migration.getVersion().toString()).toList());
+            fresh.validate();
+            assertEquals(0, fresh.migrate().migrationsExecuted);
+            assertEquals(2, fresh.info().applied().length);
+
+            TestDatabase.reset(schema);
+            assertEquals(1, TestDatabase.migration(schema).target("1").load().migrate().migrationsExecuted);
+            var upgrade = TestDatabase.migration(schema).load();
+            assertEquals(1, upgrade.migrate().migrationsExecuted);
+            upgrade.validate();
+            assertEquals(2, upgrade.info().applied().length);
+            assertEquals(0, upgrade.migrate().migrationsExecuted);
+        } finally {
+            TestDatabase.drop(schema);
+        }
+    }
+
+    @Test
+    void mappingAndEveryHistoricalRunSurviveAClosedApplicationContext() throws Exception {
+        String schema = "e03_mapping";
+        TestDatabase.reset(schema);
+        SqlEvidence.events.clear();
+        var monitor = new Monitor(MONITOR_ID, "Mapping fixture", "http://127.0.0.1:4321/ok", 1, false,
+                NANO_TIME, NANO_TIME);
+        var successInput = new CheckRunner.CheckRun(UUID.fromString("00000000-0000-4000-8000-000000000032"),
+                MONITOR_ID, "MANUAL", "SUCCEEDED", 200, 0, null, NANO_TIME, NANO_TIME);
+        var timeoutInput = new CheckRunner.CheckRun(UUID.fromString("00000000-0000-4000-8000-000000000033"),
+                MONITOR_ID, "MANUAL", "FAILED", null, 2000, "TIMEOUT", NANO_TIME, NANO_TIME.plusSeconds(2));
+        var success = canonical(successInput);
+        var timeout = canonical(timeoutInput);
+        try {
+            try (var first = start(schema)) {
+                var store = first.getBean(MonitorStore.class);
+                assertTrue(AopUtils.isAopProxy(store), "Transactions must use the actual Spring store proxy");
+                var transaction = new TransactionTemplate(first.getBean(PlatformTransactionManager.class));
+                var entities = SharedEntityManagerCreator.createSharedEntityManager(first.getBean(EntityManagerFactory.class));
+                transaction.executeWithoutResult(status -> entities.persist(MonitorEntity.fromDomain(monitor)));
+                assertEquals(success, store.saveCheck(successInput));
+                assertEquals(timeout, store.saveCheck(timeoutInput));
+
+                var rolledBack = new AtomicReference<UUID>();
+                transaction.executeWithoutResult(status -> {
+                    rolledBack.set(store.create(new MonitorController.CreateMonitor("Rolled back",
+                            "http://127.0.0.1:4321/ok", 60, true)).monitor().id());
+                    entities.flush(); // Make the INSERT execute before the rollback assertion.
+                    status.setRollbackOnly();
+                });
+                assertEquals(404, assertThrows(ResponseStatusException.class,
+                        () -> store.monitor(rolledBack.get())).getStatusCode().value());
+            }
+
+            try (var fresh = start(schema)) {
+                var store = fresh.getBean(MonitorStore.class);
+                assertEquals(new Monitor(MONITOR_ID, monitor.name(), monitor.url(), 1, false,
+                        DATABASE_TIME, DATABASE_TIME), store.monitor(MONITOR_ID));
+                assertEquals(List.of(timeout, success), store.history(MONITOR_ID));
+                assertEquals(success, store.check(MONITOR_ID, success.id()));
+                assertEquals(timeout, store.check(MONITOR_ID, timeout.id()));
+                assertEquals(timeout, store.get(MONITOR_ID).latestCheck());
+                assertEquals(1, store.list().size());
+
+                var json = fresh.getBean(ObjectMapper.class);
+                var monitorWire = json.valueToTree(new ApiData<>(store.monitor(MONITOR_ID)));
+                assertTrue(monitorWire.at("/data/interval").isIntegralNumber());
+                assertEquals(1, monitorWire.at("/data/interval").intValue());
+                assertTrue(monitorWire.at("/data/enabled").isBoolean());
+                assertFalse(monitorWire.at("/data/enabled").booleanValue());
+                assertEquals(DATABASE_TIME.toString(), monitorWire.at("/data/createdAt").textValue());
+                var successWire = json.valueToTree(new ApiData<>(store.check(MONITOR_ID, success.id())));
+                assertTrue(successWire.at("/data/latencyMs").isIntegralNumber());
+                assertEquals(0, successWire.at("/data/latencyMs").longValue());
+                assertTrue(successWire.at("/data/failureReason").isNull());
+                var timeoutWire = json.valueToTree(new ApiData<>(store.check(MONITOR_ID, timeout.id())));
+                assertTrue(timeoutWire.at("/data/httpStatus").isNull());
+                assertEquals("TIMEOUT", timeoutWire.at("/data/failureReason").textValue());
+
+                // Read the actual PostgreSQL representations in a non-UTC session.
+                try (var connection = TestDatabase.connect(); var statement = connection.createStatement()) {
+                    statement.execute("SET TIME ZONE 'Asia/Seoul'");
+                    try (var rows = statement.executeQuery("SELECT interval_seconds, enabled, created_at FROM e03_mapping.monitors")) {
+                        assertTrue(rows.next());
+                        assertEquals(Integer.valueOf(1), rows.getObject("interval_seconds"));
+                        assertEquals(Boolean.FALSE, rows.getObject("enabled"));
+                        assertEquals(DATABASE_TIME, rows.getObject("created_at", OffsetDateTime.class).toInstant());
+                        assertFalse(rows.next());
+                    }
+                    try (var rows = statement.executeQuery("SELECT latency_ms, failure_reason FROM e03_mapping.check_runs WHERE http_status = 200")) {
+                        assertTrue(rows.next());
+                        assertEquals(Long.valueOf(0), rows.getObject("latency_ms"));
+                        assertNull(rows.getObject("failure_reason"));
+                    }
+                }
+
+                var updated = store.replace(MONITOR_ID, new MonitorController.CreateMonitor(
+                        "Mapping updated", monitor.url(), 60, true));
+                assertEquals(updated, store.get(MONITOR_ID));
+                assertEquals("Mapping updated", updated.monitor().name());
+                assertTrue(updated.monitor().enabled());
+                store.delete(MONITOR_ID);
+                assertEquals(404, assertThrows(ResponseStatusException.class,
+                        () -> store.history(MONITOR_ID)).getStatusCode().value());
+                try (var connection = TestDatabase.connect(); var query = connection.createStatement();
+                        var rows = query.executeQuery("SELECT (SELECT count(*) FROM e03_mapping.monitors) AS monitors, "
+                                + "(SELECT count(*) FROM e03_mapping.check_runs) AS checks")) {
+                    assertTrue(rows.next());
+                    assertEquals(0, rows.getInt("monitors"));
+                    assertEquals(0, rows.getInt("checks"), "Database cascade must leave no historical children");
+                }
+            }
+            assertTrue(SqlEvidence.events.stream().allMatch(SqlEvent::transaction), "All ORM SQL must run in a real transaction");
+            for (String operation : List.of("insert into e03_mapping.monitors", "insert into e03_mapping.check_runs",
+                    "update e03_mapping.monitors", "delete from e03_mapping.monitors")) {
+                assertTrue(SqlEvidence.events.stream().anyMatch(event -> event.sql().startsWith(operation)), operation);
+            }
+            assertTrue(SqlEvidence.events.stream().anyMatch(event -> event.readOnly() && event.sql().contains("from e03_mapping.check_runs")));
+            Files.createDirectories(Path.of("target"));
+            Files.writeString(Path.of("target/e03-generated-sql.txt"), String.join("\n",
+                    SqlEvidence.events.stream().map(Object::toString).toList()) + "\n");
+        } finally {
+            TestDatabase.drop(schema);
+        }
+    }
+
+    @Test
+    void startupRejectsAMissingMappedColumnBeforeWebServerReady() {
+        assertStartupFailure("e03_missing_column", "ALTER TABLE e03_missing_column.monitors DROP COLUMN name",
+                "missing column [name]");
+    }
+
+    @Test
+    void startupRejectsAnExtraRequiredInsertColumnBeforeWebServerReady() {
+        assertStartupFailure("e03_extra_required",
+                "ALTER TABLE e03_extra_required.monitors ADD COLUMN unmapped_required text NOT NULL",
+                "Unmapped required column: monitors.unmapped_required");
+    }
+
+    private static void assertStartupFailure(String schema, String ddl, String expectedMessage) {
+        TestDatabase.reset(schema);
+        var ready = new AtomicBoolean();
+        var unexpectedContext = new AtomicReference<ConfigurableApplicationContext>();
+        try {
+            assertEquals(2, TestDatabase.migration(schema).load().migrate().migrationsExecuted);
+            TestDatabase.execute(ddl);
+            ApplicationListener<WebServerInitializedEvent> readyListener = event -> ready.set(true);
+            var failure = assertThrows(RuntimeException.class, () -> unexpectedContext.set(
+                    new SpringApplicationBuilder(MonitorApplication.class).web(WebApplicationType.SERVLET)
+                            .listeners(readyListener).run(arguments(schema))));
+            StringBuilder messages = new StringBuilder();
+            for (Throwable cause = failure; cause != null; cause = cause.getCause()) messages.append(cause.getMessage()).append('\n');
+            assertTrue(messages.toString().contains(expectedMessage), messages::toString);
+            assertFalse(ready.get(), "Incompatible schema must fail before the API accepts requests");
+        } finally {
+            if (unexpectedContext.get() != null) unexpectedContext.get().close();
+            TestDatabase.drop(schema);
+        }
+    }
+
+    private static ConfigurableApplicationContext start(String schema) {
+        return new SpringApplicationBuilder(MonitorApplication.class).web(WebApplicationType.NONE).run(arguments(schema));
+    }
+
+    private static String[] arguments(String schema) {
+        return new String[]{"--spring.flyway.schemas=" + schema, "--spring.flyway.default-schema=" + schema,
+                "--spring.jpa.properties.hibernate.default_schema=" + schema,
+                "--spring.jpa.properties.hibernate.session_factory.statement_inspector=" + SqlEvidence.class.getName(),
+                "--spring.main.banner-mode=off", "--logging.level.root=OFF", "--server.port=0"};
+    }
+
+    private static CheckRunner.CheckRun canonical(CheckRunner.CheckRun check) {
+        return new CheckRunner.CheckRun(check.id(), check.monitorId(), check.trigger(), check.state(), check.httpStatus(),
+                check.latencyMs(), check.failureReason(), check.startedAt().truncatedTo(ChronoUnit.MICROS),
+                check.finishedAt().truncatedTo(ChronoUnit.MICROS));
+    }
+
+    record SqlEvent(String sql, boolean transaction, boolean readOnly) {}
+
+    public static class SqlEvidence implements StatementInspector {
+        static final List<SqlEvent> events = new CopyOnWriteArrayList<>();
+
+        @Override
+        public String inspect(String sql) {
+            events.add(new SqlEvent(sql, TransactionSynchronizationManager.isActualTransactionActive(),
+                    TransactionSynchronizationManager.isCurrentTransactionReadOnly()));
+            return sql;
+        }
+    }
+}
diff --git a/backend/src/test/java/dev/evolution/monitor/TestDatabase.java b/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
new file mode 100644
index 0000000..b87e21e
--- /dev/null
+++ b/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
@@ -0,0 +1,52 @@
+package dev.evolution.monitor;
+
+import java.sql.Connection;
+import java.sql.DriverManager;
+import java.sql.SQLException;
+import java.util.Set;
+import org.flywaydb.core.Flyway;
+import org.flywaydb.core.api.configuration.FluentConfiguration;
+import org.springframework.test.context.DynamicPropertyRegistry;
+
+final class TestDatabase {
+    private static final Set<String> SCHEMAS = Set.of("e03_functional", "e03_migrations", "e03_mapping",
+            "e03_missing_column", "e03_extra_required");
+
+    static Connection connect() throws SQLException {
+        return DriverManager.getConnection(System.getenv().getOrDefault("DB_URL", "jdbc:postgresql://127.0.0.1:15432/monitor"),
+                System.getenv().getOrDefault("DB_USER", "wse_industry"), System.getenv().getOrDefault("DB_PASSWORD", ""));
+    }
+
+    static void reset(String schema) {
+        drop(schema);
+        execute("CREATE SCHEMA \"" + schema + "\"");
+    }
+
+    static void drop(String schema) {
+        if (!SCHEMAS.contains(schema)) throw new IllegalArgumentException("Not an owned disposable schema");
+        execute("DROP SCHEMA IF EXISTS \"" + schema + "\" CASCADE");
+    }
+
+    static void execute(String sql) {
+        try (var connection = connect(); var statement = connection.createStatement()) {
+            statement.execute(sql);
+        } catch (SQLException error) {
+            throw new IllegalStateException("Disposable PostgreSQL fixture operation failed", error);
+        }
+    }
+
+    static FluentConfiguration migration(String schema) {
+        if (!SCHEMAS.contains(schema)) throw new IllegalArgumentException("Not an owned disposable schema");
+        return Flyway.configure().dataSource(
+                System.getenv().getOrDefault("DB_URL", "jdbc:postgresql://127.0.0.1:15432/monitor"),
+                System.getenv().getOrDefault("DB_USER", "wse_industry"), System.getenv().getOrDefault("DB_PASSWORD", ""))
+                .schemas(schema).defaultSchema(schema).locations("classpath:db/migration");
+    }
+
+    static void configure(DynamicPropertyRegistry properties, String schema) {
+        reset(schema);
+        properties.add("spring.flyway.schemas", () -> schema);
+        properties.add("spring.flyway.default-schema", () -> schema);
+        properties.add("spring.jpa.properties.hibernate.default_schema", () -> schema);
+    }
+}
diff --git a/compose.yaml b/compose.yaml
new file mode 100644
index 0000000..31a396d
--- /dev/null
+++ b/compose.yaml
@@ -0,0 +1,26 @@
+name: wse-industry
+services:
+  postgres:
+    image: postgres:17.11-bookworm@sha256:051f7b7b3abdd564d5d1bd1e8c4b9c1b6e77087d1dd22020ede611c096a272e0
+    # Isolated, disposable local development/test database. Never use trust in production.
+    environment:
+      POSTGRES_DB: monitor
+      POSTGRES_USER: wse_industry
+      POSTGRES_HOST_AUTH_METHOD: trust
+    ports:
+      - "127.0.0.1:15432:5432"
+    volumes:
+      - postgres-data:/var/lib/postgresql/data
+    networks:
+      - database
+    healthcheck:
+      test: ["CMD", "pg_isready", "-U", "wse_industry", "-d", "monitor"]
+      interval: 1s
+      timeout: 3s
+      retries: 30
+volumes:
+  postgres-data:
+networks:
+  database:
+    # Dedicated bridge; Docker Desktop suppresses published ports on internal-only networks.
+    driver: bridge
diff --git a/evidence/E03/ownership-fixture.md b/evidence/E03/ownership-fixture.md
new file mode 100644
index 0000000..20beb8b
--- /dev/null
+++ b/evidence/E03/ownership-fixture.md
@@ -0,0 +1,17 @@
+# E03 process-ownership guard fixture
+
+Frozen after the root's harness audit and before the first execution of this
+supplement; all original product fixtures remain unchanged.
+
+`scripts/persistence-isolation.mjs` starts one owned empty-API trap on
+`127.0.0.1:4322`. Any request would return `{ "data": [] }` and increment a counter.
+It invokes the normal persistence harness with the `fixed` evidence label. The
+harness must exit 1 at its occupied-port preflight, having spawned no fixture/API
+child and sent zero HTTP requests. The trap counter must stay zero. The wrapper
+then closes the trap and preserves `occupied-port-refusal.json` before the real
+fixed scenario runs.
+
+This is a harness isolation regression, not another A,A,B baseline or persistence
+scenario: no Monitor creation, Check, restart, or product request is executed.
+The normal fixed scenario still runs once with its original two Monitors and
+three checks, now recording distinct owned API PIDs/readiness/awaited exits.
diff --git a/evidence/E03/supplementary-fixtures.md b/evidence/E03/supplementary-fixtures.md
new file mode 100644
index 0000000..c5cba99
--- /dev/null
+++ b/evidence/E03/supplementary-fixtures.md
@@ -0,0 +1,27 @@
+# E03 supplementary fixtures, frozen before execution
+
+This file only adds new coverage to `fixtures.md`; it changes no original input,
+request sequence, expected value, or threshold.
+
+Observed file timestamp for the initial two-line addition: 2026-08-28T00:08:29Z.
+Moved to this supplement at 2026-08-28T00:10:44Z, before any E03 PostgreSQL test or
+repaired scenario invocation. Original `fixtures.md` SHA-256 restored to
+`b1da3309f875b4e9ea7197ec3920273f13ea3ec10338f3db74981c15430fdab2`.
+
+Before any PostgreSQL integration test or repaired API scenario had run, the
+mapping case was extended: after reading and comparing the original canonical
+Monitor and both CheckRuns, replace that Monitor with name `Mapping updated`, URL
+`http://127.0.0.1:4321/ok`, interval 60, enabled true. Compare the replacement with
+an authoritative read. Delete it and assert zero Monitor and CheckRun rows. The
+original mapping fixtures and assertions all execute before this extra mutation.
+
+The two extension lines were initially appended to `fixtures.md` while writing
+`PostgresPersistenceTest.java`, before its first compilation or execution. At the
+root's request they were moved here, restoring the frozen original file. The only
+scenario executed before this supplement was the unchanged A,A,B baseline; image
+resolution and `db:up` were setup operations, not repaired scenario executions.
+
+Schema-failure tests instantiate a servlet application on an OS-assigned loopback
+port and assert that no WebServerInitializedEvent fires. Successful mapping
+contexts use no web server. This avoids overlap with the existing fixed functional
+ports without changing the frozen incompatible-schema inputs or assertion.
diff --git a/package.json b/package.json
index 99ff394..9d5673e 100644
--- a/package.json
+++ b/package.json
@@ -9,6 +9,9 @@
     "build": "next build --webpack",
     "start": "next start --hostname 127.0.0.1 --port 4323",
     "fixture": "node scripts/fixture.mjs",
+    "db:up": "node scripts/database.mjs up",
+    "db:down": "node scripts/database.mjs down",
+    "db:destroy": "node scripts/database.mjs destroy",
     "api:dev": "mvn -B -ntp -f backend/pom.xml spring-boot:run",
     "api:package": "mvn -B -ntp -f backend/pom.xml -DskipTests package",
     "test:api": "mvn -B -ntp -f backend/pom.xml test",
diff --git a/playwright.config.ts b/playwright.config.ts
index befb9d7..d218fb9 100644
--- a/playwright.config.ts
+++ b/playwright.config.ts
@@ -10,7 +10,7 @@ export default defineConfig({
   use: { ...devices['Desktop Chrome'], baseURL: 'http://127.0.0.1:4323', trace: 'retain-on-failure' },
   webServer: [
     { command: 'node scripts/fixture.mjs', url: 'http://127.0.0.1:4321/ok', reuseExistingServer: false },
-    { command: 'java -jar backend/target/monitor-api-0.0.1.jar', url: 'http://127.0.0.1:4322/api/monitors', reuseExistingServer: false },
+    { command: 'node scripts/test-api.mjs', url: 'http://127.0.0.1:4322/api/monitors', reuseExistingServer: false },
     { command: 'npm run dev', url: 'http://127.0.0.1:4323/monitors', reuseExistingServer: false, timeout: 90_000 },
   ],
 });
diff --git a/scripts/database.mjs b/scripts/database.mjs
new file mode 100644
index 0000000..0f650dc
--- /dev/null
+++ b/scripts/database.mjs
@@ -0,0 +1,23 @@
+import assert from 'node:assert/strict';
+import { spawnSync } from 'node:child_process';
+
+const compose = ['compose', '--project-name', 'wse-industry', '--file', 'compose.yaml'];
+const schemas = new Set(['e03_restart', 'e03_functional', 'e03_browser', 'e03_migrations',
+  'e03_mapping', 'e03_missing_column', 'e03_extra_required']);
+const [action, schema] = process.argv.slice(2);
+let args;
+if (action === 'up') args = [...compose, 'up', '--detach', '--wait', '--wait-timeout', '30', 'postgres'];
+else if (action === 'down') args = [...compose, 'down'];
+else if (action === 'destroy') args = [...compose, 'down', '--volumes'];
+else {
+  assert.ok(action === 'reset' || action === 'drop', 'Use up, down, destroy, reset <test-schema>, or drop <test-schema>');
+  assert.ok(schemas.has(schema), 'Only the frozen disposable test schemas may be reset or dropped');
+  const sql = `DROP SCHEMA IF EXISTS "${schema}" CASCADE;`
+    + (action === 'reset' ? ` CREATE SCHEMA "${schema}";` : '');
+  args = [...compose, 'exec', '--no-TTY', 'postgres', 'psql', '--username', 'wse_industry',
+    '--dbname', 'monitor', '--set', 'ON_ERROR_STOP=1', '--command', sql];
+}
+// No credentials in arguments or logs: the compose service uses explicitly isolated local test trust.
+const result = spawnSync('docker', args, { stdio: 'inherit' });
+if (result.error) console.error(result.error.message);
+process.exitCode = result.status ?? 1;
diff --git a/scripts/persistence-isolation.mjs b/scripts/persistence-isolation.mjs
new file mode 100644
index 0000000..7d420d1
--- /dev/null
+++ b/scripts/persistence-isolation.mjs
@@ -0,0 +1,29 @@
+import assert from 'node:assert/strict';
+import { spawn } from 'node:child_process';
+import { once } from 'node:events';
+import { copyFileSync, readFileSync } from 'node:fs';
+import { createServer } from 'node:http';
+
+// Fixed guard regression: an existing empty API must receive zero requests/mutations.
+let requests = 0;
+const existing = createServer((_request, response) => {
+  requests++;
+  response.setHeader('Content-Type', 'application/json');
+  response.end('{"data":[]}');
+});
+existing.listen(4322, '127.0.0.1');
+await once(existing, 'listening');
+try {
+  const scenario = spawn(process.execPath, ['scripts/persistence-scenario.mjs', 'fixed'], { stdio: 'inherit' });
+  const [code] = await once(scenario, 'exit');
+  assert.equal(code, 1);
+  const evidence = JSON.parse(readFileSync('output/e03/fixed-persistence.json', 'utf8'));
+  assert.match(evidence.result, /occupied or unavailable loopback port: 4322/);
+  assert.equal(evidence.requests.length, 0);
+  assert.equal(evidence.processes.length, 0);
+  assert.equal(requests, 0);
+  copyFileSync('output/e03/fixed-persistence.json', 'output/e03/occupied-port-refusal.json');
+  console.log('PASS: existing API refused before any child process or HTTP request');
+} finally {
+  await new Promise((resolve, reject) => existing.close(error => error ? reject(error) : resolve()));
+}
diff --git a/scripts/persistence-scenario.mjs b/scripts/persistence-scenario.mjs
index 97ca700..aaee967 100644
--- a/scripts/persistence-scenario.mjs
+++ b/scripts/persistence-scenario.mjs
@@ -1,7 +1,8 @@
 import assert from 'node:assert/strict';
 import { spawn } from 'node:child_process';
 import { once } from 'node:events';
-import { appendFileSync, mkdirSync, openSync, closeSync, writeFileSync } from 'node:fs';
+import { appendFileSync, mkdirSync, openSync, closeSync, readFileSync, writeFileSync } from 'node:fs';
+import { createServer } from 'node:net';
 import { setTimeout as delay } from 'node:timers/promises';
 
 // The label changes evidence filenames only. The baseline and repair use identical inputs.
@@ -10,47 +11,86 @@ assert.ok(['baseline', 'fixed'].includes(label), 'Use baseline or fixed as the e
 const directory = 'output/e03';
 mkdirSync(directory, { recursive: true });
 const started = Date.now();
-const evidence = { label, requests: [], monitors: [], checks: [] };
+const evidence = { label, requests: [], monitors: [], checks: [], processes: [], portChecks: [] };
 const processes = [];
 const base = 'http://127.0.0.1:4322';
 
 function start(command, args, logName) {
-  const log = openSync(`${directory}/${label}-${logName}.log`, 'w');
+  const logPath = `${directory}/${label}-${logName}.log`;
+  const log = openSync(logPath, 'w');
   const child = spawn(command, args, {
     env: { ...process.env, DB_SCHEMA: 'e03_restart' }, stdio: ['ignore', log, log],
   });
   closeSync(log);
+  child.evidence = { role: logName, pid: child.pid, startedAt: new Date().toISOString(), logPath };
+  evidence.processes.push(child.evidence);
+  child.once('error', error => { child.evidence.spawnError = error.message; });
+  child.once('exit', (code, signal) => Object.assign(child.evidence, {
+    exitCode: code, signal, exitedAt: new Date().toISOString(),
+  }));
   processes.push(child);
   return child;
 }
 
-async function ready(url, child) {
+async function requireFreePort(port) {
+  const probe = createServer();
+  await new Promise((resolve, reject) => {
+    probe.once('error', () => reject(new Error(`Scenario refuses an occupied or unavailable loopback port: ${port}`)));
+    probe.listen({ host: '127.0.0.1', port, exclusive: true }, () => probe.close(error => error ? reject(error) : resolve()));
+  });
+  evidence.portChecks.push({ port, checkedAt: new Date().toISOString(), result: 'free' });
+}
+
+function requireAlive(child) {
+  assert.ok(child.pid && !child.evidence.spawnError && child.exitCode === null && child.signalCode === null,
+    `Owned ${child.evidence.role} process is not alive`);
+}
+
+async function ready(url, child, startupMarker) {
   const deadline = Date.now() + 30_000;
   while (Date.now() < deadline) {
-    if (child.exitCode !== null) throw new Error(`Owned process exited ${child.exitCode} before ready`);
-    try {
-      if ((await fetch(url, { signal: AbortSignal.timeout(1000) })).ok) return;
-    } catch { /* Bounded readiness polling, never a scenario retry. */ }
+    requireAlive(child);
+    // An arbitrary HTTP 200 cannot prove that our child bound the port. Require its own startup log too.
+    if (readFileSync(child.evidence.logPath, 'utf8').includes(startupMarker)) {
+      let response;
+      try { response = await fetch(url, { signal: AbortSignal.timeout(1000) }); }
+      catch { /* Bounded readiness polling, never a scenario retry. */ }
+      requireAlive(child);
+      if (response?.ok) {
+        child.evidence.readyAt = new Date().toISOString();
+        child.evidence.ownedStartupMarkerObserved = true;
+        return;
+      }
+    }
     await delay(100);
   }
   throw new Error(`Owned process did not become ready within 30 seconds: ${url}`);
 }
 
 async function stop(child) {
-  if (!child || child.exitCode !== null || child.signalCode !== null) return;
+  if (!child || child.evidence.spawnError) return;
+  if (child.exitCode !== null || child.signalCode !== null) {
+    child.evidence.exitObserved = true;
+    return;
+  }
   const exited = once(child, 'exit');
   child.kill('SIGTERM');
   const force = setTimeout(() => child.kill('SIGKILL'), 5000);
-  try { await exited; } finally { clearTimeout(force); }
+  try {
+    await exited;
+    child.evidence.exitAwaited = true;
+  } finally { clearTimeout(force); }
 }
 
 async function request(path, method = 'GET', body, status = 200) {
+  requireAlive(api);
   const requestStarted = Date.now();
   const response = await fetch(`${base}${path}`, {
     method, headers: body === undefined ? {} : { 'Content-Type': 'application/json' },
     body: body === undefined ? undefined : JSON.stringify(body), signal: AbortSignal.timeout(10_000),
   });
   const wire = await response.json();
+  requireAlive(api);
   evidence.requests.push({ method, path, status: response.status, wire,
     elapsedSeconds: (Date.now() - requestStarted) / 1000 });
   assert.equal(response.status, status);
@@ -63,10 +103,14 @@ async function request(path, method = 'GET', body, status = 200) {
 let api;
 let exitCode = 0;
 try {
+  // Refuse existing listeners before starting any child or sending any request.
+  await requireFreePort(4321);
+  await requireFreePort(4322);
   const fixture = start(process.execPath, ['scripts/fixture.mjs'], 'fixture');
-  await ready('http://127.0.0.1:4321/ok', fixture);
+  await ready('http://127.0.0.1:4321/ok', fixture, 'Fixture http://127.0.0.1:4321');
   api = start('java', ['-jar', 'backend/target/monitor-api-0.0.1.jar'], 'api-first');
-  await ready(`${base}/api/monitors`, api);
+  await ready(`${base}/api/monitors`, api, 'Started MonitorApplication');
+  const firstApiPid = api.pid;
   assert.deepEqual(await request('/api/monitors'), [], 'Scenario requires an isolated empty store');
   const a = (await request('/api/monitors', 'POST', {
     name: 'Persisted A', url: 'http://127.0.0.1:4321/ok', interval: 60, enabled: true,
@@ -83,8 +127,11 @@ try {
     evidence.checks.push(check);
   }
   await stop(api);
+  assert.ok(api.evidence.exitAwaited, 'First API exit must be awaited before starting the fresh process');
+  await requireFreePort(4322);
   api = start('java', ['-jar', 'backend/target/monitor-api-0.0.1.jar'], 'api-fresh');
-  await ready(`${base}/api/monitors`, api);
+  assert.notEqual(api.pid, firstApiPid, 'Persistence verification must use a distinct fresh API process');
+  await ready(`${base}/api/monitors`, api, 'Started MonitorApplication');
   const restored = await request('/api/monitors');
   assert.equal(restored.length, 2, 'Fresh API must retain both Monitors');
   for (const monitor of [a, b]) {
diff --git a/scripts/test-api.mjs b/scripts/test-api.mjs
new file mode 100644
index 0000000..a85612d
--- /dev/null
+++ b/scripts/test-api.mjs
@@ -0,0 +1,10 @@
+import { spawn, spawnSync } from 'node:child_process';
+
+const reset = spawnSync(process.execPath, ['scripts/database.mjs', 'reset', 'e03_browser'], { stdio: 'inherit' });
+if (reset.status !== 0) process.exit(reset.status ?? 1);
+const api = spawn('java', ['-jar', 'backend/target/monitor-api-0.0.1.jar'], {
+  env: { ...process.env, DB_SCHEMA: 'e03_browser' }, stdio: 'inherit',
+});
+for (const signal of ['SIGINT', 'SIGTERM']) process.on(signal, () => api.kill(signal));
+api.once('error', error => { console.error(error.message); process.exitCode = 1; });
+api.once('exit', code => { process.exitCode = code ?? 0; });
diff --git a/scripts/verify.mjs b/scripts/verify.mjs
index 534a89d..349e4cf 100644
--- a/scripts/verify.mjs
+++ b/scripts/verify.mjs
@@ -4,16 +4,33 @@ import { appendFileSync, mkdirSync } from 'node:fs';
 // No retries and no performance scenarios. Every invocation is recorded, including failures.
 mkdirSync('output/verification', { recursive: true });
 const commands = [
+  ['node', ['scripts/database.mjs', 'up']],
   ['mvn', ['-B', '-ntp', '-f', 'backend/pom.xml', 'package']],
   ['npm', ['run', 'typecheck']],
   ['npm', ['run', 'build']],
+  ['node', ['scripts/persistence-isolation.mjs']],
+  ['node', ['scripts/database.mjs', 'reset', 'e03_restart']],
+  ['node', ['scripts/persistence-scenario.mjs', 'fixed']],
   ['npm', ['run', 'test:e2e']],
 ];
-for (const [command, args] of commands) {
+function run(command, args) {
   const started = Date.now();
   const result = spawnSync(command, args, { stdio: 'inherit', env: { ...process.env, NEXT_TELEMETRY_DISABLED: '1' } });
   const entry = { command: [command, ...args].join(' '), startedAt: new Date(started).toISOString(),
     elapsedSeconds: (Date.now() - started) / 1000, exitCode: result.status, error: result.error?.message };
   appendFileSync('output/verification/runs.jsonl', `${JSON.stringify(entry)}\n`);
-  if (result.status !== 0) process.exit(result.status ?? 1);
+  return result.status ?? 1;
 }
+let status = 0;
+try {
+  for (const [command, args] of commands) {
+    status = run(command, args);
+    if (status !== 0) break;
+  }
+} finally {
+  for (const schema of ['e03_restart', 'e03_browser']) {
+    const cleanup = run('node', ['scripts/database.mjs', 'drop', schema]);
+    if (status === 0) status = cleanup;
+  }
+}
+process.exitCode = status;


