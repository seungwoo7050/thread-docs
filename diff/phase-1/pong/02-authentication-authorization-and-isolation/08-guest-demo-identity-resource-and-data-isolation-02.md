## `feat(guest): 등록 사용자 전용 route 접근 정책 적용`

diff --git a/apps/api/src/app.ts b/apps/api/src/app.ts
index 09e1255..a65c4c5 100644
--- a/apps/api/src/app.ts
+++ b/apps/api/src/app.ts
@@ -297,14 +297,16 @@ export function buildApp({
   });
 
   app.get("/profile/me", async (request) => {
-    const user = await currentUser(repo, request);
+    const user = await getCurrentUser(request);
     if (!user) unauthorized();
+    requireRegistered(user);
     return parseOutput(http.ownProfileResponseSchema, { profile: user });
   });
 
   app.patch("/profile/me", async (request) => {
-    const user = await currentUser(repo, request);
+    const user = await getCurrentUser(request);
     if (!user) unauthorized();
+    requireRegistered(user);
     const body = parseInput(http.profileUpdateBodySchema, request.body);
     return parseOutput(http.ownProfileResponseSchema, {
       profile: await repo.updateProfile(user.id, body)
@@ -312,14 +314,16 @@ export function buildApp({
   });
 
   app.get("/friends", async (request) => {
-    const user = await currentUser(repo, request);
+    const user = await getCurrentUser(request);
     if (!user) unauthorized();
+    requireRegistered(user);
     return parseOutput(http.friendsResponseSchema, { friends: await repo.listFriends(user.id) });
   });
 
   const requestFriend = async (request: FastifyRequest) => {
-    const user = await currentUser(repo, request);
+    const user = await getCurrentUser(request);
     if (!user) unauthorized();
+    requireRegistered(user);
     if (!isActive(user)) suspended();
     const body = parseInput(http.friendRequestBodySchema, request.body);
     return parseOutput(http.friendResponseSchema, {
@@ -331,8 +335,9 @@ export function buildApp({
   app.post("/friends", requestFriend);
 
   app.post("/friends/:id/accept", async (request) => {
-    const user = await currentUser(repo, request);
+    const user = await getCurrentUser(request);
     if (!user) unauthorized();
+    requireRegistered(user);
     const { id } = parseInput(http.idParamsSchema, request.params);
     return parseOutput(http.friendResponseSchema, { friend: await repo.acceptFriend(user.id, id) });
   });
@@ -343,8 +348,9 @@ export function buildApp({
   });
 
   app.post("/tournaments", async (request) => {
-    const user = await currentUser(repo, request);
+    const user = await getCurrentUser(request);
     if (!user) unauthorized();
+    requireRegistered(user);
     if (!isActive(user)) suspended();
     const body = parseInput(http.tournamentCreateBodySchema, request.body);
     return parseOutput(http.tournamentResponseSchema, {
@@ -353,40 +359,43 @@ export function buildApp({
   });
 
   app.post("/tournaments/:id/join", async (request) => {
-    const user = await currentUser(repo, request);
+    const user = await getCurrentUser(request);
     if (!user) unauthorized();
+    requireRegistered(user);
     if (!isActive(user)) suspended();
     const { id } = parseInput(http.idParamsSchema, request.params);
     return parseOutput(http.tournamentResponseSchema, { tournament: await repo.joinTournament(id, user.id) });
   });
 
-  app.get("/admin/users", async (request) => {
-    const user = await requireAdmin(repo, request);
-    return parseOutput(http.adminUsersResponseSchema, { users: await repo.listAdminUsers() });
-  });
+  if (appMode !== "demo") {
+    app.get("/admin/users", async (request) => {
+      const user = await requireAdmin(repo, request);
+      return parseOutput(http.adminUsersResponseSchema, { users: await repo.listAdminUsers() });
+    });
 
-  app.get("/admin/actions", async (request) => {
-    await requireAdmin(repo, request);
-    return parseOutput(http.adminActionsResponseSchema, { actions: await repo.listAdminActions() });
-  });
+    app.get("/admin/actions", async (request) => {
+      await requireAdmin(repo, request);
+      return parseOutput(http.adminActionsResponseSchema, { actions: await repo.listAdminActions() });
+    });
 
-  app.post("/admin/users/:id/ban", async (request) => {
-    const user = await requireAdmin(repo, request);
-    const { id } = parseInput(http.idParamsSchema, request.params);
-    const body = parseInput(http.adminBanBodySchema, request.body ?? {});
-    return parseOutput(http.publicUserResponseSchema, {
-      user: await repo.setUserBan(user.id, id, body.banned ?? true, body.reason ?? "manual review")
+    app.post("/admin/users/:id/ban", async (request) => {
+      const user = await requireAdmin(repo, request);
+      const { id } = parseInput(http.idParamsSchema, request.params);
+      const body = parseInput(http.adminBanBodySchema, request.body ?? {});
+      return parseOutput(http.publicUserResponseSchema, {
+        user: await repo.setUserBan(user.id, id, body.banned ?? true, body.reason ?? "manual review")
+      });
     });
-  });
 
-  app.patch("/admin/users/:id/status", async (request) => {
-    const user = await requireAdmin(repo, request);
-    const { id } = parseInput(http.idParamsSchema, request.params);
-    const body = parseInput(http.adminStatusBodySchema, request.body);
-    return parseOutput(http.publicUserResponseSchema, {
-      user: await repo.setUserBan(user.id, id, body.status === "banned", body.reason ?? "manual review")
+    app.patch("/admin/users/:id/status", async (request) => {
+      const user = await requireAdmin(repo, request);
+      const { id } = parseInput(http.idParamsSchema, request.params);
+      const body = parseInput(http.adminStatusBodySchema, request.body);
+      return parseOutput(http.publicUserResponseSchema, {
+        user: await repo.setUserBan(user.id, id, body.status === "banned", body.reason ?? "manual review")
+      });
     });
-  });
+  }
 
   return app;
 }


## `feat(game): GameHub guest identity와 기능 차단 연결`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index ff4ef3a..03edebe 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -21,11 +21,14 @@ import { HARD_BUFFERED_AMOUNT_BYTES, LatestSnapshotBuffer } from "./game/latestS
 import { PongAi } from "./game/pongAi";
 import { PongSimulation, type PongSimulationState } from "./game/pongSimulation";
 import { RoomSession } from "./game/roomSession";
+import type { GuestSessionUser } from "./guestAccess.js";
+
+type ConnectedUser = SessionUser | GuestSessionUser;
 
 type Client = {
   id: string;
   socket: WebSocket;
-  user: SessionUser;
+  user: ConnectedUser;
   roomId: string | null;
   heartbeat: ConnectionHeartbeat;
   snapshots: LatestSnapshotBuffer;
@@ -113,6 +116,14 @@ export class GameHub {
     if (this.clients.get(client.id) !== client) return;
     try {
       const event = parseClientEvent(payload);
+      if (isGuest(client.user) && (event.type === "chat.send" || event.type === "tournament.join")) {
+        this.send(client, {
+          type: "error",
+          code: "forbidden",
+          message: "게스트 계정에서는 사용할 수 없는 기능입니다."
+        });
+        return;
+      }
       if (event.type === "queue.join") await this.joinQueue(client, event.mode);
       if (event.type === "queue.leave") this.leaveQueue(client);
       if (event.type === "tournament.join") await this.joinTournamentMatch(client, event.matchId);
@@ -422,6 +433,7 @@ export class GameHub {
   onlinePlayers(): PublicUser[] {
     const users = new Map<string, PublicUser>();
     for (const client of this.clients.values()) {
+      if (isGuest(client.user)) continue;
       const { email: _email, ...user } = client.user;
       users.set(user.id, { ...user, online: true });
     }
@@ -686,6 +698,10 @@ function sideFor(room: Room, client: Client): PlayerSide | null {
   return null;
 }
 
+function isGuest(user: ConnectedUser): user is GuestSessionUser {
+  return "sessionKind" in user && user.sessionKind === "guest";
+}
+
 function clearQueueTimer(entry: QueueEntry): void {
   if (entry.npcFallbackTimer) {
     clearTimeout(entry.npcFallbackTimer);


## `feat(game): guest matchmaking과 room을 격리`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index 03edebe..34963a1 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -62,6 +62,7 @@ type Room = {
   session: RoomSession;
   reconnectTimer: NodeJS.Timeout | null;
   disconnectedUsers: Partial<Record<PlayerSide, string>>;
+  guest: boolean;
 };
 
 const NPC_QUEUE_FALLBACK_MS = 6000;
@@ -324,8 +325,9 @@ export class GameHub {
   private async matchQueuedClientWithNpc(entry: QueueEntry): Promise<void> {
     const index = this.queue.findIndex((queued) => queued.client.id === entry.client.id);
     if (index < 0 || entry.client.socket.readyState !== WebSocket.OPEN || entry.client.roomId) return;
-    const npc = await this.findClosestNpc(entry.client);
-    if (!npc) return;
+    const guest = isGuest(entry.client.user);
+    const npc = guest ? null : await this.findClosestNpc(entry.client);
+    if (!guest && !npc) return;
     const [queued] = this.queue.splice(index, 1);
     clearQueueTimer(queued);
     this.recordWaitSample(queued.queuedAt);
@@ -382,6 +384,7 @@ export class GameHub {
     let bestDistance = Number.POSITIVE_INFINITY;
     for (let index = 0; index < this.queue.length; index += 1) {
       const candidate = this.queue[index];
+      if (isGuest(candidate.client.user) !== isGuest(client.user)) continue;
       const distance = Math.abs(candidate.client.user.rating - client.user.rating);
       if (distance < bestDistance) {
         bestDistance = distance;
@@ -470,6 +473,7 @@ export class GameHub {
       session,
       reconnectTimer: null,
       disconnectedUsers: {},
+      guest: isGuest(left.user),
       snapshot: {
         roomId,
         tick: 0,


## `feat(game): guest 경기 결과 영속화 차단과 임시 보존`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index 34963a1..9bc848c 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -69,6 +69,7 @@ const NPC_QUEUE_FALLBACK_MS = 6000;
 const SIMULATION_TIMESTEP_MS = DEFAULT_TIMESTEP_MS;
 const CONNECTION_REPLACED_CLOSE_CODE = 4001;
 const CONNECTION_REPLACED_REASON = "connection replaced";
+const GUEST_RESULT_RETENTION_MS = 2 * 60 * 1_000;
 
 export class GameHub {
   private readonly clients = new Map<string, Client>();
@@ -78,10 +79,19 @@ export class GameHub {
   private readonly tournamentWaiters = new Map<string, Client[]>();
   private readonly waitSamples: number[] = [];
   private readonly inputGate = new InputGate();
+  private readonly recentGuestResults = new Map<string, {
+    result: GameFinished;
+    expiresAtMs: number;
+    cleanupTimer: NodeJS.Timeout;
+  }>();
 
   constructor(private readonly repo: AppRepository) {}
 
-  connect(socket: WebSocket, _request: IncomingMessage, user: SessionUser, pendingPayloads: string[] = []): void {
+  get retainedGuestResultCount(): number {
+    return this.recentGuestResults.size;
+  }
+
+  connect(socket: WebSocket, _request: IncomingMessage, user: ConnectedUser, pendingPayloads: string[] = []): void {
     const heartbeat = new ConnectionHeartbeat({
       ping: () => {
         if (socket.readyState === WebSocket.OPEN) socket.ping();
@@ -102,8 +112,12 @@ export class GameHub {
     socket.on("message", (payload) => this.receive(client, payload.toString()));
     socket.on("pong", () => heartbeat.acknowledge());
     socket.on("close", () => this.disconnect(client));
-    if (previous) this.replaceConnection(previous, client);
-    else this.recoverConnection(client);
+    if (previous) {
+      this.replaceConnection(previous, client);
+      if (!client.roomId) this.sendRecentGuestResult(client);
+    } else if (!this.recoverConnection(client)) {
+      this.sendRecentGuestResult(client);
+    }
     if (this.clients.get(client.id) === client && socket.readyState === WebSocket.OPEN) {
       heartbeat.start();
     }
@@ -622,6 +636,21 @@ export class GameHub {
     const rightUser = room.clients.right?.user ?? room.npcUser ?? null;
     const winner = winnerSide === "left" ? leftUser : rightUser;
     const loser = winnerSide === "left" ? rightUser : leftUser;
+    if (room.guest) {
+      const result: GameFinished = {
+        roomId: room.id,
+        matchId: null,
+        persisted: false,
+        winnerSide,
+        leftScore: room.snapshot.state.leftScore,
+        rightScore: room.snapshot.state.rightScore,
+        ratingDelta: 0
+      };
+      this.rememberGuestResult(room, result);
+      this.broadcastRoom(room.id, { type: "game.finished", result });
+      this.removeFinishedRoom(room);
+      return;
+    }
     const finalized = await this.repo.finalizeMatch({
       resultKey: `room:${room.id}:finished`,
       mode: room.mode,
@@ -646,6 +675,39 @@ export class GameHub {
       ratingDelta: 16
     };
     this.broadcastRoom(room.id, { type: "game.finished", result });
+    this.removeFinishedRoom(room);
+  }
+
+  private rememberGuestResult(room: Room, result: GameFinished): void {
+    const expiresAtMs = Date.now() + GUEST_RESULT_RETENTION_MS;
+    for (const client of Object.values(room.clients)) {
+      if (client && isGuest(client.user)) {
+        const userId = client.user.id;
+        const previous = this.recentGuestResults.get(userId);
+        if (previous) clearTimeout(previous.cleanupTimer);
+        const cleanupTimer = setTimeout(() => {
+          const current = this.recentGuestResults.get(userId);
+          if (current?.expiresAtMs === expiresAtMs) this.recentGuestResults.delete(userId);
+        }, GUEST_RESULT_RETENTION_MS);
+        cleanupTimer.unref();
+        this.recentGuestResults.set(userId, { result, expiresAtMs, cleanupTimer });
+      }
+    }
+  }
+
+  private sendRecentGuestResult(client: Client): void {
+    if (!isGuest(client.user)) return;
+    const recent = this.recentGuestResults.get(client.user.id);
+    if (!recent) return;
+    if (Date.now() > recent.expiresAtMs) {
+      clearTimeout(recent.cleanupTimer);
+      this.recentGuestResults.delete(client.user.id);
+      return;
+    }
+    this.send(client, { type: "game.finished", result: recent.result });
+  }
+
+  private removeFinishedRoom(room: Room): void {
     for (const client of Object.values(room.clients)) {
       if (client) client.roomId = null;
     }


