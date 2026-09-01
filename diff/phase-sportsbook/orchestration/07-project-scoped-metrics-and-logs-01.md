# 프로젝트 범위 메트릭과 로그

## `build(metrics): scrape every service endpoint`

diff --git a/observability/prometheus/prometheus.yml b/observability/prometheus/prometheus.yml
new file mode 100644
index 0000000..54efcbd
--- /dev/null
+++ b/observability/prometheus/prometheus.yml
@@ -0,0 +1,27 @@
+global:
+  scrape_interval: 5s
+  scrape_timeout: 3s
+  evaluation_interval: 5s
+
+scrape_configs:
+  - job_name: prometheus
+    static_configs:
+      - targets: ["localhost:9090"]
+
+  - job_name: sportsbook-services
+    metrics_path: /actuator/prometheus
+    static_configs:
+      - targets: ["gateway:8080"]
+        labels: {service: gateway}
+      - targets: ["wallet:8081"]
+        labels: {service: wallet}
+      - targets: ["betting:8082"]
+        labels: {service: betting}
+      - targets: ["risk:8083"]
+        labels: {service: risk}
+      - targets: ["settlement:8084"]
+        labels: {service: settlement}
+      - targets: ["odds:8085"]
+        labels: {service: odds}
+      - targets: ["admin:8090"]
+        labels: {service: admin}


## `test(metrics): verify exact scrape targets`

diff --git a/tests/test_prometheus_config.py b/tests/test_prometheus_config.py
new file mode 100644
index 0000000..09bb075
--- /dev/null
+++ b/tests/test_prometheus_config.py
@@ -0,0 +1,51 @@
+import pathlib
+import re
+import subprocess
+import unittest
+
+
+ROOT = pathlib.Path(__file__).resolve().parents[1]
+CONFIG = ROOT / "observability/prometheus/prometheus.yml"
+
+
+class PrometheusConfigTest(unittest.TestCase):
+    def test_is_valid_and_scrapes_the_exact_service_endpoints(self) -> None:
+        checked = subprocess.run(
+            [
+                "docker",
+                "run",
+                "--rm",
+                "--network",
+                "none",
+                "--volume",
+                f"{CONFIG}:/etc/prometheus/prometheus.yml:ro",
+                "--entrypoint",
+                "promtool",
+                "prom/prometheus:v2.54.1",
+                "check",
+                "config",
+                "/etc/prometheus/prometheus.yml",
+            ],
+            text=True,
+            capture_output=True,
+            check=False,
+        )
+        self.assertEqual(checked.returncode, 0, checked.stderr)
+
+        targets = set(re.findall(r'targets: \["([^\"]+)"\]', CONFIG.read_text()))
+        self.assertEqual(
+            targets - {"localhost:9090"},
+            {
+                "gateway:8080",
+                "wallet:8081",
+                "betting:8082",
+                "risk:8083",
+                "settlement:8084",
+                "odds:8085",
+                "admin:8090",
+            },
+        )
+
+
+if __name__ == "__main__":
+    unittest.main()


## `build(logs): configure local Loki retention`

diff --git a/observability/loki/loki.yml b/observability/loki/loki.yml
new file mode 100644
index 0000000..a20389d
--- /dev/null
+++ b/observability/loki/loki.yml
@@ -0,0 +1,45 @@
+auth_enabled: false
+
+server:
+  http_listen_port: 3100
+  grpc_listen_port: 9096
+  log_level: warn
+
+common:
+  instance_addr: 127.0.0.1
+  path_prefix: /loki
+  storage:
+    filesystem:
+      chunks_directory: /loki/chunks
+      rules_directory: /loki/rules
+  replication_factor: 1
+  ring:
+    kvstore:
+      store: inmemory
+
+schema_config:
+  configs:
+    - from: 2024-01-01
+      store: tsdb
+      object_store: filesystem
+      schema: v13
+      index:
+        prefix: index_
+        period: 24h
+
+limits_config:
+  retention_period: 72h
+  reject_old_samples: true
+  reject_old_samples_max_age: 168h
+  allow_structured_metadata: true
+
+compactor:
+  working_directory: /loki/compactor
+  delete_request_store: filesystem
+  retention_enabled: true
+
+ruler:
+  storage:
+    type: local
+    local:
+      directory: /loki/rules


## `test(logs): verify Loki retention config`

