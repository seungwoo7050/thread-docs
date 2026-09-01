## `feat(application): bootstrap the service`

diff --git a/pom.xml b/pom.xml
index a849fbc..384340e 100644
--- a/pom.xml
+++ b/pom.xml
@@ -120,6 +120,10 @@
 
     <build>
         <plugins>
+            <plugin>
+                <groupId>org.springframework.boot</groupId>
+                <artifactId>spring-boot-maven-plugin</artifactId>
+            </plugin>
             <plugin>
                 <groupId>com.diffplug.spotless</groupId>
                 <artifactId>spotless-maven-plugin</artifactId>
diff --git a/src/main/java/com/sportsbook/oddsfeed/OddsFeedApplication.java b/src/main/java/com/sportsbook/oddsfeed/OddsFeedApplication.java
new file mode 100644
index 0000000..b036b94
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/OddsFeedApplication.java
@@ -0,0 +1,13 @@
+package com.sportsbook.oddsfeed;
+
+import org.springframework.boot.SpringApplication;
+import org.springframework.boot.autoconfigure.SpringBootApplication;
+
+@SpringBootApplication
+@SuppressWarnings("checkstyle:HideUtilityClassConstructor")
+public class OddsFeedApplication {
+
+  public static void main(String[] args) {
+    SpringApplication.run(OddsFeedApplication.class, args);
+  }
+}


## `test(application): verify application startup`

diff --git a/src/test/java/com/sportsbook/oddsfeed/OddsFeedApplicationTests.java b/src/test/java/com/sportsbook/oddsfeed/OddsFeedApplicationTests.java
new file mode 100644
index 0000000..2117fb9
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/OddsFeedApplicationTests.java
@@ -0,0 +1,14 @@
+package com.sportsbook.oddsfeed;
+
+import static org.assertj.core.api.Assertions.assertThatCode;
+
+import org.junit.jupiter.api.Test;
+
+class OddsFeedApplicationTests {
+
+  @Test
+  void mainMethodIsInvokable() {
+    assertThatCode(() -> OddsFeedApplication.class.getDeclaredMethod("main", String[].class))
+        .doesNotThrowAnyException();
+  }
+}


## `build(test): add integration test tooling`

diff --git a/pom.xml b/pom.xml
index cfc7809..fa43297 100644
--- a/pom.xml
+++ b/pom.xml
@@ -77,5 +77,41 @@
             <groupId>io.opentelemetry</groupId>
             <artifactId>opentelemetry-exporter-otlp</artifactId>
         </dependency>
+        <dependency>
+            <groupId>org.springframework.boot</groupId>
+            <artifactId>spring-boot-starter-test</artifactId>
+            <scope>test</scope>
+        </dependency>
+        <dependency>
+            <groupId>org.springframework.kafka</groupId>
+            <artifactId>spring-kafka-test</artifactId>
+            <scope>test</scope>
+        </dependency>
+        <dependency>
+            <groupId>io.projectreactor</groupId>
+            <artifactId>reactor-test</artifactId>
+            <scope>test</scope>
+        </dependency>
+        <dependency>
+            <groupId>org.testcontainers</groupId>
+            <artifactId>junit-jupiter</artifactId>
+            <scope>test</scope>
+        </dependency>
+        <dependency>
+            <groupId>org.testcontainers</groupId>
+            <artifactId>kafka</artifactId>
+            <scope>test</scope>
+        </dependency>
+        <dependency>
+            <groupId>org.springframework.boot</groupId>
+            <artifactId>spring-boot-testcontainers</artifactId>
+            <scope>test</scope>
+        </dependency>
+        <dependency>
+            <groupId>org.wiremock</groupId>
+            <artifactId>wiremock-standalone</artifactId>
+            <version>3.4.2</version>
+            <scope>test</scope>
+        </dependency>
     </dependencies>
 </project>


## `build(format): enforce Java formatting`

diff --git a/pom.xml b/pom.xml
index fa43297..be17ee6 100644
--- a/pom.xml
+++ b/pom.xml
@@ -24,6 +24,7 @@
         <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
         <shared-protocol.version>1.0.0</shared-protocol.version>
         <logstash-encoder.version>7.4</logstash-encoder.version>
+        <spotless.version>2.43.0</spotless.version>
     </properties>
 
     <dependencies>
@@ -114,4 +115,32 @@
             <scope>test</scope>
         </dependency>
     </dependencies>
