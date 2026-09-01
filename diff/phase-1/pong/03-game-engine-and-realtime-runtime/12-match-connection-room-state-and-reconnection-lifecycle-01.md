# 경기 연결·방 상태·재접속 수명 주기

## `feat(game): 실시간 경기 방 초기화`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index ee37c9c..fc6070c 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -2,7 +2,16 @@ import type { IncomingMessage } from "node:http";
 import { randomUUID } from "node:crypto";
 import { WebSocket } from "ws";
 import type { AppRepository } from "@pong-pong/db";
-import { encodeServerEvent, type ServerEvent, type SessionUser } from "@pong-pong/shared";
+import {
+  GAME_HEIGHT,
+  GAME_WIDTH,
+  PADDLE_HEIGHT,
+  encodeServerEvent,
+  type GameSnapshot,
+  type PlayerSide,
+  type ServerEvent,
+  type SessionUser
+} from "@pong-pong/shared";
 
 type Client = {
   id: string;
@@ -11,8 +20,16 @@ type Client = {
   roomId: string | null;
 };
 
+type Room = {
+  id: string;
+  clients: Partial<Record<PlayerSide, Client>>;
+  ai: boolean;
+  snapshot: GameSnapshot;
+};
+
 export class GameHub {
   private readonly clients = new Map<string, Client>();
+  private readonly rooms = new Map<string, Room>();
 
   constructor(private readonly repo: AppRepository) {}
 
@@ -25,17 +42,74 @@ export class GameHub {
 
   private disconnect(client: Client): void {
     this.clients.delete(client.id);
+    if (client.roomId) {
+      const room = this.rooms.get(client.roomId);
+      if (room) {
+        for (const participant of Object.values(room.clients)) {
+          if (participant) participant.roomId = null;
+        }
+        this.rooms.delete(room.id);
+      }
+    }
+    this.broadcastPresence();
+  }
+
+  private createRoom(left: Client, right: Client | null, ai: boolean): void {
+    const roomId = randomUUID();
+    const room: Room = {
+      id: roomId,
+      clients: { left, ...(right ? { right } : {}) },
+      ai,
+      snapshot: {
+        roomId,
+        phase: "waiting",
+        tick: 0,
+        leftScore: 0,
+        rightScore: 0,
+        paddles: {
+          left: { y: GAME_HEIGHT / 2 - PADDLE_HEIGHT / 2, dy: 0 },
+          right: { y: GAME_HEIGHT / 2 - PADDLE_HEIGHT / 2, dy: 0 }
+        },
+        ball: { position: { x: GAME_WIDTH / 2, y: GAME_HEIGHT / 2 }, velocity: { x: 7, y: 4 } },
+        players: [
+          { id: left.user.id, handle: left.user.handle, displayName: left.user.displayName, side: "left", ready: false, ai: false },
+          {
+            id: right?.user.id ?? "ai-opponent",
+            handle: right?.user.handle ?? "ai",
+            displayName: right?.user.displayName ?? "연습 AI",
+            side: "right",
+            ready: ai,
+            ai
+          }
+        ],
+        serverTime: new Date().toISOString()
+      }
+    };
+    this.rooms.set(roomId, room);
+    left.roomId = roomId;
+    if (right) right.roomId = roomId;
+    this.send(left, { type: "queue.matched", roomId, side: "left", opponent: right?.user.displayName ?? "연습 AI" });
+    if (right) this.send(right, { type: "queue.matched", roomId, side: "right", opponent: left.user.displayName });
+    this.broadcastRoom(roomId, { type: "game.snapshot", snapshot: room.snapshot });
     this.broadcastPresence();
   }
 
   private broadcastPresence(): void {
-    this.broadcastAll({ type: "presence.changed", online: this.clients.size, playing: 0 });
+    this.broadcastAll({ type: "presence.changed", online: this.clients.size, playing: this.rooms.size * 2 });
   }
 
   private broadcastAll(event: ServerEvent): void {
     for (const client of this.clients.values()) this.send(client, event);
   }
 
+  private broadcastRoom(roomId: string, event: ServerEvent): void {
+    const room = this.rooms.get(roomId);
+    if (!room) return;
+    for (const client of Object.values(room.clients)) {
+      if (client) this.send(client, event);
+    }
+  }
+
   private send(client: Client, event: ServerEvent): void {
     if (client.socket.readyState === WebSocket.OPEN) client.socket.send(encodeServerEvent(event));
   }


## `feat(game): 서버 주도 일시정지 기능 추가`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index 87b0805..ceb68ca 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -64,6 +64,8 @@ export class GameHub {
       if (event.type === "queue.join") this.joinQueue(client, event.mode);
       if (event.type === "queue.leave") this.leaveQueue(client);
       if (event.type === "game.ready") this.markReady(client, event.roomId);
+      if (event.type === "game.pause") this.pauseRoom(client, event.roomId);
+      if (event.type === "game.resume") this.resumeRoom(client, event.roomId);
       if (event.type === "game.input") this.applyInput(client, event.roomId, event.direction);
       if (event.type === "chat.send") {
         const message = await this.repo.createChatMessage({
@@ -226,11 +228,32 @@ export class GameHub {
 
   private applyInput(client: Client, roomId: string, direction: -1 | 0 | 1): void {
     const room = this.rooms.get(roomId);
-    if (!room || room.snapshot.phase === "finished") return;
+    if (!room || room.snapshot.phase !== "playing") return;
     const side = sideFor(room, client);
     if (side) room.snapshot.paddles[side].dy = direction;
   }
 
+  private pauseRoom(client: Client, roomId: string): void {
+    const room = this.rooms.get(roomId);
+    if (!room || room.snapshot.phase !== "playing" || !sideFor(room, client)) return;
+    if (room.timer) clearInterval(room.timer);
+    room.timer = null;
+    room.snapshot.phase = "paused";
+    room.snapshot.serverTime = new Date().toISOString();
+    this.broadcastRoom(roomId, { type: "game.snapshot", snapshot: room.snapshot });
+  }
+
+  private resumeRoom(client: Client, roomId: string): void {
+    const room = this.rooms.get(roomId);
+    if (!room || room.snapshot.phase !== "paused" || !sideFor(room, client)) return;
+    room.snapshot.phase = "playing";
+    room.snapshot.serverTime = new Date().toISOString();
+    if (!room.timer) {
+      room.timer = setInterval(() => this.tick(room).catch(() => undefined), 1000 / TICK_RATE);
+    }
+    this.broadcastRoom(roomId, { type: "game.snapshot", snapshot: room.snapshot });
+  }
+
   private async tick(room: Room): Promise<void> {
     const state = room.snapshot;
     if (state.phase !== "playing") return;


## `refactor(game): 게임 방 상태 전이 모델링`

diff --git a/apps/api/src/game/roomSession.ts b/apps/api/src/game/roomSession.ts
new file mode 100644
index 0000000..4104378
--- /dev/null
+++ b/apps/api/src/game/roomSession.ts
@@ -0,0 +1,107 @@
+import type { PlayerSide } from "@pong-pong/shared";
+
+export type RoomSessionState =
+  | "waiting"
+  | "playing"
+  | "paused"
+  | "reconnecting"
+  | "finished";
+
+export interface ReconnectExpiry {
+  forfeitingSide: PlayerSide | null;
+  winnerSide: PlayerSide | null;
+}
+
+const RECONNECT_WINDOW_MS = 15_000;
+
+export class RoomSession {
+  private currentState: RoomSessionState = "waiting";
+  private resumeState: Exclude<RoomSessionState, "reconnecting" | "finished"> = "waiting";
+  private readonly ready = new Set<PlayerSide>();
+  private readonly disconnected = new Set<PlayerSide>();
+  private reconnectDeadlineMs: number | null = null;
+
+  get state(): RoomSessionState {
+    return this.currentState;
+  }
+
+  get reconnectDeadline(): number | null {
+    return this.reconnectDeadlineMs;
+  }
+
+  markReady(side: PlayerSide): RoomSessionState {
+    if (this.currentState !== "waiting") return this.currentState;
+    this.ready.add(side);
+    if (this.ready.size === 2) this.currentState = "playing";
+    return this.currentState;
+  }
+
+  pause(): RoomSessionState {
+    if (this.currentState === "playing") this.currentState = "paused";
+    return this.currentState;
+  }
+
+  resume(): RoomSessionState {
+    if (this.currentState === "paused") this.currentState = "playing";
+    return this.currentState;
+  }
+
+  disconnect(side: PlayerSide, nowMs: number): RoomSessionState {
+    if (this.currentState === "finished") return this.currentState;
+    if (this.currentState !== "reconnecting") {
+      this.resumeState = this.currentState;
+    }
+    this.disconnected.add(side);
+    this.reconnectDeadlineMs = nowMs + RECONNECT_WINDOW_MS;
+    this.currentState = "reconnecting";
+    return this.currentState;
+  }
+
+  reconnect(side: PlayerSide, nowMs: number): boolean {
+    if (
+      this.currentState !== "reconnecting" ||
+      this.reconnectDeadlineMs === null ||
+      nowMs > this.reconnectDeadlineMs ||
+      !this.disconnected.has(side)
+    ) {
+      return false;
+    }
+
+    this.disconnected.delete(side);
+    if (this.disconnected.size === 0) {
+      this.currentState = this.resumeState;
+      this.reconnectDeadlineMs = null;
+    }
+    return true;
+  }
+
+  expireReconnect(nowMs: number): ReconnectExpiry | null {
+    if (
+      this.currentState !== "reconnecting" ||
+      this.reconnectDeadlineMs === null ||
+      nowMs < this.reconnectDeadlineMs
+    ) {
+      return null;
+    }
+
+    const [firstDisconnected] = this.disconnected;
+    const bothDisconnected = this.disconnected.size !== 1;
+    const forfeitingSide = bothDisconnected ? null : firstDisconnected ?? null;
+    this.finish();
+    return {
+      forfeitingSide,
+      winnerSide: forfeitingSide ? opposite(forfeitingSide) : null
+    };
+  }
+
+  finish(): RoomSessionState {
+    this.currentState = "finished";
+    this.disconnected.clear();
+    this.reconnectDeadlineMs = null;
+    return this.currentState;
+  }
+}
+
+function opposite(side: PlayerSide): PlayerSide {
+  return side === "left" ? "right" : "left";
+}


## `test(game): 게임 방 상태 전이 검증`

diff --git a/apps/api/src/game/roomSession.test.ts b/apps/api/src/game/roomSession.test.ts
new file mode 100644
index 0000000..6a5e9bc
--- /dev/null
+++ b/apps/api/src/game/roomSession.test.ts
@@ -0,0 +1,68 @@
+import { describe, expect, it } from "vitest";
+import { RoomSession } from "./roomSession";
+
+describe("RoomSession", () => {
+  it("starts only after both sides are ready", () => {
+    const session = new RoomSession();
+    expect(session.state).toBe("waiting");
+    expect(session.markReady("left")).toBe("waiting");
+    expect(session.markReady("right")).toBe("playing");
+  });
+
+  it("allows pause and resume only from the matching state", () => {
+    const session = playingSession();
+    expect(session.pause()).toBe("paused");
+    expect(session.pause()).toBe("paused");
+    expect(session.resume()).toBe("playing");
+  });
+
+  it("restores the previous state when a side reconnects within 15 seconds", () => {
+    const session = playingSession();
+    session.pause();
+    expect(session.disconnect("left", 1_000)).toBe("reconnecting");
+    expect(session.reconnectDeadline).toBe(16_000);
+
+    expect(session.reconnect("left", 15_999)).toBe(true);
+    expect(session.state).toBe("paused");
+    expect(session.reconnectDeadline).toBeNull();
+  });
+
+  it("rejects a reconnect after the deadline", () => {
+    const session = playingSession();
+    session.disconnect("right", 5_000);
+
+    expect(session.reconnect("right", 20_001)).toBe(false);
+    expect(session.state).toBe("reconnecting");
+  });
+
+  it("turns one missing side into a single forfeit result", () => {
+    const session = playingSession();
+    session.disconnect("right", 10_000);
+
+    expect(session.expireReconnect(24_999)).toBeNull();
+    expect(session.expireReconnect(25_000)).toEqual({
+      forfeitingSide: "right",
+      winnerSide: "left"
+    });
+    expect(session.state).toBe("finished");
+    expect(session.expireReconnect(30_000)).toBeNull();
+  });
+
+  it("does not select a winner when both sides disconnect", () => {
+    const session = playingSession();
+    session.disconnect("left", 0);
+    session.disconnect("right", 1_000);
+
+    expect(session.expireReconnect(16_000)).toEqual({
+      forfeitingSide: null,
+      winnerSide: null
+    });
+  });
+});
+
+function playingSession(): RoomSession {
+  const session = new RoomSession();
+  session.markReady("left");
+  session.markReady("right");
+  return session;
+}


## `refactor(web): game connection 상태 reducer 분리`

diff --git a/apps/web/src/game/gameConnection.ts b/apps/web/src/game/gameConnection.ts
new file mode 100644
index 0000000..1bc57a8
--- /dev/null
+++ b/apps/web/src/game/gameConnection.ts
@@ -0,0 +1,50 @@
+import type { GameSnapshot } from "@pong-pong/shared";
+
+export type GameConnectionStatus =
+  | "idle"
+  | "connecting"
+  | "matching"
+  | "waitingReady"
+  | "playing"
+  | "paused"
+  | "reconnecting"
+  | "finished"
+  | "failed";
+
+export type GameConnectionState = {
+  status: GameConnectionStatus;
+  roomId: string | null;
+  opponent: string | null;
+  snapshot: GameSnapshot | null;
+  lastSnapshotSequence: number;
+  notice: string;
+  messages: string[];
+};
+
+export const initialGameConnectionState: GameConnectionState = {
+  status: "idle",
+  roomId: null,
+  opponent: null,
+  snapshot: null,
+  lastSnapshotSequence: -1,
+  notice: "대기 중",
+  messages: []
+};
+
+export type GameConnectionAction =
+  | { type: "connectStarted" }
+  | { type: "socketOpened"; notice: string }
+  | { type: "matched"; roomId: string; opponent: string }
+  | { type: "snapshotReceived"; snapshot: GameSnapshot }
+  | { type: "gameFinished"; result: { leftScore: number; rightScore: number } }
+  | { type: "chatReceived"; message: string }
+  | { type: "readySent" }
+  | { type: "socketClosed" }
+  | { type: "failed"; notice?: string };
+
+export function gameConnectionReducer(
+  state: GameConnectionState,
+  _action: GameConnectionAction
+): GameConnectionState {
+  return state;
+}


## `refactor(web): GameSocketClient 연결 수명주기 분리`

diff --git a/apps/web/src/game/GameSocketClient.ts b/apps/web/src/game/GameSocketClient.ts
new file mode 100644
index 0000000..ef60cf9
--- /dev/null
+++ b/apps/web/src/game/GameSocketClient.ts
@@ -0,0 +1,72 @@
+import {
+  parseServerEvent,
+  type ClientEvent,
+  type ServerEvent,
+  type WsTicketResponse
+} from "@pong-pong/shared";
+
+export interface GameWebSocket {
+  readyState: number;
+  send(payload: string): void;
+  close(): void;
+  onopen: (() => void) | null;
+  onmessage: ((event: { data: unknown }) => void) | null;
+  onclose: (() => void) | null;
+  onerror: (() => void) | null;
+}
+
+export interface GameSocketHandlers {
+  onConnecting(): void;
+  onOpen(): void;
+  onEvent(event: ServerEvent): void;
+  onClosed(): void;
+  onFailure(error: unknown): void;
+}
+
+type GameSocketClientOptions = {
+  url: string;
+  ticketProvider(signal?: AbortSignal): Promise<WsTicketResponse>;
+  socketFactory(url: string): GameWebSocket;
+};
+
+const CONNECTING = 0;
+const OPEN = 1;
+
+export class GameSocketClient {
+  private socket: GameWebSocket | null = null;
+  private ticketRequest: AbortController | null = null;
+  private generation = 0;
+  private inputSequence = 0;
+
+  constructor(private readonly options: GameSocketClientOptions) {}
+
+  close(): void {
+    this.replaceConnection();
+  }
+
+  private replaceConnection(): number {
+    this.generation += 1;
+    this.ticketRequest?.abort();
+    this.ticketRequest = null;
+
+    const socket = this.socket;
+    this.socket = null;
+    if (socket) {
+      socket.onopen = null;
+      socket.onmessage = null;
+      socket.onclose = null;
+      socket.onerror = null;
+      if (socket.readyState === CONNECTING || socket.readyState === OPEN) socket.close();
+    }
+    this.inputSequence = 0;
+    return this.generation;
+  }
+
+  private isCurrent(socket: GameWebSocket, generation: number): boolean {
+    return this.socket === socket && this.generation === generation;
+  }
+}
+
+function isAbortError(error: unknown): boolean {
+  return typeof error === "object" && error !== null && "name" in error && error.name === "AbortError";
+}


## `refactor(web): game connection hook 상태 연결`

diff --git a/apps/web/src/game/useGameConnection.ts b/apps/web/src/game/useGameConnection.ts
new file mode 100644
index 0000000..30548bb
--- /dev/null
+++ b/apps/web/src/game/useGameConnection.ts
@@ -0,0 +1,76 @@
+"use client";
+
+import { useCallback, useEffect, useMemo, useReducer } from "react";
+import type { ClientEvent, ServerEvent } from "@pong-pong/shared";
+import { requestWsTicket } from "@/lib/api";
+import { GameSocketClient, type GameSocketHandlers, type GameWebSocket } from "./GameSocketClient";
+import { gameConnectionReducer, initialGameConnectionState } from "./gameConnection";
+
+const WS_URL = process.env.NEXT_PUBLIC_WS_URL ?? "ws://localhost:4000/ws";
+
+export function useGameConnection() {
+  const [state, dispatch] = useReducer(gameConnectionReducer, initialGameConnectionState);
+  const client = useMemo(() => new GameSocketClient({
+    url: WS_URL,
+    ticketProvider: requestWsTicket,
+    socketFactory: (url) => new WebSocket(url) as unknown as GameWebSocket
+  }), []);
+
+  const handleEvent = useCallback((event: ServerEvent) => {
+    switch (event.type) {
+      case "queue.matched":
+        dispatch({ type: "matched", roomId: event.roomId, opponent: event.opponent });
+        return;
+      case "game.snapshot":
+        dispatch({ type: "snapshotReceived", snapshot: event.snapshot });
+        return;
+      case "game.finished":
+        dispatch({ type: "gameFinished", result: event.result });
+        return;
+      case "chat.message":
+        dispatch({
+          type: "chatReceived",
+          message: `${event.message.sender.displayName}: ${event.message.body}`
+        });
+        return;
+      case "error":
+        dispatch({ type: "failed", notice: event.message });
+        return;
+      case "presence.changed":
+        return;
+    }
+  }, []);
+
+  const connect = useCallback(async (initialEvent: ClientEvent, openNotice: string) => {
+    const handlers: GameSocketHandlers = {
+      onConnecting: () => dispatch({ type: "connectStarted" }),
+      onOpen: () => dispatch({ type: "socketOpened", notice: openNotice }),
+      onEvent: handleEvent,
+      onClosed: () => dispatch({ type: "socketClosed" }),
+      onFailure: (error) => dispatch({ type: "failed", notice: failureMessage(error) })
+    };
+    await client.connect(initialEvent, handlers);
+  }, [client, handleEvent]);
+
+  const connectQueue = useCallback((mode: "queue" | "ai") => connect(
+    { v: 1, type: "queue.join", mode },
+    mode === "ai" ? "인공지능 연습 방 생성 중" : "매칭 큐 참가 중"
+  ), [connect]);
+
+  const connectTournament = useCallback((matchId: string) => connect(
+    { v: 1, type: "tournament.join", matchId },
+    "토너먼트 경기 상대 입장 대기 중"
+  ), [connect]);
+
+  useEffect(() => () => client.close(), [client]);
+
+  return { state, connectQueue, connectTournament };
+}
+
+function failureMessage(error: unknown): string {
+  if (typeof error === "object" && error !== null && "status" in error && error.status === 401) {
+    return "로그인 후 이용할 수 있습니다.";
+  }
+  if (error instanceof Error && error.message) return error.message;
+  return "실시간 연결을 확인해 주세요.";
+}


## `refactor(play): 자동 경기 진입을 connection hook으로 전환`

diff --git a/apps/web/src/app/play/page.tsx b/apps/web/src/app/play/page.tsx
index 8ff5d6e..cc6cd9f 100644
--- a/apps/web/src/app/play/page.tsx
+++ b/apps/web/src/app/play/page.tsx
@@ -44,19 +44,14 @@ export default function PlayPage() {
     const mode = params.get("mode");
     if (tournamentMatchId) {
       autoStartedRef.current = true;
-      connectTournament(tournamentMatchId);
+      void connection.connectTournament(tournamentMatchId);
       return;
     }
-    if (mode === "ai") {
+    if (mode === "ai" || mode === "queue") {
       autoStartedRef.current = true;
-      connect("ai");
-      return;
-    }
-    if (mode === "queue") {
-      autoStartedRef.current = true;
-      connect("queue");
+      void connection.connectQueue(mode);
     }
-  }, []);
+  }, [connection.connectQueue, connection.connectTournament]);
 
   useEffect(() => {
     const handleKey = (event: KeyboardEvent) => {


