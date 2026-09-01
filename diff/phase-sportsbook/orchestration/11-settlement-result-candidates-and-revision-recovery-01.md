# 정산 결과 후보와 수정 복구

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


## `test(e2e): verify result-first settlement`

diff --git a/e2e/scenario_05_result_first.py b/e2e/scenario_05_result_first.py
new file mode 100644
index 0000000..f99af24
--- /dev/null
+++ b/e2e/scenario_05_result_first.py
@@ -0,0 +1,49 @@
+from __future__ import annotations
+
+import time
+
+from e2e.assertions import require_fields
+from e2e.model import ScenarioIds
+from e2e.runtime import E2eRuntime
+from e2e.terminal import wait_base_settlement
+from scripts.cold_gate.polling import poll_until
+
+
+NAME = "result-before-placement-settlement"
+
+
+def run(runtime: E2eRuntime) -> None:
+    fixture = ScenarioIds.create(5)
+    runtime.seed(fixture)
+    runtime.fixtures.publish(
+        "MatchResult", fixture.match_result("WON", int(time.time() * 1000) - 5_000)
+    )
+    candidate = poll_until(
+        "result-first accepted candidate",
+        lambda: runtime.corrections.accepted_candidate(fixture.event),
+        lambda row: row is not None,
+        timeout=60,
+        interval=0.25,
+    )
+    require_fields(
+        candidate,
+        {"state": "ACCEPTED", "decision_reason": "FIRST_RESULT", "accepted": "1"},
+        "result-first candidate",
+    )
+
+    token = runtime.user_token(fixture)
+    placement = runtime.bets.place(fixture, token)
+    require_fields(
+        placement.__dict__,
+        {"http_status": 201, "status": "ACCEPTED"},
+        "result-first placement",
+    )
+    wait_base_settlement(runtime, fixture, placement.bet_id, "WON", 20_000, 110_000)
+    candidates = runtime.corrections.candidates(fixture.event)
+    if len(candidates) != 1:
+        raise RuntimeError("result-first flow created duplicate candidates")
+    require_fields(
+        candidates[0],
+        {"state": "ACCEPTED", "decision_reason": "FIRST_RESULT", "decided": "1"},
+        "result-first immutable candidate",
+    )


## `test(e2e): verify payout increase correction`

