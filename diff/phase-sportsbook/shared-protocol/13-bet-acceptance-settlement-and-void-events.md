# 베팅 수락·정산·무효화 이벤트

## `feat(events): define accepted bet placements`

diff --git a/src/main/avro/com/sportsbook/protocol/event/BetPlacedRequested.avsc b/src/main/avro/com/sportsbook/protocol/event/BetPlacedRequested.avsc
new file mode 100644
index 0000000..b1c11c0
--- /dev/null
+++ b/src/main/avro/com/sportsbook/protocol/event/BetPlacedRequested.avsc
@@ -0,0 +1,41 @@
+{
+  "namespace": "com.sportsbook.protocol.event",
+  "type": "record",
+  "name": "BetPlacedRequested",
+  "doc": "Accepted-bet event published by betting-service through its transactional outbox after risk reservation commit and wallet debit are confirmed. Consumers must remain idempotent because outbox delivery is at least once.",
+  "fields": [
+    {"name": "betId", "type": "string", "doc": "Slip UUID — primary key in betting-service."},
+    {"name": "userId", "type": "string"},
+    {
+      "name": "slipType",
+      "type": {
+        "type": "enum",
+        "name": "BetSlipTypeTag",
+        "symbols": ["SINGLE", "MULTIPLE", "SYSTEM"]
+      },
+      "doc": "Tag for the BetSlipType sealed interface. SYSTEM additionally requires systemMinWins + systemTotalSelections."
+    },
+    {"name": "systemMinWins", "type": ["null", "int"], "default": null, "doc": "K parameter for SYSTEM slips. null for SINGLE/MULTIPLE."},
+    {"name": "systemTotalSelections", "type": ["null", "int"], "default": null, "doc": "N parameter for SYSTEM slips. null for SINGLE/MULTIPLE; for SINGLE/MULTIPLE the value equals selections.size."},
+    {
+      "name": "selections",
+      "type": {
+        "type": "array",
+        "items": {
+          "type": "record",
+          "name": "RequestedSelection",
+          "fields": [
+            {"name": "eventId", "type": "string"},
+            {"name": "marketId", "type": "string"},
+            {"name": "selectionId", "type": "string"},
+            {"name": "oddsAtSubmission", "type": "string", "doc": "Decimal odds the user saw at submit time (\"1.8500\", scale 4). Slippage compared against the current odds before acceptance."}
+          ]
+        }
+      },
+      "doc": "Selections the user picked. Order preserved."
+    },
+    {"name": "stake", "type": "Money", "doc": "Wager amount; same Money record reused across the event schema family."},
+    {"name": "idempotencyKey", "type": "string", "doc": "Caller-supplied idempotency key (ADR-0005). Same key replays produce the same betId / outcome."},
+    {"name": "requestedAt", "type": {"type": "long", "logicalType": "timestamp-millis"}}
+  ]
+}


## `test(events): verify bet placement records`

diff --git a/src/test/java/com/sportsbook/protocol/event/BetPlacedRequestedRecordTest.java b/src/test/java/com/sportsbook/protocol/event/BetPlacedRequestedRecordTest.java
new file mode 100644
index 0000000..c3431ca
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/event/BetPlacedRequestedRecordTest.java
@@ -0,0 +1,46 @@
+package com.sportsbook.protocol.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.time.Instant;
+import java.util.List;
+import org.junit.jupiter.api.Test;
+
+class BetPlacedRequestedRecordTest {
+
+  @Test
+  void acceptedPlacementRoundTrips() throws Exception {
+    RequestedSelection selection =
+        RequestedSelection.newBuilder()
+            .setEventId("event-1")
+            .setMarketId("market-1")
+            .setSelectionId("selection-1")
+            .setOddsAtSubmission("1.8500")
+            .build();
+    BetPlacedRequested expected =
+        BetPlacedRequested.newBuilder()
+            .setBetId("bet-1")
+            .setUserId("user-1")
+            .setSlipType(BetSlipTypeTag.SINGLE)
+            .setSystemMinWins(null)
+            .setSystemTotalSelections(null)
+            .setSelections(List.of(selection))
+            .setStake(Money.newBuilder().setAmount(10_000).setCurrency("KRW").build())
+            .setIdempotencyKey("placement-1")
+            .setRequestedAt(Instant.parse("2026-08-21T00:00:00Z"))
+            .build();
+    AvroRecordTestSupport.assertFields(
+        BetPlacedRequested.getClassSchema(),
+        "betId",
+        "userId",
+        "slipType",
+        "systemMinWins",
+        "systemTotalSelections",
+        "selections",
+        "stake",
+        "idempotencyKey",
+        "requestedAt");
+    assertThat(AvroRecordTestSupport.roundTrip(expected, BetPlacedRequested.class))
+        .isEqualTo(expected);
+  }
+}


## `feat(events): define settled bet outcomes`

