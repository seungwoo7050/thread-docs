# PostgreSQL·메모리 저장소의 계약 동등성과 정규 매핑

## `test(db): 메모리 저장소 흐름 검증`

diff --git a/packages/db/src/index.test.ts b/packages/db/src/index.test.ts
new file mode 100644
index 0000000..a2064ee
--- /dev/null
+++ b/packages/db/src/index.test.ts
@@ -0,0 +1,32 @@
+import { describe, expect, it } from "vitest";
+import { createMemoryRepository } from "./index";
+
+describe("memory repository", () => {
+  it("creates sessions and keeps match results in the dashboard", async () => {
+    const repo = createMemoryRepository();
+    await repo.ensureSeedData();
+    const left = await repo.upsertDevUser({ handle: "left", displayName: "왼쪽" });
+    const right = await repo.upsertDevUser({ handle: "right", displayName: "오른쪽" });
+    const token = await repo.createSession(left.id);
+    await repo.createMatch({ mode: "queue", winnerId: left.id, loserId: right.id, scoreLeft: 3, scoreRight: 1 });
+
+    const session = await repo.getSessionUser(token);
+    const dashboard = await repo.getDashboard(left.id);
+
+    expect(session?.handle).toBe("left");
+    expect(dashboard.recentMatches[0].result).toBe("win");
+    expect(dashboard.me.wins).toBe(1);
+  });
+
+  it("tracks friend requests and tournament entries", async () => {
+    const repo = createMemoryRepository();
+    await repo.ensureSeedData();
+    const me = await repo.upsertDevUser({ handle: "me", displayName: "나" });
+    const friend = await repo.requestFriend(me.id, "spin-doctor");
+    const tournament = await repo.createTournament({ name: "테스트 컵", createdBy: me.id });
+
+    expect(friend.status).toBe("pending");
+    expect(tournament.playerCount).toBe(1);
+    expect((await repo.listTournaments())[0].name).toBe("테스트 컵");
+  });
+});


## `refactor(db): repository user projection 타입 정렬`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index cfc0c52..1246e4d 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -1,9 +1,44 @@
 import { randomUUID } from "node:crypto";
-import { Kysely, PostgresDialect, sql } from "kysely";
+import { Kysely, PostgresDialect, sql, type Transaction } from "kysely";
 import { Pool } from "pg";
-import type { AdminActionSummary, ChatMessage, DashboardSummary, FriendSummary, LeaderboardEntry, MatchMode, MatchSummary, PublicUser, SessionUser, TournamentMatchSummary, TournamentSummary, UserRole } from "@pong-pong/shared";
-import { toAdminActionSummary, toChatMessage, toFriendSummary, toMatchSummary, toPublicUser, toSessionUser, toTournamentMatchRecord, toTournamentMatchSummary, toTournamentSummary } from "./rowMappers";
-import type { AdminActionRow, ChatMessageRow, ChatMessageWithSenderRow, Database, FriendshipWithUserRow, MatchWithHandlesRow, MemoryUserRow, TournamentMatchRow, TournamentRow, TournamentWithCreatorRow, UserRow } from "./schema";
+import type {
+  ChatMessage,
+  DashboardSummary,
+  FriendSummary,
+  AdminActionSummary,
+  LeaderboardEntry,
+  MatchMode,
+  MatchSummary,
+  PublicUser,
+  SessionUser,
+  TournamentMatchSummary,
+  TournamentSummary,
+  UserRole
+} from "@pong-pong/shared";
+import {
+  toAdminActionSummary,
+  toChatMessage,
+  toFriendSummary,
+  toMatchSummary,
+  toPublicUser,
+  toSessionUser,
+  toTournamentMatchRecord,
+  toTournamentMatchSummary,
+  toTournamentSummary
+} from "./rowMappers";
+import type {
+  AdminActionRow,
+  ChatMessageRow,
+  ChatMessageWithSenderRow,
+  Database,
+  FriendshipWithUserRow,
+  MatchWithHandlesRow,
+  TournamentMatchRow,
+  TournamentRow,
+  TournamentWithCreatorRow,
+  UserProjectionRow,
+  UserRow
+} from "./schema";
 
 export type { Database } from "./schema";
 
