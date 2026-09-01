## `test(web): game connection lifecycle 검증`

diff --git a/apps/web/src/game/GameSocketClient.test.ts b/apps/web/src/game/GameSocketClient.test.ts
new file mode 100644
index 0000000..84398f9
--- /dev/null
+++ b/apps/web/src/game/GameSocketClient.test.ts
@@ -0,0 +1,138 @@
+import { describe, expect, it, vi } from "vitest";
+import type { WsTicketResponse } from "@pong-pong/shared";
+import {
+  GameSocketClient,
+  type GameSocketHandlers,
+  type GameWebSocket
+} from "./GameSocketClient";
+
+const ticket = {
+  ticket: "a".repeat(43),
+  expiresInSeconds: 30,
+  protocolVersion: 1
+} satisfies WsTicketResponse;
+
+describe("GameSocketClient", () => {
+  it("cancels an unused one-time ticket request before starting another connection", async () => {
+    const signals: AbortSignal[] = [];
+    const ticketProvider = vi.fn((signal?: AbortSignal) => new Promise<WsTicketResponse>((resolve, reject) => {
+      if (!signal) throw new Error("AbortSignal is required");
+      signals.push(signal);
+      signal.addEventListener("abort", () => reject(new DOMException("Aborted", "AbortError")));
+      if (signals.length === 2) resolve(ticket);
+    }));
+    const sockets: FakeSocket[] = [];
+    const client = new GameSocketClient({
+      url: "ws://localhost:4000/ws",
+      ticketProvider,
+      socketFactory: (url) => {
+        const socket = new FakeSocket(url);
+        sockets.push(socket);
+        return socket;
+      }
+    });
+
+    const first = client.connect({ v: 1, type: "queue.join", mode: "queue" }, handlers());
+    const second = client.connect({ v: 1, type: "queue.join", mode: "ai" }, handlers());
+    await Promise.all([first, second]);
+
+    expect(signals).toHaveLength(2);
+    expect(signals[0].aborted).toBe(true);
+    expect(sockets).toHaveLength(1);
+    expect(sockets[0].url).toBe(`ws://localhost:4000/ws?ticket=${ticket.ticket}&v=1`);
+  });
+
+  it("parses versioned server events through the shared protocol parser", async () => {
+    const onEvent = vi.fn();
+    const onFailure = vi.fn();
+    const socket = new FakeSocket("");
+    const client = new GameSocketClient({
+      url: "ws://localhost:4000/ws",
+      ticketProvider: async () => ticket,
+      socketFactory: (url) => {
+        socket.url = url;
+        return socket;
+      }
+    });
+
+    await client.connect(
+      { v: 1, type: "queue.join", mode: "queue" },
+      handlers({ onEvent, onFailure })
+    );
+    socket.open();
+    socket.message(JSON.stringify({
+      v: 1,
+      type: "queue.matched",
+      roomId: "room-1",
+      side: "left",
+      opponent: "상대 선수"
+    }));
+    socket.message(JSON.stringify({ type: "queue.matched", roomId: "room-2" }));
+
+    expect(JSON.parse(socket.sent[0])).toEqual({ v: 1, type: "queue.join", mode: "queue" });
+    expect(onEvent).toHaveBeenCalledOnce();
+    expect(onEvent).toHaveBeenCalledWith(expect.objectContaining({ v: 1, type: "queue.matched" }));
+    expect(onFailure).toHaveBeenCalledOnce();
+  });
+
+  it("assigns a strictly increasing input sequence to every direction command", async () => {
+    const socket = new FakeSocket("");
+    const client = new GameSocketClient({
+      url: "ws://localhost:4000/ws",
+      ticketProvider: async () => ticket,
+      socketFactory: () => socket
+    });
+
+    await client.connect({ v: 1, type: "queue.join", mode: "queue" }, handlers());
+    socket.open();
+    expect(client.sendDirection("room-1", -1)).toBe(1);
+    expect(client.sendDirection("room-1", 0)).toBe(2);
+    expect(client.sendDirection("room-1", 1)).toBe(3);
+
+    expect(socket.sent.slice(1).map((payload) => JSON.parse(payload))).toEqual([
+      { v: 1, type: "game.input", roomId: "room-1", inputSeq: 1, direction: -1 },
+      { v: 1, type: "game.input", roomId: "room-1", inputSeq: 2, direction: 0 },
+      { v: 1, type: "game.input", roomId: "room-1", inputSeq: 3, direction: 1 }
+    ]);
+  });
+});
+
+class FakeSocket implements GameWebSocket {
+  readyState = 0;
+  sent: string[] = [];
+  onopen: (() => void) | null = null;
+  onmessage: ((event: { data: unknown }) => void) | null = null;
+  onclose: (() => void) | null = null;
+  onerror: (() => void) | null = null;
+
+  constructor(public url: string) {}
+
+  send(payload: string): void {
+    this.sent.push(payload);
+  }
+
+  close(): void {
+    this.readyState = 3;
+    this.onclose?.();
+  }
+
+  open(): void {
+    this.readyState = 1;
+    this.onopen?.();
+  }
+
+  message(data: unknown): void {
+    this.onmessage?.({ data });
+  }
+}
+
+function handlers(overrides: Partial<GameSocketHandlers> = {}): GameSocketHandlers {
+  return {
+    onConnecting: vi.fn(),
+    onOpen: vi.fn(),
+    onEvent: vi.fn(),
+    onClosed: vi.fn(),
+    onFailure: vi.fn(),
+    ...overrides
+  };
+}
diff --git a/apps/web/src/game/gameConnection.test.ts b/apps/web/src/game/gameConnection.test.ts
new file mode 100644
index 0000000..95041d8
--- /dev/null
+++ b/apps/web/src/game/gameConnection.test.ts
@@ -0,0 +1,131 @@
+import { describe, expect, it } from "vitest";
+import type { GameSnapshot } from "@pong-pong/shared";
+import {
+  gameConnectionReducer,
+  initialGameConnectionState,
+  type GameConnectionStatus
+} from "./gameConnection";
+
+const allowedStatuses = new Set<GameConnectionStatus>([
+  "idle",
+  "connecting",
+  "matching",
+  "waitingReady",
+  "playing",
+  "paused",
+  "reconnecting",
+  "finished",
+  "failed"
+]);
+
+describe("gameConnectionReducer", () => {
+  it("keeps connection state inside the explicit lifecycle", () => {
+    const connecting = gameConnectionReducer(initialGameConnectionState, { type: "connectStarted" });
+    const matching = gameConnectionReducer(connecting, {
+      type: "socketOpened",
+      notice: "매칭 큐 참가 중"
+    });
+    const waiting = gameConnectionReducer(matching, {
+      type: "matched",
+      roomId: "room-1",
+      opponent: "상대 선수"
+    });
+    const playing = gameConnectionReducer(waiting, {
+      type: "snapshotReceived",
+      snapshot: snapshot(1, "playing")
+    });
+    const paused = gameConnectionReducer(playing, {
+      type: "snapshotReceived",
+      snapshot: snapshot(2, "paused")
+    });
+    const reconnecting = gameConnectionReducer(paused, { type: "socketClosed" });
+    const failed = gameConnectionReducer(initialGameConnectionState, { type: "socketClosed" });
+    const finished = gameConnectionReducer(playing, {
+      type: "gameFinished",
+      result: { leftScore: 3, rightScore: 1 }
+    });
+
+    for (const state of [connecting, matching, waiting, playing, paused, reconnecting, failed, finished]) {
+      expect(allowedStatuses.has(state.status)).toBe(true);
+    }
+    expect([connecting.status, matching.status, waiting.status, playing.status, paused.status]).toEqual([
+      "connecting",
+      "matching",
+      "waitingReady",
+      "playing",
+      "paused"
+    ]);
+    expect(reconnecting).toMatchObject({ status: "reconnecting", roomId: "room-1" });
+    expect(failed.status).toBe("failed");
+    expect(finished).toMatchObject({ status: "finished", notice: "경기 종료: 3 - 1" });
+  });
+
+  it("discards snapshots whose sequence is not newer than the accepted snapshot", () => {
+    const matched = gameConnectionReducer(initialGameConnectionState, {
+      type: "matched",
+      roomId: "room-1",
+      opponent: "상대 선수"
+    });
+    const current = gameConnectionReducer(matched, {
+      type: "snapshotReceived",
+      snapshot: snapshot(7, "playing")
+    });
+
+    expect(gameConnectionReducer(current, {
+      type: "snapshotReceived",
+      snapshot: snapshot(7, "paused")
+    })).toBe(current);
+    expect(gameConnectionReducer(current, {
+      type: "snapshotReceived",
+      snapshot: snapshot(6, "paused")
+    })).toBe(current);
+
+    const next = gameConnectionReducer(current, {
+      type: "snapshotReceived",
+      snapshot: snapshot(8, "paused")
+    });
+    expect(next).toMatchObject({ status: "paused", lastSnapshotSequence: 8 });
+  });
+
+  it("clears per-room sequence state before a new connection attempt", () => {
+    const current = gameConnectionReducer({
+      ...initialGameConnectionState,
+      status: "playing",
+      roomId: "room-1",
+      snapshot: snapshot(12, "playing"),
+      lastSnapshotSequence: 12,
+      messages: ["이전 메시지"]
+    }, { type: "connectStarted" });
+
+    expect(current).toMatchObject({
+      status: "connecting",
+      roomId: null,
+      snapshot: null,
+      lastSnapshotSequence: -1,
+      messages: []
+    });
+  });
+});
+
+function snapshot(sequence: number, phase: GameSnapshot["state"]["phase"]): GameSnapshot {
+  return {
+    roomId: "room-1",
+    tick: sequence,
+    sequence,
+    serverTimeMs: sequence * 50,
+    state: {
+      phase,
+      leftScore: 0,
+      rightScore: 0,
+      paddles: {
+        left: { y: 100, dy: 0 },
+        right: { y: 100, dy: 0 }
+      },
+      ball: {
+        position: { x: 480, y: 270 },
+        velocity: { x: 0, y: 0 }
+      },
+      players: []
+    }
+  };
+}
diff --git a/apps/web/src/game/gameInput.test.ts b/apps/web/src/game/gameInput.test.ts
new file mode 100644
index 0000000..a12efd5
--- /dev/null
+++ b/apps/web/src/game/gameInput.test.ts
@@ -0,0 +1,25 @@
+import { describe, expect, it } from "vitest";
+import { directionForKey, isEditableTarget } from "./gameInput";
+
+describe("game input", () => {
+  it.each([
+    ["ArrowUp", -1],
+    ["w", -1],
+    ["W", -1],
+    ["ArrowDown", 1],
+    ["s", 1],
+    ["S", 1],
+    ["Enter", null]
+  ] as const)("maps %s to the shared direction command", (key, direction) => {
+    expect(directionForKey(key)).toBe(direction);
+  });
+
+  it.each(["INPUT", "TEXTAREA", "SELECT"])("ignores keyboard input from %s controls", (tagName) => {
+    expect(isEditableTarget({ tagName, isContentEditable: false })).toBe(true);
+  });
+
+  it("ignores keyboard input from contenteditable elements", () => {
+    expect(isEditableTarget({ tagName: "DIV", isContentEditable: true })).toBe(true);
+    expect(isEditableTarget({ tagName: "DIV", isContentEditable: false })).toBe(false);
+  });
+});


