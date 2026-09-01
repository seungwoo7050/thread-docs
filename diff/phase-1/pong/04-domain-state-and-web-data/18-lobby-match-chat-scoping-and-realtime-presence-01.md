# 로비·경기방 채팅 범위와 실시간 접속 상태

## `feat(db): 채팅 메시지 저장 구현`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index 6b38a34..4bf995c 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -1,10 +1,10 @@
 import { randomUUID } from "node:crypto";
 import { Kysely, PostgresDialect, sql } from "kysely";
 import { Pool } from "pg";
-import type { DashboardSummary, FriendSummary, LeaderboardEntry, MatchMode, MatchSummary, PublicUser, SessionUser } from "@pong-pong/shared";
+import type { ChatMessage, DashboardSummary, FriendSummary, LeaderboardEntry, MatchMode, MatchSummary, PublicUser, SessionUser } from "@pong-pong/shared";
 import { initialMigrationSql } from "./migrations";
-import { toFriendSummary, toMatchSummary, toPublicUser, toSessionUser } from "./rowMappers";
-import type { Database, FriendshipWithUserRow, MatchWithHandlesRow, MemoryUserRow, UserRow } from "./schema";
+import { toChatMessage, toFriendSummary, toMatchSummary, toPublicUser, toSessionUser } from "./rowMappers";
+import type { ChatMessageRow, ChatMessageWithSenderRow, Database, FriendshipWithUserRow, MatchWithHandlesRow, MemoryUserRow, UserRow } from "./schema";
 
 export type { Database } from "./schema";
 
