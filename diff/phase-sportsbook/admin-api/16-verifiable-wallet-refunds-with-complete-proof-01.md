# 완전한 증거로 검증하는 지갑 환불

## `feat(wallet): define refund credit payload`

diff --git a/src/main/java/com/sportsbook/admin/client/WalletCreditPayload.java b/src/main/java/com/sportsbook/admin/client/WalletCreditPayload.java
new file mode 100644
index 0000000..a6d4cd5
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/client/WalletCreditPayload.java
@@ -0,0 +1,30 @@
+package com.sportsbook.admin.client;
+
+import com.sportsbook.protocol.value.Money;
+import java.util.Objects;
+import java.util.UUID;
+
+public record WalletCreditPayload(UUID userId, Money amount, Source source, Reason reason) {
+
+  public WalletCreditPayload {
+    Objects.requireNonNull(userId, "userId");
+    Objects.requireNonNull(amount, "amount");
+    Objects.requireNonNull(source, "source");
+    Objects.requireNonNull(reason, "reason");
+    if (!amount.isPositive()) {
+      throw new IllegalArgumentException("Refund amount must be strictly positive");
+    }
+  }
+
+  public static WalletCreditPayload refund(UUID userId, Money amount) {
+    return new WalletCreditPayload(userId, amount, Source.HOUSE_POOL, Reason.REFUND);
+  }
+
+  public enum Source {
+    HOUSE_POOL
+  }
+
+  public enum Reason {
+    REFUND
+  }
+}


## `test(wallet): serialize refund meaning exactly`

diff --git a/src/test/java/com/sportsbook/admin/client/WalletCreditPayloadTest.java b/src/test/java/com/sportsbook/admin/client/WalletCreditPayloadTest.java
new file mode 100644
index 0000000..5571d93
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/client/WalletCreditPayloadTest.java
@@ -0,0 +1,34 @@
+package com.sportsbook.admin.client;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.fasterxml.jackson.databind.JsonNode;
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.protocol.value.Money;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class WalletCreditPayloadTest {
+
+  private final ObjectMapper json = new ObjectMapper().findAndRegisterModules();
+
+  @Test
+  void serializesTheCompleteWalletRefundMeaning() throws Exception {
+    UUID userId = UUID.fromString("018f0000-0000-7000-8000-000000000101");
+
+    JsonNode actual =
+        json.readTree(json.writeValueAsBytes(WalletCreditPayload.refund(userId, Money.krw(750))));
+
+    assertThat(actual)
+        .isEqualTo(
+            json.readTree(
+                """
+                {
+                  "userId":"018f0000-0000-7000-8000-000000000101",
+                  "amount":{"amount":750,"currency":"KRW"},
+                  "source":"HOUSE_POOL",
+                  "reason":"REFUND"
+                }
+                """));
+  }
+}


## `feat(wallet): verify complete refund proof`

diff --git a/src/main/java/com/sportsbook/admin/client/WalletOperationProof.java b/src/main/java/com/sportsbook/admin/client/WalletOperationProof.java
new file mode 100644
index 0000000..156c673
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/client/WalletOperationProof.java
@@ -0,0 +1,24 @@
+package com.sportsbook.admin.client;
+
+import java.util.Objects;
+import java.util.UUID;
+
+public final class WalletOperationProof {
+
+  private static final String REFUND_LEDGER_REASON = "BET_REFUND";
+
+  private WalletOperationProof() {}
+
+  public static UUID verifyRefund(WalletCreditPayload request, WalletOperationResponse response) {
+    Objects.requireNonNull(request, "request");
+    if (response == null
+        || response.operationGroupId() == null
+        || !request.userId().equals(response.userId())
+        || !request.amount().equals(response.amount())
+        || !REFUND_LEDGER_REASON.equals(response.reason())
+        || response.at() == null) {
+      throw new DownstreamContractException("complete matching Wallet refund proof");
+    }
+    return response.operationGroupId();
+  }
+}
diff --git a/src/main/java/com/sportsbook/admin/client/WalletOperationResponse.java b/src/main/java/com/sportsbook/admin/client/WalletOperationResponse.java
new file mode 100644
index 0000000..c84405f
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/client/WalletOperationResponse.java
@@ -0,0 +1,8 @@
+package com.sportsbook.admin.client;
+
+import com.sportsbook.protocol.value.Money;
+import java.time.Instant;
+import java.util.UUID;
+
+public record WalletOperationResponse(
+    UUID operationGroupId, UUID userId, Money amount, String reason, Instant at) {}


