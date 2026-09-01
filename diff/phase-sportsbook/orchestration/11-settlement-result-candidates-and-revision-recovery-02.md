## `test(e2e): bound Settlement operator restart`

diff --git a/e2e/runtime.py b/e2e/runtime.py
index 5711555..e10bc58 100644
--- a/e2e/runtime.py
+++ b/e2e/runtime.py
@@ -54,6 +54,7 @@ class E2eRuntime:
             secrets.environment["WALLET_PLATFORM_API_KEY"],
         )
         self.settlement_admin = SettlementAdminApi(ContainerHttpClient(compose, "admin"))
+        self.settlement_http = ContainerHttpClient(compose, "settlement")
         self.base = BaseOracles(database)
         self.placements = PlacementOracles(database)
         self.corrections = CorrectionOracles(database)
@@ -92,3 +93,19 @@ class E2eRuntime:
             timeout=90,
             interval=1,
         )
+
+    def stop_settlement(self) -> None:
+        self.compose.run("stop", "--timeout", "30", "settlement", capture_output=True)
+
+    def start_settlement(self) -> None:
+        self.compose.run("start", "settlement", capture_output=True)
+        response = poll_until(
+            "Settlement restart readiness",
+            lambda: self.settlement_http.request("GET", "/actuator/health/readiness"),
+            lambda value: value.status == 200,
+            timeout=60,
+            interval=0.5,
+        )
+        payload = response.json()
+        if not isinstance(payload, dict) or payload.get("status") != "UP":
+            raise RuntimeError("Settlement restart readiness payload drifted")


## `test(e2e): stage a real blocked correction`

diff --git a/e2e/decrease_stimulus.py b/e2e/decrease_stimulus.py
new file mode 100644
index 0000000..16102a5
--- /dev/null
+++ b/e2e/decrease_stimulus.py
@@ -0,0 +1,43 @@
+from __future__ import annotations
+
+import dataclasses
+import time
+
+from e2e.assertions import wait_fields
+from e2e.blocked_correction import wait_blocked_decrease
+from e2e.model import ScenarioIds
+from e2e.runtime import E2eRuntime
+from e2e.terminal import wait_base_settlement
+
+
+@dataclasses.dataclass(frozen=True)
+class BlockedCorrection:
+    fixture: ScenarioIds
+    bet_id: str
+    revision_id: str
+
+
+def create_blocked_decrease(runtime: E2eRuntime, number: int) -> BlockedCorrection:
+    fixture = ScenarioIds.create(number)
+    runtime.seed(fixture)
+    placement = runtime.bets.place(fixture, runtime.user_token(fixture))
+    if placement.http_status != 201:
+        raise RuntimeError("blocked correction placement was not accepted")
+    wait_fields(
+        "blocked correction placement projection",
+        lambda: runtime.base.settlement(placement.bet_id),
+        {"status": "PENDING"},
+        terminal={"status": frozenset({"SETTLED", "VOIDED"})},
+    )
+    base_time = int(time.time() * 1000) - 10_000
+    runtime.fixtures.publish("MatchResult", fixture.match_result("WON", base_time))
+    wait_base_settlement(runtime, fixture, placement.bet_id, "WON", 20_000, 110_000)
+    runtime.wallet_api.transfer(
+        fixture,
+        "withdraw",
+        110_000,
+        f"e2e-drain-{number:02d}",
+    )
+    runtime.fixtures.publish("MatchResult", fixture.match_result("LOST", base_time + 1_000))
+    blocked = wait_blocked_decrease(runtime, placement.bet_id)
+    return BlockedCorrection(fixture, placement.bet_id, str(blocked["revision_id"]))


## `test(e2e): expose scoped database fixtures`

diff --git a/e2e/runtime.py b/e2e/runtime.py
index e10bc58..217fa82 100644
--- a/e2e/runtime.py
+++ b/e2e/runtime.py
@@ -47,6 +47,7 @@ class E2eRuntime:
         self.context = context
         self.compose = compose
         self.environment = secrets.environment
