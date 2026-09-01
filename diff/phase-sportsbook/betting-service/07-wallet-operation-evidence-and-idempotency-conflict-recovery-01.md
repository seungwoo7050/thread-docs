# 지갑 작업 증거와 멱등성 충돌 복구

## `feat(error): classify wallet and replay conflicts`

diff --git a/src/main/java/com/sportsbook/betting/error/DuplicateBetException.java b/src/main/java/com/sportsbook/betting/error/DuplicateBetException.java
new file mode 100644
index 0000000..b94690b
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/error/DuplicateBetException.java
@@ -0,0 +1,10 @@
+package com.sportsbook.betting.error;
+
+import com.sportsbook.protocol.error.ErrorCode;
+
+public class DuplicateBetException extends BetPlacementException {
+
+  public DuplicateBetException(String message) {
+    super(ErrorCode.DUPLICATE_BET, message);
+  }
+}
diff --git a/src/main/java/com/sportsbook/betting/error/InsufficientBalanceException.java b/src/main/java/com/sportsbook/betting/error/InsufficientBalanceException.java
new file mode 100644
index 0000000..f9dd026
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/error/InsufficientBalanceException.java
@@ -0,0 +1,10 @@
+package com.sportsbook.betting.error;
+
+import com.sportsbook.protocol.error.ErrorCode;
+
+public class InsufficientBalanceException extends BetPlacementException {
+
+  public InsufficientBalanceException(String message) {
+    super(ErrorCode.INSUFFICIENT_BALANCE, message == null ? "Insufficient balance" : message);
+  }
+}


## `feat(error): retain definitive wallet rejection`

diff --git a/src/main/java/com/sportsbook/betting/error/WalletProofMismatchException.java b/src/main/java/com/sportsbook/betting/error/WalletProofMismatchException.java
new file mode 100644
index 0000000..affd0cc
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/error/WalletProofMismatchException.java
@@ -0,0 +1,17 @@
+package com.sportsbook.betting.error;
+
+import java.util.Objects;
+
+public final class WalletProofMismatchException extends DependencyUnavailableException {
+
+  private final String operation;
+
+  public WalletProofMismatchException(String operation) {
+    super("Wallet returned a mismatched " + operation + " proof");
+    this.operation = Objects.requireNonNull(operation, "operation");
+  }
+
+  public String operation() {
+    return operation;
+  }
+}
diff --git a/src/main/java/com/sportsbook/betting/error/WalletRejectedException.java b/src/main/java/com/sportsbook/betting/error/WalletRejectedException.java
new file mode 100644
index 0000000..11b3bc1
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/error/WalletRejectedException.java
@@ -0,0 +1,17 @@
+package com.sportsbook.betting.error;
+
+import com.sportsbook.protocol.error.ErrorCode;
+
+public class WalletRejectedException extends BetPlacementException {
+
+  private final String walletErrorCode;
+
+  public WalletRejectedException(String walletErrorCode, String detail) {
+    super(ErrorCode.VALIDATION_FAILED, detail);
+    this.walletErrorCode = walletErrorCode;
+  }
+
+  public String walletErrorCode() {
+    return walletErrorCode;
+  }
+}


## `feat(wallet): define debit and refund requests`

diff --git a/src/main/java/com/sportsbook/betting/client/WalletCreditRequest.java b/src/main/java/com/sportsbook/betting/client/WalletCreditRequest.java
new file mode 100644
index 0000000..01cddd0
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/client/WalletCreditRequest.java
@@ -0,0 +1,19 @@
+package com.sportsbook.betting.client;
+
+import com.sportsbook.protocol.value.Money;
+import java.util.Objects;
+import java.util.UUID;
+
+public record WalletCreditRequest(UUID userId, Money amount, String source, String reason) {
+
+  static WalletCreditRequest refund(UUID userId, Money amount) {
+    return new WalletCreditRequest(userId, amount, "USER_LOCKED", "REFUND");
+  }
+
+  public WalletCreditRequest {
+    Objects.requireNonNull(userId, "userId");
+    Objects.requireNonNull(amount, "amount");
+    Objects.requireNonNull(source, "source");
+    Objects.requireNonNull(reason, "reason");
+  }
+}
diff --git a/src/main/java/com/sportsbook/betting/client/WalletDebitRequest.java b/src/main/java/com/sportsbook/betting/client/WalletDebitRequest.java
new file mode 100644
index 0000000..8ef26f6
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/client/WalletDebitRequest.java
@@ -0,0 +1,12 @@
+package com.sportsbook.betting.client;
+
+import com.sportsbook.protocol.value.Money;
+import java.util.Objects;
+import java.util.UUID;
+
+public record WalletDebitRequest(UUID userId, Money amount) {
+  public WalletDebitRequest {
+    Objects.requireNonNull(userId, "userId");
+    Objects.requireNonNull(amount, "amount");
+  }
+}


