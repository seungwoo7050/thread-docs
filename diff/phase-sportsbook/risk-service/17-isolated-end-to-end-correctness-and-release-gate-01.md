# 격리된 엔드투엔드 정확성 검증과 릴리스 게이트

## `build(boot): package executable service`

diff --git a/pom.xml b/pom.xml
index 18cb74b..185faf8 100644
--- a/pom.xml
+++ b/pom.xml
@@ -117,6 +117,18 @@
 
     <build>
         <plugins>
+            <plugin>
+                <groupId>org.springframework.boot</groupId>
+                <artifactId>spring-boot-maven-plugin</artifactId>
+                <version>${spring-boot.version}</version>
+                <executions>
+                    <execution>
+                        <goals>
+                            <goal>repackage</goal>
+                        </goals>
+                    </execution>
+                </executions>
+            </plugin>
             <plugin>
                 <groupId>org.apache.maven.plugins</groupId>
                 <artifactId>maven-compiler-plugin</artifactId>


## `test(load): provision isolated risk dependencies`

diff --git a/load-test/docker-compose.yml b/load-test/docker-compose.yml
new file mode 100644
index 0000000..b47ecae
--- /dev/null
+++ b/load-test/docker-compose.yml
@@ -0,0 +1,34 @@
+services:
+  redis:
+    image: redis:7.4-alpine
+    ports: ["127.0.0.1:16379:6379"]
+    healthcheck:
+      test: ["CMD", "redis-cli", "ping"]
+      interval: 2s
+      timeout: 1s
+      retries: 30
+
+  kafka:
+    image: confluentinc/cp-kafka:7.7.0
+    ports: ["127.0.0.1:19092:19092"]
+    environment:
+      KAFKA_NODE_ID: 1
+      KAFKA_PROCESS_ROLES: broker,controller
+      KAFKA_LISTENERS: INTERNAL://0.0.0.0:9092,HOST://0.0.0.0:19092,CONTROLLER://0.0.0.0:9093
+      KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka:9092,HOST://localhost:19092
+      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,INTERNAL:PLAINTEXT,HOST:PLAINTEXT
+      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
+      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@localhost:9093
+      KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
+      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
+      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
+      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
+      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
+      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
+      CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk
+    healthcheck:
+      test: ["CMD-SHELL", "bash -c 'exec 3<>/dev/tcp/localhost/9092'"]
+      interval: 4s
+      timeout: 4s
+      retries: 30
+      start_period: 20s


## `test(load): verify concurrent reservation cardinality`

diff --git a/load-test/concurrent-admission.sh b/load-test/concurrent-admission.sh
new file mode 100644
index 0000000..4a66a37
--- /dev/null
+++ b/load-test/concurrent-admission.sh
@@ -0,0 +1,66 @@
+#!/usr/bin/env bash
+set -euo pipefail
+
+base_url=${RISK_BASE_URL:?}
+output_dir=${RISK_GATE_OUTPUT:?}
+betting_key=${INTERNAL_BETTING_SERVICE_API_KEY:?}
+admin_key=${INTERNAL_ADMIN_API_KEY:?}
+user_id=60000000-0000-4000-8000-000000000001
+selection_id=70000000-0000-4000-8000-000000000001
+
+auth=(-H "X-Internal-Service: betting-service" -H "X-Internal-Api-Key: ${betting_key}")
+admin=(-H "X-Internal-Service: admin-api" -H "X-Internal-Api-Key: ${admin_key}")
+json=(-H "Content-Type: application/json")
+
+body() {
+  local bet_id=$1 amount=$2
+  printf '{"userId":"%s","betId":"%s","stake":{"amount":%s,"currency":"KRW"},"selectionIds":["%s"]}' \
+    "${user_id}" "${bet_id}" "${amount}" "${selection_id}"
+}
+
+reserve() {
+  local bet_id=$1 amount=$2 target=$3
+  curl -sS "${auth[@]}" "${json[@]}" -X POST -d "$(body "${bet_id}" "${amount}")" \
+    "${base_url}/internal/v1/risk/reservations" -o "${target}.json" \
+    -w '%{http_code}' > "${target}.status"
+}
+
+same_bet=80000000-0000-4000-8000-000000000001
+pids=()
+for attempt in $(seq 1 100); do
+  reserve "${same_bet}" 10 "${output_dir}/same-${attempt}" &
+  pids+=("$!")
+done
+for pid in "${pids[@]}"; do wait "${pid}"; done
+
+test "$(sort -u "${output_dir}"/same-*.status)" = 200
+jq -s -e '
+  length == 100 and
+  all(.approved == true and .reservationState == "RESERVED") and
+  (map(select(.replayed == false)) | length) == 1 and
+  (map(select(.replayed == true)) | length) == 99 and
+  (map(.reservationToken) | unique | length) == 1
+' "${output_dir}"/same-*.json > /dev/null
+
+for type in STAKE_DAILY STAKE_WEEKLY STAKE_MONTHLY; do
+  curl -fsS "${admin[@]}" "${json[@]}" -X PATCH \
+    -d "{\"type\":\"${type}\",\"currency\":\"KRW\",\"value\":100}" \
+    "${base_url}/internal/v1/risk/limits/${user_id}" > /dev/null
+done
+
+pids=()
+for ordinal in 2 3; do
+  bet_id=80000000-0000-4000-8000-$(printf '%012d' "${ordinal}")
+  reserve "${bet_id}" 60 "${output_dir}/capacity-${ordinal}" &
+  pids+=("$!")
+done
+for pid in "${pids[@]}"; do wait "${pid}"; done
+
+test "$(sort -u "${output_dir}"/capacity-*.status)" = 200
+jq -s -e '
+  length == 2 and
+  (map(select(.approved == true and .replayed == false and
+    .reservationState == "RESERVED" and (.reservationToken | type == "string"))) | length) == 1 and
+  (map(select(.approved == false and .replayed == false and
+    .rejectionReason == "STAKE_DAILY_LIMIT_EXCEEDED" and (has("reservationToken") | not))) | length) == 1
+' "${output_dir}"/capacity-*.json > /dev/null


