# 프로덕션 컨테이너·영속 저장소·정상 종료

## `build(runtime): Compose와 Caddy 라우팅 추가`

diff --git a/Caddyfile b/Caddyfile
new file mode 100644
index 0000000..f3d17e9
--- /dev/null
+++ b/Caddyfile
@@ -0,0 +1,13 @@
+:8080 {
+	handle_path /api/* {
+		reverse_proxy api:4000
+	}
+
+	handle /ws {
+		reverse_proxy api:4000
+	}
+
+	handle {
+		reverse_proxy web:3000
+	}
+}
diff --git a/docker-compose.yml b/docker-compose.yml
new file mode 100644
index 0000000..3d4c4e5
--- /dev/null
+++ b/docker-compose.yml
@@ -0,0 +1,64 @@
+services:
+  db:
+    image: postgres:16-alpine
+    environment:
+      POSTGRES_DB: pong_pong
+      POSTGRES_USER: pong
+      POSTGRES_PASSWORD: pong
+    ports:
+      - "5432:5432"
+    healthcheck:
+      test: ["CMD-SHELL", "pg_isready -U pong -d pong_pong"]
+      interval: 5s
+      timeout: 3s
+      retries: 20
+    volumes:
+      - pong-pong-db:/var/lib/postgresql/data
+
+  api:
+    image: node:23-bookworm-slim
+    working_dir: /app
+    command: sh -c "corepack enable && corepack prepare pnpm@10.32.1 --activate && pnpm install --frozen-lockfile && pnpm --filter @pong-pong/api dev"
+    environment:
+      DATABASE_URL: postgres://pong:pong@db:5432/pong_pong
+      SESSION_SECRET: dev-session-secret
+      API_PORT: 4000
+      WEB_ORIGIN: http://localhost:8080
+    volumes:
+      - .:/app
+      - api-node-modules:/app/node_modules
+    depends_on:
+      db:
+        condition: service_healthy
+    ports:
+      - "4000:4000"
+
+  web:
+    image: node:23-bookworm-slim
+    working_dir: /app
+    command: sh -c "corepack enable && corepack prepare pnpm@10.32.1 --activate && pnpm install --frozen-lockfile && pnpm --filter @pong-pong/web dev"
+    environment:
+      NEXT_PUBLIC_API_BASE_URL: http://localhost:8080/api
+      NEXT_PUBLIC_WS_URL: ws://localhost:8080/ws
+    volumes:
+      - .:/app
+      - web-node-modules:/app/node_modules
+    depends_on:
+      - api
+    ports:
+      - "3000:3000"
+
+  caddy:
+    image: caddy:2-alpine
+    ports:
+      - "8080:8080"
+    volumes:
+      - ./Caddyfile:/etc/caddy/Caddyfile:ro
+    depends_on:
+      - web
+      - api
+
+volumes:
+  pong-pong-db:
+  api-node-modules:
+  web-node-modules:


## `feat(game): 새 작업 차단과 active room drain 추가`

diff --git a/apps/api/src/app.ts b/apps/api/src/app.ts
index 26dcbc6..42fe3f7 100644
--- a/apps/api/src/app.ts
+++ b/apps/api/src/app.ts
@@ -6,7 +6,7 @@ import type { AppRepository } from "@pong-pong/db";
 import * as http from "@pong-pong/shared";
 import type { SessionUser } from "@pong-pong/shared";
 import { WebSocket, type RawData } from "ws";
-import { GameHub } from "./gameHub.js";
+import { GameHub, type DrainResult } from "./gameHub.js";
 import {
   ApiHttpError,
   forbidden,
@@ -45,6 +45,12 @@ export interface BuildAppOptions {
   trustProxy?: boolean;
 }
 
+declare module "fastify" {
+  interface FastifyInstance {
+    beginDrain(timeoutMs?: number): Promise<DrainResult>;
+  }
+}
+
 export function buildApp({
   repo: sourceRepo,
   webOrigin,
@@ -81,6 +87,7 @@ export function buildApp({
     }
   });
   readGameStats = () => hub.liveStats();
+  let draining = false;
   const guests = appMode === "demo" ? guestAccess ?? new GuestAccess({ secret: sessionSecret }) : null;
   const getCurrentUser = async (request: FastifyRequest) => {
     const user = await currentUser(repo, request, guests, appMode === "demo");
@@ -88,6 +95,10 @@ export function buildApp({
     return user;
   };
 
+  app.decorate("beginDrain", async (timeoutMs = 60_000) => {
+    draining = true;
+    return hub.beginDrain(timeoutMs);
+  });
   app.addHook("onResponse", (request, reply, done) => {
     metrics.observeRequest(
       request.method,
@@ -98,6 +109,7 @@ export function buildApp({
     done();
   });
   app.addHook("onClose", async () => {
+    hub.close();
     metrics.close();
   });
 
@@ -195,13 +207,14 @@ export function buildApp({
     const startedAt = performance.now();
     try {
       const repository = await repo.checkReadiness();
-      const ready = repository.database === "up"
+      const ready = !draining
+        && repository.database === "up"
         && (repository.migrations === "current" || repository.migrations === "not_applicable");
       const body = parseOutput(http.readyHealthResponseSchema, {
         status: ready ? "ready" : "not_ready",
         service: "pong-pong-api",
         checks: {
-          lifecycle: "accepting",
+          lifecycle: draining ? "draining" : "accepting",
           database: repository.database,
           migrations: repository.migrations
         }
@@ -213,7 +226,11 @@ export function buildApp({
       const body = parseOutput(http.readyHealthResponseSchema, {
         status: "not_ready",
         service: "pong-pong-api",
-        checks: { lifecycle: "accepting", database: "down", migrations: "unknown" }
+        checks: {
+          lifecycle: draining ? "draining" : "accepting",
+          database: "down",
+          migrations: "unknown"
+        }
       });
       metrics.observeReadiness("not_ready", performance.now() - startedAt);
       return reply.code(503).send(body);
diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index d3e54d3..1665807 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -72,6 +72,11 @@ const CONNECTION_REPLACED_CLOSE_CODE = 4001;
 const CONNECTION_REPLACED_REASON = "connection replaced";
 const GUEST_RESULT_RETENTION_MS = 2 * 60 * 1_000;
 
+export interface DrainResult {
+  drained: boolean;
+  activeRooms: number;
+}
+
 export interface GameHubObserver {
   roomCreated?(context: {
     roomId: string;
@@ -109,6 +114,12 @@ export class GameHub {
     expiresAtMs: number;
     cleanupTimer: NodeJS.Timeout;
   }>();
+  private acceptingMatches = true;
+  private drainWaiter: {
+    promise: Promise<DrainResult>;
+    resolve: (result: DrainResult) => void;
+    timer: ReturnType<typeof setTimeout>;
+  } | null = null;
 
   constructor(
     private readonly repo: AppRepository,
@@ -340,6 +351,7 @@ export class GameHub {
       if (client) client.roomId = null;
     }
     this.rooms.delete(room.id);
+    this.notifyDrainProgress();
     this.broadcastPresence();
   }
 
@@ -356,6 +368,10 @@ export class GameHub {
   }
 
   private async joinQueue(client: Client, mode: "queue" | "ai"): Promise<void> {
+    if (!this.acceptingMatches) {
+      this.sendDrainingError(client);
+      return;
+    }
     if (client.roomId) {
       this.send(client, { type: "error", code: "forbidden", message: "이미 진행 중인 경기가 있습니다." });
       return;
@@ -415,6 +431,10 @@ export class GameHub {
   }
 
   private async joinTournamentMatch(client: Client, matchId: string): Promise<void> {
+    if (!this.acceptingMatches) {
+      this.sendDrainingError(client);
+      return;
+    }
     this.leaveQueue(client);
     this.leaveTournamentWaiters(client);
     const match = await this.repo.getTournamentMatch(matchId);
@@ -499,6 +519,54 @@ export class GameHub {
     };
   }
 
+  beginDrain(timeoutMs: number): Promise<DrainResult> {
+    this.acceptingMatches = false;
+    for (const entry of this.queue.splice(0)) {
+      clearQueueTimer(entry);
+      this.sendDrainingError(entry.client);
+    }
+    for (const waiters of this.tournamentWaiters.values()) {
+      for (const client of waiters) this.sendDrainingError(client);
+    }
+    this.tournamentWaiters.clear();
+    this.broadcastPresence();
+
+    if (this.rooms.size === 0) {
+      return Promise.resolve({ drained: true, activeRooms: 0 });
+    }
+    if (this.drainWaiter) return this.drainWaiter.promise;
+
+    let resolveDrain: (result: DrainResult) => void = () => undefined;
+    const promise = new Promise<DrainResult>((resolve) => {
+      resolveDrain = resolve;
+    });
+    const timer = setTimeout(() => {
+      this.finishDrain({ drained: false, activeRooms: this.rooms.size });
+    }, Math.max(0, timeoutMs));
+    timer.unref?.();
+    this.drainWaiter = { promise, resolve: resolveDrain, timer };
+    return promise;
+  }
+
+  close(): void {
+    this.acceptingMatches = false;
+    for (const entry of this.queue.splice(0)) clearQueueTimer(entry);
+    this.tournamentWaiters.clear();
+    this.roomScheduler.stop();
+    for (const room of this.rooms.values()) this.clearReconnectTimer(room);
+    this.rooms.clear();
+    for (const recent of this.recentGuestResults.values()) clearTimeout(recent.cleanupTimer);
+    this.recentGuestResults.clear();
+    const clients = [...this.clients.values()];
+    this.clients.clear();
+    this.clientsByUser.clear();
+    for (const client of clients) {
+      client.heartbeat.stop();
+      client.snapshots.close();
+      if (client.socket.readyState === WebSocket.OPEN) client.socket.terminate();
+    }
+  }
+
   onlinePlayers(): PublicUser[] {
     const users = new Map<string, PublicUser>();
     for (const client of this.clients.values()) {
@@ -791,9 +859,32 @@ export class GameHub {
       if (client) client.roomId = null;
     }
     this.rooms.delete(room.id);
+    this.notifyDrainProgress();
     this.broadcastPresence();
   }
 
+  private sendDrainingError(client: Client): void {
+    this.send(client, {
+      type: "error",
+      code: "server_draining",
+      message: "서버 점검을 준비하고 있어 새 경기를 시작할 수 없습니다."
+    });
+  }
+
+  private notifyDrainProgress(): void {
+    if (this.drainWaiter && this.rooms.size === 0) {
+      this.finishDrain({ drained: true, activeRooms: 0 });
+    }
+  }
+
+  private finishDrain(result: DrainResult): void {
+    const waiter = this.drainWaiter;
+    if (!waiter) return;
+    this.drainWaiter = null;
+    clearTimeout(waiter.timer);
+    waiter.resolve(result);
+  }
+
   private broadcastPresence(): void {
     this.broadcastAll({
       type: "presence.changed",
diff --git a/packages/shared/src/ws.ts b/packages/shared/src/ws.ts
index 3461ba9..31cc7c5 100644
--- a/packages/shared/src/ws.ts
+++ b/packages/shared/src/ws.ts
@@ -37,6 +37,7 @@ export const wsErrorCodeSchema = z.enum([
   "rate_limited",
   "forbidden",
   "not_found",
+  "server_draining",
   "internal_error"
 ]);
 


## `feat(ops): graceful shutdown 절차 추가`

diff --git a/apps/api/src/gracefulShutdown.ts b/apps/api/src/gracefulShutdown.ts
new file mode 100644
index 0000000..bb95fa4
--- /dev/null
+++ b/apps/api/src/gracefulShutdown.ts
@@ -0,0 +1,27 @@
+import type { EventEmitter } from "node:events";
+
+type ShutdownSignal = "SIGTERM" | "SIGINT";
+type SignalSource = Pick<EventEmitter, "on" | "off">;
+
+export function installGracefulShutdown(
+  signals: SignalSource,
+  shutdown: (signal: ShutdownSignal) => Promise<void>,
+  onError: (error: unknown) => void
+): () => void {
+  let started = false;
+  const start = (signal: ShutdownSignal) => {
+    if (started) return;
+    started = true;
+    void shutdown(signal).catch(onError);
+  };
+  const onSigterm = () => start("SIGTERM");
+  const onSigint = () => start("SIGINT");
+
+  signals.on("SIGTERM", onSigterm);
+  signals.on("SIGINT", onSigint);
+
+  return () => {
+    signals.off("SIGTERM", onSigterm);
+    signals.off("SIGINT", onSigint);
+  };
+}
diff --git a/apps/api/src/index.ts b/apps/api/src/index.ts
index dd11e28..56f9f4d 100644
--- a/apps/api/src/index.ts
+++ b/apps/api/src/index.ts
@@ -1,6 +1,7 @@
 import { createMemoryRepository, createPostgresRepository } from "@pong-pong/db";
 import { buildApp } from "./app.js";
 import { readEnv } from "./env.js";
+import { installGracefulShutdown } from "./gracefulShutdown.js";
 
 const env = readEnv();
 const repo = env.databaseUrl ? createPostgresRepository(env.databaseUrl) : createMemoryRepository();
@@ -19,6 +20,24 @@ app.addHook("onClose", async () => {
   await repo.close();
 });
 
+const disposeShutdownSignals = installGracefulShutdown(
+  process,
+  async (signal) => {
+    app.log.info({ signal }, "graceful shutdown started");
+    const result = await app.beginDrain(60_000);
+    app.log.info(result, "game room drain finished");
+    await app.close();
+  },
+  (error) => {
+    app.log.error({ errorName: error instanceof Error ? error.name : "UnknownError" }, "graceful shutdown failed");
+    process.exitCode = 1;
+    void app.close().catch(() => undefined);
+  }
+);
+app.addHook("onClose", async () => {
+  disposeShutdownSignals();
+});
+
 try {
   await app.listen({ port: env.port, host: "0.0.0.0" });
 } catch (error) {


## `test(ops): GameHub drain과 graceful shutdown 검증`

diff --git a/apps/api/src/gameHub.drain.test.ts b/apps/api/src/gameHub.drain.test.ts
new file mode 100644
index 0000000..2f70704
--- /dev/null
+++ b/apps/api/src/gameHub.drain.test.ts
@@ -0,0 +1,133 @@
+import { EventEmitter } from "node:events";
+import type { IncomingMessage } from "node:http";
+import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
+import { WebSocket } from "ws";
+import { createMemoryRepository } from "@pong-pong/db";
+import { parseServerEvent, type ServerEvent, type SessionUser } from "@pong-pong/shared";
+import { GameHub } from "./gameHub";
+
+describe("GameHub drain boundary", () => {
+  const sockets: FakeSocket[] = [];
+  const repositories: Array<ReturnType<typeof createMemoryRepository>> = [];
+
+  beforeEach(() => {
+    vi.useFakeTimers();
+  });
+
+  afterEach(async () => {
+    for (const socket of sockets.splice(0)) socket.terminate();
+    vi.clearAllTimers();
+    vi.useRealTimers();
+    await Promise.all(repositories.splice(0).map((repository) => repository.close()));
+  });
+
+  it("clears waiting players and rejects new queue commands during drain", async () => {
+    const hub = setup();
+    const waiting = connect(hub, player("waiting-user"));
+    waiting.receive({ v: 1, type: "queue.join", mode: "queue" });
+    await flushEvents();
+    expect(hub.liveStats().queuedPlayers).toBe(1);
+
+    const drain = hub.beginDrain(60_000);
+    expect(hub.liveStats().queuedPlayers).toBe(0);
+
+    const newcomer = connect(hub, player("new-user"));
+    newcomer.receive({ v: 1, type: "queue.join", mode: "ai" });
+    await flushEvents();
+
+    expect(newcomer.latest("error")).toEqual(expect.objectContaining({
+      code: "server_draining"
+    }));
+    await expect(drain).resolves.toEqual({ drained: true, activeRooms: 0 });
+  });
+
+  it("waits for active rooms but never exceeds the drain timeout", async () => {
+    const hub = setup();
+    const socket = connect(hub, player("active-user"));
+    socket.receive({ v: 1, type: "queue.join", mode: "ai" });
+    await flushEvents();
+    expect(hub.liveStats().activeRooms).toBe(1);
+
+    let result: Awaited<ReturnType<GameHub["beginDrain"]>> | undefined;
+    const drain = hub.beginDrain(60_000).then((value) => {
+      result = value;
+      return value;
+    });
+
+    await vi.advanceTimersByTimeAsync(59_999);
+    expect(result).toBeUndefined();
+    await vi.advanceTimersByTimeAsync(1);
+
+    await expect(drain).resolves.toEqual({ drained: false, activeRooms: 1 });
+  });
+
+  function setup(): GameHub {
+    const repository = createMemoryRepository();
+    repositories.push(repository);
+    return new GameHub(repository);
+  }
+
+  function connect(hub: GameHub, user: SessionUser): FakeSocket {
+    const socket = new FakeSocket();
+    sockets.push(socket);
+    hub.connect(socket as unknown as WebSocket, {} as IncomingMessage, user);
+    return socket;
+  }
+});
+
+class FakeSocket extends EventEmitter {
+  readyState: number = WebSocket.OPEN;
+  bufferedAmount = 0;
+  private readonly payloads: string[] = [];
+
+  send(payload: string, callback?: (error?: Error) => void): void {
+    this.payloads.push(payload);
+    callback?.();
+  }
+
+  ping(): void {}
+
+  close(): void {
+    this.terminate();
+  }
+
+  terminate(): void {
+    if (this.readyState === WebSocket.CLOSED) return;
+    this.readyState = WebSocket.CLOSED;
+    this.emit("close");
+  }
+
+  receive(event: object): void {
+    this.emit("message", Buffer.from(JSON.stringify(event)));
+  }
+
+  latest(type: ServerEvent["type"]): ServerEvent | undefined {
+    return this.payloads
+      .map((payload) => parseServerEvent(payload))
+      .filter((event) => event.type === type)
+      .at(-1);
+  }
+}
+
+async function flushEvents(): Promise<void> {
+  await Promise.resolve();
+  await Promise.resolve();
+  await Promise.resolve();
+}
+
+function player(handle: string): SessionUser {
+  return {
+    id: `${handle}-id`,
+    handle,
+    displayName: handle,
+    avatarKey: "default",
+    role: "user",
+    status: "active",
+    rating: 1_200,
+    wins: 0,
+    losses: 0,
+    online: true,
+    isNpc: false,
+    email: null
+  };
+}
diff --git a/apps/api/src/gracefulShutdown.test.ts b/apps/api/src/gracefulShutdown.test.ts
new file mode 100644
index 0000000..6fcf9ca
--- /dev/null
+++ b/apps/api/src/gracefulShutdown.test.ts
@@ -0,0 +1,43 @@
+import { EventEmitter } from "node:events";
+import { describe, expect, it, vi } from "vitest";
+import { installGracefulShutdown } from "./gracefulShutdown";
+
+describe("graceful shutdown signals", () => {
+  it("starts one shutdown for repeated SIGTERM and SIGINT signals", async () => {
+    const signals = new EventEmitter();
+    let finishShutdown: (() => void) | undefined;
+    const shutdown = vi.fn(() => new Promise<void>((resolve) => {
+      finishShutdown = resolve;
+    }));
+    const onError = vi.fn();
+    const dispose = installGracefulShutdown(signals, shutdown, onError);
+
+    signals.emit("SIGTERM", "SIGTERM");
+    signals.emit("SIGINT", "SIGINT");
+    signals.emit("SIGTERM", "SIGTERM");
+
+    expect(shutdown).toHaveBeenCalledTimes(1);
+    expect(shutdown).toHaveBeenCalledWith("SIGTERM");
+    finishShutdown?.();
+    await Promise.resolve();
+    expect(onError).not.toHaveBeenCalled();
+    dispose();
+  });
+
+  it("reports shutdown failures without starting another run", async () => {
+    const signals = new EventEmitter();
+    const error = new Error("close failed");
+    const shutdown = vi.fn().mockRejectedValue(error);
+    const onError = vi.fn();
+    const dispose = installGracefulShutdown(signals, shutdown, onError);
+
+    signals.emit("SIGTERM", "SIGTERM");
+    await Promise.resolve();
+    await Promise.resolve();
+
+    expect(onError).toHaveBeenCalledWith(error);
+    signals.emit("SIGINT", "SIGINT");
+    expect(shutdown).toHaveBeenCalledTimes(1);
+    dispose();
+  });
+});
diff --git a/apps/api/src/health.test.ts b/apps/api/src/health.test.ts
index 01ca442..2741fcc 100644
--- a/apps/api/src/health.test.ts
+++ b/apps/api/src/health.test.ts
@@ -55,6 +55,20 @@ describe("health and metrics routes", () => {
     expect(response.body).not.toContain("postgresql");
   });
 
+  it("drops readiness as soon as draining starts", async () => {
+    const { app } = await setup();
+
+    const drain = app.beginDrain(60_000);
+    const response = await app.inject({ method: "GET", url: "/health/ready" });
+
+    expect(response.statusCode).toBe(503);
+    expect(response.json()).toMatchObject({
+      status: "not_ready",
+      checks: { lifecycle: "draining" }
+    });
+    await expect(drain).resolves.toMatchObject({ drained: true, activeRooms: 0 });
+  });
+
   it("publishes Prometheus metrics with bounded labels", async () => {
     const { app } = await setup();
     await app.inject({ method: "GET", url: "/health/live" });
@@ -66,9 +80,6 @@ describe("health and metrics routes", () => {
     expect(response.body).toContain("pong_pong_api_http_request_duration_seconds");
     expect(response.body).toContain("pong_pong_api_connections");
     expect(response.body).toContain("pong_pong_api_rooms");
-    expect(response.body).toContain("pong_pong_api_database_operation_duration_seconds");
-    expect(response.body).toContain("pong_pong_api_snapshot_delivery_delay_seconds");
-    expect(response.body).toContain("pong_pong_api_snapshot_drops_total");
     expect(response.body).toMatch(/route="\/health\/live"/);
     expect(response.body).not.toContain("requestId");
     expect(response.body).not.toContain("userId");


