# 하위 실패의 확정·불확정 분류와 API 전달

## `feat(client): preserve downstream 4xx rejections`

diff --git a/src/main/java/com/sportsbook/admin/client/DownstreamFailureMapper.java b/src/main/java/com/sportsbook/admin/client/DownstreamFailureMapper.java
new file mode 100644
index 0000000..b327f31
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/client/DownstreamFailureMapper.java
@@ -0,0 +1,28 @@
+package com.sportsbook.admin.client;
+
+import java.util.function.Supplier;
+import org.springframework.http.HttpHeaders;
+import org.springframework.http.MediaType;
+import org.springframework.web.client.HttpClientErrorException;
+
+public final class DownstreamFailureMapper {
+
+  public <T> T execute(Supplier<T> request) {
+    try {
+      return request.get();
+    } catch (HttpClientErrorException rejection) {
+      HttpHeaders headers = rejection.getResponseHeaders();
+      MediaType contentType = headers == null ? null : headers.getContentType();
+      throw new DownstreamStatusException(
+          rejection.getStatusCode(), contentType, rejection.getResponseBodyAsByteArray());
+    }
+  }
+
+  public void execute(Runnable request) {
+    execute(
+        () -> {
+          request.run();
+          return null;
+        });
+  }
+}
diff --git a/src/main/java/com/sportsbook/admin/client/DownstreamStatusException.java b/src/main/java/com/sportsbook/admin/client/DownstreamStatusException.java
new file mode 100644
index 0000000..d4de6dd
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/client/DownstreamStatusException.java
@@ -0,0 +1,31 @@
+package com.sportsbook.admin.client;
+
+import java.util.Arrays;
+import org.springframework.http.HttpStatusCode;
+import org.springframework.http.MediaType;
+
+public final class DownstreamStatusException extends RuntimeException {
+
+  private final HttpStatusCode status;
+  private final MediaType contentType;
+  private final byte[] body;
+
+  DownstreamStatusException(HttpStatusCode status, MediaType contentType, byte[] body) {
+    super("Downstream request was rejected with status " + status.value());
+    this.status = status;
+    this.contentType = contentType;
+    this.body = Arrays.copyOf(body, body.length);
+  }
+
+  public HttpStatusCode status() {
+    return status;
+  }
+
+  public MediaType contentType() {
+    return contentType;
+  }
+
+  public byte[] body() {
+    return Arrays.copyOf(body, body.length);
+  }
+}


## `test(client): relay downstream 4xx responses`

diff --git a/src/test/java/com/sportsbook/admin/client/DownstreamFailureMapperTest.java b/src/test/java/com/sportsbook/admin/client/DownstreamFailureMapperTest.java
new file mode 100644
index 0000000..9d43de3
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/client/DownstreamFailureMapperTest.java
@@ -0,0 +1,44 @@
+package com.sportsbook.admin.client;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.catchThrowableOfType;
+
+import java.nio.charset.StandardCharsets;
+import org.junit.jupiter.api.Test;
+import org.springframework.http.HttpHeaders;
+import org.springframework.http.HttpStatus;
+import org.springframework.http.MediaType;
+import org.springframework.web.client.HttpClientErrorException;
+
+class DownstreamFailureMapperTest {
+
+  private final DownstreamFailureMapper mapper = new DownstreamFailureMapper();
+
+  @Test
+  void preservesAnExactDownstream4xxStatusContentTypeAndBody() {
+    byte[] body = "{\"errorCode\":\"LIMIT_REJECTED\"}".getBytes(StandardCharsets.UTF_8);
+    HttpHeaders headers = new HttpHeaders();
+    headers.setContentType(MediaType.APPLICATION_PROBLEM_JSON);
+    HttpClientErrorException rejection =
+        HttpClientErrorException.create(
+            HttpStatus.UNPROCESSABLE_ENTITY, "rejected", headers, body, StandardCharsets.UTF_8);
+
+    DownstreamStatusException mapped =
+        catchThrowableOfType(
+            () ->
+                mapper.execute(
+                    (java.util.function.Supplier<Object>)
+                        () -> {
+                          throw rejection;
+                        }),
+            DownstreamStatusException.class);
+
+    assertThat(mapped.status()).isEqualTo(HttpStatus.UNPROCESSABLE_ENTITY);
+    assertThat(mapped.contentType()).isEqualTo(MediaType.APPLICATION_PROBLEM_JSON);
+    assertThat(mapped.body()).containsExactly(body);
+    assertThat(mapped.getMessage()).doesNotContain("LIMIT_REJECTED");
+    byte[] exposed = mapped.body();
+    exposed[0] = 0;
+    assertThat(mapped.body()).containsExactly(body);
+  }
+}


