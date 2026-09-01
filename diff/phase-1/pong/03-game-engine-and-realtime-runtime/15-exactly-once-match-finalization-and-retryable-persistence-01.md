# 경기 결과의 단일 확정과 재시도 가능한 영속화

## `feat(db): 경기 결과 저장 구현`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index 7169fc2..6b38a34 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -14,13 +14,16 @@ export interface DevLoginInput {
   email?: string | null;
 }
 
-type MemoryMatchRecord = {
-  id: string;
+export interface CreateMatchInput {
   mode: MatchMode;
   winnerId: string | null;
   loserId: string | null;
   scoreLeft: number;
   scoreRight: number;
+}
+
+type MemoryMatchRecord = CreateMatchInput & {
+  id: string;
   ended_at: string;
 };
 
@@ -40,6 +43,7 @@ export interface AppRepository {
   listFriends(userId: string): Promise<FriendSummary[]>;
   requestFriend(requesterId: string, addresseeHandle: string): Promise<FriendSummary>;
   acceptFriend(userId: string, friendshipId: string): Promise<FriendSummary>;
+  createMatch(input: CreateMatchInput): Promise<string>;
 }
 
 export function createPostgresRepository(databaseUrl: string): AppRepository {
@@ -202,6 +206,16 @@ class PostgresRepository implements AppRepository {
     return found;
   }
 
+  async createMatch(input: CreateMatchInput): Promise<string> {
+    const result = await sql<{ id: string }>`
+      insert into matches (mode, winner_id, loser_id, score_left, score_right, rating_delta)
+      values (${input.mode}, ${input.winnerId}, ${input.loserId}, ${input.scoreLeft}, ${input.scoreRight}, 16) returning id
+    `.execute(this.db);
+    if (input.winnerId) await sql`update users set wins = wins + 1, rating = rating + 16 where id = ${input.winnerId}`.execute(this.db);
+    if (input.loserId) await sql`update users set losses = losses + 1, rating = greatest(800, rating - 12) where id = ${input.loserId}`.execute(this.db);
+    return firstRow(result).id;
+  }
+
 }
 
 class MemoryRepository implements AppRepository {
@@ -318,6 +332,16 @@ class MemoryRepository implements AppRepository {
     return friend;
   }
 
+  async createMatch(input: CreateMatchInput): Promise<string> {
+    const id = randomUUID();
+    this.matches.push({ ...input, id, ended_at: new Date().toISOString() });
+    const winner = input.winnerId ? this.users.get(input.winnerId) : undefined;
+    if (winner) { winner.wins += 1; winner.rating += 16; }
+    const loser = input.loserId ? this.users.get(input.loserId) : undefined;
+    if (loser) { loser.losses += 1; loser.rating -= 12; }
+    return id;
+  }
+
 }
 
 function firstRow<T>(result: { rows: T[] }): T {


## `feat(game): 경기 종료와 결과 저장 연결`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index e9975e5..3d6780c 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -8,8 +8,10 @@ import {
   GAME_WIDTH,
   PADDLE_HEIGHT,
   TICK_RATE,
+  WINNING_SCORE,
   encodeServerEvent,
   parseClientEvent,
+  type GameFinished,
   type GameSnapshot,
   type PlayerSide,
   type ServerEvent,
@@ -39,7 +41,7 @@ export class GameHub {
 
   constructor(private readonly repo: AppRepository) {}
 
-  connect(socket: WebSocket, _request: IncomingMessage, user: SessionUser): void {
+  connect(socket: WebSocket, request: IncomingMessage, user: SessionUser): void {
     const client: Client = { id: randomUUID(), socket, user, roomId: null };
     this.clients.set(client.id, client);
     socket.on("message", (payload) => this.receive(client, payload.toString()));
@@ -78,10 +80,7 @@ export class GameHub {
     if (client.roomId) {
       const room = this.rooms.get(client.roomId);
       if (room) {
-        for (const participant of Object.values(room.clients)) {
-          if (participant) participant.roomId = null;
-        }
-        this.rooms.delete(room.id);
+        this.finishRoom(room, client === room.clients.left ? "right" : "left").catch(() => undefined);
       }
     }
     this.broadcastPresence();
@@ -208,6 +207,38 @@ export class GameHub {
     }
     this.broadcastRoom(room.id, { type: "game.snapshot", snapshot: state });
 
+    if (state.leftScore >= WINNING_SCORE || state.rightScore >= WINNING_SCORE || state.tick >= TICK_RATE * 45) {
+      await this.finishRoom(room, state.leftScore >= state.rightScore ? "left" : "right");
+    }
+  }
+
+  private async finishRoom(room: Room, winnerSide: PlayerSide): Promise<void> {
+    if (room.timer) clearInterval(room.timer);
+    room.timer = null;
+    room.snapshot.phase = "finished";
+    const winner = winnerSide === "left" ? room.clients.left : room.clients.right;
+    const loser = winnerSide === "left" ? room.clients.right : room.clients.left;
+    const matchId = await this.repo.createMatch({
+      mode: room.ai ? "ai" : "queue",
+      winnerId: winner?.user.id ?? null,
+      loserId: loser?.user.id ?? null,
+      scoreLeft: room.snapshot.leftScore,
+      scoreRight: room.snapshot.rightScore
+    });
+    const result: GameFinished = {
+      roomId: room.id,
+      matchId,
+      winnerSide,
+      leftScore: room.snapshot.leftScore,
+      rightScore: room.snapshot.rightScore,
+      ratingDelta: 16
+    };
+    this.broadcastRoom(room.id, { type: "game.finished", result });
+    for (const client of Object.values(room.clients)) {
+      if (client) client.roomId = null;
+    }
+    this.rooms.delete(room.id);
+    this.broadcastPresence();
   }
 
   private broadcastPresence(): void {


## `feat(db): match result key와 rating history schema 추가`

diff --git a/packages/db/migrations/003_match_finalization.sql b/packages/db/migrations/003_match_finalization.sql
new file mode 100644
index 0000000..5703d4e
--- /dev/null
+++ b/packages/db/migrations/003_match_finalization.sql
@@ -0,0 +1,24 @@
+alter table matches add column if not exists result_key text;
+
+update matches
+set result_key = 'legacy:' || id::text
+where result_key is null;
+
+alter table matches alter column result_key set not null;
+
+alter table matches
+  add constraint matches_result_key_unique unique (result_key);
+
+create table if not exists rating_history (
+  id uuid primary key default gen_random_uuid(),
+  match_id uuid not null references matches(id) on delete cascade,
+  user_id uuid not null references users(id) on delete cascade,
+  rating_before integer not null,
+  rating_after integer not null,
+  delta integer not null,
+  created_at timestamptz not null default now(),
+  unique (match_id, user_id)
+);
+
+create index if not exists rating_history_user_created_idx
+  on rating_history (user_id, created_at desc);
diff --git a/packages/db/src/schema.ts b/packages/db/src/schema.ts
index f494a78..7305fc1 100644
--- a/packages/db/src/schema.ts
+++ b/packages/db/src/schema.ts
@@ -33,6 +33,7 @@ export interface WsTicketTable {
 
 export interface MatchTable {
   id: Generated<string>;
+  result_key: string;
   mode: import("@pong-pong/shared").MatchMode;
   winner_id: string | null;
   loser_id: string | null;
@@ -105,6 +106,16 @@ export interface AdminActionTable {
   created_at: Generated<Date>;
 }
 
+export interface RatingHistoryTable {
+  id: Generated<string>;
+  match_id: string;
+  user_id: string;
+  rating_before: number;
+  rating_after: number;
+  delta: number;
+  created_at: Generated<Date>;
+}
+
 export type AdminActionRow = Selectable<AdminActionTable>;
 
 export interface Database {
@@ -118,6 +129,7 @@ export interface Database {
   tournament_entries: TournamentEntryTable;
   tournament_matches: TournamentMatchTable;
   admin_actions: AdminActionTable;
+  rating_history: RatingHistoryTable;
 }
 
 export type UserRow = Selectable<UserTable>;


## `feat(db): 경기 확정 command 계약 정의`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index 66116a9..cf18fd6 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -37,6 +37,20 @@ export interface CreateMatchInput {
   scoreRight: number;
 }
 
+export interface FinalizeMatchCommand extends CreateMatchInput {
+  resultKey: string;
+  tournament?: {
+    tournamentMatchId: string;
+    roomId: string;
+  };
+}
+
+export interface FinalizeMatchResult {
+  matchId: string;
+  resultKey: string;
+  created: boolean;
+}
+
 type MemoryMatchRecord = CreateMatchInput & {
   id: string;
   ended_at: string;
@@ -768,6 +782,29 @@ function assertTicketTtl(value: number): void {
   }
 }
 
+function assertFinalizeMatchCommand(command: FinalizeMatchCommand): void {
+  if (!command.resultKey.trim() || command.resultKey.length > 200) {
+    throw new Error("invalid match result key");
+  }
+  if (command.winnerId && command.winnerId === command.loserId) {
+    throw new Error("match participants must be different");
+  }
+  if (!Number.isInteger(command.scoreLeft) || command.scoreLeft < 0) {
+    throw new Error("invalid left score");
+  }
+  if (!Number.isInteger(command.scoreRight) || command.scoreRight < 0) {
+    throw new Error("invalid right score");
+  }
+  if (command.tournament) {
+    if (command.mode !== "tournament") {
+      throw new Error("tournament link requires tournament mode");
+    }
+    if (!command.tournament.tournamentMatchId || !command.tournament.roomId.trim()) {
+      throw new Error("invalid tournament match link");
+    }
+  }
+}
+
 function avatarFor(handle: string): string {
   const avatars = ["blue", "green", "amber", "violet", "rose"];
   return avatars[Math.abs([...handle].reduce((sum, char) => sum + char.charCodeAt(0), 0)) % avatars.length];


## `feat(db): PostgreSQL 경기 결과 중복 생성을 차단`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index cf18fd6..3cd9743 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -339,6 +339,41 @@ class PostgresRepository implements AppRepository {
     return firstRow(result).id;
   }
 
+  async finalizeMatch(command: FinalizeMatchCommand): Promise<FinalizeMatchResult> {
+    assertFinalizeMatchCommand(command);
+
+    return this.db.transaction().execute(async (transaction) => {
+      const inserted = await sql<{ id: string }>`
+        insert into matches (
+          result_key, mode, winner_id, loser_id, score_left, score_right, rating_delta
+        )
+        values (
+          ${command.resultKey}, ${command.mode}, ${command.winnerId}, ${command.loserId},
+          ${command.scoreLeft}, ${command.scoreRight}, 16
+        )
+        on conflict (result_key) do nothing
+        returning id
+      `.execute(transaction);
+
+      if (!inserted.rows[0]) {
+        const existing = await sql<{ id: string }>`
+          select id from matches where result_key = ${command.resultKey} limit 1
+        `.execute(transaction);
+        return {
+          matchId: firstRow(existing).id,
+          resultKey: command.resultKey,
+          created: false
+        };
+      }
+
+      return {
+        matchId: inserted.rows[0].id,
+        resultKey: command.resultKey,
+        created: true
+      };
+    });
+  }
+
   async listLobbyChat(): Promise<ChatMessage[]> {
     const result = await sql<ChatMessageWithSenderRow>`
       select c.*, u.id as user_id, u.email, u.handle, u.display_name, u.avatar_key, u.role, u.status, u.rating, u.wins, u.losses, u.is_npc


## `feat(db): PostgreSQL 참가자 rating을 원자적으로 반영`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index 3cd9743..ac670f6 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -366,8 +366,57 @@ class PostgresRepository implements AppRepository {
         };
       }
 
+      const matchId = inserted.rows[0].id;
+      const ratings = new Map<string, number>();
+      const participantIds = [command.winnerId, command.loserId]
+        .filter((id): id is string => id !== null)
+        .filter((id, index, values) => values.indexOf(id) === index)
+        .sort();
+
+      for (const userId of participantIds) {
+        const locked = await sql<{ id: string; rating: number }>`
+          select id, rating from users where id = ${userId} for update
+        `.execute(transaction);
+        const user = firstRow(locked);
+        ratings.set(user.id, Number(user.rating));
+      }
+
+      if (command.winnerId) {
+        const ratingBefore = requireRating(ratings, command.winnerId);
+        const ratingAfter = ratingBefore + 16;
+        await sql`
+          update users set wins = wins + 1, rating = ${ratingAfter}
+          where id = ${command.winnerId}
+        `.execute(transaction);
+        await sql`
+          insert into rating_history (
+            match_id, user_id, rating_before, rating_after, delta
+          ) values (
+            ${matchId}, ${command.winnerId}, ${ratingBefore}, ${ratingAfter},
+            ${ratingAfter - ratingBefore}
+          )
+        `.execute(transaction);
+      }
+
+      if (command.loserId) {
+        const ratingBefore = requireRating(ratings, command.loserId);
+        const ratingAfter = Math.max(800, ratingBefore - 12);
+        await sql`
+          update users set losses = losses + 1, rating = ${ratingAfter}
+          where id = ${command.loserId}
+        `.execute(transaction);
+        await sql`
+          insert into rating_history (
+            match_id, user_id, rating_before, rating_after, delta
+          ) values (
+            ${matchId}, ${command.loserId}, ${ratingBefore}, ${ratingAfter},
+            ${ratingAfter - ratingBefore}
+          )
+        `.execute(transaction);
+      }
+
       return {
-        matchId: inserted.rows[0].id,
+        matchId,
         resultKey: command.resultKey,
         created: true
       };
@@ -840,6 +889,12 @@ function assertFinalizeMatchCommand(command: FinalizeMatchCommand): void {
   }
 }
 
+function requireRating(ratings: Map<string, number>, userId: string): number {
+  const rating = ratings.get(userId);
+  if (rating === undefined) throw new Error("match participant not found");
+  return rating;
+}
+
 function avatarFor(handle: string): string {
   const avatars = ["blue", "green", "amber", "violet", "rose"];
   return avatars[Math.abs([...handle].reduce((sum, char) => sum + char.charCodeAt(0), 0)) % avatars.length];


## `feat(db): PostgreSQL tournament 경기 확정을 연결`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index ac670f6..cbc4747 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -415,6 +415,80 @@ class PostgresRepository implements AppRepository {
         `.execute(transaction);
       }
 
+      if (command.tournament) {
+        const tournamentMatch = await sql<{
+          id: string;
+          tournament_id: string;
+          round: "semifinal" | "final";
+          match_id: string | null;
+          left_user_id: string | null;
+          right_user_id: string | null;
+        }>`
+          select id, tournament_id, round, match_id, left_user_id, right_user_id
+          from tournament_matches
+          where id = ${command.tournament.tournamentMatchId}
+          for update
+        `.execute(transaction);
+        const tournamentMatchRow = tournamentMatch.rows[0];
+        if (!tournamentMatchRow) {
+          throw new Error("tournament match not found");
+        }
+
+        await sql`
+          select id from tournaments
+          where id = ${tournamentMatchRow.tournament_id}
+          for update
+        `.execute(transaction);
+        if (tournamentMatchRow.match_id) {
+          throw new Error("tournament match already finalized");
+        }
+        const tournamentParticipants = [
+          tournamentMatchRow.left_user_id,
+          tournamentMatchRow.right_user_id
+        ].filter((id): id is string => id !== null);
+        if (command.winnerId && !tournamentParticipants.includes(command.winnerId)) {
+          throw new Error("winner is not in tournament match");
+        }
+        if (command.loserId && !tournamentParticipants.includes(command.loserId)) {
+          throw new Error("loser is not in tournament match");
+        }
+
+        const linked = await sql<{ id: string }>`
+          update tournament_matches
+          set status = 'finished', room_id = ${command.tournament.roomId},
+              match_id = ${matchId}, winner_id = ${command.winnerId},
+              score_left = ${command.scoreLeft}, score_right = ${command.scoreRight},
+              updated_at = now()
+          where id = ${command.tournament.tournamentMatchId} and match_id is null
+          returning id
+        `.execute(transaction);
+        firstRow(linked);
+
+        if (tournamentMatchRow.round === "semifinal") {
+          const semifinals = await sql<{ winner_id: string; slot: number }>`
+            select winner_id, slot from tournament_matches
+            where tournament_id = ${tournamentMatchRow.tournament_id}
+              and round = 'semifinal' and status = 'finished' and winner_id is not null
+            order by slot asc
+          `.execute(transaction);
+          if (semifinals.rows.length === 2) {
+            await sql`
+              insert into tournament_matches (
+                tournament_id, round, slot, left_user_id, right_user_id, status
+              ) values (
+                ${tournamentMatchRow.tournament_id}, 'final', 1,
+                ${semifinals.rows[0].winner_id}, ${semifinals.rows[1].winner_id}, 'ready'
+              ) on conflict (tournament_id, round, slot) do nothing
+            `.execute(transaction);
+          }
+        } else {
+          await sql`
+            update tournaments set status = 'finished', winner_id = ${command.winnerId}
+            where id = ${tournamentMatchRow.tournament_id}
+          `.execute(transaction);
+        }
+      }
+
       return {
         matchId,
         resultKey: command.resultKey,


## `feat(db): memory 경기 결과 중복 생성을 차단`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index cbc4747..30def85 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -54,6 +54,7 @@ export interface FinalizeMatchResult {
 type MemoryMatchRecord = CreateMatchInput & {
   id: string;
   ended_at: string;
+  resultKey?: string;
 };
 
 export interface TournamentMatchRecord {
@@ -822,6 +823,31 @@ class MemoryRepository implements AppRepository {
     return id;
   }
 
+  async finalizeMatch(command: FinalizeMatchCommand): Promise<FinalizeMatchResult> {
+    assertFinalizeMatchCommand(command);
+
+    const existing = this.matches.find((match) => match.resultKey === command.resultKey);
+    if (existing) {
+      return {
+        matchId: existing.id,
+        resultKey: command.resultKey,
+        created: false
+      };
+    }
+
+    const matchId = randomUUID();
+    this.matches.push({
+      ...command,
+      id: matchId,
+      ended_at: new Date().toISOString()
+    });
+    return {
+      matchId,
+      resultKey: command.resultKey,
+      created: true
+    };
+  }
+
   async listLobbyChat(): Promise<ChatMessage[]> { return this.chats.filter((chat) => chat.scope === "lobby").slice(-20); }
 
   async createChatMessage(input: { scope: "lobby" | "match"; roomId?: string | null; senderId: string; body: string }): Promise<ChatMessage> {


## `feat(db): memory 참가자 rating을 원자적으로 반영`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index 30def85..cdfe184 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -835,12 +835,25 @@ class MemoryRepository implements AppRepository {
       };
     }
 
+    const winner = command.winnerId ? this.users.get(command.winnerId) : undefined;
+    const loser = command.loserId ? this.users.get(command.loserId) : undefined;
+    if (command.winnerId && !winner) throw new Error("winner not found");
+    if (command.loserId && !loser) throw new Error("loser not found");
+
     const matchId = randomUUID();
     this.matches.push({
       ...command,
       id: matchId,
       ended_at: new Date().toISOString()
     });
+    if (winner) {
+      winner.wins += 1;
+      winner.rating += 16;
+    }
+    if (loser) {
+      loser.losses += 1;
+      loser.rating = Math.max(800, loser.rating - 12);
+    }
     return {
       matchId,
       resultKey: command.resultKey,


## `feat(db): memory tournament 경기 확정을 연결`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index cdfe184..73b52fc 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -840,6 +840,33 @@ class MemoryRepository implements AppRepository {
     if (command.winnerId && !winner) throw new Error("winner not found");
     if (command.loserId && !loser) throw new Error("loser not found");
 
+    const tournamentLink = command.tournament
+      ? this.tournaments
+          .map((tournament) => ({
+            tournament,
+            match: tournament.matches.find((item) => item.id === command.tournament?.tournamentMatchId)
+          }))
+          .find((link) => link.match)
+      : null;
+    if (command.tournament && (!tournamentLink || !tournamentLink.match)) {
+      throw new Error("tournament match not found");
+    }
+    if (tournamentLink?.match?.matchId) {
+      throw new Error("tournament match already finalized");
+    }
+    if (tournamentLink?.match) {
+      const participantIds = [
+        tournamentLink.match.left?.id,
+        tournamentLink.match.right?.id
+      ].filter((id): id is string => id !== undefined);
+      if (command.winnerId && !participantIds.includes(command.winnerId)) {
+        throw new Error("winner is not in tournament match");
+      }
+      if (command.loserId && !participantIds.includes(command.loserId)) {
+        throw new Error("loser is not in tournament match");
+      }
+    }
+
     const matchId = randomUUID();
     this.matches.push({
       ...command,
@@ -854,6 +881,21 @@ class MemoryRepository implements AppRepository {
       loser.losses += 1;
       loser.rating = Math.max(800, loser.rating - 12);
     }
+    if (command.tournament && tournamentLink?.match) {
+      const { tournament, match } = tournamentLink;
+      match.status = "finished";
+      match.roomId = command.tournament.roomId;
+      match.matchId = matchId;
+      match.winner = winner ? toPublicUser(winner, true) : null;
+      match.scoreLeft = command.scoreLeft;
+      match.scoreRight = command.scoreRight;
+      if (match.round === "semifinal") {
+        this.ensureMemoryFinal(tournament);
+      } else {
+        tournament.status = "finished";
+        tournament.winner = winner ? toPublicUser(winner, true) : null;
+      }
+    }
     return {
       matchId,
       resultKey: command.resultKey,


