## `feat(settlement): reject result candidates`

diff --git a/src/main/java/com/sportsbook/admin/client/SettlementClient.java b/src/main/java/com/sportsbook/admin/client/SettlementClient.java
index 73124bd..3eb1404 100644
--- a/src/main/java/com/sportsbook/admin/client/SettlementClient.java
+++ b/src/main/java/com/sportsbook/admin/client/SettlementClient.java
@@ -83,4 +83,30 @@ public class SettlementClient {
     return SettlementCandidateReceipt.verify(
         idempotencyKey, SettlementCandidateReceipt.Outcome.CANDIDATE_APPROVED, receipt);
   }
+
+  public SettlementCandidateReceipt rejectCandidate(
+      UUID candidateId, UUID idempotencyKey, SettlementRejectionPayload payload) {
+    var response =
+        failures.execute(
+            () ->
+                http.post()
+                    .uri(
+                        builder ->
+                            builder
+                                .pathSegment("internal", "admin", "result-candidates")
+                                .pathSegment(candidateId.toString(), "reject")
+                                .build())
+                    .header("Idempotency-Key", idempotencyKey.toString())
+                    .body(payload)
+                    .retrieve()
+                    .toEntity(SettlementCandidateReceipt.class));
+    SettlementCandidateReceipt receipt =
+        DownstreamContract.requireBody(
+            response,
+            HttpStatus.OK,
+            ignored -> true,
+            "Settlement candidate rejection must return HTTP 200 with a receipt");
+    return SettlementCandidateReceipt.verify(
+        idempotencyKey, SettlementCandidateReceipt.Outcome.CANDIDATE_REJECTED, receipt);
+  }
 }


## `test(settlement): send exact candidate rejection`

diff --git a/src/test/java/com/sportsbook/admin/client/SettlementClientCandidateRejectionTest.java b/src/test/java/com/sportsbook/admin/client/SettlementClientCandidateRejectionTest.java
new file mode 100644
index 0000000..e693787
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/client/SettlementClientCandidateRejectionTest.java
@@ -0,0 +1,59 @@
+package com.sportsbook.admin.client;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.springframework.http.HttpMethod.POST;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.content;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.header;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.headerDoesNotExist;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.method;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.requestTo;
+import static org.springframework.test.web.client.response.MockRestResponseCreators.withSuccess;
+
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.http.MediaType;
+import org.springframework.test.web.client.MockRestServiceServer;
+import org.springframework.web.client.RestClient;
+
+class SettlementClientCandidateRejectionTest {
+
+  @Test
+  void postsTheExactRejectionWithSettlementAuthentication() {
+    UUID candidate = UUID.fromString("018f0000-0000-7000-8000-000000000165");
+    UUID key = UUID.fromString("018f0000-0000-7000-8000-000000000166");
+    RestClient.Builder builder = settlementBuilder();
+    MockRestServiceServer server = MockRestServiceServer.bindTo(builder).build();
+    server
+        .expect(
+            requestTo(
+                "http://settlement.test/internal/admin/result-candidates/" + candidate + "/reject"))
+        .andExpect(method(POST))
+        .andExpect(header(DownstreamHeaders.SERVICE_NAME, "admin-api"))
+        .andExpect(header(DownstreamHeaders.API_KEY, ClientIsolationFixture.SETTLEMENT))
+        .andExpect(header("Idempotency-Key", key.toString()))
+        .andExpect(headerDoesNotExist(DownstreamHeaders.INTERNAL_SERVICE))
+        .andExpect(content().json("{\"reason\":\"bad result\"}", true))
+        .andRespond(
+            withSuccess(
+                """
+                {"idempotencyKey":"%s","outcome":"CANDIDATE_REJECTED","replay":true}
+                """
+                    .formatted(key),
+                MediaType.APPLICATION_JSON));
+
+    SettlementCandidateReceipt receipt =
+        new SettlementClient(builder.build())
+            .rejectCandidate(candidate, key, new SettlementRejectionPayload("  bad result  "));
+
+    assertThat(receipt.outcome()).isEqualTo(SettlementCandidateReceipt.Outcome.CANDIDATE_REJECTED);
+    assertThat(receipt.replay()).isTrue();
+    server.verify();
+  }
+
+  private static RestClient.Builder settlementBuilder() {
+    return RestClient.builder()
+        .baseUrl("http://settlement.test")
+        .defaultHeader(DownstreamHeaders.SERVICE_NAME, DownstreamHeaders.ADMIN_API)
+        .defaultHeader(DownstreamHeaders.API_KEY, ClientIsolationFixture.SETTLEMENT);
+  }
+}


