# 관리자 감사와 다운스트림 상관관계

## `build(admin): wire Wave 3 control plane`

diff --git a/compose.yaml b/compose.yaml
index 1fed522..bd24633 100644
--- a/compose.yaml
+++ b/compose.yaml
@@ -344,6 +344,55 @@ services:
     networks: [backend]
     restart: unless-stopped
 
+  admin:
+    build:
+      context: ./docker
+      dockerfile: Dockerfile.jvm
+      args:
+        JAR: admin.jar
+    environment:
+      ADMIN_HTTP_PORT: "8090"
+      ADMIN_DB_URL: jdbc:postgresql://postgres:5432/admin
+      ADMIN_DB_USER: sportsbook
+      ADMIN_DB_PASSWORD: ${POSTGRES_PASSWORD:-}
+      ADMIN_KAFKA_BOOTSTRAP: kafka:9092
+      ADMIN_WALLET_BASE_URL: http://wallet:8081
+      ADMIN_RISK_BASE_URL: http://risk:8083
+      ADMIN_ODDS_FEED_BASE_URL: http://odds:8085
+      ADMIN_SETTLEMENT_BASE_URL: http://settlement:8084
+      ADMIN_WALLET_API_KEY: ${ADMIN_WALLET_API_KEY:-}
+      ADMIN_RISK_API_KEY: ${ADMIN_RISK_API_KEY:-}
+      ADMIN_ODDS_FEED_API_KEY: ${ADMIN_ODDS_FEED_API_KEY:-}
+      ADMIN_SETTLEMENT_API_KEY: ${ADMIN_SETTLEMENT_API_KEY:-}
+      ADMIN_JWT_PUBLIC_KEY: ${ADMIN_JWT_PUBLIC_KEY:-}
+      ADMIN_JWT_ISSUER: ${ADMIN_JWT_ISSUER:-}
+      ADMIN_IP_ALLOWLIST: 127.0.0.1/32,::1/128
+      ADMIN_TRUSTED_PROXY_CIDRS: ""
+    depends_on:
+      postgres:
+        condition: service_healthy
+      kafka:
+        condition: service_healthy
+      topic-init:
+        condition: service_completed_successfully
+      wallet:
+        condition: service_healthy
+      risk:
+        condition: service_healthy
+      odds:
+        condition: service_healthy
+      settlement:
+        condition: service_healthy
+    healthcheck:
+      test: ["CMD-SHELL", "curl --fail --silent --show-error http://localhost:8090/actuator/health/readiness >/dev/null"]
+      interval: 3s
+      timeout: 3s
+      retries: 60
+      start_period: 30s
+    expose: ["8090"]
+    networks: [backend]
+    restart: unless-stopped
+
 volumes:
   postgres-data:
   kafka-data:


## `test(admin): verify auth clients and readiness`

