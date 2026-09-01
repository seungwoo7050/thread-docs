## `feat(api): 로비·친구 HTTP contract 적용`

diff --git a/apps/api/src/app.ts b/apps/api/src/app.ts
index fdf3d62..e52cc33 100644
--- a/apps/api/src/app.ts
+++ b/apps/api/src/app.ts
@@ -12,6 +12,7 @@ import {
   notFound,
   parseInput,
   parseOutput,
+  suspended as raiseSuspended,
   unauthorized as raiseUnauthorized
 } from "./httpBoundary";
 
@@ -100,39 +101,38 @@ export function buildApp({ repo, webOrigin }: BuildAppOptions) {
 
   app.get("/lobby", async (request) => {
     const user = await currentUser(repo, request);
-    return {
+    return parseOutput(http.lobbyResponseSchema, {
       me: user,
       onlinePlayers: hub.onlinePlayers(),
       recentMatches: await repo.listRecentMatches(user?.id),
       chat: await repo.listLobbyChat(),
       stats: hub.liveStats()
-    };
+    });
   });
 
-  app.post("/chat/lobby", async (request, reply) => {
+  app.post("/chat/lobby", async (request) => {
     const user = await currentUser(repo, request);
-    if (!user) return unauthorized(reply);
-    if (!isActive(user)) return suspended(reply);
-    const body = (request.body ?? {}) as { body?: string };
-    const messageBody = body.body?.trim() ?? "";
-    if (!messageBody) return reply.code(400).send({ message: "메시지를 입력해주세요." });
-    if (messageBody.length > 240) return reply.code(400).send({ message: "메시지는 240자 이내로 입력해주세요." });
-    return {
+    if (!user) raiseUnauthorized();
+    if (!isActive(user)) raiseSuspended();
+    const body = parseInput(http.chatBodySchema, request.body);
+    return parseOutput(http.chatResponseSchema, {
       message: await repo.createChatMessage({
         scope: "lobby",
         roomId: null,
         senderId: user.id,
-        body: messageBody
+        body: body.body
       })
-    };
+    });
   });
 
-  app.get("/leaderboard", async () => ({ entries: await repo.listLeaderboard() }));
+  app.get("/leaderboard", async () => parseOutput(http.leaderboardResponseSchema, {
+    entries: await repo.listLeaderboard()
+  }));
 
-  app.get("/dashboard", async (request, reply) => {
+  app.get("/dashboard", async (request) => {
     const user = await currentUser(repo, request);
-    if (!user) return unauthorized(reply);
-    return await repo.getDashboard(user.id);
+    if (!user) raiseUnauthorized();
+    return parseOutput(http.dashboardSummarySchema, await repo.getDashboard(user.id));
   });
 
   app.get("/profile/:handle", async (request) => {
@@ -160,33 +160,30 @@ export function buildApp({ repo, webOrigin }: BuildAppOptions) {
     });
   });
 
-  app.get("/friends", async (request, reply) => {
+  app.get("/friends", async (request) => {
     const user = await currentUser(repo, request);
-    if (!user) return unauthorized(reply);
-    return { friends: await repo.listFriends(user.id) };
+    if (!user) raiseUnauthorized();
+    return parseOutput(http.friendsResponseSchema, { friends: await repo.listFriends(user.id) });
   });
 
-  app.post("/friends/request", async (request, reply) => {
+  const requestFriend = async (request: FastifyRequest) => {
     const user = await currentUser(repo, request);
-    if (!user) return unauthorized(reply);
-    if (!isActive(user)) return suspended(reply);
-    const body = request.body as { handle?: string };
-    return { friend: await repo.requestFriend(user.id, body.handle ?? "") };
-  });
+    if (!user) raiseUnauthorized();
+    if (!isActive(user)) raiseSuspended();
+    const body = parseInput(http.friendRequestBodySchema, request.body);
+    return parseOutput(http.friendResponseSchema, {
+      friend: await repo.requestFriend(user.id, body.handle)
+    });
+  };
 
-  app.post("/friends", async (request, reply) => {
-    const user = await currentUser(repo, request);
-    if (!user) return unauthorized(reply);
-    if (!isActive(user)) return suspended(reply);
-    const body = request.body as { handle?: string };
-    return { friend: await repo.requestFriend(user.id, body.handle ?? "") };
-  });
+  app.post("/friends/request", requestFriend);
+  app.post("/friends", requestFriend);
 
-  app.post("/friends/:id/accept", async (request, reply) => {
+  app.post("/friends/:id/accept", async (request) => {
     const user = await currentUser(repo, request);
-    if (!user) return unauthorized(reply);
-    const { id } = request.params as { id: string };
-    return { friend: await repo.acceptFriend(user.id, id) };
+    if (!user) raiseUnauthorized();
+    const { id } = parseInput(http.idParamsSchema, request.params);
+    return parseOutput(http.friendResponseSchema, { friend: await repo.acceptFriend(user.id, id) });
   });
 
   app.get("/tournaments", async () => ({ tournaments: await repo.listTournaments() }));


## `feat(api): 토너먼트·관리 HTTP contract 적용`

diff --git a/apps/api/src/app.ts b/apps/api/src/app.ts
index e52cc33..adaefbf 100644
--- a/apps/api/src/app.ts
+++ b/apps/api/src/app.ts
@@ -8,6 +8,7 @@ import type { SessionUser } from "@pong-pong/shared";
 import type { WebSocket } from "ws";
 import { GameHub } from "./gameHub";
 import {
+  forbidden as raiseForbidden,
   installHttpErrorBoundary,
   notFound,
   parseInput,
@@ -186,54 +187,54 @@ export function buildApp({ repo, webOrigin }: BuildAppOptions) {
     return parseOutput(http.friendResponseSchema, { friend: await repo.acceptFriend(user.id, id) });
   });
 
-  app.get("/tournaments", async () => ({ tournaments: await repo.listTournaments() }));
+  app.get("/tournaments", async () => parseOutput(http.tournamentsResponseSchema, {
+    tournaments: await repo.listTournaments()
+  }));
 
-  app.post("/tournaments", async (request, reply) => {
+  app.post("/tournaments", async (request) => {
     const user = await currentUser(repo, request);
-    if (!user) return unauthorized(reply);
-    if (!isActive(user)) return suspended(reply);
-    const body = request.body as { name?: string };
-    return { tournament: await repo.createTournament({ name: body.name ?? "퐁퐁 주간 컵", createdBy: user.id }) };
+    if (!user) raiseUnauthorized();
+    if (!isActive(user)) raiseSuspended();
+    const body = parseInput(http.tournamentCreateBodySchema, request.body);
+    return parseOutput(http.tournamentResponseSchema, {
+      tournament: await repo.createTournament({ name: body.name, createdBy: user.id })
+    });
   });
 
-  app.post("/tournaments/:id/join", async (request, reply) => {
+  app.post("/tournaments/:id/join", async (request) => {
     const user = await currentUser(repo, request);
-    if (!user) return unauthorized(reply);
-    if (!isActive(user)) return suspended(reply);
-    const { id } = request.params as { id: string };
-    return { tournament: await repo.joinTournament(id, user.id) };
+    if (!user) raiseUnauthorized();
+    if (!isActive(user)) raiseSuspended();
+    const { id } = parseInput(http.idParamsSchema, request.params);
+    return parseOutput(http.tournamentResponseSchema, { tournament: await repo.joinTournament(id, user.id) });
   });
 
-  app.get("/admin/users", async (request, reply) => {
-    const user = await currentUser(repo, request);
-    if (!user) return unauthorized(reply);
-    if (user.role !== "admin") return reply.code(403).send({ message: "운영자 권한이 필요합니다." });
-    return { users: await repo.listAdminUsers() };
+  app.get("/admin/users", async (request) => {
+    const user = await requireAdmin(repo, request);
+    return parseOutput(http.adminUsersResponseSchema, { users: await repo.listAdminUsers() });
   });
 
-  app.get("/admin/actions", async (request, reply) => {
-    const user = await currentUser(repo, request);
-    if (!user) return unauthorized(reply);
-    if (user.role !== "admin") return reply.code(403).send({ message: "운영자 권한이 필요합니다." });
-    return { actions: await repo.listAdminActions() };
+  app.get("/admin/actions", async (request) => {
+    await requireAdmin(repo, request);
+    return parseOutput(http.adminActionsResponseSchema, { actions: await repo.listAdminActions() });
   });
 
-  app.post("/admin/users/:id/ban", async (request, reply) => {
-    const user = await currentUser(repo, request);
-    if (!user) return unauthorized(reply);
-    if (user.role !== "admin") return reply.code(403).send({ message: "운영자 권한이 필요합니다." });
-    const { id } = request.params as { id: string };
-    const body = request.body as { banned?: boolean; reason?: string };
-    return { user: await repo.setUserBan(user.id, id, body.banned ?? true, body.reason ?? "manual review") };
+  app.post("/admin/users/:id/ban", async (request) => {
+    const user = await requireAdmin(repo, request);
+    const { id } = parseInput(http.idParamsSchema, request.params);
+    const body = parseInput(http.adminBanBodySchema, request.body ?? {});
+    return parseOutput(http.publicUserResponseSchema, {
+      user: await repo.setUserBan(user.id, id, body.banned ?? true, body.reason ?? "manual review")
+    });
   });
 
-  app.patch("/admin/users/:id/status", async (request, reply) => {
-    const user = await currentUser(repo, request);
-    if (!user) return unauthorized(reply);
-    if (user.role !== "admin") return reply.code(403).send({ message: "운영자 권한이 필요합니다." });
-    const { id } = request.params as { id: string };
-    const body = request.body as { status?: "active" | "banned"; reason?: string };
-    return { user: await repo.setUserBan(user.id, id, body.status === "banned", body.reason ?? "manual review") };
+  app.patch("/admin/users/:id/status", async (request) => {
+    const user = await requireAdmin(repo, request);
+    const { id } = parseInput(http.idParamsSchema, request.params);
+    const body = parseInput(http.adminStatusBodySchema, request.body);
+    return parseOutput(http.publicUserResponseSchema, {
+      user: await repo.setUserBan(user.id, id, body.status === "banned", body.reason ?? "manual review")
+    });
   });
 
   return app;
@@ -259,6 +260,13 @@ function suspended(reply: FastifyReply) {
   return reply.code(403).send({ message: "정지된 계정은 이 작업을 수행할 수 없습니다." });
 }
 
+async function requireAdmin(repo: AppRepository, request: FastifyRequest): Promise<SessionUser> {
+  const user = await currentUser(repo, request);
+  if (!user) raiseUnauthorized();
+  if (user.role !== "admin") raiseForbidden();
+  return user;
+}
+
 function isActive(user: SessionUser): boolean {
   return user.status === "active";
 }


## `feat(shared): 모든 HTTP request schema를 strict하게 정의`

diff --git a/packages/shared/src/http.ts b/packages/shared/src/http.ts
index b63d418..aab95f6 100644
--- a/packages/shared/src/http.ts
+++ b/packages/shared/src/http.ts
@@ -183,6 +183,71 @@ export const adminStatusBodySchema = z.object({
   reason: z.string().trim().min(1).max(240).optional()
 }).strict();
 
+function defineHttpRequestContract<
+  Params extends z.ZodTypeAny,
+  Query extends z.ZodTypeAny,
+  Body extends z.ZodTypeAny
+>(params: Params, query: Query, body: Body) {
+  return { params, query, body } as const;
+}
+
+const emptyHttpRequestContract = defineHttpRequestContract(
+  emptyParamsSchema,
+  emptyParamsSchema,
+  emptyParamsSchema
+);
+const idHttpRequestContract = defineHttpRequestContract(
+  idParamsSchema,
+  emptyParamsSchema,
+  emptyParamsSchema
+);
+
+export const jsonHttpRequestContracts = {
+  health: emptyHttpRequestContract,
+  healthLive: emptyHttpRequestContract,
+  healthReady: emptyHttpRequestContract,
+  devLogin: defineHttpRequestContract(emptyParamsSchema, emptyParamsSchema, devLoginBodySchema),
+  guestLogin: emptyHttpRequestContract,
+  logout: emptyHttpRequestContract,
+  wsTicket: emptyHttpRequestContract,
+  me: emptyHttpRequestContract,
+  authMe: emptyHttpRequestContract,
+  userById: idHttpRequestContract,
+  lobby: emptyHttpRequestContract,
+  lobbyChat: defineHttpRequestContract(emptyParamsSchema, emptyParamsSchema, chatBodySchema),
+  leaderboard: emptyHttpRequestContract,
+  dashboard: emptyHttpRequestContract,
+  profileByHandle: defineHttpRequestContract(
+    handleParamsSchema,
+    emptyParamsSchema,
+    emptyParamsSchema
+  ),
+  ownProfile: emptyHttpRequestContract,
+  updateOwnProfile: defineHttpRequestContract(
+    emptyParamsSchema,
+    emptyParamsSchema,
+    profileUpdateBodySchema
+  ),
+  friends: emptyHttpRequestContract,
+  requestFriend: defineHttpRequestContract(
+    emptyParamsSchema,
+    emptyParamsSchema,
+    friendRequestBodySchema
+  ),
+  acceptFriend: idHttpRequestContract,
+  tournaments: emptyHttpRequestContract,
+  createTournament: defineHttpRequestContract(
+    emptyParamsSchema,
+    emptyParamsSchema,
+    tournamentCreateBodySchema
+  ),
+  joinTournament: idHttpRequestContract,
+  adminUsers: emptyHttpRequestContract,
+  adminActions: emptyHttpRequestContract,
+  adminBan: defineHttpRequestContract(idParamsSchema, emptyParamsSchema, adminBanBodySchema),
+  adminStatus: defineHttpRequestContract(idParamsSchema, emptyParamsSchema, adminStatusBodySchema)
+} as const;
+
 export const wsTicketSchema = z.string().regex(/^[A-Za-z0-9_-]{43}$/);
 export const wsHandshakeQuerySchema = z.object({
   ticket: wsTicketSchema,


