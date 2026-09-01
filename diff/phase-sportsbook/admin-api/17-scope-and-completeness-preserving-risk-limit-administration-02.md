## `test(risk): clear the exact scoped target`

diff --git a/src/test/java/com/sportsbook/admin/client/RiskClientDeleteTest.java b/src/test/java/com/sportsbook/admin/client/RiskClientDeleteTest.java
new file mode 100644
index 0000000..1cb6cc7
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/client/RiskClientDeleteTest.java
@@ -0,0 +1,49 @@
+package com.sportsbook.admin.client;
+
+import static org.springframework.http.HttpMethod.DELETE;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.header;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.method;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.requestTo;
+import static org.springframework.test.web.client.response.MockRestResponseCreators.withNoContent;
+
+import com.sportsbook.protocol.value.Currency;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.test.web.client.MockRestServiceServer;
+import org.springframework.web.client.RestClient;
+
+class RiskClientDeleteTest {
+
+  @Test
+  void sendsCurrencyOnlyForMonetaryTargets() {
+    UUID userId = UUID.fromString("018f0000-0000-7000-8000-000000000125");
+    RestClient.Builder builder =
+        RestClient.builder()
+            .baseUrl("http://risk.test")
+            .defaultHeader(DownstreamHeaders.INTERNAL_SERVICE, DownstreamHeaders.ADMIN_API)
+            .defaultHeader(DownstreamHeaders.INTERNAL_API_KEY, ClientIsolationFixture.RISK);
+    MockRestServiceServer server = MockRestServiceServer.bindTo(builder).build();
+    server
+        .expect(
+            requestTo(
+                "http://risk.test/internal/v1/risk/limits/"
+                    + userId
+                    + "/STAKE_WEEKLY?currency=USD"))
+        .andExpect(method(DELETE))
+        .andExpect(header(DownstreamHeaders.INTERNAL_SERVICE, "admin-api"))
+        .andExpect(header(DownstreamHeaders.INTERNAL_API_KEY, ClientIsolationFixture.RISK))
+        .andRespond(withNoContent());
+    server
+        .expect(
+            requestTo(
+                "http://risk.test/internal/v1/risk/limits/" + userId + "/SELECTIONS_PER_MINUTE"))
+        .andExpect(method(DELETE))
+        .andRespond(withNoContent());
+    RiskClient client = new RiskClient(builder.build());
+
+    client.clearLimit(userId, RiskLimitType.STAKE_WEEKLY, Currency.USD);
+    client.clearLimit(userId, RiskLimitType.SELECTIONS_PER_MINUTE, null);
+
+    server.verify();
+  }
+}


## `feat(risk): expose limit administration`

diff --git a/src/main/java/com/sportsbook/admin/api/RiskAdminController.java b/src/main/java/com/sportsbook/admin/api/RiskAdminController.java
new file mode 100644
index 0000000..5325f02
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/api/RiskAdminController.java
@@ -0,0 +1,64 @@
+package com.sportsbook.admin.api;
+
+import com.sportsbook.admin.audit.AdminAction;
+import com.sportsbook.admin.audit.Audited;
+import com.sportsbook.admin.client.RiskClient;
+import com.sportsbook.admin.client.RiskLimitPayload;
+import com.sportsbook.admin.client.RiskLimitType;
+import com.sportsbook.admin.client.RiskLimitsResponse;
+import com.sportsbook.admin.context.AdminContext;
+import com.sportsbook.protocol.value.Currency;
+import jakarta.validation.Valid;
+import java.util.UUID;
+import org.springframework.http.HttpStatus;
+import org.springframework.security.access.prepost.PreAuthorize;
+import org.springframework.web.bind.annotation.DeleteMapping;
+import org.springframework.web.bind.annotation.GetMapping;
+import org.springframework.web.bind.annotation.PatchMapping;
+import org.springframework.web.bind.annotation.PathVariable;
+import org.springframework.web.bind.annotation.RequestBody;
+import org.springframework.web.bind.annotation.RequestMapping;
+import org.springframework.web.bind.annotation.RequestParam;
+import org.springframework.web.bind.annotation.ResponseStatus;
+import org.springframework.web.bind.annotation.RestController;
+
+@RestController
+@RequestMapping("/admin/v1/risk/users/{userId}/limits")
+public class RiskAdminController {
+
+  private final RiskClient risk;
+
+  public RiskAdminController(RiskClient risk) {
+    this.risk = risk;
+  }
+
+  @GetMapping
+  @PreAuthorize("hasAnyRole('ADMIN','TRADER','CS','READONLY')")
+  public RiskLimitsResponse getLimits(@PathVariable UUID userId) {
+    return risk.getLimits(userId);
+  }
+
+  @PatchMapping
+  @ResponseStatus(HttpStatus.NO_CONTENT)
+  @PreAuthorize("hasAnyRole('ADMIN','TRADER')")
+  @Audited(action = AdminAction.RISK_LIMIT_UPDATE, target = "#userId + ':' + #body.type()")
+  public void setLimit(
+      @PathVariable UUID userId, @Valid @RequestBody RiskLimitPayload body, AdminContext context) {
+    risk.setLimit(userId, body);
+  }
+
+  @DeleteMapping("/{type}")
+  @ResponseStatus(HttpStatus.NO_CONTENT)
+  @PreAuthorize("hasAnyRole('ADMIN','TRADER')")
+  @Audited(action = AdminAction.RISK_LIMIT_CLEAR, target = "#userId + ':' + #type")
+  public void clearLimit(
+      @PathVariable UUID userId,
+      @PathVariable RiskLimitType type,
+      @RequestParam(required = false) Currency currency,
+      AdminContext context) {
+    if (type.requiresCurrency() != (currency != null)) {
+      throw new AdminRequestException("Risk limit currency scope does not match its type");
+    }
+    risk.clearLimit(userId, type, currency);
+  }
+}


