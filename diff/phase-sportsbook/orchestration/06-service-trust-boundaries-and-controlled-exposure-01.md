# 서비스 신뢰 경계와 제한 노출

## `build(secrets): map exact caller and callee keys`

diff --git a/config/service-secrets.map b/config/service-secrets.map
new file mode 100644
index 0000000..2671ed4
--- /dev/null
+++ b/config/service-secrets.map
@@ -0,0 +1,12 @@
+# logical|caller-service|caller-env|callee-service|callee-env
+GATEWAY_BETTING_API_KEY|gateway|GATEWAY_BETTING_API_KEY|betting|BETTING_GATEWAY_API_KEY
+GATEWAY_WALLET_API_KEY|gateway|GATEWAY_WALLET_API_KEY|wallet|WALLET_GATEWAY_API_KEY
+BETTING_RISK_API_KEY|betting|BETTING_RISK_API_KEY|risk|INTERNAL_BETTING_SERVICE_API_KEY
+BETTING_WALLET_API_KEY|betting|BETTING_WALLET_API_KEY|wallet|WALLET_BETTING_SERVICE_API_KEY
+SETTLEMENT_WALLET_API_KEY|settlement|SETTLEMENT_WALLET_API_KEY|wallet|WALLET_SETTLEMENT_SERVICE_API_KEY
+ADMIN_WALLET_API_KEY|admin|ADMIN_WALLET_API_KEY|wallet|WALLET_ADMIN_API_KEY
+ADMIN_RISK_API_KEY|admin|ADMIN_RISK_API_KEY|risk|INTERNAL_ADMIN_API_KEY
+ADMIN_ODDS_FEED_API_KEY|admin|ADMIN_ODDS_FEED_API_KEY|odds|ADMIN_API_INTERNAL_KEY
+ADMIN_SETTLEMENT_API_KEY|admin|ADMIN_SETTLEMENT_API_KEY|settlement|SETTLEMENT_ADMIN_API_KEY
+WALLET_PLATFORM_API_KEY|e2e-runner|WALLET_PLATFORM_API_KEY|wallet|WALLET_PLATFORM_API_KEY
+INTERNAL_PLATFORM_API_KEY|e2e-runner|INTERNAL_PLATFORM_API_KEY|risk|INTERNAL_PLATFORM_API_KEY


## `test(secrets): verify exact caller callee pairs`

diff --git a/tests/test_service_secret_map.py b/tests/test_service_secret_map.py
new file mode 100644
index 0000000..d655606
--- /dev/null
+++ b/tests/test_service_secret_map.py
@@ -0,0 +1,44 @@
+import pathlib
+import unittest
+
+
+ROOT = pathlib.Path(__file__).resolve().parents[1]
+MAP = ROOT / "config/service-secrets.map"
+EXPECTED = {
+    "GATEWAY_BETTING_API_KEY": ("gateway", "GATEWAY_BETTING_API_KEY", "betting", "BETTING_GATEWAY_API_KEY"),
+    "GATEWAY_WALLET_API_KEY": ("gateway", "GATEWAY_WALLET_API_KEY", "wallet", "WALLET_GATEWAY_API_KEY"),
+    "BETTING_RISK_API_KEY": ("betting", "BETTING_RISK_API_KEY", "risk", "INTERNAL_BETTING_SERVICE_API_KEY"),
+    "BETTING_WALLET_API_KEY": ("betting", "BETTING_WALLET_API_KEY", "wallet", "WALLET_BETTING_SERVICE_API_KEY"),
+    "SETTLEMENT_WALLET_API_KEY": ("settlement", "SETTLEMENT_WALLET_API_KEY", "wallet", "WALLET_SETTLEMENT_SERVICE_API_KEY"),
+    "ADMIN_WALLET_API_KEY": ("admin", "ADMIN_WALLET_API_KEY", "wallet", "WALLET_ADMIN_API_KEY"),
+    "ADMIN_RISK_API_KEY": ("admin", "ADMIN_RISK_API_KEY", "risk", "INTERNAL_ADMIN_API_KEY"),
+    "ADMIN_ODDS_FEED_API_KEY": ("admin", "ADMIN_ODDS_FEED_API_KEY", "odds", "ADMIN_API_INTERNAL_KEY"),
+    "ADMIN_SETTLEMENT_API_KEY": ("admin", "ADMIN_SETTLEMENT_API_KEY", "settlement", "SETTLEMENT_ADMIN_API_KEY"),
+    "WALLET_PLATFORM_API_KEY": ("e2e-runner", "WALLET_PLATFORM_API_KEY", "wallet", "WALLET_PLATFORM_API_KEY"),
+    "INTERNAL_PLATFORM_API_KEY": ("e2e-runner", "INTERNAL_PLATFORM_API_KEY", "risk", "INTERNAL_PLATFORM_API_KEY"),
+}
+
+
+class ServiceSecretMapTest(unittest.TestCase):
+    def test_maps_each_logical_secret_to_one_exact_direction(self) -> None:
+        rows = [
+            line.split("|")
+            for line in MAP.read_text().splitlines()
+            if line and not line.startswith("#")
+        ]
+        self.assertTrue(all(len(row) == 5 for row in rows))
+        actual = {row[0]: tuple(row[1:]) for row in rows}
+        self.assertEqual(actual, EXPECTED)
+        self.assertEqual(
+            set(actual),
+            set((ROOT / "config/required-secrets.txt").read_text().splitlines()),
+        )
+        self.assertEqual(
+            len({(row[3], row[4]) for row in rows}),
+            len(rows),
+            "callee environment variables must have one logical owner",
+        )
+
+
+if __name__ == "__main__":
+    unittest.main()


