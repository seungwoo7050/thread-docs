## `feat(auth): WebSocket ticket 생성과 HTTP 계약 정의`

diff --git a/apps/api/src/wsTicket.ts b/apps/api/src/wsTicket.ts
new file mode 100644
index 0000000..4e67b24
--- /dev/null
+++ b/apps/api/src/wsTicket.ts
@@ -0,0 +1,11 @@
+import { createHash, randomBytes } from "node:crypto";
+
+export const WS_TICKET_TTL_SECONDS = 30;
+
+export function createRawWsTicket(): string {
+  return randomBytes(32).toString("base64url");
+}
+
+export function hashWsTicket(ticket: string): string {
+  return createHash("sha256").update(ticket, "utf8").digest("hex");
+}
diff --git a/packages/shared/src/http.ts b/packages/shared/src/http.ts
index 58290e6..e475c12 100644
--- a/packages/shared/src/http.ts
+++ b/packages/shared/src/http.ts
@@ -183,6 +183,12 @@ export const adminStatusBodySchema = z.object({
   reason: z.string().trim().min(1).max(240).optional()
 }).strict();
 
+export const wsTicketSchema = z.string().regex(/^[A-Za-z0-9_-]{43}$/);
+export const wsHandshakeQuerySchema = z.object({
+  ticket: wsTicketSchema,
+  v: z.literal("1")
+}).strict();
+
 export const okResponseSchema = z.object({ ok: z.literal(true) });
 export const healthResponseSchema = z.object({ ok: z.literal(true), service: z.literal("pong-pong-api") });
 export const userResponseSchema = z.object({ user: sessionUserSchema });
