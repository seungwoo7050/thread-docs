## `refactor(db): PostgreSQL tournament helper와 admin 경계 정렬`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index 55eb298..f1c24ae 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -798,19 +798,36 @@ class PostgresRepository implements AppRepository {
     return found;
   }
 
-  private async ensureFinalMatch(tournamentId: string): Promise<void> {
-    const semis = await sql<{ winner_id: string; slot: number }>`
-      select winner_id, slot from tournament_matches
-      where tournament_id = ${tournamentId} and round = 'semifinal'
-        and status = 'finished' and winner_id is not null
-      order by slot asc
+  async listAdminUsers(): Promise<PublicUser[]> {
+    const result = await sql<UserRow>`select * from users order by created_at desc limit 50`.execute(this.db);
+    return result.rows.map((row) => toPublicUser(row, true));
+  }
+
+  async listAdminActions(): Promise<AdminActionSummary[]> {
+    const result = await sql<AdminActionRow>`
+      select *
+      from admin_actions
+      order by created_at desc
+      limit 30
+    `.execute(this.db);
+    return Promise.all(result.rows.map(async (row) => toAdminActionSummary(row, {
+      actor: row.actor_id ? await this.getUserById(row.actor_id) : null,
+      target: row.target_user_id ? await this.getUserById(row.target_user_id) : null
+    })));
+  }
+
+  async setUserBan(actorId: string, targetUserId: string, banned: boolean, reason: string): Promise<PublicUser> {
+    const result = await sql<UserRow>`
+      update users
+      set status = ${banned ? "banned" : "active"}, banned_at = ${banned ? sql`now()` : null}
+      where id = ${targetUserId}
+      returning *
     `.execute(this.db);
-    if (semis.rows.length < 2) return;
     await sql`
-      insert into tournament_matches (tournament_id, round, slot, left_user_id, right_user_id, status)
-      values (${tournamentId}, 'final', 1, ${semis.rows[0].winner_id}, ${semis.rows[1].winner_id}, 'ready')
-      on conflict (tournament_id, round, slot) do nothing
+      insert into admin_actions (actor_id, target_user_id, action, reason)
+      values (${actorId}, ${targetUserId}, ${banned ? "ban" : "unban"}, ${reason})
     `.execute(this.db);
+    return toPublicUser(firstRow(result));
   }
 
   private async tournamentFromRow(row: TournamentWithCreatorRow): Promise<TournamentSummary> {
@@ -843,25 +860,28 @@ class PostgresRepository implements AppRepository {
     `.execute(executor);
   }
 
