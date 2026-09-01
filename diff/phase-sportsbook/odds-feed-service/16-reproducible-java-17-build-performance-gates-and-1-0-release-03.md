## `test(load): exercise odds reads`

diff --git a/load-test/scenarios/odds.js b/load-test/scenarios/odds.js
new file mode 100644
index 0000000..b0f1d83
--- /dev/null
+++ b/load-test/scenarios/odds.js
@@ -0,0 +1,63 @@
+import http from "k6/http";
+import { check } from "k6";
+
+const baseUrl = __ENV.BASE_URL || "http://localhost:8085";
+const eventId = required("EVENT_ID");
+const marketId = required("MARKET_ID");
+const selectionId = required("SELECTION_ID");
+const expectedOdds = Number(required("EXPECTED_ODDS"));
+const gateStage = __ENV.GATE_STAGE || "measure";
+
+export const options = {
+  scenarios: {
+    odds: {
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
+  const response = http.get(
+    `${baseUrl}/api/v1/odds/${eventId}/${marketId}/${selectionId}`,
+  );
+  check(response, {
+    "odds returns the frozen selection": (result) => {
+      if (result.status !== 200) return false;
+      try {
+        const body = result.json();
+        return (
+          body.eventId === eventId &&
+          body.marketId === marketId &&
+          body.selectionId === selectionId &&
+          Number(body.odds) === expectedOdds
+        );
+      } catch (_error) {
+        return false;
+      }
+    },
+  });
+}
+
+function required(name) {
+  const value = __ENV[name];
+  if (!value) throw new Error(`${name} is required`);
+  return value;
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


## `test(load): repeat endpoint measurements`

diff --git a/load-test/lib/k6-gate.sh b/load-test/lib/k6-gate.sh
new file mode 100755
index 0000000..fe1a09a
--- /dev/null
+++ b/load-test/lib/k6-gate.sh
@@ -0,0 +1,23 @@
+#!/usr/bin/env bash
+
+run_endpoint_gate() {
+  local endpoint=$1
+  local scenario="${SCRIPT_DIR}/scenarios/${endpoint}.js"
+  local output_dir="${RESULT_ROOT}/${endpoint}"
+  local run
+  mkdir "${output_dir}"
+
+  echo "Warming ${endpoint} for 60 seconds"
+  GATE_STAGE=warmup DURATION=60s BASE_URL="${BASE_URL}" REQUEST_RATE="${REQUEST_RATE}" \
+    EVENT_ID="${EVENT_ID:-}" MARKET_ID="${MARKET_ID:-}" SELECTION_ID="${SELECTION_ID:-}" \
+    EXPECTED_ODDS="${EXPECTED_ODDS:-}" \
+    k6 run --quiet --summary-export "${output_dir}/warmup.json" "${scenario}"
+
+  for run in 1 2 3 4 5; do
+    echo "Measuring ${endpoint}: ${run}/5"
+    GATE_STAGE=measure DURATION=60s BASE_URL="${BASE_URL}" REQUEST_RATE="${REQUEST_RATE}" \
+      EVENT_ID="${EVENT_ID:-}" MARKET_ID="${MARKET_ID:-}" SELECTION_ID="${SELECTION_ID:-}" \
+      EXPECTED_ODDS="${EXPECTED_ODDS:-}" \
+      k6 run --quiet --summary-export "${output_dir}/measure-${run}.json" "${scenario}"
+  done
+}


## `test(load): orchestrate the HTTP release gate`

diff --git a/load-test/run-http-gate.sh b/load-test/run-http-gate.sh
new file mode 100755
index 0000000..4ebb73c
--- /dev/null
+++ b/load-test/run-http-gate.sh
@@ -0,0 +1,86 @@
+#!/usr/bin/env bash
+
+set -euo pipefail
+
+SCRIPT_DIR=$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd -P)
+REPO_ROOT=$(cd "${SCRIPT_DIR}/.." && pwd -P)
+COMPOSE_FILE="${SCRIPT_DIR}/docker-compose.yml"
+RESULT_ROOT=${RESULT_ROOT:?Set RESULT_ROOT to a new directory outside the repository}
+SERVER_PORT=${SERVER_PORT:-8085}
+REDIS_PORT=${REDIS_PORT:-6392}
+KAFKA_PORT=${KAFKA_PORT:-9096}
+REQUEST_RATE=${REQUEST_RATE:-1000}
+COMPOSE_PROJECT_NAME=${COMPOSE_PROJECT_NAME:-odds-feed-http-gate}
+MAVEN_REPO_LOCAL=${MAVEN_REPO_LOCAL:-}
+BASE_URL="http://localhost:${SERVER_PORT}"
+SERVICE_PID=''
+MAVEN_REPO_OPTION=()
+[[ -z "${MAVEN_REPO_LOCAL}" ]] || MAVEN_REPO_OPTION=(-Dmaven.repo.local="${MAVEN_REPO_LOCAL}")
+
+source "${SCRIPT_DIR}/fixtures/mock.env"
+source "${SCRIPT_DIR}/lib/runtime.sh"
+source "${SCRIPT_DIR}/fixtures/prepare-mock.sh"
+source "${SCRIPT_DIR}/lib/k6-gate.sh"
+
+for command in curl docker java jq k6 openssl sed tr; do
+  command -v "${command}" >/dev/null || {
+    echo "Missing required command: ${command}" >&2
+    exit 2
+  }
+done
+
+[[ "${RESULT_ROOT}" == /* ]] || {
+  echo "RESULT_ROOT must be absolute" >&2
+  exit 2
+}
+RESULT_ROOT=${RESULT_ROOT%/}
+[[ -n "${RESULT_ROOT}" ]] || {
+  echo "RESULT_ROOT cannot be the filesystem root" >&2
+  exit 2
+}
+RESULT_PARENT=$(cd "$(dirname "${RESULT_ROOT}")" && pwd -P) || {
+  echo "RESULT_ROOT parent must already exist" >&2
+  exit 2
+}
+RESULT_ROOT="${RESULT_PARENT}/$(basename "${RESULT_ROOT}")"
+[[ ! -e "${RESULT_ROOT}" ]] || {
+  echo "RESULT_ROOT must not already exist" >&2
+  exit 2
+}
+case "${RESULT_ROOT}/" in
+  "${REPO_ROOT}/"*)
+    echo "RESULT_ROOT must be outside the repository" >&2
+    exit 2
+    ;;
+esac
+mkdir "${RESULT_ROOT}"
+trap cleanup_runtime EXIT INT TERM
+
+cd "${REPO_ROOT}"
+ADMIN_API_INTERNAL_KEY=$(openssl rand -hex 32)
+export ADMIN_API_INTERNAL_KEY
+./mvnw "${MAVEN_REPO_OPTION[@]}" -B -ntp clean verify
+FINAL_NAME=$(./mvnw "${MAVEN_REPO_OPTION[@]}" -q -Dstyle.color=never \
+  -DforceStdout help:evaluate -Dexpression=project.build.finalName)
+JAR_PATH="${REPO_ROOT}/target/${FINAL_NAME}.jar"
+[[ -f "${JAR_PATH}" ]] || {
+  echo "Executable jar not found: ${JAR_PATH}" >&2
+  exit 2
+}
+
+reset_stack
+start_service events "${FIXTURE_FROZEN_TICK_INTERVAL_MS}"
+wait_for_service
+wait_for_events
+run_endpoint_gate events
+stop_service
+
+reset_stack
+start_service odds "${FIXTURE_FROZEN_TICK_INTERVAL_MS}"
+wait_for_service
+wait_for_events
+discover_odds_fixture
+verify_frozen_odds
+run_endpoint_gate odds
+
+echo "HTTP release gate passed; results: ${RESULT_ROOT}"


## `ci(service): verify Java 17 builds`

diff --git a/.github/workflows/ci.yml b/.github/workflows/ci.yml
new file mode 100644
index 0000000..1f500bc
--- /dev/null
+++ b/.github/workflows/ci.yml
@@ -0,0 +1,45 @@
+name: ci
+
+on:
+  pull_request:
+  push:
+    branches: [odds-feed-service]
+
+permissions:
+  contents: read
+
+jobs:
+  verify:
+    runs-on: ubuntu-latest
+    steps:
+      - name: Check out odds feed
+        uses: actions/checkout@v4
+        with:
+          path: odds-feed
+
+      - name: Check out shared protocol
+        uses: actions/checkout@v4
+        with:
+          repository: ${{ github.repository }}
+          ref: shared-protocol
+          path: shared-protocol
+
+      - name: Set up Java 17
+        uses: actions/setup-java@v4
+        with:
+          distribution: temurin
+          java-version: "17"
+          cache: maven
+
+      - name: Install shared protocol 1.0.0
+        working-directory: shared-protocol
+        run: |
+          test "$(./mvnw -q -Dstyle.color=never -DforceStdout help:evaluate -Dexpression=project.version)" = "1.0.0"
+          ./mvnw -B -ntp clean install
+
+      - name: Verify odds feed
+        working-directory: odds-feed
+        run: |
+          ADMIN_API_INTERNAL_KEY="$(openssl rand -hex 32)"
+          export ADMIN_API_INTERNAL_KEY
+          ./mvnw -B -ntp clean verify


## `build(release): release odds feed service 1.0.0`

diff --git a/pom.xml b/pom.xml
index d430885..02e138b 100644
--- a/pom.xml
+++ b/pom.xml
@@ -13,7 +13,7 @@
 
     <groupId>com.sportsbook</groupId>
     <artifactId>odds-feed-service</artifactId>
-    <version>1.0.0-SNAPSHOT</version>
+    <version>1.0.0</version>
     <packaging>jar</packaging>
 
     <name>odds-feed-service</name>