+        self.database = database
         self.signer = JwtSigner(context, secrets.private_key)
         self.bets = BetApi(HostHttpClient(f"http://127.0.0.1:{gateway_port}"))
         self.wallet_api = WalletApi(


## `test(e2e): verify Admin revision retry`

diff --git a/e2e/scenario_09_admin_revision_retry.py b/e2e/scenario_09_admin_revision_retry.py
new file mode 100644
index 0000000..597f68f
--- /dev/null
+++ b/e2e/scenario_09_admin_revision_retry.py
@@ -0,0 +1,97 @@
+from __future__ import annotations
+
+from e2e.assertions import require_fields, wait_fields
+from e2e.blocked_correction import wait_applied_decrease
+from e2e.decrease_stimulus import create_blocked_decrease
+from e2e.runtime import E2eRuntime
+from scripts.cold_gate.database import uuid_literal
+
+
+NAME = "admin-revision-retry"
+
+
+def run(runtime: E2eRuntime) -> None:
+    blocked = create_blocked_decrease(runtime, 9)
+    settlement_stopped = False
+    try:
+        runtime.stop_settlement()
+        settlement_stopped = True
+        updated = runtime.database.one(
+            "settlement",
+            f"""
+            UPDATE settlement_revision
+            SET attempt_count = 12,
+                next_retry_at = NULL,
+                last_error_code = 'WALLET_RETRY_EXHAUSTED',
+                updated_at = CURRENT_TIMESTAMP
+            WHERE revision_id = {uuid_literal(blocked.revision_id)}
+              AND state = 'BLOCKED'
+              AND wallet_status = 'BLOCKED'
+              AND lease_token IS NULL
+            RETURNING revision_id::text AS revision_id
+            """,
+        )
+        if updated["revision_id"] != blocked.revision_id:
+            raise RuntimeError("revision exhaustion fixture changed identity")
+
+        runtime.wallet_api.transfer(
+            blocked.fixture,
+            "deposit",
+            20_000,
+            "e2e-retry-proof-deposit-09",
+        )
+        wait_fields(
+            "operator retry Wallet proof",
+            lambda: runtime.corrections.wallet_adjustment(blocked.revision_id),
+            {
+                "status": "APPLIED",
+                "delta": "-20000",
+                "queue_sequence": "1",
+                "operation_group": "1",
+                "applied": "1",
+                "retry_scheduled": "0",
+            },
+            terminal={"status": frozenset({"REJECTED"})},
+            timeout=60,
+        )
+        runtime.start_settlement()
+        settlement_stopped = False
+
+        mutation = runtime.settlement_admin.retry(
+            blocked.revision_id,
+            runtime.admin_token(),
+            "44000000-0000-7000-8000-000000000009",
+        )
+        require_fields(
+            mutation.payload,
+            {
+                "outcome": "QUEUED",
+                "revisionState": "BLOCKED",
+                "attemptCount": 0,
+            },
+            "operator retry receipt",
+        )
+        if not mutation.payload.get("nextRetryAt"):
+            raise RuntimeError("operator retry receipt has no next attempt")
+        applied = wait_applied_decrease(runtime, blocked.bet_id)
+        if applied["revision_id"] != blocked.revision_id:
+            raise RuntimeError("operator retry changed revision identity")
+        wait_fields(
+            "operator retry Betting projection",
+            lambda: runtime.base.betting(blocked.bet_id),
+            {
+                "status": "SETTLED",
+                "result": "LOST",
+                "payout": "0",
+                "revision_number": "1",
+                "revision_id": blocked.revision_id,
+            },
+        )
+        wait_fields(
+            "operator retry Wallet account",
+            lambda: runtime.base.wallet(blocked.fixture.user),
+            {"available": "0", "locked": "0", "debt": "0", "frozen": "0"},
+        )
+    finally:
+        if settlement_stopped:
+            runtime.start_settlement()


## `test(e2e): hold future admin candidates`

diff --git a/e2e/scenario_08_admin_candidates.py b/e2e/scenario_08_admin_candidates.py
index 8646541..0a64c6e 100644
--- a/e2e/scenario_08_admin_candidates.py
+++ b/e2e/scenario_08_admin_candidates.py
@@ -9,12 +9,14 @@ from scripts.cold_gate.polling import poll_until
 
 
 NAME = "admin-candidate-approve-reject"
+APPROVAL_HORIZON_MILLIS = 60_000
+APPROVAL_ELIGIBILITY_TIMEOUT_SECONDS = 75
 
 
 def run(runtime: E2eRuntime) -> None:
     token = runtime.admin_token()
     approved_fixture = ScenarioIds.create(81)
-    approved_at = int(time.time() * 1000) + 3_000
+    approved_at = int(time.time() * 1000) + APPROVAL_HORIZON_MILLIS
     runtime.fixtures.publish(
         "MatchResult", approved_fixture.match_result("WON", approved_at)
     )
@@ -23,7 +25,7 @@ def run(runtime: E2eRuntime) -> None:
         "candidate approval eligibility",
         lambda: int(time.time() * 1000),
         lambda now: now >= approved_at + 100,
-        timeout=10,
+        timeout=APPROVAL_ELIGIBILITY_TIMEOUT_SECONDS,
         interval=0.1,
     )
     approval = runtime.settlement_admin.approve(
@@ -46,7 +48,9 @@ def run(runtime: E2eRuntime) -> None:
     rejected_fixture = ScenarioIds.create(82)
     runtime.fixtures.publish(
         "MatchResult",
-        rejected_fixture.match_result("LOST", int(time.time() * 1000) + 60_000),
+        rejected_fixture.match_result(
+            "LOST", int(time.time() * 1000) + APPROVAL_HORIZON_MILLIS
+        ),
     )
     rejected_candidate = _pending_candidate(runtime, rejected_fixture.event)
     rejection = runtime.settlement_admin.reject(
@@ -82,7 +86,7 @@ def _pending_candidate(runtime: E2eRuntime, event_id: str) -> dict[str, str]:
     )
     require_fields(
         candidate,
-        {"state": "PENDING", "decision_reason": "", "decided": "0"},
+        {"state": "PENDING", "decision_reason": "FUTURE_HELD", "decided": "0"},
         "pending result candidate",
     )
     return candidate
diff --git a/tests/test_admin_candidate_scenario.py b/tests/test_admin_candidate_scenario.py
new file mode 100644
index 0000000..4c1b890
--- /dev/null
+++ b/tests/test_admin_candidate_scenario.py
@@ -0,0 +1,39 @@
+import unittest
+
+from e2e import scenario_08_admin_candidates as subject
+
+
+class FakeCorrections:
+    def __init__(self, row: dict[str, str]) -> None:
+        self.row = row
+
+    def candidates(self, _event_id: str) -> list[dict[str, str]]:
+        return [self.row]
+
+
+class AdminCandidateScenarioTest(unittest.TestCase):
+    def test_requires_the_locked_future_hold_reason(self) -> None:
+        row = {
+            "candidate_id": "candidate",
+            "state": "PENDING",
+            "decision_reason": "FUTURE_HELD",
+            "decided": "0",
+        }
+        runtime = type("Runtime", (), {"corrections": FakeCorrections(row)})()
+
+        self.assertIs(subject._pending_candidate(runtime, "event"), row)
+
+        row["decision_reason"] = ""
+        with self.assertRaisesRegex(RuntimeError, "pending result candidate"):
+            subject._pending_candidate(runtime, "event")
+
+    def test_preserves_a_loaded_gate_eligibility_margin(self) -> None:
+        self.assertGreaterEqual(subject.APPROVAL_HORIZON_MILLIS, 60_000)
+        self.assertGreaterEqual(
+            subject.APPROVAL_ELIGIBILITY_TIMEOUT_SECONDS * 1_000,
+            subject.APPROVAL_HORIZON_MILLIS + 15_000,
+        )
+
+
+if __name__ == "__main__":
+    unittest.main()


## `test(e2e): fence admin revision retry`

diff --git a/e2e/scenario_09_admin_revision_retry.py b/e2e/scenario_09_admin_revision_retry.py
index 597f68f..2b8f703 100644
--- a/e2e/scenario_09_admin_revision_retry.py
+++ b/e2e/scenario_09_admin_revision_retry.py
@@ -8,6 +8,7 @@ from scripts.cold_gate.database import uuid_literal
 
 
 NAME = "admin-revision-retry"
+RETRY_HOLD_SECONDS = 60
 
 
 def run(runtime: E2eRuntime) -> None:
@@ -22,6 +23,8 @@ def run(runtime: E2eRuntime) -> None:
             UPDATE settlement_revision
             SET attempt_count = 12,
                 next_retry_at = NULL,
+                wallet_next_attempt_at = CURRENT_TIMESTAMP
+                    + {RETRY_HOLD_SECONDS} * INTERVAL '1 second',
                 last_error_code = 'WALLET_RETRY_EXHAUSTED',
                 updated_at = CURRENT_TIMESTAMP
             WHERE revision_id = {uuid_literal(blocked.revision_id)}
@@ -73,6 +76,7 @@ def run(runtime: E2eRuntime) -> None:
         )
         if not mutation.payload.get("nextRetryAt"):
             raise RuntimeError("operator retry receipt has no next attempt")
