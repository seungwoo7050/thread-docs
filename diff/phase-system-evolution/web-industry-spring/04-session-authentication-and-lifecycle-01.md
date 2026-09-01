# E04 세션 인증과 수명주기

## `Spring Security 세션 인증과 1시간 수명주기 적용`

diff --git a/backend/pom.xml b/backend/pom.xml
index 9d6a9b1..4768669 100644
--- a/backend/pom.xml
+++ b/backend/pom.xml
@@ -23,6 +23,10 @@
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-data-jpa</artifactId>
     </dependency>
+    <dependency>
+      <groupId>org.springframework.boot</groupId>
+      <artifactId>spring-boot-starter-security</artifactId>
+    </dependency>
     <dependency>
       <groupId>org.flywaydb</groupId>
       <artifactId>flyway-database-postgresql</artifactId>
diff --git a/backend/src/main/java/dev/evolution/monitor/ApiErrors.java b/backend/src/main/java/dev/evolution/monitor/ApiErrors.java
index c1f7693..9cb3d07 100644
--- a/backend/src/main/java/dev/evolution/monitor/ApiErrors.java
+++ b/backend/src/main/java/dev/evolution/monitor/ApiErrors.java
@@ -4,6 +4,7 @@ import org.springframework.http.HttpStatus;
 import org.springframework.http.MediaType;
 import org.springframework.http.ResponseEntity;
 import org.springframework.http.converter.HttpMessageNotReadableException;
+import org.springframework.security.authentication.BadCredentialsException;
 import org.springframework.web.HttpMediaTypeNotAcceptableException;
 import org.springframework.web.HttpMediaTypeNotSupportedException;
 import org.springframework.web.HttpRequestMethodNotSupportedException;