## `feat(wallet): read operation and problem outcomes`

diff --git a/src/main/java/com/sportsbook/betting/client/WalletOperationResponse.java b/src/main/java/com/sportsbook/betting/client/WalletOperationResponse.java
new file mode 100644
index 0000000..06f54a2
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/client/WalletOperationResponse.java
@@ -0,0 +1,10 @@
+package com.sportsbook.betting.client;
+
+import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
+import com.sportsbook.protocol.value.Money;
+import java.time.Instant;
+import java.util.UUID;
+
+@JsonIgnoreProperties(ignoreUnknown = true)
+public record WalletOperationResponse(
+    UUID operationGroupId, UUID userId, Money amount, String reason, Instant at) {}
diff --git a/src/main/java/com/sportsbook/betting/client/WalletProblem.java b/src/main/java/com/sportsbook/betting/client/WalletProblem.java
new file mode 100644
index 0000000..a0ffe4e
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/client/WalletProblem.java
@@ -0,0 +1,6 @@
+package com.sportsbook.betting.client;
+
+import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
+
+@JsonIgnoreProperties(ignoreUnknown = true)
+public record WalletProblem(String errorCode, String detail) {}


## `feat(wallet): classify authoritative problems`

diff --git a/src/main/java/com/sportsbook/betting/client/WalletProblemMapper.java b/src/main/java/com/sportsbook/betting/client/WalletProblemMapper.java
new file mode 100644
index 0000000..cc3ea00
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/client/WalletProblemMapper.java
@@ -0,0 +1,52 @@
+package com.sportsbook.betting.client;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.betting.error.DependencyUnavailableException;
+import com.sportsbook.betting.error.InsufficientBalanceException;
+import com.sportsbook.betting.error.WalletRejectedException;
+import java.io.IOException;
+import org.springframework.http.client.ClientHttpResponse;
+import org.springframework.stereotype.Component;
+
+@Component
+public class WalletProblemMapper {
+
+  static final String OPERATION_NOT_FOUND = "WALLET_OPERATION_NOT_FOUND";
+  static final String ACCOUNT_NOT_FOUND = "WALLET_ACCOUNT_NOT_FOUND";
+  static final String INSUFFICIENT_BALANCE = "WALLET_INSUFFICIENT_BALANCE";
+  static final String CURRENCY_MISMATCH = "WALLET_CURRENCY_MISMATCH";
+  static final String AMOUNT_OUT_OF_RANGE = "WALLET_AMOUNT_OUT_OF_RANGE";
+  static final String RECOVERY_BLOCKED = "WALLET_ACCOUNT_RECOVERY_BLOCKED";
+  static final String IDEMPOTENCY_CONFLICT = "WALLET_IDEMPOTENCY_CONFLICT";
+
+  private final ObjectMapper mapper;
+
+  public WalletProblemMapper(ObjectMapper mapper) {
+    this.mapper = mapper;
+  }
+
+  WalletProblem read(ClientHttpResponse response) {
+    try {
+      WalletProblem problem = mapper.readValue(response.getBody(), WalletProblem.class);
+      if (problem.errorCode() == null || problem.errorCode().isBlank()) {
+        throw new IOException("missing errorCode");
+      }
+      return problem;
+    } catch (IOException exception) {
+      throw new DependencyUnavailableException("Wallet returned an unreadable problem", exception);
+    }
+  }
+
+  RuntimeException map(WalletProblem problem) {
+    return switch (problem.errorCode()) {
+      case INSUFFICIENT_BALANCE -> new InsufficientBalanceException(problem.detail());
+      case ACCOUNT_NOT_FOUND,
+              CURRENCY_MISMATCH,
+              AMOUNT_OUT_OF_RANGE,
+              RECOVERY_BLOCKED,
+              IDEMPOTENCY_CONFLICT ->
+          new WalletRejectedException(problem.errorCode(), problem.detail());
+      default -> new DependencyUnavailableException("Wallet returned an unexpected problem");
+    };
+  }
+}


