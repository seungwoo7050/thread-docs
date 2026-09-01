# 멱등·감사 가능한 시장 상태 전이

## `feat(odds): normalize market action reasons`

diff --git a/src/main/java/com/sportsbook/admin/client/MarketStatusPayload.java b/src/main/java/com/sportsbook/admin/client/MarketStatusPayload.java
new file mode 100644
index 0000000..70883c3
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/client/MarketStatusPayload.java
@@ -0,0 +1,16 @@
+package com.sportsbook.admin.client;
+
+import java.util.Objects;
+
+public record MarketStatusPayload(String reason) {
+
+  public static final int MAX_REASON_LENGTH = 256;
+
+  public MarketStatusPayload {
+    Objects.requireNonNull(reason, "reason");
+    reason = reason.trim();
+    if (reason.isEmpty() || reason.length() > MAX_REASON_LENGTH) {
+      throw new IllegalArgumentException("Market status reason must contain 1 to 256 characters");
+    }
+  }
+}


## `test(odds): enforce market reason bounds`

diff --git a/src/test/java/com/sportsbook/admin/client/MarketStatusPayloadTest.java b/src/test/java/com/sportsbook/admin/client/MarketStatusPayloadTest.java
new file mode 100644
index 0000000..3d4c386
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/client/MarketStatusPayloadTest.java
@@ -0,0 +1,26 @@
+package com.sportsbook.admin.client;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import org.junit.jupiter.api.Test;
+
+class MarketStatusPayloadTest {
+
+  @Test
+  void trimsAReasonWithinTheProviderLimit() {
+    assertThat(new MarketStatusPayload("  feed investigation  ").reason())
+        .isEqualTo("feed investigation");
+    assertThat(new MarketStatusPayload("r".repeat(256)).reason()).hasSize(256);
+  }
+
+  @Test
+  void rejectsMissingBlankAndOversizedReasons() {
+    assertThatThrownBy(() -> new MarketStatusPayload(null))
+        .isInstanceOf(NullPointerException.class);
+    assertThatThrownBy(() -> new MarketStatusPayload("   "))
+        .isInstanceOf(IllegalArgumentException.class);
+    assertThatThrownBy(() -> new MarketStatusPayload("r".repeat(257)))
+        .isInstanceOf(IllegalArgumentException.class);
+  }
+}


## `feat(odds): delegate all market actions`

