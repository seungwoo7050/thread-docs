# 트랜잭션 기반 실패 폐쇄 감사 수명 주기

## `feat(audit): surface persistence failure phases`

diff --git a/src/main/java/com/sportsbook/admin/audit/AuditPersistenceException.java b/src/main/java/com/sportsbook/admin/audit/AuditPersistenceException.java
new file mode 100644
index 0000000..5a266c9
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/audit/AuditPersistenceException.java
@@ -0,0 +1,28 @@
+package com.sportsbook.admin.audit;
+
+import java.util.UUID;
+
+public final class AuditPersistenceException extends RuntimeException {
+
+  public enum Phase {
+    BEGIN,
+    COMPLETE
+  }
+
+  private final UUID actionId;
+  private final Phase phase;
+
+  AuditPersistenceException(UUID actionId, Phase phase, Throwable cause) {
+    super("Audit persistence failed during " + phase, cause);
+    this.actionId = actionId;
+    this.phase = phase;
+  }
+
+  public UUID actionId() {
+    return actionId;
+  }
+
+  public Phase phase() {
+    return phase;
+  }
+}
diff --git a/src/main/java/com/sportsbook/admin/audit/AuditService.java b/src/main/java/com/sportsbook/admin/audit/AuditService.java
new file mode 100644
index 0000000..635469a
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/audit/AuditService.java
@@ -0,0 +1,34 @@
+package com.sportsbook.admin.audit;
+
+import com.sportsbook.admin.context.AdminContext;
+import java.util.UUID;
+import org.springframework.stereotype.Service;
+
+@Service
+public class AuditService {
+
+  private final AuditWriteRepository writes;
+
+  public AuditService(AuditWriteRepository writes) {
+    this.writes = writes;
+  }
+
+  public void begin(AdminContext context, String action, String target, String reason) {
+    try {
+      writes.begin(context, action, target, reason);
+    } catch (RuntimeException failure) {
+      throw new AuditPersistenceException(
+          context.actionId(), AuditPersistenceException.Phase.BEGIN, failure);
+    }
+  }
+
+  public AuditTerminalRecord complete(
+      UUID actionId, AuditOutcome outcome, Integer httpStatus) {
+    try {
+      return writes.complete(actionId, outcome, httpStatus);
+    } catch (RuntimeException failure) {
+      throw new AuditPersistenceException(
+          actionId, AuditPersistenceException.Phase.COMPLETE, failure);
+    }
+  }
+}


## `test(audit): preserve begin and completion failure phases`

diff --git a/src/test/java/com/sportsbook/admin/audit/AuditServiceFailureTest.java b/src/test/java/com/sportsbook/admin/audit/AuditServiceFailureTest.java
new file mode 100644
index 0000000..37534ac
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/audit/AuditServiceFailureTest.java
@@ -0,0 +1,55 @@
+package com.sportsbook.admin.audit;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.catchThrowableOfType;
+import static org.mockito.Mockito.doThrow;
+import static org.mockito.Mockito.mock;
+
+import com.sportsbook.admin.context.AdminContext;
+import com.sportsbook.admin.security.AdminRole;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.dao.DataAccessResourceFailureException;
+
+class AuditServiceFailureTest {
+
+  private static final UUID ACTION_ID =
+      UUID.fromString("018f0000-0000-7000-8000-000000000041");
+
+  @Test
+  void identifiesBeginPersistenceFailures() {
+    AuditWriteRepository writes = mock(AuditWriteRepository.class);
+    AuditService service = new AuditService(writes);
+    AdminContext context = new AdminContext("operator-1", AdminRole.ADMIN, ACTION_ID, "trace-1");
+    doThrow(new DataAccessResourceFailureException("database unavailable"))
+        .when(writes)
+        .begin(context, "WALLET_REFUND", "user-1", "refund");
+
+    AuditPersistenceException failure =
+        catchThrowableOfType(
+            () -> service.begin(context, "WALLET_REFUND", "user-1", "refund"),
+            AuditPersistenceException.class);
+
+    assertThat(failure.actionId()).isEqualTo(ACTION_ID);
+    assertThat(failure.phase()).isEqualTo(AuditPersistenceException.Phase.BEGIN);
+    assertThat(failure.getCause()).isInstanceOf(DataAccessResourceFailureException.class);
+  }
+
+  @Test
+  void identifiesTerminalFinalizationFailures() {
+    AuditWriteRepository writes = mock(AuditWriteRepository.class);
+    AuditService service = new AuditService(writes);
+    doThrow(new IllegalStateException("lost STARTED claim"))
+        .when(writes)
+        .complete(ACTION_ID, AuditOutcome.UNKNOWN, null);
+
+    AuditPersistenceException failure =
+        catchThrowableOfType(
+            () -> service.complete(ACTION_ID, AuditOutcome.UNKNOWN, null),
+            AuditPersistenceException.class);
+
+    assertThat(failure.actionId()).isEqualTo(ACTION_ID);
+    assertThat(failure.phase()).isEqualTo(AuditPersistenceException.Phase.COMPLETE);
+    assertThat(failure.getMessage()).doesNotContain("lost STARTED claim");
+  }
+}


