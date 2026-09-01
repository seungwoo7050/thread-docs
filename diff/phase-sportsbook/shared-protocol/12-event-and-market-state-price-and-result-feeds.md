# 경기·마켓 상태와 가격·결과 피드

## `feat(events): define market status changes`

diff --git a/src/main/avro/com/sportsbook/protocol/event/MarketStatusChanged.avsc b/src/main/avro/com/sportsbook/protocol/event/MarketStatusChanged.avsc
new file mode 100644
index 0000000..27d595b
--- /dev/null
+++ b/src/main/avro/com/sportsbook/protocol/event/MarketStatusChanged.avsc
@@ -0,0 +1,21 @@
+{
+  "namespace": "com.sportsbook.protocol.event",
+  "type": "record",
+  "name": "MarketStatusChanged",
+  "doc": "A market opened, suspended, or closed for betting. Suspensions are common mid-match (goal scored, injury) and pause incoming slips for that market.",
+  "fields": [
+    {"name": "eventId", "type": "string"},
+    {"name": "marketId", "type": "string"},
+    {
+      "name": "previousStatus",
+      "type": {
+        "type": "enum",
+        "name": "MarketStatus",
+        "symbols": ["OPEN", "SUSPENDED", "CLOSED"]
+      }
+    },
+    {"name": "newStatus", "type": "MarketStatus"},
+    {"name": "reason", "type": ["null", "string"], "default": null, "doc": "Optional human-readable reason (e.g. \"goal scored\", \"market closed at kickoff\")."},
+    {"name": "occurredAt", "type": {"type": "long", "logicalType": "timestamp-millis"}}
+  ]
+}


## `test(events): verify market status changes`

diff --git a/src/test/java/com/sportsbook/protocol/event/MarketStatusChangedRecordTest.java b/src/test/java/com/sportsbook/protocol/event/MarketStatusChangedRecordTest.java
new file mode 100644
index 0000000..679a38c
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/event/MarketStatusChangedRecordTest.java
@@ -0,0 +1,32 @@
+package com.sportsbook.protocol.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.time.Instant;
+import org.junit.jupiter.api.Test;
+
+class MarketStatusChangedRecordTest {
+
+  @Test
+  void marketStatusChangeRoundTrips() throws Exception {
+    MarketStatusChanged expected =
+        MarketStatusChanged.newBuilder()
+            .setEventId("event-1")
+            .setMarketId("market-1")
+            .setPreviousStatus(MarketStatus.OPEN)
+            .setNewStatus(MarketStatus.SUSPENDED)
+            .setReason("goal scored")
+            .setOccurredAt(Instant.parse("2026-08-21T00:00:00Z"))
+            .build();
+    AvroRecordTestSupport.assertFields(
+        MarketStatusChanged.getClassSchema(),
+        "eventId",
+        "marketId",
+        "previousStatus",
+        "newStatus",
+        "reason",
+        "occurredAt");
+    assertThat(AvroRecordTestSupport.roundTrip(expected, MarketStatusChanged.class))
+        .isEqualTo(expected);
+  }
+}


## `feat(events): define event lifecycle changes`

diff --git a/src/main/avro/com/sportsbook/protocol/event/EventLifecycle.avsc b/src/main/avro/com/sportsbook/protocol/event/EventLifecycle.avsc
new file mode 100644
index 0000000..4e81c01
--- /dev/null
+++ b/src/main/avro/com/sportsbook/protocol/event/EventLifecycle.avsc
@@ -0,0 +1,19 @@
+{
+  "namespace": "com.sportsbook.protocol.event",
+  "type": "record",
+  "name": "EventLifecycle",
+  "doc": "Phase transition for a sports event (kickoff, end, cancellation, postponement). Triggers downstream actions: market opens/closes, settlement window, void cascade on CANCELLED.",
+  "fields": [
+    {"name": "eventId", "type": "string"},
+    {
+      "name": "status",
+      "type": {
+        "type": "enum",
+        "name": "EventLifecycleStatus",
+        "symbols": ["SCHEDULED", "IN_PLAY", "FINISHED", "CANCELLED", "POSTPONED"]
+      }
+    },
+    {"name": "occurredAt", "type": {"type": "long", "logicalType": "timestamp-millis"}, "doc": "When the transition happened."},
+    {"name": "scheduledStartAt", "type": {"type": "long", "logicalType": "timestamp-millis"}, "doc": "Originally scheduled kickoff; useful for POSTPONED diffing and UI display."}
+  ]
+}


## `test(events): verify lifecycle changes`

diff --git a/src/test/java/com/sportsbook/protocol/event/EventLifecycleRecordTest.java b/src/test/java/com/sportsbook/protocol/event/EventLifecycleRecordTest.java
new file mode 100644
index 0000000..fedca8c
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/event/EventLifecycleRecordTest.java
@@ -0,0 +1,23 @@
+package com.sportsbook.protocol.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.time.Instant;
+import org.junit.jupiter.api.Test;
+
+class EventLifecycleRecordTest {
+
+  @Test
+  void eventLifecycleChangeRoundTrips() throws Exception {
+    EventLifecycle expected =
+        EventLifecycle.newBuilder()
+            .setEventId("event-1")
+            .setStatus(EventLifecycleStatus.IN_PLAY)
+            .setOccurredAt(Instant.parse("2026-08-21T00:00:00Z"))
+            .setScheduledStartAt(Instant.parse("2026-08-21T00:00:00Z"))
+            .build();
+    AvroRecordTestSupport.assertFields(
+        EventLifecycle.getClassSchema(), "eventId", "status", "occurredAt", "scheduledStartAt");
+    assertThat(AvroRecordTestSupport.roundTrip(expected, EventLifecycle.class)).isEqualTo(expected);
+  }
+}