diff --git a/tests/test_admin_wiring.py b/tests/test_admin_wiring.py
new file mode 100644
index 0000000..e9297ec
--- /dev/null
+++ b/tests/test_admin_wiring.py
@@ -0,0 +1,63 @@
+from tests.compose_contract_fixture import ComposeContractFixture, PUBLIC_KEY
+from tests.secret_fixture import VALID
+
+
+class AdminWiringTest(ComposeContractFixture):
+    def test_wires_four_isolated_clients_authentication_and_readiness(self) -> None:
+        admin = self.service("admin")
+        environment = admin["environment"]
+
+        self.assert_runtime_build(admin, "admin.jar")
+        self.assertEqual(environment["ADMIN_DB_URL"], "jdbc:postgresql://postgres:5432/admin")
+        self.assertEqual(environment["ADMIN_KAFKA_BOOTSTRAP"], "kafka:9092")
+        self.assertEqual(environment["ADMIN_JWT_PUBLIC_KEY"], PUBLIC_KEY)
+        self.assertEqual(environment["ADMIN_IP_ALLOWLIST"], "127.0.0.1/32,::1/128")
+        self.assertEqual(environment["ADMIN_TRUSTED_PROXY_CIDRS"], "")
+        self.assertEqual(
+            {
+                name: value
+                for name, value in environment.items()
+                if name.endswith("BASE_URL")
+            },
+            {
+                "ADMIN_WALLET_BASE_URL": "http://wallet:8081",
+                "ADMIN_RISK_BASE_URL": "http://risk:8083",
+                "ADMIN_ODDS_FEED_BASE_URL": "http://odds:8085",
+                "ADMIN_SETTLEMENT_BASE_URL": "http://settlement:8084",
+            },
+        )
+        self.assertEqual(
+            {
+                name: value
+                for name, value in environment.items()
+                if name.endswith("API_KEY")
+            },
+            {
+                "ADMIN_WALLET_API_KEY": VALID["ADMIN_WALLET_API_KEY"],
+                "ADMIN_RISK_API_KEY": VALID["ADMIN_RISK_API_KEY"],
+                "ADMIN_ODDS_FEED_API_KEY": VALID["ADMIN_ODDS_FEED_API_KEY"],
+                "ADMIN_SETTLEMENT_API_KEY": VALID["ADMIN_SETTLEMENT_API_KEY"],
+            },
+        )
+        self.assert_dependency_conditions(
+            admin,
+            {
+                "postgres": "service_healthy",
+                "kafka": "service_healthy",
+                "topic-init": "service_completed_successfully",
+                "wallet": "service_healthy",
+                "risk": "service_healthy",
+                "odds": "service_healthy",
+                "settlement": "service_healthy",
+            },
+        )
+        self.assertIn(
+            "/actuator/health/readiness", admin["healthcheck"]["test"][1]
+        )
+        self.assertNotIn("/actuator/health >/dev/null", admin["healthcheck"]["test"][1])
+
+
+if __name__ == "__main__":
+    import unittest
+
+    unittest.main()


## `test(admin): verify locked Wave 3 contract`

diff --git a/tests/test_admin_handoff.py b/tests/test_admin_handoff.py
new file mode 100644
index 0000000..34381f3
--- /dev/null
+++ b/tests/test_admin_handoff.py
@@ -0,0 +1,42 @@
+import pathlib
+import subprocess
+import unittest
+
+
+ROOT = pathlib.Path(__file__).resolve().parents[1]
+ADMIN_SHA = "2fb55910475b31084e6489bf01c34cc970c96874"
+
+
+class AdminHandoffTest(unittest.TestCase):
+    def test_locks_and_consumes_the_final_wave_three_contract(self) -> None:
+        admin_row = next(
+            line
+            for line in (ROOT / "services.lock").read_text().splitlines()
+            if line.startswith("admin|")
+        )
+        self.assertEqual(
+            admin_row,
+            f"admin|admin-api|{ADMIN_SHA}|admin-api-1.0.0.jar",
+        )
+        readme = subprocess.check_output(
+            ["git", "show", f"{ADMIN_SHA}:README.md"], cwd=ROOT, text=True
+        )
+        for contract in (
+            "POST /admin/v1/wallet/{userId}/refund",
+            "PATCH /admin/v1/risk/users/{userId}/limits",
+            "POST /admin/v1/events/{eventId}/markets/{marketId}/suspend",
+            "POST /admin/v1/settlements/result-candidates/{candidateId}/approve",
+            "POST /admin/v1/settlements/revisions/{revisionId}/retry",
+            "GET /admin/v1/audit-logs/{actionId}",
+            "ADMIN_WALLET_API_KEY",
+            "ADMIN_RISK_API_KEY",
+            "ADMIN_ODDS_FEED_API_KEY",
+            "ADMIN_SETTLEMENT_API_KEY",
+        ):
+            with self.subTest(contract=contract):
+                self.assertIn(contract, readme)
+        self.assertIn("There are no settlement void or replay endpoints.", readme)
+
+
+if __name__ == "__main__":
+    unittest.main()