## `test(wallet): reject incomplete refund proofs`

diff --git a/src/test/java/com/sportsbook/admin/client/WalletOperationProofTest.java b/src/test/java/com/sportsbook/admin/client/WalletOperationProofTest.java
new file mode 100644
index 0000000..ae72953
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/client/WalletOperationProofTest.java
@@ -0,0 +1,65 @@
+package com.sportsbook.admin.client;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.protocol.value.Money;
+import java.time.Instant;
+import java.util.UUID;
+import java.util.stream.Stream;
+import org.junit.jupiter.api.Test;
+import org.junit.jupiter.params.ParameterizedTest;
+import org.junit.jupiter.params.provider.Arguments;
+import org.junit.jupiter.params.provider.MethodSource;
+
+class WalletOperationProofTest {
+
+  private static final UUID USER = UUID.fromString("018f0000-0000-7000-8000-000000000102");
+  private static final UUID GROUP = UUID.fromString("018f0000-0000-7000-8000-000000000103");
+  private static final Money AMOUNT = Money.krw(750);
+  private static final Instant AT = Instant.parse("2026-08-22T01:02:03Z");
+  private static final WalletCreditPayload REQUEST = WalletCreditPayload.refund(USER, AMOUNT);
+
+  @Test
+  void acceptsOnlyTheMatchingAuthoritativeProof() {
+    assertThat(WalletOperationProof.verifyRefund(REQUEST, valid())).isEqualTo(GROUP);
+  }
+
+  @ParameterizedTest(name = "{0}")
+  @MethodSource("invalidProofs")
+  void rejectsMalformedOrMismatchedSuccess(String label, WalletOperationResponse response) {
+    assertThatThrownBy(() -> WalletOperationProof.verifyRefund(REQUEST, response))
+        .isInstanceOf(DownstreamContractException.class)
+        .hasMessageContaining("Wallet refund proof");
+  }
+
+  private static Stream<Arguments> invalidProofs() {
+    return Stream.of(
+        Arguments.of("missing body", null),
+        Arguments.of("missing operation group", response(null, USER, AMOUNT, "BET_REFUND", AT)),
+        Arguments.of("missing user", response(GROUP, null, AMOUNT, "BET_REFUND", AT)),
+        Arguments.of(
+            "wrong user",
+            response(
+                GROUP,
+                UUID.fromString("018f0000-0000-7000-8000-000000000104"),
+                AMOUNT,
+                "BET_REFUND",
+                AT)),
+        Arguments.of("missing amount", response(GROUP, USER, null, "BET_REFUND", AT)),
+        Arguments.of("wrong amount", response(GROUP, USER, Money.krw(751), "BET_REFUND", AT)),
+        Arguments.of("wrong currency", response(GROUP, USER, Money.usd(750), "BET_REFUND", AT)),
+        Arguments.of("request reason echoed", response(GROUP, USER, AMOUNT, "REFUND", AT)),
+        Arguments.of("missing reason", response(GROUP, USER, AMOUNT, null, AT)),
+        Arguments.of("missing timestamp", response(GROUP, USER, AMOUNT, "BET_REFUND", null)));
+  }
+
+  private static WalletOperationResponse valid() {
+    return response(GROUP, USER, AMOUNT, "BET_REFUND", AT);
+  }
+
+  private static WalletOperationResponse response(
+      UUID group, UUID user, Money amount, String reason, Instant at) {
+    return new WalletOperationResponse(group, user, amount, reason, at);
+  }
+}


## `feat(wallet): delegate refunds with exact proof`

