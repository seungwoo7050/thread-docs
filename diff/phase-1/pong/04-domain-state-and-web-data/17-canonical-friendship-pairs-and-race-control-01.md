# 친구 관계의 정규 쌍과 경쟁 상태 제어

## `feat(db): 친구 관계 저장 구현`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index b30d6e4..7169fc2 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -1,10 +1,10 @@
 import { randomUUID } from "node:crypto";
 import { Kysely, PostgresDialect, sql } from "kysely";
 import { Pool } from "pg";
-import type { DashboardSummary, LeaderboardEntry, MatchMode, MatchSummary, PublicUser, SessionUser } from "@pong-pong/shared";
+import type { DashboardSummary, FriendSummary, LeaderboardEntry, MatchMode, MatchSummary, PublicUser, SessionUser } from "@pong-pong/shared";
 import { initialMigrationSql } from "./migrations";
-import { toMatchSummary, toPublicUser, toSessionUser } from "./rowMappers";
-import type { Database, MatchWithHandlesRow, MemoryUserRow, UserRow } from "./schema";
+import { toFriendSummary, toMatchSummary, toPublicUser, toSessionUser } from "./rowMappers";
+import type { Database, FriendshipWithUserRow, MatchWithHandlesRow, MemoryUserRow, UserRow } from "./schema";
 
 export type { Database } from "./schema";
 
