## `test(audit): allow one terminal transition under race`

diff --git a/src/test/java/com/sportsbook/admin/audit/AuditTerminalRaceTest.java b/src/test/java/com/sportsbook/admin/audit/AuditTerminalRaceTest.java
new file mode 100644
index 0000000..294294a
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/audit/AuditTerminalRaceTest.java
@@ -0,0 +1,89 @@
+package com.sportsbook.admin.audit;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.admin.context.AdminContext;
+import com.sportsbook.admin.security.AdminRole;
+import java.util.List;
+import java.util.UUID;
+import java.util.concurrent.CountDownLatch;
+import java.util.concurrent.ExecutionException;
+import java.util.concurrent.ExecutorService;
+import java.util.concurrent.Executors;
+import java.util.concurrent.Future;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
+import org.springframework.boot.test.autoconfigure.jdbc.JdbcTest;
+import org.springframework.context.annotation.Import;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.test.context.DynamicPropertyRegistry;
+import org.springframework.test.context.DynamicPropertySource;
+import org.testcontainers.containers.PostgreSQLContainer;
+import org.testcontainers.junit.jupiter.Container;
+import org.testcontainers.junit.jupiter.Testcontainers;
+
+@JdbcTest(properties = "spring.flyway.enabled=true")
+@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
+@Import(AuditWriteRepository.class)
+@Testcontainers
+class AuditTerminalRaceTest {
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
+  @Autowired private AuditWriteRepository auditWrites;
+  @Autowired private JdbcTemplate jdbc;
+
+  @Test
+  void exactlyOneConcurrentTerminalUpdateCanClaimStarted() throws Exception {
+    UUID actionId = UUID.fromString("018f0000-0000-7000-8000-000000000031");
+    auditWrites.begin(
+        new AdminContext("operator-1", AdminRole.ADMIN, actionId, "trace-1"),
+        "MARKET_CLOSE",
+        "market-1",
+        "close");
+    CountDownLatch start = new CountDownLatch(1);
+    ExecutorService workers = Executors.newFixedThreadPool(2);
+
+    try {
+      Future<Object> success =
+          workers.submit(() -> completeAfter(start, actionId, AuditOutcome.SUCCESS, 202));
+      Future<Object> failed =
+          workers.submit(() -> completeAfter(start, actionId, AuditOutcome.FAILED, 409));
+      start.countDown();
+      List<Object> outcomes = List.of(result(success), result(failed));
+
+      assertThat(outcomes).filteredOn(AuditTerminalRecord.class::isInstance).hasSize(1);
+      assertThat(outcomes).filteredOn(IllegalStateException.class::isInstance).hasSize(1);
+      assertThat(
+              jdbc.queryForObject(
+                  "SELECT outcome FROM audit_log WHERE action_id = ?", String.class, actionId))
+          .isIn("SUCCESS", "FAILED");
+    } finally {
+      workers.shutdownNow();
+    }
+  }
+
+  private Object completeAfter(
+      CountDownLatch start, UUID actionId, AuditOutcome outcome, int status) throws Exception {
+    start.await();
+    return auditWrites.complete(actionId, outcome, status);
+  }
+
+  private static Object result(Future<Object> future) throws Exception {
+    try {
+      return future.get();
+    } catch (ExecutionException failure) {
+      return failure.getCause();
+    }
+  }
+}


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


## `test(audit): stop before downstream when STARTED fails`

