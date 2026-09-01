## `feat(game): reconnect 예약 만료와 room 정리`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index 80ca5b8..ff4ef3a 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -155,9 +155,8 @@ export class GameHub {
     this.inputGate.releaseUser(client.user.id);
     if (client.roomId) {
       const room = this.rooms.get(client.roomId);
-      if (room) {
-        this.finishRoom(room, client === room.clients.left ? "right" : "left").catch(() => undefined);
-      }
+      const side = room ? sideFor(room, client) : null;
+      if (room && side) this.reserveRoomSide(room, side, client.user.id);
     }
     this.broadcastPresence();
   }
@@ -215,6 +214,57 @@ export class GameHub {
     return false;
   }
 
+  private reserveRoomSide(room: Room, side: PlayerSide, userId: string): void {
+    if (room.finishing || room.session.state === "finished") return;
+    room.session.disconnect(side, Date.now());
+    room.disconnectedUsers[side] = userId;
+    room.scheduler?.stop();
+    room.snapshot.state.paddles[side].dy = 0;
+    room.snapshot.state.phase = "paused";
+    this.armReconnectTimer(room);
+    this.broadcastSnapshot(room);
+  }
+
+  private armReconnectTimer(room: Room): void {
+    this.clearReconnectTimer(room);
+    const deadline = room.session.reconnectDeadline;
+    if (deadline === null) return;
+    room.reconnectTimer = setTimeout(() => this.expireReconnect(room.id), Math.max(0, deadline - Date.now()));
+  }
+
+  private expireReconnect(roomId: string): void {
+    const room = this.rooms.get(roomId);
+    if (!room || room.finishing) return;
+    room.reconnectTimer = null;
+    const expiry = room.session.expireReconnect(Date.now());
+    if (!expiry) {
+      this.armReconnectTimer(room);
+      return;
+    }
+
+    room.disconnectedUsers = {};
+    if (!expiry.winnerSide) {
+      this.abandonRoom(room);
+      return;
+    }
+    if (expiry.winnerSide === "left") {
+      room.snapshot.state.leftScore = Math.max(room.snapshot.state.leftScore, WINNING_SCORE);
+    } else {
+      room.snapshot.state.rightScore = Math.max(room.snapshot.state.rightScore, WINNING_SCORE);
+    }
+    this.finishRoom(room, expiry.winnerSide).catch(() => undefined);
+  }
+
+  private abandonRoom(room: Room): void {
+    room.scheduler?.stop();
+    this.clearReconnectTimer(room);
+    for (const client of Object.values(room.clients)) {
+      if (client) client.roomId = null;
+    }
+    this.rooms.delete(room.id);
+    this.broadcastPresence();
+  }
+
   private clearReconnectTimer(room: Room): void {
     if (room.reconnectTimer) clearTimeout(room.reconnectTimer);
     room.reconnectTimer = null;
@@ -228,6 +278,10 @@ export class GameHub {
   }
 
   private async joinQueue(client: Client, mode: "queue" | "ai"): Promise<void> {
+    if (client.roomId) {
+      this.send(client, { type: "error", code: "forbidden", message: "이미 진행 중인 경기가 있습니다." });
+      return;
+    }
     this.leaveQueue(client);
     this.pruneQueue();
     if (mode === "ai") {
@@ -544,6 +598,9 @@ export class GameHub {
 
   private async finalizeRoom(room: Room, winnerSide: PlayerSide): Promise<void> {
     room.scheduler?.stop();
+    this.clearReconnectTimer(room);
+    room.disconnectedUsers = {};
+    room.session.finish();
     room.snapshot.state.phase = "finished";
     const leftUser = room.clients.left?.user ?? null;
     const rightUser = room.clients.right?.user ?? room.npcUser ?? null;


## `test(game): reconnect 복구 동작 검증`

diff --git a/apps/api/src/gameHub.reconnect.test.ts b/apps/api/src/gameHub.reconnect.test.ts
new file mode 100644
index 0000000..7f6f999
--- /dev/null
+++ b/apps/api/src/gameHub.reconnect.test.ts
@@ -0,0 +1,215 @@
+import { EventEmitter } from "node:events";
+import type { IncomingMessage } from "node:http";
+import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
+import { WebSocket } from "ws";
+import { createMemoryRepository } from "@pong-pong/db";
+import { parseServerEvent, type ServerEvent, type SessionUser } from "@pong-pong/shared";
+import { GameHub } from "./gameHub";
+
+describe("GameHub connection recovery", () => {
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
+  it("replaces an active connection without starting a forfeit timeout", async () => {
+    const { hub, finalizeMatch } = setup();
+    const first = connect(hub, player("left-user", "왼쪽 사용자"));
+    const roomId = await joinAiMatch(first);
+    const replacement = connect(hub, player("left-user", "왼쪽 사용자"));
+    await flushEvents();
+
+    expect(first.closed).toEqual({ code: 4001, reason: "connection replaced" });
+    expect(replacement.latest("queue.matched")).toEqual(
+      expect.objectContaining({ roomId, side: "left" })
+    );
+    expect(replacement.latest("game.snapshot")).toEqual(
+      expect.objectContaining({ snapshot: expect.objectContaining({ roomId }) })
+    );
+    first.receive({ v: 1, type: "queue.join", mode: "ai" });
+    await flushEvents();
+    expect(hub.liveStats().activeRooms).toBe(1);
+
+    await vi.advanceTimersByTimeAsync(15_001);
+    expect(finalizeMatch).not.toHaveBeenCalled();
+  });
+
+  it("restores the reserved side and sends the latest snapshot within 15 seconds", async () => {
+    const { hub, finalizeMatch } = setup();
+    const first = connect(hub, player("left-user", "왼쪽 사용자"));
+    const roomId = await joinAiMatch(first);
+    first.receive({ v: 1, type: "game.ready", roomId });
+    await vi.advanceTimersByTimeAsync(100);
+
+    const sequenceBeforeDisconnect = snapshotSequence(first);
+    first.terminate();
+    await flushEvents();
+    await vi.advanceTimersByTimeAsync(14_999);
+
+    const recovered = connect(hub, player("left-user", "왼쪽 사용자"));
+    await flushEvents();
+    const snapshot = recovered.latest("game.snapshot");
+
+    expect(recovered.latest("queue.matched")).toEqual(
+      expect.objectContaining({ roomId, side: "left" })
+    );
+    expect(snapshot).toEqual(
+      expect.objectContaining({
+        snapshot: expect.objectContaining({
+          roomId,
+          sequence: expect.any(Number),
+          state: expect.objectContaining({ phase: "playing" })
+        })
+      })
+    );
+    if (snapshot?.type !== "game.snapshot") throw new Error("expected recovered snapshot");
+    expect(snapshot.snapshot.sequence).toBeGreaterThan(sequenceBeforeDisconnect);
+
+    await vi.advanceTimersByTimeAsync(2);
+    expect(finalizeMatch).not.toHaveBeenCalled();
+  });
+
+  it("finalizes one forfeit when the reserved side does not reconnect", async () => {
+    const { hub, finalizeMatch } = setup();
+    const leftUser = player("left-user", "왼쪽 사용자");
+    const rightUser = player("right-user", "오른쪽 사용자");
+    const left = connect(hub, leftUser);
+    const right = connect(hub, rightUser);
+
+    left.receive({ v: 1, type: "queue.join", mode: "queue" });
+    await flushEvents();
+    right.receive({ v: 1, type: "queue.join", mode: "queue" });
+    await flushEvents();
+    const matched = right.latest("queue.matched");
+    if (matched?.type !== "queue.matched") throw new Error("expected a match");
+
+    left.receive({ v: 1, type: "game.ready", roomId: matched.roomId });
+    right.receive({ v: 1, type: "game.ready", roomId: matched.roomId });
+    await flushEvents();
+    left.terminate();
+
+    await vi.advanceTimersByTimeAsync(14_999);
+    expect(finalizeMatch).not.toHaveBeenCalled();
+    await vi.advanceTimersByTimeAsync(1);
+    await flushEvents();
+
+    expect(finalizeMatch).toHaveBeenCalledTimes(1);
+    expect(finalizeMatch).toHaveBeenCalledWith(
+      expect.objectContaining({
+        resultKey: `room:${matched.roomId}:finished`,
+        winnerId: rightUser.id,
+        loserId: leftUser.id
+      })
+    );
+    expect(right.latest("game.finished")).toEqual(
+      expect.objectContaining({
+        result: expect.objectContaining({ roomId: matched.roomId, winnerSide: "right" })
+      })
+    );
+
+    await vi.advanceTimersByTimeAsync(60_000);
+    expect(finalizeMatch).toHaveBeenCalledTimes(1);
+  });
+
+  function setup() {
+    const repository = createMemoryRepository();
+    repositories.push(repository);
+    const finalizeMatch = vi.spyOn(repository, "finalizeMatch").mockResolvedValue({
+      matchId: "forfeit-match",
+      resultKey: "forfeit-result",
+      created: true
+    });
+    return { hub: new GameHub(repository), finalizeMatch };
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
+  closed: { code: number; reason: string } | null = null;
+  private readonly payloads: string[] = [];
+
+  send(payload: string, callback?: (error?: Error) => void): void {
+    this.payloads.push(payload);
+    callback?.();
+  }
+
+  ping(): void {}
+
+  close(code = 1000, reason = ""): void {
+    if (this.readyState === WebSocket.CLOSED) return;
+    this.closed = { code, reason };
+    this.readyState = WebSocket.CLOSED;
+    this.emit("close");
+  }
+
+  terminate(): void {
+    this.close(1006, "terminated");
+  }
+
+  receive(event: object): void {
+    this.emit("message", Buffer.from(JSON.stringify(event)));
+  }
+
+  latest(type: ServerEvent["type"]): ServerEvent | undefined {
+    return this.events().filter((event) => event.type === type).at(-1);
+  }
+
+  private events(): ServerEvent[] {
+    return this.payloads.map((payload) => parseServerEvent(payload));
+  }
+}
+
+async function joinAiMatch(socket: FakeSocket): Promise<string> {
+  socket.receive({ v: 1, type: "queue.join", mode: "ai" });
+  await flushEvents();
+  const matched = socket.latest("queue.matched");
+  if (matched?.type !== "queue.matched") throw new Error("expected an AI match");
+  return matched.roomId;
+}
+
+function snapshotSequence(socket: FakeSocket): number {
+  const event = socket.latest("game.snapshot");
+  if (event?.type !== "game.snapshot") throw new Error("expected a snapshot");
+  return event.snapshot.sequence;
+}
+
+async function flushEvents(): Promise<void> {
+  await Promise.resolve();
+  await Promise.resolve();
+  await Promise.resolve();
+}
+
+function player(handle: string, displayName: string): SessionUser {
+  return {
+    id: `${handle}-id`,
+    handle,
+    displayName,
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


## `fix(web): 중단된 game reconnect 복구`

diff --git a/apps/web/src/app/page.tsx b/apps/web/src/app/page.tsx
index b1f56d4..28b1274 100644
--- a/apps/web/src/app/page.tsx
+++ b/apps/web/src/app/page.tsx
@@ -9,7 +9,12 @@ import { LoginPanel } from "@/components/LoginPanel";
 import { PongCanvas } from "@/components/PongCanvas";
 import { StatCard } from "@/components/StatCard";
 import { requestWsTicket, sendLobbyChat } from "@/lib/api";
-import { demoLobbyPresentation, isDemoMode } from "@/lib/demoPolicy";
+import {
+  demoLobbyPresentation,
+  formatTransientResultNotice,
+  isDemoMode,
+  shouldResumeGameFromLobby
+} from "@/lib/demoPolicy";
 import {
   invalidateExactQueries,
   lobbyQueryOptions,
@@ -58,6 +63,14 @@ export default function HomePage() {
         socketRef.current = socket;
         socket.onmessage = (event) => {
           const message = parseServerEvent(event.data);
+          if (demoMode && shouldResumeGameFromLobby(message)) {
+            window.location.assign("/play");
+            return;
+          }
+          if (message.type === "game.finished" && !message.result.persisted) {
+            setNotice(formatTransientResultNotice(message.result));
+            return;
+          }
           if (message.type === "chat.message" && message.message.scope === "lobby") {
             queryClient.setQueryData<LobbyResponse>(queryKeys.lobby(), (current) => current ? {
               ...current,
@@ -87,7 +100,7 @@ export default function HomePage() {
       if (socket.readyState === WebSocket.OPEN || socket.readyState === WebSocket.CONNECTING) socket.close();
       if (socketRef.current === socket) socketRef.current = null;
     };
-  }, [queryClient, userId]);
+  }, [demoMode, queryClient, userId]);
 
   async function submitLobbyChat(event: React.FormEvent<HTMLFormElement>) {
     event.preventDefault();
@@ -136,6 +149,11 @@ export default function HomePage() {
               ? demoLobbyPresentation.description
               : "빠른 매칭으로 상대를 찾거나 인공지능을 상대로 손을 풀어 보세요. 경기가 끝나면 전적과 순위가 바로 갱신됩니다."}
           </p>
+          {notice ? (
+            <p className="mt-3 rounded-lg bg-amber-50 px-3 py-2 text-sm font-bold text-amber-700" role="status">
+              {notice}
+            </p>
+          ) : null}
           <div className="mt-5 flex flex-wrap gap-3">
             <a className="focus-ring rounded-lg bg-blue-600 px-5 py-3 text-sm font-black text-white" href="/play">
               빠른 매칭
diff --git a/apps/web/src/app/play/page.tsx b/apps/web/src/app/play/page.tsx
index 389087b..1b0ced2 100644
--- a/apps/web/src/app/play/page.tsx
+++ b/apps/web/src/app/play/page.tsx
@@ -6,6 +6,7 @@ import { AppShell } from "@/components/AppShell";
 import { demoLobbyPresentation, isDemoMode } from "@/lib/demoPolicy";
 import { PongCanvas } from "@/components/PongCanvas";
 import { directionForKey, isEditableTarget } from "@/game/gameInput";
+import { canStartNewMatch } from "@/game/gameConnection";
 import { useGameConnection } from "@/game/useGameConnection";
 
 export default function PlayPage() {
@@ -37,6 +38,7 @@ export default function PlayPage() {
   const canPause = Boolean(roomId && state.status === "playing");
   const canResume = Boolean(roomId && state.status === "paused");
   const canMove = Boolean(roomId && state.status === "playing");
+  const canStartMatch = canStartNewMatch(state);
   const opponent = snapshot?.state.players.find((player) => player.side === "right");
   const opponentName = state.opponent ?? opponent?.displayName ?? "대기 중";
 
@@ -125,10 +127,10 @@ export default function PlayPage() {
               <p className="mt-2 text-sm font-semibold text-muted">방향키나 W/S 키를 누르거나 화면 조작 버튼으로 패들을 움직입니다.</p>
             </div>
             <div className="flex gap-3">
-              <button className="focus-ring rounded-lg bg-blue-600 px-4 py-3 text-sm font-black text-white" onClick={() => startQueue("queue")}>
+              <button className="focus-ring rounded-lg bg-blue-600 px-4 py-3 text-sm font-black text-white disabled:cursor-not-allowed disabled:bg-slate-300" disabled={!canStartMatch} onClick={() => startQueue("queue")}>
                 매칭 큐 참가
               </button>
-              <button className="focus-ring rounded-lg bg-green-600 px-4 py-3 text-sm font-black text-white" onClick={() => startQueue("ai")}>
+              <button className="focus-ring rounded-lg bg-green-600 px-4 py-3 text-sm font-black text-white disabled:cursor-not-allowed disabled:bg-slate-300" disabled={!canStartMatch} onClick={() => startQueue("ai")}>
                 인공지능 연습 시작
               </button>
             </div>
diff --git a/apps/web/src/game/GameSocketClient.ts b/apps/web/src/game/GameSocketClient.ts
index d68997c..ad8ad9d 100644
--- a/apps/web/src/game/GameSocketClient.ts
+++ b/apps/web/src/game/GameSocketClient.ts
@@ -17,9 +17,9 @@ export interface GameWebSocket {
 
 export interface GameSocketHandlers {
   onConnecting(): void;
-  onOpen(): void;
+  onOpen(reconnected: boolean): void;
   onEvent(event: ServerEvent): void;
-  onClosed(): void;
+  onClosed(): boolean | void;
   onFailure(error: unknown): void;
 }
 
@@ -31,26 +31,42 @@ type GameSocketClientOptions = {
 
 const CONNECTING = 0;
 const OPEN = 1;
+const RECONNECT_WINDOW_MS = 15_000;
+const INITIAL_RECONNECT_DELAY_MS = 250;
+const MAX_RECONNECT_DELAY_MS = 2_000;
 
 export class GameSocketClient {
   private socket: GameWebSocket | null = null;
   private ticketRequest: AbortController | null = null;
   private generation = 0;
   private inputSequence = 0;
+  private reconnectTimer: ReturnType<typeof setTimeout> | null = null;
+  private reconnectDeadlineMs = 0;
+  private reconnectAttempts = 0;
 
   constructor(private readonly options: GameSocketClientOptions) {}
 
   async connect(initialEvent: ClientEvent, handlers: GameSocketHandlers): Promise<void> {
     const generation = this.replaceConnection();
+    handlers.onConnecting();
+    await this.openSocket(generation, initialEvent, handlers, false);
+  }
+
+  private async openSocket(
+    generation: number,
+    initialEvent: ClientEvent | null,
+    handlers: GameSocketHandlers,
+    reconnected: boolean
+  ): Promise<void> {
     const controller = new AbortController();
     this.ticketRequest = controller;
-    handlers.onConnecting();
 
     let ticket: WsTicketResponse;
     try {
       ticket = await this.options.ticketProvider(controller.signal);
     } catch (error) {
       if (controller.signal.aborted || generation !== this.generation || isAbortError(error)) return;
+      if (reconnected && this.scheduleReconnect(generation, handlers)) return;
       handlers.onFailure(error);
       return;
     } finally {
@@ -68,8 +84,10 @@ export class GameSocketClient {
 
     socket.onopen = () => {
       if (!this.isCurrent(socket, generation)) return;
-      handlers.onOpen();
-      socket.send(JSON.stringify(initialEvent));
+      this.reconnectAttempts = 0;
+      this.reconnectDeadlineMs = 0;
+      handlers.onOpen(reconnected);
+      if (initialEvent) socket.send(JSON.stringify(initialEvent));
     };
     socket.onmessage = (event) => {
       if (!this.isCurrent(socket, generation)) return;
@@ -81,12 +99,12 @@ export class GameSocketClient {
       }
     };
     socket.onerror = () => {
-      if (this.isCurrent(socket, generation)) handlers.onFailure(new Error("실시간 연결에서 오류가 발생했습니다."));
+      if (this.isCurrent(socket, generation)) socket.close();
     };
     socket.onclose = () => {
       if (!this.isCurrent(socket, generation)) return;
       this.socket = null;
-      handlers.onClosed();
+      if (handlers.onClosed() === true) this.scheduleReconnect(generation, handlers);
     };
   }
 
@@ -117,6 +135,10 @@ export class GameSocketClient {
     this.generation += 1;
     this.ticketRequest?.abort();
     this.ticketRequest = null;
+    if (this.reconnectTimer) clearTimeout(this.reconnectTimer);
+    this.reconnectTimer = null;
+    this.reconnectDeadlineMs = 0;
+    this.reconnectAttempts = 0;
 
     const socket = this.socket;
     this.socket = null;
@@ -134,6 +156,28 @@ export class GameSocketClient {
   private isCurrent(socket: GameWebSocket, generation: number): boolean {
     return this.socket === socket && this.generation === generation;
   }
+
+  private scheduleReconnect(generation: number, handlers: GameSocketHandlers): boolean {
+    if (generation !== this.generation) return false;
+    if (this.reconnectTimer) return true;
+    const nowMs = Date.now();
+    if (this.reconnectDeadlineMs === 0) this.reconnectDeadlineMs = nowMs + RECONNECT_WINDOW_MS;
+    if (nowMs >= this.reconnectDeadlineMs) {
+      handlers.onFailure(new Error("경기 재연결 제한 시간을 초과했습니다."));
+      return true;
+    }
+    const delayMs = Math.min(
+      INITIAL_RECONNECT_DELAY_MS * (2 ** this.reconnectAttempts),
+      MAX_RECONNECT_DELAY_MS,
+      this.reconnectDeadlineMs - nowMs
+    );
+    this.reconnectAttempts += 1;
+    this.reconnectTimer = setTimeout(() => {
+      this.reconnectTimer = null;
+      void this.openSocket(generation, null, handlers, true);
+    }, delayMs);
+    return true;
+  }
 }
 
 function isAbortError(error: unknown): boolean {
diff --git a/apps/web/src/game/gameConnection.ts b/apps/web/src/game/gameConnection.ts
index 39b85e1..6374e85 100644
--- a/apps/web/src/game/gameConnection.ts
+++ b/apps/web/src/game/gameConnection.ts
@@ -34,6 +34,7 @@ export const initialGameConnectionState: GameConnectionState = {
 export type GameConnectionAction =
   | { type: "connectStarted" }
   | { type: "socketOpened"; notice: string }
+  | { type: "socketReopened" }
   | { type: "matched"; roomId: string; opponent: string }
   | { type: "snapshotReceived"; snapshot: GameSnapshot }
   | { type: "gameFinished"; result: { leftScore: number; rightScore: number } }
@@ -55,6 +56,8 @@ export function gameConnectionReducer(
       };
     case "socketOpened":
       return { ...state, status: "matching", notice: action.notice };
+    case "socketReopened":
+      return { ...state, status: "reconnecting", notice: "경기 상태 복구 중" };
     case "matched":
       return {
         ...state,
@@ -98,6 +101,10 @@ export function gameConnectionReducer(
   }
 }
 
+export function canStartNewMatch(state: GameConnectionState): boolean {
+  return state.roomId === null && ["idle", "finished", "failed"].includes(state.status);
+}
+
 function statusForSnapshot(snapshot: GameSnapshot): GameConnectionStatus {
   switch (snapshot.state.phase) {
     case "playing":
diff --git a/apps/web/src/game/useGameConnection.ts b/apps/web/src/game/useGameConnection.ts
index 084ff94..0826804 100644
--- a/apps/web/src/game/useGameConnection.ts
+++ b/apps/web/src/game/useGameConnection.ts
@@ -1,15 +1,17 @@
 "use client";
 
-import { useCallback, useEffect, useMemo, useReducer } from "react";
+import { useCallback, useEffect, useMemo, useReducer, useRef } from "react";
 import type { ClientEvent, ServerEvent } from "@pong-pong/shared";
 import { requestWsTicket } from "@/lib/api";
 import { GameSocketClient, type GameSocketHandlers, type GameWebSocket } from "./GameSocketClient";
-import { gameConnectionReducer, initialGameConnectionState } from "./gameConnection";
+import { canStartNewMatch, gameConnectionReducer, initialGameConnectionState } from "./gameConnection";
 
 const WS_URL = process.env.NEXT_PUBLIC_WS_URL ?? "ws://localhost:4000/ws";
 
 export function useGameConnection() {
   const [state, dispatch] = useReducer(gameConnectionReducer, initialGameConnectionState);
+  const stateRef = useRef(state);
+  stateRef.current = state;
   const client = useMemo(() => new GameSocketClient({
     url: WS_URL,
     ticketProvider: requestWsTicket,
@@ -42,11 +44,17 @@ export function useGameConnection() {
   }, []);
 
   const connect = useCallback(async (initialEvent: ClientEvent, openNotice: string) => {
+    if (!canStartNewMatch(stateRef.current)) return;
     const handlers: GameSocketHandlers = {
       onConnecting: () => dispatch({ type: "connectStarted" }),
-      onOpen: () => dispatch({ type: "socketOpened", notice: openNotice }),
+      onOpen: (reconnected) => dispatch(reconnected
+        ? { type: "socketReopened" }
+        : { type: "socketOpened", notice: openNotice }),
       onEvent: handleEvent,
-      onClosed: () => dispatch({ type: "socketClosed" }),
+      onClosed: () => {
+        dispatch({ type: "socketClosed" });
+        return Boolean(stateRef.current.roomId);
+      },
       onFailure: (error) => dispatch({ type: "failed", notice: failureMessage(error) })
     };
     await client.connect(initialEvent, handlers);
diff --git a/apps/web/src/lib/demoPolicy.ts b/apps/web/src/lib/demoPolicy.ts
index afad31c..d826e5d 100644
--- a/apps/web/src/lib/demoPolicy.ts
+++ b/apps/web/src/lib/demoPolicy.ts
@@ -47,3 +47,15 @@ export function isDemoRestrictedPath(pathname: string): boolean {
 export function isDemoMode(): boolean {
   return process.env.NEXT_PUBLIC_APP_MODE === "demo";
 }
+
+export function formatTransientResultNotice(result: {
+  persisted: false;
+  leftScore: number;
+  rightScore: number;
+}): string {
+  return `임시 경기 종료: ${result.leftScore} - ${result.rightScore} · 전적에 저장되지 않았습니다.`;
+}
+
+export function shouldResumeGameFromLobby(event: { type: string }): boolean {
+  return event.type === "queue.matched" || event.type === "game.snapshot";
+}
