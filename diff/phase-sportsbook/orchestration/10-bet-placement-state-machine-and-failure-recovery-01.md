# 베팅 접수 상태 머신과 장애 복구

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


## `test(e2e): verify authenticated bet settlement`

diff --git a/e2e/scenario_01_authenticated_settlement.py b/e2e/scenario_01_authenticated_settlement.py
new file mode 100644
index 0000000..4856597
--- /dev/null
+++ b/e2e/scenario_01_authenticated_settlement.py
@@ -0,0 +1,86 @@
+from __future__ import annotations
+
+import time
+
+from e2e.assertions import require_fields, wait_fields
+from e2e.model import ScenarioIds
+from e2e.runtime import E2eRuntime
+
+
+NAME = "authenticated-placement-and-settlement"
+
+
+def run(runtime: E2eRuntime) -> None:
+    fixture = ScenarioIds.create(1)
+    runtime.seed(fixture)
+    token = runtime.user_token(fixture)
+    placement = runtime.bets.place(fixture, token)
+    require_fields(
+        placement.__dict__,
+        {"http_status": 201, "status": "ACCEPTED"},
+        "authenticated placement",
+    )
+    wait_fields(
+        "Settlement placement projection",
+        lambda: runtime.base.settlement(placement.bet_id),
+        {"status": "PENDING", "revision_number": "0"},
+        terminal={"status": frozenset({"SETTLED", "VOIDED"})},
+    )
+
+    runtime.fixtures.publish(
+        "MatchResult",
+        fixture.match_result("WON", int(time.time() * 1000) - 5_000),
+    )
+    wait_fields(
+        "Settlement base resolution",
+        lambda: runtime.base.settlement(placement.bet_id),
+        {
+            "status": "SETTLED",
+            "result": "WON",
+            "payout": "20000",
+            "currency": "KRW",
+            "revision_number": "0",
+        },
+        terminal={"status": frozenset({"VOIDED"})},
+    )
+    wait_fields(
+        "Betting base resolution",
+        lambda: runtime.base.betting(placement.bet_id),
+        {
+            "status": "SETTLED",
+            "placement_phase": "RISK_COMMITTED",
+            "risk_committed": "1",
+            "wallet_confirmed": "1",
+            "result": "WON",
+            "payout": "20000",
+            "currency": "KRW",
+            "revision_number": "0",
+        },
+        terminal={"status": frozenset({"REJECTED", "VOIDED"})},
+    )
+    wait_fields(
+        "Wallet base resolution",
+        lambda: runtime.base.wallet(fixture.user),
+        {"available": "110000", "locked": "0", "debt": "0", "frozen": "0"},
+    )
+    outbox = wait_fields(
+        "Settlement event publication",
+        lambda: runtime.placements.settlement_outbox(fixture.event, "BetSettled"),
+        {"event_count": "1", "topic": "bet.settled.v1", "published": "1"},
+    )
+    require_fields(outbox, {"event_count": "1"}, "Settlement outbox")
+
+    queried = runtime.bets.get(placement.bet_id, token)
+    resolution = queried.get("resolution")
+    if not isinstance(resolution, dict):
+        raise RuntimeError("public settled bet has no resolution")
+    require_fields(
+        resolution,
+        {
+            "settlementResult": "WON",
+            "settledPayout": {"amount": 20_000, "currency": "KRW"},
+            "resolutionEventId": fixture.event,
+            "resolutionRevisionNumber": 0,
+        },
+        "public settled resolution",
+    )


## `test(e2e): verify Risk outage recovery`

