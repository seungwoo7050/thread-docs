## `feat: adopt phase-1 operational boundaries and containers`

diff --git a/.dockerignore b/.dockerignore
new file mode 100644
index 0000000..9f342c8
--- /dev/null
+++ b/.dockerignore
@@ -0,0 +1,10 @@
+**
+!package.json
+!package-lock.json
+!next.config.mjs
+!tsconfig.json
+!app/
+!app/**
+!backend/
+!backend/target/
+!backend/target/monitor-api-0.0.1.jar
diff --git a/.github/workflows/ci.yml b/.github/workflows/ci.yml
index dc35b36..5cae676 100644
--- a/.github/workflows/ci.yml
+++ b/.github/workflows/ci.yml
@@ -1,11 +1,16 @@
-name: E04 verification
+name: Phase-1 verification
 on: [push, pull_request]
 permissions:
   contents: read
 jobs:
-  unit-functional-browser:
+  verification:
+    name: ${{ matrix.gate }}
     runs-on: ubuntu-24.04
     timeout-minutes: 15
+    strategy:
+      fail-fast: false
+      matrix:
+        gate: [unit, integration, browser, container]
     env:
       NEXT_TELEMETRY_DISABLED: '1'
     steps:
@@ -27,9 +32,46 @@ jobs:
           echo "$RUNNER_TEMP/apache-maven-3.9.11/bin" >> "$GITHUB_PATH"
       - run: npm install --global npm@11.17.0
       - run: npm ci
+        if: matrix.gate == 'browser' || matrix.gate == 'container'
       - run: npx playwright install --with-deps chromium
-      - name: Unit, PostgreSQL, functional, restart, type, build and browser gates
-        run: npm run verify
-      - name: Remove only the isolated CI database project
+        if: matrix.gate == 'browser' || matrix.gate == 'container'
+      - name: Unit gate
+        if: matrix.gate == 'unit'
+        run: mvn -B -ntp -f backend/pom.xml -Dtest=CheckRunnerTest,ApiErrorBoundaryTest,HttpObservationsTest test
+      - name: Start the isolated PostgreSQL authority
+        if: matrix.gate != 'unit'
+        run: npm run db:up
+      - name: PostgreSQL integration and persistence gate
+        if: matrix.gate == 'integration'
+        shell: bash
+        run: |
+          mvn -B -ntp -f backend/pom.xml '-Dtest=!CheckRunnerTest,!ApiErrorBoundaryTest,!HttpObservationsTest,!WorkerRecoveryTest,!HistoryQueryPlanTest' package
+          node scripts/persistence-isolation.mjs
+          node scripts/database.mjs reset e04_restart
+          node scripts/persistence-scenario.mjs fixed
+      - name: Browser production artifact and regression gate
+        if: matrix.gate == 'browser'
+        shell: bash
+        run: |
+          npm run api:package
+          npm run typecheck
+          npm run build
+          npm run test:worker
+          npm run test:e2e
+      - name: Build the pinned production containers
+        if: matrix.gate == 'container'
+        shell: bash
+        run: |
+          npm run api:package
+          DB_SCHEMA=e24_container docker compose -p wse-industry-e24 -f compose.production.yaml build api frontend
+      - name: Explicit container readiness, metrics and browser gate
+        if: matrix.gate == 'container'
+        run: npm run test:container -- ci
+      - name: Remove only owned integration/browser schemas
         if: always()
-        run: npm run db:destroy
+        shell: bash
+        run: |
+          if [[ '${{ matrix.gate }}' == integration || '${{ matrix.gate }}' == browser ]]; then
+            node scripts/database.mjs drop e04_restart
+            node scripts/database.mjs drop e04_browser
+          fi
diff --git a/Dockerfile.api b/Dockerfile.api
new file mode 100644
index 0000000..3c80f1e
--- /dev/null
+++ b/Dockerfile.api
@@ -0,0 +1,7 @@
+FROM eclipse-temurin:21.0.7_6-jre-jammy@sha256:c4e6542e774de504da9b4729ff8d761287c965c1d788528ca78da30024efdb23
+WORKDIR /app
+# The Maven enforcer pins the host/CI artifact build to Java21.0.7 and Maven3.9.11.
+COPY --chown=10001:10001 backend/target/monitor-api-0.0.1.jar /app/app.jar
+USER 10001:10001
+EXPOSE 4322 4324
+ENTRYPOINT ["java", "-jar", "/app/app.jar"]
diff --git a/Dockerfile.frontend b/Dockerfile.frontend
new file mode 100644
index 0000000..0e9d71b
--- /dev/null
+++ b/Dockerfile.frontend
@@ -0,0 +1,20 @@
+FROM node:24.19.0-bookworm-slim@sha256:a9f5f7c91a432850b2a8a7797adf5eadb6c733ceed61167806cee7ea7fbc29df AS build
+WORKDIR /app
+RUN npm install --global npm@11.17.0
+COPY package.json package-lock.json ./
+RUN npm ci --no-audit --no-fund
+COPY next.config.mjs tsconfig.json ./
+COPY app ./app
+ARG API_ORIGIN=http://api:4322
+ENV API_ORIGIN=${API_ORIGIN} NEXT_TELEMETRY_DISABLED=1
+RUN npm run build
+
+FROM node:24.19.0-bookworm-slim@sha256:a9f5f7c91a432850b2a8a7797adf5eadb6c733ceed61167806cee7ea7fbc29df
+WORKDIR /app
+ENV NODE_ENV=production HOSTNAME=0.0.0.0 PORT=4323 NEXT_TELEMETRY_DISABLED=1 API_ORIGIN=http://api:4322
+COPY --from=build --chown=1000:1000 /app/.next/standalone ./
+COPY --from=build --chown=1000:1000 /app/.next/static ./.next/static
+USER 1000:1000
+EXPOSE 4323
+ENTRYPOINT ["node"]
+CMD ["server.js"]
diff --git a/backend/pom.xml b/backend/pom.xml
index 4768669..ee79515 100644
--- a/backend/pom.xml
+++ b/backend/pom.xml
@@ -27,6 +27,10 @@
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-security</artifactId>
     </dependency>
+    <dependency>
+      <groupId>org.springframework.boot</groupId>
+      <artifactId>spring-boot-starter-actuator</artifactId>
+    </dependency>
     <dependency>
       <groupId>org.flywaydb</groupId>
       <artifactId>flyway-database-postgresql</artifactId>
diff --git a/backend/src/main/java/dev/evolution/monitor/ApiErrors.java b/backend/src/main/java/dev/evolution/monitor/ApiErrors.java
index 4614d7b..042b667 100644
--- a/backend/src/main/java/dev/evolution/monitor/ApiErrors.java
+++ b/backend/src/main/java/dev/evolution/monitor/ApiErrors.java
@@ -5,6 +5,8 @@ import org.springframework.http.MediaType;
 import org.springframework.http.ResponseEntity;
 import org.springframework.http.converter.HttpMessageNotReadableException;
 import org.springframework.security.authentication.BadCredentialsException;
+import org.springframework.dao.DataAccessResourceFailureException;
+import org.springframework.transaction.CannotCreateTransactionException;
 import org.springframework.web.HttpMediaTypeNotAcceptableException;
 import org.springframework.web.HttpMediaTypeNotSupportedException;
 import org.springframework.web.HttpRequestMethodNotSupportedException;
@@ -61,6 +63,12 @@ public class ApiErrors {
                 "The service could not complete the request.");
     }
 
+    @ExceptionHandler({DataAccessResourceFailureException.class, CannotCreateTransactionException.class})
+    ResponseEntity<Failure> authorityUnavailable() {
+        return response(HttpStatus.SERVICE_UNAVAILABLE, Code.INTERNAL_ERROR,
+                "The service could not complete the request.");
+    }
+
     private static ResponseEntity<Failure> response(HttpStatus status, Code code, String message) {
         return ResponseEntity.status(status).contentType(MediaType.APPLICATION_JSON)
                 .body(new Failure(new Detail(code, message)));
diff --git a/backend/src/main/java/dev/evolution/monitor/AuthenticationConfig.java b/backend/src/main/java/dev/evolution/monitor/AuthenticationConfig.java
index 3288033..0e0f353 100644
--- a/backend/src/main/java/dev/evolution/monitor/AuthenticationConfig.java
+++ b/backend/src/main/java/dev/evolution/monitor/AuthenticationConfig.java
@@ -74,6 +74,7 @@ public class AuthenticationConfig {
                         .dispatcherTypeMatchers(DispatcherType.ERROR).permitAll()
                         .requestMatchers(HttpMethod.POST, "/api/session/login").permitAll()
                         .requestMatchers(HttpMethod.GET, "/api/session/csrf").permitAll()
+                        .requestMatchers(HttpMethod.GET, "/ops/health/**", "/ops/metrics/**").permitAll()
                         .anyRequest().authenticated())
                 .exceptionHandling(errors -> errors
                         .authenticationEntryPoint((request, response, failure) -> {
diff --git a/backend/src/main/java/dev/evolution/monitor/AuthorityHealthIndicator.java b/backend/src/main/java/dev/evolution/monitor/AuthorityHealthIndicator.java
new file mode 100644
index 0000000..049da46
--- /dev/null
+++ b/backend/src/main/java/dev/evolution/monitor/AuthorityHealthIndicator.java
@@ -0,0 +1,25 @@
+package dev.evolution.monitor;
+
+import org.springframework.boot.actuate.health.Health;
+import org.springframework.boot.actuate.health.HealthIndicator;
+import org.springframework.dao.DataAccessException;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.stereotype.Component;
+
+@Component("authority")
+class AuthorityHealthIndicator implements HealthIndicator {
+    private final JdbcTemplate database;
+
+    AuthorityHealthIndicator(JdbcTemplate database) { this.database = database; }
+
+    @Override
+    public Health health() {
+        try {
+            return Integer.valueOf(1).equals(database.queryForObject("select 1", Integer.class))
+                    ? Health.up().build() : Health.down().build();
+        } catch (DataAccessException unavailable) {
+            // Neither connection details nor exception text belongs in the health response.
+            return Health.down().build();
+        }
+    }
+}
diff --git a/backend/src/main/java/dev/evolution/monitor/CheckQueue.java b/backend/src/main/java/dev/evolution/monitor/CheckQueue.java
index edb0dc8..c50208d 100644
--- a/backend/src/main/java/dev/evolution/monitor/CheckQueue.java
+++ b/backend/src/main/java/dev/evolution/monitor/CheckQueue.java
@@ -15,6 +15,13 @@ public class CheckQueue {
 
     public record Execution(CheckRunner.CheckRun check, String url) {}
 
+    @Transactional(readOnly = true)
+    public double oldestQueuedAgeSeconds() {
+        return ((Number) entities.createNativeQuery("select coalesce(greatest(0, extract(epoch from "
+                        + "clock_timestamp() - min(queued_at))), 0) from {h-schema}check_runs where state = 'QUEUED'")
+                .getSingleResult()).doubleValue();
+    }
+
     @Transactional
     public Execution startNext(UUID owner) {
         // The selected row stays locked until RUNNING and its owner commit together.
diff --git a/backend/src/main/java/dev/evolution/monitor/CheckWorker.java b/backend/src/main/java/dev/evolution/monitor/CheckWorker.java
index f8948b1..0796bad 100644
--- a/backend/src/main/java/dev/evolution/monitor/CheckWorker.java
+++ b/backend/src/main/java/dev/evolution/monitor/CheckWorker.java
@@ -11,21 +11,28 @@ import org.springframework.stereotype.Component;
 public class CheckWorker {
     private final CheckQueue queue;
     private final CheckRunner runner;
+    private final ProcessObservations observations;
     private final UUID owner = UUID.randomUUID();
     private volatile boolean acceptingClaims = true;
 
-    CheckWorker(CheckQueue queue, CheckRunner runner) {
+    CheckWorker(CheckQueue queue, CheckRunner runner, ProcessObservations observations) {
         this.queue = queue;
         this.runner = runner;
+        this.observations = observations;
     }
 
     public boolean executeNext() {
         if (!acceptingClaims) return false;
         var execution = queue.startNext(owner); // The RUNNING transaction commits before outbound I/O.
         if (execution == null) return false;
-        var result = runner.run(execution.check(), execution.url());
-        queue.finish(result, owner); // Only this execution owner may record the observed outcome.
-        return true;
+        observations.claimed();
+        try {
+            var result = runner.run(execution.check(), execution.url());
+            queue.finish(result, owner); // Only this execution owner may record the observed outcome.
+            return true;
+        } finally {
+            observations.finishedWork();
+        }
     }
 
     UUID ownerId() { return owner; }
diff --git a/backend/src/main/java/dev/evolution/monitor/HttpObservations.java b/backend/src/main/java/dev/evolution/monitor/HttpObservations.java
new file mode 100644
index 0000000..34deaa2
--- /dev/null
+++ b/backend/src/main/java/dev/evolution/monitor/HttpObservations.java
@@ -0,0 +1,88 @@
+package dev.evolution.monitor;
+
+import io.micrometer.common.KeyValue;
+import io.micrometer.common.KeyValues;
+import jakarta.servlet.FilterChain;
+import jakarta.servlet.ServletException;
+import jakarta.servlet.http.HttpServletRequest;
+import jakarta.servlet.http.HttpServletResponse;
+import java.io.IOException;
+import java.util.Set;
+import java.util.UUID;
+import org.springframework.boot.autoconfigure.condition.ConditionalOnWebApplication;
+import org.springframework.boot.web.servlet.FilterRegistrationBean;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.core.Ordered;
+import org.springframework.core.env.Environment;
+import org.springframework.http.server.observation.DefaultServerRequestObservationConvention;
+import org.springframework.http.server.observation.ServerRequestObservationContext;
+import org.springframework.http.server.observation.ServerRequestObservationConvention;
+import org.springframework.web.filter.OncePerRequestFilter;
+import org.springframework.web.servlet.HandlerMapping;
+
+@Configuration(proxyBeanMethods = false)
+@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
+class HttpObservations {
+    private static final Set<String> METHODS = Set.of("GET", "HEAD", "POST", "PUT", "PATCH", "DELETE", "OPTIONS", "TRACE");
+    private static final Set<String> ROUTES = Set.of("/api/monitors", "/api/monitors/{id}",
+            "/api/monitors/{id}/checks", "/api/monitors/{id}/checks/{checkId}", "/api/session",
+            "/api/session/login", "/api/session/logout", "/api/session/csrf", "/ops/health",
+            "/ops/health/**", "/ops/health/{*path}", "/ops/metrics", "/ops/metrics/{requiredMetricName}");
+
+    static String method(String value) { return METHODS.contains(value) ? value : "OTHER"; }
+    static String route(String value) { return value != null && ROUTES.contains(value) ? value : "UNMATCHED"; }
+    static String route(HttpServletRequest request, String pattern) {
+        String bounded = route(pattern);
+        if (!bounded.equals("UNMATCHED")) return bounded;
+        // Security endpoints can finish before a controller supplies its pattern. Only fixed literal paths qualify.
+        String path = request.getServletPath();
+        return path != null && !path.contains("{") && ROUTES.contains(path) ? path : "UNMATCHED";
+    }
+
+    @Bean
+    ServerRequestObservationConvention boundedHttpMetrics(Environment environment) {
+        String role = ProcessObservations.role(environment);
+        return new DefaultServerRequestObservationConvention() {
+            @Override
+            public KeyValues getLowCardinalityKeyValues(ServerRequestObservationContext context) {
+                int status = context.getResponse().getStatus();
+                return KeyValues.of(KeyValue.of("method", HttpObservations.method(context.getCarrier().getMethod())),
+                        KeyValue.of("uri", route(context.getCarrier(), context.getPathPattern())),
+                        KeyValue.of("status", status >= 100 && status <= 599 ? Integer.toString(status) : "UNKNOWN"),
+                        KeyValue.of("process_role", role));
+            }
+
+            @Override
+            public KeyValues getHighCardinalityKeyValues(ServerRequestObservationContext context) { return KeyValues.empty(); }
+
+            @Override
+            public String getContextualName(ServerRequestObservationContext context) {
+                return HttpObservations.method(context.getCarrier().getMethod()) + " " + route(context.getCarrier(), context.getPathPattern());
+            }
+        };
+    }
+
+    @Bean
+    FilterRegistrationBean<OncePerRequestFilter> requestCorrelation(ProcessObservations observations) {
+        var registration = new FilterRegistrationBean<OncePerRequestFilter>();
+        registration.setFilter(new OncePerRequestFilter() {
+            @Override
+            protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
+                    throws ServletException, IOException {
+                String requestId = UUID.randomUUID().toString(); // Never echo an inbound correlation/header value.
+                response.setHeader("X-Request-Id", requestId);
+                long started = System.nanoTime();
+                try { chain.doFilter(request, response); }
+                finally {
+                    Object pattern = request.getAttribute(HandlerMapping.BEST_MATCHING_PATTERN_ATTRIBUTE);
+                    observations.request(requestId, method(request.getMethod()),
+                            route(request, pattern instanceof String value ? value : null), response.getStatus(),
+                            (System.nanoTime() - started) / 1_000_000);
+                }
+            }
+        });
+        registration.setOrder(Ordered.HIGHEST_PRECEDENCE + 2);
+        return registration;
+    }
+}
diff --git a/backend/src/main/java/dev/evolution/monitor/ProcessObservations.java b/backend/src/main/java/dev/evolution/monitor/ProcessObservations.java
new file mode 100644
index 0000000..ad5c324
--- /dev/null
+++ b/backend/src/main/java/dev/evolution/monitor/ProcessObservations.java
@@ -0,0 +1,72 @@
+package dev.evolution.monitor;
+
+import io.micrometer.core.instrument.Counter;
+import io.micrometer.core.instrument.Gauge;
+import io.micrometer.core.instrument.MeterRegistry;
+import java.util.UUID;
+import java.util.concurrent.atomic.AtomicInteger;
+import org.slf4j.Logger;
+import org.slf4j.LoggerFactory;
+import org.springframework.core.env.Environment;
+import org.springframework.dao.DataAccessException;
+import org.springframework.stereotype.Component;
+import org.springframework.transaction.TransactionException;
+
+@Component
+class ProcessObservations {
+    private static final Logger LOG = LoggerFactory.getLogger(ProcessObservations.class);
+    private final String role;
+    private final String processId = UUID.randomUUID().toString();
+    private final AtomicInteger active = new AtomicInteger();
+    private final Counter claims;
+    private final Counter recoveries;
+
+    ProcessObservations(MeterRegistry meters, CheckQueue queue, Environment environment) {
+        role = role(environment);
+        Gauge.builder("check.queue.age", queue, source -> {
+            try { return source.oldestQueuedAgeSeconds(); }
+            catch (DataAccessException | TransactionException unavailable) { return Double.NaN; }
+        }).baseUnit("seconds").tag("process_role", role).register(meters);
+        if (role.equals("worker")) {
+            Gauge.builder("check.worker.active", active, AtomicInteger::get)
+                    .tag("process_role", role).register(meters);
+            claims = Counter.builder("check.claims").tag("process_role", role).register(meters);
+            recoveries = Counter.builder("check.recoveries").tag("process_role", role).register(meters);
+        } else {
+            claims = null;
+            recoveries = null;
+        }
+    }
+
+    static String role(Environment environment) { return environment.matchesProfiles("worker") ? "worker" : "api"; }
+
+    void claimed() {
+        // startNext has returned through the transaction proxy: this is a committed claim.
+        if (claims != null) claims.increment();
+        active.incrementAndGet();
+        event("check_claimed");
+    }
+
+    void finishedWork() { active.decrementAndGet(); }
+
+    void recovered(int count) {
+        if (count == 0) return;
+        // recoverExpired has committed before this callback; never count an attempted update.
+        if (recoveries != null) recoveries.increment(count);
+        event("checks_recovered");
+    }
+
+    void storeUnavailable() { event("worker_store_unavailable"); }
+
+    private void event(String event) {
+        LOG.atInfo().addKeyValue("event", event).addKeyValue("process_role", role)
+                .addKeyValue("process_id", processId).log("Worker observation");
+    }
+
+    void request(String requestId, String method, String route, int status, long durationMs) {
+        LOG.atInfo().addKeyValue("event", "http_request").addKeyValue("process_role", role)
+                .addKeyValue("process_id", processId).addKeyValue("request_id", requestId)
+                .addKeyValue("method", method).addKeyValue("route", route).addKeyValue("status", status)
+                .addKeyValue("duration_ms", durationMs).log("HTTP request completed");
+    }
+}
diff --git a/backend/src/main/java/dev/evolution/monitor/WorkerManagement.java b/backend/src/main/java/dev/evolution/monitor/WorkerManagement.java
new file mode 100644
index 0000000..75c8cd5
--- /dev/null
+++ b/backend/src/main/java/dev/evolution/monitor/WorkerManagement.java
@@ -0,0 +1,101 @@
+package dev.evolution.monitor;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sun.net.httpserver.HttpExchange;
+import com.sun.net.httpserver.HttpServer;
+import io.micrometer.core.instrument.MeterRegistry;
+import java.io.IOException;
+import java.net.InetSocketAddress;
+import java.util.List;
+import java.util.Map;
+import java.util.Set;
+import java.util.concurrent.ArrayBlockingQueue;
+import java.util.concurrent.ThreadPoolExecutor;
+import java.util.concurrent.TimeUnit;
+import org.springframework.beans.factory.annotation.Value;
+import org.springframework.boot.actuate.health.Status;
+import org.springframework.boot.actuate.metrics.MetricsEndpoint;
+import org.springframework.boot.availability.ApplicationAvailability;
+import org.springframework.boot.availability.ReadinessState;
+import org.springframework.context.SmartLifecycle;
+import org.springframework.context.annotation.Profile;
+import org.springframework.stereotype.Component;
+
+/** Fixed health/metrics transport for the existing non-web worker; no application routes. */
+@Component
+@Profile("worker")
+class WorkerManagement implements SmartLifecycle {
+    private static final Set<String> METRICS = Set.of("check.queue.age", "check.worker.active", "check.claims", "check.recoveries");
+    private final AuthorityHealthIndicator authority;
+    private final ApplicationAvailability availability;
+    private final CheckWorker worker;
+    private final MetricsEndpoint metrics;
+    private final ObjectMapper json;
+    private final String address;
+    private final int port;
+    private HttpServer server;
+    private ThreadPoolExecutor executor;
+    private volatile boolean running;
+
+    WorkerManagement(AuthorityHealthIndicator authority, ApplicationAvailability availability, CheckWorker worker,
+            MeterRegistry meters, ObjectMapper json, @Value("${monitor.management-address:127.0.0.1}") String address,
+            @Value("${monitor.management-port:4324}") int port) {
+        this.authority = authority;
+        this.availability = availability;
+        this.worker = worker;
+        this.metrics = new MetricsEndpoint(meters);
+        this.json = json;
+        this.address = address;
+        this.port = port;
+    }
+
+    @Override
+    public void start() {
+        try {
+            server = HttpServer.create(new InetSocketAddress(address, port), 8);
+            executor = new ThreadPoolExecutor(2, 2, 0, TimeUnit.MILLISECONDS, new ArrayBlockingQueue<>(8),
+                    Thread.ofPlatform().name("worker-management-", 0).factory(), new ThreadPoolExecutor.AbortPolicy());
+            server.setExecutor(executor);
+            server.createContext("/ops/", this::handle);
+            running = true;
+            server.start();
+        } catch (IOException error) { throw new IllegalStateException("Worker management listener could not bind", error); }
+    }
+
+    private void handle(HttpExchange exchange) throws IOException {
+        try (exchange) {
+            String path = exchange.getRequestURI().getPath();
+            if (!exchange.getRequestMethod().equals("GET")) { send(exchange, 405, Map.of("status", "METHOD_NOT_ALLOWED")); return; }
+            if (path.equals("/ops/health/liveness")) {
+                send(exchange, running ? 200 : 503, Map.of("status", running ? "UP" : "DOWN"));
+            } else if (path.equals("/ops/health/readiness")) {
+                boolean ready = running && worker.acceptingClaims()
+                        && availability.getReadinessState() == ReadinessState.ACCEPTING_TRAFFIC
+                        && authority.health().getStatus().equals(Status.UP);
+                send(exchange, ready ? 200 : 503, Map.of("status", ready ? "UP" : "DOWN"));
+            } else if (path.equals("/ops/metrics")) {
+                send(exchange, 200, Map.of("names", METRICS.stream().sorted().toList()));
+            } else if (path.startsWith("/ops/metrics/") && METRICS.contains(path.substring("/ops/metrics/".length()))) {
+                var value = metrics.metric(path.substring("/ops/metrics/".length()), List.of());
+                send(exchange, value == null ? 404 : 200, value == null ? Map.of("status", "NOT_FOUND") : value);
+            } else send(exchange, 404, Map.of("status", "NOT_FOUND"));
+        }
+    }
+
+    private void send(HttpExchange exchange, int status, Object value) throws IOException {
+        byte[] body = json.writeValueAsBytes(value);
+        exchange.getResponseHeaders().set("Content-Type", "application/json");
+        exchange.getResponseHeaders().set("Cache-Control", "no-store");
+        exchange.sendResponseHeaders(status, body.length);
+        exchange.getResponseBody().write(body);
+    }
+
+    @Override
+    public void stop() {
+        running = false;
+        if (server != null) server.stop(0);
+        if (executor != null) executor.shutdownNow();
+    }
+
+    @Override public boolean isRunning() { return running; }
+}
diff --git a/backend/src/main/java/dev/evolution/monitor/WorkerProcess.java b/backend/src/main/java/dev/evolution/monitor/WorkerProcess.java
index bca2033..8033fee 100644
--- a/backend/src/main/java/dev/evolution/monitor/WorkerProcess.java
+++ b/backend/src/main/java/dev/evolution/monitor/WorkerProcess.java
@@ -8,6 +8,8 @@ import org.springframework.scheduling.annotation.EnableScheduling;
 import org.springframework.scheduling.annotation.Scheduled;
 import org.springframework.stereotype.Component;
 import org.springframework.web.context.WebApplicationContext;