diff --git a/tests/test_loki_config.py b/tests/test_loki_config.py
new file mode 100644
index 0000000..50867fc
--- /dev/null
+++ b/tests/test_loki_config.py
@@ -0,0 +1,38 @@
+import pathlib
+import subprocess
+import unittest
+
+
+ROOT = pathlib.Path(__file__).resolve().parents[1]
+CONFIG = ROOT / "observability/loki/loki.yml"
+
+
+class LokiConfigTest(unittest.TestCase):
+    def test_validates_single_node_storage_and_bounded_retention(self) -> None:
+        checked = subprocess.run(
+            [
+                "docker",
+                "run",
+                "--rm",
+                "--network",
+                "none",
+                "--volume",
+                f"{CONFIG}:/etc/loki/loki.yml:ro",
+                "grafana/loki:3.1.1",
+                "-verify-config=true",
+                "-config.file=/etc/loki/loki.yml",
+            ],
+            text=True,
+            capture_output=True,
+            check=False,
+        )
+        self.assertEqual(checked.returncode, 0, checked.stderr)
+        config = CONFIG.read_text()
+        self.assertIn("replication_factor: 1", config)
+        self.assertIn("retention_period: 72h", config)
+        self.assertIn("retention_enabled: true", config)
+        self.assertNotIn("s3", config.lower())
+
+
+if __name__ == "__main__":
+    unittest.main()


## `build(observability): compose isolated metrics and logs`

diff --git a/compose.observability.yaml b/compose.observability.yaml
new file mode 100644
index 0000000..55856bf
--- /dev/null
+++ b/compose.observability.yaml
@@ -0,0 +1,78 @@
+services:
+  prometheus:
+    image: prom/prometheus:v2.54.1
+    command:
+      - --config.file=/etc/prometheus/prometheus.yml
+      - --storage.tsdb.retention.time=72h
+    volumes:
+      - ./observability/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
+      - prometheus-data:/prometheus
+    ports:
+      - {target: 9090, published: "${PROMETHEUS_HOST_PORT:-9090}", host_ip: 127.0.0.1}
+    healthcheck:
+      test: ["CMD", "promtool", "query", "instant", "http://localhost:9090", "up"]
+      interval: 5s
+      timeout: 3s
+      retries: 30
+    networks: [backend]
+    restart: unless-stopped
+
+  loki:
+    image: grafana/loki:3.1.1
+    command: ["-config.file=/etc/loki/loki.yml"]
+    volumes:
+      - ./observability/loki/loki.yml:/etc/loki/loki.yml:ro
+      - loki-data:/loki
+    ports:
+      - {target: 3100, published: "${LOKI_HOST_PORT:-3100}", host_ip: 127.0.0.1}
+    healthcheck:
+      test: ["CMD", "/usr/bin/loki", "-version"]
+      interval: 5s
+      timeout: 3s
+      retries: 30
+    networks: [backend]
+    restart: unless-stopped
+
+  grafana:
+    image: grafana/grafana:11.2.0
+    environment:
+      GF_SECURITY_ADMIN_USER: admin
+      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_ADMIN_PASSWORD:?set GRAFANA_ADMIN_PASSWORD}
+      GF_AUTH_ANONYMOUS_ENABLED: "false"
+      GF_USERS_DEFAULT_THEME: dark
+    volumes:
+      - ./observability/grafana/provisioning:/etc/grafana/provisioning:ro
+      - grafana-data:/var/lib/grafana
+    ports:
+      - {target: 3000, published: "${GRAFANA_HOST_PORT:-3000}", host_ip: 127.0.0.1}
+    healthcheck:
+      test: ["CMD-SHELL", "wget --quiet --spider http://localhost:3000/api/health"]
+      interval: 5s
+      timeout: 3s
+      retries: 30
+    depends_on:
+      prometheus: {condition: service_healthy}
+      loki: {condition: service_healthy}
+    networks: [backend]
+    restart: unless-stopped
+
+  promtail:
+    image: grafana/promtail:3.1.1
+    command:
+      - -config.file=/etc/promtail/promtail.yml
+      - -config.expand-env=true
+    environment:
+      COMPOSE_PROJECT_NAME: ${COMPOSE_PROJECT_NAME:-sportsbook}
+    volumes:
+      - ./observability/promtail/promtail.yml:/etc/promtail/promtail.yml:ro
+      - ${DOCKER_SOCKET_PATH:-/var/run/docker.sock}:/var/run/docker.sock:ro
+      - ${DOCKER_LOG_ROOT:-/var/lib/docker/containers}:/var/lib/docker/containers:ro
+    depends_on:
+      loki: {condition: service_healthy}
+    networks: [backend]
+    restart: unless-stopped
+
+volumes:
+  prometheus-data:
+  loki-data:
+  grafana-data:


## `build(observability): provision internal datasources`