## `feat(client): classify ambiguous downstream failures`

diff --git a/src/main/java/com/sportsbook/admin/client/DownstreamFailureMapper.java b/src/main/java/com/sportsbook/admin/client/DownstreamFailureMapper.java
index b327f31..f69c89e 100644
--- a/src/main/java/com/sportsbook/admin/client/DownstreamFailureMapper.java
+++ b/src/main/java/com/sportsbook/admin/client/DownstreamFailureMapper.java
@@ -1,9 +1,12 @@
 package com.sportsbook.admin.client;
 
+import java.net.SocketTimeoutException;
 import java.util.function.Supplier;
 import org.springframework.http.HttpHeaders;
 import org.springframework.http.MediaType;
 import org.springframework.web.client.HttpClientErrorException;
+import org.springframework.web.client.HttpServerErrorException;
+import org.springframework.web.client.ResourceAccessException;
 
 public final class DownstreamFailureMapper {
 
@@ -15,6 +18,17 @@ public final class DownstreamFailureMapper {
       MediaType contentType = headers == null ? null : headers.getContentType();
       throw new DownstreamStatusException(
           rejection.getStatusCode(), contentType, rejection.getResponseBodyAsByteArray());
+    } catch (HttpServerErrorException serverError) {
+      throw new DownstreamUnavailableException(
+          DownstreamUnavailableException.Reason.SERVER_ERROR,
+          serverError.getStatusCode(),
+          serverError);
+    } catch (ResourceAccessException transportFailure) {
+      DownstreamUnavailableException.Reason reason =
+          hasCause(transportFailure, SocketTimeoutException.class)
+              ? DownstreamUnavailableException.Reason.TIMEOUT
+              : DownstreamUnavailableException.Reason.TRANSPORT;
+      throw new DownstreamUnavailableException(reason, null, transportFailure);
     }
   }
 
@@ -25,4 +39,13 @@ public final class DownstreamFailureMapper {
           return null;
         });
   }
+
+  private static boolean hasCause(Throwable failure, Class<? extends Throwable> type) {
+    for (Throwable cause = failure; cause != null; cause = cause.getCause()) {
+      if (type.isInstance(cause)) {
+        return true;
+      }
+    }
+    return false;
+  }
 }
diff --git a/src/main/java/com/sportsbook/admin/client/DownstreamUnavailableException.java b/src/main/java/com/sportsbook/admin/client/DownstreamUnavailableException.java
new file mode 100644
index 0000000..732b821
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/client/DownstreamUnavailableException.java
@@ -0,0 +1,29 @@
+package com.sportsbook.admin.client;
+
+import org.springframework.http.HttpStatusCode;
+
+public final class DownstreamUnavailableException extends RuntimeException {
+
+  public enum Reason {
+    SERVER_ERROR,
+    TIMEOUT,
+    TRANSPORT
+  }
+
+  private final Reason reason;
+  private final HttpStatusCode status;
+
+  DownstreamUnavailableException(Reason reason, HttpStatusCode status, Throwable cause) {
+    super("Downstream outcome is unknown: " + reason, cause);
+    this.reason = reason;
+    this.status = status;
+  }
+
+  public Reason reason() {
+    return reason;
+  }
+
+  public HttpStatusCode status() {
+    return status;
+  }
+}


## `test(client): mark timeout and 5xx outcomes unknown`

