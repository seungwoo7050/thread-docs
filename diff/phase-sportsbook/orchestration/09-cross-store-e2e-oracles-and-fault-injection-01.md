# 다중 저장소 E2E 오라클과 장애 주입

## `test(e2e): define isolated scenario identities`

diff --git a/e2e/model.py b/e2e/model.py
new file mode 100644
index 0000000..a612305
--- /dev/null
+++ b/e2e/model.py
@@ -0,0 +1,75 @@
+from __future__ import annotations
+
+import dataclasses
+
+
+OUTCOMES = frozenset({"WON", "LOST", "PUSH", "VOID"})
+
+
+@dataclasses.dataclass(frozen=True)
+class ScenarioIds:
+    number: int
+    user: str
+    event: str
+    market: str
+    selection: str
+
+    @classmethod
+    def create(cls, number: int) -> "ScenarioIds":
+        if number < 1 or number > 999_999_999_999:
+            raise ValueError("scenario number is out of range")
+        suffix = f"{number:012d}"
+        return cls(
+            number,
+            f"01000000-0000-7000-8000-{suffix}",
+            f"11000000-0000-7000-8000-{suffix}",
+            f"22000000-0000-7000-8000-{suffix}",
+            f"33000000-0000-7000-8000-{suffix}",
+        )
+
+    def account(self) -> dict[str, object]:
+        return {"userId": self.user, "currency": "KRW"}
+
+    def transfer(self, amount: int) -> dict[str, object]:
+        if amount <= 0:
+            raise ValueError("transfer amount must be positive")
+        return {"userId": self.user, "amount": money(amount)}
+
+    def placement(self) -> dict[str, object]:
+        return {
+            "slipType": {"type": "SINGLE"},
+            "selections": [
+                {
+                    "eventId": self.event,
+                    "marketId": self.market,
+                    "selectionId": self.selection,
+                    "odds": 2.0000,
+                }
+            ],
+            "stake": money(10_000),
+        }
+
+    def match_result(self, outcome: str, settled_at_ms: int) -> dict[str, object]:
+        if outcome not in OUTCOMES or settled_at_ms <= 0:
+            raise ValueError("match result is invalid")
+        return {
+            "eventId": self.event,
+            "score": "1-0",
+            "finalStatus": "COMPLETED",
+            "resultDetail": {self.selection: outcome},
+            "settledAt": settled_at_ms,
+        }
+
+    def cancelled(self, occurred_at_ms: int) -> dict[str, object]:
+        if occurred_at_ms <= 60_000:
+            raise ValueError("lifecycle timestamp is invalid")
+        return {
+            "eventId": self.event,
+            "status": "CANCELLED",
+            "occurredAt": occurred_at_ms,
+            "scheduledStartAt": occurred_at_ms - 60_000,
+        }
+
+
+def money(amount: int) -> dict[str, object]:
+    return {"amount": amount, "currency": "KRW"}


## `test(e2e): verify scenario fixture boundaries`

