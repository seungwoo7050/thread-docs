## `feat(audit): weave the fail-closed action lifecycle`

diff --git a/src/main/java/com/sportsbook/admin/audit/AuditAspect.java b/src/main/java/com/sportsbook/admin/audit/AuditAspect.java
new file mode 100644
index 0000000..4beb8ce
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/audit/AuditAspect.java
@@ -0,0 +1,85 @@
+package com.sportsbook.admin.audit;
+
+import com.sportsbook.admin.context.AdminContext;
+import java.lang.reflect.Method;
+import java.util.Arrays;
+import org.aspectj.lang.ProceedingJoinPoint;
+import org.aspectj.lang.annotation.Around;
+import org.aspectj.lang.annotation.Aspect;
+import org.aspectj.lang.reflect.MethodSignature;
+import org.springframework.context.expression.MethodBasedEvaluationContext;
+import org.springframework.core.DefaultParameterNameDiscoverer;
+import org.springframework.core.annotation.AnnotatedElementUtils;
+import org.springframework.expression.ExpressionParser;
+import org.springframework.expression.spel.standard.SpelExpressionParser;
+import org.springframework.stereotype.Component;
+
+@Aspect
+@Component
+public final class AuditAspect {
+
+  private final AuditService audits;
+  private final AuditOutcomeClassifier outcomes = new AuditOutcomeClassifier();
+  private final ExpressionParser expressions = new SpelExpressionParser();
+  private final DefaultParameterNameDiscoverer parameterNames =
+      new DefaultParameterNameDiscoverer();
+
+  public AuditAspect(AuditService audits) {
+    this.audits = audits;
+  }
+
+  @Around("@annotation(com.sportsbook.admin.audit.Audited)")
+  public Object record(ProceedingJoinPoint invocation) throws Throwable {
+    Method method = ((MethodSignature) invocation.getSignature()).getMethod();
+    Audited audited = AnnotatedElementUtils.findMergedAnnotation(method, Audited.class);
+    if (audited == null) {
+      throw new IllegalStateException("Audit metadata is required");
+    }
+    AdminContext context = context(invocation.getArgs());
+    String target = evaluate(audited.target(), method, invocation.getTarget(), invocation.getArgs());
+    String reason = evaluate(audited.reason(), method, invocation.getTarget(), invocation.getArgs());
+    audits.begin(context, audited.action().name(), target, reason);
+
+    Object result = null;
+    Throwable originalFailure = null;
+    try {
+      result = invocation.proceed();
+    } catch (Throwable failure) {
+      originalFailure = failure;
+    }
+
+    AuditOutcomeClassifier.AuditDecision decision =
+        originalFailure == null ? outcomes.result(result) : outcomes.failure(originalFailure);
+    try {
+      audits.complete(context.actionId(), decision.outcome(), decision.httpStatus());
+    } catch (AuditPersistenceException finalizationFailure) {
+      if (originalFailure != null) {
+        finalizationFailure.addSuppressed(originalFailure);
+      }
+      throw finalizationFailure;
+    }
+    if (originalFailure != null) {
+      throw originalFailure;
+    }
+    return result;
+  }
+
+  private AdminContext context(Object[] arguments) {
+    return Arrays.stream(arguments)
+        .filter(AdminContext.class::isInstance)
+        .map(AdminContext.class::cast)
+        .findFirst()
+        .orElseThrow(() -> new IllegalStateException("Audited method requires AdminContext"));
+  }
+
+  private String evaluate(
+      String expression, Method method, Object target, Object[] arguments) {
+    if (expression.isBlank()) {
+      return null;
+    }
+    MethodBasedEvaluationContext evaluation =
+        new MethodBasedEvaluationContext(target, method, arguments, parameterNames);
+    Object value = expressions.parseExpression(expression).getValue(evaluation);
+    return value == null ? null : value.toString();
+  }
+}


## `feat(error): fail closed when audit finalization fails`

diff --git a/src/main/java/com/sportsbook/admin/api/AdminExceptionHandler.java b/src/main/java/com/sportsbook/admin/api/AdminExceptionHandler.java
index 8b8f200..cef41df 100644
--- a/src/main/java/com/sportsbook/admin/api/AdminExceptionHandler.java
+++ b/src/main/java/com/sportsbook/admin/api/AdminExceptionHandler.java
@@ -1,8 +1,10 @@
 package com.sportsbook.admin.api;
 
+import com.sportsbook.admin.audit.AuditPersistenceException;
 import com.sportsbook.admin.client.DownstreamContractException;
 import com.sportsbook.admin.client.DownstreamStatusException;
 import com.sportsbook.admin.client.DownstreamUnavailableException;
