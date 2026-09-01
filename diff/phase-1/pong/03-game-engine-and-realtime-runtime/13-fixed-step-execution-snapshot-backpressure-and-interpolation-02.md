## `feat(game): latest snapshot buffer를 GameHub에 연결`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index f52e1aa..05a3e5f 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -516,9 +516,19 @@ export class GameHub {
   }
 
   private send(client: Client, event: VersionlessServerEvent): void {
-    if (client.socket.readyState === WebSocket.OPEN) {
-      client.socket.send(encodeServerEvent({ ...event, v: 1 } as ServerEvent));
+    if (client.socket.readyState !== WebSocket.OPEN) return;
+    const payload = encodeServerEvent({ ...event, v: 1 } as ServerEvent);
+    if (event.type === "game.snapshot") {
+      client.snapshots.enqueue(payload);
+      return;
+    }
+    if (client.socket.bufferedAmount >= HARD_BUFFERED_AMOUNT_BYTES) {
+      client.socket.terminate();
+      return;
     }
+    client.socket.send(payload, (error) => {
+      if (error && client.socket.readyState === WebSocket.OPEN) client.socket.terminate();
+    });
   }
 }
 


## `refactor(game): shared room scheduler 추가`

diff --git a/apps/api/src/game/sharedRoomScheduler.ts b/apps/api/src/game/sharedRoomScheduler.ts
new file mode 100644
index 0000000..b89295c
--- /dev/null
+++ b/apps/api/src/game/sharedRoomScheduler.ts
@@ -0,0 +1,42 @@
+import { FixedStepScheduler } from "./fixedStepScheduler.js";
+
+type SharedRoomSchedulerOptions = {
+  now?: () => number;
+};
+
+export class SharedRoomScheduler {
+  private readonly roomSteps = new Map<string, () => void>();
+  private readonly scheduler: FixedStepScheduler;
+
+  constructor(options: SharedRoomSchedulerOptions = {}) {
+    this.scheduler = new FixedStepScheduler(() => this.stepRooms(), {
+      now: options.now,
+      timestepMs: 50,
+      maxTicksPerLoop: 5,
+      maxAccumulatedMs: 250
+    });
+  }
+
+  get activeRooms(): number {
+    return this.roomSteps.size;
+  }
+
+  register(roomId: string, step: () => void): void {
+    this.roomSteps.set(roomId, step);
+    this.scheduler.start();
+  }
+
+  unregister(roomId: string): void {
+    this.roomSteps.delete(roomId);
+    if (this.roomSteps.size === 0) this.scheduler.stop();
+  }
+
+  stop(): void {
+    this.roomSteps.clear();
+    this.scheduler.stop();
+  }
+
+  private stepRooms(): void {
+    for (const step of [...this.roomSteps.values()]) step();
+  }
+}


## `refactor(game): GameHub가 shared room scheduler 사용`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index 9bc848c..306f017 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -14,13 +14,14 @@ import {
   type SessionUser,
   WINNING_SCORE
 } from "@pong-pong/shared";
-import { DEFAULT_TIMESTEP_MS, FixedStepScheduler } from "./game/fixedStepScheduler";
+import { DEFAULT_TIMESTEP_MS } from "./game/fixedStepScheduler";
 import { ConnectionHeartbeat } from "./game/heartbeat";
 import { InputGate } from "./game/inputGate";
 import { HARD_BUFFERED_AMOUNT_BYTES, LatestSnapshotBuffer } from "./game/latestSnapshotBuffer";
 import { PongAi } from "./game/pongAi";
 import { PongSimulation, type PongSimulationState } from "./game/pongSimulation";
 import { RoomSession } from "./game/roomSession";
+import { SharedRoomScheduler } from "./game/sharedRoomScheduler.js";
 import type { GuestSessionUser } from "./guestAccess.js";
 
 type ConnectedUser = SessionUser | GuestSessionUser;