+        _release_retry(runtime, blocked.revision_id)
         applied = wait_applied_decrease(runtime, blocked.bet_id)
         if applied["revision_id"] != blocked.revision_id:
             raise RuntimeError("operator retry changed revision identity")
@@ -95,3 +99,25 @@ def run(runtime: E2eRuntime) -> None:
     finally:
         if settlement_stopped:
             runtime.start_settlement()
+
+
+def _release_retry(runtime: E2eRuntime, revision_id: str) -> None:
+    updated = runtime.database.one(
+        "settlement",
+        f"""
+        UPDATE settlement_revision
+        SET wallet_next_attempt_at = CURRENT_TIMESTAMP,
+            next_retry_at = CURRENT_TIMESTAMP,
+            updated_at = CURRENT_TIMESTAMP
+        WHERE revision_id = {uuid_literal(revision_id)}
+          AND state = 'BLOCKED'
+          AND wallet_status = 'BLOCKED'
+          AND attempt_count = 0
+          AND lease_token IS NULL
+          AND wallet_next_attempt_at > CURRENT_TIMESTAMP
+          AND next_retry_at > CURRENT_TIMESTAMP
+        RETURNING revision_id::text AS revision_id
+        """,
+    )
+    if updated["revision_id"] != revision_id:
+        raise RuntimeError("operator retry release changed identity")
diff --git a/tests/test_admin_revision_scenario.py b/tests/test_admin_revision_scenario.py
new file mode 100644
index 0000000..0f36f34
--- /dev/null
+++ b/tests/test_admin_revision_scenario.py
@@ -0,0 +1,38 @@
+import unittest
+
+from e2e import scenario_09_admin_revision_retry as subject
+
+
+REVISION = "55000000-0000-7000-8000-000000000009"
+
+
+class FakeDatabase:
+    def __init__(self) -> None:
+        self.calls: list[tuple[str, str]] = []
+
+    def one(self, service: str, statement: str) -> dict[str, str]:
+        self.calls.append((service, statement))
+        return {"revision_id": REVISION}
+
+
+class AdminRevisionScenarioTest(unittest.TestCase):
+    def test_holds_the_queued_receipt_outside_the_scanner_window(self) -> None:
+        self.assertGreaterEqual(subject.RETRY_HOLD_SECONDS, 60)
+
+    def test_releases_both_retry_clocks_atomically(self) -> None:
+        database = FakeDatabase()
+        runtime = type("Runtime", (), {"database": database})()
+
+        subject._release_retry(runtime, REVISION)
+
+        self.assertEqual(database.calls[0][0], "settlement")
+        statement = database.calls[0][1]
+        self.assertIn("wallet_next_attempt_at = CURRENT_TIMESTAMP", statement)
+        self.assertIn("next_retry_at = CURRENT_TIMESTAMP", statement)
+        self.assertIn("attempt_count = 0", statement)
+        self.assertIn("wallet_next_attempt_at > CURRENT_TIMESTAMP", statement)
+        self.assertIn("next_retry_at > CURRENT_TIMESTAMP", statement)
+
+
+if __name__ == "__main__":
+    unittest.main()


