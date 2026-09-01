# E02 HTTP 런타임 계약과 오류 경계

## `fix(http): enforce the Monitor runtime contract`

diff --git a/TRACK.md b/TRACK.md
index a7be6ee..65fe0ad 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -2,7 +2,7 @@
 
 Spec revision: `0a006589477f8ae47bad3faa5510c999cff85ee4`
 
-E01 is a local development product: Next.js/React renders the Monitor form and terminal Check result; Spring MVC owns an in-memory Monitor map and latest Check result. A manual check is synchronous. There are no accounts, database, scheduler, workers, cache, broker, or production containers.
+E02 is a local development product: Next.js/React renders the Monitor form and terminal Check result; Spring MVC owns an in-memory Monitor map and latest Check result. A manual check is synchronous. There are no accounts, database, scheduler, workers, cache, broker, or production containers.
 
 ## Pinned toolchain
 
@@ -49,7 +49,17 @@ All three defaults bind to `127.0.0.1`. Fixture port is 4321, API port 4322, UI
 - Checks send one GET with no request body, use no proxy, and never follow redirects. `/redirect` therefore records `FAILED / 302`, with no request to `/ok`.
 - Connect timeout is 1 second and response-header read timeout is 2 seconds. No response body is materialized or retained. This is a controlled fixture implementation, not general Internet monitoring or general SSRF defense.
 - `200..299` is `SUCCEEDED`. Other observed HTTP statuses are `FAILED / HTTP_STATUS`. No HTTP response produces a null status and `TIMEOUT` or `CONNECTION_FAILURE`; no synthetic status is invented.
-- Latest results survive page reload, but all state disappears with the API process. Interval and enabled are stored, without automatic scheduling. Basic HTTP errors are shown in the UI; no uniform runtime error contract is claimed in E01.
+- Latest results survive page reload, but all state disappears with the API process. Interval and enabled are stored, without automatic scheduling.
+
+## HTTP contract (E02)
+
+- Create input must be a JSON object with string name and URL, a numeric integer interval, and boolean enabled. Required fields cannot be null or omitted; scalar strings/numbers/booleans are not coerced into other types.
+- Names are stripped of surrounding whitespace and must contain 1–100 UTF-16 code units. Interval is 1–86400 seconds inclusive; the JSON number `60.0` is the integer value 60, but the string `"60"` is invalid.
+- URL syntax must be absolute HTTP(S). The existing fixture policy then restricts it to the configured HTTP hostname and port, without credentials or fragments. This does not enable general public or HTTPS monitoring.
+- Successful list/create/check responses contain exactly `{ "data": <payload> }`. Create returns 201; list and completed synchronous checks return 200. The existing MonitorView/CheckRun payload fields are preserved, including explicit nulls.
+- API failures contain `{ "error": { "code": "...", "message": "..." } }`: INVALID_INPUT / 400, NOT_FOUND / 404, or INTERNAL_ERROR / 500. Malformed JSON and IDs use INVALID_INPUT. Unexpected exception details are not returned.
+- Endpoint HTTP 503 is still an API success containing a FAILED CheckRun with HTTP_STATUS; timeout and connection failure retain null httpStatus. API errors never fabricate a CheckRun.
+- The browser validates the envelope and the displayed payload shape, selects errors by the stable code/status pair, and does not classify or display arbitrary server prose. Network or malformed responses use the INTERNAL_ERROR UI fallback without applying the mutation.
 
 ## Verification
 