## `test(secrets): reject every duplicate key pair`

diff --git a/tests/test_secret_distinctness.py b/tests/test_secret_distinctness.py
new file mode 100644
index 0000000..6bb49af
--- /dev/null
+++ b/tests/test_secret_distinctness.py
@@ -0,0 +1,19 @@
+import itertools
+import unittest
+
+from tests.secret_fixture import NAMES, VALID, assert_values_redacted, run_preflight
+
+
+class SecretDistinctnessTest(unittest.TestCase):
+    def test_rejects_every_pairwise_duplicate_without_rendering_values(self) -> None:
+        for left, right in itertools.combinations(NAMES, 2):
+            with self.subTest(left=left, right=right):
+                values = VALID | {right: VALID[left]}
+                result = run_preflight(values)
+                self.assertNotEqual(result.returncode, 0)
+                self.assertIn(f"{left} and {right}: values must differ", result.stderr)
+                assert_values_redacted(self, result)
+
+
+if __name__ == "__main__":
+    unittest.main()


## `test(secrets): verify presence and minimum length`

diff --git a/tests/secret_fixture.py b/tests/secret_fixture.py
new file mode 100644
index 0000000..39c9353
--- /dev/null
+++ b/tests/secret_fixture.py
@@ -0,0 +1,31 @@
+import os
+import pathlib
+import subprocess
+import sys
+
+
+ROOT = pathlib.Path(__file__).resolve().parents[1]
+SCRIPT = ROOT / "scripts/check-secrets.py"
+NAMES = tuple((ROOT / "config/required-secrets.txt").read_text().splitlines())
+VALID = {
+    name: f"contract-key-{index:02d}-" + (chr(65 + index) * 32)
+    for index, name in enumerate(NAMES)
+}
+
+
+def run_preflight(values: dict[str, str]) -> subprocess.CompletedProcess[str]:
+    environment = {"PATH": os.environ.get("PATH", "")} | values
+    return subprocess.run(
+        [sys.executable, str(SCRIPT)],
+        cwd=ROOT,
+        env=environment,
+        text=True,
+        capture_output=True,
+        check=False,
+    )
+
+
+def assert_values_redacted(test, result: subprocess.CompletedProcess[str]) -> None:
+    output = result.stdout + result.stderr
+    for value in VALID.values():
+        test.assertNotIn(value, output)
diff --git a/tests/test_secret_requirements.py b/tests/test_secret_requirements.py
new file mode 100644
index 0000000..9f52008
--- /dev/null
+++ b/tests/test_secret_requirements.py
@@ -0,0 +1,35 @@
+import unittest
+
+from tests.secret_fixture import NAMES, VALID, assert_values_redacted, run_preflight
+
+
+class SecretRequirementsTest(unittest.TestCase):
+    def test_accepts_eleven_complete_keys_without_rendering_values(self) -> None:
+        result = run_preflight(VALID)
+
+        self.assertEqual(result.returncode, 0, result.stderr)
+        self.assertEqual(result.stdout.strip(), "secret-preflight: validated 11 distinct keys")
+        assert_values_redacted(self, result)
+
+    def test_rejects_each_missing_key_without_rendering_values(self) -> None:
+        for name in NAMES:
+            with self.subTest(name=name):
+                values = VALID.copy()
+                del values[name]
+                result = run_preflight(values)
+                self.assertNotEqual(result.returncode, 0)
+                self.assertIn(f"{name}: missing", result.stderr)
+                assert_values_redacted(self, result)
+
+    def test_rejects_each_short_key_without_rendering_values(self) -> None:
+        for name in NAMES:
+            with self.subTest(name=name):
+                values = VALID | {name: "x" * 31}
+                result = run_preflight(values)
+                self.assertNotEqual(result.returncode, 0)
+                self.assertIn(f"{name}: shorter than 32 characters", result.stderr)
+                assert_values_redacted(self, result)
+
+
+if __name__ == "__main__":
+    unittest.main()