diff --git a/src/main/avro/com/sportsbook/protocol/event/BetSettled.avsc b/src/main/avro/com/sportsbook/protocol/event/BetSettled.avsc
new file mode 100644
index 0000000..a6bf2d6
--- /dev/null
+++ b/src/main/avro/com/sportsbook/protocol/event/BetSettled.avsc
@@ -0,0 +1,24 @@
+{
+  "namespace": "com.sportsbook.protocol.event",
+  "type": "record",
+  "name": "BetSettled",
+  "doc": "Settlement outcome for a bet (ADR-0006 async settlement). Published by settlement-service after MatchResult; betting-service consumes to flip bet.status to SETTLED. Distinct from BetVoided (no-action refund).",
+  "fields": [
+    {"name": "betId", "type": "string", "doc": "Slip UUID — primary key in betting-service."},
+    {"name": "userId", "type": "string"},
+    {"name": "eventId", "type": "string", "doc": "Driving event whose MatchResult triggered this settlement."},
+    {
+      "name": "result",
+      "type": {
+        "type": "enum",
+        "name": "SettlementResultAvro",
+        "symbols": ["WON", "LOST", "PUSH", "VOID"]
+      },
+      "doc": "Mirrors the shared-protocol SettlementResult enum (ADR-0013). WON/PUSH pay out, LOST pays nothing, VOID refunds the stake."
+    },
+    {"name": "stake", "type": "Money", "doc": "Original wager amount; same Money record reused across the event schema family."},
+    {"name": "payout", "type": "Money", "doc": "Amount credited to the wallet. Positive for WON/PUSH (PUSH returns the stake), zero for LOST."},
+    {"name": "settledAt", "type": {"type": "long", "logicalType": "timestamp-millis"}},
+    {"name": "resultDetail", "type": ["null", {"type": "map", "values": "string"}], "default": null, "doc": "Optional per-combination adjudication details (e.g. which legs of a SYSTEM slip won). Schema-less so new detail keys do not trigger Avro evolution."}
+  ]
+}


## `test(events): verify settled bet outcomes`

diff --git a/src/test/java/com/sportsbook/protocol/event/BetSettledRecordTest.java b/src/test/java/com/sportsbook/protocol/event/BetSettledRecordTest.java
new file mode 100644
index 0000000..7ca5b68
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/event/BetSettledRecordTest.java
@@ -0,0 +1,36 @@
+package com.sportsbook.protocol.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.time.Instant;
+import java.util.Map;
+import org.junit.jupiter.api.Test;
+
+class BetSettledRecordTest {
+
+  @Test
+  void settledOutcomeRoundTrips() throws Exception {
+    BetSettled expected =
+        BetSettled.newBuilder()
+            .setBetId("bet-1")
+            .setUserId("user-1")
+            .setEventId("event-1")
+            .setResult(SettlementResultAvro.WON)
+            .setStake(Money.newBuilder().setAmount(10_000).setCurrency("KRW").build())
+            .setPayout(Money.newBuilder().setAmount(18_500).setCurrency("KRW").build())
+            .setSettledAt(Instant.parse("2026-08-21T00:00:00Z"))
+            .setResultDetail(Map.of("selection-1", "WON"))
+            .build();
+    AvroRecordTestSupport.assertFields(
+        BetSettled.getClassSchema(),
+        "betId",
+        "userId",
+        "eventId",
+        "result",
+        "stake",
+        "payout",
+        "settledAt",
+        "resultDetail");
+    assertThat(AvroRecordTestSupport.roundTrip(expected, BetSettled.class)).isEqualTo(expected);
+  }
+}


## `feat(events): define voided bet outcomes`

diff --git a/src/main/avro/com/sportsbook/protocol/event/BetVoided.avsc b/src/main/avro/com/sportsbook/protocol/event/BetVoided.avsc
new file mode 100644
index 0000000..2d12d6c
--- /dev/null
+++ b/src/main/avro/com/sportsbook/protocol/event/BetVoided.avsc
@@ -0,0 +1,22 @@
+{
+  "namespace": "com.sportsbook.protocol.event",
+  "type": "record",
+  "name": "BetVoided",
+  "doc": "Bet voided and stake fully refunded (ADR-0012: cancelled/postponed events void). Published by settlement-service; betting-service consumes to flip bet.status to VOIDED, wallet-service credits the refund.",
+  "fields": [
+    {"name": "betId", "type": "string", "doc": "Slip UUID — primary key in betting-service."},
+    {"name": "userId", "type": "string"},
+    {"name": "eventId", "type": "string", "doc": "Event whose cancellation/postponement triggered the void."},
+    {
+      "name": "reason",
+      "type": {
+        "type": "enum",
+        "name": "VoidReason",
+        "symbols": ["EVENT_CANCELLED", "EVENT_POSTPONED", "MARKET_VOID", "ADMIN_VOID"]
+      },
+      "doc": "Why the bet was voided. EVENT_CANCELLED/EVENT_POSTPONED come from event lifecycle (ADR-0012); MARKET_VOID voids a single market; ADMIN_VOID is a manual admin-api action."
+    },
+    {"name": "refund", "type": "Money", "doc": "Stake refunded in full; same Money record reused across the event schema family."},
+    {"name": "voidedAt", "type": {"type": "long", "logicalType": "timestamp-millis"}}
+  ]
+}


## `test(events): verify voided bet outcomes`

diff --git a/src/test/java/com/sportsbook/protocol/event/BetVoidedRecordTest.java b/src/test/java/com/sportsbook/protocol/event/BetVoidedRecordTest.java
new file mode 100644
index 0000000..f428168
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/event/BetVoidedRecordTest.java
@@ -0,0 +1,25 @@
+package com.sportsbook.protocol.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.time.Instant;
+import org.junit.jupiter.api.Test;
+
+class BetVoidedRecordTest {
+
+  @Test
+  void voidedOutcomeRoundTrips() throws Exception {
+    BetVoided expected =
+        BetVoided.newBuilder()
+            .setBetId("bet-1")
+            .setUserId("user-1")
+            .setEventId("event-1")
+            .setReason(VoidReason.EVENT_CANCELLED)
+            .setRefund(Money.newBuilder().setAmount(10_000).setCurrency("KRW").build())
+            .setVoidedAt(Instant.parse("2026-08-21T00:00:00Z"))
+            .build();
+    AvroRecordTestSupport.assertFields(
+        BetVoided.getClassSchema(), "betId", "userId", "eventId", "reason", "refund", "voidedAt");
+    assertThat(AvroRecordTestSupport.roundTrip(expected, BetVoided.class)).isEqualTo(expected);
+  }
+}