## `test(e2e): call audited market controls`

diff --git a/e2e/market_admin_api.py b/e2e/market_admin_api.py
new file mode 100644
index 0000000..76edde5
--- /dev/null
+++ b/e2e/market_admin_api.py
@@ -0,0 +1,49 @@
+from __future__ import annotations
+
+from e2e.assertions import require_uuidv7
+from e2e.model import ScenarioIds
+from scripts.cold_gate.container_http import ContainerHttpClient
+
+
+class MarketAdminApi:
+    def __init__(self, client: ContainerHttpClient) -> None:
+        if client.service != "admin":
+            raise ValueError("market operator calls must originate in Admin")
+        self.client = client
+
+    def suspend(
+        self,
+        fixture: ScenarioIds,
+        token: str,
+        idempotency_key: str,
+        traceparent: str,
+        reason: str,
+    ) -> str:
+        if not idempotency_key.startswith("e2e-") or not reason:
+            raise ValueError("market operator fixture is invalid")
+        response = self.client.request(
+            "POST",
+            f"/admin/v1/events/{fixture.event}/markets/{fixture.market}/suspend",
+            headers={
+                "Authorization": "Bearer " + token,
+                "Idempotency-Key": idempotency_key,
+                "traceparent": traceparent,
+            },
+            body={"reason": reason},
+        ).require_status(202)
+        if response.body:
+            raise RuntimeError("market operator response must be empty")
+        return require_uuidv7(
+            response.header("X-Admin-Action-Id"), "market operator action ID"
+        )
+
+    def audit(self, action_id: str, token: str) -> dict[str, object]:
+        response = self.client.request(
+            "GET",
+            "/admin/v1/audit-logs/" + action_id,
+            headers={"Authorization": "Bearer " + token},
+        ).require_status(200)
+        payload = response.json()
+        if not isinstance(payload, dict) or payload.get("actionId") != action_id:
+            raise RuntimeError("Admin audit lookup returned the wrong action")
+        return payload


## `test(e2e): bind audited market controls`

diff --git a/e2e/runtime.py b/e2e/runtime.py
index 89dda30..4a6a35b 100644
--- a/e2e/runtime.py
+++ b/e2e/runtime.py
@@ -5,6 +5,7 @@ import time
 from e2e.base_oracles import BaseOracles
 from e2e.bet_api import BetApi
 from e2e.correction_oracles import CorrectionOracles
+from e2e.market_admin_api import MarketAdminApi
 from e2e.model import ScenarioIds
 from e2e.placement_oracles import PlacementOracles
 from e2e.replay_oracle import ReplayOracle
@@ -49,6 +50,7 @@ class E2eRuntime:
         self.context = context
         self.compose = compose
         self.environment = secrets.environment
+        self.artifacts = artifacts
         self.database = database
         self.signer = JwtSigner(context, secrets.private_key)
         self.bets = BetApi(HostHttpClient(f"http://127.0.0.1:{gateway_port}"))
@@ -57,6 +59,7 @@ class E2eRuntime:
             secrets.environment["WALLET_PLATFORM_API_KEY"],
         )
         self.settlement_admin = SettlementAdminApi(ContainerHttpClient(compose, "admin"))
+        self.market_admin = MarketAdminApi(ContainerHttpClient(compose, "admin"))
         self.settlement_http = ContainerHttpClient(compose, "settlement")
         self.betting_http = ContainerHttpClient(compose, "betting")
         self.base = BaseOracles(database)


## `test(e2e): correlate Admin and Odds evidence`

