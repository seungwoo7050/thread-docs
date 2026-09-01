# 성공 응답의 구조·의미 계약 검증

## `feat(client): reject malformed success responses`

diff --git a/src/main/java/com/sportsbook/admin/client/DownstreamContract.java b/src/main/java/com/sportsbook/admin/client/DownstreamContract.java
new file mode 100644
index 0000000..c88eb74
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/client/DownstreamContract.java
@@ -0,0 +1,30 @@
+package com.sportsbook.admin.client;
+
+import java.util.function.Predicate;
+import org.springframework.http.HttpStatus;
+import org.springframework.http.ResponseEntity;
+
+public final class DownstreamContract {
+
+  private DownstreamContract() {}
+
+  public static <T> T requireBody(
+      ResponseEntity<T> response, HttpStatus expectedStatus, Predicate<T> proof, String contract) {
+    T body = response.getBody();
+    if (response.getStatusCode().value() != expectedStatus.value()
+        || body == null
+        || !proof.test(body)) {
+      throw new DownstreamContractException(contract);
+    }
+    return body;
+  }
+
+  public static void requireEmpty(
+      ResponseEntity<byte[]> response, HttpStatus expectedStatus, String contract) {
+    byte[] body = response.getBody();
+    if (response.getStatusCode().value() != expectedStatus.value()
+        || (body != null && body.length != 0)) {
+      throw new DownstreamContractException(contract);
+    }
+  }
+}
diff --git a/src/main/java/com/sportsbook/admin/client/DownstreamContractException.java b/src/main/java/com/sportsbook/admin/client/DownstreamContractException.java
new file mode 100644
index 0000000..2f3192b
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/client/DownstreamContractException.java
@@ -0,0 +1,8 @@
+package com.sportsbook.admin.client;
+
+public final class DownstreamContractException extends RuntimeException {
+
+  DownstreamContractException(String contract) {
+    super("Downstream success response violated contract: " + contract);
+  }
+}


## `test(client): detect downstream contract violations`

diff --git a/src/test/java/com/sportsbook/admin/client/DownstreamContractTest.java b/src/test/java/com/sportsbook/admin/client/DownstreamContractTest.java
new file mode 100644
index 0000000..c9844af
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/client/DownstreamContractTest.java
@@ -0,0 +1,64 @@
+package com.sportsbook.admin.client;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import java.nio.charset.StandardCharsets;
+import org.junit.jupiter.api.Test;
+import org.springframework.http.HttpStatus;
+import org.springframework.http.ResponseEntity;
+
+class DownstreamContractTest {
+
+  @Test
+  void acceptsOnlyTheExactStatusAndProvenResponseBody() {
+    Receipt receipt = new Receipt("operation-1", true);
+
+    assertThat(
+            DownstreamContract.requireBody(
+                ResponseEntity.ok(receipt), HttpStatus.OK, Receipt::valid, "wallet receipt"))
+        .isSameAs(receipt);
+    assertThatThrownBy(
+            () ->
+                DownstreamContract.requireBody(
+                    ResponseEntity.status(HttpStatus.ACCEPTED).body(receipt),
+                    HttpStatus.OK,
+                    Receipt::valid,
+                    "wallet receipt"))
+        .isInstanceOf(DownstreamContractException.class);
+    assertThatThrownBy(
+            () ->
+                DownstreamContract.requireBody(
+                    ResponseEntity.ok(new Receipt(null, true)),
+                    HttpStatus.OK,
+                    Receipt::valid,
+                    "wallet receipt"))
+        .isInstanceOf(DownstreamContractException.class);
+    assertThatThrownBy(
+            () ->
+                DownstreamContract.requireBody(
+                    ResponseEntity.ok().build(), HttpStatus.OK, Receipt::valid, "wallet receipt"))
+        .isInstanceOf(DownstreamContractException.class);
+  }
+
+  @Test
+  void requiresAnExactlyEmptyBodyForAcknowledgementResponses() {
+    DownstreamContract.requireEmpty(
+        ResponseEntity.status(HttpStatus.ACCEPTED).build(), HttpStatus.ACCEPTED, "odds ack");
+
+    assertThatThrownBy(
+            () ->
+                DownstreamContract.requireEmpty(
+                    ResponseEntity.status(HttpStatus.ACCEPTED)
+                        .body("unexpected".getBytes(StandardCharsets.UTF_8)),
+                    HttpStatus.ACCEPTED,
+                    "odds ack"))
+        .isInstanceOf(DownstreamContractException.class);
+  }
+
+  private record Receipt(String operationId, boolean accepted) {
+    boolean valid() {
+      return operationId != null && accepted;
+    }
+  }
+}