## `test(load): verify reservation lifecycle replay`

diff --git a/load-test/lifecycle-replay.sh b/load-test/lifecycle-replay.sh
new file mode 100644
index 0000000..aabe3fa
--- /dev/null
+++ b/load-test/lifecycle-replay.sh
@@ -0,0 +1,51 @@
+#!/usr/bin/env bash
+set -euo pipefail
+
+base_url=${RISK_BASE_URL:?}
+output_dir=${RISK_GATE_OUTPUT:?}
+betting_key=${INTERNAL_BETTING_SERVICE_API_KEY:?}
+auth=(-H "X-Internal-Service: betting-service" -H "X-Internal-Api-Key: ${betting_key}")
+json=(-H "Content-Type: application/json")
+user_id=61000000-0000-4000-8000-000000000001
+selection_id=71000000-0000-4000-8000-000000000001
+
+reserve() {
+  local bet_id=$1 target=$2
+  curl -fsS "${auth[@]}" "${json[@]}" -X POST \
+    -d "{\"userId\":\"${user_id}\",\"betId\":\"${bet_id}\",\"stake\":{\"amount\":10,\"currency\":\"KRW\"},\"selectionIds\":[\"${selection_id}\"]}" \
+    "${base_url}/internal/v1/risk/reservations" -o "${target}"
+  jq -e '.approved == true and .replayed == false and
+    .reservationState == "RESERVED" and (.reservationToken | type == "string")' \
+    "${target}" > /dev/null
+}
+
+committed=81000000-0000-4000-8000-000000000001
+reserve "${committed}" "${output_dir}/committed.json"
+token=$(jq -er '.reservationToken' "${output_dir}/committed.json")
+for attempt in 1 2; do
+  status=$(curl -sS "${auth[@]}" -H "X-Risk-Reservation-Token: ${token}" \
+    -X PUT -o /dev/null -w '%{http_code}' \
+    "${base_url}/internal/v1/risk/reservations/${committed}/commit")
+  test "${status}" = 204
+done
+status=$(curl -sS "${auth[@]}" -X DELETE \
+  -o "${output_dir}/committed-release.json" -w '%{http_code}' \
+  "${base_url}/internal/v1/risk/reservations/${committed}")
+test "${status}" = 409
+jq -e '.errorCode == "RISK_RESERVATION_COMMITTED"' \
+  "${output_dir}/committed-release.json" > /dev/null
+
+released=81000000-0000-4000-8000-000000000002
+reserve "${released}" "${output_dir}/released.json"
+released_token=$(jq -er '.reservationToken' "${output_dir}/released.json")
+for attempt in 1 2; do
+  status=$(curl -sS "${auth[@]}" -X DELETE -o /dev/null -w '%{http_code}' \
+    "${base_url}/internal/v1/risk/reservations/${released}")
+  test "${status}" = 204
+done
+status=$(curl -sS "${auth[@]}" -H "X-Risk-Reservation-Token: ${released_token}" \
+  -X PUT -o "${output_dir}/released-commit.json" -w '%{http_code}' \
+  "${base_url}/internal/v1/risk/reservations/${released}/commit")
+test "${status}" = 404
+jq -e '.errorCode == "RISK_RESERVATION_NOT_FOUND"' \
+  "${output_dir}/released-commit.json" > /dev/null