diff --git a/e2e/scenario_06_payout_increase.py b/e2e/scenario_06_payout_increase.py
new file mode 100644
index 0000000..9e4e5b6
--- /dev/null
+++ b/e2e/scenario_06_payout_increase.py
@@ -0,0 +1,86 @@
+from __future__ import annotations
+
+import time
+
+from e2e.assertions import require_fields, wait_fields
+from e2e.model import ScenarioIds
+from e2e.runtime import E2eRuntime
+from e2e.terminal import wait_base_settlement
+
+
+NAME = "payout-increase-correction"
+
+
+def run(runtime: E2eRuntime) -> None:
+    fixture = ScenarioIds.create(6)
+    runtime.seed(fixture)
+    placement = runtime.bets.place(fixture, runtime.user_token(fixture))
+    require_fields(placement.__dict__, {"http_status": 201}, "increase placement")
+    wait_fields(
+        "increase placement projection",
+        lambda: runtime.base.settlement(placement.bet_id),
+        {"status": "PENDING"},
+        terminal={"status": frozenset({"SETTLED", "VOIDED"})},
+    )
+    base_time = int(time.time() * 1000) - 10_000
+    runtime.fixtures.publish("MatchResult", fixture.match_result("LOST", base_time))
+    wait_base_settlement(runtime, fixture, placement.bet_id, "LOST", 0, 90_000)
+
+    runtime.fixtures.publish("MatchResult", fixture.match_result("WON", base_time + 1_000))
+    revision = wait_fields(
+        "payout increase revision",
+        lambda: runtime.corrections.revision(placement.bet_id),
+        {
+            "state": "APPLIED",
+            "attempt_count": "1",
+            "previous_result": "LOST",
+            "new_result": "WON",
+            "previous_payout": "0",
+            "new_payout": "20000",
+            "wallet_status": "APPLIED",
+            "queue_sequence": "",
+            "operation_group": "1",
+            "applied": "1",
+        },
+        terminal={"state": frozenset({"REJECTED", "EXHAUSTED"})},
+        timeout=90,
+    )
+    revision_id = str(revision["revision_id"])
+    require_fields(
+        runtime.corrections.wallet_adjustment(revision_id) or {},
+        {
+            "status": "APPLIED",
+            "delta": "20000",
+            "queue_sequence": "",
+            "operation_group": "1",
+            "applied": "1",
+            "retry_scheduled": "0",
+        },
+        "payout increase Wallet adjustment",
+    )
+    wait_fields(
+        "payout increase Betting projection",
+        lambda: runtime.base.betting(placement.bet_id),
+        {
+            "status": "SETTLED",
+            "result": "WON",
+            "payout": "20000",
+            "revision_number": "1",
+            "revision_id": revision_id,
+        },
+    )
+    wait_fields(
+        "payout increase Wallet balance",
+        lambda: runtime.base.wallet(fixture.user),
+        {"available": "110000", "locked": "0", "debt": "0", "frozen": "0"},
+    )
+    candidates = runtime.corrections.candidates(fixture.event)
+    if len(candidates) != 2:
+        raise RuntimeError("payout increase candidate inventory drifted")
+    require_fields(candidates[0], {"state": "SUPERSEDED", "decision_reason": "AUTO_CORRECTION"}, "base candidate")
+    require_fields(candidates[1], {"state": "ACCEPTED", "decision_reason": "AUTO_CORRECTION"}, "corrected candidate")
+    wait_fields(
+        "payout increase event publication",
+        lambda: runtime.placements.settlement_outbox(placement.bet_id, "BetResolutionRevised"),
+        {"event_count": "1", "topic": "bet.resolution.revised.v1", "published": "1"},
+    )


## `test(e2e): bound blocked correction recovery`

diff --git a/e2e/blocked_correction.py b/e2e/blocked_correction.py
new file mode 100644
index 0000000..f19ce89
--- /dev/null
+++ b/e2e/blocked_correction.py
@@ -0,0 +1,65 @@
+from __future__ import annotations
+
+from collections.abc import Mapping
+
+from e2e.assertions import require_fields, wait_fields
+from e2e.runtime import E2eRuntime
+from scripts.cold_gate.polling import poll_until
+
+
+def wait_blocked_decrease(runtime: E2eRuntime, bet_id: str) -> Mapping[str, object]:
+    def blocked(row: dict[str, str] | None) -> bool:
+        if row is None:
+            return False
+        if row.get("state") in {"REJECTED", "EXHAUSTED"}:
+            raise RuntimeError(f"payout decrease entered terminal state {row['state']}")
+        attempts = int(row.get("attempt_count", "0"))
+        if attempts >= 12:
+            raise RuntimeError("payout decrease exhausted before recovery deposit")
+        if row.get("state") != "BLOCKED":
+            return False
+        require_fields(
+            row,
+            {
+                "previous_result": "WON",
+                "new_result": "LOST",
+                "previous_payout": "20000",
+                "new_payout": "0",
+                "wallet_status": "BLOCKED",
+                "queue_sequence": "1",
+                "operation_group": "0",
+                "applied": "0",
+            },
+            "blocked payout decrease",
+        )
+        if attempts < 1:
+            raise RuntimeError("blocked payout decrease has no attempt proof")
+        return True
+
+    return poll_until(
+        "blocked payout decrease",
+        lambda: runtime.corrections.revision(bet_id),
+        blocked,
+        timeout=15,
+        interval=0.1,
+    )
+
+
+def wait_applied_decrease(runtime: E2eRuntime, bet_id: str) -> Mapping[str, object]:
+    return wait_fields(
+        "applied payout decrease",
+        lambda: runtime.corrections.revision(bet_id),
+        {
+            "state": "APPLIED",
+            "previous_result": "WON",
+            "new_result": "LOST",
+            "previous_payout": "20000",
+            "new_payout": "0",
+            "wallet_status": "APPLIED",
+            "queue_sequence": "1",
+            "operation_group": "1",
+            "applied": "1",
+        },
+        terminal={"state": frozenset({"REJECTED", "EXHAUSTED"})},
+        timeout=90,
+    )


