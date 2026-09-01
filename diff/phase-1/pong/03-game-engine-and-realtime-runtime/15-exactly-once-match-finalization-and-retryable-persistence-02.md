## `test(db): 경기 결과 단일 확정 조건 검증`

diff --git a/packages/db/src/index.test.ts b/packages/db/src/index.test.ts
index 69c1c38..773738e 100644
--- a/packages/db/src/index.test.ts
+++ b/packages/db/src/index.test.ts
@@ -33,6 +33,80 @@ describe("memory repository", () => {
     expect(dashboard.me.rating).toBe(left.rating + 32);
   });
 
+  it("finalizes the same match result once when commands are repeated", async () => {
+    const repo = createMemoryRepository();
+    const winner = await repo.upsertDevUser({ handle: "winner", displayName: "승자" });
+    const loser = await repo.upsertDevUser({ handle: "loser", displayName: "패자" });
+    const command = {
+      resultKey: "room:memory-finalize:finished",
+      mode: "queue" as const,
+      winnerId: winner.id,
+      loserId: loser.id,
+      scoreLeft: 3,
+      scoreRight: 1
+    };
+
+    const results = await Promise.all(
+      Array.from({ length: 20 }, () => repo.finalizeMatch(command))
+    );
+    const dashboard = await repo.getDashboard(winner.id);
+    const updatedLoser = await repo.getUserById(loser.id);
+
+    expect(new Set(results.map((result) => result.matchId)).size).toBe(1);
+    expect(results.filter((result) => result.created)).toHaveLength(1);
+    expect(dashboard.recentMatches).toHaveLength(1);
+    expect(dashboard.me.wins).toBe(1);
+    expect(dashboard.me.rating).toBe(winner.rating + 16);
+    expect(updatedLoser?.losses).toBe(1);
+    expect(updatedLoser?.rating).toBe(loser.rating - 12);
+  });
+
+  it("links concurrent semifinal results and creates one final", async () => {
+    const repo = createMemoryRepository();
+    const players = await Promise.all(
+      ["semi-one", "semi-two", "semi-three", "semi-four"].map((handle, index) =>
+        repo.upsertDevUser({ handle, displayName: `선수 ${index + 1}` })
+      )
+    );
+    const tournament = await repo.createTournament({
+      name: "동시 종료 컵",
+      createdBy: players[0].id
+    });
+    await repo.joinTournament(tournament.id, players[1].id);
+    await repo.joinTournament(tournament.id, players[2].id);
+    const ready = await repo.joinTournament(tournament.id, players[3].id);
+    const [semiA, semiB] = ready.matches.filter((match) => match.round === "semifinal");
+
+    await Promise.all([
+      repo.finalizeMatch({
+        resultKey: "room:memory-semi-a:finished",
+        mode: "tournament",
+        winnerId: semiA.left?.id ?? null,
+        loserId: semiA.right?.id ?? null,
+        scoreLeft: 3,
+        scoreRight: 1,
+        tournament: { tournamentMatchId: semiA.id, roomId: "memory-semi-a" }
+      }),
+      repo.finalizeMatch({
+        resultKey: "room:memory-semi-b:finished",
+        mode: "tournament",
+        winnerId: semiB.left?.id ?? null,
+        loserId: semiB.right?.id ?? null,
+        scoreLeft: 3,
+        scoreRight: 2,
+        tournament: { tournamentMatchId: semiB.id, roomId: "memory-semi-b" }
+      })
+    ]);
+
+    const completed = (await repo.listTournaments()).find((item) => item.id === tournament.id);
+    const finalMatches = completed?.matches.filter((match) => match.round === "final") ?? [];
+
+    expect(finalMatches).toHaveLength(1);
+    expect(finalMatches[0].left?.id).toBe(semiA.left?.id);
+    expect(finalMatches[0].right?.id).toBe(semiB.left?.id);
+    expect(completed?.matches.filter((match) => match.round === "semifinal" && match.matchId)).toHaveLength(2);
+  });
+
   it("derives the best streak from recent match results", async () => {
     const repo = createMemoryRepository();
     await repo.ensureSeedData();
diff --git a/packages/db/src/postgres.integration.test.ts b/packages/db/src/postgres.integration.test.ts
index bfbd941..6eebc2b 100644
--- a/packages/db/src/postgres.integration.test.ts
+++ b/packages/db/src/postgres.integration.test.ts
@@ -57,6 +57,7 @@ describe("PostgreSQL integration", () => {
         "chat_messages",
         "friendships",
         "matches",
+        "rating_history",
         "sessions",
         "tournament_entries",
         "tournament_matches",
@@ -65,7 +66,7 @@ describe("PostgreSQL integration", () => {
         "ws_tickets"
       ]));
       const firstMigrations = await appliedMigrations(pool);
-      expect(firstMigrations).toEqual(["001_initial", "002_ws_tickets"]);
+      expect(firstMigrations).toEqual(["001_initial", "002_ws_tickets", "003_match_finalization"]);
 
       await migrateDatabase(databaseUrl);
 
@@ -181,6 +182,220 @@ describe("PostgreSQL integration", () => {
     });
   });
 
+  it("applies a match result and rating changes once across 20 concurrent calls", async () => {
+    await withIsolatedDatabase(async ({ openPool, openRepository }) => {
+      const repository = openRepository();
+      const pool = openPool();
+      const winner = await repository.upsertDevUser({
+        handle: "finalize-winner",
+        displayName: "Finalize Winner"
+      });
+      const loser = await repository.upsertDevUser({
+        handle: "finalize-loser",
+        displayName: "Finalize Loser"
+      });
+      const command = {
+        resultKey: "room:postgres-finalize:finished",
+        mode: "queue" as const,
+        winnerId: winner.id,
+        loserId: loser.id,
+        scoreLeft: 3,
+        scoreRight: 1
+      };
+
+      const results = await Promise.all(
+        Array.from({ length: 20 }, () => repository.finalizeMatch(command))
+      );
+
+      expect(new Set(results.map((result) => result.matchId)).size).toBe(1);
+      expect(results.filter((result) => result.created)).toHaveLength(1);
+
+      const matches = await pool.query<{
+        id: string;
+        result_key: string;
+      }>("select id, result_key from matches");
+      expect(matches.rows).toEqual([{
+        id: results[0].matchId,
+        result_key: command.resultKey
+      }]);
+
+      const users = await pool.query<{
+        handle: string;
+        rating: number;
+        wins: number;
+        losses: number;
+      }>(
+        "select handle, rating, wins, losses from users where id = any($1::uuid[]) order by handle",
+        [[winner.id, loser.id]]
+      );
+      expect(users.rows).toEqual([
+        { handle: "finalize-loser", rating: loser.rating - 12, wins: 0, losses: 1 },
+        { handle: "finalize-winner", rating: winner.rating + 16, wins: 1, losses: 0 }
+      ]);
+
+      const history = await pool.query<{
+        handle: string;
+        rating_before: number;
+        rating_after: number;
+        delta: number;
+      }>(`
+        select u.handle, h.rating_before, h.rating_after, h.delta
+        from rating_history h
+        join users u on u.id = h.user_id
+        order by u.handle
+      `);
+      expect(history.rows).toEqual([
+        {
+          handle: "finalize-loser",
+          rating_before: loser.rating,
+          rating_after: loser.rating - 12,
+          delta: -12
+        },
+        {
+          handle: "finalize-winner",
+          rating_before: winner.rating,
+          rating_after: winner.rating + 16,
+          delta: 16
+        }
+      ]);
+    });
+  });
+
+  it("rolls back the result and ratings when tournament linking fails", async () => {
+    await withIsolatedDatabase(async ({ openPool, openRepository }) => {
+      const repository = openRepository();
+      const pool = openPool();
+      const winner = await repository.upsertDevUser({
+        handle: "rollback-winner",
+        displayName: "Rollback Winner"
+      });
+      const loser = await repository.upsertDevUser({
+        handle: "rollback-loser",
+        displayName: "Rollback Loser"
+      });
+      const resultKey = "room:rollback-finalize:finished";
+
+      await expect(repository.finalizeMatch({
+        resultKey,
+        mode: "tournament",
+        winnerId: winner.id,
+        loserId: loser.id,
+        scoreLeft: 3,
+        scoreRight: 0,
+        tournament: {
+          tournamentMatchId: randomUUID(),
+          roomId: "rollback-finalize"
+        }
+      })).rejects.toThrow("tournament match not found");
+
+      const counts = await pool.query<{
+        matches: number;
+        history: number;
+      }>(`
+        select
+          (select count(*)::integer from matches where result_key = $1) as matches,
+          (select count(*)::integer from rating_history) as history
+      `, [resultKey]);
+      expect(counts.rows[0]).toEqual({ matches: 0, history: 0 });
+
+      const users = await pool.query<{
+        handle: string;
+        rating: number;
+        wins: number;
+        losses: number;
+      }>(
+        "select handle, rating, wins, losses from users where id = any($1::uuid[]) order by handle",
+        [[winner.id, loser.id]]
+      );
+      expect(users.rows).toEqual([
+        { handle: "rollback-loser", rating: loser.rating, wins: 0, losses: 0 },
+        { handle: "rollback-winner", rating: winner.rating, wins: 0, losses: 0 }
+      ]);
+    });
+  });
+
+  it("links concurrent semifinals and creates exactly one final", async () => {
+    await withIsolatedDatabase(async ({ openPool, openRepository }) => {
+      const repository = openRepository();
+      const pool = openPool();
+      const players = await Promise.all(
+        ["pg-semi-one", "pg-semi-two", "pg-semi-three", "pg-semi-four"].map((handle, index) =>
+          repository.upsertDevUser({ handle, displayName: `Postgres Player ${index + 1}` })
+        )
+      );
+      const tournament = await repository.createTournament({
+        name: "Postgres Concurrent Cup",
+        createdBy: players[0].id
+      });
+      await repository.joinTournament(tournament.id, players[1].id);
+      await repository.joinTournament(tournament.id, players[2].id);
+      const ready = await repository.joinTournament(tournament.id, players[3].id);
+      const [semiA, semiB] = ready.matches.filter((match) => match.round === "semifinal");
+
+      const [resultA, resultB] = await Promise.all([
+        repository.finalizeMatch({
+          resultKey: "room:postgres-semi-a:finished",
+          mode: "tournament",
+          winnerId: semiA.left?.id ?? null,
+          loserId: semiA.right?.id ?? null,
+          scoreLeft: 3,
+          scoreRight: 1,
+          tournament: { tournamentMatchId: semiA.id, roomId: "postgres-semi-a" }
+        }),
+        repository.finalizeMatch({
+          resultKey: "room:postgres-semi-b:finished",
+          mode: "tournament",
+          winnerId: semiB.left?.id ?? null,
+          loserId: semiB.right?.id ?? null,
+          scoreLeft: 3,
+          scoreRight: 2,
+          tournament: { tournamentMatchId: semiB.id, roomId: "postgres-semi-b" }
+        })
+      ]);
+
+      const tournamentMatches = await pool.query<{
+        round: string;
+        slot: number;
+        status: string;
+        left_user_id: string | null;
+        right_user_id: string | null;
+        match_id: string | null;
+      }>(`
+        select round, slot, status, left_user_id, right_user_id, match_id
+        from tournament_matches
+        where tournament_id = $1
+        order by case when round = 'semifinal' then 1 else 2 end, slot
+      `, [tournament.id]);
+      const semifinals = tournamentMatches.rows.filter((match) => match.round === "semifinal");
+      const finals = tournamentMatches.rows.filter((match) => match.round === "final");
+
+      expect(semifinals.map((match) => match.match_id).sort()).toEqual(
+        [resultA.matchId, resultB.matchId].sort()
+      );
+      expect(semifinals.every((match) => match.status === "finished")).toBe(true);
+      expect(finals).toEqual([expect.objectContaining({
+        round: "final",
+        slot: 1,
+        status: "ready",
+        left_user_id: semiA.left?.id,
+        right_user_id: semiB.left?.id,
+        match_id: null
+      })]);
+
+      const counts = await pool.query<{
+        matches: number;
+        history: number;
+        finals: number;
+      }>(`
+        select
+          (select count(*)::integer from matches) as matches,
+          (select count(*)::integer from rating_history) as history,
+          (select count(*)::integer from tournament_matches where tournament_id = $1 and round = 'final') as finals
+      `, [tournament.id]);
+      expect(counts.rows[0]).toEqual({ matches: 2, history: 4, finals: 1 });
+    });
+  });
+
   it("uses a fresh schema for each isolated database", async () => {
     let firstSchema = "";
 


## `refactor(game): 경기 결과 확정 boundary 사용`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index 4cc25c3..fea4d82 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -48,6 +48,7 @@ type Room = {
   npcUser: PublicUser | null;
   simulation: PongSimulationState;
   aiController: PongAi | null;
+  finishing: Promise<void> | null;
 };
 
 const NPC_QUEUE_FALLBACK_MS = 6000;
@@ -295,6 +296,7 @@ export class GameHub {
       npcUser,
       simulation,
       aiController: options.ai ? new PongAi(roomId, npcUser?.rating ?? 1200) : null,
+      finishing: null,
       snapshot: {
         roomId,
         tick: 0,
@@ -400,7 +402,17 @@ export class GameHub {
     }
   }
 
-  private async finishRoom(room: Room, winnerSide: PlayerSide): Promise<void> {
+  private finishRoom(room: Room, winnerSide: PlayerSide): Promise<void> {
+    if (room.finishing) return room.finishing;
+    const finalization = this.finalizeRoom(room, winnerSide);
+    room.finishing = finalization;
+    void finalization.catch(() => {
+      if (room.finishing === finalization) room.finishing = null;
+    });
+    return finalization;
+  }
+
+  private async finalizeRoom(room: Room, winnerSide: PlayerSide): Promise<void> {
     if (room.timer) clearInterval(room.timer);
     room.timer = null;
     room.snapshot.state.phase = "finished";
@@ -408,16 +420,23 @@ export class GameHub {
     const rightUser = room.clients.right?.user ?? room.npcUser ?? null;
     const winner = winnerSide === "left" ? leftUser : rightUser;
     const loser = winnerSide === "left" ? rightUser : leftUser;
-    const matchId = await this.repo.createMatch({
+    const finalized = await this.repo.finalizeMatch({
+      resultKey: `room:${room.id}:finished`,
       mode: room.mode,
       winnerId: winner?.id ?? null,
       loserId: loser?.id ?? null,
       scoreLeft: room.snapshot.state.leftScore,
-      scoreRight: room.snapshot.state.rightScore
+      scoreRight: room.snapshot.state.rightScore,
+      ...(room.tournamentMatchId ? {
+        tournament: {
+          tournamentMatchId: room.tournamentMatchId,
+          roomId: room.id
+        }
+      } : {})
     });
     const result: GameFinished = {
       roomId: room.id,
-      matchId,
+      matchId: finalized.matchId,
       persisted: true,
       winnerSide,
       leftScore: room.snapshot.state.leftScore,
@@ -425,16 +444,6 @@ export class GameHub {
       ratingDelta: 16
     };
     this.broadcastRoom(room.id, { type: "game.finished", result });
-    if (room.tournamentMatchId) {
-      await this.repo.completeTournamentMatch({
-        tournamentMatchId: room.tournamentMatchId,
-        roomId: room.id,
-        matchId,
-        winnerId: winner?.id ?? null,
-        scoreLeft: room.snapshot.state.leftScore,
-        scoreRight: room.snapshot.state.rightScore
-      });
-    }
     for (const client of Object.values(room.clients)) {
       if (client) client.roomId = null;
     }


## `fix(game): 경기 결과 저장 실패를 재시도 가능한 상태로 유지`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index 832735c..3738faa 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -69,6 +69,7 @@ type Room = {
   simulation: PongSimulationState;
   aiController: PongAi | null;
   finishing: Promise<void> | null;
+  finalizationRetryTimer: NodeJS.Timeout | null;
   session: RoomSession;
   reconnectTimer: NodeJS.Timeout | null;
   disconnectedUsers: Partial<Record<PlayerSide, string>>;
@@ -81,6 +82,8 @@ const SIMULATION_TIMESTEP_MS = DEFAULT_TIMESTEP_MS;
 const SNAPSHOT_DELIVERY_DIVISOR = 2;
 const CONNECTION_REPLACED_CLOSE_CODE = 4001;
 const CONNECTION_REPLACED_REASON = "connection replaced";
+const FINALIZATION_RETRY_BASE_DELAY_MS = 250;
+const FINALIZATION_RETRY_MAX_DELAY_MS = 5_000;
 const GUEST_RESULT_RETENTION_MS = 2 * 60 * 1_000;
 const INVALID_EVENT_MESSAGE = "올바르지 않은 메시지입니다.";
 const INTERNAL_ERROR_MESSAGE = "메시지를 처리하지 못했습니다.";
@@ -392,6 +395,7 @@ export class GameHub {
   private abandonRoom(room: Room): void {
     this.roomScheduler.unregister(room.id);
     this.clearReconnectTimer(room);
+    this.clearFinalizationRetryTimer(room);
     this.releaseMatchmakingReservations(room);
     for (const client of Object.values(room.clients)) {
       if (client) client.roomId = null;
@@ -406,6 +410,11 @@ export class GameHub {
     room.reconnectTimer = null;
   }
 
+  private clearFinalizationRetryTimer(room: Room): void {
+    if (room.finalizationRetryTimer) clearTimeout(room.finalizationRetryTimer);
+    room.finalizationRetryTimer = null;
+  }
+
   private sendMatchContext(client: Client, room: Room, side: PlayerSide): void {
     const opponent = side === "left"
       ? room.clients.right?.user.displayName ?? room.npcUser?.displayName ?? "연습 AI"
@@ -654,6 +663,7 @@ export class GameHub {
     this.roomScheduler.stop();
     for (const room of this.rooms.values()) {
       this.clearReconnectTimer(room);
+      this.clearFinalizationRetryTimer(room);
       this.releaseMatchmakingReservations(room);
     }
     this.rooms.clear();
@@ -708,6 +718,7 @@ export class GameHub {
       simulation,
       aiController: options.ai ? new PongAi(roomId, npcUser?.rating ?? 1200) : null,
       finishing: null,
+      finalizationRetryTimer: null,
       session,
       reconnectTimer: null,
       disconnectedUsers: {},
@@ -908,32 +919,36 @@ export class GameHub {
       return;
     }
     let finalized: Awaited<ReturnType<MatchResultRepository["finalizeMatch"]>>;
-    try {
-      finalized = await this.repo.finalizeMatch({
-        resultKey: `room:${room.id}:finished`,
-        mode: room.mode,
-        winnerId: winner?.id ?? null,
-        loserId: loser?.id ?? null,
-        scoreLeft: room.snapshot.state.leftScore,
-        scoreRight: room.snapshot.state.rightScore,
-        ...(room.tournamentMatchId ? {
-          tournament: {
-            tournamentMatchId: room.tournamentMatchId,
-            roomId: room.id
-          }
-        } : {})
-      });
-    } catch (error) {
-      this.releaseMatchmakingReservations(room);
-      this.observer.matchFinalized?.({
-        outcome: "failure",
-        persistence: "database",
-        created: null,
-        roomId: room.id,
-        matchId: null,
-        userIds: roomUserIds(room)
-      });
-      throw error;
+    let retryAttempt = 0;
+    while (true) {
+      try {
+        finalized = await this.repo.finalizeMatch({
+          resultKey: `room:${room.id}:finished`,
+          mode: room.mode,
+          winnerId: winner?.id ?? null,
+          loserId: loser?.id ?? null,
+          scoreLeft: room.snapshot.state.leftScore,
+          scoreRight: room.snapshot.state.rightScore,
+          ...(room.tournamentMatchId ? {
+            tournament: {
+              tournamentMatchId: room.tournamentMatchId,
+              roomId: room.id
+            }
+          } : {})
+        });
+        break;
+      } catch {
+        retryAttempt += 1;
+        this.observer.matchFinalized?.({
+          outcome: "failure",
+          persistence: "database",
+          created: null,
+          roomId: room.id,
+          matchId: null,
+          userIds: roomUserIds(room)
+        });
+        if (!await this.waitForFinalizationRetry(room, retryAttempt)) return;
+      }
     }
     try {
       this.observer.matchFinalized?.({
@@ -959,6 +974,23 @@ export class GameHub {
     }
   }
 
+  private waitForFinalizationRetry(room: Room, attempt: number): Promise<boolean> {
+    if (this.rooms.get(room.id) !== room) return Promise.resolve(false);
+    this.clearFinalizationRetryTimer(room);
+    const delayMs = Math.min(
+      FINALIZATION_RETRY_BASE_DELAY_MS * 2 ** Math.min(Math.max(0, attempt - 1), 10),
+      FINALIZATION_RETRY_MAX_DELAY_MS
+    );
+    return new Promise((resolve) => {
+      const timer = setTimeout(() => {
+        if (room.finalizationRetryTimer === timer) room.finalizationRetryTimer = null;
+        resolve(this.rooms.get(room.id) === room);
+      }, delayMs);
+      timer.unref?.();
+      room.finalizationRetryTimer = timer;
+    });
+  }
+
   private rememberGuestResult(room: Room, result: GameFinished): void {
     const expiresAtMs = Date.now() + GUEST_RESULT_RETENTION_MS;
     for (const client of Object.values(room.clients)) {
@@ -990,6 +1022,7 @@ export class GameHub {
 
   private removeFinishedRoom(room: Room): void {
     this.roomScheduler.unregister(room.id);
+    this.clearFinalizationRetryTimer(room);
     this.releaseMatchmakingReservations(room);
     for (const client of Object.values(room.clients)) {
       if (client) client.roomId = null;


