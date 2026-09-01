# 레이팅 매칭 큐와 AI 폴백 예약

## `feat(game): 실시간 매칭 대기열 연결`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index fc6070c..44f9124 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -7,6 +7,7 @@ import {
   GAME_WIDTH,
   PADDLE_HEIGHT,
   encodeServerEvent,
+  parseClientEvent,
   type GameSnapshot,
   type PlayerSide,
   type ServerEvent,
@@ -29,6 +30,7 @@ type Room = {
 
 export class GameHub {
   private readonly clients = new Map<string, Client>();
+  private readonly queue: Client[] = [];
   private readonly rooms = new Map<string, Room>();
 
   constructor(private readonly repo: AppRepository) {}
@@ -36,11 +38,23 @@ export class GameHub {
   connect(socket: WebSocket, _request: IncomingMessage, user: SessionUser): void {
     const client: Client = { id: randomUUID(), socket, user, roomId: null };
     this.clients.set(client.id, client);
+    socket.on("message", (payload) => this.receive(client, payload.toString()));
     socket.on("close", () => this.disconnect(client));
     this.broadcastPresence();
   }
 
+  private receive(client: Client, payload: string): void {
+    try {
+      const event = parseClientEvent(payload);
+      if (event.type === "queue.join") this.joinQueue(client, event.mode);
+      if (event.type === "queue.leave") this.leaveQueue(client);
+    } catch (error) {
+      this.send(client, { type: "error", message: error instanceof Error ? error.message : "메시지를 처리하지 못했습니다." });
+    }
+  }
+
   private disconnect(client: Client): void {
+    this.leaveQueue(client);
     this.clients.delete(client.id);
     if (client.roomId) {
       const room = this.rooms.get(client.roomId);
@@ -54,6 +68,26 @@ export class GameHub {
     this.broadcastPresence();
   }
 
+  private joinQueue(client: Client, mode: "queue" | "ai"): void {
+    this.leaveQueue(client);
+    if (mode === "ai") {
+      this.createRoom(client, null, true);
+      return;
+    }
+    const opponent = this.queue.shift();
+    if (!opponent) {
+      this.queue.push(client);
+      this.broadcastPresence();
+      return;
+    }
+    this.createRoom(opponent, client, false);
+  }
+
+  private leaveQueue(client: Client): void {
+    const index = this.queue.findIndex((queued) => queued.id === client.id);
+    if (index >= 0) this.queue.splice(index, 1);
+  }
+
   private createRoom(left: Client, right: Client | null, ai: boolean): void {
     const roomId = randomUUID();
     const room: Room = {


## `fix(game): 닫힌 WebSocket 대기열 참가자 제거`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index 9b8b8ca..e3ad76d 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -91,6 +91,7 @@ export class GameHub {
 
   private joinQueue(client: Client, mode: "queue" | "ai"): void {
     this.leaveQueue(client);
+    this.pruneQueue();
     if (mode === "ai") {
       this.createRoom(client, null, true);
       return;
@@ -109,6 +110,14 @@ export class GameHub {
     if (index >= 0) this.queue.splice(index, 1);
   }
 
+  private pruneQueue(): void {
+    for (let index = this.queue.length - 1; index >= 0; index -= 1) {
+      if (this.queue[index].socket.readyState !== WebSocket.OPEN) {
+        this.queue.splice(index, 1);
+      }
+    }
+  }
+
   private createRoom(left: Client, right: Client | null, ai: boolean): void {
     const roomId = randomUUID();
     const room: Room = {


## `feat(game): 대기 플레이어 NPC fallback 구성`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index 1e98762..28125cb 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -30,6 +30,7 @@ type Client = {
 type QueueEntry = {
   client: Client;
   queuedAt: number;
+  npcFallbackTimer: NodeJS.Timeout | null;
 };
 
 type Room = {
@@ -41,6 +42,7 @@ type Room = {
   timer: NodeJS.Timeout | null;
   mode: MatchMode;
   tournamentMatchId: string | null;
+  npcUser: PublicUser | null;
 };
 
 const INITIAL_BALL_VELOCITY = { x: 10, y: 5 };
@@ -119,15 +121,33 @@ export class GameHub {
     }
     const opponentIndex = this.findClosestQueuedOpponent(client);
     if (opponentIndex < 0) {
-      this.queue.push({ client, queuedAt: Date.now() });
+      const entry: QueueEntry = { client, queuedAt: Date.now(), npcFallbackTimer: null };
+      entry.npcFallbackTimer = setTimeout(() => {
+        this.matchQueuedClientWithNpc(entry).catch((error) => {
+          this.send(client, { type: "error", message: error instanceof Error ? error.message : "AI 상대를 찾지 못했습니다." });
+        });
+      }, NPC_QUEUE_FALLBACK_MS);
+      this.queue.push(entry);
       this.broadcastPresence();
       return;
     }
     const [opponent] = this.queue.splice(opponentIndex, 1);
+    clearQueueTimer(opponent);
     this.recordWaitSample(opponent.queuedAt);
     this.createRoom(opponent.client, client, { ai: false, mode: "queue" });
   }
 
+  private async matchQueuedClientWithNpc(entry: QueueEntry): Promise<void> {
+    const index = this.queue.findIndex((queued) => queued.client.id === entry.client.id);
+    if (index < 0 || entry.client.socket.readyState !== WebSocket.OPEN || entry.client.roomId) return;
+    const npc = await this.findClosestNpc(entry.client);
+    if (!npc) return;
+    const [queued] = this.queue.splice(index, 1);
+    clearQueueTimer(queued);
+    this.recordWaitSample(queued.queuedAt);
+    this.createRoom(queued.client, null, { ai: true, mode: "queue", npc });
+  }
+
   private async findClosestNpc(client: Client): Promise<PublicUser | null> {
     const npcs = await this.repo.listNpcOpponents();
     let closest: PublicUser | null = null;
@@ -189,7 +209,10 @@ export class GameHub {
 
   private leaveQueue(client: Client): void {
     const index = this.queue.findIndex((queued) => queued.client.id === client.id);
-    if (index >= 0) this.queue.splice(index, 1);
+    if (index >= 0) {
+      const [entry] = this.queue.splice(index, 1);
+      clearQueueTimer(entry);
+    }
   }
 
   private leaveTournamentWaiters(client: Client): void {
@@ -203,7 +226,8 @@ export class GameHub {
   private pruneQueue(): void {
     for (let index = this.queue.length - 1; index >= 0; index -= 1) {
       if (this.queue[index].client.socket.readyState !== WebSocket.OPEN) {
-        this.queue.splice(index, 1);
+        const [entry] = this.queue.splice(index, 1);
+        clearQueueTimer(entry);
       }
     }
   }
@@ -241,7 +265,8 @@ export class GameHub {
 
   private createRoom(left: Client, right: Client | null, options: { ai: boolean; mode: MatchMode; tournamentMatchId?: string | null; npc?: PublicUser | null }): string {
     const roomId = randomUUID();
-    const rightPlayer = right?.user ?? options.npc ?? null;
+    const npcUser = options.npc ?? null;
+    const rightPlayer = right?.user ?? npcUser;
     const room: Room = {
       id: roomId,
       clients: { left, ...(right ? { right } : {}) },
@@ -250,6 +275,7 @@ export class GameHub {
       timer: null,
       mode: options.mode,
       tournamentMatchId: options.tournamentMatchId ?? null,
+      npcUser,
       snapshot: {
         roomId,
         phase: "waiting",
@@ -376,7 +402,7 @@ export class GameHub {
     room.timer = null;
     room.snapshot.phase = "finished";
     const leftUser = room.clients.left?.user ?? null;
-    const rightUser = room.clients.right?.user ?? room.snapshot.players.find((player) => player.side === "right" && player.ai) ?? null;
+    const rightUser = room.clients.right?.user ?? room.npcUser ?? null;
     const winner = winnerSide === "left" ? leftUser : rightUser;
     const loser = winnerSide === "left" ? rightUser : leftUser;
     const matchId = await this.repo.createMatch({
@@ -445,6 +471,13 @@ function sideFor(room: Room, client: Client): PlayerSide | null {
   return null;
 }
 
+function clearQueueTimer(entry: QueueEntry): void {
+  if (entry.npcFallbackTimer) {
+    clearTimeout(entry.npcFallbackTimer);
+    entry.npcFallbackTimer = null;
+  }
+}
+
 function clamp(value: number, min: number, max: number): number {
   return Math.max(min, Math.min(max, value));
 }


## `refactor(game): matchmaking player와 fallback 계약 정의`

diff --git a/apps/api/src/game/matchmaker.ts b/apps/api/src/game/matchmaker.ts
new file mode 100644
index 0000000..374b592
--- /dev/null
+++ b/apps/api/src/game/matchmaker.ts
@@ -0,0 +1,53 @@
+export type MatchmakingKind = "registered" | "guest";
+
+export interface MatchmakingPlayer {
+  userId: string;
+  rating: number;
+  kind: MatchmakingKind;
+}
+
+export interface MatchmakingPair {
+  left: MatchmakingPlayer;
+  right: MatchmakingPlayer;
+  ratingDifference: number;
+}
+
+export type MatchmakerJoinResult =
+  | { type: "queued"; queuedAtMs: number; aiFallbackAtMs: number }
+  | { type: "matched"; match: MatchmakingPair }
+  | { type: "duplicate"; status: MatchmakerPlayerStatus };
+
+export type AiFallbackResult =
+  | { type: "waiting"; remainingMs: number }
+  | { type: "ready"; player: MatchmakingPlayer; waitedMs: number }
+  | { type: "unavailable" };
+
+export type MatchmakerPlayerStatus = "queued" | "matched";
+
+export interface MatchmakerOptions {
+  clock: () => number;
+  maxRatingDifference: number;
+}
+
+interface QueueEntry {
+  player: MatchmakingPlayer;
+  queuedAtMs: number;
+}
+
+export const MATCHMAKER_AI_FALLBACK_MS = 6_000;
+
+function validatePlayer(player: MatchmakingPlayer): void {
+  if (player.userId.trim().length === 0) {
+    throw new TypeError("userId must not be empty");
+  }
+  if (!Number.isSafeInteger(player.rating)) {
+    throw new TypeError("rating must be a safe integer");
+  }
+  if (player.kind !== "registered" && player.kind !== "guest") {
+    throw new TypeError("kind must be registered or guest");
+  }
+}
+
+function copyPlayer(player: MatchmakingPlayer): MatchmakingPlayer {
+  return { userId: player.userId, rating: player.rating, kind: player.kind };
+}


## `refactor(game): rating 기반 closest-pair queue 구현`

diff --git a/apps/api/src/game/matchmaker.ts b/apps/api/src/game/matchmaker.ts
index 374b592..865689d 100644
--- a/apps/api/src/game/matchmaker.ts
+++ b/apps/api/src/game/matchmaker.ts
@@ -36,6 +36,86 @@ interface QueueEntry {
 
 export const MATCHMAKER_AI_FALLBACK_MS = 6_000;
 
+export class Matchmaker {
+  private readonly queue: QueueEntry[] = [];
+  private readonly playerStatuses = new Map<string, MatchmakerPlayerStatus>();
+  private readonly clock: () => number;
+  private readonly maxRatingDifference: number;
+
+  constructor(options: MatchmakerOptions) {
+    if (!Number.isFinite(options.maxRatingDifference) || options.maxRatingDifference < 0) {
+      throw new RangeError("maxRatingDifference must be a non-negative finite number");
+    }
+    this.clock = options.clock;
+    this.maxRatingDifference = options.maxRatingDifference;
+  }
+
+  get queuedCount(): number {
+    return this.queue.length;
+  }
+
+  enqueue(player: MatchmakingPlayer): MatchmakerJoinResult {
+    validatePlayer(player);
+    const existingStatus = this.playerStatuses.get(player.userId);
+    if (existingStatus) {
+      return { type: "duplicate", status: existingStatus };
+    }
+
+    const entrant = copyPlayer(player);
+    const opponentIndex = this.findClosestOpponent(entrant);
+    if (opponentIndex >= 0) {
+      const [opponent] = this.queue.splice(opponentIndex, 1);
+      this.playerStatuses.set(opponent.player.userId, "matched");
+      this.playerStatuses.set(entrant.userId, "matched");
+      return {
+        type: "matched",
+        match: {
+          left: copyPlayer(opponent.player),
+          right: copyPlayer(entrant),
+          ratingDifference: Math.abs(opponent.player.rating - entrant.rating)
+        }
+      };
+    }
+
+    const queuedAtMs = this.now();
+    this.queue.push({ player: entrant, queuedAtMs });
+    this.playerStatuses.set(entrant.userId, "queued");
+    return {
+      type: "queued",
+      queuedAtMs,
+      aiFallbackAtMs: queuedAtMs + MATCHMAKER_AI_FALLBACK_MS
+    };
+  }
+
+  queuedPlayers(): MatchmakingPlayer[] {
+    return this.queue.map((entry) => copyPlayer(entry.player));
+  }
+
+  private findClosestOpponent(entrant: MatchmakingPlayer): number {
+    let closestIndex = -1;
+    let closestDifference = Number.POSITIVE_INFINITY;
+
+    for (let index = 0; index < this.queue.length; index += 1) {
+      const candidate = this.queue[index].player;
+      if (candidate.kind !== entrant.kind) continue;
+      const difference = Math.abs(candidate.rating - entrant.rating);
+      if (difference > this.maxRatingDifference || difference >= closestDifference) continue;
+      closestIndex = index;
+      closestDifference = difference;
+    }
+
+    return closestIndex;
+  }
+
+  private now(): number {
+    const nowMs = this.clock();
+    if (!Number.isFinite(nowMs)) {
+      throw new RangeError("clock must return a finite timestamp");
+    }
+    return nowMs;
+  }
+}
+
 function validatePlayer(player: MatchmakingPlayer): void {
   if (player.userId.trim().length === 0) {
     throw new TypeError("userId must not be empty");


## `refactor(game): AI fallback과 reservation lifecycle 구현`

diff --git a/apps/api/src/game/matchmaker.ts b/apps/api/src/game/matchmaker.ts
index 865689d..027a53e 100644
--- a/apps/api/src/game/matchmaker.ts
+++ b/apps/api/src/game/matchmaker.ts
@@ -87,6 +87,51 @@ export class Matchmaker {
     };
   }
 
+  claimAiFallback(userId: string): AiFallbackResult {
+    if (this.playerStatuses.get(userId) !== "queued") {
+      return { type: "unavailable" };
+    }
+
+    const entryIndex = this.queue.findIndex((entry) => entry.player.userId === userId);
+    if (entryIndex < 0) {
+      this.playerStatuses.delete(userId);
+      return { type: "unavailable" };
+    }
+
+    const entry = this.queue[entryIndex];
+    const waitedMs = Math.max(0, this.now() - entry.queuedAtMs);
+    if (waitedMs < MATCHMAKER_AI_FALLBACK_MS) {
+      return { type: "waiting", remainingMs: MATCHMAKER_AI_FALLBACK_MS - waitedMs };
+    }
+
+    this.queue.splice(entryIndex, 1);
+    this.playerStatuses.set(userId, "matched");
+    return {
+      type: "ready",
+      player: copyPlayer(entry.player),
+      waitedMs
+    };
+  }
+
+  leaveQueue(userId: string): boolean {
+    if (this.playerStatuses.get(userId) !== "queued") return false;
+    const entryIndex = this.queue.findIndex((entry) => entry.player.userId === userId);
+    if (entryIndex >= 0) this.queue.splice(entryIndex, 1);
+    this.playerStatuses.delete(userId);
+    return entryIndex >= 0;
+  }
+
+  release(userId: string): boolean {
+    const status = this.playerStatuses.get(userId);
+    if (!status) return false;
+    if (status === "queued") {
+      const entryIndex = this.queue.findIndex((entry) => entry.player.userId === userId);
+      if (entryIndex >= 0) this.queue.splice(entryIndex, 1);
+    }
+    this.playerStatuses.delete(userId);
+    return true;
+  }
+
   queuedPlayers(): MatchmakingPlayer[] {
     return this.queue.map((entry) => copyPlayer(entry.player));
   }


## `test(game): matchmaking 규칙 검증`

diff --git a/apps/api/src/game/matchmaker.test.ts b/apps/api/src/game/matchmaker.test.ts
new file mode 100644
index 0000000..74232e6
--- /dev/null
+++ b/apps/api/src/game/matchmaker.test.ts
@@ -0,0 +1,128 @@
+import { describe, expect, it } from "vitest";
+import { Matchmaker, type MatchmakingPlayer } from "./matchmaker";
+
+describe("Matchmaker", () => {
+  it("matches the closest queued player within the configured rating difference", () => {
+    const clock = mutableClock(1_000);
+    const matchmaker = new Matchmaker({ clock: clock.now, maxRatingDifference: 150 });
+
+    expect(matchmaker.enqueue(player("farther", 1_000))).toMatchObject({ type: "queued" });
+    clock.advance(10);
+    expect(matchmaker.enqueue(player("closer", 1_180))).toMatchObject({ type: "queued" });
+    clock.advance(10);
+
+    expect(matchmaker.enqueue(player("entrant", 1_120))).toEqual({
+      type: "matched",
+      match: {
+        left: player("closer", 1_180),
+        right: player("entrant", 1_120),
+        ratingDifference: 60
+      }
+    });
+    expect(matchmaker.queuedPlayers()).toEqual([player("farther", 1_000)]);
+  });
+
+  it("keeps players queued when their rating difference is outside the limit", () => {
+    const matchmaker = new Matchmaker({ clock: () => 0, maxRatingDifference: 100 });
+
+    expect(matchmaker.enqueue(player("first", 1_000))).toMatchObject({ type: "queued" });
+    expect(matchmaker.enqueue(player("second", 1_101))).toMatchObject({ type: "queued" });
+    expect(matchmaker.queuedCount).toBe(2);
+  });
+
+  it("never matches a guest with a registered user", () => {
+    const matchmaker = new Matchmaker({ clock: () => 0, maxRatingDifference: 500 });
+
+    matchmaker.enqueue(player("registered", 1_200));
+    expect(matchmaker.enqueue(player("guest-one", 1_200, "guest"))).toMatchObject({ type: "queued" });
+
+    expect(matchmaker.enqueue(player("guest-two", 1_200, "guest"))).toEqual({
+      type: "matched",
+      match: {
+        left: player("guest-one", 1_200, "guest"),
+        right: player("guest-two", 1_200, "guest"),
+        ratingDifference: 0
+      }
+    });
+    expect(matchmaker.queuedPlayers()).toEqual([player("registered", 1_200)]);
+  });
+
+  it("makes a queued player eligible for AI fallback after exactly six seconds", () => {
+    const clock = mutableClock(10_000);
+    const matchmaker = new Matchmaker({ clock: clock.now, maxRatingDifference: 100 });
+    expect(matchmaker.enqueue(player("waiting", 1_200, "guest"))).toEqual({
+      type: "queued",
+      queuedAtMs: 10_000,
+      aiFallbackAtMs: 16_000
+    });
+
+    clock.advance(5_999);
+    expect(matchmaker.claimAiFallback("waiting")).toEqual({
+      type: "waiting",
+      remainingMs: 1
+    });
+
+    clock.advance(1);
+    expect(matchmaker.claimAiFallback("waiting")).toEqual({
+      type: "ready",
+      player: player("waiting", 1_200, "guest"),
+      waitedMs: 6_000
+    });
+    expect(matchmaker.queuedCount).toBe(0);
+    expect(matchmaker.claimAiFallback("waiting")).toEqual({ type: "unavailable" });
+  });
+
+  it("prevents the same user from joining twice until their slot is released", () => {
+    const matchmaker = new Matchmaker({ clock: () => 0, maxRatingDifference: 100 });
+
+    matchmaker.enqueue(player("same-user", 1_200));
+    expect(matchmaker.enqueue(player("same-user", 1_250, "guest"))).toEqual({
+      type: "duplicate",
+      status: "queued"
+    });
+    expect(matchmaker.queuedCount).toBe(1);
+
+    matchmaker.enqueue(player("opponent", 1_200));
+    expect(matchmaker.enqueue(player("same-user", 1_200))).toEqual({
+      type: "duplicate",
+      status: "matched"
+    });
+
+    expect(matchmaker.release("same-user")).toBe(true);
+    expect(matchmaker.enqueue(player("same-user", 1_200))).toMatchObject({ type: "queued" });
+  });
+
+  it("removes only queued players and leaves matched reservations intact", () => {
+    const matchmaker = new Matchmaker({ clock: () => 0, maxRatingDifference: 100 });
+
+    matchmaker.enqueue(player("queued", 1_000));
+    expect(matchmaker.leaveQueue("queued")).toBe(true);
+    expect(matchmaker.leaveQueue("queued")).toBe(false);
+
+    matchmaker.enqueue(player("left", 1_200));
+    matchmaker.enqueue(player("right", 1_200));
+    expect(matchmaker.leaveQueue("left")).toBe(false);
+    expect(matchmaker.enqueue(player("left", 1_200))).toEqual({
+      type: "duplicate",
+      status: "matched"
+    });
+  });
+});
+
+function player(
+  userId: string,
+  rating: number,
+  kind: MatchmakingPlayer["kind"] = "registered"
+): MatchmakingPlayer {
+  return { userId, rating, kind };
+}
+
+function mutableClock(initialMs: number): { now: () => number; advance: (milliseconds: number) => void } {
+  let nowMs = initialMs;
+  return {
+    now: () => nowMs,
+    advance: (milliseconds) => {
+      nowMs += milliseconds;
+    }
+  };
+}


## `refactor(game): Matchmaker queue reservation을 GameHub에 연결`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index 8aa3bab..65b0a7f 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -18,6 +18,7 @@ import { DEFAULT_TIMESTEP_MS } from "./game/fixedStepScheduler.js";
 import { ConnectionHeartbeat } from "./game/heartbeat.js";
 import { InputGate } from "./game/inputGate.js";
 import { HARD_BUFFERED_AMOUNT_BYTES, LatestSnapshotBuffer } from "./game/latestSnapshotBuffer.js";
+import { Matchmaker, type MatchmakingPlayer } from "./game/matchmaker.js";
 import { PongAi } from "./game/pongAi.js";
 import { PongSimulation, type PongSimulationState } from "./game/pongSimulation.js";
 import { RoomSession } from "./game/roomSession.js";
@@ -45,6 +46,7 @@ type VersionlessServerEvent = ServerEvent extends infer Event
 type QueueEntry = {
   client: Client;
   queuedAt: number;
+  queuedAtMs: number;
   npcFallbackTimer: NodeJS.Timeout | null;
 };
 
@@ -75,6 +77,7 @@ type Room = {
 };
 
 const NPC_QUEUE_FALLBACK_MS = 6000;
+const MAX_MATCHMAKING_RATING_DIFFERENCE = 200;
 const SIMULATION_TIMESTEP_MS = DEFAULT_TIMESTEP_MS;
 const CONNECTION_REPLACED_CLOSE_CODE = 4001;
 const CONNECTION_REPLACED_REASON = "connection replaced";
@@ -112,6 +115,11 @@ export class GameHub {
   private readonly clients = new Map<string, Client>();
   private readonly clientsByUser = new Map<string, Client>();
   private readonly queue: QueueEntry[] = [];
+  private readonly queueEntries = new Map<string, QueueEntry>();
+  private readonly matchmaker = new Matchmaker({
+    clock: () => Date.now(),
+    maxRatingDifference: MAX_MATCHMAKING_RATING_DIFFERENCE
+  });
   private readonly rooms = new Map<string, Room>();
   private readonly tournamentWaiters = new Map<string, Client[]>();
   private readonly waitSamples: number[] = [];
@@ -384,32 +392,50 @@ export class GameHub {
       this.send(client, { type: "error", code: "forbidden", message: "이미 진행 중인 경기가 있습니다." });
       return;
     }
-    this.leaveQueue(client);
     this.pruneQueue();
     if (mode === "ai") {
+      this.leaveQueue(client);
       this.createRoom(client, null, { ai: true, mode: "ai" });
       return;
     }
-    const opponentIndex = this.findClosestQueuedOpponent(client);
-    if (opponentIndex < 0) {
-      const entry: QueueEntry = { client, queuedAt: Date.now(), npcFallbackTimer: null };
-      entry.npcFallbackTimer = setTimeout(() => {
-        this.matchQueuedClientWithNpc(entry).catch((error) => {
-          this.send(client, {
-            type: "error",
-            code: "internal_error",
-            message: error instanceof Error ? error.message : "AI 상대를 찾지 못했습니다."
-          });
-        });
-      }, NPC_QUEUE_FALLBACK_MS);
-      this.queue.push(entry);
+
+    const join = this.matchmaker.enqueue(matchmakingPlayer(client));
+    if (join.type === "duplicate") {
+      this.send(client, {
+        type: "error",
+        code: "forbidden",
+        message: join.status === "queued" ? "이미 대기열에 참가했습니다." : "이미 경기가 배정되었습니다."
+      });
+      return;
+    }
+    if (join.type === "queued") {
+      const entry: QueueEntry = {
+        client,
+        queuedAt: join.queuedAtMs,
+        queuedAtMs: join.queuedAtMs,
+        npcFallbackTimer: null
+      };
+      this.queueEntries.set(client.user.id, entry);
       this.broadcastPresence();
       return;
     }
-    const [opponent] = this.queue.splice(opponentIndex, 1);
+
+    const opponent = this.queueEntries.get(join.match.left.userId);
+    if (!opponent) {
+      this.matchmaker.release(join.match.left.userId);
+      this.matchmaker.release(join.match.right.userId);
+      throw new Error("대기 중인 상대 연결을 찾지 못했습니다.");
+    }
+    this.queueEntries.delete(opponent.client.user.id);
     clearQueueTimer(opponent);
-    this.recordWaitSample(opponent.queuedAt);
-    this.createRoom(opponent.client, client, { ai: false, mode: "queue" });
+    this.recordWaitSample(opponent.queuedAtMs);
+    try {
+      this.createRoom(opponent.client, client, { ai: false, mode: "queue" });
+    } catch (error) {
+      this.matchmaker.release(opponent.client.user.id);
+      this.matchmaker.release(client.user.id);
+      throw error;
+    }
   }
 
   private async matchQueuedClientWithNpc(entry: QueueEntry): Promise<void> {
@@ -946,6 +972,14 @@ function isGuest(user: ConnectedUser): user is GuestSessionUser {
   return "sessionKind" in user && user.sessionKind === "guest";
 }
 
+function matchmakingPlayer(client: Client): MatchmakingPlayer {
+  return {
+    userId: client.user.id,
+    rating: client.user.rating,
+    kind: isGuest(client.user) ? "guest" : "registered"
+  };
+}
+
 function roomUserIds(room: Room): string[] {
   return Object.values(room.clients)
     .filter((client): client is Client => Boolean(client))