## `feat(audit): classify terminal admin outcomes`

diff --git a/src/main/java/com/sportsbook/admin/audit/AdminAction.java b/src/main/java/com/sportsbook/admin/audit/AdminAction.java
new file mode 100644
index 0000000..f411dc1
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/audit/AdminAction.java
@@ -0,0 +1,13 @@
+package com.sportsbook.admin.audit;
+
+public enum AdminAction {
+  WALLET_REFUND,
+  RISK_LIMIT_UPDATE,
+  RISK_LIMIT_CLEAR,
+  MARKET_SUSPEND,
+  MARKET_CLOSE,
+  MARKET_REOPEN,
+  RESULT_CANDIDATE_APPROVE,
+  RESULT_CANDIDATE_REJECT,
+  SETTLEMENT_REVISION_RETRY
+}
diff --git a/src/main/java/com/sportsbook/admin/audit/AuditOutcomeClassifier.java b/src/main/java/com/sportsbook/admin/audit/AuditOutcomeClassifier.java
new file mode 100644
index 0000000..f70a170
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/audit/AuditOutcomeClassifier.java
@@ -0,0 +1,54 @@
+package com.sportsbook.admin.audit;
+
+import com.sportsbook.admin.client.DownstreamContractException;
+import com.sportsbook.admin.client.DownstreamStatusException;
+import com.sportsbook.admin.client.DownstreamUnavailableException;
+import org.springframework.http.HttpStatus;
+import org.springframework.http.HttpStatusCode;
+import org.springframework.http.ResponseEntity;
+import org.springframework.security.access.AccessDeniedException;
+
+final class AuditOutcomeClassifier {
+
+  AuditDecision result(Object result) {
+    int status = result instanceof ResponseEntity<?> response ? response.getStatusCode().value() : 200;
+    if (HttpStatusCode.valueOf(status).is2xxSuccessful()) {
+      return new AuditDecision(AuditOutcome.SUCCESS, status);
+    }
+    if (HttpStatusCode.valueOf(status).is4xxClientError()) {
+      return new AuditDecision(AuditOutcome.FAILED, status);
+    }
+    return new AuditDecision(AuditOutcome.UNKNOWN, status);
+  }
+
+  AuditDecision failure(Throwable failure) {
+    if (failure instanceof DownstreamStatusException rejection) {
+      return new AuditDecision(AuditOutcome.FAILED, rejection.status().value());
+    }
+    if (failure instanceof DownstreamUnavailableException unavailable) {
+      int status =
+          unavailable.reason() == DownstreamUnavailableException.Reason.TIMEOUT
+              ? HttpStatus.GATEWAY_TIMEOUT.value()
+              : HttpStatus.BAD_GATEWAY.value();
+      return new AuditDecision(AuditOutcome.UNKNOWN, status);
+    }
+    if (failure instanceof DownstreamContractException) {
+      return new AuditDecision(AuditOutcome.UNKNOWN, HttpStatus.BAD_GATEWAY.value());
+    }
+    if (failure instanceof AccessDeniedException) {
+      return new AuditDecision(AuditOutcome.FAILED, HttpStatus.FORBIDDEN.value());
+    }
+    if (failure instanceof IllegalArgumentException) {
+      return new AuditDecision(AuditOutcome.FAILED, HttpStatus.BAD_REQUEST.value());
+    }
+    return new AuditDecision(AuditOutcome.UNKNOWN, HttpStatus.INTERNAL_SERVER_ERROR.value());
+  }
+
+  record AuditDecision(AuditOutcome outcome, Integer httpStatus) {
+    AuditDecision {
+      if (!outcome.isTerminal()) {
+        throw new IllegalArgumentException("Audit decision must be terminal");
+      }
+    }
+  }
+}


