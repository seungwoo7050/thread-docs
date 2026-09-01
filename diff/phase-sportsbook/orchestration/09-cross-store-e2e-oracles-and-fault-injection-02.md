## `test(e2e): bind one isolated cold runtime`

diff --git a/e2e/runtime.py b/e2e/runtime.py
new file mode 100644
index 0000000..90e3045
--- /dev/null
+++ b/e2e/runtime.py
@@ -0,0 +1,92 @@
+from __future__ import annotations
+
+import time
+
+from e2e.base_oracles import BaseOracles
+from e2e.bet_api import BetApi
+from e2e.correction_oracles import CorrectionOracles
+from e2e.model import ScenarioIds
+from e2e.placement_oracles import PlacementOracles
+from e2e.wallet_api import WalletApi
+from scripts.cold_gate.build import ReleaseArtifacts
+from scripts.cold_gate.chaos import ChaosClient
+from scripts.cold_gate.compose import ComposeProject
+from scripts.cold_gate.container_http import ContainerHttpClient
+from scripts.cold_gate.context import ColdGateContext
+from scripts.cold_gate.database import PostgresClient
+from scripts.cold_gate.fixtures import FixturePublisher
+from scripts.cold_gate.http import HostHttpClient
+from scripts.cold_gate.jwt import JwtSigner
+from scripts.cold_gate.kafka import KafkaAdmin
+from scripts.cold_gate.polling import poll_until
+from scripts.cold_gate.redis import RedisClient
+from scripts.cold_gate.secrets import RuntimeSecrets
+from scripts.cold_gate.stack import ColdStack
+
+
+SETTLEMENT_ASSIGNMENTS = frozenset(
+    (topic, partition)
+    for topic in ("bet.placed.v1", "event.lifecycle", "match.result")
+    for partition in range(3)
+)
+
+
+class E2eRuntime:
+    def __init__(
+        self,
+        context: ColdGateContext,
+        compose: ComposeProject,
+        artifacts: ReleaseArtifacts,
+        secrets: RuntimeSecrets,
+    ) -> None:
+        if compose.context is not context:
+            raise RuntimeError("E2E runtime ownership mismatch")
+        gateway_port = ColdStack(context, compose).loopback_port("gateway", 8080)
+        database = PostgresClient(compose)
+        self.context = context
+        self.compose = compose
+        self.environment = secrets.environment
+        self.signer = JwtSigner(context, secrets.private_key)
+        self.bets = BetApi(HostHttpClient(f"http://127.0.0.1:{gateway_port}"))
+        self.wallet_api = WalletApi(
+            ContainerHttpClient(compose, "wallet"),
+            secrets.environment["WALLET_PLATFORM_API_KEY"],
+        )
+        self.base = BaseOracles(database)
+        self.placements = PlacementOracles(database)
+        self.corrections = CorrectionOracles(database)
+        self.odds = RedisClient(compose, "redis-odds")
+        self.risk = RedisClient(compose, "redis-risk")
+        self.kafka = KafkaAdmin(compose)
+        self.fixtures = FixturePublisher(context, compose, artifacts.fixture_jar)
+        self.chaos = ChaosClient(compose)
+
+    def user_token(self, fixture: ScenarioIds) -> str:
+        return self.signer.user(fixture.user, int(time.time()))
+
+    def admin_token(self) -> str:
+        return self.signer.admin(int(time.time()))
+
+    def seed(self, fixture: ScenarioIds) -> None:
+        self.wallet_api.open_and_fund(fixture)
+        if self.odds.scalar("SET", f"market:{fixture.event}:{fixture.market}", "OPEN", "EX", "3600") != "OK":
+            raise RuntimeError("market fixture was not seeded")
+        if self.odds.scalar(
+            "SET", f"odds:{fixture.event}:{fixture.market}:{fixture.selection}", "2.0000", "EX", "3600"
+        ) != "OK":
+            raise RuntimeError("odds fixture was not seeded")
+
+    def wait_for_settlement_assignments(self) -> None:
+        def complete(actual: frozenset[tuple[str, int]]) -> bool:
+            unexpected = actual - SETTLEMENT_ASSIGNMENTS
+            if unexpected:
+                raise RuntimeError(f"Settlement owns unexpected Kafka assignments: {unexpected}")
+            return actual == SETTLEMENT_ASSIGNMENTS
+
+        poll_until(
+            "Settlement Kafka assignments",
+            lambda: self.kafka.assignments("settlement-service"),
+            complete,
+            timeout=90,
+            interval=1,
+        )


