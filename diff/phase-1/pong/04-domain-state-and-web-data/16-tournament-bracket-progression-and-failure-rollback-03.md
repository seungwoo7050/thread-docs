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


## `feat(db): PostgreSQL tournament 참가를 원자화`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index 4437522..f9fd6ff 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -556,17 +556,48 @@ class PostgresRepository implements AppRepository {
   }
 
   async joinTournament(tournamentId: string, userId: string): Promise<TournamentSummary> {
-    const count = await sql<{ count: string }>`select count(*)::text from tournament_entries where tournament_id = ${tournamentId}`.execute(this.db);
-    const status = await sql<{ capacity: number; joined: boolean }>`
-      select capacity, exists(select 1 from tournament_entries where tournament_id = ${tournamentId} and user_id = ${userId}) as joined
-      from tournaments where id = ${tournamentId} limit 1
-    `.execute(this.db);
-    const tournament = firstRow(status);
-    if (!tournament.joined && Number(firstRow(count).count) >= Number(tournament.capacity)) throw new Error("tournament full");
-    const seed = Number(firstRow(count).count) + 1;
-    await sql`insert into tournament_entries (tournament_id, user_id, seed) values (${tournamentId}, ${userId}, ${seed}) on conflict (tournament_id, user_id) do nothing`.execute(this.db);
-    await sql`update tournaments set status = case when (select count(*) from tournament_entries where tournament_id = ${tournamentId}) >= capacity then 'running' else status end where id = ${tournamentId}`.execute(this.db);
-    await this.ensureTournamentBracket(tournamentId);
+    await this.db.transaction().execute(async (transaction) => {
+      const tournament = await sql<{ capacity: number }>`
+        select capacity
+        from tournaments
+        where id = ${tournamentId}
+        for update
+      `.execute(transaction);
+      const tournamentRow = firstRow(tournament);
+      const existing = await sql<{ id: string }>`
+        select id
+        from tournament_entries
+        where tournament_id = ${tournamentId} and user_id = ${userId}
+        limit 1
+      `.execute(transaction);
+      if (existing.rows[0]) return;
+
+      const entryState = await sql<{ count: number; next_seed: number }>`
+        select
+          count(*)::integer as count,
+          (coalesce(max(seed), 0) + 1)::integer as next_seed
+        from tournament_entries
+        where tournament_id = ${tournamentId}
+      `.execute(transaction);
+      const state = firstRow(entryState);
+      if (Number(state.count) >= Number(tournamentRow.capacity)) {
+        throw new Error("tournament full");
+      }
+
+      await sql`
+        insert into tournament_entries (tournament_id, user_id, seed)
+        values (${tournamentId}, ${userId}, ${state.next_seed})
+      `.execute(transaction);
+      const playerCount = Number(state.count) + 1;
+      if (playerCount >= Number(tournamentRow.capacity)) {
+        await sql`
+          update tournaments
+          set status = 'running'
+          where id = ${tournamentId}
+        `.execute(transaction);
+        await this.ensureTournamentBracket(tournamentId, transaction);
+      }
+    });
     const found = (await this.listTournaments()).find((item) => item.id === tournamentId);
     if (!found) throw new Error("tournament not found");
     return found;
@@ -630,10 +661,10 @@ class PostgresRepository implements AppRepository {
     return summary;
   }
 
-  private async ensureTournamentBracket(tournamentId: string): Promise<void> {
+  private async ensureTournamentBracket(tournamentId: string, executor: Kysely<Database> = this.db): Promise<void> {
     const entries = await sql<{ user_id: string; seed: number }>`
       select user_id, seed from tournament_entries where tournament_id = ${tournamentId} order by seed asc
-    `.execute(this.db);
+    `.execute(executor);
     if (entries.rows.length < 4) return;
     await sql`
       insert into tournament_matches (tournament_id, round, slot, left_user_id, right_user_id, status)
@@ -641,7 +672,7 @@ class PostgresRepository implements AppRepository {
         (${tournamentId}, 'semifinal', 1, ${entries.rows[0].user_id}, ${entries.rows[3].user_id}, 'ready'),
         (${tournamentId}, 'semifinal', 2, ${entries.rows[1].user_id}, ${entries.rows[2].user_id}, 'ready')
       on conflict (tournament_id, round, slot) do nothing
-    `.execute(this.db);
+    `.execute(executor);
   }
 
   async listAdminUsers(): Promise<PublicUser[]> {


## `test(db): friendship와 tournament 경쟁 상태 검증`

diff --git a/packages/db/src/index.test.ts b/packages/db/src/index.test.ts
index 773738e..d6131e5 100644
--- a/packages/db/src/index.test.ts
+++ b/packages/db/src/index.test.ts
@@ -163,6 +163,71 @@ describe("memory repository", () => {
     expect((await repo.listTournaments())[0].name).toBe("테스트 컵");
   });
 
+  it("keeps one friendship for both request directions", async () => {
+    const repo = createMemoryRepository();
+    const firstUser = await repo.upsertDevUser({ handle: "friend-first", displayName: "첫 번째 사용자" });
+    const secondUser = await repo.upsertDevUser({ handle: "friend-second", displayName: "두 번째 사용자" });
+
+    await expect(repo.requestFriend(firstUser.id, firstUser.handle)).rejects.toThrow("cannot friend yourself");
+
+    const firstRequest = await repo.requestFriend(firstUser.id, secondUser.handle);
+    const repeatedRequest = await repo.requestFriend(firstUser.id, secondUser.handle);
+    const reverseRequest = await repo.requestFriend(secondUser.id, firstUser.handle);
+
+    expect(firstRequest.status).toBe("pending");
+    expect(repeatedRequest).toEqual(firstRequest);
+    expect(reverseRequest.id).toBe(firstRequest.id);
+    expect(reverseRequest.status).toBe("accepted");
+    expect(reverseRequest.user.id).toBe(firstUser.id);
+    await expect(repo.listFriends(firstUser.id)).resolves.toEqual([
+      expect.objectContaining({ id: firstRequest.id, status: "accepted", user: expect.objectContaining({ id: secondUser.id }) })
+    ]);
+    await expect(repo.listFriends(secondUser.id)).resolves.toEqual([
+      expect.objectContaining({ id: firstRequest.id, status: "accepted", user: expect.objectContaining({ id: firstUser.id }) })
+    ]);
+  });
+
+  it("admits one of ten users into the final tournament slot", async () => {
+    const repo = createMemoryRepository();
+    const creator = await repo.upsertDevUser({ handle: "memory-capacity-owner", displayName: "개설자" });
+    const earlyEntries = await Promise.all(
+      ["memory-capacity-two", "memory-capacity-three"].map((handle) =>
+        repo.upsertDevUser({ handle, displayName: handle })
+      )
+    );
+    const candidates = await Promise.all(
+      Array.from({ length: 10 }, (_, index) =>
+        repo.upsertDevUser({ handle: `memory-candidate-${index}`, displayName: `후보 ${index}` })
+      )
+    );
+    const tournament = await repo.createTournament({ name: "마지막 자리", createdBy: creator.id });
+    await repo.joinTournament(tournament.id, earlyEntries[0].id);
+    await repo.joinTournament(tournament.id, earlyEntries[1].id);
+
+    const attempts = await Promise.allSettled(
+      candidates.map((candidate) => repo.joinTournament(tournament.id, candidate.id))
+    );
+    const accepted = attempts.filter((attempt) => attempt.status === "fulfilled");
+    const rejected = attempts.filter((attempt) => attempt.status === "rejected");
+    const completed = (await repo.listTournaments()).find((item) => item.id === tournament.id);
+    const semifinalSlots = completed?.matches
+      .filter((match) => match.round === "semifinal")
+      .map((match) => match.slot)
+      .sort() ?? [];
+
+    expect(accepted).toHaveLength(1);
+    expect(rejected).toHaveLength(9);
+    expect(rejected.every((attempt) => String(attempt.reason).includes("tournament full"))).toBe(true);
+    expect(completed?.playerCount).toBe(4);
+    expect(new Set(completed?.entries.map((entry) => entry.id)).size).toBe(4);
+    expect(semifinalSlots).toEqual([1, 2]);
+
+    const acceptedUser = completed?.entries.find((entry) => candidates.some((candidate) => candidate.id === entry.id));
+    await expect(repo.joinTournament(tournament.id, acceptedUser?.id ?? "")).resolves.toMatchObject({
+      playerCount: 4
+    });
+  });
+
   it("consumes websocket tickets once and rejects expired or suspended users", async () => {
     const repo = createMemoryRepository();
     const user = await repo.upsertDevUser({ handle: "ws-user", displayName: "WS 사용자" });
diff --git a/packages/db/src/postgres.integration.test.ts b/packages/db/src/postgres.integration.test.ts
index 6eebc2b..4261c92 100644
--- a/packages/db/src/postgres.integration.test.ts
+++ b/packages/db/src/postgres.integration.test.ts
@@ -66,7 +66,12 @@ describe("PostgreSQL integration", () => {
         "ws_tickets"
       ]));
       const firstMigrations = await appliedMigrations(pool);
-      expect(firstMigrations).toEqual(["001_initial", "002_ws_tickets", "003_match_finalization"]);
+      expect(firstMigrations).toEqual([
+        "001_initial",
+        "002_ws_tickets",
+        "003_match_finalization",
+        "004_friendship_tournament_invariants"
+      ]);
 
       await migrateDatabase(databaseUrl);
 
@@ -396,6 +401,124 @@ describe("PostgreSQL integration", () => {
     });
   });
 
+  it("enforces one friendship across both request directions", async () => {
+    await withIsolatedDatabase(async ({ openPool, openRepository }) => {
+      const repository = openRepository();
+      const pool = openPool();
+      const firstUser = await repository.upsertDevUser({
+        handle: "pg-friend-first",
+        displayName: "Postgres Friend One"
+      });
+      const secondUser = await repository.upsertDevUser({
+        handle: "pg-friend-second",
+        displayName: "Postgres Friend Two"
+      });
+
+      await expect(repository.requestFriend(firstUser.id, firstUser.handle)).rejects.toThrow("cannot friend yourself");
+
+      const firstRequest = await repository.requestFriend(firstUser.id, secondUser.handle);
+      const repeatedRequest = await repository.requestFriend(firstUser.id, secondUser.handle);
+      const reverseRequest = await repository.requestFriend(secondUser.id, firstUser.handle);
+
+      expect(firstRequest.status).toBe("pending");
+      expect(repeatedRequest).toEqual(firstRequest);
+      expect(reverseRequest).toEqual(expect.objectContaining({
+        id: firstRequest.id,
+        status: "accepted",
+        user: expect.objectContaining({ id: firstUser.id })
+      }));
+      await expect(repository.listFriends(firstUser.id)).resolves.toEqual([
+        expect.objectContaining({ id: firstRequest.id, status: "accepted", user: expect.objectContaining({ id: secondUser.id }) })
+      ]);
+      await expect(repository.listFriends(secondUser.id)).resolves.toEqual([
+        expect.objectContaining({ id: firstRequest.id, status: "accepted", user: expect.objectContaining({ id: firstUser.id }) })
+      ]);
+
+      const stored = await pool.query<{
+        requester_id: string;
+        addressee_id: string;
+        status: string;
+      }>("select requester_id, addressee_id, status from friendships");
+      expect(stored.rows).toEqual([{
+        requester_id: firstUser.id,
+        addressee_id: secondUser.id,
+        status: "accepted"
+      }]);
+
+      await expect(pool.query(
+        "insert into friendships (requester_id, addressee_id, status) values ($1, $1, 'pending')",
+        [firstUser.id]
+      )).rejects.toMatchObject({ constraint: "friendships_distinct_users_check" });
+    });
+  });
+
+  it("admits exactly one of ten concurrent requests into the final tournament slot", async () => {
+    await withIsolatedDatabase(async ({ openPool, openRepository }) => {
+      const repository = openRepository();
+      const pool = openPool();
+      const creator = await repository.upsertDevUser({
+        handle: "pg-capacity-owner",
+        displayName: "Postgres Capacity Owner"
+      });
+      const earlyEntries = await Promise.all(
+        ["pg-capacity-two", "pg-capacity-three"].map((handle) =>
+          repository.upsertDevUser({ handle, displayName: handle })
+        )
+      );
+      const candidates = await Promise.all(
+        Array.from({ length: 10 }, (_, index) =>
+          repository.upsertDevUser({
+            handle: `pg-capacity-candidate-${index}`,
+            displayName: `Postgres Candidate ${index}`
+          })
+        )
+      );
+      const tournament = await repository.createTournament({
+        name: "Postgres Final Slot",
+        createdBy: creator.id
+      });
+      await repository.joinTournament(tournament.id, earlyEntries[0].id);
+      await repository.joinTournament(tournament.id, earlyEntries[1].id);
+
+      const attempts = await Promise.allSettled(
+        candidates.map((candidate) => repository.joinTournament(tournament.id, candidate.id))
+      );
+      const accepted = attempts.filter((attempt) => attempt.status === "fulfilled");
+      const rejected = attempts.filter((attempt) => attempt.status === "rejected");
+
+      expect(accepted).toHaveLength(1);
+      expect(rejected).toHaveLength(9);
+      expect(rejected.every((attempt) => String(attempt.reason).includes("tournament full"))).toBe(true);
+
+      const entries = await pool.query<{ user_id: string; seed: number }>(
+        "select user_id, seed from tournament_entries where tournament_id = $1 order by seed",
+        [tournament.id]
+      );
+      const matches = await pool.query<{ round: string; slot: number }>(
+        "select round, slot from tournament_matches where tournament_id = $1 order by round, slot",
+        [tournament.id]
+      );
+      expect(entries.rows).toHaveLength(4);
+      expect(entries.rows.map((entry) => entry.seed)).toEqual([1, 2, 3, 4]);
+      expect(new Set(entries.rows.map((entry) => entry.user_id)).size).toBe(4);
+      expect(matches.rows).toEqual([
+        { round: "semifinal", slot: 1 },
+        { round: "semifinal", slot: 2 }
+      ]);
+
+      const acceptedUserId = entries.rows.find((entry) => candidates.some((candidate) => candidate.id === entry.user_id))?.user_id;
+      await expect(repository.joinTournament(tournament.id, acceptedUserId ?? "")).resolves.toMatchObject({
+        playerCount: 4
+      });
+      const unchanged = await pool.query<{ entries: number; matches: number }>(`
+        select
+          (select count(*)::integer from tournament_entries where tournament_id = $1) as entries,
+          (select count(*)::integer from tournament_matches where tournament_id = $1) as matches
+      `, [tournament.id]);
+      expect(unchanged.rows[0]).toEqual({ entries: 4, matches: 2 });
+    });
+  });
+
   it("uses a fresh schema for each isolated database", async () => {
     let firstSchema = "";
 


## `fix(db): tournament start 상태 갱신 여부 확인`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index 16482bb..7d601c7 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -785,11 +785,13 @@ class PostgresRepository implements AppRepository {
   }
 
   async startTournamentMatch(matchId: string, roomId: string): Promise<void> {
-    await sql`
+    const updated = await sql<{ id: string }>`
       update tournament_matches
       set status = 'running', room_id = ${roomId}, updated_at = now()
       where id = ${matchId} and status in ('ready', 'running')
+      returning id
     `.execute(this.db);
+    if (updated.rows.length !== 1) throw new Error("tournament match not found");
   }
 
   async completeTournamentMatch(input: { tournamentMatchId: string; roomId: string; matchId: string; winnerId: string | null; scoreLeft: number; scoreRight: number }): Promise<TournamentSummary> {


## `fix(game): tournament 시작 실패 시 room 상태 복원`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index 3e999ab..fb49327 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -553,7 +553,13 @@ export class GameHub {
     const left = client.user.id === match.leftUserId ? client : opponent;
     const right = left === client ? opponent : client;
     const roomId = this.createRoom(left, right, { ai: false, mode: "tournament", tournamentMatchId: matchId });
-    await this.repo.startTournamentMatch(matchId, roomId);
+    try {
+      await this.repo.startTournamentMatch(matchId, roomId);
+    } catch (error) {
+      const room = this.rooms.get(roomId);
+      if (room) this.abandonRoom(room);
+      throw error;
+    }
   }
 
   private leaveQueue(client: Client): void {


## `test(game): tournament start rollback 검증`

diff --git a/apps/api/src/gameHub.tournament.test.ts b/apps/api/src/gameHub.tournament.test.ts
new file mode 100644
index 0000000..cb430a4
--- /dev/null
+++ b/apps/api/src/gameHub.tournament.test.ts
@@ -0,0 +1,107 @@
+import { EventEmitter } from "node:events";
+import type { IncomingMessage } from "node:http";
+import { afterEach, describe, expect, it, vi } from "vitest";
+import { WebSocket } from "ws";
+import { createMemoryRepository } from "@pong-pong/db";
+import { parseServerEvent, type ServerEvent, type SessionUser } from "@pong-pong/shared";
+import { GameHub } from "./gameHub.js";
+
+describe("GameHub tournament boundary", () => {
+  const repositories: Array<ReturnType<typeof createMemoryRepository>> = [];
+  const hubs: GameHub[] = [];
+
+  afterEach(async () => {
+    for (const hub of hubs.splice(0)) hub.close();
+    await Promise.all(repositories.splice(0).map((repository) => repository.close()));
+  });
+
+  it("rolls back the room when marking the tournament match as started fails", async () => {
+    const repository = createMemoryRepository();
+    repositories.push(repository);
+    vi.spyOn(repository, "getTournamentMatch").mockResolvedValue({
+      id: "tournament-match-1",
+      tournamentId: "tournament-1",
+      round: "semifinal",
+      slot: 1,
+      status: "ready",
+      leftUserId: "left-id",
+      rightUserId: "right-id",
+      winnerId: null
+    });
+    const startTournamentMatch = vi.spyOn(repository, "startTournamentMatch")
+      .mockRejectedValueOnce(new Error("database start failed"))
+      .mockResolvedValueOnce(undefined);
+    const hub = new GameHub(repository);
+    hubs.push(hub);
+    const left = connect(hub, player("left"));
+    const right = connect(hub, player("right"));
+
+    left.receive({ v: 1, type: "tournament.join", matchId: "tournament-match-1" });
+    right.receive({ v: 1, type: "tournament.join", matchId: "tournament-match-1" });
+
+    await expect.poll(() => startTournamentMatch.mock.calls.length).toBe(1);
+    await expect.poll(() => hub.liveStats().activeRooms).toBe(0);
+    expect(hub.scheduledRoomCount).toBe(0);
+
+    left.receive({ v: 1, type: "tournament.join", matchId: "tournament-match-1" });
+    right.receive({ v: 1, type: "tournament.join", matchId: "tournament-match-1" });
+
+    await expect.poll(() => startTournamentMatch.mock.calls.length).toBe(2);
+    expect(hub.liveStats().activeRooms).toBe(1);
+    expect(left.latest("queue.matched")).toEqual(expect.objectContaining({ side: "left" }));
+    expect(right.latest("queue.matched")).toEqual(expect.objectContaining({ side: "right" }));
+  });
+});
+
+function connect(hub: GameHub, user: SessionUser): FakeSocket {
+  const socket = new FakeSocket();
+  hub.connect(socket as unknown as WebSocket, {} as IncomingMessage, user);
+  return socket;
+}
+
+class FakeSocket extends EventEmitter {
+  readyState: number = WebSocket.OPEN;
+  bufferedAmount = 0;
+  private readonly payloads: string[] = [];
+
+  send(payload: string, callback?: (error?: Error) => void): void {
+    this.payloads.push(payload);
+    callback?.();
+  }
+
+  ping(): void {}
+
+  terminate(): void {
+    if (this.readyState === WebSocket.CLOSED) return;
+    this.readyState = WebSocket.CLOSED;
+    this.emit("close");
+  }
+
+  receive(event: object): void {
+    this.emit("message", Buffer.from(JSON.stringify(event)));
+  }
+
+  latest(type: ServerEvent["type"]): ServerEvent | undefined {
+    return this.payloads
+      .map((payload) => parseServerEvent(payload))
+      .filter((event) => event.type === type)
+      .at(-1);
+  }
+}
+
+function player(handle: "left" | "right"): SessionUser {
+  return {
+    id: `${handle}-id`,
+    handle,
+    displayName: handle,
+    avatarKey: "default",
+    role: "user",
+    status: "active",
+    rating: 1_200,
+    wins: 0,
+    losses: 0,
+    online: true,
+    isNpc: false,
+    email: null
+  };
+}
