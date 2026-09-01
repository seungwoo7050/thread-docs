## `fix(chaos): control toxiproxy inside the stack`

diff --git a/scripts/cold_gate/chaos.py b/scripts/cold_gate/chaos.py
index f77dbf0..cbeaa5e 100644
--- a/scripts/cold_gate/chaos.py
+++ b/scripts/cold_gate/chaos.py
@@ -1,9 +1,7 @@
 from __future__ import annotations
 
-import urllib.request
-from collections.abc import Callable
-
-from scripts.cold_gate.http import HostHttpClient, HttpResponse
+from scripts.cold_gate.container_http import ContainerHttpClient
+from scripts.cold_gate.http import HttpResponse
 
 
 PROXIES = frozenset({"betting_to_risk", "betting_to_wallet", "settlement_to_wallet"})
@@ -11,14 +9,10 @@ LOST_RESPONSE_TOXIC = "e2e_wallet_response_timeout"
 
 
 class ChaosClient:
-    def __init__(
-        self,
-        port: int,
-        opener: Callable[..., object] = urllib.request.urlopen,
-    ) -> None:
-        if not 0 < port <= 65535:
-            raise RuntimeError("Toxiproxy did not publish one loopback port")
-        self.http = HostHttpClient(f"http://127.0.0.1:{port}", opener)
+    def __init__(self, http: ContainerHttpClient) -> None:
+        if http.service != "toxiproxy":
+            raise RuntimeError("chaos control must use the owned Toxiproxy container")
+        self.http = http
 
     def proxy(self, name: str) -> dict[str, object]:
         return self._json("GET", self._proxy_path(name), expected=(200,))
diff --git a/scripts/cold_gate/container_http.py b/scripts/cold_gate/container_http.py
index 9f663b9..bb87a54 100644
--- a/scripts/cold_gate/container_http.py
+++ b/scripts/cold_gate/container_http.py
@@ -17,6 +17,7 @@ SERVICE_PORTS = {
     "odds": 8085,
     "gateway": 8080,
     "admin": 8090,
+    "toxiproxy": 8474,
 }
 HEADER_NAME = re.compile(r"^[A-Za-z0-9-]+$")
 
@@ -24,7 +25,7 @@ HEADER_NAME = re.compile(r"^[A-Za-z0-9-]+$")
 class ContainerHttpClient:
     def __init__(self, compose: ComposeProject, service: str) -> None:
         if service not in SERVICE_PORTS:
-            raise ValueError("container HTTP service is not an application")
+            raise ValueError("container HTTP service is not supported")
         self.compose = compose
         self.service = service
 


## `test(http): route chaos control over the stack network`

diff --git a/e2e/runtime.py b/e2e/runtime.py
index fffef55..85d66b6 100644
--- a/e2e/runtime.py
+++ b/e2e/runtime.py
@@ -68,7 +68,9 @@ class E2eRuntime:
         self.kafka = KafkaAdmin(compose)
         self.fixtures = FixturePublisher(context, compose, artifacts.fixture_jar)
         self.probe = KafkaProbe(context, compose, artifacts.fixture_jar)
-        self.chaos = ChaosClient(ContainerHttpClient(compose, "toxiproxy"))
+        self.chaos = ChaosClient(
+            ContainerHttpClient(compose, "toxiproxy", executor="gateway")
+        )
 
     def user_token(self, fixture: ScenarioIds) -> str:
         return self.signer.user(fixture.user, int(time.time()))
diff --git a/tests/test_container_http.py b/tests/test_container_http.py
index 01fb59d..97aed38 100644
--- a/tests/test_container_http.py
+++ b/tests/test_container_http.py
@@ -53,14 +53,19 @@ class ContainerHttpTest(unittest.TestCase):
                 with self.assertRaises(ValueError):
                     client.request(**options)
 
-    def test_reaches_the_toxiproxy_control_api_inside_its_container(self) -> None:
+    def test_reaches_toxiproxy_through_an_application_health_tool(self) -> None:
         compose = FakeCompose()
 
-        ContainerHttpClient(compose, "toxiproxy").request("GET", "/version")
+        ContainerHttpClient(
+            compose, "toxiproxy", executor="gateway"
+        ).request("GET", "/version")
 
         arguments, _options = compose.calls[0]
-        self.assertEqual(arguments[:4], ("exec", "-T", "toxiproxy", "curl"))
-        self.assertEqual(arguments[-1], "http://localhost:8474/version")
+        self.assertEqual(arguments[:4], ("exec", "-T", "gateway", "curl"))
+        self.assertEqual(arguments[-1], "http://toxiproxy:8474/version")
+
+        with self.assertRaisesRegex(ValueError, "no health tool"):
+            ContainerHttpClient(compose, "toxiproxy")
 
     def test_does_not_relay_transport_output_in_errors(self) -> None:
         client = ContainerHttpClient(FakeCompose(failure=True), "admin")


## `test(e2e): run the fixed scenario inventory`

diff --git a/e2e/scenarios.py b/e2e/scenarios.py
new file mode 100644
index 0000000..5b56aba
--- /dev/null
+++ b/e2e/scenarios.py
@@ -0,0 +1,54 @@
+from __future__ import annotations
+
+from collections.abc import Callable
+
+from e2e import (
+    scenario_01_authenticated_settlement,
+    scenario_02_risk_recovery,
+    scenario_03_wallet_lost_response,
+    scenario_04_lifecycle_refund,
+    scenario_05_result_first,
+    scenario_06_payout_increase,
+    scenario_07_blocked_recovery,
+    scenario_08_admin_candidates,
+    scenario_09_admin_revision_retry,
+    scenario_10_revision_ordering,
+    scenario_11_replay_invariance,
+    scenario_12_partition_dlt,
+    scenario_13_admin_correlation,
+)
+from e2e.runtime import E2eRuntime
+
+
+SCENARIOS = (
+    scenario_01_authenticated_settlement,
+    scenario_02_risk_recovery,
+    scenario_03_wallet_lost_response,
+    scenario_04_lifecycle_refund,
+    scenario_05_result_first,
+    scenario_06_payout_increase,
+    scenario_07_blocked_recovery,
+    scenario_08_admin_candidates,
+    scenario_09_admin_revision_retry,
+    scenario_10_revision_ordering,
+    scenario_11_replay_invariance,
+    scenario_12_partition_dlt,
+    scenario_13_admin_correlation,
+)
+
+
+def run_all(
+    runtime: E2eRuntime,
+    completed: Callable[[str], None] | None = None,
+) -> tuple[str, ...]:
+    names = tuple(module.NAME for module in SCENARIOS)
+    if len(names) != 13 or len(set(names)) != 13:
+        raise RuntimeError("E2E scenario inventory is invalid")
+    runtime.wait_for_settlement_assignments()
+    passed = []
+    for module in SCENARIOS:
+        module.run(runtime)
+        passed.append(module.NAME)
+        if completed is not None:
+            completed(module.NAME)
+    return tuple(passed)
