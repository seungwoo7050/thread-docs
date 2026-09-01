# 타입 안전 UUID 식별자

## `feat(identity): define event identities`

diff --git a/src/main/java/com/sportsbook/protocol/value/EventId.java b/src/main/java/com/sportsbook/protocol/value/EventId.java
new file mode 100644
index 0000000..41f8e6e
--- /dev/null
+++ b/src/main/java/com/sportsbook/protocol/value/EventId.java
@@ -0,0 +1,27 @@
+package com.sportsbook.protocol.value;
+
+import com.fasterxml.jackson.annotation.JsonCreator;
+import com.fasterxml.jackson.annotation.JsonValue;
+import java.util.Objects;
+import java.util.UUID;
+
+/**
+ * Typed ID wrapper for a betting Event (ADR-0003). UUID v7 generation lives in each consuming
+ * service's persistence layer (Hibernate {@code @UuidGenerator(style = TIME)}); shared-protocol
+ * only provides compile-time type safety.
+ *
+ * <p>{@code @JsonValue} on the component serializes as a raw UUID string ({@code "uuid-..."})
+ * rather than a nested object ({@code {"value":"..."}}). DTO consumers see {@code "eventId":
+ * "uuid"} which matches the typical REST shape.
+ */
+public record EventId(@JsonValue UUID value) {
+
+  public EventId {
+    Objects.requireNonNull(value, "value");
+  }
+
+  @JsonCreator
+  public static EventId of(UUID value) {
+    return new EventId(value);
+  }
+}
diff --git a/src/main/java/com/sportsbook/protocol/value/MarketId.java b/src/main/java/com/sportsbook/protocol/value/MarketId.java
new file mode 100644
index 0000000..52714f8
--- /dev/null
+++ b/src/main/java/com/sportsbook/protocol/value/MarketId.java
@@ -0,0 +1,19 @@
+package com.sportsbook.protocol.value;
+
+import com.fasterxml.jackson.annotation.JsonCreator;
+import com.fasterxml.jackson.annotation.JsonValue;
+import java.util.Objects;
+import java.util.UUID;
+
+/** Typed ID wrapper for a Market within an Event. See {@link EventId} for rationale. */
+public record MarketId(@JsonValue UUID value) {
+
+  public MarketId {
+    Objects.requireNonNull(value, "value");
+  }
+
+  @JsonCreator
+  public static MarketId of(UUID value) {
+    return new MarketId(value);
+  }
+}
diff --git a/src/main/java/com/sportsbook/protocol/value/SelectionId.java b/src/main/java/com/sportsbook/protocol/value/SelectionId.java
new file mode 100644
index 0000000..360d71d
--- /dev/null
+++ b/src/main/java/com/sportsbook/protocol/value/SelectionId.java
@@ -0,0 +1,19 @@
+package com.sportsbook.protocol.value;
+
+import com.fasterxml.jackson.annotation.JsonCreator;
+import com.fasterxml.jackson.annotation.JsonValue;
+import java.util.Objects;
+import java.util.UUID;
+
+/** Typed ID wrapper for a Selection within a Market. See {@link EventId} for rationale. */
+public record SelectionId(@JsonValue UUID value) {
+
+  public SelectionId {
+    Objects.requireNonNull(value, "value");
+  }
+
+  @JsonCreator
+  public static SelectionId of(UUID value) {
+    return new SelectionId(value);
+  }
+}


## `test(identity): verify event identity invariants`