## `test(risk): delegate every limit operation`

diff --git a/src/test/java/com/sportsbook/admin/api/RiskAdminControllerDelegationTest.java b/src/test/java/com/sportsbook/admin/api/RiskAdminControllerDelegationTest.java
new file mode 100644
index 0000000..1a389bd
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/api/RiskAdminControllerDelegationTest.java
@@ -0,0 +1,44 @@
+package com.sportsbook.admin.api;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.admin.client.RiskClient;
+import com.sportsbook.admin.client.RiskLimitPayload;
+import com.sportsbook.admin.client.RiskLimitType;
+import com.sportsbook.admin.client.RiskLimitsResponse;
+import com.sportsbook.admin.context.AdminContext;
+import com.sportsbook.admin.security.AdminRole;
+import com.sportsbook.protocol.value.Currency;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class RiskAdminControllerDelegationTest {
+
+  @Test
+  void delegatesEveryLimitOperationWithoutChangingItsScope() {
+    RiskClient risk = mock(RiskClient.class);
+    UUID userId = UUID.fromString("018f0000-0000-7000-8000-000000000126");
+    RiskLimitsResponse snapshot = new RiskLimitsResponse(userId, List.of());
+    AdminContext context =
+        new AdminContext(
+            "operator-1",
+            AdminRole.TRADER,
+            UUID.fromString("018f0000-0000-7000-8000-000000000127"),
+            "trace-1");
+    RiskLimitPayload update = new RiskLimitPayload(RiskLimitType.STAKE_DAILY, Currency.KRW, 750L);
+    when(risk.getLimits(userId)).thenReturn(snapshot);
+    RiskAdminController controller = new RiskAdminController(risk);
+
+    assertThat(controller.getLimits(userId)).isSameAs(snapshot);
+    controller.setLimit(userId, update, context);
+    controller.clearLimit(userId, RiskLimitType.STAKE_WEEKLY, Currency.USD, context);
+
+    verify(risk).getLimits(userId);
+    verify(risk).setLimit(userId, update);
+    verify(risk).clearLimit(userId, RiskLimitType.STAKE_WEEKLY, Currency.USD);
+  }
+}


## `test(risk): reject invalid clear scopes`

diff --git a/src/test/java/com/sportsbook/admin/api/RiskAdminControllerScopeTest.java b/src/test/java/com/sportsbook/admin/api/RiskAdminControllerScopeTest.java
new file mode 100644
index 0000000..5f3c374
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/api/RiskAdminControllerScopeTest.java
@@ -0,0 +1,35 @@
+package com.sportsbook.admin.api;
+
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verifyNoInteractions;
+
+import com.sportsbook.admin.client.RiskClient;
+import com.sportsbook.admin.client.RiskLimitType;
+import com.sportsbook.admin.context.AdminContext;
+import com.sportsbook.admin.security.AdminRole;
+import com.sportsbook.protocol.value.Currency;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class RiskAdminControllerScopeTest {
+
+  private static final UUID USER = UUID.fromString("018f0000-0000-7000-8000-000000000179");
+  private static final AdminContext CONTEXT =
+      new AdminContext("operator-1", AdminRole.TRADER, UUID.randomUUID(), "trace-1");
+
+  @Test
+  void rejectsMissingAndForbiddenCurrencyScopesBeforeCallingRisk() {
+    RiskClient risk = mock(RiskClient.class);
+    RiskAdminController controller = new RiskAdminController(risk);
+
+    assertThatThrownBy(() -> controller.clearLimit(USER, RiskLimitType.STAKE_DAILY, null, CONTEXT))
+        .isInstanceOf(AdminRequestException.class);
+    assertThatThrownBy(
+            () ->
+                controller.clearLimit(
+                    USER, RiskLimitType.SELECTIONS_PER_MINUTE, Currency.KRW, CONTEXT))
+        .isInstanceOf(AdminRequestException.class);
+    verifyNoInteractions(risk);
+  }
+}