diff --git a/src/main/java/com/sportsbook/admin/client/DownstreamHeaders.java b/src/main/java/com/sportsbook/admin/client/DownstreamHeaders.java
index 02cdbfb..5e924f8 100644
--- a/src/main/java/com/sportsbook/admin/client/DownstreamHeaders.java
+++ b/src/main/java/com/sportsbook/admin/client/DownstreamHeaders.java
@@ -6,6 +6,7 @@ public final class DownstreamHeaders {
   public static final String INTERNAL_API_KEY = "X-Internal-Api-Key";
   public static final String SERVICE_NAME = "X-Service-Name";
   public static final String API_KEY = "X-API-Key";
+  public static final String ADMIN_ACTION_ID = "X-Admin-Action-Id";
   public static final String ADMIN_API = "admin-api";
 
   private DownstreamHeaders() {}
diff --git a/src/main/java/com/sportsbook/admin/client/MarketClient.java b/src/main/java/com/sportsbook/admin/client/MarketClient.java
new file mode 100644
index 0000000..a1feb25
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/client/MarketClient.java
@@ -0,0 +1,60 @@
+package com.sportsbook.admin.client;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import java.util.UUID;
+import org.springframework.beans.factory.annotation.Qualifier;
+import org.springframework.http.HttpStatus;
+import org.springframework.http.MediaType;
+import org.springframework.stereotype.Component;
+import org.springframework.web.client.RestClient;
+
+@Component
+public class MarketClient {
+
+  private final RestClient http;
+  private final DownstreamFailureMapper failures = new DownstreamFailureMapper();
+
+  public MarketClient(@Qualifier("oddsRestClient") RestClient http) {
+    this.http = http;
+  }
+
+  public void changeStatus(
+      UUID eventId,
+      UUID marketId,
+      Action action,
+      MarketStatusPayload body,
+      IdempotencyKey idempotencyKey,
+      UUID adminActionId) {
+    var response =
+        failures.execute(
+            () ->
+                http.post()
+                    .uri(
+                        builder ->
+                            builder
+                                .pathSegment("internal", "v1", "events")
+                                .pathSegment(eventId.toString(), "markets", marketId.toString())
+                                .pathSegment(action.wireValue)
+                                .build())
+                    .header("Idempotency-Key", idempotencyKey.value())
+                    .header(DownstreamHeaders.ADMIN_ACTION_ID, adminActionId.toString())
+                    .contentType(MediaType.APPLICATION_JSON)
+                    .body(body)
+                    .retrieve()
+                    .toEntity(byte[].class));
+    DownstreamContract.requireEmpty(
+        response, HttpStatus.ACCEPTED, "Odds market action must return empty HTTP 202");
+  }
+
+  public enum Action {
+    SUSPEND("suspend"),
+    CLOSE("close"),
+    REOPEN("reopen");
+
+    private final String wireValue;
+
+    Action(String wireValue) {
+      this.wireValue = wireValue;
+    }
+  }
+}


## `test(odds): preserve four market headers`

diff --git a/src/test/java/com/sportsbook/admin/client/MarketClientHeadersTest.java b/src/test/java/com/sportsbook/admin/client/MarketClientHeadersTest.java
new file mode 100644
index 0000000..143fead
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/client/MarketClientHeadersTest.java
@@ -0,0 +1,61 @@
+package com.sportsbook.admin.client;
+
+import static org.springframework.http.HttpMethod.POST;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.content;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.header;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.method;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.requestTo;
+import static org.springframework.test.web.client.response.MockRestResponseCreators.withStatus;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.http.HttpStatus;
+import org.springframework.test.web.client.MockRestServiceServer;
+import org.springframework.web.client.RestClient;
+
+class MarketClientHeadersTest {
+
+  @Test
+  void preservesTheLogicalKeyWhileEachAttemptUsesItsOwnActionId() {
+    UUID eventId = UUID.fromString("018f0000-0000-7000-8000-000000000131");
+    UUID marketId = UUID.fromString("018f0000-0000-7000-8000-000000000132");
+    UUID firstAction = UUID.fromString("018f0000-0000-7000-8000-000000000133");
+    UUID retryAction = UUID.fromString("018f0000-0000-7000-8000-000000000134");
+    RestClient.Builder builder =
+        RestClient.builder()
+            .baseUrl("http://odds.test")
+            .defaultHeader(DownstreamHeaders.INTERNAL_SERVICE, DownstreamHeaders.ADMIN_API)
+            .defaultHeader(DownstreamHeaders.INTERNAL_API_KEY, ClientIsolationFixture.ODDS);
+    MockRestServiceServer server = MockRestServiceServer.bindTo(builder).build();
+    expectClose(server, eventId, marketId, firstAction);
+    expectClose(server, eventId, marketId, retryAction);
+    MarketClient client = new MarketClient(builder.build());
+    MarketStatusPayload reason = new MarketStatusPayload("  feed investigation  ");
+    IdempotencyKey key = IdempotencyKey.of("market close retry 01");
+
+    client.changeStatus(eventId, marketId, MarketClient.Action.CLOSE, reason, key, firstAction);
+    client.changeStatus(eventId, marketId, MarketClient.Action.CLOSE, reason, key, retryAction);
+
+    server.verify();
+  }
+
+  private static void expectClose(
+      MockRestServiceServer server, UUID eventId, UUID marketId, UUID actionId) {
+    server
+        .expect(
+            requestTo(
+                "http://odds.test/internal/v1/events/"
+                    + eventId
+                    + "/markets/"
+                    + marketId
+                    + "/close"))
+        .andExpect(method(POST))
+        .andExpect(header(DownstreamHeaders.INTERNAL_SERVICE, "admin-api"))
+        .andExpect(header(DownstreamHeaders.INTERNAL_API_KEY, ClientIsolationFixture.ODDS))
+        .andExpect(header("Idempotency-Key", "market close retry 01"))
+        .andExpect(header(DownstreamHeaders.ADMIN_ACTION_ID, actionId.toString()))
+        .andExpect(content().json("{\"reason\":\"feed investigation\"}"))
+        .andRespond(withStatus(HttpStatus.ACCEPTED));
+  }
+}


## `feat(odds): expose market suspension`

diff --git a/src/main/java/com/sportsbook/admin/api/MarketAdminController.java b/src/main/java/com/sportsbook/admin/api/MarketAdminController.java
new file mode 100644
index 0000000..e6b63d7
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/api/MarketAdminController.java
@@ -0,0 +1,61 @@
+package com.sportsbook.admin.api;
+
+import com.sportsbook.admin.audit.AdminAction;
+import com.sportsbook.admin.audit.Audited;
+import com.sportsbook.admin.client.MarketClient;
+import com.sportsbook.admin.client.MarketStatusPayload;
+import com.sportsbook.admin.context.AdminContext;
+import jakarta.servlet.http.HttpServletRequest;
+import java.util.UUID;
+import org.springframework.http.HttpStatus;
+import org.springframework.security.access.prepost.PreAuthorize;
+import org.springframework.web.bind.annotation.PathVariable;
+import org.springframework.web.bind.annotation.PostMapping;
+import org.springframework.web.bind.annotation.RequestBody;
+import org.springframework.web.bind.annotation.RequestMapping;
+import org.springframework.web.bind.annotation.ResponseStatus;
+import org.springframework.web.bind.annotation.RestController;
+
+@RestController
+@RequestMapping("/admin/v1/events/{eventId}/markets/{marketId}")
+public class MarketAdminController {
+
+  private final MarketClient markets;
+
+  public MarketAdminController(MarketClient markets) {
+    this.markets = markets;
+  }
+
+  @PostMapping("/suspend")
+  @ResponseStatus(HttpStatus.ACCEPTED)
+  @PreAuthorize("hasAnyRole('ADMIN','TRADER')")
+  @Audited(
+      action = AdminAction.MARKET_SUSPEND,
+      target = "#eventId + '/' + #marketId",
+      reason = "#body.reason()")
+  public void suspend(
+      @PathVariable UUID eventId,
+      @PathVariable UUID marketId,
+      @RequestBody MarketStatusPayload body,
+      AdminContext context,
+      HttpServletRequest servletRequest) {
+    changeStatus(
+        eventId, marketId, MarketClient.Action.SUSPEND, body, context, servletRequest);
+  }
+
+  private void changeStatus(
+      UUID eventId,
+      UUID marketId,
+      MarketClient.Action action,
+      MarketStatusPayload body,
+      AdminContext context,
+      HttpServletRequest request) {
+    markets.changeStatus(
+        eventId,
+        marketId,
+        action,
+        body,
+        AdminRequestHeaders.requireIdempotencyKey(request),
+        context.actionId());
+  }
+}


## `test(odds): guard and audit suspension`

diff --git a/src/test/java/com/sportsbook/admin/api/MarketAdminControllerTest.java b/src/test/java/com/sportsbook/admin/api/MarketAdminControllerTest.java
new file mode 100644
index 0000000..52f1fd9
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/api/MarketAdminControllerTest.java
@@ -0,0 +1,80 @@
+package com.sportsbook.admin.api;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verify;
+
+import com.sportsbook.admin.audit.AdminAction;
+import com.sportsbook.admin.audit.Audited;
+import com.sportsbook.admin.client.MarketClient;
+import com.sportsbook.admin.client.MarketStatusPayload;
+import com.sportsbook.admin.context.AdminContext;
+import com.sportsbook.admin.security.AdminRole;
+import com.sportsbook.protocol.value.IdempotencyKey;
+import jakarta.servlet.http.HttpServletRequest;
+import java.lang.reflect.Method;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.http.HttpStatus;
+import org.springframework.mock.web.MockHttpServletRequest;
+import org.springframework.security.access.prepost.PreAuthorize;
+import org.springframework.web.bind.annotation.ResponseStatus;
+
+class MarketAdminControllerTest {
+
+  private static final UUID EVENT =
+      UUID.fromString("018f0000-0000-7000-8000-000000000135");
+  private static final UUID MARKET =
+      UUID.fromString("018f0000-0000-7000-8000-000000000136");
+  private static final UUID ACTION =
+      UUID.fromString("018f0000-0000-7000-8000-000000000137");
+
+  @Test
+  void delegatesSuspensionWithTheAttemptIdentity() {
+    MarketClient markets = mock(MarketClient.class);
+    MarketStatusPayload body = new MarketStatusPayload("feed investigation");
+    AdminContext context = new AdminContext("operator-1", AdminRole.TRADER, ACTION, "trace-1");
+    MockHttpServletRequest request = requestWithKey();
+
+    new MarketAdminController(markets).suspend(EVENT, MARKET, body, context, request);
+
+    verify(markets)
+        .changeStatus(
+            EVENT,
+            MARKET,
+            MarketClient.Action.SUSPEND,
+            body,
+            IdempotencyKey.of("market action 01"),
+            ACTION);
+  }
+
+  @Test
+  void guardsAndAuditsSuspension() throws NoSuchMethodException {
+    Method method = actionMethod("suspend");
+
+    assertThat(method.getAnnotation(PreAuthorize.class).value())
+        .isEqualTo("hasAnyRole('ADMIN','TRADER')");
+    assertThat(method.getAnnotation(ResponseStatus.class).value())
+        .isEqualTo(HttpStatus.ACCEPTED);
+    Audited audited = method.getAnnotation(Audited.class);
+    assertThat(audited.action()).isEqualTo(AdminAction.MARKET_SUSPEND);
+    assertThat(audited.target()).isEqualTo("#eventId + '/' + #marketId");
+    assertThat(audited.reason()).isEqualTo("#body.reason()");
+  }
+
+  private static Method actionMethod(String name) throws NoSuchMethodException {
+    return MarketAdminController.class.getMethod(
+        name,
+        UUID.class,
+        UUID.class,
+        MarketStatusPayload.class,
+        AdminContext.class,
+        HttpServletRequest.class);
+  }
+
+  private static MockHttpServletRequest requestWithKey() {
+    MockHttpServletRequest request = new MockHttpServletRequest();
+    request.addHeader(AdminRequestHeaders.IDEMPOTENCY_KEY, "market action 01");
+    return request;
+  }
+}


## `feat(odds): expose market closure`

diff --git a/src/main/java/com/sportsbook/admin/api/MarketAdminController.java b/src/main/java/com/sportsbook/admin/api/MarketAdminController.java
index e6b63d7..0f168ad 100644
--- a/src/main/java/com/sportsbook/admin/api/MarketAdminController.java
+++ b/src/main/java/com/sportsbook/admin/api/MarketAdminController.java
@@ -43,6 +43,22 @@ public class MarketAdminController {
         eventId, marketId, MarketClient.Action.SUSPEND, body, context, servletRequest);
   }
 
