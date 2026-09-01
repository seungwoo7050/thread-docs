## `test(e2e): match single slip responses`

diff --git a/e2e/bet_api.py b/e2e/bet_api.py
index dff447e..e383dc1 100644
--- a/e2e/bet_api.py
+++ b/e2e/bet_api.py
@@ -47,7 +47,8 @@ class BetApi:
             response.header("Location") != "/api/v1/bets/" + bet_id
             or payload.get("userId") != fixture.user
             or payload.get("status") != expected_status
-            or payload.get("slipType") != {"type": "SINGLE"}
+            or payload.get("slipType")
+            != {"type": "SINGLE", "minWins": None, "totalSelections": None}
             or payload.get("stake") != {"amount": 10_000, "currency": "KRW"}
             or payload.get("maxPayout") != {"amount": 20_000, "currency": "KRW"}
             or payload.get("selections") != [expected_selection]


## `test(e2e): await cold placement recovery`

diff --git a/e2e/scenario_01_authenticated_settlement.py b/e2e/scenario_01_authenticated_settlement.py
index 4856597..55f3ed0 100644
--- a/e2e/scenario_01_authenticated_settlement.py
+++ b/e2e/scenario_01_authenticated_settlement.py
@@ -15,11 +15,24 @@ def run(runtime: E2eRuntime) -> None:
     runtime.seed(fixture)
     token = runtime.user_token(fixture)
     placement = runtime.bets.place(fixture, token)
-    require_fields(
-        placement.__dict__,
-        {"http_status": 201, "status": "ACCEPTED"},
-        "authenticated placement",
-    )
+    if placement.status == "PENDING":
+        wait_fields(
+            "authenticated placement recovery",
+            lambda: runtime.base.betting(placement.bet_id),
+            {
+                "status": "ACCEPTED",
+                "placement_phase": "RISK_COMMITTED",
+                "risk_committed": "1",
+                "wallet_confirmed": "1",
+            },
+            terminal={"status": frozenset({"REJECTED", "VOIDED", "SETTLED"})},
+        )
+    else:
+        require_fields(
+            placement.__dict__,
+            {"http_status": 201, "status": "ACCEPTED"},
+            "authenticated placement",
+        )
     wait_fields(
         "Settlement placement projection",
         lambda: runtime.base.settlement(placement.bet_id),