@@ -37,6 +37,9 @@ export interface AppRepository {
   listLeaderboard(): Promise<LeaderboardEntry[]>;
   listRecentMatches(userId?: string): Promise<MatchSummary[]>;
   getDashboard(userId: string): Promise<DashboardSummary>;
+  listFriends(userId: string): Promise<FriendSummary[]>;
+  requestFriend(requesterId: string, addresseeHandle: string): Promise<FriendSummary>;
+  acceptFriend(userId: string, friendshipId: string): Promise<FriendSummary>;
 }
 
 export function createPostgresRepository(databaseUrl: string): AppRepository {
@@ -172,12 +175,40 @@ class PostgresRepository implements AppRepository {
     return { me: { ...user, email: null }, recentMatches: await this.listRecentMatches(userId), winRate: percentage(user.wins, user.losses), bestStreak: Math.max(1, Math.min(12, user.wins - user.losses + 3)) };
   }
 
+  async listFriends(userId: string): Promise<FriendSummary[]> {
+    const result = await sql<FriendshipWithUserRow>`
+      select f.id as friendship_id, f.status as friendship_status, u.*
+      from friendships f join users u on u.id = case when f.requester_id = ${userId} then f.addressee_id else f.requester_id end
+      where f.requester_id = ${userId} or f.addressee_id = ${userId}
+      order by f.updated_at desc
+    `.execute(this.db);
+    return result.rows.map(toFriendSummary);
+  }
+
+  async requestFriend(requesterId: string, addresseeHandle: string): Promise<FriendSummary> {
+    const addressee = await this.getUserByHandle(addresseeHandle);
+    if (!addressee) throw new Error("friend not found");
+    const result = await sql<{ id: string; status: FriendSummary["status"] }>`
+      insert into friendships (requester_id, addressee_id, status) values (${requesterId}, ${addressee.id}, 'pending')
+      on conflict (requester_id, addressee_id) do update set updated_at = now() returning id, status
+    `.execute(this.db);
+    return { id: firstRow(result).id, status: firstRow(result).status, user: addressee };
+  }
+
+  async acceptFriend(userId: string, friendshipId: string): Promise<FriendSummary> {
+    await sql`update friendships set status = 'accepted', updated_at = now() where id = ${friendshipId} and addressee_id = ${userId}`.execute(this.db);
+    const found = (await this.listFriends(userId)).find((friend) => friend.id === friendshipId);
+    if (!found) throw new Error("friendship not found");
+    return found;
+  }
+
 }
 
 class MemoryRepository implements AppRepository {
   private readonly users = new Map<string, MemoryUserRow>();
   private readonly sessions = new Map<string, string>();
   private readonly matches: MemoryMatchRecord[] = [];
+  private readonly friendships: FriendSummary[] = [];
 
   async close(): Promise<void> {}
 
@@ -270,6 +301,23 @@ class MemoryRepository implements AppRepository {
     return { me: { ...user, email: null }, recentMatches: await this.listRecentMatches(userId), winRate: percentage(user.wins, user.losses), bestStreak: 3 };
   }
 
+  async listFriends(): Promise<FriendSummary[]> { return this.friendships; }
+
+  async requestFriend(_requesterId: string, addresseeHandle: string): Promise<FriendSummary> {
+    const user = await this.getUserByHandle(addresseeHandle);
+    if (!user) throw new Error("friend not found");
+    const friend = { id: randomUUID(), user, status: "pending" as const };
+    this.friendships.push(friend);
+    return friend;
+  }
+
+  async acceptFriend(_userId: string, friendshipId: string): Promise<FriendSummary> {
+    const friend = this.friendships.find((item) => item.id === friendshipId);
+    if (!friend) throw new Error("friendship not found");
+    friend.status = "accepted";
+    return friend;
+  }
+
 }
 
 function firstRow<T>(result: { rows: T[] }): T {
diff --git a/packages/db/src/rowMappers.ts b/packages/db/src/rowMappers.ts
index 5fa2552..b73e1fd 100644
--- a/packages/db/src/rowMappers.ts
+++ b/packages/db/src/rowMappers.ts
@@ -1,5 +1,5 @@
-import type { MatchSummary, PublicUser, SessionUser } from "@pong-pong/shared";
-import type { MatchWithHandlesRow, UserProjectionRow } from "./schema";
+import type { FriendSummary, MatchSummary, PublicUser, SessionUser } from "@pong-pong/shared";
+import type { FriendshipWithUserRow, MatchWithHandlesRow, UserProjectionRow } from "./schema";
 
 export function toPublicUser(row: UserProjectionRow, online = false): PublicUser {
   return {
@@ -33,3 +33,7 @@ export function toMatchSummary(row: MatchWithHandlesRow, userId?: string): Match
     endedAt: row.ended_at.toISOString()
   };
 }
+
+export function toFriendSummary(row: FriendshipWithUserRow): FriendSummary {
+  return { id: row.friendship_id, status: row.friendship_status, user: toPublicUser(row, true) };
+}
diff --git a/packages/db/src/schema.ts b/packages/db/src/schema.ts
index 791f0b9..1670446 100644
--- a/packages/db/src/schema.ts
+++ b/packages/db/src/schema.ts
@@ -35,10 +35,20 @@ export interface MatchTable {
   ended_at: Generated<Date>;
 }
 
+export interface FriendshipTable {
+  id: Generated<string>;
+  requester_id: string;
+  addressee_id: string;
+  status: import("@pong-pong/shared").FriendshipStatus;
+  created_at: Generated<Date>;
+  updated_at: Generated<Date>;
+}
+
 export interface Database {
   users: UserTable;
   sessions: SessionTable;
   matches: MatchTable;
+  friendships: FriendshipTable;
 }
 
 export type UserRow = Selectable<UserTable>;
@@ -49,6 +59,11 @@ export interface MatchWithHandlesRow extends MatchRow {
   winner_handle: string | null;
   loser_handle: string | null;
 }
+
+export interface FriendshipWithUserRow extends UserRow {
+  friendship_id: string;
+  friendship_status: import("@pong-pong/shared").FriendshipStatus;
+}
 export type UserProjectionRow = Pick<
   UserRow,
   | "id"


## `feat(profile): 친구 요청 동작 연결`

diff --git a/apps/web/src/app/profile/[handle]/page.tsx b/apps/web/src/app/profile/[handle]/page.tsx
index 09b08ef..ab4ac86 100644
--- a/apps/web/src/app/profile/[handle]/page.tsx
+++ b/apps/web/src/app/profile/[handle]/page.tsx
@@ -7,18 +7,32 @@ import { AppShell } from "@/components/AppShell";
 import { StatCard } from "@/components/StatCard";
 import { sampleUsers } from "@/lib/sample";
 import { Target, Trophy, X } from "lucide-react";
+import { getProfile, requestFriend } from "@/lib/api";
 
 export default function ProfilePage({ params }: { params: Promise<{ handle: string }> }) {
   const [handle, setHandle] = useState("pongmaster42");
   const [user, setUser] = useState<PublicUser>(sampleUsers[0]);
+  const [message, setMessage] = useState("친구 요청은 로그인 후 보낼 수 있습니다.");
 
   useEffect(() => {
     params.then(({ handle: resolved }) => {
       setHandle(resolved);
       setUser(sampleUsers.find((item) => item.handle === resolved) ?? { ...sampleUsers[0], handle: resolved, displayName: "퐁마스터" });
+      getProfile(resolved)
+        .then((profile) => setUser(profile.user))
+        .catch(() => undefined);
     });
   }, [params]);
 
+  async function addFriend() {
+    try {
+      const friend = await requestFriend(handle);
+      setMessage(`${friend.user.displayName}에게 친구 요청을 보냈습니다.`);
+    } catch {
+      setMessage("친구 요청을 보내려면 로그인 상태와 대상 핸들을 확인해야 합니다.");
+    }
+  }
+
   return (
     <AppShell>
       <section className="card p-6">
@@ -32,16 +46,17 @@ export default function ProfilePage({ params }: { params: Promise<{ handle: stri
             </div>
           </div>
           <div className="flex gap-3">
-            <button className="focus-ring rounded-lg border border-line px-4 py-3 text-sm font-black text-ink">
+            <button className="focus-ring rounded-lg border border-line px-4 py-3 text-sm font-black text-ink" onClick={addFriend}>
               <UserPlus size={18} className="mr-2 inline" />
               친구 추가
             </button>
-            <button className="focus-ring rounded-lg border border-line px-4 py-3 text-sm font-black text-ink">
+            <button className="cursor-not-allowed rounded-lg border border-line bg-slate-50 px-4 py-3 text-sm font-black text-muted" disabled title="공유 링크 복사는 추후 프로필 배포 기능에서 다룹니다.">
               <Share2 size={18} className="mr-2 inline" />
-              공유
+              공유 예정
             </button>
           </div>
         </div>
+        <p className="mt-4 text-sm font-bold text-blue-700">{message}</p>
       </section>
       <section className="mt-5 grid gap-4 md:grid-cols-3">
         <StatCard icon={Trophy} label="승리" value={String(user.wins)} hint="누적 기록" tone="green" />


## `feat(db): friendship canonical pair 제약 추가`

diff --git a/packages/db/migrations/004_friendship_tournament_invariants.sql b/packages/db/migrations/004_friendship_tournament_invariants.sql
new file mode 100644
index 0000000..0c29204
--- /dev/null
+++ b/packages/db/migrations/004_friendship_tournament_invariants.sql
@@ -0,0 +1,41 @@
+delete from friendships
+where requester_id = addressee_id;
+
+update friendships as friendship
+set
+  status = 'accepted',
+  updated_at = greatest(friendship.updated_at, reverse_friendship.updated_at)
+from friendships as reverse_friendship
+where friendship.requester_id = reverse_friendship.addressee_id
+  and friendship.addressee_id = reverse_friendship.requester_id
+  and friendship.id <> reverse_friendship.id
+  and friendship.status = 'pending';
+
+with ranked_friendships as (
+  select
+    id,
+    row_number() over (
+      partition by least(requester_id, addressee_id), greatest(requester_id, addressee_id)
+      order by case when status = 'accepted' then 0 else 1 end, created_at asc, id asc
+    ) as position
+  from friendships
+)
+delete from friendships
+where id in (
+  select id
+  from ranked_friendships
+  where position > 1
+);
+
+alter table friendships
+  drop constraint if exists friendships_requester_id_addressee_id_key;
+
+alter table friendships
+  add constraint friendships_distinct_users_check
+  check (requester_id <> addressee_id);
+
+create unique index friendships_canonical_pair_unique
+  on friendships (
+    least(requester_id, addressee_id),
+    greatest(requester_id, addressee_id)
+  );


## `feat(db): PostgreSQL friendship 요청을 원자화`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index 5e7a060..4437522 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -317,18 +317,45 @@ class PostgresRepository implements AppRepository {
   async requestFriend(requesterId: string, addresseeHandle: string): Promise<FriendSummary> {
     const addressee = await this.getUserByHandle(addresseeHandle);
     if (!addressee) throw new Error("friend not found");
+    if (requesterId === addressee.id) throw new Error("cannot friend yourself");
     const result = await sql<{ id: string; status: FriendSummary["status"] }>`
-      insert into friendships (requester_id, addressee_id, status) values (${requesterId}, ${addressee.id}, 'pending')
-      on conflict (requester_id, addressee_id) do update set updated_at = now() returning id, status
+      insert into friendships (requester_id, addressee_id, status)
+      values (${requesterId}, ${addressee.id}, 'pending')
+      on conflict (
+        (least(requester_id, addressee_id)),
+        (greatest(requester_id, addressee_id))
+      ) do update set
+        status = case
+          when friendships.status = 'pending'
+            and friendships.requester_id = excluded.addressee_id
+            and friendships.addressee_id = excluded.requester_id
+          then 'accepted'
+          else friendships.status
+        end,
+        updated_at = case
+          when friendships.status = 'pending'
+            and friendships.requester_id = excluded.addressee_id
+            and friendships.addressee_id = excluded.requester_id
+          then now()
+          else friendships.updated_at
+        end
+      returning id, status
     `.execute(this.db);
-    return { id: firstRow(result).id, status: firstRow(result).status, user: addressee };
+    const friendship = firstRow(result);
+    return { id: friendship.id, status: friendship.status, user: addressee };
   }
 
   async acceptFriend(userId: string, friendshipId: string): Promise<FriendSummary> {
-    await sql`update friendships set status = 'accepted', updated_at = now() where id = ${friendshipId} and addressee_id = ${userId}`.execute(this.db);
-    const found = (await this.listFriends(userId)).find((friend) => friend.id === friendshipId);
-    if (!found) throw new Error("friendship not found");
-    return found;
+    const result = await sql<{ id: string; status: FriendSummary["status"]; requester_id: string }>`
+      update friendships
+      set status = 'accepted', updated_at = now()
+      where id = ${friendshipId} and addressee_id = ${userId}
+      returning id, status, requester_id
+    `.execute(this.db);
+    const friendship = firstRow(result);
+    const requester = await this.getUserById(friendship.requester_id);
+    if (!requester) throw new Error("friend not found");
+    return { id: friendship.id, status: friendship.status, user: requester };
   }
 
   async createMatch(input: CreateMatchInput): Promise<string> {


## `feat(db): memory friendship invariant 적용`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index f9fd6ff..fd4fdb1 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -7,6 +7,13 @@ import type { AdminActionRow, ChatMessageRow, ChatMessageWithSenderRow, Database
 
 export type { Database } from "./schema";
 
+type MemoryFriendship = {
+  id: string;
+  requesterId: string;
+  addresseeId: string;
+  status: FriendSummary["status"];
+};
+
 export interface DevLoginInput {
   handle: string;
   displayName: string;
@@ -701,7 +708,7 @@ class MemoryRepository implements AppRepository {
   private readonly sessions = new Map<string, string>();
   private readonly wsTickets = new Map<string, { userId: string; expiresAt: number }>();
   private readonly matches: MemoryMatchRecord[] = [];
-  private readonly friendships: FriendSummary[] = [];
+  private readonly friendships: MemoryFriendship[] = [];
   private readonly chats: ChatMessage[] = [];
   private readonly tournaments: TournamentSummary[] = [];
   private readonly adminActions: AdminActionSummary[] = [];
@@ -853,21 +860,55 @@ class MemoryRepository implements AppRepository {
     return { me: { ...user, email: null }, recentMatches, winRate: percentage(user.wins, user.losses), bestStreak: bestWinningStreak(recentMatches) };
   }
 
-  async listFriends(): Promise<FriendSummary[]> { return this.friendships; }
+  async listFriends(userId: string): Promise<FriendSummary[]> {
+    return this.friendships
+      .filter((friendship) => friendship.requesterId === userId || friendship.addresseeId === userId)
+      .map((friendship) => {
+        const otherUserId = friendship.requesterId === userId
+          ? friendship.addresseeId
+          : friendship.requesterId;
+        const otherUser = this.users.get(otherUserId);
+        if (!otherUser) throw new Error("friend not found");
+        return {
+          id: friendship.id,
+          status: friendship.status,
+          user: toPublicUser(otherUser, true)
+        };
+      });
+  }
 
-  async requestFriend(_requesterId: string, addresseeHandle: string): Promise<FriendSummary> {
+  async requestFriend(requesterId: string, addresseeHandle: string): Promise<FriendSummary> {
     const user = await this.getUserByHandle(addresseeHandle);
     if (!user) throw new Error("friend not found");
-    const friend = { id: randomUUID(), user, status: "pending" as const };
-    this.friendships.push(friend);
-    return friend;
+    if (requesterId === user.id) throw new Error("cannot friend yourself");
+    const existing = this.friendships.find((friendship) =>
+      (friendship.requesterId === requesterId && friendship.addresseeId === user.id)
+      || (friendship.requesterId === user.id && friendship.addresseeId === requesterId)
+    );
+    if (existing) {
+      const isReversePending = existing.status === "pending"
+        && existing.requesterId === user.id
+        && existing.addresseeId === requesterId;
+      if (isReversePending) existing.status = "accepted";
+      return { id: existing.id, status: existing.status, user };
+    }
+    const friendship: MemoryFriendship = {
+      id: randomUUID(),
+      requesterId,
+      addresseeId: user.id,
+      status: "pending"
+    };
+    this.friendships.push(friendship);
+    return { id: friendship.id, status: friendship.status, user };
   }
 
-  async acceptFriend(_userId: string, friendshipId: string): Promise<FriendSummary> {
+  async acceptFriend(userId: string, friendshipId: string): Promise<FriendSummary> {
     const friend = this.friendships.find((item) => item.id === friendshipId);
-    if (!friend) throw new Error("friendship not found");
+    if (!friend || friend.addresseeId !== userId) throw new Error("friendship not found");
     friend.status = "accepted";
-    return friend;
+    const requester = this.users.get(friend.requesterId);
+    if (!requester) throw new Error("friend not found");
+    return { id: friend.id, status: friend.status, user: toPublicUser(requester, true) };
   }
 
   async createMatch(input: CreateMatchInput): Promise<string> {