## `build(secrets): fail closed before stack startup`

diff --git a/config/required-secrets.txt b/config/required-secrets.txt
new file mode 100644
index 0000000..4e5c977
--- /dev/null
+++ b/config/required-secrets.txt
@@ -0,0 +1,11 @@
+GATEWAY_BETTING_API_KEY
+GATEWAY_WALLET_API_KEY
+BETTING_RISK_API_KEY
+BETTING_WALLET_API_KEY
+SETTLEMENT_WALLET_API_KEY
+ADMIN_WALLET_API_KEY
+ADMIN_RISK_API_KEY
+ADMIN_ODDS_FEED_API_KEY
+ADMIN_SETTLEMENT_API_KEY
+WALLET_PLATFORM_API_KEY
+INTERNAL_PLATFORM_API_KEY
diff --git a/scripts/check-secrets.py b/scripts/check-secrets.py
new file mode 100755
index 0000000..d7617d7
--- /dev/null
+++ b/scripts/check-secrets.py
@@ -0,0 +1,53 @@
+#!/usr/bin/env python3
+import os
+import pathlib
+import sys
+
+
+ROOT = pathlib.Path(__file__).resolve().parents[1]
+NAMES_FILE = ROOT / "config/required-secrets.txt"
+
+
+def required_names() -> list[str]:
+    names = [
+        line.strip()
+        for line in NAMES_FILE.read_text().splitlines()
+        if line.strip() and not line.startswith("#")
+    ]
+    if len(names) != 11 or len(set(names)) != len(names):
+        raise SystemExit("secret-preflight: invalid required-secret inventory")
+    return names
+
+
+def main() -> int:
+    errors = []
+    values: dict[str, str] = {}
+    for name in required_names():
+        value = os.environ.get(name, "")
+        values[name] = value
+        if not value:
+            errors.append(f"{name}: missing")
+        elif len(value) < 32:
+            errors.append(f"{name}: shorter than 32 characters")
+        elif not value.isascii() or not value.isprintable() or " " in value:
+            errors.append(f"{name}: must use printable non-space ASCII")
+
+    seen: dict[str, str] = {}
+    for name, value in values.items():
+        if not value:
+            continue
+        if value in seen:
+            errors.append(f"{seen[value]} and {name}: values must differ")
+        else:
+            seen[value] = name
+
+    if errors:
+        for error in errors:
+            print(f"secret-preflight: {error}", file=sys.stderr)
+        return 1
+    print("secret-preflight: validated 11 distinct keys")
+    return 0
+
+
+if __name__ == "__main__":
+    raise SystemExit(main())


