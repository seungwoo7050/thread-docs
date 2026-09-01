## `fix(api): 모든 route input을 runtime 검증`

diff --git a/apps/api/src/app.ts b/apps/api/src/app.ts
index a9f93f0..846fb03 100644
--- a/apps/api/src/app.ts
+++ b/apps/api/src/app.ts
@@ -12,7 +12,7 @@ import {
   forbidden,
   installHttpErrorBoundary,
   notFound,
-  parseInput,
+  parseHttpRequest,
   parseOutput,
   suspended,
   unauthorized
@@ -193,17 +193,24 @@ export function buildApp({
     });
   });
 
-  app.get("/health", async () => parseOutput(http.healthResponseSchema, {
-    ok: true,
-    service: "pong-pong-api"
-  }));
+  app.get("/health", async (request) => {
+    parseHttpRequest(http.jsonHttpRequestContracts.health, request);
+    return parseOutput(http.healthResponseSchema, {
+      ok: true,
+      service: "pong-pong-api"
+    });
+  });
 
-  app.get("/health/live", async () => parseOutput(http.liveHealthResponseSchema, {
-    status: "ok",
-    service: "pong-pong-api"
-  }));
+  app.get("/health/live", async (request) => {
+    parseHttpRequest(http.jsonHttpRequestContracts.healthLive, request);
+    return parseOutput(http.liveHealthResponseSchema, {
+      status: "ok",
+      service: "pong-pong-api"
+    });
+  });
 
   app.get("/health/ready", async (request, reply) => {
+    parseHttpRequest(http.jsonHttpRequestContracts.healthReady, request);
     const startedAt = performance.now();
     try {
       const repository = await repo.checkReadiness();
@@ -244,7 +251,7 @@ export function buildApp({
 
   if (appMode === "development" || appMode === "test") {
     app.post("/auth/dev-login", async (request, reply) => {
-      const body = parseInput(http.devLoginBodySchema, request.body);
+      const { body } = parseHttpRequest(http.jsonHttpRequestContracts.devLogin, request);
       const user = await repo.upsertDevUser(body);
       const token = await repo.createSession(user.id);
       reply.setCookie("pp_session", token, {
@@ -260,7 +267,7 @@ export function buildApp({
 
   if (appMode === "demo" && guests) {
     app.post("/auth/guest", async (request, reply) => {
-      parseInput(http.emptyParamsSchema, request.body ?? {});
+      parseHttpRequest(http.jsonHttpRequestContracts.guestLogin, request);
       try {
         const session = guests.createSession(request.ip);
         reply.setCookie("pp_guest", session.cookieValue, {
@@ -285,6 +292,7 @@ export function buildApp({
   }
 
   app.post("/auth/logout", async (request, reply) => {
+    parseHttpRequest(http.jsonHttpRequestContracts.logout, request);
     if (!isGuestSession(await getCurrentUser(request))) {
       await repo.deleteSession(readSessionToken(request));
     }
@@ -294,7 +302,7 @@ export function buildApp({
   });
 
   app.post("/auth/ws-ticket", async (request) => {
-    parseInput(http.emptyParamsSchema, request.body ?? {});
+    parseHttpRequest(http.jsonHttpRequestContracts.wsTicket, request);
     const user = await getCurrentUser(request);
     if (!user) unauthorized();
     if (!isActive(user)) suspended();
@@ -325,26 +333,29 @@ export function buildApp({
   });
 
   app.get("/me", async (request) => {
+    parseHttpRequest(http.jsonHttpRequestContracts.me, request);
     const user = await getCurrentUser(request);
     if (!user) unauthorized();
     return parseOutput(http.userResponseSchema, { user });
   });
 
   app.get("/auth/me", async (request) => {
+    parseHttpRequest(http.jsonHttpRequestContracts.authMe, request);
     const user = await getCurrentUser(request);
     if (!user) unauthorized();
     return parseOutput(http.userResponseSchema, { user });
   });
 
   app.get("/users/:id", async (request) => {
+    const { params: { id } } = parseHttpRequest(http.jsonHttpRequestContracts.userById, request);
     if (appMode === "demo") notFound("데모 모드에서는 제공하지 않는 기능입니다.");
-    const { id } = parseInput(http.idParamsSchema, request.params);
     const user = await repo.getUserById(id);
     if (!user) notFound("사용자를 찾을 수 없습니다.");
     return parseOutput(http.publicUserResponseSchema, { user });
   });
 
   app.get("/lobby", async (request) => {
+    parseHttpRequest(http.jsonHttpRequestContracts.lobby, request);
     const user = await getCurrentUser(request);
     const guest = isGuestSession(user);
     return parseOutput(http.lobbyResponseSchema, {
@@ -357,11 +368,11 @@ export function buildApp({
   });
 
   app.post("/chat/lobby", async (request) => {
+    const { body } = parseHttpRequest(http.jsonHttpRequestContracts.lobbyChat, request);
     const user = await getCurrentUser(request);
     if (!user) unauthorized();
     requireRegistered(user);
     if (!isActive(user)) suspended();
-    const body = parseInput(http.chatBodySchema, request.body);
     return parseOutput(http.chatResponseSchema, {
       message: await repo.createChatMessage({
         scope: "lobby",
@@ -372,12 +383,14 @@ export function buildApp({
     });
   });
 
-  app.get("/leaderboard", async () => {
+  app.get("/leaderboard", async (request) => {
+    parseHttpRequest(http.jsonHttpRequestContracts.leaderboard, request);
     if (appMode === "demo") notFound("데모 모드에서는 제공하지 않는 기능입니다.");
     return parseOutput(http.leaderboardResponseSchema, { entries: await repo.listLeaderboard() });
   });
 
   app.get("/dashboard", async (request) => {
+    parseHttpRequest(http.jsonHttpRequestContracts.dashboard, request);
     const user = await getCurrentUser(request);
     if (!user) unauthorized();
     requireRegistered(user);
@@ -385,8 +398,11 @@ export function buildApp({
   });
 
   app.get("/profile/:handle", async (request) => {
+    const { params: { handle } } = parseHttpRequest(
+      http.jsonHttpRequestContracts.profileByHandle,
+      request
+    );
     if (appMode === "demo") notFound("데모 모드에서는 제공하지 않는 기능입니다.");
-    const { handle } = parseInput(http.handleParamsSchema, request.params);
     const user = await repo.getUserByHandle(handle);
     if (!user) notFound("프로필을 찾을 수 없습니다.");
     return parseOutput(http.profileResponseSchema, {
@@ -396,6 +412,7 @@ export function buildApp({
   });
 
   app.get("/profile/me", async (request) => {
+    parseHttpRequest(http.jsonHttpRequestContracts.ownProfile, request);
     const user = await getCurrentUser(request);
     if (!user) unauthorized();
     requireRegistered(user);
@@ -403,16 +420,17 @@ export function buildApp({
   });
 
   app.patch("/profile/me", async (request) => {
+    const { body } = parseHttpRequest(http.jsonHttpRequestContracts.updateOwnProfile, request);
     const user = await getCurrentUser(request);
     if (!user) unauthorized();
     requireRegistered(user);
-    const body = parseInput(http.profileUpdateBodySchema, request.body);
     return parseOutput(http.ownProfileResponseSchema, {
       profile: await repo.updateProfile(user.id, body)
     });
   });
 
   app.get("/friends", async (request) => {
+    parseHttpRequest(http.jsonHttpRequestContracts.friends, request);
     const user = await getCurrentUser(request);
     if (!user) unauthorized();
     requireRegistered(user);
@@ -420,11 +438,11 @@ export function buildApp({
   });
 
   const requestFriend = async (request: FastifyRequest) => {
+    const { body } = parseHttpRequest(http.jsonHttpRequestContracts.requestFriend, request);
     const user = await getCurrentUser(request);
     if (!user) unauthorized();
     requireRegistered(user);
     if (!isActive(user)) suspended();
-    const body = parseInput(http.friendRequestBodySchema, request.body);
     return parseOutput(http.friendResponseSchema, {
       friend: await repo.requestFriend(user.id, body.handle)
     });
@@ -434,62 +452,75 @@ export function buildApp({
   app.post("/friends", requestFriend);
 
   app.post("/friends/:id/accept", async (request) => {
+    const { params: { id } } = parseHttpRequest(
+      http.jsonHttpRequestContracts.acceptFriend,
+      request
+    );
     const user = await getCurrentUser(request);
     if (!user) unauthorized();
     requireRegistered(user);
-    const { id } = parseInput(http.idParamsSchema, request.params);
     return parseOutput(http.friendResponseSchema, { friend: await repo.acceptFriend(user.id, id) });
   });
 
-  app.get("/tournaments", async () => {
+  app.get("/tournaments", async (request) => {
+    parseHttpRequest(http.jsonHttpRequestContracts.tournaments, request);
     if (appMode === "demo") notFound("데모 모드에서는 제공하지 않는 기능입니다.");
     return parseOutput(http.tournamentsResponseSchema, { tournaments: await repo.listTournaments() });
   });
 
   app.post("/tournaments", async (request) => {
+    const { body } = parseHttpRequest(http.jsonHttpRequestContracts.createTournament, request);
     const user = await getCurrentUser(request);
     if (!user) unauthorized();
     requireRegistered(user);
     if (!isActive(user)) suspended();
-    const body = parseInput(http.tournamentCreateBodySchema, request.body);
     return parseOutput(http.tournamentResponseSchema, {
       tournament: await repo.createTournament({ name: body.name, createdBy: user.id })
     });
   });
 
   app.post("/tournaments/:id/join", async (request) => {
+    const { params: { id } } = parseHttpRequest(
+      http.jsonHttpRequestContracts.joinTournament,
+      request
+    );
     const user = await getCurrentUser(request);
     if (!user) unauthorized();
     requireRegistered(user);
     if (!isActive(user)) suspended();
-    const { id } = parseInput(http.idParamsSchema, request.params);
     return parseOutput(http.tournamentResponseSchema, { tournament: await repo.joinTournament(id, user.id) });
   });
 
   if (appMode !== "demo") {
     app.get("/admin/users", async (request) => {
+      parseHttpRequest(http.jsonHttpRequestContracts.adminUsers, request);
       const user = await requireAdmin(repo, request);
       return parseOutput(http.adminUsersResponseSchema, { users: await repo.listAdminUsers() });
     });
 
     app.get("/admin/actions", async (request) => {
+      parseHttpRequest(http.jsonHttpRequestContracts.adminActions, request);
       await requireAdmin(repo, request);
       return parseOutput(http.adminActionsResponseSchema, { actions: await repo.listAdminActions() });
     });
 
     app.post("/admin/users/:id/ban", async (request) => {
+      const {
+        params: { id },
+        body
+      } = parseHttpRequest(http.jsonHttpRequestContracts.adminBan, request);
       const user = await requireAdmin(repo, request);
-      const { id } = parseInput(http.idParamsSchema, request.params);
-      const body = parseInput(http.adminBanBodySchema, request.body ?? {});
       return parseOutput(http.publicUserResponseSchema, {
         user: await repo.setUserBan(user.id, id, body.banned ?? true, body.reason ?? "manual review")
       });
     });
 
     app.patch("/admin/users/:id/status", async (request) => {
+      const {
+        params: { id },
+        body
+      } = parseHttpRequest(http.jsonHttpRequestContracts.adminStatus, request);
       const user = await requireAdmin(repo, request);
-      const { id } = parseInput(http.idParamsSchema, request.params);
-      const body = parseInput(http.adminStatusBodySchema, request.body);
       return parseOutput(http.publicUserResponseSchema, {
         user: await repo.setUserBan(user.id, id, body.status === "banned", body.reason ?? "manual review")
       });
diff --git a/apps/api/src/auth-boundary.test.ts b/apps/api/src/auth-boundary.test.ts
index 74e8192..8a87305 100644
--- a/apps/api/src/auth-boundary.test.ts
+++ b/apps/api/src/auth-boundary.test.ts
@@ -63,7 +63,8 @@ describe("authentication boundary", () => {
 
     expect(cookieResponse.statusCode).toBe(200);
     expectApiError(authorizationResponse, 401);
-    expectApiError(queryResponse, 401);
+    expectApiError(queryResponse, 400);
+    expect(queryResponse.json<{ error: { code: string } }>().error.code).toBe("validation_error");
   });
 
   it("does not grant administrator privileges from the dev-login handle", async () => {
diff --git a/apps/api/src/guest-demo.test.ts b/apps/api/src/guest-demo.test.ts
index 186940b..f5ff2f7 100644
--- a/apps/api/src/guest-demo.test.ts
+++ b/apps/api/src/guest-demo.test.ts
@@ -77,7 +77,7 @@ describe("guest demo HTTP boundary", () => {
       url: "/auth/guest",
       payload: { displayName: "직접 정한 이름" }
     });
-    expectApiError(withBody, 400, "validation_failed");
+    expectApiError(withBody, 400, "validation_error");
 
     for (let count = 0; count < 10; count += 1) {
       const response = await app.inject({ method: "POST", url: "/auth/guest" });
diff --git a/apps/api/src/httpBoundary.ts b/apps/api/src/httpBoundary.ts
index 264815f..6867d77 100644
--- a/apps/api/src/httpBoundary.ts
+++ b/apps/api/src/httpBoundary.ts
@@ -25,12 +25,27 @@ export function parseInput<T>(schema: ZodType<T>, input: unknown): T {
   }
   throw new ApiHttpError(
     400,
-    "validation_failed",
+    "validation_error",
     "입력값을 확인해주세요.",
     Object.keys(fieldErrors).length > 0 ? fieldErrors : undefined
   );
 }
 
+export function parseHttpRequest<Params, Query, Body>(
+  contract: {
+    params: ZodType<Params>;
+    query: ZodType<Query>;
+    body: ZodType<Body>;
+  },
+  request: FastifyRequest
+): { params: Params; query: Query; body: Body } {
+  return {
+    params: parseInput(contract.params, request.params ?? {}),
+    query: parseInput(contract.query, request.query ?? {}),
+    body: parseInput(contract.body, request.body ?? {})
+  };
+}
+
 export function parseOutput<T>(schema: ZodType<T>, output: unknown): T {
   const result = schema.safeParse(output);
   if (result.success) return result.data;


## `test(api): strict request contract 검증`

diff --git a/apps/api/src/guest-demo.test.ts b/apps/api/src/guest-demo.test.ts
index f5ff2f7..3f4e740 100644
--- a/apps/api/src/guest-demo.test.ts
+++ b/apps/api/src/guest-demo.test.ts
@@ -124,6 +124,36 @@ describe("guest demo HTTP boundary", () => {
     }
   });
 
+  it("ignores forwarded client addresses when no trusted proxy is configured", async () => {
+    const direct = buildApp({
+      repo,
+      webOrigin: "http://localhost:3000",
+      appMode: "demo",
+      guestAccess: new GuestAccess({
+        secret: "guest-demo-test-secret-that-is-long-enough",
+        creationLimitPerMinute: 1
+      })
+    });
+    await direct.ready();
+    try {
+      const first = await direct.inject({
+        method: "POST",
+        url: "/auth/guest",
+        headers: { "x-forwarded-for": "203.0.113.51" }
+      });
+      const spoofed = await direct.inject({
+        method: "POST",
+        url: "/auth/guest",
+        headers: { "x-forwarded-for": "203.0.113.52" }
+      });
+
+      expect(first.statusCode).toBe(200);
+      expectApiError(spoofed, 429, "guest_creation_rate_limited");
+    } finally {
+      await direct.close();
+    }
+  });
+
   it("does not expose guest login outside demo mode", async () => {
     const developmentApp = buildApp({ repo, webOrigin: "http://localhost:3000", appMode: "test" });
     await developmentApp.ready();
diff --git a/apps/api/src/http-contract.test.ts b/apps/api/src/http-contract.test.ts
new file mode 100644
index 0000000..26ff272
--- /dev/null
+++ b/apps/api/src/http-contract.test.ts
@@ -0,0 +1,180 @@
+import { afterEach, beforeEach, describe, expect, it } from "vitest";
+import { apiErrorBodySchema, jsonHttpRequestContracts } from "@pong-pong/shared";
+import { createMemoryRepository, type AppRepository } from "@pong-pong/db";
+import { buildApp } from "./app";
+
+type JsonRouteCase = {
+  method: "GET" | "POST" | "PATCH";
+  url: string;
+  payload?: Record<string, unknown>;
+};
+
+const userId = "018f4af4-3223-7a17-a0c1-2f4f2404d8ef";
+
+const jsonRoutes: JsonRouteCase[] = [
+  { method: "GET", url: "/health" },
+  { method: "GET", url: "/health/live" },
+  { method: "GET", url: "/health/ready" },
+  {
+    method: "POST",
+    url: "/auth/dev-login",
+    payload: { handle: "contract-user", displayName: "계약 사용자" }
+  },
+  { method: "POST", url: "/auth/logout" },
+  { method: "POST", url: "/auth/ws-ticket" },
+  { method: "GET", url: "/me" },
+  { method: "GET", url: "/auth/me" },
+  { method: "GET", url: `/users/${userId}` },
+  { method: "GET", url: "/lobby" },
+  { method: "POST", url: "/chat/lobby", payload: { body: "계약 검사" } },
+  { method: "GET", url: "/leaderboard" },
+  { method: "GET", url: "/dashboard" },
+  { method: "GET", url: "/profile/contract-user" },
+  { method: "GET", url: "/profile/me" },
+  { method: "PATCH", url: "/profile/me", payload: { displayName: "새 이름" } },
+  { method: "GET", url: "/friends" },
+  { method: "POST", url: "/friends/request", payload: { handle: "opponent" } },
+  { method: "POST", url: "/friends", payload: { handle: "opponent" } },
+  { method: "POST", url: `/friends/${userId}/accept` },
+  { method: "GET", url: "/tournaments" },
+  { method: "POST", url: "/tournaments", payload: { name: "계약 대회" } },
+  { method: "POST", url: `/tournaments/${userId}/join` },
+  { method: "GET", url: "/admin/users" },
+  { method: "GET", url: "/admin/actions" },
+  { method: "POST", url: `/admin/users/${userId}/ban` },
+  {
+    method: "PATCH",
+    url: `/admin/users/${userId}/status`,
+    payload: { status: "banned" }
+  }
+];
+
+const jsonBodyRoutes: JsonRouteCase[] = [
+  {
+    method: "POST",
+    url: "/auth/dev-login",
+    payload: { handle: "contract-user", displayName: "계약 사용자" }
+  },
+  { method: "POST", url: "/auth/logout", payload: {} },
+  { method: "POST", url: "/auth/ws-ticket", payload: {} },
+  { method: "POST", url: "/chat/lobby", payload: { body: "계약 검사" } },
+  { method: "PATCH", url: "/profile/me", payload: { displayName: "새 이름" } },
+  { method: "POST", url: "/friends/request", payload: { handle: "opponent" } },
+  { method: "POST", url: "/friends", payload: { handle: "opponent" } },
+  { method: "POST", url: `/friends/${userId}/accept`, payload: {} },
+  { method: "POST", url: "/tournaments", payload: { name: "계약 대회" } },
+  { method: "POST", url: `/tournaments/${userId}/join`, payload: {} },
+  { method: "POST", url: `/admin/users/${userId}/ban`, payload: {} },
+  {
+    method: "PATCH",
+    url: `/admin/users/${userId}/status`,
+    payload: { status: "banned" }
+  }
+];
+
+describe("JSON HTTP request contracts", () => {
+  let repo: AppRepository;
+  let app: ReturnType<typeof buildApp>;
+
+  beforeEach(async () => {
+    repo = createMemoryRepository();
+    await repo.ensureSeedData();
+    app = buildApp({ repo, webOrigin: "http://localhost:3000", appMode: "test" });
+    await app.ready();
+  });
+
+  afterEach(async () => {
+    await app.close();
+    await repo.close();
+  });
+
+  it.each(jsonRoutes)(
+    "$method $url rejects an unknown query field with the shared error envelope",
+    async ({ method, url, payload }) => {
+      const response = await app.inject({
+        method,
+        url: `${url}?unexpected=1`,
+        ...(payload ? { payload } : {})
+      });
+
+      expectValidationError(response);
+    }
+  );
+
+  it.each(jsonBodyRoutes)(
+    "$method $url rejects an unknown body field with the shared error envelope",
+    async ({ method, url, payload = {} }) => {
+      const response = await app.inject({
+        method,
+        url,
+        payload: { ...payload, unexpected: true }
+      });
+
+      expectValidationError(response);
+    }
+  );
+
+  it("keeps an explicit strict body contract for a bodyless JSON GET route", () => {
+    expect(jsonHttpRequestContracts.leaderboard.body.safeParse({ unexpected: true }).success)
+      .toBe(false);
+  });
+
+  it.each([
+    { method: "GET" as const, url: "/users/not-a-uuid" },
+    { method: "GET" as const, url: `/profile/${"a".repeat(65)}` },
+    { method: "POST" as const, url: "/friends/not-a-uuid/accept" },
+    { method: "POST" as const, url: "/tournaments/not-a-uuid/join" },
+    { method: "POST" as const, url: "/admin/users/not-a-uuid/ban" },
+    {
+      method: "PATCH" as const,
+      url: "/admin/users/not-a-uuid/status",
+      payload: { status: "banned" }
+    }
+  ])("$method $url validates path parameters before route work", async ({ method, url, payload }) => {
+    const response = await app.inject({
+      method,
+      url,
+      ...(payload ? { payload } : {})
+    });
+
+    expectValidationError(response);
+  });
+
+  it("keeps the demo guest body and query contracts strict", async () => {
+    await app.close();
+    await repo.close();
+
+    repo = createMemoryRepository();
+    app = buildApp({
+      repo,
+      webOrigin: "http://localhost:3000",
+      appMode: "demo",
+      sessionSecret: "guest-contract-session-secret-32-bytes"
+    });
+    await app.ready();
+
+    const bodyResponse = await app.inject({
+      method: "POST",
+      url: "/auth/guest",
+      payload: { unexpected: true }
+    });
+    const queryResponse = await app.inject({
+      method: "POST",
+      url: "/auth/guest?unexpected=1"
+    });
+
+    expectValidationError(bodyResponse);
+    expectValidationError(queryResponse);
+  });
+});
+
+function expectValidationError(response: {
+  statusCode: number;
+  json<T>(): T;
+}): void {
+  expect(response.statusCode).toBe(400);
+  const body = apiErrorBodySchema.parse(response.json());
+  expect(body.error.code).toBe("validation_error");
+  expect(body.error.requestId).not.toHaveLength(0);
+  expect(body.error.fieldErrors).toBeDefined();
+}
diff --git a/packages/shared/src/http.test.ts b/packages/shared/src/http.test.ts
index 6eb2230..4b9c8a4 100644
--- a/packages/shared/src/http.test.ts
+++ b/packages/shared/src/http.test.ts
@@ -59,7 +59,7 @@ describe("HTTP contracts", () => {
   it("keeps the API error envelope stable", () => {
     const body = {
       error: {
-        code: "validation_failed",
+        code: "validation_error",
         message: "입력값을 확인해주세요.",
         requestId: "req-42",
         fieldErrors: { displayName: ["값을 입력해주세요."] }
