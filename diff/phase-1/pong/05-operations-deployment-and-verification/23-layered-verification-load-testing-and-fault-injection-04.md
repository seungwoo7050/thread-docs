## `test(e2e): 비회원 체험 브라우저 흐름 검증`

diff --git a/package.json b/package.json
index 33246cc..9f3762c 100644
--- a/package.json
+++ b/package.json
@@ -17,6 +17,7 @@
     "smoke:ws": "node tests/smoke-ws.mjs",
     "verify:build": "node tests/build-artifacts.mjs",
     "e2e": "playwright test",
+    "e2e:guest-demo": "E2E_APP_MODE=demo playwright test tests/e2e/guest-demo.spec.ts --workers=1",
     "test:e2e": "pnpm e2e",
     "dev": "pnpm --parallel --filter @pong-pong/api --filter @pong-pong/web dev"
   },
diff --git a/tests/e2e/guest-demo.spec.ts b/tests/e2e/guest-demo.spec.ts
new file mode 100644
index 0000000..4c433e6
--- /dev/null
+++ b/tests/e2e/guest-demo.spec.ts
@@ -0,0 +1,156 @@
+import {
+  expect,
+  test,
+  type Browser,
+  type BrowserContext,
+  type Page,
+  type WebSocketRoute
+} from "@playwright/test";
+
+const baseURL = process.env.E2E_BASE_URL ?? "http://localhost:3000";
+const demoMode = process.env.E2E_APP_MODE === "demo";
+
+test.describe("게스트 데모 브라우저 흐름", () => {
+  test.describe.configure({ mode: "serial" });
+  test.skip(!demoMode, "APP_MODE=demo 서버를 대상으로 별도 실행한다.");
+
+  test("입력 없이 게스트로 진입하고 제한된 메뉴만 보여 준다", async ({ page }) => {
+    const displayName = await enterAsGuest(page);
+
+    await expect(page.getByRole("heading", { name: `다시 오신 것을 환영합니다, ${displayName}` })).toBeVisible();
+    await expect(page.getByRole("navigation").getByRole("link")).toHaveText(["로비", "경기"]);
+    await expect(page.getByRole("link", { name: "관리" })).toHaveCount(0);
+    await expect(page.getByText("빠른 매칭으로 다른 게스트를 찾고, 상대가 없으면 인공지능과 바로 경기할 수 있습니다.")).toBeVisible();
+  });
+
+  test("서로 다른 두 게스트를 같은 PvP 방에 연결한다", async ({ browser }, testInfo) => {
+    test.skip(testInfo.project.name !== "chromium-desktop", "두 브라우저 흐름은 desktop 프로젝트에서 한 번만 실행한다.");
+
+    const left = await createGuestPage(browser);
+    const right = await createGuestPage(browser);
+    try {
+      const [leftName, rightName] = await Promise.all([
+        enterAsGuest(left.page),
+        enterAsGuest(right.page)
+      ]);
+
+      await Promise.all([
+        openPlayPage(left.page),
+        openPlayPage(right.page)
+      ]);
+      await Promise.all([
+        left.page.getByRole("button", { name: "매칭 큐 참가" }).click(),
+        right.page.getByRole("button", { name: "매칭 큐 참가" }).click()
+      ]);
+
+      await expect(left.page.getByText("준비 대기 중")).toBeVisible();
+      await expect(right.page.getByText("준비 대기 중")).toBeVisible();
+      await expect(left.page.getByText(rightName, { exact: true })).toBeVisible();
+      await expect(right.page.getByText(leftName, { exact: true })).toBeVisible();
+
+      await Promise.all([
+        left.page.getByRole("button", { name: "준비" }).click(),
+        right.page.getByRole("button", { name: "준비" }).click()
+      ]);
+      await expect(left.page.getByText("경기 진행 중")).toBeVisible();
+      await expect(right.page.getByText("경기 진행 중")).toBeVisible();
+    } finally {
+      await Promise.all([left.context.close(), right.context.close()]);
+    }
+  });
+
+  test("대기 중인 게스트를 6초 뒤 AI 방으로 옮긴다", async ({ page }, testInfo) => {
+    test.skip(testInfo.project.name !== "chromium-desktop", "시간 검증은 desktop 프로젝트에서 한 번만 실행한다.");
+
+    await enterAsGuest(page);
+    await openPlayPage(page);
+    const frames = watchJsonFrames(page);
+    await page.getByRole("button", { name: "매칭 큐 참가" }).click();
+
+    await expect(page.getByText("준비 대기 중")).toBeVisible({ timeout: 12_000 });
+    await expect(page.getByText("연습 AI", { exact: true })).toBeVisible();
+
+    const joined = frames.find((frame) => frame.direction === "sent" && frame.type === "queue.join");
+    const matched = frames.find((frame) => frame.direction === "received" && frame.type === "queue.matched");
+    expect(joined).toBeDefined();
+    expect(matched).toBeDefined();
+    expect(matched!.atMs - joined!.atMs).toBeGreaterThanOrEqual(5_500);
+    expect(matched!.atMs - joined!.atMs).toBeLessThan(10_000);
+
+    await page.getByRole("button", { name: "준비" }).click();
+    await expect(page.getByText("경기 진행 중")).toBeVisible();
+  });
+
+  test("경기 중 WebSocket이 끊겨도 새 ticket으로 복구한다", async ({ page }, testInfo) => {
+    test.skip(testInfo.project.name !== "chromium-desktop", "재연결 흐름은 desktop 프로젝트에서 한 번만 실행한다.");
+
+    await enterAsGuest(page);
+    const connections: Array<{ page: WebSocketRoute; server: WebSocketRoute }> = [];
+    await page.routeWebSocket(/.*/, async (socket) => {
+      if (connections.length > 0) await new Promise((resolve) => setTimeout(resolve, 600));
+      connections.push({ page: socket, server: socket.connectToServer() });
+    });
+    await openPlayPage(page);
+
+    await page.getByRole("button", { name: "인공지능 연습 시작" }).click();
+    await expect(page.getByText("준비 대기 중")).toBeVisible();
+    await page.getByRole("button", { name: "준비" }).click();
+    await expect(page.getByText("경기 진행 중")).toBeVisible();
+    expect(connections).toHaveLength(1);
+
+    await connections[0].page.close({ code: 1012, reason: "e2e reconnect" });
+    await expect(page.getByText("재연결 대기 중")).toBeVisible({ timeout: 2_000 });
+    await expect.poll(() => connections.length, { timeout: 5_000 }).toBe(2);
+    await expect(page.getByText("경기 진행 중")).toBeVisible({ timeout: 5_000 });
+    await expect(page.getByRole("button", { name: "일시정지" })).toBeEnabled();
+  });
+});
+
+async function createGuestPage(browser: Browser): Promise<{ context: BrowserContext; page: Page }> {
+  const context = await browser.newContext({ baseURL });
+  return { context, page: await context.newPage() };
+}
+
+async function enterAsGuest(page: Page): Promise<string> {
+  await page.goto("/");
+  await expect(page.getByRole("heading", { name: "퐁퐁" })).toBeVisible();
+  await expect(page.getByLabel("핸들")).toHaveCount(0);
+  await page.getByRole("button", { name: "게스트로 시작" }).click();
+
+  const welcome = page.getByRole("heading", { name: /다시 오신 것을 환영합니다, 게스트 [0-9]{4}/ });
+  await expect(welcome).toBeVisible();
+  const text = await welcome.textContent();
+  const displayName = text?.replace("다시 오신 것을 환영합니다, ", "").trim();
+  expect(displayName).toMatch(/^게스트 [0-9]{4}$/);
+  return displayName!;
+}
+
+async function openPlayPage(page: Page): Promise<void> {
+  await page.goto("/play");
+  await expect(page.getByRole("heading", { name: "경기장" })).toBeVisible();
+  await expect(page.getByText("경기 전")).toBeVisible();
+}
+
+type JsonFrame = {
+  direction: "sent" | "received";
+  type: string;
+  atMs: number;
+};
+
+function watchJsonFrames(page: Page): JsonFrame[] {
+  const frames: JsonFrame[] = [];
+  page.on("websocket", (socket) => {
+    socket.on("framesent", (event) => record("sent", event.payload));
+    socket.on("framereceived", (event) => record("received", event.payload));
+  });
+  return frames;
+
+  function record(direction: JsonFrame["direction"], payload: string | Buffer): void {
+    try {
+      const value = JSON.parse(payload.toString()) as { type?: unknown };
+      if (typeof value.type === "string") frames.push({ direction, type: value.type, atMs: Date.now() });
+    } catch {
+      // JSON이 아닌 WebSocket 프레임은 이 시나리오의 시간 측정 대상이 아니다.
+    }
+  }
+}