diff --git a/e2e/admin_correlation.py b/e2e/admin_correlation.py
new file mode 100644
index 0000000..be76636
--- /dev/null
+++ b/e2e/admin_correlation.py
@@ -0,0 +1,68 @@
+from __future__ import annotations
+
+from e2e.model import ScenarioIds
+from e2e.runtime import E2eRuntime
+from scripts.cold_gate.kafka_record import KafkaRecord
+from scripts.cold_gate.polling import poll_until
+
+
+def admin_topic_offsets(runtime: E2eRuntime) -> tuple[int, int, int]:
+    return tuple(runtime.kafka.end_offset("admin.action", partition) for partition in range(3))
+
+
+def wait_admin_record(
+    runtime: E2eRuntime,
+    before: tuple[int, int, int],
+) -> KafkaRecord:
+    def appended(after: tuple[int, int, int]) -> bool:
+        deltas = tuple(current - prior for current, prior in zip(after, before, strict=True))
+        if any(delta < 0 for delta in deltas) or sum(deltas) > 1:
+            raise RuntimeError("Admin action topic changed by more than one record")
+        return sorted(deltas) == [0, 0, 1]
+
+    after = poll_until(
+        "Admin action publication",
+        lambda: admin_topic_offsets(runtime),
+        appended,
+        timeout=60,
+        interval=0.5,
+    )
+    partition = next(
+        index for index, (current, prior) in enumerate(zip(after, before, strict=True))
+        if current == prior + 1
+    )
+    schema = (
+        runtime.artifacts.sources
+        / "admin/src/main/avro/com/sportsbook/admin/event/AdminActionRecorded.avsc"
+    )
+    return runtime.probe.read("admin.action", partition, before[partition], schema)
+
+
+def require_odds_correlation(
+    runtime: E2eRuntime,
+    fixture: ScenarioIds,
+    action_id: str,
+) -> None:
+    mapping_key = poll_until(
+        "Odds action mapping",
+        lambda: runtime.odds.scalar("GET", "oddsfeed:operator:action:" + action_id),
+        lambda value: value.startswith("oddsfeed:operator:idempotency:"),
+        timeout=30,
+        interval=0.25,
+    )
+    metadata = runtime.odds.scalar("GET", mapping_key).split("|")
+    if len(metadata) != 5 or metadata[1] != action_id:
+        raise RuntimeError("Odds action metadata drifted")
+    sequence = metadata[2]
+    if not sequence.isdigit() or int(sequence) < 1:
+        raise RuntimeError("Odds action sequence is invalid")
+    committed_key = f"oddsfeed:operator:committed:{fixture.event}:{fixture.market}"
+    poll_until(
+        "Odds action commit",
+        lambda: runtime.odds.scalar("GET", committed_key),
+        lambda value: value == sequence,
+        timeout=30,
+        interval=0.25,
+    )
+    if runtime.odds.scalar("GET", f"market:{fixture.event}:{fixture.market}") != "SUSPENDED":
+        raise RuntimeError("Odds market state did not commit the operator action")


## `test(e2e): verify Admin action correlation`

diff --git a/e2e/scenario_13_admin_correlation.py b/e2e/scenario_13_admin_correlation.py
new file mode 100644
index 0000000..15d0240
--- /dev/null
+++ b/e2e/scenario_13_admin_correlation.py
@@ -0,0 +1,60 @@
+from __future__ import annotations
+
+from e2e.admin_correlation import (
+    admin_topic_offsets,
+    require_odds_correlation,
+    wait_admin_record,
+)
+from e2e.assertions import require_fields
+from e2e.model import ScenarioIds
+from e2e.runtime import E2eRuntime
+from scripts.cold_gate.polling import poll_until
+
+
+NAME = "admin-audit-downstream-correlation"
+TRACE_ID = "1" * 32
+TRACEPARENT = f"00-{TRACE_ID}-{'2' * 16}-01"
+REASON = "e2e correlation"
+
+
+def run(runtime: E2eRuntime) -> None:
+    fixture = ScenarioIds.create(13)
+    runtime.seed(fixture)
+    before = admin_topic_offsets(runtime)
+    token = runtime.admin_token()
+    action_id = runtime.market_admin.suspend(
+        fixture,
+        token,
+        "e2e-admin-market-13",
+        TRACEPARENT,
+        REASON,
+    )
+    audit = poll_until(
+        "authoritative Admin audit row",
+        lambda: runtime.market_admin.audit(action_id, token),
+        lambda payload: payload.get("outcome") == "SUCCESS",
+        timeout=30,
+        interval=0.25,
+    )
+    expected = {
+        "actionId": action_id,
+        "actorId": "e2e-admin",
+        "actorRole": "ADMIN",
+        "action": "MARKET_SUSPEND",
+        "target": f"{fixture.event}/{fixture.market}",
+        "outcome": "SUCCESS",
+        "httpStatus": 202,
+        "reason": REASON,
+        "traceId": TRACE_ID,
+    }
+    require_fields(audit, expected, "authoritative Admin audit")
+    if not audit.get("startedAt") or not audit.get("completedAt"):
+        raise RuntimeError("Admin audit lifecycle timestamps are incomplete")
+
+    record = wait_admin_record(runtime, before)
+    if record.key != "e2e-admin" or record.avro is None:
+        raise RuntimeError("Admin action Kafka identity drifted")
+    require_fields(record.avro, expected, "Admin action Kafka record")
+    if not record.avro.get("startedAt") or not record.avro.get("completedAt"):
+        raise RuntimeError("Admin action Kafka lifecycle timestamps are incomplete")
+    require_odds_correlation(runtime, fixture, action_id)


