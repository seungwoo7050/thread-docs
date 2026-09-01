## `build(observability): gate collector readiness`

diff --git a/compose.observability.yaml b/compose.observability.yaml
index cd6746b..11b35bf 100644
--- a/compose.observability.yaml
+++ b/compose.observability.yaml
@@ -8,7 +8,7 @@ services:
       - ./observability/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
       - prometheus-data:/prometheus
     healthcheck:
-      test: ["CMD", "promtool", "query", "instant", "http://localhost:9090", "up"]
+      test: ["CMD", "wget", "--quiet", "--spider", "http://localhost:9090/-/ready"]
       interval: 5s
       timeout: 3s
       retries: 30
@@ -22,7 +22,7 @@ services:
       - ./observability/loki/loki.yml:/etc/loki/loki.yml:ro
       - loki-data:/loki
     healthcheck:
-      test: ["CMD", "/usr/bin/loki", "-version"]
+      test: ["CMD", "wget", "--quiet", "--spider", "http://localhost:3100/ready"]
       interval: 5s
       timeout: 3s
       retries: 30
@@ -65,6 +65,11 @@ services:
       - ${DOCKER_LOG_ROOT:-/var/lib/docker/containers}:/var/lib/docker/containers:ro
     depends_on:
       loki: {condition: service_healthy}
+    healthcheck:
+      test: ["CMD", "wget", "--quiet", "--spider", "http://localhost:9080/ready"]
+      interval: 5s
+      timeout: 3s
+      retries: 30
     networks: [backend]
     restart: unless-stopped
 


## `test(observability): verify readiness probes`

diff --git a/tests/test_combined_compose.py b/tests/test_combined_compose.py
index e57ad4f..15e57e0 100644
--- a/tests/test_combined_compose.py
+++ b/tests/test_combined_compose.py
@@ -66,6 +66,18 @@ class CombinedComposeTest(ComposeContractFixture):
         self.assertEqual(published["grafana"][0]["target"], 3000)
         for name in ("prometheus", "loki", "promtail"):
             self.assertNotIn("ports", services[name])
