## `refactor(game): queue와 reservation cleanup 일원화`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index e911180..d254a47 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -45,7 +45,6 @@ type VersionlessServerEvent = ServerEvent extends infer Event
 
 type QueueEntry = {
   client: Client;
-  queuedAt: number;
   queuedAtMs: number;
   npcFallbackTimer: NodeJS.Timeout | null;
 };
@@ -76,7 +75,6 @@ type Room = {
   guest: boolean;
 };
 
-const NPC_QUEUE_FALLBACK_MS = 6000;
 const MAX_MATCHMAKING_RATING_DIFFERENCE = 200;
 const SIMULATION_TIMESTEP_MS = DEFAULT_TIMESTEP_MS;
 const CONNECTION_REPLACED_CLOSE_CODE = 4001;
@@ -114,7 +112,6 @@ export interface GameHubObserver {
 export class GameHub {
   private readonly clients = new Map<string, Client>();
   private readonly clientsByUser = new Map<string, Client>();
-  private readonly queue: QueueEntry[] = [];
   private readonly queueEntries = new Map<string, QueueEntry>();
   private readonly matchmaker = new Matchmaker({
     clock: () => Date.now(),
@@ -363,6 +360,7 @@ export class GameHub {
   private abandonRoom(room: Room): void {
     this.roomScheduler.unregister(room.id);
     this.clearReconnectTimer(room);
+    this.releaseMatchmakingReservations(room);
     for (const client of Object.values(room.clients)) {
       if (client) client.roomId = null;
     }
@@ -411,7 +409,6 @@ export class GameHub {
     if (join.type === "queued") {
       const entry: QueueEntry = {
         client,
-        queuedAt: join.queuedAtMs,
         queuedAtMs: join.queuedAtMs,
         npcFallbackTimer: null
       };
@@ -546,27 +543,13 @@ export class GameHub {
     await this.repo.startTournamentMatch(matchId, roomId);
   }
 
-  private findClosestQueuedOpponent(client: Client): number {
-    let bestIndex = -1;
-    let bestDistance = Number.POSITIVE_INFINITY;
-    for (let index = 0; index < this.queue.length; index += 1) {
-      const candidate = this.queue[index];
-      if (isGuest(candidate.client.user) !== isGuest(client.user)) continue;
-      const distance = Math.abs(candidate.client.user.rating - client.user.rating);
-      if (distance < bestDistance) {
-        bestDistance = distance;
-        bestIndex = index;
-      }
-    }
-    return bestIndex;
-  }
-
   private leaveQueue(client: Client): void {
-    const index = this.queue.findIndex((queued) => queued.client.id === client.id);
-    if (index >= 0) {
-      const [entry] = this.queue.splice(index, 1);
-      clearQueueTimer(entry);
-    }
+    const entry = this.queueEntries.get(client.user.id);
+    const leftQueue = this.matchmaker.leaveQueue(client.user.id);
+    if (!entry) return;
+    if (!leftQueue) this.matchmaker.release(client.user.id);
+    this.queueEntries.delete(client.user.id);
+    clearQueueTimer(entry);
   }
 
   private leaveTournamentWaiters(client: Client): void {
@@ -578,11 +561,8 @@ export class GameHub {
   }
 
   private pruneQueue(): void {
-    for (let index = this.queue.length - 1; index >= 0; index -= 1) {
-      if (this.queue[index].client.socket.readyState !== WebSocket.OPEN) {
-        const [entry] = this.queue.splice(index, 1);
-        clearQueueTimer(entry);
-      }
+    for (const entry of this.queueEntries.values()) {
+      if (entry.client.socket.readyState !== WebSocket.OPEN) this.leaveQueue(entry.client);
     }
   }
 
@@ -594,7 +574,7 @@ export class GameHub {
     return {
       onlinePlayers: this.clients.size,
       playingPlayers,
-      queuedPlayers: this.queue.length,
+      queuedPlayers: this.matchmaker.queuedCount,
       activeRooms: this.rooms.size,
       averageWaitSeconds
     };
@@ -602,8 +582,8 @@ export class GameHub {
 
   beginDrain(timeoutMs: number): Promise<DrainResult> {
     this.acceptingMatches = false;
-    for (const entry of this.queue.splice(0)) {
-      clearQueueTimer(entry);
+    for (const entry of [...this.queueEntries.values()]) {
+      this.leaveQueue(entry.client);
       this.sendDrainingError(entry.client);
     }
     for (const waiters of this.tournamentWaiters.values()) {
@@ -631,10 +611,13 @@ export class GameHub {
 
   close(): void {
     this.acceptingMatches = false;
-    for (const entry of this.queue.splice(0)) clearQueueTimer(entry);
+    for (const entry of [...this.queueEntries.values()]) this.leaveQueue(entry.client);
     this.tournamentWaiters.clear();
     this.roomScheduler.stop();
-    for (const room of this.rooms.values()) this.clearReconnectTimer(room);
+    for (const room of this.rooms.values()) {
+      this.clearReconnectTimer(room);
+      this.releaseMatchmakingReservations(room);
+    }
     this.rooms.clear();
     for (const recent of this.recentGuestResults.values()) clearTimeout(recent.cleanupTimer);
     this.recentGuestResults.clear();
@@ -847,19 +830,22 @@ export class GameHub {
         rightScore: room.snapshot.state.rightScore,
         ratingDelta: 0
       };
-      this.observer.matchFinalized?.({
-        outcome: "success",
-        persistence: "memory",
-        roomId: room.id,
-        matchId: null,
-        userIds: roomUserIds(room)
-      });
-      this.rememberGuestResult(room, result);
-      this.broadcastRoom(room.id, { type: "game.finished", result });
-      this.removeFinishedRoom(room);
+      try {
+        this.observer.matchFinalized?.({
+          outcome: "success",
+          persistence: "memory",
+          roomId: room.id,
+          matchId: null,
+          userIds: roomUserIds(room)
+        });
+        this.rememberGuestResult(room, result);
+        this.broadcastRoom(room.id, { type: "game.finished", result });
+      } finally {
+        this.removeFinishedRoom(room);
+      }
       return;
     }
-    let finalized: Awaited<ReturnType<AppRepository["finalizeMatch"]>>;
+    let finalized: Awaited<ReturnType<MatchResultRepository["finalizeMatch"]>>;
     try {
       finalized = await this.repo.finalizeMatch({
         resultKey: `room:${room.id}:finished`,
@@ -876,6 +862,7 @@ export class GameHub {
         } : {})
       });
     } catch (error) {
+      this.releaseMatchmakingReservations(room);
       this.observer.matchFinalized?.({
         outcome: "failure",
         persistence: "database",
@@ -936,6 +923,7 @@ export class GameHub {
 
   private removeFinishedRoom(room: Room): void {
     this.roomScheduler.unregister(room.id);
+    this.releaseMatchmakingReservations(room);
     for (const client of Object.values(room.clients)) {
       if (client) client.roomId = null;
     }
@@ -944,6 +932,12 @@ export class GameHub {
     this.broadcastPresence();
   }
 
+  private releaseMatchmakingReservations(room: Room): void {
+    for (const client of Object.values(room.clients)) {
+      if (client) this.matchmaker.release(client.user.id);
+    }
+  }
+
   private sendDrainingError(client: Client): void {
     this.send(client, {
       type: "error",


## `test(game): matchmaking lifecycle 검증`

diff --git a/apps/api/src/gameHub.matchmaking.test.ts b/apps/api/src/gameHub.matchmaking.test.ts
new file mode 100644
index 0000000..a4df4e8
--- /dev/null
+++ b/apps/api/src/gameHub.matchmaking.test.ts
@@ -0,0 +1,211 @@
+import { EventEmitter } from "node:events";
+import type { IncomingMessage } from "node:http";
+import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
+import { WebSocket } from "ws";
+import { createMemoryRepository } from "@pong-pong/db";
+import { parseServerEvent, type ServerEvent, type SessionUser } from "@pong-pong/shared";
+import { GameHub } from "./gameHub.js";
+
+describe("GameHub matchmaking boundary", () => {
+  const repositories: Array<ReturnType<typeof createMemoryRepository>> = [];
+  const sockets: FakeSocket[] = [];
+
+  beforeEach(() => {
+    vi.useFakeTimers();
+    vi.setSystemTime(new Date("2026-01-01T00:00:00.000Z"));
+  });
+
+  afterEach(async () => {
+    for (const socket of sockets.splice(0)) socket.terminate();
+    vi.clearAllTimers();
+    vi.useRealTimers();
+    await Promise.all(repositories.splice(0).map((repository) => repository.close()));
+  });
+
+  it("keeps a player queued when the rating gap is too large", async () => {
+    const hub = setup().hub;
+    const lowerRated = connect(hub, player("lower-rated", 1_000));
+    const higherRated = connect(hub, player("higher-rated", 2_000));
+
+    lowerRated.receive({ v: 1, type: "queue.join", mode: "queue" });
+    higherRated.receive({ v: 1, type: "queue.join", mode: "queue" });
+    await flushEvents();
+
+    expect(lowerRated.latest("queue.matched")).toBeUndefined();
+    expect(higherRated.latest("queue.matched")).toBeUndefined();
+    expect(hub.liveStats().queuedPlayers).toBe(2);
+
+    const nearby = connect(hub, player("nearby", 1_050));
+    nearby.receive({ v: 1, type: "queue.join", mode: "queue" });
+    await flushEvents();
+
+    expect(lowerRated.latest("queue.matched")).toEqual(
+      expect.objectContaining({ opponent: "nearby" })
+    );
+    expect(nearby.latest("queue.matched")).toEqual(
+      expect.objectContaining({ opponent: "lower-rated" })
+    );
+    expect(higherRated.latest("queue.matched")).toBeUndefined();
+    expect(hub.liveStats().queuedPlayers).toBe(1);
+  });
+
+  it("releases both matched reservations after a forfeit is finalized", async () => {
+    const { hub, repository } = setup();
+    vi.spyOn(repository, "finalizeMatch").mockResolvedValue({
+      matchId: "forfeit-match",
+      resultKey: "forfeit-result",
+      created: true
+    });
+    const left = connect(hub, player("left", 1_200));
+    const right = connect(hub, player("right", 1_200));
+    const roomId = await pair(left, right);
+
+    left.receive({ v: 1, type: "game.ready", roomId });
+    right.receive({ v: 1, type: "game.ready", roomId });
+    await flushEvents();
+    left.terminate();
+    await vi.advanceTimersByTimeAsync(15_000);
+    await flushEvents();
+
+    expect(hub.liveStats().activeRooms).toBe(0);
+    const reconnectedLeft = connect(hub, player("left", 1_200));
+    right.receive({ v: 1, type: "queue.join", mode: "queue" });
+    reconnectedLeft.receive({ v: 1, type: "queue.join", mode: "queue" });
+    await flushEvents();
+
+    expect(right.latest("queue.matched")).toEqual(
+      expect.objectContaining({ opponent: "left" })
+    );
+    expect(reconnectedLeft.latest("queue.matched")).toEqual(
+      expect.objectContaining({ opponent: "right" })
+    );
+  });
+
+  it("releases both matched reservations when an empty room is abandoned", async () => {
+    const { hub } = setup();
+    const left = connect(hub, player("abandoned-left", 1_200));
+    const right = connect(hub, player("abandoned-right", 1_200));
+    await pair(left, right);
+
+    left.terminate();
+    right.terminate();
+    await vi.advanceTimersByTimeAsync(15_000);
+    await flushEvents();
+
+    expect(hub.liveStats().activeRooms).toBe(0);
+    const recoveredLeft = connect(hub, player("abandoned-left", 1_200));
+    const recoveredRight = connect(hub, player("abandoned-right", 1_200));
+    await pair(recoveredLeft, recoveredRight);
+  });
+
+  it("rolls back the room and reservations when room creation fails", async () => {
+    const repository = createMemoryRepository();
+    repositories.push(repository);
+    let failOnce = true;
+    const hub = new GameHub(repository, {
+      roomCreated: () => {
+        if (!failOnce) return;
+        failOnce = false;
+        throw new Error("observer failed");
+      }
+    });
+    const left = connect(hub, player("retry-left", 1_200));
+    const right = connect(hub, player("retry-right", 1_200));
+
+    left.receive({ v: 1, type: "queue.join", mode: "queue" });
+    right.receive({ v: 1, type: "queue.join", mode: "queue" });
+    await flushEvents();
+    expect(hub.liveStats()).toEqual(expect.objectContaining({ activeRooms: 0, queuedPlayers: 0 }));
+
+    left.receive({ v: 1, type: "queue.join", mode: "queue" });
+    right.receive({ v: 1, type: "queue.join", mode: "queue" });
+    await flushEvents();
+
+    expect(left.latest("queue.matched")).toEqual(
+      expect.objectContaining({ opponent: "retry-right" })
+    );
+    expect(right.latest("queue.matched")).toEqual(
+      expect.objectContaining({ opponent: "retry-left" })
+    );
+    expect(hub.liveStats()).toEqual(expect.objectContaining({ activeRooms: 1, queuedPlayers: 0 }));
+  });
+
+  function setup() {
+    const repository = createMemoryRepository();
+    repositories.push(repository);
+    return { repository, hub: new GameHub(repository) };
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
+async function pair(left: FakeSocket, right: FakeSocket): Promise<string> {
+  left.receive({ v: 1, type: "queue.join", mode: "queue" });
+  right.receive({ v: 1, type: "queue.join", mode: "queue" });
+  await flushEvents();
+  const matched = right.latest("queue.matched");
+  if (matched?.type !== "queue.matched") throw new Error("expected a match");
+  return matched.roomId;
+}
+
+async function flushEvents(): Promise<void> {
+  await Promise.resolve();
+  await Promise.resolve();
+  await Promise.resolve();
+}
+
+function player(handle: string, rating: number): SessionUser {
+  return {
+    id: `${handle}-id`,
+    handle,
+    displayName: handle,
+    avatarKey: "default",
+    role: "user",
+    status: "active",
+    rating,
+    wins: 0,
+    losses: 0,
+    online: true,
+    isNpc: false,
+    email: null
+  };
+}
