# 공통 오류 어휘와 Problem Detail

## `feat(errors): define protocol error codes`

diff --git a/src/main/java/com/sportsbook/protocol/error/ErrorCode.java b/src/main/java/com/sportsbook/protocol/error/ErrorCode.java
new file mode 100644
index 0000000..636b76b
--- /dev/null
+++ b/src/main/java/com/sportsbook/protocol/error/ErrorCode.java
@@ -0,0 +1,39 @@
+package com.sportsbook.protocol.error;
+
+import java.net.URI;
+
+/** Stable error vocabulary shared by sportsbook service boundaries. */
+public enum ErrorCode {
+  VALIDATION_FAILED(400, "validation-failed", "Validation failed"),
+  ODDS_DRIFT(409, "odds-drift", "Odds drift exceeds tolerance"),
+  DUPLICATE_BET(409, "duplicate-bet", "Duplicate bet (idempotency violation)"),
+  INSUFFICIENT_BALANCE(409, "insufficient-balance", "Insufficient wallet balance"),
+  LIMIT_EXCEEDED(403, "limit-exceeded", "User or market limit exceeded"),
+  EVENT_CLOSED(422, "event-closed", "Event is closed for betting"),
+  SERVICE_UNAVAILABLE(503, "service-unavailable", "Dependent service unavailable"),
+  INTERNAL_ERROR(500, "internal-error", "Internal server error");
+
+  private static final String TYPE_BASE = "https://sportsbook/errors/";
+
+  private final int httpStatus;
+  private final URI type;
+  private final String title;
+
+  ErrorCode(int httpStatus, String type, String title) {
+    this.httpStatus = httpStatus;
+    this.type = URI.create(TYPE_BASE + type);
+    this.title = title;
+  }
+
+  public int httpStatus() {
+    return httpStatus;
+  }
+
+  public URI type() {
+    return type;
+  }
+
+  public String title() {
+    return title;
+  }
+}


## `test(errors): verify error classifications`

diff --git a/src/test/java/com/sportsbook/protocol/error/ErrorCodeTest.java b/src/test/java/com/sportsbook/protocol/error/ErrorCodeTest.java
new file mode 100644
index 0000000..73e21d2
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/error/ErrorCodeTest.java
@@ -0,0 +1,38 @@
+package com.sportsbook.protocol.error;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.util.Arrays;
+import org.junit.jupiter.api.Test;
+
+class ErrorCodeTest {
+
+  @Test
+  void catalogContainsSharedBoundaryCodes() {
+    assertThat(ErrorCode.values())
+        .containsExactly(
+            ErrorCode.VALIDATION_FAILED,
+            ErrorCode.ODDS_DRIFT,
+            ErrorCode.DUPLICATE_BET,
+            ErrorCode.INSUFFICIENT_BALANCE,
+            ErrorCode.LIMIT_EXCEEDED,
+            ErrorCode.EVENT_CLOSED,
+            ErrorCode.SERVICE_UNAVAILABLE,
+            ErrorCode.INTERNAL_ERROR);
+  }
+
+  @Test
+  void statusesAreClientOrServerErrors() {
+    assertThat(Arrays.stream(ErrorCode.values()).map(ErrorCode::httpStatus))
+        .allMatch(status -> status >= 400 && status < 600);
+  }
+
+  @Test
+  void metadataIsStableAndComplete() {
+    assertThat(Arrays.stream(ErrorCode.values()).map(ErrorCode::title))
+        .allMatch(title -> !title.isBlank());
+    assertThat(Arrays.stream(ErrorCode.values()).map(ErrorCode::type))
+        .doesNotHaveDuplicates()
+        .allMatch(uri -> uri.toString().startsWith("https://sportsbook/errors/"));
+  }
+}


## `test(errors): verify retry semantics`

diff --git a/src/test/java/com/sportsbook/protocol/error/ErrorRetryTest.java b/src/test/java/com/sportsbook/protocol/error/ErrorRetryTest.java
new file mode 100644
index 0000000..e095974
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/error/ErrorRetryTest.java
@@ -0,0 +1,20 @@
+package com.sportsbook.protocol.error;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import org.junit.jupiter.api.Test;
+
+class ErrorRetryTest {
+
+  @Test
+  void unavailableDependencyUsesRetryableHttpStatus() {
+    assertThat(ErrorCode.SERVICE_UNAVAILABLE.httpStatus()).isEqualTo(503);
+  }
+
+  @Test
+  void businessRejectionsRemainClientErrors() {
+    assertThat(ErrorCode.ODDS_DRIFT.httpStatus()).isBetween(400, 499);
+    assertThat(ErrorCode.INSUFFICIENT_BALANCE.httpStatus()).isBetween(400, 499);
+    assertThat(ErrorCode.LIMIT_EXCEEDED.httpStatus()).isBetween(400, 499);
+  }
+}


## `feat(errors): define problem details`

