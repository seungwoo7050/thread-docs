# 버전형 WebSocket 이벤트 프로토콜

## `feat(shared): WebSocket 이벤트 메시지 검증`

diff --git a/packages/shared/src/index.ts b/packages/shared/src/index.ts
index 7bb2940..45a906e 100644
--- a/packages/shared/src/index.ts
+++ b/packages/shared/src/index.ts
@@ -1,2 +1,3 @@
 export * from "./http";
 export * from "./game";
+export * from "./ws";
diff --git a/packages/shared/src/ws.ts b/packages/shared/src/ws.ts
new file mode 100644
index 0000000..fc49e3d
--- /dev/null
+++ b/packages/shared/src/ws.ts
@@ -0,0 +1,62 @@
+import { z } from "zod";
+import type { ChatMessage } from "./http";
+import type { GameFinished, GameSnapshot, PlayerSide } from "./game";
+
+export const clientEventSchema = z.discriminatedUnion("type", [
+  z.object({
+    type: z.literal("queue.join"),
+    mode: z.enum(["queue", "ai"]).default("queue")
+  }),
+  z.object({ type: z.literal("queue.leave") }),
+  z.object({ type: z.literal("game.ready"), roomId: z.string() }),
+  z.object({
+    type: z.literal("game.input"),
+    roomId: z.string(),
+    direction: z.union([z.literal(-1), z.literal(0), z.literal(1)])
+  }),
+  z.object({
+    type: z.literal("chat.send"),
+    scope: z.enum(["lobby", "match"]),
+    roomId: z.string().nullable().optional(),
+    body: z.string().trim().min(1).max(240)
+  })
+]);
+
+export type ClientEvent = z.infer<typeof clientEventSchema>;
+
+export type ServerEvent =
+  | {
+      type: "queue.matched";
+      roomId: string;
+      side: PlayerSide;
+      opponent: string;
+    }
+  | {
+      type: "game.snapshot";
+      snapshot: GameSnapshot;
+    }
+  | {
+      type: "game.finished";
+      result: GameFinished;
+    }
+  | {
+      type: "chat.message";
+      message: ChatMessage;
+    }
+  | {
+      type: "presence.changed";
+      online: number;
+      playing: number;
+    }
+  | {
+      type: "error";
+      message: string;
+    };
+
+export function parseClientEvent(payload: string): ClientEvent {
+  return clientEventSchema.parse(JSON.parse(payload));
+}
+
+export function encodeServerEvent(event: ServerEvent): string {
+  return JSON.stringify(event);
+}


## `test(shared): WebSocket 프로토콜 검증`

diff --git a/Makefile b/Makefile
index 878f0a9..859ca3b 100644
--- a/Makefile
+++ b/Makefile
@@ -1,4 +1,4 @@
-.PHONY: install typecheck build
+.PHONY: install typecheck build test
 
 install:
 	pnpm install
@@ -8,3 +8,6 @@ typecheck:
 
 build:
 	pnpm -r build
+
+test:
+	pnpm -r test
diff --git a/package.json b/package.json
index 8e619ab..031dc85 100644
--- a/package.json
+++ b/package.json
@@ -5,10 +5,12 @@
   "packageManager": "pnpm@10.32.1",
   "scripts": {
     "build": "pnpm -r build",
-    "typecheck": "pnpm -r typecheck"
+    "typecheck": "pnpm -r typecheck",
+    "test": "pnpm -r test"
   },
   "devDependencies": {
     "@types/node": "^22.15.3",
-    "typescript": "^5.8.3"
+    "typescript": "^5.8.3",
+    "vitest": "^3.1.2"
   }
 }