diff --git a/src/test/java/com/sportsbook/protocol/value/EventIdentityTest.java b/src/test/java/com/sportsbook/protocol/value/EventIdentityTest.java
new file mode 100644
index 0000000..bc18e24
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/value/EventIdentityTest.java
@@ -0,0 +1,40 @@
+package com.sportsbook.protocol.value;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatNullPointerException;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class EventIdentityTest {
+
+  private final ObjectMapper mapper = new ObjectMapper();
+
+  @Test
+  void eachIdentityWrapsItsUuid() {
+    UUID value = UUID.randomUUID();
+    assertThat(EventId.of(value).value()).isEqualTo(value);
+    assertThat(MarketId.of(value).value()).isEqualTo(value);
+    assertThat(SelectionId.of(value).value()).isEqualTo(value);
+  }
+
+  @Test
+  void identitiesRemainTypeDistinct() {
+    UUID value = UUID.randomUUID();
+    assertThat((Object) EventId.of(value)).isNotEqualTo(MarketId.of(value));
+  }
+
+  @Test
+  void nullUuidIsRejected() {
+    assertThatNullPointerException().isThrownBy(() -> EventId.of(null));
+    assertThatNullPointerException().isThrownBy(() -> MarketId.of(null));
+  }
+
+  @Test
+  void jsonUsesCanonicalUuidString() throws Exception {
+    EventId id = EventId.of(UUID.fromString("018f0000-0000-7000-8000-000000000001"));
+    assertThat(mapper.writeValueAsString(id)).isEqualTo("\"018f0000-0000-7000-8000-000000000001\"");
+    assertThat(mapper.readValue(mapper.writeValueAsString(id), EventId.class)).isEqualTo(id);
+  }
+}


## `feat(identity): define account identities`

diff --git a/src/main/java/com/sportsbook/protocol/value/BetId.java b/src/main/java/com/sportsbook/protocol/value/BetId.java
new file mode 100644
index 0000000..d677af5
--- /dev/null
+++ b/src/main/java/com/sportsbook/protocol/value/BetId.java
@@ -0,0 +1,19 @@
+package com.sportsbook.protocol.value;
+
+import com.fasterxml.jackson.annotation.JsonCreator;
+import com.fasterxml.jackson.annotation.JsonValue;
+import java.util.Objects;
+import java.util.UUID;
+
+/** Typed ID wrapper for a Bet slip. See {@link EventId} for rationale. */
+public record BetId(@JsonValue UUID value) {
+
+  public BetId {
+    Objects.requireNonNull(value, "value");
+  }
+
+  @JsonCreator
+  public static BetId of(UUID value) {
+    return new BetId(value);
+  }
+}
diff --git a/src/main/java/com/sportsbook/protocol/value/UserId.java b/src/main/java/com/sportsbook/protocol/value/UserId.java
new file mode 100644
index 0000000..ac5c0c6
--- /dev/null
+++ b/src/main/java/com/sportsbook/protocol/value/UserId.java
@@ -0,0 +1,19 @@
+package com.sportsbook.protocol.value;
+
+import com.fasterxml.jackson.annotation.JsonCreator;
+import com.fasterxml.jackson.annotation.JsonValue;
+import java.util.Objects;
+import java.util.UUID;
+
+/** Typed ID wrapper for an end User. See {@link EventId} for rationale. */
+public record UserId(@JsonValue UUID value) {
+
+  public UserId {
+    Objects.requireNonNull(value, "value");
+  }
+
+  @JsonCreator
+  public static UserId of(UUID value) {
+    return new UserId(value);
+  }
+}


## `test(identity): verify account identity invariants`

diff --git a/src/test/java/com/sportsbook/protocol/value/AccountIdentityTest.java b/src/test/java/com/sportsbook/protocol/value/AccountIdentityTest.java
new file mode 100644
index 0000000..7bf4bb2
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/value/AccountIdentityTest.java
@@ -0,0 +1,38 @@
+package com.sportsbook.protocol.value;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatNullPointerException;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class AccountIdentityTest {
+
+  private final ObjectMapper mapper = new ObjectMapper();
+
+  @Test
+  void accountIdentitiesWrapUuidValues() {
+    UUID value = UUID.randomUUID();
+    assertThat(BetId.of(value).value()).isEqualTo(value);
+    assertThat(UserId.of(value).value()).isEqualTo(value);
+  }
+
+  @Test
+  void accountIdentitiesRemainTypeDistinct() {
+    UUID value = UUID.randomUUID();
+    assertThat((Object) BetId.of(value)).isNotEqualTo(UserId.of(value));
+  }
+
+  @Test
+  void nullUuidIsRejected() {
+    assertThatNullPointerException().isThrownBy(() -> BetId.of(null));
+    assertThatNullPointerException().isThrownBy(() -> UserId.of(null));
+  }
+
+  @Test
+  void jsonRoundTripsAsUuidString() throws Exception {
+    UserId id = UserId.of(UUID.randomUUID());
+    assertThat(mapper.readValue(mapper.writeValueAsString(id), UserId.class)).isEqualTo(id);
+  }
+}
