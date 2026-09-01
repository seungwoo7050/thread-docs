## `test(gate): verify exact JWT claim boundaries`

diff --git a/tests/test_cold_gate_jwt.py b/tests/test_cold_gate_jwt.py
new file mode 100644
index 0000000..e8ce8d2
--- /dev/null
+++ b/tests/test_cold_gate_jwt.py
@@ -0,0 +1,85 @@
+import base64
+import json
+import pathlib
+import subprocess
+import tempfile
+import unittest
+
+from scripts.cold_gate.context import ColdGateContext
+from scripts.cold_gate.jwt import JwtSigner
+
+
+SHA = "0123456789abcdef0123456789abcdef01234567"
+USER = "01000000-0000-7000-8000-000000000001"
+
+
+def decode(segment: str) -> dict[str, object]:
+    padding = "=" * (-len(segment) % 4)
+    return json.loads(base64.urlsafe_b64decode(segment + padding))
+
+
+class ColdGateJwtTest(unittest.TestCase):
+    def fixture(self, root: pathlib.Path) -> tuple[ColdGateContext, pathlib.Path]:
+        context = ColdGateContext.create(root, SHA, "00000001")
+        secret_dir = context.runtime / "secrets"
+        secret_dir.mkdir(mode=0o700)
+        private_key = secret_dir / "jwt-private.pem"
+        private_key.write_text("fixture private key")
+        private_key.chmod(0o600)
+        return context, private_key
+
+    def test_signs_exact_user_and_admin_claim_shapes(self) -> None:
+        calls = []
+
+        def runner(command, **options):
+            calls.append((command, options))
+            return subprocess.CompletedProcess(command, 0, stdout=b"signature")
+
+        with tempfile.TemporaryDirectory() as temporary:
+            context, key = self.fixture(pathlib.Path(temporary).resolve())
+            signer = JwtSigner(context, key, runner)
+
+            user = signer.user(USER, 1_700_000_000)
+            admin = signer.admin(1_700_000_000)
+
+        user_header, user_payload, user_signature = user.split(".")
+        self.assertEqual(decode(user_header), {"alg": "RS256", "typ": "JWT"})
+        self.assertEqual(
+            decode(user_payload),
+            {"exp": 1_700_001_200, "iat": 1_700_000_000, "roles": ["USER"], "sub": USER},
+        )
+        self.assertEqual(
+            decode(admin.split(".")[1]),
+            {
+                "exp": 1_700_001_200,
+                "iat": 1_700_000_000,
+                "iss": "sportsbook-admin-e2e",
+                "nbf": 1_699_999_995,
+                "role": "ADMIN",
+                "sub": "e2e-admin",
+            },
+        )
+        self.assertEqual(user_signature, "c2lnbmF0dXJl")
+        self.assertEqual(len(calls), 2)
+        self.assertEqual(calls[0][0][:4], ["openssl", "dgst", "-sha256", "-sign"])
+        self.assertNotIn("fixture private key", str(calls))
+
+    def test_rejects_non_uuidv7_subjects(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            context, key = self.fixture(pathlib.Path(temporary).resolve())
+            signer = JwtSigner(context, key, lambda *args, **kwargs: None)
+
+            with self.assertRaisesRegex(ValueError, "UUIDv7"):
+                signer.user("00000000-0000-4000-8000-000000000001", 1)
+
+    def test_rejects_broad_private_key_permissions(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            context, key = self.fixture(pathlib.Path(temporary).resolve())
+            key.chmod(0o644)
+
+            with self.assertRaisesRegex(RuntimeError, "permissions"):
+                JwtSigner(context, key)
+
+
+if __name__ == "__main__":
+    unittest.main()


## `build(network): expose only the loopback gateway`

diff --git a/compose.yaml b/compose.yaml
index 0438c90..734e845 100644
--- a/compose.yaml
+++ b/compose.yaml
@@ -330,6 +330,11 @@ services:
       retries: 60
       start_period: 30s
     expose: ["8080"]
+    ports:
+      - target: 8080
+        published: ${GATEWAY_HOST_PORT:-8080}
+        host_ip: 127.0.0.1
+        protocol: tcp
     deploy:
       replicas: 1
     networks: [backend]


## `test(network): verify loopback and service isolation`

diff --git a/tests/test_network_exposure.py b/tests/test_network_exposure.py
new file mode 100644
index 0000000..9184bd4
--- /dev/null
+++ b/tests/test_network_exposure.py
@@ -0,0 +1,39 @@
+from tests.compose_contract_fixture import ComposeContractFixture
+
+
+class NetworkExposureTest(ComposeContractFixture):
+    def test_exposes_only_one_loopback_gateway_on_an_internal_network(self) -> None:
+        self.environment["GATEWAY_HOST_PORT"] = "18080"
+        rendered = self.rendered()
+        services = rendered["services"]
+
+        published = {name for name, service in services.items() if service.get("ports")}
+        self.assertEqual(published, {"gateway"})
+        self.assertEqual(
+            services["gateway"]["ports"],
+            [
+                {
+                    "mode": "ingress",
+                    "host_ip": "127.0.0.1",
+                    "target": 8080,
+                    "published": "18080",
+                    "protocol": "tcp",
+                }
+            ],
+        )
+        self.assertEqual(set(rendered["networks"]), {"backend"})
+        self.assertTrue(rendered["networks"]["backend"]["internal"])
+
+        for name, service in services.items():
+            with self.subTest(service=name):
+                self.assertNotEqual(service.get("network_mode"), "host")
+                if name == "secret-preflight":
+                    self.assertEqual(service["network_mode"], "none")
+                else:
+                    self.assertEqual(set(service["networks"]), {"backend"})
+
+
+if __name__ == "__main__":
+    import unittest
+
+    unittest.main()


## `fix(harness): allocate a loopback gateway port`

diff --git a/scripts/cold_gate/secrets.py b/scripts/cold_gate/secrets.py
index b4c8f4a..bad7180 100644
--- a/scripts/cold_gate/secrets.py
+++ b/scripts/cold_gate/secrets.py
@@ -3,12 +3,19 @@ from __future__ import annotations
 import dataclasses
 import os
 import secrets
+import socket
 import subprocess
 from pathlib import Path
 
 from scripts.cold_gate.context import ColdGateContext
 
 
+def _available_loopback_port() -> str:
+    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as candidate:
+        candidate.bind(("127.0.0.1", 0))
+        return str(candidate.getsockname()[1])
+
+
 @dataclasses.dataclass(frozen=True)
 class RuntimeSecrets:
     environment: dict[str, str]
@@ -68,7 +75,7 @@ class RuntimeSecrets:
                 "GATEWAY_JWT_PUBLIC_KEY": public_pem,
                 "ADMIN_JWT_PUBLIC_KEY": public_pem,
                 "ADMIN_JWT_ISSUER": "sportsbook-admin-e2e",
-                "GATEWAY_HOST_PORT": "0",
+                "GATEWAY_HOST_PORT": _available_loopback_port(),
                 "COMPOSE_PROJECT_NAME": context.project,
             }
         )


