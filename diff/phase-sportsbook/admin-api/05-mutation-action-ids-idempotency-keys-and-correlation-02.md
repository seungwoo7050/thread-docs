## `test(security): initialize mutation identities after JWT`

diff --git a/src/test/java/com/sportsbook/admin/security/SecurityChainTest.java b/src/test/java/com/sportsbook/admin/security/SecurityChainTest.java
index b71bb9e..674ed3b 100644
--- a/src/test/java/com/sportsbook/admin/security/SecurityChainTest.java
+++ b/src/test/java/com/sportsbook/admin/security/SecurityChainTest.java
@@ -1,11 +1,16 @@
 package com.sportsbook.admin.security;
 
+import static org.assertj.core.api.Assertions.assertThat;
 import static org.springframework.http.HttpHeaders.AUTHORIZATION;
 import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
 import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.header;
 import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
 import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
 
+import com.sportsbook.admin.context.AdminContext;
+import com.sportsbook.admin.context.AdminContextArgumentResolver;
 import org.junit.jupiter.api.Test;
 import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.boot.test.autoconfigure.actuate.observability.AutoConfigureObservability;
@@ -17,6 +22,7 @@ import org.springframework.test.context.DynamicPropertyRegistry;
 import org.springframework.test.context.DynamicPropertySource;
 import org.springframework.test.web.servlet.MockMvc;
 import org.springframework.web.bind.annotation.GetMapping;