## `test(events): provide embedded Kafka fixtures`

diff --git a/src/test/java/com/sportsbook/risk/event/BetPlacedKafkaIntegrationSupport.java b/src/test/java/com/sportsbook/risk/event/BetPlacedKafkaIntegrationSupport.java
new file mode 100644
index 0000000..a401207
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/event/BetPlacedKafkaIntegrationSupport.java
@@ -0,0 +1,66 @@
+package com.sportsbook.risk.event;
+
+import static org.mockito.Mockito.reset;
+
+import java.time.Duration;
+import java.util.Map;
+import java.util.UUID;
+import java.util.concurrent.TimeUnit;
+import org.apache.kafka.clients.consumer.Consumer;
+import org.apache.kafka.clients.consumer.ConsumerConfig;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
+import org.apache.kafka.common.serialization.ByteArrayDeserializer;
+import org.apache.kafka.common.serialization.StringDeserializer;
+import org.junit.jupiter.api.BeforeEach;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.boot.test.mock.mockito.MockBean;
+import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.kafka.test.EmbeddedKafkaBroker;
+import org.springframework.kafka.test.context.EmbeddedKafka;
+import org.springframework.kafka.test.utils.KafkaTestUtils;
+import org.springframework.test.annotation.DirtiesContext;
+
+@SpringBootTest(
+    properties = {
+      "risk.auth.betting-service-api-key=bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
+      "risk.auth.admin-api-key=aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
+      "risk.auth.platform-api-key=pppppppppppppppppppppppppppppppp",
+      "management.health.redis.enabled=false",
+      "management.endpoint.health.validate-group-membership=false"
+    })
+@EmbeddedKafka(
+    partitions = 1,
+    topics = {"bet.placed.v1", "bet.placed.v1.DLT"},
+    bootstrapServersProperty = "spring.kafka.bootstrap-servers")
+@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_CLASS)
+abstract class BetPlacedKafkaIntegrationSupport {
+  static final String DLT_TOPIC = "bet.placed.v1.DLT";
+
+  @Autowired private KafkaTemplate<String, byte[]> kafka;
+  @Autowired private EmbeddedKafkaBroker broker;
+  @MockBean protected AcceptedBetReconciler reconciler;
+
+  @BeforeEach
+  void resetReconciler() {
+    reset(reconciler);
+  }
+
+  void publish(String key, byte[] payload) throws Exception {
+    kafka.send("bet.placed.v1", key, payload).get(10, TimeUnit.SECONDS);
+  }
+
+  ConsumerRecord<String, byte[]> consumeDeadLetter() {
+    Map<String, Object> properties =
+        KafkaTestUtils.consumerProps("risk-dlt-test-" + UUID.randomUUID(), "false", broker);
+    properties.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
+    try (Consumer<String, byte[]> consumer =
+        new DefaultKafkaConsumerFactory<>(
+                properties, new StringDeserializer(), new ByteArrayDeserializer())
+            .createConsumer()) {
+      broker.consumeFromAnEmbeddedTopic(consumer, DLT_TOPIC);
+      return KafkaTestUtils.getSingleRecord(consumer, DLT_TOPIC, Duration.ofSeconds(10));
+    }
+  }
+}


## `test(events): verify accepted-bet broker delivery`