-  async listAdminUsers(): Promise<PublicUser[]> {
-    const result = await sql<UserRow>`select * from users order by created_at desc limit 50`.execute(this.db);
-    return result.rows.map((row) => toPublicUser(row, true));
-  }
-
-  async listAdminActions(): Promise<AdminActionSummary[]> {
-    const result = await sql<AdminActionRow>`select * from admin_actions order by created_at desc limit 30`.execute(this.db);
-    return Promise.all(result.rows.map(async (row) => toAdminActionSummary(row, {
-      actor: row.actor_id ? await this.getUserById(row.actor_id) : null,
-      target: row.target_user_id ? await this.getUserById(row.target_user_id) : null
-    })));
+  private async ensureFinalMatch(tournamentId: string): Promise<void> {
+    const semis = await sql<{ winner_id: string; slot: number }>`
+      select winner_id, slot
+      from tournament_matches
+      where tournament_id = ${tournamentId} and round = 'semifinal' and status = 'finished' and winner_id is not null
+      order by slot asc
+    `.execute(this.db);
+    if (semis.rows.length < 2) return;
+    await sql`
+      insert into tournament_matches (tournament_id, round, slot, left_user_id, right_user_id, status)
+      values (${tournamentId}, 'final', 1, ${semis.rows[0].winner_id}, ${semis.rows[1].winner_id}, 'ready')
+      on conflict (tournament_id, round, slot) do nothing
+    `.execute(this.db);
   }
 
-  async setUserBan(actorId: string, targetUserId: string, banned: boolean, reason: string): Promise<PublicUser> {
-    const result = await sql<UserRow>`update users set status = ${banned ? "banned" : "active"}, banned_at = ${banned ? sql`now()` : null} where id = ${targetUserId} returning *`.execute(this.db);
-    await sql`insert into admin_actions (actor_id, target_user_id, action, reason) values (${actorId}, ${targetUserId}, ${banned ? "ban" : "unban"}, ${reason})`.execute(this.db);
-    return toPublicUser(firstRow(result));
+  private async tournamentMatchFromRow(row: TournamentMatchRow): Promise<TournamentMatchSummary> {
+    return toTournamentMatchSummary(row, {
+      left: row.left_user_id ? await this.getUserById(row.left_user_id) : null,
+      right: row.right_user_id ? await this.getUserById(row.right_user_id) : null,
+      winner: row.winner_id ? await this.getUserById(row.winner_id) : null
+    });
   }
-
 }
 
 class MemoryRepository implements AppRepository {


## `refactor(db): tournament relation mapper 계약 정렬`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index f1c24ae..12816da 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -831,24 +831,35 @@ class PostgresRepository implements AppRepository {
   }
 
   private async tournamentFromRow(row: TournamentWithCreatorRow): Promise<TournamentSummary> {
-    const entries = await sql<UserRow>`select u.* from tournament_entries e join users u on u.id = e.user_id where e.tournament_id = ${row.id} order by e.seed asc`.execute(this.db);
+    const entries = await sql<UserRow>`
+      select u.*
+      from tournament_entries e
+      join users u on u.id = e.user_id
+      where e.tournament_id = ${row.id}
+      order by e.seed asc
+    `.execute(this.db);
     const matches = await sql<TournamentMatchRow>`
-      select * from tournament_matches where tournament_id = ${row.id}
+      select *
+      from tournament_matches
+      where tournament_id = ${row.id}
       order by case when round = 'semifinal' then 1 else 2 end, slot asc
     `.execute(this.db);
-    const summaries = await Promise.all(matches.rows.map(async (match) => toTournamentMatchSummary(match, {
-      left: match.left_user_id ? await this.getUserById(match.left_user_id) : null,
-      right: match.right_user_id ? await this.getUserById(match.right_user_id) : null,
-      winner: match.winner_id ? await this.getUserById(match.winner_id) : null
-    })));
-    const summary = toTournamentSummary(row, entries.rows.map((entry) => toPublicUser(entry, true)), summaries);
-    summary.winner = row.winner_id ? await this.getUserById(row.winner_id) : null;
-    return summary;
+    return toTournamentSummary(row, {
+      entries: entries.rows.map((entry) => toPublicUser(entry, true)),
+      matches: await Promise.all(matches.rows.map((match) => this.tournamentMatchFromRow(match))),
+      winner: row.winner_id ? await this.getUserById(row.winner_id) : null
+    });
   }
 
-  private async ensureTournamentBracket(tournamentId: string, executor: Kysely<Database> = this.db): Promise<void> {
+  private async ensureTournamentBracket(
+    tournamentId: string,
+    executor: Kysely<Database> | Transaction<Database> = this.db
+  ): Promise<void> {
     const entries = await sql<{ user_id: string; seed: number }>`
-      select user_id, seed from tournament_entries where tournament_id = ${tournamentId} order by seed asc
+      select user_id, seed
+      from tournament_entries
+      where tournament_id = ${tournamentId}
+      order by seed asc
     `.execute(executor);
     if (entries.rows.length < 4) return;
     await sql`
diff --git a/packages/db/src/rowMappers.ts b/packages/db/src/rowMappers.ts
index ded6a0b..435a09a 100644
--- a/packages/db/src/rowMappers.ts
+++ b/packages/db/src/rowMappers.ts
@@ -94,21 +94,7 @@ export function toChatMessage(row: ChatMessageWithSenderRow): ChatMessage {
   };
 }
 
-export function toTournamentSummary(row: TournamentWithCreatorRow, entries: PublicUser[], matches: TournamentMatchSummary[] = []): TournamentSummary {
-  return {
-    id: row.id,
-    name: row.name,
-    status: row.status,
-    createdBy: toPublicUser({ ...row, id: row.creator_id, status: row.user_status }),
-    playerCount: entries.length,
-    capacity: Number(row.capacity),
-    winner: null,
-    entries,
-    matches
-  };
-}
-
-export function toTournamentMatchRecord(row: TournamentMatchRow) {
+export function toTournamentMatchRecord(row: TournamentMatchRow): TournamentMatchRecordView {
   return {
     id: row.id,
     tournamentId: row.tournament_id,
@@ -141,6 +127,39 @@ export function toTournamentMatchSummary(
   };
 }
 
+export function toTournamentSummary(
+  row: TournamentWithCreatorRow,
+  related: {
+    entries: PublicUser[];
+    matches: TournamentMatchSummary[];
+    winner: PublicUser | null;
+  }
+): TournamentSummary {
+  return {
+    id: row.id,
+    name: row.name,
+    status: row.status,
+    createdBy: toPublicUser({
+      id: row.creator_id,
+      email: row.email,
+      handle: row.handle,
+      display_name: row.display_name,
+      avatar_key: row.avatar_key,
+      role: row.role,
+      status: row.user_status,
+      rating: row.rating,
+      wins: row.wins,
+      losses: row.losses,
+      is_npc: row.is_npc
+    }),
+    playerCount: related.entries.length,
+    capacity: Number(row.capacity),
+    winner: related.winner,
+    entries: related.entries,
+    matches: related.matches
+  };
+}
+
 export function toAdminActionSummary(
   row: AdminActionRow,
   users: { actor: PublicUser | null; target: PublicUser | null }


## `test(db): database row mapping contract 검증`

diff --git a/packages/db/src/rowMappers.test.ts b/packages/db/src/rowMappers.test.ts
new file mode 100644
index 0000000..e8cd374
--- /dev/null
+++ b/packages/db/src/rowMappers.test.ts
@@ -0,0 +1,204 @@
+import { describe, expect, it } from "vitest";
+import {
+  toAdminActionSummary,
+  toChatMessage,
+  toFriendSummary,
+  toMatchSummary,
+  toPublicUser,
+  toTournamentMatchRecord,
+  toTournamentMatchSummary,
+  toTournamentSummary
+} from "./rowMappers";
+import type {
+  AdminActionRow,
+  ChatMessageWithSenderRow,
+  FriendshipWithUserRow,
+  MatchWithHandlesRow,
+  TournamentMatchRow,
+  TournamentWithCreatorRow,
+  UserRow
+} from "./schema";
+
+const USER_ID = "00000000-0000-4000-8000-000000000001";
+const OTHER_USER_ID = "00000000-0000-4000-8000-000000000002";
+const MATCH_ID = "00000000-0000-4000-8000-000000000003";
+const TOURNAMENT_ID = "00000000-0000-4000-8000-000000000004";
+const CREATED_AT = new Date("2026-07-23T01:02:03.000Z");
+
+const userRow: UserRow = {
+  id: USER_ID,
+  email: "player@example.com",
+  handle: "typed-player",
+  display_name: "타입 선수",
+  avatar_key: "blue",
+  role: "user",
+  status: "active",
+  rating: 1342,
+  wins: 12,
+  losses: 7,
+  is_npc: false,
+  created_at: CREATED_AT,
+  banned_at: null
+};
+
+describe("database row mappers", () => {
+  it("maps a database user row without leaking column names", () => {
+    expect(toPublicUser(userRow, true)).toEqual({
+      id: USER_ID,
+      handle: "typed-player",
+      displayName: "타입 선수",
+      avatarKey: "blue",
+      role: "user",
+      status: "active",
+      rating: 1342,
+      wins: 12,
+      losses: 7,
+      online: true,
+      isNpc: false
+    });
+  });
+
+  it("maps joined match, friendship, and chat rows", () => {
+    const match: MatchWithHandlesRow = {
+      id: MATCH_ID,
+      result_key: "room:typed:finished",
+      mode: "queue",
+      winner_id: OTHER_USER_ID,
+      loser_id: USER_ID,
+      score_left: 3,
+      score_right: 1,
+      rating_delta: 16,
+      started_at: CREATED_AT,
+      ended_at: CREATED_AT,
+      winner_handle: "winner",
+      loser_handle: "typed-player"
+    };
+    const friendship: FriendshipWithUserRow = {
+      ...userRow,
+      friendship_id: "00000000-0000-4000-8000-000000000005",
+      friendship_status: "accepted"
+    };
+    const chat: ChatMessageWithSenderRow = {
+      id: "00000000-0000-4000-8000-000000000006",
+      scope: "lobby",
+      room_id: null,
+      sender_id: USER_ID,
+      body: "타입이 확인된 메시지",
+      created_at: CREATED_AT,
+      user_id: USER_ID,
+      email: userRow.email,
+      handle: userRow.handle,
+      display_name: userRow.display_name,
+      avatar_key: userRow.avatar_key,
+      role: userRow.role,
+      status: userRow.status,
+      rating: userRow.rating,
+      wins: userRow.wins,
+      losses: userRow.losses,
+      is_npc: userRow.is_npc
+    };
+
+    expect(toMatchSummary(match, USER_ID)).toMatchObject({
+      id: MATCH_ID,
+      opponentHandle: "winner",
+      result: "loss",
+      ratingDelta: -12,
+      endedAt: CREATED_AT.toISOString()
+    });
+    expect(toFriendSummary(friendship)).toMatchObject({
+      id: friendship.friendship_id,
+      status: "accepted",
+      user: { id: USER_ID, online: true }
+    });
+    expect(toChatMessage(chat)).toMatchObject({
+      id: chat.id,
+      sender: { id: USER_ID },
+      body: chat.body,
+      createdAt: CREATED_AT.toISOString()
+    });
+  });
+
+  it("maps tournament and admin rows with their related users", () => {
+    const match: TournamentMatchRow = {
+      id: MATCH_ID,
+      tournament_id: TOURNAMENT_ID,
+      round: "semifinal",
+      slot: 1,
+      status: "ready",
+      left_user_id: USER_ID,
+      right_user_id: OTHER_USER_ID,
+      winner_id: null,
+      room_id: null,
+      match_id: null,
+      score_left: null,
+      score_right: null,
+      created_at: CREATED_AT,
+      updated_at: CREATED_AT
+    };
+    const creator: TournamentWithCreatorRow = {
+      id: TOURNAMENT_ID,
+      name: "타입 컵",
+      status: "running",
+      created_by: USER_ID,
+      winner_id: null,
+      capacity: 4,
+      created_at: CREATED_AT,
+      creator_id: USER_ID,
+      email: userRow.email,
+      handle: userRow.handle,
+      display_name: userRow.display_name,
+      avatar_key: userRow.avatar_key,
+      role: userRow.role,
+      user_status: userRow.status,
+      rating: userRow.rating,
+      wins: userRow.wins,
+      losses: userRow.losses,
+      is_npc: userRow.is_npc
+    };
+    const publicUser = toPublicUser(userRow);
+    const matchSummary = toTournamentMatchSummary(match, {
+      left: publicUser,
+      right: null,
+      winner: null
+    });
+    const adminAction: AdminActionRow = {
+      id: "00000000-0000-4000-8000-000000000007",
+      actor_id: USER_ID,
+      target_user_id: OTHER_USER_ID,
+      action: "ban",
+      reason: "운영 정책 위반",
+      created_at: CREATED_AT
+    };
+
+    expect(toTournamentMatchRecord(match)).toEqual({
+      id: MATCH_ID,
+      tournamentId: TOURNAMENT_ID,
+      round: "semifinal",
+      slot: 1,
+      status: "ready",
+      leftUserId: USER_ID,
+      rightUserId: OTHER_USER_ID,
+      winnerId: null
+    });
+    expect(toTournamentSummary(creator, {
+      entries: [publicUser],
+      matches: [matchSummary],
+      winner: null
+    })).toMatchObject({
+      id: TOURNAMENT_ID,
+      name: "타입 컵",
+      createdBy: { id: USER_ID },
+      playerCount: 1,
+      capacity: 4
+    });
+    expect(toAdminActionSummary(adminAction, {
+      actor: publicUser,
+      target: null
+    })).toMatchObject({
+      action: "ban",
+      actor: { id: USER_ID },
+      target: null,
+      createdAt: CREATED_AT.toISOString()
+    });
+  });
+});