## `feat(settlement): expose lifecycle queries`

diff --git a/src/main/java/com/sportsbook/admin/api/SettlementQueryController.java b/src/main/java/com/sportsbook/admin/api/SettlementQueryController.java
new file mode 100644
index 0000000..7754980
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/api/SettlementQueryController.java
@@ -0,0 +1,34 @@
+package com.sportsbook.admin.api;
+
+import com.sportsbook.admin.client.SettlementCandidateView;
+import com.sportsbook.admin.client.SettlementClient;
+import com.sportsbook.admin.client.SettlementRevisionView;
+import java.util.UUID;
+import org.springframework.security.access.prepost.PreAuthorize;
+import org.springframework.web.bind.annotation.GetMapping;
+import org.springframework.web.bind.annotation.PathVariable;
+import org.springframework.web.bind.annotation.RequestMapping;
+import org.springframework.web.bind.annotation.RestController;
+
+@RestController
+@RequestMapping("/admin/v1/settlements")
+public class SettlementQueryController {
+
+  private final SettlementClient settlements;
+
+  public SettlementQueryController(SettlementClient settlements) {
+    this.settlements = settlements;
+  }
+
+  @GetMapping("/result-candidates/{candidateId}")
+  @PreAuthorize("hasAnyRole('ADMIN','TRADER','CS','READONLY')")
+  public SettlementCandidateView getCandidate(@PathVariable UUID candidateId) {
+    return settlements.getCandidate(candidateId);
+  }
+
+  @GetMapping("/revisions/{revisionId}")
+  @PreAuthorize("hasAnyRole('ADMIN','TRADER','CS','READONLY')")
+  public SettlementRevisionView getRevision(@PathVariable UUID revisionId) {
+    return settlements.getRevision(revisionId);
+  }
+}


## `test(settlement): guard lifecycle queries`

diff --git a/src/test/java/com/sportsbook/admin/api/SettlementQueryControllerTest.java b/src/test/java/com/sportsbook/admin/api/SettlementQueryControllerTest.java
new file mode 100644
index 0000000..823a24b
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/api/SettlementQueryControllerTest.java
@@ -0,0 +1,50 @@
+package com.sportsbook.admin.api;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verify;
+
+import com.sportsbook.admin.audit.Audited;
+import com.sportsbook.admin.client.SettlementClient;
+import java.lang.reflect.Method;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.security.access.prepost.PreAuthorize;
+import org.springframework.web.bind.annotation.GetMapping;
+import org.springframework.web.bind.annotation.RequestMapping;
+
+class SettlementQueryControllerTest {
+
+  private static final UUID TARGET = UUID.fromString("018f0000-0000-7000-8000-000000000171");
+
+  @Test
+  void exposesCandidateEvidenceToEveryAdminRoleWithoutMutationAudit() throws NoSuchMethodException {
+    SettlementClient settlements = mock(SettlementClient.class);
+
+    new SettlementQueryController(settlements).getCandidate(TARGET);
+
+    verify(settlements).getCandidate(TARGET);
+    assertReadContract("getCandidate", "/result-candidates/{candidateId}");
+  }
+
+  @Test
+  void exposesRevisionEvidenceToEveryAdminRoleWithoutMutationAudit() throws NoSuchMethodException {
+    SettlementClient settlements = mock(SettlementClient.class);
+
+    new SettlementQueryController(settlements).getRevision(TARGET);
+
+    verify(settlements).getRevision(TARGET);
+    assertReadContract("getRevision", "/revisions/{revisionId}");
+  }
+
+  private static void assertReadContract(String methodName, String path)
+      throws NoSuchMethodException {
+    assertThat(SettlementQueryController.class.getAnnotation(RequestMapping.class).value())
+        .containsExactly("/admin/v1/settlements");
+    Method method = SettlementQueryController.class.getMethod(methodName, UUID.class);
+    assertThat(method.getAnnotation(GetMapping.class).value()).containsExactly(path);
+    assertThat(method.getAnnotation(PreAuthorize.class).value())
+        .isEqualTo("hasAnyRole('ADMIN','TRADER','CS','READONLY')");
+    assertThat(method.getAnnotation(Audited.class)).isNull();
+  }
+}


## `feat(settlement): expose candidate decisions`

