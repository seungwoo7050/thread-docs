# 레이팅 기반 결정적 퐁 AI

## `feat(db): NPC 사용자 contract와 schema 추가`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index c97e4e7..268fd96 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -241,7 +241,7 @@ class PostgresRepository implements AppRepository {
 
   async listLobbyChat(): Promise<ChatMessage[]> {
     const result = await sql<ChatMessageWithSenderRow>`
-      select c.*, u.id as user_id, u.email, u.handle, u.display_name, u.avatar_key, u.role, u.status, u.rating, u.wins, u.losses
+      select c.*, u.id as user_id, u.email, u.handle, u.display_name, u.avatar_key, u.role, u.status, u.rating, u.wins, u.losses, u.is_npc
       from chat_messages c join users u on u.id = c.sender_id where c.scope = 'lobby'
       order by c.created_at desc limit 20
     `.execute(this.db);
@@ -257,7 +257,7 @@ class PostgresRepository implements AppRepository {
   }
 
   async listTournaments(): Promise<TournamentSummary[]> {
-    const result = await sql<TournamentWithCreatorRow>`select t.*, u.id as creator_id, u.email, u.handle, u.display_name, u.avatar_key, u.role, u.status as user_status, u.rating, u.wins, u.losses from tournaments t join users u on u.id = t.created_by order by t.created_at desc limit 10`.execute(this.db);
+    const result = await sql<TournamentWithCreatorRow>`select t.*, u.id as creator_id, u.email, u.handle, u.display_name, u.avatar_key, u.role, u.status as user_status, u.rating, u.wins, u.losses, u.is_npc from tournaments t join users u on u.id = t.created_by order by t.created_at desc limit 10`.execute(this.db);
     const summaries: TournamentSummary[] = [];
     for (const row of result.rows) summaries.push(await this.tournamentFromRow(row));
     return summaries;
@@ -415,7 +415,8 @@ class MemoryRepository implements AppRepository {
       status: "active",
       rating: handle === "admin" ? 1680 : 1200,
       wins: 0,
-      losses: 0
+      losses: 0,
+      is_npc: false
     };
     user.display_name = input.displayName || user.display_name;
     this.users.set(user.id, user);
diff --git a/packages/db/src/migrations.ts b/packages/db/src/migrations.ts
index 4324fad..72cf363 100644
--- a/packages/db/src/migrations.ts
+++ b/packages/db/src/migrations.ts
@@ -12,10 +12,13 @@ create table if not exists users (
   rating integer not null default 1200,
   wins integer not null default 0,
   losses integer not null default 0,
+  is_npc boolean not null default false,
   created_at timestamptz not null default now(),
   banned_at timestamptz
 );
 
+alter table users add column if not exists is_npc boolean not null default false;
+
 create table if not exists sessions (
   token text primary key,
   user_id uuid not null references users(id) on delete cascade,
diff --git a/packages/db/src/rowMappers.ts b/packages/db/src/rowMappers.ts
index d19c1b5..010ef24 100644
--- a/packages/db/src/rowMappers.ts
+++ b/packages/db/src/rowMappers.ts
@@ -12,7 +12,8 @@ export function toPublicUser(row: UserProjectionRow, online = false): PublicUser
     rating: Number(row.rating),
     wins: Number(row.wins),
     losses: Number(row.losses),
-    online
+    online,
+    isNpc: Boolean(row.is_npc)
   };
 }
 
@@ -53,7 +54,8 @@ export function toChatMessage(row: ChatMessageWithSenderRow): ChatMessage {
       status: row.status,
       rating: row.rating,
       wins: row.wins,
-      losses: row.losses
+      losses: row.losses,
+      is_npc: row.is_npc
     }),
     body: row.body,
     createdAt: row.created_at.toISOString()
