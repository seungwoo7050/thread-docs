# 결정적 서버 권위형 퐁 시뮬레이션

## `feat(shared): 퐁 시뮬레이션 계약 추가`

diff --git a/packages/shared/src/game.ts b/packages/shared/src/game.ts
new file mode 100644
index 0000000..8bfecf7
--- /dev/null
+++ b/packages/shared/src/game.ts
@@ -0,0 +1,55 @@
+export const GAME_WIDTH = 960;
+export const GAME_HEIGHT = 540;
+export const PADDLE_WIDTH = 18;
+export const PADDLE_HEIGHT = 112;
+export const BALL_RADIUS = 10;
+export const WINNING_SCORE = 3;
+export const TICK_RATE = 20;
+
+export type PlayerSide = "left" | "right";
+export type GamePhase = "waiting" | "countdown" | "playing" | "finished";
+
+export interface Vec2 {
+  x: number;
+  y: number;
+}
+
+export interface PaddleState {
+  y: number;
+  dy: -1 | 0 | 1;
+}
+
+export interface BallState {
+  position: Vec2;
+  velocity: Vec2;
+}
+
+export interface PlayerSlot {
+  id: string;
+  handle: string;
+  displayName: string;
+  side: PlayerSide;
+  ready: boolean;
+  ai: boolean;
+}
+
+export interface GameSnapshot {
+  roomId: string;
+  phase: GamePhase;
+  tick: number;
+  leftScore: number;
+  rightScore: number;
+  paddles: Record<PlayerSide, PaddleState>;
+  ball: BallState;
+  players: PlayerSlot[];
+  serverTime: string;
+}
+
+export interface GameFinished {
+  roomId: string;
+  matchId: string;
+  winnerSide: PlayerSide;
+  leftScore: number;
+  rightScore: number;
+  ratingDelta: number;
+}
diff --git a/packages/shared/src/index.ts b/packages/shared/src/index.ts
index b4f0c19..7bb2940 100644
--- a/packages/shared/src/index.ts
+++ b/packages/shared/src/index.ts
@@ -1 +1,2 @@
 export * from "./http";
+export * from "./game";


