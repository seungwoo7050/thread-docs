## `feat(security): require trusted Origin for browser mutations`

diff --git a/TRACK.md b/TRACK.md
index 4fc5b46..314610d 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -84,7 +84,7 @@ The default connection is database `monitor`, local test identity `wse_industry`
 
 ## PostgreSQL boundary (E03)
 
-- Flyway is the sole schema writer: unchanged V1 creates `monitors`, unchanged V2 creates `check_runs`, and appended V3 creates `users`. Repeated startup validates migration checksums and applies only pending migrations. Hibernate uses `ddl-auto=validate`, never create/update.
+- Flyway is the schema migration authority: unchanged V1 creates `monitors`, unchanged V2 creates `check_runs`, V3 creates `users`, and V4 requires explicitly assigned Monitor owners (see the E05 upgrade procedure below). Repeated startup validates migration checksums and applies only pending migrations. Hibernate uses `ddl-auto=validate`, never create/update.
 - A startup metadata check supplements Hibernate validation by rejecting unmapped required columns without defaults. Missing mapped columns or incompatible insert requirements prevent the web server from becoming ready.
 - `MonitorEntity.fromDomain/toDomain` and `CheckRunEntity.fromDomain/toDomain` explicitly map canonical immutable records. Entities never reach JSON. UUIDs remain UUIDs; interval is PostgreSQL integer, enabled boolean, latency bigint, and nullable HTTP status/reason retain null rather than zero/empty text.
 - Timestamp values are truncated to microseconds before the first response and stored as `timestamp(6) with time zone`. Canonical Java Instant/JSON values remain UTC even in a non-UTC PostgreSQL session.
@@ -119,8 +119,8 @@ The default connection is database `monitor`, local test identity `wse_industry`
   `GET /api/session/csrf` supplies its request token; the UI fetches a fresh token
   before each mutation and sends it only in the indicated header. This token is
   not a login credential. Missing authentication stays 401 even when CSRF rejects
-  first; an authenticated invalid-token request is 403. Additional E05 ownership,
-  cross-origin policy and authorization matrices are not introduced here.
+  first; an authenticated invalid-token request is 403. E05 adds the owner and
+  exact-Origin boundaries described below while preserving this token mechanism.
 - The Security starter is the only new dependency: it directly supplies the
   required filter boundary, credential verification, session strategies and
   existing framework CSRF protection. All managed versions remain fixed.