diff --git a/src/test/java/com/sportsbook/risk/event/BetPlacedKafkaProjectionIntegrationTest.java b/src/test/java/com/sportsbook/risk/event/BetPlacedKafkaProjectionIntegrationTest.java
new file mode 100644
index 0000000..4b5e810
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/event/BetPlacedKafkaProjectionIntegrationTest.java
@@ -0,0 +1,37 @@
+package com.sportsbook.risk.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.ArgumentMatchers.any;
+import static org.mockito.Mockito.timeout;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+
+import org.junit.jupiter.api.Test;
+import org.mockito.ArgumentCaptor;
+
+class BetPlacedKafkaProjectionIntegrationTest extends BetPlacedKafkaIntegrationSupport {
+  @Test
+  void binaryAcceptedBetReachesTheReconciler() throws Exception {
+    when(reconciler.reconcile(any())).thenReturn(AcceptedBetReconciliation.PROJECTED);
+
+    publish(BetPlacedEventFixture.USER_ID, BetPlacedEventFixture.payload());
+
+    ArgumentCaptor<AcceptedBetEnvelope> envelope =
+        ArgumentCaptor.forClass(AcceptedBetEnvelope.class);
+    verify(reconciler, timeout(10_000)).reconcile(envelope.capture());
+    assertThat(envelope.getValue().command().userId().value().toString())
+        .isEqualTo(BetPlacedEventFixture.USER_ID);
+    assertThat(envelope.getValue().command().stake().amount()).isEqualTo(10_000L);
+  }
+
+  @Test
+  void transientReconciliationFailureRedeliversTheSameEvent() throws Exception {
+    when(reconciler.reconcile(any()))
+        .thenThrow(new IllegalStateException("redis unavailable"))
+        .thenReturn(AcceptedBetReconciliation.PROJECTED);
+
+    publish(BetPlacedEventFixture.USER_ID, BetPlacedEventFixture.payload());
+
+    verify(reconciler, timeout(10_000).times(2)).reconcile(any());
+  }
+}


## `test(events): verify dead-letter broker delivery`

diff --git a/src/test/java/com/sportsbook/risk/event/BetPlacedKafkaDeadLetterIntegrationTest.java b/src/test/java/com/sportsbook/risk/event/BetPlacedKafkaDeadLetterIntegrationTest.java
new file mode 100644
index 0000000..0eaad0d
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/event/BetPlacedKafkaDeadLetterIntegrationTest.java
@@ -0,0 +1,27 @@
+package com.sportsbook.risk.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.verifyNoInteractions;
+
+import java.nio.charset.StandardCharsets;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
+import org.junit.jupiter.api.Test;
+
+class BetPlacedKafkaDeadLetterIntegrationTest extends BetPlacedKafkaIntegrationSupport {
+  @Test
+  void malformedBinaryEventIsPublishedToTheDeadLetterTopic() throws Exception {
+    byte[] malformed = {1, 2, 3};
+
+    publish(BetPlacedEventFixture.USER_ID, malformed);
+
+    ConsumerRecord<String, byte[]> deadLetter = consumeDeadLetter();
+    assertThat(deadLetter.key()).isEqualTo(BetPlacedEventFixture.USER_ID);
+    assertThat(deadLetter.value()).containsExactly(malformed);
+    assertThat(
+            new String(
+                deadLetter.headers().lastHeader("risk-dlt-reason").value(),
+                StandardCharsets.US_ASCII))
+        .isEqualTo(BetPlacedFailureReason.MALFORMED_EVENT.name());
+    verifyNoInteractions(reconciler);
+  }
+}


## `test(events): provide Kafka Redis fixtures`