## `test(e2e): use database candidate eligibility`

diff --git a/e2e/scenario_08_admin_candidates.py b/e2e/scenario_08_admin_candidates.py
index 0a64c6e..a9d5d96 100644
--- a/e2e/scenario_08_admin_candidates.py
+++ b/e2e/scenario_08_admin_candidates.py
@@ -5,6 +5,7 @@ import time
 from e2e.assertions import require_fields, wait_fields
 from e2e.model import ScenarioIds
 from e2e.runtime import E2eRuntime
+from scripts.cold_gate.database import uuid_literal
 from scripts.cold_gate.polling import poll_until
 
 
@@ -21,13 +22,7 @@ def run(runtime: E2eRuntime) -> None:
         "MatchResult", approved_fixture.match_result("WON", approved_at)
     )
     approved_candidate = _pending_candidate(runtime, approved_fixture.event)
-    poll_until(
-        "candidate approval eligibility",
-        lambda: int(time.time() * 1000),
-        lambda now: now >= approved_at + 100,
-        timeout=APPROVAL_ELIGIBILITY_TIMEOUT_SECONDS,
-        interval=0.1,
-    )
+    _wait_until_eligible(runtime, approved_candidate["candidate_id"])
     approval = runtime.settlement_admin.approve(
         approved_candidate["candidate_id"],
         token,
@@ -92,6 +87,23 @@ def _pending_candidate(runtime: E2eRuntime, event_id: str) -> dict[str, str]:
     return candidate
 
 
+def _wait_until_eligible(runtime: E2eRuntime, candidate_id: str) -> None:
+    poll_until(
+        "candidate approval eligibility",
+        lambda: runtime.database.one(
+            "settlement",
+            f"""
+            SELECT (settled_at <= CURRENT_TIMESTAMP)::int::text AS eligible
+            FROM result_candidate
+            WHERE candidate_id = {uuid_literal(candidate_id)}
+            """,
+        ),
+        lambda row: row.get("eligible") == "1",
+        timeout=APPROVAL_ELIGIBILITY_TIMEOUT_SECONDS,
+        interval=0.1,
+    )
+
+
 def _single_candidate(runtime: E2eRuntime, event_id: str) -> dict[str, str] | None:
     rows = runtime.corrections.candidates(event_id)
     if not rows:
diff --git a/tests/test_admin_candidate_scenario.py b/tests/test_admin_candidate_scenario.py
index 4c1b890..55f7039 100644
--- a/tests/test_admin_candidate_scenario.py
+++ b/tests/test_admin_candidate_scenario.py
@@ -1,4 +1,5 @@
 import unittest
+from unittest import mock
 
 from e2e import scenario_08_admin_candidates as subject
 
@@ -11,6 +12,15 @@ class FakeCorrections:
         return [self.row]
 
 
+class FakeDatabase:
+    def __init__(self) -> None:
+        self.calls: list[tuple[str, str]] = []
+
+    def one(self, service: str, statement: str) -> dict[str, str]:
+        self.calls.append((service, statement))
+        return {"eligible": "1"}
+
+
 class AdminCandidateScenarioTest(unittest.TestCase):
     def test_requires_the_locked_future_hold_reason(self) -> None:
         row = {
@@ -34,6 +44,18 @@ class AdminCandidateScenarioTest(unittest.TestCase):
             subject.APPROVAL_HORIZON_MILLIS + 15_000,
         )
 
+    def test_uses_the_settlement_database_eligibility_clock(self) -> None:
+        database = FakeDatabase()
+        runtime = type("Runtime", (), {"database": database})()
+        candidate = "77000000-0000-7000-8000-000000000081"
+
+        with mock.patch.object(subject.time, "time", side_effect=AssertionError):
+            subject._wait_until_eligible(runtime, candidate)
+
+        self.assertEqual(database.calls[0][0], "settlement")
+        self.assertIn("settled_at <= CURRENT_TIMESTAMP", database.calls[0][1])
+        self.assertIn(candidate, database.calls[0][1])
+
 
 if __name__ == "__main__":
     unittest.main()


## `test(e2e): rearm revision retry hold`

diff --git a/e2e/scenario_09_admin_revision_retry.py b/e2e/scenario_09_admin_revision_retry.py
index 2b8f703..db99baa 100644
--- a/e2e/scenario_09_admin_revision_retry.py
+++ b/e2e/scenario_09_admin_revision_retry.py
@@ -8,7 +8,7 @@ from scripts.cold_gate.database import uuid_literal
 
 
 NAME = "admin-revision-retry"
-RETRY_HOLD_SECONDS = 60
+RETRY_HOLD_SECONDS = 120
 
 
 def run(runtime: E2eRuntime) -> None:
@@ -57,6 +57,7 @@ def run(runtime: E2eRuntime) -> None:
             terminal={"status": frozenset({"REJECTED"})},
             timeout=60,
         )