diff --git a/packages/db/src/schema.ts b/packages/db/src/schema.ts
index 280bd70..f716d23 100644
--- a/packages/db/src/schema.ts
+++ b/packages/db/src/schema.ts
@@ -12,6 +12,7 @@ export interface UserTable {
   rating: Generated<number>;
   wins: Generated<number>;
   losses: Generated<number>;
+  is_npc: Generated<boolean>;
   created_at: Generated<Date>;
   banned_at: Date | null;
 }
@@ -137,6 +138,7 @@ export interface ChatMessageWithSenderRow extends ChatMessageRow {
   rating: number;
   wins: number;
   losses: number;
+  is_npc: boolean;
 }
 
 export type TournamentRow = Selectable<TournamentTable>;
@@ -152,6 +154,7 @@ export interface TournamentWithCreatorRow extends TournamentRow {
   rating: number;
   wins: number;
   losses: number;
+  is_npc: boolean;
 }
 export type UserProjectionRow = Pick<
   UserRow,
@@ -165,4 +168,5 @@ export type UserProjectionRow = Pick<
   | "rating"
   | "wins"
   | "losses"
+  | "is_npc"
 >;
diff --git a/packages/shared/src/http.ts b/packages/shared/src/http.ts
index 301fd7f..4cc1348 100644
--- a/packages/shared/src/http.ts
+++ b/packages/shared/src/http.ts
@@ -15,6 +15,7 @@ export interface PublicUser {
   wins: number;
   losses: number;
   online: boolean;
+  isNpc: boolean;
 }
 
 export interface SessionUser extends PublicUser {
diff --git a/packages/shared/src/ws.test.ts b/packages/shared/src/ws.test.ts
index 48eddbb..c919775 100644
--- a/packages/shared/src/ws.test.ts
+++ b/packages/shared/src/ws.test.ts
@@ -170,7 +170,8 @@ describe("encodeServerEvent", () => {
           rating: 1000,
           wins: 1,
           losses: 0,
-          online: true
+          online: true,
+          isNpc: false
         },
         body: "hello",
         createdAt: "2026-07-23T00:00:00.000Z"


## `feat(db): rating 구간별 NPC 상대 저장`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index 268fd96..b31c811 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -14,6 +14,14 @@ export interface DevLoginInput {
   email?: string | null;
 }
 
+type NpcSeed = { handle: string; displayName: string; rating: number; avatarKey: string };
+const NPC_PLAYERS: NpcSeed[] = [
+  { handle: "npc-rally-1100", displayName: "AI 랠리 1100", rating: 1100, avatarKey: "green" },
+  { handle: "npc-block-1200", displayName: "AI 블록 1200", rating: 1200, avatarKey: "blue" },
+  { handle: "npc-spin-1300", displayName: "AI 스핀 1300", rating: 1300, avatarKey: "amber" },
+  { handle: "npc-smash-1400", displayName: "AI 스매시 1400", rating: 1400, avatarKey: "rose" }
+];
+
 export interface CreateMatchInput {
   mode: MatchMode;
   winnerId: string | null;
@@ -48,6 +56,7 @@ export interface AppRepository {
   getUserByHandle(handle: string): Promise<PublicUser | null>;
   updateProfile(userId: string, input: { displayName?: string; avatarKey?: string }): Promise<SessionUser>;
   listOnlineUsers(): Promise<PublicUser[]>;
+  listNpcOpponents(): Promise<PublicUser[]>;
   listLeaderboard(): Promise<LeaderboardEntry[]>;
   listRecentMatches(userId?: string): Promise<MatchSummary[]>;
   getDashboard(userId: string): Promise<DashboardSummary>;
@@ -101,6 +110,7 @@ class PostgresRepository implements AppRepository {
     for (const player of players) {
       await this.upsertDevUser(player);
     }
+    for (const npc of NPC_PLAYERS) await this.upsertNpc(npc);
     await sql`update users set role = 'admin', rating = 1680 where handle = 'admin'`.execute(this.db);
     await sql`update users set rating = 1723, wins = 32, losses = 11 where handle = 'spin-doctor'`.execute(this.db);
     await sql`update users set rating = 1640, wins = 24, losses = 13 where handle = 'paddle-pro'`.execute(this.db);
@@ -113,16 +123,26 @@ class PostgresRepository implements AppRepository {
     const email = input.email ?? `${handle}@dev.pong-pong.local`;
     const displayName = input.displayName.trim() || handle;
     const result = await sql<UserRow>`
-      insert into users (email, handle, display_name, avatar_key, role)
-      values (${email}, ${handle}, ${displayName}, ${avatarFor(handle)}, ${handle === "admin" ? "admin" : "user"})
+      insert into users (email, handle, display_name, avatar_key, role, is_npc)
+      values (${email}, ${handle}, ${displayName}, ${avatarFor(handle)}, ${handle === "admin" ? "admin" : "user"}, false)
       on conflict (handle) do update set
         email = excluded.email,
-        display_name = excluded.display_name
+        display_name = excluded.display_name,
+        is_npc = false
       returning *
     `.execute(this.db);
     return toSessionUser(firstRow(result));
   }
 
+  private async upsertNpc(input: NpcSeed): Promise<void> {
+    await sql`
+      insert into users (email, handle, display_name, avatar_key, role, status, rating, wins, losses, is_npc)
+      values (null, ${input.handle}, ${input.displayName}, ${input.avatarKey}, 'user', 'active', ${input.rating}, 0, 0, true)
+      on conflict (handle) do update set display_name = excluded.display_name, avatar_key = excluded.avatar_key,
+        status = 'active', rating = excluded.rating, is_npc = true
+    `.execute(this.db);
+  }
+
   async createSession(userId: string): Promise<string> {
     const token = randomUUID();
     await sql`
@@ -173,6 +193,11 @@ class PostgresRepository implements AppRepository {
     return result.rows.map((row) => toPublicUser(row, true));
   }
 
+  async listNpcOpponents(): Promise<PublicUser[]> {
+    const result = await sql<UserRow>`select * from users where status = 'active' and is_npc = true order by rating asc`.execute(this.db);
+    return result.rows.map((row) => toPublicUser(row, false));
+  }
+
   async listLeaderboard(): Promise<LeaderboardEntry[]> {
     const result = await sql<UserRow>`select * from users order by rating desc, wins desc limit 20`.execute(this.db);
     return result.rows.map((row, index) => ({
@@ -400,6 +425,16 @@ class MemoryRepository implements AppRepository {
     ]) {
       await this.upsertDevUser(player);
     }
+    for (const npc of NPC_PLAYERS) {
+      const existing = [...this.users.values()].find((user) => user.handle === npc.handle);
+      const user: MemoryUserRow = existing ?? { id: randomUUID(), email: null, handle: npc.handle, display_name: npc.displayName, avatar_key: npc.avatarKey, role: "user", status: "active", rating: npc.rating, wins: 0, losses: 0, is_npc: true };
+      user.display_name = npc.displayName;
+      user.avatar_key = npc.avatarKey;
+      user.rating = npc.rating;
+      user.status = "active";
+      user.is_npc = true;
+      this.users.set(user.id, user);
+    }
   }
 
   async upsertDevUser(input: DevLoginInput): Promise<SessionUser> {
@@ -419,6 +454,7 @@ class MemoryRepository implements AppRepository {
       is_npc: false
     };
     user.display_name = input.displayName || user.display_name;
+    user.is_npc = false;
     this.users.set(user.id, user);
     return toSessionUser(user, true);
   }
@@ -457,6 +493,10 @@ class MemoryRepository implements AppRepository {
     return [...this.users.values()].sort((a, b) => b.rating - a.rating).map((user) => toPublicUser(user, true));
   }
 
+  async listNpcOpponents(): Promise<PublicUser[]> {
+    return [...this.users.values()].filter((user) => user.is_npc && user.status === "active").sort((a, b) => a.rating - b.rating).map((user) => toPublicUser(user, false));
+  }
+
   async listLeaderboard(): Promise<LeaderboardEntry[]> {
     return [...this.users.values()]
       .sort((a, b) => b.rating - a.rating || b.wins - a.wins)


## `feat(game): NPC 상대를 경기 방에 연결`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index 72c1abf..1e98762 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -46,6 +46,8 @@ type Room = {
 const INITIAL_BALL_VELOCITY = { x: 10, y: 5 };
 const BALL_ACCELERATION_PER_TICK = 0.015;
 const MAX_BALL_SPEED = 18;
+const NPC_QUEUE_FALLBACK_MS = 6000;
+
 
 export class GameHub {
   private readonly clients = new Map<string, Client>();
@@ -70,7 +72,7 @@ export class GameHub {
   private async receive(client: Client, payload: string): Promise<void> {
     try {
       const event = parseClientEvent(payload);
-      if (event.type === "queue.join") this.joinQueue(client, event.mode);
+      if (event.type === "queue.join") await this.joinQueue(client, event.mode);
       if (event.type === "queue.leave") this.leaveQueue(client);
       if (event.type === "tournament.join") await this.joinTournamentMatch(client, event.matchId);
       if (event.type === "game.ready") this.markReady(client, event.roomId);
@@ -108,7 +110,7 @@ export class GameHub {
     this.broadcastPresence();
   }
 
-  private joinQueue(client: Client, mode: "queue" | "ai"): void {
+  private async joinQueue(client: Client, mode: "queue" | "ai"): Promise<void> {
     this.leaveQueue(client);
     this.pruneQueue();
     if (mode === "ai") {
@@ -126,6 +128,20 @@ export class GameHub {
     this.createRoom(opponent.client, client, { ai: false, mode: "queue" });
   }
 
+  private async findClosestNpc(client: Client): Promise<PublicUser | null> {
+    const npcs = await this.repo.listNpcOpponents();
+    let closest: PublicUser | null = null;
+    let closestDistance = Number.POSITIVE_INFINITY;
+    for (const npc of npcs) {
+      const distance = Math.abs(npc.rating - client.user.rating);
+      if (distance < closestDistance) {
+        closest = npc;
+        closestDistance = distance;
+      }
+    }
+    return closest;
+  }
+
   private async joinTournamentMatch(client: Client, matchId: string): Promise<void> {
     this.leaveQueue(client);
     this.leaveTournamentWaiters(client);
@@ -223,8 +239,9 @@ export class GameHub {
     }
   }
 
-  private createRoom(left: Client, right: Client | null, options: { ai: boolean; mode: MatchMode; tournamentMatchId?: string | null }): string {
+  private createRoom(left: Client, right: Client | null, options: { ai: boolean; mode: MatchMode; tournamentMatchId?: string | null; npc?: PublicUser | null }): string {
     const roomId = randomUUID();
+    const rightPlayer = right?.user ?? options.npc ?? null;
     const room: Room = {
       id: roomId,
       clients: { left, ...(right ? { right } : {}) },
@@ -250,9 +267,9 @@ export class GameHub {
         players: [
           { id: left.user.id, handle: left.user.handle, displayName: left.user.displayName, side: "left", ready: false, ai: false },
           {
-            id: right?.user.id ?? "ai-opponent",
-            handle: right?.user.handle ?? "ai",
-            displayName: right?.user.displayName ?? "연습 AI",
+            id: rightPlayer?.id ?? "ai-opponent",
+            handle: rightPlayer?.handle ?? "ai",
+            displayName: rightPlayer?.displayName ?? "연습 AI",
             side: "right",
             ready: options.ai,
             ai: options.ai
@@ -264,7 +281,7 @@ export class GameHub {
     this.rooms.set(roomId, room);
     left.roomId = roomId;
     if (right) right.roomId = roomId;
-    this.send(left, { type: "queue.matched", roomId, side: "left", opponent: right?.user.displayName ?? "연습 AI" });
+    this.send(left, { type: "queue.matched", roomId, side: "left", opponent: rightPlayer?.displayName ?? "연습 AI" });
     if (right) this.send(right, { type: "queue.matched", roomId, side: "right", opponent: left.user.displayName });
     this.broadcastRoom(roomId, { type: "game.snapshot", snapshot: room.snapshot });
     this.broadcastPresence();
@@ -358,12 +375,14 @@ export class GameHub {
     if (room.timer) clearInterval(room.timer);
     room.timer = null;
     room.snapshot.phase = "finished";
-    const winner = winnerSide === "left" ? room.clients.left : room.clients.right;
-    const loser = winnerSide === "left" ? room.clients.right : room.clients.left;
+    const leftUser = room.clients.left?.user ?? null;
+    const rightUser = room.clients.right?.user ?? room.snapshot.players.find((player) => player.side === "right" && player.ai) ?? null;
+    const winner = winnerSide === "left" ? leftUser : rightUser;
+    const loser = winnerSide === "left" ? rightUser : leftUser;
     const matchId = await this.repo.createMatch({
       mode: room.mode,
-      winnerId: winner?.user.id ?? null,
-      loserId: loser?.user.id ?? null,
+      winnerId: winner?.id ?? null,
+      loserId: loser?.id ?? null,
       scoreLeft: room.snapshot.leftScore,
       scoreRight: room.snapshot.rightScore
     });
@@ -381,7 +400,7 @@ export class GameHub {
         tournamentMatchId: room.tournamentMatchId,
         roomId: room.id,
         matchId,
-        winnerId: winner?.user.id ?? null,
+        winnerId: winner?.id ?? null,
         scoreLeft: room.snapshot.leftScore,
         scoreRight: room.snapshot.rightScore
       });


## `feat(game): rating 기반 NPC AI policy 구현`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index 28125cb..965e933 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -43,6 +43,7 @@ type Room = {
   mode: MatchMode;
   tournamentMatchId: string | null;
   npcUser: PublicUser | null;
+  aiTargetY: number;
 };
 
 const INITIAL_BALL_VELOCITY = { x: 10, y: 5 };
@@ -50,6 +51,13 @@ const BALL_ACCELERATION_PER_TICK = 0.015;
 const MAX_BALL_SPEED = 18;
 const NPC_QUEUE_FALLBACK_MS = 6000;
 
+type AiProfile = {
+  reactionTicks: number;
+  predictionNoise: number;
+  mistakeChance: number;
+  paddleSpeedMultiplier: number;
+  deadZone: number;
+};
 
 export class GameHub {
   private readonly clients = new Map<string, Client>();
@@ -276,6 +284,7 @@ export class GameHub {
       mode: options.mode,
       tournamentMatchId: options.tournamentMatchId ?? null,
       npcUser,
+      aiTargetY: GAME_HEIGHT / 2,
       snapshot: {
         roomId,
         phase: "waiting",
@@ -368,10 +377,10 @@ export class GameHub {
     const speed = 13;
     state.paddles.left.y = clamp(state.paddles.left.y + state.paddles.left.dy * speed, 16, GAME_HEIGHT - PADDLE_HEIGHT - 16);
     if (room.ai) {
-      const center = state.paddles.right.y + PADDLE_HEIGHT / 2;
-      state.paddles.right.dy = state.ball.position.y > center + 14 ? 1 : state.ball.position.y < center - 14 ? -1 : 0;
+      updateAiPaddleIntent(room);
     }
-    state.paddles.right.y = clamp(state.paddles.right.y + state.paddles.right.dy * speed, 16, GAME_HEIGHT - PADDLE_HEIGHT - 16);
+    const rightSpeed = room.ai ? speed * aiProfileFor(room.npcUser?.rating ?? 1200).paddleSpeedMultiplier : speed;
+    state.paddles.right.y = clamp(state.paddles.right.y + state.paddles.right.dy * rightSpeed, 16, GAME_HEIGHT - PADDLE_HEIGHT - 16);
 
     state.ball.position.x += state.ball.velocity.x;
     state.ball.position.y += state.ball.velocity.y;
@@ -482,6 +491,64 @@ function clamp(value: number, min: number, max: number): number {
   return Math.max(min, Math.min(max, value));
 }
 
+function updateAiPaddleIntent(room: Room): void {
+  const state = room.snapshot;
+  const profile = aiProfileFor(room.npcUser?.rating ?? 1200);
+  if (state.tick % profile.reactionTicks === 0) {
+    const targetBase = state.ball.velocity.x > 0 ? predictedBallY(state) : GAME_HEIGHT / 2;
+    const noise = signedDeterministic(room.id, state.tick, 1) * profile.predictionNoise;
+    const mistake = deterministicUnit(room.id, state.tick, 2) < profile.mistakeChance;
+    const mistakeOffset = mistake ? signedDeterministic(room.id, state.tick, 3) * 110 : 0;
+    room.aiTargetY = clamp(targetBase + noise + mistakeOffset, 16 + PADDLE_HEIGHT / 2, GAME_HEIGHT - 16 - PADDLE_HEIGHT / 2);
+  }
+  const center = state.paddles.right.y + PADDLE_HEIGHT / 2;
+  state.paddles.right.dy = room.aiTargetY > center + profile.deadZone ? 1 : room.aiTargetY < center - profile.deadZone ? -1 : 0;
+}
+
+function aiProfileFor(rating: number): AiProfile {
+  if (rating >= 1400) {
+    return { reactionTicks: 3, predictionNoise: 20, mistakeChance: 0.04, paddleSpeedMultiplier: 1.05, deadZone: 10 };
+  }
+  if (rating >= 1300) {
+    return { reactionTicks: 4, predictionNoise: 34, mistakeChance: 0.08, paddleSpeedMultiplier: 0.96, deadZone: 14 };
+  }
+  if (rating >= 1200) {
+    return { reactionTicks: 6, predictionNoise: 54, mistakeChance: 0.12, paddleSpeedMultiplier: 0.86, deadZone: 18 };
+  }
+  return { reactionTicks: 8, predictionNoise: 78, mistakeChance: 0.18, paddleSpeedMultiplier: 0.74, deadZone: 24 };
+}
+
+function predictedBallY(state: GameSnapshot): number {
+  if (state.ball.velocity.x <= 0) return state.ball.position.y;
+  const distance = GAME_WIDTH - 32 - state.ball.position.x;
+  const ticks = distance / Math.max(1, state.ball.velocity.x);
+  let y = state.ball.position.y + state.ball.velocity.y * ticks;
+  const min = BALL_RADIUS;
+  const max = GAME_HEIGHT - BALL_RADIUS;
+  while (y < min || y > max) {
+    if (y < min) y = min + (min - y);
+    if (y > max) y = max - (y - max);
+  }
+  return y;
+}
+
+function deterministicUnit(seed: string, tick: number, salt: number): number {
+  const value = Math.sin(hashString(seed) * 0.001 + tick * 12.9898 + salt * 78.233) * 43758.5453;
+  return value - Math.floor(value);
+}
+
+function signedDeterministic(seed: string, tick: number, salt: number): number {
+  return deterministicUnit(seed, tick, salt) * 2 - 1;
+}
+
+function hashString(value: string): number {
+  let hash = 0;
+  for (let index = 0; index < value.length; index += 1) {
+    hash = (hash * 31 + value.charCodeAt(index)) >>> 0;
+  }
+  return hash;
+}
+
 function resetBall(state: GameSnapshot, xDirection: 1 | -1): void {
   state.ball.position = { x: GAME_WIDTH / 2, y: GAME_HEIGHT / 2 };
   const elapsedBoost = Math.min(1.35, 1 + state.tick / (TICK_RATE * 90));


