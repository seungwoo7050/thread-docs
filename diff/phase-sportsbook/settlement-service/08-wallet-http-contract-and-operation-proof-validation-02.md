## `feat(wallet): reject unexpected HTTP statuses`

diff --git a/src/main/java/com/sportsbook/settlement/client/WalletFailurePolicy.java b/src/main/java/com/sportsbook/settlement/client/WalletFailurePolicy.java
index f2d38f8..f5dec68 100644
--- a/src/main/java/com/sportsbook/settlement/client/WalletFailurePolicy.java
+++ b/src/main/java/com/sportsbook/settlement/client/WalletFailurePolicy.java
@@ -12,6 +12,9 @@ public final class WalletFailurePolicy {
 
   public static void throwFor(ClientHttpResponse response) throws IOException {
     int status = response.getStatusCode().value();
+    if (status < 400) {
+      throw malformedResponse(status);
+    }
     String errorCode = readErrorCode(response, status);
     if (status == 429 || status >= 500) {
       throw new TransientFailure(status, errorCode);
@@ -20,7 +23,11 @@ public final class WalletFailurePolicy {
   }
 
   public static TransientFailure malformedSuccess() {
-    return new TransientFailure(200, "WALLET_MALFORMED_RESPONSE");
+    return malformedResponse(200);
+  }
+
+  public static TransientFailure malformedResponse(int status) {
+    return new TransientFailure(status, "WALLET_MALFORMED_RESPONSE");
   }
 
   private static String readErrorCode(ClientHttpResponse response, int status) throws IOException {


## `feat(wallet): couple adjustment statuses and proofs`

diff --git a/src/main/java/com/sportsbook/settlement/client/WalletClient.java b/src/main/java/com/sportsbook/settlement/client/WalletClient.java
index 747f1b1..44bd397 100644
--- a/src/main/java/com/sportsbook/settlement/client/WalletClient.java
+++ b/src/main/java/com/sportsbook/settlement/client/WalletClient.java
@@ -7,7 +7,6 @@ import java.util.Objects;
 import java.util.UUID;
 import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.beans.factory.annotation.Qualifier;
-import org.springframework.http.HttpStatusCode;
 import org.springframework.http.MediaType;
 import org.springframework.http.client.ClientHttpRequestFactory;
 import org.springframework.stereotype.Component;
@@ -85,7 +84,7 @@ public class WalletClient {
       UUID userId,
       Money previousPayout,
       Money newPayout) {
-    WalletAdjustmentProof proof =
+    var response =
         http.post()
             .uri(ADJUSTMENT_PATH)
             .header(IDEMPOTENCY_HEADER, "settlement:revision:" + revisionId)
@@ -95,11 +94,17 @@ public class WalletClient {
                     revisionId, betId, revisionNumber, userId, previousPayout, newPayout))
             .retrieve()
             .onStatus(
-                HttpStatusCode::isError,
+                status -> status.value() != 200 && status.value() != 202,
                 (request, httpResponse) -> WalletFailurePolicy.throwFor(httpResponse))
-            .body(WalletAdjustmentProof.class);
-    if (proof == null || proof.revisionId() == null || proof.status() == null) {
-      throw WalletFailurePolicy.malformedSuccess();
+            .toEntity(WalletAdjustmentProof.class);
+    WalletAdjustmentProof proof = response.getBody();
+    int status = response.getStatusCode().value();
+    boolean coupled =
+        proof != null
+            && ((status == 200 && proof.status() == WalletAdjustmentProof.Status.APPLIED)
+                || (status == 202 && proof.status() == WalletAdjustmentProof.Status.BLOCKED));
+    if (!coupled || proof.revisionId() == null) {
+      throw WalletFailurePolicy.malformedResponse(status);
     }
     return proof;
   }
@@ -110,7 +115,7 @@ public class WalletClient {
             .uri(ADJUSTMENT_PATH + "/{revisionId}", revisionId)
             .retrieve()
             .onStatus(
-                HttpStatusCode::isError,
+                status -> status.value() != 200,
                 (request, httpResponse) -> WalletFailurePolicy.throwFor(httpResponse))
             .body(WalletAdjustmentProof.class);
     if (proof == null || !revisionId.equals(proof.revisionId()) || proof.status() == null) {


## `feat(wallet): reject malformed success responses`

diff --git a/src/main/java/com/sportsbook/settlement/client/WalletClient.java b/src/main/java/com/sportsbook/settlement/client/WalletClient.java
index c9ee386..d647bcd 100644
--- a/src/main/java/com/sportsbook/settlement/client/WalletClient.java
+++ b/src/main/java/com/sportsbook/settlement/client/WalletClient.java
@@ -2,7 +2,6 @@ package com.sportsbook.settlement.client;
 
 import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
 import com.sportsbook.protocol.value.Money;
-import java.util.Objects;
 import java.util.UUID;
 import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.beans.factory.annotation.Qualifier;
@@ -58,7 +57,7 @@ public class WalletClient {
                 HttpStatusCode::isError,
                 (request, httpResponse) -> WalletFailurePolicy.throwFor(httpResponse))
             .body(CreditResponse.class);
-    return Objects.requireNonNull(response, "wallet credit response").operationGroupId();
+    return requireOperationGroupId(response);
   }
 
   public UUID forfeit(String idempotencyKey, UUID userId, Money amount) {
@@ -73,7 +72,14 @@ public class WalletClient {
                 HttpStatusCode::isError,
                 (request, httpResponse) -> WalletFailurePolicy.throwFor(httpResponse))
             .body(CreditResponse.class);
-    return Objects.requireNonNull(response, "wallet forfeit response").operationGroupId();
+    return requireOperationGroupId(response);
+  }
+
+  private static UUID requireOperationGroupId(CreditResponse response) {
+    if (response == null || response.operationGroupId() == null) {
+      throw WalletFailurePolicy.malformedSuccess();
+    }
+    return response.operationGroupId();
   }
 
   private record CreditRequest(UUID userId, Money amount, String source, String reason) {}
diff --git a/src/main/java/com/sportsbook/settlement/client/WalletFailurePolicy.java b/src/main/java/com/sportsbook/settlement/client/WalletFailurePolicy.java
index deae4f0..f2d38f8 100644
--- a/src/main/java/com/sportsbook/settlement/client/WalletFailurePolicy.java
+++ b/src/main/java/com/sportsbook/settlement/client/WalletFailurePolicy.java
@@ -19,6 +19,10 @@ public final class WalletFailurePolicy {
     throw new PermanentFailure(status, errorCode);
   }
 
+  public static TransientFailure malformedSuccess() {
+    return new TransientFailure(200, "WALLET_MALFORMED_RESPONSE");
+  }
+
   private static String readErrorCode(ClientHttpResponse response, int status) throws IOException {
     try {
       WalletProblemDetail problem = JSON.readValue(response.getBody(), WalletProblemDetail.class);


## `test(wallet): reject mismatched operation proofs`

diff --git a/src/test/java/com/sportsbook/settlement/client/WalletMalformedSuccessTest.java b/src/test/java/com/sportsbook/settlement/client/WalletMalformedSuccessTest.java
index 5fc10b1..8815808 100644
--- a/src/test/java/com/sportsbook/settlement/client/WalletMalformedSuccessTest.java
+++ b/src/test/java/com/sportsbook/settlement/client/WalletMalformedSuccessTest.java
@@ -6,6 +6,7 @@ import static org.springframework.test.web.client.response.MockRestResponseCreat
 
 import com.sportsbook.protocol.value.Money;
 import java.net.URI;
+import java.util.List;
 import java.util.UUID;
 import org.junit.jupiter.api.Test;
 import org.springframework.http.MediaType;
@@ -15,7 +16,27 @@ import org.springframework.web.client.RestClient;
 class WalletMalformedSuccessTest {
 
   @Test
-  void treatsMissingOperationIdentityAsTransientMalformedSuccess() {
+  void treatsIncompleteOrMismatchedOperationProofsAsTransient() {
+    UUID userId = UUID.randomUUID();
+    UUID operationId = UUID.randomUUID();
+    String exact =
+        """
+        {"operationGroupId":"%s","userId":"%s",
+         "amount":{"amount":100,"currency":"KRW"},"reason":"BET_REFUND",
+         "at":"2026-08-22T00:00:00Z"}
+        """
+            .formatted(operationId, userId);
+    List.of(
+            "{\"extra\":\"allowed\"}",
+            exact.replace(userId.toString(), UUID.randomUUID().toString()),
+            exact.replace("\"amount\":100", "\"amount\":101"),
+            exact.replace("KRW", "USD"),
+            exact.replace("BET_REFUND", "BET_PAYOUT"),
+            exact.replace("\"at\":\"2026-08-22T00:00:00Z\"", "\"at\":null"))
+        .forEach(body -> assertMalformed(userId, body));
+  }
+
+  private static void assertMalformed(UUID userId, String body) {
     RestClient.Builder builder = RestClient.builder();
     MockRestServiceServer server = MockRestServiceServer.bindTo(builder).build();
     WalletClient client =
@@ -26,13 +47,13 @@ class WalletMalformedSuccessTest {
                 new WalletCredentials("0123456789abcdef0123456789abcdef")));
     server
         .expect(requestTo("http://wallet.test" + WalletClient.CREDIT_PATH))
-        .andRespond(withSuccess("{\"extra\":\"allowed\"}", MediaType.APPLICATION_JSON));
+        .andRespond(withSuccess(body, MediaType.APPLICATION_JSON));
 
     assertThatThrownBy(
             () ->
                 client.credit(
                     "settlement:refund:test",
-                    UUID.randomUUID(),
+                    userId,
                     Money.krw(100),
                     WalletCreditPurpose.RETURNED_STAKE))
         .isInstanceOf(WalletFailurePolicy.TransientFailure.class)