diff --git a/tests/test_e2e_model.py b/tests/test_e2e_model.py
new file mode 100644
index 0000000..6e983ef
--- /dev/null
+++ b/tests/test_e2e_model.py
@@ -0,0 +1,55 @@
+import unittest
+
+from e2e.model import ScenarioIds
+
+
+class E2eModelTest(unittest.TestCase):
+    def test_allocates_disjoint_canonical_uuidv7_boundaries(self) -> None:
+        first = ScenarioIds.create(1)
+        second = ScenarioIds.create(2)
+
+        self.assertEqual(first.user, "01000000-0000-7000-8000-000000000001")
+        self.assertEqual(first.event, "11000000-0000-7000-8000-000000000001")
+        self.assertEqual(first.market, "22000000-0000-7000-8000-000000000001")
+        self.assertEqual(first.selection, "33000000-0000-7000-8000-000000000001")
+        self.assertTrue({first.user, first.event, first.market, first.selection}.isdisjoint(
+            {second.user, second.event, second.market, second.selection}
+        ))
+
+    def test_builds_the_fixed_wallet_and_single_bet_contracts(self) -> None:
+        fixture = ScenarioIds.create(7)
+
+        self.assertEqual(fixture.account(), {"userId": fixture.user, "currency": "KRW"})
+        self.assertEqual(
+            fixture.transfer(100_000),
+            {"userId": fixture.user, "amount": {"amount": 100_000, "currency": "KRW"}},
+        )
+        placement = fixture.placement()
+        self.assertEqual(placement["slipType"], {"type": "SINGLE"})
+        self.assertEqual(placement["stake"], {"amount": 10_000, "currency": "KRW"})
+        self.assertEqual(placement["selections"][0]["odds"], 2.0)
+
+    def test_builds_exact_result_and_lifecycle_fixtures(self) -> None:
+        fixture = ScenarioIds.create(9)
+        result = fixture.match_result("WON", 1_700_000_000_000)
+        lifecycle = fixture.cancelled(1_700_000_000_000)
+
+        self.assertEqual(result["resultDetail"], {fixture.selection: "WON"})
+        self.assertEqual(result["finalStatus"], "COMPLETED")
+        self.assertEqual(lifecycle["status"], "CANCELLED")
+        self.assertEqual(lifecycle["scheduledStartAt"], 1_699_999_940_000)
+
+    def test_rejects_out_of_range_scenarios_and_invalid_events(self) -> None:
+        with self.assertRaises(ValueError):
+            ScenarioIds.create(0)
+        fixture = ScenarioIds.create(1)
+        with self.assertRaises(ValueError):
+            fixture.transfer(0)
+        with self.assertRaises(ValueError):
+            fixture.match_result("UNKNOWN", 1)
+        with self.assertRaises(ValueError):
+            fixture.cancelled(60_000)
+
+
+if __name__ == "__main__":
+    unittest.main()


## `test(e2e): seed Wallet through platform APIs`

diff --git a/e2e/wallet_api.py b/e2e/wallet_api.py
new file mode 100644
index 0000000..fe52dbd
--- /dev/null
+++ b/e2e/wallet_api.py
@@ -0,0 +1,62 @@
+from __future__ import annotations
+
+from e2e.model import ScenarioIds
+from scripts.cold_gate.container_http import ContainerHttpClient
+
+
+class WalletApi:
+    def __init__(self, client: ContainerHttpClient, platform_key: str) -> None:
+        if client.service != "wallet" or len(platform_key) < 32:
+            raise ValueError("Wallet E2E credentials are invalid")
+        self.client = client
+        self.headers = {
+            "X-Internal-Service": "platform",
+            "X-Internal-Api-Key": platform_key,
+        }
+
+    def open_and_fund(self, fixture: ScenarioIds) -> None:
+        opened = self.client.request(
+            "POST",
+            "/internal/v1/wallet/accounts",
+            headers=self.headers,
+            body=fixture.account(),
+        ).require_status(200)
+        payload = _object(opened.json())
+        if (
+            payload.get("userId") != fixture.user
+            or payload.get("currency") != "KRW"
+            or payload.get("available") != {"amount": 0, "currency": "KRW"}
+            or payload.get("locked") != {"amount": 0, "currency": "KRW"}
+            or payload.get("outboundFrozen") is not False
+        ):
+            raise RuntimeError("Wallet account fixture response drifted")
+        self.transfer(fixture, "deposit", 100_000, f"e2e-deposit-{fixture.number:02d}")
+
+    def transfer(
+        self, fixture: ScenarioIds, operation: str, amount: int, idempotency_key: str
+    ) -> dict[str, object]:
+        if operation not in {"deposit", "withdraw"} or not idempotency_key.startswith("e2e-"):
+            raise ValueError("Wallet fixture transfer is outside the E2E contract")
+        headers = {**self.headers, "Idempotency-Key": idempotency_key}
+        response = self.client.request(
+            "POST",
+            "/internal/v1/wallet/transactions/" + operation,
+            headers=headers,
+            body=fixture.transfer(amount),
+        ).require_status(200)
+        payload = _object(response.json())
+        expected_reason = "DEPOSIT" if operation == "deposit" else "WITHDRAW"
+        if (
+            payload.get("userId") != fixture.user
+            or payload.get("amount") != {"amount": amount, "currency": "KRW"}
+            or payload.get("reason") != expected_reason
+            or not payload.get("operationGroupId")
+        ):
+            raise RuntimeError("Wallet transfer receipt drifted")
+        return payload
+
+
+def _object(value: object) -> dict[str, object]:
+    if not isinstance(value, dict):
+        raise RuntimeError("HTTP payload is not an object")
+    return value


