# 위험 거절과 탐지 신호

## `feat(events): define risk limit signals`

diff --git a/src/main/avro/com/sportsbook/protocol/event/RiskLimitViolated.avsc b/src/main/avro/com/sportsbook/protocol/event/RiskLimitViolated.avsc
new file mode 100644
index 0000000..d6ada24
--- /dev/null
+++ b/src/main/avro/com/sportsbook/protocol/event/RiskLimitViolated.avsc
@@ -0,0 +1,22 @@
+{
+  "namespace": "com.sportsbook.protocol.event",
+  "type": "record",
+  "name": "RiskLimitViolated",
+  "doc": "A bet placement attempt exceeded a per-user or per-market limit; rejected by risk-service before reaching wallet (ADR-0006 Saga compensation).",
+  "fields": [
+    {"name": "userId", "type": "string"},
+    {
+      "name": "limitType",
+      "type": {
+        "type": "enum",
+        "name": "RiskLimitType",
+        "symbols": ["STAKE_DAILY", "OPEN_EXPOSURE", "SELECTIONS_PER_MINUTE"]
+      },
+      "doc": "Which limit was breached. Drives the rejection message returned to the user."
+    },
+    {"name": "currentValue", "type": "long", "doc": "User's value before the attempt (monetary types in minor units, count types as raw long)."},
+    {"name": "limitValue", "type": "long", "doc": "Configured threshold."},
+    {"name": "requestedAmount", "type": "Money", "doc": "Stake of the attempted bet; meaningful for monetary limit types."},
+    {"name": "occurredAt", "type": {"type": "long", "logicalType": "timestamp-millis"}}
+  ]
+}


## `test(events): verify risk limit signals`

diff --git a/src/test/java/com/sportsbook/protocol/event/RiskLimitViolatedRecordTest.java b/src/test/java/com/sportsbook/protocol/event/RiskLimitViolatedRecordTest.java
new file mode 100644
index 0000000..7c3c124
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/event/RiskLimitViolatedRecordTest.java
@@ -0,0 +1,32 @@
+package com.sportsbook.protocol.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.time.Instant;
+import org.junit.jupiter.api.Test;
+
+class RiskLimitViolatedRecordTest {
+
+  @Test
+  void riskLimitSignalRoundTrips() throws Exception {
+    RiskLimitViolated expected =
+        RiskLimitViolated.newBuilder()
+            .setUserId("user-1")
+            .setLimitType(RiskLimitType.STAKE_DAILY)
+            .setCurrentValue(90_000)
+            .setLimitValue(100_000)
+            .setRequestedAmount(Money.newBuilder().setAmount(20_000).setCurrency("KRW").build())
+            .setOccurredAt(Instant.parse("2026-08-21T00:00:00Z"))
+            .build();
+    AvroRecordTestSupport.assertFields(
+        RiskLimitViolated.getClassSchema(),
+        "userId",
+        "limitType",
+        "currentValue",
+        "limitValue",
+        "requestedAmount",
+        "occurredAt");
+    assertThat(AvroRecordTestSupport.roundTrip(expected, RiskLimitViolated.class))
+        .isEqualTo(expected);
+  }
+}


## `feat(events): define risk pattern signals`

diff --git a/src/main/avro/com/sportsbook/protocol/event/RiskPatternSuspected.avsc b/src/main/avro/com/sportsbook/protocol/event/RiskPatternSuspected.avsc
new file mode 100644
index 0000000..e322d23
--- /dev/null
+++ b/src/main/avro/com/sportsbook/protocol/event/RiskPatternSuspected.avsc
@@ -0,0 +1,24 @@
+{
+  "namespace": "com.sportsbook.protocol.event",
+  "type": "record",
+  "name": "RiskPatternSuspected",
+  "doc": "Heuristic flagged a suspicious betting pattern. Not a hard rejection — risk-service may decide to throttle, alert, or escalate downstream.",
+  "fields": [
+    {"name": "userId", "type": "string"},
+    {
+      "name": "patternType",
+      "type": {
+        "type": "enum",
+        "name": "RiskPatternType",
+        "symbols": ["RAPID_BETTING", "SUDDEN_STAKE_INCREASE", "REPEATED_SAME_SELECTION"]
+      },
+      "doc": "Which heuristic matched."
+    },
+    {
+      "name": "evidence",
+      "type": {"type": "map", "values": "string"},
+      "doc": "Free-form key/value evidence (e.g. window seconds, stake delta, selection id) for ops/audit. Schema-less by design — adding a key requires no Avro evolution."
+    },
+    {"name": "occurredAt", "type": {"type": "long", "logicalType": "timestamp-millis"}}
+  ]
+}


## `test(events): verify risk pattern signals`

diff --git a/src/test/java/com/sportsbook/protocol/event/RiskPatternSuspectedRecordTest.java b/src/test/java/com/sportsbook/protocol/event/RiskPatternSuspectedRecordTest.java
new file mode 100644
index 0000000..2054205
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/event/RiskPatternSuspectedRecordTest.java
@@ -0,0 +1,25 @@
+package com.sportsbook.protocol.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.time.Instant;
+import java.util.Map;
+import org.junit.jupiter.api.Test;
+
+class RiskPatternSuspectedRecordTest {
+
+  @Test
+  void riskPatternSignalRoundTrips() throws Exception {
+    RiskPatternSuspected expected =
+        RiskPatternSuspected.newBuilder()
+            .setUserId("user-1")
+            .setPatternType(RiskPatternType.RAPID_BETTING)
+            .setEvidence(Map.of("windowSeconds", "60", "attempts", "12"))
+            .setOccurredAt(Instant.parse("2026-08-21T00:00:00Z"))
+            .build();
+    AvroRecordTestSupport.assertFields(
+        RiskPatternSuspected.getClassSchema(), "userId", "patternType", "evidence", "occurredAt");
+    assertThat(AvroRecordTestSupport.roundTrip(expected, RiskPatternSuspected.class))
+        .isEqualTo(expected);
+  }
+}
