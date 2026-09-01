## `refactor(game): 결정적 정수 난수 생성기 추가`

diff --git a/apps/api/src/game/pongAi.ts b/apps/api/src/game/pongAi.ts
new file mode 100644
index 0000000..214318e
--- /dev/null
+++ b/apps/api/src/game/pongAi.ts
@@ -0,0 +1,53 @@
+import { BALL_RADIUS, GAME_HEIGHT, GAME_WIDTH, PADDLE_HEIGHT } from "@pong-pong/shared";
+import type { PaddleDirection, PongSimulationState } from "./pongSimulation";
+
+interface AiProfile {
+  reactionTicks: number;
+  predictionNoise: number;
+  mistakeBasisPoints: number;
+  deadZone: number;
+}
+
+export interface PongAiSnapshot {
+  randomState: number;
+  targetY: number;
+  nextReactionTick: number;
+}
+
+export class SeededIntegerPrng {
+  private state: number;
+
+  constructor(seed: number | string) {
+    const normalized = typeof seed === "number" ? seed >>> 0 : hashSeed(seed);
+    this.state = normalized === 0 ? 0x6d2b79f5 : normalized;
+  }
+
+  nextUint32(): number {
+    let value = this.state;
+    value ^= value << 13;
+    value ^= value >>> 17;
+    value ^= value << 5;
+    this.state = value >>> 0;
+    return this.state;
+  }
+
+  nextInt(maxExclusive: number): number {
+    if (!Number.isSafeInteger(maxExclusive) || maxExclusive <= 0) {
+      throw new RangeError("maxExclusive must be a positive safe integer");
+    }
+    return this.nextUint32() % maxExclusive;
+  }
+
+  snapshot(): number {
+    return this.state;
+  }
+}
+
+function hashSeed(value: string): number {
+  let hash = 0x811c9dc5;
+  for (let index = 0; index < value.length; index += 1) {
+    hash ^= value.charCodeAt(index);
+    hash = Math.imul(hash, 0x01000193);
+  }
+  return hash >>> 0;
+}


## `refactor(game): rating 기반 Pong AI 정책 분리`

diff --git a/apps/api/src/game/pongAi.ts b/apps/api/src/game/pongAi.ts
index 214318e..4bf2a5a 100644
--- a/apps/api/src/game/pongAi.ts
+++ b/apps/api/src/game/pongAi.ts
@@ -43,6 +43,77 @@ export class SeededIntegerPrng {
   }
 }
 
+export class PongAi {
+  private readonly random: SeededIntegerPrng;
+  private readonly profile: AiProfile;
+  private targetY = GAME_HEIGHT / 2;
+  private nextReactionTick = 0;
+
+  constructor(seed: number | string, rating: number) {
+    this.random = new SeededIntegerPrng(seed);
+    this.profile = profileFor(rating);
+  }
+
+  nextDirection(state: Readonly<PongSimulationState>): PaddleDirection {
+    if (state.phase !== "playing") return 0;
+
+    if (state.tick >= this.nextReactionTick) {
+      const targetBase = state.ball.velocity.x > 0
+        ? predictedBallY(state)
+        : GAME_HEIGHT / 2;
+      const noise = this.random.nextInt(this.profile.predictionNoise * 2 + 1) - this.profile.predictionNoise;
+      const makesMistake = this.random.nextInt(10_000) < this.profile.mistakeBasisPoints;
+      const mistakeOffset = makesMistake ? this.random.nextInt(221) - 110 : 0;
+      this.targetY = clamp(
+        targetBase + noise + mistakeOffset,
+        16 + PADDLE_HEIGHT / 2,
+        GAME_HEIGHT - 16 - PADDLE_HEIGHT / 2
+      );
+      this.nextReactionTick = state.tick + this.profile.reactionTicks;
+    }
+
+    const center = state.paddles.right.y + PADDLE_HEIGHT / 2;
+    if (this.targetY > center + this.profile.deadZone) return 1;
+    if (this.targetY < center - this.profile.deadZone) return -1;
+    return 0;
+  }
+
+  snapshot(): PongAiSnapshot {
+    return {
+      randomState: this.random.snapshot(),
+      targetY: this.targetY,
+      nextReactionTick: this.nextReactionTick
+    };
+  }
+}
+
+function profileFor(rating: number): AiProfile {
+  if (rating >= 1400) {
+    return { reactionTicks: 3, predictionNoise: 20, mistakeBasisPoints: 400, deadZone: 10 };
+  }
+  if (rating >= 1300) {
+    return { reactionTicks: 4, predictionNoise: 34, mistakeBasisPoints: 800, deadZone: 14 };
+  }
+  if (rating >= 1200) {
+    return { reactionTicks: 6, predictionNoise: 54, mistakeBasisPoints: 1_200, deadZone: 18 };
+  }
+  return { reactionTicks: 8, predictionNoise: 78, mistakeBasisPoints: 1_800, deadZone: 24 };
+}
+
+function predictedBallY(state: Readonly<PongSimulationState>): number {
+  if (state.ball.velocity.x <= 0) return state.ball.position.y;
+  const distance = GAME_WIDTH - 32 - state.ball.position.x;
+  const ticks = distance / Math.max(1, state.ball.velocity.x);
+  let y = state.ball.position.y + state.ball.velocity.y * ticks;
+  const min = BALL_RADIUS;
+  const max = GAME_HEIGHT - BALL_RADIUS;
+  while (y < min || y > max) {
+    if (y < min) y = min + (min - y);
+    if (y > max) y = max - (y - max);
+  }
+  return y;
+}
+
 function hashSeed(value: string): number {
   let hash = 0x811c9dc5;
   for (let index = 0; index < value.length; index += 1) {
@@ -51,3 +122,7 @@ function hashSeed(value: string): number {
   }
   return hash >>> 0;
 }
+
+function clamp(value: number, min: number, max: number): number {
+  return Math.max(min, Math.min(max, value));
+}


## `refactor(game): GameHub에 결정적 AI controller 연결`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index cafaac9..1be5283 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -19,6 +19,7 @@ import {
   type ServerEvent,
   type SessionUser
 } from "@pong-pong/shared";
+import { PongAi } from "./game/pongAi";
 import { PongSimulation, type PongSimulationState } from "./game/pongSimulation";
 
 type Client = {
@@ -46,6 +47,7 @@ type Room = {
   npcUser: PublicUser | null;
   aiTargetY: number;
   simulation: PongSimulationState;
+  aiController: PongAi | null;
 };
 
 const INITIAL_BALL_VELOCITY = { x: 10, y: 5 };
@@ -290,6 +292,7 @@ export class GameHub {
       npcUser,
       aiTargetY: GAME_HEIGHT / 2,
       simulation,
+      aiController: options.ai ? new PongAi(roomId, npcUser?.rating ?? 1200) : null,
       snapshot: {
         roomId,
         phase: "waiting",
@@ -375,12 +378,12 @@ export class GameHub {
 
   private async tick(room: Room): Promise<void> {
     if (room.snapshot.phase !== "playing") return;
-    if (room.ai) {
-      updateAiPaddleIntent(room);
-    }
+    const rightDirection = room.aiController
+      ? room.aiController.nextDirection(room.simulation)
+      : room.snapshot.paddles.right.dy;
     room.simulation = PongSimulation.step(room.simulation, {
       left: room.snapshot.paddles.left.dy,
-      right: room.snapshot.paddles.right.dy
+      right: rightDirection
     }, SIMULATION_TIMESTEP_MS);
     syncSnapshot(room);
     this.broadcastRoom(room.id, { type: "game.snapshot", snapshot: room.snapshot });