## `test(startup): verify isolated secret preflight`

diff --git a/tests/test_compose_secret_preflight.py b/tests/test_compose_secret_preflight.py
new file mode 100644
index 0000000..6863d6d
--- /dev/null
+++ b/tests/test_compose_secret_preflight.py
@@ -0,0 +1,23 @@
+from tests.compose_contract_fixture import ComposeContractFixture
+from tests.secret_fixture import VALID
+
+
+class ComposeSecretPreflightTest(ComposeContractFixture):
+    def test_runs_before_services_without_network_or_value_disclosure(self) -> None:
+        preflight = self.service("secret-preflight")
+        self.assertEqual(preflight["network_mode"], "none")
+        self.assertNotIn("networks", preflight)
+
+        result = self.compose("run", "--rm", "--no-deps", "secret-preflight")
+
+        self.assertEqual(result.returncode, 0, result.stderr)
+        self.assertIn("validated 11 distinct keys", result.stdout)
+        output = result.stdout + result.stderr
+        for value in VALID.values():
+            self.assertNotIn(value, output)
+
+
+if __name__ == "__main__":
+    import unittest
+
+    unittest.main()


## `build(gate): generate isolated runtime secrets`

diff --git a/scripts/cold_gate/secrets.py b/scripts/cold_gate/secrets.py
new file mode 100644
index 0000000..a38e5a2
--- /dev/null
+++ b/scripts/cold_gate/secrets.py
@@ -0,0 +1,75 @@
+from __future__ import annotations
+
+import dataclasses
+import os
+import secrets
+import subprocess
+from pathlib import Path
+
+from scripts.cold_gate.context import ColdGateContext
+
+
+@dataclasses.dataclass(frozen=True)
+class RuntimeSecrets:
+    environment: dict[str, str]
+    private_key: Path
+    secret_values: tuple[str, ...]
+
+    @classmethod
+    def generate(cls, context: ColdGateContext) -> "RuntimeSecrets":
+        context.require_owned()
+        names = tuple(
+            line.strip()
+            for line in (context.root / "config/required-secrets.txt").read_text().splitlines()
+            if line.strip()
+        )
+        if len(names) != 11 or len(set(names)) != 11:
+            raise RuntimeError("required secret inventory is invalid")
+        values = tuple(secrets.token_urlsafe(32) for _ in names)
+        if len(set(values)) != 11 or any(len(value) < 32 for value in values):
+            raise RuntimeError("generated service keys are invalid")
+
+        secret_directory = context.runtime / "secrets"
+        secret_directory.mkdir(mode=0o700)
+        private_key = secret_directory / "jwt-private.pem"
+        public_key = secret_directory / "jwt-public.pem"
+        subprocess.run(
+            [
+                "openssl",
+                "genpkey",
+                "-algorithm",
+                "RSA",
+                "-pkeyopt",
+                "rsa_keygen_bits:2048",
+                "-out",
+                str(private_key),
+            ],
+            check=True,
+            stdout=subprocess.DEVNULL,
+            stderr=subprocess.DEVNULL,
+        )
+        subprocess.run(
+            ["openssl", "pkey", "-in", str(private_key), "-pubout", "-out", str(public_key)],
+            check=True,
+            stdout=subprocess.DEVNULL,
+            stderr=subprocess.DEVNULL,
+        )
+        private_key.chmod(0o600)
+        public_key.chmod(0o600)
+        public_pem = public_key.read_text()
+        postgres_password = secrets.token_urlsafe(32)
+        grafana_password = secrets.token_urlsafe(32)
+        environment = os.environ.copy()
+        environment.update(dict(zip(names, values, strict=True)))
+        environment.update(
+            {
+                "POSTGRES_PASSWORD": postgres_password,
+                "GRAFANA_ADMIN_PASSWORD": grafana_password,
+                "GATEWAY_JWT_PUBLIC_KEY": public_pem,
+                "ADMIN_JWT_PUBLIC_KEY": public_pem,
+                "ADMIN_JWT_ISSUER": "sportsbook-admin-e2e",
+                "GATEWAY_HOST_PORT": "0",
+            }
+        )
+        sensitive = (*values, postgres_password, grafana_password, public_pem)
+        return cls(environment, private_key, sensitive)


