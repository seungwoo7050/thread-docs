## `fix(db): 최근 경기에서 최고 연승 계산`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index f6c16af..c97e4e7 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -198,7 +198,8 @@ class PostgresRepository implements AppRepository {
   async getDashboard(userId: string): Promise<DashboardSummary> {
     const user = await this.getUserById(userId);
     if (!user) throw new Error("user not found");
-    return { me: { ...user, email: null }, recentMatches: await this.listRecentMatches(userId), winRate: percentage(user.wins, user.losses), bestStreak: Math.max(1, Math.min(12, user.wins - user.losses + 3)) };
+    const recentMatches = await this.listRecentMatches(userId);
+    return { me: { ...user, email: null }, recentMatches, winRate: percentage(user.wins, user.losses), bestStreak: bestWinningStreak(recentMatches) };
   }
 
   async listFriends(userId: string): Promise<FriendSummary[]> {
@@ -476,7 +477,8 @@ class MemoryRepository implements AppRepository {
   async getDashboard(userId: string): Promise<DashboardSummary> {
     const user = await this.getUserById(userId);
     if (!user) throw new Error("user not found");
-    return { me: { ...user, email: null }, recentMatches: await this.listRecentMatches(userId), winRate: percentage(user.wins, user.losses), bestStreak: 3 };
+    const recentMatches = await this.listRecentMatches(userId);
+    return { me: { ...user, email: null }, recentMatches, winRate: percentage(user.wins, user.losses), bestStreak: bestWinningStreak(recentMatches) };
   }
 
   async listFriends(): Promise<FriendSummary[]> { return this.friendships; }
@@ -637,6 +639,20 @@ function memoryMatchSummary(row: MemoryMatchRecord, userId?: string): MatchSumma
   };
 }
 
+function bestWinningStreak(matches: MatchSummary[]): number {
+  let best = 0;
+  let current = 0;
+  for (const match of [...matches].reverse()) {
+    if (match.result === "win") {
+      current += 1;
+      best = Math.max(best, current);
+    } else {
+      current = 0;
+    }
+  }
+  return best;
+}
+
 function memoryTournamentMatch(tournamentId: string, round: "semifinal" | "final", slot: number, left: PublicUser | null, right: PublicUser | null): TournamentMatchSummary {
   return { id: randomUUID(), tournamentId, round, slot, status: "ready", left, right, winner: null, scoreLeft: null, scoreRight: null, roomId: null, matchId: null };
 }


## `fix(dashboard): 빈 rating history를 정확히 표시`

diff --git a/apps/web/src/app/dashboard/page.tsx b/apps/web/src/app/dashboard/page.tsx
index 9842dcd..5eaaa41 100644
--- a/apps/web/src/app/dashboard/page.tsx
+++ b/apps/web/src/app/dashboard/page.tsx
@@ -29,8 +29,9 @@ export default function DashboardPage() {
     );
   }
 
-  const ratingPoints = buildRatingPoints(dashboard.me.rating, dashboard.recentMatches);
-  const chartPoints = toChartPoints(ratingPoints);
+  const hasRatingHistory = dashboard.recentMatches.length > 0;
+  const ratingPoints = hasRatingHistory ? buildRatingPoints(dashboard.me.rating, dashboard.recentMatches) : [];
+  const chartPoints = hasRatingHistory ? toChartPoints(ratingPoints) : "";
 
   return (
     <AppShell>
@@ -46,13 +47,19 @@ export default function DashboardPage() {
         <div className="card p-5">
           <h2 className="text-lg font-black text-ink">점수 흐름</h2>
           <div className="mt-5 h-64 rounded-lg border border-line bg-gradient-to-b from-blue-50 to-white p-5">
-            <svg viewBox="0 0 640 220" className="h-full w-full" role="img" aria-label="점수 상승 그래프">
-              <polyline points={chartPoints} fill="none" stroke="#1768f2" strokeWidth="8" strokeLinecap="round" strokeLinejoin="round" />
-              <line x1="0" y1="180" x2="640" y2="180" stroke="#d8e1ef" />
-              <line x1="0" y1="110" x2="640" y2="110" stroke="#d8e1ef" strokeDasharray="8 8" />
-            </svg>
+            {hasRatingHistory ? (
+              <svg viewBox="0 0 640 220" className="h-full w-full" role="img" aria-label="점수 상승 그래프">
+                <polyline points={chartPoints} fill="none" stroke="#1768f2" strokeWidth="8" strokeLinecap="round" strokeLinejoin="round" />
+                <line x1="0" y1="180" x2="640" y2="180" stroke="#d8e1ef" />
+                <line x1="0" y1="110" x2="640" y2="110" stroke="#d8e1ef" strokeDasharray="8 8" />
+              </svg>
+            ) : (
+              <div className="flex h-full items-center justify-center text-center text-sm font-bold text-muted">저장된 경기 후 점수 흐름이 표시됩니다.</div>
+            )}
           </div>
-          <p className="mt-3 text-sm font-bold text-muted">현재 점수 {dashboard.me.rating} 기준 최근 경기 변화를 역산해 표시합니다.</p>
+          <p className="mt-3 text-sm font-bold text-muted">
+            {hasRatingHistory ? `현재 점수 ${dashboard.me.rating} 기준 최근 경기 변화를 역산해 표시합니다.` : "아직 저장된 경기가 없어 점수 흐름을 표시하지 않습니다."}
+          </p>
         </div>
         <div className="card p-5">
           <h2 className="text-lg font-black text-ink">최근 경기</h2>
@@ -82,7 +89,7 @@ function buildRatingPoints(currentRating: number, recentMatches: MatchSummary[])
     rating += match.ratingDelta;
     points.push(rating);
   }
-  return points.length === 1 ? [currentRating - 1, currentRating] : points;
+  return points;
 }
 
 function toChartPoints(points: number[]): string {