+        _arm_retry_hold(runtime, blocked.revision_id)
         runtime.start_settlement()
         settlement_stopped = False
 
@@ -121,3 +122,24 @@ def _release_retry(runtime: E2eRuntime, revision_id: str) -> None:
     )
     if updated["revision_id"] != revision_id:
         raise RuntimeError("operator retry release changed identity")
+
+
+def _arm_retry_hold(runtime: E2eRuntime, revision_id: str) -> None:
+    updated = runtime.database.one(
+        "settlement",
+        f"""
+        UPDATE settlement_revision
+        SET wallet_next_attempt_at = CURRENT_TIMESTAMP
+                + {RETRY_HOLD_SECONDS} * INTERVAL '1 second',
+            updated_at = CURRENT_TIMESTAMP
+        WHERE revision_id = {uuid_literal(revision_id)}
+          AND state = 'BLOCKED'
+          AND wallet_status = 'BLOCKED'
+          AND attempt_count = 12
+          AND next_retry_at IS NULL
+          AND lease_token IS NULL
+        RETURNING revision_id::text AS revision_id
+        """,
+    )
+    if updated["revision_id"] != revision_id:
+        raise RuntimeError("operator retry hold changed identity")
diff --git a/tests/test_admin_revision_scenario.py b/tests/test_admin_revision_scenario.py
index 0f36f34..6f05397 100644
--- a/tests/test_admin_revision_scenario.py
+++ b/tests/test_admin_revision_scenario.py
@@ -17,7 +17,18 @@ class FakeDatabase:
 
 class AdminRevisionScenarioTest(unittest.TestCase):
     def test_holds_the_queued_receipt_outside_the_scanner_window(self) -> None:
-        self.assertGreaterEqual(subject.RETRY_HOLD_SECONDS, 60)
+        self.assertGreaterEqual(subject.RETRY_HOLD_SECONDS, 120)
+
+    def test_rearms_the_hold_before_settlement_restarts(self) -> None:
+        database = FakeDatabase()
+        runtime = type("Runtime", (), {"database": database})()
+
+        subject._arm_retry_hold(runtime, REVISION)
+
+        statement = database.calls[0][1]
+        self.assertIn("wallet_next_attempt_at = CURRENT_TIMESTAMP", statement)
+        self.assertIn("attempt_count = 12", statement)
+        self.assertIn("next_retry_at IS NULL", statement)
 
     def test_releases_both_retry_clocks_atomically(self) -> None:
         database = FakeDatabase()
