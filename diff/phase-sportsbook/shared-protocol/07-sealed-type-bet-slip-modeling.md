# 봉인 타입 기반 베팅 조합

## `feat(bet): classify bet slips`

diff --git a/src/main/java/com/sportsbook/protocol/domain/BetSlipType.java b/src/main/java/com/sportsbook/protocol/domain/BetSlipType.java
new file mode 100644
index 0000000..ba40754
--- /dev/null
+++ b/src/main/java/com/sportsbook/protocol/domain/BetSlipType.java
@@ -0,0 +1,60 @@
+package com.sportsbook.protocol.domain;
+
+import com.fasterxml.jackson.annotation.JsonSubTypes;
+import com.fasterxml.jackson.annotation.JsonTypeInfo;
+
+/**
+ * Bet slip shape per ADR-0008: Single / Multiple / System(K-of-N). System is generalized (minWins,
+ * totalSelections) rather than enumerating named variants — Trixie / Yankee / Lucky 15 etc. stay as
+ * frontend labels so adding a new K-of-N flavor needs zero protocol change.
+ *
+ * <p>Jackson polymorphism via {@code @JsonTypeInfo} + {@code @JsonSubTypes} produces a tagged shape
+ * on the wire: {@code {"type":"SINGLE"}}, {@code {"type":"MULTIPLE"}}, {@code
+ * {"type":"SYSTEM","minWins":2,"totalSelections":3}}.
+ */
+@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "type")
+@JsonSubTypes({
+  @JsonSubTypes.Type(value = BetSlipType.Single.class, name = "SINGLE"),
+  @JsonSubTypes.Type(value = BetSlipType.Multiple.class, name = "MULTIPLE"),
+  @JsonSubTypes.Type(value = BetSlipType.System.class, name = "SYSTEM"),
+})
+public sealed interface BetSlipType
+    permits BetSlipType.Single, BetSlipType.Multiple, BetSlipType.System {
+
+  /** 1 selection per slip. */
+  record Single() implements BetSlipType {}
+
+  /** All N selections must win (parlay / accumulator). */
+  record Multiple() implements BetSlipType {}
+
+  /**
+   * K-of-N partial-win slip. {@link #MAX_TOTAL_SELECTIONS} bounds the structurally possible shapes
+   * (also keeps the C(total, min) combination explosion tractable). The runtime policy bound
+   * ({@code betting.policy.max-selections} in application.yml) may be tighter.
+   */
+  record System(int minWins, int totalSelections) implements BetSlipType {
+
+    public static final int MIN_TOTAL_SELECTIONS = 2;
+    public static final int MAX_TOTAL_SELECTIONS = 15;
+
+    public System {
+      if (totalSelections < MIN_TOTAL_SELECTIONS || totalSelections > MAX_TOTAL_SELECTIONS) {
+        throw new IllegalArgumentException(
+            "System.totalSelections ("
+                + totalSelections
+                + ") must be in "
+                + MIN_TOTAL_SELECTIONS
+                + ".."
+                + MAX_TOTAL_SELECTIONS);
+      }
+      if (minWins < 1 || minWins > totalSelections) {
+        throw new IllegalArgumentException(
+            "System.minWins ("
+                + minWins
+                + ") must be in 1..totalSelections ("
+                + totalSelections
+                + ")");
+      }
+    }
+  }
+}


## `test(bet): verify slip type invariants`

diff --git a/src/test/java/com/sportsbook/protocol/domain/BetSlipTypeTest.java b/src/test/java/com/sportsbook/protocol/domain/BetSlipTypeTest.java
new file mode 100644
index 0000000..55bbcb3
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/domain/BetSlipTypeTest.java
@@ -0,0 +1,95 @@
+package com.sportsbook.protocol.domain;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatIllegalArgumentException;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import org.junit.jupiter.api.Test;
+
+class BetSlipTypeTest {
+
+  private final ObjectMapper mapper = new ObjectMapper();
+
+  @Test
+  void singleAndMultipleEmptyRecordsCompareEqualByValue() {
+    assertThat(new BetSlipType.Single()).isEqualTo(new BetSlipType.Single());
+    assertThat(new BetSlipType.Multiple()).isEqualTo(new BetSlipType.Multiple());
+  }
+
+  @Test
+  void systemAcceptsValidBounds() {
+    BetSlipType.System sys = new BetSlipType.System(2, 3);
+    assertThat(sys.minWins()).isEqualTo(2);
+    assertThat(sys.totalSelections()).isEqualTo(3);
+  }
+
+  @Test
+  void systemBoundaryAcceptsMinAndMaxTotal() {
+    new BetSlipType.System(1, 2);
+    new BetSlipType.System(15, 15);
+  }
+
+  @Test
+  void systemRejectsTotalSelectionsBelowMin() {
+    assertThatIllegalArgumentException().isThrownBy(() -> new BetSlipType.System(1, 1));
+  }
+
+  @Test
+  void systemRejectsTotalSelectionsAboveMax() {
+    assertThatIllegalArgumentException().isThrownBy(() -> new BetSlipType.System(1, 16));
+  }
+
+  @Test
+  void systemRejectsMinWinsBelowOne() {
+    assertThatIllegalArgumentException().isThrownBy(() -> new BetSlipType.System(0, 3));
+  }
+
+  @Test
+  void systemRejectsMinWinsAboveTotalSelections() {
+    assertThatIllegalArgumentException().isThrownBy(() -> new BetSlipType.System(4, 3));
+  }
+
+  @Test
+  void singleJsonRoundTrip() throws Exception {
+    BetSlipType original = new BetSlipType.Single();
+    String json = mapper.writeValueAsString(original);
+    assertThat(json).isEqualTo("{\"type\":\"SINGLE\"}");
+    assertThat(mapper.readValue(json, BetSlipType.class)).isEqualTo(original);
+  }
+
+  @Test
+  void multipleJsonRoundTrip() throws Exception {
+    BetSlipType original = new BetSlipType.Multiple();
+    String json = mapper.writeValueAsString(original);
+    assertThat(json).isEqualTo("{\"type\":\"MULTIPLE\"}");
+    assertThat(mapper.readValue(json, BetSlipType.class)).isEqualTo(original);
+  }
+
+  @Test
+  void systemJsonRoundTripCarriesMinWinsAndTotal() throws Exception {
+    BetSlipType original = new BetSlipType.System(2, 3);
+    String json = mapper.writeValueAsString(original);
+    assertThat(json).isEqualTo("{\"type\":\"SYSTEM\",\"minWins\":2,\"totalSelections\":3}");
+    assertThat(mapper.readValue(json, BetSlipType.class)).isEqualTo(original);
+  }
+
+  @Test
+  void instanceofPatternsDispatchOverEachVariant() {
+    assertThat(label(new BetSlipType.Single())).isEqualTo("single");
+    assertThat(label(new BetSlipType.Multiple())).isEqualTo("multiple");
+    assertThat(label(new BetSlipType.System(2, 3))).isEqualTo("system-2-of-3");
+  }
+
+  private String label(BetSlipType type) {
+    if (type instanceof BetSlipType.System sys) {
+      return "system-" + sys.minWins() + "-of-" + sys.totalSelections();
+    }
+    if (type instanceof BetSlipType.Multiple) {
+      return "multiple";
+    }
+    if (type instanceof BetSlipType.Single) {
+      return "single";
+    }
+    throw new IllegalStateException("Unknown: " + type);
+  }
+}