@@ -44,6 +44,8 @@ export interface AppRepository {
   requestFriend(requesterId: string, addresseeHandle: string): Promise<FriendSummary>;
   acceptFriend(userId: string, friendshipId: string): Promise<FriendSummary>;
   createMatch(input: CreateMatchInput): Promise<string>;
+  listLobbyChat(): Promise<ChatMessage[]>;
+  createChatMessage(input: { scope: "lobby" | "match"; roomId?: string | null; senderId: string; body: string }): Promise<ChatMessage>;
 }
 
 export function createPostgresRepository(databaseUrl: string): AppRepository {
@@ -216,6 +218,23 @@ class PostgresRepository implements AppRepository {
     return firstRow(result).id;
   }
 
+  async listLobbyChat(): Promise<ChatMessage[]> {
+    const result = await sql<ChatMessageWithSenderRow>`
+      select c.*, u.id as user_id, u.email, u.handle, u.display_name, u.avatar_key, u.role, u.status, u.rating, u.wins, u.losses
+      from chat_messages c join users u on u.id = c.sender_id where c.scope = 'lobby'
+      order by c.created_at desc limit 20
+    `.execute(this.db);
+    return result.rows.reverse().map(toChatMessage);
+  }
+
+  async createChatMessage(input: { scope: "lobby" | "match"; roomId?: string | null; senderId: string; body: string }): Promise<ChatMessage> {
+    const result = await sql<ChatMessageRow>`insert into chat_messages (scope, room_id, sender_id, body) values (${input.scope}, ${input.roomId ?? null}, ${input.senderId}, ${input.body}) returning *`.execute(this.db);
+    const user = await this.getUserById(input.senderId);
+    if (!user) throw new Error("chat sender not found");
+    const row = firstRow(result);
+    return { id: row.id, scope: row.scope, roomId: row.room_id, sender: user, body: row.body, createdAt: new Date(row.created_at).toISOString() };
+  }
+
 }
 
 class MemoryRepository implements AppRepository {
@@ -223,6 +242,7 @@ class MemoryRepository implements AppRepository {
   private readonly sessions = new Map<string, string>();
   private readonly matches: MemoryMatchRecord[] = [];
   private readonly friendships: FriendSummary[] = [];
+  private readonly chats: ChatMessage[] = [];
 
   async close(): Promise<void> {}
 
@@ -342,6 +362,16 @@ class MemoryRepository implements AppRepository {
     return id;
   }
 
+  async listLobbyChat(): Promise<ChatMessage[]> { return this.chats.filter((chat) => chat.scope === "lobby").slice(-20); }
+
+  async createChatMessage(input: { scope: "lobby" | "match"; roomId?: string | null; senderId: string; body: string }): Promise<ChatMessage> {
+    const sender = await this.getUserById(input.senderId);
+    if (!sender) throw new Error("chat sender not found");
+    const message = { id: randomUUID(), scope: input.scope, roomId: input.roomId ?? null, sender, body: input.body, createdAt: new Date().toISOString() };
+    this.chats.push(message);
+    return message;
+  }
+
 }
 
 function firstRow<T>(result: { rows: T[] }): T {
diff --git a/packages/db/src/rowMappers.ts b/packages/db/src/rowMappers.ts
index b73e1fd..fbea331 100644
--- a/packages/db/src/rowMappers.ts
+++ b/packages/db/src/rowMappers.ts
@@ -1,5 +1,5 @@
-import type { FriendSummary, MatchSummary, PublicUser, SessionUser } from "@pong-pong/shared";
-import type { FriendshipWithUserRow, MatchWithHandlesRow, UserProjectionRow } from "./schema";
+import type { ChatMessage, FriendSummary, MatchSummary, PublicUser, SessionUser } from "@pong-pong/shared";
+import type { ChatMessageWithSenderRow, FriendshipWithUserRow, MatchWithHandlesRow, UserProjectionRow } from "./schema";
 
 export function toPublicUser(row: UserProjectionRow, online = false): PublicUser {
   return {
@@ -37,3 +37,25 @@ export function toMatchSummary(row: MatchWithHandlesRow, userId?: string): Match
 export function toFriendSummary(row: FriendshipWithUserRow): FriendSummary {
   return { id: row.friendship_id, status: row.friendship_status, user: toPublicUser(row, true) };
 }
+
+export function toChatMessage(row: ChatMessageWithSenderRow): ChatMessage {
+  return {
+    id: row.id,
+    scope: row.scope,
+    roomId: row.room_id,
+    sender: toPublicUser({
+      id: row.user_id,
+      email: row.email,
+      handle: row.handle,
+      display_name: row.display_name,
+      avatar_key: row.avatar_key,
+      role: row.role,
+      status: row.status,
+      rating: row.rating,
+      wins: row.wins,
+      losses: row.losses
+    }),
+    body: row.body,
+    createdAt: row.created_at.toISOString()
+  };
+}
diff --git a/packages/db/src/schema.ts b/packages/db/src/schema.ts
index 1670446..74059a9 100644
--- a/packages/db/src/schema.ts
+++ b/packages/db/src/schema.ts
@@ -44,11 +44,21 @@ export interface FriendshipTable {
   updated_at: Generated<Date>;
 }
 
+export interface ChatMessageTable {
+  id: Generated<string>;
+  scope: "lobby" | "match";
+  room_id: string | null;
+  sender_id: string;
+  body: string;
+  created_at: Generated<Date>;
+}
+
 export interface Database {
   users: UserTable;
   sessions: SessionTable;
   matches: MatchTable;
   friendships: FriendshipTable;
+  chat_messages: ChatMessageTable;
 }
 
 export type UserRow = Selectable<UserTable>;
@@ -64,6 +74,20 @@ export interface FriendshipWithUserRow extends UserRow {
   friendship_id: string;
   friendship_status: import("@pong-pong/shared").FriendshipStatus;
 }
+
+export type ChatMessageRow = Selectable<ChatMessageTable>;
+export interface ChatMessageWithSenderRow extends ChatMessageRow {
+  user_id: string;
+  email: string | null;
+  handle: string;
+  display_name: string;
+  avatar_key: string;
+  role: import("@pong-pong/shared").UserRole;
+  status: import("@pong-pong/shared").UserStatus;
+  rating: number;
+  wins: number;
+  losses: number;
+}
 export type UserProjectionRow = Pick<
   UserRow,
   | "id"


## `feat(game): 실시간 경기 채팅 전달`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index 44f9124..228e237 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -43,11 +43,24 @@ export class GameHub {
     this.broadcastPresence();
   }
 
-  private receive(client: Client, payload: string): void {
+  private async receive(client: Client, payload: string): Promise<void> {
     try {
       const event = parseClientEvent(payload);
       if (event.type === "queue.join") this.joinQueue(client, event.mode);
       if (event.type === "queue.leave") this.leaveQueue(client);
+      if (event.type === "chat.send") {
+        const message = await this.repo.createChatMessage({
+          scope: event.scope,
+          roomId: event.roomId ?? null,
+          senderId: client.user.id,
+          body: event.body
+        });
+        if (event.scope === "match" && event.roomId) {
+          this.broadcastRoom(event.roomId, { type: "chat.message", message });
+        } else {
+          this.broadcastAll({ type: "chat.message", message });
+        }
+      }
     } catch (error) {
       this.send(client, { type: "error", message: error instanceof Error ? error.message : "메시지를 처리하지 못했습니다." });
     }


## `feat(lobby): 실시간 로비 지표 API 추가`

diff --git a/apps/api/src/app.ts b/apps/api/src/app.ts
index bc04acb..2f7ae0e 100644
--- a/apps/api/src/app.ts
+++ b/apps/api/src/app.ts
@@ -89,7 +89,8 @@ export function buildApp({ repo, webOrigin }: BuildAppOptions) {
       me: user,
       onlinePlayers: await repo.listOnlineUsers(),
       recentMatches: await repo.listRecentMatches(user?.id),
-      chat: await repo.listLobbyChat()
+      chat: await repo.listLobbyChat(),
+      stats: hub.liveStats()
     };
   });
 
diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index e3ad76d..25d05ef 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -25,6 +25,11 @@ type Client = {
   roomId: string | null;
 };
 
+type QueueEntry = {
+  client: Client;
+  queuedAt: number;
+};
+
 type Room = {
   id: string;
   clients: Partial<Record<PlayerSide, Client>>;
@@ -36,8 +41,9 @@ type Room = {
 
 export class GameHub {
   private readonly clients = new Map<string, Client>();
-  private readonly queue: Client[] = [];
+  private readonly queue: QueueEntry[] = [];
   private readonly rooms = new Map<string, Room>();
+  private readonly waitSamples: number[] = [];
 
   constructor(private readonly repo: AppRepository) {}
 
@@ -98,26 +104,49 @@ export class GameHub {
     }
     const opponent = this.queue.shift();
     if (!opponent) {
-      this.queue.push(client);
+      this.queue.push({ client, queuedAt: Date.now() });
       this.broadcastPresence();
       return;
     }
-    this.createRoom(opponent, client, false);
+    this.recordWaitSample(opponent.queuedAt);
+    this.createRoom(opponent.client, client, false);
   }
 
   private leaveQueue(client: Client): void {
-    const index = this.queue.findIndex((queued) => queued.id === client.id);
+    const index = this.queue.findIndex((queued) => queued.client.id === client.id);
     if (index >= 0) this.queue.splice(index, 1);
   }
 
   private pruneQueue(): void {
     for (let index = this.queue.length - 1; index >= 0; index -= 1) {
-      if (this.queue[index].socket.readyState !== WebSocket.OPEN) {
+      if (this.queue[index].client.socket.readyState !== WebSocket.OPEN) {
         this.queue.splice(index, 1);
       }
     }
   }
 
+  liveStats() {
+    const playingPlayers = [...this.rooms.values()].reduce((count, room) => count + Object.values(room.clients).filter(Boolean).length, 0);
+    const averageWaitSeconds = this.waitSamples.length === 0
+      ? null
+      : Math.round(this.waitSamples.reduce((sum, value) => sum + value, 0) / this.waitSamples.length);
+    return {
+      onlinePlayers: this.clients.size,
+      playingPlayers,
+      queuedPlayers: this.queue.length,
+      activeRooms: this.rooms.size,
+      averageWaitSeconds
+    };
+  }
+
+  private recordWaitSample(queuedAt: number): void {
+    const seconds = Math.max(0, Math.round((Date.now() - queuedAt) / 1000));
+    this.waitSamples.push(seconds);
+    if (this.waitSamples.length > 20) {
+      this.waitSamples.shift();
+    }
+  }
+
   private createRoom(left: Client, right: Client | null, ai: boolean): void {
     const roomId = randomUUID();
     const room: Room = {
diff --git a/packages/shared/src/http.ts b/packages/shared/src/http.ts
index 98b96bf..36f0bea 100644
--- a/packages/shared/src/http.ts
+++ b/packages/shared/src/http.ts
@@ -60,6 +60,22 @@ export interface ChatMessage {
   createdAt: string;
 }
 
+export interface LobbyStats {
+  onlinePlayers: number;
+  playingPlayers: number;
+  queuedPlayers: number;
+  activeRooms: number;
+  averageWaitSeconds: number | null;
+}
+
+export interface LobbyResponse {
+  me: SessionUser | null;
+  onlinePlayers: PublicUser[];
+  recentMatches: MatchSummary[];
+  chat: ChatMessage[];
+  stats: LobbyStats;
+}
+
 export interface TournamentSummary {
   id: string;
   name: string;


## `feat(chat): 쓰기 가능한 로비 채팅 API 추가`

diff --git a/apps/api/src/app.ts b/apps/api/src/app.ts
index 2f7ae0e..55994eb 100644
--- a/apps/api/src/app.ts
+++ b/apps/api/src/app.ts
@@ -94,6 +94,23 @@ export function buildApp({ repo, webOrigin }: BuildAppOptions) {
     };
   });
 
+  app.post("/chat/lobby", async (request, reply) => {
+    const user = await currentUser(repo, request);
+    if (!user) return unauthorized(reply);
+    const body = request.body as { body?: string };
+    const messageBody = body.body?.trim() ?? "";
+    if (!messageBody) return reply.code(400).send({ message: "메시지를 입력해주세요." });
+    if (messageBody.length > 240) return reply.code(400).send({ message: "메시지는 240자 이내로 입력해주세요." });
+    return {
+      message: await repo.createChatMessage({
+        scope: "lobby",
+        roomId: null,
+        senderId: user.id,
+        body: messageBody
+      })
+    };
+  });
+
   app.get("/leaderboard", async () => ({ entries: await repo.listLeaderboard() }));
 
   app.get("/dashboard", async (request, reply) => {


## `feat(chat): 로비 채팅과 접속 상태 실시간 반영`

diff --git a/apps/web/src/app/page.tsx b/apps/web/src/app/page.tsx
index ef1999c..f635156 100644
--- a/apps/web/src/app/page.tsx
+++ b/apps/web/src/app/page.tsx
@@ -1,15 +1,17 @@
 "use client";
 
-import { useEffect, useState } from "react";
+import { useCallback, useEffect, useRef, useState } from "react";
 import { Bot, Clock, MessageCircle, Trophy, Users, Zap } from "lucide-react";
-import type { ChatMessage, LobbyStats, PublicUser, SessionUser } from "@pong-pong/shared";
+import type { ChatMessage, LobbyStats, PublicUser, ServerEvent, SessionUser } from "@pong-pong/shared";
 import { AppShell } from "@/components/AppShell";
 import { LoginPanel } from "@/components/LoginPanel";
 import { PongCanvas } from "@/components/PongCanvas";
 import { StatCard } from "@/components/StatCard";
-import { getLobby, getMe, sendLobbyChat } from "@/lib/api";
+import { getLobby, getMe, getToken, sendLobbyChat } from "@/lib/api";
 import { sampleChat, sampleUsers } from "@/lib/sample";
 
+const WS_URL = process.env.NEXT_PUBLIC_WS_URL ?? "ws://localhost:4000/ws";
+
 export default function HomePage() {
   const [me, setMe] = useState<SessionUser | null>(null);
   const [players, setPlayers] = useState<PublicUser[]>(sampleUsers);
@@ -17,27 +19,61 @@ export default function HomePage() {
   const [stats, setStats] = useState<LobbyStats | null>(null);
   const [chatInput, setChatInput] = useState("");
   const [notice, setNotice] = useState("");
+  const socketRef = useRef<WebSocket | null>(null);
+  const userId = me?.id;
+
+  const loadLobby = useCallback(async () => {
+    const lobby = await getLobby();
+    setPlayers(lobby.onlinePlayers);
+    setChat(lobby.chat);
+    setStats(lobby.stats);
+    if (lobby.me) setMe(lobby.me);
+    setNotice("");
+  }, []);
 
   useEffect(() => {
     getMe().then(setMe);
-    getLobby()
-      .then((lobby) => {
-        setPlayers(lobby.onlinePlayers);
-        setChat(lobby.chat);
-        setStats(lobby.stats);
-        if (lobby.me) setMe(lobby.me);
-        setNotice("");
-      })
-      .catch(() => setNotice("서버 로비 정보를 불러오지 못해 샘플 화면을 표시합니다."));
-  }, []);
+    loadLobby().catch(() => setNotice("서버 로비 정보를 불러오지 못해 샘플 화면을 표시합니다."));
+  }, [loadLobby]);
+
+  useEffect(() => {
+    const token = getToken();
+    if (!userId || !token) return;
+    const socket = new WebSocket(`${WS_URL}?session=${token}`);
+    socketRef.current = socket;
+    socket.onmessage = (event) => {
+      const message = JSON.parse(event.data) as ServerEvent;
+      if (message.type === "chat.message" && message.message.scope === "lobby") {
+        setChat((current) => [...current.filter((item) => item.id !== message.message.id).slice(-19), message.message]);
+      }
+      if (message.type === "presence.changed") {
+        loadLobby().catch(() => setNotice("로비 지표를 갱신하지 못했습니다."));
+      }
+      if (message.type === "error") setNotice(message.message);
+    };
+    socket.onclose = () => {
+      if (socketRef.current === socket) socketRef.current = null;
+    };
+    return () => {
+      socket.onclose = null;
+      socket.onmessage = null;
+      if (socket.readyState === WebSocket.OPEN || socket.readyState === WebSocket.CONNECTING) socket.close();
+      if (socketRef.current === socket) socketRef.current = null;
+    };
+  }, [loadLobby, userId]);
 
   async function submitLobbyChat(event: React.FormEvent<HTMLFormElement>) {
     event.preventDefault();
     const body = chatInput.trim();
     if (!body) return;
     try {
-      const message = await sendLobbyChat(body);
-      setChat((current) => [...current.slice(-19), message]);
+      const socket = socketRef.current;
+      if (socket?.readyState === WebSocket.OPEN) {
+        socket.send(JSON.stringify({ type: "chat.send", scope: "lobby", roomId: null, body }));
+      } else {
+        const message = await sendLobbyChat(body);
+        setChat((current) => [...current.slice(-19), message]);
+      }
       setChatInput("");
       setNotice("");
     } catch {


## `feat(lobby): 연결 중인 WebSocket 사용자 목록 추가`

diff --git a/apps/api/src/app.ts b/apps/api/src/app.ts
index 9c74a49..142354e 100644
--- a/apps/api/src/app.ts
+++ b/apps/api/src/app.ts
@@ -93,7 +93,7 @@ export function buildApp({ repo, webOrigin }: BuildAppOptions) {
     const user = await currentUser(repo, request);
     return {
       me: user,
-      onlinePlayers: await repo.listOnlineUsers(),
+      onlinePlayers: hub.onlinePlayers(),
       recentMatches: await repo.listRecentMatches(user?.id),
       chat: await repo.listLobbyChat(),
       stats: hub.liveStats()
diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index 06e125d..9931aed 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -15,6 +15,7 @@ import {
   type GameSnapshot,
   type MatchMode,
   type PlayerSide,
+  type PublicUser,
   type ServerEvent,
   type SessionUser
 } from "@pong-pong/shared";
@@ -201,6 +202,15 @@ export class GameHub {
     };
   }
 
+  onlinePlayers(): PublicUser[] {
+    const users = new Map<string, PublicUser>();
+    for (const client of this.clients.values()) {
+      const { email: _email, ...user } = client.user;
+      users.set(user.id, { ...user, online: true });
+    }
+    return [...users.values()].sort((left, right) => right.rating - left.rating || left.displayName.localeCompare(right.displayName));
+  }
+
   private recordWaitSample(queuedAt: number): void {
     const seconds = Math.max(0, Math.round((Date.now() - queuedAt) / 1000));
     this.waitSamples.push(seconds);


## `fix(protocol): 채팅 scope와 room 식별자 조합 제한`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index 6cd8465..d4dbeef 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -230,13 +230,14 @@ export class GameHub {
       if (event.type === "game.resume") this.resumeRoom(client, event.roomId);
       if (event.type === "game.input") this.applyInput(client, event.roomId, event.inputSeq, event.direction);
       if (event.type === "chat.send") {
+        const roomId = event.scope === "match" ? event.roomId : null;
         const message = await this.repo.createChatMessage({
           scope: event.scope,
-          roomId: event.roomId ?? null,
+          roomId,
           senderId: client.user.id,
           body: event.body
         });
-        if (event.scope === "match" && event.roomId) {
+        if (event.scope === "match") {
           this.broadcastRoom(event.roomId, { type: "chat.message", message });
         } else {
           this.broadcastAll({ type: "chat.message", message });
diff --git a/apps/web/src/app/page.tsx b/apps/web/src/app/page.tsx
index 28b1274..517b64b 100644
--- a/apps/web/src/app/page.tsx
+++ b/apps/web/src/app/page.tsx
@@ -109,7 +109,7 @@ export default function HomePage() {
     try {
       const socket = socketRef.current;
       if (socket?.readyState === WebSocket.OPEN) {
-        socket.send(JSON.stringify({ v: 1, type: "chat.send", scope: "lobby", roomId: null, body }));
+        socket.send(JSON.stringify({ v: 1, type: "chat.send", scope: "lobby", body }));
       } else {
         await chatMutation.mutateAsync(body);
       }
diff --git a/packages/shared/src/ws.test.ts b/packages/shared/src/ws.test.ts
index 16f800c..ccd188f 100644
--- a/packages/shared/src/ws.test.ts
+++ b/packages/shared/src/ws.test.ts
@@ -16,7 +16,7 @@ describe("version 1 client events", () => {
     { payload: { v: 1, type: "game.pause", roomId: "room-1" } },
     { payload: { v: 1, type: "game.resume", roomId: "room-1" } },
     { payload: { v: 1, type: "game.input", roomId: "room-1", inputSeq: 7, direction: -1 } },
-    { payload: { v: 1, type: "chat.send", scope: "match", roomId: "room-1", body: "hello" } }
+    { payload: { v: 1, type: "chat.send", scope: "match", roomId: "11111111-1111-4111-8111-111111111111", body: "hello" } }
   ])("accepts $payload.type", ({ payload }) => {
     expect(parseClientEvent(JSON.stringify(payload))).toEqual(payload);
   });
diff --git a/packages/shared/src/ws.ts b/packages/shared/src/ws.ts
index 31cc7c5..4876c82 100644
--- a/packages/shared/src/ws.ts
+++ b/packages/shared/src/ws.ts
@@ -4,8 +4,9 @@ import { gameFinishedSchema, gameSnapshotSchema, playerSideSchema } from "./game
 
 const version = { v: z.literal(1) } as const;
 const roomIdSchema = z.string().min(1);
+const chatBodySchema = z.string().trim().min(1).max(240);
 
-export const clientEventSchema = z.discriminatedUnion("type", [
+const gameplayClientEventSchema = z.discriminatedUnion("type", [
   z.object({
     ...version,
     type: z.literal("queue.join"),
@@ -23,15 +24,29 @@ export const clientEventSchema = z.discriminatedUnion("type", [
     inputSeq: z.number().int().nonnegative().max(Number.MAX_SAFE_INTEGER),
     direction: z.union([z.literal(-1), z.literal(0), z.literal(1)])
   }).strict(),
+]);
+
+const chatClientEventSchema = z.discriminatedUnion("scope", [
+  z.object({
+    ...version,
+    type: z.literal("chat.send"),
+    scope: z.literal("lobby"),
+    body: chatBodySchema
+  }).strict(),
   z.object({
     ...version,
     type: z.literal("chat.send"),
-    scope: z.enum(["lobby", "match"]),
-    roomId: z.string().nullable().optional(),
-    body: z.string().trim().min(1).max(240)
+    scope: z.literal("match"),
+    roomId: z.string().uuid(),
+    body: chatBodySchema
   }).strict()
 ]);
 
+export const clientEventSchema = z.union([
+  gameplayClientEventSchema,
+  chatClientEventSchema
+]);
+
 export const wsErrorCodeSchema = z.enum([
   "invalid_event",
   "rate_limited",
diff --git a/tests/smoke-ws.mjs b/tests/smoke-ws.mjs
index 4795f90..0bb762e 100644
--- a/tests/smoke-ws.mjs
+++ b/tests/smoke-ws.mjs
@@ -20,7 +20,7 @@ try {
     return onlineHandles.includes("left-smoke") && onlineHandles.includes("right-smoke") ? lobby : null;
   });
 
-  send(leftSocket, { type: "chat.send", scope: "lobby", roomId: null, body: "로비 실시간 확인" });
+  send(leftSocket, { type: "chat.send", scope: "lobby", body: "로비 실시간 확인" });
   await waitFor(() => events.find((item) => item.event.type === "chat.message" && item.event.message.scope === "lobby"));
 
   send(leftSocket, { type: "queue.join", mode: "queue" });