## `test(e2e): verify blocked payout recovery`

diff --git a/e2e/scenario_07_blocked_recovery.py b/e2e/scenario_07_blocked_recovery.py
new file mode 100644
index 0000000..48eaee7
--- /dev/null
+++ b/e2e/scenario_07_blocked_recovery.py
@@ -0,0 +1,95 @@
+from __future__ import annotations
+
+import time
+
+from e2e.assertions import require_fields, wait_fields
+from e2e.blocked_correction import wait_applied_decrease, wait_blocked_decrease
+from e2e.model import ScenarioIds
+from e2e.runtime import E2eRuntime
+from e2e.terminal import wait_base_settlement
+
+
+NAME = "payout-decrease-blocked-recovery"
+
+
+def run(runtime: E2eRuntime) -> None:
+    fixture = ScenarioIds.create(7)
+    runtime.seed(fixture)
+    placement = runtime.bets.place(fixture, runtime.user_token(fixture))
+    require_fields(placement.__dict__, {"http_status": 201}, "decrease placement")
+    wait_fields(
+        "decrease placement projection",
+        lambda: runtime.base.settlement(placement.bet_id),
+        {"status": "PENDING"},
+        terminal={"status": frozenset({"SETTLED", "VOIDED"})},
+    )
+    base_time = int(time.time() * 1000) - 10_000
+    runtime.fixtures.publish("MatchResult", fixture.match_result("WON", base_time))
+    wait_base_settlement(runtime, fixture, placement.bet_id, "WON", 20_000, 110_000)
+
+    runtime.wallet_api.transfer(fixture, "withdraw", 110_000, "e2e-withdraw-07")
+    require_fields(
+        runtime.base.wallet(fixture.user) or {},
+        {"available": "0", "locked": "0", "debt": "0", "frozen": "0"},
+        "drained payout balance",
+    )
+    runtime.fixtures.publish("MatchResult", fixture.match_result("LOST", base_time + 1_000))
+    blocked = wait_blocked_decrease(runtime, placement.bet_id)
+    revision_id = str(blocked["revision_id"])
+    require_fields(
+        runtime.base.wallet(fixture.user) or {},
+        {"available": "0", "locked": "0", "debt": "20000", "frozen": "1"},
+        "blocked Wallet account",
+    )
+    require_fields(
+        runtime.corrections.wallet_adjustment(revision_id) or {},
+        {
+            "status": "BLOCKED",
+            "delta": "-20000",
+            "queue_sequence": "1",
+            "operation_group": "0",
+            "applied": "0",
+            "retry_scheduled": "1",
+        },
+        "blocked Wallet adjustment",
+    )
+
+    runtime.wallet_api.transfer(
+        fixture, "deposit", 20_000, "e2e-recovery-deposit-07"
+    )
+    applied = wait_applied_decrease(runtime, placement.bet_id)
+    if applied["revision_id"] != revision_id:
+        raise RuntimeError("recovery changed the correction identity")
+    require_fields(
+        runtime.corrections.wallet_adjustment(revision_id) or {},
+        {
+            "status": "APPLIED",
+            "delta": "-20000",
+            "queue_sequence": "1",
+            "operation_group": "1",
+            "applied": "1",
+            "retry_scheduled": "0",
+        },
+        "recovered Wallet adjustment",
+    )
+    wait_fields(
+        "recovered Settlement projection",
+        lambda: runtime.base.settlement(placement.bet_id),
+        {"status": "SETTLED", "result": "LOST", "payout": "0", "revision_number": "1"},
+    )
+    wait_fields(
+        "recovered Betting projection",
+        lambda: runtime.base.betting(placement.bet_id),
+        {
+            "status": "SETTLED",
+            "result": "LOST",
+            "payout": "0",
+            "revision_number": "1",
+            "revision_id": revision_id,
+        },
+    )
+    wait_fields(
+        "recovered Wallet account",
+        lambda: runtime.base.wallet(fixture.user),
+        {"available": "0", "locked": "0", "debt": "0", "frozen": "0"},
+    )