diff --git a/e2e/scenario_02_risk_recovery.py b/e2e/scenario_02_risk_recovery.py
new file mode 100644
index 0000000..f221fdf
--- /dev/null
+++ b/e2e/scenario_02_risk_recovery.py
@@ -0,0 +1,94 @@
+from __future__ import annotations
+
+import time
+
+from e2e.assertions import require_fields, wait_fields
+from e2e.model import ScenarioIds
+from e2e.runtime import E2eRuntime
+
+
+NAME = "risk-outage-pending-recovery"
+
+
+def run(runtime: E2eRuntime) -> None:
+    fixture = ScenarioIds.create(2)
+    runtime.seed(fixture)
+    token = runtime.user_token(fixture)
+    placement = None
+    try:
+        disabled = runtime.chaos.set_enabled("betting_to_risk", False)
+        require_fields(disabled, {"enabled": False}, "Risk proxy disable")
+        placement = runtime.bets.place(fixture, token)
+        require_fields(
+            placement.__dict__,
+            {"http_status": 202, "status": "PENDING"},
+            "Risk-outage placement",
+        )
+        require_fields(
+            runtime.base.betting(placement.bet_id) or {},
+            {
+                "status": "PENDING",
+                "placement_phase": "CREATED",
+                "risk_committed": "0",
+                "wallet_confirmed": "0",
+            },
+            "Risk-outage Betting checkpoint",
+        )
+        require_fields(
+            runtime.base.wallet(fixture.user) or {},
+            {"available": "100000", "locked": "0"},
+            "Risk-outage Wallet state",
+        )
+        require_fields(
+            runtime.placements.wallet_debit(placement.bet_id),
+            {"operation_count": "0", "ledger_count": "0", "outbox_count": "0"},
+            "Risk-outage Wallet effects",
+        )
+        if runtime.risk.scalar("EXISTS", f"risk:reservation:{placement.bet_id}") != "0":
+            raise RuntimeError("Risk outage created a reservation")
+    finally:
+        enabled = runtime.chaos.set_enabled("betting_to_risk", True)
+        require_fields(enabled, {"enabled": True}, "Risk proxy restoration")
+    if placement is None:
+        raise RuntimeError("Risk-outage placement did not return a bet")
+
+    wait_fields(
+        "Risk-outage placement recovery",
+        lambda: runtime.base.betting(placement.bet_id),
+        {
+            "status": "ACCEPTED",
+            "placement_phase": "RISK_COMMITTED",
+            "risk_committed": "1",
+            "wallet_confirmed": "1",
+        },
+        terminal={"status": frozenset({"REJECTED", "VOIDED", "SETTLED"})},
+    )
+    wait_fields(
+        "Risk-outage Wallet debit",
+        lambda: runtime.base.wallet(fixture.user),
+        {"available": "90000", "locked": "10000"},
+    )
+    reservation = f"risk:reservation:{placement.bet_id}"
+    expected = {
+        "state": "COMMITTED",
+        "userId": fixture.user,
+        "betId": placement.bet_id,
+        "stake": "10000",
+        "currency": "KRW",
+        "selectionCount": "1",
+        "selections": fixture.selection,
+    }
+    for field, value in expected.items():
+        if runtime.risk.scalar("HGET", reservation, field) != value:
+            raise RuntimeError(f"Risk reservation {field} drifted")
+    if runtime.risk.scalar("EXISTS", f"risk:reservations:user:{{{fixture.user}}}:bets") != "0":
+        raise RuntimeError("Risk retained an active reservation footprint")
+
+    runtime.fixtures.publish(
+        "MatchResult", fixture.match_result("LOST", int(time.time() * 1000) - 5_000)
+    )
+    wait_fields(
+        "Risk-outage terminal cleanup",
+        lambda: runtime.base.wallet(fixture.user),
+        {"available": "90000", "locked": "0", "debt": "0", "frozen": "0"},
+    )


## `test(e2e): verify exactly-once Wallet debit`

