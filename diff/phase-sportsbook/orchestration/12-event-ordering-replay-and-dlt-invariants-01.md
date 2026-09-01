# 이벤트 순서·재처리·DLT 불변식

## `feat(fixtures): constrain Kafka records`

diff --git a/fixtures/avro-publisher/src/main/java/com/sportsbook/orchestration/fixture/FixtureRecord.java b/fixtures/avro-publisher/src/main/java/com/sportsbook/orchestration/fixture/FixtureRecord.java
new file mode 100644
index 0000000..4a72f80
--- /dev/null
+++ b/fixtures/avro-publisher/src/main/java/com/sportsbook/orchestration/fixture/FixtureRecord.java
@@ -0,0 +1,46 @@
+package com.sportsbook.orchestration.fixture;
+
+import java.io.IOException;
+import java.nio.file.Path;
+import java.util.UUID;
+import org.apache.kafka.clients.producer.ProducerRecord;
+
+public record FixtureRecord(
+    String topic, String key, Integer partition, byte[] payload, String fingerprint) {
+  private static final byte[] POISON_PAYLOAD = {(byte) 0x80};
+
+  public FixtureRecord {
+    validatePartition(partition);
+    payload = payload.clone();
+  }
+
+  public static FixtureRecord fromJson(FixtureType type, Path jsonPath, Integer partition)
+      throws IOException {
+    FixtureEncoder.EncodedFixture encoded = FixtureEncoder.encode(type, jsonPath);
+    return new FixtureRecord(
+        type.topic(), encoded.key(), partition, encoded.payload(), type.fingerprint());
+  }
+
+  public static FixtureRecord poisonMatchResult(String eventId) {
+    String canonicalEventId = UUID.fromString(eventId).toString();
+    if (!canonicalEventId.equals(eventId)) {
+      throw new IllegalArgumentException("eventId must be a canonical UUID");
+    }
+    return new FixtureRecord("match.result", eventId, 2, POISON_PAYLOAD, "malformed");
+  }
+
+  public ProducerRecord<String, byte[]> producerRecord() {
+    return new ProducerRecord<>(topic, partition, key, payload());
+  }
+
+  @Override
+  public byte[] payload() {
+    return payload.clone();
+  }
+
+  private static void validatePartition(Integer partition) {
+    if (partition != null && (partition < 0 || partition > 2)) {
+      throw new IllegalArgumentException("partition must be between 0 and 2");
+    }
+  }
+}


## `test(fixtures): verify schema keys and poison routing`

