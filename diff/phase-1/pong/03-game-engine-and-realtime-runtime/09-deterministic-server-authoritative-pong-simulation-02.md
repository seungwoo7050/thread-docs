## `refactor(game): GameHub frame 계산을 simulation에 위임`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index 0e27218..cafaac9 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -52,6 +52,7 @@ const INITIAL_BALL_VELOCITY = { x: 10, y: 5 };
 const BALL_ACCELERATION_PER_TICK = 0.015;
 const MAX_BALL_SPEED = 18;
 const NPC_QUEUE_FALLBACK_MS = 6000;
+const SIMULATION_TIMESTEP_MS = 50;
 
 type AiProfile = {
   reactionTicks: number;
@@ -339,7 +340,7 @@ export class GameHub {
     if (room.ai) room.ready.right = true;
     if (room.ready.left && room.ready.right && !room.timer) {
       room.snapshot.phase = "playing";
-      room.timer = setInterval(() => this.tick(room).catch(() => undefined), 1000 / TICK_RATE);
+      room.timer = setInterval(() => this.tick(room).catch(() => undefined), SIMULATION_TIMESTEP_MS);
     }
     this.broadcastRoom(roomId, { type: "game.snapshot", snapshot: room.snapshot });
   }
@@ -367,46 +368,25 @@ export class GameHub {
     room.snapshot.phase = "playing";
     room.snapshot.serverTime = new Date().toISOString();
     if (!room.timer) {
-      room.timer = setInterval(() => this.tick(room).catch(() => undefined), 1000 / TICK_RATE);
+      room.timer = setInterval(() => this.tick(room).catch(() => undefined), SIMULATION_TIMESTEP_MS);
     }
     this.broadcastRoom(roomId, { type: "game.snapshot", snapshot: room.snapshot });
   }
 
   private async tick(room: Room): Promise<void> {
-    const state = room.snapshot;
-    if (state.phase !== "playing") return;
-    state.tick += 1;
-    state.serverTime = new Date().toISOString();
-
-    const speed = 13;
-    state.paddles.left.y = clamp(state.paddles.left.y + state.paddles.left.dy * speed, 16, GAME_HEIGHT - PADDLE_HEIGHT - 16);
+    if (room.snapshot.phase !== "playing") return;
     if (room.ai) {
       updateAiPaddleIntent(room);
     }
-    const rightSpeed = room.ai ? speed * aiProfileFor(room.npcUser?.rating ?? 1200).paddleSpeedMultiplier : speed;
-    state.paddles.right.y = clamp(state.paddles.right.y + state.paddles.right.dy * rightSpeed, 16, GAME_HEIGHT - PADDLE_HEIGHT - 16);
-
-    state.ball.position.x += state.ball.velocity.x;
-    state.ball.position.y += state.ball.velocity.y;
-    if (state.ball.position.y < BALL_RADIUS || state.ball.position.y > GAME_HEIGHT - BALL_RADIUS) {
-      state.ball.velocity.y *= -1;
-    }
-    collidePaddle(state, "left", 32);
-    collidePaddle(state, "right", GAME_WIDTH - 32);
-
-    if (state.ball.position.x < 0) {
-      state.rightScore += 1;
-      resetBall(state, -1);
-    }
-    if (state.ball.position.x > GAME_WIDTH) {
-      state.leftScore += 1;
-      resetBall(state, 1);
-    }
-    accelerateBall(state);
-    this.broadcastRoom(room.id, { type: "game.snapshot", snapshot: state });
-
-    if (state.leftScore >= WINNING_SCORE || state.rightScore >= WINNING_SCORE || state.tick >= TICK_RATE * 45) {
-      await this.finishRoom(room, state.leftScore >= state.rightScore ? "left" : "right");
+    room.simulation = PongSimulation.step(room.simulation, {
+      left: room.snapshot.paddles.left.dy,
+      right: room.snapshot.paddles.right.dy
+    }, SIMULATION_TIMESTEP_MS);
+    syncSnapshot(room);
+    this.broadcastRoom(room.id, { type: "game.snapshot", snapshot: room.snapshot });
+
+    if (room.simulation.phase === "finished" && room.simulation.winnerSide) {
+      await this.finishRoom(room, room.simulation.winnerSide);
     }
   }
 
@@ -491,6 +471,26 @@ function clearQueueTimer(entry: QueueEntry): void {
   }
 }
 
+function syncSnapshot(room: Room): void {
+  const state = room.simulation;
+  room.snapshot.tick = state.tick;
+  room.snapshot.leftScore = state.leftScore;
+  room.snapshot.rightScore = state.rightScore;
+  room.snapshot.paddles.left = {
+    y: state.paddles.left.y,
+    dy: state.paddles.left.direction
+  };
+  room.snapshot.paddles.right = {
+    y: state.paddles.right.y,
+    dy: state.paddles.right.direction
+  };
+  room.snapshot.ball = {
+    position: { ...state.ball.position },
+    velocity: { ...state.ball.velocity }
+  };
+  room.snapshot.serverTime = new Date().toISOString();
+}
+
 function clamp(value: number, min: number, max: number): number {
   return Math.max(min, Math.min(max, value));
 }


## `test(game): versioned match replay fixture 추가`

diff --git a/apps/api/src/game/fixtures/replay-v1.json b/apps/api/src/game/fixtures/replay-v1.json
new file mode 100644
index 0000000..acdf1fd
--- /dev/null
+++ b/apps/api/src/game/fixtures/replay-v1.json
@@ -0,0 +1,54 @@
+{
+  "protocolVersion": 1,
+  "seed": "replay-seed-2026",
+  "timestepMs": 50,
+  "ticks": 1000,
+  "inputEncoding": {
+    "0": 0,
+    "-": -1,
+    "+": 1
+  },
+  "initialState": {
+    "tick": 0,
+    "phase": "playing",
+    "leftScore": 0,
+    "rightScore": 0,
+    "paddles": {
+      "left": {
+        "y": 214,
+        "direction": 0
+      },
+      "right": {
+        "y": 214,
+        "direction": 0
+      }
+    },
+    "ball": {
+      "position": {
+        "x": 480,
+        "y": 270
+      },
+      "velocity": {
+        "x": 10,
+        "y": 5
+      }
+    },
+    "winnerSide": null
+  },
+  "inputs": {
+    "left": "--------------------++++++++++++++++++++00000000000000000000--------------------++++++++++++++++++++00000000000000000000--------------------++++++++++++++++++++00000000000000000000--------------------++++++++++++++++++++00000000000000000000--------------------++++++++++++++++++++00000000000000000000--------------------++++++++++++++++++++00000000000000000000--------------------++++++++++++++++++++00000000000000000000--------------------++++++++++++++++++++00000000000000000000--------------------++++++++++++++++++++00000000000000000000--------------------++++++++++++++++++++00000000000000000000--------------------++++++++++++++++++++00000000000000000000--------------------++++++++++++++++++++00000000000000000000--------------------++++++++++++++++++++00000000000000000000--------------------++++++++++++++++++++00000000000000000000--------------------++++++++++++++++++++00000000000000000000--------------------++++++++++++++++++++00000000000000000000--------------------++++++++++++++++++++",
+    "right": [
+      "+++++++++++++++00000000000000000000000000000----------------0000+000-000000000000000++00--00+++0-000",
+      "-000++00-000-000+++++++000000000++00--000000++0000000000---00000++++---0++00-000-------0---00000+++0",
+      "-000-000++00-000+00000000000+000-0000000--00-000+000+++0--00--00++00--00000000000000+000++00----++++",
+      "0000-000++000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000",
+      "0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000",
+      "0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000",
+      "0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000",
+      "0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000",
+      "0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000",
+      "0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000"
+    ]
+  },
+  "finalHash": "f0a9d7a9f2453620dac1c8a718b001bb939b4a546ec1ca192da89f329d2a3f61"
+}
diff --git a/apps/api/src/game/replayFixture.test.ts b/apps/api/src/game/replayFixture.test.ts
new file mode 100644
index 0000000..70ce18c
--- /dev/null
+++ b/apps/api/src/game/replayFixture.test.ts
@@ -0,0 +1,64 @@
+import { createHash } from "node:crypto";
+import { readFileSync } from "node:fs";
+import { fileURLToPath } from "node:url";
+import { describe, expect, it } from "vitest";
+import {
+  PongSimulation,
+  type PongSimulationInputs,
+  type PongSimulationState
+} from "./pongSimulation";
+
+type InputCharacter = "-" | "0" | "+";
+
+interface ReplayFixture {
+  protocolVersion: 1;
+  seed: string;
+  timestepMs: number;
+  ticks: number;
+  inputEncoding: Record<InputCharacter, -1 | 0 | 1>;
+  initialState: PongSimulationState;
+  inputs: { left: string; right: string[] };
+  finalHash: string;
+}
+
+const fixture = JSON.parse(readFileSync(fileURLToPath(
+  new URL("./fixtures/replay-v1.json", import.meta.url)
+), "utf8")) as ReplayFixture;
+
+describe("versioned simulation replay fixture", () => {
+  it("records every input and reproduces the 1,000 tick final hash", () => {
+    expect(fixture).toMatchObject({
+      protocolVersion: 1,
+      seed: "replay-seed-2026",
+      timestepMs: 50,
+      ticks: 1_000,
+      initialState: PongSimulation.initialState()
+    });
+    expect(fixture.inputs.left).toHaveLength(fixture.ticks);
+    const rightInputs = fixture.inputs.right.join("");
+    expect(fixture.inputs.right).toHaveLength(10);
+    for (const segment of fixture.inputs.right) {
+      expect(segment).toMatch(/^[-+0]{100}$/);
+    }
+    expect(rightInputs).toHaveLength(fixture.ticks);
+
+    let state = structuredClone(fixture.initialState);
+    for (let tick = 0; tick < fixture.ticks; tick += 1) {
+      const inputs: PongSimulationInputs = {
+        left: decode(fixture.inputs.left[tick]),
+        right: decode(rightInputs[tick])
+      };
+      state = PongSimulation.step(state, inputs, fixture.timestepMs);
+    }
+
+    const hash = createHash("sha256").update(JSON.stringify(state)).digest("hex");
+    expect(hash).toBe(fixture.finalHash);
+  });
+});
+
+function decode(character: string | undefined): -1 | 0 | 1 {
+  if (character !== "-" && character !== "0" && character !== "+") {
+    throw new Error(`unknown replay input: ${character ?? "missing"}`);
+  }
+  return fixture.inputEncoding[character];
+}
