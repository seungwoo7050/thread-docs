## `test(auth): WebSocket ticket 경계 검증`

diff --git a/apps/api/src/ws-ticket.test.ts b/apps/api/src/ws-ticket.test.ts
new file mode 100644
index 0000000..cc357a4
--- /dev/null
+++ b/apps/api/src/ws-ticket.test.ts
@@ -0,0 +1,309 @@
+import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
+import { WebSocket } from "ws";
+import { createMemoryRepository, type AppRepository } from "@pong-pong/db";
+import { buildApp } from "./app";
+import { createRawWsTicket, hashWsTicket } from "./wsTicket";
+
+type InjectResponse = {
+  statusCode: number;
+  headers: Record<string, string | string[] | number | undefined>;
+  json<T = unknown>(): T;
+};
+
+type CloseDetails = {
+  code: number;
+  reason: string;
+};
+
+describe("one-time websocket tickets", () => {
+  let repo: AppRepository;
+  let app: ReturnType<typeof buildApp>;
+  let sockets: WebSocket[];
+  let wsBaseUrl: string;
+
+  beforeEach(async () => {
+    sockets = [];
+    repo = createMemoryRepository();
+    await repo.ensureSeedData("development");
+    app = buildApp({ repo, webOrigin: "http://localhost:3000", appMode: "test" });
+    const address = await app.listen({ host: "127.0.0.1", port: 0 });
+    wsBaseUrl = address.replace(/^http/, "ws");
+  });
+
+  afterEach(async () => {
+    for (const socket of sockets) {
+      if (socket.readyState !== WebSocket.CLOSED) socket.terminate();
+    }
+    await app.close();
+    await repo.close();
+    vi.restoreAllMocks();
+  });
+
+  it("issues a random raw ticket for exactly 30 seconds after cookie authentication", async () => {
+    const unauthenticated = await app.inject({ method: "POST", url: "/auth/ws-ticket" });
+    expectApiError(unauthenticated, 401, "authentication_required");
+
+    const { cookie, userId } = await login("ticket-issuer");
+    const authorizationOnly = await app.inject({
+      method: "POST",
+      url: "/auth/ws-ticket",
+      headers: { authorization: `Bearer ${cookie.slice("pp_session=".length)}` }
+    });
+    expectApiError(authorizationOnly, 401, "authentication_required");
+
+    const createTicket = vi.spyOn(repo, "createWsTicket");
+    const first = await issueTicket(cookie);
+    const second = await issueTicket(cookie);
+
+    expect(first).toMatchObject({ expiresInSeconds: 30, protocolVersion: 1 });
+    expect(first.ticket).toMatch(/^[A-Za-z0-9_-]{43}$/);
+    expect(second.ticket).not.toBe(first.ticket);
+    expect(createTicket).toHaveBeenNthCalledWith(1, {
+      userId,
+      ticketHash: hashWsTicket(first.ticket),
+      ttlSeconds: 30
+    });
+    expect(createTicket.mock.calls[0]?.[0].ticketHash).not.toBe(first.ticket);
+  });
+
+  it("does not issue tickets for suspended users", async () => {
+    const { cookie, userId } = await login("suspended-issuer");
+    await repo.setUserBan(userId, userId, true, "ws ticket test");
+
+    const response = await app.inject({
+      method: "POST",
+      url: "/auth/ws-ticket",
+      headers: { cookie }
+    });
+
+    expectApiError(response, 403, "account_suspended");
+  });
+
+  it("accepts a valid ticket once and rejects its reuse", async () => {
+    const { cookie } = await login("single-use");
+    const { ticket } = await issueTicket(cookie);
+    const accepted = await connect(`/ws?ticket=${ticket}&v=1`);
+    await expectAccepted(accepted);
+    accepted.close(1000, "test complete");
+
+    const reused = await connect(`/ws?ticket=${ticket}&v=1`);
+    await expectClose(reused, 1008, "invalid websocket ticket");
+  });
+
+  it("rejects forged and expired tickets with the stable authentication close", async () => {
+    const { cookie, userId } = await login("invalid-ticket");
+    const { ticket } = await issueTicket(cookie);
+    const forgedTicket = `${ticket.slice(0, -1)}${ticket.endsWith("A") ? "B" : "A"}`;
+    const forged = await connect(`/ws?ticket=${forgedTicket}&v=1`);
+    await expectClose(forged, 1008, "invalid websocket ticket");
+
+    const expiredTicket = createRawWsTicket();
+    await repo.createWsTicket({
+      userId,
+      ticketHash: hashWsTicket(expiredTicket),
+      ttlSeconds: 0
+    });
+    const expired = await connect(`/ws?ticket=${expiredTicket}&v=1`);
+    await expectClose(expired, 1008, "invalid websocket ticket");
+  });
+
+  it("rejects a ticket when its user becomes suspended", async () => {
+    const { cookie, userId } = await login("suspended-socket");
+    const { ticket } = await issueTicket(cookie);
+    await repo.setUserBan(userId, userId, true, "ws connection test");
+
+    const socket = await connect(`/ws?ticket=${ticket}&v=1`);
+
+    await expectClose(socket, 1008, "invalid websocket ticket");
+  });
+
+  it("rejects unsupported versions without consuming the ticket", async () => {
+    const { cookie } = await login("version-check");
+    const { ticket } = await issueTicket(cookie);
+    const unsupported = await connect(`/ws?ticket=${ticket}&v=2`);
+    await expectClose(unsupported, 1008, "unsupported websocket version");
+
+    const supported = await connect(`/ws?ticket=${ticket}&v=1`);
+    await expectAccepted(supported);
+    supported.close(1000, "test complete");
+  });
+
+  it("does not authenticate a long session through cookie or Authorization", async () => {
+    const { cookie } = await login("session-only");
+    const sessionToken = cookie.slice("pp_session=".length);
+
+    const socket = await connect(`/ws?v=1&session=${encodeURIComponent(sessionToken)}`, {
+      cookie,
+      authorization: `Bearer ${sessionToken}`
+    });
+
+    await expectClose(socket, 1008, "invalid websocket ticket");
+  });
+
+  it("closes on an individual pre-authentication payload above 8 KiB", async () => {
+    const { socket, releaseAuthentication } = await connectWithDelayedAuthentication();
+    try {
+      const closed = closeDetails(socket);
+      socket.send(Buffer.alloc(8 * 1024 + 1));
+      expect(await closed).toEqual({ code: 1009, reason: "pre-auth payload too large" });
+    } finally {
+      releaseAuthentication();
+    }
+  });
+
+  it("allows 16 pre-authentication messages and closes on the seventeenth", async () => {
+    const { socket, releaseAuthentication } = await connectWithDelayedAuthentication();
+    try {
+      for (let index = 0; index < 16; index += 1) socket.send("{}");
+      await nextTurn();
+      expect(socket.readyState).toBe(WebSocket.OPEN);
+
+      const closed = closeDetails(socket);
+      socket.send("{}");
+      expect(await closed).toEqual({ code: 1009, reason: "pre-auth buffer limit exceeded" });
+    } finally {
+      releaseAuthentication();
+    }
+  });
+
+  it("allows 32 KiB of pre-authentication data and closes above the total limit", async () => {
+    const { socket, releaseAuthentication } = await connectWithDelayedAuthentication();
+    try {
+      for (let index = 0; index < 4; index += 1) socket.send(Buffer.alloc(8 * 1024, 97));
+      await nextTurn();
+      expect(socket.readyState).toBe(WebSocket.OPEN);
+
+      const closed = closeDetails(socket);
+      socket.send("a");
+      expect(await closed).toEqual({ code: 1009, reason: "pre-auth buffer limit exceeded" });
+    } finally {
+      releaseAuthentication();
+    }
+  });
+
+  async function login(handle: string): Promise<{ cookie: string; userId: string }> {
+    const response = await app.inject({
+      method: "POST",
+      url: "/auth/dev-login",
+      payload: { handle, displayName: handle }
+    });
+    expect(response.statusCode).toBe(200);
+    return {
+      cookie: sessionCookie(response),
+      userId: response.json<{ user: { id: string } }>().user.id
+    };
+  }
+
+  async function issueTicket(cookie: string): Promise<{
+    ticket: string;
+    expiresInSeconds: number;
+    protocolVersion: number;
+  }> {
+    const response = await app.inject({
+      method: "POST",
+      url: "/auth/ws-ticket",
+      headers: { cookie }
+    });
+    expect(response.statusCode).toBe(200);
+    return response.json();
+  }
+
+  async function connect(path: string, headers: Record<string, string> = {}): Promise<WebSocket> {
+    const socket = new WebSocket(`${wsBaseUrl}${path}`, { headers });
+    sockets.push(socket);
+    await new Promise<void>((resolve, reject) => {
+      socket.once("open", () => resolve());
+      socket.once("error", reject);
+    });
+    return socket;
+  }
+
+  async function connectWithDelayedAuthentication(): Promise<{
+    socket: WebSocket;
+    releaseAuthentication(): void;
+  }> {
+    const { cookie } = await login(`buffer-${Math.random().toString(36).slice(2)}`);
+    const { ticket } = await issueTicket(cookie);
+    const gate = deferred<void>();
+    const consumeTicket = repo.consumeWsTicket.bind(repo);
+    repo.consumeWsTicket = async (ticketHash) => {
+      await gate.promise;
+      return consumeTicket(ticketHash);
+    };
+    const socket = await connect(`/ws?ticket=${ticket}&v=1`);
+    return { socket, releaseAuthentication: () => gate.resolve() };
+  }
+});
+
+function sessionCookie(response: InjectResponse): string {
+  const value = response.headers["set-cookie"];
+  const header = Array.isArray(value)
+    ? value.find((item) => item.startsWith("pp_session="))
+    : typeof value === "string" ? value : undefined;
+  if (!header) throw new Error("pp_session cookie was not set");
+  return header.split(";", 1)[0];
+}
+
+function expectApiError(response: InjectResponse, statusCode: number, code: string): void {
+  expect(response.statusCode).toBe(statusCode);
+  expect(response.json()).toEqual({
+    error: expect.objectContaining({
+      code,
+      message: expect.any(String),
+      requestId: expect.any(String)
+    })
+  });
+}
+
+async function expectAccepted(socket: WebSocket): Promise<void> {
+  await new Promise<void>((resolve, reject) => {
+    const timer = setTimeout(() => {
+      socket.off("close", onClose);
+      resolve();
+    }, 30);
+    const onClose = (code: number, reason: Buffer) => {
+      clearTimeout(timer);
+      reject(new Error(`WebSocket closed during authentication: ${code} ${reason.toString("utf8")}`));
+    };
+    socket.once("close", onClose);
+  });
+  expect(socket.readyState).toBe(WebSocket.OPEN);
+}
+
+async function expectClose(socket: WebSocket, code: number, reason: string): Promise<void> {
+  await expect(closeDetails(socket)).resolves.toEqual({ code, reason });
+}
+
+function closeDetails(socket: WebSocket): Promise<CloseDetails> {
+  return new Promise((resolve, reject) => {
+    const timer = setTimeout(() => {
+      socket.terminate();
+      reject(new Error("Timed out waiting for WebSocket close"));
+    }, 2_000);
+    socket.once("close", (code, reason) => {
+      clearTimeout(timer);
+      resolve({ code, reason: reason.toString("utf8") });
+    });
+    socket.once("error", (error) => {
+      clearTimeout(timer);
+      reject(error);
+    });
+  });
+}
+
+function deferred<T>(): { promise: Promise<T>; resolve(value?: T): void } {
+  let resolvePromise!: (value: T | PromiseLike<T>) => void;
+  const promise = new Promise<T>((resolve) => {
+    resolvePromise = resolve;
+  });
+  return {
+    promise,
+    resolve(value?: T) {
+      resolvePromise(value as T);
+    }
+  };
+}
+
+function nextTurn(): Promise<void> {
+  return new Promise((resolve) => setImmediate(resolve));
+}
diff --git a/packages/db/src/index.test.ts b/packages/db/src/index.test.ts
index e766b53..69c1c38 100644
--- a/packages/db/src/index.test.ts
+++ b/packages/db/src/index.test.ts
@@ -1,3 +1,4 @@
+import { createHash, randomBytes } from "node:crypto";
 import { describe, expect, it } from "vitest";
 import { createMemoryRepository } from "./index";
 
