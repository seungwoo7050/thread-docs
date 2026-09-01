## `ci(db): PostgreSQL integration 검사 실행`

diff --git a/.github/workflows/ci.yml b/.github/workflows/ci.yml
index 1f3e746..2c280bb 100644
--- a/.github/workflows/ci.yml
+++ b/.github/workflows/ci.yml
@@ -39,3 +39,29 @@ jobs:
 
       - name: Build
         run: pnpm build
+
+  postgres-integration:
+    name: PostgreSQL integration tests
+    runs-on: ubuntu-latest
+    timeout-minutes: 15
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
+      - name: Run PostgreSQL integration tests
+        run: pnpm postgres-integration


## `test(smoke): cookie 기반 realtime protocol 검증`

diff --git a/tests/e2e/pong-pong.spec.ts b/tests/e2e/pong-pong.spec.ts
index d609636..c2fbd7d 100644
--- a/tests/e2e/pong-pong.spec.ts
+++ b/tests/e2e/pong-pong.spec.ts
@@ -91,27 +91,29 @@ test("프로필 친구 요청과 공유 복사를 확인한다", async ({ page }
   await expect(page.getByText(/공유 링크를/)).toBeVisible();
 });
 
-test("토너먼트 브래킷과 경기 입장 액션을 확인한다", async ({ page, request }, testInfo) => {
+test("토너먼트 브래킷과 경기 입장 액션을 확인한다", async ({ page, playwright }, testInfo) => {
   const suffix = `${testInfo.project.name}-${Date.now()}`.replace(/[^a-z0-9-]/gi, "-").toLowerCase();
   await login(page, `cup-player-${suffix}`, "컵선수");
-  const token = await page.evaluate(() => window.localStorage.getItem("pong-pong-token"));
   const name = `E2E 퐁퐁 컵 ${suffix}`;
-  const created = await request.post(`${apiBase}/tournaments`, {
-    headers: { authorization: `Bearer ${token}` },
+  const created = await page.request.post(`${apiBase}/tournaments`, {
     data: { name }
   });
+  expect(created.ok()).toBe(true);
   const tournament = (await created.json()).tournament as { id: string };
-  await request.post(`${apiBase}/tournaments/${tournament.id}/join`, {
-    headers: { authorization: `Bearer ${token}` }
-  });
+  const duplicateJoin = await page.request.post(`${apiBase}/tournaments/${tournament.id}/join`);
+  expect(duplicateJoin.ok()).toBe(true);
   for (const handle of ["cup-two", "cup-three", "cup-four"]) {
-    const loginResponse = await request.post(`${apiBase}/auth/dev-login`, {
-      data: { handle: `${handle}-${suffix}`, displayName: handle }
-    });
-    const playerToken = (await loginResponse.json()).token as string;
-    await request.post(`${apiBase}/tournaments/${tournament.id}/join`, {
-      headers: { authorization: `Bearer ${playerToken}` }
-    });
+    const playerRequest = await playwright.request.newContext({ baseURL: apiBase });
+    try {
+      const loginResponse = await playerRequest.post("/auth/dev-login", {
+        data: { handle: `${handle}-${suffix}`, displayName: handle }
+      });
+      expect(loginResponse.ok()).toBe(true);
+      const joinResponse = await playerRequest.post(`/tournaments/${tournament.id}/join`);
+      expect(joinResponse.ok()).toBe(true);
+    } finally {
+      await playerRequest.dispose();
+    }
   }
   await page.getByRole("link", { name: "토너먼트" }).click();
   await page.getByRole("button", { name }).click();
@@ -119,13 +121,9 @@ test("토너먼트 브래킷과 경기 입장 액션을 확인한다", async ({
   await expect(page.getByRole("link", { name: "경기 입장" })).toBeVisible();
 });
 
-test("관리 화면에서 사용자 상태를 변경한다", async ({ page }) => {
-  const reason = `E2E 운영 확인 ${Date.now()} ${Math.random()}`;
+test("admin 핸들만으로 운영자 권한을 얻지 못한다", async ({ page }) => {
   await login(page, "admin", "운영자");
   await page.getByRole("link", { name: "관리" }).click();
-  await expect(page.getByText("사용자 목록과 감사 로그를 불러왔습니다.")).toBeVisible();
-  await page.getByLabel("조치 사유").fill(reason);
-  await page.getByRole("button", { name: /정지|해제/ }).first().click();
-  await expect(page.getByText(/상태를|운영자 권한/)).toBeVisible();
-  await expect(page.getByText(reason)).toBeVisible();
+  await expect(page.getByText("운영자 권한이 필요합니다.")).toBeVisible();
+  await expect(page.getByRole("button", { name: /정지|해제/ })).toHaveCount(0);
 });
diff --git a/tests/smoke-api.mjs b/tests/smoke-api.mjs
index 673f4f8..7e8961a 100644
--- a/tests/smoke-api.mjs
+++ b/tests/smoke-api.mjs
@@ -4,41 +4,53 @@ const login = await request("/auth/dev-login", {
   method: "POST",
   body: JSON.stringify({ handle: "smoke", displayName: "스모크" })
 });
+if (!login.cookie) throw new Error("dev login did not set the session cookie");
+if ("token" in login.body) throw new Error("dev login exposed a JSON session token");
 
-await request("/me", { headers: { authorization: `Bearer ${login.token}` } });
-await request("/lobby", { headers: { authorization: `Bearer ${login.token}` } });
+await request("/me", { cookie: login.cookie });
+await request("/lobby", { cookie: login.cookie });
 await request("/chat/lobby", {
   method: "POST",
-  headers: { authorization: `Bearer ${login.token}` },
+  cookie: login.cookie,
   body: JSON.stringify({ body: "스모크 로비 채팅" })
 });
 await request("/leaderboard");
-await request("/dashboard", { headers: { authorization: `Bearer ${login.token}` } });
+await request("/dashboard", { cookie: login.cookie });
 await request("/tournaments");
 await request("/tournaments", {
   method: "POST",
-  headers: { authorization: `Bearer ${login.token}` },
+  cookie: login.cookie,
   body: JSON.stringify({ name: "스모크 컵" })
 });
 
-const admin = await request("/auth/dev-login", {
+const adminHandle = await request("/auth/dev-login", {
   method: "POST",
   body: JSON.stringify({ handle: "admin", displayName: "운영자" })
 });
-await request("/admin/actions", { headers: { authorization: `Bearer ${admin.token}` } });
+if (!adminHandle.cookie) throw new Error("admin-handle login did not set a cookie");
+await request("/admin/actions", {
+  cookie: adminHandle.cookie,
+  expectedStatus: 403
+});
 
 console.log("api smoke ok");
 
 async function request(path, init = {}) {
   const response = await fetch(`${baseUrl}${path}`, {
-    ...init,
+    method: init.method,
+    body: init.body,
     headers: {
       "content-type": "application/json",
-      ...init.headers
+      ...(init.cookie ? { cookie: init.cookie } : {})
     }
   });
-  if (!response.ok) {
-    throw new Error(`${path} failed with ${response.status}: ${await response.text()}`);
+  const expectedStatus = init.expectedStatus ?? 200;
+  if (response.status !== expectedStatus) {
+    throw new Error(`${path} returned ${response.status}, expected ${expectedStatus}: ${await response.text()}`);
   }
-  return response.json();
+  const setCookie = response.headers.get("set-cookie");
+  return {
+    body: await response.json(),
+    cookie: setCookie?.split(";", 1)[0]
+  };
 }
diff --git a/tests/smoke-ws.mjs b/tests/smoke-ws.mjs
index ba55273..3025d19 100644
--- a/tests/smoke-ws.mjs
+++ b/tests/smoke-ws.mjs
@@ -4,12 +4,12 @@ const wsUrl = process.env.WS_URL ?? "ws://localhost:4000/ws";
 const left = await login("left-smoke", "왼쪽");
 const right = await login("right-smoke", "오른쪽");
 
-const leftSocket = connect(left.token);
-const rightSocket = connect(right.token);
+const leftSocket = connect(await issueTicket(left.cookie));
+const rightSocket = connect(await issueTicket(right.cookie));
 const events = [];
 
-leftSocket.addEventListener("message", (event) => events.push({ side: "left", event: JSON.parse(event.data) }));
-rightSocket.addEventListener("message", (event) => events.push({ side: "right", event: JSON.parse(event.data) }));
+leftSocket.addEventListener("message", (event) => events.push({ side: "left", event: parseEvent(event.data) }));
+rightSocket.addEventListener("message", (event) => events.push({ side: "right", event: parseEvent(event.data) }));
 
 try {
   await opened(leftSocket);
@@ -20,53 +20,65 @@ try {
     return onlineHandles.includes("left-smoke") && onlineHandles.includes("right-smoke") ? lobby : null;
   });
 
-  leftSocket.send(JSON.stringify({ type: "chat.send", scope: "lobby", roomId: null, body: "로비 실시간 확인" }));
+  send(leftSocket, { type: "chat.send", scope: "lobby", roomId: null, body: "로비 실시간 확인" });
   await waitFor(() => events.find((item) => item.event.type === "chat.message" && item.event.message.scope === "lobby"));
 
-  leftSocket.send(JSON.stringify({ type: "queue.join", mode: "queue" }));
-  rightSocket.send(JSON.stringify({ type: "queue.join", mode: "queue" }));
+  send(leftSocket, { type: "queue.join", mode: "queue" });
+  send(rightSocket, { type: "queue.join", mode: "queue" });
 
   const leftMatched = await waitFor(() => events.find((item) => item.side === "left" && item.event.type === "queue.matched"));
   const rightMatched = await waitFor(() => events.find((item) => item.side === "right" && item.event.type === "queue.matched"));
   const roomId = leftMatched.event.roomId === rightMatched.event.roomId ? leftMatched.event.roomId : null;
   if (!roomId) throw new Error("matched sockets joined different rooms");
 
-  leftSocket.send(JSON.stringify({ type: "game.ready", roomId }));
-  rightSocket.send(JSON.stringify({ type: "game.ready", roomId }));
-  const firstPlaying = await waitFor(() => events.find((item) => item.event.type === "game.snapshot" && item.event.snapshot.phase === "playing"));
-  const initialSpeed = speedOf(firstPlaying.event.snapshot.ball.velocity);
+  send(leftSocket, { type: "game.ready", roomId });
+  send(rightSocket, { type: "game.ready", roomId });
+  const firstPlaying = await waitFor(() => events.find((item) =>
+    item.event.type === "game.snapshot" && item.event.snapshot.state.phase === "playing"
+  ));
+  const initialSpeed = speedOf(firstPlaying.event.snapshot.state.ball.velocity);
   if (initialSpeed < 11) throw new Error(`ball starts too slowly: ${initialSpeed}`);
-  const accelerated = await waitFor(() =>
-    events.find((item) => item.event.type === "game.snapshot" && item.event.snapshot.phase === "playing" && item.event.snapshot.tick >= firstPlaying.event.snapshot.tick + 20)
-  );
-  const acceleratedSpeed = speedOf(accelerated.event.snapshot.ball.velocity);
+  const accelerated = await waitFor(() => events.find((item) =>
+    item.event.type === "game.snapshot"
+      && item.event.snapshot.state.phase === "playing"
+      && item.event.snapshot.tick >= firstPlaying.event.snapshot.tick + 20
+  ));
+  const acceleratedSpeed = speedOf(accelerated.event.snapshot.state.ball.velocity);
   if (acceleratedSpeed <= initialSpeed) throw new Error(`ball did not accelerate: ${initialSpeed} -> ${acceleratedSpeed}`);
 
-  leftSocket.send(JSON.stringify({ type: "game.pause", roomId }));
-  await waitFor(() => events.find((item) => item.event.type === "game.snapshot" && item.event.snapshot.phase === "paused"));
-  leftSocket.send(JSON.stringify({ type: "game.resume", roomId }));
-  await waitFor(() => events.filter((item) => item.event.type === "game.snapshot" && item.event.snapshot.phase === "playing").length >= 2);
+  send(leftSocket, { type: "game.pause", roomId });
+  await waitFor(() => events.find((item) =>
+    item.event.type === "game.snapshot" && item.event.snapshot.state.phase === "paused"
+  ));
+  send(leftSocket, { type: "game.resume", roomId });
+  await waitFor(() => events.filter((item) =>
+    item.event.type === "game.snapshot" && item.event.snapshot.state.phase === "playing"
+  ).length >= 2);
 
-  leftSocket.send(JSON.stringify({ type: "chat.send", scope: "match", roomId, body: "준비됐습니다." }));
+  send(leftSocket, { type: "chat.send", scope: "match", roomId, body: "준비됐습니다." });
   await waitFor(() => events.find((item) => item.event.type === "chat.message"));
-
 } finally {
   leftSocket.close();
   rightSocket.close();
 }
 
 const solo = await login("solo-smoke", "혼자큐");
-const soloSocket = connect(solo.token);
+const soloSocket = connect(await issueTicket(solo.cookie));
 const soloEvents = [];
-soloSocket.addEventListener("message", (event) => soloEvents.push(JSON.parse(event.data)));
+soloSocket.addEventListener("message", (event) => soloEvents.push(parseEvent(event.data)));
 
 try {
   await opened(soloSocket);
-  soloSocket.send(JSON.stringify({ type: "queue.join", mode: "queue" }));
-  const matched = await waitFor(() => soloEvents.find((event) => event.type === "queue.matched" && event.opponent.includes("AI")), 8_000);
-  soloSocket.send(JSON.stringify({ type: "game.ready", roomId: matched.roomId }));
-  const npcSnapshot = await waitFor(() => soloEvents.find((event) => event.type === "game.snapshot" && event.snapshot.players.some((player) => player.ai && player.handle.startsWith("npc-"))));
-  const npcPlayer = npcSnapshot.snapshot.players.find((player) => player.ai);
+  send(soloSocket, { type: "queue.join", mode: "queue" });
+  const matched = await waitFor(() =>
+    soloEvents.find((event) => event.type === "queue.matched" && event.opponent.includes("AI")),
+  8_000);
+  send(soloSocket, { type: "game.ready", roomId: matched.roomId });
+  const npcSnapshot = await waitFor(() => soloEvents.find((event) =>
+    event.type === "game.snapshot"
+      && event.snapshot.state.players.some((player) => player.ai && player.handle.startsWith("npc-"))
+  ));
+  const npcPlayer = npcSnapshot.snapshot.state.players.find((player) => player.ai);
   if (!npcPlayer) throw new Error("npc snapshot missing ai player");
 } finally {
   soloSocket.close();
@@ -81,7 +93,22 @@ async function login(handle, displayName) {
     body: JSON.stringify({ handle, displayName })
   });
   if (!response.ok) throw new Error(`login failed: ${response.status}`);
-  return response.json();
+  const cookie = response.headers.get("set-cookie")?.split(";", 1)[0];
+  if (!cookie) throw new Error("login did not set a session cookie");
+  const body = await response.json();
+  if ("token" in body) throw new Error("login exposed a JSON session token");
+  return { cookie, user: body.user };
+}
+
+async function issueTicket(cookie) {
+  const response = await fetch(`${baseUrl}/auth/ws-ticket`, {
+    method: "POST",
+    headers: { cookie }
+  });
+  if (!response.ok) throw new Error(`ticket request failed: ${response.status}`);
+  const body = await response.json();
+  if (body.protocolVersion !== 1) throw new Error(`unsupported ticket protocol: ${body.protocolVersion}`);
+  return body.ticket;
 }
 
 async function fetchJson(url) {
@@ -90,8 +117,18 @@ async function fetchJson(url) {
   return response.json();
 }
 
-function connect(token) {
-  return new WebSocket(`${wsUrl}?session=${token}`);
+function connect(ticket) {
+  return new WebSocket(`${wsUrl}?ticket=${encodeURIComponent(ticket)}&v=1`);
+}
+
+function send(socket, event) {
+  socket.send(JSON.stringify({ v: 1, ...event }));
+}
+
+function parseEvent(payload) {
+  const event = JSON.parse(payload);
+  if (event.v !== 1) throw new Error(`received unversioned websocket event: ${payload}`);
+  return event;
 }
 
 function opened(socket) {


## `test(load): 실시간 부하 임계값 정의`

diff --git a/tests/load/load-harness.test.mjs b/tests/load/load-harness.test.mjs
new file mode 100644
index 0000000..4284cd0
--- /dev/null
+++ b/tests/load/load-harness.test.mjs
@@ -0,0 +1,105 @@
+import assert from "node:assert/strict";
+import { readFile } from "node:fs/promises";
+import test from "node:test";
+import { createLoadProfile } from "./load-profile.mjs";
+import { buildProxyDefinitions, toxicForCommand } from "./toxiproxy-control.mjs";
+
+test("default profile attempts 500 connections and observes 50 rooms", () => {
+  const profile = createLoadProfile({});
+
+  assert.equal(profile.connections, 500);
+  assert.equal(profile.rooms, 50);
+  assert.equal(profile.playerConnections, 100);
+  assert.equal(profile.minimumSuccessfulConnections, 495);
+  assert.deepEqual(profile.options.scenarios.pong, {
+    executor: "per-vu-iterations",
+    vus: 500,
+    iterations: 1,
+    maxDuration: "4m"
+  });
+  assert.deepEqual(profile.options.thresholds.connection_success, ["rate>=0.99"]);
+  assert.deepEqual(profile.options.thresholds.reconnect_success, ["rate>=0.99"]);
+  assert.deepEqual(profile.options.thresholds.snapshot_delay_ms, ["p(95)<=150", "p(99)<=250"]);
+  assert.deepEqual(profile.options.thresholds.normal_snapshot_drop_rate, ["rate<0.01"]);
+  assert.deepEqual(profile.options.thresholds.finalize_failures, ["count==0"]);
+  assert.deepEqual(profile.options.thresholds.finalize_duplicates, ["count==0"]);
+  assert.deepEqual(profile.options.thresholds.finalize_results, ["count>=50"]);
+  assert.deepEqual(profile.options.thresholds.online_connections, ["max>=495"]);
+  assert.deepEqual(profile.options.thresholds.active_rooms, ["max>=50"]);
+});
+
+test("extended profile makes 1,000 connections an explicit environment choice", () => {
+  const extended = createLoadProfile({ EXTENDED_LOAD: "1" });
+  const explicit = createLoadProfile({ CONNECTIONS: "1000", ROOMS: "50" });
+
+  assert.equal(extended.connections, 1000);
+  assert.equal(extended.minimumSuccessfulConnections, 990);
+  assert.equal(extended.options.scenarios.pong.vus, 1000);
+  assert.equal(explicit.connections, 1000);
+  assert.throws(
+    () => createLoadProfile({ CONNECTIONS: "99", ROOMS: "50" }),
+    /at least twice the room count/
+  );
+});
+
+test("k6 scenario records every required service-level indicator", async () => {
+  const source = await readFile(new URL("./pong-load.js", import.meta.url), "utf8");
+
+  for (const metric of [
+    "connection_success",
+    "reconnect_success",
+    "snapshot_delay_ms",
+    "normal_snapshot_drop_rate",
+    "finalize_results",
+    "finalize_failures",
+    "finalize_duplicates",
+    "online_connections",
+    "active_rooms"
+  ]) {
+    assert.match(source, new RegExp(`new (?:Rate|Trend|Counter)\\(\"${metric}\"\\)`));
+  }
+  assert.match(source, /POST.*auth\/dev-login/s);
+  assert.match(source, /auth\/ws-ticket/);
+  assert.match(source, /type: "queue\.join"/);
+  assert.match(source, /type: "game\.ready"/);
+  assert.match(source, /inputSeq/);
+  assert.match(source, /serverTimeMs/);
+});
+
+test("Toxiproxy plan separates PostgreSQL and edge failure paths", () => {
+  assert.deepEqual(buildProxyDefinitions({}), [
+    { name: "postgres", listen: "0.0.0.0:15432", upstream: "db:5432", enabled: true },
+    { name: "edge", listen: "0.0.0.0:18080", upstream: "caddy:8080", enabled: true }
+  ]);
+  assert.deepEqual(toxicForCommand("db-latency", ["300", "50"]), {
+    proxy: "postgres",
+    toxic: {
+      name: "db-latency",
+      type: "latency",
+      stream: "downstream",
+      toxicity: 1,
+      attributes: { latency: 300, jitter: 50 }
+    }
+  });
+  assert.deepEqual(toxicForCommand("edge-reset", ["750"]), {
+    proxy: "edge",
+    toxic: {
+      name: "edge-reset",
+      type: "reset_peer",
+      stream: "downstream",
+      toxicity: 1,
+      attributes: { timeout: 750 }
+    }
+  });
+  assert.throws(() => toxicForCommand("db-latency", ["bad"]), /positive integer/);
+});
+
+test("load overlay routes API database traffic and the public edge through Toxiproxy", async () => {
+  const compose = await readFile(new URL("../../docker-compose.load.yml", import.meta.url), "utf8");
+
+  assert.match(compose, /ghcr\.io\/shopify\/toxiproxy:2\.12\.0/);
+  assert.match(compose, /DATABASE_URL: postgres:\/\/pong:.*@toxiproxy:15432\/pong_pong/);
+  assert.match(compose, /127\.0\.0\.1:\$\{TOXIPROXY_EDGE_PORT:-18080\}:18080/);
+  assert.match(compose, /toxiproxy-bootstrap:/);
+  assert.match(compose, /service_completed_successfully/);
+});