diff --git a/e2e/scenario_03_wallet_lost_response.py b/e2e/scenario_03_wallet_lost_response.py
new file mode 100644
index 0000000..95c91a2
--- /dev/null
+++ b/e2e/scenario_03_wallet_lost_response.py
@@ -0,0 +1,99 @@
+from __future__ import annotations
+
+import time
+
+from e2e.assertions import require_fields, wait_fields
+from e2e.model import ScenarioIds
+from e2e.runtime import E2eRuntime
+
+
+NAME = "wallet-lost-response-exactly-once"
+
+
+def run(runtime: E2eRuntime) -> None:
+    fixture = ScenarioIds.create(3)
+    runtime.seed(fixture)
+    token = runtime.user_token(fixture)
+    placement = None
+    toxic_created = False
+    try:
+        toxic = runtime.chaos.add_wallet_response_timeout()
+        toxic_created = True
+        require_fields(
+            toxic,
+            {"name": "e2e_wallet_response_timeout", "type": "timeout", "stream": "downstream"},
+            "Wallet response toxic",
+        )
+        placement = runtime.bets.place(fixture, token)
+        require_fields(
+            placement.__dict__,
+            {"http_status": 202, "status": "PENDING"},
+            "lost-response placement",
+        )
+        wait_fields(
+            "lost-response Wallet commit",
+            lambda: runtime.placements.wallet_debit(placement.bet_id),
+            {
+                "operation_count": "1",
+                "caller": "BETTING",
+                "kind": "BET_DEBIT",
+                "status": "SUCCEEDED",
+                "operation_group": "1",
+                "ledger_count": "2",
+                "outbox_count": "1",
+                "outbox_topic": "wallet.debited.v1",
+                "outbox_schema": "WalletDebited",
+            },
+        )
+        require_fields(
+            runtime.base.betting(placement.bet_id) or {},
+            {
+                "status": "PENDING",
+                "placement_phase": "RISK_RESERVED",
+                "risk_committed": "0",
+                "wallet_confirmed": "0",
+            },
+            "lost-response Betting checkpoint",
+        )
+        require_fields(
+            runtime.base.wallet(fixture.user) or {},
+            {"available": "90000", "locked": "10000"},
+            "lost-response Wallet state",
+        )
+        if runtime.risk.scalar("HGET", f"risk:reservation:{placement.bet_id}", "state") != "RESERVED":
+            raise RuntimeError("Risk reservation advanced before Wallet proof recovery")
+        require_fields(
+            runtime.placements.betting_outbox(fixture.user, placement.bet_id),
+            {"event_count": "0"},
+            "pre-recovery Betting outbox",
+        )
+    finally:
+        if toxic_created:
+            runtime.chaos.remove_wallet_response_timeout()
+    if placement is None:
+        raise RuntimeError("lost-response placement did not return a bet")
+
+    wait_fields(
+        "lost-response GET-first recovery",
+        lambda: runtime.base.betting(placement.bet_id),
+        {"status": "ACCEPTED", "placement_phase": "RISK_COMMITTED"},
+        terminal={"status": frozenset({"REJECTED", "VOIDED", "SETTLED"})},
+    )
+    require_fields(
+        runtime.placements.wallet_debit(placement.bet_id),
+        {"operation_count": "1", "ledger_count": "2", "outbox_count": "1"},
+        "recovered Wallet debit",
+    )
+    wait_fields(
+        "recovered Betting outbox",
+        lambda: runtime.placements.betting_outbox(fixture.user, placement.bet_id),
+        {"event_count": "1", "topic": "bet.placed.v1", "schema": "BetPlacedRequested"},
+    )
+    runtime.fixtures.publish(
+        "MatchResult", fixture.match_result("LOST", int(time.time() * 1000) - 5_000)
+    )
+    wait_fields(
+        "lost-response terminal cleanup",
+        lambda: runtime.base.wallet(fixture.user),
+        {"available": "90000", "locked": "0", "debt": "0", "frozen": "0"},
+    )


## `test(e2e): compare Wallet operation text keys`

diff --git a/e2e/placement_oracles.py b/e2e/placement_oracles.py
index fb5a2ec..3cc3f14 100644
--- a/e2e/placement_oracles.py
+++ b/e2e/placement_oracles.py
@@ -8,7 +8,7 @@ class PlacementOracles:
         self.database = database
 
     def wallet_debit(self, bet_id: str) -> dict[str, str]:
-        key = uuid_literal(bet_id)
+        key = uuid_literal(bet_id) + "::text"
         return self.database.one(
             "wallet",
             f"""