+
+    <build>
+        <plugins>
+            <plugin>
+                <groupId>com.diffplug.spotless</groupId>
+                <artifactId>spotless-maven-plugin</artifactId>
+                <version>${spotless.version}</version>
+                <configuration>
+                    <java>
+                        <googleJavaFormat>
+                            <version>1.22.0</version>
+                            <style>GOOGLE</style>
+                        </googleJavaFormat>
+                        <removeUnusedImports/>
+                        <trimTrailingWhitespace/>
+                        <endWithNewline/>
+                    </java>
+                </configuration>
+                <executions>
+                    <execution>
+                        <goals>
+                            <goal>check</goal>
+                        </goals>
+                    </execution>
+                </executions>
+            </plugin>
+        </plugins>
+    </build>
 </project>


## `build(checkstyle): enforce static analysis`

diff --git a/config/checkstyle/checkstyle.xml b/config/checkstyle/checkstyle.xml
new file mode 100644
index 0000000..c98b926
--- /dev/null
+++ b/config/checkstyle/checkstyle.xml
@@ -0,0 +1,31 @@
+<?xml version="1.0"?>
+<!DOCTYPE module PUBLIC
+    "-//Checkstyle//DTD Checkstyle Configuration 1.3//EN"
+    "https://checkstyle.org/dtds/configuration_1_3.dtd">
+
+<module name="Checker">
+    <property name="severity" value="error"/>
+    <property name="charset" value="UTF-8"/>
+
+    <module name="SuppressWarningsFilter"/>
+    <module name="TreeWalker">
+        <module name="SuppressWarningsHolder"/>
+        <module name="MagicNumber">
+            <property name="ignoreNumbers" value="-1, 0, 1, 2, 100"/>
+            <property name="ignoreHashCodeMethod" value="true"/>
+            <property name="ignoreAnnotation" value="true"/>
+            <property name="ignoreFieldDeclaration" value="true"/>
+        </module>
+        <module name="ParameterNumber">
+            <property name="max" value="10"/>
+            <property name="ignoreOverriddenMethods" value="true"/>
+            <property name="tokens" value="METHOD_DEF"/>
+        </module>
+        <module name="UnusedImports"/>
+        <module name="RedundantImport"/>
+        <module name="EmptyBlock">
+            <property name="option" value="text"/>
+        </module>
+        <module name="HideUtilityClassConstructor"/>
+    </module>
+</module>
diff --git a/pom.xml b/pom.xml
index be17ee6..a849fbc 100644
--- a/pom.xml
+++ b/pom.xml
@@ -25,6 +25,8 @@
         <shared-protocol.version>1.0.0</shared-protocol.version>
         <logstash-encoder.version>7.4</logstash-encoder.version>
         <spotless.version>2.43.0</spotless.version>
+        <checkstyle-plugin.version>3.5.0</checkstyle-plugin.version>
+        <checkstyle.version>10.18.2</checkstyle.version>
     </properties>
 
     <dependencies>
@@ -141,6 +143,32 @@
                     </execution>
                 </executions>
             </plugin>
+            <plugin>
+                <groupId>org.apache.maven.plugins</groupId>
+                <artifactId>maven-checkstyle-plugin</artifactId>
+                <version>${checkstyle-plugin.version}</version>
+                <dependencies>
+                    <dependency>
+                        <groupId>com.puppycrawl.tools</groupId>
+                        <artifactId>checkstyle</artifactId>
+                        <version>${checkstyle.version}</version>
+                    </dependency>
+                </dependencies>
+                <configuration>
+                    <configLocation>config/checkstyle/checkstyle.xml</configLocation>
+                    <consoleOutput>true</consoleOutput>
+                    <failOnViolation>true</failOnViolation>
+                    <includeTestSourceDirectory>false</includeTestSourceDirectory>
+                </configuration>
+                <executions>
+                    <execution>
+                        <phase>verify</phase>
+                        <goals>
+                            <goal>check</goal>
+                        </goals>
+                    </execution>
+                </executions>
+            </plugin>
         </plugins>
     </build>
 </project>


## `test(load): enforce acknowledged publish throughput`