@@ -87,4 +88,32 @@ describe("memory repository", () => {
     expect(final?.right?.id).toBe(p2.id);
     expect((await repo.listTournaments())[0].name).toBe("테스트 컵");
   });
+
+  it("consumes websocket tickets once and rejects expired or suspended users", async () => {
+    const repo = createMemoryRepository();
+    const user = await repo.upsertDevUser({ handle: "ws-user", displayName: "WS 사용자" });
+    const ticketHash = newTicketHash();
+    await repo.createWsTicket({ userId: user.id, ticketHash, ttlSeconds: 30 });
+
+    const attempts = await Promise.all([
+      repo.consumeWsTicket(ticketHash),
+      repo.consumeWsTicket(ticketHash)
+    ]);
+    expect(attempts.filter((result) => result !== null)).toHaveLength(1);
+    await expect(repo.consumeWsTicket(ticketHash)).resolves.toBeNull();
+
+    const expiredHash = newTicketHash();
+    await repo.createWsTicket({ userId: user.id, ticketHash: expiredHash, ttlSeconds: 0 });
+    await expect(repo.consumeWsTicket(expiredHash)).resolves.toBeNull();
+
+    const suspendedHash = newTicketHash();
+    await repo.createWsTicket({ userId: user.id, ticketHash: suspendedHash, ttlSeconds: 30 });
+    await repo.setUserBan(user.id, user.id, true, "ticket test");
+    await expect(repo.consumeWsTicket(suspendedHash)).resolves.toBeNull();
+  });
 });