+import com.sportsbook.admin.context.AdminContextArgumentResolver;
 import com.sportsbook.protocol.error.ProblemDetail;
 import jakarta.servlet.http.HttpServletRequest;
 import java.net.URI;
@@ -17,6 +19,8 @@ import org.springframework.web.bind.annotation.RestControllerAdvice;
 @RestControllerAdvice
 public final class AdminExceptionHandler {
 
+  private static final URI AUDIT_UNAVAILABLE =
+      URI.create("https://sportsbook/errors/audit-unavailable");
   private static final URI BAD_GATEWAY = URI.create("https://sportsbook/errors/bad-gateway");
   private static final URI GATEWAY_TIMEOUT =
       URI.create("https://sportsbook/errors/gateway-timeout");
@@ -64,6 +68,34 @@ public final class AdminExceptionHandler {
         request);
   }
 
+  @ExceptionHandler(AuditPersistenceException.class)
+  ResponseEntity<ProblemDetail> auditPersistence(
+      AuditPersistenceException failure, HttpServletRequest request) {
+    String code =
+        failure.phase() == AuditPersistenceException.Phase.BEGIN
+            ? "AUDIT_UNAVAILABLE"
+            : "AUDIT_FINALIZATION_FAILED";
+    String detail =
+        failure.phase() == AuditPersistenceException.Phase.BEGIN
+            ? "The audit trail could not be started"
+            : "The audit trail could not be finalized";
+    ProblemDetail body =
+        new ProblemDetail(
+            AUDIT_UNAVAILABLE,
+            "Service Unavailable",
+            HttpStatus.SERVICE_UNAVAILABLE.value(),
+            code,
+            detail,
+            URI.create(request.getRequestURI()),
+            MDC.get("traceId"));
+    HttpHeaders headers = new HttpHeaders();
+    headers.setContentType(MediaType.APPLICATION_PROBLEM_JSON);
+    headers.setCacheControl("no-store");
+    headers.set(
+        AdminContextArgumentResolver.ACTION_ID_HEADER, failure.actionId().toString());
+    return new ResponseEntity<>(body, headers, HttpStatus.SERVICE_UNAVAILABLE);
+  }
+
   private static ResponseEntity<ProblemDetail> problem(
       HttpStatus status,
       URI type,


## `test(audit): provide PostgreSQL WireMock environment`

diff --git a/src/test/java/com/sportsbook/admin/audit/AuditHttpTestEnvironment.java b/src/test/java/com/sportsbook/admin/audit/AuditHttpTestEnvironment.java
new file mode 100644
index 0000000..f91dd2b
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/audit/AuditHttpTestEnvironment.java
@@ -0,0 +1,41 @@
+package com.sportsbook.admin.audit;
+
+import com.github.tomakehurst.wiremock.WireMockServer;
+import com.github.tomakehurst.wiremock.core.WireMockConfiguration;
+import com.sportsbook.admin.security.TestJwtKeys;
+import org.springframework.test.context.DynamicPropertyRegistry;
+import org.testcontainers.containers.PostgreSQLContainer;
+
+final class AuditHttpTestEnvironment {
+
+  static final PostgreSQLContainer<?> POSTGRES = new PostgreSQLContainer<>("postgres:16-alpine");
+  static final WireMockServer DOWNSTREAM =
+      new WireMockServer(WireMockConfiguration.options().dynamicPort());
+
+  static {
+    POSTGRES.start();
+    DOWNSTREAM.start();
+  }
+
+  private AuditHttpTestEnvironment() {}
+
+  static void register(DynamicPropertyRegistry registry) {
+    registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
+    registry.add("spring.datasource.username", POSTGRES::getUsername);
+    registry.add("spring.datasource.password", POSTGRES::getPassword);
+    registry.add("admin.security.jwt.public-key", TestJwtKeys::publicKeyPem);
+    registry.add("admin.downstream.wallet-base-url", AuditHttpTestEnvironment::baseUrl);
+    registry.add("admin.downstream.risk-base-url", AuditHttpTestEnvironment::baseUrl);
+    registry.add("admin.downstream.odds-feed-base-url", AuditHttpTestEnvironment::baseUrl);
+    registry.add("admin.downstream.settlement-base-url", AuditHttpTestEnvironment::baseUrl);
+  }
+
+  static void stop() {
+    DOWNSTREAM.stop();
+    POSTGRES.stop();
+  }
+
+  private static String baseUrl() {
+    return "http://127.0.0.1:" + DOWNSTREAM.port();
+  }
+}


## `test(audit): provide HTTP audit fixture`

diff --git a/src/test/java/com/sportsbook/admin/audit/AuditHttpIntegrationTest.java b/src/test/java/com/sportsbook/admin/audit/AuditHttpIntegrationTest.java
new file mode 100644
index 0000000..dfd647b
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/audit/AuditHttpIntegrationTest.java
@@ -0,0 +1,82 @@
+package com.sportsbook.admin.audit;
+
+import static org.springframework.http.HttpHeaders.AUTHORIZATION;
+import static org.springframework.http.MediaType.APPLICATION_JSON;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
+
+import com.github.tomakehurst.wiremock.client.ResponseDefinitionBuilder;
+import com.github.tomakehurst.wiremock.client.WireMock;
+import com.sportsbook.admin.security.TestJwtKeys;
+import java.util.UUID;
+import org.junit.jupiter.api.AfterAll;
+import org.junit.jupiter.api.BeforeEach;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.autoconfigure.actuate.observability.AutoConfigureObservability;
+import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.boot.test.mock.mockito.MockBean;
+import org.springframework.test.context.DynamicPropertyRegistry;
+import org.springframework.test.context.DynamicPropertySource;
+import org.springframework.test.web.servlet.ResultActions;
+import org.springframework.test.web.servlet.MockMvc;
+
+@SpringBootTest(
+    properties = {
+      "spring.kafka.bootstrap-servers=127.0.0.1:1",
+      "admin.audit.stale-scan-interval=PT1H",
+      "admin.downstream.read-timeout=250ms",
+      "admin.downstream.credentials.wallet-api-key=wallet-admin-http-key-000000000001",
+      "admin.downstream.credentials.risk-api-key=risk-admin-http-key-00000000000002",
+      "admin.downstream.credentials.odds-feed-api-key=odds-admin-http-key-00000000000003",
+      "admin.downstream.credentials.settlement-api-key=settlement-admin-http-key-000000004"
+    })
+@AutoConfigureMockMvc
+@AutoConfigureObservability
+class AuditHttpIntegrationTest {
+
+  private static final UUID EVENT_ID = UUID.fromString("3f9b0ba6-558f-4df1-a31c-835f3cd57f9d");
+  private static final UUID MARKET_ID = UUID.fromString("9f50e81c-327a-461e-91cd-0596a0d22865");
+  private static final String PATH =
+      "/internal/v1/events/" + EVENT_ID + "/markets/" + MARKET_ID + "/suspend";
+
+  @Autowired private MockMvc mvc;
+  @Autowired private AuditLogRepository auditLogs;
+  @MockBean private AdminActionPublisher publisher;
+
+  @DynamicPropertySource
+  static void dependencies(DynamicPropertyRegistry registry) {
+    AuditHttpTestEnvironment.register(registry);
+  }
+
+  @BeforeEach
+  void reset() {
+    auditLogs.deleteAll();
+    AuditHttpTestEnvironment.DOWNSTREAM.resetAll();
+  }
+
+  @AfterAll
+  static void stopDependencies() {
+    AuditHttpTestEnvironment.stop();
+  }
+
+  private ResultActions suspend() throws Exception {
+    return mvc.perform(
+        post("/admin/v1/events/{eventId}/markets/{marketId}/suspend", EVENT_ID, MARKET_ID)
+            .header(AUTHORIZATION, TestJwtKeys.bearer("operator-17", "TRADER"))
+            .header("Idempotency-Key", "e292ac36-1c66-4c17-9027-d6aa63df1ae9")
+            .contentType(APPLICATION_JSON)
+            .content("{\"reason\":\"feed investigation\"}"));
+  }
+
+  private void stubOdds(int status, String body, int delayMillis) {
+    ResponseDefinitionBuilder response = WireMock.aResponse().withStatus(status);
+    if (body != null) {
+      response.withHeader("Content-Type", "application/problem+json").withBody(body);
+    }
+    if (delayMillis > 0) {
+      response.withFixedDelay(delayMillis);
+    }
+    AuditHttpTestEnvironment.DOWNSTREAM.stubFor(
+        WireMock.post(WireMock.urlEqualTo(PATH)).willReturn(response));
+  }
+}


## `test(audit): expose STARTED before success`

diff --git a/src/test/java/com/sportsbook/admin/audit/AuditHttpIntegrationTest.java b/src/test/java/com/sportsbook/admin/audit/AuditHttpIntegrationTest.java
index dfd647b..83a7643 100644
--- a/src/test/java/com/sportsbook/admin/audit/AuditHttpIntegrationTest.java
+++ b/src/test/java/com/sportsbook/admin/audit/AuditHttpIntegrationTest.java
@@ -1,15 +1,23 @@
 package com.sportsbook.admin.audit;
 
+import static org.assertj.core.api.Assertions.assertThat;
 import static org.springframework.http.HttpHeaders.AUTHORIZATION;
 import static org.springframework.http.MediaType.APPLICATION_JSON;
 import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
 
 import com.github.tomakehurst.wiremock.client.ResponseDefinitionBuilder;
 import com.github.tomakehurst.wiremock.client.WireMock;
 import com.sportsbook.admin.security.TestJwtKeys;
+import java.time.Duration;
+import java.time.Instant;
 import java.util.UUID;
+import java.util.concurrent.CompletableFuture;
+import java.util.concurrent.CompletionException;
+import java.util.concurrent.TimeUnit;
 import org.junit.jupiter.api.AfterAll;
 import org.junit.jupiter.api.BeforeEach;
+import org.junit.jupiter.api.Test;
 import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.boot.test.autoconfigure.actuate.observability.AutoConfigureObservability;
 import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
@@ -17,14 +25,14 @@ import org.springframework.boot.test.context.SpringBootTest;
 import org.springframework.boot.test.mock.mockito.MockBean;
 import org.springframework.test.context.DynamicPropertyRegistry;
 import org.springframework.test.context.DynamicPropertySource;
-import org.springframework.test.web.servlet.ResultActions;
 import org.springframework.test.web.servlet.MockMvc;
+import org.springframework.test.web.servlet.ResultActions;
 
 @SpringBootTest(
     properties = {
       "spring.kafka.bootstrap-servers=127.0.0.1:1",
       "admin.audit.stale-scan-interval=PT1H",
-      "admin.downstream.read-timeout=250ms",
+      "admin.downstream.read-timeout=750ms",
       "admin.downstream.credentials.wallet-api-key=wallet-admin-http-key-000000000001",
       "admin.downstream.credentials.risk-api-key=risk-admin-http-key-00000000000002",
       "admin.downstream.credentials.odds-feed-api-key=odds-admin-http-key-00000000000003",
@@ -59,6 +67,55 @@ class AuditHttpIntegrationTest {
     AuditHttpTestEnvironment.stop();
   }
 
+  @Test
+  void exposesStartedBeforeCompletingAnExactSuccess() throws Exception {
+    stubOdds(202, null, 400);
+    CompletableFuture<ResultActions> request =
+        CompletableFuture.supplyAsync(
+            () -> {
+              try {
+                return suspend();
+              } catch (Exception failure) {
+                throw new CompletionException(failure);
+              }
+            });
+
+    AuditLogEntity started = awaitStarted();
+    assertThat(started.getActorId()).isEqualTo("operator-17");
+    assertThat(started.getActorRole()).isEqualTo(com.sportsbook.admin.security.AdminRole.TRADER);
+    assertThat(started.getAction()).isEqualTo("MARKET_SUSPEND");
+    assertThat(started.getTarget()).isEqualTo(EVENT_ID + "/" + MARKET_ID);
+    assertThat(started.getReason()).isEqualTo("feed investigation");
+    assertThat(started.getTraceId()).isNotBlank();
+    assertThat(started.getHttpStatus()).isNull();
+    assertThat(started.getCompletedAt()).isNull();
+
+    ResultActions response = request.get(3, TimeUnit.SECONDS);
+    String actionHeader =
+        response
+            .andExpect(status().isAccepted())
+            .andReturn()
+            .getResponse()
+            .getHeader("X-Admin-Action-Id");
+    assertThat(actionHeader).isEqualTo(started.getActionId().toString());
+    AuditLogEntity terminal = auditLogs.findById(started.getActionId()).orElseThrow();
+    assertThat(terminal.getOutcome()).isEqualTo(AuditOutcome.SUCCESS);
+    assertThat(terminal.getHttpStatus()).isEqualTo(202);
+    assertThat(terminal.getCompletedAt()).isAfterOrEqualTo(terminal.getStartedAt());
+  }
+
+  private AuditLogEntity awaitStarted() throws InterruptedException {
+    Instant deadline = Instant.now().plus(Duration.ofSeconds(2));
+    while (Instant.now().isBefore(deadline)) {
+      var rows = auditLogs.findAll();
+      if (rows.size() == 1 && rows.get(0).getOutcome() == AuditOutcome.STARTED) {
+        return rows.get(0);
+      }
+      Thread.sleep(20);
+    }
+    throw new AssertionError("Audit STARTED row was not externally visible");
+  }
+
   private ResultActions suspend() throws Exception {
     return mvc.perform(
         post("/admin/v1/events/{eventId}/markets/{marketId}/suspend", EVENT_ID, MARKET_ID)


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