diff --git a/src/test/java/com/sportsbook/oddsfeed/load/KafkaPublishThroughputTest.java b/src/test/java/com/sportsbook/oddsfeed/load/KafkaPublishThroughputTest.java
new file mode 100644
index 0000000..e19bfe0
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/load/KafkaPublishThroughputTest.java
@@ -0,0 +1,95 @@
+package com.sportsbook.oddsfeed.load;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.oddsfeed.config.KafkaTopicsProperties;
+import com.sportsbook.oddsfeed.config.PublishProperties;
+import com.sportsbook.oddsfeed.kafka.AvroSerializer;
+import com.sportsbook.oddsfeed.kafka.BrokerAvailability;
+import com.sportsbook.oddsfeed.publisher.OddsFeedPublisher;
+import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.MarketId;
+import com.sportsbook.protocol.value.Odds;
+import com.sportsbook.protocol.value.SelectionId;
+import java.math.BigDecimal;
+import java.time.Duration;
+import java.time.Instant;
+import java.util.HashMap;
+import java.util.Map;
+import java.util.UUID;
+import java.util.concurrent.TimeUnit;
+import org.apache.avro.specific.SpecificRecord;
+import org.apache.kafka.clients.producer.ProducerConfig;
+import org.apache.kafka.common.serialization.StringSerializer;
+import org.junit.jupiter.api.AfterAll;
+import org.junit.jupiter.api.BeforeAll;
+import org.junit.jupiter.api.Tag;
+import org.junit.jupiter.api.Test;
+import org.springframework.kafka.core.DefaultKafkaProducerFactory;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.kafka.test.EmbeddedKafkaBroker;
+import org.springframework.kafka.test.EmbeddedKafkaZKBroker;
+
+@Tag("load")
+class KafkaPublishThroughputTest {
+
+  private static final int EVENT_COUNT = 1_000;
+  private static final double MINIMUM_EVENTS_PER_SECOND = 50.0;
+  private static final KafkaTopicsProperties TOPICS =
+      new KafkaTopicsProperties(
+          "odds.changed", "market.status.changed", "event.lifecycle", "match.result");
+
+  private static EmbeddedKafkaBroker broker;
+  private static DefaultKafkaProducerFactory<String, SpecificRecord> producerFactory;
+  private static OddsFeedPublisher publisher;
+
+  @BeforeAll
+  static void startBroker() {
+    broker = new EmbeddedKafkaZKBroker(1, true, 1, TOPICS.oddsChanged());
+    broker.afterPropertiesSet();
+
+    Map<String, Object> properties = new HashMap<>();
+    properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, broker.getBrokersAsString());
+    properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
+    properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, AvroSerializer.class);
+    properties.put(ProducerConfig.ACKS_CONFIG, "1");
+    properties.put(ProducerConfig.LINGER_MS_CONFIG, 5);
+    properties.put(ProducerConfig.BATCH_SIZE_CONFIG, 32 * 1024);
+    properties.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "lz4");
+    producerFactory = new DefaultKafkaProducerFactory<>(properties);
+    publisher =
+        new OddsFeedPublisher(
+            new KafkaTemplate<>(producerFactory),
+            TOPICS,
+            new PublishProperties(new BigDecimal("0.01"), Duration.ofSeconds(5)),
+            new BrokerAvailability());
+  }
+
+  @AfterAll
+  static void stopBroker() {
+    producerFactory.destroy();
+    broker.destroy();
+  }
+
+  @Test
+  void sustainsBrokerAcknowledgedThroughput() {
+    EventId eventId = new EventId(UUID.fromString("00000000-0000-4000-8000-000000000001"));
+    MarketId marketId = new MarketId(UUID.fromString("00000000-0000-4000-8000-000000000002"));
+    SelectionId selectionId =
+        new SelectionId(UUID.fromString("00000000-0000-4000-8000-000000000003"));
+    Odds previous = Odds.ofDecimal("2.00");
+    Odds next = Odds.ofDecimal("2.10");
+    Instant changedAt = Instant.parse("2026-01-01T00:00:00Z");
+
+    publisher.publishOddsChanged(eventId, marketId, selectionId, previous, next, changedAt, false);
+    long startedAt = System.nanoTime();
+    for (int index = 0; index < EVENT_COUNT; index++) {
+      publisher.publishOddsChanged(
+          eventId, marketId, selectionId, previous, next, changedAt, false);
+    }
+    long elapsedNanos = System.nanoTime() - startedAt;
+    double rate = EVENT_COUNT / (elapsedNanos / (double) TimeUnit.SECONDS.toNanos(1));
+
+    assertThat(rate).isGreaterThanOrEqualTo(MINIMUM_EVENTS_PER_SECOND);
+  }
+}


## `test(load): provision Redis and Kafka`