@@ -704,7 +739,7 @@ class PostgresRepository implements AppRepository {
 }
 
 class MemoryRepository implements AppRepository {
-  private readonly users = new Map<string, MemoryUserRow>();
+  private readonly users = new Map<string, UserProjectionRow>();
   private readonly sessions = new Map<string, string>();
   private readonly wsTickets = new Map<string, { userId: string; expiresAt: number }>();
   private readonly matches: MemoryMatchRecord[] = [];
@@ -733,7 +768,19 @@ class MemoryRepository implements AppRepository {
     }
     for (const npc of NPC_PLAYERS) {
       const existing = [...this.users.values()].find((user) => user.handle === npc.handle);
-      const user: MemoryUserRow = existing ?? { id: randomUUID(), email: null, handle: npc.handle, display_name: npc.displayName, avatar_key: npc.avatarKey, role: "user", status: "active", rating: npc.rating, wins: 0, losses: 0, is_npc: true };
+      const user: UserProjectionRow = existing ?? {
+        id: randomUUID(),
+        email: null,
+        handle: npc.handle,
+        display_name: npc.displayName,
+        avatar_key: npc.avatarKey,
+        role: "user",
+        status: "active",
+        rating: npc.rating,
+        wins: 0,
+        losses: 0,
+        is_npc: true
+      };
       user.display_name = npc.displayName;
       user.avatar_key = npc.avatarKey;
       user.rating = npc.rating;
@@ -746,7 +793,7 @@ class MemoryRepository implements AppRepository {
   async upsertDevUser(input: DevLoginInput): Promise<SessionUser> {
     const handle = normalizeHandle(input.handle);
     const existing = [...this.users.values()].find((user) => user.handle === handle);
-    const user: MemoryUserRow = existing ?? {
+    const user: UserProjectionRow = existing ?? {
       id: randomUUID(),
       email: input.email ?? `${handle}@dev.pong-pong.local`,
       handle,


## `refactor(db): canonical row schema 타입 정렬`

diff --git a/packages/db/src/schema.ts b/packages/db/src/schema.ts
index 7305fc1..580c9dc 100644
--- a/packages/db/src/schema.ts
+++ b/packages/db/src/schema.ts
@@ -1,5 +1,16 @@
+import type {
+  FriendshipStatus,
+  MatchMode,
+  TournamentStatus,
+  UserRole,
+  UserStatus
+} from "@pong-pong/shared";
 import type { Generated, Selectable } from "kysely";
-import type { UserRole, UserStatus } from "@pong-pong/shared";
+
+export type TournamentRound = "semifinal" | "final";
+export type TournamentMatchStatus = "pending" | "ready" | "running" | "finished";
+export type ChatScope = "lobby" | "match";
+export type AdminAction = "ban" | "unban";
 
 export interface UserTable {
   id: Generated<string>;
@@ -24,17 +35,19 @@ export interface SessionTable {
   created_at: Generated<Date>;
 }
 
-export interface WsTicketTable {
-  ticket_hash: string;
-  user_id: string;
-  expires_at: Date;
+export interface FriendshipTable {
+  id: Generated<string>;
+  requester_id: string;
+  addressee_id: string;
+  status: FriendshipStatus;
   created_at: Generated<Date>;
+  updated_at: Generated<Date>;
 }
 
 export interface MatchTable {
   id: Generated<string>;
   result_key: string;
-  mode: import("@pong-pong/shared").MatchMode;
+  mode: MatchMode;
   winner_id: string | null;
   loser_id: string | null;
   score_left: number;
@@ -44,18 +57,9 @@ export interface MatchTable {
   ended_at: Generated<Date>;
 }
 
-export interface FriendshipTable {
-  id: Generated<string>;
-  requester_id: string;
-  addressee_id: string;
-  status: import("@pong-pong/shared").FriendshipStatus;
-  created_at: Generated<Date>;
-  updated_at: Generated<Date>;
-}
-
 export interface ChatMessageTable {
   id: Generated<string>;
-  scope: "lobby" | "match";
+  scope: ChatScope;
   room_id: string | null;
   sender_id: string;
   body: string;
@@ -65,7 +69,7 @@ export interface ChatMessageTable {
 export interface TournamentTable {
   id: Generated<string>;
   name: string;
-  status: Generated<import("@pong-pong/shared").TournamentStatus>;
+  status: Generated<TournamentStatus>;
   created_by: string;
   winner_id: string | null;
   capacity: Generated<number>;
@@ -83,9 +87,9 @@ export interface TournamentEntryTable {
 export interface TournamentMatchTable {
   id: Generated<string>;
   tournament_id: string;
-  round: "semifinal" | "final";
+  round: TournamentRound;
   slot: number;
-  status: Generated<"pending" | "ready" | "running" | "finished">;
+  status: Generated<TournamentMatchStatus>;
   left_user_id: string | null;
   right_user_id: string | null;
   winner_id: string | null;
@@ -101,11 +105,18 @@ export interface AdminActionTable {
   id: Generated<string>;
   actor_id: string | null;
   target_user_id: string | null;
-  action: "ban" | "unban";
+  action: AdminAction;
   reason: string;
   created_at: Generated<Date>;
 }
 
+export interface WsTicketTable {
+  ticket_hash: string;
+  user_id: string;
+  expires_at: Date;
+  created_at: Generated<Date>;
+}
+
 export interface RatingHistoryTable {
   id: Generated<string>;
   match_id: string;
@@ -116,25 +127,40 @@ export interface RatingHistoryTable {
   created_at: Generated<Date>;
 }
 
-export type AdminActionRow = Selectable<AdminActionTable>;
-
 export interface Database {
   users: UserTable;
   sessions: SessionTable;
-  ws_tickets: WsTicketTable;
-  matches: MatchTable;
   friendships: FriendshipTable;
+  matches: MatchTable;
   chat_messages: ChatMessageTable;
   tournaments: TournamentTable;
   tournament_entries: TournamentEntryTable;
   tournament_matches: TournamentMatchTable;
   admin_actions: AdminActionTable;
+  ws_tickets: WsTicketTable;
   rating_history: RatingHistoryTable;
 }
 
 export type UserRow = Selectable<UserTable>;
-export type MemoryUserRow = Omit<UserRow, "created_at" | "banned_at">;
+export type UserProjectionRow = Pick<
+  UserRow,
+  | "id"
+  | "email"
+  | "handle"
+  | "display_name"
+  | "avatar_key"
+  | "role"
+  | "status"
+  | "rating"
+  | "wins"
+  | "losses"
+  | "is_npc"
+>;
 export type MatchRow = Selectable<MatchTable>;
+export type ChatMessageRow = Selectable<ChatMessageTable>;
+export type TournamentRow = Selectable<TournamentTable>;
+export type TournamentMatchRow = Selectable<TournamentMatchTable>;
+export type AdminActionRow = Selectable<AdminActionTable>;
 
 export interface MatchWithHandlesRow extends MatchRow {
   winner_handle: string | null;
@@ -143,50 +169,33 @@ export interface MatchWithHandlesRow extends MatchRow {
 
 export interface FriendshipWithUserRow extends UserRow {
   friendship_id: string;
-  friendship_status: import("@pong-pong/shared").FriendshipStatus;
+  friendship_status: FriendshipStatus;
 }
 
-export type ChatMessageRow = Selectable<ChatMessageTable>;
 export interface ChatMessageWithSenderRow extends ChatMessageRow {
   user_id: string;
   email: string | null;
   handle: string;
   display_name: string;
   avatar_key: string;
-  role: import("@pong-pong/shared").UserRole;
-  status: import("@pong-pong/shared").UserStatus;
+  role: UserRole;
+  status: UserStatus;
   rating: number;
   wins: number;
   losses: number;
   is_npc: boolean;
 }
 
-export type TournamentRow = Selectable<TournamentTable>;
-export type TournamentMatchRow = Selectable<TournamentMatchTable>;
 export interface TournamentWithCreatorRow extends TournamentRow {
   creator_id: string;
   email: string | null;
   handle: string;
   display_name: string;
   avatar_key: string;
-  role: import("@pong-pong/shared").UserRole;
-  user_status: import("@pong-pong/shared").UserStatus;
+  role: UserRole;
+  user_status: UserStatus;
   rating: number;
   wins: number;
   losses: number;
   is_npc: boolean;
 }
-export type UserProjectionRow = Pick<
-  UserRow,
-  | "id"
-  | "email"
-  | "handle"
-  | "display_name"
-  | "avatar_key"
-  | "role"
-  | "status"
-  | "rating"
-  | "wins"
-  | "losses"
-  | "is_npc"
->;


## `refactor(db): row mapper record 타입 정렬`

diff --git a/packages/db/src/rowMappers.ts b/packages/db/src/rowMappers.ts
index 010ef24..ded6a0b 100644
--- a/packages/db/src/rowMappers.ts
+++ b/packages/db/src/rowMappers.ts
@@ -1,5 +1,33 @@
-import type { AdminActionSummary, ChatMessage, FriendSummary, MatchSummary, PublicUser, SessionUser, TournamentMatchSummary, TournamentSummary } from "@pong-pong/shared";
-import type { AdminActionRow, ChatMessageWithSenderRow, FriendshipWithUserRow, MatchWithHandlesRow, TournamentMatchRow, TournamentWithCreatorRow, UserProjectionRow } from "./schema";
+import type {
+  AdminActionSummary,
+  ChatMessage,
+  FriendSummary,
+  MatchSummary,
+  PublicUser,
+  SessionUser,
+  TournamentMatchSummary,
+  TournamentSummary
+} from "@pong-pong/shared";
+import type {
+  AdminActionRow,
+  ChatMessageWithSenderRow,
+  FriendshipWithUserRow,
+  MatchWithHandlesRow,
+  TournamentMatchRow,
+  TournamentWithCreatorRow,
+  UserProjectionRow
+} from "./schema";
+
+export interface TournamentMatchRecordView {
+  id: string;
+  tournamentId: string;
+  round: TournamentMatchRow["round"];
+  slot: number;
+  status: TournamentMatchRow["status"];
+  leftUserId: string | null;
+  rightUserId: string | null;
+  winnerId: string | null;
+}
 
 export function toPublicUser(row: UserProjectionRow, online = false): PublicUser {
   return {
@@ -36,7 +64,11 @@ export function toMatchSummary(row: MatchWithHandlesRow, userId?: string): Match
 }
 
 export function toFriendSummary(row: FriendshipWithUserRow): FriendSummary {
-  return { id: row.friendship_id, status: row.friendship_status, user: toPublicUser(row, true) };
+  return {
+    id: row.friendship_id,
+    status: row.friendship_status,
+    user: toPublicUser(row, true)
+  };
 }
 
 export function toChatMessage(row: ChatMessageWithSenderRow): ChatMessage {
@@ -102,8 +134,8 @@ export function toTournamentMatchSummary(
     left: users.left,
     right: users.right,
     winner: users.winner,
-    scoreLeft: row.score_left,
-    scoreRight: row.score_right,
+    scoreLeft: row.score_left == null ? null : Number(row.score_left),
+    scoreRight: row.score_right == null ? null : Number(row.score_right),
     roomId: row.room_id,
     matchId: row.match_id
   };
@@ -113,5 +145,12 @@ export function toAdminActionSummary(
   row: AdminActionRow,
   users: { actor: PublicUser | null; target: PublicUser | null }
 ): AdminActionSummary {
-  return { id: row.id, actor: users.actor, target: users.target, action: row.action, reason: row.reason, createdAt: row.created_at.toISOString() };
+  return {
+    id: row.id,
+    actor: users.actor,
+    target: users.target,
+    action: row.action,
+    reason: row.reason,
+    createdAt: row.created_at.toISOString()
+  };
 }


## `refactor(db): dashboard와 friendship 조회 경계 정렬`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index 1346904..ea4ce20 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -344,15 +344,24 @@ class PostgresRepository implements AppRepository {
   }
 
   async listRecentMatches(userId?: string): Promise<MatchSummary[]> {
-    const filter = userId ? sql`where m.winner_id = ${userId} or m.loser_id = ${userId}` : sql``;
-    const result = await sql<MatchWithHandlesRow>`
-      select m.*, winner.handle as winner_handle, loser.handle as loser_handle
-      from matches m
-      left join users winner on winner.id = m.winner_id
-      left join users loser on loser.id = m.loser_id
-      ${filter}
-      order by m.ended_at desc limit 8
-    `.execute(this.db);
+    const result = userId
+      ? await sql<MatchWithHandlesRow>`
+        select m.*, winner.handle as winner_handle, loser.handle as loser_handle
+        from matches m
+        left join users winner on winner.id = m.winner_id
+        left join users loser on loser.id = m.loser_id
+        where m.winner_id = ${userId} or m.loser_id = ${userId}
+        order by m.ended_at desc
+        limit 8
+      `.execute(this.db)
+      : await sql<MatchWithHandlesRow>`
+        select m.*, winner.handle as winner_handle, loser.handle as loser_handle
+        from matches m
+        left join users winner on winner.id = m.winner_id
+        left join users loser on loser.id = m.loser_id
+        order by m.ended_at desc
+        limit 8
+      `.execute(this.db);
     return result.rows.map((row) => toMatchSummary(row, userId));
   }
 
@@ -360,13 +369,19 @@ class PostgresRepository implements AppRepository {
     const user = await this.getUserById(userId);
     if (!user) throw new Error("user not found");
     const recentMatches = await this.listRecentMatches(userId);
-    return { me: { ...user, email: null }, recentMatches, winRate: percentage(user.wins, user.losses), bestStreak: bestWinningStreak(recentMatches) };
+    return {
+      me: { ...user, email: null },
+      recentMatches,
+      winRate: percentage(user.wins, user.losses),
+      bestStreak: bestWinningStreak(recentMatches)
+    };
   }
 
   async listFriends(userId: string): Promise<FriendSummary[]> {
     const result = await sql<FriendshipWithUserRow>`
       select f.id as friendship_id, f.status as friendship_status, u.*
-      from friendships f join users u on u.id = case when f.requester_id = ${userId} then f.addressee_id else f.requester_id end
+      from friendships f
+      join users u on u.id = case when f.requester_id = ${userId} then f.addressee_id else f.requester_id end
       where f.requester_id = ${userId} or f.addressee_id = ${userId}
       order by f.updated_at desc
     `.execute(this.db);


## `refactor(db): PostgreSQL chat과 tournament CRUD 정렬`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index b2ba038..55eb298 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -657,29 +657,55 @@ class PostgresRepository implements AppRepository {
   async listLobbyChat(): Promise<ChatMessage[]> {
     const result = await sql<ChatMessageWithSenderRow>`
       select c.*, u.id as user_id, u.email, u.handle, u.display_name, u.avatar_key, u.role, u.status, u.rating, u.wins, u.losses, u.is_npc
-      from chat_messages c join users u on u.id = c.sender_id where c.scope = 'lobby'
-      order by c.created_at desc limit 20
+      from chat_messages c
+      join users u on u.id = c.sender_id
+      where c.scope = 'lobby'
+      order by c.created_at desc
+      limit 20
     `.execute(this.db);
     return result.rows.reverse().map(toChatMessage);
   }
 
   async createChatMessage(input: { scope: "lobby" | "match"; roomId?: string | null; senderId: string; body: string }): Promise<ChatMessage> {
-    const result = await sql<ChatMessageRow>`insert into chat_messages (scope, room_id, sender_id, body) values (${input.scope}, ${input.roomId ?? null}, ${input.senderId}, ${input.body}) returning *`.execute(this.db);
+    const result = await sql<ChatMessageRow>`
+      insert into chat_messages (scope, room_id, sender_id, body)
+      values (${input.scope}, ${input.roomId ?? null}, ${input.senderId}, ${input.body})
+      returning *
+    `.execute(this.db);
     const user = await this.getUserById(input.senderId);
     if (!user) throw new Error("chat sender not found");
     const row = firstRow(result);
-    return { id: row.id, scope: row.scope, roomId: row.room_id, sender: user, body: row.body, createdAt: new Date(row.created_at).toISOString() };
+    return {
+      id: row.id,
+      scope: row.scope,
+      roomId: row.room_id,
+      sender: user,
+      body: row.body,
+      createdAt: new Date(row.created_at).toISOString()
+    };
   }
 
   async listTournaments(): Promise<TournamentSummary[]> {
-    const result = await sql<TournamentWithCreatorRow>`select t.*, u.id as creator_id, u.email, u.handle, u.display_name, u.avatar_key, u.role, u.status as user_status, u.rating, u.wins, u.losses, u.is_npc from tournaments t join users u on u.id = t.created_by order by t.created_at desc limit 10`.execute(this.db);
+    const result = await sql<TournamentWithCreatorRow>`
+      select t.*, u.id as creator_id, u.email, u.handle, u.display_name, u.avatar_key, u.role, u.status as user_status, u.rating, u.wins, u.losses, u.is_npc
+      from tournaments t
+      join users u on u.id = t.created_by
+      order by t.created_at desc
+      limit 10
+    `.execute(this.db);
     const summaries: TournamentSummary[] = [];
-    for (const row of result.rows) summaries.push(await this.tournamentFromRow(row));
+    for (const row of result.rows) {
+      summaries.push(await this.tournamentFromRow(row));
+    }
     return summaries;
   }
 
   async createTournament(input: { name: string; createdBy: string }): Promise<TournamentSummary> {
-    const result = await sql<TournamentRow>`insert into tournaments (name, created_by, capacity) values (${input.name}, ${input.createdBy}, 4) returning *`.execute(this.db);
+    const result = await sql<TournamentRow>`
+      insert into tournaments (name, created_by, capacity)
+      values (${input.name}, ${input.createdBy}, 4)
+      returning *
+    `.execute(this.db);
     await this.joinTournament(firstRow(result).id, input.createdBy);
     const tournaments = await this.listTournaments();
     return tournaments.find((item) => item.id === firstRow(result).id) ?? tournaments[0];
@@ -728,7 +754,8 @@ class PostgresRepository implements AppRepository {
         await this.ensureTournamentBracket(tournamentId, transaction);
       }
     });
-    const found = (await this.listTournaments()).find((item) => item.id === tournamentId);
+    const tournaments = await this.listTournaments();
+    const found = tournaments.find((item) => item.id === tournamentId);
     if (!found) throw new Error("tournament not found");
     return found;
   }
@@ -740,7 +767,8 @@ class PostgresRepository implements AppRepository {
 
   async startTournamentMatch(matchId: string, roomId: string): Promise<void> {
     await sql`
-      update tournament_matches set status = 'running', room_id = ${roomId}, updated_at = now()
+      update tournament_matches
+      set status = 'running', room_id = ${roomId}, updated_at = now()
       where id = ${matchId} and status in ('ready', 'running')
     `.execute(this.db);
   }
@@ -748,14 +776,24 @@ class PostgresRepository implements AppRepository {
   async completeTournamentMatch(input: { tournamentMatchId: string; roomId: string; matchId: string; winnerId: string | null; scoreLeft: number; scoreRight: number }): Promise<TournamentSummary> {
     const updated = await sql<TournamentMatchRow>`
       update tournament_matches
-      set status = 'finished', room_id = ${input.roomId}, match_id = ${input.matchId},
-          winner_id = ${input.winnerId}, score_left = ${input.scoreLeft}, score_right = ${input.scoreRight}, updated_at = now()
-      where id = ${input.tournamentMatchId} returning *
+      set status = 'finished',
+          room_id = ${input.roomId},
+          match_id = ${input.matchId},
+          winner_id = ${input.winnerId},
+          score_left = ${input.scoreLeft},
+          score_right = ${input.scoreRight},
+          updated_at = now()
+      where id = ${input.tournamentMatchId}
+      returning *
     `.execute(this.db);
     const row = firstRow(updated);
-    if (row.round === "semifinal") await this.ensureFinalMatch(row.tournament_id);
-    else await sql`update tournaments set status = 'finished', winner_id = ${input.winnerId} where id = ${row.tournament_id}`.execute(this.db);
-    const found = (await this.listTournaments()).find((item) => item.id === row.tournament_id);
+    if (row.round === "semifinal") {
+      await this.ensureFinalMatch(row.tournament_id);
+    } else {
+      await sql`update tournaments set status = 'finished', winner_id = ${input.winnerId} where id = ${row.tournament_id}`.execute(this.db);
+    }
+    const tournaments = await this.listTournaments();
+    const found = tournaments.find((item) => item.id === row.tournament_id);
     if (!found) throw new Error("tournament not found");
     return found;
   }