@@ -132,7 +132,7 @@ npx playwright install chromium
 npm run verify
 ```
 
-`verify` starts the isolated PostgreSQL project, runs Maven unit, real-HTTP authentication/functional and real-PostgreSQL integration tests, and packages the API. It then checks TypeScript, compiles Next for production, runs the unchanged A,A,B process-restart/lifecycle product sequence with authentication setup, and runs Chromium against the local development UI and packaged API. It does not retry failed tests. Command outcomes and elapsed times are appended to `output/verification/runs.jsonl`; the process-restart probe saves only product wire evidence in `output/e03`. Committed evidence is in `evidence/E01` through `evidence/E04`.
+`verify` starts the isolated PostgreSQL project, runs Maven unit, real-HTTP authentication/functional and real-PostgreSQL integration tests, and packages the API. It then checks TypeScript, compiles Next for production, runs the unchanged A,A,B process-restart/lifecycle product sequence with authentication setup, and runs Chromium against the local development UI and packaged API. It does not retry failed tests. Command outcomes and elapsed times are appended to `output/verification/runs.jsonl`; the process-restart probe saves only product wire evidence in `output/e03`. Committed evidence is in `evidence/E01` through `evidence/E05`.
 
 Tests create and remove isolated schemas for functional, browser, restart, mapping, migration, and incompatible-schema fixtures. The standard runner cleans up its browser/restart schemas even after a failure. The database remains available afterward; use `npm run db:down` to stop only this project. The Java tests explicitly close independent application contexts to verify restart persistence, capture actual generated SQL/transaction flags, check rollback and cascade, and assert startup rejection for both incompatible-schema cases.
 
@@ -142,6 +142,12 @@ and real browser login/logout and session-loss recovery. Existing browser cases
 receive only auth setup; their product inputs and assertions remain unchanged.
 Credential-bearing traces, screenshots, videos and storage-state files are disabled.
 
+E05 adds a fixed two-user authorization matrix, no-write/no-outbound assertions,
+the complete Origin/CSRF mutation matrix, separate CORS response-header checks,
+explicit previous-V3 ownership upgrade/refusal, and two isolated real browser
+sessions. New Java fixtures use only declared `e05_*` schemas; all earlier product
+assertions remain unchanged with required authentication/owner/Origin setup.
+
 The CI workflow installs the exact toolchain and runs the same gates. No hosted CI run is claimed by local verification. The browser gate starts and stops its own processes and refuses existing servers. There are no load tests, benchmarks, or parameter sweeps.
 
 Individual commands:
@@ -219,3 +225,41 @@ If ownership cannot be established, keep startup stopped and preserve the data;
 do not invent an owner or delete rows to make migration pass. Startup also checks
 owner nullability and the users foreign key, because Hibernate's column-type
 validation alone does not establish those guarantees.
+
+## Browser mutation and CORS boundary (E05)
+
+Every state-changing request, including login and logout, requires exactly one
+`Origin: http://127.0.0.1:4323` header **and** the existing session-bound Spring
+Security CSRF token. Missing, literal `null`, foreign or multiple Origins are
+rejected. The trusted origin is fixed for this loopback product, not inferred from
+Host, forwarded headers, Referer, a suffix match or a user-provided URL. A valid
+Origin never bypasses the default `CsrfFilter`, its session repository or its
+masked-token handler. Unauthenticated failures remain 401 / UNAUTHENTICATED;
+authenticated invalid evidence is 403 / FORBIDDEN before any controller mutation,
+outbound request or logout. An anonymous CSRF bootstrap does not authenticate.
+
+Login uses a pre-authentication CSRF session obtained from `/api/session/csrf`.
+Only a valid token for that session and the exact browser Origin can submit
+credentials. Successful login retains E04 session-ID rotation and token reset;
+expiry remains exactly one absolute hour and logout still invalidates the old
+session. The browser receives no authentication token in JavaScript.
+
+CORS is a separate browser response-sharing policy. The application grants no
+cross-origin access and rejects preflight requests with 403 / FORBIDDEN and no
+`Access-Control-Allow-Origin` or `Access-Control-Allow-Credentials` headers. Even
+the trusted UI must call its same-origin Next `/api` proxy; direct cross-origin
+API-port requests do not receive a CORS grant. Lack of CORS permission is not used
+as the CSRF defense: simple cross-origin writes are independently rejected by
+Origin and session CSRF checks. Safe authenticated reads need no Origin, but do
+not acquire cross-origin response-sharing permission.
+
+The existing browser fetch code already supplies the session CSRF header and the
+browser supplies Origin automatically. Non-browser regression clients explicitly
+send the same trusted Origin; no frontend state-cache or rendering redesign is
+part of E05. The filter runs after session-expiry/context loading and before the
+default CSRF and logout filters.
+
+References: [Spring Security CSRF](https://docs.spring.io/spring-security/reference/servlet/exploits/csrf.html),
+[Spring Security CORS](https://docs.spring.io/spring-security/reference/servlet/integrations/cors.html),
+and the installed Next 16.3.3 `rewrites` guide. Runtime behavior is verified against
+the pinned framework, not inferred only from the documentation.
diff --git a/backend/src/main/java/dev/evolution/monitor/AuthenticationConfig.java b/backend/src/main/java/dev/evolution/monitor/AuthenticationConfig.java
index 48908f9..3288033 100644
--- a/backend/src/main/java/dev/evolution/monitor/AuthenticationConfig.java
+++ b/backend/src/main/java/dev/evolution/monitor/AuthenticationConfig.java
@@ -24,6 +24,7 @@ import org.springframework.security.web.context.HttpSessionSecurityContextReposi
 import org.springframework.security.web.context.SecurityContextHolderFilter;
 import org.springframework.security.web.context.SecurityContextRepository;
 import org.springframework.security.web.csrf.CsrfTokenRepository;
+import org.springframework.security.web.csrf.CsrfFilter;
 import org.springframework.security.web.csrf.HttpSessionCsrfTokenRepository;
 
 @Configuration
@@ -62,6 +63,8 @@ public class AuthenticationConfig {
         return http
                 // Keep the framework's session-backed CSRF protection for browser requests.
                 .csrf(csrf -> csrf.csrfTokenRepository(csrfTokens))
+                // Same-origin Next proxy only: do not emit credentialed cross-origin permissions.
+                .cors(AbstractHttpConfigurer::disable)
                 .httpBasic(AbstractHttpConfigurer::disable)
                 .formLogin(AbstractHttpConfigurer::disable)
                 .requestCache(AbstractHttpConfigurer::disable)
@@ -103,6 +106,7 @@ public class AuthenticationConfig {
                         }))
                 // Invalidate before the repository can load an expired SecurityContext.
                 .addFilterBefore(new SessionExpiryFilter(clock), SecurityContextHolderFilter.class)
+                .addFilterBefore(new BrowserOriginFilter(json), CsrfFilter.class)
                 .build();
     }
 }
diff --git a/backend/src/main/java/dev/evolution/monitor/BrowserOriginFilter.java b/backend/src/main/java/dev/evolution/monitor/BrowserOriginFilter.java
new file mode 100644
index 0000000..d6b9446
--- /dev/null
+++ b/backend/src/main/java/dev/evolution/monitor/BrowserOriginFilter.java
@@ -0,0 +1,53 @@
+package dev.evolution.monitor;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import jakarta.servlet.FilterChain;
+import jakarta.servlet.ServletException;
+import jakarta.servlet.http.HttpServletRequest;
+import jakarta.servlet.http.HttpServletResponse;
+import java.io.IOException;
+import org.springframework.http.MediaType;
+import org.springframework.security.authentication.AnonymousAuthenticationToken;
+import org.springframework.security.core.context.SecurityContextHolder;
+import org.springframework.security.web.csrf.CsrfFilter;
+import org.springframework.web.cors.CorsUtils;
+import org.springframework.web.filter.OncePerRequestFilter;
+
+final class BrowserOriginFilter extends OncePerRequestFilter {
+    static final String TRUSTED_ORIGIN = "http://127.0.0.1:4323";
+    private final ObjectMapper json;
+
+    BrowserOriginFilter(ObjectMapper json) { this.json = json; }
+
+    @Override
+    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
+            throws ServletException, IOException {
+        // Browser API calls use the same-origin Next proxy. No cross-origin CORS grant exists,
+        // including for the trusted UI when it tries to address the API port directly.
+        if (CorsUtils.isPreFlightRequest(request)) {
+            reject(response, 403);
+            return;
+        }
+        if (CsrfFilter.DEFAULT_CSRF_MATCHER.matches(request) && !trustedOrigin(request)) {
+            var authentication = SecurityContextHolder.getContext().getAuthentication();
+            boolean signedIn = authentication != null && authentication.isAuthenticated()
+                    && !(authentication instanceof AnonymousAuthenticationToken);
+            reject(response, signedIn ? 403 : 401);
+            return;
+        }
+        // A matching Origin is additional evidence; the default CsrfFilter still checks the session token.
+        chain.doFilter(request, response);
+    }
+
+    private static boolean trustedOrigin(HttpServletRequest request) {
+        var origins = request.getHeaders("Origin");
+        return origins.hasMoreElements() && TRUSTED_ORIGIN.equals(origins.nextElement()) && !origins.hasMoreElements();
+    }
+
+    private void reject(HttpServletResponse response, int status) throws IOException {
+        response.setStatus(status);
+        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
+        json.writeValue(response.getOutputStream(), status == 401 ? ApiErrors.unauthenticatedBody()
+                : new ApiErrors.Failure(new ApiErrors.Detail(ApiErrors.Code.FORBIDDEN, "Request could not be verified.")));
+    }
+}
diff --git a/backend/src/test/java/dev/evolution/monitor/OwnershipAuthorizationTest.java b/backend/src/test/java/dev/evolution/monitor/OwnershipAuthorizationTest.java
index 7e85da2..f56c51d 100644
--- a/backend/src/test/java/dev/evolution/monitor/OwnershipAuthorizationTest.java
+++ b/backend/src/test/java/dev/evolution/monitor/OwnershipAuthorizationTest.java
@@ -27,7 +27,15 @@ import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.boot.test.context.SpringBootTest;
 import org.springframework.boot.test.web.client.TestRestTemplate;
 import org.springframework.http.HttpMethod;
