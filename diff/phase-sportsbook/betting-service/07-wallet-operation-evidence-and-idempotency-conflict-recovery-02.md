## `test(wallet): verify debit lookup distinction`

diff --git a/src/test/java/com/sportsbook/betting/client/WalletClientTest.java b/src/test/java/com/sportsbook/betting/client/WalletClientTest.java
index 53b809b..238a3c7 100644
--- a/src/test/java/com/sportsbook/betting/client/WalletClientTest.java
+++ b/src/test/java/com/sportsbook/betting/client/WalletClientTest.java
@@ -10,6 +10,7 @@ import static org.springframework.test.web.client.response.MockRestResponseCreat
 
 import com.fasterxml.jackson.databind.ObjectMapper;
 import com.sportsbook.betting.error.InsufficientBalanceException;
+import com.sportsbook.betting.error.WalletRejectedException;
 import com.sportsbook.protocol.value.Money;
 import java.util.UUID;
 import org.junit.jupiter.api.BeforeEach;
@@ -76,6 +77,53 @@ class WalletClientTest {
         .hasMessage("not enough");
   }
 
+  @Test
+  void treatsOnlyOperationNotFoundAsAbsent() {
+    UUID betId = UUID.randomUUID();
+    UUID userId = UUID.randomUUID();
+    server
+        .expect(requestTo("http://wallet/internal/v1/wallet/transactions/debit/" + betId))
+        .andRespond(
+            withStatus(HttpStatus.NOT_FOUND)
+                .contentType(MediaType.APPLICATION_JSON)
+                .body(
+                    "{\"errorCode\":\"WALLET_OPERATION_NOT_FOUND\"," + "\"detail\":\"missing\"}"));
+
+    assertThat(client.findDebit(betId, userId, Money.krw(1_000))).isEmpty();
+  }
+
+  @Test
+  void retainsStoredAccountNotFoundVerdict() {
+    UUID betId = UUID.randomUUID();
+    UUID userId = UUID.randomUUID();
+    server
+        .expect(requestTo("http://wallet/internal/v1/wallet/transactions/debit/" + betId))
+        .andRespond(
+            withStatus(HttpStatus.NOT_FOUND)
+                .contentType(MediaType.APPLICATION_JSON)
+                .body(
+                    "{\"errorCode\":\"WALLET_ACCOUNT_NOT_FOUND\","
+                        + "\"detail\":\"account missing\"}"));
+
+    assertThatThrownBy(() -> client.findDebit(betId, userId, Money.krw(1_000)))
+        .isInstanceOf(WalletRejectedException.class)
+        .hasMessage("account missing");
+  }
+
+  @Test
+  void rejectsMismatchedRecoveredDebitProof() {
+    UUID betId = UUID.randomUUID();
+    UUID userId = UUID.randomUUID();
+    server
+        .expect(requestTo("http://wallet/internal/v1/wallet/transactions/debit/" + betId))
+        .andRespond(
+            withSuccess(
+                proof(UUID.randomUUID(), UUID.randomUUID(), 1_000, "BET_DEBIT"),
+                MediaType.APPLICATION_JSON));
+
+    assertThatThrownBy(() -> client.findDebit(betId, userId, Money.krw(1_000)))
+        .isInstanceOf(WalletRejectedException.class);
+  }
   private static String proof(UUID operationId, UUID userId, long amount, String reason) {
     return "{\"operationGroupId\":\""
         + operationId


## `feat(wallet): refund locked exposure idempotently`

diff --git a/src/main/java/com/sportsbook/betting/client/WalletClient.java b/src/main/java/com/sportsbook/betting/client/WalletClient.java
index 3ce09fd..13bba09 100644
--- a/src/main/java/com/sportsbook/betting/client/WalletClient.java
+++ b/src/main/java/com/sportsbook/betting/client/WalletClient.java
@@ -19,6 +19,7 @@ import org.springframework.web.client.RestClientException;
 public class WalletClient {
 
   private static final String DEBIT = "/internal/v1/wallet/transactions/debit";
+  private static final String CREDIT = "/internal/v1/wallet/transactions/credit";
   private static final String DEBIT_REASON = "BET_DEBIT";
   private static final String REFUND_REASON = "BET_REFUND";
 
@@ -86,11 +87,27 @@ public class WalletClient {
     }
   }
 
-  private static UUID requireOperationId(WalletOperationResponse response) {
-    if (response == null || response.operationGroupId() == null) {
-      throw new DependencyUnavailableException("Wallet returned no operationGroupId");
+  public UUID refund(UUID betId, UUID userId, Money fullExposure) {
+    try {
+      WalletOperationResponse response =
+          http.post()
+              .uri(CREDIT)
+              .header("Idempotency-Key", "refund:" + betId)
+              .contentType(MediaType.APPLICATION_JSON)
+              .body(WalletCreditRequest.refund(userId, fullExposure))
+              .retrieve()
+              .onStatus(
+                  HttpStatusCode::is4xxClientError,
+                  (request, error) -> {
+                    throw problems.map(problems.read(error));
+                  })
+              .body(WalletOperationResponse.class);
+      return requireProof(response, userId, fullExposure, REFUND_REASON);
+    } catch (BetPlacementException exception) {
+      throw exception;
+    } catch (RestClientException exception) {
+      throw new DependencyUnavailableException("Wallet refund is unavailable", exception);
     }
-    return response.operationGroupId();
   }
 
   private static UUID requireProof(


## `test(wallet): verify authorized refund contract`

diff --git a/src/test/java/com/sportsbook/betting/client/WalletClientTest.java b/src/test/java/com/sportsbook/betting/client/WalletClientTest.java
index 238a3c7..e36c212 100644
--- a/src/test/java/com/sportsbook/betting/client/WalletClientTest.java
+++ b/src/test/java/com/sportsbook/betting/client/WalletClientTest.java
@@ -124,6 +124,39 @@ class WalletClientTest {
     assertThatThrownBy(() -> client.findDebit(betId, userId, Money.krw(1_000)))
         .isInstanceOf(WalletRejectedException.class);
   }
+
+  @Test
+  void refundsLockedExposureUnderIndependentKey() {
+    UUID betId = UUID.randomUUID();
+    UUID operationId = UUID.randomUUID();
+    UUID userId = UUID.randomUUID();
+    server
+        .expect(requestTo("http://wallet/internal/v1/wallet/transactions/credit"))
+        .andExpect(header("Idempotency-Key", "refund:" + betId))
+        .andExpect(jsonPath("$.source").value("USER_LOCKED"))
+        .andExpect(jsonPath("$.reason").value("REFUND"))
+        .andRespond(
+            withSuccess(
+                proof(operationId, userId, 6_000, "BET_REFUND"), MediaType.APPLICATION_JSON));
+
+    assertThat(client.refund(betId, userId, Money.krw(6_000))).isEqualTo(operationId);
+  }
+
+  @Test
+  void rejectsMismatchedRefundProof() {
+    UUID userId = UUID.randomUUID();
+    server
+        .expect(requestTo("http://wallet/internal/v1/wallet/transactions/credit"))
+        .andRespond(
+            withSuccess(
+                proof(UUID.randomUUID(), userId, 5_000, "BET_REFUND"), MediaType.APPLICATION_JSON));
+
+    assertThatThrownBy(() -> client.refund(UUID.randomUUID(), userId, Money.krw(6_000)))
+        .isInstanceOf(com.sportsbook.betting.error.WalletProofMismatchException.class)
+        .extracting("operation")
+        .isEqualTo("refund");
+  }
+
   private static String proof(UUID operationId, UUID userId, long amount, String reason) {
     return "{\"operationGroupId\":\""
         + operationId


## `feat(wallet): bound dependency failures with circuit breaker`

diff --git a/src/main/java/com/sportsbook/betting/client/WalletClient.java b/src/main/java/com/sportsbook/betting/client/WalletClient.java
index 13bba09..36666d2 100644
--- a/src/main/java/com/sportsbook/betting/client/WalletClient.java
+++ b/src/main/java/com/sportsbook/betting/client/WalletClient.java
@@ -5,6 +5,7 @@ import com.sportsbook.betting.error.DependencyUnavailableException;
 import com.sportsbook.betting.error.WalletProofMismatchException;
 import com.sportsbook.betting.error.WalletRejectedException;
 import com.sportsbook.protocol.value.Money;
+import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
 import java.util.Optional;
 import java.util.UUID;
 import org.springframework.beans.factory.annotation.Qualifier;
@@ -32,6 +33,7 @@ public class WalletClient {
     this.problems = problems;
   }
 
+  @CircuitBreaker(name = "walletClient", fallbackMethod = "debitFallback")
   public UUID debit(UUID betId, UUID userId, Money fullExposure) {
     try {
       WalletOperationResponse response =
@@ -55,6 +57,7 @@ public class WalletClient {
     }
   }
 
+  @CircuitBreaker(name = "walletClient", fallbackMethod = "findDebitFallback")
   public Optional<WalletOperationResponse> findDebit(UUID betId, UUID userId, Money fullExposure) {
     try {
       WalletOperationResponse response =
@@ -87,6 +90,7 @@ public class WalletClient {
     }
   }
 
+  @CircuitBreaker(name = "walletClient", fallbackMethod = "refundFallback")
   public UUID refund(UUID betId, UUID userId, Money fullExposure) {
     try {
       WalletOperationResponse response =
@@ -134,6 +138,26 @@ public class WalletClient {
     }
   }
 
+  private UUID debitFallback(UUID betId, UUID userId, Money exposure, Throwable failure) {
+    throw fallback(failure);
+  }
+
+  private Optional<WalletOperationResponse> findDebitFallback(
+      UUID betId, UUID userId, Money amount, Throwable failure) {
+    throw fallback(failure);
+  }
+
+  private UUID refundFallback(UUID betId, UUID userId, Money exposure, Throwable failure) {
+    throw fallback(failure);
+  }
+
+  private static RuntimeException fallback(Throwable failure) {
+    if (failure instanceof BetPlacementException verdict) {
+      return verdict;
+    }
+    return new DependencyUnavailableException("Wallet circuit is unavailable", failure);
+  }
+
   private static final class DebitAbsent extends RuntimeException {
     private static final long serialVersionUID = 1L;
   }


## `test(wallet): verify lifecycle circuit coverage`

diff --git a/src/test/java/com/sportsbook/betting/client/WalletClientResilienceTest.java b/src/test/java/com/sportsbook/betting/client/WalletClientResilienceTest.java
new file mode 100644
index 0000000..d7ff9d6
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/client/WalletClientResilienceTest.java
@@ -0,0 +1,32 @@
+package com.sportsbook.betting.client;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.value.Money;
+import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
+import java.lang.reflect.Method;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class WalletClientResilienceTest {
+
+  @Test
+  void protectsEveryWalletLifecycleCall() throws NoSuchMethodException {
+    assertFallback(
+        WalletClient.class.getMethod("debit", UUID.class, UUID.class, Money.class),
+        "debitFallback");
+    assertFallback(
+        WalletClient.class.getMethod("findDebit", UUID.class, UUID.class, Money.class),
+        "findDebitFallback");
+    assertFallback(
+        WalletClient.class.getMethod("refund", UUID.class, UUID.class, Money.class),
+        "refundFallback");
+  }
+
+  private void assertFallback(Method method, String expected) {
+    CircuitBreaker breaker = method.getAnnotation(CircuitBreaker.class);
+    assertThat(breaker).isNotNull();
+    assertThat(breaker.name()).isEqualTo("walletClient");
+    assertThat(breaker.fallbackMethod()).isEqualTo(expected);
+  }
+}


## `feat(wallet): reconcile debit idempotency conflicts`

diff --git a/src/main/java/com/sportsbook/betting/client/WalletClient.java b/src/main/java/com/sportsbook/betting/client/WalletClient.java
index cbe10ce..8b6579b 100644
--- a/src/main/java/com/sportsbook/betting/client/WalletClient.java
+++ b/src/main/java/com/sportsbook/betting/client/WalletClient.java
@@ -43,6 +43,15 @@ public class WalletClient {
               .contentType(MediaType.APPLICATION_JSON)
               .body(new WalletDebitRequest(userId, fullExposure))
               .retrieve()
+              .onStatus(
+                  status -> status.value() == HttpStatus.CONFLICT.value(),
+                  (request, error) -> {
+                    WalletProblem problem = problems.read(error);
+                    if (WalletProblemMapper.IDEMPOTENCY_CONFLICT.equals(problem.errorCode())) {
+                      throw new DebitConflict();
+                    }
+                    throw problems.map(problem);
+                  })
               .onStatus(
                   HttpStatusCode::is4xxClientError,
                   (request, error) -> {
@@ -55,6 +64,14 @@ public class WalletClient {
                   })
               .body(WalletOperationResponse.class);
       return requireDebitProof(response, userId, fullExposure);
+    } catch (DebitConflict conflict) {
+      return findDebit(betId, userId, fullExposure)
+          .map(WalletOperationResponse::operationGroupId)
+          .orElseThrow(
+              () ->
+                  new WalletRejectedException(
+                      WalletProblemMapper.IDEMPOTENCY_CONFLICT,
+                      "Wallet debit identity could not be proven"));
     } catch (BetPlacementException exception) {
       throw exception;
     } catch (RestClientException exception) {
@@ -110,6 +127,15 @@ public class WalletClient {
               .contentType(MediaType.APPLICATION_JSON)
               .body(WalletCreditRequest.refund(userId, fullExposure))
               .retrieve()
+              .onStatus(
+                  status -> status.value() == HttpStatus.CONFLICT.value(),
+                  (request, error) -> {
+                    WalletProblem problem = problems.read(error);
+                    if (WalletProblemMapper.IDEMPOTENCY_CONFLICT.equals(problem.errorCode())) {
+                      throw new WalletProofMismatchException("refund");
+                    }
+                    throw problems.map(problem);
+                  })
               .onStatus(
                   HttpStatusCode::is4xxClientError,
                   (request, error) -> {
@@ -176,4 +202,8 @@ public class WalletClient {
   private static final class DebitAbsent extends RuntimeException {
     private static final long serialVersionUID = 1L;
   }
+
+  private static final class DebitConflict extends RuntimeException {
+    private static final long serialVersionUID = 1L;
+  }
 }
diff --git a/src/main/java/com/sportsbook/betting/client/WalletProblemMapper.java b/src/main/java/com/sportsbook/betting/client/WalletProblemMapper.java
index cc3ea00..b9eddd9 100644
--- a/src/main/java/com/sportsbook/betting/client/WalletProblemMapper.java
+++ b/src/main/java/com/sportsbook/betting/client/WalletProblemMapper.java
@@ -40,12 +40,10 @@ public class WalletProblemMapper {
   RuntimeException map(WalletProblem problem) {
     return switch (problem.errorCode()) {
       case INSUFFICIENT_BALANCE -> new InsufficientBalanceException(problem.detail());
-      case ACCOUNT_NOT_FOUND,
-              CURRENCY_MISMATCH,
-              AMOUNT_OUT_OF_RANGE,
-              RECOVERY_BLOCKED,
-              IDEMPOTENCY_CONFLICT ->
+      case ACCOUNT_NOT_FOUND, CURRENCY_MISMATCH, AMOUNT_OUT_OF_RANGE, RECOVERY_BLOCKED ->
           new WalletRejectedException(problem.errorCode(), problem.detail());
+      case IDEMPOTENCY_CONFLICT ->
+          new DependencyUnavailableException("Wallet idempotency identity conflicts");
       default -> new DependencyUnavailableException("Wallet returned an unexpected problem");
     };
   }


## `test(wallet): verify authoritative conflict recovery`

diff --git a/src/test/java/com/sportsbook/betting/client/WalletClientTest.java b/src/test/java/com/sportsbook/betting/client/WalletClientTest.java
index 011f4d3..96cd740 100644
--- a/src/test/java/com/sportsbook/betting/client/WalletClientTest.java
+++ b/src/test/java/com/sportsbook/betting/client/WalletClientTest.java
@@ -185,6 +185,68 @@ class WalletClientTest {
     }
   }
 
+  @Test
+  void adoptsOnlyAnAuthoritativeDebitAfterIdempotencyConflict() {
+    UUID betId = UUID.randomUUID();
+    UUID userId = UUID.randomUUID();
+    UUID operationId = UUID.randomUUID();
+    expectDebitConflict(betId);
+    server
+        .expect(requestTo("http://wallet/internal/v1/wallet/transactions/debit/" + betId))
+        .andRespond(
+            withSuccess(
+                proof(operationId, userId, 1_000, "BET_DEBIT"), MediaType.APPLICATION_JSON));
+
+    assertThat(client.debit(betId, userId, Money.krw(1_000))).isEqualTo(operationId);
+  }
+
+  @Test
+  void isolatesForeignDebitProofAfterIdempotencyConflict() {
+    UUID betId = UUID.randomUUID();
+    UUID userId = UUID.randomUUID();
+    expectDebitConflict(betId);
+    server
+        .expect(requestTo("http://wallet/internal/v1/wallet/transactions/debit/" + betId))
+        .andRespond(
+            withSuccess(
+                proof(UUID.randomUUID(), UUID.randomUUID(), 1_000, "BET_DEBIT"),
+                MediaType.APPLICATION_JSON));
+
+    assertThatThrownBy(() -> client.debit(betId, userId, Money.krw(1_000)))
+        .isInstanceOf(WalletRejectedException.class);
+  }
+
+  @Test
+  void isolatesMissingDebitProofAfterIdempotencyConflict() {
+    UUID betId = UUID.randomUUID();
+    UUID userId = UUID.randomUUID();
+    expectDebitConflict(betId);
+    server
+        .expect(requestTo("http://wallet/internal/v1/wallet/transactions/debit/" + betId))
+        .andRespond(
+            withStatus(HttpStatus.NOT_FOUND)
+                .contentType(MediaType.APPLICATION_JSON)
+                .body("{\"errorCode\":\"WALLET_OPERATION_NOT_FOUND\"}"));
+
+    assertThatThrownBy(() -> client.debit(betId, userId, Money.krw(1_000)))
+        .isInstanceOf(WalletRejectedException.class)
+        .extracting("walletErrorCode")
+        .isEqualTo("WALLET_IDEMPOTENCY_CONFLICT");
+  }
+
+  @Test
+  void keepsRefundConflictIncomplete() {
+    server
+        .expect(requestTo("http://wallet/internal/v1/wallet/transactions/credit"))
+        .andRespond(
+            withStatus(HttpStatus.CONFLICT)
+                .contentType(MediaType.APPLICATION_JSON)
+                .body("{\"errorCode\":\"WALLET_IDEMPOTENCY_CONFLICT\"}"));
+
+    assertThatThrownBy(() -> client.refund(UUID.randomUUID(), UUID.randomUUID(), Money.krw(1_000)))
+        .isInstanceOf(com.sportsbook.betting.error.WalletProofMismatchException.class);
+  }
+
   private static String proof(UUID operationId, UUID userId, long amount, String reason) {
     return "{\"operationGroupId\":\""
         + operationId
@@ -196,4 +258,14 @@ class WalletClientTest {
         + reason
         + "\",\"at\":\"2026-08-22T00:00:00Z\"}";
   }
+
+  private void expectDebitConflict(UUID betId) {
+    server
+        .expect(requestTo("http://wallet/internal/v1/wallet/transactions/debit"))
+        .andExpect(header("Idempotency-Key", betId.toString()))
+        .andRespond(
+            withStatus(HttpStatus.CONFLICT)
+                .contentType(MediaType.APPLICATION_JSON)
+                .body("{\"errorCode\":\"WALLET_IDEMPOTENCY_CONFLICT\"}"));
+  }
 }
diff --git a/src/test/java/com/sportsbook/betting/client/WalletProblemMapperTest.java b/src/test/java/com/sportsbook/betting/client/WalletProblemMapperTest.java
index 9c97a8a..8c8ef2f 100644
--- a/src/test/java/com/sportsbook/betting/client/WalletProblemMapperTest.java
+++ b/src/test/java/com/sportsbook/betting/client/WalletProblemMapperTest.java
@@ -29,13 +29,14 @@ class WalletProblemMapperTest {
         java.util.List.of(
             "WALLET_CURRENCY_MISMATCH",
             "WALLET_AMOUNT_OUT_OF_RANGE",
-            "WALLET_ACCOUNT_RECOVERY_BLOCKED",
-            "WALLET_IDEMPOTENCY_CONFLICT")) {
+            "WALLET_ACCOUNT_RECOVERY_BLOCKED")) {
       assertThat(mapper.map(read(code, "durable rejection")))
           .isInstanceOf(WalletRejectedException.class)
           .extracting("walletErrorCode")
           .isEqualTo(code);
     }
+    assertThat(mapper.map(read("WALLET_IDEMPOTENCY_CONFLICT", "conflict")))
+        .isInstanceOf(com.sportsbook.betting.error.DependencyUnavailableException.class);
   }
 
   private WalletProblem read(String code, String detail) {


## `feat(wallet): require exact provider success statuses`

diff --git a/src/main/java/com/sportsbook/betting/client/WalletClient.java b/src/main/java/com/sportsbook/betting/client/WalletClient.java
index 36666d2..cbe10ce 100644
--- a/src/main/java/com/sportsbook/betting/client/WalletClient.java
+++ b/src/main/java/com/sportsbook/betting/client/WalletClient.java
@@ -48,6 +48,11 @@ public class WalletClient {
                   (request, error) -> {
                     throw problems.map(problems.read(error));
                   })
+              .onStatus(
+                  status -> status.value() != HttpStatus.OK.value(),
+                  (request, error) -> {
+                    throw new DependencyUnavailableException("Wallet debit was not accepted");
+                  })
               .body(WalletOperationResponse.class);
       return requireDebitProof(response, userId, fullExposure);
     } catch (BetPlacementException exception) {
@@ -78,6 +83,11 @@ public class WalletClient {
                   (request, error) -> {
                     throw problems.map(problems.read(error));
                   })
+              .onStatus(
+                  status -> status.value() != HttpStatus.OK.value(),
+                  (request, error) -> {
+                    throw new DependencyUnavailableException("Wallet lookup was not accepted");
+                  })
               .body(WalletOperationResponse.class);
       requireDebitProof(response, userId, fullExposure);
       return Optional.of(response);
@@ -105,6 +115,11 @@ public class WalletClient {
                   (request, error) -> {
                     throw problems.map(problems.read(error));
                   })
+              .onStatus(
+                  status -> status.value() != HttpStatus.OK.value(),
+                  (request, error) -> {
+                    throw new DependencyUnavailableException("Wallet refund was not accepted");
+                  })
               .body(WalletOperationResponse.class);
       return requireProof(response, userId, fullExposure, REFUND_REASON);
     } catch (BetPlacementException exception) {