## `feat(game): WebSocket heartbeat 추가`

diff --git a/apps/api/src/game/heartbeat.ts b/apps/api/src/game/heartbeat.ts
new file mode 100644
index 0000000..038186f
--- /dev/null
+++ b/apps/api/src/game/heartbeat.ts
@@ -0,0 +1,48 @@
+export const HEARTBEAT_PING_INTERVAL_MS = 15_000;
+export const HEARTBEAT_TIMEOUT_MS = 45_000;
+
+type HeartbeatTarget = {
+  ping: () => void;
+  terminate: () => void;
+};
+
+export class ConnectionHeartbeat {
+  private pingTimer: ReturnType<typeof setInterval> | null = null;
+  private timeoutTimer: ReturnType<typeof setTimeout> | null = null;
+
+  constructor(private readonly target: HeartbeatTarget) {}
+
+  start(): void {
+    if (this.pingTimer || this.timeoutTimer) return;
+    this.armTimeout();
+    this.pingTimer = setInterval(() => {
+      try {
+        this.target.ping();
+      } catch {
+        this.terminate();
+      }
+    }, HEARTBEAT_PING_INTERVAL_MS);
+  }
+
+  acknowledge(): void {
+    if (!this.pingTimer && !this.timeoutTimer) return;
+    this.armTimeout();
+  }
+
+  stop(): void {
+    if (this.pingTimer) clearInterval(this.pingTimer);
+    if (this.timeoutTimer) clearTimeout(this.timeoutTimer);
+    this.pingTimer = null;
+    this.timeoutTimer = null;
+  }
+
+  private armTimeout(): void {
+    if (this.timeoutTimer) clearTimeout(this.timeoutTimer);
+    this.timeoutTimer = setTimeout(() => this.terminate(), HEARTBEAT_TIMEOUT_MS);
+  }
+
+  private terminate(): void {
+    this.stop();
+    this.target.terminate();
+  }
+}