## `test(e2e): enforce fail-fast projection barriers`

diff --git a/e2e/assertions.py b/e2e/assertions.py
new file mode 100644
index 0000000..1f3e3a8
--- /dev/null
+++ b/e2e/assertions.py
@@ -0,0 +1,53 @@
+from __future__ import annotations
+
+import uuid
+from collections.abc import Callable, Mapping
+
+from scripts.cold_gate.polling import poll_until
+
+
+def require_fields(actual: Mapping[str, object], expected: Mapping[str, object], name: str) -> None:
+    drift = {
+        key: {"expected": value, "actual": actual.get(key)}
+        for key, value in expected.items()
+        if actual.get(key) != value
+    }
+    if drift:
+        raise RuntimeError(f"{name} drifted: {drift}")
+
+
+def wait_fields(
+    description: str,
+    probe: Callable[[], Mapping[str, object] | None],
+    expected: Mapping[str, object],
+    *,
+    timeout: float = 60,
+    terminal: Mapping[str, frozenset[object]] | None = None,
+) -> Mapping[str, object]:
+    def accepted(actual: Mapping[str, object] | None) -> bool:
+        if actual is None:
+            return False
+        for field, forbidden in (terminal or {}).items():
+            if actual.get(field) in forbidden and actual.get(field) != expected.get(field):
+                raise RuntimeError(
+                    f"{description} entered terminal {field}={actual.get(field)!r}"
+                )
+        return all(actual.get(key) == value for key, value in expected.items())
+
+    return poll_until(
+        description,
+        probe,
+        accepted,
+        timeout=timeout,
+        interval=0.25,
+    )
+
+
+def require_uuidv7(value: str, name: str) -> str:
+    try:
+        parsed = uuid.UUID(value)
+    except ValueError as error:
+        raise RuntimeError(f"{name} is not a UUID") from error
+    if str(parsed) != value or parsed.version != 7:
+        raise RuntimeError(f"{name} is not a canonical UUIDv7")
+    return value


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


## `test(e2e): wait for exact consumer progress`

diff --git a/e2e/kafka_barrier.py b/e2e/kafka_barrier.py
new file mode 100644
index 0000000..5f7980a
--- /dev/null
+++ b/e2e/kafka_barrier.py
@@ -0,0 +1,21 @@
+from __future__ import annotations
+
+from e2e.runtime import E2eRuntime
+from scripts.cold_gate.fixture_receipt import FixtureReceipt
+from scripts.cold_gate.polling import poll_until
+
+
+def wait_consumed(
+    runtime: E2eRuntime,
+    group: str,
+    receipt: FixtureReceipt,
+    *,
+    timeout: float = 60,
+) -> None:
+    poll_until(
+        f"{group} consumption of {receipt.topic}",
+        lambda: runtime.kafka.committed_offset(group, receipt.topic, receipt.partition),
+        lambda offset: offset > receipt.offset,
+        timeout=timeout,
+        interval=0.5,
+    )


## `build(chaos): proxy synchronous dependency paths`