diff --git a/src/main/java/com/sportsbook/admin/client/WalletClient.java b/src/main/java/com/sportsbook/admin/client/WalletClient.java
new file mode 100644
index 0000000..3064a0d
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/client/WalletClient.java
@@ -0,0 +1,42 @@
+package com.sportsbook.admin.client;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import java.util.UUID;
+import org.springframework.beans.factory.annotation.Qualifier;
+import org.springframework.http.HttpStatus;
+import org.springframework.http.MediaType;
+import org.springframework.stereotype.Component;
+import org.springframework.web.client.RestClient;
+
+@Component
+public class WalletClient {
+
+  private final RestClient http;
+  private final DownstreamFailureMapper failures = new DownstreamFailureMapper();
+
+  public WalletClient(@Qualifier("walletRestClient") RestClient http) {
+    this.http = http;
+  }
+
+  public UUID refund(UUID userId, Money amount, IdempotencyKey idempotencyKey) {
+    WalletCreditPayload request = WalletCreditPayload.refund(userId, amount);
+    var response =
+        failures.execute(
+            () ->
+                http.post()
+                    .uri("/internal/v1/wallet/transactions/credit")
+                    .header("Idempotency-Key", idempotencyKey.value())
+                    .contentType(MediaType.APPLICATION_JSON)
+                    .body(request)
+                    .retrieve()
+                    .toEntity(WalletOperationResponse.class));
+    WalletOperationResponse proof =
+        DownstreamContract.requireBody(
+            response,
+            HttpStatus.OK,
+            ignored -> true,
+            "Wallet refund must return HTTP 200 with a body");
+    return WalletOperationProof.verifyRefund(request, proof);
+  }
+}


## `test(wallet): provide exact request capture fixture`

diff --git a/src/test/java/com/sportsbook/admin/client/WalletClientExactRequestTest.java b/src/test/java/com/sportsbook/admin/client/WalletClientExactRequestTest.java
new file mode 100644
index 0000000..e5dd313
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/client/WalletClientExactRequestTest.java
@@ -0,0 +1,30 @@
+package com.sportsbook.admin.client;
+
+import com.sun.net.httpserver.HttpServer;
+import java.net.URI;
+import org.springframework.web.client.RestClient;
+
+class WalletClientExactRequestTest {
+
+  private static RestClient walletRestClient(HttpServer server) {
+    DownstreamProperties defaults = ClientIsolationFixture.properties();
+    var properties =
+        new DownstreamProperties(
+            URI.create("http://127.0.0.1:" + server.getAddress().getPort()),
+            defaults.riskBaseUrl(),
+            defaults.oddsFeedBaseUrl(),
+            defaults.settlementBaseUrl(),
+            defaults.connectTimeout(),
+            defaults.readTimeout());
+    return new DownstreamClientConfiguration()
+        .walletRestClient(RestClient.builder(), properties, ClientIsolationFixture.credentials());
+  }
+
+  private record CapturedRequest(
+      String method,
+      String path,
+      String service,
+      String apiKey,
+      String idempotencyKey,
+      byte[] body) {}
+}


## `test(wallet): send exact refund request`

diff --git a/src/test/java/com/sportsbook/admin/client/WalletClientExactRequestTest.java b/src/test/java/com/sportsbook/admin/client/WalletClientExactRequestTest.java
index e5dd313..c2c323e 100644
--- a/src/test/java/com/sportsbook/admin/client/WalletClientExactRequestTest.java
+++ b/src/test/java/com/sportsbook/admin/client/WalletClientExactRequestTest.java
@@ -1,11 +1,82 @@
 package com.sportsbook.admin.client;
 
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.fasterxml.jackson.databind.JsonNode;
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
 import com.sun.net.httpserver.HttpServer;
+import java.net.InetSocketAddress;
 import java.net.URI;