## `feat(game): 게임 방 상태를 RoomSession에 연결`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index 05a3e5f..e3434da 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -11,7 +11,8 @@ import {
   type PlayerSide,
   type PublicUser,
   type ServerEvent,
-  type SessionUser
+  type SessionUser,
+  WINNING_SCORE
 } from "@pong-pong/shared";
 import { DEFAULT_TIMESTEP_MS, FixedStepScheduler } from "./game/fixedStepScheduler";
 import { ConnectionHeartbeat } from "./game/heartbeat";
@@ -19,6 +20,7 @@ import { InputGate } from "./game/inputGate";
 import { HARD_BUFFERED_AMOUNT_BYTES, LatestSnapshotBuffer } from "./game/latestSnapshotBuffer";
 import { PongAi } from "./game/pongAi";
 import { PongSimulation, type PongSimulationState } from "./game/pongSimulation";
+import { RoomSession } from "./game/roomSession";
 
 type Client = {
   id: string;
@@ -54,6 +56,9 @@ type Room = {
   simulation: PongSimulationState;
   aiController: PongAi | null;
   finishing: Promise<void> | null;
+  session: RoomSession;
+  reconnectTimer: NodeJS.Timeout | null;
+  disconnectedUsers: Partial<Record<PlayerSide, string>>;
 };
 
 const NPC_QUEUE_FALLBACK_MS = 6000;
@@ -306,6 +311,8 @@ export class GameHub {
     const npcUser = options.npc ?? null;
     const rightPlayer = right?.user ?? npcUser;
     const simulation = PongSimulation.initialState();
+    const session = new RoomSession();
+    if (options.ai) session.markReady("right");
     const room: Room = {
       id: roomId,
       clients: { left, ...(right ? { right } : {}) },
@@ -318,6 +325,9 @@ export class GameHub {
       simulation,
       aiController: options.ai ? new PongAi(roomId, npcUser?.rating ?? 1200) : null,
       finishing: null,
+      session,
+      reconnectTimer: null,
+      disconnectedUsers: {},
       snapshot: {
         roomId,
         tick: 0,
@@ -369,8 +379,9 @@ export class GameHub {
       if (player.side === side) player.ready = true;
     }
     if (room.ai) room.ready.right = true;
-    if (room.ready.left && room.ready.right && room.snapshot.state.phase === "waiting") {
-      room.snapshot.state.phase = "playing";
+    const sessionState = room.session.markReady(side);
+    if (room.ready.left && room.ready.right && sessionState === "playing") {
+      room.snapshot.state.phase = sessionState;
       this.startRoomScheduler(room);
     }
     this.broadcastSnapshot(room);
@@ -403,14 +414,18 @@ export class GameHub {
     const room = this.rooms.get(roomId);
     if (!room || room.snapshot.state.phase !== "playing" || !sideFor(room, client)) return;
     room.scheduler?.stop();
-    room.snapshot.state.phase = "paused";
+    const sessionState = room.session.pause();
+    if (sessionState !== "paused") return;
+    room.snapshot.state.phase = sessionState;
     this.broadcastSnapshot(room);
   }
 
   private resumeRoom(client: Client, roomId: string): void {
     const room = this.rooms.get(roomId);
     if (!room || room.snapshot.state.phase !== "paused" || !sideFor(room, client)) return;
-    room.snapshot.state.phase = "playing";
+    const sessionState = room.session.resume();
+    if (sessionState !== "playing") return;
+    room.snapshot.state.phase = sessionState;
     this.startRoomScheduler(room);
     this.broadcastSnapshot(room);
   }


## `feat(game): 사용자별 active connection 교체`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index e3434da..704ed11 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -63,9 +63,12 @@ type Room = {
 
 const NPC_QUEUE_FALLBACK_MS = 6000;
 const SIMULATION_TIMESTEP_MS = DEFAULT_TIMESTEP_MS;
+const CONNECTION_REPLACED_CLOSE_CODE = 4001;
+const CONNECTION_REPLACED_REASON = "connection replaced";
 
 export class GameHub {
   private readonly clients = new Map<string, Client>();
+  private readonly clientsByUser = new Map<string, Client>();
   private readonly queue: QueueEntry[] = [];
   private readonly rooms = new Map<string, Room>();
   private readonly tournamentWaiters = new Map<string, Client[]>();
@@ -89,11 +92,16 @@ export class GameHub {
       heartbeat,
       snapshots: new LatestSnapshotBuffer(socket)
     };
+    const previous = this.clientsByUser.get(user.id);
     this.clients.set(client.id, client);
+    this.clientsByUser.set(user.id, client);
     socket.on("message", (payload) => this.receive(client, payload.toString()));
     socket.on("pong", () => heartbeat.acknowledge());
     socket.on("close", () => this.disconnect(client));
-    heartbeat.start();
+    if (previous) this.replaceConnection(previous, client);
+    if (this.clients.get(client.id) === client && socket.readyState === WebSocket.OPEN) {
+      heartbeat.start();
+    }
     this.broadcastPresence();
     for (const payload of pendingPayloads) {
       this.receive(client, payload).catch(() => undefined);
@@ -101,6 +109,7 @@ export class GameHub {
   }
 
   private async receive(client: Client, payload: string): Promise<void> {
+    if (this.clients.get(client.id) !== client) return;
     try {
       const event = parseClientEvent(payload);
       if (event.type === "queue.join") await this.joinQueue(client, event.mode);
@@ -139,9 +148,10 @@ export class GameHub {
     this.leaveQueue(client);
     this.leaveTournamentWaiters(client);
     this.clients.delete(client.id);
-    if (![...this.clients.values()].some((candidate) => candidate.user.id === client.user.id)) {
-      this.inputGate.releaseUser(client.user.id);
+    if (this.clientsByUser.get(client.user.id)?.id === client.id) {
+      this.clientsByUser.delete(client.user.id);
     }
+    this.inputGate.releaseUser(client.user.id);
     if (client.roomId) {
       const room = this.rooms.get(client.roomId);
       if (room) {
@@ -151,6 +161,38 @@ export class GameHub {
     this.broadcastPresence();
   }
 
+  private replaceConnection(previous: Client, replacement: Client): void {
+    previous.heartbeat.stop();
+    previous.snapshots.close();
+    this.leaveQueue(previous);
+    this.leaveTournamentWaiters(previous);
+    this.clients.delete(previous.id);
+    this.inputGate.releaseUser(previous.user.id);
+
+    if (previous.roomId) {
+      const room = this.rooms.get(previous.roomId);
+      const side = room ? sideFor(room, previous) : null;
+      if (room && side) {
+        room.clients[side] = replacement;
+        replacement.roomId = room.id;
+        previous.roomId = null;
+        this.sendMatchContext(replacement, room, side);
+        this.send(replacement, { type: "game.snapshot", snapshot: room.snapshot });
+      }
+    }
+
+    if (previous.socket.readyState === WebSocket.OPEN) {
+      previous.socket.close(CONNECTION_REPLACED_CLOSE_CODE, CONNECTION_REPLACED_REASON);
+    }
+  }
+
+  private sendMatchContext(client: Client, room: Room, side: PlayerSide): void {
+    const opponent = side === "left"
+      ? room.clients.right?.user.displayName ?? room.npcUser?.displayName ?? "연습 AI"
+      : room.clients.left?.user.displayName ?? "상대 선수";
+    this.send(client, { type: "queue.matched", roomId: room.id, side, opponent });
+  }
+
   private async joinQueue(client: Client, mode: "queue" | "ai"): Promise<void> {
     this.leaveQueue(client);
     this.pruneQueue();


## `feat(game): 예약된 room connection 복구`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index 704ed11..80ca5b8 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -99,6 +99,7 @@ export class GameHub {
     socket.on("pong", () => heartbeat.acknowledge());
     socket.on("close", () => this.disconnect(client));
     if (previous) this.replaceConnection(previous, client);
+    else this.recoverConnection(client);
     if (this.clients.get(client.id) === client && socket.readyState === WebSocket.OPEN) {
       heartbeat.start();
     }
@@ -186,6 +187,39 @@ export class GameHub {
     }
   }
 
+  private recoverConnection(client: Client): boolean {
+    const nowMs = Date.now();
+    for (const room of this.rooms.values()) {
+      for (const side of ["left", "right"] as const) {
+        if (room.disconnectedUsers[side] !== client.user.id) continue;
+        if (!room.session.reconnect(side, nowMs)) continue;
+
+        const disconnected = room.clients[side];
+        if (disconnected) disconnected.roomId = null;
+        room.clients[side] = client;
+        client.roomId = room.id;
+        delete room.disconnectedUsers[side];
+        this.sendMatchContext(client, room, side);
+
+        if (room.session.state === "reconnecting") {
+          this.send(client, { type: "game.snapshot", snapshot: room.snapshot });
+        } else {
+          this.clearReconnectTimer(room);
+          room.snapshot.state.phase = room.session.state;
+          if (room.session.state === "playing") this.startRoomScheduler(room);
+          this.broadcastSnapshot(room);
+        }
+        return true;
+      }
+    }
+    return false;
+  }
+
+  private clearReconnectTimer(room: Room): void {
+    if (room.reconnectTimer) clearTimeout(room.reconnectTimer);
+    room.reconnectTimer = null;
+  }
+
   private sendMatchContext(client: Client, room: Room, side: PlayerSide): void {
     const opponent = side === "left"
       ? room.clients.right?.user.displayName ?? room.npcUser?.displayName ?? "연습 AI"