diff --git a/observability/grafana/provisioning/datasources/datasources.yml b/observability/grafana/provisioning/datasources/datasources.yml
new file mode 100644
index 0000000..7b849c0
--- /dev/null
+++ b/observability/grafana/provisioning/datasources/datasources.yml
@@ -0,0 +1,27 @@
+apiVersion: 1
+
+deleteDatasources:
+  - name: Prometheus
+    orgId: 1
+  - name: Loki
+    orgId: 1
+
+datasources:
+  - name: Prometheus
+    uid: prometheus
+    type: prometheus
+    access: proxy
+    url: http://prometheus:9090
+    isDefault: true
+    editable: false
+    jsonData:
+      timeInterval: 5s
+
+  - name: Loki
+    uid: loki
+    type: loki
+    access: proxy
+    url: http://loki:3100
+    editable: false
+    jsonData:
+      maxLines: 1000


## `test(observability): verify internal datasource origins`

diff --git a/tests/test_grafana_datasources.py b/tests/test_grafana_datasources.py
new file mode 100644
index 0000000..2b2a5f3
--- /dev/null
+++ b/tests/test_grafana_datasources.py
@@ -0,0 +1,25 @@
+import pathlib
+import unittest
+
+
+ROOT = pathlib.Path(__file__).resolve().parents[1]
+CONFIG = ROOT / "observability/grafana/provisioning/datasources/datasources.yml"
+
+
+class GrafanaDatasourceTest(unittest.TestCase):
+    def test_provisions_only_internal_prometheus_and_loki_origins(self) -> None:
+        config = CONFIG.read_text()
+
+        self.assertEqual(config.count("  - name: Prometheus\n"), 2)
+        self.assertEqual(config.count("  - name: Loki\n"), 2)
+        self.assertEqual(config.count("uid: prometheus"), 1)
+        self.assertEqual(config.count("uid: loki"), 1)
+        self.assertIn("url: http://prometheus:9090", config)
+        self.assertIn("url: http://loki:3100", config)
+        self.assertEqual(config.count("isDefault: true"), 1)
+        self.assertNotIn("localhost", config)
+        self.assertNotIn("password", config.lower())
+
+
+if __name__ == "__main__":
+    unittest.main()


## `build(logs): scope collection to the active project`