## `test(e2e): call authenticated bet boundaries`

diff --git a/e2e/bet_api.py b/e2e/bet_api.py
new file mode 100644
index 0000000..dc5ff8e
--- /dev/null
+++ b/e2e/bet_api.py
@@ -0,0 +1,74 @@
+from __future__ import annotations
+
+import dataclasses
+import uuid
+
+from e2e.model import ScenarioIds
+from scripts.cold_gate.http import HostHttpClient
+
+
+@dataclasses.dataclass(frozen=True)
+class PlacementReceipt:
+    bet_id: str
+    http_status: int
+    status: str
+
+
+class BetApi:
+    def __init__(self, client: HostHttpClient) -> None:
+        self.client = client
+
+    def place(self, fixture: ScenarioIds, token: str) -> PlacementReceipt:
+        response = self.client.request(
+            "POST",
+            "/api/v1/bets",
+            headers={
+                "Authorization": "Bearer " + token,
+                "Idempotency-Key": f"e2e-place-{fixture.number:02d}",
+            },
+            body=fixture.placement(),
+        ).require_status(201, 202)
+        payload = _object(response.json())
+        bet_id = str(payload.get("betId", ""))
+        try:
+            parsed = uuid.UUID(bet_id)
+        except ValueError as error:
+            raise RuntimeError("placement did not return a bet UUID") from error
+        if str(parsed) != bet_id:
+            raise RuntimeError("placement bet UUID is not canonical")
+        expected_status = "ACCEPTED" if response.status == 201 else "PENDING"
+        expected_selection = {
+            "eventId": fixture.event,
+            "marketId": fixture.market,
+            "selectionId": fixture.selection,
+            "oddsAtSubmission": "2.0000",
+        }
+        if (
+            response.header("Location") != "/api/v1/bets/" + bet_id
+            or payload.get("userId") != fixture.user
+            or payload.get("status") != expected_status
+            or payload.get("slipType") != {"type": "SINGLE"}
+            or payload.get("stake") != {"amount": 10_000, "currency": "KRW"}
+            or payload.get("maxPayout") != {"amount": 20_000, "currency": "KRW"}
+            or payload.get("selections") != [expected_selection]
+        ):
+            raise RuntimeError("placement response drifted")
+        return PlacementReceipt(bet_id, response.status, expected_status)
+
+    def get(self, bet_id: str, token: str) -> dict[str, object]:
+        uuid.UUID(bet_id)
+        response = self.client.request(
+            "GET",
+            "/api/v1/bets/" + bet_id,
+            headers={"Authorization": "Bearer " + token},
+        ).require_status(200)
+        payload = _object(response.json())
+        if payload.get("betId") != bet_id:
+            raise RuntimeError("bet query returned the wrong aggregate")
+        return payload
+
+
+def _object(value: object) -> dict[str, object]:
+    if not isinstance(value, dict):
+        raise RuntimeError("HTTP payload is not an object")
+    return value


## `test(e2e): read base service projections`

