# 계정 정지·감사 기록·실시간 권한 철회

## `feat(db): 관리자 상태 변경 저장 구현`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index a1bb4dd..e8f15ac 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -49,6 +49,8 @@ export interface AppRepository {
   listTournaments(): Promise<TournamentSummary[]>;
   createTournament(input: { name: string; createdBy: string }): Promise<TournamentSummary>;
   joinTournament(tournamentId: string, userId: string): Promise<TournamentSummary>;
+  listAdminUsers(): Promise<PublicUser[]>;
+  setUserBan(actorId: string, targetUserId: string, banned: boolean, reason: string): Promise<PublicUser>;
 }
 
 export function createPostgresRepository(databaseUrl: string): AppRepository {
@@ -263,6 +265,17 @@ class PostgresRepository implements AppRepository {
     return toTournamentSummary(row, entries.rows.map((entry) => toPublicUser(entry, true)));
   }
 
+  async listAdminUsers(): Promise<PublicUser[]> {
+    const result = await sql<UserRow>`select * from users order by created_at desc limit 50`.execute(this.db);
+    return result.rows.map((row) => toPublicUser(row, true));
+  }
+
+  async setUserBan(actorId: string, targetUserId: string, banned: boolean, reason: string): Promise<PublicUser> {
+    const result = await sql<UserRow>`update users set status = ${banned ? "banned" : "active"}, banned_at = ${banned ? sql`now()` : null} where id = ${targetUserId} returning *`.execute(this.db);
+    await sql`insert into admin_actions (actor_id, target_user_id, action, reason) values (${actorId}, ${targetUserId}, ${banned ? "ban" : "unban"}, ${reason})`.execute(this.db);
+    return toPublicUser(firstRow(result));
+  }
+
 }
 
 class MemoryRepository implements AppRepository {
@@ -421,6 +434,15 @@ class MemoryRepository implements AppRepository {
     return tournament;
   }
 
+  async listAdminUsers(): Promise<PublicUser[]> { return this.listOnlineUsers(); }
+
+  async setUserBan(_actorId: string, targetUserId: string, banned: boolean): Promise<PublicUser> {
+    const user = this.users.get(targetUserId);
+    if (!user) throw new Error("user not found");
+    user.status = banned ? "banned" : "active";
+    return toPublicUser(user, true);
+  }
+
 }
 
 function firstRow<T>(result: { rows: T[] }): T {
diff --git a/packages/db/src/schema.ts b/packages/db/src/schema.ts
index a3ab1b8..1f5b404 100644
--- a/packages/db/src/schema.ts
+++ b/packages/db/src/schema.ts
@@ -71,6 +71,15 @@ export interface TournamentEntryTable {
   created_at: Generated<Date>;
 }
 
+export interface AdminActionTable {
+  id: Generated<string>;
+  actor_id: string | null;
+  target_user_id: string | null;
+  action: "ban" | "unban";
+  reason: string;
+  created_at: Generated<Date>;
+}
+
 export interface Database {
   users: UserTable;
   sessions: SessionTable;
@@ -79,6 +88,7 @@ export interface Database {
   chat_messages: ChatMessageTable;
   tournaments: TournamentTable;
   tournament_entries: TournamentEntryTable;
+  admin_actions: AdminActionTable;
 }
 
 export type UserRow = Selectable<UserTable>;


## `feat(admin): 감사 가능한 사용자 상태 API 추가`

diff --git a/apps/api/src/app.ts b/apps/api/src/app.ts
index 63908bc..927613e 100644
--- a/apps/api/src/app.ts
+++ b/apps/api/src/app.ts
@@ -33,6 +33,10 @@ export function buildApp({ repo, webOrigin }: BuildAppOptions) {
             socket.close(1008, "unauthorized");
             return;
           }
+          if (user.status !== "active") {
+            socket.close(1008, "account suspended");
+            return;
+          }
           socket.off("message", bufferPayload);
           hub.connect(socket as WebSocket, request.raw, user, pendingPayloads);
         })