## `test(audit): separate failed from unknown outcomes`

diff --git a/src/test/java/com/sportsbook/admin/audit/AuditOutcomeClassifierTest.java b/src/test/java/com/sportsbook/admin/audit/AuditOutcomeClassifierTest.java
new file mode 100644
index 0000000..5ca0799
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/audit/AuditOutcomeClassifierTest.java
@@ -0,0 +1,87 @@
+package com.sportsbook.admin.audit;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.catchThrowable;
+
+import com.sportsbook.admin.client.DownstreamContract;
+import com.sportsbook.admin.client.DownstreamFailureMapper;
+import java.net.SocketTimeoutException;
+import java.util.Arrays;
+import java.util.function.Supplier;
+import org.junit.jupiter.api.Test;
+import org.springframework.http.HttpStatus;
+import org.springframework.http.ResponseEntity;
+import org.springframework.security.access.AccessDeniedException;
+import org.springframework.web.client.HttpClientErrorException;
+import org.springframework.web.client.HttpServerErrorException;
+import org.springframework.web.client.ResourceAccessException;
+
+class AuditOutcomeClassifierTest {
+
+  private final AuditOutcomeClassifier classifier = new AuditOutcomeClassifier();
+  private final DownstreamFailureMapper failures = new DownstreamFailureMapper();
+
+  @Test
+  void classifiesResultsByTheirActualHttpStatus() {
+    assertThat(classifier.result("ok").outcome()).isEqualTo(AuditOutcome.SUCCESS);
+    assertThat(classifier.result(ResponseEntity.accepted().build()).outcome())
+        .isEqualTo(AuditOutcome.SUCCESS);
+    assertThat(classifier.result(ResponseEntity.badRequest().build()).outcome())
+        .isEqualTo(AuditOutcome.FAILED);
+    assertThat(
+            classifier
+                .result(ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE).build())
+                .outcome())
+        .isEqualTo(AuditOutcome.UNKNOWN);
+  }
+
+  @Test
+  void treatsOnlyExplicit4xxAndLocalDenialsAsFailed() {
+    Throwable downstream4xx =
+        mapped(new HttpClientErrorException(HttpStatus.CONFLICT));
+
+    assertThat(classifier.failure(downstream4xx).outcome()).isEqualTo(AuditOutcome.FAILED);
+    assertThat(classifier.failure(new AccessDeniedException("denied")).outcome())
+        .isEqualTo(AuditOutcome.FAILED);
+    assertThat(classifier.failure(new IllegalArgumentException("invalid")).httpStatus())
+        .isEqualTo(400);
+  }
+
+  @Test
+  void treatsAmbiguousAndMalformedMutationOutcomesAsUnknown() {
+    Throwable serverError =
+        mapped(new HttpServerErrorException(HttpStatus.SERVICE_UNAVAILABLE));
+    Throwable timeout =
+        mapped(new ResourceAccessException("timeout", new SocketTimeoutException()));
+    Throwable malformed =
+        catchThrowable(
+            () ->
+                DownstreamContract.requireBody(
+                    ResponseEntity.ok().build(),
+                    HttpStatus.OK,
+                    value -> true,
+                    "missing receipt"));
+
+    assertThat(classifier.failure(serverError).outcome()).isEqualTo(AuditOutcome.UNKNOWN);
+    assertThat(classifier.failure(timeout).httpStatus()).isEqualTo(504);
+    assertThat(classifier.failure(malformed).outcome()).isEqualTo(AuditOutcome.UNKNOWN);
+    assertThat(classifier.failure(new IllegalStateException("unexpected")).outcome())
+        .isEqualTo(AuditOutcome.UNKNOWN);
+  }
+
+  @Test
+  void containsNoRemovedVoidOrReplayActions() {
+    assertThat(Arrays.stream(AdminAction.values()).map(Enum::name))
+        .noneMatch(name -> name.contains("VOID") || name.contains("REPLAY"));
+  }
+
+  private Throwable mapped(RuntimeException source) {
+    return catchThrowable(
+        () ->
+            failures.execute(
+                (Supplier<Object>)
+                    () -> {
+                      throw source;
+                    }));
+  }
+}