## `test(wallet): verify authoritative problem mapping`

diff --git a/src/test/java/com/sportsbook/betting/client/WalletProblemMapperTest.java b/src/test/java/com/sportsbook/betting/client/WalletProblemMapperTest.java
new file mode 100644
index 0000000..9c97a8a
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/client/WalletProblemMapperTest.java
@@ -0,0 +1,47 @@
+package com.sportsbook.betting.client;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.betting.error.InsufficientBalanceException;
+import com.sportsbook.betting.error.WalletRejectedException;
+import java.nio.charset.StandardCharsets;
+import org.junit.jupiter.api.Test;
+import org.springframework.http.HttpStatus;
+import org.springframework.mock.http.client.MockClientHttpResponse;
+
+class WalletProblemMapperTest {
+
+  private final WalletProblemMapper mapper = new WalletProblemMapper(new ObjectMapper());
+
+  @Test
+  void mapsStoredBusinessProblemsWithoutChangingTheirMeaning() {
+    WalletProblem insufficient = read("WALLET_INSUFFICIENT_BALANCE", "not enough balance");
+    WalletProblem missing = read("WALLET_ACCOUNT_NOT_FOUND", "account missing");
+
+    assertThat(mapper.map(insufficient)).isInstanceOf(InsufficientBalanceException.class);
+    assertThat(mapper.map(missing))
+        .isInstanceOf(WalletRejectedException.class)
+        .extracting("walletErrorCode")
+        .isEqualTo("WALLET_ACCOUNT_NOT_FOUND");
+
+    for (String code :
+        java.util.List.of(
+            "WALLET_CURRENCY_MISMATCH",
+            "WALLET_AMOUNT_OUT_OF_RANGE",
+            "WALLET_ACCOUNT_RECOVERY_BLOCKED",
+            "WALLET_IDEMPOTENCY_CONFLICT")) {
+      assertThat(mapper.map(read(code, "durable rejection")))
+          .isInstanceOf(WalletRejectedException.class)
+          .extracting("walletErrorCode")
+          .isEqualTo(code);
+    }
+  }
+
+  private WalletProblem read(String code, String detail) {
+    String json = "{\"errorCode\":\"" + code + "\",\"detail\":\"" + detail + "\"}";
+    return mapper.read(
+        new MockClientHttpResponse(
+            json.getBytes(StandardCharsets.UTF_8), HttpStatus.UNPROCESSABLE_ENTITY));
+  }
+}


## `feat(wallet): debit full exposure idempotently`