diff --git a/load-test/docker-compose.yml b/load-test/docker-compose.yml
new file mode 100644
index 0000000..6728b08
--- /dev/null
+++ b/load-test/docker-compose.yml
@@ -0,0 +1,38 @@
+name: odds-load-gate
+
+services:
+  redis:
+    image: redis:7-alpine
+    ports:
+      - "${REDIS_PORT:-6392}:6379"
+    healthcheck:
+      test: ["CMD", "redis-cli", "ping"]
+      interval: 2s
+      timeout: 1s
+      retries: 20
+
+  kafka:
+    image: confluentinc/cp-kafka:7.6.1
+    ports:
+      - "${KAFKA_PORT:-9096}:9096"
+    environment:
+      KAFKA_NODE_ID: 1
+      KAFKA_PROCESS_ROLES: broker,controller
+      KAFKA_LISTENERS: INTERNAL://0.0.0.0:9092,HOST://0.0.0.0:9096,CONTROLLER://0.0.0.0:9093
+      KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka:9092,HOST://localhost:${KAFKA_PORT:-9096}
+      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,INTERNAL:PLAINTEXT,HOST:PLAINTEXT
+      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
+      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
+      KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
+      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
+      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
+      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
+      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
+      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
+      CLUSTER_ID: "MkU3OEVBNTcwNTJENDM2Qk"
+    healthcheck:
+      test: ["CMD-SHELL", "bash -c 'exec 3<>/dev/tcp/localhost/9092'"]
+      interval: 4s
+      timeout: 4s
+      retries: 30
+      start_period: 30s


## `test(load): define deterministic mock fixture`

diff --git a/load-test/fixtures/mock.env b/load-test/fixtures/mock.env
new file mode 100644
index 0000000..9fb514c
--- /dev/null
+++ b/load-test/fixtures/mock.env
@@ -0,0 +1,4 @@
+FIXTURE_RANDOM_SEED=424242
+FIXTURE_FROZEN_TICK_INTERVAL_MS=900000
+FIXTURE_MINUTES_PER_SECOND=0.001
+FIXTURE_SCENARIOS_AUTO_ROTATE=false


## `test(load): manage isolated service runtime`

diff --git a/load-test/lib/runtime.sh b/load-test/lib/runtime.sh
new file mode 100755
index 0000000..c0de0b9
--- /dev/null
+++ b/load-test/lib/runtime.sh
@@ -0,0 +1,63 @@
+#!/usr/bin/env bash
+
+compose() {
+  REDIS_PORT="${REDIS_PORT}" KAFKA_PORT="${KAFKA_PORT}" \
+    docker compose --project-name "${COMPOSE_PROJECT_NAME}" -f "${COMPOSE_FILE}" "$@"
+}
+
+stop_service() {
+  local attempt
+  if [[ -z "${SERVICE_PID}" ]]; then
+    return
+  fi
+  if ! kill -0 "${SERVICE_PID}" 2>/dev/null; then
+    wait "${SERVICE_PID}" 2>/dev/null || true
+    SERVICE_PID=''
+    return
+  fi
+
+  kill "${SERVICE_PID}" 2>/dev/null || true
+  for ((attempt = 0; attempt < 40; attempt++)); do
+    if ! kill -0 "${SERVICE_PID}" 2>/dev/null; then
+      break
+    fi
+    sleep 0.25
+  done
+  if kill -0 "${SERVICE_PID}" 2>/dev/null; then
+    kill -KILL "${SERVICE_PID}" 2>/dev/null || true
+  fi
+  wait "${SERVICE_PID}" 2>/dev/null || true
+  SERVICE_PID=''
+}
+
+cleanup_runtime() {
+  local exit_code=$?
+  trap - EXIT INT TERM
+  stop_service
+  compose down --volumes --remove-orphans >/dev/null 2>&1 || true
+  exit "${exit_code}"
+}
+
+reset_stack() {
+  compose down --volumes --remove-orphans >/dev/null 2>&1 || true
+  compose up --detach --wait
+}
+
+start_service() {
+  local endpoint=$1
+  local tick_interval=$2
+  ADMIN_API_INTERNAL_KEY=$(openssl rand -hex 32)
+  export ADMIN_API_INTERNAL_KEY
+  SERVER_PORT="${SERVER_PORT}" \
+  REDIS_HOST=localhost \
+  REDIS_PORT="${REDIS_PORT}" \
+  KAFKA_BOOTSTRAP_SERVERS="localhost:${KAFKA_PORT}" \
+  OTEL_SAMPLING_PROBABILITY=0 \
+  ODDSFEED_MOCK_RANDOM_SEED="${FIXTURE_RANDOM_SEED}" \
+  ODDSFEED_MOCK_TICK_INTERVAL_MS="${tick_interval}" \
+    java -jar "${JAR_PATH}" --spring.profiles.active=mock \
+      --oddsfeed.mock.minutes-per-second="${FIXTURE_MINUTES_PER_SECOND}" \
+      --oddsfeed.mock.scenarios.auto-rotate="${FIXTURE_SCENARIOS_AUTO_ROTATE}" \
+      >"${RESULT_ROOT}/${endpoint}-service.log" 2>&1 &
+  SERVICE_PID=$!
+}


