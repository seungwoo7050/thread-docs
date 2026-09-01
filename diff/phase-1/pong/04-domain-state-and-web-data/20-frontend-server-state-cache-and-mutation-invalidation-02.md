## `refactor(web): tournament 조회와 mutation을 query cache로 전환`

diff --git a/apps/web/src/app/tournaments/page.tsx b/apps/web/src/app/tournaments/page.tsx
index 5bd900d..08557b5 100644
--- a/apps/web/src/app/tournaments/page.tsx
+++ b/apps/web/src/app/tournaments/page.tsx
@@ -1,50 +1,61 @@
 "use client";
 
-import { useEffect, useState } from "react";
+import { useState } from "react";
+import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
 import { Plus, Trophy } from "lucide-react";
-import type { SessionUser, TournamentMatchSummary, TournamentSummary } from "@pong-pong/shared";
+import type { SessionUser, TournamentMatchSummary } from "@pong-pong/shared";
 import { AppShell } from "@/components/AppShell";
-import { createTournament, getMe, getTournaments, joinTournament } from "@/lib/api";
+import { createTournament, joinTournament } from "@/lib/api";
+import {
+  invalidateExactQueries,
+  meQueryOptions,
+  mutationInvalidations,
+  tournamentsQueryOptions
+} from "@/lib/query";
 
 export default function TournamentsPage() {
-  const [items, setItems] = useState<TournamentSummary[]>([]);
+  const queryClient = useQueryClient();
+  const tournamentsQuery = useQuery(tournamentsQueryOptions());
+  const { data: me = null } = useQuery(meQueryOptions());
+  const items = tournamentsQuery.data ?? [];
   const [selectedId, setSelectedId] = useState("");
-  const [message, setMessage] = useState("대회 목록을 불러오는 중입니다.");
-  const [me, setMe] = useState<SessionUser | null>(null);
+  const [notice, setNotice] = useState("");
   const selected = items.find((item) => item.id === selectedId) ?? items[0];
-
-  useEffect(() => {
-    getMe().then(setMe);
-    getTournaments()
-      .then((tournaments) => {
-        setItems(tournaments);
-        setSelectedId((current) => current || tournaments[0]?.id || "");
-        setMessage(tournaments.length === 0 ? "진행 중인 대회가 없습니다." : "대회를 선택하면 브래킷과 참가 상태를 확인할 수 있습니다.");
-      })
-      .catch(() => setMessage("대회 목록을 불러오지 못했습니다."));
-  }, []);
-
-  async function create() {
-    try {
-      const tournament = await createTournament("새로운 퐁퐁 컵");
-      setItems((current) => [tournament, ...current.filter((item) => item.id !== tournament.id)]);
+  const message = notice || (tournamentsQuery.isError
+    ? "대회 목록을 불러오지 못했습니다."
+    : tournamentsQuery.isPending
+      ? "대회 목록을 불러오는 중입니다."
+      : items.length === 0
+        ? "진행 중인 대회가 없습니다."
+        : "대회를 선택하면 브래킷과 참가 상태를 확인할 수 있습니다.");
+  const createMutation = useMutation({
+    mutationFn: () => createTournament("새로운 퐁퐁 컵"),
+    onSuccess: async (tournament) => {
+      setSelectedId(tournament.id);
+      setNotice(`${tournament.name}을 생성했습니다.`);
+      await invalidateExactQueries(queryClient, mutationInvalidations.tournamentChange());
+    },
+    onError: () => setNotice("토너먼트 생성에는 로그인이 필요합니다.")
+  });
+  const joinMutation = useMutation({
+    mutationFn: (id: string) => joinTournament(id),
+    onSuccess: async (tournament) => {
       setSelectedId(tournament.id);
-      setMessage(`${tournament.name}을 생성했습니다.`);
-    } catch {
-      setMessage("토너먼트 생성에는 로그인이 필요합니다.");
+      setNotice(`${tournament.name}에 참가했습니다.`);
+      await invalidateExactQueries(queryClient, mutationInvalidations.tournamentChange());
+    },
+    onError: () => setNotice("토너먼트 참가에는 로그인이 필요합니다.")
+  });
+
+  function create() {
+    if (!createMutation.isPending) {
+      createMutation.mutate();
     }
   }
 
-  async function join() {
+  function join() {
     if (!selected) return;
-    try {
-      const tournament = await joinTournament(selected.id);
-      setItems((current) => current.map((item) => (item.id === tournament.id ? tournament : item)));
-      setSelectedId(tournament.id);
-      setMessage(`${tournament.name}에 참가했습니다.`);
-    } catch {
-      setMessage("토너먼트 참가에는 로그인이 필요합니다.");
-    }
+    joinMutation.mutate(selected.id);
   }
 
   return (
@@ -55,7 +66,7 @@ export default function TournamentsPage() {
           <p className="mt-2 text-sm font-semibold text-muted">4인 싱글 엘리미네이션으로 짧은 컵 대회를 운영합니다.</p>
           <p className="mt-2 text-sm font-bold text-blue-700">{message}</p>
         </div>
-        <button className="focus-ring rounded-lg bg-blue-600 px-4 py-3 text-sm font-black text-white" onClick={create}>
+        <button className="focus-ring rounded-lg bg-blue-600 px-4 py-3 text-sm font-black text-white disabled:cursor-not-allowed disabled:bg-slate-300" onClick={create} disabled={createMutation.isPending}>
           <Plus size={18} className="mr-2 inline" />
           토너먼트 생성
         </button>
@@ -71,7 +82,7 @@ export default function TournamentsPage() {
                 className={`focus-ring rounded-lg border p-4 text-left hover:border-blue-300 ${selected?.id === item.id ? "border-blue-500 bg-blue-50" : "border-line"}`}
                 onClick={() => {
                   setSelectedId(item.id);
-                  setMessage(`${item.name}을 선택했습니다.`);
+                  setNotice(`${item.name}을 선택했습니다.`);
                 }}
               >
                 <p className="font-black text-ink">{item.name}</p>
@@ -95,7 +106,7 @@ export default function TournamentsPage() {
                   {selected.winner ? ` · 우승 ${selected.winner.displayName}` : ""}
                 </p>
               </div>
-              <button className="focus-ring rounded-lg bg-green-600 px-4 py-2 text-sm font-black text-white disabled:cursor-not-allowed disabled:bg-slate-300" onClick={join} disabled={selected.playerCount >= selected.capacity}>
+              <button className="focus-ring rounded-lg bg-green-600 px-4 py-2 text-sm font-black text-white disabled:cursor-not-allowed disabled:bg-slate-300" onClick={join} disabled={selected.playerCount >= selected.capacity || joinMutation.isPending}>
                 참가
               </button>
             </div>


## `refactor(web): admin 조회와 mutation을 query cache로 전환`

diff --git a/apps/web/src/app/admin/page.tsx b/apps/web/src/app/admin/page.tsx
index 72035b7..af39e1e 100644
--- a/apps/web/src/app/admin/page.tsx
+++ b/apps/web/src/app/admin/page.tsx
@@ -1,36 +1,46 @@
 "use client";
 
-import { useEffect, useState } from "react";
+import { useState } from "react";
+import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
 import { Shield } from "lucide-react";
-import type { AdminActionSummary, PublicUser } from "@pong-pong/shared";
+import type { PublicUser } from "@pong-pong/shared";
 import { AppShell } from "@/components/AppShell";
-import { getAdminActions, getAdminUsers, setUserStatus } from "@/lib/api";
+import { setUserStatus } from "@/lib/api";
+import {
+  adminActionsQueryOptions,
+  adminUsersQueryOptions,
+  invalidateExactQueries,
+  mutationInvalidations
+} from "@/lib/query";
 
 export default function AdminPage() {
-  const [users, setUsers] = useState<PublicUser[]>([]);
-  const [actions, setActions] = useState<AdminActionSummary[]>([]);
+  const queryClient = useQueryClient();
+  const usersQuery = useQuery(adminUsersQueryOptions());
+  const actionsQuery = useQuery(adminActionsQueryOptions());
+  const users = usersQuery.data ?? [];
+  const actions = actionsQuery.data ?? [];
   const [reason, setReason] = useState("운영자 검토");
-  const [message, setMessage] = useState("운영자 계정으로 로그인하면 상태 변경이 저장됩니다.");
+  const [notice, setNotice] = useState("");
+  const message = notice || (usersQuery.isError || actionsQuery.isError
+    ? "운영자 권한이 필요합니다."
+    : usersQuery.isPending || actionsQuery.isPending
+      ? "운영자 계정 정보를 확인하고 있습니다."
+      : "사용자 목록과 감사 로그를 불러왔습니다.");
+  const statusMutation = useMutation({
+    mutationFn: ({ user, nextStatus }: { user: PublicUser; nextStatus: "active" | "banned" }) =>
+      setUserStatus(user.id, nextStatus, reason.trim() || "운영자 검토"),
+    onSuccess: async (updated) => {
+      setNotice(`${updated.displayName} 상태를 ${updated.status === "active" ? "정상" : "정지"}으로 변경했습니다.`);
+      await invalidateExactQueries(queryClient, mutationInvalidations.adminStatus());
+    },
+    onError: () => setNotice("상태 변경은 운영자 권한으로 로그인해야 가능합니다.")
+  });
 
-  useEffect(() => {
-    Promise.all([getAdminUsers(), getAdminActions()])
-      .then(([userItems, actionItems]) => {
-        setUsers(userItems);
-        setActions(actionItems);
-        setMessage("사용자 목록과 감사 로그를 불러왔습니다.");
-      })
-      .catch(() => setMessage("운영자 권한이 필요합니다."));
-  }, []);
-
-  async function toggleUser(user: PublicUser) {
-    try {
-      const updated = await setUserStatus(user.id, user.status === "active" ? "banned" : "active", reason.trim() || "운영자 검토");
-      setUsers((current) => current.map((item) => (item.id === updated.id ? updated : item)));
-      setActions(await getAdminActions());
-      setMessage(`${updated.displayName} 상태를 ${updated.status === "active" ? "정상" : "정지"}으로 변경했습니다.`);
-    } catch {
-      setMessage("상태 변경은 운영자 권한으로 로그인해야 가능합니다.");
-    }
+  function toggleUser(user: PublicUser) {
+    statusMutation.mutate({
+      user,
+      nextStatus: user.status === "active" ? "banned" : "active"
+    });
   }
 
   return (
@@ -68,7 +78,7 @@ export default function AdminPage() {
             </div>
             <span className="text-right font-black text-green-600">{user.rating}</span>
             <span className="text-right text-sm font-black text-ink">{user.status === "active" ? "정상" : "정지"}</span>
-            <button className="focus-ring justify-self-end rounded-lg border border-line px-3 py-2 text-sm font-black text-ink" onClick={() => toggleUser(user)}>
+            <button className="focus-ring justify-self-end rounded-lg border border-line px-3 py-2 text-sm font-black text-ink disabled:cursor-not-allowed disabled:bg-slate-50 disabled:text-muted" onClick={() => toggleUser(user)} disabled={statusMutation.isPending}>
               {user.status === "active" ? "정지" : "해제"}
             </button>
           </div>


## `refactor(web): shell의 session 소비를 query cache로 통합`

diff --git a/apps/web/src/components/AppShell.tsx b/apps/web/src/components/AppShell.tsx
index 8c33678..0b277b8 100644
--- a/apps/web/src/components/AppShell.tsx
+++ b/apps/web/src/components/AppShell.tsx
@@ -1,15 +1,14 @@
 "use client";
 
-import { useEffect, useState } from "react";
+import { useQuery } from "@tanstack/react-query";
 import Link from "next/link";
 import { usePathname } from "next/navigation";
 import { BarChart3, Gamepad2, Home, Shield, Trophy, UserRound, Users } from "lucide-react";
-import type { SessionUser } from "@pong-pong/shared";
-import { getMe } from "@/lib/api";
+import { meQueryOptions } from "@/lib/query";
 
 export function AppShell({ children }: { children: React.ReactNode }) {
   const pathname = usePathname();
-  const [me, setMe] = useState<SessionUser | null>(null);
+  const { data: me = null } = useQuery(meQueryOptions());
   const profileHref = me ? `/profile/${me.handle}` : "/";
   const nav = [
     { id: "lobby", href: "/", label: "로비", icon: Home },
@@ -21,10 +20,6 @@ export function AppShell({ children }: { children: React.ReactNode }) {
     { id: "admin", href: "/admin", label: "관리", icon: Shield }
   ];
 
-  useEffect(() => {
-    getMe().then(setMe).catch(() => setMe(null));
-  }, []);
-
   return (
     <div className="min-h-screen lg:grid lg:grid-cols-[248px_1fr]">
       <aside className="border-b border-line bg-white lg:min-h-screen lg:border-b-0 lg:border-r">


## `test(web): query cache key·retry·invalidation 검증`

diff --git a/apps/web/src/lib/query.test.ts b/apps/web/src/lib/query.test.ts
new file mode 100644
index 0000000..376e775
--- /dev/null
+++ b/apps/web/src/lib/query.test.ts
@@ -0,0 +1,110 @@
+import { QueryClient, QueryObserver } from "@tanstack/react-query";
+import { describe, expect, it } from "vitest";
+import { ApiError } from "./api";
+import {
+  expireSession,
+  invalidateExactQueries,
+  mutationInvalidations,
+  queryKeys,
+  shouldRetryQuery
+} from "./query";
+
+describe("query key contract", () => {
+  it("keeps screen data in stable, scoped keys", () => {
+    expect(queryKeys.me()).toEqual(["user", "me"]);
+    expect(queryKeys.lobby()).toEqual(["lobby"]);
+    expect(queryKeys.dashboard()).toEqual(["dashboard"]);
+    expect(queryKeys.profile("pong-master")).toEqual(["profiles", "pong-master"]);
+    expect(queryKeys.leaderboard()).toEqual(["leaderboard"]);
+    expect(queryKeys.friends()).toEqual(["friends"]);
+    expect(queryKeys.tournaments()).toEqual(["tournaments"]);
+    expect(queryKeys.adminUsers()).toEqual(["admin", "users"]);
+    expect(queryKeys.adminActions()).toEqual(["admin", "actions"]);
+  });
+
+  it("invalidates only the data affected by each mutation", () => {
+    expect(mutationInvalidations.login()).toEqual([
+      queryKeys.me(),
+      queryKeys.lobby()
+    ]);
+    expect(mutationInvalidations.lobbyChat()).toEqual([queryKeys.lobby()]);
+    expect(mutationInvalidations.friendRequest()).toEqual([queryKeys.friends()]);
+    expect(mutationInvalidations.tournamentChange()).toEqual([
+      queryKeys.tournaments()
+    ]);
+    expect(mutationInvalidations.adminStatus()).toEqual([
+      queryKeys.adminUsers(),
+      queryKeys.adminActions()
+    ]);
+  });
+
+  it("marks exact mutation keys stale without touching adjacent caches", async () => {
+    const client = new QueryClient();
+    client.setQueryData(queryKeys.adminUsers(), [{ id: "user-1" }]);
+    client.setQueryData(queryKeys.adminActions(), [{ id: "action-1" }]);
+    client.setQueryData(queryKeys.leaderboard(), [{ rank: 1 }]);
+
+    await invalidateExactQueries(client, mutationInvalidations.adminStatus());
+
+    expect(client.getQueryState(queryKeys.adminUsers())?.isInvalidated).toBe(true);
+    expect(client.getQueryState(queryKeys.adminActions())?.isInvalidated).toBe(true);
+    expect(client.getQueryState(queryKeys.leaderboard())?.isInvalidated).toBe(false);
+  });
+});
+
+describe("session expiration", () => {
+  it("does not retry an expired cookie session", () => {
+    const unauthorized = new ApiError(401, "UNAUTHORIZED", "로그인이 필요합니다.", "req-401");
+
+    expect(shouldRetryQuery(0, unauthorized)).toBe(false);
+    expect(shouldRetryQuery(0, new Error("temporary"))).toBe(true);
+    expect(shouldRetryQuery(1, new Error("temporary"))).toBe(false);
+  });
+
+  it("drops session-scoped data while keeping public caches", () => {
+    const client = new QueryClient();
+    client.setQueryData(queryKeys.me(), { id: "user-1" });
+    client.setQueryData(queryKeys.lobby(), { me: { id: "user-1" } });
+    client.setQueryData(queryKeys.dashboard(), { wins: 2 });
+    client.setQueryData(queryKeys.friends(), [{ id: "friend-1" }]);
+    client.setQueryData(queryKeys.adminUsers(), [{ id: "user-1" }]);
+    client.setQueryData(queryKeys.adminActions(), [{ id: "action-1" }]);
+    client.setQueryData(queryKeys.leaderboard(), [{ rank: 1 }]);
+    client.setQueryData(queryKeys.profile("pong-master"), { rating: 1_000 });
+    client.setQueryData(queryKeys.tournaments(), [{ id: "tournament-1" }]);
+
+    expireSession(client);
+
+    expect(client.getQueryData(queryKeys.me())).toBeNull();
+    expect(client.getQueryData(queryKeys.lobby())).toBeUndefined();
+    expect(client.getQueryData(queryKeys.dashboard())).toBeUndefined();
+    expect(client.getQueryData(queryKeys.friends())).toBeUndefined();
+    expect(client.getQueryData(queryKeys.adminUsers())).toBeUndefined();
+    expect(client.getQueryData(queryKeys.adminActions())).toBeUndefined();
+    expect(client.getQueryData(queryKeys.leaderboard())).toEqual([{ rank: 1 }]);
+    expect(client.getQueryData(queryKeys.profile("pong-master"))).toEqual({ rating: 1_000 });
+    expect(client.getQueryData(queryKeys.tournaments())).toEqual([{ id: "tournament-1" }]);
+  });
+
+  it("lets an active unauthorized query settle instead of leaving it pending", async () => {
+    const client = new QueryClient({
+      defaultOptions: { queries: { retry: false } }
+    });
+    const observer = new QueryObserver(client, {
+      queryKey: queryKeys.adminUsers(),
+      queryFn: async () => {
+        expireSession(client);
+        throw new Error("unauthorized");
+      }
+    });
+    const unsubscribe = observer.subscribe(() => undefined);
+
+    await new Promise((resolve) => setTimeout(resolve, 10));
+
+    expect(observer.getCurrentResult()).toMatchObject({
+      status: "error",
+      fetchStatus: "idle"
+    });
+    unsubscribe();
+  });
+});