diff --git a/chaos/toxiproxy.json b/chaos/toxiproxy.json
new file mode 100644
index 0000000..ff85b59
--- /dev/null
+++ b/chaos/toxiproxy.json
@@ -0,0 +1,20 @@
+[
+  {
+    "name": "betting_to_risk",
+    "listen": "0.0.0.0:18083",
+    "upstream": "risk:8083",
+    "enabled": true
+  },
+  {
+    "name": "betting_to_wallet",
+    "listen": "0.0.0.0:18081",
+    "upstream": "wallet:8081",
+    "enabled": true
+  },
+  {
+    "name": "settlement_to_wallet",
+    "listen": "0.0.0.0:28081",
+    "upstream": "wallet:8081",
+    "enabled": true
+  }
+]
diff --git a/compose.toxiproxy.yaml b/compose.toxiproxy.yaml
new file mode 100644
index 0000000..f067c9f
--- /dev/null
+++ b/compose.toxiproxy.yaml
@@ -0,0 +1,32 @@
+services:
+  toxiproxy:
+    image: ghcr.io/shopify/toxiproxy:2.9.0
+    command: ["-host=0.0.0.0", "-config=/config/toxiproxy.json"]
+    volumes:
+      - ./chaos/toxiproxy.json:/config/toxiproxy.json:ro
+    ports:
+      - target: 8474
+        published: ${TOXIPROXY_HOST_PORT:-8474}
+        host_ip: 127.0.0.1
+        protocol: tcp
+    healthcheck:
+      test: ["CMD", "/toxiproxy", "-version"]
+      interval: 3s
+      timeout: 3s
+      retries: 30
+    networks: [backend]
+
+  betting:
+    environment:
+      RISK_BASE_URL: http://toxiproxy:18083
+      WALLET_BASE_URL: http://toxiproxy:18081
+    depends_on:
+      toxiproxy:
+        condition: service_healthy
+
+  settlement:
+    environment:
+      SETTLEMENT_WALLET_BASE_URL: http://toxiproxy:28081
+    depends_on:
+      toxiproxy:
+        condition: service_healthy


## `test(chaos): verify canonical URI overrides`

diff --git a/tests/test_toxiproxy_overlay.py b/tests/test_toxiproxy_overlay.py
new file mode 100644
index 0000000..dcc7c66
--- /dev/null
+++ b/tests/test_toxiproxy_overlay.py
@@ -0,0 +1,76 @@
+import json
+import pathlib
+import subprocess
+
+from tests.compose_contract_fixture import ComposeContractFixture
+
+
+ROOT = pathlib.Path(__file__).resolve().parents[1]
+OVERLAY = ROOT / "compose.toxiproxy.yaml"
+
+
+class ToxiproxyOverlayTest(ComposeContractFixture):
+    def test_overrides_only_three_canonical_http_dependencies(self) -> None:
+        self.environment["TOXIPROXY_HOST_PORT"] = "18474"
+        result = subprocess.run(
+            [
+                "docker",
+                "compose",
+                "--project-name",
+                self.project,
+                "--file",
+                str(ROOT / "compose.yaml"),
+                "--file",
+                str(OVERLAY),
+                "config",
+                "--format",
+                "json",
+            ],
+            cwd=ROOT,
+            env=self.environment,
+            text=True,
+            capture_output=True,
+            check=False,
+        )
+        self.assertEqual(result.returncode, 0, result.stderr)
+        services = json.loads(result.stdout)["services"]
+
+        self.assertEqual(
+            services["betting"]["environment"]["RISK_BASE_URL"],
+            "http://toxiproxy:18083",
+        )
+        self.assertEqual(
+            services["betting"]["environment"]["WALLET_BASE_URL"],
+            "http://toxiproxy:18081",
+        )
+        settlement = services["settlement"]["environment"]
+        self.assertEqual(
+            settlement["SETTLEMENT_WALLET_BASE_URL"], "http://toxiproxy:28081"
+        )
+        self.assertNotIn("WALLET_BASE_URL", settlement)
+        self.assertEqual(
+            services["toxiproxy"]["ports"][0]["host_ip"], "127.0.0.1"
+        )
+        self.assertEqual(services["toxiproxy"]["ports"][0]["published"], "18474")
+        self.assertEqual(set(services["toxiproxy"]["networks"]), {"backend"})
+        self.assertEqual(
+            services["settlement"]["depends_on"]["toxiproxy"]["condition"],
+            "service_healthy",
+        )
+
+        proxies = json.loads((ROOT / "chaos/toxiproxy.json").read_text())
+        self.assertEqual(
+            {(proxy["name"], proxy["listen"], proxy["upstream"]) for proxy in proxies},
+            {
+                ("betting_to_risk", "0.0.0.0:18083", "risk:8083"),
+                ("betting_to_wallet", "0.0.0.0:18081", "wallet:8081"),
+                ("settlement_to_wallet", "0.0.0.0:28081", "wallet:8081"),
+            },
+        )
+        self.assertNotIn("kafka", json.dumps(proxies).lower())
+
+
+if __name__ == "__main__":
+    import unittest
+
+    unittest.main()


## `build(chaos): allocate proxy port dynamically`