## `feat(client): classify malformed success bodies`

diff --git a/src/main/java/com/sportsbook/admin/client/DownstreamContractException.java b/src/main/java/com/sportsbook/admin/client/DownstreamContractException.java
index 2f3192b..1dafb9a 100644
--- a/src/main/java/com/sportsbook/admin/client/DownstreamContractException.java
+++ b/src/main/java/com/sportsbook/admin/client/DownstreamContractException.java
@@ -5,4 +5,8 @@ public final class DownstreamContractException extends RuntimeException {
   DownstreamContractException(String contract) {
     super("Downstream success response violated contract: " + contract);
   }
+
+  DownstreamContractException(String contract, Throwable cause) {
+    super("Downstream success response violated contract: " + contract, cause);
+  }
 }
diff --git a/src/main/java/com/sportsbook/admin/client/DownstreamFailureMapper.java b/src/main/java/com/sportsbook/admin/client/DownstreamFailureMapper.java
index f69c89e..3462b94 100644
--- a/src/main/java/com/sportsbook/admin/client/DownstreamFailureMapper.java
+++ b/src/main/java/com/sportsbook/admin/client/DownstreamFailureMapper.java
@@ -4,7 +4,9 @@ import java.net.SocketTimeoutException;
 import java.util.function.Supplier;
 import org.springframework.http.HttpHeaders;
 import org.springframework.http.MediaType;
+import org.springframework.http.converter.HttpMessageConversionException;
 import org.springframework.web.client.HttpClientErrorException;
+import org.springframework.web.client.RestClientException;
 import org.springframework.web.client.HttpServerErrorException;
 import org.springframework.web.client.ResourceAccessException;
 
@@ -29,6 +31,12 @@ public final class DownstreamFailureMapper {
               ? DownstreamUnavailableException.Reason.TIMEOUT
               : DownstreamUnavailableException.Reason.TRANSPORT;
       throw new DownstreamUnavailableException(reason, null, transportFailure);
+    } catch (RestClientException clientFailure) {
+      if (hasCause(clientFailure, HttpMessageConversionException.class)) {
+        throw new DownstreamContractException("deserializable response body", clientFailure);
+      }
+      throw new DownstreamUnavailableException(
+          DownstreamUnavailableException.Reason.TRANSPORT, null, clientFailure);
     }
   }
 


## `test(client): classify malformed success bodies`

diff --git a/src/test/java/com/sportsbook/admin/client/DownstreamMalformedSuccessTest.java b/src/test/java/com/sportsbook/admin/client/DownstreamMalformedSuccessTest.java
new file mode 100644
index 0000000..05e6e1a
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/client/DownstreamMalformedSuccessTest.java
@@ -0,0 +1,37 @@
+package com.sportsbook.admin.client;
+
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.requestTo;
+import static org.springframework.test.web.client.response.MockRestResponseCreators.withSuccess;
+
+import org.junit.jupiter.api.Test;
+import org.springframework.http.MediaType;
+import org.springframework.test.web.client.MockRestServiceServer;
+import org.springframework.web.client.RestClient;
+
+class DownstreamMalformedSuccessTest {
+
+  @Test
+  void classifiesAnUnreadableSuccessBodyAsAContractViolation() {
+    RestClient.Builder builder =
+        RestClient.builder()
+            .baseUrl("http://downstream.test")
+            .defaultHeader(DownstreamHeaders.SERVICE_NAME, DownstreamHeaders.ADMIN_API);
+    MockRestServiceServer server = MockRestServiceServer.bindTo(builder).build();
+    server
+        .expect(requestTo("http://downstream.test/malformed"))
+        .andRespond(withSuccess("{", MediaType.APPLICATION_JSON));
+
+    RestClient client = builder.build();
+    assertThatThrownBy(
+            () ->
+                new DownstreamFailureMapper()
+                    .execute(
+                        () -> client.get().uri("/malformed").retrieve().body(ProbeResponse.class)))
+        .isInstanceOf(DownstreamContractException.class)
+        .hasMessageContaining("deserializable response body");
+    server.verify();
+  }
+
+  private record ProbeResponse(String value) {}
+}
