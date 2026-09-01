# 멱등성 키 Wire 계약

## `feat(idempotency): define request keys`

diff --git a/src/main/java/com/sportsbook/protocol/value/IdempotencyKey.java b/src/main/java/com/sportsbook/protocol/value/IdempotencyKey.java
new file mode 100644
index 0000000..f2997a2
--- /dev/null
+++ b/src/main/java/com/sportsbook/protocol/value/IdempotencyKey.java
@@ -0,0 +1,57 @@
+package com.sportsbook.protocol.value;
+
+import com.fasterxml.jackson.annotation.JsonCreator;
+import com.fasterxml.jackson.annotation.JsonValue;
+import java.util.Objects;
+import java.util.UUID;
+import java.util.regex.Pattern;
+
+/**
+ * Caller-supplied idempotency key for mutating requests (ADR-0005).
+ *
+ * <p>The first request with a given key succeeds and its outcome is recorded; any retry with the
+ * same key returns the same outcome rather than re-executing the side effect. The protocol layer
+ * here only validates the key's wire shape — the actual dedup happens in each service via a unique
+ * DB constraint (strong) plus a Redis cache (fast path), as described in ADR-0005.
+ *
+ * <p>Constraints:
+ *
+ * <ul>
+ *   <li>Non-blank, length ≤ 128 — fits comfortably in an HTTP header and a typical varchar column.
+ *   <li>Printable ASCII only ({@code 0x20–0x7E}) — avoids encoding ambiguity when the key crosses
+ *       Kafka headers, HTTP headers, and DB columns.
+ * </ul>
+ *
+ * <p>{@code toString} returns the raw value: idempotency keys are not secrets and showing them in
+ * logs is desirable for tracing a retry across services.
+ */
+public record IdempotencyKey(@JsonValue String value) {
+
+  public static final int MAX_LENGTH = 128;
+  private static final Pattern ASCII_PRINTABLE = Pattern.compile("\\A[\\x20-\\x7E]+\\z");
+
+  public IdempotencyKey {
+    Objects.requireNonNull(value, "value");
+    if (value.isBlank()) {
+      throw new IllegalArgumentException("IdempotencyKey must not be blank");
+    }
+    if (value.length() > MAX_LENGTH) {
+      throw new IllegalArgumentException(
+          "IdempotencyKey length " + value.length() + " exceeds max " + MAX_LENGTH);
+    }
+    if (!ASCII_PRINTABLE.matcher(value).matches()) {
+      throw new IllegalArgumentException(
+          "IdempotencyKey must contain only printable ASCII (0x20-0x7E)");
+    }
+  }
+
+  @JsonCreator
+  public static IdempotencyKey of(String value) {
+    return new IdempotencyKey(value);
+  }
+
+  /** Generates a fresh key as a UUID v4 string. Useful for clients that don't supply one. */
+  public static IdempotencyKey random() {
+    return new IdempotencyKey(UUID.randomUUID().toString());
+  }
+}


## `test(idempotency): verify canonical request keys`

diff --git a/src/test/java/com/sportsbook/protocol/value/IdempotencyKeyTest.java b/src/test/java/com/sportsbook/protocol/value/IdempotencyKeyTest.java
new file mode 100644
index 0000000..e7bfe57
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/value/IdempotencyKeyTest.java
@@ -0,0 +1,41 @@
+package com.sportsbook.protocol.value;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class IdempotencyKeyTest {
+
+  private final ObjectMapper mapper = new ObjectMapper();
+
+  @Test
+  void factoryBuildsCanonicalKey() {
+    IdempotencyKey key = IdempotencyKey.of("client-request-42");
+    assertThat(key.value()).isEqualTo("client-request-42");
+    assertThat(key).isEqualTo(new IdempotencyKey("client-request-42"));
+  }
+
+  @Test
+  void maximumLengthIsAccepted() {
+    IdempotencyKey key = IdempotencyKey.of("a".repeat(IdempotencyKey.MAX_LENGTH));
+    assertThat(key.value()).hasSize(IdempotencyKey.MAX_LENGTH);
+  }
+
+  @Test
+  void randomKeysAreDistinctUuidStrings() {
+    IdempotencyKey first = IdempotencyKey.random();
+    IdempotencyKey second = IdempotencyKey.random();
+    assertThat(first).isNotEqualTo(second);
+    assertThat(UUID.fromString(first.value()).toString()).isEqualTo(first.value());
+  }
+
+  @Test
+  void jsonRoundTripsAsRawString() throws Exception {
+    IdempotencyKey key = IdempotencyKey.of("client-request-42");
+    assertThat(mapper.writeValueAsString(key)).isEqualTo("\"client-request-42\"");
+    assertThat(mapper.readValue(mapper.writeValueAsString(key), IdempotencyKey.class))
+        .isEqualTo(key);
+  }
+}


## `test(idempotency): reject malformed request keys`

diff --git a/src/test/java/com/sportsbook/protocol/value/IdempotencyKeyValidationTest.java b/src/test/java/com/sportsbook/protocol/value/IdempotencyKeyValidationTest.java
new file mode 100644
index 0000000..78e9eed
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/value/IdempotencyKeyValidationTest.java
@@ -0,0 +1,36 @@
+package com.sportsbook.protocol.value;
+
+import static org.assertj.core.api.Assertions.assertThatIllegalArgumentException;
+import static org.assertj.core.api.Assertions.assertThatNullPointerException;
+
+import org.junit.jupiter.api.Test;
+
+class IdempotencyKeyValidationTest {
+
+  @Test
+  void nullAndBlankKeysAreRejected() {
+    assertThatNullPointerException().isThrownBy(() -> IdempotencyKey.of(null));
+    assertThatIllegalArgumentException().isThrownBy(() -> IdempotencyKey.of(""));
+    assertThatIllegalArgumentException().isThrownBy(() -> IdempotencyKey.of("   "));
+  }
+
+  @Test
+  void oversizedKeysAreRejected() {
+    assertThatIllegalArgumentException()
+        .isThrownBy(() -> IdempotencyKey.of("a".repeat(IdempotencyKey.MAX_LENGTH + 1)))
+        .withMessageContaining("exceeds max");
+  }
+
+  @Test
+  void nonAsciiKeysAreRejected() {
+    assertThatIllegalArgumentException()
+        .isThrownBy(() -> IdempotencyKey.of("요청"))
+        .withMessageContaining("printable ASCII");
+  }
+
+  @Test
+  void controlCharactersAreRejected() {
+    assertThatIllegalArgumentException().isThrownBy(() -> IdempotencyKey.of("line\nbreak"));
+    assertThatIllegalArgumentException().isThrownBy(() -> IdempotencyKey.of("tab\tkey"));
+  }
+}
