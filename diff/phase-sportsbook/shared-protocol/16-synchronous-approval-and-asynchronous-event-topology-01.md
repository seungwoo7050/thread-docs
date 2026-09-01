# 동기 승인과 비동기 이벤트 토폴로지

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


## `feat(events): define wallet debit confirmations`

diff --git a/src/main/avro/com/sportsbook/protocol/event/WalletDebited.avsc b/src/main/avro/com/sportsbook/protocol/event/WalletDebited.avsc
new file mode 100644
index 0000000..d1cb2d5
--- /dev/null
+++ b/src/main/avro/com/sportsbook/protocol/event/WalletDebited.avsc
@@ -0,0 +1,13 @@
+{
+  "namespace": "com.sportsbook.protocol.event",
+  "type": "record",
+  "name": "WalletDebited",
+  "doc": "Wallet successfully debited for a bet stake; published after the ledger entry commits (ADR-0006 Saga + Outbox).",
+  "fields": [
+    {"name": "userId", "type": "string", "doc": "User UUID."},
+    {"name": "amount", "type": "Money", "doc": "Amount removed from the user's available balance."},
+    {"name": "idempotencyKey", "type": "string", "doc": "Caller-supplied idempotency key (ADR-0005)."},
+    {"name": "ledgerTxId", "type": "string", "doc": "Ledger journal entry UUID — primary key in wallet-service's double-entry ledger."},
+    {"name": "occurredAt", "type": {"type": "long", "logicalType": "timestamp-millis"}, "doc": "Server-side commit time."}
+  ]
+}


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


## `feat(events): define resolution revision snapshots`

diff --git a/src/main/avro/com/sportsbook/protocol/event/BetResolutionRevised.avsc b/src/main/avro/com/sportsbook/protocol/event/BetResolutionRevised.avsc
new file mode 100644
index 0000000..22664d6
--- /dev/null
+++ b/src/main/avro/com/sportsbook/protocol/event/BetResolutionRevised.avsc
@@ -0,0 +1,19 @@
+{
+  "namespace": "com.sportsbook.protocol.event",
+  "type": "record",
+  "name": "BetResolutionRevised",
+  "doc": "A completed SETTLED bet resolution revised after a corrected MatchResult. Carries the previous and full new result snapshots so consumers can project revisions independently of cross-topic delivery order. VOIDED lifecycle corrections are outside this V1 contract.",
+  "fields": [
+    {"name": "revisionId", "type": "string", "doc": "Stable UUID for this durable per-bet revision; retries and redeliveries keep the same value."},
+    {"name": "revisionNumber", "type": "long", "doc": "Strictly increasing per bet from 1. The initial BetSettled snapshot is logical revision 0."},
+    {"name": "betId", "type": "string", "doc": "Slip UUID and Kafka partition key."},
+    {"name": "userId", "type": "string", "doc": "Owning user UUID for private gateway fan-out."},
+    {"name": "eventId", "type": "string", "doc": "Event whose corrected MatchResult caused this revision."},
+    {"name": "previousResult", "type": "SettlementResultAvro", "doc": "Immediately preceding committed SETTLED result."},
+    {"name": "newResult", "type": "SettlementResultAvro", "doc": "Full replacement SETTLED result after correction."},
+    {"name": "previousPayout", "type": "Money", "doc": "Immediately preceding committed payout snapshot."},
+    {"name": "newPayout", "type": "Money", "doc": "Full replacement payout snapshot after wallet adjustment."},
+    {"name": "sourceResultSettledAt", "type": {"type": "long", "logicalType": "timestamp-millis"}, "doc": "MatchResult.settledAt for the corrected source result."},
+    {"name": "revisedAt", "type": {"type": "long", "logicalType": "timestamp-millis"}, "doc": "When wallet adjustment proof and the durable revision/outbox state were completed."}
+  ]
+}