diff --git a/src/test/java/com/sportsbook/risk/event/BetPlacedKafkaRedisIntegrationSupport.java b/src/test/java/com/sportsbook/risk/event/BetPlacedKafkaRedisIntegrationSupport.java
new file mode 100644
index 0000000..9f221fe
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/event/BetPlacedKafkaRedisIntegrationSupport.java
@@ -0,0 +1,59 @@
+package com.sportsbook.risk.event;
+
+import com.sportsbook.risk.support.RedisTestSupport;
+import java.util.Map;
+import java.util.concurrent.TimeUnit;
+import org.apache.kafka.clients.admin.Admin;
+import org.apache.kafka.clients.admin.AdminClientConfig;
+import org.apache.kafka.clients.consumer.OffsetAndMetadata;
+import org.apache.kafka.common.TopicPartition;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.kafka.test.EmbeddedKafkaBroker;
+import org.springframework.kafka.test.context.EmbeddedKafka;
+import org.springframework.test.annotation.DirtiesContext;
+import org.springframework.test.context.DynamicPropertyRegistry;
+import org.springframework.test.context.DynamicPropertySource;
+
+@SpringBootTest(
+    properties = {
+      "risk.auth.betting-service-api-key=bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
+      "risk.auth.admin-api-key=aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
+      "risk.auth.platform-api-key=pppppppppppppppppppppppppppppppp"
+    })
+@EmbeddedKafka(
+    partitions = 1,
+    topics = {"bet.placed.v1", "bet.placed.v1.DLT"},
+    bootstrapServersProperty = "spring.kafka.bootstrap-servers")
+@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_CLASS)
+abstract class BetPlacedKafkaRedisIntegrationSupport extends RedisTestSupport {
+  @Autowired private KafkaTemplate<String, byte[]> kafka;
+  @Autowired private EmbeddedKafkaBroker broker;
+
+  @DynamicPropertySource
+  static void redisProperties(DynamicPropertyRegistry properties) {
+    properties.add("spring.data.redis.host", REDIS::getHost);
+    properties.add("spring.data.redis.port", REDIS::getFirstMappedPort);
+  }
+
+  void publishAcceptedBet() throws Exception {
+    kafka
+        .send("bet.placed.v1", BetPlacedEventFixture.USER_ID, BetPlacedEventFixture.payload())
+        .get(10, TimeUnit.SECONDS);
+  }
+
+  long committedSourceOffset() throws Exception {
+    try (Admin admin =
+        Admin.create(
+            Map.of(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, broker.getBrokersAsString()))) {
+      OffsetAndMetadata offset =
+          admin
+              .listConsumerGroupOffsets("risk.bet-placed-consumer")
+              .partitionsToOffsetAndMetadata()
+              .get(10, TimeUnit.SECONDS)
+              .get(new TopicPartition("bet.placed.v1", 0));
+      return offset == null ? 0L : offset.offset();
+    }
+  }
+}


## `test(events): verify accepted-bet Redis projection`

diff --git a/src/test/java/com/sportsbook/risk/event/BetPlacedKafkaRedisIntegrationTest.java b/src/test/java/com/sportsbook/risk/event/BetPlacedKafkaRedisIntegrationTest.java
new file mode 100644
index 0000000..23a8f83
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/event/BetPlacedKafkaRedisIntegrationTest.java
@@ -0,0 +1,36 @@
+package com.sportsbook.risk.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.awaitility.Awaitility.await;
+
+import java.time.Duration;
+import org.junit.jupiter.api.Test;
+
+class BetPlacedKafkaRedisIntegrationTest extends BetPlacedKafkaRedisIntegrationSupport {
+  private static final String BET_ID = "20000000-0000-4000-8000-000000000001";
+  private static final String DAILY_BASE =
+      "risk:limit:{" + BetPlacedEventFixture.USER_ID + "}:stake-daily:krw";
+
+  @Test
+  void acceptedBetProjectsOnceAcrossBrokerRedelivery() throws Exception {
+    publishAcceptedBet();
+
+    await()
+        .atMost(Duration.ofSeconds(10))
+        .untilAsserted(
+            () -> {
+              assertThat(redis.opsForValue().get(DAILY_BASE + ":sum")).isEqualTo("10000");
+              assertThat(redis.opsForValue().get("risk:event:fingerprint:" + BET_ID))
+                  .matches("[0-9a-f]{64}");
+              assertThat(committedSourceOffset()).isEqualTo(1L);
+            });
+
+    publishAcceptedBet();
+
+    await()
+        .atMost(Duration.ofSeconds(10))
+        .untilAsserted(() -> assertThat(committedSourceOffset()).isEqualTo(2L));
+    assertThat(redis.opsForValue().get(DAILY_BASE + ":sum")).isEqualTo("10000");
+    assertThat(redis.opsForZSet().size(DAILY_BASE + ":entries")).isEqualTo(1L);
+  }
+}


## `test(load): run isolated correctness gate`