## `test(gate): verify ephemeral secret preflight`

diff --git a/tests/test_runtime_secrets.py b/tests/test_runtime_secrets.py
new file mode 100644
index 0000000..c18bb83
--- /dev/null
+++ b/tests/test_runtime_secrets.py
@@ -0,0 +1,55 @@
+import pathlib
+import stat
+import subprocess
+import sys
+import tempfile
+import unittest
+
+from scripts.cold_gate.context import ColdGateContext
+from scripts.cold_gate.secrets import RuntimeSecrets
+
+
+ROOT = pathlib.Path(__file__).resolve().parents[1]
+SHA = "0123456789abcdef0123456789abcdef01234567"
+
+
+class RuntimeSecretsTest(unittest.TestCase):
+    def test_generates_distinct_keys_and_inline_public_pem(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            root = pathlib.Path(temporary).resolve()
+            (root / "config").mkdir()
+            (root / "config/required-secrets.txt").write_text(
+                (ROOT / "config/required-secrets.txt").read_text()
+            )
+            context = ColdGateContext.create(root, SHA, "00000001")
+
+            generated = RuntimeSecrets.generate(context)
+
+            names = (ROOT / "config/required-secrets.txt").read_text().splitlines()
+            service_values = [generated.environment[name] for name in names]
+            self.assertEqual(len(service_values), 11)
+            self.assertEqual(len(set(service_values)), 11)
+            self.assertTrue(all(len(value) >= 32 for value in service_values))
+            public_key = generated.environment["GATEWAY_JWT_PUBLIC_KEY"]
+            self.assertEqual(public_key, generated.environment["ADMIN_JWT_PUBLIC_KEY"])
+            self.assertTrue(public_key.startswith("-----BEGIN PUBLIC KEY-----"))
+            self.assertEqual(
+                stat.S_IMODE(generated.private_key.stat().st_mode), 0o600
+            )
+            self.assertFalse(any(context.runtime.rglob("*.env")))
+
+            checked = subprocess.run(
+                [sys.executable, str(ROOT / "scripts/check-secrets.py")],
+                cwd=ROOT,
+                env=generated.environment,
+                text=True,
+                capture_output=True,
+                check=False,
+            )
+            self.assertEqual(checked.returncode, 0, checked.stderr)
+            for value in generated.secret_values:
+                self.assertNotIn(value, checked.stdout + checked.stderr)
+
+
+if __name__ == "__main__":
+    unittest.main()


## `build(gateway): wire inline JWT edge runtime`

diff --git a/compose.yaml b/compose.yaml
index 4ff42ec..33488d5 100644
--- a/compose.yaml
+++ b/compose.yaml
@@ -251,6 +251,54 @@ services:
     networks: [backend]
     restart: unless-stopped
 
+  gateway:
+    build:
+      context: ./docker
+      dockerfile: Dockerfile.jvm
+      args:
+        JAR: gateway.jar
+    environment:
+      GATEWAY_HTTP_PORT: "8080"
+      GATEWAY_REDIS_HOST: redis-gateway
+      GATEWAY_REDIS_PORT: "6379"
+      GATEWAY_REDIS_DATABASE: "0"
+      GATEWAY_REDIS_SSL: "false"
+      GATEWAY_KAFKA_BOOTSTRAP: kafka:9092
+      GATEWAY_BETTING_URI: http://betting:8082
+      GATEWAY_WALLET_URI: http://wallet:8081
+      GATEWAY_ODDS_FEED_URI: http://odds:8085
+      GATEWAY_BETTING_API_KEY: ${GATEWAY_BETTING_API_KEY:-}
+      GATEWAY_WALLET_API_KEY: ${GATEWAY_WALLET_API_KEY:-}
+      GATEWAY_JWT_PUBLIC_KEY: ${GATEWAY_JWT_PUBLIC_KEY:-}
+      GATEWAY_TOPIC_ODDS_CHANGED: odds.changed
+      GATEWAY_TOPIC_BET_SETTLED: bet.settled.v1
+      GATEWAY_TOPIC_BET_VOIDED: bet.voided.v1
+      GATEWAY_TOPIC_BET_RESOLUTION_REVISED: bet.resolution.revised.v1
+    depends_on:
+      kafka:
+        condition: service_healthy
+      redis-gateway:
+        condition: service_healthy
+      topic-init:
+        condition: service_completed_successfully
+      wallet:
+        condition: service_healthy
+      odds:
+        condition: service_healthy
+      betting:
+        condition: service_healthy
+    healthcheck:
+      test: ["CMD-SHELL", "curl --fail --silent --show-error http://localhost:8080/actuator/health/readiness >/dev/null"]
+      interval: 3s
+      timeout: 3s
+      retries: 60
+      start_period: 30s
+    expose: ["8080"]
+    deploy:
+      replicas: 1
+    networks: [backend]
+    restart: unless-stopped
+
 volumes:
   postgres-data:
   kafka-data:


## `test(gateway): verify inline PEM and single replica`

diff --git a/tests/test_gateway_wiring.py b/tests/test_gateway_wiring.py
new file mode 100644
index 0000000..7a81e8f
--- /dev/null
+++ b/tests/test_gateway_wiring.py
@@ -0,0 +1,62 @@
+from tests.compose_contract_fixture import ComposeContractFixture, PUBLIC_KEY
+from tests.secret_fixture import VALID
+
+
+class GatewayWiringTest(ComposeContractFixture):
+    def test_wires_inline_pem_single_replica_and_exact_dependencies(self) -> None:
+        gateway = self.service("gateway")
+        environment = gateway["environment"]
+
+        self.assert_runtime_build(gateway, "gateway.jar")
+        self.assertEqual(environment["GATEWAY_REDIS_HOST"], "redis-gateway")
+        self.assertEqual(environment["GATEWAY_KAFKA_BOOTSTRAP"], "kafka:9092")
+        self.assertEqual(environment["GATEWAY_BETTING_URI"], "http://betting:8082")
+        self.assertEqual(environment["GATEWAY_WALLET_URI"], "http://wallet:8081")
+        self.assertEqual(environment["GATEWAY_ODDS_FEED_URI"], "http://odds:8085")
+        self.assertEqual(environment["GATEWAY_JWT_PUBLIC_KEY"], PUBLIC_KEY)
+        self.assertIn("\n", environment["GATEWAY_JWT_PUBLIC_KEY"])
+        self.assertEqual(
+            {
+                name: value
+                for name, value in environment.items()
+                if name.endswith("API_KEY")
+            },
+            {
+                "GATEWAY_BETTING_API_KEY": VALID["GATEWAY_BETTING_API_KEY"],
+                "GATEWAY_WALLET_API_KEY": VALID["GATEWAY_WALLET_API_KEY"],
+            },
+        )
+        self.assertEqual(gateway["deploy"]["replicas"], 1)
+        self.assertIn(
+            "/actuator/health/readiness", gateway["healthcheck"]["test"][1]
+        )
+        self.assert_dependency_conditions(
+            gateway,
+            {
+                "kafka": "service_healthy",
+                "redis-gateway": "service_healthy",
+                "topic-init": "service_completed_successfully",
+                "wallet": "service_healthy",
+                "odds": "service_healthy",
+                "betting": "service_healthy",
+            },
+        )
+        self.assertEqual(
+            {
+                name: value
+                for name, value in environment.items()
+                if name.startswith("GATEWAY_TOPIC_")
+            },
+            {
+                "GATEWAY_TOPIC_ODDS_CHANGED": "odds.changed",
+                "GATEWAY_TOPIC_BET_SETTLED": "bet.settled.v1",
+                "GATEWAY_TOPIC_BET_VOIDED": "bet.voided.v1",
+                "GATEWAY_TOPIC_BET_RESOLUTION_REVISED": "bet.resolution.revised.v1",
+            },
+        )
+
+
+if __name__ == "__main__":
+    import unittest
+
+    unittest.main()


