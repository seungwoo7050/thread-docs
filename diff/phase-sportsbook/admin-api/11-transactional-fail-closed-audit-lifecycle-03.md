## `feat(audit): order audit before method authorization`

diff --git a/src/main/java/com/sportsbook/admin/audit/AuditAspect.java b/src/main/java/com/sportsbook/admin/audit/AuditAspect.java
index 5f7acee..6a9dcd5 100644
--- a/src/main/java/com/sportsbook/admin/audit/AuditAspect.java
+++ b/src/main/java/com/sportsbook/admin/audit/AuditAspect.java
@@ -9,13 +9,16 @@ import org.aspectj.lang.annotation.Aspect;
 import org.aspectj.lang.reflect.MethodSignature;
 import org.springframework.context.expression.MethodBasedEvaluationContext;
 import org.springframework.core.DefaultParameterNameDiscoverer;
+import org.springframework.core.Ordered;
 import org.springframework.core.annotation.AnnotatedElementUtils;
+import org.springframework.core.annotation.Order;
 import org.springframework.expression.ExpressionParser;
 import org.springframework.expression.spel.standard.SpelExpressionParser;
 import org.springframework.stereotype.Component;
 
 @Aspect
 @Component
+@Order(Ordered.HIGHEST_PRECEDENCE)
 public final class AuditAspect {
 
   private final AuditService audits;
@@ -36,8 +39,10 @@ public final class AuditAspect {
       throw new IllegalStateException("Audit metadata is required");
     }
     AdminContext context = context(invocation.getArgs());
-    String target = evaluate(audited.target(), method, invocation.getTarget(), invocation.getArgs());
-    String reason = evaluate(audited.reason(), method, invocation.getTarget(), invocation.getArgs());
+    String target =
+        evaluate(audited.target(), method, invocation.getTarget(), invocation.getArgs());
+    String reason =
+        evaluate(audited.reason(), method, invocation.getTarget(), invocation.getArgs());
     audits.begin(context, audited.action().name(), target, reason);
 
     Object result = null;
@@ -74,8 +79,7 @@ public final class AuditAspect {
         .orElseThrow(() -> new IllegalStateException("Audited method requires AdminContext"));
   }
 
-  private String evaluate(
-      String expression, Method method, Object target, Object[] arguments) {
+  private String evaluate(String expression, Method method, Object target, Object[] arguments) {
     if (expression.isBlank()) {
       return null;
     }


## `test(audit): record method authorization denials`

diff --git a/src/test/java/com/sportsbook/admin/audit/AuditMethodSecurityOrderingTest.java b/src/test/java/com/sportsbook/admin/audit/AuditMethodSecurityOrderingTest.java
new file mode 100644
index 0000000..90d7e37
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/audit/AuditMethodSecurityOrderingTest.java
@@ -0,0 +1,95 @@
+package com.sportsbook.admin.audit;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verify;
+
+import com.sportsbook.admin.context.AdminContext;
+import com.sportsbook.admin.security.AdminRole;
+import java.util.UUID;
+import java.util.concurrent.atomic.AtomicInteger;
+import org.junit.jupiter.api.AfterEach;
+import org.junit.jupiter.api.Test;
+import org.springframework.context.annotation.AnnotationConfigApplicationContext;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.context.annotation.EnableAspectJAutoProxy;
+import org.springframework.security.access.AccessDeniedException;
+import org.springframework.security.access.prepost.PreAuthorize;
+import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
+import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
+import org.springframework.security.core.authority.SimpleGrantedAuthority;
+import org.springframework.security.core.context.SecurityContextHolder;
+
+class AuditMethodSecurityOrderingTest {
+
+  private static final UUID ACTION_ID = UUID.fromString("018f0000-0000-7000-8000-000000000071");
+
+  @AfterEach
+  void clearSecurityContext() {
+    SecurityContextHolder.clearContext();
+  }
+
+  @Test
+  void recordsAStartedAndFailedRowAroundAMethodAuthorizationDenial() {
+    try (AnnotationConfigApplicationContext context =
+        new AnnotationConfigApplicationContext(DeniedAuditConfiguration.class)) {
+      AuditService audits = context.getBean(AuditService.class);
+      SecuredOperations operations = context.getBean(SecuredOperations.class);
+      UsernamePasswordAuthenticationToken reader =
+          UsernamePasswordAuthenticationToken.authenticated(
+              "reader-1", "n/a", java.util.List.of(new SimpleGrantedAuthority("ROLE_READONLY")));
+      SecurityContextHolder.getContext().setAuthentication(reader);
+      AdminContext adminContext =
+          new AdminContext("reader-1", AdminRole.READONLY, ACTION_ID, "trace-1");
+
+      assertThatThrownBy(() -> operations.close(adminContext))
+          .isInstanceOf(AccessDeniedException.class);
+
+      verify(audits)
+          .begin(adminContext, AdminAction.MARKET_CLOSE.name(), "market-1", "operator request");
+      verify(audits).complete(ACTION_ID, AuditOutcome.FAILED, 403);
+      assertThat(operations.invocations()).hasValue(0);
+    }
+  }
+
+  @Configuration(proxyBeanMethods = false)
+  @EnableAspectJAutoProxy
+  @EnableMethodSecurity
+  static class DeniedAuditConfiguration {
+
+    @Bean
+    AuditService auditService() {
+      return mock(AuditService.class);
+    }
+
+    @Bean
+    AuditAspect auditAspect(AuditService auditService) {
+      return new AuditAspect(auditService);
+    }
+
+    @Bean
+    SecuredOperations securedOperations() {
+      return new SecuredOperations();
+    }
+  }
+
+  static class SecuredOperations {
+
+    private final AtomicInteger invocations = new AtomicInteger();
+
+    @Audited(
+        action = AdminAction.MARKET_CLOSE,
+        target = "'market-1'",
+        reason = "'operator request'")
+    @PreAuthorize("hasRole('TRADER')")
+    public void close(AdminContext context) {
+      invocations.incrementAndGet();
+    }
+
+    AtomicInteger invocations() {
+      return invocations;
+    }
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


## `test(audit): return 503 with the action identity`

diff --git a/src/test/java/com/sportsbook/admin/audit/AuditFinalizationResponseTest.java b/src/test/java/com/sportsbook/admin/audit/AuditFinalizationResponseTest.java
new file mode 100644
index 0000000..9545e6c
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/audit/AuditFinalizationResponseTest.java
@@ -0,0 +1,59 @@
+package com.sportsbook.admin.audit;
+
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.header;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
+
+import com.sportsbook.admin.api.AdminExceptionHandler;
+import com.sportsbook.admin.context.AdminContextArgumentResolver;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.http.MediaType;
+import org.springframework.test.web.servlet.MockMvc;
+import org.springframework.test.web.servlet.setup.MockMvcBuilders;
+import org.springframework.web.bind.annotation.PostMapping;
+import org.springframework.web.bind.annotation.RestController;
+
+class AuditFinalizationResponseTest {
+
+  private static final UUID ACTION_ID = UUID.fromString("018f0000-0000-7000-8000-000000000051");
+
+  @Test
+  void returns503WithTheSameActionIdWhenFinalizationFails() throws Exception {
+    AuditPersistenceException failure =
+        new AuditPersistenceException(
+            ACTION_ID,
+            AuditPersistenceException.Phase.COMPLETE,
+            new IllegalStateException("database unavailable"));
+    MockMvc mvc =
+        MockMvcBuilders.standaloneSetup(new FailureController(failure))
+            .setControllerAdvice(new AdminExceptionHandler())
+            .build();
+
+    mvc.perform(post("/admin/v1/test/finalize"))
+        .andExpect(status().isServiceUnavailable())
+        .andExpect(content().contentTypeCompatibleWith(MediaType.APPLICATION_PROBLEM_JSON))
+        .andExpect(
+            header().string(AdminContextArgumentResolver.ACTION_ID_HEADER, ACTION_ID.toString()))
+        .andExpect(jsonPath("$.errorCode").value("AUDIT_FINALIZATION_FAILED"))
+        .andExpect(jsonPath("$.status").value(503))
+        .andExpect(jsonPath("$.detail").value("The audit trail could not be finalized"));
+  }
+
+  @RestController
+  private static final class FailureController {
+
+    private final RuntimeException failure;
+
+    private FailureController(RuntimeException failure) {
+      this.failure = failure;
+    }
+
+    @PostMapping("/admin/v1/test/finalize")
+    void fail() {
+      throw failure;
+    }
+  }
+}


## `test(audit): persist lifecycle records in PostgreSQL`

diff --git a/src/test/java/com/sportsbook/admin/audit/AuditLogRepositoryTest.java b/src/test/java/com/sportsbook/admin/audit/AuditLogRepositoryTest.java
new file mode 100644
index 0000000..8899275
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/audit/AuditLogRepositoryTest.java
@@ -0,0 +1,63 @@
+package com.sportsbook.admin.audit;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.admin.security.AdminRole;
+import java.time.Instant;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
+import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
+import org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManager;
+import org.springframework.test.context.DynamicPropertyRegistry;
+import org.springframework.test.context.DynamicPropertySource;
+import org.testcontainers.containers.PostgreSQLContainer;
+import org.testcontainers.junit.jupiter.Container;
+import org.testcontainers.junit.jupiter.Testcontainers;
+
+@DataJpaTest(properties = {"spring.jpa.hibernate.ddl-auto=validate", "spring.flyway.enabled=true"})
+@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
+@Testcontainers
+class AuditLogRepositoryTest {
+
+  @Container
+  private static final PostgreSQLContainer<?> POSTGRES =
+      new PostgreSQLContainer<>("postgres:16-alpine");
+
+  @DynamicPropertySource
+  static void database(DynamicPropertyRegistry registry) {
+    registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
+    registry.add("spring.datasource.username", POSTGRES::getUsername);
+    registry.add("spring.datasource.password", POSTGRES::getPassword);
+  }
+
+  @Autowired private AuditLogRepository repository;
+  @Autowired private TestEntityManager entities;
+
+  @Test
+  void persistsAndReadsNullableStartedLifecycleEvidence() {
+    UUID actionId = UUID.fromString("018f0000-0000-7000-8000-000000000011");
+    Instant startedAt = Instant.parse("2026-08-23T00:00:00Z");
+    repository.saveAndFlush(
+        new AuditLogEntity(
+            actionId,
+            "operator-1",
+            AdminRole.ADMIN,
+            "WALLET_REFUND",
+            "user-1",
+            AuditOutcome.STARTED,
+            null,
+            "refund",
+            "trace-1",
+            startedAt,
+            null));
+    entities.clear();
+
+    AuditLogEntity found = repository.findById(actionId).orElseThrow();
+    assertThat(found.getOutcome()).isEqualTo(AuditOutcome.STARTED);
+    assertThat(found.getHttpStatus()).isNull();
+    assertThat(found.getStartedAt()).isEqualTo(startedAt);
+    assertThat(found.getCompletedAt()).isNull();
+  }
+}