diff --git a/src/main/java/com/sportsbook/admin/api/SettlementCandidateCommandController.java b/src/main/java/com/sportsbook/admin/api/SettlementCandidateCommandController.java
new file mode 100644
index 0000000..373e330
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/api/SettlementCandidateCommandController.java
@@ -0,0 +1,51 @@
+package com.sportsbook.admin.api;
+
+import com.sportsbook.admin.audit.AdminAction;
+import com.sportsbook.admin.audit.Audited;
+import com.sportsbook.admin.client.SettlementCandidateReceipt;
+import com.sportsbook.admin.client.SettlementClient;
+import com.sportsbook.admin.client.SettlementRejectionPayload;
+import com.sportsbook.admin.context.AdminContext;
+import jakarta.servlet.http.HttpServletRequest;
+import java.util.UUID;
+import org.springframework.security.access.prepost.PreAuthorize;
+import org.springframework.web.bind.annotation.PathVariable;
+import org.springframework.web.bind.annotation.PostMapping;
+import org.springframework.web.bind.annotation.RequestBody;
+import org.springframework.web.bind.annotation.RequestMapping;
+import org.springframework.web.bind.annotation.RestController;
+
+@RestController
+@RequestMapping("/admin/v1/settlements/result-candidates/{candidateId}")
+public class SettlementCandidateCommandController {
+
+  private final SettlementClient settlements;
+
+  public SettlementCandidateCommandController(SettlementClient settlements) {
+    this.settlements = settlements;
+  }
+
+  @PostMapping("/approve")
+  @PreAuthorize("hasAnyRole('ADMIN','TRADER')")
+  @Audited(action = AdminAction.RESULT_CANDIDATE_APPROVE, target = "#candidateId")
+  public SettlementCandidateReceipt approve(
+      @PathVariable UUID candidateId, AdminContext context, HttpServletRequest servletRequest) {
+    return settlements.approveCandidate(
+        candidateId, AdminRequestHeaders.requireUuidIdempotencyKey(servletRequest));
+  }
+
+  @PostMapping("/reject")
+  @PreAuthorize("hasAnyRole('ADMIN','TRADER')")
+  @Audited(
+      action = AdminAction.RESULT_CANDIDATE_REJECT,
+      target = "#candidateId",
+      reason = "#body.reason()")
+  public SettlementCandidateReceipt reject(
+      @PathVariable UUID candidateId,
+      @RequestBody SettlementRejectionPayload body,
+      AdminContext context,
+      HttpServletRequest servletRequest) {
+    return settlements.rejectCandidate(
+        candidateId, AdminRequestHeaders.requireUuidIdempotencyKey(servletRequest), body);
+  }
+}


## `test(settlement): guard candidate approvals`

diff --git a/src/test/java/com/sportsbook/admin/api/SettlementCandidateApprovalControllerTest.java b/src/test/java/com/sportsbook/admin/api/SettlementCandidateApprovalControllerTest.java
new file mode 100644
index 0000000..00d0c98
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/api/SettlementCandidateApprovalControllerTest.java
@@ -0,0 +1,62 @@
+package com.sportsbook.admin.api;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.admin.audit.AdminAction;
+import com.sportsbook.admin.audit.Audited;
+import com.sportsbook.admin.client.SettlementCandidateReceipt;
+import com.sportsbook.admin.client.SettlementClient;
+import com.sportsbook.admin.context.AdminContext;
+import com.sportsbook.admin.security.AdminRole;
+import jakarta.servlet.http.HttpServletRequest;
+import java.lang.reflect.Method;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.mock.web.MockHttpServletRequest;
+import org.springframework.security.access.prepost.PreAuthorize;
+import org.springframework.web.bind.annotation.PostMapping;
+
+class SettlementCandidateApprovalControllerTest {
+
+  private static final UUID CANDIDATE = UUID.fromString("018f0000-0000-7000-8000-000000000172");
+  private static final UUID KEY = UUID.fromString("018f0000-0000-7000-8000-000000000173");
+
+  @Test
+  void delegatesTheTypedApprovalAndReturnsItsReceipt() throws NoSuchMethodException {
+    SettlementClient settlements = mock(SettlementClient.class);
+    SettlementCandidateReceipt receipt =
+        new SettlementCandidateReceipt(
+            KEY, SettlementCandidateReceipt.Outcome.CANDIDATE_APPROVED, false);
+    when(settlements.approveCandidate(CANDIDATE, KEY)).thenReturn(receipt);
+
+    SettlementCandidateReceipt result =
+        new SettlementCandidateCommandController(settlements)
+            .approve(CANDIDATE, context(), requestWithKey());
+
+    assertThat(result).isSameAs(receipt);
+    verify(settlements).approveCandidate(CANDIDATE, KEY);
+    Method method =
+        SettlementCandidateCommandController.class.getMethod(
+            "approve", UUID.class, AdminContext.class, HttpServletRequest.class);
+    assertThat(method.getAnnotation(PostMapping.class).value()).containsExactly("/approve");
+    assertThat(method.getAnnotation(PreAuthorize.class).value())
+        .isEqualTo("hasAnyRole('ADMIN','TRADER')");
+    Audited audited = method.getAnnotation(Audited.class);
+    assertThat(audited.action()).isEqualTo(AdminAction.RESULT_CANDIDATE_APPROVE);
+    assertThat(audited.target()).isEqualTo("#candidateId");
+    assertThat(audited.reason()).isEmpty();
+  }
+
+  private static MockHttpServletRequest requestWithKey() {
+    MockHttpServletRequest request = new MockHttpServletRequest();
+    request.addHeader(AdminRequestHeaders.IDEMPOTENCY_KEY, KEY.toString());
+    return request;
+  }
+
+  private static AdminContext context() {
+    return new AdminContext("operator-1", AdminRole.TRADER, UUID.randomUUID(), "trace-1");
+  }
+}


