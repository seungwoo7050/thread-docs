## `build(settlement): wire canonical database runtime`

diff --git a/compose.yaml b/compose.yaml
index 33488d5..1fed522 100644
--- a/compose.yaml
+++ b/compose.yaml
@@ -299,6 +299,51 @@ services:
     networks: [backend]
     restart: unless-stopped
 
+  settlement:
+    build:
+      context: ./docker
+      dockerfile: Dockerfile.jvm
+      args:
+        JAR: settlement.jar
+    environment:
+      SETTLEMENT_HTTP_PORT: "8084"
+      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/settlement
+      SPRING_DATASOURCE_USERNAME: sportsbook
+      SPRING_DATASOURCE_PASSWORD: ${POSTGRES_PASSWORD:-}
+      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
+      SETTLEMENT_ADMIN_API_KEY: ${ADMIN_SETTLEMENT_API_KEY:-}
+      SETTLEMENT_WALLET_API_KEY: ${SETTLEMENT_WALLET_API_KEY:-}
+      SETTLEMENT_WALLET_BASE_URL: http://wallet:8081
+      SETTLEMENT_WORKERS_ENABLED: "true"
+      SETTLEMENT_RECOVERY_INTERVAL: PT1S
+      SETTLEMENT_LEASE_DURATION: PT10S
+      SETTLEMENT_BATCH_SIZE: "100"
+      SETTLEMENT_OUTBOX_INTERVAL: PT500MS
+      SETTLEMENT_TOPIC_BET_PLACED: bet.placed.v1
+      SETTLEMENT_TOPIC_MATCH_RESULT: match.result
+      SETTLEMENT_TOPIC_EVENT_LIFECYCLE: event.lifecycle
+      SETTLEMENT_TOPIC_BET_SETTLED: bet.settled.v1
+      SETTLEMENT_TOPIC_BET_VOIDED: bet.voided.v1
+      SETTLEMENT_TOPIC_BET_REVISED: bet.resolution.revised.v1
+    depends_on:
+      postgres:
+        condition: service_healthy
+      kafka:
+        condition: service_healthy
+      topic-init:
+        condition: service_completed_successfully
+      wallet:
+        condition: service_healthy
+    healthcheck:
+      test: ["CMD-SHELL", "curl --fail --silent --show-error http://localhost:8084/actuator/health/readiness >/dev/null"]
+      interval: 3s
+      timeout: 3s
+      retries: 60
+      start_period: 30s
+    expose: ["8084"]
+    networks: [backend]
+    restart: unless-stopped
+
 volumes:
   postgres-data:
   kafka-data:


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


## `build(startup): gate the service dependency DAG`

diff --git a/compose.yaml b/compose.yaml
index bd24633..0438c90 100644
--- a/compose.yaml
+++ b/compose.yaml
@@ -74,8 +74,38 @@ services:
     depends_on:
       kafka:
         condition: service_healthy
+      postgres:
+        condition: service_healthy
+      redis-risk:
+        condition: service_healthy
+      redis-odds:
+        condition: service_healthy
+      redis-wallet:
+        condition: service_healthy
+      redis-gateway:
+        condition: service_healthy
     networks: [backend]
 
+  secret-preflight:
+    image: python:3.12-alpine
+    entrypoint: ["python3", "/opt/sportsbook/check-secrets.py"]
+    environment:
+      GATEWAY_BETTING_API_KEY: ${GATEWAY_BETTING_API_KEY:-}
+      GATEWAY_WALLET_API_KEY: ${GATEWAY_WALLET_API_KEY:-}
+      BETTING_RISK_API_KEY: ${BETTING_RISK_API_KEY:-}
+      BETTING_WALLET_API_KEY: ${BETTING_WALLET_API_KEY:-}
+      SETTLEMENT_WALLET_API_KEY: ${SETTLEMENT_WALLET_API_KEY:-}
+      ADMIN_WALLET_API_KEY: ${ADMIN_WALLET_API_KEY:-}
+      ADMIN_RISK_API_KEY: ${ADMIN_RISK_API_KEY:-}
+      ADMIN_ODDS_FEED_API_KEY: ${ADMIN_ODDS_FEED_API_KEY:-}
+      ADMIN_SETTLEMENT_API_KEY: ${ADMIN_SETTLEMENT_API_KEY:-}
+      WALLET_PLATFORM_API_KEY: ${WALLET_PLATFORM_API_KEY:-}
+      INTERNAL_PLATFORM_API_KEY: ${INTERNAL_PLATFORM_API_KEY:-}
+    volumes:
+      - ./scripts/check-secrets.py:/opt/sportsbook/check-secrets.py:ro
+      - ./config/required-secrets.txt:/opt/config/required-secrets.txt:ro
+    network_mode: none
+
   redis-risk:
     <<: *redis-service
     volumes:
