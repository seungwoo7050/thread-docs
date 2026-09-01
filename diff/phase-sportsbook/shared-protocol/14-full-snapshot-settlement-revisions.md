# 정산 정정 전체 스냅샷

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


## `build(avro): order shared named schemas`

diff --git a/pom.xml b/pom.xml
index 9bbfbd7..14f8026 100644
--- a/pom.xml
+++ b/pom.xml
@@ -89,6 +89,10 @@
                             <goal>schema</goal>
                         </goals>
                         <configuration>
+                            <imports>
+                                <import>${project.basedir}/src/main/avro/com/sportsbook/protocol/event/Money.avsc</import>
+                                <import>${project.basedir}/src/main/avro/com/sportsbook/protocol/event/BetSettled.avsc</import>
+                            </imports>
                             <sourceDirectory>${project.basedir}/src/main/avro</sourceDirectory>
                             <outputDirectory>${project.build.directory}/generated-sources/avro</outputDirectory>
                             <stringType>String</stringType>


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


## `test(events): verify revision schema contract`

diff --git a/src/test/java/com/sportsbook/protocol/event/BetResolutionRevisedSchemaTest.java b/src/test/java/com/sportsbook/protocol/event/BetResolutionRevisedSchemaTest.java
new file mode 100644
index 0000000..d701afa
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/event/BetResolutionRevisedSchemaTest.java
@@ -0,0 +1,50 @@
+package com.sportsbook.protocol.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import org.apache.avro.Schema;
+import org.junit.jupiter.api.Test;
+
+class BetResolutionRevisedSchemaTest {
+
+  @Test
+  void revisionUsesExactRequiredFields() {
+    Schema schema = BetResolutionRevised.getClassSchema();
+    AvroRecordTestSupport.assertFields(
+        schema,
+        "revisionId",
+        "revisionNumber",
+        "betId",
+        "userId",
+        "eventId",
+        "previousResult",
+        "newResult",
+        "previousPayout",
+        "newPayout",
+        "sourceResultSettledAt",
+        "revisedAt");
+    assertThat(schema.getFields()).allMatch(field -> !field.hasDefaultValue());
+  }
+
+  @Test
+  void revisionReusesSettlementAndMoneyTypes() {
+    Schema schema = BetResolutionRevised.getClassSchema();
+    assertThat(schema.getField("previousResult").schema().getFullName())
+        .isEqualTo("com.sportsbook.protocol.event.SettlementResultAvro");
+    assertThat(schema.getField("newResult").schema())
+        .isEqualTo(schema.getField("previousResult").schema());
+    assertThat(schema.getField("previousPayout").schema().getFullName())
+        .isEqualTo("com.sportsbook.protocol.event.Money");
+    assertThat(schema.getField("newPayout").schema())
+        .isEqualTo(schema.getField("previousPayout").schema());
+  }
+
+  @Test
+  void revisionTimestampsUseMillisLogicalType() {
+    Schema schema = BetResolutionRevised.getClassSchema();
+    assertThat(schema.getField("sourceResultSettledAt").schema().getLogicalType().getName())
+        .isEqualTo("timestamp-millis");
+    assertThat(schema.getField("revisedAt").schema().getLogicalType().getName())
+        .isEqualTo("timestamp-millis");
+  }
+}


## `test(events): verify payout increase revisions`

diff --git a/src/test/java/com/sportsbook/protocol/event/BetResolutionPayoutIncreaseTest.java b/src/test/java/com/sportsbook/protocol/event/BetResolutionPayoutIncreaseTest.java
new file mode 100644
index 0000000..d0bc807
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/event/BetResolutionPayoutIncreaseTest.java
@@ -0,0 +1,37 @@
+package com.sportsbook.protocol.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.time.Instant;
+import org.junit.jupiter.api.Test;
+
+class BetResolutionPayoutIncreaseTest {
+
+  @Test
+  void lostBetCanBeRevisedToWonSnapshot() throws Exception {
+    BetResolutionRevised expected =
+        revision(SettlementResultAvro.LOST, 0, SettlementResultAvro.WON, 18_500);
+    assertThat(AvroRecordTestSupport.roundTrip(expected, BetResolutionRevised.class))
+        .isEqualTo(expected);
+  }
+
+  private BetResolutionRevised revision(
+      SettlementResultAvro previousResult,
+      long previousPayout,
+      SettlementResultAvro newResult,
+      long newPayout) {
+    return BetResolutionRevised.newBuilder()
+        .setRevisionId("0198f77e-dc00-7000-8000-000000000001")
+        .setRevisionNumber(1)
+        .setBetId("0198f77e-dc00-7000-8000-000000000002")
+        .setUserId("0198f77e-dc00-7000-8000-000000000003")
+        .setEventId("0198f77e-dc00-7000-8000-000000000004")
+        .setPreviousResult(previousResult)
+        .setNewResult(newResult)
+        .setPreviousPayout(Money.newBuilder().setAmount(previousPayout).setCurrency("KRW").build())
+        .setNewPayout(Money.newBuilder().setAmount(newPayout).setCurrency("KRW").build())
+        .setSourceResultSettledAt(Instant.parse("2026-08-21T00:00:00Z"))
+        .setRevisedAt(Instant.parse("2026-08-21T00:00:01Z"))
+        .build();
+  }
+}


## `test(events): verify payout decrease revisions`

diff --git a/src/test/java/com/sportsbook/protocol/event/BetResolutionPayoutDecreaseTest.java b/src/test/java/com/sportsbook/protocol/event/BetResolutionPayoutDecreaseTest.java
new file mode 100644
index 0000000..62f4ca3
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/event/BetResolutionPayoutDecreaseTest.java
@@ -0,0 +1,29 @@
+package com.sportsbook.protocol.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.time.Instant;
+import org.junit.jupiter.api.Test;
+
+class BetResolutionPayoutDecreaseTest {
+
+  @Test
+  void wonBetCanBeRevisedToLostSnapshot() throws Exception {
+    BetResolutionRevised expected =
+        BetResolutionRevised.newBuilder()
+            .setRevisionId("0198f77e-dc00-7000-8000-000000000001")
+            .setRevisionNumber(1)
+            .setBetId("0198f77e-dc00-7000-8000-000000000002")
+            .setUserId("0198f77e-dc00-7000-8000-000000000003")
+            .setEventId("0198f77e-dc00-7000-8000-000000000004")
+            .setPreviousResult(SettlementResultAvro.WON)
+            .setNewResult(SettlementResultAvro.LOST)
+            .setPreviousPayout(Money.newBuilder().setAmount(18_500).setCurrency("KRW").build())
+            .setNewPayout(Money.newBuilder().setAmount(0).setCurrency("KRW").build())
+            .setSourceResultSettledAt(Instant.parse("2026-08-21T00:00:00Z"))
+            .setRevisedAt(Instant.parse("2026-08-21T00:00:01Z"))
+            .build();
+    assertThat(AvroRecordTestSupport.roundTrip(expected, BetResolutionRevised.class))
+        .isEqualTo(expected);
+  }
+}