diff --git a/fixtures/avro-publisher/src/test/java/com/sportsbook/orchestration/fixture/FixtureRecordTest.java b/fixtures/avro-publisher/src/test/java/com/sportsbook/orchestration/fixture/FixtureRecordTest.java
new file mode 100644
index 0000000..1779d20
--- /dev/null
+++ b/fixtures/avro-publisher/src/test/java/com/sportsbook/orchestration/fixture/FixtureRecordTest.java
@@ -0,0 +1,71 @@
+package com.sportsbook.orchestration.fixture;
+
+import static org.junit.jupiter.api.Assertions.assertArrayEquals;
+import static org.junit.jupiter.api.Assertions.assertEquals;
+import static org.junit.jupiter.api.Assertions.assertNull;
+import static org.junit.jupiter.api.Assertions.assertThrows;
+
+import java.nio.file.Files;
+import java.nio.file.Path;
+import org.apache.kafka.clients.producer.ProducerRecord;
+import org.junit.jupiter.api.Test;
+import org.junit.jupiter.api.io.TempDir;
+
+class FixtureRecordTest {
+  private static final String EVENT_ID = "00000000-0000-0000-0000-0000000000ab";
+
+  @TempDir Path temporaryDirectory;
+
+  @Test
+  void derivesTopicKeyAndFingerprintFromTheLockedSchema() throws Exception {
+    Path json =
+        Files.writeString(
+            temporaryDirectory.resolve("event.json"),
+            """
+            {
+              "eventId": "%s",
+              "status": "FINISHED",
+              "occurredAt": 1700000000000,
+              "scheduledStartAt": 1699990000000
+            }
+            """
+                .formatted(EVENT_ID));
+
+    FixtureRecord fixture = FixtureRecord.fromJson(FixtureType.EVENT_LIFECYCLE, json, 1);
+    ProducerRecord<String, byte[]> record = fixture.producerRecord();
+
+    assertEquals("event.lifecycle", record.topic());
+    assertEquals(EVENT_ID, record.key());
+    assertEquals(1, record.partition());
+    assertEquals("e47d6dbd952bc721", fixture.fingerprint());
+    assertArrayEquals(fixture.payload(), record.value());
+  }
+
+  @Test
+  void permitsBrokerPartitioningButRejectsOutOfRangePartitions() throws Exception {
+    FixtureRecord fixture =
+        new FixtureRecord("match.result", EVENT_ID, null, new byte[] {1}, "fingerprint");
+
+    assertNull(fixture.producerRecord().partition());
+    assertThrows(
+        IllegalArgumentException.class,
+        () -> new FixtureRecord("match.result", EVENT_ID, -1, new byte[] {1}, "fingerprint"));
+    assertThrows(
+        IllegalArgumentException.class,
+        () -> new FixtureRecord("match.result", EVENT_ID, 3, new byte[] {1}, "fingerprint"));
+  }
+
+  @Test
+  void fixesPoisonRecordToMatchResultPartitionTwo() {
+    FixtureRecord poison = FixtureRecord.poisonMatchResult(EVENT_ID);
+
+    assertEquals("match.result", poison.topic());
+    assertEquals(EVENT_ID, poison.key());
+    assertEquals(2, poison.partition());
+    assertArrayEquals(new byte[] {(byte) 0x80}, poison.payload());
+    assertEquals("malformed", poison.fingerprint());
+    assertThrows(
+        IllegalArgumentException.class,
+        () -> FixtureRecord.poisonMatchResult(EVENT_ID.toUpperCase()));
+  }
+}


## `test(e2e): read void refund proof`