diff --git a/src/test/java/com/sportsbook/admin/client/DownstreamUnavailableMappingTest.java b/src/test/java/com/sportsbook/admin/client/DownstreamUnavailableMappingTest.java
new file mode 100644
index 0000000..ce2c5c7
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/client/DownstreamUnavailableMappingTest.java
@@ -0,0 +1,66 @@
+package com.sportsbook.admin.client;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.catchThrowableOfType;
+
+import java.net.ConnectException;
+import java.net.SocketTimeoutException;
+import java.util.concurrent.atomic.AtomicInteger;
+import java.util.function.Supplier;
+import org.junit.jupiter.api.Test;
+import org.springframework.http.HttpStatus;
+import org.springframework.web.client.HttpServerErrorException;
+import org.springframework.web.client.ResourceAccessException;
+
+class DownstreamUnavailableMappingTest {
+
+  private final DownstreamFailureMapper mapper = new DownstreamFailureMapper();
+
+  @Test
+  void classifies5xxAsAnUnknownServerOutcome() {
+    DownstreamUnavailableException mapped =
+        mapped(new HttpServerErrorException(HttpStatus.SERVICE_UNAVAILABLE));
+
+    assertThat(mapped.reason()).isEqualTo(DownstreamUnavailableException.Reason.SERVER_ERROR);
+    assertThat(mapped.status()).isEqualTo(HttpStatus.SERVICE_UNAVAILABLE);
+  }
+
+  @Test
+  void distinguishesTimeoutFromOtherTransportFailures() {
+    DownstreamUnavailableException timeout =
+        mapped(new ResourceAccessException("timed out", new SocketTimeoutException()));
+    DownstreamUnavailableException transport =
+        mapped(new ResourceAccessException("connection lost", new ConnectException()));
+
+    assertThat(timeout.reason()).isEqualTo(DownstreamUnavailableException.Reason.TIMEOUT);
+    assertThat(timeout.status()).isNull();
+    assertThat(transport.reason()).isEqualTo(DownstreamUnavailableException.Reason.TRANSPORT);
+  }
+
+  @Test
+  void neverRetriesAnAmbiguousMutationAutomatically() {
+    AtomicInteger attempts = new AtomicInteger();
+
+    mapped(
+        new ResourceAccessException("lost response", new SocketTimeoutException()), attempts);
+
+    assertThat(attempts).hasValue(1);
+  }
+
+  private DownstreamUnavailableException mapped(RuntimeException failure) {
+    return mapped(failure, new AtomicInteger());
+  }
+
+  private DownstreamUnavailableException mapped(
+      RuntimeException failure, AtomicInteger attempts) {
+    return catchThrowableOfType(
+        () ->
+            mapper.execute(
+                (Supplier<Object>)
+                    () -> {
+                      attempts.incrementAndGet();
+                      throw failure;
+                    }),
+        DownstreamUnavailableException.class);
+  }
+}


## `feat(client): recognize wrapped read timeouts`

diff --git a/src/main/java/com/sportsbook/admin/client/DownstreamFailureMapper.java b/src/main/java/com/sportsbook/admin/client/DownstreamFailureMapper.java
index 3462b94..0bd360f 100644
--- a/src/main/java/com/sportsbook/admin/client/DownstreamFailureMapper.java
+++ b/src/main/java/com/sportsbook/admin/client/DownstreamFailureMapper.java
@@ -6,9 +6,9 @@ import org.springframework.http.HttpHeaders;
 import org.springframework.http.MediaType;
 import org.springframework.http.converter.HttpMessageConversionException;
 import org.springframework.web.client.HttpClientErrorException;
-import org.springframework.web.client.RestClientException;
 import org.springframework.web.client.HttpServerErrorException;
 import org.springframework.web.client.ResourceAccessException;
+import org.springframework.web.client.RestClientException;
 
 public final class DownstreamFailureMapper {
 
@@ -35,8 +35,11 @@ public final class DownstreamFailureMapper {
       if (hasCause(clientFailure, HttpMessageConversionException.class)) {
         throw new DownstreamContractException("deserializable response body", clientFailure);
       }
-      throw new DownstreamUnavailableException(
-          DownstreamUnavailableException.Reason.TRANSPORT, null, clientFailure);
+      DownstreamUnavailableException.Reason reason =
+          hasCause(clientFailure, SocketTimeoutException.class)
+              ? DownstreamUnavailableException.Reason.TIMEOUT
+              : DownstreamUnavailableException.Reason.TRANSPORT;
+      throw new DownstreamUnavailableException(reason, null, clientFailure);
     }
   }
 


## `test(client): recognize wrapped read timeouts`

diff --git a/src/test/java/com/sportsbook/admin/client/DownstreamUnavailableMappingTest.java b/src/test/java/com/sportsbook/admin/client/DownstreamUnavailableMappingTest.java
index ce2c5c7..ab99df1 100644
--- a/src/test/java/com/sportsbook/admin/client/DownstreamUnavailableMappingTest.java
+++ b/src/test/java/com/sportsbook/admin/client/DownstreamUnavailableMappingTest.java
@@ -11,6 +11,7 @@ import org.junit.jupiter.api.Test;
 import org.springframework.http.HttpStatus;
 import org.springframework.web.client.HttpServerErrorException;
 import org.springframework.web.client.ResourceAccessException;