## `feat(audit): declare audited action metadata`

diff --git a/src/main/java/com/sportsbook/admin/audit/Audited.java b/src/main/java/com/sportsbook/admin/audit/Audited.java
new file mode 100644
index 0000000..b6fa085
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/audit/Audited.java
@@ -0,0 +1,17 @@
+package com.sportsbook.admin.audit;
+
+import java.lang.annotation.ElementType;
+import java.lang.annotation.Retention;
+import java.lang.annotation.RetentionPolicy;
+import java.lang.annotation.Target;
+
+@Target(ElementType.METHOD)
+@Retention(RetentionPolicy.RUNTIME)
+public @interface Audited {
+
+  AdminAction action();
+
+  String target() default "";
+
+  String reason() default "";
+}


## `test(audit): verify audited action metadata`

diff --git a/src/test/java/com/sportsbook/admin/audit/AuditedMetadataTest.java b/src/test/java/com/sportsbook/admin/audit/AuditedMetadataTest.java
new file mode 100644
index 0000000..c38f777
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/audit/AuditedMetadataTest.java
@@ -0,0 +1,31 @@
+package com.sportsbook.admin.audit;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.lang.annotation.ElementType;
+import java.lang.annotation.Retention;
+import java.lang.annotation.RetentionPolicy;
+import java.lang.annotation.Target;
+import org.junit.jupiter.api.Test;
+
+class AuditedMetadataTest {
+
+  @Test
+  void retainsMethodMetadataForRuntimeInterception() throws Exception {
+    assertThat(Audited.class.getAnnotation(Retention.class).value())
+        .isEqualTo(RetentionPolicy.RUNTIME);
+    assertThat(Audited.class.getAnnotation(Target.class).value())
+        .containsExactly(ElementType.METHOD);
+
+    Audited metadata = Probe.class.getDeclaredMethod("refund").getAnnotation(Audited.class);
+    assertThat(metadata.action()).isEqualTo(AdminAction.WALLET_REFUND);
+    assertThat(metadata.target()).isEqualTo("#p0");
+    assertThat(metadata.reason()).isEqualTo("'operator request'");
+  }
+
+  static class Probe {
+
+    @Audited(action = AdminAction.WALLET_REFUND, target = "#p0", reason = "'operator request'")
+    void refund() {}
+  }
+}


## `feat(audit): persist STARTED before downstream work`

diff --git a/src/main/java/com/sportsbook/admin/audit/AuditWriteRepository.java b/src/main/java/com/sportsbook/admin/audit/AuditWriteRepository.java
new file mode 100644
index 0000000..795f415
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/audit/AuditWriteRepository.java
@@ -0,0 +1,42 @@
+package com.sportsbook.admin.audit;
+
+import com.sportsbook.admin.context.AdminContext;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.stereotype.Repository;
+import org.springframework.transaction.annotation.Propagation;
+import org.springframework.transaction.annotation.Transactional;
+
+@Repository
+public class AuditWriteRepository {
+
+  private static final String INSERT_STARTED =
+      """
+      INSERT INTO audit_log
+          (action_id, actor_id, actor_role, action, target, outcome,
+           http_status, reason, trace_id, started_at, completed_at)
+      VALUES (?, ?, ?, ?, ?, 'STARTED', NULL, ?, ?, CURRENT_TIMESTAMP, NULL)
+      """;
+
+  private final JdbcTemplate jdbc;
+
+  public AuditWriteRepository(JdbcTemplate jdbc) {
+    this.jdbc = jdbc;
+  }
+
+  @Transactional(propagation = Propagation.REQUIRES_NEW, timeout = 5)
+  public void begin(AdminContext context, String action, String target, String reason) {
+    int inserted =
+        jdbc.update(
+            INSERT_STARTED,
+            context.actionId(),
+            context.actorId(),
+            context.actorRole().name(),
+            action,
+            target,
+            reason,
+            context.traceId());
+    if (inserted != 1) {
+      throw new IllegalStateException("Audit STARTED insertion did not affect exactly one row");
+    }
+  }
+}


## `test(audit): stop when STARTED persistence fails`

