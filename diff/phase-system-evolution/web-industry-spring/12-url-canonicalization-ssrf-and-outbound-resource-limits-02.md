## `feat(e12): adopt preserved outbound destination safeguards`

diff --git a/backend/src/main/java/dev/evolution/monitor/CheckRunner.java b/backend/src/main/java/dev/evolution/monitor/CheckRunner.java
index 59abb52..6ae5ef6 100644
--- a/backend/src/main/java/dev/evolution/monitor/CheckRunner.java
+++ b/backend/src/main/java/dev/evolution/monitor/CheckRunner.java
@@ -1,12 +1,34 @@
 package dev.evolution.monitor;
 
+import jakarta.annotation.PreDestroy;
+import java.io.EOFException;
 import java.io.IOException;
-import java.net.HttpURLConnection;
+import java.io.InputStream;
+import java.net.InetAddress;
+import java.net.InetSocketAddress;
 import java.net.Proxy;
+import java.net.Socket;
 import java.net.SocketTimeoutException;
 import java.net.URI;
+import java.nio.charset.StandardCharsets;
 import java.time.Instant;
+import java.util.List;
+import java.util.Locale;
 import java.util.UUID;
+import java.util.concurrent.ExecutionException;
+import java.util.concurrent.Future;
+import java.util.concurrent.SynchronousQueue;
+import java.util.concurrent.ThreadPoolExecutor;
+import java.util.concurrent.TimeUnit;
+import java.util.concurrent.TimeoutException;
+import java.util.concurrent.atomic.AtomicReference;
+import javax.net.ssl.SNIHostName;
+import javax.net.ssl.SSLParameters;
+import javax.net.ssl.SSLSocket;
+import javax.net.ssl.SSLSocketFactory;
+import org.slf4j.Logger;
+import org.slf4j.LoggerFactory;
+import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.beans.factory.annotation.Value;
 import org.springframework.http.HttpStatus;
 import org.springframework.stereotype.Component;