## `test(load): event-loop lag를 부하 profile에 노출`

diff --git a/docker-compose.load.yml b/docker-compose.load.yml
index 8f9628d..800954f 100644
--- a/docker-compose.load.yml
+++ b/docker-compose.load.yml
@@ -20,6 +20,8 @@ services:
     restart: "no"
 
   api:
+    ports:
+      - "127.0.0.1:${API_METRICS_PORT:-14000}:4000"
     environment:
       DATABASE_URL: postgres://pong:${POSTGRES_PASSWORD:?POSTGRES_PASSWORD must be set}@toxiproxy:15432/pong_pong
     depends_on:
diff --git a/tests/load/load-profile.mjs b/tests/load/load-profile.mjs
index 50d6b17..b607f1a 100644
--- a/tests/load/load-profile.mjs
+++ b/tests/load/load-profile.mjs
@@ -51,6 +51,7 @@ export function createLoadProfile(environment = {}) {
         connection_success: ["rate>=0.99"],
         reconnect_success: ["rate>=0.99"],
         snapshot_delay_ms: ["p(95)<=150", "p(99)<=250"],
+        event_loop_lag_p95_ms: ["p(95)<=50"],
         normal_snapshot_drop_rate: ["rate<0.01"],
         finalize_results: [`count>=${rooms}`],
         finalize_failures: ["count==0"],
diff --git a/tests/load/pong-load.js b/tests/load/pong-load.js
index 4dc5138..6e7da9e 100644
--- a/tests/load/pong-load.js
+++ b/tests/load/pong-load.js
@@ -7,11 +7,13 @@ import { createLoadProfile } from "./load-profile.mjs";
 
 const profile = createLoadProfile(__ENV);
 const apiBaseUrl = (__ENV.API_BASE_URL || "http://127.0.0.1:4000").replace(/\/$/, "");
+const metricsBaseUrl = (__ENV.METRICS_BASE_URL || "http://127.0.0.1:14000").replace(/\/$/, "");
 const websocketUrl = __ENV.WS_URL || "ws://127.0.0.1:4000/ws";
 
 const connectionSuccess = new Rate("connection_success");
 const reconnectSuccess = new Rate("reconnect_success");
 const snapshotDelay = new Trend("snapshot_delay_ms");
+const eventLoopLagP95 = new Trend("event_loop_lag_p95_ms");
 const normalSnapshotDropRate = new Rate("normal_snapshot_drop_rate");
 const finalizeResults = new Counter("finalize_results");
 const finalizeFailures = new Counter("finalize_failures");
@@ -30,6 +32,24 @@ export function setup() {
   }
 }
 