## `feat(game): 서버 주도 퐁 물리 갱신`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index 228e237..17d4306 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -3,6 +3,7 @@ import { randomUUID } from "node:crypto";
 import { WebSocket } from "ws";
 import type { AppRepository } from "@pong-pong/db";
 import {
+  BALL_RADIUS,
   GAME_HEIGHT,
   GAME_WIDTH,
   PADDLE_HEIGHT,
@@ -117,7 +118,10 @@ export class GameHub {
           left: { y: GAME_HEIGHT / 2 - PADDLE_HEIGHT / 2, dy: 0 },
           right: { y: GAME_HEIGHT / 2 - PADDLE_HEIGHT / 2, dy: 0 }
         },
-        ball: { position: { x: GAME_WIDTH / 2, y: GAME_HEIGHT / 2 }, velocity: { x: 7, y: 4 } },
+        ball: {
+          position: { x: GAME_WIDTH / 2, y: GAME_HEIGHT / 2 },
+          velocity: { x: 7, y: 4 }
+        },
         players: [
           { id: left.user.id, handle: left.user.handle, displayName: left.user.displayName, side: "left", ready: false, ai: false },
           {
@@ -141,8 +145,46 @@ export class GameHub {
     this.broadcastPresence();
   }
 
+  private async tick(room: Room): Promise<void> {
+    const state = room.snapshot;
+    if (state.phase !== "playing") return;
+    state.tick += 1;
+    state.serverTime = new Date().toISOString();
+
+    const speed = 13;
+    state.paddles.left.y = clamp(state.paddles.left.y + state.paddles.left.dy * speed, 16, GAME_HEIGHT - PADDLE_HEIGHT - 16);
+    if (room.ai) {
+      const center = state.paddles.right.y + PADDLE_HEIGHT / 2;
+      state.paddles.right.dy = state.ball.position.y > center + 14 ? 1 : state.ball.position.y < center - 14 ? -1 : 0;
+    }
+    state.paddles.right.y = clamp(state.paddles.right.y + state.paddles.right.dy * speed, 16, GAME_HEIGHT - PADDLE_HEIGHT - 16);
+
+    state.ball.position.x += state.ball.velocity.x;
+    state.ball.position.y += state.ball.velocity.y;
+    if (state.ball.position.y < BALL_RADIUS || state.ball.position.y > GAME_HEIGHT - BALL_RADIUS) {
+      state.ball.velocity.y *= -1;
+    }
+    collidePaddle(state, "left", 32);
+    collidePaddle(state, "right", GAME_WIDTH - 32);
+
+    if (state.ball.position.x < 0) {
+      state.rightScore += 1;
+      resetBall(state, -1);
+    }
+    if (state.ball.position.x > GAME_WIDTH) {
+      state.leftScore += 1;
+      resetBall(state, 1);
+    }
+    this.broadcastRoom(room.id, { type: "game.snapshot", snapshot: state });
+
+  }
+
   private broadcastPresence(): void {
-    this.broadcastAll({ type: "presence.changed", online: this.clients.size, playing: this.rooms.size * 2 });
+    this.broadcastAll({
+      type: "presence.changed",
+      online: this.clients.size,
+      playing: this.rooms.size * 2
+    });
   }
 
   private broadcastAll(event: ServerEvent): void {
@@ -158,6 +200,29 @@ export class GameHub {
   }
 
   private send(client: Client, event: ServerEvent): void {
-    if (client.socket.readyState === WebSocket.OPEN) client.socket.send(encodeServerEvent(event));
+    if (client.socket.readyState === WebSocket.OPEN) {
+      client.socket.send(encodeServerEvent(event));
+    }
+  }
+}
+
+function clamp(value: number, min: number, max: number): number {
+  return Math.max(min, Math.min(max, value));
+}
+
+function resetBall(state: GameSnapshot, xDirection: 1 | -1): void {
+  state.ball.position = { x: GAME_WIDTH / 2, y: GAME_HEIGHT / 2 };
+  state.ball.velocity = { x: 7 * xDirection, y: state.tick % 2 === 0 ? 4 : -4 };
+}
+
+function collidePaddle(state: GameSnapshot, side: PlayerSide, x: number): void {
+  const paddle = state.paddles[side];
+  const ball = state.ball;
+  const withinY = ball.position.y >= paddle.y && ball.position.y <= paddle.y + PADDLE_HEIGHT;
+  const withinX = side === "left" ? ball.position.x - BALL_RADIUS <= x + 18 : ball.position.x + BALL_RADIUS >= x - 18;
+  if (withinX && withinY && Math.sign(ball.velocity.x) === (side === "left" ? -1 : 1)) {
+    ball.velocity.x *= -1.04;
+    const offset = (ball.position.y - (paddle.y + PADDLE_HEIGHT / 2)) / (PADDLE_HEIGHT / 2);
+    ball.velocity.y = offset * 7;
   }
 }


## `fix(game): 경기 시간에 따라 공 속도 증가`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index 9931aed..72c1abf 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -43,6 +43,10 @@ type Room = {
   tournamentMatchId: string | null;
 };
 
+const INITIAL_BALL_VELOCITY = { x: 10, y: 5 };
+const BALL_ACCELERATION_PER_TICK = 0.015;
+const MAX_BALL_SPEED = 18;
+
 export class GameHub {
   private readonly clients = new Map<string, Client>();
   private readonly queue: QueueEntry[] = [];
@@ -241,7 +245,7 @@ export class GameHub {
         },
         ball: {
           position: { x: GAME_WIDTH / 2, y: GAME_HEIGHT / 2 },
-          velocity: { x: 7, y: 4 }
+          velocity: { x: INITIAL_BALL_VELOCITY.x, y: INITIAL_BALL_VELOCITY.y }
         },
         players: [
           { id: left.user.id, handle: left.user.handle, displayName: left.user.displayName, side: "left", ready: false, ai: false },
@@ -342,6 +346,7 @@ export class GameHub {
       state.leftScore += 1;
       resetBall(state, 1);
     }
+    accelerateBall(state);
     this.broadcastRoom(room.id, { type: "game.snapshot", snapshot: state });
 
     if (state.leftScore >= WINNING_SCORE || state.rightScore >= WINNING_SCORE || state.tick >= TICK_RATE * 45) {
@@ -427,7 +432,11 @@ function clamp(value: number, min: number, max: number): number {
 
 function resetBall(state: GameSnapshot, xDirection: 1 | -1): void {
   state.ball.position = { x: GAME_WIDTH / 2, y: GAME_HEIGHT / 2 };
-  state.ball.velocity = { x: 7 * xDirection, y: state.tick % 2 === 0 ? 4 : -4 };
+  const elapsedBoost = Math.min(1.35, 1 + state.tick / (TICK_RATE * 90));
+  state.ball.velocity = {
+    x: INITIAL_BALL_VELOCITY.x * elapsedBoost * xDirection,
+    y: INITIAL_BALL_VELOCITY.y * elapsedBoost * (state.tick % 2 === 0 ? 1 : -1)
+  };
 }
 
 function collidePaddle(state: GameSnapshot, side: PlayerSide, x: number): void {
@@ -441,3 +450,14 @@ function collidePaddle(state: GameSnapshot, side: PlayerSide, x: number): void {
     ball.velocity.y = offset * 7;
   }
 }
+
+function accelerateBall(state: GameSnapshot): void {
+  const velocity = state.ball.velocity;
+  const currentSpeed = Math.hypot(velocity.x, velocity.y);
+  if (currentSpeed <= 0 || currentSpeed >= MAX_BALL_SPEED) return;
+  const elapsedMinimum = Math.min(MAX_BALL_SPEED, Math.hypot(INITIAL_BALL_VELOCITY.x, INITIAL_BALL_VELOCITY.y) + state.tick * BALL_ACCELERATION_PER_TICK);
+  const nextSpeed = Math.min(MAX_BALL_SPEED, Math.max(currentSpeed + BALL_ACCELERATION_PER_TICK, elapsedMinimum));
+  const scale = nextSpeed / currentSpeed;
+  velocity.x *= scale;
+  velocity.y *= scale;
+}


## `refactor(game): Pong simulation 상태와 초기화 분리`

diff --git a/apps/api/src/game/pongSimulation.ts b/apps/api/src/game/pongSimulation.ts
new file mode 100644
index 0000000..99c9a09
--- /dev/null
+++ b/apps/api/src/game/pongSimulation.ts
@@ -0,0 +1,70 @@
+import {
+  GAME_HEIGHT,
+  GAME_WIDTH,
+  PADDLE_HEIGHT,
+  TICK_RATE,
+  type BallState,
+  type PlayerSide
+} from "@pong-pong/shared";
+
+export type PaddleDirection = -1 | 0 | 1;
+
+export interface SimulationPaddleState {
+  y: number;
+  direction: PaddleDirection;
+}
+
+export interface PongSimulationState {
+  tick: number;
+  phase: "playing" | "finished";
+  leftScore: number;
+  rightScore: number;
+  paddles: Record<PlayerSide, SimulationPaddleState>;
+  ball: BallState;
+  winnerSide: PlayerSide | null;
+}
+
+export interface PongSimulationInputs {
+  left: PaddleDirection;
+  right: PaddleDirection;
+}
+
+const INITIAL_BALL_VELOCITY = { x: 10, y: 5 } as const;
+
+export class PongSimulation {
+  static initialState(): PongSimulationState {
+    return {
+      tick: 0,
+      phase: "playing",
+      leftScore: 0,
+      rightScore: 0,
+      paddles: {
+        left: { y: GAME_HEIGHT / 2 - PADDLE_HEIGHT / 2, direction: 0 },
+        right: { y: GAME_HEIGHT / 2 - PADDLE_HEIGHT / 2, direction: 0 }
+      },
+      ball: {
+        position: { x: GAME_WIDTH / 2, y: GAME_HEIGHT / 2 },
+        velocity: { ...INITIAL_BALL_VELOCITY }
+      },
+      winnerSide: null
+    };
+  }
+}
+
+function cloneState(state: Readonly<PongSimulationState>): PongSimulationState {
+  return {
+    tick: state.tick,
+    phase: state.phase,
+    leftScore: state.leftScore,
+    rightScore: state.rightScore,
+    paddles: {
+      left: { ...state.paddles.left },
+      right: { ...state.paddles.right }
+    },
+    ball: {
+      position: { ...state.ball.position },
+      velocity: { ...state.ball.velocity }
+    },
+    winnerSide: state.winnerSide
+  };
+}


## `refactor(game): paddle 이동과 벽 반사 모델링`

diff --git a/apps/api/src/game/pongSimulation.ts b/apps/api/src/game/pongSimulation.ts
index 99c9a09..10c1b06 100644
--- a/apps/api/src/game/pongSimulation.ts
+++ b/apps/api/src/game/pongSimulation.ts
@@ -1,4 +1,5 @@
 import {
+  BALL_RADIUS,
   GAME_HEIGHT,
   GAME_WIDTH,
   PADDLE_HEIGHT,
@@ -30,6 +31,9 @@ export interface PongSimulationInputs {
 }
 
 const INITIAL_BALL_VELOCITY = { x: 10, y: 5 } as const;
+const FIXED_TIMESTEP_MS = 1000 / TICK_RATE;
+const PADDLE_SPEED_PER_TICK = 13;
+const ARENA_PADDING = 16;
 
 export class PongSimulation {
   static initialState(): PongSimulationState {
@@ -49,6 +53,27 @@ export class PongSimulation {
       winnerSide: null
     };
   }
+
+  static step(
+    state: Readonly<PongSimulationState>,
+    inputs: Readonly<PongSimulationInputs>,
+    deltaMs: number
+  ): PongSimulationState {
+    if (!Number.isFinite(deltaMs) || deltaMs <= 0) {
+      throw new RangeError("deltaMs must be a positive finite number");
+    }
+    if (state.phase === "finished") return cloneState(state);
+
+    const next = cloneState(state);
+    const timestepScale = deltaMs / FIXED_TIMESTEP_MS;
+    next.tick += 1;
+    movePaddle(next, "left", inputs.left, timestepScale);
+    movePaddle(next, "right", inputs.right, timestepScale);
+    next.ball.position.x += next.ball.velocity.x * timestepScale;
+    next.ball.position.y += next.ball.velocity.y * timestepScale;
+    reflectVerticalWall(next.ball);
+    return next;
+  }
 }
 
 function cloneState(state: Readonly<PongSimulationState>): PongSimulationState {
@@ -68,3 +93,34 @@ function cloneState(state: Readonly<PongSimulationState>): PongSimulationState {
     winnerSide: state.winnerSide
   };
 }
+
+function movePaddle(
+  state: PongSimulationState,
+  side: PlayerSide,
+  direction: PaddleDirection,
+  timestepScale: number
+): void {
+  const paddle = state.paddles[side];
+  paddle.direction = direction;
+  paddle.y = clamp(
+    paddle.y + direction * PADDLE_SPEED_PER_TICK * timestepScale,
+    ARENA_PADDING,
+    GAME_HEIGHT - PADDLE_HEIGHT - ARENA_PADDING
+  );
+}
+
+function reflectVerticalWall(ball: BallState): void {
+  const min = BALL_RADIUS;
+  const max = GAME_HEIGHT - BALL_RADIUS;
+  if (ball.position.y < min) {
+    ball.position.y = min + (min - ball.position.y);
+    ball.velocity.y = Math.abs(ball.velocity.y);
+  } else if (ball.position.y > max) {
+    ball.position.y = max - (ball.position.y - max);
+    ball.velocity.y = -Math.abs(ball.velocity.y);
+  }
+}
+
+function clamp(value: number, min: number, max: number): number {
+  return Math.max(min, Math.min(max, value));
+}


## `refactor(game): 득점과 충돌을 simulation에 통합`

diff --git a/apps/api/src/game/pongSimulation.ts b/apps/api/src/game/pongSimulation.ts
index 10c1b06..b080a03 100644
--- a/apps/api/src/game/pongSimulation.ts
+++ b/apps/api/src/game/pongSimulation.ts
@@ -3,7 +3,9 @@ import {
   GAME_HEIGHT,
   GAME_WIDTH,
   PADDLE_HEIGHT,
+  PADDLE_WIDTH,
   TICK_RATE,
+  WINNING_SCORE,
   type BallState,
   type PlayerSide
 } from "@pong-pong/shared";
@@ -30,9 +32,12 @@ export interface PongSimulationInputs {
   right: PaddleDirection;
 }
 
-const INITIAL_BALL_VELOCITY = { x: 10, y: 5 } as const;
 const FIXED_TIMESTEP_MS = 1000 / TICK_RATE;
+const INITIAL_BALL_VELOCITY = { x: 10, y: 5 } as const;
 const PADDLE_SPEED_PER_TICK = 13;
+const BALL_ACCELERATION_PER_TICK = 0.015;
+const MAX_BALL_SPEED = 18;
+const MAX_MATCH_TICKS = TICK_RATE * 45;
 const ARENA_PADDING = 16;
 
 export class PongSimulation {
@@ -69,9 +74,33 @@ export class PongSimulation {
     next.tick += 1;
     movePaddle(next, "left", inputs.left, timestepScale);
     movePaddle(next, "right", inputs.right, timestepScale);
+
     next.ball.position.x += next.ball.velocity.x * timestepScale;
     next.ball.position.y += next.ball.velocity.y * timestepScale;
     reflectVerticalWall(next.ball);
+    collidePaddle(next, "left", 32);
+    collidePaddle(next, "right", GAME_WIDTH - 32);
+
+    if (next.ball.position.x < 0) {
+      next.rightScore += 1;
+      resetBall(next, -1);
+    } else if (next.ball.position.x > GAME_WIDTH) {
+      next.leftScore += 1;
+      resetBall(next, 1);
+    }
+
+    accelerateBall(next, timestepScale);
+    if (
+      next.leftScore >= WINNING_SCORE ||
+      next.rightScore >= WINNING_SCORE ||
+      next.tick >= MAX_MATCH_TICKS
+    ) {
+      next.phase = "finished";
+      next.winnerSide = next.leftScore >= next.rightScore ? "left" : "right";
+      next.paddles.left.direction = 0;
+      next.paddles.right.direction = 0;
+    }
+
     return next;
   }
 }
@@ -121,6 +150,50 @@ function reflectVerticalWall(ball: BallState): void {
   }
 }
 
+function collidePaddle(state: PongSimulationState, side: PlayerSide, x: number): void {
+  const paddle = state.paddles[side];
+  const ball = state.ball;
+  const withinY = ball.position.y >= paddle.y && ball.position.y <= paddle.y + PADDLE_HEIGHT;
+  const halfPaddle = PADDLE_WIDTH / 2;
+  const withinX = side === "left"
+    ? ball.position.x - BALL_RADIUS <= x + halfPaddle
+    : ball.position.x + BALL_RADIUS >= x - halfPaddle;
+  const approaching = Math.sign(ball.velocity.x) === (side === "left" ? -1 : 1);
+  if (!withinX || !withinY || !approaching) return;
+
+  ball.velocity.x *= -1.04;
+  const offset = (ball.position.y - (paddle.y + PADDLE_HEIGHT / 2)) / (PADDLE_HEIGHT / 2);
+  ball.velocity.y = offset * 7;
+}
+
+function resetBall(state: PongSimulationState, xDirection: 1 | -1): void {
+  state.ball.position = { x: GAME_WIDTH / 2, y: GAME_HEIGHT / 2 };
+  const elapsedBoost = Math.min(1.35, 1 + state.tick / (TICK_RATE * 90));
+  state.ball.velocity = {
+    x: INITIAL_BALL_VELOCITY.x * elapsedBoost * xDirection,
+    y: INITIAL_BALL_VELOCITY.y * elapsedBoost * (state.tick % 2 === 0 ? 1 : -1)
+  };
+}
+
+function accelerateBall(state: PongSimulationState, timestepScale: number): void {
+  const velocity = state.ball.velocity;
+  const currentSpeed = Math.hypot(velocity.x, velocity.y);
+  if (currentSpeed <= 0 || currentSpeed >= MAX_BALL_SPEED) return;
+
+  const elapsedMinimum = Math.min(
+    MAX_BALL_SPEED,
+    Math.hypot(INITIAL_BALL_VELOCITY.x, INITIAL_BALL_VELOCITY.y) +
+      state.tick * BALL_ACCELERATION_PER_TICK
+  );
+  const nextSpeed = Math.min(
+    MAX_BALL_SPEED,
+    Math.max(currentSpeed + BALL_ACCELERATION_PER_TICK * timestepScale, elapsedMinimum)
+  );
+  const scale = nextSpeed / currentSpeed;
+  velocity.x *= scale;
+  velocity.y *= scale;
+}
+
 function clamp(value: number, min: number, max: number): number {
   return Math.max(min, Math.min(max, value));
 }


## `test(game): 결정적 simulation 검증`

diff --git a/apps/api/src/game/pongAi.test.ts b/apps/api/src/game/pongAi.test.ts
new file mode 100644
index 0000000..0c003bf
--- /dev/null
+++ b/apps/api/src/game/pongAi.test.ts
@@ -0,0 +1,51 @@
+import { readFile } from "node:fs/promises";
+import { describe, expect, it } from "vitest";
+import { PongAi, SeededIntegerPrng } from "./pongAi";
+import { PongSimulation } from "./pongSimulation";
+
+describe("SeededIntegerPrng", () => {
+  it("produces the same integer stream for the same seed", () => {
+    const left = new SeededIntegerPrng("same-seed");
+    const right = new SeededIntegerPrng("same-seed");
+
+    expect(Array.from({ length: 20 }, () => left.nextUint32())).toEqual(
+      Array.from({ length: 20 }, () => right.nextUint32())
+    );
+  });
+
+  it("does not rely on floating point pseudo-random helpers", async () => {
+    const source = await readFile(new URL("./pongAi.ts", import.meta.url), "utf8");
+    expect(source).not.toContain("Math.random");
+    expect(source).not.toContain("Math.sin");
+  });
+});
+
+describe("PongAi", () => {
+  it("returns the same input sequence for the same seed and states", () => {
+    const first = new PongAi("room-seed", 1_300);
+    const second = new PongAi("room-seed", 1_300);
+    let state = PongSimulation.initialState();
+    const firstDirections: number[] = [];
+    const secondDirections: number[] = [];
+
+    for (let tick = 0; tick < 120; tick += 1) {
+      const firstDirection = first.nextDirection(state);
+      const secondDirection = second.nextDirection(state);
+      firstDirections.push(firstDirection);
+      secondDirections.push(secondDirection);
+      state = PongSimulation.step(state, { left: 0, right: firstDirection }, 50);
+    }
+
+    expect(firstDirections).toEqual(secondDirections);
+    expect(first.snapshot()).toEqual(second.snapshot());
+  });
+
+  it("stops producing movement after a match finishes", () => {
+    const ai = new PongAi(42, 1_400);
+    const state = PongSimulation.initialState();
+    state.phase = "finished";
+    state.winnerSide = "left";
+
+    expect(ai.nextDirection(state)).toBe(0);
+  });
+});
diff --git a/apps/api/src/game/pongSimulation.test.ts b/apps/api/src/game/pongSimulation.test.ts
new file mode 100644
index 0000000..d9ca958
--- /dev/null
+++ b/apps/api/src/game/pongSimulation.test.ts
@@ -0,0 +1,88 @@
+import { createHash } from "node:crypto";
+import { describe, expect, it } from "vitest";
+import { GAME_HEIGHT, PADDLE_HEIGHT } from "@pong-pong/shared";
+import { PongAi } from "./pongAi";
+import { PongSimulation, type PongSimulationInputs } from "./pongSimulation";
+
+const FIXED_DELTA_MS = 50;
+
+describe("PongSimulation", () => {
+  it("returns a deterministic next state without mutating its input", () => {
+    const initial = PongSimulation.initialState();
+    const before = structuredClone(initial);
+    const inputs = { left: -1, right: 1 } as const;
+
+    const first = PongSimulation.step(initial, inputs, FIXED_DELTA_MS);
+    const second = PongSimulation.step(initial, inputs, FIXED_DELTA_MS);
+
+    expect(first).toEqual(second);
+    expect(initial).toEqual(before);
+    expect(first).not.toBe(initial);
+    expect(first.paddles.left).not.toBe(initial.paddles.left);
+    expect(first.ball).not.toBe(initial.ball);
+  });
+
+  it("scales movement by delta while clamping paddles to the arena", () => {
+    const initial = PongSimulation.initialState();
+    const halfStep = PongSimulation.step(initial, { left: 1, right: 0 }, 25);
+    const fullStep = PongSimulation.step(initial, { left: 1, right: 0 }, 50);
+
+    expect(fullStep.paddles.left.y - initial.paddles.left.y).toBeCloseTo(
+      (halfStep.paddles.left.y - initial.paddles.left.y) * 2
+    );
+
+    let state = initial;
+    for (let tick = 0; tick < 100; tick += 1) {
+      state = PongSimulation.step(state, { left: 1, right: -1 }, FIXED_DELTA_MS);
+    }
+    expect(state.paddles.left.y).toBeLessThanOrEqual(GAME_HEIGHT - PADDLE_HEIGHT - 16);
+    expect(state.paddles.right.y).toBeGreaterThanOrEqual(16);
+  });
+
+  it("finishes when the winning score is reached", () => {
+    const state = PongSimulation.initialState();
+    state.rightScore = 2;
+    state.ball.position.x = -1;
+    state.ball.velocity = { x: 0, y: 0 };
+
+    const finished = PongSimulation.step(state, { left: 0, right: 0 }, FIXED_DELTA_MS);
+
+    expect(finished).toMatchObject({
+      phase: "finished",
+      leftScore: 0,
+      rightScore: 3,
+      winnerSide: "right"
+    });
+  });
+
+  it("rejects invalid time deltas", () => {
+    const state = PongSimulation.initialState();
+    expect(() => PongSimulation.step(state, { left: 0, right: 0 }, 0)).toThrow(RangeError);
+    expect(() => PongSimulation.step(state, { left: 0, right: 0 }, Number.NaN)).toThrow(RangeError);
+  });
+
+  it("replays one thousand ticks to the same final hash", () => {
+    const first = replayHash("replay-seed-2026");
+    const second = replayHash("replay-seed-2026");
+
+    expect(first).toBe(second);
+    expect(first).toMatch(/^[a-f0-9]{64}$/);
+  });
+});
+
+function replayHash(seed: string): string {
+  const ai = new PongAi(seed, 1_300);
+  let state = PongSimulation.initialState();
+
+  for (let tick = 0; tick < 1_000; tick += 1) {
+    const inputs: PongSimulationInputs = {
+      left: tick % 60 < 20 ? -1 : tick % 60 < 40 ? 1 : 0,
+      right: ai.nextDirection(state)
+    };
+    state = PongSimulation.step(state, inputs, FIXED_DELTA_MS);
+  }
+
+  return createHash("sha256")
+    .update(JSON.stringify({ state, ai: ai.snapshot() }))
+    .digest("hex");
+}