+import org.springframework.dao.DataAccessException;
+import org.springframework.transaction.TransactionException;
 
 @Component
 @Profile("worker")
@@ -16,22 +18,29 @@ class WorkerProcess {
     private final CheckQueue queue;
     private final CheckWorker worker;
     private final boolean scheduling;
+    private final ProcessObservations observations;
 
-    WorkerProcess(CheckQueue queue, CheckWorker worker, ConfigurableApplicationContext context,
+    WorkerProcess(CheckQueue queue, CheckWorker worker, ProcessObservations observations, ConfigurableApplicationContext context,
             @Value("${monitor.scheduler-enabled:true}") boolean scheduling) {
         if (context instanceof WebApplicationContext) {
             throw new IllegalStateException("Worker requires --spring.main.web-application-type=none.");
         }
         this.queue = queue;
         this.worker = worker;
+        this.observations = observations;
         this.scheduling = scheduling;
     }
 
     @Scheduled(fixedDelay = 250)
     void tick() {
         if (!worker.acceptingClaims()) return;
-        queue.recoverExpired(Instant.now());
-        if (scheduling) queue.scheduleDue(Instant.now());
-        worker.executeNext();
+        try {
+            observations.recovered(queue.recoverExpired(Instant.now()));
+            if (scheduling) queue.scheduleDue(Instant.now());
+            worker.executeNext();
+        } catch (DataAccessException | TransactionException unavailable) {
+            // A later normal tick may read authority again; never invent a claim or endpoint outcome.
+            observations.storeUnavailable();
+        }
     }
 }
diff --git a/backend/src/main/resources/application-worker.properties b/backend/src/main/resources/application-worker.properties
index acaf764..5bc13b7 100644
--- a/backend/src/main/resources/application-worker.properties
+++ b/backend/src/main/resources/application-worker.properties
@@ -1,3 +1,5 @@
 # Spring stops periodic scheduling, then drains the current task before destroying JPA.
 # If this bound expires, the durable lease recovers its unknown outcome as ABORTED.
 spring.lifecycle.timeout-per-shutdown-phase=3s
+monitor.management-address=${MANAGEMENT_ADDRESS:127.0.0.1}
+monitor.management-port=${MANAGEMENT_PORT:4324}
diff --git a/backend/src/main/resources/application.properties b/backend/src/main/resources/application.properties
index 8387302..f2492f5 100644
--- a/backend/src/main/resources/application.properties
+++ b/backend/src/main/resources/application.properties
@@ -1,5 +1,5 @@
 spring.application.name=monitor-api
-server.address=127.0.0.1
+server.address=${API_ADDRESS:127.0.0.1}
 server.port=${API_PORT:4322}
 monitor.fixture-origin=${FIXTURE_ORIGIN:http://127.0.0.1:4321}
 monitor.allow-test-fixture=${ALLOW_TEST_FIXTURE:false}
@@ -7,6 +7,8 @@ spring.jackson.deserialization.fail-on-trailing-tokens=true
 spring.datasource.url=${DB_URL:jdbc:postgresql://127.0.0.1:15432/monitor}
 spring.datasource.username=${DB_USER:wse_industry}
 spring.datasource.password=${DB_PASSWORD:}
+spring.datasource.hikari.connection-timeout=1000
+spring.datasource.hikari.validation-timeout=500
 spring.flyway.schemas=${DB_SCHEMA:public}
 spring.flyway.default-schema=${DB_SCHEMA:public}
 spring.flyway.clean-disabled=true
@@ -22,3 +24,15 @@ server.servlet.session.cookie.http-only=true
 server.servlet.session.cookie.same-site=lax
 # The product remains loopback HTTP. Enable for a TLS deployment; never put tokens in JavaScript.
 server.servlet.session.cookie.secure=${SESSION_COOKIE_SECURE:false}
+management.endpoints.web.base-path=/ops
+management.endpoints.web.exposure.include=health,metrics
+management.endpoint.health.probes.enabled=true
+management.endpoint.health.group.liveness.include=livenessState
+management.endpoint.health.group.readiness.include=readinessState,authority
+management.endpoint.health.show-details=never
+management.health.db.enabled=false
+management.metrics.enable.all=false
+management.metrics.enable.http.server.requests=true
+management.metrics.enable.check=true
+logging.structured.format.console=ecs
+spring.main.banner-mode=off
diff --git a/backend/src/test/java/dev/evolution/monitor/ApiErrorBoundaryTest.java b/backend/src/test/java/dev/evolution/monitor/ApiErrorBoundaryTest.java
index 1812e95..ddecc91 100644
--- a/backend/src/test/java/dev/evolution/monitor/ApiErrorBoundaryTest.java
+++ b/backend/src/test/java/dev/evolution/monitor/ApiErrorBoundaryTest.java
@@ -9,9 +9,28 @@ import com.fasterxml.jackson.databind.ObjectMapper;
 import org.junit.jupiter.api.Test;
 import org.springframework.http.MediaType;
 import org.springframework.security.web.method.annotation.AuthenticationPrincipalArgumentResolver;
+import org.springframework.transaction.CannotCreateTransactionException;
 import org.springframework.test.web.servlet.setup.MockMvcBuilders;
 
 class ApiErrorBoundaryTest {
+    @Test
+    void unavailableAuthorityRejectsTheRequestWithoutPrivateConnectionDetails() throws Exception {
+        CheckRunner checks = mock(CheckRunner.class);
+        when(checks.canonicalUrl("http://127.0.0.1:4321/ok"))
+                .thenThrow(new CannotCreateTransactionException("Private authority detail"));
+        var mvc = MockMvcBuilders.standaloneSetup(new MonitorController(checks, mock(MonitorStore.class)))
+                .setCustomArgumentResolvers(new AuthenticationPrincipalArgumentResolver())
+                .setControllerAdvice(new ApiErrors()).build();
+        String body = mvc.perform(post("/api/monitors").contentType(MediaType.APPLICATION_JSON).content("""
+                {"name":"Fixture monitor","url":"http://127.0.0.1:4321/ok","interval":60,"enabled":true}
+                """))
+                .andExpect(status().isServiceUnavailable())
+                .andExpect(jsonPath("$.error.code").value("INTERNAL_ERROR"))
+                .andExpect(jsonPath("$.data").doesNotExist())
+                .andReturn().getResponse().getContentAsString();
+        assertFalse(body.contains("Private authority detail"));
+    }
+
     @Test
     void unexpectedFailureIsAnInternalErrorWithoutPrivateExceptionDetails() throws Exception {
         CheckRunner checks = mock(CheckRunner.class);
diff --git a/backend/src/test/java/dev/evolution/monitor/HttpObservationsTest.java b/backend/src/test/java/dev/evolution/monitor/HttpObservationsTest.java
new file mode 100644
index 0000000..bab98d3
--- /dev/null
+++ b/backend/src/test/java/dev/evolution/monitor/HttpObservationsTest.java
@@ -0,0 +1,51 @@
+package dev.evolution.monitor;
+
+import static org.junit.jupiter.api.Assertions.*;
+import static org.mockito.ArgumentMatchers.*;
+import static org.mockito.Mockito.*;
+
+import java.util.HashMap;
+import java.util.Map;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.http.server.observation.ServerRequestObservationContext;
+import org.springframework.mock.env.MockEnvironment;
+import org.springframework.mock.web.MockHttpServletRequest;
+import org.springframework.mock.web.MockHttpServletResponse;
+import org.springframework.web.servlet.HandlerMapping;
+
+class HttpObservationsTest {
+    @Test
+    void arbitraryMethodAndUnmatchedPathCannotBecomeMetricLabels() {
+        var request = new MockHttpServletRequest("ARBITRARY", "/arbitrary/private/path");
+        request.setServletPath("/arbitrary/private/path");
+        var response = new MockHttpServletResponse();
+        response.setStatus(404);
+        var context = new ServerRequestObservationContext(request, response);
+        var convention = new HttpObservations().boundedHttpMetrics(new MockEnvironment());
+        var tags = new HashMap<String, String>();
+        convention.getLowCardinalityKeyValues(context).forEach(value -> tags.put(value.getKey(), value.getValue()));
+        assertEquals(Map.of("method", "OTHER", "uri", "UNMATCHED", "status", "404", "process_role", "api"), tags);
+        assertFalse(convention.getHighCardinalityKeyValues(context).iterator().hasNext());
+        context.setPathPattern("/api/monitors/{id}");
+        assertEquals("OTHER /api/monitors/{id}", convention.getContextualName(context));
+    }
+
+    @Test
+    void responseCorrelationIsGeneratedAndRecordedWithoutEchoingInput() throws Exception {
+        var observations = mock(ProcessObservations.class);
+        var filter = new HttpObservations().requestCorrelation(observations).getFilter();
+        var request = new MockHttpServletRequest("GET", "/api/monitors/missing");
+        String untrusted = UUID.randomUUID().toString();
+        request.addHeader("X-Request-Id", untrusted);
+        var response = new MockHttpServletResponse();
+        filter.doFilter(request, response, (incoming, outgoing) -> {
+            incoming.setAttribute(HandlerMapping.BEST_MATCHING_PATTERN_ATTRIBUTE, "/api/monitors/{id}");
+            response.setStatus(404);
+        });
+        String issued = response.getHeader("X-Request-Id");
+        assertTrue(issued != null && !issued.equals(untrusted));
+        assertDoesNotThrow(() -> UUID.fromString(issued));
+        verify(observations).request(eq(issued), eq("GET"), eq("/api/monitors/{id}"), eq(404), anyLong());
+    }
+}
diff --git a/backend/src/test/java/dev/evolution/monitor/OperationsIntegrationTest.java b/backend/src/test/java/dev/evolution/monitor/OperationsIntegrationTest.java
new file mode 100644
index 0000000..2e7caac
--- /dev/null
+++ b/backend/src/test/java/dev/evolution/monitor/OperationsIntegrationTest.java
@@ -0,0 +1,97 @@
+package dev.evolution.monitor;
+
+import static org.junit.jupiter.api.Assertions.*;
+import static org.mockito.ArgumentMatchers.*;
+import static org.mockito.Mockito.*;
+
+import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.time.Instant;
+import java.util.UUID;
+import java.util.concurrent.CountDownLatch;
+import java.util.concurrent.Executors;
+import java.util.concurrent.TimeUnit;
+import org.junit.jupiter.api.AfterAll;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.context.ConfigurableApplicationContext;
+import org.springframework.mock.env.MockEnvironment;
+import org.springframework.test.annotation.DirtiesContext;
+import org.springframework.test.context.DynamicPropertyRegistry;
+import org.springframework.test.context.DynamicPropertySource;
+import org.springframework.test.context.bean.override.mockito.MockitoSpyBean;
+
+@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
+@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_CLASS)
+class OperationsIntegrationTest {
+    @Autowired CheckQueue queue;
+    @Autowired MonitorStore store;
+    @Autowired UserAccounts accounts;
+    @Autowired ConfigurableApplicationContext context;
+    @MockitoSpyBean CheckRunner runner;
+
+    @DynamicPropertySource
+    static void database(DynamicPropertyRegistry properties) { TestDatabase.configure(properties, "e24_metrics"); }
+
+    @AfterAll
+    static void cleanup() { TestDatabase.drop("e24_metrics"); }
+
+    @Test
+    void actualQueueClaimActivityAndRecoveryAreMeasuredAfterCommit() throws Exception {
+        accounts.bootstrap(SessionClient.password(), SessionClient.password());
+        UUID alice;
+        try (var connection = TestDatabase.connect(); var statement = connection.createStatement();
+                var rows = statement.executeQuery("select id from e24_metrics.users where username='alice-e04'")) {
+            assertTrue(rows.next());
+            alice = rows.getObject(1, UUID.class);
+        }
+        UUID monitor = store.create(alice, new MonitorController.CreateMonitor("E24 metric fixture",
+                "http://127.0.0.1:4321/ok", 60, false)).monitor().id();
+        var first = store.enqueueCheck(alice, monitor, "e24-metric-one");
+        TestDatabase.execute("update e24_metrics.check_runs set queued_at=clock_timestamp()-interval '10 seconds'");
+        var environment = new MockEnvironment();
+        environment.setActiveProfiles("worker");
+        var entered = new CountDownLatch(1);
+        var release = new CountDownLatch(1);
+        try (var meters = new SimpleMeterRegistry(); var executor = Executors.newSingleThreadExecutor()) {
+            var observations = new ProcessObservations(meters, queue, environment);
+            var worker = new CheckWorker(queue, runner, observations);
+            double age = meters.get("check.queue.age").gauge().value();
+            assertTrue(Double.isFinite(age) && age >= 9, "Actual queued age is positive");
+            doAnswer(invocation -> {
+                CheckRunner.CheckRun check = invocation.getArgument(0);
+                entered.countDown();
+                assertTrue(release.await(2, TimeUnit.SECONDS), "Held runner released within existing lease");
+                return new CheckRunner.CheckRun(check.id(), check.monitorId(), check.trigger(), "SUCCEEDED", 200,
+                        1L, null, check.startedAt(), Instant.now());
+            }).when(runner).run(any(CheckRunner.CheckRun.class), anyString());
+            var result = executor.submit(worker::executeNext);
+            try {
+                assertTrue(entered.await(3, TimeUnit.SECONDS), "Production worker reached held runner");
+                assertEquals(1, meters.get("check.worker.active").gauge().value());
+                assertEquals(1, meters.get("check.claims").counter().count());
+                assertEquals("RUNNING", store.check(alice, monitor, first.id()).state());
+            } finally { release.countDown(); }
+            assertTrue(result.get(3, TimeUnit.SECONDS));
+            assertEquals(0, meters.get("check.worker.active").gauge().value());
+            assertEquals("SUCCEEDED", store.check(alice, monitor, first.id()).state());
+
+            var expired = store.enqueueCheck(alice, monitor, "e24-metric-two");
+            assertEquals(expired.id(), queue.startNext(worker.ownerId()).check().id());
+            TestDatabase.execute("update e24_metrics.check_runs set lease_expires_at=clock_timestamp()-interval '1 second' "
+                    + "where state='RUNNING'");
+            new WorkerProcess(queue, worker, observations, context, false).tick();
+            assertEquals("ABORTED", store.check(alice, monitor, expired.id()).state());
+            assertEquals(1, meters.get("check.recoveries").counter().count());
+            assertEquals(1, meters.get("check.claims").counter().count());
+            assertEquals(0, meters.get("check.queue.age").gauge().value());
+            verify(runner, times(1)).run(any(CheckRunner.CheckRun.class), anyString());
+            Files.writeString(Path.of("target/e24-metrics.json"), """
+                    {"result":"PASS","positiveQueueAge":true,"heldWorkerActive":1,"committedWorkerClaims":1,
+                     "committedRecoveries":1,"idleActive":0,"emptyQueueAge":0,"processCrashes":0,"dependencyStops":0}
+                    """);
+        } finally { release.countDown(); }
+    }
+}
diff --git a/backend/src/test/java/dev/evolution/monitor/TestDatabase.java b/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
index 26b2afa..ea85a3a 100644
--- a/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
+++ b/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
@@ -13,7 +13,7 @@ final class TestDatabase {
             "e04_missing_column", "e04_extra_required", "e04_session", "e04_users",
             "e04_missing_user_column", "e04_extra_user_required", "e05_ownership", "e05_fresh",
             "e05_upgrade", "e05_missing_owner", "e05_nullable_owner", "e05_missing_owner_fk", "e07_history", "e07_index_upgrade",
-            "e09_scheduler", "e10_ownership", "e11_recovery", "e20_plan");
+            "e09_scheduler", "e10_ownership", "e11_recovery", "e20_plan", "e24_metrics");
 
     static Connection connect() throws SQLException {
         return DriverManager.getConnection(System.getenv().getOrDefault("DB_URL", "jdbc:postgresql://127.0.0.1:15432/monitor"),
diff --git a/compose.e24.yaml b/compose.e24.yaml
new file mode 100644
index 0000000..57df833
--- /dev/null
+++ b/compose.e24.yaml
@@ -0,0 +1,20 @@
+# Test-only loopback exception and controlled fixture. Production defaults stay unsafe-denying.
+services:
+  api:
+    environment:
+      ALLOW_TEST_FIXTURE: 'true'
+  worker:
+    environment:
+      ALLOW_TEST_FIXTURE: 'true'
+    ports:
+      - "127.0.0.1:4321:4321"
+  fixture:
+    image: node:24.19.0-bookworm-slim@sha256:a9f5f7c91a432850b2a8a7797adf5eadb6c733ceed61167806cee7ea7fbc29df
+    user: '1000:1000'
+    network_mode: service:worker
+    entrypoint: ["node"]
+    command: ["/fixture.mjs"]
+    environment:
+      E24_BODY_SENTINEL: ${E24_BODY_SENTINEL:?Provide runtime-only fixture input}
+    volumes:
+      - ./scripts/e24-fixture.mjs:/fixture.mjs:ro
diff --git a/compose.production.yaml b/compose.production.yaml
new file mode 100644
index 0000000..40e29c4
--- /dev/null
+++ b/compose.production.yaml
@@ -0,0 +1,46 @@
+name: wse-industry-e24
+services:
+  api:
+    image: wse-industry-e24-api:local
+    build:
+      context: .
+      dockerfile: Dockerfile.api
+    environment:
+      API_ADDRESS: 0.0.0.0
+      DB_URL: jdbc:postgresql://postgres:5432/monitor
+      DB_SCHEMA: ${DB_SCHEMA:?Provide the intended database schema}
+      DB_USER: ${DB_USER:-wse_industry}
+      DB_PASSWORD: ${DB_PASSWORD:-}
+    ports:
+      - "127.0.0.1:4322:4322"
+    networks: [default, database]
+    stop_grace_period: 5s
+  worker:
+    image: wse-industry-e24-api:local
+    command: ["--spring.profiles.active=worker", "--spring.main.web-application-type=none"]
+    environment:
+      MANAGEMENT_ADDRESS: 0.0.0.0
+      DB_URL: jdbc:postgresql://postgres:5432/monitor
+      DB_SCHEMA: ${DB_SCHEMA:?Provide the intended database schema}
+      DB_USER: ${DB_USER:-wse_industry}
+      DB_PASSWORD: ${DB_PASSWORD:-}
+    ports:
+      - "127.0.0.1:4324:4324"
+    networks: [default, database]
+    stop_grace_period: 5s
+  frontend:
+    image: wse-industry-e24-frontend:local
+    build:
+      context: .
+      dockerfile: Dockerfile.frontend
+      args:
+        API_ORIGIN: http://api:4322
+    environment:
+      API_ORIGIN: http://api:4322
+    ports:
+      - "127.0.0.1:4323:4323"
+    stop_grace_period: 5s
+networks:
+  database:
+    external: true
+    name: wse-industry_database
diff --git a/evidence/phase-1/E24/repair1/attempt1-native.json b/evidence/phase-1/E24/repair1/attempt1-native.json
new file mode 100644
index 0000000..908d69a
--- /dev/null
+++ b/evidence/phase-1/E24/repair1/attempt1-native.json
@@ -0,0 +1,61 @@
+{
+  "profile": "phase-1",
+  "thread": "E24",
+  "attempt": 1,
+  "specRevision": "2ada57a71cd34fa2fae9809415c362a8bbfcdf02",
+  "preservedManifestSha256": "49e7b4b4bba5a113213791ec8eca6c460ba7298dbe19e242bfcff124e2283861",
+  "classification": "Original restricted-shell dependency DNS failure before compilation or tests; preserved without retry or modification.",
+  "budgetBeforeRepair": {
+    "baseline_invocations": 1,
+    "baseline_seconds": 7.477,
+    "affected_maven_invocations": 1,
+    "affected_maven_failures": 1,
+    "affected_maven_wall_seconds": 1.724,
+    "java_tests_executed": 0,
+    "image_build_invocations": 0,
+    "full_container_runs": 0,
+    "postgres_stop_restore_sequences": 0,
+    "root_full_container_runs": 0,
+    "load_runs": 0,
+    "parameter_sweeps": 0,
+    "automatic_retries": 0,
+    "repair_tasks_used_after_dispatch": 1
+  },
+  "outputs": [
+    {
+      "source": "output/phase-1/e24/affected-java.log",
+      "sha256": "55a0316da21b3a7be95b501a69a4d63a9f5acd8ad133b0520a104a4adf6435a7",
+      "bytes": 1591,
+      "encoding": "utf-8",
+      "raw": "[INFO] Scanning for projects...\n[INFO] \n[INFO] ---------------------< dev.evolution:monitor-api >----------------------\n[INFO] Building monitor-api 0.0.1\n[INFO]   from pom.xml\n[INFO] --------------------------------[ jar ]---------------------------------\n[INFO] ------------------------------------------------------------------------\n[INFO] BUILD FAILURE\n[INFO] ------------------------------------------------------------------------\n[INFO] Total time:  0.586 s\n[INFO] Finished at: 2026-08-28T18:11:47+09:00\n[INFO] ------------------------------------------------------------------------\n[ERROR] Failed to execute goal on project monitor-api: Could not collect dependencies for project dev.evolution:monitor-api:jar:0.0.1\n[ERROR] Failed to read artifact descriptor for org.springframework.boot:spring-boot-starter-actuator:jar:3.5.16\n[ERROR] \tCaused by: The following artifacts could not be resolved: org.springframework.boot:spring-boot-starter-actuator:pom:3.5.16 (absent): Could not transfer artifact org.springframework.boot:spring-boot-starter-actuator:pom:3.5.16 from/to central (https://repo.maven.apache.org/maven2): repo.maven.apache.org: nodename nor servname provided, or not known\n[ERROR] -> [Help 1]\n[ERROR] \n[ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.\n[ERROR] Re-run Maven using the -X switch to enable full debug logging.\n[ERROR] \n[ERROR] For more information about the errors and possible solutions, please read the following articles:\n[ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/DependencyResolutionException\n"
+    },
+    {
+      "source": "output/phase-1/e24/affected-java.started.json",
+      "sha256": "9d1b9eb55b4fa66a9c6bcf944effa2f75c6622b557a650b8b12f5c101f285a21",
+      "bytes": 167,
+      "encoding": "utf-8",
+      "raw": "{\"command\":\"mvn -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package\",\"startedAt\":\"2026-08-28T09:11:45.983Z\"}\n"
+    },
+    {
+      "source": "output/phase-1/e24/baseline.json",
+      "sha256": "01a0fcb3a4758721ac02b66b67e5a359f10ee66a117225a5a0d55e450f35ed79",
+      "bytes": 1074,
+      "encoding": "utf-8",
+      "raw": "{\n  \"start\": \"563b325ef871fe6d1fbfef7cf39a6581f2d0a94d\",\n  \"fixtureSha256\": \"47f86919e5f251c3f8ee394727235b906fc03a5eac67e36cc3e273486a21a8fd\",\n  \"completed\": [\n    \"fixed seed and authentication; one liveness observation; no further guarantee probed\"\n  ],\n  \"postgresStopSequences\": 0,\n  \"workersStarted\": 0,\n  \"applicationContainersStarted\": 0,\n  \"seedCounts\": {\n    \"users\": 2,\n    \"monitors\": 2,\n    \"checks\": 4\n  },\n  \"observation\": {\n    \"method\": \"GET\",\n    \"path\": \"/ops/health/liveness\",\n    \"authenticated\": true,\n    \"status\": 404,\n    \"requiredStatus\": 200,\n    \"criterionMet\": false,\n    \"apiPid\": 97310\n  },\n  \"result\": \"COUNTEREXAMPLE: liveness interface is absent at unchanged START\",\n  \"apiExit\": {\n    \"code\": 143,\n    \"signal\": null,\n    \"elapsedMs\": 42,\n    \"awaited\": true,\n    \"purpose\": \"baseline cleanup\"\n  },\n  \"logInspection\": {\n    \"lines\": 52,\n    \"runtimeSentinelMatches\": 0,\n    \"rawLogsPersisted\": false\n  },\n  \"cleanup\": {\n    \"ownedSchemaDropped\": true,\n    \"cleanupFailures\": [],\n    \"apiExitAwaited\": true\n  },\n  \"elapsedSeconds\": 7.477\n}\n"
+    },
+    {
+      "source": "output/phase-1/e24/baseline.started.json",
+      "sha256": "e080e5e5299cf57ccb5988a5ec48a856e0df8f0e3bf233da0ff765e10ca51dfc",
+      "bytes": 134,
+      "encoding": "utf-8",
+      "raw": "{\"command\":\"node scripts/e24-baseline.mjs\",\"startedAt\":\"2026-08-28T08:51:05.183Z\",\"start\":\"563b325ef871fe6d1fbfef7cf39a6581f2d0a94d\"}\n"
+    },
+    {
+      "source": "output/phase-1/e24/invocations.jsonl",
+      "sha256": "d027b711d8f765008f06cdc9a73aad099dbefe38fffdf20e96db8af556dacfa0",
+      "bytes": 336,
+      "encoding": "utf-8",
+      "raw": "{\"command\":\"node scripts/e24-baseline.mjs\",\"startedAt\":\"2026-08-28T08:51:05.183Z\",\"elapsedSeconds\":7.477,\"exitCode\":0}\n{\"command\":\"mvn -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package\",\"startedAt\":\"2026-08-28T09:11:45.983Z\",\"elapsedSeconds\":1.724,\"exitCode\":1,\"signal\":null}\n"
+    }
+  ]
+}
diff --git a/next.config.mjs b/next.config.mjs
index 7a1e01e..2571ccd 100644
--- a/next.config.mjs
+++ b/next.config.mjs
@@ -1,5 +1,6 @@
 /** @type {import('next').NextConfig} */
 const config = {
+  output: 'standalone',
   poweredByHeader: false,
   async rewrites() {
     return [{ source: '/api/:path*', destination: `${process.env.API_ORIGIN ?? 'http://127.0.0.1:4322'}/api/:path*` }];
diff --git a/package.json b/package.json
index afee327..be9e5a3 100644
--- a/package.json
+++ b/package.json
@@ -20,6 +20,7 @@
     "typecheck": "next typegen && tsc --noEmit",
     "test:e2e": "playwright test",
     "test:worker": "E09_MANUAL_WORKER=1 playwright test tests/browser/worker.spec.ts",
+    "test:container": "node scripts/e24-container.mjs",
     "verify": "node scripts/verify.mjs"
   },
   "dependencies": {
diff --git a/scripts/e24-fixture.mjs b/scripts/e24-fixture.mjs
new file mode 100644
index 0000000..992a8e2
--- /dev/null
+++ b/scripts/e24-fixture.mjs
@@ -0,0 +1,43 @@
+import { createServer } from 'node:http';
+
+const body = process.env.E24_BODY_SENTINEL;
+if (!body) throw new Error('Runtime-only response sentinel is required');
+const held = new Set();
+let outboundCalls = 0;
+let holdRequests = 0;
+let releases = 0;
+let watchdogReleases = 0;
+let lastHeldMs = null;
+function release(response, watchdog) {
+  if (!held.delete(response)) return;
+  clearTimeout(response.releaseTimer);
+  lastHeldMs = (performance.now() - response.heldAt);
+  if (watchdog) watchdogReleases++; else releases++;
+  response.writeHead(200, { 'Content-Type': 'text/plain' }).end(body);
+}
+const server = createServer((request, response) => {
+  const path = new URL(request.url, 'http://127.0.0.1:4321').pathname;
+  if (path === '/hold') {
+    outboundCalls++;
+    holdRequests++;
+    held.add(response);
+    response.heldAt = performance.now();
+    response.releaseTimer = setTimeout(() => release(response, true), 350);
+    response.once('close', () => { held.delete(response); clearTimeout(response.releaseTimer); });
+  } else if (path === '/ok') {
+    outboundCalls++;
+    response.writeHead(200, { 'Content-Type': 'text/plain' }).end(body);
+  } else if (path === '/__e24/status') {
+    response.writeHead(200, { 'Content-Type': 'application/json' }).end(JSON.stringify({
+      outboundCalls, holdRequests, held: held.size, releases, watchdogReleases, lastHeldMs,
+    }));
+  } else if (path === '/__e24/release' && request.method === 'POST') {
+    for (const pending of held) release(pending, false);
+    response.writeHead(200).end();
+  } else response.writeHead(404).end();
+});
+server.listen(4321, '0.0.0.0');
+process.on('SIGTERM', () => {
+  for (const response of held) response.destroy();
+  server.close(() => process.exit(143));
+});