@@ -198,7 +204,7 @@ export const tournamentResponseSchema = z.object({ tournament: tournamentSummary
 export const adminUsersResponseSchema = z.object({ users: z.array(publicUserSchema) });
 export const adminActionsResponseSchema = z.object({ actions: z.array(adminActionSummarySchema) });
 export const wsTicketResponseSchema = z.object({
-  ticket: z.string().min(32),
+  ticket: wsTicketSchema,
   expiresInSeconds: z.literal(30),
   protocolVersion: z.literal(1)
 });


## `feat(db): PostgreSQL WebSocket ticket 저장 추가`

diff --git a/packages/db/migrations/002_ws_tickets.sql b/packages/db/migrations/002_ws_tickets.sql
new file mode 100644
index 0000000..e7221da
--- /dev/null
+++ b/packages/db/migrations/002_ws_tickets.sql
@@ -0,0 +1,8 @@
+create table ws_tickets (
+  ticket_hash text primary key check (ticket_hash ~ '^[a-f0-9]{64}$'),
+  user_id uuid not null references users(id) on delete cascade,
+  expires_at timestamptz not null,
+  created_at timestamptz not null default now()
+);
+
+create index ws_tickets_expires_at_idx on ws_tickets (expires_at);
diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index 643b612..e349f9f 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -13,6 +13,12 @@ export interface DevLoginInput {
   email?: string | null;
 }
 
+export interface CreateWsTicketInput {
+  userId: string;
+  ticketHash: string;
+  ttlSeconds: number;
+}
+
 export type SeedProfile = "development" | "demo";
 
 type NpcSeed = { handle: string; displayName: string; rating: number; avatarKey: string };
@@ -177,6 +183,36 @@ class PostgresRepository implements AppRepository {
     await sql`delete from sessions where token = ${token}`.execute(this.db);
   }
 
+  async createWsTicket(input: CreateWsTicketInput): Promise<void> {
+    assertWsTicketHash(input.ticketHash);
+    assertTicketTtl(input.ttlSeconds);
+    await sql`
+      insert into ws_tickets (ticket_hash, user_id, expires_at)
+      values (
+        ${input.ticketHash},
+        ${input.userId},
+        now() + (${input.ttlSeconds} * interval '1 second')
+      )
+    `.execute(this.db);
+  }
+
+  async consumeWsTicket(ticketHash: string): Promise<SessionUser | null> {
+    assertWsTicketHash(ticketHash);
+    const result = await sql<UserRow>`
+      with consumed as (
+        delete from ws_tickets
+        where ticket_hash = ${ticketHash}
+        returning user_id, expires_at
+      )
+      select u.*
+      from consumed c
+      join users u on u.id = c.user_id
+      where c.expires_at > now() and u.status = 'active'
+      limit 1
+    `.execute(this.db);
+    return result.rows[0] ? toSessionUser(result.rows[0], true) : null;
+  }
+
   async setUserRoleByHandle(handle: string, role: UserRole): Promise<PublicUser> {
     const result = await sql<UserRow>`
       update users
@@ -698,6 +734,18 @@ function normalizeHandle(value: string): string {
   return value.trim().toLowerCase().replace(/[^a-z0-9_-]/g, "-").replace(/-+/g, "-").replace(/^-|-$/g, "") || "player";
 }
 
+function assertWsTicketHash(value: string): void {
+  if (!/^[a-f0-9]{64}$/.test(value)) {
+    throw new Error("invalid websocket ticket hash");
+  }
+}
+
+function assertTicketTtl(value: number): void {
+  if (!Number.isInteger(value) || value < 0) {
+    throw new Error("invalid websocket ticket ttl");
+  }
+}
+
 function avatarFor(handle: string): string {
   const avatars = ["blue", "green", "amber", "violet", "rose"];
   return avatars[Math.abs([...handle].reduce((sum, char) => sum + char.charCodeAt(0), 0)) % avatars.length];
diff --git a/packages/db/src/schema.ts b/packages/db/src/schema.ts
index f716d23..f494a78 100644
--- a/packages/db/src/schema.ts
+++ b/packages/db/src/schema.ts
@@ -24,6 +24,13 @@ export interface SessionTable {
   created_at: Generated<Date>;
 }
 
+export interface WsTicketTable {
+  ticket_hash: string;
+  user_id: string;
+  expires_at: Date;
+  created_at: Generated<Date>;
+}
+
 export interface MatchTable {
   id: Generated<string>;
   mode: import("@pong-pong/shared").MatchMode;
@@ -103,6 +110,7 @@ export type AdminActionRow = Selectable<AdminActionTable>;
 export interface Database {
   users: UserTable;
   sessions: SessionTable;
+  ws_tickets: WsTicketTable;
   matches: MatchTable;
   friendships: FriendshipTable;
   chat_messages: ChatMessageTable;


## `feat(db): memory WebSocket ticket 소비 구현`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index e349f9f..66116a9 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -60,6 +60,8 @@ export interface AppRepository {
   createSession(userId: string): Promise<string>;
   getSessionUser(token: string | undefined): Promise<SessionUser | null>;
   deleteSession(token: string | undefined): Promise<void>;
+  createWsTicket(input: CreateWsTicketInput): Promise<void>;
+  consumeWsTicket(ticketHash: string): Promise<SessionUser | null>;
   setUserRoleByHandle(handle: string, role: UserRole): Promise<PublicUser>;
   getUserById(id: string): Promise<PublicUser | null>;
   getUserByHandle(handle: string): Promise<PublicUser | null>;
@@ -467,6 +469,7 @@ class PostgresRepository implements AppRepository {
 class MemoryRepository implements AppRepository {
   private readonly users = new Map<string, MemoryUserRow>();
   private readonly sessions = new Map<string, string>();
+  private readonly wsTickets = new Map<string, { userId: string; expiresAt: number }>();
   private readonly matches: MemoryMatchRecord[] = [];
   private readonly friendships: FriendSummary[] = [];
   private readonly chats: ChatMessage[] = [];
@@ -543,6 +546,25 @@ class MemoryRepository implements AppRepository {
     if (token) this.sessions.delete(token);
   }
 
+  async createWsTicket(input: CreateWsTicketInput): Promise<void> {
+    assertWsTicketHash(input.ticketHash);
+    assertTicketTtl(input.ttlSeconds);
+    this.wsTickets.set(input.ticketHash, {
+      userId: input.userId,
+      expiresAt: Date.now() + input.ttlSeconds * 1_000
+    });
+  }
+
+  async consumeWsTicket(ticketHash: string): Promise<SessionUser | null> {
+    assertWsTicketHash(ticketHash);
+    const ticket = this.wsTickets.get(ticketHash);
+    if (!ticket) return null;
+    this.wsTickets.delete(ticketHash);
+    const user = this.users.get(ticket.userId);
+    if (!user || ticket.expiresAt <= Date.now() || user.status !== "active") return null;
+    return toSessionUser(user, true);
+  }
+
   async setUserRoleByHandle(handle: string, role: UserRole): Promise<PublicUser> {
     const user = [...this.users.values()].find((item) => item.handle === normalizeHandle(handle) && !item.is_npc);
     if (!user) throw new Error("user not found");


## `feat(auth): ticket 기반 WebSocket 인증 연결`

diff --git a/apps/api/src/app.ts b/apps/api/src/app.ts
index 006b29a..9d207dc 100644
--- a/apps/api/src/app.ts
+++ b/apps/api/src/app.ts
@@ -5,7 +5,7 @@ import Fastify, { type FastifyRequest } from "fastify";
 import type { AppRepository } from "@pong-pong/db";
 import * as http from "@pong-pong/shared";
 import type { SessionUser } from "@pong-pong/shared";
-import type { WebSocket } from "ws";
+import { WebSocket, type RawData } from "ws";
 import { GameHub } from "./gameHub";
 import {
   forbidden,
@@ -16,6 +16,13 @@ import {
   suspended,
   unauthorized
 } from "./httpBoundary";
+import { createRawWsTicket, hashWsTicket, WS_TICKET_TTL_SECONDS } from "./wsTicket";
+
+const WS_POLICY_VIOLATION = 1008;
+const WS_MESSAGE_TOO_BIG = 1009;
+const PRE_AUTH_MESSAGE_MAX_BYTES = 8 * 1024;
+const PRE_AUTH_MESSAGE_MAX_COUNT = 16;
+const PRE_AUTH_BUFFER_MAX_BYTES = 32 * 1024;
 
 export type AppMode = "development" | "test" | "production" | "demo";
 
@@ -41,22 +48,57 @@ export function buildApp({ repo, webOrigin, appMode = readAppMode() }: BuildAppO
     await realtime.register(websocket);
     realtime.get("/ws", { websocket: true }, (socket, request) => {
       const pendingPayloads: string[] = [];
-      const bufferPayload = (payload: Buffer) => pendingPayloads.push(payload.toString());
+      let pendingBytes = 0;
+      let authenticationClosed = false;
+      const closeAuthentication = (code: number, reason: string) => {
+        if (authenticationClosed) return;
+        authenticationClosed = true;
+        socket.off("message", bufferPayload);
+        socket.close(code, reason);
+      };
+      const bufferPayload = (payload: RawData) => {
+        if (authenticationClosed) return;
+        const buffer = rawDataToBuffer(payload);
+        if (buffer.byteLength > PRE_AUTH_MESSAGE_MAX_BYTES) {
+          closeAuthentication(WS_MESSAGE_TOO_BIG, "pre-auth payload too large");
+          return;
+        }
+        if (
+          pendingPayloads.length >= PRE_AUTH_MESSAGE_MAX_COUNT
+          || pendingBytes + buffer.byteLength > PRE_AUTH_BUFFER_MAX_BYTES
+        ) {
+          closeAuthentication(WS_MESSAGE_TOO_BIG, "pre-auth buffer limit exceeded");
+          return;
+        }
+        pendingBytes += buffer.byteLength;
+        pendingPayloads.push(buffer.toString("utf8"));
+      };
       socket.on("message", bufferPayload);
-      currentUser(repo, request)
+
+      const query = request.query as Record<string, unknown>;
+      if (query?.v !== "1") {
+        closeAuthentication(WS_POLICY_VIOLATION, "unsupported websocket version");
+        return;
+      }
+      const parsedQuery = http.wsHandshakeQuerySchema.safeParse(query);
+      if (!parsedQuery.success) {
+        closeAuthentication(WS_POLICY_VIOLATION, "invalid websocket ticket");
+        return;
+      }
+
+      repo.consumeWsTicket(hashWsTicket(parsedQuery.data.ticket))
         .then((user) => {
           if (!user) {
-            socket.close(1008, "unauthorized");
+            closeAuthentication(WS_POLICY_VIOLATION, "invalid websocket ticket");
             return;
           }
-          if (user.status !== "active") {
-            socket.close(1008, "account suspended");
+          if (authenticationClosed || socket.readyState !== WebSocket.OPEN) {
             return;
           }
           socket.off("message", bufferPayload);
           hub.connect(socket as WebSocket, request.raw, user, pendingPayloads);
         })
-        .catch(() => socket.close(1011, "authentication failed"));
+        .catch(() => closeAuthentication(1011, "websocket authentication failed"));
     });
   });
 
@@ -87,6 +129,25 @@ export function buildApp({ repo, webOrigin, appMode = readAppMode() }: BuildAppO
     return parseOutput(http.okResponseSchema, { ok: true });
   });
 
+  app.post("/auth/ws-ticket", async (request) => {
+    parseInput(http.emptyParamsSchema, request.body ?? {});
+    const user = await currentUser(repo, request);
+    if (!user) unauthorized();
+    if (!isActive(user)) suspended();
+
+    const ticket = createRawWsTicket();
+    await repo.createWsTicket({
+      userId: user.id,
+      ticketHash: hashWsTicket(ticket),
+      ttlSeconds: WS_TICKET_TTL_SECONDS
+    });
+    return parseOutput(http.wsTicketResponseSchema, {
+      ticket,
+      expiresInSeconds: WS_TICKET_TTL_SECONDS,
+      protocolVersion: 1
+    });
+  });
+
   app.get("/me", async (request) => {
     const user = await currentUser(repo, request);
     if (!user) unauthorized();
@@ -275,3 +336,9 @@ function readAppMode(input = process.env): AppMode {
 function useSecureCookies(mode: AppMode): boolean {
   return mode === "production" || mode === "demo";
 }
+
+function rawDataToBuffer(payload: RawData): Buffer {
+  if (Array.isArray(payload)) return Buffer.concat(payload);
+  if (payload instanceof ArrayBuffer) return Buffer.from(payload);
+  return Buffer.from(payload.buffer, payload.byteOffset, payload.byteLength);
+}