@@ -16,10 +17,20 @@ import org.springframework.web.servlet.resource.NoResourceFoundException;
 
 @RestControllerAdvice
 public class ApiErrors {
-    public enum Code { INVALID_INPUT, NOT_FOUND, INTERNAL_ERROR }
+    public enum Code { INVALID_INPUT, NOT_FOUND, UNAUTHENTICATED, FORBIDDEN, INTERNAL_ERROR }
     public record Detail(Code code, String message) {}
     public record Failure(Detail error) {}
 
+    static Failure unauthenticatedBody() {
+        return new Failure(new Detail(Code.UNAUTHENTICATED, "Authentication required."));
+    }
+
+    @ExceptionHandler(BadCredentialsException.class)
+    ResponseEntity<Failure> unauthenticated() {
+        return ResponseEntity.status(HttpStatus.UNAUTHORIZED).contentType(MediaType.APPLICATION_JSON)
+                .body(unauthenticatedBody());
+    }
+
     @ExceptionHandler(ResponseStatusException.class)
     ResponseEntity<Failure> rejected(ResponseStatusException error) {
         return switch (error.getStatusCode().value()) {
diff --git a/backend/src/main/java/dev/evolution/monitor/AuthenticationConfig.java b/backend/src/main/java/dev/evolution/monitor/AuthenticationConfig.java
new file mode 100644
index 0000000..48908f9
--- /dev/null
+++ b/backend/src/main/java/dev/evolution/monitor/AuthenticationConfig.java
@@ -0,0 +1,108 @@
+package dev.evolution.monitor;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import jakarta.servlet.DispatcherType;
+import java.time.Clock;
+import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
+import org.springframework.boot.autoconfigure.condition.ConditionalOnWebApplication;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.http.HttpMethod;
+import org.springframework.http.MediaType;
+import org.springframework.security.authentication.AnonymousAuthenticationToken;
+import org.springframework.security.authentication.AuthenticationManager;
+import org.springframework.security.authentication.ProviderManager;
+import org.springframework.security.authentication.dao.DaoAuthenticationProvider;
+import org.springframework.security.config.annotation.web.builders.HttpSecurity;
+import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
+import org.springframework.security.config.http.SessionCreationPolicy;
+import org.springframework.security.core.context.SecurityContextHolder;
+import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
+import org.springframework.security.crypto.password.PasswordEncoder;
+import org.springframework.security.web.SecurityFilterChain;
+import org.springframework.security.web.context.HttpSessionSecurityContextRepository;
+import org.springframework.security.web.context.SecurityContextHolderFilter;
+import org.springframework.security.web.context.SecurityContextRepository;
+import org.springframework.security.web.csrf.CsrfTokenRepository;
+import org.springframework.security.web.csrf.HttpSessionCsrfTokenRepository;
+
+@Configuration
+public class AuthenticationConfig {
+    static final String COOKIE_NAME = "WSESESSION";
+
+    @Bean
+    PasswordEncoder passwords() { return new BCryptPasswordEncoder(10); }
+
+    @Bean
+    @ConditionalOnMissingBean(Clock.class)
+    Clock sessionClock() { return Clock.systemUTC(); }
+
+    @Bean
+    AuthenticationManager authenticationManager(UserAccounts accounts, PasswordEncoder passwords) {
+        var provider = new DaoAuthenticationProvider(accounts);
+        provider.setPasswordEncoder(passwords);
+        // ProviderManager erases authenticated credential material before it reaches a session.
+        return new ProviderManager(provider);
+    }
+
+    @Bean
+    SecurityContextRepository securityContexts() {
+        var repository = new HttpSessionSecurityContextRepository();
+        repository.setDisableUrlRewriting(true);
+        return repository;
+    }
+
+    @Bean
+    CsrfTokenRepository csrfTokens() { return new HttpSessionCsrfTokenRepository(); }
+
+    @Bean
+    @ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
+    SecurityFilterChain apiSecurity(HttpSecurity http, SecurityContextRepository contexts,
+            CsrfTokenRepository csrfTokens, ObjectMapper json, Clock clock) throws Exception {
+        return http
+                // Keep the framework's session-backed CSRF protection for browser requests.
+                .csrf(csrf -> csrf.csrfTokenRepository(csrfTokens))
+                .httpBasic(AbstractHttpConfigurer::disable)
+                .formLogin(AbstractHttpConfigurer::disable)
+                .requestCache(AbstractHttpConfigurer::disable)
+                .securityContext(security -> security.securityContextRepository(contexts).requireExplicitSave(true))
+                .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED))
+                .authorizeHttpRequests(authorize -> authorize
+                        .dispatcherTypeMatchers(DispatcherType.ERROR).permitAll()
+                        .requestMatchers(HttpMethod.POST, "/api/session/login").permitAll()
+                        .requestMatchers(HttpMethod.GET, "/api/session/csrf").permitAll()
+                        .anyRequest().authenticated())
+                .exceptionHandling(errors -> errors
+                        .authenticationEntryPoint((request, response, failure) -> {
+                            response.setStatus(401);
+                            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
+                            json.writeValue(response.getOutputStream(), ApiErrors.unauthenticatedBody());
+                        })
+                        .accessDeniedHandler((request, response, failure) -> {
+                            var current = SecurityContextHolder.getContext().getAuthentication();
+                            boolean authenticated = current != null && current.isAuthenticated()
+                                    && !(current instanceof AnonymousAuthenticationToken);
+                            response.setStatus(authenticated ? 403 : 401);
+                            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
+                            json.writeValue(response.getOutputStream(), authenticated
+                                    ? new ApiErrors.Failure(new ApiErrors.Detail(ApiErrors.Code.FORBIDDEN, "Request could not be verified."))
+                                    : ApiErrors.unauthenticatedBody());
+                        }))
+                .logout(logout -> logout
+                        .logoutRequestMatcher(request -> "POST".equals(request.getMethod())
+                                && "/api/session/logout".equals(request.getServletPath()))
+                        .invalidateHttpSession(true).clearAuthentication(true).deleteCookies(COOKIE_NAME)
+                        .logoutSuccessHandler((request, response, authentication) -> {
+                            // LogoutFilter precedes AuthorizationFilter: a CSRF session alone is not a login.
+                            boolean authenticated = authentication != null && authentication.isAuthenticated()
+                                    && !(authentication instanceof AnonymousAuthenticationToken);
+                            response.setStatus(authenticated ? 200 : 401);
+                            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
+                            json.writeValue(response.getOutputStream(), authenticated
+                                    ? new ApiData<>(null) : ApiErrors.unauthenticatedBody());
+                        }))
+                // Invalidate before the repository can load an expired SecurityContext.
+                .addFilterBefore(new SessionExpiryFilter(clock), SecurityContextHolderFilter.class)
+                .build();
+    }
+}
diff --git a/backend/src/main/java/dev/evolution/monitor/BootstrapUsers.java b/backend/src/main/java/dev/evolution/monitor/BootstrapUsers.java
new file mode 100644
index 0000000..42952de
--- /dev/null
+++ b/backend/src/main/java/dev/evolution/monitor/BootstrapUsers.java
@@ -0,0 +1,32 @@
+package dev.evolution.monitor;
+
+import org.springframework.boot.ApplicationArguments;
+import org.springframework.boot.ApplicationRunner;
+import org.springframework.boot.SpringApplication;
+import org.springframework.context.ConfigurableApplicationContext;
+import org.springframework.context.annotation.Profile;
+import org.springframework.stereotype.Component;
+import org.springframework.web.context.WebApplicationContext;
+
+@Component
+@Profile("bootstrap")
+class BootstrapUsers implements ApplicationRunner {
+    private final UserAccounts accounts;
+    private final ConfigurableApplicationContext context;
+
+    BootstrapUsers(UserAccounts accounts, ConfigurableApplicationContext context) {
+        this.accounts = accounts;
+        this.context = context;
+    }
+
+    @Override
+    public void run(ApplicationArguments arguments) {
+        if (context instanceof WebApplicationContext) {
+            throw new IllegalStateException("Bootstrap requires --spring.main.web-application-type=none.");
+        }
+        // Read only environment secrets, not command-line arguments or committed defaults.
+        accounts.bootstrap(System.getenv("E04_ALICE_PASSWORD"), System.getenv("E04_BOB_PASSWORD"));
+        System.out.println("Fixture users prepared: count=2");
+        SpringApplication.exit(context);
+    }
+}
diff --git a/backend/src/main/java/dev/evolution/monitor/SchemaCompatibility.java b/backend/src/main/java/dev/evolution/monitor/SchemaCompatibility.java
index 70270a2..8f112f3 100644
--- a/backend/src/main/java/dev/evolution/monitor/SchemaCompatibility.java
+++ b/backend/src/main/java/dev/evolution/monitor/SchemaCompatibility.java
@@ -29,7 +29,8 @@ public class SchemaCompatibility implements InitializingBean {
     public void afterPropertiesSet() throws SQLException {
         Map<String, Set<String>> mapped = Map.of(
                 MonitorEntity.class.getAnnotation(Table.class).name(), columns(MonitorEntity.class),
-                CheckRunEntity.class.getAnnotation(Table.class).name(), columns(CheckRunEntity.class));
+                CheckRunEntity.class.getAnnotation(Table.class).name(), columns(CheckRunEntity.class),
+                UserEntity.class.getAnnotation(Table.class).name(), columns(UserEntity.class));
         // Hibernate validate does not reject extra NOT NULL columns which make its INSERTs impossible.
         try (var connection = database.getConnection(); var query = connection.prepareStatement("""
                 SELECT table_name, column_name FROM information_schema.columns
diff --git a/backend/src/main/java/dev/evolution/monitor/SessionController.java b/backend/src/main/java/dev/evolution/monitor/SessionController.java
new file mode 100644
index 0000000..d8ef2d1
--- /dev/null
+++ b/backend/src/main/java/dev/evolution/monitor/SessionController.java
@@ -0,0 +1,78 @@
+package dev.evolution.monitor;
+
+import com.fasterxml.jackson.databind.JsonNode;
+import jakarta.servlet.http.HttpServletRequest;
+import jakarta.servlet.http.HttpServletResponse;
+import java.nio.charset.StandardCharsets;
+import java.security.Principal;
+import java.time.Clock;
+import org.springframework.security.authentication.AuthenticationManager;
+import org.springframework.security.authentication.BadCredentialsException;
+import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
+import org.springframework.security.core.context.SecurityContextHolder;
+import org.springframework.security.web.authentication.session.ChangeSessionIdAuthenticationStrategy;
+import org.springframework.security.web.context.SecurityContextRepository;
+import org.springframework.security.web.csrf.CsrfAuthenticationStrategy;
+import org.springframework.security.web.csrf.CsrfToken;
+import org.springframework.security.web.csrf.CsrfTokenRepository;
+import org.springframework.web.bind.annotation.GetMapping;
+import org.springframework.web.bind.annotation.PostMapping;
+import org.springframework.web.bind.annotation.RequestBody;
+import org.springframework.web.bind.annotation.RestController;
+
+@RestController
+public class SessionController {
+    private final AuthenticationManager authentication;
+    private final SecurityContextRepository contexts;
+    private final Clock clock;
+    private final ChangeSessionIdAuthenticationStrategy rotation = new ChangeSessionIdAuthenticationStrategy();
+    private final CsrfAuthenticationStrategy csrfAuthentication;
+
+    public SessionController(AuthenticationManager authentication, SecurityContextRepository contexts,
+            Clock clock, CsrfTokenRepository csrfTokens) {
+        this.authentication = authentication;
+        this.contexts = contexts;
+        this.clock = clock;
+        this.csrfAuthentication = new CsrfAuthenticationStrategy(csrfTokens);
+    }
+
+    @PostMapping("/api/session/login")
+    public ApiData<SessionUser> login(@RequestBody JsonNode body,
+            HttpServletRequest request, HttpServletResponse response) {
+        if (body == null || !body.isObject() || !body.path("username").isTextual()
+                || !body.path("password").isTextual()) throw invalidCredentials();
+        String username = body.get("username").textValue();
+        String password = body.get("password").textValue();
+        if (username.isBlank() || username.length() > 100 || username.indexOf('\0') >= 0
+                || password.isEmpty() || password.getBytes(StandardCharsets.UTF_8).length > 72) throw invalidCredentials();
+        var authenticated = authentication.authenticate(UsernamePasswordAuthenticationToken.unauthenticated(username, password));
+        // This controller explicitly performs the session strategies and saves the authenticated context.
+        rotation.onAuthentication(authenticated, request, response);
+        csrfAuthentication.onAuthentication(authenticated, request, response);
+        var session = request.getSession(true);
+        session.setMaxInactiveInterval(Math.toIntExact(SessionExpiryFilter.TTL.toSeconds()));
+        session.setAttribute(SessionExpiryFilter.EXPIRES_AT, clock.instant().plus(SessionExpiryFilter.TTL));
+        var context = SecurityContextHolder.createEmptyContext();
+        context.setAuthentication(authenticated);
+        SecurityContextHolder.setContext(context);
+        contexts.saveContext(context, request, response);
+        return new ApiData<>(new SessionUser(authenticated.getName()));
+    }
+
+    @GetMapping("/api/session")
+    public ApiData<SessionUser> current(Principal principal) {
+        return new ApiData<>(new SessionUser(principal.getName()));
+    }
+
+    @GetMapping("/api/session/csrf")
+    public ApiData<BrowserCsrf> csrf(CsrfToken csrf) {
+        return new ApiData<>(new BrowserCsrf(csrf.getHeaderName(), csrf.getToken()));
+    }
+
+    private static BadCredentialsException invalidCredentials() {
+        return new BadCredentialsException("Authentication required.");
+    }
+
+    public record SessionUser(String username) {}
+    public record BrowserCsrf(String headerName, String token) {}
+}
diff --git a/backend/src/main/java/dev/evolution/monitor/SessionExpiryFilter.java b/backend/src/main/java/dev/evolution/monitor/SessionExpiryFilter.java
new file mode 100644
index 0000000..9edcbf9
--- /dev/null
+++ b/backend/src/main/java/dev/evolution/monitor/SessionExpiryFilter.java
@@ -0,0 +1,34 @@
+package dev.evolution.monitor;
+
+import jakarta.servlet.FilterChain;
+import jakarta.servlet.ServletException;
+import jakarta.servlet.http.HttpServletRequest;
+import jakarta.servlet.http.HttpServletResponse;
+import java.io.IOException;
+import java.time.Clock;
+import java.time.Duration;
+import java.time.Instant;
+import org.springframework.security.web.context.HttpSessionSecurityContextRepository;
+import org.springframework.web.filter.OncePerRequestFilter;
+
+final class SessionExpiryFilter extends OncePerRequestFilter {
+    static final Duration TTL = Duration.ofHours(1);
+    static final String EXPIRES_AT = SessionExpiryFilter.class.getName() + ".expiresAt";
+    private final Clock clock;
+
+    SessionExpiryFilter(Clock clock) { this.clock = clock; }
+
+    @Override
+    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
+            throws ServletException, IOException {
+        var session = request.getSession(false);
+        if (session != null) {
+            Object deadline = session.getAttribute(EXPIRES_AT);
+            boolean authenticated = session.getAttribute(HttpSessionSecurityContextRepository.SPRING_SECURITY_CONTEXT_KEY) != null;
+            // Anonymous CSRF bootstrap sessions use only the container's bounded idle lifetime.
+            if ((deadline instanceof Instant expiry && !clock.instant().isBefore(expiry))
+                    || (authenticated && !(deadline instanceof Instant))) session.invalidate();
+        }
+        chain.doFilter(request, response);
+    }
+}
diff --git a/backend/src/main/java/dev/evolution/monitor/UserAccounts.java b/backend/src/main/java/dev/evolution/monitor/UserAccounts.java
new file mode 100644
index 0000000..957611b
--- /dev/null
+++ b/backend/src/main/java/dev/evolution/monitor/UserAccounts.java
@@ -0,0 +1,58 @@
+package dev.evolution.monitor;
+
+import jakarta.persistence.EntityManager;
+import jakarta.persistence.PersistenceContext;
+import java.nio.charset.StandardCharsets;
+import java.util.List;
+import org.springframework.security.core.userdetails.User;
+import org.springframework.security.core.userdetails.UserDetails;
+import org.springframework.security.core.userdetails.UserDetailsService;
+import org.springframework.security.core.userdetails.UsernameNotFoundException;
+import org.springframework.security.crypto.password.PasswordEncoder;
+import org.springframework.stereotype.Repository;
+import org.springframework.transaction.annotation.Transactional;
+
+@Repository
+@Transactional(readOnly = true)
+public class UserAccounts implements UserDetailsService {
+    @PersistenceContext private EntityManager entities;
+    private final PasswordEncoder passwords;
+
+    public UserAccounts(PasswordEncoder passwords) { this.passwords = passwords; }
+
+    @Override
+    public UserDetails loadUserByUsername(String username) {
+        var row = find(username).stream().findFirst()
+                .orElseThrow(() -> new UsernameNotFoundException("Authentication required."));
+        return new User(row.username(), row.passwordHash(), List.of());
+    }
+
+    @Transactional
+    public void bootstrap(String alicePassword, String bobPassword) {
+        requireBootstrapPassword(alicePassword);
+        requireBootstrapPassword(bobPassword);
+        if (alicePassword.equals(bobPassword)) throw new IllegalArgumentException("Fixture passwords must be independent.");
+        seed("alice-e04", alicePassword);
+        seed("bob-e04", bobPassword);
+    }
+
+    private void seed(String username, String password) {
+        var rows = find(username);
+        if (rows.isEmpty()) entities.persist(new UserEntity(username, passwords.encode(password)));
+        else if (!passwords.matches(password, rows.getFirst().passwordHash())) {
+            throw new IllegalStateException("Existing fixture account does not match runtime input; use a private empty schema.");
+        }
+    }
+
+    private List<UserEntity> find(String username) {
+        return entities.createQuery("select u from UserEntity u where u.username = :username", UserEntity.class)
+                .setParameter("username", username).getResultList();
+    }
+
+    private static void requireBootstrapPassword(String password) {
+        if (password == null || password.getBytes(StandardCharsets.UTF_8).length < 24
+                || password.getBytes(StandardCharsets.UTF_8).length > 72) {
+            throw new IllegalArgumentException("Supply each fixture password through runtime input (24–72 UTF-8 bytes).");
+        }
+    }
+}
diff --git a/backend/src/main/java/dev/evolution/monitor/UserEntity.java b/backend/src/main/java/dev/evolution/monitor/UserEntity.java
new file mode 100644
index 0000000..1665ee3
--- /dev/null
+++ b/backend/src/main/java/dev/evolution/monitor/UserEntity.java
@@ -0,0 +1,27 @@
+package dev.evolution.monitor;
+
+import jakarta.persistence.Column;
+import jakarta.persistence.Entity;
+import jakarta.persistence.Id;
+import jakarta.persistence.Table;
+import java.util.UUID;
+
+@Entity
+@Table(name = "users")
+public class UserEntity {
+    @Id @Column(name = "id", nullable = false) private UUID id;
+    @Column(name = "username", nullable = false, length = 100) private String username;
+    @Column(name = "password_hash", nullable = false, length = 60) private String passwordHash;
+
+    protected UserEntity() {}
+
+    UserEntity(String username, String passwordHash) {
+        this.id = UUID.randomUUID();
+        this.username = username;
+        this.passwordHash = passwordHash;
+    }
+
+    String username() { return username; }
+    String passwordHash() { return passwordHash; }
+    // No entity serialization or generated toString: the hash must not enter API payloads or logs.
+}
diff --git a/backend/src/main/resources/application.properties b/backend/src/main/resources/application.properties
index 3c72628..cce4f2a 100644
--- a/backend/src/main/resources/application.properties
+++ b/backend/src/main/resources/application.properties
@@ -13,3 +13,11 @@ spring.jpa.hibernate.ddl-auto=validate
 spring.jpa.open-in-view=false
 spring.jpa.properties.hibernate.default_schema=${DB_SCHEMA:public}
 spring.jpa.properties.hibernate.jdbc.time_zone=UTC
+server.servlet.session.timeout=3600s
+server.servlet.session.persistent=false
+server.servlet.session.tracking-modes=cookie
+server.servlet.session.cookie.name=WSESESSION
+server.servlet.session.cookie.http-only=true
+server.servlet.session.cookie.same-site=lax
+# The product remains loopback HTTP. Enable for a TLS deployment; never put tokens in JavaScript.
+server.servlet.session.cookie.secure=${SESSION_COOKIE_SECURE:false}
diff --git a/backend/src/main/resources/db/migration/V3__create_users.sql b/backend/src/main/resources/db/migration/V3__create_users.sql
new file mode 100644
index 0000000..444de52
--- /dev/null
+++ b/backend/src/main/resources/db/migration/V3__create_users.sql
@@ -0,0 +1,5 @@
+CREATE TABLE users (
+    id uuid PRIMARY KEY,
+    username varchar(100) NOT NULL UNIQUE,
+    password_hash varchar(60) NOT NULL
+);
diff --git a/backend/src/test/java/dev/evolution/monitor/MonitorFunctionalTest.java b/backend/src/test/java/dev/evolution/monitor/MonitorFunctionalTest.java
index 349fd94..d5d7fdc 100644
--- a/backend/src/test/java/dev/evolution/monitor/MonitorFunctionalTest.java
+++ b/backend/src/test/java/dev/evolution/monitor/MonitorFunctionalTest.java
@@ -43,9 +43,18 @@ class MonitorFunctionalTest {
     @Autowired ObjectMapper json;
     @MockitoSpyBean CheckRunner checks;
 
+    @BeforeAll
+    static void authenticateProductFixture(@Autowired TestRestTemplate api, @Autowired UserAccounts accounts) {
+        String alice = SessionClient.password();
+        accounts.bootstrap(alice, SessionClient.password());
+        var session = new SessionClient(api);
+        assertEquals(200, session.login("alice-e04", alice).getStatusCode().value());
+        session.authorizeExistingProductRequests();
+    }
+
     @DynamicPropertySource
     static void database(DynamicPropertyRegistry properties) {
-        TestDatabase.configure(properties, "e03_functional");
+        TestDatabase.configure(properties, "e04_functional");
     }
 
     @BeforeAll
@@ -89,7 +98,7 @@ class MonitorFunctionalTest {
     static void stopFixture() {
         if (fixture != null) fixture.stop(0);
         if (forbidden != null) forbidden.stop(0);
-        TestDatabase.drop("e03_functional");
+        TestDatabase.drop("e04_functional");
     }
 
     @Test
diff --git a/backend/src/test/java/dev/evolution/monitor/PostgresPersistenceTest.java b/backend/src/test/java/dev/evolution/monitor/PostgresPersistenceTest.java
index da64061..c8c9f29 100644
--- a/backend/src/test/java/dev/evolution/monitor/PostgresPersistenceTest.java
+++ b/backend/src/test/java/dev/evolution/monitor/PostgresPersistenceTest.java
@@ -35,23 +35,24 @@ class PostgresPersistenceTest {
 
     @Test
     void freshMigrationChainRepeatsWithoutDdlAndUpgradesAnIndependentV1Schema() {
-        String schema = "e03_migrations";
+        String schema = "e04_migrations";
         TestDatabase.reset(schema);
         try {
             var fresh = TestDatabase.migration(schema).load();
-            assertEquals(2, fresh.migrate().migrationsExecuted);
-            assertEquals(List.of("1", "2"), java.util.Arrays.stream(fresh.info().applied())
+            assertEquals(3, fresh.migrate().migrationsExecuted);
+            assertEquals(List.of("1", "2", "3"), java.util.Arrays.stream(fresh.info().applied())
                     .map(migration -> migration.getVersion().toString()).toList());
             fresh.validate();
             assertEquals(0, fresh.migrate().migrationsExecuted);
-            assertEquals(2, fresh.info().applied().length);
+            assertEquals(3, fresh.info().applied().length);
 
             TestDatabase.reset(schema);
             assertEquals(1, TestDatabase.migration(schema).target("1").load().migrate().migrationsExecuted);
+            assertEquals(1, TestDatabase.migration(schema).target("2").load().migrate().migrationsExecuted);
             var upgrade = TestDatabase.migration(schema).load();
             assertEquals(1, upgrade.migrate().migrationsExecuted);
             upgrade.validate();
-            assertEquals(2, upgrade.info().applied().length);
+            assertEquals(3, upgrade.info().applied().length);
             assertEquals(0, upgrade.migrate().migrationsExecuted);
         } finally {
             TestDatabase.drop(schema);
@@ -60,7 +61,7 @@ class PostgresPersistenceTest {
 
     @Test
     void mappingAndEveryHistoricalRunSurviveAClosedApplicationContext() throws Exception {
-        String schema = "e03_mapping";
+        String schema = "e04_mapping";
         TestDatabase.reset(schema);
         SqlEvidence.events.clear();
         var monitor = new Monitor(MONITOR_ID, "Mapping fixture", "http://127.0.0.1:4321/ok", 1, false,
@@ -120,14 +121,14 @@ class PostgresPersistenceTest {
                 // Read the actual PostgreSQL representations in a non-UTC session.
                 try (var connection = TestDatabase.connect(); var statement = connection.createStatement()) {
                     statement.execute("SET TIME ZONE 'Asia/Seoul'");
-                    try (var rows = statement.executeQuery("SELECT interval_seconds, enabled, created_at FROM e03_mapping.monitors")) {
+                    try (var rows = statement.executeQuery("SELECT interval_seconds, enabled, created_at FROM e04_mapping.monitors")) {
                         assertTrue(rows.next());
                         assertEquals(Integer.valueOf(1), rows.getObject("interval_seconds"));
                         assertEquals(Boolean.FALSE, rows.getObject("enabled"));
                         assertEquals(DATABASE_TIME, rows.getObject("created_at", OffsetDateTime.class).toInstant());
                         assertFalse(rows.next());
                     }
-                    try (var rows = statement.executeQuery("SELECT latency_ms, failure_reason FROM e03_mapping.check_runs WHERE http_status = 200")) {
+                    try (var rows = statement.executeQuery("SELECT latency_ms, failure_reason FROM e04_mapping.check_runs WHERE http_status = 200")) {
                         assertTrue(rows.next());
                         assertEquals(Long.valueOf(0), rows.getObject("latency_ms"));
                         assertNull(rows.getObject("failure_reason"));
@@ -143,19 +144,19 @@ class PostgresPersistenceTest {
                 assertEquals(404, assertThrows(ResponseStatusException.class,
                         () -> store.history(MONITOR_ID)).getStatusCode().value());
                 try (var connection = TestDatabase.connect(); var query = connection.createStatement();
-                        var rows = query.executeQuery("SELECT (SELECT count(*) FROM e03_mapping.monitors) AS monitors, "
-                                + "(SELECT count(*) FROM e03_mapping.check_runs) AS checks")) {
+                        var rows = query.executeQuery("SELECT (SELECT count(*) FROM e04_mapping.monitors) AS monitors, "
+                                + "(SELECT count(*) FROM e04_mapping.check_runs) AS checks")) {
                     assertTrue(rows.next());
                     assertEquals(0, rows.getInt("monitors"));
                     assertEquals(0, rows.getInt("checks"), "Database cascade must leave no historical children");
                 }
             }
             assertTrue(SqlEvidence.events.stream().allMatch(SqlEvent::transaction), "All ORM SQL must run in a real transaction");
-            for (String operation : List.of("insert into e03_mapping.monitors", "insert into e03_mapping.check_runs",
-                    "update e03_mapping.monitors", "delete from e03_mapping.monitors")) {
+            for (String operation : List.of("insert into e04_mapping.monitors", "insert into e04_mapping.check_runs",
+                    "update e04_mapping.monitors", "delete from e04_mapping.monitors")) {
                 assertTrue(SqlEvidence.events.stream().anyMatch(event -> event.sql().startsWith(operation)), operation);
             }
-            assertTrue(SqlEvidence.events.stream().anyMatch(event -> event.readOnly() && event.sql().contains("from e03_mapping.check_runs")));
+            assertTrue(SqlEvidence.events.stream().anyMatch(event -> event.readOnly() && event.sql().contains("from e04_mapping.check_runs")));
             Files.createDirectories(Path.of("target"));
             Files.writeString(Path.of("target/e03-generated-sql.txt"), String.join("\n",
                     SqlEvidence.events.stream().map(Object::toString).toList()) + "\n");
@@ -166,14 +167,14 @@ class PostgresPersistenceTest {
 
     @Test
     void startupRejectsAMissingMappedColumnBeforeWebServerReady() {
-        assertStartupFailure("e03_missing_column", "ALTER TABLE e03_missing_column.monitors DROP COLUMN name",
+        assertStartupFailure("e04_missing_column", "ALTER TABLE e04_missing_column.monitors DROP COLUMN name",
                 "missing column [name]");
     }
 
     @Test
     void startupRejectsAnExtraRequiredInsertColumnBeforeWebServerReady() {
-        assertStartupFailure("e03_extra_required",
-                "ALTER TABLE e03_extra_required.monitors ADD COLUMN unmapped_required text NOT NULL",
+        assertStartupFailure("e04_extra_required",
+                "ALTER TABLE e04_extra_required.monitors ADD COLUMN unmapped_required text NOT NULL",
                 "Unmapped required column: monitors.unmapped_required");
     }
 
@@ -182,7 +183,7 @@ class PostgresPersistenceTest {
         var ready = new AtomicBoolean();
         var unexpectedContext = new AtomicReference<ConfigurableApplicationContext>();
         try {
-            assertEquals(2, TestDatabase.migration(schema).load().migrate().migrationsExecuted);
+            assertEquals(3, TestDatabase.migration(schema).load().migrate().migrationsExecuted);
             TestDatabase.execute(ddl);
             ApplicationListener<WebServerInitializedEvent> readyListener = event -> ready.set(true);
             var failure = assertThrows(RuntimeException.class, () -> unexpectedContext.set(
diff --git a/backend/src/test/java/dev/evolution/monitor/SessionAuthenticationTest.java b/backend/src/test/java/dev/evolution/monitor/SessionAuthenticationTest.java
new file mode 100644
index 0000000..ba78beb
--- /dev/null
+++ b/backend/src/test/java/dev/evolution/monitor/SessionAuthenticationTest.java
@@ -0,0 +1,191 @@
+package dev.evolution.monitor;
+
+import static org.junit.jupiter.api.Assertions.*;
+
+import com.fasterxml.jackson.databind.JsonNode;
+import com.fasterxml.jackson.databind.ObjectMapper;
+import jakarta.servlet.ServletContext;
+import jakarta.servlet.SessionTrackingMode;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.time.Clock;
+import java.time.Instant;
+import java.time.ZoneId;
+import java.time.ZoneOffset;
+import java.util.LinkedHashMap;
+import java.util.List;
+import java.util.Map;
+import java.util.Set;
+import org.junit.jupiter.api.AfterAll;
+import org.junit.jupiter.api.BeforeAll;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.boot.test.context.TestConfiguration;
+import org.springframework.boot.test.web.client.TestRestTemplate;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Primary;
+import org.springframework.http.HttpHeaders;
+import org.springframework.http.HttpMethod;
+import org.springframework.http.ResponseEntity;
+import org.springframework.test.annotation.DirtiesContext;
+import org.springframework.test.context.DynamicPropertyRegistry;
+import org.springframework.test.context.DynamicPropertySource;
+
+@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
+@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_CLASS)
+class SessionAuthenticationTest {
+    private static final Instant START = Instant.parse("2026-08-28T01:00:00.000Z");
+    private static final String ALICE = SessionClient.password();
+    private static final String BOB = SessionClient.password();
+    @Autowired TestRestTemplate api;
+    @Autowired MutableClock clock;
+    @Autowired ServletContext servlet;
+    @Autowired ObjectMapper json;
+
+    @DynamicPropertySource
+    static void database(DynamicPropertyRegistry properties) { TestDatabase.configure(properties, "e04_session"); }
+
+    @BeforeAll
+    static void users(@Autowired UserAccounts accounts) { accounts.bootstrap(ALICE, BOB); }
+
+    @AfterAll
+    static void cleanup() { TestDatabase.drop("e04_session"); }
+
+    @Test
+    void anonymousCsrfBootstrapIsNotAnAuthenticatedLogout() {
+        unauthenticated(new SessionClient(api).mutate("/api/session/logout", HttpMethod.POST, null));
+    }
+
+    @Test
+    void fixedTwoUserLifecycleRotatesRevokesAndExpiresAtExactlyOneHour() throws Exception {
+        clock.now = START;
+        Map<String, Object> evidence = new LinkedHashMap<>();
+        var alice = new SessionClient(api);
+        unauthenticated(alice.get("/api/monitors"));
+        evidence.put("anonymousStatus", 401);
+
+        alice.csrf();
+        var beforeLogin = alice.copyCredential();
+        var login = alice.login("alice-e04", ALICE);
+        assertEquals(200, login.getStatusCode().value());
+        assertEquals(Set.of("data"), keys(login.getBody()));
+        assertEquals(Set.of("username"), keys(login.getBody().get("data")));
+        assertEquals("alice-e04", login.getBody().at("/data/username").textValue());
+        assertFalse(alice.sameCredential(beforeLogin), "Login must rotate even the anonymous CSRF session");
+        String cookie = login.getHeaders().getFirst(HttpHeaders.SET_COOKIE);
+        assertTrue(cookie != null && cookie.contains("HttpOnly") && cookie.contains("SameSite=Lax"));
+        assertFalse(cookie.contains("Secure"), "This fixture uses loopback HTTP");
+        assertEquals(Set.of(SessionTrackingMode.COOKIE), servlet.getEffectiveSessionTrackingModes());
+        evidence.put("httpOnlyCookie", true);
+        evidence.put("sameSiteLax", true);
+        evidence.put("cookieOnlyTracking", true);
+        evidence.put("anonymousSessionRotated", true);
+
+        var invalid = new SessionClient(api);
+        JsonNode wrong = unauthenticated(invalid.login("alice-e04", SessionClient.password()));
+        JsonNode unknown = unauthenticated(invalid.login("unknown-e04", SessionClient.password()));
+        assertEquals(wrong, unknown, "Login failures must not disclose account existence");
+        unauthenticated(invalid.mutate("/api/session/login", HttpMethod.POST, Map.of()));
+        unauthenticated(invalid.login("alice-e04", SessionClient.password().repeat(2)));
+        evidence.put("invalidMissingAndOversizeLoginStatus", 401);
+        evidence.put("unknownAndWrongPasswordSameEnvelope", true);
+
+        assertEquals(200, alice.get("/api/monitors").getStatusCode().value());
+        var firstCredential = alice.copyCredential();
+        assertEquals(200, alice.login("alice-e04", ALICE).getStatusCode().value());
+        assertFalse(alice.sameCredential(firstCredential), "Reauthentication must change the actual container identifier");
+        unauthenticated(firstCredential.get("/api/monitors"));
+        assertEquals(200, alice.get("/api/monitors").getStatusCode().value());
+        evidence.put("reloginRotated", true);
+        evidence.put("oldIdentifierStatus", 401);
+
+        // Preserve the framework default; no E05 ownership or cross-origin matrix is introduced.
+        var missingToken = alice.send("/api/monitors", HttpMethod.POST, Map.of(), false);
+        assertEquals(403, missingToken.getStatusCode().value());
+        assertEquals("FORBIDDEN", missingToken.getBody().at("/error/code").textValue());
+        evidence.put("frameworkCsrfStillEnforced", true);
+
+        var revoked = alice.copyCredential();
+        var logout = alice.mutate("/api/session/logout", HttpMethod.POST, null);
+        assertEquals(200, logout.getStatusCode().value());
+        assertTrue(logout.getBody().get("data").isNull());
+        assertTrue(logout.getHeaders().getOrEmpty(HttpHeaders.SET_COOKIE).stream().anyMatch(value ->
+                value.startsWith(AuthenticationConfig.COOKIE_NAME + "=") && value.contains("Max-Age=0")));
+        unauthenticated(revoked.get("/api/monitors"));
+        evidence.put("logoutCookieCleared", true);
+        evidence.put("revokedStatus", 401);
+
+        var expiring = new SessionClient(api);
+        assertEquals(200, expiring.login("alice-e04", ALICE).getStatusCode().value());
+        assertEquals(3_600_000, SessionExpiryFilter.TTL.toMillis());
+        clock.now = Instant.parse("2026-08-28T01:59:59.999Z");
+        assertEquals(200, expiring.get("/api/monitors").getStatusCode().value());
+        clock.now = Instant.parse("2026-08-28T02:00:00.000Z");
+        unauthenticated(expiring.get("/api/monitors"));
+        evidence.put("ttlMs", 3_600_000);
+        evidence.put("fixedStart", START.toString());
+        evidence.put("beforeExpiryStatus", 200);
+        evidence.put("exactExpiryStatus", 401);
+
+        var bob = new SessionClient(api);
+        assertEquals(200, bob.login("bob-e04", BOB).getStatusCode().value());
+        assertEquals("bob-e04", bob.get("/api/session").getBody().at("/data/username").textValue());
+        assertEquals(200, bob.get("/api/monitors").getStatusCode().value());
+        evidence.put("bobIndependentSessionStatus", 200);
+
+        Files.createDirectories(Path.of("target"));
+        Files.writeString(Path.of("target/e04-session-evidence.json"), json.writerWithDefaultPrettyPrinter().writeValueAsString(evidence) + "\n");
+    }
+
+    @Test
+    void everyProtectedMethodRejectsMissingAndInvalidCredentialsBeforeProductCode() {
+        String id = "00000000-0000-4000-8000-000000000000";
+        var paths = List.of(
+                Map.entry(HttpMethod.GET, "/api/monitors"),
+                Map.entry(HttpMethod.POST, "/api/monitors"),
+                Map.entry(HttpMethod.GET, "/api/monitors/" + id),
+                Map.entry(HttpMethod.PUT, "/api/monitors/" + id),
+                Map.entry(HttpMethod.DELETE, "/api/monitors/" + id),
+                Map.entry(HttpMethod.GET, "/api/monitors/" + id + "/checks"),
+                Map.entry(HttpMethod.POST, "/api/monitors/" + id + "/checks"),
+                Map.entry(HttpMethod.GET, "/api/monitors/" + id + "/checks/" + id),
+                Map.entry(HttpMethod.POST, "/api/monitors/not-a-uuid/checks"),
+                Map.entry(HttpMethod.GET, "/api/session"));
+        for (var route : paths) {
+            var anonymous = new SessionClient(api);
+            unauthenticated(anonymous.send(route.getValue(), route.getKey(), null, false));
+            var invalid = new SessionClient(api);
+            invalid.useInvalidCredential();
+            unauthenticated(invalid.send(route.getValue(), route.getKey(), null, false));
+        }
+        unauthenticated(new SessionClient(api).send("/api/monitors", HttpMethod.POST, "{", false));
+    }
+
+    private static JsonNode unauthenticated(ResponseEntity<JsonNode> response) {
+        assertEquals(401, response.getStatusCode().value());
+        assertEquals(Set.of("error"), keys(response.getBody()));
+        assertEquals(Set.of("code", "message"), keys(response.getBody().get("error")));
+        assertEquals("UNAUTHENTICATED", response.getBody().at("/error/code").textValue());
+        assertEquals("Authentication required.", response.getBody().at("/error/message").textValue());
+        return response.getBody();
+    }
+
+    private static Set<String> keys(JsonNode value) {
+        var names = new java.util.HashSet<String>();
+        value.fieldNames().forEachRemaining(names::add);
+        return names;
+    }
+
+    static class MutableClock extends Clock {
+        volatile Instant now = START;
+        @Override public ZoneId getZone() { return ZoneOffset.UTC; }
+        @Override public Clock withZone(ZoneId zone) { return this; }
+        @Override public Instant instant() { return now; }
+    }
+
+    @TestConfiguration
+    static class TimeFixture {
+        @Bean @Primary MutableClock fixedSessionClock() { return new MutableClock(); }
+    }
+}
diff --git a/backend/src/test/java/dev/evolution/monitor/SessionClient.java b/backend/src/test/java/dev/evolution/monitor/SessionClient.java
new file mode 100644
index 0000000..27600e2
--- /dev/null
+++ b/backend/src/test/java/dev/evolution/monitor/SessionClient.java
@@ -0,0 +1,82 @@
+package dev.evolution.monitor;
+
+import static org.junit.jupiter.api.Assertions.*;
+
+import com.fasterxml.jackson.databind.JsonNode;
+import java.security.SecureRandom;
+import java.util.Base64;
+import java.util.Map;
+import org.springframework.boot.test.web.client.TestRestTemplate;
+import org.springframework.http.HttpEntity;
+import org.springframework.http.HttpHeaders;
+import org.springframework.http.HttpMethod;
+import org.springframework.http.MediaType;
+import org.springframework.http.ResponseEntity;
+
+// All sensitive values stay in memory; assertions and evidence use only status/flags.
+final class SessionClient {
+    private final TestRestTemplate api;
+    private String cookie;
+    private String csrfHeader;
+    private String csrfToken;
+
+    SessionClient(TestRestTemplate api) { this.api = api; }
+
+    static String password() {
+        byte[] bytes = new byte[32];
+        new SecureRandom().nextBytes(bytes);
+        return Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
+    }
+
+    SessionClient copyCredential() {
+        var copy = new SessionClient(api);
+        copy.cookie = cookie;
+        return copy;
+    }
+
+    boolean sameCredential(SessionClient other) { return cookie != null && cookie.equals(other.cookie); }
+
+    void useInvalidCredential() { cookie = AuthenticationConfig.COOKIE_NAME + "=" + password(); }
+
+    ResponseEntity<JsonNode> login(String username, String password) {
+        return mutate("/api/session/login", HttpMethod.POST, Map.of("username", username, "password", password));
+    }
+
+    ResponseEntity<JsonNode> get(String path) { return send(path, HttpMethod.GET, null, false); }
+
+    void csrf() {
+        var response = get("/api/session/csrf");
+        assertEquals(200, response.getStatusCode().value());
+        csrfHeader = response.getBody().at("/data/headerName").textValue();
+        csrfToken = response.getBody().at("/data/token").textValue();
+        assertTrue(csrfHeader != null && csrfToken != null, "CSRF bootstrap must supply a header and token");
+    }
+
+    ResponseEntity<JsonNode> mutate(String path, HttpMethod method, Object body) {
+        csrf();
+        return send(path, method, body, true);
+    }
+
+    ResponseEntity<JsonNode> send(String path, HttpMethod method, Object body, boolean includeCsrf) {
+        var headers = new HttpHeaders();
+        if (cookie != null) headers.set(HttpHeaders.COOKIE, cookie);
+        if (includeCsrf) headers.set(csrfHeader, csrfToken);
+        if (body != null) headers.setContentType(MediaType.APPLICATION_JSON);
+        var response = api.exchange(path, method, new HttpEntity<>(body, headers), JsonNode.class);
+        for (String value : response.getHeaders().getOrEmpty(HttpHeaders.SET_COOKIE)) {
+            if (value.startsWith(AuthenticationConfig.COOKIE_NAME + "=")) cookie = value.split(";", 2)[0];
+        }
+        return response;
+    }
+
+    void authorizeExistingProductRequests() {
+        csrf();
+        api.getRestTemplate().getInterceptors().add((request, body, execution) -> {
+            request.getHeaders().set(HttpHeaders.COOKIE, cookie);
+            if (request.getMethod() != HttpMethod.GET && request.getMethod() != HttpMethod.HEAD) {
+                request.getHeaders().set(csrfHeader, csrfToken);
+            }
+            return execution.execute(request, body);
+        });
+    }
+}
diff --git a/backend/src/test/java/dev/evolution/monitor/TestDatabase.java b/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
index b87e21e..4ea16ee 100644
--- a/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
+++ b/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
@@ -9,8 +9,9 @@ import org.flywaydb.core.api.configuration.FluentConfiguration;
 import org.springframework.test.context.DynamicPropertyRegistry;
 
 final class TestDatabase {
-    private static final Set<String> SCHEMAS = Set.of("e03_functional", "e03_migrations", "e03_mapping",
-            "e03_missing_column", "e03_extra_required");
+    private static final Set<String> SCHEMAS = Set.of("e04_functional", "e04_migrations", "e04_mapping",
+            "e04_missing_column", "e04_extra_required", "e04_session", "e04_users",
+            "e04_missing_user_column", "e04_extra_user_required");
 
     static Connection connect() throws SQLException {
         return DriverManager.getConnection(System.getenv().getOrDefault("DB_URL", "jdbc:postgresql://127.0.0.1:15432/monitor"),
diff --git a/backend/src/test/java/dev/evolution/monitor/UserAccountPersistenceTest.java b/backend/src/test/java/dev/evolution/monitor/UserAccountPersistenceTest.java
new file mode 100644
index 0000000..7d33ac4
--- /dev/null
+++ b/backend/src/test/java/dev/evolution/monitor/UserAccountPersistenceTest.java
@@ -0,0 +1,161 @@
+package dev.evolution.monitor;
+
+import static org.junit.jupiter.api.Assertions.*;
+
+import jakarta.persistence.EntityManagerFactory;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.util.List;
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
+import org.springframework.security.authentication.AuthenticationManager;
+import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
+import org.springframework.security.core.userdetails.UserDetails;
+import org.springframework.security.crypto.password.PasswordEncoder;
+import org.springframework.transaction.PlatformTransactionManager;
+import org.springframework.transaction.support.TransactionSynchronizationManager;
+import org.springframework.transaction.support.TransactionTemplate;
+
+class UserAccountPersistenceTest {
+    @Test
+    void v2UpgradeAndBootstrapUseRealTransactionsAndSaltedPersistentPasswords() throws Exception {
+        String schema = "e04_users";
+        String alice = SessionClient.password();
+        String bob = SessionClient.password();
+        TestDatabase.reset(schema);
+        SqlEvidence.events.clear();
+        try {
+            assertEquals(2, TestDatabase.migration(schema).target("2").load().migrate().migrationsExecuted);
+            var latest = TestDatabase.migration(schema).load();
+            assertEquals(1, latest.migrate().migrationsExecuted);
+            assertEquals(3, latest.info().applied().length);
+            assertEquals(0, latest.migrate().migrationsExecuted);
+
+            try (var first = start(schema)) {
+                var accounts = first.getBean(UserAccounts.class);
+                assertTrue(AopUtils.isAopProxy(accounts));
+                var transactions = new TransactionTemplate(first.getBean(PlatformTransactionManager.class));
+                var entities = SharedEntityManagerCreator.createSharedEntityManager(first.getBean(EntityManagerFactory.class));
+                transactions.executeWithoutResult(status -> {
+                    accounts.bootstrap(alice, bob);
+                    entities.flush();
+                    status.setRollbackOnly();
+                });
+                assertEquals(0, count());
+                accounts.bootstrap(alice, bob);
+                assertEquals(2, count());
+                String before = accounts.loadUserByUsername("alice-e04").getPassword();
+                accounts.bootstrap(alice, bob);
+                assertEquals(2, count());
+                assertTrue(before.equals(accounts.loadUserByUsername("alice-e04").getPassword()),
+                        "Idempotent bootstrap must not rewrite a matching account");
+            }
+
+            try (var fresh = start(schema)) {
+                var accounts = fresh.getBean(UserAccounts.class);
+                var passwords = fresh.getBean(PasswordEncoder.class);
+                var manager = fresh.getBean(AuthenticationManager.class);
+                for (var pair : List.of(new String[]{"alice-e04", alice}, new String[]{"bob-e04", bob})) {
+                    String stored = accounts.loadUserByUsername(pair[0]).getPassword();
+                    assertFalse(stored.equals(pair[1]), "The database must not contain plaintext");
+                    assertTrue(stored.startsWith("$2a$10$") && stored.length() == 60, "Stored representation is bcrypt cost 10");
+                    assertTrue(passwords.matches(pair[1], stored));
+                    String anotherHash = passwords.encode(pair[1]);
+                    assertFalse(stored.substring(0, 29).equals(anotherHash.substring(0, 29)), "Independent encodes need independent salts");
+                    var authenticated = manager.authenticate(UsernamePasswordAuthenticationToken.unauthenticated(pair[0], pair[1]));
+                    assertTrue(authenticated.isAuthenticated());
+                    assertNull(authenticated.getCredentials());
+                    assertTrue(authenticated.getPrincipal() instanceof UserDetails user && user.getPassword() == null,
+                            "Neither plaintext nor a password hash belongs in the stored security principal");
+                }
+            }
+
+            assertTrue(SqlEvidence.events.stream().allMatch(Event::transaction));
+            assertTrue(SqlEvidence.events.stream().anyMatch(event -> !event.readOnly() && event.sql().startsWith("insert into e04_users.users")));
+            assertTrue(SqlEvidence.events.stream().anyMatch(event -> event.readOnly() && event.sql().contains("from e04_users.users")));
+            Files.createDirectories(Path.of("target"));
+            Files.writeString(Path.of("target/e04-user-sql.txt"), String.join("\n",
+                    SqlEvidence.events.stream().map(Object::toString).toList()) + "\n");
+            Files.writeString(Path.of("target/e04-user-evidence.json"), """
+                    {"users":2,"bcryptCost":10,"plaintextAtRest":false,"independentSalts":true,
+                     "bootstrapRollback":true,"idempotentBootstrap":true,"reopenedContextAuthentication":true,
+                     "authenticatedCredentialsErased":true,"allSqlTransactional":true,"v2UpgradeApplied":1,"repeatApplied":0}
+                    """);
+        } finally {
+            TestDatabase.drop(schema);
+        }
+    }
+
+    @Test
+    void startupRejectsMissingAccountHashColumnBeforeReadiness() {
+        startupFailure("e04_missing_user_column", "ALTER TABLE e04_missing_user_column.users DROP COLUMN password_hash",
+                "missing column [password_hash]");
+    }
+
+    @Test
+    void startupRejectsExtraRequiredAccountColumnBeforeReadiness() {
+        startupFailure("e04_extra_user_required", "ALTER TABLE e04_extra_user_required.users ADD COLUMN unmapped_required text NOT NULL",
+                "Unmapped required column: users.unmapped_required");
+    }
+
+    private static int count() throws Exception {
+        try (var connection = TestDatabase.connect(); var query = connection.createStatement();
+                var rows = query.executeQuery("SELECT count(*) FROM e04_users.users")) {
+            assertTrue(rows.next());
+            return rows.getInt(1);
+        }
+    }
+
+    private static void startupFailure(String schema, String ddl, String expectedMessage) {
+        TestDatabase.reset(schema);
+        var ready = new AtomicBoolean();
+        var unexpected = new AtomicReference<ConfigurableApplicationContext>();
+        try {
+            assertEquals(3, TestDatabase.migration(schema).load().migrate().migrationsExecuted);
+            TestDatabase.execute(ddl);
+            ApplicationListener<WebServerInitializedEvent> listener = event -> ready.set(true);
+            RuntimeException failure = assertThrows(RuntimeException.class, () -> unexpected.set(
+                    new SpringApplicationBuilder(MonitorApplication.class).web(WebApplicationType.SERVLET)
+                            .listeners(listener).run(arguments(schema))));
+            var messages = new StringBuilder();
+            for (Throwable cause = failure; cause != null; cause = cause.getCause()) messages.append(cause.getMessage()).append('\n');
+            assertTrue(messages.toString().contains(expectedMessage), "The account schema must be rejected for the expected reason");
+            assertFalse(ready.get());
+        } finally {
+            if (unexpected.get() != null) unexpected.get().close();
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
+    record Event(String sql, boolean transaction, boolean readOnly) {}
+
+    public static class SqlEvidence implements StatementInspector {
+        static final List<Event> events = new CopyOnWriteArrayList<>();
+        @Override public String inspect(String sql) {
+            if (sql.contains(".users")) events.add(new Event(sql, TransactionSynchronizationManager.isActualTransactionActive(),
+                    TransactionSynchronizationManager.isCurrentTransactionReadOnly()));
+            return sql;
+        }
+    }
+}


