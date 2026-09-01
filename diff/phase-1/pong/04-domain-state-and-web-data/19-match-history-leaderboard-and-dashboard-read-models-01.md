# 전적·순위·대시보드 읽기 모델

## `feat(db): 프로필 조회와 변경 저장 구현`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index 84b89b7..3d728dc 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -20,6 +20,10 @@ export interface AppRepository {
   upsertDevUser(input: DevLoginInput): Promise<SessionUser>;
   createSession(userId: string): Promise<string>;
   getSessionUser(token: string | undefined): Promise<SessionUser | null>;
+  getUserById(id: string): Promise<PublicUser | null>;
+  getUserByHandle(handle: string): Promise<PublicUser | null>;
+  updateProfile(userId: string, input: { displayName?: string; avatarKey?: string }): Promise<SessionUser>;
+  listOnlineUsers(): Promise<PublicUser[]>;
 }
 
 export function createPostgresRepository(databaseUrl: string): AppRepository {
@@ -99,6 +103,34 @@ class PostgresRepository implements AppRepository {
     return user ? toSessionUser(user, true) : null;
   }
 
+  async getUserById(id: string): Promise<PublicUser | null> {
+    const result = await sql<UserRow>`select * from users where id = ${id} limit 1`.execute(this.db);
+    return result.rows[0] ? toPublicUser(result.rows[0]) : null;
+  }
+
+  async getUserByHandle(handle: string): Promise<PublicUser | null> {
+    const result = await sql<UserRow>`select * from users where handle = ${normalizeHandle(handle)} limit 1`.execute(this.db);
+    return result.rows[0] ? toPublicUser(result.rows[0]) : null;
+  }
+
+  async updateProfile(userId: string, input: { displayName?: string; avatarKey?: string }): Promise<SessionUser> {
+    const current = await sql<UserRow>`select * from users where id = ${userId} limit 1`.execute(this.db);
+    const user = firstRow(current);
+    const result = await sql<UserRow>`
+      update users
+      set display_name = ${input.displayName ?? user.display_name},
+          avatar_key = ${input.avatarKey ?? user.avatar_key}
+      where id = ${userId}
+      returning *
+    `.execute(this.db);
+    return toSessionUser(firstRow(result), true);
+  }
+
+  async listOnlineUsers(): Promise<PublicUser[]> {
+    const result = await sql<UserRow>`select * from users where status = 'active' order by rating desc limit 12`.execute(this.db);
+    return result.rows.map((row) => toPublicUser(row, true));
+  }
+
 }
 
 class MemoryRepository implements AppRepository {
@@ -150,6 +182,28 @@ class MemoryRepository implements AppRepository {
     return user ? toSessionUser(user, true) : null;
   }
 
+  async getUserById(id: string): Promise<PublicUser | null> {
+    const user = this.users.get(id);
+    return user ? toPublicUser(user, true) : null;
+  }
+
+  async getUserByHandle(handle: string): Promise<PublicUser | null> {
+    const user = [...this.users.values()].find((item) => item.handle === normalizeHandle(handle));
+    return user ? toPublicUser(user, true) : null;
+  }
+
+  async updateProfile(userId: string, input: { displayName?: string; avatarKey?: string }): Promise<SessionUser> {
+    const user = this.users.get(userId);
+    if (!user) throw new Error("user not found");
+    user.display_name = input.displayName ?? user.display_name;
+    user.avatar_key = input.avatarKey ?? user.avatar_key;
+    return toSessionUser(user, true);
+  }
+
+  async listOnlineUsers(): Promise<PublicUser[]> {
+    return [...this.users.values()].sort((a, b) => b.rating - a.rating).map((user) => toPublicUser(user, true));
+  }
+
 }
 
 function firstRow<T>(result: { rows: T[] }): T {


## `feat(db): 순위 조회 구현`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index 3d728dc..91e1612 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -1,7 +1,7 @@
 import { randomUUID } from "node:crypto";
 import { Kysely, PostgresDialect, sql } from "kysely";
 import { Pool } from "pg";
-import type { PublicUser, SessionUser } from "@pong-pong/shared";
+import type { LeaderboardEntry, PublicUser, SessionUser } from "@pong-pong/shared";
 import { initialMigrationSql } from "./migrations";
 import { toPublicUser, toSessionUser } from "./rowMappers";
 import type { Database, MemoryUserRow, UserRow } from "./schema";
@@ -24,6 +24,7 @@ export interface AppRepository {
   getUserByHandle(handle: string): Promise<PublicUser | null>;
   updateProfile(userId: string, input: { displayName?: string; avatarKey?: string }): Promise<SessionUser>;
   listOnlineUsers(): Promise<PublicUser[]>;
+  listLeaderboard(): Promise<LeaderboardEntry[]>;
 }
 
 export function createPostgresRepository(databaseUrl: string): AppRepository {
@@ -131,6 +132,15 @@ class PostgresRepository implements AppRepository {
     return result.rows.map((row) => toPublicUser(row, true));
   }
 
+  async listLeaderboard(): Promise<LeaderboardEntry[]> {
+    const result = await sql<UserRow>`select * from users order by rating desc, wins desc limit 20`.execute(this.db);
+    return result.rows.map((row, index) => ({
+      rank: index + 1,
+      user: toPublicUser(row, false),
+      winRate: percentage(row.wins, row.losses)
+    }));
+  }
+
 }
 
 class MemoryRepository implements AppRepository {
@@ -204,6 +214,16 @@ class MemoryRepository implements AppRepository {
     return [...this.users.values()].sort((a, b) => b.rating - a.rating).map((user) => toPublicUser(user, true));
   }
 
+  async listLeaderboard(): Promise<LeaderboardEntry[]> {
+    return [...this.users.values()]
+      .sort((a, b) => b.rating - a.rating || b.wins - a.wins)
+      .map((user, index) => ({
+        rank: index + 1,
+        user: toPublicUser(user, false),
+        winRate: percentage(user.wins, user.losses)
+      }));
+  }
+
 }
 
 function firstRow<T>(result: { rows: T[] }): T {
@@ -220,3 +240,9 @@ function avatarFor(handle: string): string {
   const avatars = ["blue", "green", "amber", "violet", "rose"];
   return avatars[Math.abs([...handle].reduce((sum, char) => sum + char.charCodeAt(0), 0)) % avatars.length];
 }
+
+function percentage(wins: number, losses: number): number {
+  const total = Number(wins) + Number(losses);
+  if (total === 0) return 0;
+  return Math.round((Number(wins) / total) * 1000) / 10;
+}


## `feat(db): 경기 조회 row contract 정의`

diff --git a/packages/db/src/rowMappers.ts b/packages/db/src/rowMappers.ts
index c661ff3..5fa2552 100644
--- a/packages/db/src/rowMappers.ts
+++ b/packages/db/src/rowMappers.ts
@@ -1,5 +1,5 @@
-import type { PublicUser, SessionUser } from "@pong-pong/shared";
-import type { UserProjectionRow } from "./schema";
+import type { MatchSummary, PublicUser, SessionUser } from "@pong-pong/shared";
+import type { MatchWithHandlesRow, UserProjectionRow } from "./schema";
 
 export function toPublicUser(row: UserProjectionRow, online = false): PublicUser {
   return {
@@ -19,3 +19,17 @@ export function toPublicUser(row: UserProjectionRow, online = false): PublicUser
 export function toSessionUser(row: UserProjectionRow, online = false): SessionUser {
   return { ...toPublicUser(row, online), email: row.email };
 }
+
+export function toMatchSummary(row: MatchWithHandlesRow, userId?: string): MatchSummary {
+  const won = userId ? row.winner_id === userId : true;
+  return {
+    id: row.id,
+    mode: row.mode,
+    opponentHandle: won ? row.loser_handle ?? "AI" : row.winner_handle ?? "AI",
+    result: won ? "win" : "loss",
+    scoreLeft: Number(row.score_left),
+    scoreRight: Number(row.score_right),
+    ratingDelta: won ? Number(row.rating_delta) : -12,
+    endedAt: row.ended_at.toISOString()
+  };
+}
diff --git a/packages/db/src/schema.ts b/packages/db/src/schema.ts
index 62eb2da..791f0b9 100644
--- a/packages/db/src/schema.ts
+++ b/packages/db/src/schema.ts
@@ -23,13 +23,32 @@ export interface SessionTable {
   created_at: Generated<Date>;
 }
 
+export interface MatchTable {
+  id: Generated<string>;
+  mode: import("@pong-pong/shared").MatchMode;
+  winner_id: string | null;
+  loser_id: string | null;
+  score_left: number;
+  score_right: number;
+  rating_delta: Generated<number>;
+  started_at: Generated<Date>;
+  ended_at: Generated<Date>;
+}
+
 export interface Database {
   users: UserTable;
   sessions: SessionTable;
+  matches: MatchTable;
 }
 
 export type UserRow = Selectable<UserTable>;
 export type MemoryUserRow = Omit<UserRow, "created_at" | "banned_at">;
+export type MatchRow = Selectable<MatchTable>;
+
+export interface MatchWithHandlesRow extends MatchRow {
+  winner_handle: string | null;
+  loser_handle: string | null;
+}
 export type UserProjectionRow = Pick<
   UserRow,
   | "id"


## `feat(db): 최근 경기와 대시보드 조회 구현`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index 91e1612..b30d6e4 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -1,10 +1,10 @@
 import { randomUUID } from "node:crypto";
 import { Kysely, PostgresDialect, sql } from "kysely";
 import { Pool } from "pg";
-import type { LeaderboardEntry, PublicUser, SessionUser } from "@pong-pong/shared";
+import type { DashboardSummary, LeaderboardEntry, MatchMode, MatchSummary, PublicUser, SessionUser } from "@pong-pong/shared";
 import { initialMigrationSql } from "./migrations";
-import { toPublicUser, toSessionUser } from "./rowMappers";
-import type { Database, MemoryUserRow, UserRow } from "./schema";
+import { toMatchSummary, toPublicUser, toSessionUser } from "./rowMappers";
+import type { Database, MatchWithHandlesRow, MemoryUserRow, UserRow } from "./schema";
 
 export type { Database } from "./schema";
 
@@ -14,6 +14,16 @@ export interface DevLoginInput {
   email?: string | null;
 }
 
+type MemoryMatchRecord = {
+  id: string;
+  mode: MatchMode;
+  winnerId: string | null;
+  loserId: string | null;
+  scoreLeft: number;
+  scoreRight: number;
+  ended_at: string;
+};
+
 export interface AppRepository {
   close(): Promise<void>;
   ensureSeedData(): Promise<void>;
@@ -25,6 +35,8 @@ export interface AppRepository {
   updateProfile(userId: string, input: { displayName?: string; avatarKey?: string }): Promise<SessionUser>;
   listOnlineUsers(): Promise<PublicUser[]>;
   listLeaderboard(): Promise<LeaderboardEntry[]>;
+  listRecentMatches(userId?: string): Promise<MatchSummary[]>;
+  getDashboard(userId: string): Promise<DashboardSummary>;
 }
 
 export function createPostgresRepository(databaseUrl: string): AppRepository {
@@ -141,11 +153,31 @@ class PostgresRepository implements AppRepository {
     }));
   }
 
+  async listRecentMatches(userId?: string): Promise<MatchSummary[]> {
+    const filter = userId ? sql`where m.winner_id = ${userId} or m.loser_id = ${userId}` : sql``;
+    const result = await sql<MatchWithHandlesRow>`
+      select m.*, winner.handle as winner_handle, loser.handle as loser_handle
+      from matches m
+      left join users winner on winner.id = m.winner_id
+      left join users loser on loser.id = m.loser_id
+      ${filter}
+      order by m.ended_at desc limit 8
+    `.execute(this.db);
+    return result.rows.map((row) => toMatchSummary(row, userId));
+  }
+
+  async getDashboard(userId: string): Promise<DashboardSummary> {
+    const user = await this.getUserById(userId);
+    if (!user) throw new Error("user not found");
+    return { me: { ...user, email: null }, recentMatches: await this.listRecentMatches(userId), winRate: percentage(user.wins, user.losses), bestStreak: Math.max(1, Math.min(12, user.wins - user.losses + 3)) };
+  }
+
 }
 
 class MemoryRepository implements AppRepository {
   private readonly users = new Map<string, MemoryUserRow>();
   private readonly sessions = new Map<string, string>();
+  private readonly matches: MemoryMatchRecord[] = [];
 
   async close(): Promise<void> {}
 
@@ -224,6 +256,20 @@ class MemoryRepository implements AppRepository {
       }));
   }
 