diff --git a/src/main/java/com/sportsbook/betting/client/WalletClient.java b/src/main/java/com/sportsbook/betting/client/WalletClient.java
new file mode 100644
index 0000000..a1190e0
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/client/WalletClient.java
@@ -0,0 +1,85 @@
+package com.sportsbook.betting.client;
+
+import com.sportsbook.betting.error.BetPlacementException;
+import com.sportsbook.betting.error.DependencyUnavailableException;
+import com.sportsbook.betting.error.WalletProofMismatchException;
+import com.sportsbook.betting.error.WalletRejectedException;
+import com.sportsbook.protocol.value.Money;
+import java.util.UUID;
+import org.springframework.beans.factory.annotation.Qualifier;
+import org.springframework.http.HttpStatusCode;
+import org.springframework.http.MediaType;
+import org.springframework.stereotype.Component;
+import org.springframework.web.client.RestClient;
+import org.springframework.web.client.RestClientException;
+
+@Component
+public class WalletClient {
+
+  private static final String DEBIT = "/internal/v1/wallet/transactions/debit";
+  private static final String DEBIT_REASON = "BET_DEBIT";
+  private static final String REFUND_REASON = "BET_REFUND";
+
+  private final RestClient http;
+  private final WalletProblemMapper problems;
+
+  public WalletClient(
+      @Qualifier("walletRestClient") RestClient http, WalletProblemMapper problems) {
+    this.http = http;
+    this.problems = problems;
+  }
+
+  public UUID debit(UUID betId, UUID userId, Money fullExposure) {
+    try {
+      WalletOperationResponse response =
+          http.post()
+              .uri(DEBIT)
+              .header("Idempotency-Key", betId.toString())
+              .contentType(MediaType.APPLICATION_JSON)
+              .body(new WalletDebitRequest(userId, fullExposure))
+              .retrieve()
+              .onStatus(
+                  HttpStatusCode::is4xxClientError,
+                  (request, error) -> {
+                    throw problems.map(problems.read(error));
+                  })
+              .body(WalletOperationResponse.class);
+      return requireDebitProof(response, userId, fullExposure);
+    } catch (BetPlacementException exception) {
+      throw exception;
+    } catch (RestClientException exception) {
+      throw new DependencyUnavailableException("Wallet debit is unavailable", exception);
+    }
+  }
+
+  private static UUID requireOperationId(WalletOperationResponse response) {
+    if (response == null || response.operationGroupId() == null) {
+      throw new DependencyUnavailableException("Wallet returned no operationGroupId");
+    }
+    return response.operationGroupId();
+  }
+
+  private static UUID requireProof(
+      WalletOperationResponse response, UUID userId, Money amount, String reason) {
+    if (response == null
+        || response.operationGroupId() == null
+        || !userId.equals(response.userId())
+        || !amount.equals(response.amount())
+        || !reason.equals(response.reason())
+        || response.at() == null) {
+      String operation = reason.equals(REFUND_REASON) ? "refund" : "debit";
+      throw new WalletProofMismatchException(operation);
+    }
+    return response.operationGroupId();
+  }
+
+  private static UUID requireDebitProof(
+      WalletOperationResponse response, UUID userId, Money amount) {
+    try {
+      return requireProof(response, userId, amount, DEBIT_REASON);
+    } catch (WalletProofMismatchException mismatch) {
+      throw new WalletRejectedException(
+          "WALLET_OPERATION_MISMATCH", "Wallet debit proof did not match this bet");
+    }
+  }
+}


## `test(wallet): verify authoritative debit outcomes`

diff --git a/src/test/java/com/sportsbook/betting/client/WalletClientTest.java b/src/test/java/com/sportsbook/betting/client/WalletClientTest.java
new file mode 100644
index 0000000..53b809b
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/client/WalletClientTest.java
@@ -0,0 +1,90 @@
+package com.sportsbook.betting.client;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.header;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.jsonPath;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.requestTo;
+import static org.springframework.test.web.client.response.MockRestResponseCreators.withStatus;
+import static org.springframework.test.web.client.response.MockRestResponseCreators.withSuccess;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.betting.error.InsufficientBalanceException;
+import com.sportsbook.protocol.value.Money;
+import java.util.UUID;
+import org.junit.jupiter.api.BeforeEach;
+import org.junit.jupiter.api.Test;
+import org.springframework.http.HttpStatus;
+import org.springframework.http.MediaType;
+import org.springframework.test.web.client.MockRestServiceServer;
+import org.springframework.web.client.RestClient;
+
+class WalletClientTest {
+
+  private MockRestServiceServer server;
+  private WalletClient client;
+
+  @BeforeEach
+  void setUp() {
+    RestClient.Builder builder = RestClient.builder().baseUrl("http://wallet");
+    server = MockRestServiceServer.bindTo(builder).build();
+    client = new WalletClient(builder.build(), new WalletProblemMapper(new ObjectMapper()));
+  }
+
+  @Test
+  void debitsCanonicalBetKeyAndFullExposure() {
+    UUID betId = UUID.randomUUID();
+    UUID operationId = UUID.randomUUID();
+    UUID userId = UUID.randomUUID();
+    server
+        .expect(requestTo("http://wallet/internal/v1/wallet/transactions/debit"))
+        .andExpect(header("Idempotency-Key", betId.toString()))
+        .andExpect(jsonPath("$.amount.amount").value(6_000))
+        .andRespond(
+            withSuccess(
+                proof(operationId, userId, 6_000, "BET_DEBIT"), MediaType.APPLICATION_JSON));
+
+    assertThat(client.debit(betId, userId, Money.krw(6_000))).isEqualTo(operationId);
+  }
+
+  @Test
+  void rejectsMismatchedDebitProof() {
+    UUID userId = UUID.randomUUID();
+    server
+        .expect(requestTo("http://wallet/internal/v1/wallet/transactions/debit"))
+        .andRespond(
+            withSuccess(
+                proof(UUID.randomUUID(), userId, 6_000, "BET_REFUND"), MediaType.APPLICATION_JSON));
+
+    assertThatThrownBy(() -> client.debit(UUID.randomUUID(), userId, Money.krw(6_000)))
+        .isInstanceOf(WalletRejectedException.class);
+  }
+
+  @Test
+  void retainsInsufficientBalanceAsBusinessVerdict() {
+    server
+        .expect(requestTo("http://wallet/internal/v1/wallet/transactions/debit"))
+        .andRespond(
+            withStatus(HttpStatus.UNPROCESSABLE_ENTITY)
+                .contentType(MediaType.APPLICATION_JSON)
+                .body(
+                    "{\"errorCode\":\"WALLET_INSUFFICIENT_BALANCE\","
+                        + "\"detail\":\"not enough\"}"));
+
+    assertThatThrownBy(() -> client.debit(UUID.randomUUID(), UUID.randomUUID(), Money.krw(1_000)))
+        .isInstanceOf(InsufficientBalanceException.class)
+        .hasMessage("not enough");
+  }
+
+  private static String proof(UUID operationId, UUID userId, long amount, String reason) {
+    return "{\"operationGroupId\":\""
+        + operationId
+        + "\",\"userId\":\""
+        + userId
+        + "\",\"amount\":{\"amount\":"
+        + amount
+        + ",\"currency\":\"KRW\"},\"reason\":\""
+        + reason
+        + "\",\"at\":\"2026-08-22T00:00:00Z\"}";
+  }
+}