## `fix(harness): expose the owned gateway port`

diff --git a/scripts/cold_gate/secrets.py b/scripts/cold_gate/secrets.py
index bad7180..1cd57be 100644
--- a/scripts/cold_gate/secrets.py
+++ b/scripts/cold_gate/secrets.py
@@ -22,6 +22,13 @@ class RuntimeSecrets:
     private_key: Path
     secret_values: tuple[str, ...]
 
+    @property
+    def gateway_port(self) -> int:
+        port = int(self.environment["GATEWAY_HOST_PORT"])
+        if not 0 < port <= 65535:
+            raise RuntimeError("generated gateway port is invalid")
+        return port
+
     @classmethod
     def generate(cls, context: ColdGateContext) -> "RuntimeSecrets":
         context.require_owned()


## `test(e2e): use the owned gateway port`

diff --git a/e2e/runtime.py b/e2e/runtime.py
index 4a6a35b..9cf56a0 100644
--- a/e2e/runtime.py
+++ b/e2e/runtime.py
@@ -25,7 +25,6 @@ from scripts.cold_gate.polling import poll_until
 from scripts.cold_gate.probe import KafkaProbe
 from scripts.cold_gate.redis import RedisClient
 from scripts.cold_gate.secrets import RuntimeSecrets
-from scripts.cold_gate.stack import ColdStack
 
 
 SETTLEMENT_ASSIGNMENTS = frozenset(
@@ -45,7 +44,7 @@ class E2eRuntime:
     ) -> None:
         if compose.context is not context:
             raise RuntimeError("E2E runtime ownership mismatch")
-        gateway_port = ColdStack(context, compose).loopback_port("gateway", 8080)
+        gateway_port = secrets.gateway_port
         database = PostgresClient(compose)
         self.context = context
         self.compose = compose
diff --git a/tests/test_runtime_secrets.py b/tests/test_runtime_secrets.py
index 411620d..e1b8618 100644
--- a/tests/test_runtime_secrets.py
+++ b/tests/test_runtime_secrets.py
@@ -34,7 +34,7 @@ class RuntimeSecretsTest(unittest.TestCase):
             self.assertEqual(public_key, generated.environment["ADMIN_JWT_PUBLIC_KEY"])
             self.assertTrue(public_key.startswith("-----BEGIN PUBLIC KEY-----"))
             self.assertEqual(generated.environment["COMPOSE_PROJECT_NAME"], context.project)
-            gateway_port = int(generated.environment["GATEWAY_HOST_PORT"])
+            gateway_port = generated.gateway_port
             self.assertTrue(0 < gateway_port <= 65535)
             self.assertEqual(
                 stat.S_IMODE(generated.private_key.stat().st_mode), 0o600
@@ -53,6 +53,12 @@ class RuntimeSecretsTest(unittest.TestCase):
             for value in generated.secret_values:
                 self.assertNotIn(value, checked.stdout + checked.stderr)
 
+    def test_rejects_an_invalid_gateway_port(self) -> None:
+        generated = RuntimeSecrets({"GATEWAY_HOST_PORT": "0"}, pathlib.Path("key"), ())
+
+        with self.assertRaisesRegex(RuntimeError, "gateway port"):
+            _ = generated.gateway_port
+
 
 if __name__ == "__main__":
     unittest.main()