## `test(e2e): call Settlement operator APIs`

diff --git a/e2e/settlement_admin_api.py b/e2e/settlement_admin_api.py
new file mode 100644
index 0000000..488daf0
--- /dev/null
+++ b/e2e/settlement_admin_api.py
@@ -0,0 +1,93 @@
+from __future__ import annotations
+
+import dataclasses
+import uuid
+
+from e2e.assertions import require_uuidv7
+from scripts.cold_gate.container_http import ContainerHttpClient
+
+
+@dataclasses.dataclass(frozen=True)
+class AdminMutation:
+    action_id: str
+    payload: dict[str, object]
+
+
+class SettlementAdminApi:
+    def __init__(self, client: ContainerHttpClient) -> None:
+        if client.service != "admin":
+            raise ValueError("Settlement admin calls must originate in Admin")
+        self.client = client
+
+    def approve(self, candidate_id: str, token: str, key: str) -> AdminMutation:
+        return self._candidate(candidate_id, "approve", token, key, None)
+
+    def reject(
+        self, candidate_id: str, token: str, key: str, reason: str
+    ) -> AdminMutation:
+        if not reason or len(reason) > 256:
+            raise ValueError("candidate rejection reason is invalid")
+        return self._candidate(candidate_id, "reject", token, key, {"reason": reason})
+
+    def retry(self, revision_id: str, token: str, key: str) -> AdminMutation:
+        _canonical_uuid(revision_id, "revision")
+        response = self.client.request(
+            "POST",
+            f"/admin/v1/settlements/revisions/{revision_id}/retry",
+            headers=self._headers(token, key),
+        ).require_status(202)
+        payload = _object(response.json())
+        if payload.get("idempotencyKey") != key or payload.get("outcome") not in {
+            "QUEUED",
+            "REPLAY",
+        }:
+            raise RuntimeError("revision retry receipt drifted")
+        return AdminMutation(
+            require_uuidv7(response.header("X-Admin-Action-Id"), "admin action ID"),
+            payload,
+        )
+
+    def _candidate(
+        self,
+        candidate_id: str,
+        action: str,
+        token: str,
+        key: str,
+        body: object | None,
+    ) -> AdminMutation:
+        _canonical_uuid(candidate_id, "candidate")
+        response = self.client.request(
+            "POST",
+            f"/admin/v1/settlements/result-candidates/{candidate_id}/{action}",
+            headers=self._headers(token, key),
+            body=body,
+        ).require_status(200)
+        payload = _object(response.json())
+        expected = "CANDIDATE_APPROVED" if action == "approve" else "CANDIDATE_REJECTED"
+        if (
+            payload.get("idempotencyKey") != key
+            or payload.get("outcome") != expected
+            or payload.get("replay") is not False
+        ):
+            raise RuntimeError("candidate mutation receipt drifted")
+        return AdminMutation(
+            require_uuidv7(response.header("X-Admin-Action-Id"), "admin action ID"),
+            payload,
+        )
+
+    @staticmethod
+    def _headers(token: str, key: str) -> dict[str, str]:
+        _canonical_uuid(key, "idempotency")
+        return {"Authorization": "Bearer " + token, "Idempotency-Key": key}
+
+
+def _canonical_uuid(value: str, name: str) -> None:
+    parsed = uuid.UUID(value)
+    if str(parsed) != value:
+        raise ValueError(f"{name} ID must be canonical")
+
+
+def _object(value: object) -> dict[str, object]:
+    if not isinstance(value, dict):
+        raise RuntimeError("Admin response is not an object")
+    return value


## `test(e2e): bind Settlement operator client`

diff --git a/e2e/runtime.py b/e2e/runtime.py
index 90e3045..5711555 100644
--- a/e2e/runtime.py
+++ b/e2e/runtime.py
@@ -7,6 +7,7 @@ from e2e.bet_api import BetApi
 from e2e.correction_oracles import CorrectionOracles
 from e2e.model import ScenarioIds
 from e2e.placement_oracles import PlacementOracles