## `feat(wallet): distinguish absent debit from stored failure`

diff --git a/src/main/java/com/sportsbook/betting/client/WalletClient.java b/src/main/java/com/sportsbook/betting/client/WalletClient.java
index a1190e0..3ce09fd 100644
--- a/src/main/java/com/sportsbook/betting/client/WalletClient.java
+++ b/src/main/java/com/sportsbook/betting/client/WalletClient.java
@@ -5,8 +5,10 @@ import com.sportsbook.betting.error.DependencyUnavailableException;
 import com.sportsbook.betting.error.WalletProofMismatchException;
 import com.sportsbook.betting.error.WalletRejectedException;
 import com.sportsbook.protocol.value.Money;
+import java.util.Optional;
 import java.util.UUID;
 import org.springframework.beans.factory.annotation.Qualifier;
+import org.springframework.http.HttpStatus;
 import org.springframework.http.HttpStatusCode;
 import org.springframework.http.MediaType;
 import org.springframework.stereotype.Component;
@@ -52,6 +54,38 @@ public class WalletClient {
     }
   }
 
+  public Optional<WalletOperationResponse> findDebit(UUID betId, UUID userId, Money fullExposure) {
+    try {
+      WalletOperationResponse response =
+          http.get()
+              .uri(DEBIT + "/{betId}", betId)
+              .retrieve()
+              .onStatus(
+                  status -> status.value() == HttpStatus.NOT_FOUND.value(),
+                  (request, error) -> {
+                    WalletProblem problem = problems.read(error);
+                    if (WalletProblemMapper.OPERATION_NOT_FOUND.equals(problem.errorCode())) {
+                      throw new DebitAbsent();
+                    }
+                    throw problems.map(problem);
+                  })
+              .onStatus(
+                  HttpStatusCode::is4xxClientError,
+                  (request, error) -> {
+                    throw problems.map(problems.read(error));
+                  })
+              .body(WalletOperationResponse.class);
+      requireDebitProof(response, userId, fullExposure);
+      return Optional.of(response);
+    } catch (DebitAbsent exception) {
+      return Optional.empty();
+    } catch (BetPlacementException exception) {
+      throw exception;
+    } catch (RestClientException exception) {
+      throw new DependencyUnavailableException("Wallet debit lookup is unavailable", exception);
+    }
+  }
+
   private static UUID requireOperationId(WalletOperationResponse response) {
     if (response == null || response.operationGroupId() == null) {
       throw new DependencyUnavailableException("Wallet returned no operationGroupId");
@@ -82,4 +116,8 @@ public class WalletClient {
           "WALLET_OPERATION_MISMATCH", "Wallet debit proof did not match this bet");
     }
   }
+
+  private static final class DebitAbsent extends RuntimeException {
+    private static final long serialVersionUID = 1L;
+  }
 }


