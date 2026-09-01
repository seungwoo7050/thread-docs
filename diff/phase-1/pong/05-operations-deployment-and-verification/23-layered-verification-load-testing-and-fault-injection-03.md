## `test(load): 실시간 fault injection 도구 추가`

diff --git a/docker-compose.load.yml b/docker-compose.load.yml
new file mode 100644
index 0000000..54055aa
--- /dev/null
+++ b/docker-compose.load.yml
@@ -0,0 +1,27 @@
+services:
+  toxiproxy:
+    image: ghcr.io/shopify/toxiproxy:2.12.0
+    ports:
+      - "127.0.0.1:${TOXIPROXY_CONTROL_PORT:-8474}:8474"
+      - "127.0.0.1:${TOXIPROXY_EDGE_PORT:-18080}:18080"
+
+  toxiproxy-bootstrap:
+    image: node:24.18.0-alpine
+    command: ["node", "/load/toxiproxy-control.mjs", "ensure"]
+    environment:
+      TOXIPROXY_API_URL: http://toxiproxy:8474
+    volumes:
+      - ./tests/load:/load:ro
+    depends_on:
+      db:
+        condition: service_healthy
+      toxiproxy:
+        condition: service_started
+    restart: "no"
+
+  api:
+    environment:
+      DATABASE_URL: postgres://pong:${POSTGRES_PASSWORD:-pong}@toxiproxy:15432/pong_pong
+    depends_on:
+      toxiproxy-bootstrap:
+        condition: service_completed_successfully
diff --git a/tests/load/load-profile.mjs b/tests/load/load-profile.mjs
new file mode 100644
index 0000000..50d6b17
--- /dev/null
+++ b/tests/load/load-profile.mjs
@@ -0,0 +1,72 @@
+const DEFAULT_CONNECTIONS = 500;
+const EXTENDED_CONNECTIONS = 1_000;
+const DEFAULT_ROOMS = 50;
+
+export function createLoadProfile(environment = {}) {
+  const extended = environment.EXTENDED_LOAD === "1";
+  const connections = positiveInteger(
+    "CONNECTIONS",
+    environment.CONNECTIONS,
+    extended ? EXTENDED_CONNECTIONS : DEFAULT_CONNECTIONS
+  );
+  const rooms = positiveInteger("ROOMS", environment.ROOMS, DEFAULT_ROOMS);
+  if (connections < rooms * 2) {
+    throw new RangeError("CONNECTIONS must be at least twice the room count");
+  }
+
+  const minimumSuccessfulConnections = Math.ceil(connections * 0.99);
+  const playerConnections = rooms * 2;
+  const initialHoldMs = positiveInteger("INITIAL_HOLD_MS", environment.INITIAL_HOLD_MS, 90_000);
+  const playerReconnectDelayMs = positiveInteger(
+    "PLAYER_RECONNECT_DELAY_MS",
+    environment.PLAYER_RECONNECT_DELAY_MS,
+    10_000
+  );
+  const reconnectedHoldMs = positiveInteger(
+    "RECONNECTED_HOLD_MS",
+    environment.RECONNECTED_HOLD_MS,
+    60_000
+  );
+  const maxDuration = environment.MAX_DURATION || "4m";
+
+  return {
+    connections,
+    rooms,
+    playerConnections,
+    minimumSuccessfulConnections,
+    initialHoldMs,
+    playerReconnectDelayMs,
+    reconnectedHoldMs,
+    options: {
+      discardResponseBodies: true,
+      scenarios: {
+        pong: {
+          executor: "per-vu-iterations",
+          vus: connections,
+          iterations: 1,
+          maxDuration
+        }
+      },
+      thresholds: {
+        connection_success: ["rate>=0.99"],
+        reconnect_success: ["rate>=0.99"],
+        snapshot_delay_ms: ["p(95)<=150", "p(99)<=250"],
+        normal_snapshot_drop_rate: ["rate<0.01"],
+        finalize_results: [`count>=${rooms}`],
+        finalize_failures: ["count==0"],
+        finalize_duplicates: ["count==0"],
+        online_connections: [`max>=${minimumSuccessfulConnections}`],
+        active_rooms: [`max>=${rooms}`]
+      },
+      summaryTrendStats: ["avg", "min", "med", "max", "p(90)", "p(95)", "p(99)"]
+    }
+  };
+}
+
+function positiveInteger(name, rawValue, fallback) {
+  const value = rawValue === undefined || rawValue === "" ? fallback : Number(rawValue);
+  if (!Number.isSafeInteger(value) || value <= 0) {
+    throw new RangeError(`${name} must be a positive integer`);
+  }
+  return value;
+}
diff --git a/tests/load/pong-load.js b/tests/load/pong-load.js
new file mode 100644
index 0000000..4dc5138
--- /dev/null
+++ b/tests/load/pong-load.js
@@ -0,0 +1,239 @@
+import exec from "k6/execution";
+import http from "k6/http";
+import ws from "k6/ws";
+import { check, fail } from "k6";
+import { Counter, Rate, Trend } from "k6/metrics";
+import { createLoadProfile } from "./load-profile.mjs";
+
+const profile = createLoadProfile(__ENV);
+const apiBaseUrl = (__ENV.API_BASE_URL || "http://127.0.0.1:4000").replace(/\/$/, "");
+const websocketUrl = __ENV.WS_URL || "ws://127.0.0.1:4000/ws";
+
+const connectionSuccess = new Rate("connection_success");
+const reconnectSuccess = new Rate("reconnect_success");
+const snapshotDelay = new Trend("snapshot_delay_ms");
+const normalSnapshotDropRate = new Rate("normal_snapshot_drop_rate");
+const finalizeResults = new Counter("finalize_results");
+const finalizeFailures = new Counter("finalize_failures");
+const finalizeDuplicates = new Counter("finalize_duplicates");
+const onlineConnections = new Trend("online_connections");
+const activeRooms = new Trend("active_rooms");
+
+export const options = profile.options;
+
+export function setup() {
+  const response = http.get(`${apiBaseUrl}${__ENV.READY_PATH || "/health/ready"}`, {
+    tags: { operation: "readiness" }
+  });
+  if (response.status !== 200) {
+    fail(`API readiness failed with ${response.status}`);
+  }
+}
+
+export default function () {
+  const vuId = exec.vu.idInTest;
+  const player = vuId <= profile.playerConnections;
+  const finishedMatchIds = new Set();
+  const finishedRoomIds = new Set();
+  finalizeFailures.add(0);
+  finalizeDuplicates.add(0);
+
+  if (!login(vuId)) {
+    connectionSuccess.add(false);
+    return;
+  }
+
+  const initialTicket = issueTicket();
+  if (!initialTicket) {
+    connectionSuccess.add(false);
+    return;
+  }
+
+  let initial;
+  try {
+    initial = connectSession({
+      ticket: initialTicket,
+      phase: "initial",
+      player,
+      expectedRoomId: null,
+      finishedMatchIds,
+      finishedRoomIds
+    });
+  } catch {
+    connectionSuccess.add(false);
+    return;
+  }
+  connectionSuccess.add(initial.connected);
+
+  if (!player) return;
+  if (!initial.connected || !initial.roomId) {
+    reconnectSuccess.add(false);
+    return;
+  }
+
+  const reconnectTicket = issueTicket();
+  if (!reconnectTicket) {
+    reconnectSuccess.add(false);
+    return;
+  }
+
+  let reconnected;
+  try {
+    reconnected = connectSession({
+      ticket: reconnectTicket,
+      phase: "reconnect",
+      player: true,
+      expectedRoomId: initial.roomId,
+      finishedMatchIds,
+      finishedRoomIds
+    });
+  } catch {
+    reconnectSuccess.add(false);
+    return;
+  }
+  reconnectSuccess.add(reconnected.connected && reconnected.recovered);
+}
+
+function login(vuId) {
+  const response = http.request(
+    "POST",
+    `${apiBaseUrl}/auth/dev-login`,
+    JSON.stringify({
+      handle: `load-user-${vuId}`,
+      displayName: `부하 테스트 ${vuId}`
+    }),
+    {
+      headers: { "content-type": "application/json" },
+      tags: { operation: "dev-login" }
+    }
+  );
+  return check(response, { "development login succeeds": (value) => value.status === 200 });
+}
+
+function issueTicket() {
+  const response = http.request("POST", `${apiBaseUrl}/auth/ws-ticket`, null, {
+    responseType: "text",
+    tags: { operation: "ws-ticket" }
+  });
+  if (response.status !== 200 || !response.body) return null;
+  try {
+    const body = JSON.parse(response.body);
+    return body.protocolVersion === 1 && typeof body.ticket === "string" ? body.ticket : null;
+  } catch {
+    return null;
+  }
+}
+
+function connectSession({ ticket, phase, player, expectedRoomId, finishedMatchIds, finishedRoomIds }) {
+  const result = {
+    connected: false,
+    recovered: expectedRoomId === null,
+    roomId: expectedRoomId,
+    side: null
+  };
+  let queueJoined = false;
+  let inputSeq = 0;
+  let lastSequence = null;
+  let reconnectCloseArmed = false;
+
+  const response = ws.connect(
+    `${websocketUrl}?ticket=${encodeURIComponent(ticket)}&v=1`,
+    { tags: { phase, player: String(player) } },
+    (socket) => {
+      socket.on("open", () => {
+        result.connected = true;
+      });
+      socket.on("message", (payload) => {
+        let event;
+        try {
+          event = JSON.parse(payload);
+        } catch {
+          return;
+        }
+        if (event.v !== 1) return;
+
+        if (event.type === "presence.changed") {
+          onlineConnections.add(event.online);
+          activeRooms.add(event.playing / 2);
+          if (
+            phase === "initial"
+            && player
+            && !queueJoined
+            && event.online >= profile.minimumSuccessfulConnections
+          ) {
+            queueJoined = true;
+            socket.send(JSON.stringify({ v: 1, type: "queue.join", mode: "queue" }));
+          }
+          return;
+        }
+
+        if (event.type === "queue.matched") {
+          result.roomId = event.roomId;
+          result.side = event.side;
+          if (expectedRoomId !== null && event.roomId === expectedRoomId) result.recovered = true;
+          socket.send(JSON.stringify({ v: 1, type: "game.ready", roomId: event.roomId }));
+          if (!reconnectCloseArmed && phase === "initial") {
+            reconnectCloseArmed = true;
+            socket.setTimeout(() => socket.close(), profile.playerReconnectDelayMs);
+          }
+          socket.setInterval(() => {
+            inputSeq += 1;
+            const direction = inputSeq % 30 < 10 ? -1 : inputSeq % 30 < 20 ? 1 : 0;
+            socket.send(JSON.stringify({
+              v: 1,
+              type: "game.input",
+              roomId: event.roomId,
+              inputSeq,
+              direction
+            }));
+          }, 100);
+          return;
+        }
+
+        if (event.type === "game.snapshot") {
+          const snapshot = event.snapshot;
+          if (expectedRoomId !== null && snapshot.roomId === expectedRoomId) result.recovered = true;
+          snapshotDelay.add(Math.max(0, Date.now() - snapshot.serverTimeMs));
+          if (lastSequence !== null && snapshot.sequence > lastSequence) {
+            const missed = snapshot.sequence - lastSequence - 1;
+            for (let index = 0; index < missed; index += 1) normalSnapshotDropRate.add(true);
+            normalSnapshotDropRate.add(false);
+          } else if (lastSequence === null) {
+            normalSnapshotDropRate.add(false);
+          }
+          if (lastSequence === null || snapshot.sequence > lastSequence) {
+            lastSequence = snapshot.sequence;
+          }
+          return;
+        }
+
+        if (event.type === "game.finished" && result.side === "left") {
+          const matchId = event.result?.matchId;
+          const roomId = event.result?.roomId;
+          if (
+            event.result?.persisted !== true
+            || typeof matchId !== "string"
+            || matchId.length === 0
+            || typeof roomId !== "string"
+            || roomId !== result.roomId
+          ) {
+            finalizeFailures.add(1);
+          } else if (finishedMatchIds.has(matchId) || finishedRoomIds.has(roomId)) {
+            finalizeDuplicates.add(1);
+          } else {
+            finishedMatchIds.add(matchId);
+            finishedRoomIds.add(roomId);
+            finalizeResults.add(1);
+          }
+        }
+      });
+      socket.setTimeout(
+        () => socket.close(),
+        phase === "initial" ? profile.initialHoldMs : profile.reconnectedHoldMs
+      );
+    }
+  );
+
+  result.connected = result.connected && response?.status === 101;
+  return result;
+}
diff --git a/tests/load/toxiproxy-control.mjs b/tests/load/toxiproxy-control.mjs
new file mode 100644
index 0000000..d84a1d6
--- /dev/null
+++ b/tests/load/toxiproxy-control.mjs
@@ -0,0 +1,192 @@
+import { pathToFileURL } from "node:url";
+
+const DEFAULT_API_URL = "http://127.0.0.1:8474";
+const COMMANDS = new Set([
+  "plan",
+  "ensure",
+  "reset",
+  "db-latency",
+  "db-down",
+  "db-up",
+  "edge-latency",
+  "edge-reset",
+  "edge-down",
+  "edge-up"
+]);
+
+export function buildProxyDefinitions(environment = {}) {
+  return [
+    {
+      name: "postgres",
+      listen: environment.TOXIPROXY_POSTGRES_LISTEN || "0.0.0.0:15432",
+      upstream: environment.TOXIPROXY_POSTGRES_UPSTREAM || "db:5432",
+      enabled: true
+    },
+    {
+      name: "edge",
+      listen: environment.TOXIPROXY_EDGE_LISTEN || "0.0.0.0:18080",
+      upstream: environment.TOXIPROXY_EDGE_UPSTREAM || "caddy:8080",
+      enabled: true
+    }
+  ];
+}
+
+export function toxicForCommand(command, args = []) {
+  if (command === "db-latency" || command === "edge-latency") {
+    const latency = positiveInteger(args[0] ?? "250");
+    const jitter = nonnegativeInteger(args[1] ?? "25");
+    const proxy = command.startsWith("db-") ? "postgres" : "edge";
+    return {
+      proxy,
+      toxic: {
+        name: command,
+        type: "latency",
+        stream: "downstream",
+        toxicity: 1,
+        attributes: { latency, jitter }
+      }
+    };
+  }
+  if (command === "edge-reset") {
+    return {
+      proxy: "edge",
+      toxic: {
+        name: command,
+        type: "reset_peer",
+        stream: "downstream",
+        toxicity: 1,
+        attributes: { timeout: nonnegativeInteger(args[0] ?? "0") }
+      }
+    };
+  }
+  throw new RangeError(`command does not define a toxic: ${command}`);
+}
+
+export async function runCommand(command, args = [], environment = process.env) {
+  if (!COMMANDS.has(command)) throw new RangeError(`unknown command: ${command}`);
+  const apiUrl = (environment.TOXIPROXY_API_URL || DEFAULT_API_URL).replace(/\/$/, "");
+  const proxies = buildProxyDefinitions(environment);
+  if (command === "plan") return { apiUrl, proxies };
+
+  await waitForApi(apiUrl);
+  await ensureProxies(apiUrl, proxies);
+  if (command === "ensure") return { apiUrl, proxies };
+  if (command === "reset") {
+    for (const proxy of proxies) {
+      await removeAllToxics(apiUrl, proxy.name);
+      await setEnabled(apiUrl, proxy.name, true);
+    }
+    return { reset: proxies.map((proxy) => proxy.name) };
+  }
+  if (command.endsWith("-down")) {
+    const proxy = command.startsWith("db-") ? "postgres" : "edge";
+    await setEnabled(apiUrl, proxy, false);
+    return { proxy, enabled: false };
+  }
+  if (command.endsWith("-up")) {
+    const proxy = command.startsWith("db-") ? "postgres" : "edge";
+    await removeAllToxics(apiUrl, proxy);
+    await setEnabled(apiUrl, proxy, true);
+    return { proxy, enabled: true };
+  }
+
+  const planned = toxicForCommand(command, args);
+  await removeAllToxics(apiUrl, planned.proxy);
+  await requestJson(apiUrl, `/proxies/${planned.proxy}/toxics`, {
+    method: "POST",
+    body: planned.toxic
+  });
+  return planned;
+}
+
+async function waitForApi(apiUrl) {
+  let lastError;
+  for (let attempt = 0; attempt < 30; attempt += 1) {
+    try {
+      await requestJson(apiUrl, "/version");
+      return;
+    } catch (error) {
+      lastError = error;
+      await new Promise((resolve) => setTimeout(resolve, 1_000));
+    }
+  }
+  throw lastError ?? new Error("Toxiproxy API did not become ready");
+}
+
+async function ensureProxies(apiUrl, definitions) {
+  const existing = await requestJson(apiUrl, "/proxies");
+  for (const definition of definitions) {
+    if (existing[definition.name]) {
+      await requestJson(apiUrl, `/proxies/${definition.name}`, {
+        method: "POST",
+        body: definition
+      });
+    } else {
+      await requestJson(apiUrl, "/proxies", { method: "POST", body: definition });
+    }
+  }
+}
+
+async function removeAllToxics(apiUrl, proxy) {
+  const toxics = await requestJson(apiUrl, `/proxies/${proxy}/toxics`);
+  for (const toxic of toxics) await removeToxic(apiUrl, proxy, toxic.name);
+}
+
+async function removeToxic(apiUrl, proxy, name) {
+  await requestJson(apiUrl, `/proxies/${proxy}/toxics/${name}`, {
+    method: "DELETE",
+    allowNotFound: true
+  });
+}
+
+async function setEnabled(apiUrl, proxy, enabled) {
+  const current = await requestJson(apiUrl, `/proxies/${proxy}`);
+  await requestJson(apiUrl, `/proxies/${proxy}`, {
+    method: "POST",
+    body: {
+      name: current.name,
+      listen: current.listen,
+      upstream: current.upstream,
+      enabled
+    }
+  });
+}
+
+async function requestJson(apiUrl, path, options = {}) {
+  const response = await fetch(`${apiUrl}${path}`, {
+    method: options.method ?? "GET",
+    headers: options.body ? { "content-type": "application/json" } : undefined,
+    body: options.body ? JSON.stringify(options.body) : undefined
+  });
+  if (options.allowNotFound && response.status === 404) return null;
+  if (!response.ok) {
+    const detail = await response.text();
+    throw new Error(`${options.method ?? "GET"} ${path} failed (${response.status}): ${detail}`);
+  }
+  if (response.status === 204) return null;
+  const text = await response.text();
+  return text ? JSON.parse(text) : null;
+}
+
+function positiveInteger(rawValue) {
+  const value = Number(rawValue);
+  if (!Number.isSafeInteger(value) || value <= 0) throw new RangeError("value must be a positive integer");
+  return value;
+}
+
+function nonnegativeInteger(rawValue) {
+  const value = Number(rawValue);
+  if (!Number.isSafeInteger(value) || value < 0) throw new RangeError("value must be a nonnegative integer");
+  return value;
+}
+
+const invokedAsScript = process.argv[1] && import.meta.url === pathToFileURL(process.argv[1]).href;
+if (invokedAsScript) {
+  const [command = "plan", ...args] = process.argv.slice(2);
+  runCommand(command, args)
+    .then((result) => process.stdout.write(`${JSON.stringify(result, null, 2)}\n`))
+    .catch((error) => {
+      process.stderr.write(`${error instanceof Error ? error.message : String(error)}\n`);
+      process.exitCode = 1;
+    });
+}