+export function teardown() {
+  const response = http.get(`${metricsBaseUrl}/metrics`, {
+    responseType: "text",
+    tags: { operation: "metrics" }
+  });
+  if (response.status !== 200 || typeof response.body !== "string") {
+    fail(`API metrics failed with ${response.status}`);
+  }
+  const match = response.body.match(
+    /^pong_pong_api_event_loop_lag_p95_seconds\s+([0-9.eE+-]+)$/m
+  );
+  const seconds = Number(match?.[1]);
+  if (!Number.isFinite(seconds) || seconds < 0) {
+    fail("API event-loop p95 metric is missing or invalid");
+  }
+  eventLoopLagP95.add(seconds * 1_000);
+}
+
 export default function () {
   const vuId = exec.vu.idInTest;
   const player = vuId <= profile.playerConnections;


## `test(load): fault recovery 검사 자동화`

diff --git a/package.json b/package.json
index 03deaf8..29105db 100644
--- a/package.json
+++ b/package.json
@@ -13,6 +13,7 @@
     "test": "pnpm unit",
     "unit": "pnpm -r test",
     "postgres-integration": "pnpm --filter @pong-pong/db postgres-integration",
+    "load:faults": "node tests/load/fault-scenario.mjs",
     "smoke:http": "node tests/smoke-api.mjs",
     "smoke:ws": "node tests/smoke-ws.mjs",
     "verify:build": "node tests/build-artifacts.mjs",
diff --git a/tests/load/fault-scenario.mjs b/tests/load/fault-scenario.mjs
new file mode 100644
index 0000000..d2c1f17
--- /dev/null
+++ b/tests/load/fault-scenario.mjs
@@ -0,0 +1,316 @@
+import { pathToFileURL } from "node:url";
+import { runCommand } from "./toxiproxy-control.mjs";
+
+const DEFAULT_TOXIPROXY_API_URL = "http://127.0.0.1:8474";
+const DEFAULT_API_READINESS_URL = "http://127.0.0.1:14000/health/ready";
+const DEFAULT_EDGE_READINESS_URL = "http://127.0.0.1:18080/api/health/ready";
+
+export function createFaultScenarioConfig(environment = {}) {
+  const config = {
+    toxiproxyApiUrl: loopbackUrl(
+      "TOXIPROXY_API_URL",
+      environment.TOXIPROXY_API_URL || DEFAULT_TOXIPROXY_API_URL
+    ),
+    apiReadinessUrl: loopbackUrl(
+      "FAULT_API_READINESS_URL",
+      environment.FAULT_API_READINESS_URL || DEFAULT_API_READINESS_URL
+    ),
+    edgeReadinessUrl: loopbackUrl(
+      "FAULT_EDGE_READINESS_URL",
+      environment.FAULT_EDGE_READINESS_URL || DEFAULT_EDGE_READINESS_URL
+    ),
+    databaseLatencyMs: positiveInteger(
+      "FAULT_DATABASE_LATENCY_MS",
+      environment.FAULT_DATABASE_LATENCY_MS,
+      300
+    ),
+    edgeLatencyMs: positiveInteger(
+      "FAULT_EDGE_LATENCY_MS",
+      environment.FAULT_EDGE_LATENCY_MS,
+      150
+    ),
+    requestTimeoutMs: positiveInteger(
+      "FAULT_REQUEST_TIMEOUT_MS",
+      environment.FAULT_REQUEST_TIMEOUT_MS,
+      5_000
+    ),
+    recoveryTimeoutMs: positiveInteger(
+      "FAULT_RECOVERY_TIMEOUT_MS",
+      environment.FAULT_RECOVERY_TIMEOUT_MS,
+      15_000
+    ),
+    pollIntervalMs: positiveInteger(
+      "FAULT_POLL_INTERVAL_MS",
+      environment.FAULT_POLL_INTERVAL_MS,
+      250
+    ),
+    includeEdge: booleanFlag("FAULT_INCLUDE_EDGE", environment.FAULT_INCLUDE_EDGE, true)
+  };
+
+  if (config.pollIntervalMs > config.recoveryTimeoutMs) {
+    throw new RangeError("FAULT_POLL_INTERVAL_MS must not exceed FAULT_RECOVERY_TIMEOUT_MS");
+  }
+  return config;
+}
+
+export async function runFaultScenario(config, overrides = {}) {
+  const dependencies = {
+    applyToxiproxyCommand: overrides.applyToxiproxyCommand
+      ?? ((command, args = []) => runCommand(command, args, {
+        TOXIPROXY_API_URL: config.toxiproxyApiUrl
+      })),
+    probeReadiness: overrides.probeReadiness ?? probeReadiness,
+    sleep: overrides.sleep ?? delay,
+    now: overrides.now ?? (() => new Date().toISOString())
+  };
+  const report = {
+    schemaVersion: 1,
+    startedAt: dependencies.now(),
+    finishedAt: null,
+    passed: false,
+    targets: {
+      toxiproxyApiUrl: config.toxiproxyApiUrl,
+      apiReadinessUrl: config.apiReadinessUrl,
+      edgeReadinessUrl: config.edgeReadinessUrl
+    },
+    settings: {
+      databaseLatencyMs: config.databaseLatencyMs,
+      edgeLatencyMs: config.edgeLatencyMs,
+      requestTimeoutMs: config.requestTimeoutMs,
+      recoveryTimeoutMs: config.recoveryTimeoutMs,
+      pollIntervalMs: config.pollIntervalMs,
+      includeEdge: config.includeEdge
+    },
+    steps: []
+  };
+
+  let scenarioError;
+  try {
+    await dependencies.applyToxiproxyCommand("reset", []);
+    report.steps.push(await observeStep({
+      name: "baseline",
+      url: config.apiReadinessUrl,
+      expected: isReady,
+      config,
+      dependencies
+    }));
+
+    await dependencies.applyToxiproxyCommand(
+      "db-latency",
+      [String(config.databaseLatencyMs), "0"]
+    );
+    report.steps.push(await observeStep({
+      name: "database_latency",
+      url: config.apiReadinessUrl,
+      expected: isReady,
+      config,
+      dependencies
+    }));
+
+    await dependencies.applyToxiproxyCommand("db-down", []);
+    report.steps.push(await observeStep({
+      name: "database_down",
+      url: config.apiReadinessUrl,
+      expected: isDatabaseDown,
+      config,
+      dependencies
+    }));
+
+    await dependencies.applyToxiproxyCommand("db-up", []);
+    report.steps.push(await observeStep({
+      name: "database_recovery",
+      url: config.apiReadinessUrl,
+      expected: isReady,
+      config,
+      dependencies
+    }));
+
+    if (config.includeEdge) {
+      await dependencies.applyToxiproxyCommand(
+        "edge-latency",
+        [String(config.edgeLatencyMs), "0"]
+      );
+      report.steps.push(await observeStep({
+        name: "edge_latency",
+        url: config.edgeReadinessUrl,
+        expected: isReady,
+        config,
+        dependencies
+      }));
+
+      await dependencies.applyToxiproxyCommand("edge-reset", ["0"]);
+      report.steps.push(await observeStep({
+        name: "edge_reset",
+        url: config.edgeReadinessUrl,
+        expected: (observation) => (
+          observation.status === null
+          || (observation.status >= 500 && observation.status <= 599)
+        ),
+        config,
+        dependencies
+      }));
+
+      await dependencies.applyToxiproxyCommand("edge-up", []);
+      report.steps.push(await observeStep({
+        name: "edge_recovery",
+        url: config.edgeReadinessUrl,
+        expected: isReady,
+        config,
+        dependencies
+      }));
+    }
+  } catch (error) {
+    scenarioError = error;
+  }
+
+  try {
+    await dependencies.applyToxiproxyCommand("reset", []);
+  } catch (cleanupError) {
+    if (!scenarioError) {
+      scenarioError = cleanupError;
+    } else if (scenarioError instanceof Error) {
+      scenarioError.cause = cleanupError;
+    }
+  }
+
+  if (scenarioError) throw scenarioError;
+  report.finishedAt = dependencies.now();
+  report.passed = report.steps.every((step) => step.passed);
+  return report;
+}
+
+export function formatFaultReport(report) {
+  return `${JSON.stringify(report, null, 2)}\n`;
+}
+
+async function observeStep({ name, url, expected, config, dependencies }) {
+  const deadline = Date.now() + config.recoveryTimeoutMs;
+  let lastObservation;
+
+  do {
+    lastObservation = await dependencies.probeReadiness(url, {
+      timeoutMs: config.requestTimeoutMs
+    });
+    if (expected(lastObservation)) {
+      return {
+        name,
+        passed: true,
+        ...lastObservation
+      };
+    }
+    if (Date.now() >= deadline) break;
+    await dependencies.sleep(config.pollIntervalMs);
+  } while (Date.now() < deadline);
+
+  throw new Error(
+    `${name} did not reach the expected state: ${summarizeObservation(lastObservation)}`
+  );
+}
+
+async function probeReadiness(url, { timeoutMs }) {
+  const startedAt = performance.now();
+  try {
+    const response = await fetch(url, {
+      headers: { accept: "application/json" },
+      signal: AbortSignal.timeout(timeoutMs)
+    });
+    const text = await response.text();
+    let body = null;
+    try {
+      body = text ? JSON.parse(text) : null;
+    } catch {
+      body = null;
+    }
+    return {
+      status: response.status,
+      durationMs: elapsedMilliseconds(startedAt),
+      body
+    };
+  } catch (error) {
+    return {
+      status: null,
+      durationMs: elapsedMilliseconds(startedAt),
+      error: error instanceof Error ? error.message : String(error)
+    };
+  }
+}
+
+function isReady(observation) {
+  return (
+    observation.status === 200
+    && observation.body?.status === "ready"
+    && observation.body?.checks?.database === "up"
+  );
+}
+
+function isDatabaseDown(observation) {
+  return (
+    observation.status === 503
+    && observation.body?.status === "not_ready"
+    && observation.body?.checks?.database === "down"
+  );
+}
+
+function summarizeObservation(observation) {
+  if (!observation) return "no response";
+  if (observation.status === null) return observation.error || "network failure";
+  return `HTTP ${observation.status} in ${observation.durationMs}ms`;
+}
+
+function elapsedMilliseconds(startedAt) {
+  return Math.round(Math.max(0, performance.now() - startedAt));
+}
+
+function loopbackUrl(name, rawValue) {
+  let parsed;
+  try {
+    parsed = new URL(rawValue);
+  } catch {
+    throw new RangeError(`${name} must be a valid loopback URL`);
+  }
+  const loopbackHosts = new Set(["localhost", "127.0.0.1", "[::1]", "::1"]);
+  if (parsed.protocol !== "http:" || !loopbackHosts.has(parsed.hostname)) {
+    throw new RangeError(`${name} must use an HTTP loopback URL`);
+  }
+  return parsed.href.replace(/\/$/, "");
+}
+
+function positiveInteger(name, rawValue, fallback) {
+  const value = rawValue === undefined || rawValue === "" ? fallback : Number(rawValue);
+  if (!Number.isSafeInteger(value) || value <= 0) {
+    throw new RangeError(`${name} must be a positive integer`);
+  }
+  return value;
+}
+
+function booleanFlag(name, rawValue, fallback) {
+  if (rawValue === undefined || rawValue === "") return fallback;
+  if (rawValue === "1") return true;
+  if (rawValue === "0") return false;
+  throw new RangeError(`${name} must be 0 or 1`);
+}
+
+function delay(durationMs) {
+  return new Promise((resolve) => setTimeout(resolve, durationMs));
+}
+
+const invokedAsScript = process.argv[1] && import.meta.url === pathToFileURL(process.argv[1]).href;
+if (invokedAsScript) {
+  runFromCommandLine();
+}
+
+async function runFromCommandLine() {
+  try {
+    const report = await runFaultScenario(createFaultScenarioConfig(process.env));
+    process.stdout.write(formatFaultReport(report));
+  } catch (error) {
+    process.stdout.write(formatFaultReport({
+      schemaVersion: 1,
+      passed: false,
+      error: {
+        message: error instanceof Error ? error.message : String(error)
+      }
+    }));
+    process.exitCode = 1;
+  }
+}


