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
 


## `feat(web): profile과 friend 조회 query 추가`

diff --git a/apps/web/src/lib/api.ts b/apps/web/src/lib/api.ts
index 57ac0d0..8d127c7 100644
--- a/apps/web/src/lib/api.ts
+++ b/apps/web/src/lib/api.ts
@@ -5,9 +5,11 @@ import {
   chatResponseSchema,
   dashboardSummarySchema,
   friendResponseSchema,
+  friendsResponseSchema,
   guestAuthResponseSchema,
   leaderboardResponseSchema,
   lobbyResponseSchema,
+  ownProfileResponseSchema,
   profileResponseSchema,
   publicUserResponseSchema,
   tournamentResponseSchema,
@@ -23,6 +25,7 @@ import {
   type LeaderboardEntry,
   type LobbyResponse,
   type MatchSummary,
+  type ProfileUpdateBody,
   type PublicUser,
   type SessionUser,
   type TournamentSummary,
@@ -120,6 +123,10 @@ export async function getDashboard(signal?: AbortSignal): Promise<DashboardSumma
   return apiFetch("/dashboard", dashboardSummarySchema, { signal });
 }
 
+export async function getFriends(signal?: AbortSignal): Promise<FriendSummary[]> {
+  return (await apiFetch("/friends", friendsResponseSchema, { signal })).friends;
+}
+
 export async function getLeaderboard(signal?: AbortSignal): Promise<LeaderboardEntry[]> {
   return (await apiFetch("/leaderboard", leaderboardResponseSchema, { signal })).entries;
 }
@@ -150,6 +157,21 @@ export async function getProfile(
   return apiFetch(`/profile/${handle}`, profileResponseSchema, { signal });
 }
 
+export async function getOwnProfile(signal?: AbortSignal): Promise<SessionUser> {
+  return (await apiFetch("/profile/me", ownProfileResponseSchema, { signal })).profile;
+}
+
+export async function updateOwnProfile(
+  input: ProfileUpdateBody,
+  signal?: AbortSignal
+): Promise<SessionUser> {
+  return (await apiFetch("/profile/me", ownProfileResponseSchema, {
+    method: "PATCH",
+    body: JSON.stringify(input),
+    signal
+  })).profile;
+}
+
 export async function requestFriend(handle: string, signal?: AbortSignal): Promise<FriendSummary> {
   return (await apiFetch("/friends/request", friendResponseSchema, {
     method: "POST",
diff --git a/apps/web/src/lib/query.ts b/apps/web/src/lib/query.ts
index 4ed39cf..8833c68 100644
--- a/apps/web/src/lib/query.ts
+++ b/apps/web/src/lib/query.ts
@@ -8,9 +8,11 @@ import {
   getAdminActions,
   getAdminUsers,
   getDashboard,
+  getFriends,
   getLeaderboard,
   getLobby,
   getMe,
+  getOwnProfile,
   getProfile,
   getTournaments
 } from "./api";
@@ -19,6 +21,7 @@ export const queryKeys = {
   me: () => ["user", "me"] as const,
   lobby: () => ["lobby"] as const,
   dashboard: () => ["dashboard"] as const,
+  ownProfile: () => ["user", "profile"] as const,
   profile: (handle: string) => ["profiles", handle] as const,
   leaderboard: () => ["leaderboard"] as const,
   friends: () => ["friends"] as const,
@@ -31,6 +34,18 @@ export const mutationInvalidations = {
   login: () => [queryKeys.me(), queryKeys.lobby()] as const,
   lobbyChat: () => [queryKeys.lobby()] as const,
   friendRequest: () => [queryKeys.friends()] as const,
+  profileUpdate: (handle: string) => [
+    queryKeys.me(),
+    queryKeys.ownProfile(),
+    queryKeys.profile(handle),
+    queryKeys.lobby(),
+    queryKeys.dashboard(),
+    queryKeys.friends(),
+    queryKeys.leaderboard(),
+    queryKeys.tournaments(),
+    queryKeys.adminUsers(),
+    queryKeys.adminActions()
+  ] as const,
   tournamentChange: () => [queryKeys.tournaments()] as const,
   adminStatus: () => [queryKeys.adminUsers(), queryKeys.adminActions()] as const
 };
@@ -53,12 +68,24 @@ export const dashboardQueryOptions = () => queryOptions({
   staleTime: 10_000
 });
 
+export const ownProfileQueryOptions = () => queryOptions({
+  queryKey: queryKeys.ownProfile(),
+  queryFn: ({ signal }) => getOwnProfile(signal),
+  staleTime: 30_000
+});
+
 export const profileQueryOptions = (handle: string) => queryOptions({
   queryKey: queryKeys.profile(handle),
   queryFn: ({ signal }) => getProfile(handle, signal),
   staleTime: 30_000
 });
 
+export const friendsQueryOptions = () => queryOptions({
+  queryKey: queryKeys.friends(),
+  queryFn: ({ signal }) => getFriends(signal),
+  staleTime: 10_000
+});
+
 export const leaderboardQueryOptions = () => queryOptions({
   queryKey: queryKeys.leaderboard(),
   queryFn: ({ signal }) => getLeaderboard(signal),
@@ -99,6 +126,7 @@ export function expireSession(client: QueryClient): void {
   const sessionScopedKeys = [
     queryKeys.lobby(),
     queryKeys.dashboard(),
+    queryKeys.ownProfile(),
     queryKeys.friends(),
     queryKeys.adminUsers(),
     queryKeys.adminActions()


## `test(web): profile과 friend 조회 규칙 검증`

diff --git a/apps/web/src/lib/api.test.ts b/apps/web/src/lib/api.test.ts
index 2ee83d1..0593a86 100644
--- a/apps/web/src/lib/api.test.ts
+++ b/apps/web/src/lib/api.test.ts
@@ -1,5 +1,10 @@
 import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
-import { okResponseSchema, type PublicUser, type SessionUser } from "@pong-pong/shared";
+import {
+  okResponseSchema,
+  type FriendSummary,
+  type PublicUser,
+  type SessionUser
+} from "@pong-pong/shared";
 import {
   ApiError,
   SESSION_EXPIRED_EVENT,
@@ -9,9 +14,11 @@ import {
   getAdminActions,
   getAdminUsers,
   getDashboard,
+  getFriends,
   getLeaderboard,
   getLobby,
   getMe,
+  getOwnProfile,
   getProfile,
   getTournaments,
   guestLogin,
@@ -19,7 +26,8 @@ import {
   requestFriend,
   requestWsTicket,
   sendLobbyChat,
-  setUserStatus
+  setUserStatus,
+  updateOwnProfile
 } from "./api";
 
 const USER_ID = "11111111-1111-4111-8111-111111111111";
@@ -44,6 +52,12 @@ const sessionUser = {
   email: "tester@example.com"
 } satisfies SessionUser;
 
+const friend = {
+  id: ITEM_ID,
+  user: publicUser,
+  status: "accepted"
+} satisfies FriendSummary;
+
 function jsonResponse(body: unknown, init: ResponseInit = {}): Response {
   return new Response(JSON.stringify(body), {
     status: 200,
@@ -233,6 +247,47 @@ describe("API endpoint helpers", () => {
     await expect(getMe()).rejects.toMatchObject({ status: 503, code: "UNAVAILABLE" });
   });
 
+  it("reads the current profile and friend list through their shared response contracts", async () => {
+    const controller = new AbortController();
+    fetchMock
+      .mockResolvedValueOnce(jsonResponse({ profile: sessionUser }))
+      .mockResolvedValueOnce(jsonResponse({ friends: [friend] }));
+
+    await expect(getOwnProfile(controller.signal)).resolves.toEqual(sessionUser);
+    await expect(getFriends(controller.signal)).resolves.toEqual([friend]);
+
+    expect(fetchMock).toHaveBeenNthCalledWith(
+      1,
+      "http://localhost:4000/profile/me",
+      expect.objectContaining({ credentials: "include", signal: controller.signal })
+    );
+    expect(fetchMock).toHaveBeenNthCalledWith(
+      2,
+      "http://localhost:4000/friends",
+      expect.objectContaining({ credentials: "include", signal: controller.signal })
+    );
+  });
+
+  it("updates only the supplied current-profile fields", async () => {
+    const updated = { ...sessionUser, displayName: "새 이름", avatarKey: "avatar-2" };
+    const controller = new AbortController();
+    fetchMock.mockResolvedValue(jsonResponse({ profile: updated }));
+
+    await expect(updateOwnProfile({
+      displayName: "새 이름",
+      avatarKey: "avatar-2"
+    }, controller.signal)).resolves.toEqual(updated);
+
+    const [url, init] = fetchMock.mock.calls[0] as [string, RequestInit];
+    expect(url).toBe("http://localhost:4000/profile/me");
+    expect(init).toMatchObject({
+      method: "PATCH",
+      credentials: "include",
+      body: JSON.stringify({ displayName: "새 이름", avatarKey: "avatar-2" }),
+      signal: controller.signal
+    });
+  });
+
   it("requests and validates a one-time websocket ticket", async () => {
     const ticketResponse = {
       ticket: "a".repeat(43),
@@ -287,10 +342,13 @@ describe("API endpoint helpers", () => {
     { name: "getLobby", call: () => getLobby() },
     { name: "sendLobbyChat", call: () => sendLobbyChat("안녕하세요") },
     { name: "getDashboard", call: () => getDashboard() },
+    { name: "getFriends", call: () => getFriends() },
     { name: "getLeaderboard", call: () => getLeaderboard() },
     { name: "getTournaments", call: () => getTournaments() },
     { name: "createTournament", call: () => createTournament("주간 컵") },
     { name: "joinTournament", call: () => joinTournament(ITEM_ID) },
+    { name: "getOwnProfile", call: () => getOwnProfile() },
+    { name: "updateOwnProfile", call: () => updateOwnProfile({ displayName: "새 이름" }) },
     { name: "getProfile", call: () => getProfile("tester") },
     { name: "requestFriend", call: () => requestFriend("friend") },
     { name: "getAdminUsers", call: () => getAdminUsers() },
diff --git a/apps/web/src/lib/query.test.ts b/apps/web/src/lib/query.test.ts
index 376e775..3db99ff 100644
--- a/apps/web/src/lib/query.test.ts
+++ b/apps/web/src/lib/query.test.ts
@@ -3,8 +3,10 @@ import { describe, expect, it } from "vitest";
 import { ApiError } from "./api";
 import {
   expireSession,
+  friendsQueryOptions,
   invalidateExactQueries,
   mutationInvalidations,
+  ownProfileQueryOptions,
   queryKeys,
   shouldRetryQuery
 } from "./query";
@@ -14,6 +16,7 @@ describe("query key contract", () => {
     expect(queryKeys.me()).toEqual(["user", "me"]);
     expect(queryKeys.lobby()).toEqual(["lobby"]);
     expect(queryKeys.dashboard()).toEqual(["dashboard"]);
+    expect(queryKeys.ownProfile()).toEqual(["user", "profile"]);
     expect(queryKeys.profile("pong-master")).toEqual(["profiles", "pong-master"]);
     expect(queryKeys.leaderboard()).toEqual(["leaderboard"]);
     expect(queryKeys.friends()).toEqual(["friends"]);
@@ -29,6 +32,18 @@ describe("query key contract", () => {
     ]);
     expect(mutationInvalidations.lobbyChat()).toEqual([queryKeys.lobby()]);
     expect(mutationInvalidations.friendRequest()).toEqual([queryKeys.friends()]);
+    expect(mutationInvalidations.profileUpdate("pong-master")).toEqual([
+      queryKeys.me(),
+      queryKeys.ownProfile(),
+      queryKeys.profile("pong-master"),
+      queryKeys.lobby(),
+      queryKeys.dashboard(),
+      queryKeys.friends(),
+      queryKeys.leaderboard(),
+      queryKeys.tournaments(),
+      queryKeys.adminUsers(),
+      queryKeys.adminActions()
+    ]);
     expect(mutationInvalidations.tournamentChange()).toEqual([
       queryKeys.tournaments()
     ]);
@@ -38,6 +53,11 @@ describe("query key contract", () => {
     ]);
   });
 
+  it("connects private profile and friend reads to their scoped cache keys", () => {
+    expect(ownProfileQueryOptions().queryKey).toEqual(queryKeys.ownProfile());
+    expect(friendsQueryOptions().queryKey).toEqual(queryKeys.friends());
+  });
+
   it("marks exact mutation keys stale without touching adjacent caches", async () => {
     const client = new QueryClient();
     client.setQueryData(queryKeys.adminUsers(), [{ id: "user-1" }]);
@@ -66,6 +86,7 @@ describe("session expiration", () => {
     client.setQueryData(queryKeys.me(), { id: "user-1" });
     client.setQueryData(queryKeys.lobby(), { me: { id: "user-1" } });
     client.setQueryData(queryKeys.dashboard(), { wins: 2 });
+    client.setQueryData(queryKeys.ownProfile(), { id: "user-1" });
     client.setQueryData(queryKeys.friends(), [{ id: "friend-1" }]);
     client.setQueryData(queryKeys.adminUsers(), [{ id: "user-1" }]);
     client.setQueryData(queryKeys.adminActions(), [{ id: "action-1" }]);
@@ -78,6 +99,7 @@ describe("session expiration", () => {
     expect(client.getQueryData(queryKeys.me())).toBeNull();
     expect(client.getQueryData(queryKeys.lobby())).toBeUndefined();
     expect(client.getQueryData(queryKeys.dashboard())).toBeUndefined();
+    expect(client.getQueryData(queryKeys.ownProfile())).toBeUndefined();
     expect(client.getQueryData(queryKeys.friends())).toBeUndefined();
     expect(client.getQueryData(queryKeys.adminUsers())).toBeUndefined();
     expect(client.getQueryData(queryKeys.adminActions())).toBeUndefined();