@@ -131,6 +161,8 @@ services:
         condition: service_healthy
       topic-init:
         condition: service_completed_successfully
+      secret-preflight:
+        condition: service_completed_successfully
     healthcheck:
       test: ["CMD-SHELL", "curl --fail --silent --show-error http://localhost:8081/actuator/health >/dev/null"]
       interval: 3s
@@ -162,6 +194,8 @@ services:
         condition: service_healthy
       topic-init:
         condition: service_completed_successfully
+      secret-preflight:
+        condition: service_completed_successfully
     healthcheck:
       test: ["CMD-SHELL", "curl --fail --silent --show-error http://localhost:8083/actuator/health/readiness >/dev/null"]
       interval: 3s
@@ -195,6 +229,8 @@ services:
         condition: service_healthy
       topic-init:
         condition: service_completed_successfully
+      secret-preflight:
+        condition: service_completed_successfully
     healthcheck:
       test: ["CMD-SHELL", "curl --fail --silent --show-error http://localhost:8085/actuator/health/readiness >/dev/null"]
       interval: 3s
@@ -299,6 +335,25 @@ services:
     networks: [backend]
     restart: unless-stopped
 
+  consumer-assignment:
+    image: apache/kafka:3.8.0
+    entrypoint: ["/bin/sh", "/opt/sportsbook/wait-consumer-assignments.sh"]
+    environment:
+      KAFKA_BOOTSTRAP_SERVERS: kafka:9092
+      ASSIGNMENT_TIMEOUT_SECONDS: "180"
+    volumes:
+      - ./docker/kafka/wait-consumer-assignments.sh:/opt/sportsbook/wait-consumer-assignments.sh:ro
+    depends_on:
+      kafka:
+        condition: service_healthy
+      topic-init:
+        condition: service_completed_successfully
+      betting:
+        condition: service_healthy
+      gateway:
+        condition: service_healthy
+    networks: [backend]
+
   settlement:
     build:
       context: ./docker
@@ -334,6 +389,8 @@ services:
         condition: service_completed_successfully
       wallet:
         condition: service_healthy
+      consumer-assignment:
+        condition: service_completed_successfully
     healthcheck:
       test: ["CMD-SHELL", "curl --fail --silent --show-error http://localhost:8084/actuator/health/readiness >/dev/null"]
       interval: 3s
diff --git a/docker/kafka/wait-consumer-assignments.sh b/docker/kafka/wait-consumer-assignments.sh
new file mode 100755
index 0000000..9ebcaf4
--- /dev/null
+++ b/docker/kafka/wait-consumer-assignments.sh
@@ -0,0 +1,43 @@
+#!/bin/sh
+set -eu
+
+BOOTSTRAP=${KAFKA_BOOTSTRAP_SERVERS:-kafka:9092}
+CONSUMER_GROUPS=${KAFKA_CONSUMER_GROUPS:-/opt/kafka/bin/kafka-consumer-groups.sh}
+TIMEOUT=${ASSIGNMENT_TIMEOUT_SECONDS:-180}
+POLL=${ASSIGNMENT_POLL_SECONDS:-2}
+WORK=$(mktemp -d)
+trap 'rm -rf "$WORK"' EXIT INT TERM
+
+cat >"$WORK/expected" <<'EOF'
+bet.resolution.revised.v1:0
+bet.resolution.revised.v1:1
+bet.resolution.revised.v1:2
+bet.settled.v1:0
+bet.settled.v1:1
+bet.settled.v1:2
+bet.voided.v1:0
+bet.voided.v1:1
+bet.voided.v1:2
+EOF
+
+group_ready() {
+  group=$1
+  "$CONSUMER_GROUPS" --bootstrap-server "$BOOTSTRAP" --describe --group "$group" \
+    >"$WORK/$group.out" 2>/dev/null || return 1
+  awk -v group="$group" '
+    $1 == group && $7 != "-" { print $2 ":" $3 }
+  ' "$WORK/$group.out" | sort -u >"$WORK/$group.actual"
+  cmp -s "$WORK/expected" "$WORK/$group.actual"
+}
+
+deadline=$(( $(date +%s) + TIMEOUT ))
+while [ "$(date +%s)" -lt "$deadline" ]; do
+  if group_ready gateway-bets && group_ready betting-resolution; then
+    printf 'consumer-assignment: gateway-bets=9 betting-resolution=9\n'
+    exit 0
+  fi
+  sleep "$POLL"
+done
+
+printf 'consumer-assignment: timed out waiting for exact active assignments\n' >&2
+exit 1


