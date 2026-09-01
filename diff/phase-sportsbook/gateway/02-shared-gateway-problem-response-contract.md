# 공통 게이트웨이 Problem 응답 계약

## `feat(errors): define gateway problem responses`

diff --git a/src/main/java/com/sportsbook/gateway/error/GatewayErrorCode.java b/src/main/java/com/sportsbook/gateway/error/GatewayErrorCode.java
new file mode 100644
index 0000000..f046e46
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/error/GatewayErrorCode.java
@@ -0,0 +1,42 @@
+package com.sportsbook.gateway.error;
+
+import java.net.URI;
+
+public enum GatewayErrorCode {
+  GATEWAY_UNAUTHORIZED(401, "unauthorized", "Unauthorized", "Authentication is required."),
+  GATEWAY_FORBIDDEN(403, "forbidden", "Forbidden", "Access is denied."),
+  GATEWAY_RATE_LIMITED(429, "rate-limited", "Too Many Requests", "Request rate limit exceeded."),
+  GATEWAY_BAD_GATEWAY(
+      502, "upstream-unavailable", "Bad Gateway", "An upstream service is unavailable."),
+  GATEWAY_TIMEOUT(504, "upstream-timeout", "Gateway Timeout", "An upstream service timed out.");
+
+  private static final String TYPE_BASE = "https://sportsbook/errors/";
+
+  private final int status;
+  private final URI type;
+  private final String title;
+  private final String detail;
+
+  GatewayErrorCode(int status, String type, String title, String detail) {
+    this.status = status;
+    this.type = URI.create(TYPE_BASE + type);
+    this.title = title;
+    this.detail = detail;
+  }
+
+  public int status() {
+    return status;
+  }
+
+  public URI type() {
+    return type;
+  }
+
+  public String title() {
+    return title;
+  }
+
+  public String detail() {
+    return detail;
+  }
+}
diff --git a/src/main/java/com/sportsbook/gateway/error/GatewayProblemWriter.java b/src/main/java/com/sportsbook/gateway/error/GatewayProblemWriter.java
new file mode 100644
index 0000000..5884681
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/error/GatewayProblemWriter.java
@@ -0,0 +1,51 @@
+package com.sportsbook.gateway.error;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.protocol.error.ProblemDetail;
+import io.micrometer.tracing.Span;
+import io.micrometer.tracing.Tracer;
+import jakarta.servlet.http.HttpServletRequest;
+import jakarta.servlet.http.HttpServletResponse;
+import java.io.IOException;
+import java.net.URI;
+import java.util.UUID;
+import org.springframework.beans.factory.ObjectProvider;
+import org.springframework.http.MediaType;
+import org.springframework.stereotype.Component;
+
+@Component
+public final class GatewayProblemWriter {
+
+  private final ObjectMapper objectMapper;
+  private final ObjectProvider<Tracer> tracerProvider;
+
+  public GatewayProblemWriter(ObjectMapper objectMapper, ObjectProvider<Tracer> tracerProvider) {
+    this.objectMapper = objectMapper;
+    this.tracerProvider = tracerProvider;
+  }
+
+  public void write(
+      HttpServletRequest request, HttpServletResponse response, GatewayErrorCode error)
+      throws IOException {
+    response.setStatus(error.status());
+    response.setContentType(MediaType.APPLICATION_PROBLEM_JSON_VALUE);
+    objectMapper.writeValue(response.getOutputStream(), problem(request, error));
+  }
+
+  ProblemDetail problem(HttpServletRequest request, GatewayErrorCode error) {
+    return new ProblemDetail(
+        error.type(),
+        error.title(),
+        error.status(),
+        error.name(),
+        error.detail(),
+        URI.create(request.getRequestURI()),
+        correlationId());
+  }
+
+  private String correlationId() {
+    Tracer tracer = tracerProvider.getIfAvailable();
+    Span span = tracer == null ? null : tracer.currentSpan();
+    return span == null ? UUID.randomUUID().toString() : span.context().traceId();
+  }
+}


## `test(errors): verify gateway problem shapes`

diff --git a/src/test/java/com/sportsbook/gateway/error/GatewayProblemWriterTest.java b/src/test/java/com/sportsbook/gateway/error/GatewayProblemWriterTest.java
new file mode 100644
index 0000000..8867db1
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/error/GatewayProblemWriterTest.java
@@ -0,0 +1,42 @@
+package com.sportsbook.gateway.error;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.fasterxml.jackson.databind.JsonNode;
+import com.fasterxml.jackson.databind.ObjectMapper;
+import io.micrometer.tracing.Tracer;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.support.DefaultListableBeanFactory;
+import org.springframework.http.MediaType;
+import org.springframework.mock.web.MockHttpServletRequest;
+import org.springframework.mock.web.MockHttpServletResponse;
+
+class GatewayProblemWriterTest {
+
+  private final ObjectMapper objectMapper = new ObjectMapper();
+
+  @Test
+  void writesStableProblemContractWithoutRequestSecrets() throws Exception {
+    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api/v1/bets");
+    request.addHeader("Authorization", "Bearer must-not-leak");
+    MockHttpServletResponse response = new MockHttpServletResponse();
+    DefaultListableBeanFactory beans = new DefaultListableBeanFactory();
+    GatewayProblemWriter writer =
+        new GatewayProblemWriter(objectMapper, beans.getBeanProvider(Tracer.class));
+
+    writer.write(request, response, GatewayErrorCode.GATEWAY_UNAUTHORIZED);
+
+    JsonNode body = objectMapper.readTree(response.getContentAsByteArray());
+    assertThat(response.getStatus()).isEqualTo(401);
+    assertThat(response.getContentType()).isEqualTo(MediaType.APPLICATION_PROBLEM_JSON_VALUE);
+    assertThat(body.get("type").asText()).isEqualTo("https://sportsbook/errors/unauthorized");
+    assertThat(body.get("title").asText()).isEqualTo("Unauthorized");
+    assertThat(body.get("status").asInt()).isEqualTo(401);
+    assertThat(body.get("errorCode").asText()).isEqualTo("GATEWAY_UNAUTHORIZED");
+    assertThat(body.get("detail").asText()).isEqualTo("Authentication is required.");
+    assertThat(body.get("instance").asText()).isEqualTo("/api/v1/bets");
+    assertThat(UUID.fromString(body.get("correlationId").asText())).isNotNull();
+    assertThat(response.getContentAsString()).doesNotContain("must-not-leak");
+  }
+}