diff --git a/src/test/java/com/sportsbook/admin/audit/AuditStartedInsertionTest.java b/src/test/java/com/sportsbook/admin/audit/AuditStartedInsertionTest.java
new file mode 100644
index 0000000..981c282
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/audit/AuditStartedInsertionTest.java
@@ -0,0 +1,70 @@
+package com.sportsbook.admin.audit;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.admin.context.AdminContext;
+import com.sportsbook.admin.security.AdminRole;
+import java.util.UUID;
+import java.util.concurrent.atomic.AtomicInteger;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
+import org.springframework.boot.test.autoconfigure.jdbc.JdbcTest;
+import org.springframework.context.annotation.Import;
+import org.springframework.dao.DataAccessException;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.test.context.DynamicPropertyRegistry;
+import org.springframework.test.context.DynamicPropertySource;
+import org.testcontainers.containers.PostgreSQLContainer;
+import org.testcontainers.junit.jupiter.Container;
+import org.testcontainers.junit.jupiter.Testcontainers;
+
+@JdbcTest(properties = "spring.flyway.enabled=true")
+@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
+@Import(AuditWriteRepository.class)
+@Testcontainers
+class AuditStartedInsertionTest {
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
+  @Autowired private AuditWriteRepository auditWrites;
+  @Autowired private JdbcTemplate jdbc;
+
+  @Test
+  void insertsStartedAndPreventsDownstreamWorkWhenThatGateFails() {
+    UUID actionId = UUID.fromString("018f0000-0000-7000-8000-000000000021");
+    AdminContext context = new AdminContext("operator-1", AdminRole.ADMIN, actionId, "trace-1");
+    auditWrites.begin(context, "WALLET_REFUND", "user-1", "refund");
+
+    assertThat(
+            jdbc.queryForObject(
+                "SELECT outcome FROM audit_log WHERE action_id = ?", String.class, actionId))
+        .isEqualTo("STARTED");
+    assertThat(
+            jdbc.queryForObject(
+                "SELECT http_status IS NULL AND completed_at IS NULL "
+                    + "FROM audit_log WHERE action_id = ?",
+                Boolean.class,
+                actionId))
+        .isTrue();
+
+    AtomicInteger downstreamCalls = new AtomicInteger();
+    assertThatThrownBy(
+            () -> {
+              auditWrites.begin(context, "WALLET_REFUND", "user-1", "refund");
+              downstreamCalls.incrementAndGet();
+            })
+        .isInstanceOf(DataAccessException.class);
+    assertThat(downstreamCalls).hasValue(0);
+  }
+}


## `test(audit): finalize successful actions once`

diff --git a/src/test/java/com/sportsbook/admin/audit/AuditAspectTest.java b/src/test/java/com/sportsbook/admin/audit/AuditAspectTest.java
new file mode 100644
index 0000000..81209e8
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/audit/AuditAspectTest.java
@@ -0,0 +1,70 @@
+package com.sportsbook.admin.audit;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.doAnswer;
+import static org.mockito.Mockito.mock;
+
+import com.sportsbook.admin.context.AdminContext;
+import com.sportsbook.admin.security.AdminRole;
+import java.util.ArrayList;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.aop.aspectj.annotation.AspectJProxyFactory;
+
+class AuditAspectTest {
+
+  private static final UUID ACTION_ID =
+      UUID.fromString("018f0000-0000-7000-8000-000000000061");
+  private static final AdminContext CONTEXT =
+      new AdminContext("operator-1", AdminRole.ADMIN, ACTION_ID, "trace-1");
+
+  @Test
+  void recordsStartedBeforeAndSuccessAfterTheDownstreamCall() {
+    List<String> events = new ArrayList<>();
+    AuditService audits = mock(AuditService.class);
+    doAnswer(
+            invocation -> {
+              events.add("begin");
+              return null;
+            })
+        .when(audits)
+        .begin(CONTEXT, AdminAction.WALLET_REFUND.name(), "user-1", "operator request");
+    doAnswer(
+            invocation -> {
+              events.add("complete");
+              return null;
+            })
+        .when(audits)
+        .complete(ACTION_ID, AuditOutcome.SUCCESS, 200);
+    AuditedOperations operations = proxy(audits, events);
+
+    assertThat(operations.success("user-1", CONTEXT)).isEqualTo("ok");
+
+    assertThat(events).containsExactly("begin", "downstream", "complete");
+  }
+
+  private static AuditedOperations proxy(AuditService audits, List<String> events) {
+    AspectJProxyFactory factory = new AspectJProxyFactory(new AuditedOperations(events));
+    factory.addAspect(new AuditAspect(audits));
+    return factory.getProxy();
+  }
+
+  static class AuditedOperations {
+
+    private final List<String> events;
+
+    AuditedOperations(List<String> events) {
+      this.events = events;
+    }
+
+    @Audited(
+        action = AdminAction.WALLET_REFUND,
+        target = "#p0",
+        reason = "'operator request'")
+    public String success(String userId, AdminContext context) {
+      events.add("downstream");
+      return "ok";
+    }
+  }
+}