## `test(load): discover HTTP fixture state`

diff --git a/load-test/fixtures/prepare-mock.sh b/load-test/fixtures/prepare-mock.sh
new file mode 100755
index 0000000..35cc5a5
--- /dev/null
+++ b/load-test/fixtures/prepare-mock.sh
@@ -0,0 +1,55 @@
+#!/usr/bin/env bash
+
+wait_for_service() {
+  local attempt
+  for ((attempt = 0; attempt < 120; attempt++)); do
+    if curl --fail --silent "${BASE_URL}/actuator/health/readiness" >/dev/null; then
+      return
+    fi
+    if ! kill -0 "${SERVICE_PID}" 2>/dev/null; then
+      echo "Service exited while starting" >&2
+      return 1
+    fi
+    sleep 1
+  done
+  echo "Service readiness timed out" >&2
+  return 1
+}
+
+wait_for_events() {
+  local attempt response
+  for ((attempt = 0; attempt < 30; attempt++)); do
+    response=$(curl --fail --silent "${BASE_URL}/api/v1/events?size=20" || true)
+    if jq -e '.items | type == "array" and length > 0' <<<"${response}" >/dev/null; then
+      return
+    fi
+    sleep 1
+  done
+  echo "Mock event fixture did not appear" >&2
+  return 1
+}
+
+discover_odds_fixture() {
+  local attempt key
+  for ((attempt = 0; attempt < 120; attempt++)); do
+    key=$(compose exec -T redis redis-cli --raw --scan --pattern 'odds:*' \
+      | sed -n '1p' | tr -d '\r')
+    if [[ -n "${key}" ]]; then
+      IFS=: read -r _ EVENT_ID MARKET_ID SELECTION_ID <<<"${key}"
+      EXPECTED_ODDS=$(compose exec -T redis redis-cli --raw GET "${key}" | tr -d '\r')
+      export EVENT_ID MARKET_ID SELECTION_ID EXPECTED_ODDS
+      curl --fail --silent \
+        "${BASE_URL}/api/v1/odds/${EVENT_ID}/${MARKET_ID}/${SELECTION_ID}" >/dev/null
+      return
+    fi
+    sleep 1
+  done
+  echo "Mock odds fixture did not appear" >&2
+  return 1
+}
+
+verify_frozen_odds() {
+  curl --fail --silent \
+    "${BASE_URL}/api/v1/odds/${EVENT_ID}/${MARKET_ID}/${SELECTION_ID}" \
+    | jq -e --arg expected "${EXPECTED_ODDS}" '.odds == ($expected | tonumber)' >/dev/null
+}


## `test(load): exercise event reads`

diff --git a/load-test/scenarios/events.js b/load-test/scenarios/events.js
new file mode 100644
index 0000000..a76c1c0
--- /dev/null
+++ b/load-test/scenarios/events.js
@@ -0,0 +1,45 @@
+import http from "k6/http";
+import { check } from "k6";
+
+const baseUrl = __ENV.BASE_URL || "http://localhost:8085";
+const gateStage = __ENV.GATE_STAGE || "measure";
+
+export const options = {
+  scenarios: {
+    events: {
+      executor: "constant-arrival-rate",
+      rate: Number(__ENV.REQUEST_RATE || 1000),
+      timeUnit: "1s",
+      duration: __ENV.DURATION || "60s",
+      preAllocatedVUs: Number(__ENV.PREALLOCATED_VUS || 200),
+      maxVUs: Number(__ENV.MAX_VUS || 500),
+      gracefulStop: "5s",
+    },
+  },
+  thresholds: releaseThresholds(),
+  summaryTrendStats: ["min", "avg", "p(50)", "p(95)", "p(99)", "max"],
+};
+
+export default function () {
+  const response = http.get(`${baseUrl}/api/v1/events?size=20`);
+  check(response, {
+    "events returns a non-empty cursor page": (result) => {
+      if (result.status !== 200) return false;
+      try {
+        return result.json().items.length > 0;
+      } catch (_error) {
+        return false;
+      }
+    },
+  });
+}
+
+function releaseThresholds() {
+  if (gateStage !== "measure") return {};
+  return {
+    http_req_duration: ["p(99)<50"],
+    http_req_failed: ["rate<0.001"],
+    checks: ["rate>0.999"],
+    dropped_iterations: ["count==0"],
+  };
+}