## `test(risk): guard and audit limit mutations`

diff --git a/src/test/java/com/sportsbook/admin/api/RiskAdminControllerContractTest.java b/src/test/java/com/sportsbook/admin/api/RiskAdminControllerContractTest.java
new file mode 100644
index 0000000..6e5ba25
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/api/RiskAdminControllerContractTest.java
@@ -0,0 +1,51 @@
+package com.sportsbook.admin.api;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.admin.audit.AdminAction;
+import com.sportsbook.admin.audit.Audited;
+import com.sportsbook.admin.client.RiskLimitPayload;
+import com.sportsbook.admin.client.RiskLimitType;
+import com.sportsbook.admin.context.AdminContext;
+import com.sportsbook.protocol.value.Currency;
+import java.lang.reflect.Method;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.http.HttpStatus;
+import org.springframework.security.access.prepost.PreAuthorize;
+import org.springframework.web.bind.annotation.ResponseStatus;
+
+class RiskAdminControllerContractTest {
+
+  @Test
+  void permitsEveryRoleToReadWithoutAuditingTheRead() throws NoSuchMethodException {
+    Method read = RiskAdminController.class.getMethod("getLimits", UUID.class);
+
+    assertThat(read.getAnnotation(PreAuthorize.class).value())
+        .isEqualTo("hasAnyRole('ADMIN','TRADER','CS','READONLY')");
+    assertThat(read.getAnnotation(Audited.class)).isNull();
+  }
+
+  @Test
+  void restrictsAndAuditsBothMutations() throws NoSuchMethodException {
+    Method update =
+        RiskAdminController.class.getMethod(
+            "setLimit", UUID.class, RiskLimitPayload.class, AdminContext.class);
+    Method clear =
+        RiskAdminController.class.getMethod(
+            "clearLimit", UUID.class, RiskLimitType.class, Currency.class, AdminContext.class);
+
+    assertMutation(update, AdminAction.RISK_LIMIT_UPDATE);
+    assertMutation(clear, AdminAction.RISK_LIMIT_CLEAR);
+    assertThat(update.getAnnotation(Audited.class).target())
+        .isEqualTo("#userId + ':' + #body.type()");
+    assertThat(clear.getAnnotation(Audited.class).target()).isEqualTo("#userId + ':' + #type");
+  }
+
+  private static void assertMutation(Method method, AdminAction action) {
+    assertThat(method.getAnnotation(PreAuthorize.class).value())
+        .isEqualTo("hasAnyRole('ADMIN','TRADER')");
+    assertThat(method.getAnnotation(ResponseStatus.class).value()).isEqualTo(HttpStatus.NO_CONTENT);
+    assertThat(method.getAnnotation(Audited.class).action()).isEqualTo(action);
+  }
+}


## `test(api): render invalid risk scopes`

diff --git a/src/test/java/com/sportsbook/admin/api/RiskScopeRequestTest.java b/src/test/java/com/sportsbook/admin/api/RiskScopeRequestTest.java
new file mode 100644
index 0000000..36faa6e
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/api/RiskScopeRequestTest.java
@@ -0,0 +1,62 @@
+package com.sportsbook.admin.api;
+
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verifyNoInteractions;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.delete;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.header;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
+
+import com.sportsbook.admin.client.RiskClient;
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
+class RiskScopeRequestTest {
+
+  private static final UUID USER = UUID.fromString("018f0000-0000-7000-8000-000000000181");
+
+  @Test
+  void rendersInvalidDeleteScopesBeforeCallingRisk() throws Exception {
+    RiskClient risk = mock(RiskClient.class);
+    MockMvc mvc =
+        MockMvcBuilders.standaloneSetup(new RiskAdminController(risk))
+            .setCustomArgumentResolvers(new AdminContextArgumentResolver())
+            .setControllerAdvice(new AdminExceptionHandler())
+            .build();
+
+    mvc.perform(delete(path("STAKE_DAILY")).principal(authentication()))
+        .andExpect(status().isBadRequest())
+        .andExpect(jsonPath("$.errorCode").value("VALIDATION_FAILED"))
+        .andExpect(header().exists(AdminContextArgumentResolver.ACTION_ID_HEADER));
+    mvc.perform(
+            delete(path("SELECTIONS_PER_MINUTE"))
+                .queryParam("currency", "KRW")
+                .principal(authentication()))
+        .andExpect(status().isBadRequest())
+        .andExpect(jsonPath("$.errorCode").value("VALIDATION_FAILED"))
+        .andExpect(header().exists(AdminContextArgumentResolver.ACTION_ID_HEADER));
+    verifyNoInteractions(risk);
+  }
+
+  private static String path(String type) {
+    return "/admin/v1/risk/users/" + USER + "/limits/" + type;
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
