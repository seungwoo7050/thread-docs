## `fix(db): 채팅 행의 scope와 room 불변식 강제`

diff --git a/packages/db/migrations/006_chat_invariants.sql b/packages/db/migrations/006_chat_invariants.sql
new file mode 100644
index 0000000..d470b8d
--- /dev/null
+++ b/packages/db/migrations/006_chat_invariants.sql
@@ -0,0 +1,27 @@
+update chat_messages
+set room_id = null
+where scope = 'lobby' and room_id is not null;
+
+delete from chat_messages
+where scope not in ('lobby', 'match')
+   or (
+     scope = 'match'
+     and (
+       room_id is null
+       or room_id !~* '^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$'
+     )
+   );
+
+alter table chat_messages
+  drop constraint if exists chat_messages_scope_room_check;
+
+alter table chat_messages
+  add constraint chat_messages_scope_room_check
+  check (
+    (scope = 'lobby' and room_id is null)
+    or (
+      scope = 'match'
+      and room_id is not null
+      and room_id ~* '^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$'
+    )
+  );
diff --git a/packages/db/src/index.test.ts b/packages/db/src/index.test.ts
index 34212c1..68d5836 100644
--- a/packages/db/src/index.test.ts
+++ b/packages/db/src/index.test.ts
@@ -136,10 +136,11 @@ describe("memory repository", () => {
     const user = await repo.upsertDevUser({ handle: "speaker", displayName: "말하는선수" });
 
     const lobby = await repo.createChatMessage({ scope: "lobby", senderId: user.id, body: "로비 메시지" });
-    const match = await repo.createChatMessage({ scope: "match", roomId: "room-1", senderId: user.id, body: "매치 메시지" });
+    const roomId = "11111111-1111-4111-8111-111111111111";
+    const match = await repo.createChatMessage({ scope: "match", roomId, senderId: user.id, body: "매치 메시지" });
 
     expect(lobby.sender.handle).toBe("speaker");
-    expect(match.roomId).toBe("room-1");
+    expect(match.roomId).toBe(roomId);
     expect((await repo.listLobbyChat()).map((message) => message.body)).toContain("로비 메시지");
   });
 
diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index e46ffdf..a7f8572 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -698,6 +698,7 @@ class PostgresRepository implements AppRepository {
   }
 
   async createChatMessage(input: { scope: "lobby" | "match"; roomId?: string | null; senderId: string; body: string }): Promise<ChatMessage> {
+    assertChatRoom(input);
     const result = await sql<ChatMessageRow>`
       insert into chat_messages (scope, room_id, sender_id, body)
       values (${input.scope}, ${input.roomId ?? null}, ${input.senderId}, ${input.body})
@@ -1213,6 +1214,7 @@ class MemoryRepository implements AppRepository {
   }
 
   async createChatMessage(input: { scope: "lobby" | "match"; roomId?: string | null; senderId: string; body: string }): Promise<ChatMessage> {
+    assertChatRoom(input);
     const sender = await this.getUserById(input.senderId);
     if (!sender) throw new Error("chat sender not found");
     const message: ChatMessage = {
@@ -1360,6 +1362,18 @@ function assertTicketTtl(value: number): void {
   }
 }
 
+function assertChatRoom(input: { scope: "lobby" | "match"; roomId?: string | null }): void {
+  if (input.scope === "lobby") {
+    if (input.roomId !== undefined && input.roomId !== null) {
+      throw new Error("lobby chat must not identify a room");
+    }
+    return;
+  }
+  if (!input.roomId || !/^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i.test(input.roomId)) {
+    throw new Error("match chat requires a UUID room");
+  }
+}
+
 function assertFinalizeMatchCommand(command: FinalizeMatchCommand): void {
   if (!command.resultKey.trim() || command.resultKey.length > 200) {
     throw new Error("invalid match result key");
diff --git a/packages/db/src/postgres.integration.test.ts b/packages/db/src/postgres.integration.test.ts
index 307a9b8..1591c24 100644
--- a/packages/db/src/postgres.integration.test.ts
+++ b/packages/db/src/postgres.integration.test.ts
@@ -90,7 +90,8 @@ describe("PostgreSQL integration", () => {
         "002_ws_tickets",
         "003_match_finalization",
         "004_friendship_tournament_invariants",
-        "005_expire_legacy_sessions"
+        "005_expire_legacy_sessions",
+        "006_chat_invariants"
       ]);
 
       await migrateDatabase(databaseUrl);


## `fix(game): 매치 채팅의 좌석과 audience 검증`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index d4dbeef..fa74507 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -230,16 +230,30 @@ export class GameHub {
       if (event.type === "game.resume") this.resumeRoom(client, event.roomId);
       if (event.type === "game.input") this.applyInput(client, event.roomId, event.inputSeq, event.direction);
       if (event.type === "chat.send") {
-        const roomId = event.scope === "match" ? event.roomId : null;
-        const message = await this.repo.createChatMessage({
-          scope: event.scope,
-          roomId,
-          senderId: client.user.id,
-          body: event.body
-        });
         if (event.scope === "match") {
+          const room = this.rooms.get(event.roomId);
+          if (!room || client.roomId !== room.id || !sideFor(room, client)) {
+            this.send(client, {
+              type: "error",
+              code: "forbidden",
+              message: "현재 경기방에만 채팅을 보낼 수 있습니다."
+            });
+            return;
+          }
+          const message = await this.repo.createChatMessage({
+            scope: "match",
+            roomId: event.roomId,
+            senderId: client.user.id,
+            body: event.body
+          });
           this.broadcastRoom(event.roomId, { type: "chat.message", message });
         } else {
+          const message = await this.repo.createChatMessage({
+            scope: "lobby",
+            roomId: null,
+            senderId: client.user.id,
+            body: event.body
+          });
           this.broadcastAll({ type: "chat.message", message });
         }
       }


## `test(game): 타 경기방 채팅 주입 차단 검증`

diff --git a/apps/api/src/gameHub.chat.test.ts b/apps/api/src/gameHub.chat.test.ts
new file mode 100644
index 0000000..031eacb
--- /dev/null
+++ b/apps/api/src/gameHub.chat.test.ts
@@ -0,0 +1,209 @@
+import { randomUUID } from "node:crypto";
+import { EventEmitter } from "node:events";
+import type { IncomingMessage } from "node:http";
+import { afterEach, describe, expect, it, vi } from "vitest";
+import { WebSocket } from "ws";
+import { createMemoryRepository } from "@pong-pong/db";
+import {
+  parseServerEvent,
+  type ChatMessage,
+  type ServerEvent,
+  type SessionUser
+} from "@pong-pong/shared";
+import { GameHub } from "./gameHub.js";
+
+describe("GameHub chat authorization", () => {
+  const hubs: GameHub[] = [];
+  const repositories: Array<ReturnType<typeof createMemoryRepository>> = [];
+
+  afterEach(async () => {
+    for (const hub of hubs.splice(0)) hub.close();
+    await Promise.all(repositories.splice(0).map((repository) => repository.close()));
+  });
+
+  it("rejects cross-room injection before persistence", async () => {
+    const context = setup();
+    const roomA = await pair(context, "a-left", "a-right");
+    const roomB = await pair(context, "b-left", "b-right");
+    const intruder = context.sockets.get("b-left");
+    if (!intruder) throw new Error("missing intruder socket");
+
+    intruder.receive({
+      v: 1,
+      type: "chat.send",
+      scope: "match",
+      roomId: roomA,
+      body: "cross-room message"
+    });
+    await flushEvents();
+
+    expect(roomB).not.toBe(roomA);
+    expect(context.createChatMessage).not.toHaveBeenCalled();
+    expect(intruder.latest("error")).toEqual(expect.objectContaining({
+      type: "error",
+      code: "forbidden"
+    }));
+    expect([...context.sockets.values()].flatMap((socket) => socket.events("chat.message"))).toEqual([]);
+  });
+
+  it("delivers match chat only to the active room audience", async () => {
+    const context = setup();
+    const roomA = await pair(context, "a-left", "a-right");
+    await pair(context, "b-left", "b-right");
+    const sender = context.sockets.get("a-left");
+    if (!sender) throw new Error("missing sender socket");
+
+    sender.receive({
+      v: 1,
+      type: "chat.send",
+      scope: "match",
+      roomId: roomA,
+      body: "room A only"
+    });
+    await flushEvents();
+
+    expect(context.createChatMessage).toHaveBeenCalledWith(expect.objectContaining({
+      scope: "match",
+      roomId: roomA,
+      senderId: context.users.get("a-left")?.id
+    }));
+    expect(context.sockets.get("a-left")?.events("chat.message")).toHaveLength(1);
+    expect(context.sockets.get("a-right")?.events("chat.message")).toHaveLength(1);
+    expect(context.sockets.get("b-left")?.events("chat.message")).toHaveLength(0);
+    expect(context.sockets.get("b-right")?.events("chat.message")).toHaveLength(0);
+  });
+
+  it("normalizes lobby chat to a null room and broadcasts it globally", async () => {
+    const context = setup();
+    await pair(context, "a-left", "a-right");
+    await pair(context, "b-left", "b-right");
+    const sender = context.sockets.get("a-left");
+    if (!sender) throw new Error("missing sender socket");
+
+    sender.receive({
+      v: 1,
+      type: "chat.send",
+      scope: "lobby",
+      body: "lobby message"
+    });
+    await flushEvents();
+
+    expect(context.createChatMessage).toHaveBeenCalledWith(expect.objectContaining({
+      scope: "lobby",
+      roomId: null,
+      senderId: context.users.get("a-left")?.id
+    }));
+    for (const socket of context.sockets.values()) {
+      expect(socket.events("chat.message")).toHaveLength(1);
+    }
+  });
+
+  function setup() {
+    const repository = createMemoryRepository();
+    repositories.push(repository);
+    const hub = new GameHub(repository);
+    hubs.push(hub);
+    const users = new Map<string, SessionUser>();
+    const sockets = new Map<string, FakeSocket>();
+    const createChatMessage = vi.spyOn(repository, "createChatMessage").mockImplementation(async (input) => {
+      const sender = [...users.values()].find((candidate) => candidate.id === input.senderId);
+      if (!sender) throw new Error("missing chat sender");
+      const message: ChatMessage = {
+        id: randomUUID(),
+        scope: input.scope,
+        roomId: input.roomId ?? null,
+        sender,
+        body: input.body,
+        createdAt: new Date().toISOString()
+      };
+      return message;
+    });
+    return { repository, hub, users, sockets, createChatMessage };
+  }
+});
+
+interface ConnectionContext {
+  hub: GameHub;
+  users: Map<string, SessionUser>;
+  sockets: Map<string, FakeSocket>;
+}
+
+async function pair(context: ConnectionContext, leftHandle: string, rightHandle: string): Promise<string> {
+  const left = connect(context, leftHandle);
+  const right = connect(context, rightHandle);
+  left.receive({ v: 1, type: "queue.join", mode: "queue" });
+  right.receive({ v: 1, type: "queue.join", mode: "queue" });
+  await flushEvents();
+  const matched = left.latest("queue.matched");
+  if (matched?.type !== "queue.matched") throw new Error("expected a match");
+  return matched.roomId;
+}
+
+function connect(context: ConnectionContext, handle: string): FakeSocket {
+  const user = player(handle, context.users.size + 1);
+  const socket = new FakeSocket();
+  context.users.set(handle, user);
+  context.sockets.set(handle, socket);
+  context.hub.connect(socket as unknown as WebSocket, {} as IncomingMessage, user);
+  return socket;
+}
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
+    return this.events(type).at(-1);
+  }
+
+  events(type: ServerEvent["type"]): ServerEvent[] {
+    return this.payloads
+      .map((payload) => parseServerEvent(payload))
+      .filter((event) => event.type === type);
+  }
+}
+
+async function flushEvents(): Promise<void> {
+  await Promise.resolve();
+  await Promise.resolve();
+  await Promise.resolve();
+}
+
+function player(handle: string, ordinal: number): SessionUser {
+  return {
+    id: `00000000-0000-4000-8000-${ordinal.toString().padStart(12, "0")}`,
+    handle,
+    displayName: handle,
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


## `fix(web): 현재 경기방의 채팅만 표시`

diff --git a/apps/web/src/game/chatScope.ts b/apps/web/src/game/chatScope.ts
new file mode 100644
index 0000000..1b2663c
--- /dev/null
+++ b/apps/web/src/game/chatScope.ts
@@ -0,0 +1,10 @@
+import type { ChatMessage } from "@pong-pong/shared";
+
+export function isChatForActiveRoom(
+  message: ChatMessage,
+  activeRoomId: string | null
+): boolean {
+  return activeRoomId !== null
+    && message.scope === "match"
+    && message.roomId === activeRoomId;
+}
diff --git a/apps/web/src/game/useGameConnection.ts b/apps/web/src/game/useGameConnection.ts
index 0826804..fbc4a83 100644
--- a/apps/web/src/game/useGameConnection.ts
+++ b/apps/web/src/game/useGameConnection.ts
@@ -3,6 +3,7 @@
 import { useCallback, useEffect, useMemo, useReducer, useRef } from "react";
 import type { ClientEvent, ServerEvent } from "@pong-pong/shared";
 import { requestWsTicket } from "@/lib/api";
+import { isChatForActiveRoom } from "./chatScope";
 import { GameSocketClient, type GameSocketHandlers, type GameWebSocket } from "./GameSocketClient";
 import { canStartNewMatch, gameConnectionReducer, initialGameConnectionState } from "./gameConnection";
 
@@ -30,6 +31,7 @@ export function useGameConnection() {
         dispatch({ type: "gameFinished", result: event.result });
         return;
       case "chat.message":
+        if (!isChatForActiveRoom(event.message, stateRef.current.roomId)) return;
         dispatch({
           type: "chatReceived",
           message: `${event.message.sender.displayName}: ${event.message.body}`


## `test(web): 매치 채팅 room filtering 검증`

diff --git a/apps/web/src/game/chatScope.test.ts b/apps/web/src/game/chatScope.test.ts
new file mode 100644
index 0000000..6187aaf
--- /dev/null
+++ b/apps/web/src/game/chatScope.test.ts
@@ -0,0 +1,38 @@
+import { describe, expect, it } from "vitest";
+import type { ChatMessage } from "@pong-pong/shared";
+import { isChatForActiveRoom } from "./chatScope";
+
+const activeRoomId = "11111111-1111-4111-8111-111111111111";
+const otherRoomId = "22222222-2222-4222-8222-222222222222";
+
+describe("isChatForActiveRoom", () => {
+  it("accepts only match messages for the current room", () => {
+    expect(isChatForActiveRoom(message("match", activeRoomId), activeRoomId)).toBe(true);
+    expect(isChatForActiveRoom(message("match", otherRoomId), activeRoomId)).toBe(false);
+    expect(isChatForActiveRoom(message("lobby", null), activeRoomId)).toBe(false);
+    expect(isChatForActiveRoom(message("match", activeRoomId), null)).toBe(false);
+  });
+});
+
+function message(scope: ChatMessage["scope"], roomId: string | null): ChatMessage {
+  return {
+    id: "33333333-3333-4333-8333-333333333333",
+    scope,
+    roomId,
+    sender: {
+      id: "44444444-4444-4444-8444-444444444444",
+      handle: "sender",
+      displayName: "Sender",
+      avatarKey: "default",
+      role: "user",
+      status: "active",
+      rating: 1_200,
+      wins: 0,
+      losses: 0,
+      online: true,
+      isNpc: false
+    },
+    body: "hello",
+    createdAt: "2026-08-13T00:00:00.000Z"
+  };
+}