## `ci(repo): process와 browser 검증 job 추가`

diff --git a/.github/workflows/ci.yml b/.github/workflows/ci.yml
index e6fcb48..73e472a 100644
--- a/.github/workflows/ci.yml
+++ b/.github/workflows/ci.yml
@@ -68,3 +68,92 @@ jobs:
 
       - name: Run PostgreSQL integration tests
         run: pnpm postgres-integration
+
+  process-and-browser:
+    name: HTTP, WebSocket, and browser tests
+    runs-on: ubuntu-latest
+    timeout-minutes: 20
+    services:
+      postgres:
+        image: postgres:16-alpine
+        env:
+          POSTGRES_DB: pong_pong_test
+          POSTGRES_USER: pong
+          POSTGRES_PASSWORD: pong
+        ports:
+          - 5432:5432
+        options: >-
+          --health-cmd "pg_isready -U pong -d pong_pong_test"
+          --health-interval 5s
+          --health-timeout 3s
+          --health-retries 20
+    env:
+      DATABASE_URL: postgresql://pong:pong@127.0.0.1:5432/pong_pong_test
+      API_BASE_URL: http://127.0.0.1:4000
+      WS_URL: ws://127.0.0.1:4000/ws
+
+    steps:
+      - name: Check out repository
+        uses: actions/checkout@v4
+
+      - name: Set up pnpm
+        uses: pnpm/action-setup@v4
+        with:
+          version: 10.32.1
+
+      - name: Set up Node.js
+        uses: actions/setup-node@v4
+        with:
+          node-version: 24.18.0
+          cache: pnpm
+
+      - name: Install dependencies
+        run: pnpm install --frozen-lockfile
+
+      - name: Install Chromium
+        run: pnpm exec playwright install --with-deps chromium
+
+      - name: Build production processes
+        run: pnpm build
+
+      - name: Prepare the process-test database
+        run: |
+          pnpm --filter @pong-pong/db migrate
+          pnpm --filter @pong-pong/db seed:dev
+
+      - name: Run process smoke and browser tests
+        env:
+          APP_MODE: development
+          SESSION_SECRET: ci-process-session-secret-32-bytes
+          LOG_LEVEL: warn
+        run: |
+          set -euo pipefail
+          node apps/api/dist/index.js >/tmp/pong-api.log 2>&1 &
+          API_PID=$!
+          pnpm --filter @pong-pong/web start >/tmp/pong-web.log 2>&1 &
+          WEB_PID=$!
+          cleanup() {
+            kill "$API_PID" "$WEB_PID" 2>/dev/null || true
+            wait "$API_PID" "$WEB_PID" 2>/dev/null || true
+          }
+          trap cleanup EXIT
+
+          for attempt in $(seq 1 60); do
+            if curl --fail --silent http://127.0.0.1:4000/health/ready >/dev/null; then
+              break
+            fi
+            sleep 1
+          done
+          curl --fail --silent http://127.0.0.1:4000/health/ready >/dev/null
+
+          for attempt in $(seq 1 60); do
+            if curl --fail --silent http://127.0.0.1:3000 >/dev/null; then
+              break
+            fi
+            sleep 1
+          done
+          curl --fail --silent http://127.0.0.1:3000 >/dev/null
+
+          pnpm smoke:http
+          pnpm smoke:ws
+          pnpm e2e