+import org.springframework.http.HttpEntity;
+import org.springframework.http.HttpHeaders;
 import org.springframework.http.ResponseEntity;
+import org.springframework.security.web.SecurityFilterChain;
+import org.springframework.security.web.authentication.logout.LogoutFilter;
+import org.springframework.security.web.context.SecurityContextHolderFilter;
+import org.springframework.security.web.csrf.CsrfFilter;
+import org.springframework.security.web.csrf.CsrfTokenRepository;
+import org.springframework.security.web.csrf.HttpSessionCsrfTokenRepository;
 import org.springframework.test.annotation.DirtiesContext;
 import org.springframework.test.context.DynamicPropertyRegistry;
 import org.springframework.test.context.DynamicPropertySource;
@@ -47,6 +55,8 @@ class OwnershipAuthorizationTest {
     @Autowired ObjectMapper json;
     @Autowired MonitorStore store;
     @Autowired UserAccounts accounts;
+    @Autowired SecurityFilterChain security;
+    @Autowired CsrfTokenRepository csrfTokens;
     @MockitoSpyBean CheckRunner checks;
     private SessionClient alice;
     private SessionClient bob;
@@ -175,6 +185,130 @@ class OwnershipAuthorizationTest {
                         "proxyAndSqlChecked", true, "outboundTransactionAbsent", true)) + "\n");
     }
 