diff --git a/observability/promtail/promtail.yml b/observability/promtail/promtail.yml
new file mode 100644
index 0000000..7afb1f9
--- /dev/null
+++ b/observability/promtail/promtail.yml
@@ -0,0 +1,39 @@
+server:
+  http_listen_port: 9080
+  grpc_listen_port: 0
+  log_level: warn
+
+positions:
+  filename: /tmp/positions.yml
+
+clients:
+  - url: http://loki:3100/loki/api/v1/push
+
+scrape_configs:
+  - job_name: project-containers
+    docker_sd_configs:
+      - host: unix:///var/run/docker.sock
+        refresh_interval: 5s
+    relabel_configs:
+      - source_labels: ["__meta_docker_container_label_com_docker_compose_project"]
+        regex: ${COMPOSE_PROJECT_NAME}
+        action: keep
+      - source_labels: ["__meta_docker_container_label_com_docker_compose_project"]
+        target_label: project
+      - source_labels: ["__meta_docker_container_label_com_docker_compose_service"]
+        target_label: service
+      - source_labels: ["__meta_docker_container_name"]
+        regex: "/(.*)"
+        target_label: container
+      - source_labels: ["__meta_docker_container_id"]
+        target_label: __path__
+        replacement: /var/lib/docker/containers/$1/*-json.log
+    pipeline_stages:
+      - docker: {}
+      - json:
+          expressions:
+            level: level
+            trace_id: traceId
+            span_id: spanId
+      - labels:
+          level:


## `test(logs): verify dynamic project scoping`

diff --git a/tests/test_promtail_config.py b/tests/test_promtail_config.py
new file mode 100644
index 0000000..e94045b
--- /dev/null
+++ b/tests/test_promtail_config.py
@@ -0,0 +1,44 @@
+import pathlib
+import subprocess
+import unittest
+
+
+ROOT = pathlib.Path(__file__).resolve().parents[1]
+CONFIG = ROOT / "observability/promtail/promtail.yml"
+
+
+class PromtailConfigTest(unittest.TestCase):
+    def test_validates_dynamic_project_scoping_without_label_overwrite(self) -> None:
+        checked = subprocess.run(
+            [
+                "docker",
+                "run",
+                "--rm",
+                "--network",
+                "none",
+                "--env",
+                "COMPOSE_PROJECT_NAME=sportsbook-contract",
+                "--volume",
+                f"{CONFIG}:/etc/promtail/promtail.yml:ro",
+                "grafana/promtail:3.1.1",
+                "-check-syntax",
+                "-config.file=/etc/promtail/promtail.yml",
+                "-config.expand-env=true",
+            ],
+            text=True,
+            capture_output=True,
+            check=False,
+        )
+        self.assertEqual(checked.returncode, 0, checked.stderr)
+
+        config = CONFIG.read_text()
+        self.assertIn("regex: ${COMPOSE_PROJECT_NAME}", config)
+        self.assertIn("target_label: project", config)
+        self.assertIn("target_label: service", config)
+        self.assertIn("/var/lib/docker/containers/$1/*-json.log", config)
+        self.assertNotIn("regex: sportsbook", config)
+        self.assertNotIn("- template:", config)
+
+
+if __name__ == "__main__":
+    unittest.main()


## `build(logs): filter Docker discovery by project`

diff --git a/compose.observability.yaml b/compose.observability.yaml
index 11b35bf..41e1585 100644
--- a/compose.observability.yaml
+++ b/compose.observability.yaml
@@ -62,7 +62,6 @@ services:
     volumes:
       - ./observability/promtail/promtail.yml:/etc/promtail/promtail.yml:ro
       - ${DOCKER_SOCKET_PATH:-/var/run/docker.sock}:/var/run/docker.sock:ro
-      - ${DOCKER_LOG_ROOT:-/var/lib/docker/containers}:/var/lib/docker/containers:ro
     depends_on:
       loki: {condition: service_healthy}
     healthcheck:
diff --git a/observability/promtail/promtail.yml b/observability/promtail/promtail.yml
index 7afb1f9..643db83 100644
--- a/observability/promtail/promtail.yml
+++ b/observability/promtail/promtail.yml
@@ -14,6 +14,10 @@ scrape_configs:
     docker_sd_configs:
       - host: unix:///var/run/docker.sock
         refresh_interval: 5s
+        filters:
+          - name: label
+            values:
+              - com.docker.compose.project=${COMPOSE_PROJECT_NAME}
     relabel_configs:
       - source_labels: ["__meta_docker_container_label_com_docker_compose_project"]
         regex: ${COMPOSE_PROJECT_NAME}
@@ -25,9 +29,6 @@ scrape_configs:
       - source_labels: ["__meta_docker_container_name"]
         regex: "/(.*)"
         target_label: container
-      - source_labels: ["__meta_docker_container_id"]
-        target_label: __path__
-        replacement: /var/lib/docker/containers/$1/*-json.log
     pipeline_stages:
       - docker: {}
       - json:


## `test(logs): verify project scoped Docker discovery`

diff --git a/tests/test_combined_compose.py b/tests/test_combined_compose.py
index 15e57e0..93b9a93 100644
--- a/tests/test_combined_compose.py
+++ b/tests/test_combined_compose.py
@@ -78,6 +78,12 @@ class CombinedComposeTest(ComposeContractFixture):
             services["promtail"]["healthcheck"]["test"],
             ["CMD", "wget", "--quiet", "--spider", "http://localhost:9080/ready"],
         )
+        promtail_mounts = services["promtail"]["volumes"]
+        self.assertEqual(
+            {mount["target"] for mount in promtail_mounts},
+            {"/etc/promtail/promtail.yml", "/var/run/docker.sock"},
+        )
+        self.assertTrue(all(mount["read_only"] for mount in promtail_mounts))
 
         self.assertEqual(
             set(rendered["volumes"]),
diff --git a/tests/test_promtail_config.py b/tests/test_promtail_config.py
index e94045b..1c90843 100644
--- a/tests/test_promtail_config.py
+++ b/tests/test_promtail_config.py
@@ -35,7 +35,11 @@ class PromtailConfigTest(unittest.TestCase):
         self.assertIn("regex: ${COMPOSE_PROJECT_NAME}", config)
         self.assertIn("target_label: project", config)
         self.assertIn("target_label: service", config)
-        self.assertIn("/var/lib/docker/containers/$1/*-json.log", config)
+        self.assertIn(
+            "com.docker.compose.project=${COMPOSE_PROJECT_NAME}", config
+        )
+        self.assertNotIn("target_label: __path__", config)
+        self.assertNotIn("/var/lib/docker/containers", config)
         self.assertNotIn("regex: sportsbook", config)
         self.assertNotIn("- template:", config)
 


## `build(observability): isolate project endpoints`

diff --git a/compose.observability.yaml b/compose.observability.yaml
index 55856bf..cd6746b 100644
--- a/compose.observability.yaml
+++ b/compose.observability.yaml
@@ -7,8 +7,6 @@ services:
     volumes:
       - ./observability/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
       - prometheus-data:/prometheus
-    ports:
-      - {target: 9090, published: "${PROMETHEUS_HOST_PORT:-9090}", host_ip: 127.0.0.1}
     healthcheck:
       test: ["CMD", "promtool", "query", "instant", "http://localhost:9090", "up"]
       interval: 5s
@@ -23,8 +21,6 @@ services:
     volumes:
       - ./observability/loki/loki.yml:/etc/loki/loki.yml:ro
       - loki-data:/loki
-    ports:
-      - {target: 3100, published: "${LOKI_HOST_PORT:-3100}", host_ip: 127.0.0.1}
     healthcheck:
       test: ["CMD", "/usr/bin/loki", "-version"]
       interval: 5s
@@ -44,7 +40,7 @@ services:
       - ./observability/grafana/provisioning:/etc/grafana/provisioning:ro
       - grafana-data:/var/lib/grafana
     ports:
-      - {target: 3000, published: "${GRAFANA_HOST_PORT:-3000}", host_ip: 127.0.0.1}
+      - {target: 3000, host_ip: 127.0.0.1}
     healthcheck:
       test: ["CMD-SHELL", "wget --quiet --spider http://localhost:3000/api/health"]
       interval: 5s
@@ -62,7 +58,7 @@ services:
       - -config.file=/etc/promtail/promtail.yml
       - -config.expand-env=true
     environment:
-      COMPOSE_PROJECT_NAME: ${COMPOSE_PROJECT_NAME:-sportsbook}
+      COMPOSE_PROJECT_NAME: ${COMPOSE_PROJECT_NAME}
     volumes:
       - ./observability/promtail/promtail.yml:/etc/promtail/promtail.yml:ro
       - ${DOCKER_SOCKET_PATH:-/var/run/docker.sock}:/var/run/docker.sock:ro


## `test(observability): verify project local endpoints`

diff --git a/tests/test_combined_compose.py b/tests/test_combined_compose.py
index 144014d..e57ad4f 100644
--- a/tests/test_combined_compose.py
+++ b/tests/test_combined_compose.py
@@ -13,12 +13,7 @@ class CombinedComposeTest(ComposeContractFixture):
         self.environment.update(
             {
                 "GATEWAY_HOST_PORT": "18080",
-                "TOXIPROXY_HOST_PORT": "18474",
-                "PROMETHEUS_HOST_PORT": "19090",
-                "GRAFANA_HOST_PORT": "13000",
-                "LOKI_HOST_PORT": "13100",
                 "GRAFANA_ADMIN_PASSWORD": "grafana-contract-password",
-                "COMPOSE_PROJECT_NAME": self.project,
             }
         )
         result = subprocess.run(
@@ -61,11 +56,16 @@ class CombinedComposeTest(ComposeContractFixture):
             for name, service in services.items()
             if service.get("ports")
         }
-        self.assertEqual(
-            set(published), {"gateway", "toxiproxy", "prometheus", "grafana", "loki"}
-        )
+        self.assertEqual(set(published), {"gateway", "toxiproxy", "grafana"})
         for ports in published.values():
             self.assertTrue(all(port["host_ip"] == "127.0.0.1" for port in ports))
+        self.assertEqual(published["gateway"][0]["published"], "18080")
+        self.assertNotIn("published", published["toxiproxy"][0])
+        self.assertEqual(published["toxiproxy"][0]["target"], 8474)
+        self.assertNotIn("published", published["grafana"][0])
+        self.assertEqual(published["grafana"][0]["target"], 3000)
+        for name in ("prometheus", "loki", "promtail"):
+            self.assertNotIn("ports", services[name])
 
         self.assertEqual(
             set(rendered["volumes"]),
@@ -81,12 +81,15 @@ class CombinedComposeTest(ComposeContractFixture):
                 "grafana-data",
             },
         )
+        for logical, volume in rendered["volumes"].items():
+            self.assertEqual(volume["name"], f"{self.project}_{logical}")
         source = "\n".join(
             path.read_text()
             for path in (ROOT / "compose.toxiproxy.yaml", ROOT / "compose.observability.yaml")
         )
         self.assertNotIn("sportsbook_default", source)
         self.assertNotIn("external: true", source)
+        self.assertNotIn("${COMPOSE_PROJECT_NAME:-sportsbook}", source)
 
 
 if __name__ == "__main__":