## `test(e2e): select the correlated admin record`

diff --git a/e2e/admin_correlation.py b/e2e/admin_correlation.py
index be76636..7ba0cb3 100644
--- a/e2e/admin_correlation.py
+++ b/e2e/admin_correlation.py
@@ -6,6 +6,9 @@ from scripts.cold_gate.kafka_record import KafkaRecord
 from scripts.cold_gate.polling import poll_until
 
 
+MAX_ADMIN_SCAN_RECORDS = 32
+
+
 def admin_topic_offsets(runtime: E2eRuntime) -> tuple[int, int, int]:
     return tuple(runtime.kafka.end_offset("admin.action", partition) for partition in range(3))
 
@@ -13,29 +16,43 @@ def admin_topic_offsets(runtime: E2eRuntime) -> tuple[int, int, int]:
 def wait_admin_record(
     runtime: E2eRuntime,
     before: tuple[int, int, int],
+    action_id: str,
 ) -> KafkaRecord:
+    schema = (
+        runtime.artifacts.sources
+        / "admin/src/main/avro/com/sportsbook/admin/event/AdminActionRecorded.avsc"
+    )
+
+    found: KafkaRecord | None = None
+
     def appended(after: tuple[int, int, int]) -> bool:
+        nonlocal found
         deltas = tuple(current - prior for current, prior in zip(after, before, strict=True))
-        if any(delta < 0 for delta in deltas) or sum(deltas) > 1:
-            raise RuntimeError("Admin action topic changed by more than one record")
-        return sorted(deltas) == [0, 0, 1]
+        if any(delta < 0 for delta in deltas) or sum(deltas) > MAX_ADMIN_SCAN_RECORDS:
+            raise RuntimeError("Admin action scan crossed its bounded window")
+        matches = []
+        for partition, (start, stop) in enumerate(zip(before, after, strict=True)):
+            for offset in range(start, stop):
+                record = runtime.probe.read("admin.action", partition, offset, schema)
+                if record.avro is None:
+                    raise RuntimeError("Admin action record is not typed Avro")
+                if record.avro.get("actionId") == action_id:
+                    matches.append(record)
+        if len(matches) > 1:
+            raise RuntimeError("Admin action ID was published more than once")
+        found = matches[0] if matches else None
+        return found is not None
 