diff --git a/compose.toxiproxy.yaml b/compose.toxiproxy.yaml
index f067c9f..29f67aa 100644
--- a/compose.toxiproxy.yaml
+++ b/compose.toxiproxy.yaml
@@ -6,7 +6,6 @@ services:
       - ./chaos/toxiproxy.json:/config/toxiproxy.json:ro
     ports:
       - target: 8474
-        published: ${TOXIPROXY_HOST_PORT:-8474}
         host_ip: 127.0.0.1
         protocol: tcp
     healthcheck:


## `test(chaos): verify dynamic proxy exposure`

diff --git a/tests/test_toxiproxy_overlay.py b/tests/test_toxiproxy_overlay.py
index dcc7c66..a6a6e6a 100644
--- a/tests/test_toxiproxy_overlay.py
+++ b/tests/test_toxiproxy_overlay.py
@@ -11,7 +11,6 @@ OVERLAY = ROOT / "compose.toxiproxy.yaml"
 
 class ToxiproxyOverlayTest(ComposeContractFixture):
     def test_overrides_only_three_canonical_http_dependencies(self) -> None:
-        self.environment["TOXIPROXY_HOST_PORT"] = "18474"
         result = subprocess.run(
             [
                 "docker",
@@ -48,11 +47,15 @@ class ToxiproxyOverlayTest(ComposeContractFixture):
             settlement["SETTLEMENT_WALLET_BASE_URL"], "http://toxiproxy:28081"
         )
         self.assertNotIn("WALLET_BASE_URL", settlement)
+        admin_port = services["toxiproxy"]["ports"][0]
+        self.assertEqual(admin_port["host_ip"], "127.0.0.1")
+        self.assertEqual(admin_port["target"], 8474)
+        self.assertNotIn("published", admin_port)
+        self.assertEqual(set(services["toxiproxy"]["networks"]), {"backend"})
         self.assertEqual(
-            services["toxiproxy"]["ports"][0]["host_ip"], "127.0.0.1"
+            services["betting"]["depends_on"]["toxiproxy"]["condition"],
+            "service_healthy",
         )
-        self.assertEqual(services["toxiproxy"]["ports"][0]["published"], "18474")
-        self.assertEqual(set(services["toxiproxy"]["networks"]), {"backend"})
         self.assertEqual(
             services["settlement"]["depends_on"]["toxiproxy"]["condition"],
             "service_healthy",
@@ -67,6 +70,7 @@ class ToxiproxyOverlayTest(ComposeContractFixture):
                 ("settlement_to_wallet", "0.0.0.0:28081", "wallet:8081"),
             },
         )
+        self.assertTrue(all(proxy["enabled"] for proxy in proxies))
         self.assertNotIn("kafka", json.dumps(proxies).lower())
 
 


## `build(gate): control scoped dependency faults`

diff --git a/scripts/cold_gate/chaos.py b/scripts/cold_gate/chaos.py
new file mode 100644
index 0000000..da4325b
--- /dev/null
+++ b/scripts/cold_gate/chaos.py
@@ -0,0 +1,69 @@
+from __future__ import annotations
+
+import re
+import urllib.request
+from collections.abc import Callable
+
+from scripts.cold_gate.compose import ComposeProject
+from scripts.cold_gate.http import HostHttpClient, HttpResponse
+
+
+PROXIES = frozenset({"betting_to_risk", "betting_to_wallet", "settlement_to_wallet"})
+LOST_RESPONSE_TOXIC = "e2e_wallet_response_timeout"
+
+
+class ChaosClient:
+    def __init__(
+        self,
+        compose: ComposeProject,
+        opener: Callable[..., object] = urllib.request.urlopen,
+    ) -> None:
+        result = compose.run("port", "toxiproxy", "8474", capture_output=True)
+        endpoint = result.stdout.strip()
+        match = re.fullmatch(r"127\.0\.0\.1:([1-9][0-9]{0,4})", endpoint)
+        if match is None or int(match.group(1)) > 65535:
+            raise RuntimeError("Toxiproxy did not publish one loopback port")
+        self.http = HostHttpClient(f"http://{endpoint}", opener)
+
+    def proxy(self, name: str) -> dict[str, object]:
+        return self._json("GET", self._proxy_path(name), expected=(200,))
+
+    def set_enabled(self, name: str, enabled: bool) -> dict[str, object]:
+        return self._json(
+            "POST", self._proxy_path(name), body={"enabled": enabled}, expected=(200,)
+        )
+
+    def add_wallet_response_timeout(self) -> dict[str, object]:
+        return self._json(
+            "POST",
+            self._proxy_path("betting_to_wallet") + "/toxics",
+            body={
+                "name": LOST_RESPONSE_TOXIC,
+                "type": "timeout",
+                "stream": "downstream",
+                "attributes": {"timeout": 0},
+            },
+            expected=(200,),
+        )
+
+    def remove_wallet_response_timeout(self) -> None:
+        response = self.http.request(
+            "DELETE",
+            self._proxy_path("betting_to_wallet") + "/toxics/" + LOST_RESPONSE_TOXIC,
+        )
+        response.require_status(204)
+
+    def _json(
+        self, method: str, path: str, *, body=None, expected: tuple[int, ...]
+    ) -> dict[str, object]:
+        response: HttpResponse = self.http.request(method, path, body=body)
+        payload = response.require_status(*expected).json()
+        if not isinstance(payload, dict):
+            raise RuntimeError("Toxiproxy response is not an object")
+        return payload
+
+    @staticmethod
+    def _proxy_path(name: str) -> str:
+        if name not in PROXIES:
+            raise ValueError("proxy is outside the cold gate inventory")
+        return "/proxies/" + name