+import org.springframework.web.client.RestClientException;
 
 class DownstreamUnavailableMappingTest {
 
@@ -37,12 +38,20 @@ class DownstreamUnavailableMappingTest {
     assertThat(transport.reason()).isEqualTo(DownstreamUnavailableException.Reason.TRANSPORT);
   }
 
+  @Test
+  void recognizesReadTimeoutsWrappedDuringBodyExtraction() {
+    DownstreamUnavailableException timeout =
+        mapped(new RestClientException("extract response", new SocketTimeoutException()));
+
+    assertThat(timeout.reason()).isEqualTo(DownstreamUnavailableException.Reason.TIMEOUT);
+    assertThat(timeout.status()).isNull();
+  }
+
   @Test
   void neverRetriesAnAmbiguousMutationAutomatically() {
     AtomicInteger attempts = new AtomicInteger();
 
-    mapped(
-        new ResourceAccessException("lost response", new SocketTimeoutException()), attempts);
+    mapped(new ResourceAccessException("lost response", new SocketTimeoutException()), attempts);
 
     assertThat(attempts).hasValue(1);
   }
@@ -51,8 +60,7 @@ class DownstreamUnavailableMappingTest {
     return mapped(failure, new AtomicInteger());
   }
 
-  private DownstreamUnavailableException mapped(
-      RuntimeException failure, AtomicInteger attempts) {
+  private DownstreamUnavailableException mapped(RuntimeException failure, AtomicInteger attempts) {
     return catchThrowableOfType(
         () ->
             mapper.execute(


## `feat(api): map unknown downstream outcomes`

diff --git a/src/main/java/com/sportsbook/admin/api/AdminExceptionHandler.java b/src/main/java/com/sportsbook/admin/api/AdminExceptionHandler.java
index 187f492..8b8f200 100644
--- a/src/main/java/com/sportsbook/admin/api/AdminExceptionHandler.java
+++ b/src/main/java/com/sportsbook/admin/api/AdminExceptionHandler.java
@@ -1,7 +1,14 @@
 package com.sportsbook.admin.api;
 
+import com.sportsbook.admin.client.DownstreamContractException;
 import com.sportsbook.admin.client.DownstreamStatusException;
+import com.sportsbook.admin.client.DownstreamUnavailableException;
+import com.sportsbook.protocol.error.ProblemDetail;
+import jakarta.servlet.http.HttpServletRequest;
+import java.net.URI;
+import org.slf4j.MDC;
 import org.springframework.http.HttpHeaders;
+import org.springframework.http.HttpStatus;
 import org.springframework.http.MediaType;
 import org.springframework.http.ResponseEntity;
 import org.springframework.web.bind.annotation.ExceptionHandler;
@@ -10,6 +17,12 @@ import org.springframework.web.bind.annotation.RestControllerAdvice;
 @RestControllerAdvice
 public final class AdminExceptionHandler {
 
+  private static final URI BAD_GATEWAY = URI.create("https://sportsbook/errors/bad-gateway");
+  private static final URI GATEWAY_TIMEOUT =
+      URI.create("https://sportsbook/errors/gateway-timeout");
+  private static final URI CONTRACT_VIOLATION =
+      URI.create("https://sportsbook/errors/downstream-contract-violation");
+
   @ExceptionHandler(DownstreamStatusException.class)
   ResponseEntity<byte[]> relayDownstream(DownstreamStatusException failure) {
     HttpHeaders headers = new HttpHeaders();
@@ -20,4 +33,55 @@ public final class AdminExceptionHandler {
     headers.setCacheControl("no-store");
     return new ResponseEntity<>(failure.body(), headers, failure.status());
   }
+
+  @ExceptionHandler(DownstreamUnavailableException.class)
+  ResponseEntity<ProblemDetail> downstreamUnavailable(
+      DownstreamUnavailableException failure, HttpServletRequest request) {
+    if (failure.reason() == DownstreamUnavailableException.Reason.TIMEOUT) {
+      return problem(
+          HttpStatus.GATEWAY_TIMEOUT,
+          GATEWAY_TIMEOUT,
+          "GATEWAY_TIMEOUT",
+          "The downstream outcome is unknown after a timeout",
+          request);
+    }
+    return problem(
+        HttpStatus.BAD_GATEWAY,
+        BAD_GATEWAY,
+        "BAD_GATEWAY",
+        "The downstream outcome is unknown",
+        request);
+  }
+
+  @ExceptionHandler(DownstreamContractException.class)
+  ResponseEntity<ProblemDetail> downstreamContract(
+      DownstreamContractException failure, HttpServletRequest request) {
+    return problem(
+        HttpStatus.BAD_GATEWAY,
+        CONTRACT_VIOLATION,
+        "DOWNSTREAM_CONTRACT_VIOLATION",
+        "The downstream success response violated its contract",
+        request);
+  }
+
+  private static ResponseEntity<ProblemDetail> problem(
+      HttpStatus status,
+      URI type,
+      String code,
+      String detail,
+      HttpServletRequest request) {
+    ProblemDetail body =
+        new ProblemDetail(
+            type,
+            status.getReasonPhrase(),
+            status.value(),
+            code,
+            detail,
+            URI.create(request.getRequestURI()),
+            MDC.get("traceId"));
+    return ResponseEntity.status(status)
+        .contentType(MediaType.APPLICATION_PROBLEM_JSON)
+        .cacheControl(org.springframework.http.CacheControl.noStore())
+        .body(body);
+  }
 }


## `test(api): render unknown downstream outcomes`

diff --git a/src/test/java/com/sportsbook/admin/api/DownstreamUnknownResponseTest.java b/src/test/java/com/sportsbook/admin/api/DownstreamUnknownResponseTest.java
new file mode 100644
index 0000000..3842e8b
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/api/DownstreamUnknownResponseTest.java
@@ -0,0 +1,68 @@
+package com.sportsbook.admin.api;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.catchThrowableOfType;
+
+import com.sportsbook.admin.client.DownstreamContract;
+import com.sportsbook.admin.client.DownstreamContractException;
+import com.sportsbook.admin.client.DownstreamFailureMapper;
+import com.sportsbook.admin.client.DownstreamUnavailableException;
+import java.net.SocketTimeoutException;
+import org.junit.jupiter.api.Test;
+import org.springframework.http.HttpStatus;
+import org.springframework.http.MediaType;
+import org.springframework.http.ResponseEntity;
+import org.springframework.mock.web.MockHttpServletRequest;
+import org.springframework.web.client.ResourceAccessException;
+
+class DownstreamUnknownResponseTest {
+
+  @Test
+  void mapsTimeoutsToAnOpaqueGatewayTimeout() {
+    DownstreamUnavailableException failure =
+        catchThrowableOfType(
+            () ->
+                new DownstreamFailureMapper()
+                    .execute(
+                        () -> {
+                          throw new ResourceAccessException(
+                              "read timed out", new SocketTimeoutException("secret host"));
+                        }),
+            DownstreamUnavailableException.class);
+
+    var response =
+        new AdminExceptionHandler().downstreamUnavailable(failure, request("/admin/v1/test"));
+
+    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.GATEWAY_TIMEOUT);
+    assertThat(response.getHeaders().getContentType())
+        .isEqualTo(MediaType.APPLICATION_PROBLEM_JSON);
+    assertThat(response.getBody().errorCode()).isEqualTo("GATEWAY_TIMEOUT");
+    assertThat(response.getBody().detail()).doesNotContain("secret host");
+  }
+
+  @Test
+  void mapsMalformedSuccessToAnOpaqueBadGateway() {
+    DownstreamContractException failure =
+        catchThrowableOfType(
+            () ->
+                DownstreamContract.requireBody(
+                    ResponseEntity.ok().build(),
+                    HttpStatus.OK,
+                    ignored -> true,
+                    "secret contract detail"),
+            DownstreamContractException.class);
+
+    var response =
+        new AdminExceptionHandler().downstreamContract(failure, request("/admin/v1/test"));
+
+    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.BAD_GATEWAY);
+    assertThat(response.getBody().errorCode()).isEqualTo("DOWNSTREAM_CONTRACT_VIOLATION");
+    assertThat(response.getBody().detail()).doesNotContain("secret contract detail");
+  }
+
+  private static MockHttpServletRequest request(String path) {
+    MockHttpServletRequest request = new MockHttpServletRequest();
+    request.setRequestURI(path);
+    return request;
+  }
+}