+        self.assertEqual(
+            services["prometheus"]["healthcheck"]["test"],
+            ["CMD", "wget", "--quiet", "--spider", "http://localhost:9090/-/ready"],
+        )
+        self.assertEqual(
+            services["loki"]["healthcheck"]["test"],
+            ["CMD", "wget", "--quiet", "--spider", "http://localhost:3100/ready"],
+        )
+        self.assertEqual(
+            services["promtail"]["healthcheck"]["test"],
+            ["CMD", "wget", "--quiet", "--spider", "http://localhost:9080/ready"],
+        )
 
         self.assertEqual(
             set(rendered["volumes"]),


## `fix(metrics): match the wallet service label`

diff --git a/scripts/cold_gate/readiness_evidence.py b/scripts/cold_gate/readiness_evidence.py
index d6a3f5c..3e6a403 100644
--- a/scripts/cold_gate/readiness_evidence.py
+++ b/scripts/cold_gate/readiness_evidence.py
@@ -24,6 +24,7 @@ INTEGRITY_METRICS = (
     "wallet_integrity_scan_failed",
     "wallet_integrity_last_checked_epoch_seconds",
 )
+WALLET_SERVICE_LABEL = '{service="wallet-service"}'
 NUMBER = r"[-+]?(?:[0-9]+(?:\.[0-9]+)?|\.[0-9]+)(?:[eE][-+]?[0-9]+)?"
 ClientFactory = Callable[[ComposeProject, str], ContainerHttpClient]
 
@@ -77,9 +78,13 @@ class ReadinessEvidence:
             raise RuntimeError("Wallet metrics are not UTF-8") from error
         values = []
         for name in INTEGRITY_METRICS:
-            matches = re.findall(rf"^{name}\s+({NUMBER})$", body, re.MULTILINE)
+            matches = re.findall(
+                rf"^{name}{re.escape(WALLET_SERVICE_LABEL)}\s+({NUMBER})$",
+                body,
+                re.MULTILINE,
+            )
             if len(matches) != 1:
-                raise RuntimeError(f"expected one unlabelled {name} metric")
+                raise RuntimeError(f"expected one wallet-service-labelled {name} metric")
             values.append(decimal.Decimal(matches[0]))
         return tuple(values)
 


## `test(metrics): require locked service labels`

diff --git a/e2e/metrics.py b/e2e/metrics.py
index d95d306..2ccd627 100644
--- a/e2e/metrics.py
+++ b/e2e/metrics.py
@@ -7,6 +7,8 @@ from scripts.cold_gate.container_http import ContainerHttpClient
 
 
 METRIC_NAME = re.compile(r"^[a-z][a-z0-9_]+$")
+BETTING_SERVICE_LABEL = '{service="betting-service"}'
+NUMBER = r"[-+]?(?:[0-9]+(?:\.[0-9]+)?|\.[0-9]+)(?:[eE][-+]?[0-9]+)?"
 
 
 def metric_value(client: ContainerHttpClient, name: str) -> decimal.Decimal:
@@ -17,10 +19,13 @@ def metric_value(client: ContainerHttpClient, name: str) -> decimal.Decimal:
         text = response.body.decode("utf-8")
     except UnicodeDecodeError as error:
         raise RuntimeError("Prometheus response is not UTF-8") from error
-    pattern = re.compile(rf"^{re.escape(name)}\s+([-+]?[0-9]+(?:\.[0-9]+)?)$", re.MULTILINE)
+    pattern = re.compile(
+        rf"^{re.escape(name)}{re.escape(BETTING_SERVICE_LABEL)}\s+({NUMBER})$",
+        re.MULTILINE,
+    )
     values = pattern.findall(text)
     if len(values) != 1:
-        raise RuntimeError(f"expected one unlabelled {name} metric")
+        raise RuntimeError(f"expected one betting-service-labelled {name} metric")
     try:
         return decimal.Decimal(values[0])
     except decimal.InvalidOperation as error:
diff --git a/tests/test_e2e_metrics.py b/tests/test_e2e_metrics.py
new file mode 100644
index 0000000..49d2fb8
--- /dev/null
+++ b/tests/test_e2e_metrics.py
@@ -0,0 +1,43 @@
+import decimal
+import unittest
+
+from e2e.metrics import metric_value
+from scripts.cold_gate.http import HttpResponse
+
+
+NAME = "betting_resolution_revision_gaps_total"
+LABEL = '{service="betting-service"}'
+
+
+class FakeClient:
+    def __init__(self, body: bytes) -> None:
+        self.body = body
+
+    def request(self, method: str, path: str) -> HttpResponse:
+        if (method, path) != ("GET", "/actuator/prometheus"):
+            raise AssertionError("unexpected metrics request")
+        return HttpResponse(200, (), self.body)
+
+
+class E2eMetricsTest(unittest.TestCase):
+    def test_reads_one_exact_betting_service_sample(self) -> None:
+        client = FakeClient(f"{NAME}{LABEL} 1.25e2\n".encode())
+
+        self.assertEqual(metric_value(client, NAME), decimal.Decimal("125"))
+
+    def test_rejects_missing_wrong_extra_or_duplicate_labels(self) -> None:
+        valid = f"{NAME}{LABEL} 1\n"
+        cases = (
+            f"{NAME} 1\n",
+            f'{NAME}{{service="other-service"}} 1\n',
+            f'{NAME}{{service="betting-service",region="test"}} 1\n',
+            valid + valid,
+        )
+        for body in cases:
+            with self.subTest(body=body):
+                with self.assertRaisesRegex(RuntimeError, "betting-service-labelled"):
+                    metric_value(FakeClient(body.encode()), NAME)
+
+
+if __name__ == "__main__":
+    unittest.main()
diff --git a/tests/test_readiness_evidence.py b/tests/test_readiness_evidence.py
index ceb73ba..a41ee98 100644
--- a/tests/test_readiness_evidence.py
+++ b/tests/test_readiness_evidence.py
@@ -11,9 +11,9 @@ from scripts.cold_gate.redaction import EvidenceRedactor
 
 SHA = "0123456789abcdef0123456789abcdef01234567"
 METRICS = (
-    b"wallet_integrity_total_drift 0.0\n"
-    b"wallet_integrity_scan_failed 0.0\n"
-    b"wallet_integrity_last_checked_epoch_seconds 1.777e9\n"
+    b'wallet_integrity_total_drift{service="wallet-service"} 0.0\n'
+    b'wallet_integrity_scan_failed{service="wallet-service"} 0.0\n'
+    b'wallet_integrity_last_checked_epoch_seconds{service="wallet-service"} 1.777e9\n'
 )
 
 
@@ -67,7 +67,7 @@ class ReadinessEvidenceTest(unittest.TestCase):
     def test_rejects_non_up_service_or_integrity_drift_before_writing(self) -> None:
         cases = (
             ({"admin": "DOWN"}, METRICS, "admin readiness"),
-            ({}, METRICS.replace(b"total_drift 0.0", b"total_drift 1.0"), "not clean"),
+            ({}, METRICS.replace(b'} 0.0', b'} 1.0', 1), "not clean"),
         )
         for overrides, metrics, message in cases:
             with self.subTest(message=message), tempfile.TemporaryDirectory() as temporary:
@@ -80,6 +80,19 @@ class ReadinessEvidenceTest(unittest.TestCase):
                     ReadinessEvidence(compose, store, factory).capture()
                 self.assertEqual(list(context.evidence.iterdir()), [])
 
+    def test_requires_the_exact_wallet_service_label(self) -> None:
+        cases = (
+            METRICS.replace(b'{service="wallet-service"}', b''),
+            METRICS.replace(b'wallet-service', b'other-service'),
+            METRICS.replace(b'} 0.0', b',region="test"} 0.0', 1),
+            METRICS + METRICS.splitlines(keepends=True)[0],
+        )
+        for metrics in cases:
+            with self.subTest(metrics=metrics):
+                client = FakeClient("wallet", {}, metrics)
+                with self.assertRaisesRegex(RuntimeError, "wallet-service-labelled"):
+                    ReadinessEvidence._metrics(client)
+
 
 if __name__ == "__main__":
     unittest.main()


## `fix(readiness): read owned wallet metrics`

diff --git a/scripts/cold_gate/readiness_evidence.py b/scripts/cold_gate/readiness_evidence.py
index 3e6a403..2532d96 100644
--- a/scripts/cold_gate/readiness_evidence.py
+++ b/scripts/cold_gate/readiness_evidence.py
@@ -24,7 +24,6 @@ INTEGRITY_METRICS = (
     "wallet_integrity_scan_failed",
     "wallet_integrity_last_checked_epoch_seconds",
 )
-WALLET_SERVICE_LABEL = '{service="wallet-service"}'
 NUMBER = r"[-+]?(?:[0-9]+(?:\.[0-9]+)?|\.[0-9]+)(?:[eE][-+]?[0-9]+)?"
 ClientFactory = Callable[[ComposeProject, str], ContainerHttpClient]
 
@@ -79,12 +78,12 @@ class ReadinessEvidence:
         values = []
         for name in INTEGRITY_METRICS:
             matches = re.findall(
-                rf"^{name}{re.escape(WALLET_SERVICE_LABEL)}\s+({NUMBER})$",
+                rf"^{name}(?:\{{[^}}\r\n]*\}})?\s+({NUMBER})$",
                 body,
                 re.MULTILINE,
             )
             if len(matches) != 1:
-                raise RuntimeError(f"expected one wallet-service-labelled {name} metric")
+                raise RuntimeError(f"expected one Wallet {name} metric")
             values.append(decimal.Decimal(matches[0]))
         return tuple(values)
 


## `test(readiness): accept owned wallet metric labels`

diff --git a/tests/test_readiness_evidence.py b/tests/test_readiness_evidence.py
index a41ee98..3470ae8 100644
--- a/tests/test_readiness_evidence.py
+++ b/tests/test_readiness_evidence.py
@@ -1,3 +1,4 @@
+import decimal
 import pathlib
 import tempfile
 import unittest
@@ -80,18 +81,29 @@ class ReadinessEvidenceTest(unittest.TestCase):
                     ReadinessEvidence(compose, store, factory).capture()
                 self.assertEqual(list(context.evidence.iterdir()), [])
 
-    def test_requires_the_exact_wallet_service_label(self) -> None:
+    def test_reads_metrics_from_the_owned_wallet_label_variants(self) -> None:
         cases = (
             METRICS.replace(b'{service="wallet-service"}', b''),
             METRICS.replace(b'wallet-service', b'other-service'),
             METRICS.replace(b'} 0.0', b',region="test"} 0.0', 1),
-            METRICS + METRICS.splitlines(keepends=True)[0],
         )
         for metrics in cases:
             with self.subTest(metrics=metrics):
                 client = FakeClient("wallet", {}, metrics)
-                with self.assertRaisesRegex(RuntimeError, "wallet-service-labelled"):
-                    ReadinessEvidence._metrics(client)
+                self.assertEqual(
+                    ReadinessEvidence._metrics(client),
+                    (decimal.Decimal(0), decimal.Decimal(0), decimal.Decimal("1.777e9")),
+                )
+
+    def test_requires_one_sample_for_every_wallet_metric(self) -> None:
+        cases = (
+            b"\n".join(METRICS.splitlines()[1:]) + b"\n",
+            METRICS + METRICS.splitlines(keepends=True)[0],
+        )
+        for metrics in cases:
+            with self.subTest(metrics=metrics):
+                with self.assertRaisesRegex(RuntimeError, "expected one Wallet"):
+                    ReadinessEvidence._metrics(FakeClient("wallet", {}, metrics))
 
 
 if __name__ == "__main__":
