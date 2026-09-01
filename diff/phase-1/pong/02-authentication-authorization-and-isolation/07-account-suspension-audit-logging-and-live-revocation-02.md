## `feat(db): 명시적 사용자 role 할당 추가`

diff --git a/apps/api/src/admin.test.ts b/apps/api/src/admin.test.ts
index c525f49..cb7a8f9 100644
--- a/apps/api/src/admin.test.ts
+++ b/apps/api/src/admin.test.ts
@@ -24,6 +24,7 @@ describe("admin routes", () => {
       url: "/auth/dev-login",
       payload: { handle: "admin", displayName: "운영자" }
     });
+    await repo.setUserRoleByHandle("admin", "admin");
     const targetLogin = await app.inject({
       method: "POST",
       url: "/auth/dev-login",
diff --git a/packages/db/package.json b/packages/db/package.json
index d6f84c5..e5e3e5e 100644
--- a/packages/db/package.json
+++ b/packages/db/package.json
@@ -12,6 +12,7 @@
     "migrate": "tsx src/cli.ts migrate",
     "seed:dev": "tsx src/cli.ts seed:dev",
     "seed:demo": "tsx src/cli.ts seed:demo",
+    "user:set-role": "tsx src/cli.ts user:set-role",
     "memory-smoke": "tsx src/cli.ts memory-smoke",
     "test": "vitest run --exclude \"**/*.integration.test.ts\"",
     "postgres-integration": "vitest run src/postgres.integration.test.ts --testTimeout=120000 --hookTimeout=120000 --teardownTimeout=30000 --no-file-parallelism --maxWorkers=1"
diff --git a/packages/db/src/cli.ts b/packages/db/src/cli.ts
index 30b7716..66bbfda 100644
--- a/packages/db/src/cli.ts
+++ b/packages/db/src/cli.ts
@@ -25,8 +25,16 @@ if (command === "memory-smoke") {
       if (command === "seed:dev" || command === "seed:demo") {
         await repo.ensureSeedData(command === "seed:dev" ? "development" : "demo");
         console.log(command === "seed:dev" ? "development seed complete" : "demo seed complete");
+      } else if (command === "user:set-role") {
+        const handle = process.argv[3];
+        const role = process.argv[4];
+        if (!handle || (role !== "user" && role !== "admin")) {
+          throw new Error("Usage: pnpm --filter @pong-pong/db user:set-role -- <handle> <user|admin>");
+        }
+        const user = await repo.setUserRoleByHandle(handle, role);
+        console.log(`${user.handle} role set to ${user.role}`);
       } else {
-        throw new Error("Usage: pnpm --filter @pong-pong/db migrate|seed:dev|seed:demo|memory-smoke");
+        throw new Error("Usage: pnpm --filter @pong-pong/db migrate|seed:dev|seed:demo|user:set-role|memory-smoke");
       }
     } finally {
       await repo.close();
diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index 97a431a..643b612 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -1,7 +1,7 @@
 import { randomUUID } from "node:crypto";
 import { Kysely, PostgresDialect, sql } from "kysely";
 import { Pool } from "pg";
-import type { AdminActionSummary, ChatMessage, DashboardSummary, FriendSummary, LeaderboardEntry, MatchMode, MatchSummary, PublicUser, SessionUser, TournamentMatchSummary, TournamentSummary } from "@pong-pong/shared";
+import type { AdminActionSummary, ChatMessage, DashboardSummary, FriendSummary, LeaderboardEntry, MatchMode, MatchSummary, PublicUser, SessionUser, TournamentMatchSummary, TournamentSummary, UserRole } from "@pong-pong/shared";
 import { toAdminActionSummary, toChatMessage, toFriendSummary, toMatchSummary, toPublicUser, toSessionUser, toTournamentMatchRecord, toTournamentMatchSummary, toTournamentSummary } from "./rowMappers";
 import type { AdminActionRow, ChatMessageRow, ChatMessageWithSenderRow, Database, FriendshipWithUserRow, MatchWithHandlesRow, MemoryUserRow, TournamentMatchRow, TournamentRow, TournamentWithCreatorRow, UserRow } from "./schema";
 
@@ -54,6 +54,7 @@ export interface AppRepository {
   createSession(userId: string): Promise<string>;
   getSessionUser(token: string | undefined): Promise<SessionUser | null>;
   deleteSession(token: string | undefined): Promise<void>;
+  setUserRoleByHandle(handle: string, role: UserRole): Promise<PublicUser>;
   getUserById(id: string): Promise<PublicUser | null>;
   getUserByHandle(handle: string): Promise<PublicUser | null>;
   updateProfile(userId: string, input: { displayName?: string; avatarKey?: string }): Promise<SessionUser>;
@@ -129,10 +130,11 @@ class PostgresRepository implements AppRepository {
     const displayName = input.displayName.trim() || handle;
     const result = await sql<UserRow>`
       insert into users (email, handle, display_name, avatar_key, role, is_npc)
-      values (${email}, ${handle}, ${displayName}, ${avatarFor(handle)}, ${handle === "admin" ? "admin" : "user"}, false)
+      values (${email}, ${handle}, ${displayName}, ${avatarFor(handle)}, 'user', false)
       on conflict (handle) do update set
         email = excluded.email,
         display_name = excluded.display_name,
+        role = 'user',
         is_npc = false
       returning *
     `.execute(this.db);
@@ -175,6 +177,17 @@ class PostgresRepository implements AppRepository {
     await sql`delete from sessions where token = ${token}`.execute(this.db);
   }
 
+  async setUserRoleByHandle(handle: string, role: UserRole): Promise<PublicUser> {
+    const result = await sql<UserRow>`
+      update users
+      set role = ${role}
+      where handle = ${normalizeHandle(handle)} and is_npc = false
+      returning *
+    `.execute(this.db);
+    if (!result.rows[0]) throw new Error("user not found");
+    return toPublicUser(result.rows[0]);
+  }
+
   async getUserById(id: string): Promise<PublicUser | null> {
     const result = await sql<UserRow>`select * from users where id = ${id} limit 1`.execute(this.db);
     return result.rows[0] ? toPublicUser(result.rows[0]) : null;
@@ -436,6 +449,11 @@ class MemoryRepository implements AppRepository {
       ]) {
         await this.upsertDevUser(player);
       }
+      const admin = [...this.users.values()].find((user) => user.handle === "admin");
+      if (admin) {
+        admin.role = "admin";
+        admin.rating = 1680;
+      }
     }
     for (const npc of NPC_PLAYERS) {
       const existing = [...this.users.values()].find((user) => user.handle === npc.handle);
@@ -458,14 +476,16 @@ class MemoryRepository implements AppRepository {
       handle,
       display_name: input.displayName || handle,
       avatar_key: avatarFor(handle),
-      role: handle === "admin" ? "admin" : "user",
+      role: "user",
       status: "active",
-      rating: handle === "admin" ? 1680 : 1200,
+      rating: 1200,
       wins: 0,
       losses: 0,
       is_npc: false
     };
     user.display_name = input.displayName || user.display_name;
+    user.email = input.email ?? user.email;
+    user.role = "user";
     user.is_npc = false;
     this.users.set(user.id, user);
     return toSessionUser(user, true);
@@ -487,6 +507,13 @@ class MemoryRepository implements AppRepository {
     if (token) this.sessions.delete(token);
   }
 
+  async setUserRoleByHandle(handle: string, role: UserRole): Promise<PublicUser> {
+    const user = [...this.users.values()].find((item) => item.handle === normalizeHandle(handle) && !item.is_npc);
+    if (!user) throw new Error("user not found");
+    user.role = role;
+    return toPublicUser(user, true);
+  }
+
   async getUserById(id: string): Promise<PublicUser | null> {
     const user = this.users.get(id);
     return user ? toPublicUser(user, true) : null;


## `fix(db): 차단 감사 기록을 원자적으로 저장`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index 9c7004b..16482bb 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -836,17 +836,19 @@ class PostgresRepository implements AppRepository {
   }
 
   async setUserBan(actorId: string, targetUserId: string, banned: boolean, reason: string): Promise<PublicUser> {
-    const result = await sql<UserRow>`
-      update users
-      set status = ${banned ? "banned" : "active"}, banned_at = ${banned ? sql`now()` : null}
-      where id = ${targetUserId}
-      returning *
-    `.execute(this.db);
-    await sql`
-      insert into admin_actions (actor_id, target_user_id, action, reason)
-      values (${actorId}, ${targetUserId}, ${banned ? "ban" : "unban"}, ${reason})
-    `.execute(this.db);
-    return toPublicUser(firstRow(result));
+    return this.db.transaction().execute(async (transaction) => {
+      const result = await sql<UserRow>`
+        update users
+        set status = ${banned ? "banned" : "active"}, banned_at = ${banned ? sql`now()` : null}
+        where id = ${targetUserId}
+        returning *
+      `.execute(transaction);
+      await sql`
+        insert into admin_actions (actor_id, target_user_id, action, reason)
+        values (${actorId}, ${targetUserId}, ${banned ? "ban" : "unban"}, ${reason})
+      `.execute(transaction);
+      return toPublicUser(firstRow(result));
+    });
   }
 
   private async tournamentFromRow(row: TournamentWithCreatorRow): Promise<TournamentSummary> {


## `test(db): 차단 감사 기록 atomicity 검증`

diff --git a/packages/db/src/postgres.integration.test.ts b/packages/db/src/postgres.integration.test.ts
index 14405bb..85882de 100644
--- a/packages/db/src/postgres.integration.test.ts
+++ b/packages/db/src/postgres.integration.test.ts
@@ -232,6 +232,42 @@ describe("PostgreSQL integration", () => {
     });
   });
 
+  it("rolls back a ban when its audit record cannot be written", async () => {
+    await withIsolatedDatabase(async ({ openPool, openRepository }) => {
+      const repository = openRepository();
+      const pool = openPool();
+      const actor = await repository.upsertDevUser({
+        handle: "ban-transaction-actor",
+        displayName: "Ban Transaction Actor"
+      });
+      const target = await repository.upsertDevUser({
+        handle: "ban-transaction-target",
+        displayName: "Ban Transaction Target"
+      });
+      await pool.query(`
+        alter table admin_actions
+        add constraint admin_actions_reject_test_reason
+        check (reason <> 'force audit failure')
+      `);
+
+      await expect(repository.setUserBan(
+        actor.id,
+        target.id,
+        true,
+        "force audit failure"
+      )).rejects.toMatchObject({ constraint: "admin_actions_reject_test_reason" });
+
+      const storedUser = await pool.query<{ status: string; banned_at: Date | null }>(
+        "select status, banned_at from users where id = $1",
+        [target.id]
+      );
+      expect(storedUser.rows).toEqual([{ status: "active", banned_at: null }]);
+      await expect(pool.query<{ count: number }>(
+        "select count(*)::integer as count from admin_actions"
+      )).resolves.toMatchObject({ rows: [{ count: 0 }] });
+    });
+  });
+
   it("stores only ticket hashes and consumes a ticket atomically once", async () => {
     await withIsolatedDatabase(async ({ openPool, openRepository }) => {
       const repository = openRepository();


## `fix(auth): 정지된 관리자 login 거부`

diff --git a/apps/api/src/app.ts b/apps/api/src/app.ts
index 42fe3f7..b4bf9c7 100644
--- a/apps/api/src/app.ts
+++ b/apps/api/src/app.ts
@@ -517,6 +517,7 @@ async function currentUser(
 async function requireAdmin(repo: AppRepository, request: FastifyRequest): Promise<SessionUser> {
   const user = await currentUser(repo, request);
   if (!user) unauthorized();
+  if (!isActive(user)) suspended();
   if (user.role !== "admin") forbidden();
   return user;
 }


## `fix(auth): 정지된 사용자의 열린 연결 폐기`

diff --git a/apps/api/src/app.ts b/apps/api/src/app.ts
index 846fb03..1f2a824 100644
--- a/apps/api/src/app.ts
+++ b/apps/api/src/app.ts
@@ -510,8 +510,11 @@ export function buildApp({
         body
       } = parseHttpRequest(http.jsonHttpRequestContracts.adminBan, request);
       const user = await requireAdmin(repo, request);
+      const banned = body.banned ?? true;
+      const target = await repo.setUserBan(user.id, id, banned, body.reason ?? "manual review");
+      if (banned) hub.revokeUser(id);
       return parseOutput(http.publicUserResponseSchema, {
-        user: await repo.setUserBan(user.id, id, body.banned ?? true, body.reason ?? "manual review")
+        user: target
       });
     });
 
@@ -521,8 +524,11 @@ export function buildApp({
         body
       } = parseHttpRequest(http.jsonHttpRequestContracts.adminStatus, request);
       const user = await requireAdmin(repo, request);
+      const banned = body.status === "banned";
+      const target = await repo.setUserBan(user.id, id, banned, body.reason ?? "manual review");
+      if (banned) hub.revokeUser(id);
       return parseOutput(http.publicUserResponseSchema, {
-        user: await repo.setUserBan(user.id, id, body.status === "banned", body.reason ?? "manual review")
+        user: target
       });
     });
   }
diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index 3738faa..624ec11 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -82,6 +82,8 @@ const SIMULATION_TIMESTEP_MS = DEFAULT_TIMESTEP_MS;
 const SNAPSHOT_DELIVERY_DIVISOR = 2;
 const CONNECTION_REPLACED_CLOSE_CODE = 4001;
 const CONNECTION_REPLACED_REASON = "connection replaced";
+const ACCOUNT_SUSPENDED_CLOSE_CODE = 4003;
+const ACCOUNT_SUSPENDED_REASON = "account suspended";
 const FINALIZATION_RETRY_BASE_DELAY_MS = 250;
 const FINALIZATION_RETRY_MAX_DELAY_MS = 5_000;
 const GUEST_RESULT_RETENTION_MS = 2 * 60 * 1_000;
@@ -202,6 +204,27 @@ export class GameHub {
     }
   }
 
+  revokeUser(userId: string): void {
+    const client = this.clientsByUser.get(userId);
+    if (!client) return;
+    client.heartbeat.stop();
+    client.snapshots.close();
+    this.leaveQueue(client);
+    this.leaveTournamentWaiters(client);
+    this.clients.delete(client.id);
+    this.clientsByUser.delete(userId);
+    this.inputGate.releaseUser(userId);
+    if (client.roomId) {
+      const room = this.rooms.get(client.roomId);
+      const side = room ? sideFor(room, client) : null;
+      if (room && side) this.reserveRoomSide(room, side, userId);
+    }
+    if (client.socket.readyState === WebSocket.OPEN) {
+      client.socket.close(ACCOUNT_SUSPENDED_CLOSE_CODE, ACCOUNT_SUSPENDED_REASON);
+    }
+    this.broadcastPresence();
+  }
+
   private async receive(client: Client, payload: string): Promise<void> {
     if (this.clients.get(client.id) !== client) return;
     let event: ReturnType<typeof parseClientEvent>;


## `test(auth): 계정 정지의 기존 WebSocket 차단 검증`

diff --git a/apps/api/src/admin.test.ts b/apps/api/src/admin.test.ts
index a9ffe0b..ddea853 100644
--- a/apps/api/src/admin.test.ts
+++ b/apps/api/src/admin.test.ts
@@ -1,4 +1,5 @@
 import { afterEach, beforeEach, describe, expect, it } from "vitest";
+import { WebSocket } from "ws";
 import { createMemoryRepository, type AppRepository } from "@pong-pong/db";
 import { buildApp } from "./app";
 
@@ -6,6 +7,8 @@ describe("admin routes", () => {
   let repo: AppRepository;
   let app: ReturnType<typeof buildApp>;
   let adminCookie: string;
+  let address: string;
+  const sockets: WebSocket[] = [];
 
   beforeEach(async () => {
     repo = createMemoryRepository();
@@ -14,10 +17,13 @@ describe("admin routes", () => {
     if (!admin) throw new Error("seed:dev admin was not created");
     adminCookie = `pp_session=${await repo.createSession(admin.id)}`;
     app = buildApp({ repo, webOrigin: "http://localhost:3000" });
-    await app.ready();
+    address = await app.listen({ host: "127.0.0.1", port: 0 });
   });
 
   afterEach(async () => {
+    for (const socket of sockets.splice(0)) {
+      if (socket.readyState !== WebSocket.CLOSED) socket.terminate();
+    }
     await app.close();
     await repo.close();
   });
@@ -74,6 +80,47 @@ describe("admin routes", () => {
       })
     });
   });
+
+  it("closes an existing websocket immediately after account suspension", async () => {
+    const targetLogin = await app.inject({
+      method: "POST",
+      url: "/auth/dev-login",
+      payload: { handle: "live-target", displayName: "실시간 대상" }
+    });
+    const targetId = targetLogin.json<{ user: { id: string } }>().user.id;
+    const ticketResponse = await app.inject({
+      method: "POST",
+      url: "/auth/ws-ticket",
+      headers: { cookie: sessionCookie(targetLogin) }
+    });
+    const socket = new WebSocket(
+      `${address.replace(/^http/, "ws")}/ws?ticket=${ticketResponse.json<{ ticket: string }>().ticket}&v=1`
+    );
+    sockets.push(socket);
+    await new Promise<void>((resolve, reject) => {
+      socket.once("open", () => resolve());
+      socket.once("error", reject);
+    });
+    const closed = new Promise<{ code: number; reason: string }>((resolve) => {
+      socket.once("close", (code, reason) => resolve({ code, reason: reason.toString("utf8") }));
+    });
+
+    const ban = await app.inject({
+      method: "POST",
+      url: `/admin/users/${targetId}/ban`,
+      headers: { cookie: adminCookie },
+      payload: { banned: true, reason: "live revoke test" }
+    });
+
+    expect(ban.statusCode).toBe(200);
+    await expect(closed).resolves.toEqual({ code: 4003, reason: "account suspended" });
+    const newTicket = await app.inject({
+      method: "POST",
+      url: "/auth/ws-ticket",
+      headers: { cookie: sessionCookie(targetLogin) }
+    });
+    expect(newTicket.statusCode).toBe(403);
+  });
 });
 
 function sessionCookie(response: { headers: Record<string, string | string[] | number | undefined> }): string {