+from e2e.settlement_admin_api import SettlementAdminApi
 from e2e.wallet_api import WalletApi
 from scripts.cold_gate.build import ReleaseArtifacts
 from scripts.cold_gate.chaos import ChaosClient
@@ -52,6 +53,7 @@ class E2eRuntime:
             ContainerHttpClient(compose, "wallet"),
             secrets.environment["WALLET_PLATFORM_API_KEY"],
         )
+        self.settlement_admin = SettlementAdminApi(ContainerHttpClient(compose, "admin"))
         self.base = BaseOracles(database)
         self.placements = PlacementOracles(database)
         self.corrections = CorrectionOracles(database)


## `test(e2e): verify candidate operator decisions`

diff --git a/e2e/scenario_08_admin_candidates.py b/e2e/scenario_08_admin_candidates.py
new file mode 100644
index 0000000..8646541
--- /dev/null
+++ b/e2e/scenario_08_admin_candidates.py
@@ -0,0 +1,97 @@
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
+NAME = "admin-candidate-approve-reject"
+
+
+def run(runtime: E2eRuntime) -> None:
+    token = runtime.admin_token()
+    approved_fixture = ScenarioIds.create(81)
+    approved_at = int(time.time() * 1000) + 3_000
+    runtime.fixtures.publish(
+        "MatchResult", approved_fixture.match_result("WON", approved_at)
+    )
+    approved_candidate = _pending_candidate(runtime, approved_fixture.event)
+    poll_until(
+        "candidate approval eligibility",
+        lambda: int(time.time() * 1000),
+        lambda now: now >= approved_at + 100,
+        timeout=10,
+        interval=0.1,
+    )
+    approval = runtime.settlement_admin.approve(
+        approved_candidate["candidate_id"],
+        token,
+        "44000000-0000-7000-8000-000000000081",
+    )
+    require_fields(
+        approval.payload,
+        {"outcome": "CANDIDATE_APPROVED", "replay": False},
+        "candidate approval receipt",
+    )
+    wait_fields(
+        "approved candidate state",
+        lambda: _single_candidate(runtime, approved_fixture.event),
+        {"state": "ACCEPTED", "decision_reason": "OPERATOR_APPROVED", "decided": "1"},
+        terminal={"state": frozenset({"REJECTED"})},
+    )
+
+    rejected_fixture = ScenarioIds.create(82)
+    runtime.fixtures.publish(
+        "MatchResult",
+        rejected_fixture.match_result("LOST", int(time.time() * 1000) + 60_000),
+    )
+    rejected_candidate = _pending_candidate(runtime, rejected_fixture.event)
+    rejection = runtime.settlement_admin.reject(
+        rejected_candidate["candidate_id"],
+        token,
+        "44000000-0000-7000-8000-000000000082",
+        "e2e rejected candidate",
+    )
+    require_fields(
+        rejection.payload,
+        {"outcome": "CANDIDATE_REJECTED", "replay": False},
+        "candidate rejection receipt",
+    )
+    wait_fields(
+        "rejected candidate state",
+        lambda: _single_candidate(runtime, rejected_fixture.event),
+        {
+            "state": "REJECTED",
+            "decision_reason": "e2e rejected candidate",
+            "decided": "1",
+        },
+        terminal={"state": frozenset({"ACCEPTED", "SUPERSEDED"})},
+    )
+
+
+def _pending_candidate(runtime: E2eRuntime, event_id: str) -> dict[str, str]:
+    candidate = poll_until(
+        "pending result candidate",
+        lambda: _single_candidate(runtime, event_id),
+        lambda row: row is not None,
+        timeout=30,
+        interval=0.25,
+    )
+    require_fields(
+        candidate,
+        {"state": "PENDING", "decision_reason": "", "decided": "0"},
+        "pending result candidate",
+    )
+    return candidate
+
+
+def _single_candidate(runtime: E2eRuntime, event_id: str) -> dict[str, str] | None:
+    rows = runtime.corrections.candidates(event_id)
+    if not rows:
+        return None
+    if len(rows) != 1:
+        raise RuntimeError("operator scenario has conflicting candidates")
+    return rows[0]