@@ -52,7 +53,6 @@ type Room = {
   ai: boolean;
   ready: Partial<Record<PlayerSide, boolean>>;
   snapshot: GameSnapshot;
-  scheduler: FixedStepScheduler | null;
   mode: MatchMode;
   tournamentMatchId: string | null;
   npcUser: PublicUser | null;
@@ -79,6 +79,7 @@ export class GameHub {
   private readonly tournamentWaiters = new Map<string, Client[]>();
   private readonly waitSamples: number[] = [];
   private readonly inputGate = new InputGate();
+  private readonly roomScheduler = new SharedRoomScheduler();
   private readonly recentGuestResults = new Map<string, {
     result: GameFinished;
     expiresAtMs: number;
@@ -91,6 +92,10 @@ export class GameHub {
     return this.recentGuestResults.size;
   }
 
+  get scheduledRoomCount(): number {
+    return this.roomScheduler.activeRooms;
+  }
+
   connect(socket: WebSocket, _request: IncomingMessage, user: ConnectedUser, pendingPayloads: string[] = []): void {
     const heartbeat = new ConnectionHeartbeat({
       ping: () => {
@@ -244,7 +249,7 @@ export class GameHub {
     if (room.finishing || room.session.state === "finished") return;
     room.session.disconnect(side, Date.now());
     room.disconnectedUsers[side] = userId;
-    room.scheduler?.stop();
+    this.roomScheduler.unregister(room.id);
     room.snapshot.state.paddles[side].dy = 0;
     room.snapshot.state.phase = "paused";
     this.armReconnectTimer(room);
@@ -282,7 +287,7 @@ export class GameHub {
   }
 
   private abandonRoom(room: Room): void {
-    room.scheduler?.stop();
+    this.roomScheduler.unregister(room.id);
     this.clearReconnectTimer(room);
     for (const client of Object.values(room.clients)) {
       if (client) client.roomId = null;
@@ -477,7 +482,6 @@ export class GameHub {
       clients: { left, ...(right ? { right } : {}) },
       ai: options.ai,
       ready: {},
-      scheduler: null,
       mode: options.mode,
       tournamentMatchId: options.tournamentMatchId ?? null,
       npcUser,
@@ -573,7 +577,7 @@ export class GameHub {
   private pauseRoom(client: Client, roomId: string): void {
     const room = this.rooms.get(roomId);
     if (!room || room.snapshot.state.phase !== "playing" || !sideFor(room, client)) return;
-    room.scheduler?.stop();
+    this.roomScheduler.unregister(room.id);
     const sessionState = room.session.pause();
     if (sessionState !== "paused") return;
     room.snapshot.state.phase = sessionState;
@@ -591,12 +595,7 @@ export class GameHub {
   }
 
   private startRoomScheduler(room: Room): void {
-    room.scheduler ??= new FixedStepScheduler(() => this.tick(room), {
-      timestepMs: SIMULATION_TIMESTEP_MS,
-      maxTicksPerLoop: 5,
-      maxAccumulatedMs: 250
-    });
-    room.scheduler.start();
+    this.roomScheduler.register(room.id, () => this.tick(room));
   }
 
   private tick(room: Room): void {
@@ -627,7 +626,7 @@ export class GameHub {
   }
 
   private async finalizeRoom(room: Room, winnerSide: PlayerSide): Promise<void> {
-    room.scheduler?.stop();
+    this.roomScheduler.unregister(room.id);
     this.clearReconnectTimer(room);
     room.disconnectedUsers = {};
     room.session.finish();
@@ -708,6 +707,7 @@ export class GameHub {
   }
 
   private removeFinishedRoom(room: Room): void {
+    this.roomScheduler.unregister(room.id);
     for (const client of Object.values(room.clients)) {
       if (client) client.roomId = null;
     }


## `fix(game): 부하 중 snapshot cadence 안정화`

diff --git a/apps/api/src/app.ts b/apps/api/src/app.ts
index b4bf9c7..a9f93f0 100644
--- a/apps/api/src/app.ts
+++ b/apps/api/src/app.ts
@@ -75,7 +75,7 @@ export function buildApp({
       app.log.info(context, "game connection recovery recorded");
     },
     matchFinalized: (context) => {
-      metrics.recordFinalization(context.persistence, context.outcome);
+      metrics.recordFinalization(context.persistence, context.outcome, context.created);
       const level = context.outcome === "success" ? "info" : "warn";
       app.log[level](context, "match finalization recorded");
     },
diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index fb49327..6cd8465 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -73,10 +73,12 @@ type Room = {
   reconnectTimer: NodeJS.Timeout | null;
   disconnectedUsers: Partial<Record<PlayerSide, string>>;
   guest: boolean;
+  snapshotDeliverySlot: number;
 };
 
 const MAX_MATCHMAKING_RATING_DIFFERENCE = 200;
 const SIMULATION_TIMESTEP_MS = DEFAULT_TIMESTEP_MS;
+const SNAPSHOT_DELIVERY_DIVISOR = 2;
 const CONNECTION_REPLACED_CLOSE_CODE = 4001;
 const CONNECTION_REPLACED_REASON = "connection replaced";
 const GUEST_RESULT_RETENTION_MS = 2 * 60 * 1_000;
@@ -103,6 +105,7 @@ export interface GameHubObserver {
   matchFinalized?(context: {
     outcome: "success" | "failure";
     persistence: "database" | "memory";
+    created: boolean | null;
     roomId: string;
     matchId: string | null;
     userIds: string[];
@@ -124,6 +127,7 @@ export class GameHub {
   private readonly waitSamples: number[] = [];
   private readonly inputGate = new InputGate();
   private readonly roomScheduler = new SharedRoomScheduler();
+  private nextSnapshotDeliverySlot = 0;
   private readonly recentGuestResults = new Map<string, {
     result: GameFinished;
     expiresAtMs: number;
@@ -674,6 +678,9 @@ export class GameHub {
     const rightPlayer = right?.user ?? npcUser;
     const simulation = PongSimulation.initialState();
     const session = new RoomSession();
+    const snapshotDeliverySlot = this.nextSnapshotDeliverySlot;
+    this.nextSnapshotDeliverySlot =
+      (this.nextSnapshotDeliverySlot + 1) % SNAPSHOT_DELIVERY_DIVISOR;
     if (options.ai) session.markReady("right");
     const room: Room = {
       id: roomId,
@@ -690,6 +697,7 @@ export class GameHub {
       reconnectTimer: null,
       disconnectedUsers: {},
       guest: isGuest(left.user),
+      snapshotDeliverySlot,
       snapshot: {
         roomId,
         tick: 0,
@@ -823,7 +831,11 @@ export class GameHub {
       right: rightDirection
     }, SIMULATION_TIMESTEP_MS);
     syncSnapshot(room);
-    this.broadcastSnapshot(room);
+    if (
+      (room.simulation.tick + room.snapshotDeliverySlot) % SNAPSHOT_DELIVERY_DIVISOR === 0
+    ) {
+      this.broadcastSnapshot(room);
+    }
 
     if (room.simulation.phase === "finished" && room.simulation.winnerSide) {
       this.finishRoom(room, room.simulation.winnerSide).catch(() => undefined);
@@ -864,6 +876,7 @@ export class GameHub {
         this.observer.matchFinalized?.({
           outcome: "success",
           persistence: "memory",
+          created: null,
           roomId: room.id,
           matchId: null,
           userIds: roomUserIds(room)
@@ -896,6 +909,7 @@ export class GameHub {
       this.observer.matchFinalized?.({
         outcome: "failure",
         persistence: "database",
+        created: null,
         roomId: room.id,
         matchId: null,
         userIds: roomUserIds(room)
@@ -906,6 +920,7 @@ export class GameHub {
       this.observer.matchFinalized?.({
         outcome: "success",
         persistence: "database",
+        created: finalized.created,
         roomId: room.id,
         matchId: finalized.matchId,
         userIds: roomUserIds(room)
diff --git a/apps/api/src/observability.ts b/apps/api/src/observability.ts
index b4585b7..588ee2f 100644
--- a/apps/api/src/observability.ts
+++ b/apps/api/src/observability.ts
@@ -108,6 +108,11 @@ export class ApiMetrics {
     labelNames: ["persistence", "outcome"] as const,
     registers: [this.registry]
   });
+  private readonly matchFinalizationDuplicates = new Counter({
+    name: "pong_pong_api_match_finalization_duplicates_total",
+    help: "Match finalizations that returned an existing persisted result",
+    registers: [this.registry]
+  });
   private readonly reconnects = new Counter({
     name: "pong_pong_api_reconnects_total",
     help: "Websocket room reconnection outcomes",
@@ -167,8 +172,15 @@ export class ApiMetrics {
     this.snapshotDrops.inc({ reason });
   }
 
-  recordFinalization(persistence: "database" | "memory", outcome: "success" | "failure"): void {
+  recordFinalization(
+    persistence: "database" | "memory",
+    outcome: "success" | "failure",
+    created: boolean | null
+  ): void {
     this.matchFinalizations.inc({ persistence, outcome });
+    if (persistence === "database" && outcome === "success" && created === false) {
+      this.matchFinalizationDuplicates.inc();
+    }
   }
 
   recordReconnect(outcome: "success" | "expired"): void {


## `fix(game): callback 지연을 snapshot congestion으로 오판하지 않음`

diff --git a/apps/api/src/game/latestSnapshotBuffer.test.ts b/apps/api/src/game/latestSnapshotBuffer.test.ts
index 80e32a0..49971d8 100644
--- a/apps/api/src/game/latestSnapshotBuffer.test.ts
+++ b/apps/api/src/game/latestSnapshotBuffer.test.ts
@@ -9,7 +9,7 @@ import {
 describe("LatestSnapshotBuffer", () => {
   afterEach(() => vi.useRealTimers());
 
-  it("keeps only the newest snapshot while a send is in flight", () => {
+  it("sends snapshots while a send callback is in flight", () => {
     const socket = fakeSocket();
     const buffer = new LatestSnapshotBuffer(socket);
 
@@ -17,9 +17,9 @@ describe("LatestSnapshotBuffer", () => {
     buffer.enqueue("snapshot-2");
     buffer.enqueue("snapshot-3");
 
-    expect(socket.sent).toEqual(["snapshot-1"]);
+    expect(socket.sent).toEqual(["snapshot-1", "snapshot-2", "snapshot-3"]);
     socket.completeSend();
-    expect(socket.sent).toEqual(["snapshot-1", "snapshot-3"]);
+    expect(socket.sent).toEqual(["snapshot-1", "snapshot-2", "snapshot-3"]);
     socket.completeSend();
     buffer.close();
   });
diff --git a/apps/api/src/game/latestSnapshotBuffer.ts b/apps/api/src/game/latestSnapshotBuffer.ts
index a7675b2..0666fb4 100644
--- a/apps/api/src/game/latestSnapshotBuffer.ts
+++ b/apps/api/src/game/latestSnapshotBuffer.ts
@@ -29,7 +29,6 @@ export class LatestSnapshotBuffer {
   private readonly onDelivered: (delayMs: number) => void;
   private readonly onDropped: (reason: SnapshotDropReason) => void;
   private pendingSnapshot: PendingSnapshot | null = null;
-  private sending = false;
   private congestionStartedAtMs: number | null = null;
   private retryTimer: ReturnType<typeof setTimeout> | null = null;
   private closed = false;
@@ -79,33 +78,23 @@ export class LatestSnapshotBuffer {
     }
 
     this.congestionStartedAtMs = null;
-    if (this.sending) {
-      this.armRetry();
-      return;
-    }
-
     const snapshot = this.pendingSnapshot;
     if (snapshot === null) return;
     this.pendingSnapshot = null;
-    this.sending = true;
     try {
       this.socket.send(snapshot.payload, (error) => {
-        this.sending = false;
         if (error) {
           this.onDropped("connection_closed");
           this.terminate("connection_closed");
           return;
         }
         this.onDelivered(Math.max(0, this.now() - snapshot.enqueuedAtMs));
-        this.drain();
       });
     } catch {
-      this.sending = false;
       this.onDropped("connection_closed");
       this.terminate("connection_closed");
       return;
     }
-    if (this.sending || this.pendingSnapshot !== null) this.armRetry();
   }
 
   private armRetry(): void {


## `test(game): callback 지연과 실제 congestion 구분`

diff --git a/apps/api/src/game/latestSnapshotBuffer.test.ts b/apps/api/src/game/latestSnapshotBuffer.test.ts
index 49971d8..bd87d19 100644
--- a/apps/api/src/game/latestSnapshotBuffer.test.ts
+++ b/apps/api/src/game/latestSnapshotBuffer.test.ts
@@ -9,17 +9,19 @@ import {
 describe("LatestSnapshotBuffer", () => {
   afterEach(() => vi.useRealTimers());
 
-  it("sends snapshots while a send callback is in flight", () => {
+  it("does not treat delayed send callbacks as socket congestion", () => {
+    const onDropped = vi.fn();
     const socket = fakeSocket();
-    const buffer = new LatestSnapshotBuffer(socket);
+    const buffer = new LatestSnapshotBuffer(socket, { onDropped });
 
     buffer.enqueue("snapshot-1");
     buffer.enqueue("snapshot-2");
     buffer.enqueue("snapshot-3");
 
     expect(socket.sent).toEqual(["snapshot-1", "snapshot-2", "snapshot-3"]);
+    expect(onDropped).not.toHaveBeenCalled();
+    socket.completeSend();
     socket.completeSend();
-    expect(socket.sent).toEqual(["snapshot-1", "snapshot-2", "snapshot-3"]);
     socket.completeSend();
     buffer.close();
   });
@@ -27,16 +29,17 @@ describe("LatestSnapshotBuffer", () => {
   it("replaces congested snapshots and sends the latest after pressure clears", () => {
     vi.useFakeTimers();
     const socket = fakeSocket();
-    socket.bufferedAmount = SOFT_BUFFERED_AMOUNT_BYTES + 1;
     const buffer = new LatestSnapshotBuffer(socket);
 
     buffer.enqueue("snapshot-1");
+    socket.bufferedAmount = SOFT_BUFFERED_AMOUNT_BYTES + 1;
     buffer.enqueue("snapshot-2");
-    expect(socket.sent).toEqual([]);
+    buffer.enqueue("snapshot-3");
+    expect(socket.sent).toEqual(["snapshot-1"]);
 
     socket.bufferedAmount = 0;
     vi.advanceTimersByTime(50);
-    expect(socket.sent).toEqual(["snapshot-2"]);
+    expect(socket.sent).toEqual(["snapshot-1", "snapshot-3"]);
   });
 
   it("reports replacement drops and delivery delay without connection identifiers", () => {
@@ -101,18 +104,17 @@ type FakeSnapshotSocket = SnapshotSocket & {
 };
 
 function fakeSocket(): FakeSnapshotSocket {
-  let completion: ((error?: Error) => void) | null = null;
+  const completions: Array<(error?: Error) => void> = [];
   return {
     readyState: 1,
     bufferedAmount: 0,
     sent: [],
     send(payload, callback) {
       this.sent.push(payload);
-      completion = callback;
+      completions.push(callback);
     },
     completeSend(error) {
-      const callback = completion;
-      completion = null;
+      const callback = completions.shift();
       callback?.(error);
     },
     terminate: vi.fn()
