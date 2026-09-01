## `feat(metrics): game room과 reconnect 관측 추가`

diff --git a/apps/api/src/app.ts b/apps/api/src/app.ts
index 02cb42b..7f54600 100644
--- a/apps/api/src/app.ts
+++ b/apps/api/src/app.ts
@@ -60,10 +60,22 @@ export function buildApp({
   let readGameStats = () => ({ onlinePlayers: 0, queuedPlayers: 0, activeRooms: 0 });
   const metrics = new ApiMetrics(() => readGameStats());
   const repo = instrumentRepository(sourceRepo, metrics);
-  const hub = new GameHub(repo);
+  const hub = new GameHub(repo, {
+    roomCreated: (context) => {
+      app.log.info(context, "game room created");
+    },
+    reconnect: (context) => {
+      metrics.recordReconnect(context.outcome);
+      app.log.info(context, "game connection recovery recorded");
+    }
+  });
   readGameStats = () => hub.liveStats();
   const guests = appMode === "demo" ? guestAccess ?? new GuestAccess({ secret: sessionSecret }) : null;
-  const getCurrentUser = (request: FastifyRequest) => currentUser(repo, request, guests, appMode === "demo");
+  const getCurrentUser = async (request: FastifyRequest) => {
+    const user = await currentUser(repo, request, guests, appMode === "demo");
+    if (user) request.log.debug({ userId: user.id }, "request authenticated");
+    return user;
+  };
 
   app.addHook("onResponse", (request, reply, done) => {
     metrics.observeRequest(
@@ -151,7 +163,8 @@ export function buildApp({
           }
           if (lease) socket.once("close", () => lease.release());
           socket.off("message", bufferPayload);
-          hub.connect(socket as WebSocket, request.raw, user, pendingPayloads);
+          request.log.info({ userId: user.id }, "websocket authenticated");
+          hub.connect(socket as WebSocket, request.raw, user, pendingPayloads, String(request.id));
         })
         .catch(() => closeAuthentication(1011, "websocket authentication failed"));
     });
diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index d7bb212..add8829 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -33,6 +33,7 @@ type Client = {
   roomId: string | null;
   heartbeat: ConnectionHeartbeat;
   snapshots: LatestSnapshotBuffer;
+  requestId: string | null;
 };
 
 type VersionlessServerEvent = ServerEvent extends infer Event
@@ -71,6 +72,20 @@ const CONNECTION_REPLACED_CLOSE_CODE = 4001;
 const CONNECTION_REPLACED_REASON = "connection replaced";
 const GUEST_RESULT_RETENTION_MS = 2 * 60 * 1_000;
 
+export interface GameHubObserver {
+  roomCreated?(context: {
+    roomId: string;
+    requestIds: string[];
+    userIds: string[];
+  }): void;
+  reconnect?(context: {
+    outcome: "success" | "expired";
+    roomId: string;
+    requestId?: string;
+    userId?: string;
+  }): void;
+}
+
 export class GameHub {
   private readonly clients = new Map<string, Client>();
   private readonly clientsByUser = new Map<string, Client>();
@@ -86,7 +101,10 @@ export class GameHub {
     cleanupTimer: NodeJS.Timeout;
   }>();
 
-  constructor(private readonly repo: AppRepository) {}
+  constructor(
+    private readonly repo: AppRepository,
+    private readonly observer: GameHubObserver = {}
+  ) {}
 
   get retainedGuestResultCount(): number {
     return this.recentGuestResults.size;
@@ -96,7 +114,13 @@ export class GameHub {
     return this.roomScheduler.activeRooms;
   }
 
-  connect(socket: WebSocket, _request: IncomingMessage, user: ConnectedUser, pendingPayloads: string[] = []): void {
+  connect(
+    socket: WebSocket,
+    _request: IncomingMessage,
+    user: ConnectedUser,
+    pendingPayloads: string[] = [],
+    requestId: string | null = null
+  ): void {
     const heartbeat = new ConnectionHeartbeat({
       ping: () => {
         if (socket.readyState === WebSocket.OPEN) socket.ping();
@@ -109,7 +133,8 @@ export class GameHub {
       user,
       roomId: null,
       heartbeat,
-      snapshots: new LatestSnapshotBuffer(socket)
+      snapshots: new LatestSnapshotBuffer(socket),
+      requestId
     };
     const previous = this.clientsByUser.get(user.id);
     this.clients.set(client.id, client);
@@ -230,6 +255,12 @@ export class GameHub {
         client.roomId = room.id;
         delete room.disconnectedUsers[side];
         this.sendMatchContext(client, room, side);
+        this.observer.reconnect?.({
+          outcome: "success",
+          roomId: room.id,
+          requestId: client.requestId ?? undefined,
+          userId: client.user.id
+        });
 
         if (room.session.state === "reconnecting") {
           this.send(client, { type: "game.snapshot", snapshot: room.snapshot });
@@ -273,6 +304,10 @@ export class GameHub {
       return;
     }
 
+    for (const userId of Object.values(room.disconnectedUsers)) {
+      if (userId) this.observer.reconnect?.({ outcome: "expired", roomId, userId });
+    }
+
     room.disconnectedUsers = {};
     if (!expiry.winnerSide) {
       this.abandonRoom(room);
@@ -524,6 +559,12 @@ export class GameHub {
       }
     };
     this.rooms.set(roomId, room);
+    this.observer.roomCreated?.({
+      roomId,
+      requestIds: [left.requestId, right?.requestId]
+        .filter((requestId): requestId is string => Boolean(requestId)),
+      userIds: [left.user.id, ...(right ? [right.user.id] : [])]
+    });
     left.roomId = roomId;
     if (right) right.roomId = roomId;
     this.send(left, { type: "queue.matched", roomId, side: "left", opponent: rightPlayer?.displayName ?? "연습 AI" });
diff --git a/apps/api/src/observability.ts b/apps/api/src/observability.ts
index 97d46aa..5085d39 100644
--- a/apps/api/src/observability.ts
+++ b/apps/api/src/observability.ts
@@ -1,4 +1,5 @@
 import {
+  Counter,
   Gauge,
   Histogram,
   Registry,
@@ -87,6 +88,12 @@ export class ApiMetrics {
     help: "Current game room count",
     registers: [this.registry]
   });
+  private readonly reconnects = new Counter({
+    name: "pong_pong_api_reconnects_total",
+    help: "Websocket room reconnection outcomes",
+    labelNames: ["outcome"] as const,
+    registers: [this.registry]
+  });
 
   constructor(private readonly readGameStats: () => LiveGameStats) {
     collectDefaultMetrics({
@@ -119,6 +126,10 @@ export class ApiMetrics {
     }, Math.max(0, durationMs) / 1_000);
   }
 
+  recordReconnect(outcome: "success" | "expired"): void {
+    this.reconnects.inc({ outcome });
+  }
+
   async scrape(): Promise<string> {
     const stats = this.readGameStats();
     this.connections.set(stats.onlinePlayers);


## `feat(metrics): match finalization 결과 관측 추가`

diff --git a/apps/api/src/app.ts b/apps/api/src/app.ts
index 7f54600..b14e4eb 100644
--- a/apps/api/src/app.ts
+++ b/apps/api/src/app.ts
@@ -67,6 +67,11 @@ export function buildApp({
     reconnect: (context) => {
       metrics.recordReconnect(context.outcome);
       app.log.info(context, "game connection recovery recorded");
+    },
+    matchFinalized: (context) => {
+      metrics.recordFinalization(context.persistence, context.outcome);
+      const level = context.outcome === "success" ? "info" : "warn";
+      app.log[level](context, "match finalization recorded");
     }
   });
   readGameStats = () => hub.liveStats();
diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index add8829..b8610ef 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -84,6 +84,13 @@ export interface GameHubObserver {
     requestId?: string;
     userId?: string;
   }): void;
+  matchFinalized?(context: {
+    outcome: "success" | "failure";
+    persistence: "database" | "memory";
+    roomId: string;
+    matchId: string | null;
+    userIds: string[];
+  }): void;
 }
 
 export class GameHub {
@@ -686,24 +693,50 @@ export class GameHub {
         rightScore: room.snapshot.state.rightScore,
         ratingDelta: 0
       };
+      this.observer.matchFinalized?.({
+        outcome: "success",
+        persistence: "memory",
+        roomId: room.id,
+        matchId: null,
+        userIds: roomUserIds(room)
+      });
       this.rememberGuestResult(room, result);
       this.broadcastRoom(room.id, { type: "game.finished", result });
       this.removeFinishedRoom(room);
       return;
     }
-    const finalized = await this.repo.finalizeMatch({
-      resultKey: `room:${room.id}:finished`,
-      mode: room.mode,
-      winnerId: winner?.id ?? null,
-      loserId: loser?.id ?? null,
-      scoreLeft: room.snapshot.state.leftScore,
-      scoreRight: room.snapshot.state.rightScore,
-      ...(room.tournamentMatchId ? {
-        tournament: {
-          tournamentMatchId: room.tournamentMatchId,
-          roomId: room.id
-        }
-      } : {})
+    let finalized: Awaited<ReturnType<AppRepository["finalizeMatch"]>>;
+    try {
+      finalized = await this.repo.finalizeMatch({
+        resultKey: `room:${room.id}:finished`,
+        mode: room.mode,
+        winnerId: winner?.id ?? null,
+        loserId: loser?.id ?? null,
+        scoreLeft: room.snapshot.state.leftScore,
+        scoreRight: room.snapshot.state.rightScore,
+        ...(room.tournamentMatchId ? {
+          tournament: {
+            tournamentMatchId: room.tournamentMatchId,
+            roomId: room.id
+          }
+        } : {})
+      });
+    } catch (error) {
+      this.observer.matchFinalized?.({
+        outcome: "failure",
+        persistence: "database",
+        roomId: room.id,
+        matchId: null,
+        userIds: roomUserIds(room)
+      });
+      throw error;
+    }
+    this.observer.matchFinalized?.({
+      outcome: "success",
+      persistence: "database",
+      roomId: room.id,
+      matchId: finalized.matchId,
+      userIds: roomUserIds(room)
     });
     const result: GameFinished = {
       roomId: room.id,
@@ -809,6 +842,12 @@ function isGuest(user: ConnectedUser): user is GuestSessionUser {
   return "sessionKind" in user && user.sessionKind === "guest";
 }
 
+function roomUserIds(room: Room): string[] {
+  return Object.values(room.clients)
+    .filter((client): client is Client => Boolean(client))
+    .map((client) => client.user.id);
+}
+
 function clearQueueTimer(entry: QueueEntry): void {
   if (entry.npcFallbackTimer) {
     clearTimeout(entry.npcFallbackTimer);
diff --git a/apps/api/src/observability.ts b/apps/api/src/observability.ts
index 5085d39..586666c 100644
--- a/apps/api/src/observability.ts
+++ b/apps/api/src/observability.ts
@@ -88,6 +88,12 @@ export class ApiMetrics {
     help: "Current game room count",
     registers: [this.registry]
   });
+  private readonly matchFinalizations = new Counter({
+    name: "pong_pong_api_match_finalizations_total",
+    help: "Completed match finalization attempts",
+    labelNames: ["persistence", "outcome"] as const,
+    registers: [this.registry]
+  });
   private readonly reconnects = new Counter({
     name: "pong_pong_api_reconnects_total",
     help: "Websocket room reconnection outcomes",
@@ -126,6 +132,10 @@ export class ApiMetrics {
     }, Math.max(0, durationMs) / 1_000);
   }
 
+  recordFinalization(persistence: "database" | "memory", outcome: "success" | "failure"): void {
+    this.matchFinalizations.inc({ persistence, outcome });
+  }
+
   recordReconnect(outcome: "success" | "expired"): void {
     this.reconnects.inc({ outcome });
   }


## `feat(metrics): snapshot delivery와 drop 관측 추가`

diff --git a/apps/api/src/app.ts b/apps/api/src/app.ts
index b14e4eb..26dcbc6 100644
--- a/apps/api/src/app.ts
+++ b/apps/api/src/app.ts
@@ -72,6 +72,12 @@ export function buildApp({
       metrics.recordFinalization(context.persistence, context.outcome);
       const level = context.outcome === "success" ? "info" : "warn";
       app.log[level](context, "match finalization recorded");
+    },
+    snapshotDelivered: (delayMs) => {
+      metrics.observeSnapshotDelivery(delayMs);
+    },
+    snapshotDropped: (reason) => {
+      metrics.recordSnapshotDrop(reason);
     }
   });
   readGameStats = () => hub.liveStats();
diff --git a/apps/api/src/game/latestSnapshotBuffer.ts b/apps/api/src/game/latestSnapshotBuffer.ts
index 1da83e7..a7675b2 100644
--- a/apps/api/src/game/latestSnapshotBuffer.ts
+++ b/apps/api/src/game/latestSnapshotBuffer.ts
@@ -13,11 +13,22 @@ export type SnapshotSocket = {
 
 type SnapshotBufferOptions = {
   now?: () => number;
+  onDelivered?: (delayMs: number) => void;
+  onDropped?: (reason: SnapshotDropReason) => void;
+};
+
+export type SnapshotDropReason = "replaced" | "connection_closed" | "congestion";
+
+type PendingSnapshot = {
+  payload: string;
+  enqueuedAtMs: number;
 };
 
 export class LatestSnapshotBuffer {
   private readonly now: () => number;
-  private pendingSnapshot: string | null = null;
+  private readonly onDelivered: (delayMs: number) => void;
+  private readonly onDropped: (reason: SnapshotDropReason) => void;
+  private pendingSnapshot: PendingSnapshot | null = null;
   private sending = false;
   private congestionStartedAtMs: number | null = null;
   private retryTimer: ReturnType<typeof setTimeout> | null = null;
@@ -25,16 +36,20 @@ export class LatestSnapshotBuffer {
 
   constructor(private readonly socket: SnapshotSocket, options: SnapshotBufferOptions = {}) {
     this.now = options.now ?? (() => performance.now());
+    this.onDelivered = options.onDelivered ?? (() => undefined);
+    this.onDropped = options.onDropped ?? (() => undefined);
   }
 
   enqueue(payload: string): void {
     if (this.closed) return;
-    this.pendingSnapshot = payload;
+    if (this.pendingSnapshot) this.onDropped("replaced");
+    this.pendingSnapshot = { payload, enqueuedAtMs: this.now() };
     this.drain();
   }
 
-  close(): void {
+  close(reason: SnapshotDropReason = "connection_closed"): void {
     this.closed = true;
+    if (this.pendingSnapshot) this.onDropped(reason);
     this.pendingSnapshot = null;
     if (this.retryTimer) clearTimeout(this.retryTimer);
     this.retryTimer = null;
@@ -49,14 +64,14 @@ export class LatestSnapshotBuffer {
 
     const nowMs = this.now();
     if (this.socket.bufferedAmount >= HARD_BUFFERED_AMOUNT_BYTES) {
-      this.terminate();
+      this.terminate("congestion");
       return;
     }
 
     if (this.socket.bufferedAmount > SOFT_BUFFERED_AMOUNT_BYTES) {
       this.congestionStartedAtMs ??= nowMs;
       if (nowMs - this.congestionStartedAtMs >= MAX_CONGESTION_MS) {
-        this.terminate();
+        this.terminate("congestion");
         return;
       }
       this.armRetry();
@@ -69,22 +84,25 @@ export class LatestSnapshotBuffer {
       return;
     }
 
-    const payload = this.pendingSnapshot;
-    if (payload === null) return;
+    const snapshot = this.pendingSnapshot;
+    if (snapshot === null) return;
     this.pendingSnapshot = null;
     this.sending = true;
     try {
-      this.socket.send(payload, (error) => {
+      this.socket.send(snapshot.payload, (error) => {
         this.sending = false;
         if (error) {
-          this.terminate();
+          this.onDropped("connection_closed");
+          this.terminate("connection_closed");
           return;
         }
+        this.onDelivered(Math.max(0, this.now() - snapshot.enqueuedAtMs));
         this.drain();
       });
     } catch {
       this.sending = false;
-      this.terminate();
+      this.onDropped("connection_closed");
+      this.terminate("connection_closed");
       return;
     }
     if (this.sending || this.pendingSnapshot !== null) this.armRetry();
@@ -98,9 +116,9 @@ export class LatestSnapshotBuffer {
     }, RETRY_INTERVAL_MS);
   }
 
-  private terminate(): void {
+  private terminate(reason: SnapshotDropReason): void {
     if (this.closed) return;
-    this.close();
+    this.close(reason);
     this.socket.terminate();
   }
 }
diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index b8610ef..d3e54d3 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -91,6 +91,8 @@ export interface GameHubObserver {
     matchId: string | null;
     userIds: string[];
   }): void;
+  snapshotDelivered?(delayMs: number): void;
+  snapshotDropped?(reason: "replaced" | "connection_closed" | "congestion"): void;
 }
 
 export class GameHub {
@@ -140,7 +142,10 @@ export class GameHub {
       user,
       roomId: null,
       heartbeat,
-      snapshots: new LatestSnapshotBuffer(socket),
+      snapshots: new LatestSnapshotBuffer(socket, {
+        onDelivered: (delayMs) => this.observer.snapshotDelivered?.(delayMs),
+        onDropped: (reason) => this.observer.snapshotDropped?.(reason)
+      }),
       requestId
     };
     const previous = this.clientsByUser.get(user.id);
diff --git a/apps/api/src/observability.ts b/apps/api/src/observability.ts
index 586666c..0e63442 100644
--- a/apps/api/src/observability.ts
+++ b/apps/api/src/observability.ts
@@ -73,6 +73,18 @@ export class ApiMetrics {
     buckets: [0.001, 0.0025, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5],
     registers: [this.registry]
   });
+  private readonly snapshotDeliveryDelay = new Histogram({
+    name: "pong_pong_api_snapshot_delivery_delay_seconds",
+    help: "Time from snapshot enqueue to websocket send completion",
+    buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.15, 0.25, 0.5, 1],
+    registers: [this.registry]
+  });
+  private readonly snapshotDrops = new Counter({
+    name: "pong_pong_api_snapshot_drops_total",
+    help: "Snapshots discarded before successful delivery",
+    labelNames: ["reason"] as const,
+    registers: [this.registry]
+  });
   private readonly connections = new Gauge({
     name: "pong_pong_api_connections",
     help: "Current websocket connection count",
@@ -132,6 +144,14 @@ export class ApiMetrics {
     }, Math.max(0, durationMs) / 1_000);
   }
 
+  observeSnapshotDelivery(delayMs: number): void {
+    this.snapshotDeliveryDelay.observe(Math.max(0, delayMs) / 1_000);
+  }
+
+  recordSnapshotDrop(reason: "replaced" | "connection_closed" | "congestion"): void {
+    this.snapshotDrops.inc({ reason });
+  }
+
   recordFinalization(persistence: "database" | "memory", outcome: "success" | "failure"): void {
     this.matchFinalizations.inc({ persistence, outcome });
   }


## `feat(metrics): event-loop lag 측정 추가`

diff --git a/apps/api/src/observability.ts b/apps/api/src/observability.ts
index 0e63442..e536d29 100644
--- a/apps/api/src/observability.ts
+++ b/apps/api/src/observability.ts
@@ -5,6 +5,7 @@ import {
   Registry,
   collectDefaultMetrics
 } from "prom-client";
+import { monitorEventLoopDelay, type IntervalHistogram } from "node:perf_hooks";
 import type { AppRepository } from "@pong-pong/db";
 
 interface LiveGameStats {
@@ -52,6 +53,8 @@ const REPOSITORY_OPERATIONS = new Set([
 
 export class ApiMetrics {
   private readonly registry = new Registry();
+  private readonly eventLoopDelay: IntervalHistogram;
+  private readonly eventLoopLagP95: Gauge;
   private readonly requestDuration = new Histogram({
     name: "pong_pong_api_http_request_duration_seconds",
     help: "HTTP request duration in seconds",
@@ -114,6 +117,19 @@ export class ApiMetrics {
   });
 
   constructor(private readonly readGameStats: () => LiveGameStats) {
+    this.eventLoopDelay = monitorEventLoopDelay({ resolution: 20 });
+    this.eventLoopLagP95 = new Gauge({
+      name: "pong_pong_api_event_loop_lag_p95_seconds",
+      help: "95th percentile of recorded event loop delay in seconds",
+      registers: [this.registry],
+      collect: () => {
+        const delayNanoseconds = this.eventLoopDelay.percentile(95);
+        this.eventLoopLagP95.set(
+          Number.isFinite(delayNanoseconds) ? delayNanoseconds / 1_000_000_000 : 0
+        );
+      }
+    });
+    this.eventLoopDelay.enable();
     collectDefaultMetrics({
       register: this.registry,
       prefix: "pong_pong_api_",
@@ -169,6 +185,7 @@ export class ApiMetrics {
   }
 
   close(): void {
+    this.eventLoopDelay.disable();
     this.registry.clear();
   }
 }


## `fix(db): idle connection pool 오류에서 복구`

diff --git a/apps/api/src/index.ts b/apps/api/src/index.ts
index ccfa61d..e0738af 100644
--- a/apps/api/src/index.ts
+++ b/apps/api/src/index.ts
@@ -1,10 +1,24 @@
-import { createMemoryRepository, createPostgresRepository } from "@pong-pong/db";
+import {
+  createMemoryRepository,
+  createPostgresRepository,
+  type PostgresPoolErrorEvent
+} from "@pong-pong/db";
 import { buildApp } from "./app.js";
 import { readEnv } from "./env.js";
 import { installGracefulShutdown } from "./gracefulShutdown.js";
 
 const env = readEnv();
-const repo = env.databaseUrl ? createPostgresRepository(env.databaseUrl) : createMemoryRepository();
+const earlyPoolErrors: PostgresPoolErrorEvent[] = [];
+let reportPoolError = (event: PostgresPoolErrorEvent) => {
+  earlyPoolErrors.push(event);
+};
+const repo = env.databaseUrl
+  ? createPostgresRepository(env.databaseUrl, {
+      onPoolError: (event) => {
+        reportPoolError(event);
+      }
+    })
+  : createMemoryRepository();
 
 const app = buildApp({
   repo,
@@ -13,6 +27,12 @@ const app = buildApp({
   sessionSecret: env.sessionSecret,
   trustProxy: env.trustProxy
 });
+reportPoolError = (event) => {
+  app.log.error(event, "PostgreSQL idle client connection failed");
+};
+for (const event of earlyPoolErrors.splice(0)) {
+  reportPoolError(event);
+}
 app.addHook("onClose", async () => {
   await repo.close();
 });
diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index e9077a5..e46ffdf 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -40,8 +40,13 @@ import type {
   UserRow
 } from "./schema.js";
 import { inspectMigrationSet } from "./migrator.js";
+import {
+  installPostgresPoolErrorHandler,
+  type PostgresPoolErrorReporter
+} from "./poolError.js";
 
 export type { Database } from "./schema.js";
+export type { PostgresPoolErrorEvent, PostgresPoolErrorReporter } from "./poolError.js";
 
 type MemoryFriendship = {
   id: string;
@@ -166,8 +171,16 @@ export interface AppRepository extends MatchResultRepository {
   setUserBan(actorId: string, targetUserId: string, banned: boolean, reason: string): Promise<PublicUser>;
 }
 
-export function createPostgresRepository(databaseUrl: string): AppRepository {
+export interface PostgresRepositoryOptions {
+  onPoolError?: PostgresPoolErrorReporter;
+}
+
+export function createPostgresRepository(
+  databaseUrl: string,
+  options: PostgresRepositoryOptions = {}
+): AppRepository {
   const pool = new Pool({ connectionString: databaseUrl });
+  installPostgresPoolErrorHandler(pool, options.onPoolError);
   const db = new Kysely<Database>({ dialect: new PostgresDialect({ pool }) });
   return new PostgresRepository(db, pool);
 }
diff --git a/packages/db/src/poolError.ts b/packages/db/src/poolError.ts
new file mode 100644
index 0000000..44cf498
--- /dev/null
+++ b/packages/db/src/poolError.ts
@@ -0,0 +1,56 @@
+import type { Pool } from "pg";
+
+export interface PostgresPoolErrorEvent {
+  kind: "idle_client_error";
+  errorName: string;
+  errorCode: string | null;
+}
+
+export type PostgresPoolErrorReporter = (event: PostgresPoolErrorEvent) => void;
+
+const FALLBACK_EVENT: PostgresPoolErrorEvent = {
+  kind: "idle_client_error",
+  errorName: "UnknownError",
+  errorCode: null
+};
+
+export function installPostgresPoolErrorHandler(
+  pool: Pick<Pool, "on">,
+  onPoolError?: PostgresPoolErrorReporter
+): void {
+  pool.on("error", (error) => {
+    let event = FALLBACK_EVENT;
+    try {
+      event = toSafePoolErrorEvent(error);
+    } catch {
+      // A malformed error object must not escape the pool's EventEmitter boundary.
+    }
+
+    try {
+      onPoolError?.(event);
+    } catch {
+      // Reporting is best-effort and must not turn an idle client failure into a process crash.
+    }
+  });
+}
+
+function toSafePoolErrorEvent(error: Error): PostgresPoolErrorEvent {
+  const errorName = safeLabel(error.name, "UnknownError");
+  const errorCode = safeLabel(readErrorCode(error), null);
+  return {
+    kind: "idle_client_error",
+    errorName,
+    errorCode
+  };
+}
+
+function readErrorCode(error: Error): unknown {
+  return "code" in error ? error.code : undefined;
+}
+
+function safeLabel<T extends string | null>(value: unknown, fallback: T): string | T {
+  if (typeof value !== "string" || !/^[A-Za-z0-9_]{1,64}$/.test(value)) {
+    return fallback;
+  }
+  return value;
+}