diff --git a/src/test/java/com/sportsbook/admin/audit/AuditAspectTest.java b/src/test/java/com/sportsbook/admin/audit/AuditAspectTest.java
index 471ddca..bc8bb59 100644
--- a/src/test/java/com/sportsbook/admin/audit/AuditAspectTest.java
+++ b/src/test/java/com/sportsbook/admin/audit/AuditAspectTest.java
@@ -3,7 +3,9 @@ package com.sportsbook.admin.audit;
 import static org.assertj.core.api.Assertions.assertThat;
 import static org.assertj.core.api.Assertions.assertThatThrownBy;
 import static org.mockito.Mockito.doAnswer;
+import static org.mockito.Mockito.doThrow;
 import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.never;
 import static org.mockito.Mockito.verify;
 
 import com.sportsbook.admin.context.AdminContext;
@@ -16,8 +18,7 @@ import org.springframework.aop.aspectj.annotation.AspectJProxyFactory;
 
 class AuditAspectTest {
 
-  private static final UUID ACTION_ID =
-      UUID.fromString("018f0000-0000-7000-8000-000000000061");
+  private static final UUID ACTION_ID = UUID.fromString("018f0000-0000-7000-8000-000000000061");
   private static final AdminContext CONTEXT =
       new AdminContext("operator-1", AdminRole.ADMIN, ACTION_ID, "trace-1");
 
@@ -56,11 +57,30 @@ class AuditAspectTest {
         .isInstanceOf(IllegalArgumentException.class)
         .hasMessage("invalid command");
 
-    verify(audits)
-        .begin(CONTEXT, AdminAction.WALLET_REFUND.name(), "user-2", "operator request");
+    verify(audits).begin(CONTEXT, AdminAction.WALLET_REFUND.name(), "user-2", "operator request");
     verify(audits).complete(ACTION_ID, AuditOutcome.FAILED, 400);
   }
 
+  @Test
+  void neverInvokesDownstreamOrCompletionWhenStartedCannotPersist() {
+    List<String> events = new ArrayList<>();
+    AuditService audits = mock(AuditService.class);
+    doThrow(
+            new AuditPersistenceException(
+                ACTION_ID,
+                AuditPersistenceException.Phase.BEGIN,
+                new IllegalStateException("db down")))
+        .when(audits)
+        .begin(CONTEXT, AdminAction.WALLET_REFUND.name(), "user-3", "operator request");
+    AuditedOperations operations = proxy(audits, events);
+
+    assertThatThrownBy(() -> operations.success("user-3", CONTEXT))
+        .isInstanceOf(AuditPersistenceException.class);
+
+    assertThat(events).isEmpty();
+    verify(audits, never()).complete(ACTION_ID, AuditOutcome.SUCCESS, 200);
+  }
+
   private static AuditedOperations proxy(AuditService audits, List<String> events) {
     AspectJProxyFactory factory = new AspectJProxyFactory(new AuditedOperations(events));
     factory.addAspect(new AuditAspect(audits));
@@ -75,19 +95,13 @@ class AuditAspectTest {
       this.events = events;
     }
 
-    @Audited(
-        action = AdminAction.WALLET_REFUND,
-        target = "#p0",
-        reason = "'operator request'")
+    @Audited(action = AdminAction.WALLET_REFUND, target = "#p0", reason = "'operator request'")
     public String success(String userId, AdminContext context) {
       events.add("downstream");
       return "ok";
     }
 
-    @Audited(
-        action = AdminAction.WALLET_REFUND,
-        target = "#p0",
-        reason = "'operator request'")
+    @Audited(action = AdminAction.WALLET_REFUND, target = "#p0", reason = "'operator request'")
     public String fail(String userId, AdminContext context) {
       events.add("downstream");
       throw new IllegalArgumentException("invalid command");


## `feat(audit): guard the terminal lifecycle transition`

diff --git a/src/main/java/com/sportsbook/admin/audit/AuditTerminalRecord.java b/src/main/java/com/sportsbook/admin/audit/AuditTerminalRecord.java
new file mode 100644
index 0000000..3fff2aa
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/audit/AuditTerminalRecord.java
@@ -0,0 +1,36 @@
+package com.sportsbook.admin.audit;
+
+import com.sportsbook.admin.security.AdminRole;
+import java.time.Instant;
+import java.util.Objects;
+import java.util.UUID;
+
+public record AuditTerminalRecord(
+    UUID actionId,
+    String actorId,
+    AdminRole actorRole,
+    String action,
+    String target,
+    AuditOutcome outcome,
+    Integer httpStatus,
+    String reason,
+    String traceId,
+    Instant startedAt,
+    Instant completedAt) {
+
+  public AuditTerminalRecord {
+    Objects.requireNonNull(actionId, "actionId");
+    Objects.requireNonNull(actorId, "actorId");
+    Objects.requireNonNull(actorRole, "actorRole");
+    Objects.requireNonNull(action, "action");
+    Objects.requireNonNull(outcome, "outcome");
+    Objects.requireNonNull(startedAt, "startedAt");
+    Objects.requireNonNull(completedAt, "completedAt");
+    if (!outcome.isTerminal()) {
+      throw new IllegalArgumentException("Terminal audit record cannot be STARTED");
+    }
+    if (outcome != AuditOutcome.UNKNOWN && httpStatus == null) {
+      throw new IllegalArgumentException("SUCCESS and FAILED require an HTTP status");
+    }
+  }
+}
diff --git a/src/main/java/com/sportsbook/admin/audit/AuditWriteRepository.java b/src/main/java/com/sportsbook/admin/audit/AuditWriteRepository.java
index 795f415..99e746d 100644
--- a/src/main/java/com/sportsbook/admin/audit/AuditWriteRepository.java
+++ b/src/main/java/com/sportsbook/admin/audit/AuditWriteRepository.java
@@ -1,6 +1,10 @@
 package com.sportsbook.admin.audit;
 
 import com.sportsbook.admin.context.AdminContext;
+import com.sportsbook.admin.security.AdminRole;
+import java.sql.Timestamp;
+import java.util.List;
+import java.util.UUID;
 import org.springframework.jdbc.core.JdbcTemplate;
 import org.springframework.stereotype.Repository;
 import org.springframework.transaction.annotation.Propagation;
@@ -17,6 +21,15 @@ public class AuditWriteRepository {
       VALUES (?, ?, ?, ?, ?, 'STARTED', NULL, ?, ?, CURRENT_TIMESTAMP, NULL)
       """;
 
+  private static final String COMPLETE_STARTED =
+      """
+      UPDATE audit_log
+      SET outcome = ?, http_status = ?, completed_at = CURRENT_TIMESTAMP
+      WHERE action_id = ? AND outcome = 'STARTED'
+      RETURNING action_id, actor_id, actor_role, action, target, outcome,
+                http_status, reason, trace_id, started_at, completed_at
+      """;
+
   private final JdbcTemplate jdbc;
 
   public AuditWriteRepository(JdbcTemplate jdbc) {
@@ -39,4 +52,39 @@ public class AuditWriteRepository {
       throw new IllegalStateException("Audit STARTED insertion did not affect exactly one row");
     }
   }
+
+  @Transactional(propagation = Propagation.REQUIRES_NEW, timeout = 5)
+  public AuditTerminalRecord complete(
+      UUID actionId, AuditOutcome outcome, Integer httpStatus) {
+    if (!outcome.isTerminal()) {
+      throw new IllegalArgumentException("Cannot complete an audit row as STARTED");
+    }
+    List<AuditTerminalRecord> updated =
+        jdbc.query(
+            COMPLETE_STARTED,
+            (result, rowNumber) ->
+                new AuditTerminalRecord(
+                    result.getObject("action_id", UUID.class),
+                    result.getString("actor_id"),
+                    AdminRole.valueOf(result.getString("actor_role")),
+                    result.getString("action"),
+                    result.getString("target"),
+                    AuditOutcome.valueOf(result.getString("outcome")),
+                    result.getObject("http_status", Integer.class),
+                    result.getString("reason"),
+                    result.getString("trace_id"),
+                    timestamp(result.getTimestamp("started_at")),
+                    timestamp(result.getTimestamp("completed_at"))),
+            outcome.name(),
+            httpStatus,
+            actionId);
+    if (updated.size() != 1) {
+      throw new IllegalStateException("Audit terminal update did not claim exactly one STARTED row");
+    }
+    return updated.get(0);
+  }
+
+  private static java.time.Instant timestamp(Timestamp timestamp) {
+    return timestamp.toInstant();
+  }
 }


