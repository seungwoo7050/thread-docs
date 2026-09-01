# 고정 스텝 실행과 스냅샷 백프레셔·보간

## `refactor(web): PongCanvas snapshot state 렌더링`

diff --git a/apps/web/src/components/PongCanvas.tsx b/apps/web/src/components/PongCanvas.tsx
index 433b30b..1039edb 100644
--- a/apps/web/src/components/PongCanvas.tsx
+++ b/apps/web/src/components/PongCanvas.tsx
@@ -64,19 +64,19 @@ function drawSnapshot(ctx: CanvasRenderingContext2D, snapshot: GameSnapshot) {
   ctx.stroke();
   ctx.setLineDash([]);
 
-  drawPaddle(ctx, 32, snapshot.paddles.left.y, "#1768f2");
-  drawPaddle(ctx, GAME_WIDTH - 50, snapshot.paddles.right.y, "#12b76a");
+  drawPaddle(ctx, 32, snapshot.state.paddles.left.y, "#1768f2");
+  drawPaddle(ctx, GAME_WIDTH - 50, snapshot.state.paddles.right.y, "#12b76a");
   ctx.beginPath();
   ctx.fillStyle = "#26364f";
-  ctx.arc(snapshot.ball.position.x, snapshot.ball.position.y, BALL_RADIUS, 0, Math.PI * 2);
+  ctx.arc(snapshot.state.ball.position.x, snapshot.state.ball.position.y, BALL_RADIUS, 0, Math.PI * 2);
   ctx.fill();
 
   ctx.fillStyle = "#1768f2";
   ctx.font = "bold 42px system-ui";
   ctx.textAlign = "center";
-  ctx.fillText(String(snapshot.leftScore), GAME_WIDTH / 2 - 46, 68);
+  ctx.fillText(String(snapshot.state.leftScore), GAME_WIDTH / 2 - 46, 68);
   ctx.fillStyle = "#12b76a";
-  ctx.fillText(String(snapshot.rightScore), GAME_WIDTH / 2 + 46, 68);
+  ctx.fillText(String(snapshot.state.rightScore), GAME_WIDTH / 2 + 46, 68);
 }
 
 function drawPaddle(ctx: CanvasRenderingContext2D, x: number, y: number, color: string) {
@@ -88,15 +88,18 @@ function drawPaddle(ctx: CanvasRenderingContext2D, x: number, y: number, color:
 function toRenderSample(snapshot: GameSnapshot): RenderSample {
   return {
     ...snapshot,
-    paddles: {
-      left: { ...snapshot.paddles.left },
-      right: { ...snapshot.paddles.right }
+    state: {
+      ...snapshot.state,
+      paddles: {
+        left: { ...snapshot.state.paddles.left },
+        right: { ...snapshot.state.paddles.right }
+      },
+      ball: {
+        position: { ...snapshot.state.ball.position },
+        velocity: { ...snapshot.state.ball.velocity }
+      },
+      players: snapshot.state.players.map((player) => ({ ...player }))
     },
-    ball: {
-      position: { ...snapshot.ball.position },
-      velocity: { ...snapshot.ball.velocity }
-    },
-    players: snapshot.players.map((player) => ({ ...player })),
     receivedAt: performance.now()
   };
 }
@@ -104,20 +107,23 @@ function toRenderSample(snapshot: GameSnapshot): RenderSample {
 function emptySnapshot(): GameSnapshot {
   return {
     roomId: "",
-    phase: "waiting",
     tick: 0,
-    leftScore: 0,
-    rightScore: 0,
-    paddles: {
-      left: { y: GAME_HEIGHT / 2 - PADDLE_HEIGHT / 2, dy: 0 },
-      right: { y: GAME_HEIGHT / 2 - PADDLE_HEIGHT / 2, dy: 0 }
-    },
-    ball: {
-      position: { x: GAME_WIDTH / 2, y: GAME_HEIGHT / 2 },
-      velocity: { x: 0, y: 0 }
-    },
-    players: [],
-    serverTime: new Date(0).toISOString()
+    sequence: 0,
+    serverTimeMs: 0,
+    state: {
+      phase: "waiting",
+      leftScore: 0,
+      rightScore: 0,
+      paddles: {
+        left: { y: GAME_HEIGHT / 2 - PADDLE_HEIGHT / 2, dy: 0 },
+        right: { y: GAME_HEIGHT / 2 - PADDLE_HEIGHT / 2, dy: 0 }
+      },
+      ball: {
+        position: { x: GAME_WIDTH / 2, y: GAME_HEIGHT / 2 },
+        velocity: { x: 0, y: 0 }
+      },
+      players: []
+    }
   };
 }
 
@@ -140,15 +146,24 @@ function interpolateSnapshot(previous: RenderSample, next: RenderSample, ratio:
   const mix = (from: number, to: number) => from + (to - from) * ratio;
   return {
     ...next,
-    paddles: {
-      left: { ...next.paddles.left, y: mix(previous.paddles.left.y, next.paddles.left.y) },
-      right: { ...next.paddles.right, y: mix(previous.paddles.right.y, next.paddles.right.y) }
-    },
-    ball: {
-      ...next.ball,
-      position: {
-        x: mix(previous.ball.position.x, next.ball.position.x),
-        y: mix(previous.ball.position.y, next.ball.position.y)
+    state: {
+      ...next.state,
+      paddles: {
+        left: {
+          ...next.state.paddles.left,
+          y: mix(previous.state.paddles.left.y, next.state.paddles.left.y)
+        },
+        right: {
+          ...next.state.paddles.right,
+          y: mix(previous.state.paddles.right.y, next.state.paddles.right.y)
+        }
+      },
+      ball: {
+        ...next.state.ball,
+        position: {
+          x: mix(previous.state.ball.position.x, next.state.ball.position.x),
+          y: mix(previous.state.ball.position.y, next.state.ball.position.y)
+        }
       }
     }
   };


## `feat(game): fixed-step scheduler 추가`

diff --git a/apps/api/src/game/fixedStepScheduler.ts b/apps/api/src/game/fixedStepScheduler.ts
new file mode 100644
index 0000000..41b92a7
--- /dev/null
+++ b/apps/api/src/game/fixedStepScheduler.ts
@@ -0,0 +1,111 @@
+export const DEFAULT_TIMESTEP_MS = 50;
+export const DEFAULT_MAX_TICKS_PER_LOOP = 5;
+export const DEFAULT_MAX_ACCUMULATED_MS = 250;
+
+type AccumulatorOptions = {
+  initialTimeMs: number;
+  timestepMs?: number;
+  maxTicksPerLoop?: number;
+  maxAccumulatedMs?: number;
+};
+
+export class FixedStepAccumulator {
+  private readonly timestepMs: number;
+  private readonly maxTicksPerLoop: number;
+  private readonly maxAccumulatedMs: number;
+  private previousTimeMs: number;
+  private lagMs = 0;
+
+  constructor(options: AccumulatorOptions) {
+    this.timestepMs = options.timestepMs ?? DEFAULT_TIMESTEP_MS;
+    this.maxTicksPerLoop = options.maxTicksPerLoop ?? DEFAULT_MAX_TICKS_PER_LOOP;
+    this.maxAccumulatedMs = options.maxAccumulatedMs ?? DEFAULT_MAX_ACCUMULATED_MS;
+    assertPositiveFinite(this.timestepMs, "timestepMs");
+    assertPositiveInteger(this.maxTicksPerLoop, "maxTicksPerLoop");
+    assertPositiveFinite(this.maxAccumulatedMs, "maxAccumulatedMs");
+    if (this.maxAccumulatedMs < this.timestepMs) {
+      throw new RangeError("maxAccumulatedMs must be at least one timestep");
+    }
+    this.previousTimeMs = options.initialTimeMs;
+  }
+
+  get accumulatedMs(): number {
+    return this.lagMs;
+  }
+
+  advance(nowMs: number): number {
+    if (!Number.isFinite(nowMs)) return 0;
+    const elapsedMs = Math.max(0, nowMs - this.previousTimeMs);
+    if (nowMs > this.previousTimeMs) this.previousTimeMs = nowMs;
+    this.lagMs = Math.min(this.maxAccumulatedMs, this.lagMs + elapsedMs);
+
+    const availableTicks = Math.floor(this.lagMs / this.timestepMs);
+    const ticks = Math.min(this.maxTicksPerLoop, availableTicks);
+    this.lagMs -= ticks * this.timestepMs;
+    return ticks;
+  }
+}
+
+type SchedulerOptions = {
+  now?: () => number;
+  timestepMs?: number;
+  maxTicksPerLoop?: number;
+  maxAccumulatedMs?: number;
+};
+
+export class FixedStepScheduler {
+  private readonly now: () => number;
+  private readonly timestepMs: number;
+  private readonly maxTicksPerLoop: number;
+  private readonly maxAccumulatedMs: number;
+  private timer: ReturnType<typeof setInterval> | null = null;
+  private accumulator: FixedStepAccumulator | null = null;
+
+  constructor(private readonly step: () => void, options: SchedulerOptions = {}) {
+    this.now = options.now ?? (() => performance.now());
+    this.timestepMs = options.timestepMs ?? DEFAULT_TIMESTEP_MS;
+    this.maxTicksPerLoop = options.maxTicksPerLoop ?? DEFAULT_MAX_TICKS_PER_LOOP;
+    this.maxAccumulatedMs = options.maxAccumulatedMs ?? DEFAULT_MAX_ACCUMULATED_MS;
+  }
+
+  get running(): boolean {
+    return this.timer !== null;
+  }
+
+  start(): void {
+    if (this.timer) return;
+    this.accumulator = new FixedStepAccumulator({
+      initialTimeMs: this.now(),
+      timestepMs: this.timestepMs,
+      maxTicksPerLoop: this.maxTicksPerLoop,
+      maxAccumulatedMs: this.maxAccumulatedMs
+    });
+    this.timer = setInterval(() => this.runLoop(), this.timestepMs);
+  }
+
+  stop(): void {
+    if (this.timer) clearInterval(this.timer);
+    this.timer = null;
+    this.accumulator = null;
+  }
+
+  private runLoop(): void {
+    if (!this.timer || !this.accumulator) return;
+    const ticks = this.accumulator.advance(this.now());
+    for (let tick = 0; tick < ticks && this.timer; tick += 1) {
+      this.step();
+    }
+  }
+}
+
+function assertPositiveFinite(value: number, name: string): void {
+  if (!Number.isFinite(value) || value <= 0) {
+    throw new RangeError(`${name} must be positive`);
+  }
+}
+
+function assertPositiveInteger(value: number, name: string): void {
+  if (!Number.isInteger(value) || value <= 0) {
+    throw new RangeError(`${name} must be a positive integer`);
+  }
+}


## `test(game): fixed-step 보정 범위 검증`

diff --git a/apps/api/src/game/fixedStepScheduler.test.ts b/apps/api/src/game/fixedStepScheduler.test.ts
new file mode 100644
index 0000000..5c584cf
--- /dev/null
+++ b/apps/api/src/game/fixedStepScheduler.test.ts
@@ -0,0 +1,59 @@
+import { describe, expect, it, vi } from "vitest";
+import { FixedStepAccumulator, FixedStepScheduler } from "./fixedStepScheduler";
+
+describe("FixedStepAccumulator", () => {
+  it("turns elapsed monotonic time into fixed fifty millisecond steps", () => {
+    const accumulator = new FixedStepAccumulator({
+      initialTimeMs: 1_000,
+      timestepMs: 50,
+      maxTicksPerLoop: 5,
+      maxAccumulatedMs: 250
+    });
+
+    expect(accumulator.advance(1_049)).toBe(0);
+    expect(accumulator.advance(1_050)).toBe(1);
+    expect(accumulator.advance(1_149)).toBe(1);
+    expect(accumulator.advance(1_150)).toBe(1);
+  });
+
+  it("caps catch-up work at five ticks and accumulated lag at 250ms", () => {
+    const accumulator = new FixedStepAccumulator({
+      initialTimeMs: 0,
+      timestepMs: 50,
+      maxTicksPerLoop: 5,
+      maxAccumulatedMs: 250
+    });
+
+    expect(accumulator.advance(10_000)).toBe(5);
+    expect(accumulator.accumulatedMs).toBe(0);
+    expect(accumulator.advance(10_049)).toBe(0);
+    expect(accumulator.advance(10_050)).toBe(1);
+  });
+
+  it("ignores a clock that moves backwards", () => {
+    const accumulator = new FixedStepAccumulator({ initialTimeMs: 100 });
+
+    expect(accumulator.advance(80)).toBe(0);
+    expect(accumulator.advance(150)).toBe(1);
+  });
+});
+
+describe("FixedStepScheduler", () => {
+  it("uses the injected monotonic clock and stops stepping after stop", () => {
+    vi.useFakeTimers();
+    let nowMs = 0;
+    const step = vi.fn();
+    const scheduler = new FixedStepScheduler(step, { now: () => nowMs });
+
+    scheduler.start();
+    nowMs = 250;
+    vi.advanceTimersByTime(50);
+    expect(step).toHaveBeenCalledTimes(5);
+
+    scheduler.stop();
+    nowMs = 500;
+    vi.advanceTimersByTime(500);
+    expect(step).toHaveBeenCalledTimes(5);
+    vi.useRealTimers();
+  });
+});


## `feat(game): latest snapshot buffer 추가`

diff --git a/apps/api/src/game/latestSnapshotBuffer.ts b/apps/api/src/game/latestSnapshotBuffer.ts
new file mode 100644
index 0000000..1da83e7
--- /dev/null
+++ b/apps/api/src/game/latestSnapshotBuffer.ts
@@ -0,0 +1,106 @@
+export const SOFT_BUFFERED_AMOUNT_BYTES = 256 * 1_024;
+export const HARD_BUFFERED_AMOUNT_BYTES = 1_024 * 1_024;
+export const MAX_CONGESTION_MS = 5_000;
+const RETRY_INTERVAL_MS = 50;
+const SOCKET_OPEN = 1;
+
+export type SnapshotSocket = {
+  readyState: number;
+  bufferedAmount: number;
+  send: (payload: string, callback: (error?: Error) => void) => void;
+  terminate: () => void;
+};
+
+type SnapshotBufferOptions = {
+  now?: () => number;
+};
+
+export class LatestSnapshotBuffer {
+  private readonly now: () => number;
+  private pendingSnapshot: string | null = null;
+  private sending = false;
+  private congestionStartedAtMs: number | null = null;
+  private retryTimer: ReturnType<typeof setTimeout> | null = null;
+  private closed = false;
+
+  constructor(private readonly socket: SnapshotSocket, options: SnapshotBufferOptions = {}) {
+    this.now = options.now ?? (() => performance.now());
+  }
+
+  enqueue(payload: string): void {
+    if (this.closed) return;
+    this.pendingSnapshot = payload;
+    this.drain();
+  }
+
+  close(): void {
+    this.closed = true;
+    this.pendingSnapshot = null;
+    if (this.retryTimer) clearTimeout(this.retryTimer);
+    this.retryTimer = null;
+  }
+
+  private drain(): void {
+    if (this.closed) return;
+    if (this.socket.readyState !== SOCKET_OPEN) {
+      this.close();
+      return;
+    }
+
+    const nowMs = this.now();
+    if (this.socket.bufferedAmount >= HARD_BUFFERED_AMOUNT_BYTES) {
+      this.terminate();
+      return;
+    }
+
+    if (this.socket.bufferedAmount > SOFT_BUFFERED_AMOUNT_BYTES) {
+      this.congestionStartedAtMs ??= nowMs;
+      if (nowMs - this.congestionStartedAtMs >= MAX_CONGESTION_MS) {
+        this.terminate();
+        return;
+      }
+      this.armRetry();
+      return;
+    }
+
+    this.congestionStartedAtMs = null;
+    if (this.sending) {
+      this.armRetry();
+      return;
+    }
+
+    const payload = this.pendingSnapshot;
+    if (payload === null) return;
+    this.pendingSnapshot = null;
+    this.sending = true;
+    try {
+      this.socket.send(payload, (error) => {
+        this.sending = false;
+        if (error) {
+          this.terminate();
+          return;
+        }
+        this.drain();
+      });
+    } catch {
+      this.sending = false;
+      this.terminate();
+      return;
+    }
+    if (this.sending || this.pendingSnapshot !== null) this.armRetry();
+  }
+
+  private armRetry(): void {
+    if (this.retryTimer || this.closed) return;
+    this.retryTimer = setTimeout(() => {
+      this.retryTimer = null;
+      this.drain();
+    }, RETRY_INTERVAL_MS);
+  }
+
+  private terminate(): void {
+    if (this.closed) return;
+    this.close();
+    this.socket.terminate();
+  }
+}


## `test(game): snapshot replacement와 congestion 검증`

diff --git a/apps/api/src/game/latestSnapshotBuffer.test.ts b/apps/api/src/game/latestSnapshotBuffer.test.ts
new file mode 100644
index 0000000..45f21ca
--- /dev/null
+++ b/apps/api/src/game/latestSnapshotBuffer.test.ts
@@ -0,0 +1,94 @@
+import { afterEach, describe, expect, it, vi } from "vitest";
+import {
+  HARD_BUFFERED_AMOUNT_BYTES,
+  LatestSnapshotBuffer,
+  SOFT_BUFFERED_AMOUNT_BYTES,
+  type SnapshotSocket
+} from "./latestSnapshotBuffer";
+
+describe("LatestSnapshotBuffer", () => {
+  afterEach(() => vi.useRealTimers());
+
+  it("keeps only the newest snapshot while a send is in flight", () => {
+    const socket = fakeSocket();
+    const buffer = new LatestSnapshotBuffer(socket);
+
+    buffer.enqueue("snapshot-1");
+    buffer.enqueue("snapshot-2");
+    buffer.enqueue("snapshot-3");
+
+    expect(socket.sent).toEqual(["snapshot-1"]);
+    socket.completeSend();
+    expect(socket.sent).toEqual(["snapshot-1", "snapshot-3"]);
+    socket.completeSend();
+    buffer.close();
+  });
+
+  it("replaces congested snapshots and sends the latest after pressure clears", () => {
+    vi.useFakeTimers();
+    const socket = fakeSocket();
+    socket.bufferedAmount = SOFT_BUFFERED_AMOUNT_BYTES + 1;
+    const buffer = new LatestSnapshotBuffer(socket);
+
+    buffer.enqueue("snapshot-1");
+    buffer.enqueue("snapshot-2");
+    expect(socket.sent).toEqual([]);
+
+    socket.bufferedAmount = 0;
+    vi.advanceTimersByTime(50);
+    expect(socket.sent).toEqual(["snapshot-2"]);
+  });
+
+  it("terminates immediately at one MiB of buffered data", () => {
+    const socket = fakeSocket();
+    socket.bufferedAmount = HARD_BUFFERED_AMOUNT_BYTES;
+    const buffer = new LatestSnapshotBuffer(socket);
+
+    buffer.enqueue("snapshot");
+
+    expect(socket.terminate).toHaveBeenCalledTimes(1);
+    expect(socket.sent).toEqual([]);
+  });
+
+  it("terminates after five seconds above the soft limit", () => {
+    vi.useFakeTimers();
+    let nowMs = 0;
+    const socket = fakeSocket();
+    socket.bufferedAmount = SOFT_BUFFERED_AMOUNT_BYTES + 1;
+    const buffer = new LatestSnapshotBuffer(socket, { now: () => nowMs });
+
+    buffer.enqueue("snapshot");
+    nowMs = 4_999;
+    vi.advanceTimersByTime(4_999);
+    expect(socket.terminate).not.toHaveBeenCalled();
+
+    nowMs = 5_000;
+    vi.advanceTimersByTime(1);
+    expect(socket.terminate).toHaveBeenCalledTimes(1);
+  });
+});
+
+type FakeSnapshotSocket = SnapshotSocket & {
+  sent: string[];
+  completeSend: (error?: Error) => void;
+  terminate: ReturnType<typeof vi.fn>;
+};
+
+function fakeSocket(): FakeSnapshotSocket {
+  let completion: ((error?: Error) => void) | null = null;
+  return {
+    readyState: 1,
+    bufferedAmount: 0,
+    sent: [],
+    send(payload, callback) {
+      this.sent.push(payload);
+      completion = callback;
+    },
+    completeSend(error) {
+      const callback = completion;
+      completion = null;
+      callback?.(error);
+    },
+    terminate: vi.fn()
+  };
+}


## `feat(game): fixed-step scheduler를 GameHub에 연결`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index fea4d82..096adb7 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -13,6 +13,10 @@ import {
   type ServerEvent,
   type SessionUser
 } from "@pong-pong/shared";
+import { DEFAULT_TIMESTEP_MS, FixedStepScheduler } from "./game/fixedStepScheduler";
+import { ConnectionHeartbeat } from "./game/heartbeat";
+import { InputGate } from "./game/inputGate";
+import { HARD_BUFFERED_AMOUNT_BYTES, LatestSnapshotBuffer } from "./game/latestSnapshotBuffer";
 import { PongAi } from "./game/pongAi";
 import { PongSimulation, type PongSimulationState } from "./game/pongSimulation";
 
@@ -42,7 +46,7 @@ type Room = {
   ai: boolean;
   ready: Partial<Record<PlayerSide, boolean>>;
   snapshot: GameSnapshot;
-  timer: NodeJS.Timeout | null;
+  scheduler: FixedStepScheduler | null;
   mode: MatchMode;
   tournamentMatchId: string | null;
   npcUser: PublicUser | null;
@@ -52,7 +56,7 @@ type Room = {
 };
 
 const NPC_QUEUE_FALLBACK_MS = 6000;
-const SIMULATION_TIMESTEP_MS = 50;
+const SIMULATION_TIMESTEP_MS = DEFAULT_TIMESTEP_MS;
 
 export class GameHub {
   private readonly clients = new Map<string, Client>();
@@ -290,7 +294,7 @@ export class GameHub {
       clients: { left, ...(right ? { right } : {}) },
       ai: options.ai,
       ready: {},
-      timer: null,
+      scheduler: null,
       mode: options.mode,
       tournamentMatchId: options.tournamentMatchId ?? null,
       npcUser,
@@ -348,9 +352,9 @@ export class GameHub {
       if (player.side === side) player.ready = true;
     }
     if (room.ai) room.ready.right = true;
-    if (room.ready.left && room.ready.right && !room.timer) {
+    if (room.ready.left && room.ready.right && room.snapshot.state.phase === "waiting") {
       room.snapshot.state.phase = "playing";
-      room.timer = setInterval(() => this.tick(room).catch(() => undefined), SIMULATION_TIMESTEP_MS);
+      this.startRoomScheduler(room);
     }
     this.broadcastSnapshot(room);
   }
@@ -369,8 +373,7 @@ export class GameHub {
   private pauseRoom(client: Client, roomId: string): void {
     const room = this.rooms.get(roomId);
     if (!room || room.snapshot.state.phase !== "playing" || !sideFor(room, client)) return;
-    if (room.timer) clearInterval(room.timer);
-    room.timer = null;
+    room.scheduler?.stop();
     room.snapshot.state.phase = "paused";
     this.broadcastSnapshot(room);
   }
@@ -379,13 +382,20 @@ export class GameHub {
     const room = this.rooms.get(roomId);
     if (!room || room.snapshot.state.phase !== "paused" || !sideFor(room, client)) return;
     room.snapshot.state.phase = "playing";
-    if (!room.timer) {
-      room.timer = setInterval(() => this.tick(room).catch(() => undefined), SIMULATION_TIMESTEP_MS);
-    }
+    this.startRoomScheduler(room);
     this.broadcastSnapshot(room);
   }
 
-  private async tick(room: Room): Promise<void> {
+  private startRoomScheduler(room: Room): void {
+    room.scheduler ??= new FixedStepScheduler(() => this.tick(room), {
+      timestepMs: SIMULATION_TIMESTEP_MS,
+      maxTicksPerLoop: 5,
+      maxAccumulatedMs: 250
+    });
+    room.scheduler.start();
+  }
+
+  private tick(room: Room): void {
     if (room.snapshot.state.phase !== "playing") return;
     const rightDirection = room.aiController
       ? room.aiController.nextDirection(room.simulation)
@@ -398,7 +408,7 @@ export class GameHub {
     this.broadcastSnapshot(room);
 
     if (room.simulation.phase === "finished" && room.simulation.winnerSide) {
-      await this.finishRoom(room, room.simulation.winnerSide);
+      this.finishRoom(room, room.simulation.winnerSide).catch(() => undefined);
     }
   }
 
@@ -413,8 +423,7 @@ export class GameHub {
   }
 
   private async finalizeRoom(room: Room, winnerSide: PlayerSide): Promise<void> {
-    if (room.timer) clearInterval(room.timer);
-    room.timer = null;
+    room.scheduler?.stop();
     room.snapshot.state.phase = "finished";
     const leftUser = room.clients.left?.user ?? null;
     const rightUser = room.clients.right?.user ?? room.npcUser ?? null;