diff --git a/e2e/placement_oracles.py b/e2e/placement_oracles.py
index 3cc3f14..4d445b0 100644
--- a/e2e/placement_oracles.py
+++ b/e2e/placement_oracles.py
@@ -64,3 +64,16 @@ class PlacementOracles:
               AND schema_name = '{schema_name}'
             """,
         )
+
+    def wallet_void_refund(self, bet_id: str) -> dict[str, str]:
+        return self.database.one(
+            "wallet",
+            f"""
+            SELECT count(*)::text AS operation_count,
+                   COALESCE(min(caller_id), '') AS caller,
+                   COALESCE(min(operation_kind), '') AS kind,
+                   COALESCE(min(status), '') AS status
+            FROM wallet_operation
+            WHERE idempotency_key = 'void:refund:' || {uuid_literal(bet_id)}::text
+            """,
+        )


## `test(e2e): verify lifecycle-first refund`

diff --git a/e2e/scenario_04_lifecycle_refund.py b/e2e/scenario_04_lifecycle_refund.py
new file mode 100644
index 0000000..85ed8e1
--- /dev/null
+++ b/e2e/scenario_04_lifecycle_refund.py
@@ -0,0 +1,90 @@
+from __future__ import annotations
+
+import time
+
+from e2e.assertions import require_fields, wait_fields
+from e2e.model import ScenarioIds
+from e2e.runtime import E2eRuntime
+from scripts.cold_gate.polling import poll_until
+
+
+NAME = "lifecycle-before-placement-refund"
+
+
+def run(runtime: E2eRuntime) -> None:
+    fixture = ScenarioIds.create(4)
+    runtime.seed(fixture)
+    occurred_at = int(time.time() * 1000) - 5_000
+    runtime.fixtures.publish("EventLifecycle", fixture.cancelled(occurred_at))
+    poll_until(
+        "cancelled event tombstone",
+        lambda: runtime.base.tombstone(fixture.event),
+        lambda status: status == "CANCELLED",
+        timeout=60,
+        interval=0.25,
+    )
+
+    token = runtime.user_token(fixture)
+    placement = runtime.bets.place(fixture, token)
+    require_fields(
+        placement.__dict__,
+        {"http_status": 201, "status": "ACCEPTED"},
+        "lifecycle-first placement",
+    )
+    wait_fields(
+        "lifecycle-first Settlement void",
+        lambda: runtime.base.settlement(placement.bet_id),
+        {
+            "status": "VOIDED",
+            "result": "VOID",
+            "payout": "10000",
+            "currency": "KRW",
+            "revision_number": "0",
+        },
+        terminal={"status": frozenset({"SETTLED"})},
+    )
+    wait_fields(
+        "lifecycle-first Betting void",
+        lambda: runtime.base.betting(placement.bet_id),
+        {
+            "status": "VOIDED",
+            "void_reason": "EVENT_CANCELLED",
+            "revision_number": "0",
+        },
+        terminal={"status": frozenset({"REJECTED", "SETTLED"})},
+    )
+    wait_fields(
+        "lifecycle-first Wallet refund",
+        lambda: runtime.base.wallet(fixture.user),
+        {"available": "100000", "locked": "0", "debt": "0", "frozen": "0"},
+    )
+    require_fields(
+        runtime.placements.wallet_void_refund(placement.bet_id),
+        {
+            "operation_count": "1",
+            "caller": "SETTLEMENT",
+            "kind": "BET_REFUND",
+            "status": "SUCCEEDED",
+        },
+        "void refund operation",
+    )
+    wait_fields(
+        "BetVoided publication",
+        lambda: runtime.placements.settlement_outbox(fixture.event, "BetVoided"),
+        {"event_count": "1", "topic": "bet.voided.v1", "published": "1"},
+    )
+    queried = runtime.bets.get(placement.bet_id, token)
+    resolution = queried.get("resolution")
+    if not isinstance(resolution, dict):
+        raise RuntimeError("public voided bet has no resolution")
+    require_fields(
+        resolution,
+        {
+            "voidReason": "EVENT_CANCELLED",
+            "resolutionEventId": fixture.event,
+            "resolutionRevisionNumber": 0,
+        },
+        "public void resolution",
+    )
+    if "settlementResult" in resolution or "settledPayout" in resolution:
+        raise RuntimeError("public void resolution exposed settlement fields")


## `test(e2e): share terminal settlement barriers`

diff --git a/e2e/terminal.py b/e2e/terminal.py
new file mode 100644
index 0000000..581a92d
--- /dev/null
+++ b/e2e/terminal.py
@@ -0,0 +1,50 @@
+from __future__ import annotations
+
+from e2e.assertions import wait_fields
+from e2e.model import ScenarioIds
+from e2e.runtime import E2eRuntime
+
+
+def wait_base_settlement(
+    runtime: E2eRuntime,
+    fixture: ScenarioIds,
+    bet_id: str,
+    outcome: str,
+    payout: int,
+    available: int,
+) -> None:
+    wait_fields(
+        f"Settlement {outcome} base resolution",
+        lambda: runtime.base.settlement(bet_id),
+        {
+            "status": "SETTLED",
+            "result": outcome,
+            "payout": str(payout),
+            "currency": "KRW",
+            "revision_number": "0",
+        },
+        terminal={"status": frozenset({"VOIDED"})},
+    )
+    wait_fields(
+        f"Betting {outcome} base resolution",
+        lambda: runtime.base.betting(bet_id),
+        {
+            "status": "SETTLED",
+            "placement_phase": "RISK_COMMITTED",
+            "result": outcome,
+            "payout": str(payout),
+            "currency": "KRW",
+            "revision_number": "0",
+        },
+        terminal={"status": frozenset({"REJECTED", "VOIDED"})},
+    )
+    wait_fields(
+        f"Wallet {outcome} base resolution",
+        lambda: runtime.base.wallet(fixture.user),
+        {
+            "available": str(available),
+            "locked": "0",
+            "debt": "0",
+            "frozen": "0",
+        },
+    )


## `test(e2e): define full revision snapshots`

diff --git a/e2e/revision_fixture.py b/e2e/revision_fixture.py
new file mode 100644
index 0000000..7e19c4e
--- /dev/null
+++ b/e2e/revision_fixture.py
@@ -0,0 +1,46 @@
+from __future__ import annotations
+
+import uuid
+
+from e2e.model import OUTCOMES, ScenarioIds, money
+
+
+def revision_payload(
+    fixture: ScenarioIds,
+    bet_id: str,
+    revision_id: str,
+    revision_number: int,
+    previous_result: str,
+    new_result: str,
+    previous_payout: int,
+    new_payout: int,
+    source_result_settled_at: int,
+    revised_at: int,
+) -> dict[str, object]:
+    for value in (bet_id, revision_id):
+        parsed = uuid.UUID(value)
+        if str(parsed) != value:
+            raise ValueError("revision fixture UUID is not canonical")
+    if (
+        revision_number < 1
+        or previous_result not in OUTCOMES
+        or new_result not in OUTCOMES
+        or previous_payout < 0
+        or new_payout < 0
+        or source_result_settled_at <= 0
+        or revised_at < source_result_settled_at
+    ):
+        raise ValueError("revision fixture is invalid")
+    return {
+        "revisionId": revision_id,
+        "revisionNumber": revision_number,
+        "betId": bet_id,
+        "userId": fixture.user,
+        "eventId": fixture.event,
+        "previousResult": previous_result,
+        "newResult": new_result,
+        "previousPayout": money(previous_payout),
+        "newPayout": money(new_payout),
+        "sourceResultSettledAt": source_result_settled_at,
+        "revisedAt": revised_at,
+    }


## `test(e2e): stage revision ordering conflicts`

diff --git a/e2e/revision_ordering_stimulus.py b/e2e/revision_ordering_stimulus.py
new file mode 100644
index 0000000..4438036
--- /dev/null
+++ b/e2e/revision_ordering_stimulus.py
@@ -0,0 +1,98 @@
+from __future__ import annotations
+
+import dataclasses
+import decimal
+import time
+
+from e2e.assertions import require_fields, wait_fields
+from e2e.metrics import metric_value
+from e2e.model import ScenarioIds
+from e2e.revision_fixture import revision_payload
+from e2e.runtime import E2eRuntime
+from scripts.cold_gate.fixture_receipt import FixtureReceipt
+from scripts.cold_gate.polling import poll_until
+
+
+GAP_METRIC = "betting_resolution_revision_gaps_total"
+
+
+@dataclasses.dataclass(frozen=True)
+class OrderingState:
+    fixture: ScenarioIds
+    bet_id: str
+    revision_id: str
+    payload_sha256: str
+    source_time: int
+    gap_expected: decimal.Decimal
+
+
+def stage_revision_ordering(runtime: E2eRuntime) -> OrderingState:
+    fixture = ScenarioIds.create(10)
+    runtime.seed(fixture)
+    placement = runtime.bets.place(fixture, runtime.user_token(fixture))
+    wait_fields(
+        "ordering placement projection",
+        lambda: runtime.base.settlement(placement.bet_id),
+        {"status": "PENDING"},
+        terminal={"status": frozenset({"SETTLED", "VOIDED"})},
+    )
+    gap_expected = metric_value(runtime.betting_http, GAP_METRIC) + 1
+    source_time = int(time.time() * 1000) - 10_000
+    revision_two = revision_payload(
+        fixture, placement.bet_id, "55000000-0000-7000-8000-000000000010",
+        2, "LOST", "WON", 0, 20_000, source_time, source_time + 1_000,
+    )
+    first = runtime.fixtures.publish("BetResolutionRevised", revision_two)
+    wait_consumed(runtime, first)
+    wait_fields(
+        "revision-before-base projection",
+        lambda: runtime.base.betting(placement.bet_id),
+        {
+            "status": "SETTLED", "result": "WON", "payout": "20000",
+            "revision_number": "2", "revision_id": revision_two["revisionId"],
+            "payload_sha256": first.sha256,
+        },
+    )
+    wait_gap(runtime, gap_expected)
+
+    duplicate = runtime.fixtures.publish("BetResolutionRevised", revision_two)
+    if duplicate.sha256 != first.sha256:
+        raise RuntimeError("duplicate revision bytes drifted")
+    wait_consumed(runtime, duplicate)
+    revision_one = revision_payload(
+        fixture, placement.bet_id, "55000000-0000-7000-8000-000000000011",
+        1, "LOST", "LOST", 0, 0, source_time, source_time + 2_000,
+    )
+    lower = runtime.fixtures.publish("BetResolutionRevised", revision_one)
+    wait_consumed(runtime, lower)
+    require_fields(
+        runtime.base.betting(placement.bet_id) or {},
+        {"revision_number": "2", "revision_id": revision_two["revisionId"],
+         "payload_sha256": first.sha256},
+        "duplicate and lower revision projection",
+    )
+    wait_gap(runtime, gap_expected)
+    return OrderingState(
+        fixture, placement.bet_id, str(revision_two["revisionId"]), first.sha256,
+        source_time, gap_expected,
+    )
+
+
+def wait_consumed(runtime: E2eRuntime, receipt: FixtureReceipt) -> None:
+    poll_until(
+        "Betting revision consumption",
+        lambda: runtime.kafka.committed_offset("betting-resolution", receipt.topic, receipt.partition),
+        lambda offset: offset > receipt.offset,
+        timeout=60,
+        interval=0.5,
+    )
+
+
+def wait_gap(runtime: E2eRuntime, expected: decimal.Decimal) -> None:
+    poll_until(
+        "revision gap counter",
+        lambda: metric_value(runtime.betting_http, GAP_METRIC),
+        lambda value: value == expected,
+        timeout=30,
+        interval=0.25,
+    )


## `test(e2e): verify revision ordering projection`

diff --git a/e2e/scenario_10_revision_ordering.py b/e2e/scenario_10_revision_ordering.py
new file mode 100644
index 0000000..114aba2
--- /dev/null
+++ b/e2e/scenario_10_revision_ordering.py
@@ -0,0 +1,52 @@
+from __future__ import annotations
+
+from e2e.assertions import require_fields, wait_fields
+from e2e.revision_ordering_stimulus import stage_revision_ordering, wait_gap
+from e2e.runtime import E2eRuntime
+from scripts.cold_gate.polling import poll_until
+
+
+NAME = "revision-ordering-projection"
+
+
+def run(runtime: E2eRuntime) -> None:
+    state = stage_revision_ordering(runtime)
+    runtime.fixtures.publish(
+        "MatchResult", state.fixture.match_result("WON", state.source_time)
+    )
+    wait_fields(
+        "late base Settlement resolution",
+        lambda: runtime.base.settlement(state.bet_id),
+        {
+            "status": "SETTLED",
+            "result": "WON",
+            "payout": "20000",
+            "revision_number": "0",
+        },
+        terminal={"status": frozenset({"VOIDED"})},
+    )
+    wait_fields(
+        "late base Wallet settlement",
+        lambda: runtime.base.wallet(state.fixture.user),
+        {"available": "110000", "locked": "0", "debt": "0", "frozen": "0"},
+    )
+    poll_until(
+        "late base Betting consumption",
+        lambda: runtime.kafka.topic_lag("betting-resolution", "bet.settled.v1"),
+        lambda lag: lag == 0,
+        timeout=60,
+        interval=0.5,
+    )
+    require_fields(
+        runtime.base.betting(state.bet_id) or {},
+        {
+            "status": "SETTLED",
+            "result": "WON",
+            "payout": "20000",
+            "revision_number": "2",
+            "revision_id": state.revision_id,
+            "payload_sha256": state.payload_sha256,
+        },
+        "late base ordering projection",
+    )
+    wait_gap(runtime, state.gap_expected)


## `test(e2e): await the late base outbox`

diff --git a/e2e/scenario_10_revision_ordering.py b/e2e/scenario_10_revision_ordering.py
index 114aba2..a773d3c 100644
--- a/e2e/scenario_10_revision_ordering.py
+++ b/e2e/scenario_10_revision_ordering.py
@@ -30,6 +30,11 @@ def run(runtime: E2eRuntime) -> None:
         lambda: runtime.base.wallet(state.fixture.user),
         {"available": "110000", "locked": "0", "debt": "0", "frozen": "0"},
     )
+    wait_fields(
+        "late base Settlement outbox",
+        lambda: runtime.placements.settlement_outbox(state.fixture.event, "BetSettled"),
+        {"event_count": "1", "topic": "bet.settled.v1", "published": "1"},
+    )
     poll_until(
         "late base Betting consumption",
         lambda: runtime.kafka.topic_lag("betting-resolution", "bet.settled.v1"),


## `test(e2e): capture stable replay projections`

diff --git a/e2e/replay_oracle.py b/e2e/replay_oracle.py
new file mode 100644
index 0000000..31825f6
--- /dev/null
+++ b/e2e/replay_oracle.py
@@ -0,0 +1,57 @@
+from __future__ import annotations
+
+import json
+
+from e2e.base_oracles import BaseOracles
+from e2e.model import ScenarioIds
+from scripts.cold_gate.database import PostgresClient, uuid_literal
+
+
+class ReplayOracle:
+    def __init__(self, database: PostgresClient, base: BaseOracles) -> None:
+        self.database = database
+        self.base = base
+
+    def snapshot(self, fixture: ScenarioIds, bet_id: str) -> str:
+        bet = uuid_literal(bet_id)
+        event = uuid_literal(fixture.event)
+        user = uuid_literal(fixture.user)
+        settlement_counts = self.database.one(
+            "settlement",
+            f"""
+            SELECT
+              (SELECT count(*) FROM result_candidate WHERE event_id = {event})::text AS candidates,
+              (SELECT count(*) FROM settlement_revision WHERE bet_id = {bet})::text AS revisions,
+              (SELECT count(*) FROM outbox_event
+               WHERE partition_key IN ({event}::text, {bet}::text))::text AS outbox_events,
+              (SELECT count(*) FROM settlement_attempt WHERE bet_id = {bet})::text AS attempts
+            """,
+        )
+        wallet_counts = self.database.one(
+            "wallet",
+            f"""
+            SELECT
+              (SELECT count(*) FROM ledger_entry WHERE account_id = {user})::text AS ledger_entries,
+              (SELECT count(*) FROM wallet_operation WHERE user_id = {user})::text AS operations,
+              (SELECT count(*) FROM wallet_adjustment WHERE user_id = {user})::text AS adjustments,
+              (SELECT count(*) FROM outbox_event o JOIN wallet_operation w
+               ON w.idempotency_key = o.operation_key WHERE w.user_id = {user})::text AS outbox_events
+            """,
+        )
+        betting_counts = self.database.one(
+            "betting",
+            f"""
+            SELECT
+              (SELECT count(*) FROM outbox_event)::text AS outbox_events,
+              (SELECT count(*) FROM wallet_event_receipt WHERE bet_id = {bet})::text AS wallet_receipts
+            """,
+        )
+        snapshot = {
+            "betting": self.base.betting(bet_id),
+            "settlement": self.base.settlement(bet_id),
+            "wallet": self.base.wallet(fixture.user),
+            "settlementCounts": settlement_counts,
+            "walletCounts": wallet_counts,
+            "bettingCounts": betting_counts,
+        }
+        return json.dumps(snapshot, sort_keys=True, separators=(",", ":"))