## `test(settlement): guard candidate rejections`

diff --git a/src/test/java/com/sportsbook/admin/api/SettlementCandidateRejectionControllerTest.java b/src/test/java/com/sportsbook/admin/api/SettlementCandidateRejectionControllerTest.java
new file mode 100644
index 0000000..cc7e9cd
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/api/SettlementCandidateRejectionControllerTest.java
@@ -0,0 +1,68 @@
+package com.sportsbook.admin.api;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.admin.audit.AdminAction;
+import com.sportsbook.admin.audit.Audited;
+import com.sportsbook.admin.client.SettlementCandidateReceipt;
+import com.sportsbook.admin.client.SettlementClient;
+import com.sportsbook.admin.client.SettlementRejectionPayload;
+import com.sportsbook.admin.context.AdminContext;
+import com.sportsbook.admin.security.AdminRole;
+import jakarta.servlet.http.HttpServletRequest;
+import java.lang.reflect.Method;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.mock.web.MockHttpServletRequest;
+import org.springframework.security.access.prepost.PreAuthorize;
+import org.springframework.web.bind.annotation.PostMapping;
+
+class SettlementCandidateRejectionControllerTest {
+
+  private static final UUID CANDIDATE = UUID.fromString("018f0000-0000-7000-8000-000000000174");
+  private static final UUID KEY = UUID.fromString("018f0000-0000-7000-8000-000000000175");
+
+  @Test
+  void delegatesGuardsAndAuditsTheTypedRejection() throws NoSuchMethodException {
+    SettlementClient settlements = mock(SettlementClient.class);
+    SettlementRejectionPayload body = new SettlementRejectionPayload("bad result");
+    SettlementCandidateReceipt receipt =
+        new SettlementCandidateReceipt(
+            KEY, SettlementCandidateReceipt.Outcome.CANDIDATE_REJECTED, false);
+    when(settlements.rejectCandidate(CANDIDATE, KEY, body)).thenReturn(receipt);
+
+    SettlementCandidateReceipt result =
+        new SettlementCandidateCommandController(settlements)
+            .reject(CANDIDATE, body, context(), requestWithKey());
+
+    assertThat(result).isSameAs(receipt);
+    verify(settlements).rejectCandidate(CANDIDATE, KEY, body);
+    Method method =
+        SettlementCandidateCommandController.class.getMethod(
+            "reject",
+            UUID.class,
+            SettlementRejectionPayload.class,
+            AdminContext.class,
+            HttpServletRequest.class);
+    assertThat(method.getAnnotation(PostMapping.class).value()).containsExactly("/reject");
+    assertThat(method.getAnnotation(PreAuthorize.class).value())
+        .isEqualTo("hasAnyRole('ADMIN','TRADER')");
+    Audited audited = method.getAnnotation(Audited.class);
+    assertThat(audited.action()).isEqualTo(AdminAction.RESULT_CANDIDATE_REJECT);
+    assertThat(audited.target()).isEqualTo("#candidateId");
+    assertThat(audited.reason()).isEqualTo("#body.reason()");
+  }
+
+  private static MockHttpServletRequest requestWithKey() {
+    MockHttpServletRequest request = new MockHttpServletRequest();
+    request.addHeader(AdminRequestHeaders.IDEMPOTENCY_KEY, KEY.toString());
+    return request;
+  }
+
+  private static AdminContext context() {
+    return new AdminContext("operator-1", AdminRole.ADMIN, UUID.randomUUID(), "trace-1");
+  }
+}