diff --git a/e2e/base_oracles.py b/e2e/base_oracles.py
new file mode 100644
index 0000000..be2ff81
--- /dev/null
+++ b/e2e/base_oracles.py
@@ -0,0 +1,73 @@
+from __future__ import annotations
+
+from scripts.cold_gate.database import PostgresClient, uuid_literal
+
+
+class BaseOracles:
+    def __init__(self, database: PostgresClient) -> None:
+        self.database = database
+
+    def betting(self, bet_id: str) -> dict[str, str] | None:
+        rows = self.database.query(
+            "betting",
+            f"""
+            SELECT status, placement_phase, risk_commit_observed::int AS risk_committed,
+                   (wallet_operation_id IS NOT NULL)::int AS wallet_confirmed,
+                   COALESCE(settlement_result, '') AS result,
+                   COALESCE(settled_payout_amount::text, '') AS payout,
+                   COALESCE(settled_payout_currency, '') AS currency,
+                   COALESCE(void_reason, '') AS void_reason,
+                   COALESCE(resolution_revision_number::text, '') AS revision_number,
+                   COALESCE(resolution_revision_id::text, '') AS revision_id,
+                   COALESCE(resolution_payload_sha256, '') AS payload_sha256
+            FROM bet WHERE bet_id = {uuid_literal(bet_id)}
+            """,
+        )
+        return _optional_one(rows, "Betting")
+
+    def settlement(self, bet_id: str) -> dict[str, str] | None:
+        rows = self.database.query(
+            "settlement",
+            f"""
+            SELECT status, COALESCE(result, '') AS result,
+                   COALESCE(payout_amount::text, '') AS payout,
+                   COALESCE(payout_currency, '') AS currency,
+                   revision_number::text AS revision_number
+            FROM bet WHERE bet_id = {uuid_literal(bet_id)}
+            """,
+        )
+        return _optional_one(rows, "Settlement")
+
+    def wallet(self, user_id: str) -> dict[str, str] | None:
+        rows = self.database.query(
+            "wallet",
+            f"""
+            SELECT available_amount::text AS available, locked_amount::text AS locked,
+                   recovery_debt_amount::text AS debt,
+                   (recovery_frozen_at IS NOT NULL)::int AS frozen
+            FROM account WHERE user_id = {uuid_literal(user_id)}
+            """,
+        )
+        return _optional_one(rows, "Wallet")
+
+    def tombstone(self, event_id: str) -> str | None:
+        rows = self.database.query(
+            "settlement",
+            f"""
+            SELECT terminal_status FROM event_lifecycle_tombstone
+            WHERE event_id = {uuid_literal(event_id)}
+            """,
+        )
+        if not rows:
+            return None
+        if len(rows) != 1:
+            raise RuntimeError("Settlement has conflicting lifecycle tombstones")
+        return rows[0]["terminal_status"]
+
+
+def _optional_one(rows: list[dict[str, str]], owner: str) -> dict[str, str] | None:
+    if not rows:
+        return None
+    if len(rows) != 1:
+        raise RuntimeError(f"{owner} returned conflicting aggregate rows")
+    return rows[0]


## `test(e2e): read correction evidence`

diff --git a/e2e/correction_oracles.py b/e2e/correction_oracles.py
new file mode 100644
index 0000000..c506e3e
--- /dev/null
+++ b/e2e/correction_oracles.py
@@ -0,0 +1,77 @@
+from __future__ import annotations
+
+from scripts.cold_gate.database import PostgresClient, uuid_literal
+
+
+class CorrectionOracles:
+    def __init__(self, database: PostgresClient) -> None:
+        self.database = database
+
+    def candidates(self, event_id: str) -> list[dict[str, str]]:
+        return self.database.query(
+            "settlement",
+            f"""
+            SELECT candidate_id::text AS candidate_id, candidate_sequence::text AS sequence,
+                   state, COALESCE(decision_reason, '') AS decision_reason,
+                   (decided_at IS NOT NULL)::int AS decided,
+                   COALESCE(replaces_candidate_id::text, '') AS replaces_candidate_id
+            FROM result_candidate WHERE event_id = {uuid_literal(event_id)}
+            ORDER BY candidate_sequence
+            """,
+        )
+
+    def accepted_candidate(self, event_id: str) -> dict[str, str] | None:
+        rows = self.database.query(
+            "settlement",
+            f"""
+            SELECT candidate_id::text AS candidate_id, state, decision_reason,
+                   (match_result.accepted_candidate_id = candidate_id)::int AS accepted
+            FROM result_candidate
+            JOIN match_result USING (event_id)
+            WHERE event_id = {uuid_literal(event_id)} AND state = 'ACCEPTED'
+            """,
+        )
+        return _optional_one(rows, "accepted result candidate")
+
+    def revision(self, bet_id: str, number: int = 1) -> dict[str, str] | None:
+        if number < 1:
+            raise ValueError("revision number must be positive")
+        rows = self.database.query(
+            "settlement",
+            f"""
+            SELECT revision_id::text AS revision_id, state, attempt_count::text AS attempt_count,
+                   previous_result, new_result,
+                   previous_payout_amount::text AS previous_payout,
+                   new_payout_amount::text AS new_payout,
+                   COALESCE(wallet_status, '') AS wallet_status,
+                   COALESCE(wallet_queue_sequence::text, '') AS queue_sequence,
+                   (wallet_operation_group_id IS NOT NULL)::int AS operation_group,
+                   (applied_at IS NOT NULL)::int AS applied,
+                   COALESCE(last_error_code, '') AS last_error
+            FROM settlement_revision
+            WHERE bet_id = {uuid_literal(bet_id)} AND revision_number = {number}
+            """,
+        )
+        return _optional_one(rows, "settlement revision")
+
+    def wallet_adjustment(self, revision_id: str) -> dict[str, str] | None:
+        rows = self.database.query(
+            "wallet",
+            f"""
+            SELECT status, delta_amount::text AS delta,
+                   COALESCE(queue_sequence::text, '') AS queue_sequence,
+                   (operation_group_id IS NOT NULL)::int AS operation_group,
+                   (applied_at IS NOT NULL)::int AS applied,
+                   (next_attempt_at IS NOT NULL)::int AS retry_scheduled
+            FROM wallet_adjustment WHERE revision_id = {uuid_literal(revision_id)}
+            """,
+        )
+        return _optional_one(rows, "Wallet adjustment")
+
+
+def _optional_one(rows: list[dict[str, str]], name: str) -> dict[str, str] | None:
+    if not rows:
+        return None
+    if len(rows) != 1:
+        raise RuntimeError(f"conflicting {name} rows")
+    return rows[0]