+
+function newTicketHash(): string {
+  const rawTicket = randomBytes(32).toString("base64url");
+  return createHash("sha256").update(rawTicket, "utf8").digest("hex");
+}
diff --git a/packages/db/src/postgres.integration.test.ts b/packages/db/src/postgres.integration.test.ts
index 5fdba25..bfbd941 100644
--- a/packages/db/src/postgres.integration.test.ts
+++ b/packages/db/src/postgres.integration.test.ts
@@ -1,4 +1,4 @@
-import { randomUUID } from "node:crypto";
+import { createHash, randomBytes, randomUUID } from "node:crypto";
 import { PostgreSqlContainer } from "@testcontainers/postgresql";
 import { afterAll, beforeAll, describe, expect, it } from "vitest";
 import { Pool } from "pg";
@@ -61,10 +61,11 @@ describe("PostgreSQL integration", () => {
         "tournament_entries",
         "tournament_matches",
         "tournaments",
-        "users"
+        "users",
+        "ws_tickets"
       ]));
       const firstMigrations = await appliedMigrations(pool);
-      expect(firstMigrations).toEqual(["001_initial"]);
+      expect(firstMigrations).toEqual(["001_initial", "002_ws_tickets"]);
 
       await migrateDatabase(databaseUrl);
 
@@ -133,6 +134,53 @@ describe("PostgreSQL integration", () => {
     });
   });
 