+    @Test
+    void everyStateChangeRequiresExactOriginAndSessionCsrfIncludingLogout() throws Exception {
+        assertTrue(csrfTokens instanceof HttpSessionCsrfTokenRepository);
+        var chain = security.getFilters();
+        assertTrue(index(chain, SessionExpiryFilter.class) < index(chain, SecurityContextHolderFilter.class));
+        assertTrue(index(chain, SecurityContextHolderFilter.class) < index(chain, BrowserOriginFilter.class));
+        assertTrue(index(chain, BrowserOriginFilter.class) < index(chain, CsrfFilter.class));
+        assertTrue(index(chain, CsrfFilter.class) < index(chain, LogoutFilter.class));
+        alice.csrf();
+        bob.csrf();
+        var wrong = alice.copyCredential();
+        wrong.useIncorrectCsrfProof();
+        var invalidProofs = List.of(
+                new Proof("missing-csrf", SessionClient.TRUSTED_ORIGIN, null),
+                new Proof("incorrect-csrf", SessionClient.TRUSTED_ORIGIN, wrong),
+                new Proof("other-session-csrf", SessionClient.TRUSTED_ORIGIN, bob),
+                new Proof("missing-origin", null, alice),
+                new Proof("null-origin", "null", alice),
+                new Proof("foreign-origin", "http://127.0.0.1:4999", alice));
+        var writes = List.of(
+                new Mutation("create", HttpMethod.POST, "/api/monitors", input("Verified creation", "/ok", 60, true)),
+                new Mutation("edit", HttpMethod.PUT, path(aliceId), input("Verified edit", "/ok", 90, true)),
+                new Mutation("pause", HttpMethod.PUT, path(aliceId), input("Verified edit", "/ok", 90, false)),
+                new Mutation("resume", HttpMethod.PUT, path(aliceId), input("Verified edit", "/ok", 90, true)),
+                new Mutation("check", HttpMethod.POST, path(aliceId) + "/checks", null),
+                new Mutation("delete", HttpMethod.DELETE, path(aliceId), null),
+                new Mutation("logout", HttpMethod.POST, "/api/session/logout", null));
+        for (var write : writes) {
+            for (var proof : invalidProofs) {
+                String before = snapshot();
+                int sent = outbound.get();
+                error(alice.sendWithEvidence(write.path, write.method, write.body, proof.origin, proof.session), 403, "FORBIDDEN");
+                assertEquals(before, snapshot());
+                assertEquals(sent, outbound.get());
+                assertEquals(200, alice.get("/api/session").getStatusCode().value(), "Denied logout must retain the session");
+                evidence.add(Map.of("mutation", write.name, "invalidEvidence", proof.name, "status", 403,
+                        "allRowsUnchanged", true, "outboundUnchanged", true, "sessionRetained", true));
+            }
+        }
+        JsonNode bobBefore = data(bob.get(path(bobId)), 200);
+        SessionClient revoked = null;
+        SessionClient csrfAlone = null;
+        for (var write : writes) {
+            if (write.name.equals("logout")) {
+                alice.csrf();
+                revoked = alice.copyCredential();
+                csrfAlone = alice.csrfOnly();
+            }
+            data(alice.mutate(write.path, write.method, write.body), write.name.equals("create") ? 201 : 200);
+        }
+        assertEquals(bobBefore, data(bob.get(path(bobId)), 200));
+        assertEquals(3, outbound.get());
+        assertNotNull(revoked);
+        error(revoked.get("/api/monitors"), 401, "UNAUTHENTICATED");
+        error(revoked.send("/api/session/logout", HttpMethod.POST, null, true), 401, "UNAUTHENTICATED");
+        error(csrfAlone.send("/api/monitors", HttpMethod.POST, input("Not authenticated", "/ok", 60, true), true),
+                401, "UNAUTHENTICATED");
+        error(new SessionClient(api).mutate("/api/session/logout", HttpMethod.POST, null), 401, "UNAUTHENTICATED");
+        Files.createDirectories(Path.of("target"));
+        Files.writeString(Path.of("target/e05-browser-mutation-evidence.json"), json.writerWithDefaultPrettyPrinter()
+                .writeValueAsString(Map.of("deniedWrites", evidence, "authorizedWrites", writes.stream().map(Mutation::name).toList(),
+                        "oldCookieRevoked", true, "csrfAloneDoesNotAuthenticate", true, "anonymousCsrfLogout401", true,
+                        "defaultSessionCsrfRepository", true, "runtimeFilterOrderVerified", true)) + "\n");
+    }
+
+    @Test
+    void loginRequiresPreauthenticationCsrfAndOriginWhileCorsGrantsNoCredentialAccess() throws Exception {
+        var anonymous = new SessionClient(api);
+        anonymous.csrf();
+        alice.csrf();
+        var beforeLogin = anonymous.copyCredential();
+        var wrong = anonymous.copyCredential();
+        wrong.useIncorrectCsrfProof();
+        var loginBody = Map.of("username", "alice-e04", "password", ALICE_PASSWORD);
+        for (var proof : List.of(new Proof("missing-origin", null, anonymous),
+                new Proof("null-origin", "null", anonymous), new Proof("foreign-origin", "http://127.0.0.1:4999", anonymous),
+                new Proof("missing-csrf", SessionClient.TRUSTED_ORIGIN, null),
+                new Proof("incorrect-csrf", SessionClient.TRUSTED_ORIGIN, wrong),
+                new Proof("other-session-csrf", SessionClient.TRUSTED_ORIGIN, alice))) {
+            error(anonymous.sendWithEvidence("/api/session/login", HttpMethod.POST, loginBody, proof.origin, proof.session),
+                    401, "UNAUTHENTICATED");
+            assertTrue(anonymous.sameCredential(beforeLogin), "Denied login does not rotate the anonymous session");
+            error(anonymous.get("/api/session"), 401, "UNAUTHENTICATED");
+        }
+        data(anonymous.login("alice-e04", ALICE_PASSWORD), 200);
+        assertFalse(anonymous.sameCredential(beforeLogin));
+        anonymous.csrf();
+        var signedIn = anonymous.copyCredential();
+        error(anonymous.sendWithEvidence("/api/session/login", HttpMethod.POST, loginBody, null, anonymous), 403, "FORBIDDEN");
+        assertTrue(anonymous.sameCredential(signedIn));
+        assertEquals(200, anonymous.get("/api/session").getStatusCode().value());
+
+        var cors = new ArrayList<Map<String, Object>>();
+        for (String origin : List.of(SessionClient.TRUSTED_ORIGIN, "http://127.0.0.1:4999", "null")) {
+            var headers = new HttpHeaders();
+            headers.setOrigin(origin);
+            headers.set("Access-Control-Request-Method", "POST");
+            headers.set("Access-Control-Request-Headers", "content-type,x-csrf-token");
+            var preflight = api.exchange("/api/monitors", HttpMethod.OPTIONS, new HttpEntity<>(headers), JsonNode.class);
+            error(preflight, 403, "FORBIDDEN");
+            noCorsGrant(preflight);
+            var credentialedRead = alice.sendWithEvidence("/api/monitors", HttpMethod.GET, null, origin, null);
+            data(credentialedRead, 200);
+            noCorsGrant(credentialedRead);
+            cors.add(Map.of("origin", origin, "preflightStatus", 403, "authenticatedReadStatus", 200,
+                    "allowOriginHeader", false, "allowCredentialsHeader", false));
+        }
+        Files.createDirectories(Path.of("target"));
+        Files.writeString(Path.of("target/e05-login-cors-evidence.json"), json.writerWithDefaultPrettyPrinter()
+                .writeValueAsString(Map.of("invalidAnonymousLoginEvidenceStatus", 401, "invalidAuthenticatedLoginEvidenceStatus", 403,
+                        "anonymousSessionNotRotatedOnDenial", true, "validLoginRotates", true, "cors", cors)) + "\n");
+    }
+
+    private static int index(List<jakarta.servlet.Filter> filters, Class<?> type) {
+        for (int i = 0; i < filters.size(); i++) if (type.isInstance(filters.get(i))) return i;
+        fail("Required security filter missing: " + type.getSimpleName());
+        return -1;
+    }
+
+    private static void noCorsGrant(ResponseEntity<?> response) {
+        assertFalse(response.getHeaders().containsKey("Access-Control-Allow-Origin"));
+        assertFalse(response.getHeaders().containsKey("Access-Control-Allow-Credentials"));
+    }
+
     private List<Mutation> mutations(String id) {
         return List.of(new Mutation("edit", HttpMethod.PUT, path(id), input("Forbidden edit", "/ok", 90, true)),
                 new Mutation("pause", HttpMethod.PUT, path(id), input("Monitor A", "/ok", 60, false)),
@@ -239,6 +373,7 @@ class OwnershipAuthorizationTest {
 
     private record Actor(String name, SessionClient client, String ownId, String ownCheck, String foreignId, String foreignCheck) {}
     private record Mutation(String name, HttpMethod method, String path, Object body) {}
+    private record Proof(String name, String origin, SessionClient session) {}
     record SqlEvent(String sql, boolean transaction, boolean readOnly) {}
 
     public static class SqlEvidence implements StatementInspector {
diff --git a/backend/src/test/java/dev/evolution/monitor/SessionClient.java b/backend/src/test/java/dev/evolution/monitor/SessionClient.java
index 2e1595b..0bfd5f5 100644
--- a/backend/src/test/java/dev/evolution/monitor/SessionClient.java
+++ b/backend/src/test/java/dev/evolution/monitor/SessionClient.java
@@ -32,9 +32,19 @@ final class SessionClient {
     SessionClient copyCredential() {
         var copy = new SessionClient(api);
         copy.cookie = cookie;
+        copy.csrfHeader = csrfHeader;
+        copy.csrfToken = csrfToken;
         return copy;
     }
 
+    SessionClient csrfOnly() {
+        var copy = copyCredential();
+        copy.cookie = null;
+        return copy;
+    }
+
+    void useIncorrectCsrfProof() { csrfToken = password(); }
+
     boolean sameCredential(SessionClient other) { return cookie != null && cookie.equals(other.cookie); }
 
     void useInvalidCredential() { cookie = AuthenticationConfig.COOKIE_NAME + "=" + password(); }
@@ -59,12 +69,16 @@ final class SessionClient {
     }
 
     ResponseEntity<JsonNode> send(String path, HttpMethod method, Object body, boolean includeCsrf) {
+        String origin = method == HttpMethod.GET || method == HttpMethod.HEAD || method == HttpMethod.OPTIONS
+                ? null : TRUSTED_ORIGIN;
+        return sendWithEvidence(path, method, body, origin, includeCsrf ? this : null);
+    }
+
+    ResponseEntity<JsonNode> sendWithEvidence(String path, HttpMethod method, Object body, String origin, SessionClient proof) {
         var headers = new HttpHeaders();
         if (cookie != null) headers.set(HttpHeaders.COOKIE, cookie);
-        if (method != HttpMethod.GET && method != HttpMethod.HEAD && method != HttpMethod.OPTIONS) {
-            headers.setOrigin(TRUSTED_ORIGIN);
-        }
-        if (includeCsrf) headers.set(csrfHeader, csrfToken);
+        if (origin != null) headers.setOrigin(origin);
+        if (proof != null) headers.set(proof.csrfHeader, proof.csrfToken);
         if (body != null) headers.setContentType(MediaType.APPLICATION_JSON);
         var response = api.exchange(path, method, new HttpEntity<>(body, headers), JsonNode.class);
         for (String value : response.getHeaders().getOrEmpty(HttpHeaders.SET_COOKIE)) {
diff --git a/scripts/persistence-scenario.mjs b/scripts/persistence-scenario.mjs
index 83e1f8e..fef42f7 100644
--- a/scripts/persistence-scenario.mjs
+++ b/scripts/persistence-scenario.mjs
@@ -97,7 +97,7 @@ async function csrfHeaders() {
   if (cookie) sessionCookie = cookie.split(';', 1)[0];
   const wire = await response.json();
   assert.ok(sessionCookie && wire.data?.headerName === 'X-CSRF-TOKEN' && typeof wire.data?.token === 'string');
-  return { Cookie: sessionCookie, [wire.data.headerName]: wire.data.token };
+  return { Cookie: sessionCookie, [wire.data.headerName]: wire.data.token, Origin: 'http://127.0.0.1:4323' };
 }
 
 async function authenticate() {
diff --git a/tests/browser/authenticated.ts b/tests/browser/authenticated.ts
index f11dabe..f24b6f4 100644
--- a/tests/browser/authenticated.ts
+++ b/tests/browser/authenticated.ts
@@ -13,7 +13,7 @@ export async function csrfHeaders(request: APIRequestContext): Promise<Record<st
   try { body = await response.json(); }
   catch { throw new Error('CSRF bootstrap returned invalid JSON'); }
   expect(body.data?.headerName === 'X-CSRF-TOKEN' && typeof body.data?.token === 'string').toBe(true);
-  return { [body.data.headerName]: body.data.token };
+  return { [body.data.headerName]: body.data.token, Origin: 'http://127.0.0.1:4323' };
 }
 
 export async function safeRequest(action: () => Promise<APIResponse>): Promise<APIResponse> {
diff --git a/tests/browser/ownership.spec.ts b/tests/browser/ownership.spec.ts
new file mode 100644
index 0000000..c832217
--- /dev/null
+++ b/tests/browser/ownership.spec.ts
@@ -0,0 +1,259 @@
+import { mkdirSync, writeFileSync } from 'node:fs';
+import { createServer } from 'node:http';
+import { test, expect, type APIRequestContext, type BrowserContext, type Page, type Request, type Response } from '@playwright/test';
+import { csrfHeaders, fillSecret, fixturePassword, safeRequest } from './authenticated';
+
+const ui = 'http://127.0.0.1:4323';
+const missing = '00000000-0000-0000-0000-000000000000';
+
+async function signIn(page: Page, user: 'alice' | 'bob') {
+  await page.goto('/login');
+  await page.getByLabel('Username', { exact: true }).fill(`${user}-e04`);
+  await fillSecret(page, fixturePassword(user));
+  await page.getByRole('button', { name: 'Sign in', exact: true }).click();
+  await expect(page).toHaveURL('/monitors');
+}
+
+async function clearOwned(request: APIRequestContext) {
+  const response = await safeRequest(() => request.get('/api/monitors'));
+  expect(response.status()).toBe(200);
+  const rows = (await response.json()).data;
+  for (const row of rows) {
+    const headers = await csrfHeaders(request);
+    const deleted = await safeRequest(() => request.delete(`/api/monitors/${row.monitor.id}`, { headers }));
+    expect(deleted.status()).toBe(200);
+  }
+}
+
+async function createAndCheck(page: Page, name: string, route: '/ok' | '/fail', interval: number) {
+  const listLoaded = page.waitForResponse(response => new URL(response.url()).pathname === '/api/monitors'
+    && response.request().method() === 'GET');
+  await page.reload();
+  expect((await listLoaded).status()).toBe(200);
+  await page.getByLabel('Name', { exact: true }).fill(name);
+  await page.getByLabel('URL', { exact: true }).fill(`http://127.0.0.1:4321${route}`);
+  await page.getByLabel('Interval (seconds)', { exact: true }).fill(String(interval));
+  await page.getByLabel('Enabled', { exact: true }).check();
+  const createdResponse = page.waitForResponse(response => new URL(response.url()).pathname === '/api/monitors'
+    && response.request().method() === 'POST');
+  await page.getByRole('button', { name: 'Create monitor', exact: true }).click();
+  const created = await createdResponse;
+  expect(created.status()).toBe(201);
+  const id = (await created.json()).data.monitor.id as string;
+  const article = page.getByRole('article', { name, exact: true });
+  const checkResponse = page.waitForResponse(response => new URL(response.url()).pathname === `/api/monitors/${id}/checks`
+    && response.request().method() === 'POST');
+  await article.getByRole('button', { name: 'Run check', exact: true }).click();
+  const checked = await checkResponse;
+  expect(checked.status()).toBe(200);
+  const check = (await checked.json()).data;
+  await expect(article.getByText(route === '/ok' ? 'SUCCEEDED' : 'FAILED', { exact: true })).toBeVisible();
+  await article.getByRole('button', { name: 'Show history', exact: true }).click();
+  await expect(article.getByRole('table', { name: 'Check history' }).getByRole('row')).toHaveCount(2);
+  return { id, checkId: check.id as string };
+}
+
+async function browserApi(page: Page, path: string, method = 'GET', body?: Record<string, unknown>, withCsrf = true) {
+  return page.evaluate(async ({ path, method, body, withCsrf }) => {
+    const headers: Record<string, string> = {};
+    if (method !== 'GET' && withCsrf) {
+      const bootstrap = await fetch('/api/session/csrf', { credentials: 'same-origin' });
+      const csrf = (await bootstrap.json()).data;
+      headers[csrf.headerName] = csrf.token;
+    }
+    if (body !== undefined) headers['Content-Type'] = 'application/json';
+    const response = await fetch(path, { method, credentials: 'same-origin', headers,
+      body: body === undefined ? undefined : JSON.stringify(body) });
+    // Only a product/error response leaves the browser; the CSRF response never does.
+    return { status: response.status, body: await response.json() };
+  }, { path, method, body, withCsrf });
+}
+
+test('two real browser sessions isolate resources and retain authorized lifecycle under Origin and CSRF guards', async ({ browser }) => {
+  // The orchestrator assigned this otherwise-free port for the already frozen attacker Origin.
+  const attackerServer = createServer((request, response) => {
+    if (request.url !== '/attack') { response.writeHead(204).end(); return; }
+    response.writeHead(200, { 'Content-Type': 'text/html', 'Cache-Control': 'no-store' })
+      .end('<!doctype html><title>Isolated foreign-origin fixture</title><link rel="icon" href="data:,"><body></body>');
+  });
+  await new Promise<void>((resolve, reject) => {
+    attackerServer.once('error', () => reject(new Error('Refusing occupied attacker fixture port 4999')));
+    attackerServer.listen({ host: '127.0.0.1', port: 4999, exclusive: true }, resolve);
+  });
+  let aliceContext: BrowserContext | undefined;
+  let bobContext: BrowserContext | undefined;
+  let completed: Record<string, unknown> | undefined;
+  const observed: { path: string; method: string; origin: string | null; csrfPresent: boolean }[] = [];
+  const headerReads: Promise<void>[] = [];
+  let headerReadFailed = false;
+  try {
+    aliceContext = await browser.newContext({ baseURL: ui });
+    bobContext = await browser.newContext({ baseURL: ui });
+    const alice = await aliceContext.newPage();
+    const bob = await bobContext.newPage();
+    for (const page of [alice, bob]) page.on('request', request => {
+      if (request.url().startsWith(`${ui}/api/`) && !['GET', 'HEAD', 'OPTIONS'].includes(request.method())) {
+        const headers = request.headers();
+        const entry = { path: new URL(request.url()).pathname, method: request.method(),
+          origin: null as string | null, csrfPresent: !!headers['x-csrf-token'] };
+        observed.push(entry);
+        // Browser-added headers can be absent from the initial synchronous header snapshot.
+        headerReads.push(request.headerValue('origin').then(origin => { entry.origin = origin; })
+          .catch(() => { headerReadFailed = true; }));
+      }
+    });
+    await signIn(alice, 'alice');
+    await signIn(bob, 'bob');
+    for (const page of [alice, bob]) expect(await page.evaluate(() => location.origin)).toBe(ui);
+    await clearOwned(alice.request);
+    await clearOwned(bob.request);
+    const a = await createAndCheck(alice, 'Monitor A', '/ok', 60);
+    const b = await createAndCheck(bob, 'Monitor B', '/fail', 120);
+    await alice.reload();
+    await bob.reload();
+    await expect(alice.getByRole('article')).toHaveCount(1);
+    await expect(bob.getByRole('article')).toHaveCount(1);
+    await expect(alice.getByRole('article', { name: 'Monitor B', exact: true })).toHaveCount(0);
+    await expect(bob.getByRole('article', { name: 'Monitor A', exact: true })).toHaveCount(0);
+    const notFound = (await browserApi(bob, `/api/monitors/${a.id}`)).body;
+    expect(notFound.error.code).toBe('NOT_FOUND');
+    for (const [page, own, foreign] of [[alice, a, b], [bob, b, a]] as const) {
+      const collection = await browserApi(page, '/api/monitors');
+      expect(collection.status).toBe(200);
+      expect(collection.body.data.map((row: { monitor: { id: string } }) => row.monitor.id)).toEqual([own.id]);
+      expect((await browserApi(page, `/api/monitors/${own.id}`)).body.data.latestCheck.id).toBe(own.checkId);
+      expect((await browserApi(page, `/api/monitors/${own.id}/checks`)).body.data).toHaveLength(1);
+      expect((await browserApi(page, `/api/monitors/${own.id}/checks/${own.checkId}`)).body.data.id).toBe(own.checkId);
+      for (const path of [`/api/monitors/${foreign.id}`, `/api/monitors/${foreign.id}/checks`,
+        `/api/monitors/${foreign.id}/checks/${foreign.checkId}`, `/api/monitors/${own.id}/checks/${foreign.checkId}`,
+        `/api/monitors/${missing}`, `/api/monitors/${missing}/checks`, `/api/monitors/${own.id}/checks/${missing}`]) {
+        const denied = await browserApi(page, path);
+        expect(denied.status).toBe(404);
+        expect(denied.body).toEqual(notFound);
+      }
+    }
+    const originalA = (await browserApi(alice, `/api/monitors/${a.id}`)).body;
+    const originalHistory = (await browserApi(alice, `/api/monitors/${a.id}/checks`)).body;
+    for (const [method, path, body] of [
+      ['PUT', `/api/monitors/${a.id}`, { name: 'Forbidden edit', url: 'http://127.0.0.1:4321/ok', interval: 90, enabled: true }],
+      ['PUT', `/api/monitors/${a.id}`, { name: 'Monitor A', url: 'http://127.0.0.1:4321/ok', interval: 60, enabled: false }],
+      ['PUT', `/api/monitors/${a.id}`, { name: 'Monitor A', url: 'http://127.0.0.1:4321/ok', interval: 60, enabled: true }],
+      ['DELETE', `/api/monitors/${a.id}`, undefined],
+      ['POST', `/api/monitors/${a.id}/checks`, undefined],
+    ] as const) {
+      const denied = await browserApi(bob, path, method, body);
+      expect(denied.status).toBe(404);
+      expect(denied.body).toEqual(notFound);
+      expect((await browserApi(alice, `/api/monitors/${a.id}`)).body).toEqual(originalA);
+      expect((await browserApi(alice, `/api/monitors/${a.id}/checks`)).body).toEqual(originalHistory);
+    }
+    const noCsrf = await browserApi(alice, `/api/monitors/${a.id}/checks`, 'POST', undefined, false);
+    expect(noCsrf.status).toBe(403);
+    expect(noCsrf.body.error.code).toBe('FORBIDDEN');
+    expect((await browserApi(alice, `/api/monitors/${a.id}/checks`)).body).toEqual(originalHistory);
+
+    // A real, isolated document at the unchanged frozen Origin; no browser security override.
+    const attacker = await aliceContext.newPage();
+    await attacker.goto('http://127.0.0.1:4999/attack');
+    const attackerDocumentOrigin = await attacker.evaluate(() => location.origin);
+    expect(attackerDocumentOrigin).toBe('http://127.0.0.1:4999');
+    const attackUrl = `${ui}/api/monitors/${a.id}/checks`;
+    // Observe a response OR a transport failure. Both promises settle before cleanup,
+    // so a rejected fetch cannot leave a response waiter that masks its original failure.
+    const observation = new Promise<{ response?: Response; failure?: string }>(resolve => {
+      const matches = (request: Request) => request.url() === attackUrl && request.method() === 'POST';
+      const finish = (result: { response?: Response; failure?: string }) => {
+        attacker.off('response', response);
+        attacker.off('requestfailed', failed);
+        attacker.off('close', closed);
+        resolve(result);
+      };
+      const response = (response: Response) => { if (matches(response.request())) finish({ response }); };
+      const failed = (request: Request) => { if (matches(request)) finish({ failure: request.failure()?.errorText ?? 'transport failure' }); };
+      const closed = () => finish({ failure: 'attacker page closed before network outcome' });
+      attacker.on('response', response);
+      attacker.on('requestfailed', failed);
+      attacker.on('close', closed);
+    });
+    const [network, fetchResult] = await Promise.all([observation, attacker.evaluate(async url => {
+      try { return (await fetch(url, { method: 'POST', mode: 'no-cors', credentials: 'include' })).type; }
+      catch { return 'blocked'; }
+    }, attackUrl)]);
+    expect(network.failure, 'The fixed foreign-origin POST must reach an observable HTTP response').toBeUndefined();
+    expect(network.response?.status()).toBe(403);
+    // Playwright may fall back to provisional request headers for an opaque response.
+    // Record absent metadata; the real document Origin and HTTP denial remain required.
+    const foreignRequestOriginMetadata = await network.response?.request().headerValue('origin').catch(() => null) ?? null;
+    expect(network.response?.headers()['access-control-allow-origin']).toBeUndefined();
+    expect(network.response?.headers()['access-control-allow-credentials']).toBeUndefined();
+    // A no-cors JSON error may additionally be blocked by Chromium's opaque-response protection.
+    expect(['opaque', 'blocked']).toContain(fetchResult);
+    const readBlocked = await attacker.evaluate(async url => {
+      try { await fetch(url, { credentials: 'include' }); return false; }
+      catch { return true; }
+    }, `${ui}/api/monitors`);
+    expect(readBlocked).toBe(true);
+    expect((await browserApi(alice, `/api/monitors/${a.id}/checks`)).body).toEqual(originalHistory);
+    await attacker.close();
+
+    let article = alice.getByRole('article', { name: 'Monitor A', exact: true });
+    await article.getByRole('button', { name: 'Edit', exact: true }).click();
+    await article.getByLabel('Edit name', { exact: true }).fill('Monitor A edited');
+    await article.getByLabel('Edit interval (seconds)', { exact: true }).fill('90');
+    await article.getByRole('button', { name: 'Save changes', exact: true }).click();
+    article = alice.getByRole('article', { name: 'Monitor A edited', exact: true });
+    await expect(article.getByText('90s interval · Enabled', { exact: true })).toBeVisible();
+    await article.getByRole('button', { name: 'Pause', exact: true }).click();
+    await expect(article.getByText('90s interval · Paused', { exact: true })).toBeVisible();
+    await article.getByRole('button', { name: 'Activate', exact: true }).click();
+    await expect(article.getByText('90s interval · Enabled', { exact: true })).toBeVisible();
+    await article.getByRole('button', { name: 'Run check', exact: true }).click();
+    await article.getByRole('button', { name: 'Show history', exact: true }).click();
+    await expect(article.getByRole('table', { name: 'Check history' }).getByRole('row')).toHaveCount(3);
+    await article.getByRole('button', { name: 'Delete', exact: true }).click();
+    await expect(article).toHaveCount(0);
+    expect((await browserApi(alice, `/api/monitors/${a.id}/checks/${a.checkId}`)).status).toBe(404);
+    expect((await browserApi(bob, `/api/monitors/${b.id}/checks`)).body.data).toHaveLength(1);
+    await bob.getByRole('article', { name: 'Monitor B', exact: true }).getByRole('button', { name: 'Delete', exact: true }).click();
+    await expect(bob.getByRole('article')).toHaveCount(0);
+    for (const page of [alice, bob]) {
+      await page.getByRole('button', { name: 'Sign out', exact: true }).click();
+      await expect(page).toHaveURL('/login');
+      expect((await safeRequest(() => page.request.get('/api/monitors'))).status()).toBe(401);
+    }
+    await Promise.all(headerReads);
+    // The intentionally denied missing-CSRF check is the only token-less same-origin mutation.
+    expect(observed.filter(request => !request.csrfPresent)).toEqual([
+      expect.objectContaining({ path: `/api/monitors/${a.id}/checks`, method: 'POST' }),
+    ]);
+    expect(observed.filter(request => request.path === '/api/session/login')).toHaveLength(2);
+    expect(observed.filter(request => request.path === '/api/session/logout')).toHaveLength(2);
+    completed = {
+      browserContexts: 2, fixedDataset: 'Alice A /ok60 and Bob B /fail120, one initial CheckRun each',
+      ownerReadsAndLifecycle: true, foreignReadsAndWrites404: true, deniedWritesKeepRowsAndHistory: true,
+      missingCsrf403: true, realForeignOriginWrite403: true, credentialedCrossOriginReadBlocked: true,
+      foreignOriginFetchResult: fetchResult,
+      attackerDocumentOrigin, foreignRequestOriginMetadata,
+      trustedBrowserDocumentOriginsVerified: true,
+      requestOriginMetadataUnavailable: observed.filter(request => request.origin === null).length,
+      requestOriginMetadataReadFailed: headerReadFailed,
+      logout401: true, observedBrowserMutationEvidence: observed,
+      attackerDocument: 'owned HTTP fixture http://127.0.0.1:4999/attack; occupied ports refused',
+      credentialsRecorded: false, traces: false, screenshots: false,
+    };
+  } finally {
+    const cleanup = await Promise.allSettled([
+      aliceContext?.close(), bobContext?.close(),
+      new Promise<void>((resolve, reject) => attackerServer.close(error => error ? reject(error) : resolve())),
+    ]);
+    expect(cleanup.every(result => result.status === 'fulfilled')).toBe(true);
+    expect(attackerServer.listening).toBe(false);
+    mkdirSync('output/e05', { recursive: true });
+    writeFileSync('output/e05/browser-fixture-cleanup.json', `${JSON.stringify({
+      port: 4999, serverClosedAndAwaited: true, browserContextsClosedAndAwaited: true,
+    }, null, 2)}\n`);
+    if (completed) writeFileSync('output/e05/browser-isolation.json', `${JSON.stringify({ ...completed,
+      attackerPort4999ClosedAndAwaited: true, browserContextsClosedAndAwaited: true,
+    }, null, 2)}\n`);
+  }
+});


