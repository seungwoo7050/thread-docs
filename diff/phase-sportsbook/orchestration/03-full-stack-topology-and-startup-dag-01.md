# 풀스택 토폴로지와 기동 DAG

## `build(runtime): provide Java 17 service image`

diff --git a/docker/Dockerfile.jvm b/docker/Dockerfile.jvm
new file mode 100644
index 0000000..196e285
--- /dev/null
+++ b/docker/Dockerfile.jvm
@@ -0,0 +1,12 @@
+FROM eclipse-temurin:17-jre-jammy
+
+RUN apt-get update \
+    && apt-get install --yes --no-install-recommends curl \
+    && rm -rf /var/lib/apt/lists/*
+
+ARG JAR
+WORKDIR /app
+COPY jars/${JAR} /app/app.jar
+
+USER 10001:10001
+ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75.0", "-jar", "/app/app.jar"]


## `test(runtime): verify Java class and health tools`

diff --git a/tests/test_runtime_image.py b/tests/test_runtime_image.py
new file mode 100644
index 0000000..9dcbefb
--- /dev/null
+++ b/tests/test_runtime_image.py
@@ -0,0 +1,74 @@
+import pathlib
+import shutil
+import subprocess
+import tempfile
+import unittest
+import uuid
+import zipfile
+
+
+ROOT = pathlib.Path(__file__).resolve().parents[1]
+DOCKERFILE = ROOT / "docker/Dockerfile.jvm"
+
+
+class RuntimeImageTest(unittest.TestCase):
+    def test_runs_java_17_and_provides_the_health_tool(self) -> None:
+        image = f"sportsbook-runtime-contract:{uuid.uuid4().hex}"
+        self.addCleanup(
+            subprocess.run,
+            ["docker", "image", "rm", "--force", image],
+            check=False,
+            stdout=subprocess.DEVNULL,
+            stderr=subprocess.DEVNULL,
+        )
+
+        with tempfile.TemporaryDirectory() as temporary:
+            context = pathlib.Path(temporary)
+            shutil.copy2(DOCKERFILE, context / "Dockerfile")
+            jars = context / "jars"
+            jars.mkdir()
+            with zipfile.ZipFile(jars / "probe.jar", "w") as archive:
+                archive.writestr("BOOT-INF/classes/Probe.class", b"probe")
+
+            built = subprocess.run(
+                [
+                    "docker",
+                    "build",
+                    "--quiet",
+                    "--build-arg",
+                    "JAR=probe.jar",
+                    "--tag",
+                    image,
+                    ".",
+                ],
+                cwd=context,
+                text=True,
+                capture_output=True,
+                check=False,
+            )
+            self.assertEqual(built.returncode, 0, built.stderr)
+
+        java = self.run_in_image(image, "java", "-XshowSettings:properties", "-version")
+        self.assertEqual(java.returncode, 0, java.stderr)
+        self.assertIn("java.class.version = 61.0", java.stderr)
+
+        curl = self.run_in_image(image, "curl", "--version")
+        self.assertEqual(curl.returncode, 0, curl.stderr)
+        self.assertTrue(curl.stdout.startswith("curl "))
+
+        user = self.run_in_image(image, "id", "-u")
+        self.assertEqual(user.returncode, 0, user.stderr)
+        self.assertEqual(user.stdout.strip(), "10001")
+
+    @staticmethod
+    def run_in_image(image: str, *command: str) -> subprocess.CompletedProcess[str]:
+        return subprocess.run(
+            ["docker", "run", "--rm", "--entrypoint", command[0], image, *command[1:]],
+            text=True,
+            capture_output=True,
+            check=False,
+        )
+
+
+if __name__ == "__main__":
+    unittest.main()


## `build(wallet): wire durable authenticated runtime`

diff --git a/compose.yaml b/compose.yaml
index 6d8a4e8..417e0b1 100644
--- a/compose.yaml
+++ b/compose.yaml
@@ -96,6 +96,51 @@ services:
     volumes:
       - redis-gateway-data:/data
 
+  wallet:
+    build:
+      context: ./docker
+      dockerfile: Dockerfile.jvm
+      args:
+        JAR: wallet.jar
+    environment:
+      WALLET_DB_URL: jdbc:postgresql://postgres:5432/wallet
+      WALLET_DB_USER: sportsbook
+      WALLET_DB_PASSWORD: ${POSTGRES_PASSWORD:-}
+      WALLET_KAFKA_BOOTSTRAP: kafka:9092
+      WALLET_REDIS_HOST: redis-wallet
+      WALLET_REDIS_PORT: "6379"
+      WALLET_HTTP_PORT: "8081"
+      WALLET_PLATFORM_API_KEY: ${WALLET_PLATFORM_API_KEY:-}
+      WALLET_GATEWAY_API_KEY: ${GATEWAY_WALLET_API_KEY:-}
+      WALLET_BETTING_SERVICE_API_KEY: ${BETTING_WALLET_API_KEY:-}
+      WALLET_SETTLEMENT_SERVICE_API_KEY: ${SETTLEMENT_WALLET_API_KEY:-}
+      WALLET_ADMIN_API_KEY: ${ADMIN_WALLET_API_KEY:-}
+      WALLET_OUTBOX_ENABLED: "true"
+      WALLET_INTEGRITY_ENABLED: "true"
+      WALLET_INTEGRITY_POLL_INTERVAL: PT1S
+      WALLET_RECOVERY_ENABLED: "true"
+      WALLET_RECOVERY_POLL_INTERVAL: PT1S
+      WALLET_RECOVERY_RETRY_BASE: PT1S
+      WALLET_RECOVERY_RETRY_CAP: PT5S
+    depends_on:
+      postgres:
+        condition: service_healthy
+      kafka:
+        condition: service_healthy
+      redis-wallet:
+        condition: service_healthy
+      topic-init:
+        condition: service_completed_successfully
+    healthcheck:
+      test: ["CMD-SHELL", "curl --fail --silent --show-error http://localhost:8081/actuator/health >/dev/null"]
+      interval: 3s
+      timeout: 3s
+      retries: 60
+      start_period: 30s
+    expose: ["8081"]
+    networks: [backend]
+    restart: unless-stopped
+
 volumes:
   postgres-data:
   kafka-data:


## `test(wallet): verify environment outbox and readiness`

diff --git a/tests/compose_contract_fixture.py b/tests/compose_contract_fixture.py
new file mode 100644
index 0000000..09586aa
--- /dev/null
+++ b/tests/compose_contract_fixture.py
@@ -0,0 +1,36 @@
+import json
+
+from tests.compose_fixture import ComposeFixture
+from tests.secret_fixture import VALID
+
+
+PUBLIC_KEY = "-----BEGIN PUBLIC KEY-----\ncontract-public-key\n-----END PUBLIC KEY-----"
+
+
+class ComposeContractFixture(ComposeFixture):
+    def setUp(self) -> None:
+        super().setUp()
+        self.environment.update(VALID)
+        self.environment["GATEWAY_JWT_PUBLIC_KEY"] = PUBLIC_KEY
+        self.environment["ADMIN_JWT_PUBLIC_KEY"] = PUBLIC_KEY
+        self.environment["ADMIN_JWT_ISSUER"] = "sportsbook-admin-e2e"
+
+    def rendered(self) -> dict:
+        result = self.compose("config", "--format", "json")
+        self.assertEqual(result.returncode, 0, result.stderr)
+        return json.loads(result.stdout)
+
+    def service(self, name: str) -> dict:
+        return self.rendered()["services"][name]
+
+    def assert_runtime_build(self, service: dict, jar: str) -> None:
+        self.assertTrue(service["build"]["context"].endswith("/docker"))
+        self.assertEqual(service["build"]["dockerfile"], "Dockerfile.jvm")
+        self.assertEqual(service["build"]["args"], {"JAR": jar})
+
+    def assert_dependency_conditions(self, service: dict, expected: dict[str, str]) -> None:
+        actual = {
+            name: dependency["condition"]
+            for name, dependency in service["depends_on"].items()
+        }
+        self.assertEqual(actual, expected)
diff --git a/tests/test_wallet_wiring.py b/tests/test_wallet_wiring.py
new file mode 100644
index 0000000..2221b6b
--- /dev/null
+++ b/tests/test_wallet_wiring.py
@@ -0,0 +1,53 @@
+from tests.compose_contract_fixture import ComposeContractFixture
+from tests.secret_fixture import VALID
+
+
+class WalletWiringTest(ComposeContractFixture):
+    def test_wires_canonical_environment_outbox_and_health(self) -> None:
+        wallet = self.service("wallet")
+        environment = wallet["environment"]
+
+        self.assert_runtime_build(wallet, "wallet.jar")
+        self.assertEqual(environment["WALLET_DB_URL"], "jdbc:postgresql://postgres:5432/wallet")
+        self.assertEqual(environment["WALLET_REDIS_HOST"], "redis-wallet")
+        self.assertEqual(environment["WALLET_KAFKA_BOOTSTRAP"], "kafka:9092")
+        self.assertEqual(environment["WALLET_OUTBOX_ENABLED"], "true")
+        self.assertEqual(environment["WALLET_INTEGRITY_ENABLED"], "true")
+        self.assertEqual(environment["WALLET_RECOVERY_ENABLED"], "true")
+        self.assertEqual(
+            {
+                name: value
+                for name, value in environment.items()
+                if name.endswith("API_KEY")
+            },
+            {
+                "WALLET_PLATFORM_API_KEY": VALID["WALLET_PLATFORM_API_KEY"],
+                "WALLET_GATEWAY_API_KEY": VALID["GATEWAY_WALLET_API_KEY"],
+                "WALLET_BETTING_SERVICE_API_KEY": VALID["BETTING_WALLET_API_KEY"],
+                "WALLET_SETTLEMENT_SERVICE_API_KEY": VALID["SETTLEMENT_WALLET_API_KEY"],
+                "WALLET_ADMIN_API_KEY": VALID["ADMIN_WALLET_API_KEY"],
+            },
+        )
+        self.assertEqual(
+            wallet["healthcheck"]["test"],
+            [
+                "CMD-SHELL",
+                "curl --fail --silent --show-error "
+                "http://localhost:8081/actuator/health >/dev/null",
+            ],
+        )
+        self.assert_dependency_conditions(
+            wallet,
+            {
+                "postgres": "service_healthy",
+                "kafka": "service_healthy",
+                "redis-wallet": "service_healthy",
+                "topic-init": "service_completed_successfully",
+            },
+        )
+
+
+if __name__ == "__main__":
+    import unittest
+
+    unittest.main()


## `build(risk): wire authenticated Redis runtime`

diff --git a/compose.yaml b/compose.yaml
index 417e0b1..b3b8da5 100644
--- a/compose.yaml
+++ b/compose.yaml
@@ -141,6 +141,37 @@ services:
     networks: [backend]
     restart: unless-stopped
 
+  risk:
+    build:
+      context: ./docker
+      dockerfile: Dockerfile.jvm
+      args:
+        JAR: risk.jar
+    environment:
+      SERVER_PORT: "8083"
+      REDIS_HOST: redis-risk
+      REDIS_PORT: "6379"
+      KAFKA_BOOTSTRAP: kafka:9092
+      INTERNAL_BETTING_SERVICE_API_KEY: ${BETTING_RISK_API_KEY:-}
+      INTERNAL_ADMIN_API_KEY: ${ADMIN_RISK_API_KEY:-}
+      INTERNAL_PLATFORM_API_KEY: ${INTERNAL_PLATFORM_API_KEY:-}
+    depends_on:
+      kafka:
+        condition: service_healthy
+      redis-risk:
+        condition: service_healthy
+      topic-init:
+        condition: service_completed_successfully
+    healthcheck:
+      test: ["CMD-SHELL", "curl --fail --silent --show-error http://localhost:8083/actuator/health/readiness >/dev/null"]
+      interval: 3s
+      timeout: 3s
+      retries: 60
+      start_period: 30s
+    expose: ["8083"]
+    networks: [backend]
+    restart: unless-stopped
+
 volumes:
   postgres-data:
   kafka-data:


## `test(risk): verify auth Redis and Kafka wiring`

diff --git a/tests/test_risk_wiring.py b/tests/test_risk_wiring.py
new file mode 100644
index 0000000..7f10207
--- /dev/null
+++ b/tests/test_risk_wiring.py
@@ -0,0 +1,46 @@
+from tests.compose_contract_fixture import ComposeContractFixture
+from tests.secret_fixture import VALID
+
+
+class RiskWiringTest(ComposeContractFixture):
+    def test_wires_authentication_redis_kafka_and_readiness(self) -> None:
+        risk = self.service("risk")
+        environment = risk["environment"]
+
+        self.assert_runtime_build(risk, "risk.jar")
+        self.assertEqual(environment["REDIS_HOST"], "redis-risk")
+        self.assertEqual(environment["KAFKA_BOOTSTRAP"], "kafka:9092")
+        self.assertEqual(
+            {
+                name: value
+                for name, value in environment.items()
+                if name.endswith("API_KEY")
+            },
+            {
+                "INTERNAL_BETTING_SERVICE_API_KEY": VALID["BETTING_RISK_API_KEY"],
+                "INTERNAL_ADMIN_API_KEY": VALID["ADMIN_RISK_API_KEY"],
+                "INTERNAL_PLATFORM_API_KEY": VALID["INTERNAL_PLATFORM_API_KEY"],
+            },
+        )
+        self.assertEqual(
+            risk["healthcheck"]["test"],
+            [
+                "CMD-SHELL",
+                "curl --fail --silent --show-error "
+                "http://localhost:8083/actuator/health/readiness >/dev/null",
+            ],
+        )
+        self.assert_dependency_conditions(
+            risk,
+            {
+                "kafka": "service_healthy",
+                "redis-risk": "service_healthy",
+                "topic-init": "service_completed_successfully",
+            },
+        )
+
+
+if __name__ == "__main__":
+    import unittest
+
+    unittest.main()


## `build(odds): wire critical delivery runtime`

diff --git a/compose.yaml b/compose.yaml
index b3b8da5..d272512 100644
--- a/compose.yaml
+++ b/compose.yaml
@@ -172,6 +172,39 @@ services:
     networks: [backend]
     restart: unless-stopped
 
+  odds:
+    build:
+      context: ./docker
+      dockerfile: Dockerfile.jvm
+      args:
+        JAR: odds.jar
+    environment:
+      SERVER_PORT: "8085"
+      SPRING_PROFILES_ACTIVE: mock
+      REDIS_HOST: redis-odds
+      REDIS_PORT: "6379"
+      KAFKA_BOOTSTRAP_SERVERS: kafka:9092
+      ADMIN_API_INTERNAL_KEY: ${ADMIN_ODDS_FEED_API_KEY:-}
+      HOSTNAME: odds
+      ODDSFEED_MOCK_MINUTES_PER_SECOND: "0.01"
+      ODDSFEED_MOCK_SCENARIOS_AUTO_ROTATE: "false"
+    depends_on:
+      kafka:
+        condition: service_healthy
+      redis-odds:
+        condition: service_healthy
+      topic-init:
+        condition: service_completed_successfully
+    healthcheck:
+      test: ["CMD-SHELL", "curl --fail --silent --show-error http://localhost:8085/actuator/health/readiness >/dev/null"]
+      interval: 3s
+      timeout: 3s
+      retries: 60
+      start_period: 30s
+    expose: ["8085"]
+    networks: [backend]
+    restart: unless-stopped
+
 volumes:
   postgres-data:
   kafka-data:


## `test(odds): verify auth and critical readiness`

diff --git a/tests/test_odds_wiring.py b/tests/test_odds_wiring.py
new file mode 100644
index 0000000..9156138
--- /dev/null
+++ b/tests/test_odds_wiring.py
@@ -0,0 +1,39 @@
+from tests.compose_contract_fixture import ComposeContractFixture
+from tests.secret_fixture import VALID
+
+
+class OddsWiringTest(ComposeContractFixture):
+    def test_wires_admin_auth_projection_store_and_critical_readiness(self) -> None:
+        odds = self.service("odds")
+        environment = odds["environment"]
+
+        self.assert_runtime_build(odds, "odds.jar")
+        self.assertEqual(environment["SPRING_PROFILES_ACTIVE"], "mock")
+        self.assertEqual(environment["REDIS_HOST"], "redis-odds")
+        self.assertEqual(environment["KAFKA_BOOTSTRAP_SERVERS"], "kafka:9092")
+        self.assertEqual(
+            environment["ADMIN_API_INTERNAL_KEY"], VALID["ADMIN_ODDS_FEED_API_KEY"]
+        )
+        self.assertEqual(environment["ODDSFEED_MOCK_SCENARIOS_AUTO_ROTATE"], "false")
+        self.assertEqual(
+            odds["healthcheck"]["test"],
+            [
+                "CMD-SHELL",
+                "curl --fail --silent --show-error "
+                "http://localhost:8085/actuator/health/readiness >/dev/null",
+            ],
+        )
+        self.assert_dependency_conditions(
+            odds,
+            {
+                "kafka": "service_healthy",
+                "redis-odds": "service_healthy",
+                "topic-init": "service_completed_successfully",
+            },
+        )
+
+
+if __name__ == "__main__":
+    import unittest
+
+    unittest.main()


## `build(betting): wire authenticated dependencies`

diff --git a/compose.yaml b/compose.yaml
index d272512..4ff42ec 100644
--- a/compose.yaml
+++ b/compose.yaml
@@ -205,6 +205,52 @@ services:
     networks: [backend]
     restart: unless-stopped
 
+  betting:
+    build:
+      context: ./docker
+      dockerfile: Dockerfile.jvm
+      args:
+        JAR: betting.jar
+    environment:
+      BETTING_DB_URL: jdbc:postgresql://postgres:5432/betting
+      BETTING_DB_USER: sportsbook
+      BETTING_DB_PASSWORD: ${POSTGRES_PASSWORD:-}
+      BETTING_KAFKA_BOOTSTRAP: kafka:9092
+      BETTING_REDIS_HOST: redis-odds
+      BETTING_REDIS_PORT: "6379"
+      BETTING_HTTP_PORT: "8082"
+      RISK_BASE_URL: http://risk:8083
+      WALLET_BASE_URL: http://wallet:8081
+      BETTING_GATEWAY_API_KEY: ${GATEWAY_BETTING_API_KEY:-}
+      BETTING_RISK_API_KEY: ${BETTING_RISK_API_KEY:-}
+      BETTING_WALLET_API_KEY: ${BETTING_WALLET_API_KEY:-}
+      BETTING_RECONCILIATION_POLL_INTERVAL_MS: "500"
+      BETTING_RECONCILIATION_PENDING_TIMEOUT: PT1S
+    depends_on:
+      postgres:
+        condition: service_healthy
+      kafka:
+        condition: service_healthy
+      redis-odds:
+        condition: service_healthy
+      topic-init:
+        condition: service_completed_successfully
+      wallet:
+        condition: service_healthy
+      risk:
+        condition: service_healthy
+      odds:
+        condition: service_healthy
+    healthcheck:
+      test: ["CMD-SHELL", "curl --fail --silent --show-error http://localhost:8082/actuator/health >/dev/null"]
+      interval: 3s
+      timeout: 3s
+      retries: 60
+      start_period: 30s
+    expose: ["8082"]
+    networks: [backend]
+    restart: unless-stopped
+
 volumes:
   postgres-data:
   kafka-data:


## `test(betting): verify auth dependencies and topics`

diff --git a/tests/test_betting_wiring.py b/tests/test_betting_wiring.py
new file mode 100644
index 0000000..2c3bc96
--- /dev/null
+++ b/tests/test_betting_wiring.py
@@ -0,0 +1,51 @@
+from tests.compose_contract_fixture import ComposeContractFixture
+from tests.secret_fixture import VALID
+
+
+class BettingWiringTest(ComposeContractFixture):
+    def test_wires_auth_dependencies_and_odds_projection_topics(self) -> None:
+        rendered = self.rendered()
+        betting = rendered["services"]["betting"]
+        environment = betting["environment"]
+
+        self.assert_runtime_build(betting, "betting.jar")
+        self.assertEqual(environment["BETTING_DB_URL"], "jdbc:postgresql://postgres:5432/betting")
+        self.assertEqual(environment["BETTING_KAFKA_BOOTSTRAP"], "kafka:9092")
+        self.assertEqual(environment["BETTING_REDIS_HOST"], "redis-odds")
+        self.assertEqual(
+            environment["BETTING_REDIS_HOST"],
+            rendered["services"]["odds"]["environment"]["REDIS_HOST"],
+        )
+        self.assertEqual(environment["RISK_BASE_URL"], "http://risk:8083")
+        self.assertEqual(environment["WALLET_BASE_URL"], "http://wallet:8081")
+        self.assertEqual(
+            {
+                name: value
+                for name, value in environment.items()
+                if name.endswith("API_KEY")
+            },
+            {
+                "BETTING_GATEWAY_API_KEY": VALID["GATEWAY_BETTING_API_KEY"],
+                "BETTING_RISK_API_KEY": VALID["BETTING_RISK_API_KEY"],
+                "BETTING_WALLET_API_KEY": VALID["BETTING_WALLET_API_KEY"],
+            },
+        )
+        self.assert_dependency_conditions(
+            betting,
+            {
+                "postgres": "service_healthy",
+                "kafka": "service_healthy",
+                "redis-odds": "service_healthy",
+                "topic-init": "service_completed_successfully",
+                "wallet": "service_healthy",
+                "risk": "service_healthy",
+                "odds": "service_healthy",
+            },
+        )
+        self.assertIn("/actuator/health >/dev/null", betting["healthcheck"]["test"][1])
+
+
+if __name__ == "__main__":
+    import unittest
+
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


