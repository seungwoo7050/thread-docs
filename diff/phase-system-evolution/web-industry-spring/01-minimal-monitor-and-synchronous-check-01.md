# E01 최소 Monitor와 동기 Check

## `fixture 전용 동기 Monitor Check API 도입`

diff --git a/.gitignore b/.gitignore
new file mode 100644
index 0000000..43bf81a
--- /dev/null
+++ b/.gitignore
@@ -0,0 +1,11 @@
+node_modules/
+.next/
+.m2/
+.npm-cache/
+backend/target/
+test-results/
+playwright-report/
+output/
+*.tsbuildinfo
+*.log
+.DS_Store
diff --git a/.java-version b/.java-version
new file mode 100644
index 0000000..25e6ec2
--- /dev/null
+++ b/.java-version
@@ -0,0 +1 @@
+21.0.7
diff --git a/.maven-version b/.maven-version
new file mode 100644
index 0000000..a9f8d1b
--- /dev/null
+++ b/.maven-version
@@ -0,0 +1 @@
+3.9.11
diff --git a/.mvn/maven.config b/.mvn/maven.config
new file mode 100644
index 0000000..1700a09
--- /dev/null
+++ b/.mvn/maven.config
@@ -0,0 +1 @@
+-Dmaven.repo.local=.m2/repository
diff --git a/SPEC_REVISION b/SPEC_REVISION
new file mode 100644
index 0000000..a7af57d
--- /dev/null
+++ b/SPEC_REVISION
@@ -0,0 +1 @@
+0a006589477f8ae47bad3faa5510c999cff85ee4
diff --git a/backend/pom.xml b/backend/pom.xml
new file mode 100644
index 0000000..26ef890
--- /dev/null
+++ b/backend/pom.xml
@@ -0,0 +1,53 @@
+<?xml version="1.0" encoding="UTF-8"?>
+<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
+         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
+  <modelVersion>4.0.0</modelVersion>
+  <parent>
+    <groupId>org.springframework.boot</groupId>
+    <artifactId>spring-boot-starter-parent</artifactId>
+    <version>3.5.16</version>
+    <relativePath/>
+  </parent>
+  <groupId>dev.evolution</groupId>
+  <artifactId>monitor-api</artifactId>
+  <version>0.0.1</version>
+  <properties>
+    <java.version>21</java.version>
+  </properties>
+  <dependencies>
+    <dependency>
+      <groupId>org.springframework.boot</groupId>
+      <artifactId>spring-boot-starter-web</artifactId>
+    </dependency>
+    <dependency>
+      <groupId>org.springframework.boot</groupId>
+      <artifactId>spring-boot-starter-test</artifactId>
+      <scope>test</scope>
+    </dependency>
+  </dependencies>
+  <build>
+    <plugins>
+      <plugin>
+        <groupId>org.springframework.boot</groupId>
+        <artifactId>spring-boot-maven-plugin</artifactId>
+      </plugin>
+      <plugin>
+        <groupId>org.apache.maven.plugins</groupId>
+        <artifactId>maven-enforcer-plugin</artifactId>
+        <version>3.6.2</version>
+        <executions>
+          <execution>
+            <id>pinned-runtimes</id>
+            <goals><goal>enforce</goal></goals>
+            <configuration>
+              <rules>
+                <requireJavaVersion><version>[21.0.7,21.0.8)</version></requireJavaVersion>
+                <requireMavenVersion><version>[3.9.11]</version></requireMavenVersion>
+              </rules>
+            </configuration>
+          </execution>
+        </executions>
+      </plugin>
+    </plugins>
+  </build>
+</project>
diff --git a/backend/src/main/java/dev/evolution/monitor/CheckRunner.java b/backend/src/main/java/dev/evolution/monitor/CheckRunner.java
new file mode 100644
index 0000000..d231767
--- /dev/null
+++ b/backend/src/main/java/dev/evolution/monitor/CheckRunner.java
@@ -0,0 +1,75 @@
+package dev.evolution.monitor;
+
+import java.io.IOException;
+import java.net.HttpURLConnection;
+import java.net.Proxy;
+import java.net.SocketTimeoutException;
+import java.net.URI;
+import java.time.Instant;
+import java.util.UUID;
+import org.springframework.beans.factory.annotation.Value;
+import org.springframework.http.HttpStatus;
+import org.springframework.stereotype.Component;
+import org.springframework.web.server.ResponseStatusException;
+
+@Component
+public class CheckRunner {
+    private final URI fixtureOrigin;
+
+    public CheckRunner(@Value("${monitor.fixture-origin}") String fixtureOrigin) {
+        this.fixtureOrigin = URI.create(fixtureOrigin);
+        if (!"http".equals(this.fixtureOrigin.getScheme()) || this.fixtureOrigin.getHost() == null
+                || this.fixtureOrigin.getPort() < 1 || this.fixtureOrigin.getUserInfo() != null) {
+            throw new IllegalArgumentException("Fixture origin must be an explicit http host and port");
+        }
+    }
+
+    public URI requireFixtureUrl(String value) {
+        try {
+            URI url = URI.create(value);
+            if (!"http".equals(url.getScheme()) || !fixtureOrigin.getHost().equals(url.getHost())
+                    || fixtureOrigin.getPort() != url.getPort() || url.getUserInfo() != null
+                    || url.getFragment() != null) {
+                throw new IllegalArgumentException();
+            }
+            return url;
+        } catch (IllegalArgumentException | NullPointerException error) {
+            throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "Only the configured test fixture is allowed");
+        }
+    }
+
+    public CheckRun run(UUID monitorId, String value) {
+        URI url = requireFixtureUrl(value); // Recheck at the actual outbound boundary.
+        Instant startedAt = Instant.now();
+        long startedNanos = System.nanoTime();
+        Integer status = null;
+        String failureReason = null;
+        HttpURLConnection connection = null;
+        try {
+            connection = (HttpURLConnection) url.toURL().openConnection(Proxy.NO_PROXY);
+            connection.setRequestMethod("GET");
+            connection.setInstanceFollowRedirects(false);
+            connection.setConnectTimeout(1000);
+            connection.setReadTimeout(2000);
+            // E01 observes headers only; response bodies are never materialized or retained.
+            status = connection.getResponseCode();
+            if (!isSuccess(status)) failureReason = "HTTP_STATUS";
+        } catch (SocketTimeoutException error) {
+            failureReason = "TIMEOUT";
+        } catch (IOException error) {
+            failureReason = "CONNECTION_FAILURE";
+        } finally {
+            if (connection != null) connection.disconnect();
+        }
+        return new CheckRun(UUID.randomUUID(), monitorId, "MANUAL",
+                failureReason == null ? "SUCCEEDED" : "FAILED", status,
+                (System.nanoTime() - startedNanos) / 1_000_000, failureReason, startedAt, Instant.now());
+    }
+
+    static boolean isSuccess(int status) {
+        return status >= 200 && status < 300;
+    }
+
+    public record CheckRun(UUID id, UUID monitorId, String trigger, String state, Integer httpStatus,
+                           long latencyMs, String failureReason, Instant startedAt, Instant finishedAt) {}
+}
diff --git a/backend/src/main/java/dev/evolution/monitor/MonitorApplication.java b/backend/src/main/java/dev/evolution/monitor/MonitorApplication.java
new file mode 100644
index 0000000..df3836a
--- /dev/null
+++ b/backend/src/main/java/dev/evolution/monitor/MonitorApplication.java
@@ -0,0 +1,11 @@
+package dev.evolution.monitor;
+
+import org.springframework.boot.SpringApplication;
+import org.springframework.boot.autoconfigure.SpringBootApplication;
+
+@SpringBootApplication
+public class MonitorApplication {
+    public static void main(String[] args) {
+        SpringApplication.run(MonitorApplication.class, args);
+    }
+}
diff --git a/backend/src/main/java/dev/evolution/monitor/MonitorController.java b/backend/src/main/java/dev/evolution/monitor/MonitorController.java
new file mode 100644
index 0000000..4e170c3
--- /dev/null
+++ b/backend/src/main/java/dev/evolution/monitor/MonitorController.java
@@ -0,0 +1,60 @@
+package dev.evolution.monitor;
+
+import java.time.Instant;
+import java.util.Comparator;
+import java.util.List;
+import java.util.Map;
+import java.util.UUID;
+import java.util.concurrent.ConcurrentHashMap;
+import org.springframework.http.HttpStatus;
+import org.springframework.web.bind.annotation.GetMapping;
+import org.springframework.web.bind.annotation.PathVariable;
+import org.springframework.web.bind.annotation.PostMapping;
+import org.springframework.web.bind.annotation.RequestBody;
+import org.springframework.web.bind.annotation.RequestMapping;
+import org.springframework.web.bind.annotation.ResponseStatus;
+import org.springframework.web.bind.annotation.RestController;
+import org.springframework.web.server.ResponseStatusException;
+
+@RestController
+@RequestMapping("/api/monitors")
+public class MonitorController {
+    private final Map<UUID, Monitor> monitors = new ConcurrentHashMap<>();
+    private final Map<UUID, CheckRunner.CheckRun> latestChecks = new ConcurrentHashMap<>();
+    private final CheckRunner checks;
+
+    public MonitorController(CheckRunner checks) {
+        this.checks = checks;
+    }
+
+    @GetMapping
+    public List<MonitorView> list() {
+        return monitors.values().stream().sorted(Comparator.comparing(Monitor::createdAt))
+                .map(monitor -> new MonitorView(monitor, latestChecks.get(monitor.id()))).toList();
+    }
+
+    @PostMapping
+    @ResponseStatus(HttpStatus.CREATED)
+    public MonitorView create(@RequestBody CreateMonitor input) {
+        checks.requireFixtureUrl(input.url());
+        Instant now = Instant.now();
+        Monitor monitor = new Monitor(UUID.randomUUID(), input.name(), input.url(), input.interval(),
+                input.enabled(), now, now);
+        monitors.put(monitor.id(), monitor);
+        return new MonitorView(monitor, null);
+    }
+
+    @PostMapping("/{id}/checks")
+    public CheckRunner.CheckRun check(@PathVariable UUID id) {
+        Monitor monitor = monitors.get(id);
+        if (monitor == null) throw new ResponseStatusException(HttpStatus.NOT_FOUND, "Monitor not found");
+        CheckRunner.CheckRun result = checks.run(monitor.id(), monitor.url());
+        latestChecks.put(id, result);
+        return result;
+    }
+
+    public record CreateMonitor(String name, String url, int interval, boolean enabled) {}
+    public record Monitor(UUID id, String name, String url, int interval, boolean enabled,
+                          Instant createdAt, Instant updatedAt) {}
+    public record MonitorView(Monitor monitor, CheckRunner.CheckRun latestCheck) {}
+}
diff --git a/backend/src/main/resources/application.properties b/backend/src/main/resources/application.properties
new file mode 100644
index 0000000..062c3f6
--- /dev/null
+++ b/backend/src/main/resources/application.properties
@@ -0,0 +1,4 @@
+spring.application.name=monitor-api
+server.address=127.0.0.1
+server.port=${API_PORT:4322}
+monitor.fixture-origin=${FIXTURE_ORIGIN:http://127.0.0.1:4321}
diff --git a/backend/src/test/java/dev/evolution/monitor/CheckRunnerTest.java b/backend/src/test/java/dev/evolution/monitor/CheckRunnerTest.java
new file mode 100644
index 0000000..0187dfb
--- /dev/null
+++ b/backend/src/test/java/dev/evolution/monitor/CheckRunnerTest.java
@@ -0,0 +1,28 @@
+package dev.evolution.monitor;
+
+import static org.junit.jupiter.api.Assertions.*;
+import org.junit.jupiter.api.Test;
+import org.springframework.web.server.ResponseStatusException;
+
+class CheckRunnerTest {
+    private final CheckRunner runner = new CheckRunner("http://127.0.0.1:4321");
+
+    @Test
+    void onlyTwoHundredsAreSuccessful() {
+        assertFalse(CheckRunner.isSuccess(199));
+        assertTrue(CheckRunner.isSuccess(200));
+        assertTrue(CheckRunner.isSuccess(299));
+        assertFalse(CheckRunner.isSuccess(302));
+        assertFalse(CheckRunner.isSuccess(503));
+    }
+
+    @Test
+    void destinationIsExactlyConfiguredHostPortAndScheme() {
+        assertEquals("/ok", runner.requireFixtureUrl("http://127.0.0.1:4321/ok").getPath());
+        for (String url : new String[]{"http://127.0.0.1:4324/no", "http://localhost:4321/ok",
+                "https://127.0.0.1:4321/ok", "http://user@127.0.0.1:4321/ok",
+                "http://127.0.0.1:4321/ok#fragment", "not a URL"}) {
+            assertThrows(ResponseStatusException.class, () -> runner.requireFixtureUrl(url));
+        }
+    }
+}
diff --git a/backend/src/test/java/dev/evolution/monitor/MonitorFunctionalTest.java b/backend/src/test/java/dev/evolution/monitor/MonitorFunctionalTest.java
new file mode 100644
index 0000000..67e8e12
--- /dev/null
+++ b/backend/src/test/java/dev/evolution/monitor/MonitorFunctionalTest.java
@@ -0,0 +1,134 @@
+package dev.evolution.monitor;
+
+import static org.junit.jupiter.api.Assertions.*;
+import com.sun.net.httpserver.HttpServer;
+import java.io.IOException;
+import java.net.InetSocketAddress;
+import java.util.concurrent.atomic.AtomicInteger;
+import org.junit.jupiter.api.AfterAll;
+import org.junit.jupiter.api.BeforeAll;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.boot.test.web.client.TestRestTemplate;
+import org.springframework.http.HttpStatus;
+
+@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
+class MonitorFunctionalTest {
+    static HttpServer fixture;
+    static HttpServer forbidden;
+    static final AtomicInteger okRequests = new AtomicInteger();
+    static final AtomicInteger forbiddenRequests = new AtomicInteger();
+    @Autowired TestRestTemplate api;
+
+    @BeforeAll
+    static void startFixture() throws IOException {
+        fixture = HttpServer.create(new InetSocketAddress("127.0.0.1", 4321), 0);
+        fixture.createContext("/ok", exchange -> {
+            okRequests.incrementAndGet();
+            assertEquals("GET", exchange.getRequestMethod());
+            byte[] body = "ok\n".getBytes();
+            exchange.sendResponseHeaders(200, body.length);
+            exchange.getResponseBody().write(body);
+            exchange.close();
+        });
+        fixture.createContext("/fail", exchange -> {
+            byte[] body = "unavailable\n".getBytes();
+            exchange.sendResponseHeaders(503, body.length);
+            exchange.getResponseBody().write(body);
+            exchange.close();
+        });
+        fixture.createContext("/redirect", exchange -> {
+            exchange.getResponseHeaders().set("Location", "/ok");
+            exchange.sendResponseHeaders(302, -1);
+            exchange.close();
+        });
+        fixture.createContext("/redirect-outside", exchange -> {
+            exchange.getResponseHeaders().set("Location", "http://127.0.0.1:4324/blocked");
+            exchange.sendResponseHeaders(302, -1);
+            exchange.close();
+        });
+        forbidden = HttpServer.create(new InetSocketAddress("127.0.0.1", 4324), 0);
+        forbidden.createContext("/", exchange -> {
+            forbiddenRequests.incrementAndGet();
+            exchange.sendResponseHeaders(200, -1);
+            exchange.close();
+        });
+        fixture.start();
+        forbidden.start();
+    }
+
+    @AfterAll
+    static void stopFixture() {
+        if (fixture != null) fixture.stop(0);
+        if (forbidden != null) forbidden.stop(0);
+    }
+
+    @Test
+    void createAndManuallyCheckSuccessfulMonitor() {
+        var view = create("/ok");
+        assertEquals("Fixture monitor", view.monitor().name());
+        assertEquals(60, view.monitor().interval());
+        assertTrue(view.monitor().enabled());
+        var check = check(view);
+        assertEquals("SUCCEEDED", check.state());
+        assertEquals(200, check.httpStatus());
+        assertEquals("MANUAL", check.trigger());
+        assertEquals(view.monitor().id(), check.monitorId());
+        assertNull(check.failureReason());
+        assertNotNull(check.id());
+        assertFalse(check.finishedAt().isBefore(check.startedAt()));
+        assertTrue(check.latencyMs() >= 0);
+        var listed = api.getForObject("/api/monitors", MonitorController.MonitorView[].class);
+        assertTrue(java.util.Arrays.stream(listed).anyMatch(row -> check.equals(row.latestCheck())));
+    }
+
+    @Test
+    void observed503IsAnEndpointFailure() {
+        var result = check(create("/fail"));
+        assertEquals("FAILED", result.state());
+        assertEquals(503, result.httpStatus());
+        assertEquals("HTTP_STATUS", result.failureReason());
+    }
+
+    @Test
+    void redirectsAreNotFollowedEvenInsideFixture() {
+        int before = okRequests.get();
+        var result = check(create("/redirect"));
+        assertEquals("FAILED", result.state());
+        assertEquals(302, result.httpStatus());
+        assertEquals(before, okRequests.get());
+    }
+
+    @Test
+    void redirectsCannotLeaveConfiguredFixture() {
+        var result = check(create("/redirect-outside"));
+        assertEquals(302, result.httpStatus());
+        assertEquals("FAILED", result.state());
+        assertEquals(0, forbiddenRequests.get());
+    }
+
+    @Test
+    void rejectsNonFixtureDestinationWithoutOutboundRequest() {
+        var response = api.postForEntity("/api/monitors", new MonitorController.CreateMonitor(
+                "Forbidden fixture", "http://127.0.0.1:4324/blocked", 60, true), String.class);
+        assertEquals(HttpStatus.BAD_REQUEST, response.getStatusCode());
+        assertEquals(0, forbiddenRequests.get());
+    }
+
+    private MonitorController.MonitorView create(String path) {
+        var response = api.postForEntity("/api/monitors", new MonitorController.CreateMonitor(
+                "Fixture monitor", "http://127.0.0.1:4321" + path, 60, true), MonitorController.MonitorView.class);
+        assertEquals(HttpStatus.CREATED, response.getStatusCode());
+        assertNotNull(response.getBody());
+        return response.getBody();
+    }
+
+    private CheckRunner.CheckRun check(MonitorController.MonitorView monitor) {
+        var response = api.postForEntity("/api/monitors/" + monitor.monitor().id() + "/checks", null,
+                CheckRunner.CheckRun.class);
+        assertEquals(HttpStatus.OK, response.getStatusCode());
+        assertNotNull(response.getBody());
+        return response.getBody();
+    }
+}