+import org.springframework.web.bind.annotation.PostMapping;
 import org.springframework.web.bind.annotation.RestController;
 
 @SpringBootTest(
@@ -39,6 +45,7 @@ import org.springframework.web.bind.annotation.RestController;
 class SecurityChainTest {
 
   private static final String ADMIN_ONLY = "/admin/v1/test/admin-only";
+  private static final String MUTATION = "/admin/v1/test/mutation";
 
   @DynamicPropertySource
   static void jwtKey(DynamicPropertyRegistry registry) {
@@ -85,6 +92,19 @@ class SecurityChainTest {
         .andExpect(status().isForbidden());
   }
 
+  @Test
+  void initializesOneMutationIdentityAfterJwtAuthentication() throws Exception {
+    var response =
+        mvc.perform(post(MUTATION).header(AUTHORIZATION, TestJwtKeys.bearer("admin-1", "ADMIN")))
+            .andExpect(status().isOk())
+            .andExpect(header().exists(AdminContextArgumentResolver.ACTION_ID_HEADER))
+            .andReturn()
+            .getResponse();
+
+    assertThat(response.getContentAsString())
+        .isEqualTo(response.getHeader(AdminContextArgumentResolver.ACTION_ID_HEADER));
+  }
+
   @RestController
   static class RoleProbeController {
 
@@ -93,5 +113,11 @@ class SecurityChainTest {
     String adminOnly() {
       return "ok";
     }
+
+    @PostMapping(MUTATION)
+    @PreAuthorize("hasRole('ADMIN')")
+    String mutate(AdminContext context) {
+      return context.actionId().toString();
+    }
   }
 }


## `feat(api): require one idempotency header`

diff --git a/src/main/java/com/sportsbook/admin/api/AdminRequestException.java b/src/main/java/com/sportsbook/admin/api/AdminRequestException.java
new file mode 100644
index 0000000..b318c33
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/api/AdminRequestException.java
@@ -0,0 +1,12 @@
+package com.sportsbook.admin.api;
+
+public final class AdminRequestException extends IllegalArgumentException {
+
+  public AdminRequestException(String message) {
+    super(message);
+  }
+
+  public AdminRequestException(String message, Throwable cause) {
+    super(message, cause);
+  }
+}
diff --git a/src/main/java/com/sportsbook/admin/api/AdminRequestHeaders.java b/src/main/java/com/sportsbook/admin/api/AdminRequestHeaders.java
new file mode 100644
index 0000000..141e26f
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/api/AdminRequestHeaders.java
@@ -0,0 +1,43 @@
+package com.sportsbook.admin.api;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import jakarta.servlet.http.HttpServletRequest;
+import java.util.Enumeration;
+import java.util.UUID;
+
+public final class AdminRequestHeaders {
+
+  public static final String IDEMPOTENCY_KEY = "Idempotency-Key";
+
+  private AdminRequestHeaders() {}
+
+  public static IdempotencyKey requireIdempotencyKey(HttpServletRequest request) {
+    String value = requireSingleValue(request);
+    try {
+      return IdempotencyKey.of(value);
+    } catch (IllegalArgumentException invalid) {
+      throw new AdminRequestException("Idempotency-Key is invalid", invalid);
+    }
+  }
+
+  public static UUID requireUuidIdempotencyKey(HttpServletRequest request) {
+    String value = requireSingleValue(request);
+    try {
+      return UUID.fromString(value);
+    } catch (IllegalArgumentException invalid) {
+      throw new AdminRequestException("Idempotency-Key must be a UUID", invalid);
+    }
+  }
+
+  private static String requireSingleValue(HttpServletRequest request) {
+    Enumeration<String> values = request.getHeaders(IDEMPOTENCY_KEY);
+    if (values == null || !values.hasMoreElements()) {
+      throw new AdminRequestException("Exactly one Idempotency-Key header is required");
+    }
+    String value = values.nextElement();
+    if (values.hasMoreElements()) {
+      throw new AdminRequestException("Exactly one Idempotency-Key header is required");
+    }
+    return value;
+  }
+}


## `test(api): reject ambiguous idempotency keys`

diff --git a/src/test/java/com/sportsbook/admin/api/AdminRequestHeadersTest.java b/src/test/java/com/sportsbook/admin/api/AdminRequestHeadersTest.java
new file mode 100644
index 0000000..5cf2956
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/api/AdminRequestHeadersTest.java
@@ -0,0 +1,50 @@
+package com.sportsbook.admin.api;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.mock.web.MockHttpServletRequest;
+
+class AdminRequestHeadersTest {
+
+  @Test
+  void preservesOneGeneralKeyWithoutNormalization() {
+    MockHttpServletRequest request = new MockHttpServletRequest();
+    request.addHeader(AdminRequestHeaders.IDEMPOTENCY_KEY, " retry key 01 ");
+
+    assertThat(AdminRequestHeaders.requireIdempotencyKey(request).value())
+        .isEqualTo(" retry key 01 ");
+  }
+
+  @Test
+  void parsesOneUuidKey() {
+    MockHttpServletRequest request = new MockHttpServletRequest();
+    request.addHeader(AdminRequestHeaders.IDEMPOTENCY_KEY, "018f0000-0000-7000-8000-000000000111");
+
+    assertThat(AdminRequestHeaders.requireUuidIdempotencyKey(request))
+        .isEqualTo(UUID.fromString("018f0000-0000-7000-8000-000000000111"));
+  }
+
+  @Test
+  void rejectsMissingDuplicateAndInvalidKeys() {
+    MockHttpServletRequest missing = new MockHttpServletRequest();
+    assertThatThrownBy(() -> AdminRequestHeaders.requireIdempotencyKey(missing))
+        .isInstanceOf(AdminRequestException.class)
+        .hasMessageContaining("Exactly one");
+
+    MockHttpServletRequest duplicate = new MockHttpServletRequest();
+    duplicate.addHeader(AdminRequestHeaders.IDEMPOTENCY_KEY, "first");
+    duplicate.addHeader(AdminRequestHeaders.IDEMPOTENCY_KEY, "second");
+    assertThatThrownBy(() -> AdminRequestHeaders.requireIdempotencyKey(duplicate))
+        .isInstanceOf(AdminRequestException.class)
+        .hasMessageContaining("Exactly one");
+
+    MockHttpServletRequest invalidUuid = new MockHttpServletRequest();
+    invalidUuid.addHeader(AdminRequestHeaders.IDEMPOTENCY_KEY, "not-a-uuid");
+    assertThatThrownBy(() -> AdminRequestHeaders.requireUuidIdempotencyKey(invalidUuid))
+        .isInstanceOf(AdminRequestException.class)
+        .hasMessageContaining("UUID");
+  }
+}
diff --git a/src/test/java/com/sportsbook/admin/audit/AuditOutcomeClassifierTest.java b/src/test/java/com/sportsbook/admin/audit/AuditOutcomeClassifierTest.java
index 5ca0799..a054406 100644
--- a/src/test/java/com/sportsbook/admin/audit/AuditOutcomeClassifierTest.java
+++ b/src/test/java/com/sportsbook/admin/audit/AuditOutcomeClassifierTest.java
@@ -3,6 +3,7 @@ package com.sportsbook.admin.audit;
 import static org.assertj.core.api.Assertions.assertThat;
 import static org.assertj.core.api.Assertions.catchThrowable;
 
+import com.sportsbook.admin.api.AdminRequestException;
 import com.sportsbook.admin.client.DownstreamContract;
 import com.sportsbook.admin.client.DownstreamFailureMapper;
 import java.net.SocketTimeoutException;
@@ -37,30 +38,27 @@ class AuditOutcomeClassifierTest {
 
   @Test
   void treatsOnlyExplicit4xxAndLocalDenialsAsFailed() {
-    Throwable downstream4xx =
-        mapped(new HttpClientErrorException(HttpStatus.CONFLICT));
+    Throwable downstream4xx = mapped(new HttpClientErrorException(HttpStatus.CONFLICT));
 
     assertThat(classifier.failure(downstream4xx).outcome()).isEqualTo(AuditOutcome.FAILED);
     assertThat(classifier.failure(new AccessDeniedException("denied")).outcome())
         .isEqualTo(AuditOutcome.FAILED);
     assertThat(classifier.failure(new IllegalArgumentException("invalid")).httpStatus())
         .isEqualTo(400);
+    assertThat(classifier.failure(new AdminRequestException("invalid header")))
+        .isEqualTo(new AuditOutcomeClassifier.AuditDecision(AuditOutcome.FAILED, 400));
   }
 
   @Test
   void treatsAmbiguousAndMalformedMutationOutcomesAsUnknown() {
-    Throwable serverError =
-        mapped(new HttpServerErrorException(HttpStatus.SERVICE_UNAVAILABLE));
+    Throwable serverError = mapped(new HttpServerErrorException(HttpStatus.SERVICE_UNAVAILABLE));
     Throwable timeout =
         mapped(new ResourceAccessException("timeout", new SocketTimeoutException()));
     Throwable malformed =
         catchThrowable(
             () ->
                 DownstreamContract.requireBody(
-                    ResponseEntity.ok().build(),
-                    HttpStatus.OK,
-                    value -> true,
-                    "missing receipt"));
+                    ResponseEntity.ok().build(), HttpStatus.OK, value -> true, "missing receipt"));
 
     assertThat(classifier.failure(serverError).outcome()).isEqualTo(AuditOutcome.UNKNOWN);
     assertThat(classifier.failure(timeout).httpStatus()).isEqualTo(504);


## `test(api): reject invalid mutation keys`

diff --git a/src/test/java/com/sportsbook/admin/api/SettlementIdempotencyRequestTest.java b/src/test/java/com/sportsbook/admin/api/SettlementIdempotencyRequestTest.java
new file mode 100644
index 0000000..4d3b3cd
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/api/SettlementIdempotencyRequestTest.java
@@ -0,0 +1,77 @@
+package com.sportsbook.admin.api;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verifyNoInteractions;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.header;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
+
+import com.sportsbook.admin.client.SettlementClient;
+import com.sportsbook.admin.context.AdminContextArgumentResolver;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.security.core.authority.SimpleGrantedAuthority;
+import org.springframework.security.oauth2.jwt.Jwt;
+import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
+import org.springframework.test.web.servlet.MockMvc;
+import org.springframework.test.web.servlet.setup.MockMvcBuilders;
+
+class SettlementIdempotencyRequestTest {
+
+  private static final UUID REVISION = UUID.fromString("018f0000-0000-7000-8000-000000000180");
+
+  @Test
+  void rendersMissingDuplicateAndInvalidUuidKeysAsValidationProblems() throws Exception {
+    SettlementClient settlements = mock(SettlementClient.class);
+    MockMvc mvc = mvc(settlements);
+
+    String actionId =
+        mvc.perform(post(path()).principal(authentication()))
+            .andExpect(status().isBadRequest())
+            .andExpect(jsonPath("$.errorCode").value("VALIDATION_FAILED"))
+            .andExpect(header().exists(AdminContextArgumentResolver.ACTION_ID_HEADER))
+            .andReturn()
+            .getResponse()
+            .getHeader(AdminContextArgumentResolver.ACTION_ID_HEADER);
+    assertThat(UUID.fromString(actionId).version()).isEqualTo(7);
+
+    mvc.perform(
+            post(path())
+                .principal(authentication())
+                .header(AdminRequestHeaders.IDEMPOTENCY_KEY, "first", "second"))
+        .andExpect(status().isBadRequest())
+        .andExpect(jsonPath("$.errorCode").value("VALIDATION_FAILED"));
+    mvc.perform(
+            post(path())
+                .principal(authentication())
+                .header(AdminRequestHeaders.IDEMPOTENCY_KEY, "not-a-uuid"))
+        .andExpect(status().isBadRequest())
+        .andExpect(jsonPath("$.errorCode").value("VALIDATION_FAILED"));
+    verifyNoInteractions(settlements);
+  }
+
+  private static MockMvc mvc(SettlementClient settlements) {
+    return MockMvcBuilders.standaloneSetup(new SettlementRevisionCommandController(settlements))
+        .setCustomArgumentResolvers(new AdminContextArgumentResolver())
+        .setControllerAdvice(new AdminExceptionHandler())
+        .build();
+  }
+
+  private static String path() {
+    return "/admin/v1/settlements/revisions/" + REVISION + "/retry";
+  }
+
+  private static JwtAuthenticationToken authentication() {
+    Jwt jwt =
+        Jwt.withTokenValue("token")
+            .header("alg", "RS256")
+            .subject("operator-1")
+            .claim("role", "TRADER")
+            .build();
+    return new JwtAuthenticationToken(
+        jwt, List.of(new SimpleGrantedAuthority("ROLE_TRADER")), "operator-1");
+  }
+}


## `test(security): preserve action IDs on invalid bodies`

diff --git a/src/test/java/com/sportsbook/admin/security/SecurityChainTest.java b/src/test/java/com/sportsbook/admin/security/SecurityChainTest.java
index 49b9972..1b0fa7a 100644
--- a/src/test/java/com/sportsbook/admin/security/SecurityChainTest.java
+++ b/src/test/java/com/sportsbook/admin/security/SecurityChainTest.java
@@ -13,6 +13,7 @@ import com.sportsbook.admin.audit.AuditLogRepository;
 import com.sportsbook.admin.audit.AuditWriteRepository;
 import com.sportsbook.admin.context.AdminContext;
 import com.sportsbook.admin.context.AdminContextArgumentResolver;
+import java.util.UUID;
 import org.junit.jupiter.api.Test;
 import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.boot.test.autoconfigure.actuate.observability.AutoConfigureObservability;
@@ -20,6 +21,7 @@ import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMock
 import org.springframework.boot.test.context.SpringBootTest;
 import org.springframework.boot.test.mock.mockito.MockBean;
 import org.springframework.context.annotation.Import;
+import org.springframework.http.MediaType;
 import org.springframework.security.access.prepost.PreAuthorize;
 import org.springframework.test.context.DynamicPropertyRegistry;
 import org.springframework.test.context.DynamicPropertySource;
@@ -112,6 +114,25 @@ class SecurityChainTest {
         .isEqualTo(response.getHeader(AdminContextArgumentResolver.ACTION_ID_HEADER));
   }
 
+  @Test
+  void assignsAUuid7ActionIdBeforeMutationBodyBindingFails() throws Exception {
+    String actionId =
+        mvc.perform(
+                post("/admin/v1/wallet/018f0000-0000-7000-8000-000000000187/refund")
+                    .header(AUTHORIZATION, TestJwtKeys.bearer("cs-1", "CS"))
+                    .header("Idempotency-Key", "raw refund key")
+                    .contentType(MediaType.APPLICATION_JSON)
+                    .content("{"))
+            .andExpect(status().isBadRequest())
+            .andExpect(jsonPath("$.errorCode").value("VALIDATION_FAILED"))
+            .andExpect(header().exists(AdminContextArgumentResolver.ACTION_ID_HEADER))
+            .andReturn()
+            .getResponse()
+            .getHeader(AdminContextArgumentResolver.ACTION_ID_HEADER);
+
+    assertThat(UUID.fromString(actionId).version()).isEqualTo(7);
+  }
+
   @RestController
   static class RoleProbeController {
 


## `test(audit): correlate downstream action identities`

diff --git a/src/test/java/com/sportsbook/admin/audit/AuditHttpIntegrationTest.java b/src/test/java/com/sportsbook/admin/audit/AuditHttpIntegrationTest.java
index ffee006..1bba81e 100644
--- a/src/test/java/com/sportsbook/admin/audit/AuditHttpIntegrationTest.java
+++ b/src/test/java/com/sportsbook/admin/audit/AuditHttpIntegrationTest.java
@@ -33,7 +33,7 @@ import org.springframework.test.web.servlet.ResultActions;
     properties = {
       "spring.kafka.bootstrap-servers=127.0.0.1:1",
       "admin.audit.stale-scan-interval=PT1H",
-      "admin.downstream.read-timeout=750ms",
+      "admin.downstream.read-timeout=5s",
       "admin.downstream.credentials.wallet-api-key=wallet-admin-http-key-000000000001",
       "admin.downstream.credentials.risk-api-key=risk-admin-http-key-00000000000002",
       "admin.downstream.credentials.odds-feed-api-key=odds-admin-http-key-00000000000003",
@@ -70,7 +70,7 @@ class AuditHttpIntegrationTest {
 
   @Test
   void exposesStartedBeforeCompletingAnExactSuccess() throws Exception {
-    stubOdds(202, null, 400);
+    stubOdds(202, null, 2_000);
     CompletableFuture<ResultActions> request =
         CompletableFuture.supplyAsync(
             () -> {
@@ -82,6 +82,7 @@ class AuditHttpIntegrationTest {
             });
 
     AuditLogEntity started = awaitStarted();
+    assertThat(request.isDone()).isFalse();
     assertThat(started.getActorId()).isEqualTo("operator-17");
     assertThat(started.getActorRole()).isEqualTo(com.sportsbook.admin.security.AdminRole.TRADER);
     assertThat(started.getAction()).isEqualTo("MARKET_SUSPEND");
@@ -99,6 +100,14 @@ class AuditHttpIntegrationTest {
             .getResponse()
             .getHeader("X-Admin-Action-Id");
     assertThat(actionHeader).isEqualTo(started.getActionId().toString());
+    AuditHttpTestEnvironment.DOWNSTREAM.verify(
+        WireMock.postRequestedFor(WireMock.urlEqualTo(PATH))
+            .withHeader("X-Internal-Service", WireMock.equalTo("admin-api"))
+            .withHeader(
+                "X-Internal-Api-Key", WireMock.equalTo("odds-admin-http-key-00000000000003"))
+            .withHeader("Idempotency-Key", WireMock.equalTo("e292ac36-1c66-4c17-9027-d6aa63df1ae9"))
+            .withHeader("X-Admin-Action-Id", WireMock.equalTo(actionHeader))
+            .withRequestBody(WireMock.equalToJson("{\"reason\":\"feed investigation\"}")));
     AuditLogEntity terminal = auditLogs.findById(started.getActionId()).orElseThrow();
     assertThat(terminal.getOutcome()).isEqualTo(AuditOutcome.SUCCESS);
     assertThat(terminal.getHttpStatus()).isEqualTo(202);
@@ -118,8 +127,7 @@ class AuditHttpIntegrationTest {
 
     assertThat(terminal.getOutcome()).isEqualTo(AuditOutcome.FAILED);
     assertThat(terminal.getHttpStatus()).isEqualTo(409);
-    assertThat(response.andReturn().getResponse().getHeader("Cache-Control"))
-        .isEqualTo("no-store");
+    assertThat(response.andReturn().getResponse().getHeader("Cache-Control")).isEqualTo("no-store");
   }
 
   @Test
@@ -137,7 +145,7 @@ class AuditHttpIntegrationTest {
 
   @Test
   void recordsDownstreamTimeoutsAsUnknown() throws Exception {
-    stubOdds(202, null, 1_200);
+    stubOdds(202, null, 6_000);
 
     AuditLogEntity terminal = terminalFrom(suspend().andExpect(status().isGatewayTimeout()));
 
@@ -156,7 +164,7 @@ class AuditHttpIntegrationTest {
   }
 
   private AuditLogEntity awaitStarted() throws InterruptedException {
-    Instant deadline = Instant.now().plus(Duration.ofSeconds(2));
+    Instant deadline = Instant.now().plus(Duration.ofSeconds(4));
     while (Instant.now().isBefore(deadline)) {
       var rows = auditLogs.findAll();
       if (rows.size() == 1 && rows.get(0).getOutcome() == AuditOutcome.STARTED) {