+import java.nio.charset.StandardCharsets;
+import java.util.UUID;
+import java.util.concurrent.atomic.AtomicReference;
+import org.junit.jupiter.api.Test;
 import org.springframework.web.client.RestClient;
 
 class WalletClientExactRequestTest {
 
+  private final ObjectMapper json = new ObjectMapper().findAndRegisterModules();
+
+  @Test
+  void sendsTheRawKeyAndExactRefundRequest() throws Exception {
+    UUID userId = UUID.fromString("018f0000-0000-7000-8000-000000000105");
+    UUID operationGroup = UUID.fromString("018f0000-0000-7000-8000-000000000106");
+    AtomicReference<CapturedRequest> captured = new AtomicReference<>();
+    HttpServer server = HttpServer.create(new InetSocketAddress("127.0.0.1", 0), 0);
+    server.createContext(
+        "/internal/v1/wallet/transactions/credit",
+        exchange -> {
+          captured.set(
+              new CapturedRequest(
+                  exchange.getRequestMethod(),
+                  exchange.getRequestURI().toString(),
+                  exchange.getRequestHeaders().getFirst(DownstreamHeaders.INTERNAL_SERVICE),
+                  exchange.getRequestHeaders().getFirst(DownstreamHeaders.INTERNAL_API_KEY),
+                  exchange.getRequestHeaders().getFirst("Idempotency-Key"),
+                  exchange.getRequestBody().readAllBytes()));
+          byte[] response =
+              ("{\"operationGroupId\":\""
+                      + operationGroup
+                      + "\",\"userId\":\""
+                      + userId
+                      + "\",\"amount\":{\"amount\":750,\"currency\":\"KRW\"},"
+                      + "\"reason\":\"BET_REFUND\",\"at\":\"2026-08-22T01:02:03Z\"}")
+                  .getBytes(StandardCharsets.UTF_8);
+          exchange.getResponseHeaders().set("Content-Type", "application/json");
+          exchange.sendResponseHeaders(200, response.length);
+          exchange.getResponseBody().write(response);
+          exchange.close();
+        });
+    server.start();
+
+    try {
+      RestClient http = walletRestClient(server);
+
+      UUID result =
+          new WalletClient(http).refund(userId, Money.krw(750), IdempotencyKey.of("refund key 01"));
+
+      assertThat(result).isEqualTo(operationGroup);
+      assertThat(captured.get().method()).isEqualTo("POST");
+      assertThat(captured.get().path()).isEqualTo("/internal/v1/wallet/transactions/credit");
+      assertThat(captured.get().service()).isEqualTo("admin-api");
+      assertThat(captured.get().apiKey()).isEqualTo(ClientIsolationFixture.WALLET);
+      assertThat(captured.get().idempotencyKey()).isEqualTo("refund key 01");
+      JsonNode body = json.readTree(captured.get().body());
+      assertThat(body)
+          .isEqualTo(
+              json.readTree(
+                  """
+                  {"userId":"018f0000-0000-7000-8000-000000000105",
+                   "amount":{"amount":750,"currency":"KRW"},
+                   "source":"HOUSE_POOL","reason":"REFUND"}
+                  """));
+    } finally {
+      server.stop(0);
+    }
+  }
+
   private static RestClient walletRestClient(HttpServer server) {
     DownstreamProperties defaults = ClientIsolationFixture.properties();
     var properties =


## `feat(wallet): define operator refund contract`

diff --git a/src/main/java/com/sportsbook/admin/api/RefundRequest.java b/src/main/java/com/sportsbook/admin/api/RefundRequest.java
new file mode 100644
index 0000000..f5ad120
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/api/RefundRequest.java
@@ -0,0 +1,22 @@
+package com.sportsbook.admin.api;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import jakarta.validation.constraints.NotBlank;
+import jakarta.validation.constraints.NotNull;
+import jakarta.validation.constraints.Positive;
+import jakarta.validation.constraints.Size;
+
+public record RefundRequest(
+    @Positive long amount, @NotNull Currency currency, @NotBlank @Size(max = 256) String reason) {
+
+  public RefundRequest {
+    if (reason != null) {
+      reason = reason.trim();
+    }
+  }
+
+  public Money money() {
+    return new Money(amount, currency);
+  }
+}
diff --git a/src/main/java/com/sportsbook/admin/api/RefundResponse.java b/src/main/java/com/sportsbook/admin/api/RefundResponse.java
new file mode 100644
index 0000000..0793c82
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/api/RefundResponse.java
@@ -0,0 +1,5 @@
+package com.sportsbook.admin.api;
+
+import java.util.UUID;
+
+public record RefundResponse(UUID operationGroupId, UUID actionId) {}


## `test(wallet): validate operator refund input`

diff --git a/src/test/java/com/sportsbook/admin/api/RefundRequestTest.java b/src/test/java/com/sportsbook/admin/api/RefundRequestTest.java
new file mode 100644
index 0000000..5cf6e7f
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/api/RefundRequestTest.java
@@ -0,0 +1,39 @@
+package com.sportsbook.admin.api;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import jakarta.validation.Validation;
+import jakarta.validation.Validator;
+import org.junit.jupiter.api.Test;
+
+class RefundRequestTest {
+
+  private final Validator validator = Validation.buildDefaultValidatorFactory().getValidator();
+
+  @Test
+  void normalizesTheAuditReasonAndBuildsMoney() {
+    RefundRequest request = new RefundRequest(750, Currency.KRW, "  goodwill refund  ");
+
+    assertThat(validator.validate(request)).isEmpty();
+    assertThat(request.reason()).isEqualTo("goodwill refund");
+    assertThat(request.money()).isEqualTo(Money.krw(750));
+  }
+
+  @Test
+  void rejectsNonPositiveMoneyAndInvalidReasons() {
+    assertThat(validator.validate(new RefundRequest(0, Currency.KRW, "refund")))
+        .extracting(violation -> violation.getPropertyPath().toString())
+        .containsExactly("amount");
+    assertThat(validator.validate(new RefundRequest(1, null, "refund")))
+        .extracting(violation -> violation.getPropertyPath().toString())
+        .containsExactly("currency");
+    assertThat(validator.validate(new RefundRequest(1, Currency.KRW, "   ")))
+        .extracting(violation -> violation.getPropertyPath().toString())
+        .contains("reason");
+    assertThat(validator.validate(new RefundRequest(1, Currency.KRW, "r".repeat(257))))
+        .extracting(violation -> violation.getPropertyPath().toString())
+        .containsExactly("reason");
+  }
+}


## `feat(wallet): expose audited refund endpoint`

diff --git a/src/main/java/com/sportsbook/admin/api/WalletAdminController.java b/src/main/java/com/sportsbook/admin/api/WalletAdminController.java
new file mode 100644
index 0000000..6f4cf1f
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/api/WalletAdminController.java
@@ -0,0 +1,40 @@
+package com.sportsbook.admin.api;
+
+import com.sportsbook.admin.audit.AdminAction;
+import com.sportsbook.admin.audit.Audited;
+import com.sportsbook.admin.client.WalletClient;
+import com.sportsbook.admin.context.AdminContext;
+import jakarta.servlet.http.HttpServletRequest;
+import jakarta.validation.Valid;
+import java.util.UUID;
+import org.springframework.security.access.prepost.PreAuthorize;
+import org.springframework.web.bind.annotation.PathVariable;
+import org.springframework.web.bind.annotation.PostMapping;
+import org.springframework.web.bind.annotation.RequestBody;
+import org.springframework.web.bind.annotation.RequestMapping;
+import org.springframework.web.bind.annotation.RestController;
+
+@RestController
+@RequestMapping("/admin/v1/wallet")
+public class WalletAdminController {
+
+  private final WalletClient wallet;
+
+  public WalletAdminController(WalletClient wallet) {
+    this.wallet = wallet;
+  }
+
+  @PostMapping("/{userId}/refund")
+  @PreAuthorize("hasAnyRole('ADMIN','CS')")
+  @Audited(action = AdminAction.WALLET_REFUND, target = "#userId", reason = "#body.reason()")
+  public RefundResponse refund(
+      @PathVariable UUID userId,
+      @Valid @RequestBody RefundRequest body,
+      AdminContext context,
+      HttpServletRequest servletRequest) {
+    UUID operationGroup =
+        wallet.refund(
+            userId, body.money(), AdminRequestHeaders.requireIdempotencyKey(servletRequest));
+    return new RefundResponse(operationGroup, context.actionId());
+  }
+}


