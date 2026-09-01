# 지갑 원장 결과 이벤트

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


## `test(events): verify wallet debit confirmations`

diff --git a/src/test/java/com/sportsbook/protocol/event/WalletDebitedRecordTest.java b/src/test/java/com/sportsbook/protocol/event/WalletDebitedRecordTest.java
new file mode 100644
index 0000000..25021ce
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/event/WalletDebitedRecordTest.java
@@ -0,0 +1,29 @@
+package com.sportsbook.protocol.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.time.Instant;
+import org.junit.jupiter.api.Test;
+
+class WalletDebitedRecordTest {
+
+  @Test
+  void debitConfirmationRoundTrips() throws Exception {
+    WalletDebited expected =
+        WalletDebited.newBuilder()
+            .setUserId("user-1")
+            .setAmount(Money.newBuilder().setAmount(10_000).setCurrency("KRW").build())
+            .setIdempotencyKey("bet-1:debit")
+            .setLedgerTxId("ledger-1")
+            .setOccurredAt(Instant.parse("2026-08-21T00:00:00Z"))
+            .build();
+    AvroRecordTestSupport.assertFields(
+        WalletDebited.getClassSchema(),
+        "userId",
+        "amount",
+        "idempotencyKey",
+        "ledgerTxId",
+        "occurredAt");
+    assertThat(AvroRecordTestSupport.roundTrip(expected, WalletDebited.class)).isEqualTo(expected);
+  }
+}


## `feat(events): define wallet credit confirmations`

diff --git a/src/main/avro/com/sportsbook/protocol/event/WalletCredited.avsc b/src/main/avro/com/sportsbook/protocol/event/WalletCredited.avsc
new file mode 100644
index 0000000..ed63d74
--- /dev/null
+++ b/src/main/avro/com/sportsbook/protocol/event/WalletCredited.avsc
@@ -0,0 +1,22 @@
+{
+  "namespace": "com.sportsbook.protocol.event",
+  "type": "record",
+  "name": "WalletCredited",
+  "doc": "Wallet credited as a payout, void refund, or other refund (ADR-0006 Saga + Outbox).",
+  "fields": [
+    {"name": "userId", "type": "string"},
+    {"name": "amount", "type": "Money", "doc": "Amount added to the user's available balance."},
+    {"name": "idempotencyKey", "type": "string", "doc": "Caller-supplied idempotency key (ADR-0005)."},
+    {"name": "ledgerTxId", "type": "string", "doc": "Ledger journal entry UUID."},
+    {
+      "name": "reason",
+      "type": {
+        "type": "enum",
+        "name": "WalletCreditReason",
+        "symbols": ["PAYOUT", "VOID", "REFUND"]
+      },
+      "doc": "Why the credit was issued — drives downstream reporting and ledger account selection."
+    },
+    {"name": "occurredAt", "type": {"type": "long", "logicalType": "timestamp-millis"}}
+  ]
+}


## `test(events): verify wallet credit confirmations`

diff --git a/src/test/java/com/sportsbook/protocol/event/WalletCreditedRecordTest.java b/src/test/java/com/sportsbook/protocol/event/WalletCreditedRecordTest.java
new file mode 100644
index 0000000..030866c
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/event/WalletCreditedRecordTest.java
@@ -0,0 +1,31 @@
+package com.sportsbook.protocol.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.time.Instant;
+import org.junit.jupiter.api.Test;
+
+class WalletCreditedRecordTest {
+
+  @Test
+  void creditConfirmationRoundTrips() throws Exception {
+    WalletCredited expected =
+        WalletCredited.newBuilder()
+            .setUserId("user-1")
+            .setAmount(Money.newBuilder().setAmount(18_500).setCurrency("KRW").build())
+            .setIdempotencyKey("bet-1:payout")
+            .setLedgerTxId("ledger-2")
+            .setReason(WalletCreditReason.PAYOUT)
+            .setOccurredAt(Instant.parse("2026-08-21T00:00:00Z"))
+            .build();
+    AvroRecordTestSupport.assertFields(
+        WalletCredited.getClassSchema(),
+        "userId",
+        "amount",
+        "idempotencyKey",
+        "ledgerTxId",
+        "reason",
+        "occurredAt");
+    assertThat(AvroRecordTestSupport.roundTrip(expected, WalletCredited.class)).isEqualTo(expected);
+  }
+}


## `feat(events): define wallet debit failures`

diff --git a/src/main/avro/com/sportsbook/protocol/event/WalletDebitFailed.avsc b/src/main/avro/com/sportsbook/protocol/event/WalletDebitFailed.avsc
new file mode 100644
index 0000000..4026141
--- /dev/null
+++ b/src/main/avro/com/sportsbook/protocol/event/WalletDebitFailed.avsc
@@ -0,0 +1,21 @@
+{
+  "namespace": "com.sportsbook.protocol.event",
+  "type": "record",
+  "name": "WalletDebitFailed",
+  "doc": "A debit attempt was rejected; bet placement saga compensates by rejecting the slip (ADR-0006).",
+  "fields": [
+    {"name": "userId", "type": "string"},
+    {"name": "requestedAmount", "type": "Money"},
+    {"name": "idempotencyKey", "type": "string", "doc": "Caller-supplied idempotency key (ADR-0005)."},
+    {
+      "name": "reason",
+      "type": {
+        "type": "enum",
+        "name": "WalletDebitFailureReason",
+        "symbols": ["INSUFFICIENT_BALANCE", "ACCOUNT_SUSPENDED", "ACCOUNT_NOT_FOUND", "CURRENCY_MISMATCH"]
+      },
+      "doc": "Why the debit was rejected. Drives ErrorCode mapping in betting-service's saga compensator."
+    },
+    {"name": "occurredAt", "type": {"type": "long", "logicalType": "timestamp-millis"}}
+  ]
+}


## `test(events): verify wallet debit failures`

diff --git a/src/test/java/com/sportsbook/protocol/event/WalletDebitFailedRecordTest.java b/src/test/java/com/sportsbook/protocol/event/WalletDebitFailedRecordTest.java
new file mode 100644
index 0000000..e385b96
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/event/WalletDebitFailedRecordTest.java
@@ -0,0 +1,30 @@
+package com.sportsbook.protocol.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.time.Instant;
+import org.junit.jupiter.api.Test;
+
+class WalletDebitFailedRecordTest {
+
+  @Test
+  void debitFailureRoundTrips() throws Exception {
+    WalletDebitFailed expected =
+        WalletDebitFailed.newBuilder()
+            .setUserId("user-1")
+            .setRequestedAmount(Money.newBuilder().setAmount(10_000).setCurrency("KRW").build())
+            .setIdempotencyKey("bet-1:debit")
+            .setReason(WalletDebitFailureReason.INSUFFICIENT_BALANCE)
+            .setOccurredAt(Instant.parse("2026-08-21T00:00:00Z"))
+            .build();
+    AvroRecordTestSupport.assertFields(
+        WalletDebitFailed.getClassSchema(),
+        "userId",
+        "requestedAmount",
+        "idempotencyKey",
+        "reason",
+        "occurredAt");
+    assertThat(AvroRecordTestSupport.roundTrip(expected, WalletDebitFailed.class))
+        .isEqualTo(expected);
+  }
+}
