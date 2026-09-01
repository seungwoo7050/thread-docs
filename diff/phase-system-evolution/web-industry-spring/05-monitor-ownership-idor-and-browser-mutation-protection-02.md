## `feat(authz): scope Monitor persistence to its verified owner`

diff --git a/TRACK.md b/TRACK.md
index 7aca3b4..4fc5b46 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -2,7 +2,7 @@
 
 Spec revision: `0a006589477f8ae47bad3faa5510c999cff85ee4`
 
-E04 is a local development product: Next.js/React renders login, logout, Monitor creation, editing, pause/activation, deletion and Check history. Spring Security authenticates browser sessions; PostgreSQL stores salted account passwords, Monitors and every completed CheckRun. A manual check is synchronous. There is no signup, ownership authorization, scheduler, worker, cache, broker, or production application container.
+E05 is a local development product: Next.js/React renders login, logout, Monitor creation, editing, pause/activation, deletion and Check history. Spring Security authenticates browser sessions; PostgreSQL stores salted account passwords, Monitors and every completed CheckRun. A manual check is synchronous. Monitor and CheckRun access is scoped to the signed-in user. There is no signup, scheduler, worker, cache, broker, or production application container.
 
 ## Pinned toolchain
 
@@ -167,3 +167,55 @@ npm run test:e2e
 - [Spring Security 6.5 session management](https://docs.spring.io/spring-security/reference/6.5/servlet/authentication/session-management.html)
 - [Spring Security 6.5 logout filter ordering](https://docs.spring.io/spring-security/reference/6.5/servlet/authentication/logout.html)
 - [Spring Security 6.5 CSRF defaults](https://docs.spring.io/spring-security/reference/6.5/servlet/exploits/csrf.html)
+
+
+## Monitor ownership and existing-row upgrade (E05)
+
+The server takes the immutable user UUID from the authenticated Spring Security
+principal, never from request JSON. Every Monitor read and every nested/history
+CheckRun query includes that owner. UPDATE and DELETE include both owner and ID in
+SQL; Check creation verifies ownership before outbound HTTP and again before the
+insert. Owner identity is not exposed in the existing Monitor wire model and
+cannot be changed through the API. A foreign ID and an absent ID both return the
+same 404 / NOT_FOUND response. PostgreSQL cascades only the authorized deletion.
+
+V1/V2/V3 remain byte-for-byte unchanged. V4 adds `monitors.owner_user_id` with
+NOT NULL and a foreign key to `users.id`. Empty installations upgrade normally.
+For existing V3 rows, startup refuses atomically unless **every row has an
+explicitly verified assignment to an existing user**. It never picks the first
+user or silently adopts all rows; refusal preserves all Monitors, CheckRuns and
+V3 migration history. On fresh refusal the new column is rolled back too.
+
+For an existing V3 installation, first prepare the intended accounts using the
+E04 bootstrap command, then stop the old application. An operator who can verify
+each Monitor's owner must supply its ID and the intended existing username (psql
+variables below). Do this in the chosen database schema, never by modifying old
+migration files. This example assigns one verified row; repeat that UPDATE for
+each verified mapping in the same transaction. An unknown user or unassigned row
+aborts the transaction. Do not restart E04 after this preparation; start E05 to
+validate and install V4 before accepting API requests.
+
+```sql
+BEGIN;
+LOCK TABLE monitors IN ACCESS EXCLUSIVE MODE;
+ALTER TABLE monitors ADD COLUMN IF NOT EXISTS owner_user_id uuid;
+UPDATE monitors m SET owner_user_id = u.id
+FROM users u
+WHERE m.id = :'verified_monitor_id'::uuid
+  AND u.username = :'verified_owner_username'
+  AND m.owner_user_id IS NULL;
+-- Repeat the explicit, verified per-Monitor mapping above as needed.
+DO $$
+BEGIN
+  IF EXISTS (SELECT 1 FROM monitors m LEFT JOIN users u ON u.id = m.owner_user_id
+             WHERE m.owner_user_id IS NULL OR u.id IS NULL) THEN
+    RAISE EXCEPTION 'Every existing Monitor requires an explicit verified owner';
+  END IF;
+END $$;
+COMMIT;
+```
+
+If ownership cannot be established, keep startup stopped and preserve the data;
+do not invent an owner or delete rows to make migration pass. Startup also checks
+owner nullability and the users foreign key, because Hibernate's column-type
+validation alone does not establish those guarantees.
diff --git a/backend/src/main/java/dev/evolution/monitor/MonitorController.java b/backend/src/main/java/dev/evolution/monitor/MonitorController.java
index e066e10..2eaad18 100644
--- a/backend/src/main/java/dev/evolution/monitor/MonitorController.java
+++ b/backend/src/main/java/dev/evolution/monitor/MonitorController.java
@@ -5,6 +5,7 @@ import java.net.URI;
 import java.util.List;
 import java.util.UUID;
 import org.springframework.http.HttpStatus;
+import org.springframework.security.core.annotation.AuthenticationPrincipal;
 import org.springframework.web.bind.annotation.DeleteMapping;
 import org.springframework.web.bind.annotation.GetMapping;
 import org.springframework.web.bind.annotation.PathVariable;
@@ -28,52 +29,54 @@ public class MonitorController {
     }
 
     @GetMapping
-    public ApiData<List<MonitorView>> list() {
-        return new ApiData<>(store.list());
+    public ApiData<List<MonitorView>> list(@AuthenticationPrincipal UserAccounts.AccountUser user) {
+        return new ApiData<>(store.list(user.userId()));
     }
 
     @PostMapping
     @ResponseStatus(HttpStatus.CREATED)
-    public ApiData<MonitorView> create(@RequestBody JsonNode body) {
+    public ApiData<MonitorView> create(@AuthenticationPrincipal UserAccounts.AccountUser user, @RequestBody JsonNode body) {
         CreateMonitor input = CreateMonitor.fromJson(body);
         checks.requireFixtureUrl(input.url());
-        return new ApiData<>(store.create(input));
+        return new ApiData<>(store.create(user.userId(), input));
     }
 
     @GetMapping("/{id}")
-    public ApiData<MonitorView> get(@PathVariable UUID id) {
-        return new ApiData<>(store.get(id));
+    public ApiData<MonitorView> get(@AuthenticationPrincipal UserAccounts.AccountUser user, @PathVariable UUID id) {
+        return new ApiData<>(store.get(user.userId(), id));
     }
 
     @PutMapping("/{id}")
-    public ApiData<MonitorView> replace(@PathVariable UUID id, @RequestBody JsonNode body) {
+    public ApiData<MonitorView> replace(@AuthenticationPrincipal UserAccounts.AccountUser user,
+            @PathVariable UUID id, @RequestBody JsonNode body) {
         CreateMonitor input = CreateMonitor.fromJson(body);
         checks.requireFixtureUrl(input.url());
-        return new ApiData<>(store.replace(id, input));
+        return new ApiData<>(store.replace(user.userId(), id, input));
     }
 
     @DeleteMapping("/{id}")
-    public ApiData<Void> delete(@PathVariable UUID id) {
-        store.delete(id);
+    public ApiData<Void> delete(@AuthenticationPrincipal UserAccounts.AccountUser user, @PathVariable UUID id) {
+        store.delete(user.userId(), id);
         return new ApiData<>(null);
     }
 
     @PostMapping("/{id}/checks")
-    public ApiData<CheckRunner.CheckRun> check(@PathVariable UUID id) {
+    public ApiData<CheckRunner.CheckRun> check(@AuthenticationPrincipal UserAccounts.AccountUser user, @PathVariable UUID id) {
         // Each store call crosses the Spring transaction proxy. Outbound I/O holds no transaction.
-        Monitor monitor = store.monitor(id);
+        Monitor monitor = store.monitor(user.userId(), id);
         CheckRunner.CheckRun result = checks.run(monitor.id(), monitor.url());
-        return new ApiData<>(store.saveCheck(result));
+        return new ApiData<>(store.saveCheck(user.userId(), result));
     }
 
     @GetMapping("/{id}/checks")
-    public ApiData<List<CheckRunner.CheckRun>> history(@PathVariable UUID id) {
-        return new ApiData<>(store.history(id));
+    public ApiData<List<CheckRunner.CheckRun>> history(@AuthenticationPrincipal UserAccounts.AccountUser user, @PathVariable UUID id) {
+        return new ApiData<>(store.history(user.userId(), id));
     }
 
     @GetMapping("/{id}/checks/{checkId}")
-    public ApiData<CheckRunner.CheckRun> check(@PathVariable UUID id, @PathVariable UUID checkId) {
-        return new ApiData<>(store.check(id, checkId));
+    public ApiData<CheckRunner.CheckRun> check(@AuthenticationPrincipal UserAccounts.AccountUser user,
+            @PathVariable UUID id, @PathVariable UUID checkId) {
+        return new ApiData<>(store.check(user.userId(), id, checkId));
     }
 
     public record CreateMonitor(String name, String url, int interval, boolean enabled) {
diff --git a/backend/src/main/java/dev/evolution/monitor/MonitorEntity.java b/backend/src/main/java/dev/evolution/monitor/MonitorEntity.java
index d8afb35..2afb285 100644
--- a/backend/src/main/java/dev/evolution/monitor/MonitorEntity.java
+++ b/backend/src/main/java/dev/evolution/monitor/MonitorEntity.java
@@ -12,6 +12,7 @@ import java.util.UUID;
 @Table(name = "monitors")
 public class MonitorEntity {
     @Id @Column(name = "id", nullable = false) private UUID id;
+    @Column(name = "owner_user_id", nullable = false, updatable = false) private UUID ownerUserId;
     @Column(name = "name", nullable = false, length = 100) private String name;
     @Column(name = "url", nullable = false, columnDefinition = "text") private String url;
     @Column(name = "interval_seconds", nullable = false) private int intervalSeconds;
@@ -23,9 +24,10 @@ public class MonitorEntity {
 
     protected MonitorEntity() {}
 
-    static MonitorEntity fromDomain(Monitor monitor) {
+    static MonitorEntity fromDomain(Monitor monitor, UUID ownerUserId) {
         var row = new MonitorEntity();
         row.id = monitor.id();
+        row.ownerUserId = java.util.Objects.requireNonNull(ownerUserId);
         row.name = monitor.name();
         row.url = monitor.url();
         row.intervalSeconds = monitor.interval();
@@ -36,13 +38,7 @@ public class MonitorEntity {
         return row;
     }
 
-    void replace(MonitorController.CreateMonitor input) {
-        name = input.name();
-        url = input.url();
-        intervalSeconds = input.interval();
-        enabled = input.enabled();
-        updatedAt = Instant.now().truncatedTo(ChronoUnit.MICROS);
-    }
+    UUID ownerUserId() { return ownerUserId; }
 
     Monitor toDomain() {
         return new Monitor(id, name, url, intervalSeconds, enabled, createdAt, updatedAt);
diff --git a/backend/src/main/java/dev/evolution/monitor/MonitorStore.java b/backend/src/main/java/dev/evolution/monitor/MonitorStore.java
index 5b52e5d..f0c274e 100644
--- a/backend/src/main/java/dev/evolution/monitor/MonitorStore.java
+++ b/backend/src/main/java/dev/evolution/monitor/MonitorStore.java
@@ -3,6 +3,7 @@ package dev.evolution.monitor;
 import jakarta.persistence.EntityManager;
 import jakarta.persistence.PersistenceContext;
 import java.time.Instant;
+import java.time.temporal.ChronoUnit;
 import java.util.List;
 import java.util.UUID;
 import org.springframework.http.HttpStatus;
@@ -15,73 +16,89 @@ import org.springframework.web.server.ResponseStatusException;
 public class MonitorStore {
     @PersistenceContext private EntityManager entities;
 
-    public List<MonitorController.MonitorView> list() {
-        return entities.createQuery("select m from MonitorEntity m order by m.createdAt, m.id", MonitorEntity.class)
-                .getResultList().stream().map(this::view).toList();
+    public List<MonitorController.MonitorView> list(UUID owner) {
+        return entities.createQuery("select m from MonitorEntity m where m.ownerUserId = :owner "
+                        + "order by m.createdAt, m.id", MonitorEntity.class)
+                .setParameter("owner", owner).getResultList().stream().map(this::view).toList();
     }
 
-    public MonitorController.MonitorView get(UUID id) {
-        return view(requireMonitor(id));
+    public MonitorController.MonitorView get(UUID owner, UUID id) {
+        return view(requireMonitor(owner, id));
     }
 
-    public Monitor monitor(UUID id) {
-        return requireMonitor(id).toDomain();
+    public Monitor monitor(UUID owner, UUID id) {
+        return requireMonitor(owner, id).toDomain();
     }
 
     @Transactional
-    public MonitorController.MonitorView create(MonitorController.CreateMonitor input) {
+    public MonitorController.MonitorView create(UUID owner, MonitorController.CreateMonitor input) {
         Instant now = Instant.now();
         var row = MonitorEntity.fromDomain(new Monitor(UUID.randomUUID(), input.name(), input.url(),
-                input.interval(), input.enabled(), now, now));
+                input.interval(), input.enabled(), now, now), owner);
         entities.persist(row);
         return new MonitorController.MonitorView(row.toDomain(), null);
     }
 
     @Transactional
-    public MonitorController.MonitorView replace(UUID id, MonitorController.CreateMonitor input) {
-        var row = requireMonitor(id);
-        row.replace(input); // Managed entity dirty checking is flushed before the proxy returns.
-        return view(row);
+    public MonitorController.MonitorView replace(UUID owner, UUID id, MonitorController.CreateMonitor input) {
+        // The UPDATE itself is scoped, not just a preceding existence check.
+        int changed = entities.createQuery("update MonitorEntity m set m.name = :name, m.url = :url, "
+                        + "m.intervalSeconds = :interval, m.enabled = :enabled, m.updatedAt = :updated "
+                        + "where m.id = :id and m.ownerUserId = :owner")
+                .setParameter("name", input.name()).setParameter("url", input.url())
+                .setParameter("interval", input.interval()).setParameter("enabled", input.enabled())
+                .setParameter("updated", Instant.now().truncatedTo(ChronoUnit.MICROS))
+                .setParameter("id", id).setParameter("owner", owner).executeUpdate();
+        if (changed != 1) throw notFound();
+        entities.clear(); // Bulk JPQL bypasses managed snapshots; read the row afresh in this transaction.
+        return view(requireMonitor(owner, id));
     }
 
     @Transactional
-    public void delete(UUID id) {
-        entities.remove(requireMonitor(id)); // PostgreSQL ON DELETE CASCADE removes every historical child.
+    public void delete(UUID owner, UUID id) {
+        int changed = entities.createQuery("delete from MonitorEntity m where m.id = :id and m.ownerUserId = :owner")
+                .setParameter("id", id).setParameter("owner", owner).executeUpdate();
+        if (changed != 1) throw notFound();
+        entities.clear(); // PostgreSQL cascades only this authorized parent's history.
     }
 
     @Transactional
-    public CheckRunner.CheckRun saveCheck(CheckRunner.CheckRun check) {
-        requireMonitor(check.monitorId());
+    public CheckRunner.CheckRun saveCheck(UUID owner, CheckRunner.CheckRun check) {
+        requireMonitor(owner, check.monitorId());
         var row = CheckRunEntity.fromDomain(check);
         entities.persist(row);
         return row.toDomain();
     }
 
-    public List<CheckRunner.CheckRun> history(UUID id) {
-        requireMonitor(id);
-        return entities.createQuery("select c from CheckRunEntity c where c.monitorId = :id "
+    public List<CheckRunner.CheckRun> history(UUID owner, UUID id) {
+        requireMonitor(owner, id);
+        return entities.createQuery("select c from CheckRunEntity c, MonitorEntity m "
+                        + "where c.monitorId = m.id and m.id = :id and m.ownerUserId = :owner "
                         + "order by c.finishedAt desc, c.id desc", CheckRunEntity.class)
-                .setParameter("id", id).getResultList().stream().map(CheckRunEntity::toDomain).toList();
+                .setParameter("id", id).setParameter("owner", owner).getResultList().stream()
+                .map(CheckRunEntity::toDomain).toList();
     }
 
-    public CheckRunner.CheckRun check(UUID monitorId, UUID checkId) {
-        requireMonitor(monitorId);
-        return entities.createQuery("select c from CheckRunEntity c where c.id = :checkId and c.monitorId = :monitorId",
+    public CheckRunner.CheckRun check(UUID owner, UUID monitorId, UUID checkId) {
+        return entities.createQuery("select c from CheckRunEntity c, MonitorEntity m where c.id = :checkId "
+                        + "and c.monitorId = :monitorId and c.monitorId = m.id and m.ownerUserId = :owner",
                         CheckRunEntity.class).setParameter("checkId", checkId).setParameter("monitorId", monitorId)
+                .setParameter("owner", owner)
                 .getResultStream().findFirst().map(CheckRunEntity::toDomain).orElseThrow(MonitorStore::notFound);
     }
 
-    private MonitorEntity requireMonitor(UUID id) {
-        var row = entities.find(MonitorEntity.class, id);
-        if (row == null) throw notFound();
-        return row;
+    private MonitorEntity requireMonitor(UUID owner, UUID id) {
+        return entities.createQuery("select m from MonitorEntity m where m.id = :id and m.ownerUserId = :owner",
+                        MonitorEntity.class).setParameter("id", id).setParameter("owner", owner)
+                .getResultStream().findFirst().orElseThrow(MonitorStore::notFound);
     }
 
     private MonitorController.MonitorView view(MonitorEntity row) {
         Monitor monitor = row.toDomain();
-        var latest = entities.createQuery("select c from CheckRunEntity c where c.monitorId = :id "
+        var latest = entities.createQuery("select c from CheckRunEntity c, MonitorEntity m "
+                        + "where c.monitorId = m.id and m.id = :id and m.ownerUserId = :owner "
                         + "order by c.finishedAt desc, c.id desc", CheckRunEntity.class)
-                .setParameter("id", monitor.id()).setMaxResults(1).getResultStream()
+                .setParameter("id", monitor.id()).setParameter("owner", row.ownerUserId()).setMaxResults(1).getResultStream()
                 .findFirst().map(CheckRunEntity::toDomain).orElse(null);
         return new MonitorController.MonitorView(monitor, latest);
     }
diff --git a/backend/src/main/java/dev/evolution/monitor/SchemaCompatibility.java b/backend/src/main/java/dev/evolution/monitor/SchemaCompatibility.java
index 8f112f3..c69438b 100644
--- a/backend/src/main/java/dev/evolution/monitor/SchemaCompatibility.java
+++ b/backend/src/main/java/dev/evolution/monitor/SchemaCompatibility.java
@@ -47,6 +47,26 @@ public class SchemaCompatibility implements InitializingBean {
                     }
                 }
             }
+            // Hibernate checks column types, but does not verify this authorization constraint.
+            try (var owner = connection.prepareStatement("SELECT is_nullable FROM information_schema.columns "
+                    + "WHERE table_schema = ? AND table_name = 'monitors' AND column_name = 'owner_user_id'")) {
+                owner.setString(1, schema);
+                try (var rows = owner.executeQuery()) {
+                    if (!rows.next() || !"NO".equals(rows.getString(1))) {
+                        throw new IllegalStateException("Monitor ownership must be required.");
+                    }
+                }
+            }
+            boolean ownerForeignKey = false;
+            try (var keys = connection.getMetaData().getImportedKeys(connection.getCatalog(), schema, "monitors")) {
+                while (keys.next()) {
+                    if ("owner_user_id".equals(keys.getString("FKCOLUMN_NAME"))
+                            && schema.equals(keys.getString("PKTABLE_SCHEM"))
+                            && "users".equals(keys.getString("PKTABLE_NAME"))
+                            && "id".equals(keys.getString("PKCOLUMN_NAME"))) ownerForeignKey = true;
+                }
+            }
+            if (!ownerForeignKey) throw new IllegalStateException("Monitor ownership must reference users.");
         }
     }
 
diff --git a/backend/src/main/java/dev/evolution/monitor/UserAccounts.java b/backend/src/main/java/dev/evolution/monitor/UserAccounts.java
index 957611b..5da2459 100644
--- a/backend/src/main/java/dev/evolution/monitor/UserAccounts.java
+++ b/backend/src/main/java/dev/evolution/monitor/UserAccounts.java
@@ -4,6 +4,7 @@ import jakarta.persistence.EntityManager;
 import jakarta.persistence.PersistenceContext;
 import java.nio.charset.StandardCharsets;
 import java.util.List;
+import java.util.UUID;
 import org.springframework.security.core.userdetails.User;
 import org.springframework.security.core.userdetails.UserDetails;
 import org.springframework.security.core.userdetails.UserDetailsService;
@@ -24,7 +25,7 @@ public class UserAccounts implements UserDetailsService {
     public UserDetails loadUserByUsername(String username) {
         var row = find(username).stream().findFirst()
                 .orElseThrow(() -> new UsernameNotFoundException("Authentication required."));
-        return new User(row.username(), row.passwordHash(), List.of());
+        return new AccountUser(row.id(), row.username(), row.passwordHash());
     }
 
     @Transactional
@@ -55,4 +56,16 @@ public class UserAccounts implements UserDetailsService {
             throw new IllegalArgumentException("Supply each fixture password through runtime input (24–72 UTF-8 bytes).");
         }
     }
+
+    // User retains Spring's credential erasure; only the immutable account ID is added to the session.
+    public static final class AccountUser extends User {
+        private final UUID userId;
+
+        AccountUser(UUID userId, String username, String passwordHash) {
+            super(username, passwordHash, List.of());
+            this.userId = userId;
+        }
+
+        public UUID userId() { return userId; }
+    }
 }
diff --git a/backend/src/main/java/dev/evolution/monitor/UserEntity.java b/backend/src/main/java/dev/evolution/monitor/UserEntity.java
index 1665ee3..5db569b 100644
--- a/backend/src/main/java/dev/evolution/monitor/UserEntity.java
+++ b/backend/src/main/java/dev/evolution/monitor/UserEntity.java
@@ -21,6 +21,7 @@ public class UserEntity {
         this.passwordHash = passwordHash;
     }
 
+    UUID id() { return id; }
     String username() { return username; }
     String passwordHash() { return passwordHash; }
     // No entity serialization or generated toString: the hash must not enter API payloads or logs.
diff --git a/backend/src/main/resources/db/migration/V4__require_monitor_ownership.sql b/backend/src/main/resources/db/migration/V4__require_monitor_ownership.sql
new file mode 100644
index 0000000..ea0f817
--- /dev/null
+++ b/backend/src/main/resources/db/migration/V4__require_monitor_ownership.sql
@@ -0,0 +1,18 @@
+-- An empty schema upgrades automatically. Existing rows need an explicit operator
+-- assignment while the application is stopped; no user is chosen by this migration.
+-- PostgreSQL/Flyway runs this entire migration in one transaction, including refusal.
+ALTER TABLE monitors ADD COLUMN IF NOT EXISTS owner_user_id uuid;
+
+DO $$
+BEGIN
+    IF EXISTS (SELECT 1 FROM monitors WHERE owner_user_id IS NULL) THEN
+        RAISE EXCEPTION 'E05 ownership requires explicit verified assignment for every existing Monitor; see TRACK.md';
+    END IF;
+    IF EXISTS (SELECT 1 FROM monitors m LEFT JOIN users u ON u.id = m.owner_user_id WHERE u.id IS NULL) THEN
+        RAISE EXCEPTION 'E05 ownership assignment must reference an existing verified user';
+    END IF;
+END $$;
+
+ALTER TABLE monitors ALTER COLUMN owner_user_id SET NOT NULL;
+ALTER TABLE monitors ADD CONSTRAINT monitors_owner_user_fk
+    FOREIGN KEY (owner_user_id) REFERENCES users(id);
diff --git a/backend/src/test/java/dev/evolution/monitor/ApiErrorBoundaryTest.java b/backend/src/test/java/dev/evolution/monitor/ApiErrorBoundaryTest.java
index 8bfba92..6b29ca3 100644
--- a/backend/src/test/java/dev/evolution/monitor/ApiErrorBoundaryTest.java
+++ b/backend/src/test/java/dev/evolution/monitor/ApiErrorBoundaryTest.java
@@ -8,6 +8,7 @@ import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.
 import com.fasterxml.jackson.databind.ObjectMapper;
 import org.junit.jupiter.api.Test;
 import org.springframework.http.MediaType;
+import org.springframework.security.web.method.annotation.AuthenticationPrincipalArgumentResolver;
 import org.springframework.test.web.servlet.setup.MockMvcBuilders;
 
 class ApiErrorBoundaryTest {
@@ -17,6 +18,7 @@ class ApiErrorBoundaryTest {
         when(checks.requireFixtureUrl("http://127.0.0.1:4321/ok"))
                 .thenThrow(new IllegalStateException("Private implementation detail"));
         var mvc = MockMvcBuilders.standaloneSetup(new MonitorController(checks, mock(MonitorStore.class)))
+                .setCustomArgumentResolvers(new AuthenticationPrincipalArgumentResolver())
                 .setControllerAdvice(new ApiErrors()).build();
         String body = mvc.perform(post("/api/monitors").contentType(MediaType.APPLICATION_JSON).content("""
                 {"name":"Fixture monitor","url":"http://127.0.0.1:4321/ok","interval":60,"enabled":true}
diff --git a/backend/src/test/java/dev/evolution/monitor/OwnershipAuthorizationTest.java b/backend/src/test/java/dev/evolution/monitor/OwnershipAuthorizationTest.java
new file mode 100644
index 0000000..7e85da2
--- /dev/null
+++ b/backend/src/test/java/dev/evolution/monitor/OwnershipAuthorizationTest.java
@@ -0,0 +1,254 @@
+package dev.evolution.monitor;
+
+import static org.junit.jupiter.api.Assertions.*;
+import static org.mockito.ArgumentMatchers.*;
+import static org.mockito.Mockito.doAnswer;
+
+import com.fasterxml.jackson.databind.JsonNode;
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sun.net.httpserver.HttpServer;
+import java.net.InetSocketAddress;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.util.ArrayList;
+import java.util.LinkedHashMap;
+import java.util.List;
+import java.util.Map;
+import java.util.UUID;
+import java.util.concurrent.CopyOnWriteArrayList;
+import java.util.concurrent.atomic.AtomicInteger;
+import org.hibernate.resource.jdbc.spi.StatementInspector;
+import org.junit.jupiter.api.AfterAll;
+import org.junit.jupiter.api.BeforeAll;
+import org.junit.jupiter.api.BeforeEach;
+import org.junit.jupiter.api.Test;
+import org.springframework.aop.support.AopUtils;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.boot.test.web.client.TestRestTemplate;
+import org.springframework.http.HttpMethod;
+import org.springframework.http.ResponseEntity;
+import org.springframework.test.annotation.DirtiesContext;
+import org.springframework.test.context.DynamicPropertyRegistry;
+import org.springframework.test.context.DynamicPropertySource;
+import org.springframework.test.context.bean.override.mockito.MockitoSpyBean;
+import org.springframework.transaction.support.TransactionSynchronizationManager;
+import org.springframework.web.server.ResponseStatusException;
+
+@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
+@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_CLASS)
+class OwnershipAuthorizationTest {
+    private static final String MISSING = "00000000-0000-0000-0000-000000000000";
+    private static final String ALICE_PASSWORD = SessionClient.password();
+    private static final String BOB_PASSWORD = SessionClient.password();
+    private static final AtomicInteger outbound = new AtomicInteger();
+    private static HttpServer fixture;
+    @Autowired TestRestTemplate api;
+    @Autowired ObjectMapper json;
+    @Autowired MonitorStore store;
+    @Autowired UserAccounts accounts;
+    @MockitoSpyBean CheckRunner checks;
+    private SessionClient alice;
+    private SessionClient bob;
+    private String aliceId;
+    private String bobId;
+    private String aliceCheck;
+    private String bobCheck;
+    private final List<Map<String, Object>> evidence = new ArrayList<>();
+
+    @DynamicPropertySource
+    static void database(DynamicPropertyRegistry properties) {
+        TestDatabase.configure(properties, "e05_ownership");
+        properties.add("spring.jpa.properties.hibernate.session_factory.statement_inspector", () -> SqlEvidence.class.getName());
+    }
+
+    @BeforeAll
+    static void fixture(@Autowired UserAccounts accounts) throws Exception {
+        accounts.bootstrap(ALICE_PASSWORD, BOB_PASSWORD);
+        fixture = HttpServer.create(new InetSocketAddress("127.0.0.1", 4321), 0);
+        fixture.createContext("/ok", exchange -> {
+            outbound.incrementAndGet(); exchange.sendResponseHeaders(200, -1); exchange.close();
+        });
+        fixture.createContext("/fail", exchange -> {
+            outbound.incrementAndGet(); exchange.sendResponseHeaders(503, -1); exchange.close();
+        });
+        fixture.start();
+    }
+
+    @AfterAll
+    static void cleanup() {
+        if (fixture != null) fixture.stop(0);
+        TestDatabase.drop("e05_ownership");
+    }
+
+    @BeforeEach
+    void fixedTwoUserDataset() {
+        TestDatabase.execute("DELETE FROM e05_ownership.monitors");
+        outbound.set(0);
+        doAnswer(invocation -> {
+            assertFalse(TransactionSynchronizationManager.isActualTransactionActive(), "Outbound must be outside the store transaction");
+            return invocation.callRealMethod();
+        }).when(checks).run(any(UUID.class), anyString());
+        alice = new SessionClient(api);
+        bob = new SessionClient(api);
+        assertEquals(200, alice.login("alice-e04", ALICE_PASSWORD).getStatusCode().value());
+        assertEquals(200, bob.login("bob-e04", BOB_PASSWORD).getStatusCode().value());
+        SqlEvidence.events.clear();
+        aliceId = data(alice.mutate("/api/monitors", HttpMethod.POST, input("Monitor A", "/ok", 60, true)), 201)
+                .at("/monitor/id").textValue();
+        bobId = data(bob.mutate("/api/monitors", HttpMethod.POST, input("Monitor B", "/fail", 120, true)), 201)
+                .at("/monitor/id").textValue();
+        JsonNode a = data(alice.mutate(path(aliceId) + "/checks", HttpMethod.POST, null), 200);
+        JsonNode b = data(bob.mutate(path(bobId) + "/checks", HttpMethod.POST, null), 200);
+        aliceCheck = a.get("id").textValue();
+        bobCheck = b.get("id").textValue();
+        assertEquals("SUCCEEDED", a.get("state").textValue());
+        assertEquals("FAILED", b.get("state").textValue());
+        assertEquals(2, outbound.get());
+    }
+
+    @Test
+    void ownerBoundaryCoversEveryReadAndMutationWithoutForeignWritesOrOutboundRequests() throws Exception {
+        assertTrue(AopUtils.isAopProxy(store), "Calls must enter the actual Spring transactional proxy");
+        JsonNode notFound = error(bob.get(path(aliceId)), 404, "NOT_FOUND");
+        evidence.add(Map.of("probe", "unchanged-fixed-counterexample-after-fix", "status", 404));
+        for (var actor : List.of(new Actor("alice", alice, aliceId, aliceCheck, bobId, bobCheck),
+                new Actor("bob", bob, bobId, bobCheck, aliceId, aliceCheck))) {
+            JsonNode collection = data(actor.client.get("/api/monitors"), 200);
+            assertEquals(1, collection.size());
+            assertEquals(actor.ownId, collection.at("/0/monitor/id").textValue());
+            assertEquals(actor.ownCheck, data(actor.client.get(path(actor.ownId)), 200).at("/latestCheck/id").textValue());
+            assertEquals(1, data(actor.client.get(path(actor.ownId) + "/checks"), 200).size());
+            assertEquals(actor.ownCheck, data(actor.client.get(path(actor.ownId) + "/checks/" + actor.ownCheck), 200)
+                    .get("id").textValue());
+            for (String denied : List.of(path(actor.foreignId), path(actor.foreignId) + "/checks",
+                    path(actor.foreignId) + "/checks/" + actor.foreignCheck,
+                    path(actor.ownId) + "/checks/" + actor.foreignCheck,
+                    path(actor.ownId) + "/checks/" + MISSING,
+                    path(MISSING), path(MISSING) + "/checks", path(MISSING) + "/checks/" + actor.ownCheck)) {
+                assertEquals(notFound, error(actor.client.get(denied), 404, "NOT_FOUND"));
+            }
+            for (String target : List.of(actor.foreignId, MISSING)) {
+                for (var mutation : mutations(target)) {
+                    String before = snapshot();
+                    int beforeOutbound = outbound.get();
+                    assertEquals(notFound, error(actor.client.mutate(mutation.path, mutation.method, mutation.body), 404, "NOT_FOUND"));
+                    assertEquals(before, snapshot(), "A denied mutation must leave all authoritative rows unchanged");
+                    assertEquals(beforeOutbound, outbound.get(), "A denied check must send no outbound request");
+                    evidence.add(Map.of("actor", actor.name, "target", target.equals(MISSING) ? "nonexistent" : "foreign",
+                            "mutation", mutation.name, "status", 404, "allRowsUnchanged", true, "outboundUnchanged", true));
+                }
+            }
+        }
+        String before = snapshot();
+        UUID bobOwner = ((UserAccounts.AccountUser) accounts.loadUserByUsername("bob-e04")).userId();
+        var forged = json.treeToValue(data(alice.get(path(aliceId) + "/checks/" + aliceCheck), 200), CheckRunner.CheckRun.class);
+        assertEquals(404, assertThrows(ResponseStatusException.class, () -> store.saveCheck(bobOwner, forged))
+                .getStatusCode().value());
+        assertEquals(before, snapshot());
+
+        for (boolean enabled : List.of(true, false, true)) {
+            JsonNode changed = data(alice.mutate(path(aliceId), HttpMethod.PUT, input("Monitor A edited", "/ok", 90, enabled)), 200);
+            assertEquals(enabled, changed.at("/monitor/enabled").booleanValue());
+            assertEquals(changed, data(alice.get(path(aliceId)), 200));
+        }
+        data(alice.mutate(path(aliceId) + "/checks", HttpMethod.POST, null), 200);
+        assertEquals(2, data(alice.get(path(aliceId) + "/checks"), 200).size());
+        Map<String, Object> creation = new LinkedHashMap<>(input("Extra owner creation", "/ok", 60, true));
+        creation.put("ownerUserId", bobOwner.toString());
+        String extra = data(alice.mutate("/api/monitors", HttpMethod.POST, creation), 201).at("/monitor/id").textValue();
+        assertEquals(notFound, error(bob.get(path(extra)), 404, "NOT_FOUND"));
+        assertEquals(2, data(alice.get("/api/monitors"), 200).size());
+        assertTrue(data(alice.mutate(path(extra), HttpMethod.DELETE, null), 200).isNull());
+
+        JsonNode aliceBeforeBobDelete = data(alice.get(path(aliceId)), 200);
+        assertTrue(data(bob.mutate(path(bobId), HttpMethod.DELETE, null), 200).isNull());
+        assertEquals(notFound, error(bob.get(path(bobId)), 404, "NOT_FOUND"));
+        assertEquals(notFound, error(bob.get(path(bobId) + "/checks/" + bobCheck), 404, "NOT_FOUND"));
+        assertEquals(0, data(bob.get("/api/monitors"), 200).size());
+        assertEquals(aliceBeforeBobDelete, data(alice.get(path(aliceId)), 200));
+        assertEquals(3, outbound.get());
+        verifySqlAndProxyEvidence();
+        Files.writeString(Path.of("target/e05-authorization-evidence.json"), json.writerWithDefaultPrettyPrinter()
+                .writeValueAsString(Map.of("dataset", "two users/two Monitors/two initial CheckRuns",
+                        "matrix", evidence, "ownerLifecycle", true, "foreignOwnerInputIgnored", true,
+                        "proxyAndSqlChecked", true, "outboundTransactionAbsent", true)) + "\n");
+    }
+
+    private List<Mutation> mutations(String id) {
+        return List.of(new Mutation("edit", HttpMethod.PUT, path(id), input("Forbidden edit", "/ok", 90, true)),
+                new Mutation("pause", HttpMethod.PUT, path(id), input("Monitor A", "/ok", 60, false)),
+                new Mutation("resume", HttpMethod.PUT, path(id), input("Monitor A", "/ok", 60, true)),
+                new Mutation("delete", HttpMethod.DELETE, path(id), null),
+                new Mutation("check", HttpMethod.POST, path(id) + "/checks", null));
+    }
+
+    private static Map<String, Object> input(String name, String route, int interval, boolean enabled) {
+        return Map.of("name", name, "url", "http://127.0.0.1:4321" + route, "interval", interval, "enabled", enabled);
+    }
+
+    private static String path(String id) { return "/api/monitors/" + id; }
+
+    private static JsonNode data(ResponseEntity<JsonNode> response, int status) {
+        assertEquals(status, response.getStatusCode().value());
+        assertNotNull(response.getBody());
+        assertEquals(1, response.getBody().size());
+        assertTrue(response.getBody().has("data"));
+        return response.getBody().get("data");
+    }
+
+    private static JsonNode error(ResponseEntity<JsonNode> response, int status, String code) {
+        assertEquals(status, response.getStatusCode().value());
+        assertNotNull(response.getBody());
+        assertEquals(1, response.getBody().size());
+        assertEquals(code, response.getBody().at("/error/code").textValue());
+        return response.getBody();
+    }
+
+    private static String snapshot() throws Exception {
+        // All Monitor/CheckRun columns, not only counts; never include account credentials.
+        try (var connection = TestDatabase.connect(); var query = connection.createStatement(); var rows = query.executeQuery("""
+                SELECT json_build_object(
+                  'monitors', (SELECT coalesce(json_agg(m ORDER BY m.id), '[]') FROM e05_ownership.monitors m),
+                  'checks', (SELECT coalesce(json_agg(c ORDER BY c.id), '[]') FROM e05_ownership.check_runs c))::text
+                """)) {
+            assertTrue(rows.next());
+            return rows.getString(1);
+        }
+    }
+
+    private static void verifySqlAndProxyEvidence() throws Exception {
+        assertFalse(SqlEvidence.events.isEmpty());
+        assertTrue(SqlEvidence.events.stream().allMatch(SqlEvent::transaction));
+        for (var event : SqlEvidence.events) {
+            String sql = event.sql;
+            if (sql.startsWith("select") || sql.startsWith("update") || sql.startsWith("delete")) {
+                int where = sql.indexOf(" where ");
+                assertTrue(where >= 0 && sql.substring(where).contains("owner_user_id"), "Every resource query includes the owner predicate");
+            }
+        }
+        assertTrue(SqlEvidence.events.stream().anyMatch(e -> e.readOnly && e.sql.contains("check_runs")));
+        for (String operation : List.of("insert into e05_ownership.monitors", "insert into e05_ownership.check_runs",
+                "update e05_ownership.monitors", "delete from e05_ownership.monitors")) {
+            assertTrue(SqlEvidence.events.stream().anyMatch(e -> e.sql.startsWith(operation) && !e.readOnly));
+        }
+        Files.createDirectories(Path.of("target"));
+        Files.writeString(Path.of("target/e05-owner-sql.txt"), String.join("\n",
+                SqlEvidence.events.stream().map(Object::toString).distinct().toList()) + "\n");
+    }
+
+    private record Actor(String name, SessionClient client, String ownId, String ownCheck, String foreignId, String foreignCheck) {}
+    private record Mutation(String name, HttpMethod method, String path, Object body) {}
+    record SqlEvent(String sql, boolean transaction, boolean readOnly) {}
+
+    public static class SqlEvidence implements StatementInspector {
+        static final List<SqlEvent> events = new CopyOnWriteArrayList<>();
+        @Override public String inspect(String sql) {
+            if (sql.contains("e05_ownership.monitors") || sql.contains("e05_ownership.check_runs")) {
+                events.add(new SqlEvent(sql, TransactionSynchronizationManager.isActualTransactionActive(),
+                        TransactionSynchronizationManager.isCurrentTransactionReadOnly()));
+            }
+            return sql;
+        }
+    }
+}
diff --git a/backend/src/test/java/dev/evolution/monitor/OwnershipMigrationTest.java b/backend/src/test/java/dev/evolution/monitor/OwnershipMigrationTest.java
new file mode 100644
index 0000000..4a2cb66
--- /dev/null
+++ b/backend/src/test/java/dev/evolution/monitor/OwnershipMigrationTest.java
@@ -0,0 +1,195 @@
+package dev.evolution.monitor;
+
+import static org.junit.jupiter.api.Assertions.*;
+
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.sql.Connection;
+import java.util.List;
+import java.util.UUID;
+import java.util.concurrent.atomic.AtomicBoolean;
+import java.util.concurrent.atomic.AtomicReference;
+import org.junit.jupiter.api.Test;
+import org.springframework.boot.WebApplicationType;
+import org.springframework.boot.builder.SpringApplicationBuilder;
+import org.springframework.boot.web.context.WebServerInitializedEvent;
+import org.springframework.context.ApplicationListener;
+import org.springframework.context.ConfigurableApplicationContext;
+import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
+import org.springframework.web.server.ResponseStatusException;
+
+class OwnershipMigrationTest {
+    private static final UUID ALICE = UUID.fromString("00000000-0000-4000-8000-000000000051");
+    private static final UUID BOB = UUID.fromString("00000000-0000-4000-8000-000000000052");
+    private static final UUID MONITOR = UUID.fromString("00000000-0000-4000-8000-000000000053");
+    private static final UUID RUN = UUID.fromString("00000000-0000-4000-8000-000000000054");
+
+    @Test
+    void freshV4StartupAndRepeatedMigrationRequireValidOwners() throws Exception {
+        String schema = "e05_fresh";
+        TestDatabase.reset(schema);
+        try {
+            var flyway = TestDatabase.migration(schema).load();
+            assertEquals(4, flyway.migrate().migrationsExecuted);
+            assertEquals(List.of("1", "2", "3", "4"), java.util.Arrays.stream(flyway.info().applied())
+                    .map(migration -> migration.getVersion().toString()).toList());
+            flyway.validate();
+            assertEquals(0, flyway.migrate().migrationsExecuted);
+            try (var context = start(schema)) {
+                assertNotNull(context.getBean(MonitorStore.class));
+            }
+            try (var context = start(schema)) {
+                assertNotNull(context.getBean(MonitorStore.class));
+            }
+            assertOwnerConstraints(schema);
+        } finally { TestDatabase.drop(schema); }
+    }
+
+    @Test
+    void previousV3RefusesAtomicallyThenAcceptsOnlyExplicitVerifiedAssignment() throws Exception {
+        String schema = "e05_upgrade";
+        TestDatabase.reset(schema);
+        try {
+            assertEquals(3, TestDatabase.migration(schema).target("3").load().migrate().migrationsExecuted);
+            seedPreviousRows();
+            String before = canonicalRows();
+            assertStartupFailure(schema, "E05 ownership requires explicit verified assignment");
+            assertEquals(before, canonicalRows());
+            assertEquals(3, TestDatabase.migration(schema).load().info().applied().length);
+            assertEquals("0", scalar("SELECT count(*) FROM information_schema.columns WHERE table_schema='e05_upgrade' "
+                    + "AND table_name='monitors' AND column_name='owner_user_id'"));
+
+            // An explicitly staged but unknown UUID is refused too; nothing is auto-assigned.
+            TestDatabase.execute("ALTER TABLE e05_upgrade.monitors ADD COLUMN owner_user_id uuid; "
+                    + "UPDATE e05_upgrade.monitors SET owner_user_id = '00000000-0000-0000-0000-000000000000'");
+            var unknown = assertThrows(RuntimeException.class, () -> TestDatabase.migration(schema).load().migrate());
+            assertTrue(messages(unknown).contains("assignment must reference an existing verified user"));
+            assertEquals(before, canonicalRows());
+            assertEquals(3, TestDatabase.migration(schema).load().info().applied().length);
+
+            // Bob is deliberately not the first inserted user: assignment comes from this operator decision.
+            try (var connection = TestDatabase.connect(); var update = connection.prepareStatement(
+                    "UPDATE e05_upgrade.monitors m SET owner_user_id=u.id FROM e05_upgrade.users u "
+                            + "WHERE m.id=? AND u.username=?")) {
+                update.setObject(1, MONITOR);
+                update.setString(2, "bob-e04");
+                assertEquals(1, update.executeUpdate());
+            }
+            var upgrade = TestDatabase.migration(schema).load();
+            assertEquals(1, upgrade.migrate().migrationsExecuted);
+            upgrade.validate();
+            assertEquals(4, upgrade.info().applied().length);
+            assertEquals(0, upgrade.migrate().migrationsExecuted);
+            assertEquals(before, canonicalRows());
+            assertEquals(BOB.toString(), scalar("SELECT owner_user_id::text FROM e05_upgrade.monitors"));
+            assertOwnerConstraints(schema);
+            try (var context = start(schema)) {
+                var store = context.getBean(MonitorStore.class);
+                assertEquals(MONITOR, store.get(BOB, MONITOR).monitor().id());
+                assertEquals(RUN, store.history(BOB, MONITOR).getFirst().id());
+                assertEquals(RUN, store.check(BOB, MONITOR, RUN).id());
+                assertEquals(404, assertThrows(ResponseStatusException.class, () -> store.get(ALICE, MONITOR))
+                        .getStatusCode().value());
+                assertEquals(0, store.list(ALICE).size());
+            }
+            assertEquals(before, canonicalRows());
+            Files.createDirectories(Path.of("target"));
+            Files.writeString(Path.of("target/e05-migration-evidence.json"), """
+                    {"freshVersions":[1,2,3,4],"repeatMigrations":0,"priorV3RowsPreserved":true,
+                     "unassignedStartupRefusedBeforeReady":true,"refusalRolledBackColumn":true,
+                     "unknownOwnerRefused":true,"explicitBobAssignmentAccepted":true,
+                     "monitorAndHistoryUnchanged":true,"oldMigrationsUnchanged":true}
+                    """);
+        } finally { TestDatabase.drop(schema); }
+    }
+
+    @Test
+    void startupRejectsMissingNullableAndUnconstrainedOwnerColumns() {
+        for (String[] fixture : List.of(
+                new String[]{"e05_missing_owner", "DROP COLUMN owner_user_id", "missing column [owner_user_id]"},
+                new String[]{"e05_nullable_owner", "ALTER COLUMN owner_user_id DROP NOT NULL", "Monitor ownership must be required"},
+                new String[]{"e05_missing_owner_fk", "DROP CONSTRAINT monitors_owner_user_fk", "Monitor ownership must reference users"})) {
+            String schema = fixture[0];
+            TestDatabase.reset(schema);
+            try {
+                assertEquals(4, TestDatabase.migration(schema).load().migrate().migrationsExecuted);
+                TestDatabase.execute("ALTER TABLE " + schema + ".monitors " + fixture[1]);
+                assertStartupFailure(schema, fixture[2]);
+            } finally { TestDatabase.drop(schema); }
+        }
+    }
+
+    private static void seedPreviousRows() throws Exception {
+        try (var connection = TestDatabase.connect()) {
+            connection.setAutoCommit(false);
+            try (var user = connection.prepareStatement("INSERT INTO e05_upgrade.users VALUES (?, ?, ?)")) {
+                for (var account : List.of(new Object[]{ALICE, "alice-e04"}, new Object[]{BOB, "bob-e04"})) {
+                    user.setObject(1, account[0]);
+                    user.setString(2, (String) account[1]);
+                    user.setString(3, new BCryptPasswordEncoder(10).encode(SessionClient.password()));
+                    user.executeUpdate();
+                }
+            }
+            try (var statement = connection.createStatement()) {
+                statement.execute("INSERT INTO e05_upgrade.monitors VALUES ('" + MONITOR + "', 'Previous V3 Monitor', "
+                        + "'http://127.0.0.1:4321/ok', 60, true, '2026-08-28T00:00:00.123456Z', '2026-08-28T00:00:00.123456Z')");
+                statement.execute("INSERT INTO e05_upgrade.check_runs VALUES ('" + RUN + "', '" + MONITOR
+                        + "', 'MANUAL', 'SUCCEEDED', 200, 0, null, '2026-08-28T00:00:00.123456Z', '2026-08-28T00:00:00.123456Z')");
+            }
+            connection.commit();
+        }
+    }
+
+    private static String canonicalRows() throws Exception {
+        return scalar("SELECT json_agg(m)::text FROM (SELECT id,name,url,interval_seconds,enabled,created_at,updated_at "
+                + "FROM e05_upgrade.monitors ORDER BY id) m") + scalar("SELECT json_agg(c)::text FROM "
+                + "(SELECT * FROM e05_upgrade.check_runs ORDER BY id) c");
+    }
+
+    private static String scalar(String sql) throws Exception {
+        try (var connection = TestDatabase.connect(); var query = connection.createStatement(); var rows = query.executeQuery(sql)) {
+            assertTrue(rows.next());
+            return rows.getString(1);
+        }
+    }
+
+    private static void assertOwnerConstraints(String schema) throws Exception {
+        assertEquals("NO", scalar("SELECT is_nullable FROM information_schema.columns WHERE table_schema='" + schema
+                + "' AND table_name='monitors' AND column_name='owner_user_id'"));
+        try (Connection connection = TestDatabase.connect(); var keys = connection.getMetaData()
+                .getImportedKeys(connection.getCatalog(), schema, "monitors")) {
+            boolean found = false;
+            while (keys.next()) if ("owner_user_id".equals(keys.getString("FKCOLUMN_NAME"))
+                    && "users".equals(keys.getString("PKTABLE_NAME")) && schema.equals(keys.getString("PKTABLE_SCHEM"))) found = true;
+            assertTrue(found);
+        }
+    }
+
+    private static void assertStartupFailure(String schema, String expected) {
+        var ready = new AtomicBoolean();
+        var unexpected = new AtomicReference<ConfigurableApplicationContext>();
+        try {
+            ApplicationListener<WebServerInitializedEvent> listener = event -> ready.set(true);
+            var failure = assertThrows(RuntimeException.class, () -> unexpected.set(new SpringApplicationBuilder(MonitorApplication.class)
+                    .web(WebApplicationType.SERVLET).listeners(listener).run(arguments(schema))));
+            assertTrue(messages(failure).contains(expected), "Startup must refuse for the frozen ownership condition");
+            assertFalse(ready.get());
+        } finally { if (unexpected.get() != null) unexpected.get().close(); }
+    }
+
+    private static String messages(Throwable failure) {
+        var messages = new StringBuilder();
+        for (Throwable cause = failure; cause != null; cause = cause.getCause()) messages.append(cause.getMessage()).append('\n');
+        return messages.toString();
+    }
+
+    private static ConfigurableApplicationContext start(String schema) {
+        return new SpringApplicationBuilder(MonitorApplication.class).web(WebApplicationType.NONE).run(arguments(schema));
+    }
+
+    private static String[] arguments(String schema) {
+        return new String[]{"--spring.flyway.schemas=" + schema, "--spring.flyway.default-schema=" + schema,
+                "--spring.jpa.properties.hibernate.default_schema=" + schema,
+                "--spring.main.banner-mode=off", "--logging.level.root=OFF", "--server.port=4322"};
+    }
+}
diff --git a/backend/src/test/java/dev/evolution/monitor/PostgresPersistenceTest.java b/backend/src/test/java/dev/evolution/monitor/PostgresPersistenceTest.java
index c8c9f29..ad51a5f 100644
--- a/backend/src/test/java/dev/evolution/monitor/PostgresPersistenceTest.java
+++ b/backend/src/test/java/dev/evolution/monitor/PostgresPersistenceTest.java
@@ -38,7 +38,7 @@ class PostgresPersistenceTest {
         String schema = "e04_migrations";
         TestDatabase.reset(schema);
         try {
-            var fresh = TestDatabase.migration(schema).load();
+            var fresh = TestDatabase.migration(schema).target("3").load();
             assertEquals(3, fresh.migrate().migrationsExecuted);
             assertEquals(List.of("1", "2", "3"), java.util.Arrays.stream(fresh.info().applied())
                     .map(migration -> migration.getVersion().toString()).toList());
@@ -49,7 +49,7 @@ class PostgresPersistenceTest {
             TestDatabase.reset(schema);
             assertEquals(1, TestDatabase.migration(schema).target("1").load().migrate().migrationsExecuted);
             assertEquals(1, TestDatabase.migration(schema).target("2").load().migrate().migrationsExecuted);
-            var upgrade = TestDatabase.migration(schema).load();
+            var upgrade = TestDatabase.migration(schema).target("3").load();
             assertEquals(1, upgrade.migrate().migrationsExecuted);
             upgrade.validate();
             assertEquals(3, upgrade.info().applied().length);
@@ -75,46 +75,51 @@ class PostgresPersistenceTest {
         try {
             try (var first = start(schema)) {
                 var store = first.getBean(MonitorStore.class);
+                var accounts = first.getBean(UserAccounts.class);
+                accounts.bootstrap(SessionClient.password(), SessionClient.password());
+                UUID owner = ((UserAccounts.AccountUser) accounts.loadUserByUsername("alice-e04")).userId();
                 assertTrue(AopUtils.isAopProxy(store), "Transactions must use the actual Spring store proxy");
                 var transaction = new TransactionTemplate(first.getBean(PlatformTransactionManager.class));
                 var entities = SharedEntityManagerCreator.createSharedEntityManager(first.getBean(EntityManagerFactory.class));
-                transaction.executeWithoutResult(status -> entities.persist(MonitorEntity.fromDomain(monitor)));
-                assertEquals(success, store.saveCheck(successInput));
-                assertEquals(timeout, store.saveCheck(timeoutInput));
+                transaction.executeWithoutResult(status -> entities.persist(MonitorEntity.fromDomain(monitor, owner)));
+                assertEquals(success, store.saveCheck(owner, successInput));
+                assertEquals(timeout, store.saveCheck(owner, timeoutInput));
 
                 var rolledBack = new AtomicReference<UUID>();
                 transaction.executeWithoutResult(status -> {
-                    rolledBack.set(store.create(new MonitorController.CreateMonitor("Rolled back",
+                    rolledBack.set(store.create(owner, new MonitorController.CreateMonitor("Rolled back",
                             "http://127.0.0.1:4321/ok", 60, true)).monitor().id());
                     entities.flush(); // Make the INSERT execute before the rollback assertion.
                     status.setRollbackOnly();
                 });
                 assertEquals(404, assertThrows(ResponseStatusException.class,
-                        () -> store.monitor(rolledBack.get())).getStatusCode().value());
+                        () -> store.monitor(owner, rolledBack.get())).getStatusCode().value());
             }
 
             try (var fresh = start(schema)) {
                 var store = fresh.getBean(MonitorStore.class);
+                UUID owner = ((UserAccounts.AccountUser) fresh.getBean(UserAccounts.class)
+                        .loadUserByUsername("alice-e04")).userId();
                 assertEquals(new Monitor(MONITOR_ID, monitor.name(), monitor.url(), 1, false,
-                        DATABASE_TIME, DATABASE_TIME), store.monitor(MONITOR_ID));
-                assertEquals(List.of(timeout, success), store.history(MONITOR_ID));
-                assertEquals(success, store.check(MONITOR_ID, success.id()));
-                assertEquals(timeout, store.check(MONITOR_ID, timeout.id()));
-                assertEquals(timeout, store.get(MONITOR_ID).latestCheck());
-                assertEquals(1, store.list().size());
+                        DATABASE_TIME, DATABASE_TIME), store.monitor(owner, MONITOR_ID));
+                assertEquals(List.of(timeout, success), store.history(owner, MONITOR_ID));
+                assertEquals(success, store.check(owner, MONITOR_ID, success.id()));
+                assertEquals(timeout, store.check(owner, MONITOR_ID, timeout.id()));
+                assertEquals(timeout, store.get(owner, MONITOR_ID).latestCheck());
+                assertEquals(1, store.list(owner).size());
 
                 var json = fresh.getBean(ObjectMapper.class);
-                var monitorWire = json.valueToTree(new ApiData<>(store.monitor(MONITOR_ID)));
+                var monitorWire = json.valueToTree(new ApiData<>(store.monitor(owner, MONITOR_ID)));
                 assertTrue(monitorWire.at("/data/interval").isIntegralNumber());
                 assertEquals(1, monitorWire.at("/data/interval").intValue());
                 assertTrue(monitorWire.at("/data/enabled").isBoolean());
                 assertFalse(monitorWire.at("/data/enabled").booleanValue());
                 assertEquals(DATABASE_TIME.toString(), monitorWire.at("/data/createdAt").textValue());
-                var successWire = json.valueToTree(new ApiData<>(store.check(MONITOR_ID, success.id())));
+                var successWire = json.valueToTree(new ApiData<>(store.check(owner, MONITOR_ID, success.id())));
                 assertTrue(successWire.at("/data/latencyMs").isIntegralNumber());
                 assertEquals(0, successWire.at("/data/latencyMs").longValue());
                 assertTrue(successWire.at("/data/failureReason").isNull());
-                var timeoutWire = json.valueToTree(new ApiData<>(store.check(MONITOR_ID, timeout.id())));
+                var timeoutWire = json.valueToTree(new ApiData<>(store.check(owner, MONITOR_ID, timeout.id())));
                 assertTrue(timeoutWire.at("/data/httpStatus").isNull());
                 assertEquals("TIMEOUT", timeoutWire.at("/data/failureReason").textValue());
 
@@ -135,14 +140,14 @@ class PostgresPersistenceTest {
                     }
                 }
 
-                var updated = store.replace(MONITOR_ID, new MonitorController.CreateMonitor(
+                var updated = store.replace(owner, MONITOR_ID, new MonitorController.CreateMonitor(
                         "Mapping updated", monitor.url(), 60, true));
-                assertEquals(updated, store.get(MONITOR_ID));
+                assertEquals(updated, store.get(owner, MONITOR_ID));
                 assertEquals("Mapping updated", updated.monitor().name());
                 assertTrue(updated.monitor().enabled());
-                store.delete(MONITOR_ID);
+                store.delete(owner, MONITOR_ID);
                 assertEquals(404, assertThrows(ResponseStatusException.class,
-                        () -> store.history(MONITOR_ID)).getStatusCode().value());
+                        () -> store.history(owner, MONITOR_ID)).getStatusCode().value());
                 try (var connection = TestDatabase.connect(); var query = connection.createStatement();
                         var rows = query.executeQuery("SELECT (SELECT count(*) FROM e04_mapping.monitors) AS monitors, "
                                 + "(SELECT count(*) FROM e04_mapping.check_runs) AS checks")) {
@@ -183,7 +188,7 @@ class PostgresPersistenceTest {
         var ready = new AtomicBoolean();
         var unexpectedContext = new AtomicReference<ConfigurableApplicationContext>();
         try {
-            assertEquals(3, TestDatabase.migration(schema).load().migrate().migrationsExecuted);
+            assertEquals(3, TestDatabase.migration(schema).target("3").load().migrate().migrationsExecuted);
             TestDatabase.execute(ddl);
             ApplicationListener<WebServerInitializedEvent> readyListener = event -> ready.set(true);
             var failure = assertThrows(RuntimeException.class, () -> unexpectedContext.set(
diff --git a/backend/src/test/java/dev/evolution/monitor/SessionClient.java b/backend/src/test/java/dev/evolution/monitor/SessionClient.java
index 27600e2..2e1595b 100644
--- a/backend/src/test/java/dev/evolution/monitor/SessionClient.java
+++ b/backend/src/test/java/dev/evolution/monitor/SessionClient.java
@@ -15,6 +15,7 @@ import org.springframework.http.ResponseEntity;
 
 // All sensitive values stay in memory; assertions and evidence use only status/flags.
 final class SessionClient {
+    static final String TRUSTED_ORIGIN = "http://127.0.0.1:4323";
     private final TestRestTemplate api;
     private String cookie;
     private String csrfHeader;
@@ -60,6 +61,9 @@ final class SessionClient {
     ResponseEntity<JsonNode> send(String path, HttpMethod method, Object body, boolean includeCsrf) {
         var headers = new HttpHeaders();
         if (cookie != null) headers.set(HttpHeaders.COOKIE, cookie);
+        if (method != HttpMethod.GET && method != HttpMethod.HEAD && method != HttpMethod.OPTIONS) {
+            headers.setOrigin(TRUSTED_ORIGIN);
+        }
         if (includeCsrf) headers.set(csrfHeader, csrfToken);
         if (body != null) headers.setContentType(MediaType.APPLICATION_JSON);
         var response = api.exchange(path, method, new HttpEntity<>(body, headers), JsonNode.class);
@@ -74,6 +78,7 @@ final class SessionClient {
         api.getRestTemplate().getInterceptors().add((request, body, execution) -> {
             request.getHeaders().set(HttpHeaders.COOKIE, cookie);
             if (request.getMethod() != HttpMethod.GET && request.getMethod() != HttpMethod.HEAD) {
+                request.getHeaders().setOrigin(TRUSTED_ORIGIN);
                 request.getHeaders().set(csrfHeader, csrfToken);
             }
             return execution.execute(request, body);
diff --git a/backend/src/test/java/dev/evolution/monitor/TestDatabase.java b/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
index 4ea16ee..75d9e29 100644
--- a/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
+++ b/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
@@ -11,7 +11,8 @@ import org.springframework.test.context.DynamicPropertyRegistry;
 final class TestDatabase {
     private static final Set<String> SCHEMAS = Set.of("e04_functional", "e04_migrations", "e04_mapping",
             "e04_missing_column", "e04_extra_required", "e04_session", "e04_users",
-            "e04_missing_user_column", "e04_extra_user_required");
+            "e04_missing_user_column", "e04_extra_user_required", "e05_ownership", "e05_fresh",
+            "e05_upgrade", "e05_missing_owner", "e05_nullable_owner", "e05_missing_owner_fk");
 
     static Connection connect() throws SQLException {
         return DriverManager.getConnection(System.getenv().getOrDefault("DB_URL", "jdbc:postgresql://127.0.0.1:15432/monitor"),
diff --git a/backend/src/test/java/dev/evolution/monitor/UserAccountPersistenceTest.java b/backend/src/test/java/dev/evolution/monitor/UserAccountPersistenceTest.java
index 7d33ac4..2df5833 100644
--- a/backend/src/test/java/dev/evolution/monitor/UserAccountPersistenceTest.java
+++ b/backend/src/test/java/dev/evolution/monitor/UserAccountPersistenceTest.java
@@ -36,7 +36,7 @@ class UserAccountPersistenceTest {
         SqlEvidence.events.clear();
         try {
             assertEquals(2, TestDatabase.migration(schema).target("2").load().migrate().migrationsExecuted);
-            var latest = TestDatabase.migration(schema).load();
+            var latest = TestDatabase.migration(schema).target("3").load();
             assertEquals(1, latest.migrate().migrationsExecuted);
             assertEquals(3, latest.info().applied().length);
             assertEquals(0, latest.migrate().migrationsExecuted);
@@ -121,7 +121,7 @@ class UserAccountPersistenceTest {
         var ready = new AtomicBoolean();
         var unexpected = new AtomicReference<ConfigurableApplicationContext>();
         try {
-            assertEquals(3, TestDatabase.migration(schema).load().migrate().migrationsExecuted);
+            assertEquals(3, TestDatabase.migration(schema).target("3").load().migrate().migrationsExecuted);
             TestDatabase.execute(ddl);
             ApplicationListener<WebServerInitializedEvent> listener = event -> ready.set(true);
             RuntimeException failure = assertThrows(RuntimeException.class, () -> unexpected.set(
diff --git a/evidence/E05/authorization.json b/evidence/E05/authorization.json
new file mode 100644
index 0000000..eb5a355
--- /dev/null
+++ b/evidence/E05/authorization.json
@@ -0,0 +1,151 @@
+{
+  "foreignOwnerInputIgnored" : true,
+  "outboundTransactionAbsent" : true,
+  "dataset" : "two users/two Monitors/two initial CheckRuns",
+  "proxyAndSqlChecked" : true,
+  "ownerLifecycle" : true,
+  "matrix" : [ {
+    "probe" : "unchanged-fixed-counterexample-after-fix",
+    "status" : 404
+  }, {
+    "outboundUnchanged" : true,
+    "target" : "foreign",
+    "mutation" : "edit",
+    "status" : 404,
+    "allRowsUnchanged" : true,
+    "actor" : "alice"
+  }, {
+    "outboundUnchanged" : true,
+    "target" : "foreign",
+    "mutation" : "pause",
+    "status" : 404,
+    "allRowsUnchanged" : true,
+    "actor" : "alice"
+  }, {
+    "outboundUnchanged" : true,
+    "target" : "foreign",
+    "mutation" : "resume",
+    "status" : 404,
+    "allRowsUnchanged" : true,
+    "actor" : "alice"
+  }, {
+    "outboundUnchanged" : true,
+    "target" : "foreign",
+    "mutation" : "delete",
+    "status" : 404,
+    "allRowsUnchanged" : true,
+    "actor" : "alice"
+  }, {
+    "outboundUnchanged" : true,
+    "target" : "foreign",
+    "mutation" : "check",
+    "status" : 404,
+    "allRowsUnchanged" : true,
+    "actor" : "alice"
+  }, {
+    "outboundUnchanged" : true,
+    "target" : "nonexistent",
+    "mutation" : "edit",
+    "status" : 404,
+    "allRowsUnchanged" : true,
+    "actor" : "alice"
+  }, {
+    "outboundUnchanged" : true,
+    "target" : "nonexistent",
+    "mutation" : "pause",
+    "status" : 404,
+    "allRowsUnchanged" : true,
+    "actor" : "alice"
+  }, {
+    "outboundUnchanged" : true,
+    "target" : "nonexistent",
+    "mutation" : "resume",
+    "status" : 404,
+    "allRowsUnchanged" : true,
+    "actor" : "alice"
+  }, {
+    "outboundUnchanged" : true,
+    "target" : "nonexistent",
+    "mutation" : "delete",
+    "status" : 404,
+    "allRowsUnchanged" : true,
+    "actor" : "alice"
+  }, {
+    "outboundUnchanged" : true,
+    "target" : "nonexistent",
+    "mutation" : "check",
+    "status" : 404,
+    "allRowsUnchanged" : true,
+    "actor" : "alice"
+  }, {
+    "outboundUnchanged" : true,
+    "target" : "foreign",
+    "mutation" : "edit",
+    "status" : 404,
+    "allRowsUnchanged" : true,
+    "actor" : "bob"
+  }, {
+    "outboundUnchanged" : true,
+    "target" : "foreign",
+    "mutation" : "pause",
+    "status" : 404,
+    "allRowsUnchanged" : true,
+    "actor" : "bob"
+  }, {
+    "outboundUnchanged" : true,
+    "target" : "foreign",
+    "mutation" : "resume",
+    "status" : 404,
+    "allRowsUnchanged" : true,
+    "actor" : "bob"
+  }, {
+    "outboundUnchanged" : true,
+    "target" : "foreign",
+    "mutation" : "delete",
+    "status" : 404,
+    "allRowsUnchanged" : true,
+    "actor" : "bob"
+  }, {
+    "outboundUnchanged" : true,
+    "target" : "foreign",
+    "mutation" : "check",
+    "status" : 404,
+    "allRowsUnchanged" : true,
+    "actor" : "bob"
+  }, {
+    "outboundUnchanged" : true,
+    "target" : "nonexistent",
+    "mutation" : "edit",
+    "status" : 404,
+    "allRowsUnchanged" : true,
+    "actor" : "bob"
+  }, {
+    "outboundUnchanged" : true,
+    "target" : "nonexistent",
+    "mutation" : "pause",
+    "status" : 404,
+    "allRowsUnchanged" : true,
+    "actor" : "bob"
+  }, {
+    "outboundUnchanged" : true,
+    "target" : "nonexistent",
+    "mutation" : "resume",
+    "status" : 404,
+    "allRowsUnchanged" : true,
+    "actor" : "bob"
+  }, {
+    "outboundUnchanged" : true,
+    "target" : "nonexistent",
+    "mutation" : "delete",
+    "status" : 404,
+    "allRowsUnchanged" : true,
+    "actor" : "bob"
+  }, {
+    "outboundUnchanged" : true,
+    "target" : "nonexistent",
+    "mutation" : "check",
+    "status" : 404,
+    "allRowsUnchanged" : true,
+    "actor" : "bob"
+  } ]
+}
diff --git a/evidence/E05/migration.json b/evidence/E05/migration.json
new file mode 100644
index 0000000..1d2e8a1
--- /dev/null
+++ b/evidence/E05/migration.json
@@ -0,0 +1,4 @@
+{"freshVersions":[1,2,3,4],"repeatMigrations":0,"priorV3RowsPreserved":true,
+ "unassignedStartupRefusedBeforeReady":true,"refusalRolledBackColumn":true,
+ "unknownOwnerRefused":true,"explicitBobAssignmentAccepted":true,
+ "monitorAndHistoryUnchanged":true,"oldMigrationsUnchanged":true}
diff --git a/evidence/E05/owner-sql.txt b/evidence/E05/owner-sql.txt
new file mode 100644
index 0000000..bcfa344
--- /dev/null
+++ b/evidence/E05/owner-sql.txt
@@ -0,0 +1,11 @@
+SqlEvent[sql=insert into e05_ownership.monitors (created_at,enabled,interval_seconds,name,owner_user_id,updated_at,url,id) values (?,?,?,?,?,?,?,?), transaction=true, readOnly=false]
+SqlEvent[sql=select me1_0.id,me1_0.created_at,me1_0.enabled,me1_0.interval_seconds,me1_0.name,me1_0.owner_user_id,me1_0.updated_at,me1_0.url from e05_ownership.monitors me1_0 where me1_0.id=? and me1_0.owner_user_id=?, transaction=true, readOnly=true]
+SqlEvent[sql=select me1_0.id,me1_0.created_at,me1_0.enabled,me1_0.interval_seconds,me1_0.name,me1_0.owner_user_id,me1_0.updated_at,me1_0.url from e05_ownership.monitors me1_0 where me1_0.id=? and me1_0.owner_user_id=?, transaction=true, readOnly=false]
+SqlEvent[sql=insert into e05_ownership.check_runs (failure_reason,finished_at,http_status,latency_ms,monitor_id,started_at,state,trigger_kind,id) values (?,?,?,?,?,?,?,?,?), transaction=true, readOnly=false]
+SqlEvent[sql=select me1_0.id,me1_0.created_at,me1_0.enabled,me1_0.interval_seconds,me1_0.name,me1_0.owner_user_id,me1_0.updated_at,me1_0.url from e05_ownership.monitors me1_0 where me1_0.owner_user_id=? order by me1_0.created_at,me1_0.id, transaction=true, readOnly=true]
+SqlEvent[sql=select cre1_0.id,cre1_0.failure_reason,cre1_0.finished_at,cre1_0.http_status,cre1_0.latency_ms,cre1_0.monitor_id,cre1_0.started_at,cre1_0.state,cre1_0.trigger_kind from e05_ownership.check_runs cre1_0,e05_ownership.monitors me1_0 where cre1_0.monitor_id=me1_0.id and me1_0.id=? and me1_0.owner_user_id=? order by cre1_0.finished_at desc,cre1_0.id desc fetch first ? rows only, transaction=true, readOnly=true]
+SqlEvent[sql=select cre1_0.id,cre1_0.failure_reason,cre1_0.finished_at,cre1_0.http_status,cre1_0.latency_ms,cre1_0.monitor_id,cre1_0.started_at,cre1_0.state,cre1_0.trigger_kind from e05_ownership.check_runs cre1_0,e05_ownership.monitors me1_0 where cre1_0.monitor_id=me1_0.id and me1_0.id=? and me1_0.owner_user_id=? order by cre1_0.finished_at desc,cre1_0.id desc, transaction=true, readOnly=true]
+SqlEvent[sql=select cre1_0.id,cre1_0.failure_reason,cre1_0.finished_at,cre1_0.http_status,cre1_0.latency_ms,cre1_0.monitor_id,cre1_0.started_at,cre1_0.state,cre1_0.trigger_kind from e05_ownership.check_runs cre1_0,e05_ownership.monitors me1_0 where cre1_0.id=? and cre1_0.monitor_id=? and cre1_0.monitor_id=me1_0.id and me1_0.owner_user_id=?, transaction=true, readOnly=true]
+SqlEvent[sql=update e05_ownership.monitors me1_0 set name=?,url=?,interval_seconds=?,enabled=?,updated_at=? where me1_0.id=? and me1_0.owner_user_id=?, transaction=true, readOnly=false]
+SqlEvent[sql=delete from e05_ownership.monitors me1_0 where me1_0.id=? and me1_0.owner_user_id=?, transaction=true, readOnly=false]
+SqlEvent[sql=select cre1_0.id,cre1_0.failure_reason,cre1_0.finished_at,cre1_0.http_status,cre1_0.latency_ms,cre1_0.monitor_id,cre1_0.started_at,cre1_0.state,cre1_0.trigger_kind from e05_ownership.check_runs cre1_0,e05_ownership.monitors me1_0 where cre1_0.monitor_id=me1_0.id and me1_0.id=? and me1_0.owner_user_id=? order by cre1_0.finished_at desc,cre1_0.id desc fetch first ? rows only, transaction=true, readOnly=false]