## `test(startup): verify dependency order`

diff --git a/tests/test_odds_wiring.py b/tests/test_odds_wiring.py
index 9156138..76e8e0f 100644
--- a/tests/test_odds_wiring.py
+++ b/tests/test_odds_wiring.py
@@ -29,6 +29,7 @@ class OddsWiringTest(ComposeContractFixture):
                 "kafka": "service_healthy",
                 "redis-odds": "service_healthy",
                 "topic-init": "service_completed_successfully",
+                "secret-preflight": "service_completed_successfully",
             },
         )
 
diff --git a/tests/test_risk_wiring.py b/tests/test_risk_wiring.py
index 7f10207..836d1d7 100644
--- a/tests/test_risk_wiring.py
+++ b/tests/test_risk_wiring.py
@@ -36,6 +36,7 @@ class RiskWiringTest(ComposeContractFixture):
                 "kafka": "service_healthy",
                 "redis-risk": "service_healthy",
                 "topic-init": "service_completed_successfully",
+                "secret-preflight": "service_completed_successfully",
             },
         )
 
diff --git a/tests/test_settlement_wiring.py b/tests/test_settlement_wiring.py
index a2c3028..1c4cb6e 100644
--- a/tests/test_settlement_wiring.py
+++ b/tests/test_settlement_wiring.py
@@ -52,6 +52,7 @@ class SettlementWiringTest(ComposeContractFixture):
                 "kafka": "service_healthy",
                 "topic-init": "service_completed_successfully",
                 "wallet": "service_healthy",
+                "consumer-assignment": "service_completed_successfully",
             },
         )
         self.assertIn(
diff --git a/tests/test_startup_dag.py b/tests/test_startup_dag.py
new file mode 100644
index 0000000..a731b5c
--- /dev/null
+++ b/tests/test_startup_dag.py
@@ -0,0 +1,67 @@
+from tests.compose_contract_fixture import ComposeContractFixture
+
+
+RANK = {
+    "postgres": 0,
+    "kafka": 0,
+    "redis-risk": 0,
+    "redis-odds": 0,
+    "redis-wallet": 0,
+    "redis-gateway": 0,
+    "secret-preflight": 0,
+    "topic-init": 1,
+    "wallet": 2,
+    "risk": 2,
+    "odds": 2,
+    "betting": 3,
+    "gateway": 4,
+    "consumer-assignment": 5,
+    "settlement": 6,
+    "admin": 7,
+}
+
+
+class StartupDagTest(ComposeContractFixture):
+    def test_orders_infrastructure_topics_services_and_control_plane(self) -> None:
+        services = self.rendered()["services"]
+
+        for service, rank in RANK.items():
+            for dependency in services[service].get("depends_on", {}):
+                with self.subTest(service=service, dependency=dependency):
+                    self.assertIn(dependency, RANK)
+                    self.assertLess(RANK[dependency], rank)
+
+        self.assert_dependency_conditions(
+            services["topic-init"],
+            {
+                "postgres": "service_healthy",
+                "kafka": "service_healthy",
+                "redis-risk": "service_healthy",
+                "redis-odds": "service_healthy",
+                "redis-wallet": "service_healthy",
+                "redis-gateway": "service_healthy",
+            },
+        )
+        self.assert_dependency_conditions(
+            services["consumer-assignment"],
+            {
+                "kafka": "service_healthy",
+                "topic-init": "service_completed_successfully",
+                "betting": "service_healthy",
+                "gateway": "service_healthy",
+            },
+        )
+        self.assertEqual(
+            services["settlement"]["depends_on"]["consumer-assignment"]["condition"],
+            "service_completed_successfully",
+        )
+        self.assertEqual(
+            services["admin"]["depends_on"]["settlement"]["condition"],
+            "service_healthy",
+        )
+
+
+if __name__ == "__main__":
+    import unittest
+
+    unittest.main()
diff --git a/tests/test_wallet_wiring.py b/tests/test_wallet_wiring.py
index 2221b6b..1f89f67 100644
--- a/tests/test_wallet_wiring.py
+++ b/tests/test_wallet_wiring.py
@@ -43,6 +43,7 @@ class WalletWiringTest(ComposeContractFixture):
                 "kafka": "service_healthy",
                 "redis-wallet": "service_healthy",
                 "topic-init": "service_completed_successfully",
+                "secret-preflight": "service_completed_successfully",
             },
         )
 