@@ -14,59 +36,241 @@ import org.springframework.web.server.ResponseStatusException;
 
 @Component
 public class CheckRunner {
-    private final URI fixtureOrigin;
+    static final int CONNECT_MS = 500, READ_MS = 500, TOTAL_MS = 1500, MAX_REDIRECTS = 3, MAX_HEADER_BYTES = 65536;
+    private static final Logger LOG = LoggerFactory.getLogger(CheckRunner.class);
+    private final OutboundUrl urls;
+    private final Resolver resolver;
+    private final Connector connector;
+    private final AtomicReference<Attempt> active = new AtomicReference<>();
+    // DNS may ignore interruption. At most one such task can remain; no queued tasks or replacement threads accumulate.
+    private final ThreadPoolExecutor io = new ThreadPoolExecutor(0, 1, 0, TimeUnit.MILLISECONDS,
+            new SynchronousQueue<>(), runnable -> {
+                Thread thread = new Thread(runnable, "check-outbound");
+                thread.setDaemon(true);
+                return thread;
+            });
 
-    public CheckRunner(@Value("${monitor.fixture-origin}") String fixtureOrigin) {
-        this.fixtureOrigin = URI.create(fixtureOrigin);
-        if (!"http".equals(this.fixtureOrigin.getScheme()) || this.fixtureOrigin.getHost() == null
-                || this.fixtureOrigin.getPort() < 1 || this.fixtureOrigin.getUserInfo() != null) {
-            throw new IllegalArgumentException("Fixture origin must be an explicit http host and port");
-        }
+    @Autowired
+    public CheckRunner(@Value("${monitor.fixture-origin}") String fixtureOrigin,
+            @Value("${monitor.allow-test-fixture:false}") boolean allowTestFixture) {
+        this(fixtureOrigin, allowTestFixture, InetAddress::getAllByName, CheckRunner::connect);
+    }
+
+    CheckRunner(String fixtureOrigin, boolean allowTestFixture, Resolver resolver, Connector connector) {
+        urls = new OutboundUrl(fixtureOrigin, allowTestFixture);
+        this.resolver = resolver;
+        this.connector = connector;
     }
 
-    public URI requireFixtureUrl(String value) {
+    public URI canonicalUrl(String value) {
         try {
-            URI url = URI.create(value);
-            if (!"http".equals(url.getScheme()) || !fixtureOrigin.getHost().equals(url.getHost())
-                    || fixtureOrigin.getPort() != url.getPort() || url.getUserInfo() != null
-                    || url.getFragment() != null) {
-                throw new IllegalArgumentException();
-            }
-            return url;
-        } catch (IllegalArgumentException | NullPointerException error) {
-            throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "Only the configured test fixture is allowed");
+            return urls.canonical(value);
+        } catch (OutboundUrl.Blocked error) {
+            throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "URL is not an allowed HTTP or HTTPS destination");
         }
     }
 
     public CheckRun run(CheckRun execution, String value) {
-        URI url = requireFixtureUrl(value); // Recheck at the actual outbound boundary.
         long startedNanos = System.nanoTime();
+        Attempt attempt = new Attempt(startedNanos + TimeUnit.MILLISECONDS.toNanos(TOTAL_MS));
+        // Rejection is service uncertainty: do not manufacture an endpoint result or retry.
+        Future<Integer> task = io.submit(() -> {
+            active.set(attempt);
+            try {
+                if (io.isShutdown()) throw new IllegalStateException("Outbound observation stopped");
+                return observe(value, attempt);
+            } finally {
+                attempt.closeSocket();
+                active.compareAndSet(attempt, null);
+            }
+        });
         Integer status = null;
         String failureReason = null;
-        HttpURLConnection connection = null;
         try {
-            connection = (HttpURLConnection) url.toURL().openConnection(Proxy.NO_PROXY);
-            connection.setRequestMethod("GET");
-            connection.setInstanceFollowRedirects(false);
-            connection.setConnectTimeout(1000);
-            connection.setReadTimeout(2000);
-            // E01 observes headers only; response bodies are never materialized or retained.
-            status = connection.getResponseCode();
-            if (!isSuccess(status)) failureReason = "HTTP_STATUS";
-        } catch (SocketTimeoutException error) {
-            failureReason = "TIMEOUT";
-        } catch (IOException error) {
-            failureReason = "CONNECTION_FAILURE";
-        } finally {
-            if (connection != null) connection.disconnect();
+            status = task.get(Math.max(0, attempt.remainingNanos()), TimeUnit.NANOSECONDS);
+        } catch (TimeoutException error) {
+            attempt.cancel();
+            task.cancel(true);
+            status = attempt.finalStatus();
+            if (status == null) failureReason = "TIMEOUT";
+        } catch (InterruptedException error) {
+            attempt.cancel();
+            task.cancel(true);
+            Thread.currentThread().interrupt();
+            throw new IllegalStateException("Outbound observation interrupted", error);
+        } catch (ExecutionException error) {
+            Throwable cause = error.getCause();
+            if (cause instanceof OutboundUrl.Blocked blocked) {
+                // ABORTED deliberately has no invented endpoint/latency fields (the existing E11 wire contract).
+                LOG.info("Check policy refused: {}", blocked.reason);
+                return new CheckRun(execution.id(), execution.monitorId(), execution.trigger(), "ABORTED",
+                        null, null, null, execution.startedAt(), Instant.now());
+            }
+            if (cause instanceof SocketTimeoutException) failureReason = "TIMEOUT";
+            else if (cause instanceof IOException) failureReason = "CONNECTION_FAILURE";
+            else if (cause instanceof RuntimeException runtime) throw runtime;
+            else if (cause instanceof Error fatal) throw fatal;
+            else throw new IllegalStateException("Outbound observation failed", cause);
         }
+        if (status != null && !isSuccess(status)) failureReason = "HTTP_STATUS";
         return new CheckRun(execution.id(), execution.monitorId(), execution.trigger(),
                 failureReason == null ? "SUCCEEDED" : "FAILED", status,
                 (System.nanoTime() - startedNanos) / 1_000_000, failureReason, execution.startedAt(), Instant.now());
     }
 
-    static boolean isSuccess(int status) {
-        return status >= 200 && status < 300;
+    private int observe(String value, Attempt attempt) throws IOException {
+        URI url = urls.canonical(value); // Validate again at the actual outbound boundary, including every redirect.
+        for (int redirects = 0; ; redirects++) {
+            attempt.check();
+            InetAddress[] addresses = resolver.resolve(OutboundUrl.host(url));
+            attempt.check();
+            urls.requireAddresses(url, addresses);
+            Socket socket = connector.connect(url, addresses[0], attempt);
+            try {
+                attempt.check();
+                String path = url.getRawPath() + (url.getRawQuery() == null ? "" : "?" + url.getRawQuery());
+                String request = "GET " + path + " HTTP/1.1\r\nHost: " + url.getRawAuthority()
+                        + "\r\nConnection: close\r\n\r\n";
+                socket.getOutputStream().write(request.getBytes(StandardCharsets.US_ASCII));
+                socket.setSoTimeout(attempt.timeout(READ_MS));
+                Headers headers = readHeaders(socket.getInputStream(), attempt);
+                if (isRedirect(headers.status()) && headers.location() != null) {
+                    if (redirects == MAX_REDIRECTS) throw new OutboundUrl.Blocked("REDIRECT_LIMIT");
+                    try {
+                        url = urls.canonical(url.resolve(headers.location()).toASCIIString());
+                    } catch (IllegalArgumentException error) {
+                        if (error instanceof OutboundUrl.Blocked blocked) throw blocked;
+                        throw new OutboundUrl.Blocked("INVALID_REDIRECT");
+                    }
+                } else {
+                    attempt.publishFinal(headers.status());
+                    return headers.status();
+                }
+            } finally {
+                // Final headers are authoritative. Read zero body bytes, including for oversized/trickling bodies.
+                attempt.closeSocket();
+                close(socket);
+            }
+        }
+    }
+
+    static Socket connect(URI url, InetAddress address, Attempt attempt) throws IOException {
+        Socket raw = new Socket(Proxy.NO_PROXY);
+        attempt.register(raw); // Register before blocking connect, so the total deadline can close actual I/O.
+        raw.connect(new InetSocketAddress(address, OutboundUrl.port(url)), attempt.timeout(CONNECT_MS));
+        raw.setSoTimeout(attempt.timeout(READ_MS));
+        if (!url.getScheme().equals("https")) return raw;
+        String hostname = OutboundUrl.host(url);
+        // The socket is already connected to the validated IP. Hostname is used only for TLS identity/SNI.
+        SSLSocketFactory factory = (SSLSocketFactory) SSLSocketFactory.getDefault();
+        SSLSocket tls = (SSLSocket) factory.createSocket(raw, hostname, OutboundUrl.port(url), true);
+        tls.setSSLParameters(tlsParameters(url));
+        tls.setSoTimeout(attempt.timeout(READ_MS));
+        tls.startHandshake();
+        return tls;
+    }
+
+    static SSLParameters tlsParameters(URI url) {
+        SSLParameters parameters = new SSLParameters();
+        parameters.setEndpointIdentificationAlgorithm("HTTPS");
+        String hostname = OutboundUrl.host(url);
+        if (hostname.indexOf(':') < 0 && !hostname.matches("[0-9.]+")) {
+            parameters.setServerNames(List.of(new SNIHostName(hostname)));
+        }
+        return parameters;
+    }
+
+    private static Headers readHeaders(InputStream input, Attempt attempt) throws IOException {
+        byte[] bytes = new byte[MAX_HEADER_BYTES];
+        int total = 0;
+        while (true) {
+            int start = total;
+            while (true) {
+                attempt.check();
+                if (total == bytes.length) throw new OutboundUrl.Blocked("HEADER_LIMIT");
+                int next = input.read();
+                if (next < 0) throw new EOFException("Incomplete HTTP headers");
+                bytes[total++] = (byte) next;
+                if (total - start >= 4 && bytes[total - 4] == '\r' && bytes[total - 3] == '\n'
+                        && bytes[total - 2] == '\r' && bytes[total - 1] == '\n') break;
+            }
+            String[] lines = new String(bytes, start, total - start - 4, StandardCharsets.ISO_8859_1).split("\r\n");
+            if (!lines[0].matches("HTTP/1\\.[01] [1-5][0-9]{2}( .*)?")) throw new IOException("Invalid HTTP status line");
+            int status = Integer.parseInt(lines[0].substring(9, 12));
+            String location = null;
+            for (int i = 1; i < lines.length; i++) {
+                int colon = lines[i].indexOf(':');
+                if (colon < 1 || !lines[i].substring(0, colon).matches("[!#$%&'*+.^_`|~0-9A-Za-z-]+")) {
+                    throw new IOException("Invalid HTTP header");
+                }
+                String content = lines[i].substring(colon + 1).strip();
+                for (int j = 0; j < content.length(); j++) {
+                    char character = content.charAt(j);
+                    if ((character < 32 && character != '\t') || character == 127) throw new IOException("Invalid HTTP header value");
+                }
+                if (lines[i].substring(0, colon).toLowerCase(Locale.ROOT).equals("location")) {
+                    if (location != null) throw new OutboundUrl.Blocked("INVALID_REDIRECT");
+                    location = content;
+                }
+            }
+            if (status == 101) throw new IOException("HTTP upgrade is not supported");
+            if (status >= 200) return new Headers(status, location);
+            // Informational headers are not an endpoint result; their bytes share this hop's finite budget.
+        }
+    }
+
+    private static boolean isRedirect(int status) {
+        return status == 301 || status == 302 || status == 303 || status == 307 || status == 308;
+    }
+
+    @PreDestroy
+    public void close() {
+        Attempt attempt = active.get();
+        if (attempt != null) attempt.cancel();
+        io.shutdownNow();
+    }
+
+    private static void close(Socket socket) {
+        try { socket.close(); } catch (IOException ignored) { /* No response-body/close error changes final headers. */ }
+    }
+
+    static boolean isSuccess(int status) { return status >= 200 && status < 300; }
+    private record Headers(int status, String location) {}
+    @FunctionalInterface interface Resolver { InetAddress[] resolve(String hostname) throws IOException; }
+    @FunctionalInterface interface Connector { Socket connect(URI url, InetAddress address, Attempt attempt) throws IOException; }
+
+    static final class Attempt {
+        private final long deadline;
+        private final AtomicReference<Socket> socket = new AtomicReference<>();
+        private boolean cancelled;
+        private Integer finalStatus;
+
+        Attempt(long deadline) { this.deadline = deadline; }
+        long remainingNanos() { return deadline - System.nanoTime(); }
+        synchronized void check() throws SocketTimeoutException {
+            if (cancelled || remainingNanos() <= 0) throw new SocketTimeoutException("Outbound deadline exceeded");
+        }
+        int timeout(int maximum) throws SocketTimeoutException {
+            check();
+            return (int) Math.max(1, Math.min(maximum, TimeUnit.NANOSECONDS.toMillis(remainingNanos()) + 1));
+        }
+        void register(Socket value) throws IOException {
+            socket.set(value);
+            try { check(); } catch (IOException error) { closeSocket(); throw error; }
+        }
+        synchronized void publishFinal(int status) throws SocketTimeoutException {
+            check();
+            finalStatus = status;
+        }
+        synchronized Integer finalStatus() { return finalStatus; }
+        void cancel() {
+            synchronized (this) { cancelled = true; }
+            closeSocket();
+        }
+        void closeSocket() {
+            Socket value = socket.getAndSet(null);
+            if (value != null) close(value);
+        }
     }
 
     public record CheckRun(UUID id, UUID monitorId, String trigger, String state, Integer httpStatus,
diff --git a/backend/src/main/java/dev/evolution/monitor/MonitorController.java b/backend/src/main/java/dev/evolution/monitor/MonitorController.java
index c7c4773..a14cc37 100644
--- a/backend/src/main/java/dev/evolution/monitor/MonitorController.java
+++ b/backend/src/main/java/dev/evolution/monitor/MonitorController.java
@@ -40,8 +40,8 @@ public class MonitorController {
     @ResponseStatus(HttpStatus.CREATED)
     public ApiData<MonitorView> create(@AuthenticationPrincipal UserAccounts.AccountUser user, @RequestBody JsonNode body) {
         CreateMonitor input = CreateMonitor.fromJson(body);
-        checks.requireFixtureUrl(input.url());
-        return new ApiData<>(store.create(user.userId(), input));
+        String url = checks.canonicalUrl(input.url()).toASCIIString();
+        return new ApiData<>(store.create(user.userId(), new CreateMonitor(input.name(), url, input.interval(), input.enabled())));
     }
 
     @GetMapping("/{id}")
@@ -53,8 +53,9 @@ public class MonitorController {
     public ApiData<MonitorView> replace(@AuthenticationPrincipal UserAccounts.AccountUser user,
             @PathVariable UUID id, @RequestBody JsonNode body) {
         CreateMonitor input = CreateMonitor.fromJson(body);
-        checks.requireFixtureUrl(input.url());
-        return new ApiData<>(store.replace(user.userId(), id, input));
+        String url = checks.canonicalUrl(input.url()).toASCIIString();
+        return new ApiData<>(store.replace(user.userId(), id,
+                new CreateMonitor(input.name(), url, input.interval(), input.enabled())));
     }
 
     @DeleteMapping("/{id}")
diff --git a/backend/src/main/java/dev/evolution/monitor/OutboundUrl.java b/backend/src/main/java/dev/evolution/monitor/OutboundUrl.java
new file mode 100644
index 0000000..97e32dd
--- /dev/null
+++ b/backend/src/main/java/dev/evolution/monitor/OutboundUrl.java
@@ -0,0 +1,129 @@
+package dev.evolution.monitor;
+
+import java.net.InetAddress;
+import java.net.URI;
+import java.net.UnknownHostException;
+import java.util.Locale;
+
+/** URL syntax and address decisions; constructing a Monitor never performs DNS. */
+final class OutboundUrl {
+    private final URI fixture;
+    private final boolean allowTestFixture;
+
+    OutboundUrl(String fixtureOrigin, boolean allowTestFixture) {
+        fixture = URI.create(fixtureOrigin);
+        this.allowTestFixture = allowTestFixture;
+        InetAddress address = literal(host(fixture));
+        if (!"http".equals(fixture.getScheme()) || address == null || !address.isLoopbackAddress()
+                || fixture.getPort() < 1 || fixture.getPort() > 65535 || fixture.getUserInfo() != null
+                || fixture.getRawQuery() != null || fixture.getRawFragment() != null
+                || !(fixture.getRawPath().isEmpty() || fixture.getRawPath().equals("/"))) {
+            throw new IllegalArgumentException("Test fixture must be an explicit loopback HTTP origin");
+        }
+    }
+
+    URI canonical(String value) {
+        try {
+            URI input = URI.create(value);
+            String scheme = input.getScheme() == null ? "" : input.getScheme().toLowerCase(Locale.ROOT);
+            String host = host(input).toLowerCase(Locale.ROOT);
+            if (host.endsWith(".")) host = host.substring(0, host.length() - 1);
+            if (!(scheme.equals("http") || scheme.equals("https")) || host.isEmpty()
+                    || input.getRawUserInfo() != null || input.getRawFragment() != null
+                    || host.indexOf('%') >= 0 || input.getPort() == 0 || input.getPort() > 65535
+                    || input.getRawAuthority().endsWith(":")) {
+                throw new Blocked("INVALID_URL");
+            }
+            int port = input.getPort();
+            if ((scheme.equals("http") && port == 80) || (scheme.equals("https") && port == 443)) port = -1;
+            String authority = host.indexOf(':') >= 0 ? "[" + host + "]" : host;
+            if (port >= 0) authority += ":" + port;
+            String path = input.getRawPath();
+            URI result = URI.create(scheme + "://" + authority + (path.isEmpty() ? "/" : path)
+                    + (input.getRawQuery() == null ? "" : "?" + input.getRawQuery())).normalize();
+            InetAddress address = literal(host);
+            if (!isFixture(result) && (host.equals("localhost") || host.endsWith(".localhost")
+                    || (address != null && !isPublic(address)))) {
+                throw new Blocked("UNSAFE_ADDRESS");
+            }
+            return URI.create(result.toASCIIString());
+        } catch (IllegalArgumentException | NullPointerException error) {
+            if (error instanceof Blocked blocked) throw blocked;
+            throw new Blocked("INVALID_URL");
+        }
+    }
+
+    void requireAddresses(URI url, InetAddress[] addresses) {
+        if (addresses == null || addresses.length == 0) throw new Blocked("EMPTY_DNS_ANSWER");
+        for (InetAddress address : addresses) {
+            if (address == null || (!(isFixture(url) && address.isLoopbackAddress()) && !isPublic(address))) {
+                throw new Blocked("UNSAFE_DNS_ANSWER");
+            }
+        }
+    }
+
+    boolean isFixture(URI url) {
+        return allowTestFixture && fixture.getScheme().equals(url.getScheme())
+                && fixture.getHost().equalsIgnoreCase(url.getHost()) && port(fixture) == port(url);
+    }
+
+    static String host(URI url) {
+        String host = url.getHost();
+        if (host == null) return "";
+        return host.startsWith("[") ? host.substring(1, host.length() - 1) : host;
+    }
+
+    static int port(URI url) {
+        return url.getPort() < 0 ? (url.getScheme().equals("https") ? 443 : 80) : url.getPort();
+    }
+
+    private static InetAddress literal(String host) {
+        try {
+            // A colon forces the JDK's numeric IPv6 parser; no hostname lookup is used here.
+            if (host.indexOf(':') >= 0) {
+                if (host.indexOf('%') >= 0) throw new Blocked("INVALID_URL");
+                return InetAddress.getByName(host);
+            }
+            if (!host.matches("(?i)(?:0x[0-9a-f]+|[0-9]+)(?:\\.(?:0x[0-9a-f]+|[0-9]+))*")) return null;
+            String[] parts = host.split("\\.", -1);
+            if (parts.length != 4) throw new Blocked("AMBIGUOUS_ADDRESS");
+            byte[] bytes = new byte[4];
+            for (int i = 0; i < 4; i++) {
+                if (!parts[i].matches("0|[1-9][0-9]{0,2}")) throw new Blocked("AMBIGUOUS_ADDRESS");
+                int value = Integer.parseInt(parts[i]);
+                if (value > 255) throw new Blocked("AMBIGUOUS_ADDRESS");
+                bytes[i] = (byte) value;
+            }
+            return InetAddress.getByAddress(bytes);
+        } catch (UnknownHostException error) {
+            throw new Blocked("INVALID_URL");
+        }
+    }
+
+    static boolean isPublic(InetAddress address) {
+        byte[] bytes = address.getAddress();
+        int a = Byte.toUnsignedInt(bytes[0]), b = Byte.toUnsignedInt(bytes[1]);
+        int c = Byte.toUnsignedInt(bytes[2]);
+        if (bytes.length == 4) {
+            // IANA special-use IPv4 blocks, including documentation, shared space and multicast.
+            return !(a == 0 || a == 10 || a == 127 || a >= 224
+                    || (a == 100 && b >= 64 && b <= 127) || (a == 169 && b == 254)
+                    || (a == 172 && b >= 16 && b <= 31)
+                    || (a == 192 && (b == 168 || b == 0 && (c == 0 || c == 2) || b == 88 && c == 99))
+                    || (a == 198 && (b == 18 || b == 19 || b == 51 && c == 100))
+                    || (a == 203 && b == 0 && c == 113));
+        }
+        // Only ordinary global unicast; reject protocol assignments, documentation and 6to4.
+        // This also denies mapped/compatible IPv4, NAT64, ULA, link-local and multicast IPv6.
+        return bytes.length == 16 && a >= 0x20 && a <= 0x3f
+                && !(a == 0x20 && b == 0x01 && (c & 0xfe) == 0)
+                && !(a == 0x20 && b == 0x01 && c == 0x0d && Byte.toUnsignedInt(bytes[3]) == 0xb8)
+                && !(a == 0x20 && b == 0x02)
+                && !(a == 0x3f && b == 0xff && (c & 0xf0) == 0);
+    }
+
+    static final class Blocked extends IllegalArgumentException {
+        final String reason;
+        Blocked(String reason) { super(reason); this.reason = reason; }
+    }
+}
diff --git a/backend/src/main/resources/application.properties b/backend/src/main/resources/application.properties
index cce4f2a..8387302 100644
--- a/backend/src/main/resources/application.properties
+++ b/backend/src/main/resources/application.properties
@@ -2,6 +2,7 @@ spring.application.name=monitor-api
 server.address=127.0.0.1
 server.port=${API_PORT:4322}
 monitor.fixture-origin=${FIXTURE_ORIGIN:http://127.0.0.1:4321}
+monitor.allow-test-fixture=${ALLOW_TEST_FIXTURE:false}
 spring.jackson.deserialization.fail-on-trailing-tokens=true
 spring.datasource.url=${DB_URL:jdbc:postgresql://127.0.0.1:15432/monitor}
 spring.datasource.username=${DB_USER:wse_industry}
diff --git a/backend/src/test/java/dev/evolution/monitor/ApiErrorBoundaryTest.java b/backend/src/test/java/dev/evolution/monitor/ApiErrorBoundaryTest.java
index 6b29ca3..1812e95 100644
--- a/backend/src/test/java/dev/evolution/monitor/ApiErrorBoundaryTest.java
+++ b/backend/src/test/java/dev/evolution/monitor/ApiErrorBoundaryTest.java
@@ -15,7 +15,7 @@ class ApiErrorBoundaryTest {
     @Test
     void unexpectedFailureIsAnInternalErrorWithoutPrivateExceptionDetails() throws Exception {
         CheckRunner checks = mock(CheckRunner.class);
-        when(checks.requireFixtureUrl("http://127.0.0.1:4321/ok"))
+        when(checks.canonicalUrl("http://127.0.0.1:4321/ok"))
                 .thenThrow(new IllegalStateException("Private implementation detail"));
         var mvc = MockMvcBuilders.standaloneSetup(new MonitorController(checks, mock(MonitorStore.class)))
                 .setCustomArgumentResolvers(new AuthenticationPrincipalArgumentResolver())
diff --git a/backend/src/test/java/dev/evolution/monitor/CheckRunnerTest.java b/backend/src/test/java/dev/evolution/monitor/CheckRunnerTest.java
index 1c409ef..47b6369 100644
--- a/backend/src/test/java/dev/evolution/monitor/CheckRunnerTest.java
+++ b/backend/src/test/java/dev/evolution/monitor/CheckRunnerTest.java
@@ -4,17 +4,60 @@ import static org.junit.jupiter.api.Assertions.*;
 import com.fasterxml.jackson.databind.JsonNode;
 import com.fasterxml.jackson.databind.ObjectMapper;
 import com.fasterxml.jackson.databind.SerializationFeature;
-import com.sun.net.httpserver.HttpServer;
+import java.io.ByteArrayInputStream;
+import java.io.ByteArrayOutputStream;
+import java.io.FilterInputStream;
+import java.io.IOException;
+import java.io.InputStream;
+import java.io.OutputStream;
+import java.net.InetAddress;
 import java.net.InetSocketAddress;
 import java.net.ServerSocket;
+import java.net.Socket;
+import java.net.SocketException;
+import java.net.URI;
+import java.nio.charset.StandardCharsets;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.security.MessageDigest;
 import java.time.Instant;
+import java.util.ArrayList;
+import java.util.HexFormat;
+import java.util.LinkedHashMap;
+import java.util.List;
+import java.util.Map;
 import java.util.UUID;
 import java.util.concurrent.CountDownLatch;
+import java.util.concurrent.RejectedExecutionException;
+import java.util.concurrent.TimeUnit;
+import java.util.concurrent.atomic.AtomicInteger;
+import java.util.concurrent.atomic.AtomicReference;
+import javax.net.ssl.SNIHostName;
+import org.junit.jupiter.api.AfterAll;
+import org.junit.jupiter.api.AfterEach;
 import org.junit.jupiter.api.Test;
 import org.springframework.web.server.ResponseStatusException;
 
 class CheckRunnerTest {
-    private final CheckRunner runner = new CheckRunner("http://127.0.0.1:4321");
+    private static final String PUBLIC = "http://public.e12.test/ok";
+    private static final String OK = "HTTP/1.1 200 OK\r\nContent-Length: 0\r\n\r\n";
+    private static final List<Map<String, Object>> observations = new ArrayList<>();
+    private final List<CheckRunner> runners = new ArrayList<>();
+    private final CheckRunner runner = owned(new CheckRunner("http://127.0.0.1:4321", true));
+
+    @AfterEach void closeRunners() { runners.forEach(CheckRunner::close); }
+
+    @AfterAll static void saveObservations() throws Exception {
+        Path directory = Path.of("../output/phase-1/e12");
+        Files.createDirectories(directory);
+        Map<String, Object> evidence = new LinkedHashMap<>();
+        evidence.put("fixtureSha256", HexFormat.of().formatHex(MessageDigest.getInstance("SHA-256")
+                .digest(Files.readAllBytes(Path.of("../evidence/phase-1/E12/fixtures.md")))));
+        evidence.put("observations", observations);
+        evidence.put("externalNetworkUsed", false);
+        Files.writeString(directory.resolve("outbound.json"), new ObjectMapper().writerWithDefaultPrettyPrinter()
+                .writeValueAsString(evidence) + "\n");
+    }
 
     @Test
     void onlyTwoHundredsAreSuccessful() {
@@ -26,13 +69,15 @@ class CheckRunnerTest {
     }
 
     @Test
-    void destinationIsExactlyConfiguredHostPortAndScheme() {
-        assertEquals("/ok", runner.requireFixtureUrl("http://127.0.0.1:4321/ok").getPath());
+    void fixtureExceptionIsExplicitAndStillExactlyConfiguredHostPortAndScheme() {
+        assertEquals("/ok", runner.canonicalUrl("http://127.0.0.1:4321/ok").getPath());
         for (String url : new String[]{"http://127.0.0.1:4324/no", "http://localhost:4321/ok",
                 "https://127.0.0.1:4321/ok", "http://user@127.0.0.1:4321/ok",
                 "http://127.0.0.1:4321/ok#fragment", "not a URL"}) {
-            assertThrows(ResponseStatusException.class, () -> runner.requireFixtureUrl(url));
+            assertThrows(ResponseStatusException.class, () -> runner.canonicalUrl(url));
         }
+        var production = owned(new CheckRunner("http://127.0.0.1:4321", false));
+        assertThrows(ResponseStatusException.class, () -> production.canonicalUrl("http://127.0.0.1:4321/ok"));
     }
 
     @Test
@@ -41,32 +86,334 @@ class CheckRunnerTest {
         try (ServerSocket closedFixture = new ServerSocket()) {
             closedFixture.bind(new InetSocketAddress("127.0.0.1", 4325));
         }
-        var result = new CheckRunner("http://127.0.0.1:4325").run(
+        long started = System.nanoTime();
+        var result = owned(new CheckRunner("http://127.0.0.1:4325", true)).run(
                 execution(), "http://127.0.0.1:4325/ok");
+        record("closed-local-port", result, started, Map.of("transport", "actual local TCP", "port", 4325));
         assertNoResponse(result, "CONNECTION_FAILURE");
+        assertTrue(elapsed(started) < 1750);
     }
 
     @Test
     void headerTimeoutHasNoInventedHttpStatusOnTheWire() throws Exception {
-        var releaseHeaders = new CountDownLatch(1);
-        var fixture = HttpServer.create(new InetSocketAddress("127.0.0.1", 4325), 0);
-        fixture.createContext("/stall", exchange -> {
-            try {
-                releaseHeaders.await();
-            } catch (InterruptedException interrupted) {
-                Thread.currentThread().interrupt();
-            } finally {
-                exchange.close();
-            }
+        try (var fixture = new LocalFixture("slow")) {
+            long started = System.nanoTime();
+            var result = owned(new CheckRunner("http://127.0.0.1:4325", true)).run(execution(), "http://127.0.0.1:4325/stall");
+            record("slow-headers", result, started, Map.of("transport", "actual local HTTP", "delayMs", 2000,
+                    "requests", fixture.paths.size(), "responseHeadersSent", false));
+            assertNoResponse(result, "TIMEOUT");
+            assertEquals(1, fixture.paths.size());
+            assertTrue(elapsed(started) < 1000);
+        }
+    }
+
+    @Test
+    void canonicalUrlsPerformNoDnsOrConnectorWork() {
+        AtomicInteger resolutions = new AtomicInteger(), connections = new AtomicInteger();
+        var checks = stub(host -> { resolutions.incrementAndGet(); throw new AssertionError("No create-time DNS"); },
+                (url, address, attempt) -> { connections.incrementAndGet(); throw new AssertionError("No create-time I/O"); });
+        assertEquals(URI.create(PUBLIC), checks.canonicalUrl("HTTP://PUBLIC.E12.TEST:80/a/../ok"));
+        assertEquals(URI.create("https://public.e12.test/?a=%2F"), checks.canonicalUrl("HTTPS://PUBLIC.E12.TEST:443?a=%2F"));
+        for (String url : List.of("http://user@public.e12.test/ok", "file:///etc/hosts", "http://public.e12.test/ok#fragment",
+                "http://2130706433/ok", "http://0177.0.0.1/ok", "http://[fe80::1%25lo0]/ok")) {
+            assertThrows(ResponseStatusException.class, () -> checks.canonicalUrl(url));
+        }
+        assertEquals(0, resolutions.get());
+        assertEquals(0, connections.get());
+        observations.add(new LinkedHashMap<>(Map.of("case", "canonical-create-boundary", "canonicalUrl", PUBLIC,
+                "dnsCalls", resolutions.get(), "connectorCalls", connections.get())));
+    }
+
+    @Test
+    void publicAnswersUseValidatedAddressesAndOriginalTlsHostnameWithoutSecondDns() throws Exception {
+        for (String addressText : List.of("93.184.216.34", "2606:4700:4700::1111")) {
+            InetAddress address = InetAddress.getByName(addressText); // Numeric fixture, never external DNS.
+            String url = addressText.contains(":") ? "https://public.e12.test/ok" : PUBLIC;
+            AtomicInteger resolutions = new AtomicInteger(), connections = new AtomicInteger();
+            AtomicReference<InetAddress> connected = new AtomicReference<>();
+            var socket = new ObservedSocket(OK);
+            var checks = stub(host -> {
+                assertEquals("public.e12.test", host);
+                return new InetAddress[]{resolutions.incrementAndGet() == 1 ? address : InetAddress.getByName("10.0.0.1")};
+            }, (target, validated, attempt) -> {
+                assertEquals(URI.create(url), target);
+                connected.set(validated);
+                connections.incrementAndGet();
+                attempt.register(socket);
+                return socket;
+            });
+            long started = System.nanoTime();
+            var result = checks.run(execution(), url);
+            record("validated-" + addressText, result, started, Map.of("transport", "connector stub; no live TLS",
+                    "dnsCalls", resolutions.get(), "connectorCalls", connections.get(), "connectedAddress", connected.get().getHostAddress(),
+                    "logicalHost", "public.e12.test", "bodyBytesRead", socket.readBytes.get() - OK.length(), "socketClosed", socket.closed()));
+            assertEquals("SUCCEEDED", result.state());
+            assertEquals(200, result.httpStatus());
+            assertEquals(address, connected.get());
+            assertEquals(1, resolutions.get());
+            assertEquals(1, connections.get());
+            assertEquals(OK.length(), socket.readBytes.get());
+            assertTrue(socket.closed());
+            assertTrue(socket.request.toString(StandardCharsets.US_ASCII).contains("Host: public.e12.test\r\n"));
+            assertTrue(elapsed(started) < 1750);
+        }
+        var parameters = CheckRunner.tlsParameters(URI.create("https://public.e12.test/ok"));
+        assertEquals("HTTPS", parameters.getEndpointIdentificationAlgorithm());
+        assertEquals("public.e12.test", ((SNIHostName) parameters.getServerNames().getFirst()).getAsciiName());
+        observations.add(new LinkedHashMap<>(Map.of("case", "TLS-configuration-only", "endpointIdentification", "HTTPS",
+                "sniHost", "public.e12.test", "liveTlsHandshakeTested", false)));
+    }
+
+    @Test
+    void everyUnsafeLiteralAndActualDnsAnswerIsRefusedBeforeConnection() throws Exception {
+        for (String address : List.of("127.0.0.1", "::1", "10.0.0.1", "fc00::1", "169.254.169.254", "fe80::1", "::ffff:127.0.0.1")) {
+            AtomicInteger connections = new AtomicInteger();
+            var checks = stub(host -> new InetAddress[]{InetAddress.getByName(address)}, (url, resolved, attempt) -> {
+                connections.incrementAndGet(); throw new AssertionError("Unsafe connector must never run");
+            });
+            String literal = "http://" + (address.contains(":") ? "[" + address + "]" : address) + "/ok";
+            assertThrows(ResponseStatusException.class, () -> checks.canonicalUrl(literal));
+            long started = System.nanoTime();
+            var result = checks.run(execution(), PUBLIC);
+            record("unsafe-answer-" + address, result, started, Map.of("transport", "numeric resolver stub",
+                    "unsafeConnectorCalls", connections.get()));
+            assertAborted(result);
+            assertEquals(0, connections.get());
+        }
+        for (String hostname : List.of("private.e12.test", "mixed.e12.test")) {
+            AtomicInteger connections = new AtomicInteger();
+            var checks = stub(host -> host.equals("private.e12.test")
+                    ? new InetAddress[]{InetAddress.getByName("10.0.0.1")}
+                    : new InetAddress[]{InetAddress.getByName("93.184.216.34"), InetAddress.getByName("10.0.0.1")},
+                    (url, address, attempt) -> { connections.incrementAndGet(); throw new AssertionError("No mixed-answer I/O"); });
+            long started = System.nanoTime();
+            var result = checks.run(execution(), "http://" + hostname + "/ok");
+            record(hostname, result, started, Map.of("transport", "resolver stub", "unsafeConnectorCalls", connections.get()));
+            assertAborted(result);
+            assertEquals(0, connections.get());
+        }
+    }
+
+    @Test
+    void aRedirectToPrivateSpaceIsAbortedWithoutAnIntermediateHttpOutcome() throws Exception {
+        String response = "HTTP/1.1 302 Found\r\nLocation: http://10.0.0.1/ok\r\nContent-Length: 0\r\n\r\n";
+        var socket = new ObservedSocket(response);
+        AtomicInteger connections = new AtomicInteger();
+        var checks = stub(host -> new InetAddress[]{InetAddress.getByName("93.184.216.34")}, (url, address, attempt) -> {
+            assertTrue(OutboundUrl.isPublic(address));
+            connections.incrementAndGet();
+            attempt.register(socket);
+            return socket;
         });
-        fixture.start();
+        long started = System.nanoTime();
+        var result = checks.run(execution(), "http://public.e12.test/private");
+        record("redirect-private", result, started, Map.of("transport", "connector stub", "safeConnectorCalls", connections.get(),
+                "unsafeConnectorCalls", 0, "socketClosed", socket.closed()));
+        assertAborted(result);
+        assertEquals(1, connections.get());
+        assertTrue(socket.closed());
+    }
+
+    @Test
+    void realLocalResponsesRespectFinalHeadersTotalDeadlineAndRedirectBudget() throws Exception {
+        for (String mode : List.of("ok", "body", "informational", "trickle", "redirect")) {
+            List<ObservedSocket> sockets = java.util.Collections.synchronizedList(new ArrayList<>());
+            try (var fixture = new LocalFixture(mode)) {
+                var checks = owned(new CheckRunner("http://127.0.0.1:4325", true, InetAddress::getAllByName,
+                        (url, address, attempt) -> {
+                            var socket = new ObservedSocket(CheckRunner.connect(url, address, attempt));
+                            sockets.add(socket);
+                            return socket;
+                        }));
+                long started = System.nanoTime();
+                var result = checks.run(execution(), "http://127.0.0.1:4325/" + (mode.equals("redirect") ? "redirect/0" : mode));
+                long duration = elapsed(started);
+                int reads = sockets.stream().mapToInt(socket -> socket.readBytes.get()).sum();
+                Map<String, Object> details = new LinkedHashMap<>();
+                details.put("transport", "actual pinned local HTTP");
+                details.put("paths", List.copyOf(fixture.paths));
+                details.put("connectorCalls", sockets.size());
+                details.put("inputBytesRead", reads);
+                details.put("allRawSocketsClosed", sockets.stream().allMatch(ObservedSocket::closed));
+                if (mode.equals("body")) details.put("bodyBytesOffered", 65537);
+                if (mode.equals("ok") || mode.equals("body") || mode.equals("informational")) {
+                    details.put("bodyBytesRead", reads - fixture.responseHeaders.length());
+                }
+                record("local-" + mode, result, started, details);
+                assertTrue(duration < 1750, "Fixed total completion bound: " + duration);
+                assertTrue(sockets.stream().allMatch(ObservedSocket::closed));
+                if (mode.equals("trickle")) assertNoResponse(result, "TIMEOUT");
+                else if (mode.equals("redirect")) {
+                    assertAborted(result);
+                    assertEquals(List.of("/redirect/0", "/redirect/1", "/redirect/2", "/redirect/3"), fixture.paths);
+                    assertEquals(4, sockets.size());
+                } else {
+                    assertEquals("SUCCEEDED", result.state());
+                    assertEquals(200, result.httpStatus());
+                    assertEquals(fixture.responseHeaders.length(), reads, "No body byte may be consumed");
+                    assertEquals(1, sockets.size());
+                }
+            }
+        }
+    }
+
+    @Test
+    void uninterruptibleDnsRemainsBoundedAndCannotConnectAfterTheDeadline() throws Exception {
+        var release = new CountDownLatch(1);
+        AtomicInteger resolutions = new AtomicInteger(), connections = new AtomicInteger();
+        AtomicReference<Thread> actualThread = new AtomicReference<>();
+        var checks = stub(host -> {
+            actualThread.set(Thread.currentThread());
+            resolutions.incrementAndGet();
+            boolean done = false;
+            while (!done) {
+                try { release.await(); done = true; } catch (InterruptedException ignored) { /* Deliberate noninterruptible DNS fixture. */ }
+            }
+            return new InetAddress[]{InetAddress.getByName("93.184.216.34")};
+        }, (url, address, attempt) -> { connections.incrementAndGet(); throw new AssertionError("No late connection"); });
+        long started = System.nanoTime();
+        CheckRunner.CheckRun result;
         try {
-            var result = new CheckRunner("http://127.0.0.1:4325").run(
-                    execution(), "http://127.0.0.1:4325/stall");
+            result = checks.run(execution(), PUBLIC);
             assertNoResponse(result, "TIMEOUT");
+            assertTrue(elapsed(started) < 1750);
+            // A second intent observes finite capacity; it is not an automatic retry of the first intent.
+            assertThrows(RejectedExecutionException.class, () -> checks.run(execution(), PUBLIC));
+            assertEquals(1, resolutions.get());
         } finally {
-            releaseHeaders.countDown();
-            fixture.stop(0);
+            checks.close();
+            release.countDown();
+            if (actualThread.get() != null) actualThread.get().join(5000);
+        }
+        record("blocked-DNS", result, started, Map.of("transport", "explicit resolver barrier",
+                "resolverThreads", resolutions.get(), "queuedTasksAccepted", 0, "connectorCalls", connections.get(),
+                "actualIoThreadExited", !actualThread.get().isAlive()));
+        assertFalse(actualThread.get().isAlive());
+        assertEquals(0, connections.get());
+    }
+
+    @Test
+    void authoritativeServiceUncertaintyDoesNotBecomeAnEndpointResult() {
+        var checks = stub(host -> { throw new IllegalStateException("Controlled resolver service unavailable"); },
+                (url, address, attempt) -> { throw new AssertionError("No outbound I/O"); });
+        assertThrows(IllegalStateException.class, () -> checks.run(execution(), PUBLIC));
+        observations.add(new LinkedHashMap<>(Map.of("case", "service-uncertainty", "terminalResultCreated", false,
+                "exceptionPropagated", true, "automaticRetries", 0)));
+    }
+
+    private CheckRunner owned(CheckRunner checks) { runners.add(checks); return checks; }
+    private CheckRunner stub(CheckRunner.Resolver resolver, CheckRunner.Connector connector) {
+        return owned(new CheckRunner("http://127.0.0.1:4321", false, resolver, connector));
+    }
+    private static long elapsed(long started) { return TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - started); }
+    private static void record(String label, CheckRunner.CheckRun result, long started, Map<String, ?> details) {
+        Map<String, Object> row = new LinkedHashMap<>(details);
+        row.put("case", label);
+        row.put("state", result.state());
+        row.put("httpStatus", result.httpStatus());
+        row.put("failureReason", result.failureReason());
+        row.put("elapsedMs", elapsed(started));
+        observations.add(row);
+    }
+    private static void assertAborted(CheckRunner.CheckRun result) {
+        assertEquals("ABORTED", result.state());
+        assertNull(result.httpStatus());
+        assertNull(result.latencyMs());
+        assertNull(result.failureReason());
+        assertNotNull(result.finishedAt());
+    }
+
+    /** Counts bytes actually requested by the runner; real fixtures still use the production IP connector. */
+    private static final class ObservedSocket extends Socket {
+        private final Socket delegate;
+        private final InputStream input;
+        final AtomicInteger readBytes = new AtomicInteger();
+        final ByteArrayOutputStream request = new ByteArrayOutputStream();
+        private boolean closed;
+        ObservedSocket(String response) { delegate = null; input = counted(new ByteArrayInputStream(response.getBytes(StandardCharsets.ISO_8859_1))); }
+        ObservedSocket(Socket delegate) throws IOException { this.delegate = delegate; input = counted(delegate.getInputStream()); }
+        private InputStream counted(InputStream source) {
+            return new FilterInputStream(source) {
+                @Override public int read() throws IOException { int value = in.read(); if (value >= 0) readBytes.incrementAndGet(); return value; }
+                @Override public int read(byte[] bytes, int offset, int length) throws IOException {
+                    int count = in.read(bytes, offset, length); if (count > 0) readBytes.addAndGet(count); return count;
+                }
+            };
+        }
+        @Override public InputStream getInputStream() { return input; }
+        @Override public OutputStream getOutputStream() throws IOException { return delegate == null ? request : delegate.getOutputStream(); }
+        @Override public void setSoTimeout(int timeout) throws SocketException { if (delegate != null) delegate.setSoTimeout(timeout); }
+        @Override public void close() throws IOException { closed = true; if (delegate != null) delegate.close(); }
+        boolean closed() { return delegate == null ? closed : delegate.isClosed(); }
+    }
+
+    /** One isolated HTTP listener, no redirects/DNS delegated to another client or external network. */
+    private static final class LocalFixture implements AutoCloseable {
+        private final ServerSocket server = new ServerSocket();
+        private final CountDownLatch stop = new CountDownLatch(1);
+        private final Thread thread;
+        private final AtomicReference<Throwable> failure = new AtomicReference<>();
+        private volatile boolean closed;
+        private volatile Socket accepted;
+        final List<String> paths = java.util.Collections.synchronizedList(new ArrayList<>());
+        final String responseHeaders;
+
+        LocalFixture(String mode) throws IOException {
+            responseHeaders = mode.equals("body") ? "HTTP/1.1 200 OK\r\nContent-Length: 65537\r\n\r\n"
+                    : mode.equals("informational") ? "HTTP/1.1 103 Early Hints\r\nLink: </local>\r\n\r\n" + OK : OK;
+            server.setReuseAddress(true);
+            server.bind(new InetSocketAddress("127.0.0.1", 4325)); // Fails if another listener owns this port.
+            thread = new Thread(() -> {
+                try {
+                    while (!closed) {
+                        try (Socket client = server.accept()) {
+                            accepted = client;
+                            client.setSoTimeout(5000);
+                            ByteArrayOutputStream request = new ByteArrayOutputStream();
+                            int matched = 0;
+                            while (matched < 4 && request.size() < 8192) {
+                                int next = client.getInputStream().read();
+                                if (next < 0) throw new IOException("Missing fixture request");
+                                request.write(next);
+                                matched = next == (matched % 2 == 0 ? '\r' : '\n') ? matched + 1 : next == '\r' ? 1 : 0;
+                            }
+                            String path = request.toString(StandardCharsets.US_ASCII).split(" ")[1];
+                            paths.add(path);
+                            OutputStream output = client.getOutputStream();
+                            if (mode.equals("slow") && stop.await(2000, TimeUnit.MILLISECONDS)) return;
+                            if (mode.equals("trickle")) {
+                                for (byte value : "HTTP/".getBytes(StandardCharsets.US_ASCII)) {
+                                    output.write(value); output.flush();
+                                    if (stop.await(400, TimeUnit.MILLISECONDS)) return;
+                                }
+                            } else if (mode.equals("redirect")) {
+                                int hop = Integer.parseInt(path.substring(path.lastIndexOf('/') + 1));
+                                output.write(("HTTP/1.1 302 Found\r\nLocation: /redirect/" + (hop + 1)
+                                        + "\r\nContent-Length: 0\r\n\r\n").getBytes(StandardCharsets.US_ASCII));
+                            } else {
+                                output.write(responseHeaders.getBytes(StandardCharsets.US_ASCII));
+                                output.flush();
+                                if (mode.equals("body")) output.write(new byte[65537]);
+                            }
+                        } catch (IOException clientClosed) {
+                            if (closed) return;
+                            // Header-only observation intentionally closes an oversized/trickling response.
+                            if (!(mode.equals("body") || mode.equals("trickle"))) throw clientClosed;
+                        }
+                    }
+                } catch (Exception error) { if (!closed) failure.set(error); }
+            }, "e12-local-http-fixture");
+            thread.setDaemon(true);
+            thread.start();
+        }
+        @Override public void close() throws Exception {
+            closed = true;
+            stop.countDown();
+            server.close();
+            if (accepted != null) accepted.close();
+            thread.join(5000);
+            assertFalse(thread.isAlive(), "Owned fixture thread must exit");
+            assertNull(failure.get(), "Owned fixture must not fail before cleanup");
         }
     }
 
diff --git a/backend/src/test/java/dev/evolution/monitor/E10WorkerProcess.java b/backend/src/test/java/dev/evolution/monitor/E10WorkerProcess.java
index cbefe9b..bbacd2b 100644
--- a/backend/src/test/java/dev/evolution/monitor/E10WorkerProcess.java
+++ b/backend/src/test/java/dev/evolution/monitor/E10WorkerProcess.java
@@ -23,7 +23,7 @@ public class E10WorkerProcess {
         String label = args[3];
         if (!label.equals("one") && !label.equals("two")) throw new IllegalArgumentException("Unknown worker label");
         try (var application = new SpringApplicationBuilder(MonitorApplication.class)
-                .web(WebApplicationType.NONE).run("--spring.main.banner-mode=off");
+                .web(WebApplicationType.NONE).run("--spring.main.banner-mode=off", "--monitor.allow-test-fixture=true");
                 var barrier = TestDatabase.connect()) {
             var worker = application.getBean(CheckWorker.class);
             Map<String, Object> result = new LinkedHashMap<>();
diff --git a/backend/src/test/java/dev/evolution/monitor/MonitorFunctionalTest.java b/backend/src/test/java/dev/evolution/monitor/MonitorFunctionalTest.java
index 87cebbd..6b77ab1 100644
--- a/backend/src/test/java/dev/evolution/monitor/MonitorFunctionalTest.java
+++ b/backend/src/test/java/dev/evolution/monitor/MonitorFunctionalTest.java
@@ -258,19 +258,42 @@ class MonitorFunctionalTest {
     }
 
     @Test
-    void redirectsAreNotFollowedEvenInsideFixture() {
+    void redirectsUseTheFinalValidatedFixtureResponse() {
         int before = okRequests.get();
         var result = check(create("/redirect"));
-        assertEquals("FAILED", result.state());
-        assertEquals(302, result.httpStatus());
-        assertEquals(before, okRequests.get());
+        assertEquals("SUCCEEDED", result.state());
+        assertEquals(200, result.httpStatus());
+        assertEquals(before + 1, okRequests.get());
     }
 
     @Test
     void redirectsCannotLeaveConfiguredFixture() {
         var result = check(create("/redirect-outside"));
-        assertEquals(302, result.httpStatus());
-        assertEquals("FAILED", result.state());
+        assertNull(result.httpStatus());
+        assertEquals("ABORTED", result.state());
+        assertNull(result.latencyMs());
+        assertNull(result.failureReason());
+        assertEquals(0, forbiddenRequests.get());
+    }
+
+    @Test
+    void publicUrlsAreCanonicalAndDurableWithoutDnsOrCheckIoDuringCreateAndUpdate() {
+        int before = okRequests.get();
+        ObjectNode input = validInput().put("name", "Public destination fixture")
+                .put("url", "http://public.e12.test/ok");
+        JsonNode created = assertDataEnvelope(postJson(input.toString()), HttpStatus.CREATED);
+        String path = "/api/monitors/" + created.at("/monitor/id").textValue();
+        assertEquals(input.get("url"), created.at("/monitor/url"));
+        assertTrue(created.get("latestCheck").isNull());
+        input.put("url", "HTTP://PUBLIC.E12.TEST:80/a/../ok");
+        var headers = new HttpHeaders();
+        headers.setContentType(MediaType.APPLICATION_JSON);
+        JsonNode updated = assertDataEnvelope(api.exchange(path, HttpMethod.PUT,
+                new HttpEntity<>(input.toString(), headers), JsonNode.class), HttpStatus.OK);
+        assertEquals("http://public.e12.test/ok", updated.at("/monitor/url").textValue());
+        assertEquals(updated, assertDataEnvelope(api.getForEntity(path, JsonNode.class), HttpStatus.OK));
+        assertEquals(0, assertDataEnvelope(api.getForEntity(path + "/checks", JsonNode.class), HttpStatus.OK).size());
+        assertEquals(before, okRequests.get());
         assertEquals(0, forbiddenRequests.get());
     }
 
diff --git a/backend/src/test/java/dev/evolution/monitor/TestDatabase.java b/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
index 74bce71..94845a6 100644
--- a/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
+++ b/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
@@ -48,6 +48,7 @@ final class TestDatabase {
 
     static void configure(DynamicPropertyRegistry properties, String schema) {
         reset(schema);
+        properties.add("monitor.allow-test-fixture", () -> true);
         properties.add("spring.flyway.schemas", () -> schema);
         properties.add("spring.flyway.default-schema", () -> schema);
         properties.add("spring.jpa.properties.hibernate.default_schema", () -> schema);
diff --git a/scripts/persistence-scenario.mjs b/scripts/persistence-scenario.mjs
index 5e5fca5..7588cba 100644
--- a/scripts/persistence-scenario.mjs
+++ b/scripts/persistence-scenario.mjs
@@ -24,7 +24,7 @@ function start(command, args, logName) {
   const logPath = `${directory}/${label}-${logName}.log`;
   const log = openSync(logPath, 'w');
   const child = spawn(command, args, {
-    env: { ...process.env, DB_SCHEMA: 'e04_restart' }, stdio: ['ignore', log, log],
+    env: { ...process.env, DB_SCHEMA: 'e04_restart', ALLOW_TEST_FIXTURE: 'true' }, stdio: ['ignore', log, log],
   });
   closeSync(log);
   child.evidence = { role: logName, pid: child.pid, startedAt: new Date().toISOString(), logPath };
diff --git a/scripts/test-api.mjs b/scripts/test-api.mjs
index 07a79a9..461bb39 100644
--- a/scripts/test-api.mjs
+++ b/scripts/test-api.mjs
@@ -19,12 +19,12 @@ try {
   bootstrapUsers('e04_browser', process.env);
   const { E04_ALICE_PASSWORD, E04_BOB_PASSWORD, ...runtime } = process.env;
   api = spawn('java', ['-jar', 'backend/target/monitor-api-0.0.1.jar'], {
-    env: { ...runtime, DB_SCHEMA: 'e04_browser' }, stdio: 'inherit',
+    env: { ...runtime, DB_SCHEMA: 'e04_browser', ALLOW_TEST_FIXTURE: 'true' }, stdio: 'inherit',
   });
   if (process.env.E09_MANUAL_WORKER !== '1') {
     worker = spawn('java', ['-jar', 'backend/target/monitor-api-0.0.1.jar',
       '--spring.profiles.active=worker', '--spring.main.web-application-type=none', '--monitor.scheduler-enabled=false'], {
-      env: { ...runtime, DB_SCHEMA: 'e04_browser' }, stdio: 'inherit',
+      env: { ...runtime, DB_SCHEMA: 'e04_browser', ALLOW_TEST_FIXTURE: 'true' }, stdio: 'inherit',
     });
     worker.once('exit', () => {
       if (!stopping) { workerFailed = true; api.kill('SIGTERM'); }
diff --git a/tests/browser/worker.spec.ts b/tests/browser/worker.spec.ts
index 8b89ac7..536434c 100644
--- a/tests/browser/worker.spec.ts
+++ b/tests/browser/worker.spec.ts
@@ -68,7 +68,7 @@ test('one persisted acceptance progresses in a separate worker while the browser
     const { E04_ALICE_PASSWORD, E04_BOB_PASSWORD, ...runtime } = process.env;
     worker = spawn('java', ['-jar', 'backend/target/monitor-api-0.0.1.jar', '--spring.profiles.active=worker',
       '--spring.main.web-application-type=none', '--monitor.scheduler-enabled=false'], {
-      env: { ...runtime, DB_SCHEMA: 'e04_browser' }, stdio: ['ignore', log, log],
+      env: { ...runtime, DB_SCHEMA: 'e04_browser', ALLOW_TEST_FIXTURE: 'true' }, stdio: ['ignore', log, log],
     });
     closeSync(log);
     worker.once('error', () => { workerError = true; });
@@ -186,7 +186,7 @@ test('an expired unknown execution becomes ABORTED in the current view and cache
     const { E04_ALICE_PASSWORD, E04_BOB_PASSWORD, ...runtime } = process.env;
     worker = spawn('java', ['-jar', 'backend/target/monitor-api-0.0.1.jar', '--spring.profiles.active=worker',
       '--spring.main.web-application-type=none', '--monitor.scheduler-enabled=false'], {
-      env: { ...runtime, DB_SCHEMA: 'e04_browser' }, stdio: ['ignore', log, log],
+      env: { ...runtime, DB_SCHEMA: 'e04_browser', ALLOW_TEST_FIXTURE: 'true' }, stdio: ['ignore', log, log],
     });
     closeSync(log);
     worker.once('error', () => { workerError = true; });