diff --git a/packages/shared/package.json b/packages/shared/package.json
index 610f892..cd78861 100644
--- a/packages/shared/package.json
+++ b/packages/shared/package.json
@@ -8,7 +8,8 @@
   },
   "scripts": {
     "build": "tsc --noEmit",
-    "typecheck": "tsc --noEmit"
+    "typecheck": "tsc --noEmit",
+    "test": "vitest run"
   },
   "dependencies": {
     "zod": "^3.24.4"
diff --git a/packages/shared/src/ws.test.ts b/packages/shared/src/ws.test.ts
new file mode 100644
index 0000000..48eddbb
--- /dev/null
+++ b/packages/shared/src/ws.test.ts
@@ -0,0 +1,196 @@
+import { describe, expect, it } from "vitest";
+import { encodeServerEvent, parseClientEvent, type ServerEvent } from "./ws";
+
+describe("parseClientEvent", () => {
+  it.each([
+    {
+      name: "queue join",
+      payload: { type: "queue.join", mode: "ai" },
+      expected: { type: "queue.join", mode: "ai" }
+    },
+    {
+      name: "queue leave",
+      payload: { type: "queue.leave" },
+      expected: { type: "queue.leave" }
+    },
+    {
+      name: "game ready",
+      payload: { type: "game.ready", roomId: "room-1" },
+      expected: { type: "game.ready", roomId: "room-1" }
+    },
+    {
+      name: "game input",
+      payload: { type: "game.input", roomId: "room-1", direction: -1 },
+      expected: { type: "game.input", roomId: "room-1", direction: -1 }
+    },
+    {
+      name: "chat send",
+      payload: { type: "chat.send", scope: "match", roomId: "room-1", body: "hello" },
+      expected: { type: "chat.send", scope: "match", roomId: "room-1", body: "hello" }
+    }
+  ])("accepts $name events", ({ payload, expected }) => {
+    expect(parseClientEvent(JSON.stringify(payload))).toEqual(expected);
+  });
+
+  it("defaults queue joins to queue mode", () => {
+    expect(parseClientEvent(JSON.stringify({ type: "queue.join" }))).toEqual({
+      type: "queue.join",
+      mode: "queue"
+    });
+  });
+
+  it("rejects unknown event types", () => {
+    expect(() => parseClientEvent(JSON.stringify({ type: "game.unknown" }))).toThrow();
+  });
+
+  it.each([
+    { name: "ready room id", payload: { type: "game.ready" } },
+    { name: "input room id", payload: { type: "game.input", direction: 0 } },
+    { name: "input direction", payload: { type: "game.input", roomId: "room-1" } },
+    { name: "chat scope", payload: { type: "chat.send", body: "hello" } },
+    { name: "chat body", payload: { type: "chat.send", scope: "lobby" } }
+  ])("rejects events without the required $name", ({ payload }) => {
+    expect(() => parseClientEvent(JSON.stringify(payload))).toThrow();
+  });
+
+  it.each([
+    { name: "queue mode", payload: { type: "queue.join", mode: "ranked" } },
+    { name: "chat scope", payload: { type: "chat.send", scope: "private", body: "hello" } },
+    { name: "direction below the range", payload: { type: "game.input", roomId: "room-1", direction: -2 } },
+    { name: "direction above the range", payload: { type: "game.input", roomId: "room-1", direction: 2 } },
+    { name: "non-numeric direction", payload: { type: "game.input", roomId: "room-1", direction: "1" } }
+  ])("rejects an invalid $name", ({ payload }) => {
+    expect(() => parseClientEvent(JSON.stringify(payload))).toThrow();
+  });
+
+  it.each([-1, 0, 1])("accepts %i as an input direction", (direction) => {
+    expect(
+      parseClientEvent(JSON.stringify({ type: "game.input", roomId: "room-1", direction }))
+    ).toEqual({ type: "game.input", roomId: "room-1", direction });
+  });
+
+  it("trims chat bodies", () => {
+    expect(
+      parseClientEvent(JSON.stringify({ type: "chat.send", scope: "lobby", body: "  hello  " }))
+    ).toEqual({ type: "chat.send", scope: "lobby", body: "hello" });
+  });
+
+  it("accepts chat bodies at the 1 and 240 character boundaries", () => {
+    const oneCharacter = parseClientEvent(
+      JSON.stringify({ type: "chat.send", scope: "lobby", body: "a" })
+    );
+    const twoHundredFortyCharacters = parseClientEvent(
+      JSON.stringify({ type: "chat.send", scope: "lobby", body: "a".repeat(240) })
+    );
+
+    expect(oneCharacter).toMatchObject({ body: "a" });
+    expect(twoHundredFortyCharacters).toMatchObject({ body: "a".repeat(240) });
+  });
+
+  it.each(["", "   ", "a".repeat(241)])("rejects an invalid chat body length", (body) => {
+    expect(() =>
+      parseClientEvent(JSON.stringify({ type: "chat.send", scope: "lobby", body }))
+    ).toThrow();
+  });
+
+  it("rejects malformed JSON", () => {
+    expect(() => parseClientEvent('{"type":"queue.leave"')).toThrow(SyntaxError);
+  });
+});
+
+describe("encodeServerEvent", () => {
+  const serverEvents = [
+    {
+      type: "queue.matched",
+      roomId: "room-1",
+      side: "left",
+      opponent: "opponent"
+    },
+    {
+      type: "game.snapshot",
+      snapshot: {
+        roomId: "room-1",
+        phase: "playing",
+        tick: 12,
+        leftScore: 1,
+        rightScore: 0,
+        paddles: {
+          left: { y: 100, dy: -1 },
+          right: { y: 200, dy: 1 }
+        },
+        ball: {
+          position: { x: 480, y: 270 },
+          velocity: { x: 6, y: -2 }
+        },
+        players: [
+          {
+            id: "player-1",
+            handle: "left-player",
+            displayName: "Left Player",
+            side: "left",
+            ready: true,
+            ai: false
+          },
+          {
+            id: "player-2",
+            handle: "right-player",
+            displayName: "Right Player",
+            side: "right",
+            ready: true,
+            ai: false
+          }
+        ],
+        serverTime: "2026-07-23T00:00:00.000Z"
+      }
+    },
+    {
+      type: "game.finished",
+      result: {
+        roomId: "room-1",
+        matchId: "match-1",
+        winnerSide: "left",
+        leftScore: 3,
+        rightScore: 1,
+        ratingDelta: 16
+      }
+    },
+    {
+      type: "chat.message",
+      message: {
+        id: "message-1",
+        scope: "lobby",
+        roomId: null,
+        sender: {
+          id: "player-1",
+          handle: "left-player",
+          displayName: "Left Player",
+          avatarKey: "avatar-1",
+          role: "user",
+          status: "active",
+          rating: 1000,
+          wins: 1,
+          losses: 0,
+          online: true
+        },
+        body: "hello",
+        createdAt: "2026-07-23T00:00:00.000Z"
+      }
+    },
+    {
+      type: "presence.changed",
+      online: 12,
+      playing: 4
+    },
+    {
+      type: "error",
+      message: "invalid event"
+    }
+  ] satisfies ServerEvent[];
+
+  it.each(serverEvents)("serializes $type events", (event) => {
+    const encoded = encodeServerEvent(event);
+
+    expect(encoded).toBe(JSON.stringify(event));
+    expect(JSON.parse(encoded)).toEqual(event);
+  });
+});


## `feat(protocol): versioned game snapshot 계약 정의`

diff --git a/packages/shared/src/game.ts b/packages/shared/src/game.ts
index bd49610..acd1829 100644
--- a/packages/shared/src/game.ts
+++ b/packages/shared/src/game.ts
@@ -1,3 +1,5 @@
+import { z } from "zod";
+
 export const GAME_WIDTH = 960;
 export const GAME_HEIGHT = 540;
 export const PADDLE_WIDTH = 18;
@@ -6,50 +8,80 @@ export const BALL_RADIUS = 10;
 export const WINNING_SCORE = 3;
 export const TICK_RATE = 20;
 
-export type PlayerSide = "left" | "right";
-export type GamePhase = "waiting" | "countdown" | "playing" | "paused" | "finished";
-
-export interface Vec2 {
-  x: number;
-  y: number;
-}
-
-export interface PaddleState {
-  y: number;
-  dy: -1 | 0 | 1;
-}
-
-export interface BallState {
-  position: Vec2;
-  velocity: Vec2;
-}
-
-export interface PlayerSlot {
-  id: string;
-  handle: string;
-  displayName: string;
-  side: PlayerSide;
-  ready: boolean;
-  ai: boolean;
-}
-
-export interface GameSnapshot {
-  roomId: string;
-  phase: GamePhase;
-  tick: number;
-  leftScore: number;
-  rightScore: number;
-  paddles: Record<PlayerSide, PaddleState>;
-  ball: BallState;
-  players: PlayerSlot[];
-  serverTime: string;
-}
-
-export interface GameFinished {
-  roomId: string;
-  matchId: string;
-  winnerSide: PlayerSide;
-  leftScore: number;
-  rightScore: number;
-  ratingDelta: number;
-}
+export const playerSideSchema = z.enum(["left", "right"]);
+export const gamePhaseSchema = z.enum(["waiting", "countdown", "playing", "paused", "finished"]);
+
+export const vec2Schema = z.object({
+  x: z.number().finite(),
+  y: z.number().finite()
+}).strict();
+
+export const paddleStateSchema = z.object({
+  y: z.number().finite(),
+  dy: z.union([z.literal(-1), z.literal(0), z.literal(1)])
+}).strict();
+
+export const ballStateSchema = z.object({
+  position: vec2Schema,
+  velocity: vec2Schema
+}).strict();
+
+export const playerSlotSchema = z.object({
+  id: z.string().min(1),
+  handle: z.string().min(1),
+  displayName: z.string().min(1),
+  side: playerSideSchema,
+  ready: z.boolean(),
+  ai: z.boolean()
+}).strict();
+
+export const gameStateSchema = z.object({
+  phase: gamePhaseSchema,
+  leftScore: z.number().int().nonnegative(),
+  rightScore: z.number().int().nonnegative(),
+  paddles: z.object({
+    left: paddleStateSchema,
+    right: paddleStateSchema
+  }).strict(),
+  ball: ballStateSchema,
+  players: z.array(playerSlotSchema)
+}).strict();
+
+export const gameSnapshotSchema = z.object({
+  roomId: z.string().min(1),
+  tick: z.number().int().nonnegative(),
+  sequence: z.number().int().nonnegative(),
+  serverTimeMs: z.number().int().nonnegative(),
+  state: gameStateSchema
+}).strict();
+
+const persistedGameFinishedSchema = z.object({
+  roomId: z.string().min(1),
+  matchId: z.string().min(1),
+  persisted: z.literal(true),
+  winnerSide: playerSideSchema,
+  leftScore: z.number().int().nonnegative(),
+  rightScore: z.number().int().nonnegative(),
+  ratingDelta: z.number().finite()
+}).strict();
+
+const transientGameFinishedSchema = persistedGameFinishedSchema.extend({
+  matchId: z.null(),
+  persisted: z.literal(false),
+  ratingDelta: z.literal(0)
+}).strict();
+
+export const gameFinishedSchema = z.discriminatedUnion("persisted", [
+  persistedGameFinishedSchema,
+  transientGameFinishedSchema
+]);
+
+export type PlayerSide = z.infer<typeof playerSideSchema>;
+export type GamePhase = z.infer<typeof gamePhaseSchema>;
+export type Vec2 = z.infer<typeof vec2Schema>;
+export type PaddleState = z.infer<typeof paddleStateSchema>;
+export type BallState = z.infer<typeof ballStateSchema>;
+export type PlayerSlot = z.infer<typeof playerSlotSchema>;
+export type GameState = z.infer<typeof gameStateSchema>;
+export type GameSnapshot = z.infer<typeof gameSnapshotSchema>;
+export type GameFinished = z.infer<typeof gameFinishedSchema>;


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


