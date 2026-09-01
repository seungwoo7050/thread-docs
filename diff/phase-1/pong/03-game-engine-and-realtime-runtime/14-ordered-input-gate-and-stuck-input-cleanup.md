# 순서 보장 입력 게이트와 고착 입력 정리

## `feat(protocol): versioned WebSocket event codec 연결`

diff --git a/packages/shared/src/ws.ts b/packages/shared/src/ws.ts
index 0862e83..43efae3 100644
--- a/packages/shared/src/ws.ts
+++ b/packages/shared/src/ws.ts
@@ -1,65 +1,82 @@
 import { z } from "zod";
-import type { ChatMessage } from "./http";
-import type { GameFinished, GameSnapshot, PlayerSide } from "./game";
+import { chatMessageSchema } from "./http";
+import { gameFinishedSchema, gameSnapshotSchema, playerSideSchema } from "./game";
+
+const version = { v: z.literal(1) } as const;
+const roomIdSchema = z.string().min(1);
 
 export const clientEventSchema = z.discriminatedUnion("type", [
   z.object({
+    ...version,
     type: z.literal("queue.join"),
     mode: z.enum(["queue", "ai"]).default("queue")
-  }),
-  z.object({ type: z.literal("queue.leave") }),
-  z.object({ type: z.literal("tournament.join"), matchId: z.string() }),
-  z.object({ type: z.literal("game.ready"), roomId: z.string() }),
-  z.object({ type: z.literal("game.pause"), roomId: z.string() }),
-  z.object({ type: z.literal("game.resume"), roomId: z.string() }),
+  }).strict(),
+  z.object({ ...version, type: z.literal("queue.leave") }).strict(),
+  z.object({ ...version, type: z.literal("tournament.join"), matchId: z.string().min(1) }).strict(),
+  z.object({ ...version, type: z.literal("game.ready"), roomId: roomIdSchema }).strict(),
+  z.object({ ...version, type: z.literal("game.pause"), roomId: roomIdSchema }).strict(),
+  z.object({ ...version, type: z.literal("game.resume"), roomId: roomIdSchema }).strict(),
   z.object({
+    ...version,
     type: z.literal("game.input"),
-    roomId: z.string(),
+    roomId: roomIdSchema,
+    inputSeq: z.number().int().nonnegative().max(Number.MAX_SAFE_INTEGER),
     direction: z.union([z.literal(-1), z.literal(0), z.literal(1)])
-  }),
+  }).strict(),
   z.object({
+    ...version,
     type: z.literal("chat.send"),
     scope: z.enum(["lobby", "match"]),
     roomId: z.string().nullable().optional(),
     body: z.string().trim().min(1).max(240)
-  })
+  }).strict()
 ]);
 
-export type ClientEvent = z.infer<typeof clientEventSchema>;
+export const wsErrorCodeSchema = z.enum([
+  "invalid_event",
+  "rate_limited",
+  "forbidden",
+  "not_found",
+  "internal_error"
+]);
 
-export type ServerEvent =
-  | {
-      type: "queue.matched";
-      roomId: string;
-      side: PlayerSide;
-      opponent: string;
-    }
-  | {
-      type: "game.snapshot";
-      snapshot: GameSnapshot;
-    }
-  | {
-      type: "game.finished";
-      result: GameFinished;
-    }
-  | {
-      type: "chat.message";
-      message: ChatMessage;
-    }
-  | {
-      type: "presence.changed";
-      online: number;
-      playing: number;
-    }
-  | {
-      type: "error";
-      message: string;
-    };
+export const serverEventSchema = z.discriminatedUnion("type", [
+  z.object({
+    ...version,
+    type: z.literal("queue.matched"),
+    roomId: roomIdSchema,
+    side: playerSideSchema,
+    opponent: z.string().min(1)
+  }).strict(),
+  z.object({ ...version, type: z.literal("game.snapshot"), snapshot: gameSnapshotSchema }).strict(),
+  z.object({ ...version, type: z.literal("game.finished"), result: gameFinishedSchema }).strict(),
+  z.object({ ...version, type: z.literal("chat.message"), message: chatMessageSchema }).strict(),
+  z.object({
+    ...version,
+    type: z.literal("presence.changed"),
+    online: z.number().int().nonnegative(),
+    playing: z.number().int().nonnegative()
+  }).strict(),
+  z.object({
+    ...version,
+    type: z.literal("error"),
+    code: wsErrorCodeSchema,
+    message: z.string().min(1)
+  }).strict()
+]);
+
+export type ClientEvent = z.infer<typeof clientEventSchema>;
+export type WsErrorCode = z.infer<typeof wsErrorCodeSchema>;
+export type ServerEvent = z.infer<typeof serverEventSchema>;
 
 export function parseClientEvent(payload: string): ClientEvent {
   return clientEventSchema.parse(JSON.parse(payload));
 }
 
