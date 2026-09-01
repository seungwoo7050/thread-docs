## `feat(tournament): 준결승 대진 생성과 조회 구현`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index a34580e..e9e3c2f 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -270,8 +270,19 @@ class PostgresRepository implements AppRepository {
 
   async joinTournament(tournamentId: string, userId: string): Promise<TournamentSummary> {
     const count = await sql<{ count: string }>`select count(*)::text from tournament_entries where tournament_id = ${tournamentId}`.execute(this.db);
-    await sql`insert into tournament_entries (tournament_id, user_id, seed) values (${tournamentId}, ${userId}, ${Number(firstRow(count).count) + 1}) on conflict (tournament_id, user_id) do nothing`.execute(this.db);
-    return (await this.listTournaments()).find((item) => item.id === tournamentId)!;
+    const status = await sql<{ capacity: number; joined: boolean }>`
+      select capacity, exists(select 1 from tournament_entries where tournament_id = ${tournamentId} and user_id = ${userId}) as joined
+      from tournaments where id = ${tournamentId} limit 1
+    `.execute(this.db);
+    const tournament = firstRow(status);
+    if (!tournament.joined && Number(firstRow(count).count) >= Number(tournament.capacity)) throw new Error("tournament full");
+    const seed = Number(firstRow(count).count) + 1;
+    await sql`insert into tournament_entries (tournament_id, user_id, seed) values (${tournamentId}, ${userId}, ${seed}) on conflict (tournament_id, user_id) do nothing`.execute(this.db);
+    await sql`update tournaments set status = case when (select count(*) from tournament_entries where tournament_id = ${tournamentId}) >= capacity then 'running' else status end where id = ${tournamentId}`.execute(this.db);
+    await this.ensureTournamentBracket(tournamentId);
+    const found = (await this.listTournaments()).find((item) => item.id === tournamentId);
+    if (!found) throw new Error("tournament not found");
+    return found;
   }
 
   async getTournamentMatch(matchId: string): Promise<TournamentMatchRecord | null> {
@@ -318,7 +329,32 @@ class PostgresRepository implements AppRepository {
 
   private async tournamentFromRow(row: TournamentWithCreatorRow): Promise<TournamentSummary> {
     const entries = await sql<UserRow>`select u.* from tournament_entries e join users u on u.id = e.user_id where e.tournament_id = ${row.id} order by e.seed asc`.execute(this.db);
-    return toTournamentSummary(row, entries.rows.map((entry) => toPublicUser(entry, true)));
+    const matches = await sql<TournamentMatchRow>`
+      select * from tournament_matches where tournament_id = ${row.id}
+      order by case when round = 'semifinal' then 1 else 2 end, slot asc
+    `.execute(this.db);
+    const summaries = await Promise.all(matches.rows.map(async (match) => toTournamentMatchSummary(match, {
+      left: match.left_user_id ? await this.getUserById(match.left_user_id) : null,
+      right: match.right_user_id ? await this.getUserById(match.right_user_id) : null,
+      winner: match.winner_id ? await this.getUserById(match.winner_id) : null
+    })));
+    const summary = toTournamentSummary(row, entries.rows.map((entry) => toPublicUser(entry, true)), summaries);
+    summary.winner = row.winner_id ? await this.getUserById(row.winner_id) : null;
+    return summary;
+  }
+
+  private async ensureTournamentBracket(tournamentId: string): Promise<void> {
+    const entries = await sql<{ user_id: string; seed: number }>`
+      select user_id, seed from tournament_entries where tournament_id = ${tournamentId} order by seed asc
+    `.execute(this.db);
+    if (entries.rows.length < 4) return;
+    await sql`
+      insert into tournament_matches (tournament_id, round, slot, left_user_id, right_user_id, status)
+      values
+        (${tournamentId}, 'semifinal', 1, ${entries.rows[0].user_id}, ${entries.rows[3].user_id}, 'ready'),
+        (${tournamentId}, 'semifinal', 2, ${entries.rows[1].user_id}, ${entries.rows[2].user_id}, 'ready')
+      on conflict (tournament_id, round, slot) do nothing
+    `.execute(this.db);
   }
 
   async listAdminUsers(): Promise<PublicUser[]> {


## `feat(tournament): 토너먼트 경기 방 진행`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index ceb68ca..06e125d 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -13,6 +13,7 @@ import {
   parseClientEvent,
   type GameFinished,
   type GameSnapshot,
+  type MatchMode,
   type PlayerSide,
   type ServerEvent,
   type SessionUser
@@ -37,12 +38,15 @@ type Room = {
   ready: Partial<Record<PlayerSide, boolean>>;
   snapshot: GameSnapshot;
   timer: NodeJS.Timeout | null;
+  mode: MatchMode;
+  tournamentMatchId: string | null;
 };
 
 export class GameHub {
   private readonly clients = new Map<string, Client>();
   private readonly queue: QueueEntry[] = [];
   private readonly rooms = new Map<string, Room>();
+  private readonly tournamentWaiters = new Map<string, Client[]>();
   private readonly waitSamples: number[] = [];
 
   constructor(private readonly repo: AppRepository) {}
@@ -63,6 +67,7 @@ export class GameHub {
       const event = parseClientEvent(payload);
       if (event.type === "queue.join") this.joinQueue(client, event.mode);
       if (event.type === "queue.leave") this.leaveQueue(client);
+      if (event.type === "tournament.join") await this.joinTournamentMatch(client, event.matchId);
       if (event.type === "game.ready") this.markReady(client, event.roomId);
       if (event.type === "game.pause") this.pauseRoom(client, event.roomId);
       if (event.type === "game.resume") this.resumeRoom(client, event.roomId);
@@ -87,6 +92,7 @@ export class GameHub {
 
   private disconnect(client: Client): void {
     this.leaveQueue(client);
+    this.leaveTournamentWaiters(client);
     this.clients.delete(client.id);
     if (client.roomId) {
       const room = this.rooms.get(client.roomId);
@@ -101,7 +107,7 @@ export class GameHub {
     this.leaveQueue(client);
     this.pruneQueue();
     if (mode === "ai") {
-      this.createRoom(client, null, true);
+      this.createRoom(client, null, { ai: true, mode: "ai" });
       return;
     }
     const opponentIndex = this.findClosestQueuedOpponent(client);
@@ -112,7 +118,38 @@ export class GameHub {
     }
     const [opponent] = this.queue.splice(opponentIndex, 1);
     this.recordWaitSample(opponent.queuedAt);
-    this.createRoom(opponent.client, client, false);
+    this.createRoom(opponent.client, client, { ai: false, mode: "queue" });
+  }
+
+  private async joinTournamentMatch(client: Client, matchId: string): Promise<void> {
+    this.leaveQueue(client);
+    this.leaveTournamentWaiters(client);
+    const match = await this.repo.getTournamentMatch(matchId);
+    if (!match || match.status !== "ready") {
+      this.send(client, { type: "error", message: "참가할 수 없는 토너먼트 경기입니다." });
+      return;
+    }
+    if (match.leftUserId !== client.user.id && match.rightUserId !== client.user.id) {
+      this.send(client, { type: "error", message: "토너먼트 경기 참가자가 아닙니다." });
+      return;
+    }
+    if (client.roomId) {
+      this.send(client, { type: "error", message: "이미 진행 중인 경기가 있습니다." });
+      return;
+    }
+    const waiters = this.tournamentWaiters.get(matchId) ?? [];
+    const existing = waiters.find((waiter) => waiter.user.id === client.user.id);
+    if (existing) return;
+    const opponent = waiters.find((waiter) => waiter.user.id === match.leftUserId || waiter.user.id === match.rightUserId);
+    if (!opponent) {
+      this.tournamentWaiters.set(matchId, [...waiters, client]);
+      return;
+    }
+    this.tournamentWaiters.delete(matchId);
+    const left = client.user.id === match.leftUserId ? client : opponent;
+    const right = left === client ? opponent : client;
+    const roomId = this.createRoom(left, right, { ai: false, mode: "tournament", tournamentMatchId: matchId });
+    await this.repo.startTournamentMatch(matchId, roomId);
   }
 
   private findClosestQueuedOpponent(client: Client): number {
@@ -134,6 +171,14 @@ export class GameHub {
     if (index >= 0) this.queue.splice(index, 1);
   }
 
+  private leaveTournamentWaiters(client: Client): void {
+    for (const [matchId, waiters] of this.tournamentWaiters.entries()) {
+      const next = waiters.filter((waiter) => waiter.id !== client.id);
+      if (next.length === 0) this.tournamentWaiters.delete(matchId);
+      else this.tournamentWaiters.set(matchId, next);
+    }
+  }
+
   private pruneQueue(): void {
     for (let index = this.queue.length - 1; index >= 0; index -= 1) {
       if (this.queue[index].client.socket.readyState !== WebSocket.OPEN) {
@@ -164,14 +209,16 @@ export class GameHub {
     }
   }
 
-  private createRoom(left: Client, right: Client | null, ai: boolean): void {
+  private createRoom(left: Client, right: Client | null, options: { ai: boolean; mode: MatchMode; tournamentMatchId?: string | null }): string {
     const roomId = randomUUID();
     const room: Room = {
       id: roomId,
       clients: { left, ...(right ? { right } : {}) },
-      ai,
+      ai: options.ai,
       ready: {},
       timer: null,
+      mode: options.mode,
+      tournamentMatchId: options.tournamentMatchId ?? null,
       snapshot: {
         roomId,
         phase: "waiting",
@@ -193,8 +240,8 @@ export class GameHub {
             handle: right?.user.handle ?? "ai",
             displayName: right?.user.displayName ?? "연습 AI",
             side: "right",
-            ready: ai,
-            ai
+            ready: options.ai,
+            ai: options.ai
           }
         ],
         serverTime: new Date().toISOString()
@@ -207,6 +254,7 @@ export class GameHub {
     if (right) this.send(right, { type: "queue.matched", roomId, side: "right", opponent: left.user.displayName });
     this.broadcastRoom(roomId, { type: "game.snapshot", snapshot: room.snapshot });
     this.broadcastPresence();
+    return roomId;
   }
 
   private markReady(client: Client, roomId: string): void {
@@ -298,7 +346,7 @@ export class GameHub {
     const winner = winnerSide === "left" ? room.clients.left : room.clients.right;
     const loser = winnerSide === "left" ? room.clients.right : room.clients.left;
     const matchId = await this.repo.createMatch({
-      mode: room.ai ? "ai" : "queue",
+      mode: room.mode,
       winnerId: winner?.user.id ?? null,
       loserId: loser?.user.id ?? null,
       scoreLeft: room.snapshot.leftScore,
@@ -313,6 +361,16 @@ export class GameHub {
       ratingDelta: 16
     };
     this.broadcastRoom(room.id, { type: "game.finished", result });
+    if (room.tournamentMatchId) {
+      await this.repo.completeTournamentMatch({
+        tournamentMatchId: room.tournamentMatchId,
+        roomId: room.id,
+        matchId,
+        winnerId: winner?.user.id ?? null,
+        scoreLeft: room.snapshot.leftScore,
+        scoreRight: room.snapshot.rightScore
+      });
+    }
     for (const client of Object.values(room.clients)) {
       if (client) client.roomId = null;
     }


## `feat(tournament): 플레이 가능한 대진 UI 연결`

diff --git a/apps/web/src/app/play/page.tsx b/apps/web/src/app/play/page.tsx
index 88d7958..af71210 100644
--- a/apps/web/src/app/play/page.tsx
+++ b/apps/web/src/app/play/page.tsx
@@ -25,9 +25,26 @@ export default function PlayPage() {
   const canPause = Boolean(roomId && phase === "playing");
   const canResume = Boolean(roomId && phase === "paused");
   const opponentName = snapshot?.players.find((player) => player.side === "right")?.displayName ?? "대기 중";
+  const autoStartedRef = useRef(false);
 
   useEffect(() => () => closeCurrentSocket(), []);
 
+  useEffect(() => {
+    if (autoStartedRef.current) return;
+    const params = new URLSearchParams(window.location.search);
+    const tournamentMatchId = params.get("tournamentMatchId");
+    const mode = params.get("mode");
+    if (tournamentMatchId) {
+      autoStartedRef.current = true;
+      connectTournament(tournamentMatchId);
+      return;
+    }
+    if (mode === "ai") {
+      autoStartedRef.current = true;
+      connect("ai");
+    }
+  }, []);
+
   useEffect(() => {
     const handleKey = (event: KeyboardEvent) => {
       if (event.key === "ArrowUp" || event.key.toLowerCase() === "w") {
@@ -64,6 +81,14 @@ export default function PlayPage() {
   }, [roomId, phase]);
 
   function connect(mode: "queue" | "ai") {
+    openGameSocket(mode === "ai" ? "인공지능 연습 방 생성 중" : "매칭 큐 참가 중", { type: "queue.join", mode });
+  }
+
+  function connectTournament(matchId: string) {
+    openGameSocket("토너먼트 경기 상대 입장 대기 중", { type: "tournament.join", matchId });
+  }
+
+  function openGameSocket(openStatus: string, payload: Record<string, unknown>) {
     const token = getToken();
     if (!token) {
       setStatus("로그인 후 이용할 수 있습니다.");
@@ -78,8 +103,8 @@ export default function PlayPage() {
     const socket = new WebSocket(`${WS_URL}?session=${token}`);
     socketRef.current = socket;
     socket.onopen = () => {
-      setStatus(mode === "ai" ? "인공지능 연습 방 생성 중" : "매칭 큐 참가 중");
-      socket.send(JSON.stringify({ type: "queue.join", mode }));
+      setStatus(openStatus);
+      socket.send(JSON.stringify(payload));
     };
     socket.onmessage = (event) => {
       const message = JSON.parse(event.data) as ServerEvent;
diff --git a/apps/web/src/app/tournaments/page.tsx b/apps/web/src/app/tournaments/page.tsx
index 640736e..5bd900d 100644
--- a/apps/web/src/app/tournaments/page.tsx
+++ b/apps/web/src/app/tournaments/page.tsx
@@ -2,17 +2,19 @@
 
 import { useEffect, useState } from "react";
 import { Plus, Trophy } from "lucide-react";
-import type { TournamentSummary } from "@pong-pong/shared";
+import type { SessionUser, TournamentMatchSummary, TournamentSummary } from "@pong-pong/shared";
 import { AppShell } from "@/components/AppShell";
-import { createTournament, getTournaments, joinTournament } from "@/lib/api";
+import { createTournament, getMe, getTournaments, joinTournament } from "@/lib/api";
 
 export default function TournamentsPage() {
   const [items, setItems] = useState<TournamentSummary[]>([]);
   const [selectedId, setSelectedId] = useState("");
   const [message, setMessage] = useState("대회 목록을 불러오는 중입니다.");
+  const [me, setMe] = useState<SessionUser | null>(null);
   const selected = items.find((item) => item.id === selectedId) ?? items[0];
 
   useEffect(() => {
+    getMe().then(setMe);
     getTournaments()
       .then((tournaments) => {
         setItems(tournaments);
@@ -90,6 +92,7 @@ export default function TournamentsPage() {
                 <p className="font-black text-ink">{selected.name}</p>
                 <p className="mt-1 text-sm font-semibold text-muted">
                   {selected.playerCount} / {selected.capacity}명 · {selected.status === "open" ? "모집 중" : selected.status === "running" ? "진행 중" : "종료"}
+                  {selected.winner ? ` · 우승 ${selected.winner.displayName}` : ""}
                 </p>
               </div>
               <button className="focus-ring rounded-lg bg-green-600 px-4 py-2 text-sm font-black text-white disabled:cursor-not-allowed disabled:bg-slate-300" onClick={join} disabled={selected.playerCount >= selected.capacity}>
@@ -98,22 +101,52 @@ export default function TournamentsPage() {
             </div>
           ) : null}
           <div className="mt-6 grid gap-4 md:grid-cols-3">
-            {["1라운드", "결승", "우승"].map((round, index) => (
-              <div key={round} className="rounded-lg border border-line bg-slate-50 p-4">
-                <p className="font-black text-blue-700">{round}</p>
-                <div className="mt-4 grid gap-3">
-                  {(selected?.entries ?? []).slice(index, index + 2).map((entry) => (
-                    <div key={entry.id} className="rounded-lg bg-white px-3 py-2 text-sm font-bold text-ink shadow-sm">
-                      {entry.displayName}
-                    </div>
-                  ))}
-                  {index === 2 ? <div className="rounded-lg bg-white px-3 py-2 text-sm font-bold text-muted shadow-sm">대기 중</div> : null}
-                </div>
-              </div>
-            ))}
+            <BracketColumn title="준결승" matches={(selected?.matches ?? []).filter((match) => match.round === "semifinal")} me={me} />
+            <BracketColumn title="결승" matches={(selected?.matches ?? []).filter((match) => match.round === "final")} me={me} />
+            <div className="rounded-lg border border-line bg-slate-50 p-4">
+              <p className="font-black text-blue-700">우승</p>
+              <div className="mt-4 rounded-lg bg-white px-3 py-2 text-sm font-bold text-ink shadow-sm">{selected?.winner?.displayName ?? "대기 중"}</div>
+            </div>
           </div>
+          {selected && selected.matches.length === 0 ? <p className="mt-4 text-sm font-semibold text-muted">4명이 참가하면 실제 경기 브래킷이 생성됩니다.</p> : null}
         </div>
       </section>
     </AppShell>
   );
 }
+
+function BracketColumn({ title, matches, me }: { title: string; matches: TournamentMatchSummary[]; me: SessionUser | null }) {
+  return (
+    <div className="rounded-lg border border-line bg-slate-50 p-4">
+      <p className="font-black text-blue-700">{title}</p>
+      <div className="mt-4 grid gap-3">
+        {matches.length === 0 ? <div className="rounded-lg bg-white px-3 py-2 text-sm font-bold text-muted shadow-sm">대기 중</div> : null}
+        {matches.map((match) => {
+          const participant = Boolean(me && (match.left?.id === me.id || match.right?.id === me.id));
+          const canEnter = participant && match.status === "ready";
+          return (
+            <div key={match.id} className="rounded-lg bg-white px-3 py-2 text-sm font-bold text-ink shadow-sm">
+              <div className="flex items-center justify-between gap-2">
+                <span>{match.left?.displayName ?? "대기"}</span>
+                <span className="text-muted">vs</span>
+                <span>{match.right?.displayName ?? "대기"}</span>
+              </div>
+              <div className="mt-2 flex items-center justify-between gap-2 text-xs text-muted">
+                <span>{match.status === "finished" ? `${match.scoreLeft} - ${match.scoreRight}` : statusLabel(match.status)}</span>
+                {canEnter ? <a className="rounded-md bg-blue-600 px-2 py-1 font-black text-white" href={`/play?tournamentMatchId=${match.id}`}>경기 입장</a> : null}
+              </div>
+              {match.winner ? <p className="mt-2 text-xs font-black text-green-600">승자 {match.winner.displayName}</p> : null}
+            </div>
+          );
+        })}
+      </div>
+    </div>
+  );
+}
+
+function statusLabel(status: TournamentMatchSummary["status"]): string {
+  if (status === "ready") return "입장 가능";
+  if (status === "running") return "진행 중";
+  if (status === "finished") return "종료";
+  return "대기 중";
+}


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