-    after = poll_until(
+    poll_until(
         "Admin action publication",
         lambda: admin_topic_offsets(runtime),
         appended,
         timeout=60,
         interval=0.5,
     )
-    partition = next(
-        index for index, (current, prior) in enumerate(zip(after, before, strict=True))
-        if current == prior + 1
-    )
-    schema = (
-        runtime.artifacts.sources
-        / "admin/src/main/avro/com/sportsbook/admin/event/AdminActionRecorded.avsc"
-    )
-    return runtime.probe.read("admin.action", partition, before[partition], schema)
+    if found is None:
+        raise RuntimeError("Admin action publication lost its matched record")
+    return found
 
 
 def require_odds_correlation(
diff --git a/e2e/scenario_13_admin_correlation.py b/e2e/scenario_13_admin_correlation.py
index 15d0240..7fb6075 100644
--- a/e2e/scenario_13_admin_correlation.py
+++ b/e2e/scenario_13_admin_correlation.py
@@ -51,7 +51,7 @@ def run(runtime: E2eRuntime) -> None:
     if not audit.get("startedAt") or not audit.get("completedAt"):
         raise RuntimeError("Admin audit lifecycle timestamps are incomplete")
 
-    record = wait_admin_record(runtime, before)
+    record = wait_admin_record(runtime, before, action_id)
     if record.key != "e2e-admin" or record.avro is None:
         raise RuntimeError("Admin action Kafka identity drifted")
     require_fields(record.avro, expected, "Admin action Kafka record")


## `test(e2e): bound admin record selection`

diff --git a/tests/test_admin_correlation.py b/tests/test_admin_correlation.py
new file mode 100644
index 0000000..2929f1f
--- /dev/null
+++ b/tests/test_admin_correlation.py
@@ -0,0 +1,72 @@
+import pathlib
+import types
+import unittest
+
+from e2e.admin_correlation import MAX_ADMIN_SCAN_RECORDS, wait_admin_record
+from scripts.cold_gate.kafka_record import KafkaRecord
+
+
+TARGET = "77000000-0000-7000-8000-000000000013"
+
+
+def record(partition: int, offset: int, action_id: str) -> KafkaRecord:
+    return KafkaRecord(
+        "admin.action",
+        partition,
+        offset,
+        "e2e-admin",
+        b"record",
+        "0" * 64,
+        {},
+        {"actionId": action_id},
+    )
+
+
+class FakeKafka:
+    def __init__(self, offsets: tuple[int, int, int]) -> None:
+        self.offsets = offsets
+
+    def end_offset(self, _topic: str, partition: int) -> int:
+        return self.offsets[partition]
+
+
+class FakeProbe:
+    def __init__(self, records: dict[tuple[int, int], KafkaRecord]) -> None:
+        self.records = records
+        self.calls: list[tuple[int, int]] = []
+
+    def read(self, _topic: str, partition: int, offset: int, _schema) -> KafkaRecord:
+        self.calls.append((partition, offset))
+        return self.records[(partition, offset)]
+
+
+class AdminCorrelationTest(unittest.TestCase):
+    def runtime(self, offsets, records):
+        return types.SimpleNamespace(
+            kafka=FakeKafka(offsets),
+            probe=FakeProbe(records),
+            artifacts=types.SimpleNamespace(sources=pathlib.Path("/release/sources")),
+        )
+
+    def test_selects_action_id_while_allowing_other_admin_records(self) -> None:
+        records = {
+            (0, 0): record(0, 0, "late-candidate-action"),
+            (0, 1): record(0, 1, TARGET),
+            (1, 0): record(1, 0, "late-revision-action"),
+        }
+        runtime = self.runtime((2, 1, 0), records)
+
+        observed = wait_admin_record(runtime, (0, 0, 0), TARGET)
+
+        self.assertIs(observed, records[(0, 1)])
+        self.assertEqual(runtime.probe.calls, [(0, 0), (0, 1), (1, 0)])
+
+    def test_rejects_an_unbounded_admin_record_window(self) -> None:
+        runtime = self.runtime((MAX_ADMIN_SCAN_RECORDS + 1, 0, 0), {})
+
+        with self.assertRaisesRegex(RuntimeError, "bounded window"):
+            wait_admin_record(runtime, (0, 0, 0), TARGET)
+
+
+if __name__ == "__main__":
+    unittest.main()