@@ -97,6 +101,7 @@ export function buildApp({ repo, webOrigin }: BuildAppOptions) {
   app.post("/chat/lobby", async (request, reply) => {
     const user = await currentUser(repo, request);
     if (!user) return unauthorized(reply);
+    if (!isActive(user)) return suspended(reply);
     const body = (request.body ?? {}) as { body?: string };
     const messageBody = body.body?.trim() ?? "";
     if (!messageBody) return reply.code(400).send({ message: "메시지를 입력해주세요." });
@@ -156,6 +161,7 @@ export function buildApp({ repo, webOrigin }: BuildAppOptions) {
   app.post("/friends/request", async (request, reply) => {
     const user = await currentUser(repo, request);
     if (!user) return unauthorized(reply);
+    if (!isActive(user)) return suspended(reply);
     const body = request.body as { handle?: string };
     return { friend: await repo.requestFriend(user.id, body.handle ?? "") };
   });
@@ -163,6 +169,7 @@ export function buildApp({ repo, webOrigin }: BuildAppOptions) {
   app.post("/friends", async (request, reply) => {
     const user = await currentUser(repo, request);
     if (!user) return unauthorized(reply);
+    if (!isActive(user)) return suspended(reply);
     const body = request.body as { handle?: string };
     return { friend: await repo.requestFriend(user.id, body.handle ?? "") };
   });
@@ -179,6 +186,7 @@ export function buildApp({ repo, webOrigin }: BuildAppOptions) {
   app.post("/tournaments", async (request, reply) => {
     const user = await currentUser(repo, request);
     if (!user) return unauthorized(reply);
+    if (!isActive(user)) return suspended(reply);
     const body = request.body as { name?: string };
     return { tournament: await repo.createTournament({ name: body.name ?? "퐁퐁 주간 컵", createdBy: user.id }) };
   });
@@ -186,6 +194,7 @@ export function buildApp({ repo, webOrigin }: BuildAppOptions) {
   app.post("/tournaments/:id/join", async (request, reply) => {
     const user = await currentUser(repo, request);
     if (!user) return unauthorized(reply);
+    if (!isActive(user)) return suspended(reply);
     const { id } = request.params as { id: string };
     return { tournament: await repo.joinTournament(id, user.id) };
   });
@@ -197,6 +206,13 @@ export function buildApp({ repo, webOrigin }: BuildAppOptions) {
     return { users: await repo.listAdminUsers() };
   });
 
+  app.get("/admin/actions", async (request, reply) => {
+    const user = await currentUser(repo, request);
+    if (!user) return unauthorized(reply);
+    if (user.role !== "admin") return reply.code(403).send({ message: "운영자 권한이 필요합니다." });
+    return { actions: await repo.listAdminActions() };
+  });
+
   app.post("/admin/users/:id/ban", async (request, reply) => {
     const user = await currentUser(repo, request);
     if (!user) return unauthorized(reply);
@@ -229,3 +245,11 @@ async function currentUser(repo: AppRepository, request: FastifyRequest): Promis
 function unauthorized(reply: FastifyReply) {
   return reply.code(401).send({ message: "로그인이 필요합니다." });
 }
+
+function suspended(reply: FastifyReply) {
+  return reply.code(403).send({ message: "정지된 계정은 이 작업을 수행할 수 없습니다." });
+}
+
+function isActive(user: SessionUser): boolean {
+  return user.status === "active";
+}
diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index bb8167f..f6c16af 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -1,10 +1,10 @@
 import { randomUUID } from "node:crypto";
 import { Kysely, PostgresDialect, sql } from "kysely";
 import { Pool } from "pg";
-import type { ChatMessage, DashboardSummary, FriendSummary, LeaderboardEntry, MatchMode, MatchSummary, PublicUser, SessionUser, TournamentMatchSummary, TournamentSummary } from "@pong-pong/shared";
+import type { AdminActionSummary, ChatMessage, DashboardSummary, FriendSummary, LeaderboardEntry, MatchMode, MatchSummary, PublicUser, SessionUser, TournamentMatchSummary, TournamentSummary } from "@pong-pong/shared";
 import { initialMigrationSql } from "./migrations";
-import { toChatMessage, toFriendSummary, toMatchSummary, toPublicUser, toSessionUser, toTournamentMatchRecord, toTournamentMatchSummary, toTournamentSummary } from "./rowMappers";
-import type { ChatMessageRow, ChatMessageWithSenderRow, Database, FriendshipWithUserRow, MatchWithHandlesRow, MemoryUserRow, TournamentMatchRow, TournamentRow, TournamentWithCreatorRow, UserRow } from "./schema";
+import { toAdminActionSummary, toChatMessage, toFriendSummary, toMatchSummary, toPublicUser, toSessionUser, toTournamentMatchRecord, toTournamentMatchSummary, toTournamentSummary } from "./rowMappers";
+import type { AdminActionRow, ChatMessageRow, ChatMessageWithSenderRow, Database, FriendshipWithUserRow, MatchWithHandlesRow, MemoryUserRow, TournamentMatchRow, TournamentRow, TournamentWithCreatorRow, UserRow } from "./schema";
 
 export type { Database } from "./schema";
 
@@ -64,6 +64,7 @@ export interface AppRepository {
   startTournamentMatch(matchId: string, roomId: string): Promise<void>;
   completeTournamentMatch(input: { tournamentMatchId: string; roomId: string; matchId: string; winnerId: string | null; scoreLeft: number; scoreRight: number }): Promise<TournamentSummary>;
   listAdminUsers(): Promise<PublicUser[]>;
+  listAdminActions(): Promise<AdminActionSummary[]>;
   setUserBan(actorId: string, targetUserId: string, banned: boolean, reason: string): Promise<PublicUser>;
 }
 
@@ -362,6 +363,14 @@ class PostgresRepository implements AppRepository {
     return result.rows.map((row) => toPublicUser(row, true));
   }
 
+  async listAdminActions(): Promise<AdminActionSummary[]> {
+    const result = await sql<AdminActionRow>`select * from admin_actions order by created_at desc limit 30`.execute(this.db);
+    return Promise.all(result.rows.map(async (row) => toAdminActionSummary(row, {
+      actor: row.actor_id ? await this.getUserById(row.actor_id) : null,
+      target: row.target_user_id ? await this.getUserById(row.target_user_id) : null
+    })));
+  }
+
   async setUserBan(actorId: string, targetUserId: string, banned: boolean, reason: string): Promise<PublicUser> {
     const result = await sql<UserRow>`update users set status = ${banned ? "banned" : "active"}, banned_at = ${banned ? sql`now()` : null} where id = ${targetUserId} returning *`.execute(this.db);
     await sql`insert into admin_actions (actor_id, target_user_id, action, reason) values (${actorId}, ${targetUserId}, ${banned ? "ban" : "unban"}, ${reason})`.execute(this.db);
@@ -377,6 +386,7 @@ class MemoryRepository implements AppRepository {
   private readonly friendships: FriendSummary[] = [];
   private readonly chats: ChatMessage[] = [];
   private readonly tournaments: TournamentSummary[] = [];
+  private readonly adminActions: AdminActionSummary[] = [];
 
   async close(): Promise<void> {}
 
@@ -579,11 +589,15 @@ class MemoryRepository implements AppRepository {
 
   async listAdminUsers(): Promise<PublicUser[]> { return this.listOnlineUsers(); }
 
-  async setUserBan(_actorId: string, targetUserId: string, banned: boolean): Promise<PublicUser> {
+  async listAdminActions(): Promise<AdminActionSummary[]> { return this.adminActions; }
+
+  async setUserBan(actorId: string, targetUserId: string, banned: boolean, reason: string): Promise<PublicUser> {
     const user = this.users.get(targetUserId);
     if (!user) throw new Error("user not found");
     user.status = banned ? "banned" : "active";
-    return toPublicUser(user, true);
+    const target = toPublicUser(user, true);
+    this.adminActions.unshift({ id: randomUUID(), actor: await this.getUserById(actorId), target, action: banned ? "ban" : "unban", reason, createdAt: new Date().toISOString() });
+    return target;
   }
 
 }
diff --git a/packages/db/src/rowMappers.ts b/packages/db/src/rowMappers.ts
index 987120b..d19c1b5 100644
--- a/packages/db/src/rowMappers.ts
+++ b/packages/db/src/rowMappers.ts
@@ -1,5 +1,5 @@
-import type { ChatMessage, FriendSummary, MatchSummary, PublicUser, SessionUser, TournamentMatchSummary, TournamentSummary } from "@pong-pong/shared";
-import type { ChatMessageWithSenderRow, FriendshipWithUserRow, MatchWithHandlesRow, TournamentMatchRow, TournamentWithCreatorRow, UserProjectionRow } from "./schema";
+import type { AdminActionSummary, ChatMessage, FriendSummary, MatchSummary, PublicUser, SessionUser, TournamentMatchSummary, TournamentSummary } from "@pong-pong/shared";
+import type { AdminActionRow, ChatMessageWithSenderRow, FriendshipWithUserRow, MatchWithHandlesRow, TournamentMatchRow, TournamentWithCreatorRow, UserProjectionRow } from "./schema";
 
 export function toPublicUser(row: UserProjectionRow, online = false): PublicUser {
   return {
@@ -106,3 +106,10 @@ export function toTournamentMatchSummary(
     matchId: row.match_id
   };
 }
+
+export function toAdminActionSummary(
+  row: AdminActionRow,
+  users: { actor: PublicUser | null; target: PublicUser | null }
+): AdminActionSummary {
+  return { id: row.id, actor: users.actor, target: users.target, action: row.action, reason: row.reason, createdAt: row.created_at.toISOString() };
+}
diff --git a/packages/db/src/schema.ts b/packages/db/src/schema.ts
index 816ccaf..280bd70 100644
--- a/packages/db/src/schema.ts
+++ b/packages/db/src/schema.ts
@@ -97,6 +97,8 @@ export interface AdminActionTable {
   created_at: Generated<Date>;
 }
 
+export type AdminActionRow = Selectable<AdminActionTable>;
+
 export interface Database {
   users: UserTable;
   sessions: SessionTable;
diff --git a/packages/shared/src/http.ts b/packages/shared/src/http.ts
index 8e436b7..301fd7f 100644
--- a/packages/shared/src/http.ts
+++ b/packages/shared/src/http.ts
@@ -102,3 +102,12 @@ export interface TournamentMatchSummary {
   roomId: string | null;
   matchId: string | null;
 }
+
+export interface AdminActionSummary {
+  id: string;
+  actor: PublicUser | null;
+  target: PublicUser | null;
+  action: "ban" | "unban";
+  reason: string;
+  createdAt: string;
+}


## `feat(admin): 감사 기록과 상태 변경 UI 추가`

diff --git a/apps/web/src/app/admin/page.tsx b/apps/web/src/app/admin/page.tsx
index 20b1f1e..b68287b 100644
--- a/apps/web/src/app/admin/page.tsx
+++ b/apps/web/src/app/admin/page.tsx
@@ -2,27 +2,31 @@
 
 import { useEffect, useState } from "react";
 import { Shield } from "lucide-react";
-import type { PublicUser } from "@pong-pong/shared";
+import type { AdminActionSummary, PublicUser } from "@pong-pong/shared";
 import { AppShell } from "@/components/AppShell";
-import { apiFetch, setUserStatus } from "@/lib/api";
+import { apiFetch, getAdminActions, setUserStatus } from "@/lib/api";
 
 export default function AdminPage() {
   const [users, setUsers] = useState<PublicUser[]>([]);
+  const [actions, setActions] = useState<AdminActionSummary[]>([]);
+  const [reason, setReason] = useState("운영자 검토");
   const [message, setMessage] = useState("운영자 계정으로 로그인하면 상태 변경이 저장됩니다.");
 
   useEffect(() => {
-    apiFetch<{ users: PublicUser[] }>("/admin/users")
-      .then((result) => {
+    Promise.all([apiFetch<{ users: PublicUser[] }>("/admin/users"), getAdminActions()])
+      .then(([result, actionItems]) => {
         setUsers(result.users);
-        setMessage("사용자 목록을 불러왔습니다.");
+        setActions(actionItems);
+        setMessage("사용자 목록과 감사 로그를 불러왔습니다.");
       })
       .catch(() => setMessage("운영자 권한이 필요합니다."));
   }, []);
 
   async function toggleUser(user: PublicUser) {
     try {
-      const updated = await setUserStatus(user.id, user.status === "active" ? "banned" : "active");
+      const updated = await setUserStatus(user.id, user.status === "active" ? "banned" : "active", reason.trim() || "운영자 검토");
       setUsers((current) => current.map((item) => (item.id === updated.id ? updated : item)));
+      setActions(await getAdminActions());
       setMessage(`${updated.displayName} 상태를 ${updated.status === "active" ? "정상" : "정지"}으로 변경했습니다.`);
     } catch {
       setMessage("상태 변경은 운영자 권한으로 로그인해야 가능합니다.");
@@ -39,6 +43,15 @@ export default function AdminPage() {
           <p className="mt-2 text-sm font-bold text-blue-700">{message}</p>
         </div>
       </div>
+      <section className="card mt-5 p-5">
+        <label className="text-sm font-black text-ink" htmlFor="admin-reason">조치 사유</label>
+        <input
+          id="admin-reason"
+          className="focus-ring mt-2 w-full rounded-lg border border-line px-3 py-2 text-sm font-semibold"
+          value={reason}
+          onChange={(event) => setReason(event.target.value)}
+        />
+      </section>
       <section className="card mt-5 overflow-hidden">
         <div className="grid grid-cols-[1fr_120px_120px_120px] border-b border-line px-5 py-3 text-sm font-black text-muted">
           <span>사용자</span>
@@ -61,6 +74,19 @@ export default function AdminPage() {
           </div>
         ))}
       </section>
+      <section className="card mt-5 overflow-hidden">
+        <div className="border-b border-line px-5 py-3 text-sm font-black text-muted">감사 로그</div>
+        {actions.length === 0 ? <p className="px-5 py-4 text-sm font-bold text-muted">기록된 운영 조치가 없습니다.</p> : null}
+        {actions.map((action) => (
+          <div key={action.id} className="grid gap-1 border-b border-line px-5 py-4 text-sm last:border-b-0">
+            <p className="font-black text-ink">
+              {action.target?.displayName ?? "대상 없음"} · {action.action === "ban" ? "정지" : "해제"}
+            </p>
+            <p className="font-semibold text-muted">{action.reason}</p>
+            <p className="text-xs font-bold text-muted">처리자 {action.actor?.displayName ?? "시스템"} · {new Date(action.createdAt).toLocaleString("ko-KR")}</p>
+          </div>
+        ))}
+      </section>
     </AppShell>
   );
 }
diff --git a/apps/web/src/lib/api.ts b/apps/web/src/lib/api.ts
index d42bfd7..fab12a8 100644
--- a/apps/web/src/lib/api.ts
+++ b/apps/web/src/lib/api.ts
@@ -1,4 +1,4 @@
-import type { ChatMessage, DashboardSummary, FriendSummary, LeaderboardEntry, LobbyResponse, MatchSummary, PublicUser, SessionUser, TournamentSummary } from "@pong-pong/shared";
+import type { AdminActionSummary, ChatMessage, DashboardSummary, FriendSummary, LeaderboardEntry, LobbyResponse, MatchSummary, PublicUser, SessionUser, TournamentSummary } from "@pong-pong/shared";
 
 const API_BASE = process.env.NEXT_PUBLIC_API_BASE_URL ?? "http://localhost:4000";
 
@@ -84,9 +84,13 @@ export async function requestFriend(handle: string): Promise<FriendSummary> {
   return (await apiFetch<{ friend: FriendSummary }>("/friends/request", { method: "POST", body: JSON.stringify({ handle }) })).friend;
 }
 
-export async function setUserStatus(id: string, status: "active" | "banned"): Promise<PublicUser> {
+export async function getAdminActions(): Promise<AdminActionSummary[]> {
+  return (await apiFetch<{ actions: AdminActionSummary[] }>("/admin/actions")).actions;
+}
+
+export async function setUserStatus(id: string, status: "active" | "banned", reason: string): Promise<PublicUser> {
   return (await apiFetch<{ user: PublicUser }>(`/admin/users/${id}/status`, {
     method: "PATCH",
-    body: JSON.stringify({ status, reason: "operator review" })
+    body: JSON.stringify({ status, reason })
   })).user;
 }