## `test(audit): finalize failed actions once`

diff --git a/src/test/java/com/sportsbook/admin/audit/AuditAspectTest.java b/src/test/java/com/sportsbook/admin/audit/AuditAspectTest.java
index 81209e8..471ddca 100644
--- a/src/test/java/com/sportsbook/admin/audit/AuditAspectTest.java
+++ b/src/test/java/com/sportsbook/admin/audit/AuditAspectTest.java
@@ -1,8 +1,10 @@
 package com.sportsbook.admin.audit;
 
 import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
 import static org.mockito.Mockito.doAnswer;
 import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verify;
 
 import com.sportsbook.admin.context.AdminContext;
 import com.sportsbook.admin.security.AdminRole;
@@ -44,6 +46,21 @@ class AuditAspectTest {
     assertThat(events).containsExactly("begin", "downstream", "complete");
   }
 
+  @Test
+  void finalizesAndRethrowsAnExplicitLocalFailureExactlyOnce() {
+    List<String> events = new ArrayList<>();
+    AuditService audits = mock(AuditService.class);
+    AuditedOperations operations = proxy(audits, events);
+
+    assertThatThrownBy(() -> operations.fail("user-2", CONTEXT))
+        .isInstanceOf(IllegalArgumentException.class)
+        .hasMessage("invalid command");
+
+    verify(audits)
+        .begin(CONTEXT, AdminAction.WALLET_REFUND.name(), "user-2", "operator request");
+    verify(audits).complete(ACTION_ID, AuditOutcome.FAILED, 400);
+  }
+
   private static AuditedOperations proxy(AuditService audits, List<String> events) {
     AspectJProxyFactory factory = new AspectJProxyFactory(new AuditedOperations(events));
     factory.addAspect(new AuditAspect(audits));
@@ -66,5 +83,14 @@ class AuditAspectTest {
       events.add("downstream");
       return "ok";
     }
+
+    @Audited(
+        action = AdminAction.WALLET_REFUND,
+        target = "#p0",
+        reason = "'operator request'")
+    public String fail(String userId, AdminContext context) {
+      events.add("downstream");
+      throw new IllegalArgumentException("invalid command");
+    }
   }
 }


## `feat(audit): honor declared response statuses`

diff --git a/src/main/java/com/sportsbook/admin/audit/AuditAspect.java b/src/main/java/com/sportsbook/admin/audit/AuditAspect.java
index 4beb8ce..5f7acee 100644
--- a/src/main/java/com/sportsbook/admin/audit/AuditAspect.java
+++ b/src/main/java/com/sportsbook/admin/audit/AuditAspect.java
@@ -49,7 +49,9 @@ public final class AuditAspect {
     }
 
     AuditOutcomeClassifier.AuditDecision decision =