@@ -58,7 +68,7 @@ npx playwright install chromium
 npm run verify
 ```
 
-`verify` runs Maven unit and real-HTTP functional tests and packages the API, then TypeScript checking, a Next production compilation, and two real Chromium browser tests against the local development UI and packaged API. It does not retry failed tests. Command outcomes and elapsed times are appended to `output/verification/runs.jsonl`. E01's committed evidence is in `evidence/E01`.
+`verify` runs Maven unit and real-HTTP functional tests and packages the API, then TypeScript checking, a Next production compilation, and real Chromium browser tests against the local development UI and packaged API. It does not retry failed tests. Command outcomes and elapsed times are appended to `output/verification/runs.jsonl`. Committed evidence is in `evidence/E01` and `evidence/E02`.
 
 The CI workflow installs the exact toolchain and runs the same gates. No hosted CI run is claimed by local verification. The browser gate starts and stops its own processes and refuses existing servers. There are no load tests, benchmarks, or parameter sweeps.
 
diff --git a/app/monitors/api.ts b/app/monitors/api.ts
new file mode 100644
index 0000000..fa41ef4
--- /dev/null
+++ b/app/monitors/api.ts
@@ -0,0 +1,87 @@
+export type Monitor = { id: string; name: string; url: string; interval: number; enabled: boolean };
+type CheckRun = {
+  id: string; monitorId: string; state: 'SUCCEEDED' | 'FAILED'; httpStatus: number | null;
+  latencyMs: number; failureReason: 'HTTP_STATUS' | 'CONNECTION_FAILURE' | 'TIMEOUT' | null; finishedAt: string;
+};
+export type MonitorView = { monitor: Monitor; latestCheck: CheckRun | null };
+export type ApiErrorCode = 'INVALID_INPUT' | 'NOT_FOUND' | 'INTERNAL_ERROR';
+
+export const errorMessages: Record<ApiErrorCode, string> = {
+  INVALID_INPUT: 'Invalid monitor input. Check the name, URL, interval, and enabled value.',
+  NOT_FOUND: 'Monitor not found. Reload the list and try again.',
+  INTERNAL_ERROR: 'The service could not complete the request. Try again.',
+};
+const errorStatuses: Record<ApiErrorCode, number> = { INVALID_INPUT: 400, NOT_FOUND: 404, INTERNAL_ERROR: 500 };
+
+class ApiFailure extends Error {
+  constructor(readonly code: ApiErrorCode) { super(code); }
+}
+
+export function failureCode(error: unknown): ApiErrorCode {
+  return error instanceof ApiFailure ? error.code : 'INTERNAL_ERROR';
+}
+
+function isObject(value: unknown): value is Record<string, unknown> {
+  return typeof value === 'object' && value !== null && !Array.isArray(value);
+}
+
+function isMonitor(value: unknown): value is Monitor {
+  return isObject(value) && typeof value.id === 'string' && typeof value.name === 'string'
+    && typeof value.url === 'string' && typeof value.interval === 'number'
+    && Number.isInteger(value.interval) && value.interval >= 1 && value.interval <= 86400
+    && typeof value.enabled === 'boolean';
+}
+
+function isCheckRun(value: unknown): value is CheckRun {
+  if (!isObject(value) || typeof value.id !== 'string' || typeof value.monitorId !== 'string'
+    || typeof value.finishedAt !== 'string' || typeof value.latencyMs !== 'number'
+    || !Number.isInteger(value.latencyMs) || value.latencyMs < 0) return false;
+  const status = value.httpStatus;
+  if (status === null) {
+    return value.state === 'FAILED' && (value.failureReason === 'CONNECTION_FAILURE' || value.failureReason === 'TIMEOUT');
+  }
+  if (typeof status !== 'number' || !Number.isInteger(status) || status < 100 || status > 599) return false;
+  return status >= 200 && status < 300
+    ? value.state === 'SUCCEEDED' && value.failureReason === null
+    : value.state === 'FAILED' && value.failureReason === 'HTTP_STATUS';
+}
+
+function isMonitorView(value: unknown): value is MonitorView {
+  return isObject(value) && isMonitor(value.monitor) && (value.latestCheck === null || isCheckRun(value.latestCheck));
+}
+
+async function readData<T>(response: Response, valid: (value: unknown) => value is T): Promise<T> {
+  let body: unknown;
+  try {
+    body = await response.json();
+  } catch {
+    throw new ApiFailure('INTERNAL_ERROR');
+  }
+  if (!response.ok) {
+    if (isObject(body) && isObject(body.error) && typeof body.error.message === 'string') {
+      const code = body.error.code;
+      if ((code === 'INVALID_INPUT' || code === 'NOT_FOUND' || code === 'INTERNAL_ERROR')
+        && errorStatuses[code] === response.status) throw new ApiFailure(code);
+    }
+    throw new ApiFailure('INTERNAL_ERROR');
+  }
+  if (!isObject(body) || Object.keys(body).length !== 1 || !valid(body.data)) {
+    throw new ApiFailure('INTERNAL_ERROR');
+  }
+  return body.data;
+}
+
+export async function loadMonitors(): Promise<MonitorView[]> {
+  return readData(await fetch('/api/monitors'), (value): value is MonitorView[] =>
+    Array.isArray(value) && value.every(isMonitorView));
+}
+
+export async function createMonitor(input: Omit<Monitor, 'id'>): Promise<MonitorView> {
+  return readData(await fetch('/api/monitors', {
+    method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(input),
+  }), isMonitorView);
+}
+
+export async function runCheck(id: string): Promise<CheckRun> {
+  return readData(await fetch(`/api/monitors/${id}/checks`, { method: 'POST' }), isCheckRun);
+}
diff --git a/app/monitors/page.tsx b/app/monitors/page.tsx
index 2bd0959..5f22880 100644
--- a/app/monitors/page.tsx
+++ b/app/monitors/page.tsx
@@ -1,24 +1,16 @@
 'use client';
 
 import { useEffect, useState, type FormEvent } from 'react';
-
-type Monitor = { id: string; name: string; url: string; interval: number; enabled: boolean };
-type CheckRun = {
-  id: string; monitorId: string; state: 'SUCCEEDED' | 'FAILED'; httpStatus: number | null;
-  latencyMs: number; failureReason: string | null; finishedAt: string;
-};
-type MonitorView = { monitor: Monitor; latestCheck: CheckRun | null };
+import { createMonitor, errorMessages, failureCode, loadMonitors, runCheck,
+  type ApiErrorCode, type MonitorView } from './api';
 
 export default function Monitors() {
   const [monitors, setMonitors] = useState<MonitorView[]>([]);
-  const [error, setError] = useState('');
+  const [error, setError] = useState<ApiErrorCode | null>(null);
   const [busy, setBusy] = useState(false);
 
   useEffect(() => {
-    fetch('/api/monitors').then(async response => {
-      if (!response.ok) throw new Error('Could not load monitors.');
-      setMonitors(await response.json());
-    }).catch(error => setError(error.message));
+    loadMonitors().then(setMonitors).catch(error => setError(failureCode(error)));
   }, []);
 
   async function create(event: FormEvent<HTMLFormElement>) {
@@ -26,19 +18,16 @@ export default function Monitors() {
     const form = event.currentTarget;
     const fields = new FormData(form);
     setBusy(true);
-    setError('');
+    setError(null);
     try {
-      const response = await fetch('/api/monitors', {
-        method: 'POST', headers: { 'Content-Type': 'application/json' },
-        body: JSON.stringify({ name: fields.get('name'), url: fields.get('url'),
-          interval: Number(fields.get('interval')), enabled: fields.get('enabled') === 'on' }),
+      const created = await createMonitor({
+        name: String(fields.get('name') ?? ''), url: String(fields.get('url') ?? ''),
+        interval: Number(fields.get('interval')), enabled: fields.get('enabled') === 'on',
       });
-      if (!response.ok) throw new Error(`Monitor could not be created (HTTP ${response.status}). Use the configured local fixture.`);
-      const created: MonitorView = await response.json();
       setMonitors(current => [...current, created]);
       form.reset();
     } catch (error) {
-      setError(error instanceof Error ? error.message : 'Monitor could not be created.');
+      setError(failureCode(error));
     } finally {
       setBusy(false);
     }
@@ -46,14 +35,12 @@ export default function Monitors() {
 
   async function check(id: string) {
     setBusy(true);
-    setError('');
+    setError(null);
     try {
-      const response = await fetch(`/api/monitors/${id}/checks`, { method: 'POST' });
-      if (!response.ok) throw new Error(`Check could not be completed (HTTP ${response.status}).`);
-      const latestCheck: CheckRun = await response.json();
+      const latestCheck = await runCheck(id);
       setMonitors(current => current.map(row => row.monitor.id === id ? { ...row, latestCheck } : row));
     } catch (error) {
-      setError(error instanceof Error ? error.message : 'Check could not be completed.');
+      setError(failureCode(error));
     } finally {
       setBusy(false);
     }
@@ -68,13 +55,13 @@ export default function Monitors() {
       <form onSubmit={create}>
         <label>Name<input name="name" required defaultValue="Fixture monitor" /></label>
         <label>URL<input name="url" type="url" required defaultValue="http://127.0.0.1:4321/ok" /></label>
-        <label>Interval (seconds)<input name="interval" type="number" min="1" required defaultValue="60" /></label>
+        <label>Interval (seconds)<input name="interval" type="number" min="1" max="86400" step="1" required defaultValue="60" /></label>
         <label className="checkbox"><input name="enabled" type="checkbox" defaultChecked />Enabled</label>
         <p className="hint">Interval and enabled are stored; E01 runs checks manually.</p>
         <button disabled={busy}>Create monitor</button>
       </form>
     </section>
-    {error && <p role="alert" className="error">{error}</p>}
+    {error && <p role="alert" className="error" data-error-code={error}>{errorMessages[error]}</p>}
     <section aria-labelledby="monitors-title"><h2 id="monitors-title">Monitors</h2>
       {monitors.length === 0 && <p>No monitors yet.</p>}
       {monitors.map(({ monitor, latestCheck }) => <article key={monitor.id} aria-label={monitor.name}>
diff --git a/backend/src/main/java/dev/evolution/monitor/ApiData.java b/backend/src/main/java/dev/evolution/monitor/ApiData.java
new file mode 100644
index 0000000..24bc8a9
--- /dev/null
+++ b/backend/src/main/java/dev/evolution/monitor/ApiData.java
@@ -0,0 +1,3 @@
+package dev.evolution.monitor;
+
+public record ApiData<T>(T data) {}
diff --git a/backend/src/main/java/dev/evolution/monitor/ApiErrors.java b/backend/src/main/java/dev/evolution/monitor/ApiErrors.java
new file mode 100644
index 0000000..c1f7693
--- /dev/null
+++ b/backend/src/main/java/dev/evolution/monitor/ApiErrors.java
@@ -0,0 +1,56 @@
+package dev.evolution.monitor;
+
+import org.springframework.http.HttpStatus;
+import org.springframework.http.MediaType;
+import org.springframework.http.ResponseEntity;
+import org.springframework.http.converter.HttpMessageNotReadableException;
+import org.springframework.web.HttpMediaTypeNotAcceptableException;
+import org.springframework.web.HttpMediaTypeNotSupportedException;
+import org.springframework.web.HttpRequestMethodNotSupportedException;
+import org.springframework.web.bind.annotation.ExceptionHandler;
+import org.springframework.web.bind.annotation.RestControllerAdvice;
+import org.springframework.web.method.annotation.MethodArgumentTypeMismatchException;
+import org.springframework.web.server.ResponseStatusException;
+import org.springframework.web.servlet.NoHandlerFoundException;
+import org.springframework.web.servlet.resource.NoResourceFoundException;
+
+@RestControllerAdvice
+public class ApiErrors {
+    public enum Code { INVALID_INPUT, NOT_FOUND, INTERNAL_ERROR }
+    public record Detail(Code code, String message) {}
+    public record Failure(Detail error) {}
+
+    @ExceptionHandler(ResponseStatusException.class)
+    ResponseEntity<Failure> rejected(ResponseStatusException error) {
+        return switch (error.getStatusCode().value()) {
+            case 400 -> response(HttpStatus.BAD_REQUEST, Code.INVALID_INPUT,
+                    error.getReason() == null ? "Request input is invalid." : error.getReason());
+            case 404 -> notFound();
+            default -> internalError();
+        };
+    }
+
+    @ExceptionHandler({HttpMessageNotReadableException.class, MethodArgumentTypeMismatchException.class,
+            HttpMediaTypeNotSupportedException.class, HttpMediaTypeNotAcceptableException.class,
+            HttpRequestMethodNotSupportedException.class})
+    ResponseEntity<Failure> invalidInput() {
+        return response(HttpStatus.BAD_REQUEST, Code.INVALID_INPUT, "Request input is invalid.");
+    }
+
+    @ExceptionHandler({NoResourceFoundException.class, NoHandlerFoundException.class})
+    ResponseEntity<Failure> notFound() {
+        return response(HttpStatus.NOT_FOUND, Code.NOT_FOUND, "Resource not found.");
+    }
+
+    @ExceptionHandler(Exception.class)
+    ResponseEntity<Failure> internalError() {
+        // Never expose exception messages, parser details, or stack traces on the wire.
+        return response(HttpStatus.INTERNAL_SERVER_ERROR, Code.INTERNAL_ERROR,
+                "The service could not complete the request.");
+    }
+
+    private static ResponseEntity<Failure> response(HttpStatus status, Code code, String message) {
+        return ResponseEntity.status(status).contentType(MediaType.APPLICATION_JSON)
+                .body(new Failure(new Detail(code, message)));
+    }
+}
diff --git a/backend/src/main/java/dev/evolution/monitor/MonitorController.java b/backend/src/main/java/dev/evolution/monitor/MonitorController.java
index 4e170c3..2fa6075 100644
--- a/backend/src/main/java/dev/evolution/monitor/MonitorController.java
+++ b/backend/src/main/java/dev/evolution/monitor/MonitorController.java
@@ -1,5 +1,7 @@
 package dev.evolution.monitor;
 
+import com.fasterxml.jackson.databind.JsonNode;
+import java.net.URI;
 import java.time.Instant;
 import java.util.Comparator;
 import java.util.List;
@@ -28,32 +30,72 @@ public class MonitorController {
     }
 
     @GetMapping
-    public List<MonitorView> list() {
-        return monitors.values().stream().sorted(Comparator.comparing(Monitor::createdAt))
-                .map(monitor -> new MonitorView(monitor, latestChecks.get(monitor.id()))).toList();
+    public ApiData<List<MonitorView>> list() {
+        return new ApiData<>(monitors.values().stream().sorted(Comparator.comparing(Monitor::createdAt))
+                .map(monitor -> new MonitorView(monitor, latestChecks.get(monitor.id()))).toList());
     }
 
     @PostMapping
     @ResponseStatus(HttpStatus.CREATED)
-    public MonitorView create(@RequestBody CreateMonitor input) {
+    public ApiData<MonitorView> create(@RequestBody JsonNode body) {
+        CreateMonitor input = CreateMonitor.fromJson(body);
         checks.requireFixtureUrl(input.url());
         Instant now = Instant.now();
         Monitor monitor = new Monitor(UUID.randomUUID(), input.name(), input.url(), input.interval(),
                 input.enabled(), now, now);
         monitors.put(monitor.id(), monitor);
-        return new MonitorView(monitor, null);
+        return new ApiData<>(new MonitorView(monitor, null));
     }
 
     @PostMapping("/{id}/checks")
-    public CheckRunner.CheckRun check(@PathVariable UUID id) {
+    public ApiData<CheckRunner.CheckRun> check(@PathVariable UUID id) {
         Monitor monitor = monitors.get(id);
         if (monitor == null) throw new ResponseStatusException(HttpStatus.NOT_FOUND, "Monitor not found");
         CheckRunner.CheckRun result = checks.run(monitor.id(), monitor.url());
         latestChecks.put(id, result);
-        return result;
+        return new ApiData<>(result);
     }
 
-    public record CreateMonitor(String name, String url, int interval, boolean enabled) {}
+    public record CreateMonitor(String name, String url, int interval, boolean enabled) {
+        static CreateMonitor fromJson(JsonNode body) {
+            // Inspect JSON kinds before conversion: Jackson's scalar coercions are not the contract.
+            if (body == null || !body.isObject() || !body.path("name").isTextual()
+                    || !body.path("url").isTextual() || !body.path("interval").isNumber()
+                    || !body.path("enabled").isBoolean()) {
+                throw invalid("Name and URL must be strings, interval a number, and enabled a boolean.");
+            }
+            String name = body.get("name").textValue().strip();
+            if (name.isEmpty() || name.length() > 100) {
+                throw invalid("Name must contain 1 to 100 UTF-16 code units after trimming.");
+            }
+            int interval;
+            try {
+                // 60.0 has the integer value 60; strings and fractional values are not coerced.
+                interval = body.get("interval").decimalValue().intValueExact();
+            } catch (ArithmeticException error) {
+                throw invalid("Interval must be an integer from 1 to 86400 seconds.");
+            }
+            if (interval < 1 || interval > 86400) {
+                throw invalid("Interval must be an integer from 1 to 86400 seconds.");
+            }
+            String url = body.get("url").textValue();
+            try {
+                URI parsed = URI.create(url);
+                if (!parsed.isAbsolute() || parsed.getHost() == null
+                        || !("http".equalsIgnoreCase(parsed.getScheme())
+                        || "https".equalsIgnoreCase(parsed.getScheme()))) {
+                    throw new IllegalArgumentException();
+                }
+            } catch (IllegalArgumentException error) {
+                throw invalid("URL must be an absolute HTTP or HTTPS URL.");
+            }
+            return new CreateMonitor(name, url, interval, body.get("enabled").booleanValue());
+        }
+
+        private static ResponseStatusException invalid(String message) {
+            return new ResponseStatusException(HttpStatus.BAD_REQUEST, message);
+        }
+    }
     public record Monitor(UUID id, String name, String url, int interval, boolean enabled,
                           Instant createdAt, Instant updatedAt) {}
     public record MonitorView(Monitor monitor, CheckRunner.CheckRun latestCheck) {}
diff --git a/backend/src/main/resources/application.properties b/backend/src/main/resources/application.properties
index 062c3f6..ab830cb 100644
--- a/backend/src/main/resources/application.properties
+++ b/backend/src/main/resources/application.properties
@@ -2,3 +2,4 @@ spring.application.name=monitor-api
 server.address=127.0.0.1
 server.port=${API_PORT:4322}
 monitor.fixture-origin=${FIXTURE_ORIGIN:http://127.0.0.1:4321}
+spring.jackson.deserialization.fail-on-trailing-tokens=true
diff --git a/backend/src/test/java/dev/evolution/monitor/ApiErrorBoundaryTest.java b/backend/src/test/java/dev/evolution/monitor/ApiErrorBoundaryTest.java
new file mode 100644
index 0000000..d018ba8
--- /dev/null
+++ b/backend/src/test/java/dev/evolution/monitor/ApiErrorBoundaryTest.java
@@ -0,0 +1,35 @@
+package dev.evolution.monitor;
+
+import static org.junit.jupiter.api.Assertions.*;
+import static org.mockito.Mockito.*;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import org.junit.jupiter.api.Test;
+import org.springframework.http.MediaType;
+import org.springframework.test.web.servlet.setup.MockMvcBuilders;
+
+class ApiErrorBoundaryTest {
+    @Test
+    void unexpectedFailureIsAnInternalErrorWithoutPrivateExceptionDetails() throws Exception {
+        CheckRunner checks = mock(CheckRunner.class);
+        when(checks.requireFixtureUrl("http://127.0.0.1:4321/ok"))
+                .thenThrow(new IllegalStateException("Private implementation detail"));
+        var mvc = MockMvcBuilders.standaloneSetup(new MonitorController(checks))
+                .setControllerAdvice(new ApiErrors()).build();
+        String body = mvc.perform(post("/api/monitors").contentType(MediaType.APPLICATION_JSON).content("""
+                {"name":"Fixture monitor","url":"http://127.0.0.1:4321/ok","interval":60,"enabled":true}
+                """))
+                .andExpect(status().isInternalServerError())
+                .andExpect(content().contentType(MediaType.APPLICATION_JSON))
+                .andExpect(jsonPath("$.error.code").value("INTERNAL_ERROR"))
+                .andExpect(jsonPath("$.error.message").value("The service could not complete the request."))
+                .andExpect(jsonPath("$.data").doesNotExist())
+                .andReturn().getResponse().getContentAsString();
+        var wire = new ObjectMapper().readTree(body);
+        assertEquals(1, wire.size());
+        assertEquals(2, wire.get("error").size());
+        assertFalse(body.contains("Private implementation detail"));
+    }
+}
diff --git a/backend/src/test/java/dev/evolution/monitor/CheckRunnerTest.java b/backend/src/test/java/dev/evolution/monitor/CheckRunnerTest.java
index 0187dfb..5fbd6b6 100644
--- a/backend/src/test/java/dev/evolution/monitor/CheckRunnerTest.java
+++ b/backend/src/test/java/dev/evolution/monitor/CheckRunnerTest.java
@@ -1,6 +1,14 @@
 package dev.evolution.monitor;
 
 import static org.junit.jupiter.api.Assertions.*;
+import com.fasterxml.jackson.databind.JsonNode;
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.fasterxml.jackson.databind.SerializationFeature;
+import com.sun.net.httpserver.HttpServer;
+import java.net.InetSocketAddress;
+import java.net.ServerSocket;
+import java.util.UUID;
+import java.util.concurrent.CountDownLatch;
 import org.junit.jupiter.api.Test;
 import org.springframework.web.server.ResponseStatusException;
 
@@ -25,4 +33,54 @@ class CheckRunnerTest {
             assertThrows(ResponseStatusException.class, () -> runner.requireFixtureUrl(url));
         }
     }
+
+    @Test
+    void connectionFailureHasNoInventedHttpStatusOnTheWire() throws Exception {
+        // Verify that this isolated fixture port is free, then leave it closed.
+        try (ServerSocket closedFixture = new ServerSocket()) {
+            closedFixture.bind(new InetSocketAddress("127.0.0.1", 4325));
+        }
+        var result = new CheckRunner("http://127.0.0.1:4325").run(
+                UUID.fromString("00000000-0000-4000-8000-000000000000"), "http://127.0.0.1:4325/ok");
+        assertNoResponse(result, "CONNECTION_FAILURE");
+    }
+
+    @Test
+    void headerTimeoutHasNoInventedHttpStatusOnTheWire() throws Exception {
+        var releaseHeaders = new CountDownLatch(1);
+        var fixture = HttpServer.create(new InetSocketAddress("127.0.0.1", 4325), 0);
+        fixture.createContext("/stall", exchange -> {
+            try {
+                releaseHeaders.await();
+            } catch (InterruptedException interrupted) {
+                Thread.currentThread().interrupt();
+            } finally {
+                exchange.close();
+            }
+        });
+        fixture.start();
+        try {
+            var result = new CheckRunner("http://127.0.0.1:4325").run(
+                    UUID.fromString("00000000-0000-4000-8000-000000000000"), "http://127.0.0.1:4325/stall");
+            assertNoResponse(result, "TIMEOUT");
+        } finally {
+            releaseHeaders.countDown();
+            fixture.stop(0);
+        }
+    }
+
+    private static void assertNoResponse(CheckRunner.CheckRun result, String reason) {
+        assertEquals("FAILED", result.state());
+        assertNull(result.httpStatus());
+        assertEquals(reason, result.failureReason());
+        var json = new ObjectMapper().findAndRegisterModules().disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
+        JsonNode wire = json.valueToTree(new ApiData<>(result));
+        assertEquals(1, wire.size());
+        assertTrue(wire.at("/data/httpStatus").isNull());
+        assertEquals("FAILED", wire.at("/data/state").textValue());
+        assertEquals(reason, wire.at("/data/failureReason").textValue());
+        assertTrue(wire.at("/data/latencyMs").isIntegralNumber());
+        assertTrue(wire.at("/data/startedAt").isTextual());
+        assertTrue(wire.at("/data/finishedAt").isTextual());
+    }
 }
diff --git a/backend/src/test/java/dev/evolution/monitor/MonitorFunctionalTest.java b/backend/src/test/java/dev/evolution/monitor/MonitorFunctionalTest.java
index 67e8e12..8be16ca 100644
--- a/backend/src/test/java/dev/evolution/monitor/MonitorFunctionalTest.java
+++ b/backend/src/test/java/dev/evolution/monitor/MonitorFunctionalTest.java
@@ -1,9 +1,16 @@
 package dev.evolution.monitor;
 
 import static org.junit.jupiter.api.Assertions.*;
+import com.fasterxml.jackson.databind.JsonNode;
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.fasterxml.jackson.databind.node.ObjectNode;
 import com.sun.net.httpserver.HttpServer;
 import java.io.IOException;
 import java.net.InetSocketAddress;
+import java.time.Instant;
+import java.util.HashSet;
+import java.util.Set;
+import java.util.UUID;
 import java.util.concurrent.atomic.AtomicInteger;
 import org.junit.jupiter.api.AfterAll;
 import org.junit.jupiter.api.BeforeAll;
@@ -12,6 +19,10 @@ import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.boot.test.context.SpringBootTest;
 import org.springframework.boot.test.web.client.TestRestTemplate;
 import org.springframework.http.HttpStatus;
+import org.springframework.http.HttpEntity;
+import org.springframework.http.HttpHeaders;
+import org.springframework.http.MediaType;
+import org.springframework.http.ResponseEntity;
 
 @SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
 class MonitorFunctionalTest {
@@ -20,6 +31,7 @@ class MonitorFunctionalTest {
     static final AtomicInteger okRequests = new AtomicInteger();
     static final AtomicInteger forbiddenRequests = new AtomicInteger();
     @Autowired TestRestTemplate api;
+    @Autowired ObjectMapper json;
 
     @BeforeAll
     static void startFixture() throws IOException {
@@ -64,6 +76,114 @@ class MonitorFunctionalTest {
         if (forbidden != null) forbidden.stop(0);
     }
 
+    @Test
+    void rejectsBlankNameAtRuntime() {
+        var headers = new HttpHeaders();
+        headers.setContentType(MediaType.APPLICATION_JSON);
+        var response = api.postForEntity("/api/monitors", new HttpEntity<>("""
+                {"name":"   ","url":"http://127.0.0.1:4321/ok","interval":60,"enabled":true}
+                """, headers), JsonNode.class);
+        assertApiError(response, HttpStatus.BAD_REQUEST, "INVALID_INPUT");
+    }
+
+    @Test
+    void rejectsWrongJsonTypesAndMissingFields() throws Exception {
+        for (String field : new String[]{"name", "url"}) {
+            for (String value : new String[]{"null", "60", "true", "[]", "{}"}) {
+                assertInvalidReplacement(field, json.readTree(value));
+            }
+        }
+        for (String value : new String[]{"\"60\"", "null", "true", "[]", "{}", "1.5", "0", "86401", "2147483648"}) {
+            assertInvalidReplacement("interval", json.readTree(value));
+        }
+        for (String value : new String[]{"null", "\"true\"", "0", "1", "[]", "{}"}) {
+            assertInvalidReplacement("enabled", json.readTree(value));
+        }
+        for (String field : new String[]{"name", "url", "interval", "enabled"}) {
+            ObjectNode input = validInput();
+            input.remove(field);
+            assertApiError(postJson(input.toString()), HttpStatus.BAD_REQUEST, "INVALID_INPUT");
+        }
+    }
+
+    @Test
+    void rejectsInvalidNameLengthAndUrlSyntax() {
+        for (String name : new String[]{"", "a".repeat(101), "😀".repeat(51)}) {
+            assertInvalidReplacement("name", json.getNodeFactory().textNode(name));
+        }
+        for (String url : new String[]{"not a URL", "/ok", "ftp://127.0.0.1:4321/ok"}) {
+            assertInvalidReplacement("url", json.getNodeFactory().textNode(url));
+        }
+    }
+
+    @Test
+    void acceptsTrimmedNamesAndInclusiveIntegerValueBoundaries() throws Exception {
+        String[] names = {" x ", "a".repeat(100), "😀".repeat(50), " \tFixture monitor\n "};
+        String[] storedNames = {"x", "a".repeat(100), "😀".repeat(50), "Fixture monitor"};
+        String[] intervals = {"1", "86400", "60", "60.0"};
+        for (int i = 0; i < names.length; i++) {
+            ObjectNode input = validInput().put("name", names[i]).put("enabled", i != 0);
+            input.set("interval", json.readTree(intervals[i]));
+            JsonNode stored = assertDataEnvelope(postJson(input.toString()), HttpStatus.CREATED).get("monitor");
+            assertEquals(storedNames[i], stored.get("name").textValue());
+            assertTrue(stored.get("interval").isIntegralNumber());
+            assertEquals(json.readTree(intervals[i]).intValue(), stored.get("interval").intValue());
+            assertTrue(stored.get("enabled").isBoolean());
+            assertEquals(i != 0, stored.get("enabled").booleanValue());
+        }
+    }
+
+    @Test
+    void malformedJsonRootsAndMediaTypesUseInputErrorEnvelope() {
+        for (String body : new String[]{"null", "[]", "false", "42", "\"Fixture monitor\"", "{\"name\":", "",
+                validInput().toString() + validInput()}) {
+            assertApiError(postJson(body), HttpStatus.BAD_REQUEST, "INVALID_INPUT");
+        }
+        var headers = new HttpHeaders();
+        headers.setContentType(MediaType.TEXT_PLAIN);
+        var response = api.postForEntity("/api/monitors", new HttpEntity<>(validInput().toString(), headers), JsonNode.class);
+        assertApiError(response, HttpStatus.BAD_REQUEST, "INVALID_INPUT");
+    }
+
+    @Test
+    void missingResourcesAreDifferentFromMalformedIds() {
+        assertApiError(api.postForEntity("/api/monitors/00000000-0000-4000-8000-000000000000/checks", null,
+                JsonNode.class), HttpStatus.NOT_FOUND, "NOT_FOUND");
+        assertApiError(api.postForEntity("/api/monitors/not-a-uuid/checks", null,
+                JsonNode.class), HttpStatus.BAD_REQUEST, "INVALID_INPUT");
+        assertApiError(api.getForEntity("/api/absent", JsonNode.class), HttpStatus.NOT_FOUND, "NOT_FOUND");
+    }
+
+    @Test
+    void successWireModelKeepsJsonPrimitivesAndExplicitNulls() {
+        JsonNode view = assertDataEnvelope(postJson(validInput().toString()), HttpStatus.CREATED);
+        assertEquals(Set.of("monitor", "latestCheck"), keys(view));
+        assertTrue(view.get("latestCheck").isNull());
+        JsonNode monitor = view.get("monitor");
+        assertEquals(Set.of("id", "name", "url", "interval", "enabled", "createdAt", "updatedAt"), keys(monitor));
+        UUID.fromString(monitor.get("id").textValue());
+        Instant.parse(monitor.get("createdAt").textValue());
+        Instant.parse(monitor.get("updatedAt").textValue());
+        assertTrue(monitor.get("name").isTextual());
+        assertTrue(monitor.get("url").isTextual());
+        assertTrue(monitor.get("interval").isIntegralNumber());
+        assertTrue(monitor.get("enabled").isBoolean());
+        JsonNode check = assertDataEnvelope(api.postForEntity("/api/monitors/" + monitor.get("id").textValue()
+                + "/checks", null, JsonNode.class), HttpStatus.OK);
+        assertEquals(Set.of("id", "monitorId", "trigger", "state", "httpStatus", "latencyMs", "failureReason",
+                "startedAt", "finishedAt"), keys(check));
+        UUID.fromString(check.get("id").textValue());
+        assertEquals(monitor.get("id"), check.get("monitorId"));
+        assertEquals("MANUAL", check.get("trigger").textValue());
+        assertEquals("SUCCEEDED", check.get("state").textValue());
+        assertTrue(check.get("httpStatus").isIntegralNumber());
+        assertEquals(200, check.get("httpStatus").intValue());
+        assertTrue(check.get("latencyMs").isIntegralNumber());
+        assertTrue(check.get("failureReason").isNull());
+        Instant.parse(check.get("startedAt").textValue());
+        Instant.parse(check.get("finishedAt").textValue());
+    }
+
     @Test
     void createAndManuallyCheckSuccessfulMonitor() {
         var view = create("/ok");
@@ -79,7 +199,8 @@ class MonitorFunctionalTest {
         assertNotNull(check.id());
         assertFalse(check.finishedAt().isBefore(check.startedAt()));
         assertTrue(check.latencyMs() >= 0);
-        var listed = api.getForObject("/api/monitors", MonitorController.MonitorView[].class);
+        var listed = json.convertValue(assertDataEnvelope(api.getForEntity("/api/monitors", JsonNode.class),
+                HttpStatus.OK), MonitorController.MonitorView[].class);
         assertTrue(java.util.Arrays.stream(listed).anyMatch(row -> check.equals(row.latestCheck())));
     }
 
@@ -111,24 +232,63 @@ class MonitorFunctionalTest {
     @Test
     void rejectsNonFixtureDestinationWithoutOutboundRequest() {
         var response = api.postForEntity("/api/monitors", new MonitorController.CreateMonitor(
-                "Forbidden fixture", "http://127.0.0.1:4324/blocked", 60, true), String.class);
-        assertEquals(HttpStatus.BAD_REQUEST, response.getStatusCode());
+                "Forbidden fixture", "http://127.0.0.1:4324/blocked", 60, true), JsonNode.class);
+        assertApiError(response, HttpStatus.BAD_REQUEST, "INVALID_INPUT");
         assertEquals(0, forbiddenRequests.get());
     }
 
     private MonitorController.MonitorView create(String path) {
         var response = api.postForEntity("/api/monitors", new MonitorController.CreateMonitor(
-                "Fixture monitor", "http://127.0.0.1:4321" + path, 60, true), MonitorController.MonitorView.class);
-        assertEquals(HttpStatus.CREATED, response.getStatusCode());
-        assertNotNull(response.getBody());
-        return response.getBody();
+                "Fixture monitor", "http://127.0.0.1:4321" + path, 60, true), JsonNode.class);
+        return json.convertValue(assertDataEnvelope(response, HttpStatus.CREATED), MonitorController.MonitorView.class);
+    }
+
+    private ObjectNode validInput() {
+        return json.createObjectNode().put("name", "Fixture monitor").put("url", "http://127.0.0.1:4321/ok")
+                .put("interval", 60).put("enabled", true);
+    }
+
+    private ResponseEntity<JsonNode> postJson(String body) {
+        var headers = new HttpHeaders();
+        headers.setContentType(MediaType.APPLICATION_JSON);
+        return api.postForEntity("/api/monitors", new HttpEntity<>(body, headers), JsonNode.class);
+    }
+
+    private void assertInvalidReplacement(String field, JsonNode value) {
+        ObjectNode input = validInput();
+        input.set(field, value);
+        assertApiError(postJson(input.toString()), HttpStatus.BAD_REQUEST, "INVALID_INPUT");
+    }
+
+    private static JsonNode assertDataEnvelope(ResponseEntity<JsonNode> response, HttpStatus status) {
+        assertEquals(status, response.getStatusCode());
+        JsonNode body = response.getBody();
+        assertNotNull(body);
+        assertEquals(Set.of("data"), keys(body));
+        return body.get("data");
+    }
+
+    private static Set<String> keys(JsonNode object) {
+        assertTrue(object.isObject());
+        Set<String> names = new HashSet<>();
+        object.fieldNames().forEachRemaining(names::add);
+        return names;
+    }
+
+    private static void assertApiError(ResponseEntity<JsonNode> response, HttpStatus status, String code) {
+        assertEquals(status, response.getStatusCode(), () -> "Unexpected response: " + response);
+        JsonNode body = response.getBody();
+        assertNotNull(body);
+        assertEquals(1, body.size());
+        assertTrue(body.has("error"));
+        assertEquals(2, body.get("error").size());
+        assertEquals(code, body.at("/error/code").asText());
+        assertTrue(body.at("/error/message").isTextual());
     }
 
     private CheckRunner.CheckRun check(MonitorController.MonitorView monitor) {
         var response = api.postForEntity("/api/monitors/" + monitor.monitor().id() + "/checks", null,
-                CheckRunner.CheckRun.class);
-        assertEquals(HttpStatus.OK, response.getStatusCode());
-        assertNotNull(response.getBody());
-        return response.getBody();
+                JsonNode.class);
+        return json.convertValue(assertDataEnvelope(response, HttpStatus.OK), CheckRunner.CheckRun.class);
     }
 }
diff --git a/evidence/E02/fixtures.md b/evidence/E02/fixtures.md
new file mode 100644
index 0000000..ce5f64f
--- /dev/null
+++ b/evidence/E02/fixtures.md
@@ -0,0 +1,29 @@
+# E02 immutable verification inputs
+
+Frozen before the first baseline execution. Spec revision: `0a006589477f8ae47bad3faa5510c999cff85ee4`; start: `c3495c1478182bbea5bc47d78301e0bfa5275ab9`; attempt 1.
+
+- Fixture origin `http://127.0.0.1:4321`; API `http://127.0.0.1:4322`; browser `http://127.0.0.1:4323`. Existing forbidden destination trap: port 4324.
+- Default valid create input: `{"name":"Fixture monitor","url":"http://127.0.0.1:4321/ok","interval":60,"enabled":true}`.
+- Baseline: replace name with exactly three spaces. Expect HTTP 400 and `error.code=INVALID_INPUT`. Stop baseline after this one counterexample; execute this identical assertion after the fix.
+- Required invalid replacements: interval `"60"`, `0`, `86401`; URL `"not a URL"`. Each expects 400 / INVALID_INPUT.
+- Missing resource: POST `/api/monitors/00000000-0000-4000-8000-000000000000/checks` expects 404 / NOT_FOUND. Malformed ID `not-a-uuid` expects 400 / INVALID_INPUT. GET `/api/absent` expects 404 / NOT_FOUND.
+- Default create expects 201 / data; manual check expects 200 / data with SUCCEEDED, status 200 and null failureReason. URL `/fail` expects API 200 / data with FAILED, status 503 and HTTP_STATUS. Existing E01 redirect and outbound-denial cases remain unchanged.
+- Browser create errors: HTTP 400 / INVALID_INPUT with `Arbitrary server prose A` and `Different server prose B`. Both must display the same input-error category, retain form values and create no Monitor or Check success UI.
+
+Additional boundary cases frozen at the same time:
+
+- Name: empty string, null, number 60, boolean true, array, object, omitted field, 101 ASCII `a` code units, 51 repetitions of `😀` are invalid. Valid names: `" x "` stores `x`; `" \tFixture monitor\n "` stores `Fixture monitor`; 100 ASCII `a` or 50 repetitions of `😀` are accepted. Length means UTF-16 code units after trimming.
+- URL: null, number 60, boolean true, array, object, omitted field, `/ok`, `ftp://127.0.0.1:4321/ok` are invalid. E01's HTTPS/alternate host/port/credentials/fragment denial cases remain unchanged.
+- Interval: null, true, array, object, omitted field, 1.5, 2147483648 are invalid; integer values 1, 60.0 and 86400 are accepted.
+- Enabled: null, string `"true"`, numbers 0 and 1, array, object, omitted field are invalid; booleans true and false are accepted.
+- JSON root: `null`, `[]`, `false`, `42`, `"Fixture monitor"`; malformed body `{"name":`; two consecutive default-valid JSON objects are invalid. No body is invalid. Wrong content type `text/plain` is invalid.
+- Unexpected MVC failure: test mock throws `IllegalStateException("Private implementation detail")`; expect exactly HTTP 500 / INTERNAL_ERROR without the implementation detail in the wire response.
+- No-response checks: isolated fixture port 4325; closed listener for CONNECTION_FAILURE, and a response-header barrier released only after the existing 2000 ms read timeout for TIMEOUT. Both must retain null httpStatus, FAILED and the corresponding reason. No latency threshold is an acceptance condition.
+- Serialization: exact data/error envelope keys; Monitor IDs/times are JSON strings, interval and latency are numbers, enabled is boolean, absent latestCheck/httpStatus/failureReason are JSON null (not omitted or invented).
+- Additional browser failures: NOT_FOUND/404 and INTERNAL_ERROR/500 use `Arbitrary server prose A`; malformed failure body `{"unexpected":true}` and a network abort use the INTERNAL_ERROR UI fallback. A failed check starts with an existing Monitor and null latestCheck, and must not show SUCCEEDED/FAILED as if a Check result was received. Malformed success body `{"data":{}}` must not add a Monitor.
+
+Browser scaffolding recorded before its first execution: the failed-check page receives a mocked stale Monitor row with the fixed unknown UUID above, the default valid fields, null latestCheck, and timestamp strings `2026-08-28T00:00:00Z`. It does not create a duplicate in the real API state used by the unchanged E01 browser regressions. The malformed error response uses status 400; malformed success uses status 201.
+
+These are sequential functional tests, not load or benchmarks. One Chromium worker, retries 0. No parameter changes, sweeps, profilers or performance optimization.
+
+Pre-execution clarification from the main packet owner: the additional, unexecuted `60.0` case was initially listed as invalid. After the whitespace-name baseline, but before any production edit or interval-case execution, the owner clarified that JSON integer *values* are required, not an integer lexical spelling. `60.0` therefore means numeric 60 and is accepted. Every original packet input/expectation and the reproduced whitespace-name assertion is unchanged. This is a recorded contract clarification, not tuning in response to a test result.