## `feat(events): define odds changes`

diff --git a/src/main/avro/com/sportsbook/protocol/event/OddsChanged.avsc b/src/main/avro/com/sportsbook/protocol/event/OddsChanged.avsc
new file mode 100644
index 0000000..eb81b1d
--- /dev/null
+++ b/src/main/avro/com/sportsbook/protocol/event/OddsChanged.avsc
@@ -0,0 +1,14 @@
+{
+  "namespace": "com.sportsbook.protocol.event",
+  "type": "record",
+  "name": "OddsChanged",
+  "doc": "A selection's price moved. Consumed by gateway (push to clients) and betting-service (slippage check against placed slips).",
+  "fields": [
+    {"name": "eventId", "type": "string"},
+    {"name": "marketId", "type": "string"},
+    {"name": "selectionId", "type": "string"},
+    {"name": "previousOdds", "type": "string", "doc": "Decimal odds as a plain-string BigDecimal (e.g. \"1.8500\", scale 4) — V1 keeps decimal logicalType out of the wire."},
+    {"name": "newOdds", "type": "string"},
+    {"name": "changedAt", "type": {"type": "long", "logicalType": "timestamp-millis"}}
+  ]
+}


## `test(events): verify odds changes`

diff --git a/src/test/java/com/sportsbook/protocol/event/OddsChangedRecordTest.java b/src/test/java/com/sportsbook/protocol/event/OddsChangedRecordTest.java
new file mode 100644
index 0000000..d94a682
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/event/OddsChangedRecordTest.java
@@ -0,0 +1,31 @@
+package com.sportsbook.protocol.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.time.Instant;
+import org.junit.jupiter.api.Test;
+
+class OddsChangedRecordTest {
+
+  @Test
+  void oddsChangeRoundTrips() throws Exception {
+    OddsChanged expected =
+        OddsChanged.newBuilder()
+            .setEventId("event-1")
+            .setMarketId("market-1")
+            .setSelectionId("selection-1")
+            .setPreviousOdds("1.8500")
+            .setNewOdds("1.9100")
+            .setChangedAt(Instant.parse("2026-08-21T00:00:00Z"))
+            .build();
+    AvroRecordTestSupport.assertFields(
+        OddsChanged.getClassSchema(),
+        "eventId",
+        "marketId",
+        "selectionId",
+        "previousOdds",
+        "newOdds",
+        "changedAt");
+    assertThat(AvroRecordTestSupport.roundTrip(expected, OddsChanged.class)).isEqualTo(expected);
+  }
+}


## `feat(events): define match results`

diff --git a/src/main/avro/com/sportsbook/protocol/event/MatchResult.avsc b/src/main/avro/com/sportsbook/protocol/event/MatchResult.avsc
new file mode 100644
index 0000000..da43512
--- /dev/null
+++ b/src/main/avro/com/sportsbook/protocol/event/MatchResult.avsc
@@ -0,0 +1,21 @@
+{
+  "namespace": "com.sportsbook.protocol.event",
+  "type": "record",
+  "name": "MatchResult",
+  "doc": "Final outcome of a sports event. Drives settlement-service's bet resolution. Distinct from EventLifecycle: lifecycle reports phase transitions, MatchResult delivers the data needed to settle.",
+  "fields": [
+    {"name": "eventId", "type": "string"},
+    {"name": "score", "type": "string", "doc": "Sport-specific score string (e.g. \"2-1\" for football, \"3-2,6-4,4-6,7-5\" for tennis)."},
+    {
+      "name": "finalStatus",
+      "type": {
+        "type": "enum",
+        "name": "MatchFinalStatus",
+        "symbols": ["COMPLETED", "ABANDONED", "VOIDED"]
+      },
+      "doc": "COMPLETED → settle normally. ABANDONED → settle markets that already resolved, void the rest (sport-specific rules in settlement-service). VOIDED → all bets refunded (BetStatus.VOIDED)."
+    },
+    {"name": "resultDetail", "type": {"type": "map", "values": "string"}, "doc": "Sport-specific details: yellow cards, possession, set scores, etc. Schema-less so a new sport doesn't trigger Avro evolution."},
+    {"name": "settledAt", "type": {"type": "long", "logicalType": "timestamp-millis"}}
+  ]
+}


## `test(events): verify match results`

diff --git a/src/test/java/com/sportsbook/protocol/event/MatchResultRecordTest.java b/src/test/java/com/sportsbook/protocol/event/MatchResultRecordTest.java
new file mode 100644
index 0000000..56ef662
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/event/MatchResultRecordTest.java
@@ -0,0 +1,30 @@
+package com.sportsbook.protocol.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.time.Instant;
+import java.util.Map;
+import org.junit.jupiter.api.Test;
+
+class MatchResultRecordTest {
+
+  @Test
+  void matchResultRoundTrips() throws Exception {
+    MatchResult expected =
+        MatchResult.newBuilder()
+            .setEventId("event-1")
+            .setScore("2-1")
+            .setFinalStatus(MatchFinalStatus.COMPLETED)
+            .setResultDetail(Map.of("home", "2", "away", "1"))
+            .setSettledAt(Instant.parse("2026-08-21T00:00:00Z"))
+            .build();
+    AvroRecordTestSupport.assertFields(
+        MatchResult.getClassSchema(),
+        "eventId",
+        "score",
+        "finalStatus",
+        "resultDetail",
+        "settledAt");
+    assertThat(AvroRecordTestSupport.roundTrip(expected, MatchResult.class)).isEqualTo(expected);
+  }
+}