-        originalFailure == null ? outcomes.result(result) : outcomes.failure(originalFailure);
+        originalFailure == null
+            ? outcomes.result(result, method)
+            : outcomes.failure(originalFailure);
     try {
       audits.complete(context.actionId(), decision.outcome(), decision.httpStatus());
     } catch (AuditPersistenceException finalizationFailure) {
diff --git a/src/main/java/com/sportsbook/admin/audit/AuditOutcomeClassifier.java b/src/main/java/com/sportsbook/admin/audit/AuditOutcomeClassifier.java
index f70a170..3271e22 100644
--- a/src/main/java/com/sportsbook/admin/audit/AuditOutcomeClassifier.java
+++ b/src/main/java/com/sportsbook/admin/audit/AuditOutcomeClassifier.java
@@ -3,15 +3,25 @@ package com.sportsbook.admin.audit;
 import com.sportsbook.admin.client.DownstreamContractException;
 import com.sportsbook.admin.client.DownstreamStatusException;
 import com.sportsbook.admin.client.DownstreamUnavailableException;
+import java.lang.reflect.Method;
+import org.springframework.core.annotation.AnnotatedElementUtils;
 import org.springframework.http.HttpStatus;
 import org.springframework.http.HttpStatusCode;
 import org.springframework.http.ResponseEntity;
 import org.springframework.security.access.AccessDeniedException;
+import org.springframework.web.bind.annotation.ResponseStatus;
 
 final class AuditOutcomeClassifier {
 
   AuditDecision result(Object result) {
-    int status = result instanceof ResponseEntity<?> response ? response.getStatusCode().value() : 200;
+    return result(result, null);
+  }
+
+  AuditDecision result(Object result, Method method) {
+    int status =
+        result instanceof ResponseEntity<?> response
+            ? response.getStatusCode().value()
+            : declaredStatus(method);
     if (HttpStatusCode.valueOf(status).is2xxSuccessful()) {
       return new AuditDecision(AuditOutcome.SUCCESS, status);
     }
@@ -21,6 +31,14 @@ final class AuditOutcomeClassifier {
     return new AuditDecision(AuditOutcome.UNKNOWN, status);
   }
 
+  private static int declaredStatus(Method method) {
+    ResponseStatus responseStatus =
+        method == null
+            ? null
+            : AnnotatedElementUtils.findMergedAnnotation(method, ResponseStatus.class);
+    return responseStatus == null ? HttpStatus.OK.value() : responseStatus.code().value();
+  }
+
   AuditDecision failure(Throwable failure) {
     if (failure instanceof DownstreamStatusException rejection) {
       return new AuditDecision(AuditOutcome.FAILED, rejection.status().value());


## `test(audit): classify declared response statuses`

diff --git a/src/test/java/com/sportsbook/admin/audit/AuditDeclaredStatusClassifierTest.java b/src/test/java/com/sportsbook/admin/audit/AuditDeclaredStatusClassifierTest.java
new file mode 100644
index 0000000..7e831aa
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/audit/AuditDeclaredStatusClassifierTest.java
@@ -0,0 +1,37 @@
+package com.sportsbook.admin.audit;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.lang.reflect.Method;
+import org.junit.jupiter.api.Test;
+import org.springframework.http.HttpStatus;
+import org.springframework.web.bind.annotation.ResponseStatus;
+
+class AuditDeclaredStatusClassifierTest {
+
+  private final AuditOutcomeClassifier classifier = new AuditOutcomeClassifier();
+
+  @Test
+  void recordsDeclaredAndDefaultSuccessStatuses() throws Exception {
+    assertStatus(Probe.class.getDeclaredMethod("accepted"), 202);
+    assertStatus(Probe.class.getDeclaredMethod("noContent"), 204);
+    assertStatus(Probe.class.getDeclaredMethod("ok"), 200);
+  }
+
+  private void assertStatus(Method method, int expectedStatus) {
+    AuditOutcomeClassifier.AuditDecision decision = classifier.result(null, method);
+    assertThat(decision.outcome()).isEqualTo(AuditOutcome.SUCCESS);
+    assertThat(decision.httpStatus()).isEqualTo(expectedStatus);
+  }
+
+  static class Probe {
+
+    @ResponseStatus(HttpStatus.ACCEPTED)
+    void accepted() {}
+
+    @ResponseStatus(HttpStatus.NO_CONTENT)
+    void noContent() {}
+
+    void ok() {}
+  }
+}