## `test(gate): restore every injected dependency fault`

diff --git a/tests/test_cold_gate_chaos.py b/tests/test_cold_gate_chaos.py
new file mode 100644
index 0000000..f70760e
--- /dev/null
+++ b/tests/test_cold_gate_chaos.py
@@ -0,0 +1,73 @@
+import json
+import subprocess
+import unittest
+from email.message import Message
+
+from scripts.cold_gate.chaos import ChaosClient, LOST_RESPONSE_TOXIC
+
+
+class FakeCompose:
+    def __init__(self, endpoint: str = "127.0.0.1:54321") -> None:
+        self.endpoint = endpoint
+        self.calls = []
+
+    def run(self, *arguments, **options):
+        self.calls.append((arguments, options))
+        return subprocess.CompletedProcess(arguments, 0, stdout=self.endpoint + "\n")
+
+
+class FakeResponse:
+    def __init__(self, status: int, body: bytes) -> None:
+        self.status = status
+        self.body = body
+        self.headers = Message()
+
+    def __enter__(self):
+        return self
+
+    def __exit__(self, *_args):
+        return False
+
+    def read(self) -> bytes:
+        return self.body
+
+
+class ColdGateChaosTest(unittest.TestCase):
+    def test_controls_only_the_fixed_fault_boundaries(self) -> None:
+        requests = []
+
+        def opener(request, **_options):
+            requests.append(request)
+            status = 204 if request.method == "DELETE" else 200
+            return FakeResponse(status, b"" if status == 204 else b'{"enabled":true}')
+
+        compose = FakeCompose()
+        client = ChaosClient(compose, opener)
+        client.set_enabled("betting_to_risk", False)
+        client.add_wallet_response_timeout()
+        client.remove_wallet_response_timeout()
+
+        self.assertEqual(compose.calls[0][0], ("port", "toxiproxy", "8474"))
+        self.assertEqual(json.loads(requests[0].data), {"enabled": False})
+        self.assertEqual(
+            json.loads(requests[1].data),
+            {
+                "name": LOST_RESPONSE_TOXIC,
+                "type": "timeout",
+                "stream": "downstream",
+                "attributes": {"timeout": 0},
+            },
+        )
+        self.assertTrue(requests[2].full_url.endswith("/toxics/" + LOST_RESPONSE_TOXIC))
+        self.assertEqual(requests[2].method, "DELETE")
+
+    def test_rejects_unscoped_proxy_names_and_non_loopback_publication(self) -> None:
+        with self.assertRaisesRegex(RuntimeError, "loopback"):
+            ChaosClient(FakeCompose("0.0.0.0:8474"))
+        client = ChaosClient(FakeCompose(), lambda *_args, **_options: None)
+        with self.assertRaisesRegex(ValueError, "outside"):
+            client.proxy("unrelated")
+
+
+if __name__ == "__main__":
+    unittest.main()