diff --git a/src/main/java/com/sportsbook/protocol/error/ErrorCode.java b/src/main/java/com/sportsbook/protocol/error/ErrorCode.java
index 636b76b..bc98c06 100644
--- a/src/main/java/com/sportsbook/protocol/error/ErrorCode.java
+++ b/src/main/java/com/sportsbook/protocol/error/ErrorCode.java
@@ -36,4 +36,16 @@ public enum ErrorCode {
   public String title() {
     return title;
   }
+
+  public ProblemDetail toProblemDetail() {
+    return toProblemDetail(null, null, null);
+  }
+
+  public ProblemDetail toProblemDetail(String detail) {
+    return toProblemDetail(detail, null, null);
+  }
+
+  public ProblemDetail toProblemDetail(String detail, URI instance, String correlationId) {
+    return new ProblemDetail(type, title, httpStatus, name(), detail, instance, correlationId);
+  }
 }
diff --git a/src/main/java/com/sportsbook/protocol/error/ProblemDetail.java b/src/main/java/com/sportsbook/protocol/error/ProblemDetail.java
new file mode 100644
index 0000000..c3c58a1
--- /dev/null
+++ b/src/main/java/com/sportsbook/protocol/error/ProblemDetail.java
@@ -0,0 +1,33 @@
+package com.sportsbook.protocol.error;
+
+import com.fasterxml.jackson.annotation.JsonInclude;
+import java.net.URI;
+import java.util.Objects;
+
+/**
+ * RFC 7807 ProblemDetail with two sportsbook-specific extensions: {@code errorCode} (stable
+ * machine-readable identifier — the {@link ErrorCode} enum name) and {@code correlationId}
+ * (Micrometer/OTel trace id).
+ *
+ * <p>Self-defined rather than reusing {@code org.springframework.http.ProblemDetail} so this
+ * library stays framework-neutral. Spring's class pulls in spring-web transitively which is
+ * unwanted for non-web consumers (background workers, batch jobs).
+ *
+ * <p>{@code @JsonInclude(NON_NULL)} keeps the wire compact when optional fields are absent.
+ */
+@JsonInclude(JsonInclude.Include.NON_NULL)
+public record ProblemDetail(
+    URI type,
+    String title,
+    int status,
+    String errorCode,
+    String detail,
+    URI instance,
+    String correlationId) {
+
+  public ProblemDetail {
+    Objects.requireNonNull(type, "type");
+    Objects.requireNonNull(title, "title");
+    Objects.requireNonNull(errorCode, "errorCode");
+  }
+}


## `test(errors): verify problem detail invariants`

diff --git a/src/test/java/com/sportsbook/protocol/error/ProblemDetailTest.java b/src/test/java/com/sportsbook/protocol/error/ProblemDetailTest.java
new file mode 100644
index 0000000..5792e66
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/error/ProblemDetailTest.java
@@ -0,0 +1,50 @@
+package com.sportsbook.protocol.error;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import java.net.URI;
+import org.junit.jupiter.api.Test;
+
+class ProblemDetailTest {
+
+  private final ObjectMapper mapper = new ObjectMapper();
+
+  @Test
+  void errorCodeBuildsMinimalProblemDetail() {
+    ProblemDetail detail = ErrorCode.ODDS_DRIFT.toProblemDetail();
+    assertThat(detail.type()).isEqualTo(URI.create("https://sportsbook/errors/odds-drift"));
+    assertThat(detail.title()).isEqualTo("Odds drift exceeds tolerance");
+    assertThat(detail.status()).isEqualTo(409);
+    assertThat(detail.errorCode()).isEqualTo("ODDS_DRIFT");
+    assertThat(detail.detail()).isNull();
+  }
+
+  @Test
+  void extensionsCarryRequestContext() {
+    ProblemDetail detail =
+        ErrorCode.EVENT_CLOSED.toProblemDetail(
+            "Event is no longer open", URI.create("/api/v1/events/event-1"), "trace-1");
+    assertThat(detail.detail()).isEqualTo("Event is no longer open");
+    assertThat(detail.instance()).isEqualTo(URI.create("/api/v1/events/event-1"));
+    assertThat(detail.correlationId()).isEqualTo("trace-1");
+  }
+
+  @Test
+  void jsonOmitsNullExtensions() throws Exception {
+    String json =
+        mapper.writeValueAsString(
+            ErrorCode.DUPLICATE_BET.toProblemDetail("request key already exists"));
+    assertThat(json).doesNotContain("instance", "correlationId");
+    assertThat(json).contains("\"errorCode\":\"DUPLICATE_BET\"");
+  }
+
+  @Test
+  void populatedProblemDetailRoundTrips() throws Exception {
+    ProblemDetail original =
+        ErrorCode.EVENT_CLOSED.toProblemDetail(
+            "Event is no longer open", URI.create("/api/v1/events/event-1"), "trace-1");
+    assertThat(mapper.readValue(mapper.writeValueAsString(original), ProblemDetail.class))
+        .isEqualTo(original);
+  }
+}