## `test(e2e): read exactly-once side effects`

diff --git a/e2e/placement_oracles.py b/e2e/placement_oracles.py
new file mode 100644
index 0000000..fb5a2ec
--- /dev/null
+++ b/e2e/placement_oracles.py
@@ -0,0 +1,66 @@
+from __future__ import annotations
+
+from scripts.cold_gate.database import PostgresClient, uuid_literal
+
+
+class PlacementOracles:
+    def __init__(self, database: PostgresClient) -> None:
+        self.database = database
+
+    def wallet_debit(self, bet_id: str) -> dict[str, str]:
+        key = uuid_literal(bet_id)
+        return self.database.one(
+            "wallet",
+            f"""
+            SELECT
+              (SELECT count(*) FROM wallet_operation
+               WHERE idempotency_key = {key})::text AS operation_count,
+              COALESCE((SELECT min(caller_id) FROM wallet_operation
+                        WHERE idempotency_key = {key}), '') AS caller,
+              COALESCE((SELECT min(operation_kind) FROM wallet_operation
+                        WHERE idempotency_key = {key}), '') AS kind,
+              COALESCE((SELECT min(status) FROM wallet_operation
+                        WHERE idempotency_key = {key}), '') AS status,
+              COALESCE((SELECT min((operation_group_id IS NOT NULL)::int) FROM wallet_operation
+                        WHERE idempotency_key = {key}), 0)::text AS operation_group,
+              (SELECT count(*) FROM ledger_entry
+               WHERE idempotency_key = {key})::text AS ledger_count,
+              (SELECT count(*) FROM outbox_event
+               WHERE operation_key = {key})::text AS outbox_count,
+              COALESCE((SELECT min(topic) FROM outbox_event
+                        WHERE operation_key = {key}), '') AS outbox_topic,
+              COALESCE((SELECT min(schema_name) FROM outbox_event
+                        WHERE operation_key = {key}), '') AS outbox_schema
+            """,
+        )
+
+    def betting_outbox(self, user_id: str, bet_id: str) -> dict[str, str]:
+        return self.database.one(
+            "betting",
+            f"""
+            SELECT count(*)::text AS event_count,
+                   COALESCE(min(topic), '') AS topic,
+                   COALESCE(min(schema_name), '') AS schema,
+                   min((published_at IS NOT NULL)::int)::text AS published
+            FROM outbox_event
+            WHERE partition_key = {uuid_literal(user_id)}::text
+              AND schema_name = 'BetPlacedRequested'
+              AND payload IS NOT NULL
+              AND EXISTS (SELECT 1 FROM bet WHERE bet_id = {uuid_literal(bet_id)})
+            """,
+        )
+
+    def settlement_outbox(self, partition_id: str, schema_name: str) -> dict[str, str]:
+        if schema_name not in {"BetSettled", "BetVoided", "BetResolutionRevised"}:
+            raise ValueError("Settlement outbox schema is outside the E2E contract")
+        return self.database.one(
+            "settlement",
+            f"""
+            SELECT count(*)::text AS event_count,
+                   COALESCE(min(topic), '') AS topic,
+                   min((published_at IS NOT NULL)::int)::text AS published
+            FROM outbox_event
+            WHERE partition_key = {uuid_literal(partition_id)}::text
+              AND schema_name = '{schema_name}'
+            """,
+        )