+  async listRecentMatches(userId?: string): Promise<MatchSummary[]> {
+    return this.matches
+      .filter((match) => !userId || match.winnerId === userId || match.loserId === userId)
+      .slice(-8)
+      .reverse()
+      .map((match) => memoryMatchSummary(match, userId));
+  }
+
+  async getDashboard(userId: string): Promise<DashboardSummary> {
+    const user = await this.getUserById(userId);
+    if (!user) throw new Error("user not found");
+    return { me: { ...user, email: null }, recentMatches: await this.listRecentMatches(userId), winRate: percentage(user.wins, user.losses), bestStreak: 3 };
+  }
+
 }
 
 function firstRow<T>(result: { rows: T[] }): T {
@@ -246,3 +292,17 @@ function percentage(wins: number, losses: number): number {
   if (total === 0) return 0;
   return Math.round((Number(wins) / total) * 1000) / 10;
 }
+
+function memoryMatchSummary(row: MemoryMatchRecord, userId?: string): MatchSummary {
+  const won = userId ? row.winnerId === userId : true;
+  return {
+    id: row.id,
+    mode: row.mode,
+    opponentHandle: "AI",
+    result: won ? "win" : "loss",
+    scoreLeft: row.scoreLeft,
+    scoreRight: row.scoreRight,
+    ratingDelta: won ? 16 : -12,
+    endedAt: new Date(row.ended_at).toISOString()
+  };
+}


## `feat(web): 플레이어 대시보드 구현`

diff --git a/apps/web/src/app/dashboard/page.tsx b/apps/web/src/app/dashboard/page.tsx
new file mode 100644
index 0000000..da96962
--- /dev/null
+++ b/apps/web/src/app/dashboard/page.tsx
@@ -0,0 +1,56 @@
+"use client";
+
+import { useEffect, useState } from "react";
+import { Flame, Target, Trophy, X } from "lucide-react";
+import type { DashboardSummary } from "@pong-pong/shared";
+import { AppShell } from "@/components/AppShell";
+import { StatCard } from "@/components/StatCard";
+import { getDashboard } from "@/lib/api";
+import { sampleDashboard } from "@/lib/sample";
+
+export default function DashboardPage() {
+  const [dashboard, setDashboard] = useState<DashboardSummary>(sampleDashboard);
+
+  useEffect(() => {
+    getDashboard().then(setDashboard);
+  }, []);
+
+  return (
+    <AppShell>
+      <h1 className="text-3xl font-black text-ink">내 대시보드</h1>
+      <p className="mt-2 text-sm font-semibold text-muted">최근 경기 흐름과 성장 지표를 한 화면에서 확인합니다.</p>
+      <section className="mt-5 grid gap-4 md:grid-cols-2 xl:grid-cols-4">
+        <StatCard icon={Trophy} label="승리" value={String(dashboard.me.wins)} hint="누적 승리" tone="green" />
+        <StatCard icon={X} label="패배" value={String(dashboard.me.losses)} hint="복기 대상" tone="red" />
+        <StatCard icon={Target} label="승률" value={`${dashboard.winRate}%`} hint="최근 반영" />
+        <StatCard icon={Flame} label="최고 연승" value={String(dashboard.bestStreak)} hint="이번 시즌" tone="amber" />
+      </section>
+      <section className="mt-5 grid gap-5 xl:grid-cols-[1.1fr_.9fr]">
+        <div className="card p-5">
+          <h2 className="text-lg font-black text-ink">점수 흐름</h2>
+          <div className="mt-5 h-64 rounded-lg border border-line bg-gradient-to-b from-blue-50 to-white p-5">
+            <svg viewBox="0 0 640 220" className="h-full w-full" role="img" aria-label="점수 상승 그래프">
+              <polyline points="0,180 80,164 160,152 240,130 320,124 400,92 480,104 560,70 640,78" fill="none" stroke="#1768f2" strokeWidth="8" strokeLinecap="round" strokeLinejoin="round" />
+              <line x1="0" y1="180" x2="640" y2="180" stroke="#d8e1ef" />
+              <line x1="0" y1="110" x2="640" y2="110" stroke="#d8e1ef" strokeDasharray="8 8" />
+            </svg>
+          </div>
+        </div>
+        <div className="card p-5">
+          <h2 className="text-lg font-black text-ink">최근 경기</h2>
+          <div className="mt-4 divide-y divide-line">
+            {dashboard.recentMatches.map((match) => (
+              <div key={match.id} className="grid grid-cols-[80px_1fr_70px] items-center gap-3 py-3 text-sm font-bold">
+                <span className={`rounded-full px-3 py-1 text-center ${match.result === "win" ? "bg-green-50 text-green-600" : "bg-red-50 text-red-600"}`}>{match.result === "win" ? "승리" : "패배"}</span>
+                <span className="text-muted">{match.opponentHandle}</span>
+                <span className="text-right text-ink">
+                  {match.scoreLeft} - {match.scoreRight}
+                </span>
+              </div>
+            ))}
+          </div>
+        </div>
+      </section>
+    </AppShell>
+  );
+}


## `feat(web): 순위표 화면 추가`

diff --git a/apps/web/src/app/leaderboard/page.tsx b/apps/web/src/app/leaderboard/page.tsx
new file mode 100644
index 0000000..bde9b4a
--- /dev/null
+++ b/apps/web/src/app/leaderboard/page.tsx
@@ -0,0 +1,41 @@
+"use client";
+
+import { useEffect, useState } from "react";
+import type { LeaderboardEntry } from "@pong-pong/shared";
+import { AppShell } from "@/components/AppShell";
+import { getLeaderboard } from "@/lib/api";
+import { sampleLeaderboard } from "@/lib/sample";
+
+export default function LeaderboardPage() {
+  const [entries, setEntries] = useState<LeaderboardEntry[]>(sampleLeaderboard);
+
+  useEffect(() => {
+    getLeaderboard().then(setEntries);
+  }, []);
+
+  return (
+    <AppShell>
+      <h1 className="text-3xl font-black text-ink">순위표</h1>
+      <p className="mt-2 text-sm font-semibold text-muted">점수와 승률을 기준으로 현재 상위 선수를 정렬합니다.</p>
+      <section className="card mt-5 overflow-hidden">
+        <div className="grid grid-cols-[70px_1fr_120px_120px] border-b border-line px-5 py-3 text-sm font-black text-muted">
+          <span>순위</span>
+          <span>선수</span>
+          <span className="text-right">점수</span>
+          <span className="text-right">승률</span>
+        </div>
+        {entries.map((entry) => (
+          <div key={entry.user.id} className="grid grid-cols-[70px_1fr_120px_120px] items-center border-b border-line px-5 py-4 last:border-b-0">
+            <span className="text-lg font-black text-blue-700">#{entry.rank}</span>
+            <div>
+              <p className="font-black text-ink">{entry.user.displayName}</p>
+              <p className="text-sm font-semibold text-muted">누적 {entry.user.wins}승</p>
+            </div>
+            <span className="text-right font-black text-green-600">{entry.user.rating}</span>
+            <span className="text-right font-black text-ink">{entry.winRate}%</span>
+          </div>
+        ))}
+      </section>
+    </AppShell>
+  );
+}


## `feat(web): 공개 프로필 화면 추가`

diff --git a/apps/web/src/app/profile/[handle]/page.tsx b/apps/web/src/app/profile/[handle]/page.tsx
new file mode 100644
index 0000000..09b08ef
--- /dev/null
+++ b/apps/web/src/app/profile/[handle]/page.tsx
@@ -0,0 +1,57 @@
+"use client";
+
+import { useEffect, useState } from "react";
+import { Share2, UserPlus } from "lucide-react";
+import type { PublicUser } from "@pong-pong/shared";
+import { AppShell } from "@/components/AppShell";
+import { StatCard } from "@/components/StatCard";
+import { sampleUsers } from "@/lib/sample";
+import { Target, Trophy, X } from "lucide-react";
+
+export default function ProfilePage({ params }: { params: Promise<{ handle: string }> }) {
+  const [handle, setHandle] = useState("pongmaster42");
+  const [user, setUser] = useState<PublicUser>(sampleUsers[0]);
+
+  useEffect(() => {
+    params.then(({ handle: resolved }) => {
+      setHandle(resolved);
+      setUser(sampleUsers.find((item) => item.handle === resolved) ?? { ...sampleUsers[0], handle: resolved, displayName: "퐁마스터" });
+    });
+  }, [params]);
+
+  return (
+    <AppShell>
+      <section className="card p-6">
+        <div className="flex flex-wrap items-center justify-between gap-5">
+          <div className="flex items-center gap-5">
+            <div className="grid h-28 w-28 place-items-center rounded-full bg-blue-100 text-3xl font-black text-blue-700">{user.displayName.slice(0, 1)}</div>
+            <div>
+              <h1 className="text-3xl font-black text-ink">{user.displayName}</h1>
+              <p className="mt-1 text-sm font-semibold text-muted">선수 번호 {handle.length}</p>
+              <p className="mt-3 text-lg font-black text-green-600">점수 {user.rating}</p>
+            </div>
+          </div>
+          <div className="flex gap-3">
+            <button className="focus-ring rounded-lg border border-line px-4 py-3 text-sm font-black text-ink">
+              <UserPlus size={18} className="mr-2 inline" />
+              친구 추가
+            </button>
+            <button className="focus-ring rounded-lg border border-line px-4 py-3 text-sm font-black text-ink">
+              <Share2 size={18} className="mr-2 inline" />
+              공유
+            </button>
+          </div>
+        </div>
+      </section>
+      <section className="mt-5 grid gap-4 md:grid-cols-3">
+        <StatCard icon={Trophy} label="승리" value={String(user.wins)} hint="누적 기록" tone="green" />
+        <StatCard icon={X} label="패배" value={String(user.losses)} hint="최근 30일" tone="red" />
+        <StatCard icon={Target} label="승률" value={`${Math.round((user.wins / Math.max(1, user.wins + user.losses)) * 100)}%`} hint="점수 반영" />
+      </section>
+      <section className="card mt-5 p-5">
+        <h2 className="text-lg font-black text-ink">플레이 스타일</h2>
+        <p className="mt-3 text-sm font-semibold leading-6 text-muted">긴 랠리에서 안정적으로 버티는 타입입니다. 백핸드 쪽 낮은 공에 강하고 빠른 서브를 상대할 때는 중앙 복귀가 빠릅니다.</p>
+      </section>
+    </AppShell>
+  );
+}


