# 쿠키 세션에서 일회용 WebSocket 티켓까지의 인증 경계

## `feat(db): 사용자 session 저장 구현`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index 71f9e11..84b89b7 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -18,6 +18,8 @@ export interface AppRepository {
   close(): Promise<void>;
   ensureSeedData(): Promise<void>;
   upsertDevUser(input: DevLoginInput): Promise<SessionUser>;
+  createSession(userId: string): Promise<string>;
+  getSessionUser(token: string | undefined): Promise<SessionUser | null>;
 }
 
 export function createPostgresRepository(databaseUrl: string): AppRepository {
@@ -75,10 +77,33 @@ class PostgresRepository implements AppRepository {
     return toSessionUser(firstRow(result));
   }
 
+  async createSession(userId: string): Promise<string> {
+    const token = randomUUID();
+    await sql`
+      insert into sessions (token, user_id, expires_at)
+      values (${token}, ${userId}, now() + interval '14 days')
+    `.execute(this.db);
+    return token;
+  }
+
+  async getSessionUser(token: string | undefined): Promise<SessionUser | null> {
+    if (!token) return null;
+    const result = await sql<UserRow>`
+      select u.*
+      from sessions s
+      join users u on u.id = s.user_id
+      where s.token = ${token} and s.expires_at > now()
+      limit 1
+    `.execute(this.db);
+    const user = result.rows[0];
+    return user ? toSessionUser(user, true) : null;
+  }
+
 }
 
 class MemoryRepository implements AppRepository {
   private readonly users = new Map<string, MemoryUserRow>();
+  private readonly sessions = new Map<string, string>();
 
   async close(): Promise<void> {}
 
@@ -113,6 +138,18 @@ class MemoryRepository implements AppRepository {
     return toSessionUser(user, true);
   }
 
+  async createSession(userId: string): Promise<string> {
+    const token = randomUUID();
+    this.sessions.set(token, userId);
+    return token;
+  }
+
+  async getSessionUser(token: string | undefined): Promise<SessionUser | null> {
+    const userId = token ? this.sessions.get(token) : undefined;
+    const user = userId ? this.users.get(userId) : undefined;
+    return user ? toSessionUser(user, true) : null;
+  }
+
 }
 
 function firstRow<T>(result: { rows: T[] }): T {


## `feat(api): 로그인과 로비 HTTP 경계 구현`

diff --git a/apps/api/src/app.ts b/apps/api/src/app.ts
new file mode 100644
index 0000000..0ae4ceb
--- /dev/null
+++ b/apps/api/src/app.ts
@@ -0,0 +1,80 @@
+import cookie from "@fastify/cookie";
+import cors from "@fastify/cors";
+import Fastify, { type FastifyReply, type FastifyRequest } from "fastify";
+import type { AppRepository } from "@pong-pong/db";
+import type { SessionUser } from "@pong-pong/shared";
+
+export interface BuildAppOptions {
+  repo: AppRepository;
+  webOrigin: string;
+}
+
+export function buildApp({ repo, webOrigin }: BuildAppOptions) {
+  const app = Fastify({ logger: { level: process.env.LOG_LEVEL ?? "info" } });
+  app.register(cors, {
+    origin: [webOrigin, "http://localhost:3000", "http://localhost:8080"],
+    credentials: true
+  });
+  app.register(cookie);
+  app.get("/health", async () => ({ ok: true, service: "pong-pong-api" }));
+
+  app.post("/auth/dev-login", async (request, reply) => {
+    const body = request.body as { handle?: string; displayName?: string; email?: string };
+    const user = await repo.upsertDevUser({
+      handle: body.handle ?? "player",
+      displayName: body.displayName ?? body.handle ?? "플레이어",
+      email: body.email
+    });
+    const token = await repo.createSession(user.id);
+    reply.setCookie("pp_session", token, {
+      path: "/",
+      sameSite: "lax",
+      httpOnly: true,
+      maxAge: 60 * 60 * 24 * 14
+    });
+    return { user, token };
+  });
+
+  app.post("/auth/logout", async (_request, reply) => {
+    reply.clearCookie("pp_session", { path: "/" });
+    return { ok: true };
+  });
+
+  app.get("/me", async (request, reply) => {
+    const user = await currentUser(repo, request);
+    if (!user) return unauthorized(reply);
+    return { user };
+  });
+
+  app.get("/auth/me", async (request, reply) => {
+    const user = await currentUser(repo, request);
+    if (!user) return unauthorized(reply);
+    return { user };
+  });
+
+  app.get("/lobby", async (request) => {
+    const user = await currentUser(repo, request);
+    return {
+      me: user,
+      onlinePlayers: await repo.listOnlineUsers(),
+      recentMatches: await repo.listRecentMatches(user?.id),
+      chat: await repo.listLobbyChat()
+    };
+  });
+
+  app.get("/leaderboard", async () => ({ entries: await repo.listLeaderboard() }));
+
+  return app;
+}
+
+async function currentUser(repo: AppRepository, request: FastifyRequest): Promise<SessionUser | null> {
+  const cookieToken = request.cookies?.pp_session;
+  const header = request.headers.authorization?.replace(/^Bearer\s+/i, "");
+  const queryToken = (request.query as { session?: string } | undefined)?.session;
+  const rawQueryToken = new URL(request.raw.url ?? "/", "http://localhost").searchParams.get("session") ?? undefined;
+  return repo.getSessionUser(cookieToken ?? header ?? queryToken ?? rawQueryToken);
+}
+
+function unauthorized(reply: FastifyReply) {
+  return reply.code(401).send({ message: "로그인이 필요합니다." });
+}


## `fix(auth): 인증 완료 전 WebSocket 입력 보존`

diff --git a/apps/api/src/app.ts b/apps/api/src/app.ts
index 0afad32..bc04acb 100644
--- a/apps/api/src/app.ts
+++ b/apps/api/src/app.ts
@@ -15,6 +15,7 @@ export interface BuildAppOptions {
 export function buildApp({ repo, webOrigin }: BuildAppOptions) {
   const app = Fastify({ logger: { level: process.env.LOG_LEVEL ?? "info" } });
   const hub = new GameHub(repo);
+
   app.register(cors, {
     origin: [webOrigin, "http://localhost:3000", "http://localhost:8080"],
     credentials: true
@@ -23,17 +24,22 @@ export function buildApp({ repo, webOrigin }: BuildAppOptions) {
   app.register(async (realtime) => {
     await realtime.register(websocket);
     realtime.get("/ws", { websocket: true }, (socket, request) => {
+      const pendingPayloads: string[] = [];
+      const bufferPayload = (payload: Buffer) => pendingPayloads.push(payload.toString());
+      socket.on("message", bufferPayload);
       currentUser(repo, request)
         .then((user) => {
           if (!user) {
             socket.close(1008, "unauthorized");
             return;
           }
-          hub.connect(socket as WebSocket, request.raw, user);
+          socket.off("message", bufferPayload);
+          hub.connect(socket as WebSocket, request.raw, user, pendingPayloads);
         })
         .catch(() => socket.close(1011, "authentication failed"));
     });
   });
+
   app.get("/health", async () => ({ ok: true, service: "pong-pong-api" }));
 
   app.post("/auth/dev-login", async (request, reply) => {
@@ -99,7 +105,10 @@ export function buildApp({ repo, webOrigin }: BuildAppOptions) {
     const { handle } = request.params as { handle: string };
     const user = await repo.getUserByHandle(handle);
     if (!user) return reply.code(404).send({ message: "프로필을 찾을 수 없습니다." });
-    return { user, recentMatches: await repo.listRecentMatches(user.id) };
+    return {
+      user,
+      recentMatches: await repo.listRecentMatches(user.id)
+    };
   });
 
   app.get("/profile/me", async (request, reply) => {
@@ -112,7 +121,12 @@ export function buildApp({ repo, webOrigin }: BuildAppOptions) {
     const user = await currentUser(repo, request);
     if (!user) return unauthorized(reply);
     const body = request.body as { displayName?: string; avatarKey?: string };
-    return { profile: await repo.updateProfile(user.id, body) };
+    return {
+      profile: await repo.updateProfile(user.id, {
+        displayName: body.displayName,
+        avatarKey: body.avatarKey
+      })
+    };
   });
 
   app.get("/friends", async (request, reply) => {
diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index 3d6780c..9b8b8ca 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -41,12 +41,15 @@ export class GameHub {
 
   constructor(private readonly repo: AppRepository) {}
 
-  connect(socket: WebSocket, request: IncomingMessage, user: SessionUser): void {
+  connect(socket: WebSocket, request: IncomingMessage, user: SessionUser, pendingPayloads: string[] = []): void {
     const client: Client = { id: randomUUID(), socket, user, roomId: null };
     this.clients.set(client.id, client);
     socket.on("message", (payload) => this.receive(client, payload.toString()));
     socket.on("close", () => this.disconnect(client));
     this.broadcastPresence();
+    for (const payload of pendingPayloads) {
+      this.receive(client, payload).catch(() => undefined);
+    }
   }
 
   private async receive(client: Client, payload: string): Promise<void> {


## `fix(api): logout 시 server session 폐기`

diff --git a/apps/api/src/app.ts b/apps/api/src/app.ts
index 142354e..d66c99a 100644
--- a/apps/api/src/app.ts
+++ b/apps/api/src/app.ts
@@ -65,7 +65,8 @@ export function buildApp({ repo, webOrigin }: BuildAppOptions) {
     return { user, token };
   });
 
-  app.post("/auth/logout", async (_request, reply) => {
+  app.post("/auth/logout", async (request, reply) => {
+    await repo.deleteSession(readSessionToken(request));
     reply.clearCookie("pp_session", { path: "/" });
     return { ok: true };
   });
@@ -236,12 +237,16 @@ export function buildApp({ repo, webOrigin }: BuildAppOptions) {
   return app;
 }
 
-async function currentUser(repo: AppRepository, request: FastifyRequest): Promise<SessionUser | null> {
+function readSessionToken(request: FastifyRequest): string | undefined {
   const cookieToken = request.cookies?.pp_session;
   const header = request.headers.authorization?.replace(/^Bearer\s+/i, "");
   const queryToken = (request.query as { session?: string } | undefined)?.session;
   const rawQueryToken = new URL(request.raw.url ?? "/", "http://localhost").searchParams.get("session") ?? undefined;
-  return repo.getSessionUser(cookieToken ?? header ?? queryToken ?? rawQueryToken);
+  return cookieToken ?? header ?? queryToken ?? rawQueryToken;
+}
+
+async function currentUser(repo: AppRepository, request: FastifyRequest): Promise<SessionUser | null> {
+  return repo.getSessionUser(readSessionToken(request));
 }
 
 function unauthorized(reply: FastifyReply) {
diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index b31c811..10f2f11 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -52,6 +52,7 @@ export interface AppRepository {
   upsertDevUser(input: DevLoginInput): Promise<SessionUser>;
   createSession(userId: string): Promise<string>;
   getSessionUser(token: string | undefined): Promise<SessionUser | null>;
+  deleteSession(token: string | undefined): Promise<void>;
   getUserById(id: string): Promise<PublicUser | null>;
   getUserByHandle(handle: string): Promise<PublicUser | null>;
   updateProfile(userId: string, input: { displayName?: string; avatarKey?: string }): Promise<SessionUser>;
@@ -165,6 +166,11 @@ class PostgresRepository implements AppRepository {
     return user ? toSessionUser(user, true) : null;
   }
 
+  async deleteSession(token: string | undefined): Promise<void> {
+    if (!token) return;
+    await sql`delete from sessions where token = ${token}`.execute(this.db);
+  }
+
   async getUserById(id: string): Promise<PublicUser | null> {
     const result = await sql<UserRow>`select * from users where id = ${id} limit 1`.execute(this.db);
     return result.rows[0] ? toPublicUser(result.rows[0]) : null;
@@ -471,6 +477,10 @@ class MemoryRepository implements AppRepository {
     return user ? toSessionUser(user, true) : null;
   }
 
+  async deleteSession(token: string | undefined): Promise<void> {
+    if (token) this.sessions.delete(token);
+  }
+
   async getUserById(id: string): Promise<PublicUser | null> {
     const user = this.users.get(id);
     return user ? toPublicUser(user, true) : null;


## `fix(auth): cookie-only session과 환경별 route 적용`

diff --git a/apps/api/src/app.ts b/apps/api/src/app.ts
index cb25641..006b29a 100644
--- a/apps/api/src/app.ts
+++ b/apps/api/src/app.ts
@@ -17,12 +17,15 @@ import {
   unauthorized
 } from "./httpBoundary";
 
+export type AppMode = "development" | "test" | "production" | "demo";
+
 export interface BuildAppOptions {
   repo: AppRepository;
   webOrigin: string;
+  appMode?: AppMode;
 }
 
-export function buildApp({ repo, webOrigin }: BuildAppOptions) {
+export function buildApp({ repo, webOrigin, appMode = readAppMode() }: BuildAppOptions) {
   const app = Fastify({ logger: { level: process.env.LOG_LEVEL ?? "info" } });
   const hub = new GameHub(repo);
 
@@ -31,7 +34,7 @@ export function buildApp({ repo, webOrigin }: BuildAppOptions) {
     origin: [webOrigin, "http://localhost:3000", "http://localhost:8080"],
     credentials: true,
     methods: ["GET", "POST", "PATCH", "DELETE", "OPTIONS"],
-    allowedHeaders: ["content-type", "authorization", "x-request-id"]
+    allowedHeaders: ["content-type", "x-request-id"]
   });
   app.register(cookie);
   app.register(async (realtime) => {
@@ -62,18 +65,21 @@ export function buildApp({ repo, webOrigin }: BuildAppOptions) {
     service: "pong-pong-api"
   }));
 
-  app.post("/auth/dev-login", async (request, reply) => {
-    const body = parseInput(http.devLoginBodySchema, request.body);
-    const user = await repo.upsertDevUser(body);
-    const token = await repo.createSession(user.id);
-    reply.setCookie("pp_session", token, {
-      path: "/",
-      sameSite: "lax",
-      httpOnly: true,
-      maxAge: 60 * 60 * 24 * 14
+  if (appMode === "development" || appMode === "test") {
+    app.post("/auth/dev-login", async (request, reply) => {
+      const body = parseInput(http.devLoginBodySchema, request.body);
+      const user = await repo.upsertDevUser(body);
+      const token = await repo.createSession(user.id);
+      reply.setCookie("pp_session", token, {
+        path: "/",
+        sameSite: "lax",
+        httpOnly: true,
+        secure: useSecureCookies(appMode),
+        maxAge: 60 * 60 * 24 * 14
+      });
+      return parseOutput(http.userResponseSchema, { user });
     });
-    return parseOutput(http.userResponseSchema, { user });
-  });
+  }
 
   app.post("/auth/logout", async (request, reply) => {
     await repo.deleteSession(readSessionToken(request));
@@ -241,11 +247,7 @@ export function buildApp({ repo, webOrigin }: BuildAppOptions) {
 }
 
 function readSessionToken(request: FastifyRequest): string | undefined {
-  const cookieToken = request.cookies?.pp_session;
-  const header = request.headers.authorization?.replace(/^Bearer\s+/i, "");
-  const queryToken = (request.query as { session?: string } | undefined)?.session;
-  const rawQueryToken = new URL(request.raw.url ?? "/", "http://localhost").searchParams.get("session") ?? undefined;
-  return cookieToken ?? header ?? queryToken ?? rawQueryToken;
+  return request.cookies?.pp_session;
 }
 
 async function currentUser(repo: AppRepository, request: FastifyRequest): Promise<SessionUser | null> {
@@ -262,3 +264,14 @@ async function requireAdmin(repo: AppRepository, request: FastifyRequest): Promi
 function isActive(user: SessionUser): boolean {
   return user.status === "active";
 }
+
+function readAppMode(input = process.env): AppMode {
+  if (input.APP_MODE === "demo") return "demo";
+  if (input.NODE_ENV === "production") return "production";
+  if (input.NODE_ENV === "test") return "test";
+  return "development";
+}
+
+function useSecureCookies(mode: AppMode): boolean {
+  return mode === "production" || mode === "demo";
+}