+  it("stores only ticket hashes and consumes a ticket atomically once", async () => {
+    await withIsolatedDatabase(async ({ openPool, openRepository }) => {
+      const repository = openRepository();
+      const pool = openPool();
+      const user = await repository.upsertDevUser({
+        handle: "ws-ticket-user",
+        displayName: "WS Ticket User"
+      });
+      const rawTicket = randomBytes(32).toString("base64url");
+      const ticketHash = createHash("sha256").update(rawTicket, "utf8").digest("hex");
+
+      await repository.createWsTicket({
+        userId: user.id,
+        ticketHash,
+        ttlSeconds: 30
+      });
+
+      const columns = await pool.query<{ column_name: string }>(
+        "select column_name from information_schema.columns where table_schema = current_schema() and table_name = 'ws_tickets' order by column_name"
+      );
+      expect(columns.rows.map((row) => row.column_name)).toEqual([
+        "created_at",
+        "expires_at",
+        "ticket_hash",
+        "user_id"
+      ]);
+      const stored = await pool.query<{ ticket_hash: string; ttl_seconds: number }>(
+        "select ticket_hash, extract(epoch from expires_at - created_at)::integer as ttl_seconds from ws_tickets"
+      );
+      expect(stored.rows).toEqual([{ ticket_hash: ticketHash, ttl_seconds: 30 }]);
+      expect(JSON.stringify(stored.rows)).not.toContain(rawTicket);
+
+      const attempts = await Promise.all(
+        Array.from({ length: 20 }, () => repository.consumeWsTicket(ticketHash))
+      );
+      const successful = attempts.filter((result) => result !== null);
+      expect(successful).toHaveLength(1);
+      expect(successful[0]?.id).toBe(user.id);
+      await expect(repository.consumeWsTicket(ticketHash)).resolves.toBeNull();
+
+      const remaining = await pool.query<{ count: number }>(
+        "select count(*)::integer as count from ws_tickets"
+      );
+      expect(remaining.rows[0]?.count).toBe(0);
+    });
+  });
+
   it("uses a fresh schema for each isolated database", async () => {
     let firstSchema = "";
 
diff --git a/packages/shared/src/http.test.ts b/packages/shared/src/http.test.ts
index fea29cc..caec71d 100644
--- a/packages/shared/src/http.test.ts
+++ b/packages/shared/src/http.test.ts
@@ -6,6 +6,7 @@ import {
   idParamsSchema,
   profileUpdateBodySchema,
   sessionUserSchema,
+  wsHandshakeQuerySchema,
   wsTicketResponseSchema
 } from "./http";
 
@@ -77,4 +78,13 @@ describe("HTTP contracts", () => {
     expect(wsTicketResponseSchema.parse(response)).toEqual(response);
     expect(wsTicketResponseSchema.safeParse({ ...response, protocolVersion: 2 }).success).toBe(false);
   });
+
+  it("accepts only a one-time ticket and protocol v1 in websocket query parameters", () => {
+    const query = { ticket: "a".repeat(43), v: "1" } as const;
+
+    expect(wsHandshakeQuerySchema.parse(query)).toEqual(query);
+    expect(wsHandshakeQuerySchema.safeParse({ ...query, v: "2" }).success).toBe(false);
+    expect(wsHandshakeQuerySchema.safeParse({ ...query, session: "long-session" }).success).toBe(false);
+    expect(wsHandshakeQuerySchema.safeParse({ v: "1" }).success).toBe(false);
+  });
 });