diff --git a/load-test/run-gate.sh b/load-test/run-gate.sh
new file mode 100644
index 0000000..22cdd4a
--- /dev/null
+++ b/load-test/run-gate.sh
@@ -0,0 +1,84 @@
+#!/usr/bin/env bash
+set -euo pipefail
+
+repo_root=$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)
+compose_file=${repo_root}/load-test/docker-compose.yml
+compose_project=risk-release-gate-$$
+output_dir=$(mktemp -d "${TMPDIR:-/tmp}/risk-gate.XXXXXX")
+service_pid=
+export RISK_GATE_OUTPUT=${output_dir}
+export RISK_BASE_URL=${RISK_BASE_URL:-http://localhost:18083}
+
+cleanup() {
+  status=$?
+  if [[ ${status} -ne 0 && -f ${output_dir}/service.log ]]; then
+    tail -100 "${output_dir}/service.log" >&2
+  fi
+  if [[ -n ${service_pid} ]]; then
+    kill "${service_pid}" 2>/dev/null || true
+    wait "${service_pid}" 2>/dev/null || true
+  fi
+  docker compose -p "${compose_project}" -f "${compose_file}" down -v --remove-orphans \
+    > /dev/null 2>&1 || true
+  rm -rf "${output_dir}"
+  exit "${status}"
+}
+trap cleanup EXIT INT TERM
+
+for command in docker curl jq java; do
+  command -v "${command}" > /dev/null || { echo "missing command: ${command}" >&2; exit 1; }
+done
+docker compose version > /dev/null
+for name in INTERNAL_BETTING_SERVICE_API_KEY INTERNAL_ADMIN_API_KEY INTERNAL_PLATFORM_API_KEY; do
+  value=${!name:-}
+  [[ ${#value} -ge 32 ]] || { echo "${name} must contain at least 32 characters" >&2; exit 1; }
+done
+[[ ${INTERNAL_BETTING_SERVICE_API_KEY} != "${INTERNAL_ADMIN_API_KEY}" \
+  && ${INTERNAL_BETTING_SERVICE_API_KEY} != "${INTERNAL_PLATFORM_API_KEY}" \
+  && ${INTERNAL_ADMIN_API_KEY} != "${INTERNAL_PLATFORM_API_KEY}" ]] \
+  || { echo "risk gate credentials must be distinct" >&2; exit 1; }
+
+cd "${repo_root}"
+./mvnw -B -o -Dmaven.repo.local="${RISK_MAVEN_REPO:?}" clean verify
+jar_path=$(find target -maxdepth 1 -name 'risk-service-*.jar' ! -name '*.original' -print -quit)
+[[ -n ${jar_path} ]] || { echo "risk service jar is missing" >&2; exit 1; }
+
+docker compose -p "${compose_project}" -f "${compose_file}" up -d --wait --wait-timeout 180
+SERVER_PORT=18083 REDIS_HOST=localhost REDIS_PORT=16379 KAFKA_BOOTSTRAP=localhost:19092 \
+  java -jar "${jar_path}" > "${output_dir}/service.log" 2>&1 &
+service_pid=$!
+
+ready=false
+for attempt in $(seq 1 120); do
+  if curl -fsS "${RISK_BASE_URL}/actuator/health/readiness" \
+    -o "${output_dir}/readiness.json"; then
+    ready=true
+    break
+  fi
+  kill -0 "${service_pid}" 2>/dev/null || { echo "risk service exited" >&2; exit 1; }
+  sleep 1
+done
+[[ ${ready} == true ]] || { echo "risk service readiness timed out" >&2; exit 1; }
+jq -e '.status == "UP"' "${output_dir}/readiness.json" > /dev/null
+
+bash load-test/concurrent-admission.sh
+bash load-test/lifecycle-replay.sh
+curl -fsS "${RISK_BASE_URL}/actuator/prometheus" -o "${output_dir}/metrics.txt"
+
+assert_sample() {
+  local metric=$1 expected=$2 first_label=$3 second_label=${4:-}
+  awk -v metric="${metric}" -v expected="${expected}" -v first="${first_label}" \
+    -v second="${second_label}" '$1 ~ ("^" metric "\\{") && index($1, first) &&
+      (second == "" || index($1, second)) { count++; value=$2 }
+      END { exit !(count == 1 && value + 0 == expected + 0) }' "${output_dir}/metrics.txt"
+}
+assert_sample risk_reservation_requests_total 4 'result="created"'
+assert_sample risk_reservation_requests_total 99 'result="replayed"'
+assert_sample risk_reservation_requests_total 1 'result="rejected"'
+assert_sample risk_reservation_transitions_total 1 'operation="commit"' 'result="applied"'
+assert_sample risk_reservation_transitions_total 1 'operation="commit"' 'result="replayed"'
+assert_sample risk_reservation_transitions_total 1 'operation="commit"' 'result="tombstoned"'
+assert_sample risk_reservation_transitions_total 1 'operation="release"' 'result="applied"'
+assert_sample risk_reservation_transitions_total 1 'operation="release"' 'result="replayed"'
+assert_sample risk_reservation_transitions_total 1 'operation="release"' 'result="conflict"'
+echo "risk correctness gate passed"


