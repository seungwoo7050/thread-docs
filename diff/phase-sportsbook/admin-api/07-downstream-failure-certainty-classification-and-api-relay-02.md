## `feat(api): relay downstream rejections`

diff --git a/src/main/java/com/sportsbook/admin/api/AdminExceptionHandler.java b/src/main/java/com/sportsbook/admin/api/AdminExceptionHandler.java
new file mode 100644
index 0000000..187f492
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/api/AdminExceptionHandler.java
@@ -0,0 +1,23 @@
+package com.sportsbook.admin.api;
+
+import com.sportsbook.admin.client.DownstreamStatusException;
+import org.springframework.http.HttpHeaders;
+import org.springframework.http.MediaType;
+import org.springframework.http.ResponseEntity;
+import org.springframework.web.bind.annotation.ExceptionHandler;
+import org.springframework.web.bind.annotation.RestControllerAdvice;
+
+@RestControllerAdvice
+public final class AdminExceptionHandler {
+
+  @ExceptionHandler(DownstreamStatusException.class)
+  ResponseEntity<byte[]> relayDownstream(DownstreamStatusException failure) {
+    HttpHeaders headers = new HttpHeaders();
+    headers.setContentType(
+        failure.contentType() == null
+            ? MediaType.APPLICATION_PROBLEM_JSON
+            : failure.contentType());
+    headers.setCacheControl("no-store");
+    return new ResponseEntity<>(failure.body(), headers, failure.status());
+  }
+}


## `test(api): relay exact downstream rejection`

diff --git a/src/test/java/com/sportsbook/admin/api/DownstreamRejectionResponseTest.java b/src/test/java/com/sportsbook/admin/api/DownstreamRejectionResponseTest.java
new file mode 100644
index 0000000..8ea74c3
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/api/DownstreamRejectionResponseTest.java
@@ -0,0 +1,46 @@
+package com.sportsbook.admin.api;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.catchThrowableOfType;
+
+import com.sportsbook.admin.client.DownstreamFailureMapper;
+import com.sportsbook.admin.client.DownstreamStatusException;
+import java.nio.charset.StandardCharsets;
+import org.junit.jupiter.api.Test;
+import org.springframework.http.HttpHeaders;
+import org.springframework.http.HttpStatus;
+import org.springframework.http.MediaType;
+import org.springframework.web.client.HttpClientErrorException;
+
+class DownstreamRejectionResponseTest {
+
+  @Test
+  void relaysStatusContentTypeAndBodyWithoutCaching() {
+    byte[] body =
+        "{\"status\":409,\"code\":\"IDEMPOTENCY_CONFLICT\"}".getBytes(StandardCharsets.UTF_8);
+    HttpHeaders headers = new HttpHeaders();
+    headers.setContentType(MediaType.APPLICATION_PROBLEM_JSON);
+    DownstreamStatusException failure =
+        catchThrowableOfType(
+            () ->
+                new DownstreamFailureMapper()
+                    .execute(
+                        () -> {
+                          throw HttpClientErrorException.create(
+                              HttpStatus.CONFLICT,
+                              "Conflict",
+                              headers,
+                              body,
+                              StandardCharsets.UTF_8);
+                        }),
+            DownstreamStatusException.class);
+
+    var response = new AdminExceptionHandler().relayDownstream(failure);
+
+    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CONFLICT);
+    assertThat(response.getHeaders().getContentType())
+        .isEqualTo(MediaType.APPLICATION_PROBLEM_JSON);
+    assertThat(response.getHeaders().getCacheControl()).isEqualTo("no-store");
+    assertThat(response.getBody()).containsExactly(body);
+  }
+}