## `feat(db): legacy session을 안전하게 만료`

diff --git a/packages/db/migrations/005_expire_legacy_sessions.sql b/packages/db/migrations/005_expire_legacy_sessions.sql
new file mode 100644
index 0000000..72040d3
--- /dev/null
+++ b/packages/db/migrations/005_expire_legacy_sessions.sql
@@ -0,0 +1 @@
+delete from sessions;
diff --git a/packages/db/src/migrator.ts b/packages/db/src/migrator.ts
index b2a014d..eb73440 100644
--- a/packages/db/src/migrator.ts
+++ b/packages/db/src/migrator.ts
@@ -91,7 +91,7 @@ export async function inspectMigrationSet(
   return compareMigrationSets(expectedNames, appliedNames);
 }
 
-export async function migrateDatabase(databaseUrl: string): Promise<void> {
+export async function migrateDatabase(databaseUrl: string, targetMigration?: string): Promise<void> {
   const pool = new Pool({ connectionString: databaseUrl });
   const db = new Kysely<Database>({ dialect: new PostgresDialect({ pool }) });
 
@@ -100,7 +100,9 @@ export async function migrateDatabase(databaseUrl: string): Promise<void> {
       db,
       provider: new SqlMigrationProvider()
     });
-    const { error, results } = await migrator.migrateToLatest();
+    const { error, results } = targetMigration
+      ? await migrator.migrateTo(targetMigration)
+      : await migrator.migrateToLatest();
 
     if (error) {
       const failedMigration = results?.find((result) => result.status === "Error");


## `fix(api): 내부 WebSocket 오류 숨김`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index d4b7d14..3e999ab 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -80,6 +80,8 @@ const SIMULATION_TIMESTEP_MS = DEFAULT_TIMESTEP_MS;
 const CONNECTION_REPLACED_CLOSE_CODE = 4001;
 const CONNECTION_REPLACED_REASON = "connection replaced";
 const GUEST_RESULT_RETENTION_MS = 2 * 60 * 1_000;
+const INVALID_EVENT_MESSAGE = "올바르지 않은 메시지입니다.";
+const INTERNAL_ERROR_MESSAGE = "메시지를 처리하지 못했습니다.";
 
 export interface DrainResult {
   drained: boolean;
@@ -195,8 +197,19 @@ export class GameHub {
 
   private async receive(client: Client, payload: string): Promise<void> {
     if (this.clients.get(client.id) !== client) return;
+    let event: ReturnType<typeof parseClientEvent>;
+    try {
+      event = parseClientEvent(payload);
+    } catch {
+      this.send(client, {
+        type: "error",
+        code: "invalid_event",
+        message: INVALID_EVENT_MESSAGE
+      });
+      return;
+    }
+
     try {
-      const event = parseClientEvent(payload);
       if (isGuest(client.user) && (event.type === "chat.send" || event.type === "tournament.join")) {
         this.send(client, {
           type: "error",
@@ -225,11 +238,11 @@ export class GameHub {
           this.broadcastAll({ type: "chat.message", message });
         }
       }
-    } catch (error) {
+    } catch {
       this.send(client, {
         type: "error",
-        code: "invalid_event",
-        message: error instanceof Error ? error.message : "메시지를 처리하지 못했습니다."
+        code: "internal_error",
+        message: INTERNAL_ERROR_MESSAGE
       });
     }
   }
@@ -439,11 +452,11 @@ export class GameHub {
   private armAiFallback(entry: QueueEntry, delayMs: number): void {
     clearQueueTimer(entry);
     entry.npcFallbackTimer = setTimeout(() => {
-      this.matchQueuedClientWithNpc(entry).catch((error) => {
+      this.matchQueuedClientWithNpc(entry).catch(() => {
         this.send(entry.client, {
           type: "error",
           code: "internal_error",
-          message: error instanceof Error ? error.message : "AI 상대를 찾지 못했습니다."
+          message: INTERNAL_ERROR_MESSAGE
         });
       });
     }, Math.max(0, delayMs));