+export function parseServerEvent(payload: string): ServerEvent {
+  return serverEventSchema.parse(JSON.parse(payload));
+}
+
 export function encodeServerEvent(event: ServerEvent): string {
-  return JSON.stringify(event);
+  return JSON.stringify(serverEventSchema.parse(event));
 }


## `feat(game): room별 input sequence 중복을 차단`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index 57ba45a..fec9aec 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -21,6 +21,7 @@ type Client = {
   socket: WebSocket;
   user: SessionUser;
   roomId: string | null;
+  lastInputSequenceByRoom: Map<string, number>;
 };
 
 type VersionlessServerEvent = ServerEvent extends infer Event
@@ -62,7 +63,13 @@ export class GameHub {
   constructor(private readonly repo: AppRepository) {}
 
   connect(socket: WebSocket, request: IncomingMessage, user: SessionUser, pendingPayloads: string[] = []): void {
-    const client: Client = { id: randomUUID(), socket, user, roomId: null };
+    const client: Client = {
+      id: randomUUID(),
+      socket,
+      user,
+      roomId: null,
+      lastInputSequenceByRoom: new Map()
+    };
     this.clients.set(client.id, client);
     socket.on("message", (payload) => this.receive(client, payload.toString()));
     socket.on("close", () => this.disconnect(client));
@@ -81,7 +88,7 @@ export class GameHub {
       if (event.type === "game.ready") this.markReady(client, event.roomId);
       if (event.type === "game.pause") this.pauseRoom(client, event.roomId);
       if (event.type === "game.resume") this.resumeRoom(client, event.roomId);
-      if (event.type === "game.input") this.applyInput(client, event.roomId, event.direction);
+      if (event.type === "game.input") this.applyInput(client, event.roomId, event.inputSeq, event.direction);
       if (event.type === "chat.send") {
         const message = await this.repo.createChatMessage({
           scope: event.scope,
@@ -338,11 +345,15 @@ export class GameHub {
     this.broadcastSnapshot(room);
   }
 
-  private applyInput(client: Client, roomId: string, direction: -1 | 0 | 1): void {
+  private applyInput(client: Client, roomId: string, inputSeq: number, direction: -1 | 0 | 1): void {
     const room = this.rooms.get(roomId);
     if (!room || room.snapshot.state.phase !== "playing") return;
     const side = sideFor(room, client);
-    if (side) room.snapshot.paddles[side].dy = direction;
+    if (!side) return;
+    const previousSequence = client.lastInputSequenceByRoom.get(roomId) ?? -1;
+    if (inputSeq <= previousSequence) return;
+    client.lastInputSequenceByRoom.set(roomId, inputSeq);
+    room.snapshot.state.paddles[side].dy = direction;
   }
 
   private pauseRoom(client: Client, roomId: string): void {


## `refactor(web): game input 직렬화 경계 분리`

diff --git a/apps/web/src/game/gameInput.ts b/apps/web/src/game/gameInput.ts
new file mode 100644
index 0000000..b165301
--- /dev/null
+++ b/apps/web/src/game/gameInput.ts
@@ -0,0 +1,18 @@
+export type EditableTarget = {
+  tagName?: string;
+  isContentEditable?: boolean;
+};
+
+export function directionForKey(key: string): -1 | 0 | 1 | null {
+  if (key === "ArrowUp" || key === "w" || key === "W") return -1;
+  if (key === "ArrowDown" || key === "s" || key === "S") return 1;
+  return null;
+}
+
+export function isEditableTarget(target: EditableTarget | null): boolean {
+  if (!target) return false;
+  return Boolean(
+    target.isContentEditable
+    || (target.tagName && ["INPUT", "TEXTAREA", "SELECT"].includes(target.tagName.toUpperCase()))
+  );
+}


## `feat(game): 입력 순서와 rate limit 보호`

diff --git a/apps/api/src/game/inputGate.ts b/apps/api/src/game/inputGate.ts
new file mode 100644
index 0000000..96022e5
--- /dev/null
+++ b/apps/api/src/game/inputGate.ts
@@ -0,0 +1,80 @@
+export const DEFAULT_INPUT_RATE_PER_SECOND = 30;
+export const DEFAULT_INPUT_BURST_CAPACITY = 8;
+
+export type InputGateDecision = "accepted" | "stale" | "rate_limited";
+
+export type InputGateCommand = {
+  userId: string;
+  roomId: string;
+  inputSeq: number;
+  nowMs: number;
+};
+
+type TokenBucket = {
+  tokens: number;
+  lastRefillMs: number;
+};
+
+export class InputGate {
+  private readonly ratePerMillisecond: number;
+  private readonly burstCapacity: number;
+  private readonly buckets = new Map<string, TokenBucket>();
+  private readonly lastSequences = new Map<string, number>();
+
+  constructor(options: { ratePerSecond?: number; burstCapacity?: number } = {}) {
+    const ratePerSecond = options.ratePerSecond ?? DEFAULT_INPUT_RATE_PER_SECOND;
+    const burstCapacity = options.burstCapacity ?? DEFAULT_INPUT_BURST_CAPACITY;
+    if (!Number.isFinite(ratePerSecond) || ratePerSecond <= 0) {
+      throw new RangeError("ratePerSecond must be positive");
+    }
+    if (!Number.isInteger(burstCapacity) || burstCapacity <= 0) {
+      throw new RangeError("burstCapacity must be a positive integer");
+    }
+    this.ratePerMillisecond = ratePerSecond / 1_000;
+    this.burstCapacity = burstCapacity;
+  }
+
+  check(command: InputGateCommand): InputGateDecision {
+    const sequenceKey = `${command.userId}\u0000${command.roomId}`;
+    const previousSequence = this.lastSequences.get(sequenceKey);
+    if (previousSequence !== undefined && command.inputSeq <= previousSequence) {
+      return "stale";
+    }
+
+    const bucket = this.refill(command.userId, command.nowMs);
+    if (bucket.tokens < 1) {
+      return "rate_limited";
+    }
+
+    bucket.tokens -= 1;
+    this.lastSequences.set(sequenceKey, command.inputSeq);
+    return "accepted";
+  }
+
+  releaseUser(userId: string): void {
+    this.buckets.delete(userId);
+    const prefix = `${userId}\u0000`;
+    for (const key of this.lastSequences.keys()) {
+      if (key.startsWith(prefix)) this.lastSequences.delete(key);
+    }
+  }
+
+  private refill(userId: string, nowMs: number): TokenBucket {
+    const existing = this.buckets.get(userId);
+    if (!existing) {
+      const bucket = { tokens: this.burstCapacity, lastRefillMs: nowMs };
+      this.buckets.set(userId, bucket);
+      return bucket;
+    }
+
+    if (nowMs > existing.lastRefillMs) {
+      const elapsedMs = nowMs - existing.lastRefillMs;
+      existing.tokens = Math.min(
+        this.burstCapacity,
+        existing.tokens + (elapsedMs * this.ratePerMillisecond)
+      );
+      existing.lastRefillMs = nowMs;
+    }
+    return existing;
+  }
+}


## `test(game): input gate 제한 검증`

diff --git a/apps/api/src/game/inputGate.test.ts b/apps/api/src/game/inputGate.test.ts
new file mode 100644
index 0000000..8f3940e
--- /dev/null
+++ b/apps/api/src/game/inputGate.test.ts
@@ -0,0 +1,52 @@
+import { describe, expect, it } from "vitest";
+import { InputGate } from "./inputGate";
+
+describe("InputGate", () => {
+  it("allows a short burst and then sustains thirty inputs per second", () => {
+    const gate = new InputGate({ ratePerSecond: 30, burstCapacity: 8 });
+    const input = (inputSeq: number, nowMs: number) => gate.check({
+      userId: "user-1",
+      roomId: "room-1",
+      inputSeq,
+      nowMs
+    });
+
+    for (let inputSeq = 0; inputSeq < 8; inputSeq += 1) {
+      expect(input(inputSeq, 0)).toBe("accepted");
+    }
+    expect(input(8, 0)).toBe("rate_limited");
+
+    for (let tenth = 1; tenth <= 10; tenth += 1) {
+      const nowMs = tenth * 100;
+      const firstSequence = 8 + ((tenth - 1) * 3);
+      expect(input(firstSequence, nowMs)).toBe("accepted");
+      expect(input(firstSequence + 1, nowMs)).toBe("accepted");
+      expect(input(firstSequence + 2, nowMs)).toBe("accepted");
+      expect(input(firstSequence + 3, nowMs)).toBe("rate_limited");
+    }
+  });
+
+  it("drops duplicate and older sequences without spending rate-limit capacity", () => {
+    const gate = new InputGate({ ratePerSecond: 30, burstCapacity: 2 });
+    const check = (inputSeq: number) => gate.check({
+      userId: "user-1",
+      roomId: "room-1",
+      inputSeq,
+      nowMs: 0
+    });
+
+    expect(check(10)).toBe("accepted");
+    expect(check(10)).toBe("stale");
+    expect(check(9)).toBe("stale");
+    expect(check(11)).toBe("accepted");
+    expect(check(12)).toBe("rate_limited");
+  });
+
+  it("shares the rate limit across a user's rooms while isolating other users", () => {
+    const gate = new InputGate({ ratePerSecond: 30, burstCapacity: 1 });
+
+    expect(gate.check({ userId: "same-user", roomId: "room-1", inputSeq: 0, nowMs: 0 })).toBe("accepted");
+    expect(gate.check({ userId: "same-user", roomId: "room-2", inputSeq: 0, nowMs: 0 })).toBe("rate_limited");
+    expect(gate.check({ userId: "other-user", roomId: "room-2", inputSeq: 0, nowMs: 0 })).toBe("accepted");
+  });
+});


## `feat(game): heartbeat와 input gate를 GameHub에 연결`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index 096adb7..f52e1aa 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -25,7 +25,8 @@ type Client = {
   socket: WebSocket;
   user: SessionUser;
   roomId: string | null;
-  lastInputSequenceByRoom: Map<string, number>;
+  heartbeat: ConnectionHeartbeat;
+  snapshots: LatestSnapshotBuffer;
 };
 
 type VersionlessServerEvent = ServerEvent extends infer Event
@@ -64,20 +65,30 @@ export class GameHub {
   private readonly rooms = new Map<string, Room>();
   private readonly tournamentWaiters = new Map<string, Client[]>();
   private readonly waitSamples: number[] = [];
+  private readonly inputGate = new InputGate();
 
   constructor(private readonly repo: AppRepository) {}
 
-  connect(socket: WebSocket, request: IncomingMessage, user: SessionUser, pendingPayloads: string[] = []): void {
+  connect(socket: WebSocket, _request: IncomingMessage, user: SessionUser, pendingPayloads: string[] = []): void {
+    const heartbeat = new ConnectionHeartbeat({
+      ping: () => {
+        if (socket.readyState === WebSocket.OPEN) socket.ping();
+      },
+      terminate: () => socket.terminate()
+    });
     const client: Client = {
       id: randomUUID(),
       socket,
       user,
       roomId: null,
-      lastInputSequenceByRoom: new Map()
+      heartbeat,
+      snapshots: new LatestSnapshotBuffer(socket)
     };
     this.clients.set(client.id, client);
     socket.on("message", (payload) => this.receive(client, payload.toString()));
+    socket.on("pong", () => heartbeat.acknowledge());
     socket.on("close", () => this.disconnect(client));
+    heartbeat.start();
     this.broadcastPresence();
     for (const payload of pendingPayloads) {
       this.receive(client, payload).catch(() => undefined);
@@ -117,9 +128,15 @@ export class GameHub {
   }
 
   private disconnect(client: Client): void {
+    if (!this.clients.has(client.id)) return;
+    client.heartbeat.stop();
+    client.snapshots.close();
     this.leaveQueue(client);
     this.leaveTournamentWaiters(client);
     this.clients.delete(client.id);
+    if (![...this.clients.values()].some((candidate) => candidate.user.id === client.user.id)) {
+      this.inputGate.releaseUser(client.user.id);
+    }
     if (client.roomId) {
       const room = this.rooms.get(client.roomId);
       if (room) {
@@ -364,9 +381,21 @@ export class GameHub {
     if (!room || room.snapshot.state.phase !== "playing") return;
     const side = sideFor(room, client);
     if (!side) return;
-    const previousSequence = client.lastInputSequenceByRoom.get(roomId) ?? -1;
-    if (inputSeq <= previousSequence) return;
-    client.lastInputSequenceByRoom.set(roomId, inputSeq);
+    const decision = this.inputGate.check({
+      userId: client.user.id,
+      roomId,
+      inputSeq,
+      nowMs: performance.now()
+    });
+    if (decision === "stale") return;
+    if (decision === "rate_limited") {
+      this.send(client, {
+        type: "error",
+        code: "rate_limited",
+        message: "게임 입력 전송 한도를 초과했습니다."
+      });
+      return;
+    }
     room.snapshot.state.paddles[side].dy = direction;
   }
 


## `fix(game): 일시정지 시 paddle 입력 상태 초기화`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index fa74507..832735c 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -818,6 +818,10 @@ export class GameHub {
     this.roomScheduler.unregister(room.id);
     const sessionState = room.session.pause();
     if (sessionState !== "paused") return;
+    for (const side of ["left", "right"] as const) {
+      room.snapshot.state.paddles[side].dy = 0;
+      room.simulation.paddles[side].direction = 0;
+    }
     room.snapshot.state.phase = sessionState;
     this.broadcastSnapshot(room);
   }


## `test(game): pause 전 입력이 재개 뒤 남지 않음 검증`

diff --git a/apps/api/src/gameHub.pause.test.ts b/apps/api/src/gameHub.pause.test.ts
new file mode 100644
index 0000000..c135057
--- /dev/null
+++ b/apps/api/src/gameHub.pause.test.ts
@@ -0,0 +1,128 @@
+import { EventEmitter } from "node:events";
+import type { IncomingMessage } from "node:http";
+import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
+import { WebSocket } from "ws";
+import { createMemoryRepository } from "@pong-pong/db";
+import { parseServerEvent, type ServerEvent, type SessionUser } from "@pong-pong/shared";
+import { GameHub } from "./gameHub.js";
+
+describe("GameHub pause input boundary", () => {
+  const hubs: GameHub[] = [];
+  const repositories: Array<ReturnType<typeof createMemoryRepository>> = [];
+
+  beforeEach(() => {
+    vi.useFakeTimers();
+  });
+
+  afterEach(async () => {
+    for (const hub of hubs.splice(0)) hub.close();
+    vi.clearAllTimers();
+    vi.useRealTimers();
+    await Promise.all(repositories.splice(0).map((repository) => repository.close()));
+  });
+
+  it("does not carry a pre-pause paddle direction into resume", async () => {
+    const repository = createMemoryRepository();
+    repositories.push(repository);
+    const hub = new GameHub(repository);
+    hubs.push(hub);
+    const socket = new FakeSocket();
+    hub.connect(socket as unknown as WebSocket, {} as IncomingMessage, player());
+
+    socket.receive({ v: 1, type: "queue.join", mode: "ai" });
+    await flushEvents();
+    const matched = socket.latest("queue.matched");
+    if (matched?.type !== "queue.matched") throw new Error("expected a match");
+
+    socket.receive({ v: 1, type: "game.ready", roomId: matched.roomId });
+    socket.receive({
+      v: 1,
+      type: "game.input",
+      roomId: matched.roomId,
+      inputSeq: 0,
+      direction: 1
+    });
+    socket.receive({ v: 1, type: "game.pause", roomId: matched.roomId });
+    await flushEvents();
+
+    expect(latestSnapshot(socket).state.phase).toBe("paused");
+    expect(latestSnapshot(socket).state.paddles.left.dy).toBe(0);
+
+    socket.receive({
+      v: 1,
+      type: "game.input",
+      roomId: matched.roomId,
+      inputSeq: 1,
+      direction: 0
+    });
+    socket.receive({ v: 1, type: "game.resume", roomId: matched.roomId });
+    await vi.advanceTimersByTimeAsync(100);
+    await flushEvents();
+
+    expect(latestSnapshot(socket).state.phase).toBe("playing");
+    expect(latestSnapshot(socket).state.paddles.left.dy).toBe(0);
+  });
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
+function latestSnapshot(socket: FakeSocket) {
+  const event = socket.latest("game.snapshot");
+  if (event?.type !== "game.snapshot") throw new Error("missing game snapshot");
+  return event.snapshot;
+}
+
+async function flushEvents(): Promise<void> {
+  await Promise.resolve();
+  await Promise.resolve();
+  await Promise.resolve();
+}
+
+function player(): SessionUser {
+  return {
+    id: "11111111-1111-4111-8111-111111111111",
+    handle: "pause-player",
+    displayName: "Pause Player",
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