+  @PostMapping("/close")
+  @ResponseStatus(HttpStatus.ACCEPTED)
+  @PreAuthorize("hasAnyRole('ADMIN','TRADER')")
+  @Audited(
+      action = AdminAction.MARKET_CLOSE,
+      target = "#eventId + '/' + #marketId",
+      reason = "#body.reason()")
+  public void close(
+      @PathVariable UUID eventId,
+      @PathVariable UUID marketId,
+      @RequestBody MarketStatusPayload body,
+      AdminContext context,
+      HttpServletRequest servletRequest) {
+    changeStatus(eventId, marketId, MarketClient.Action.CLOSE, body, context, servletRequest);
+  }
+
   private void changeStatus(
       UUID eventId,
       UUID marketId,


## `test(odds): guard and audit closure`

diff --git a/src/test/java/com/sportsbook/admin/api/MarketAdminControllerTest.java b/src/test/java/com/sportsbook/admin/api/MarketAdminControllerTest.java
index 52f1fd9..52fdb30 100644
--- a/src/test/java/com/sportsbook/admin/api/MarketAdminControllerTest.java
+++ b/src/test/java/com/sportsbook/admin/api/MarketAdminControllerTest.java
@@ -62,6 +62,32 @@ class MarketAdminControllerTest {
     assertThat(audited.reason()).isEqualTo("#body.reason()");
   }
 
+  @Test
+  void delegatesGuardsAndAuditsClosure() throws NoSuchMethodException {
+    MarketClient markets = mock(MarketClient.class);
+    MarketStatusPayload body = new MarketStatusPayload("event completed");
+    AdminContext context = new AdminContext("operator-1", AdminRole.ADMIN, ACTION, "trace-1");
+
+    new MarketAdminController(markets)
+        .close(EVENT, MARKET, body, context, requestWithKey());
+
+    verify(markets)
+        .changeStatus(
+            EVENT,
+            MARKET,
+            MarketClient.Action.CLOSE,
+            body,
+            IdempotencyKey.of("market action 01"),
+            ACTION);
+    Method method = actionMethod("close");
+    assertThat(method.getAnnotation(PreAuthorize.class).value())
+        .isEqualTo("hasAnyRole('ADMIN','TRADER')");
+    assertThat(method.getAnnotation(ResponseStatus.class).value())
+        .isEqualTo(HttpStatus.ACCEPTED);
+    assertThat(method.getAnnotation(Audited.class).action())
+        .isEqualTo(AdminAction.MARKET_CLOSE);
+  }
+
   private static Method actionMethod(String name) throws NoSuchMethodException {
     return MarketAdminController.class.getMethod(
         name,


## `feat(odds): expose market reopening`

diff --git a/src/main/java/com/sportsbook/admin/api/MarketAdminController.java b/src/main/java/com/sportsbook/admin/api/MarketAdminController.java
index 0f168ad..0867bcb 100644
--- a/src/main/java/com/sportsbook/admin/api/MarketAdminController.java
+++ b/src/main/java/com/sportsbook/admin/api/MarketAdminController.java
@@ -39,8 +39,7 @@ public class MarketAdminController {
       @RequestBody MarketStatusPayload body,
       AdminContext context,
       HttpServletRequest servletRequest) {
-    changeStatus(
-        eventId, marketId, MarketClient.Action.SUSPEND, body, context, servletRequest);
+    changeStatus(eventId, marketId, MarketClient.Action.SUSPEND, body, context, servletRequest);
   }
 
   @PostMapping("/close")
@@ -59,6 +58,22 @@ public class MarketAdminController {
     changeStatus(eventId, marketId, MarketClient.Action.CLOSE, body, context, servletRequest);
   }
 
+  @PostMapping("/reopen")
+  @ResponseStatus(HttpStatus.ACCEPTED)
+  @PreAuthorize("hasAnyRole('ADMIN','TRADER')")
+  @Audited(
+      action = AdminAction.MARKET_REOPEN,
+      target = "#eventId + '/' + #marketId",
+      reason = "#body.reason()")
+  public void reopen(
+      @PathVariable UUID eventId,
+      @PathVariable UUID marketId,
+      @RequestBody MarketStatusPayload body,
+      AdminContext context,
+      HttpServletRequest servletRequest) {
+    changeStatus(eventId, marketId, MarketClient.Action.REOPEN, body, context, servletRequest);
+  }
+
   private void changeStatus(
       UUID eventId,
       UUID marketId,


## `test(odds): guard and audit reopening`

diff --git a/src/test/java/com/sportsbook/admin/api/MarketAdminControllerTest.java b/src/test/java/com/sportsbook/admin/api/MarketAdminControllerTest.java
index 52fdb30..8a371df 100644
--- a/src/test/java/com/sportsbook/admin/api/MarketAdminControllerTest.java
+++ b/src/test/java/com/sportsbook/admin/api/MarketAdminControllerTest.java
@@ -22,12 +22,9 @@ import org.springframework.web.bind.annotation.ResponseStatus;
 
 class MarketAdminControllerTest {
 
-  private static final UUID EVENT =
-      UUID.fromString("018f0000-0000-7000-8000-000000000135");
-  private static final UUID MARKET =
-      UUID.fromString("018f0000-0000-7000-8000-000000000136");
-  private static final UUID ACTION =
-      UUID.fromString("018f0000-0000-7000-8000-000000000137");
+  private static final UUID EVENT = UUID.fromString("018f0000-0000-7000-8000-000000000135");
+  private static final UUID MARKET = UUID.fromString("018f0000-0000-7000-8000-000000000136");
+  private static final UUID ACTION = UUID.fromString("018f0000-0000-7000-8000-000000000137");
 
   @Test
   void delegatesSuspensionWithTheAttemptIdentity() {
@@ -54,8 +51,7 @@ class MarketAdminControllerTest {
 
     assertThat(method.getAnnotation(PreAuthorize.class).value())
         .isEqualTo("hasAnyRole('ADMIN','TRADER')");
-    assertThat(method.getAnnotation(ResponseStatus.class).value())
-        .isEqualTo(HttpStatus.ACCEPTED);
+    assertThat(method.getAnnotation(ResponseStatus.class).value()).isEqualTo(HttpStatus.ACCEPTED);
     Audited audited = method.getAnnotation(Audited.class);
     assertThat(audited.action()).isEqualTo(AdminAction.MARKET_SUSPEND);
     assertThat(audited.target()).isEqualTo("#eventId + '/' + #marketId");
@@ -68,8 +64,7 @@ class MarketAdminControllerTest {
     MarketStatusPayload body = new MarketStatusPayload("event completed");
     AdminContext context = new AdminContext("operator-1", AdminRole.ADMIN, ACTION, "trace-1");
 
-    new MarketAdminController(markets)
-        .close(EVENT, MARKET, body, context, requestWithKey());
+    new MarketAdminController(markets).close(EVENT, MARKET, body, context, requestWithKey());
 
     verify(markets)
         .changeStatus(
@@ -82,10 +77,31 @@ class MarketAdminControllerTest {
     Method method = actionMethod("close");
     assertThat(method.getAnnotation(PreAuthorize.class).value())
         .isEqualTo("hasAnyRole('ADMIN','TRADER')");
-    assertThat(method.getAnnotation(ResponseStatus.class).value())
-        .isEqualTo(HttpStatus.ACCEPTED);
-    assertThat(method.getAnnotation(Audited.class).action())
-        .isEqualTo(AdminAction.MARKET_CLOSE);
+    assertThat(method.getAnnotation(ResponseStatus.class).value()).isEqualTo(HttpStatus.ACCEPTED);
+    assertThat(method.getAnnotation(Audited.class).action()).isEqualTo(AdminAction.MARKET_CLOSE);
+  }
+
+  @Test
+  void delegatesGuardsAndAuditsReopening() throws NoSuchMethodException {
+    MarketClient markets = mock(MarketClient.class);
+    MarketStatusPayload body = new MarketStatusPayload("feed corrected");
+    AdminContext context = new AdminContext("operator-1", AdminRole.ADMIN, ACTION, "trace-1");
+
+    new MarketAdminController(markets).reopen(EVENT, MARKET, body, context, requestWithKey());
+
+    verify(markets)
+        .changeStatus(
+            EVENT,
+            MARKET,
+            MarketClient.Action.REOPEN,
+            body,
+            IdempotencyKey.of("market action 01"),
+            ACTION);
+    Method method = actionMethod("reopen");
+    assertThat(method.getAnnotation(PreAuthorize.class).value())
+        .isEqualTo("hasAnyRole('ADMIN','TRADER')");
+    assertThat(method.getAnnotation(ResponseStatus.class).value()).isEqualTo(HttpStatus.ACCEPTED);
+    assertThat(method.getAnnotation(Audited.class).action()).isEqualTo(AdminAction.MARKET_REOPEN);
   }
 
   private static Method actionMethod(String name) throws NoSuchMethodException {
