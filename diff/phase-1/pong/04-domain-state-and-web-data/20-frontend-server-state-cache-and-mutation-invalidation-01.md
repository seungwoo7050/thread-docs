# 프런트엔드 서버 상태 캐시와 변경 무효화

## `build(web): React Query 의존성 추가`

diff --git a/apps/web/package.json b/apps/web/package.json
index 16179db..e788f51 100644
--- a/apps/web/package.json
+++ b/apps/web/package.json
@@ -12,6 +12,7 @@
   },
   "dependencies": {
     "@pong-pong/shared": "workspace:*",
+    "@tanstack/react-query": "^5.101.4",
     "lucide-react": "^0.511.0",
     "next": "^15.3.2",
     "react": "^19.1.0",
diff --git a/pnpm-lock.yaml b/pnpm-lock.yaml
index 4f2b04c..2686951 100644
--- a/pnpm-lock.yaml
+++ b/pnpm-lock.yaml
@@ -66,6 +66,9 @@ importers:
       '@pong-pong/shared':
         specifier: workspace:*
         version: link:../../packages/shared
+      '@tanstack/react-query':
+        specifier: ^5.101.4
+        version: 5.101.4(react@19.2.5)
       lucide-react:
         specifier: ^0.511.0
         version: 0.511.0(react@19.2.5)
@@ -774,6 +777,14 @@ packages:
   '@swc/helpers@0.5.15':
     resolution: {integrity: sha512-JQ5TuMi45Owi4/BIMAJBoSQoOJu12oOk/gADqlcUL9JEdHB8vyjUSsxqeNXnmXHjYKMi2WcYtezGEEhqUI/E2g==}
 
+  '@tanstack/query-core@5.101.4':
+    resolution: {integrity: sha512-gNwcvOJcRbLWPOLG/2OBm+zM+Yv+MKsXKEOWC57USuZDEsI71hEErQsiEGx5wX9rzWWkfwM0fVSPoiIFSsxfiw==}
+
+  '@tanstack/react-query@5.101.4':
+    resolution: {integrity: sha512-yRg2pfOCxIs4ZJW3XYYHU/WgtD04FHSnfHlpRT7h7pR77hwkdRG4wxbKe4aq6P0RvXUTBSQpQeadS1SUYUe+KA==}
+    peerDependencies:
+      react: ^18 || ^19
+
   '@testcontainers/postgresql@12.0.4':
     resolution: {integrity: sha512-a/pLU6j5lpKKAlUTPwqweqMGhOSjgTSb6HBX69TOrXn32ifU37nnQDmNFTj8ddOAw+BQL9oTRkeOxVbZkqhgZA==}
 
@@ -2552,6 +2563,13 @@ snapshots:
     dependencies:
       tslib: 2.8.1
 
+  '@tanstack/query-core@5.101.4': {}
+
+  '@tanstack/react-query@5.101.4(react@19.2.5)':
+    dependencies:
+      '@tanstack/query-core': 5.101.4
+      react: 19.2.5
+
   '@testcontainers/postgresql@12.0.4':
     dependencies:
       testcontainers: 12.0.4


## `refactor(web): query key와 retry 정책 정의`

diff --git a/apps/web/src/lib/query.ts b/apps/web/src/lib/query.ts
new file mode 100644
index 0000000..48423ce
--- /dev/null
+++ b/apps/web/src/lib/query.ts
@@ -0,0 +1,48 @@
+import {
+  queryOptions,
+  type QueryClient,
+  type QueryKey
+} from "@tanstack/react-query";
+import {
+  ApiError,
+  getAdminActions,
+  getAdminUsers,
+  getDashboard,
+  getLeaderboard,
+  getLobby,
+  getMe,
+  getProfile,
+  getTournaments
+} from "./api";
+
+export const queryKeys = {
+  me: () => ["user", "me"] as const,
+  lobby: () => ["lobby"] as const,
+  dashboard: () => ["dashboard"] as const,
+  profile: (handle: string) => ["profiles", handle] as const,
+  leaderboard: () => ["leaderboard"] as const,
+  friends: () => ["friends"] as const,
+  tournaments: () => ["tournaments"] as const,
+  adminUsers: () => ["admin", "users"] as const,
+  adminActions: () => ["admin", "actions"] as const
+};
+
+export const mutationInvalidations = {
+  login: () => [queryKeys.me(), queryKeys.lobby()] as const,
+  lobbyChat: () => [queryKeys.lobby()] as const,
+  friendRequest: () => [queryKeys.friends()] as const,
+  tournamentChange: () => [queryKeys.tournaments()] as const,
+  adminStatus: () => [queryKeys.adminUsers(), queryKeys.adminActions()] as const
+};
+
+export async function invalidateExactQueries(
+  client: QueryClient,
+  keys: readonly QueryKey[]
+): Promise<void> {
+  await Promise.all(keys.map((queryKey) => client.invalidateQueries({ queryKey, exact: true })));
+}
+
+export function shouldRetryQuery(failureCount: number, error: unknown): boolean {
+  if (error instanceof ApiError && error.status === 401) return false;
+  return failureCount < 1;
+}


## `refactor(web): session query와 cache invalidation 추가`

diff --git a/apps/web/src/lib/query.ts b/apps/web/src/lib/query.ts
index 48423ce..4ed39cf 100644
--- a/apps/web/src/lib/query.ts
+++ b/apps/web/src/lib/query.ts
@@ -35,6 +35,54 @@ export const mutationInvalidations = {
   adminStatus: () => [queryKeys.adminUsers(), queryKeys.adminActions()] as const
 };
 
+export const meQueryOptions = () => queryOptions({
+  queryKey: queryKeys.me(),
+  queryFn: ({ signal }) => getMe(signal),
+  staleTime: 30_000
+});
+
+export const lobbyQueryOptions = () => queryOptions({
+  queryKey: queryKeys.lobby(),
+  queryFn: ({ signal }) => getLobby(signal),
+  staleTime: 5_000
+});
+
+export const dashboardQueryOptions = () => queryOptions({
+  queryKey: queryKeys.dashboard(),
+  queryFn: ({ signal }) => getDashboard(signal),
+  staleTime: 10_000
+});
+
+export const profileQueryOptions = (handle: string) => queryOptions({
+  queryKey: queryKeys.profile(handle),
+  queryFn: ({ signal }) => getProfile(handle, signal),
+  staleTime: 30_000
+});
+
+export const leaderboardQueryOptions = () => queryOptions({
+  queryKey: queryKeys.leaderboard(),
+  queryFn: ({ signal }) => getLeaderboard(signal),
+  staleTime: 15_000
+});
+
+export const tournamentsQueryOptions = () => queryOptions({
+  queryKey: queryKeys.tournaments(),
+  queryFn: ({ signal }) => getTournaments(signal),
+  staleTime: 10_000
+});
+
+export const adminUsersQueryOptions = () => queryOptions({
+  queryKey: queryKeys.adminUsers(),
+  queryFn: ({ signal }) => getAdminUsers(signal),
+  staleTime: 5_000
+});
+
+export const adminActionsQueryOptions = () => queryOptions({
+  queryKey: queryKeys.adminActions(),
+  queryFn: ({ signal }) => getAdminActions(signal),
+  staleTime: 5_000
+});
+
 export async function invalidateExactQueries(
   client: QueryClient,
   keys: readonly QueryKey[]
@@ -46,3 +94,22 @@ export function shouldRetryQuery(failureCount: number, error: unknown): boolean
   if (error instanceof ApiError && error.status === 401) return false;
   return failureCount < 1;
 }
+
+export function expireSession(client: QueryClient): void {
+  const sessionScopedKeys = [
+    queryKeys.lobby(),
+    queryKeys.dashboard(),
+    queryKeys.friends(),
+    queryKeys.adminUsers(),
+    queryKeys.adminActions()
+  ] as const;
+
+  for (const queryKey of sessionScopedKeys) {
+    if (client.getQueryState(queryKey)?.fetchStatus === "fetching") {
+      setTimeout(() => client.removeQueries({ queryKey, exact: true }), 0);
+    } else {
+      client.removeQueries({ queryKey, exact: true });
+    }
+  }
+  client.setQueryData(queryKeys.me(), null);
+}


## `refactor(web): React Query provider 연결`

diff --git a/apps/web/src/app/layout.tsx b/apps/web/src/app/layout.tsx
index fbf111c..f07a2c4 100644
--- a/apps/web/src/app/layout.tsx
+++ b/apps/web/src/app/layout.tsx
@@ -1,4 +1,5 @@
 import type { Metadata } from "next";
+import { QueryProvider } from "@/components/QueryProvider";
 import "./globals.css";
 
 export const metadata: Metadata = {
@@ -9,7 +10,9 @@ export const metadata: Metadata = {
 export default function RootLayout({ children }: { children: React.ReactNode }) {
   return (
     <html lang="ko">
-      <body>{children}</body>
+      <body>
+        <QueryProvider>{children}</QueryProvider>
+      </body>
     </html>
   );
 }
diff --git a/apps/web/src/components/QueryProvider.tsx b/apps/web/src/components/QueryProvider.tsx
new file mode 100644
index 0000000..d266fd8
--- /dev/null
+++ b/apps/web/src/components/QueryProvider.tsx
@@ -0,0 +1,28 @@
+"use client";
+
+import { useEffect, useState, type ReactNode } from "react";
+import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
+import { SESSION_EXPIRED_EVENT } from "@/lib/api";
+import { expireSession, shouldRetryQuery } from "@/lib/query";
+
+export function QueryProvider({ children }: { children: ReactNode }) {
+  const [client] = useState(() => new QueryClient({
+    defaultOptions: {
+      queries: {
+        retry: shouldRetryQuery,
+        refetchOnWindowFocus: true
+      },
+      mutations: {
+        retry: false
+      }
+    }
+  }));
+
+  useEffect(() => {
+    const onSessionExpired = () => expireSession(client);
+    window.addEventListener(SESSION_EXPIRED_EVENT, onSessionExpired);
+    return () => window.removeEventListener(SESSION_EXPIRED_EVENT, onSessionExpired);
+  }, [client]);
+
+  return <QueryClientProvider client={client}>{children}</QueryClientProvider>;
+}


## `refactor(web): lobby와 login을 query cache로 전환`

diff --git a/apps/web/src/app/page.tsx b/apps/web/src/app/page.tsx
index a1088c5..6adfe44 100644
--- a/apps/web/src/app/page.tsx
+++ b/apps/web/src/app/page.tsx
@@ -1,39 +1,47 @@
 "use client";
 
-import { useCallback, useEffect, useRef, useState } from "react";
+import { useEffect, useRef, useState } from "react";
+import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
 import { Bot, Clock, MessageCircle, Trophy, Users, Zap } from "lucide-react";
-import { parseServerEvent, type ChatMessage, type LobbyStats, type PublicUser, type SessionUser } from "@pong-pong/shared";
+import { parseServerEvent, type LobbyResponse } from "@pong-pong/shared";
 import { AppShell } from "@/components/AppShell";
 import { LoginPanel } from "@/components/LoginPanel";
 import { PongCanvas } from "@/components/PongCanvas";
 import { StatCard } from "@/components/StatCard";
-import { getLobby, getMe, requestWsTicket, sendLobbyChat } from "@/lib/api";
+import { requestWsTicket, sendLobbyChat } from "@/lib/api";
+import {
+  invalidateExactQueries,
+  lobbyQueryOptions,
+  meQueryOptions,
+  mutationInvalidations,
+  queryKeys
+} from "@/lib/query";
 
 const WS_URL = process.env.NEXT_PUBLIC_WS_URL ?? "ws://localhost:4000/ws";
 
 export default function HomePage() {
-  const [me, setMe] = useState<SessionUser | null>(null);
-  const [players, setPlayers] = useState<PublicUser[]>([]);
-  const [chat, setChat] = useState<ChatMessage[]>([]);
-  const [stats, setStats] = useState<LobbyStats | null>(null);
+  const queryClient = useQueryClient();
+  const meQuery = useQuery(meQueryOptions());
+  const lobbyQuery = useQuery(lobbyQueryOptions());
+  const lobby = lobbyQuery.data;
+  const me = lobby?.me ?? meQuery.data ?? null;
+  const players = lobby?.onlinePlayers ?? [];
+  const chat = lobby?.chat ?? [];
+  const stats = lobby?.stats ?? null;
   const [chatInput, setChatInput] = useState("");
   const [notice, setNotice] = useState("");
   const socketRef = useRef<WebSocket | null>(null);
   const userId = me?.id;
-
-  const loadLobby = useCallback(async () => {
-    const lobby = await getLobby();
-    setPlayers(lobby.onlinePlayers);
-    setChat(lobby.chat);
-    setStats(lobby.stats);
-    if (lobby.me) setMe(lobby.me);
-    setNotice("");
-  }, []);
-
-  useEffect(() => {
-    getMe().then(setMe);
-    loadLobby().catch(() => setNotice("서버 로비 정보를 불러오지 못했습니다."));
-  }, [loadLobby]);
+  const chatMutation = useMutation({
+    mutationFn: (body: string) => sendLobbyChat(body),
+    onSuccess: async (message) => {
+      queryClient.setQueryData<LobbyResponse>(queryKeys.lobby(), (current) => current ? {
+        ...current,
+        chat: [...current.chat.filter((item) => item.id !== message.id).slice(-19), message]
+      } : current);
+      await invalidateExactQueries(queryClient, mutationInvalidations.lobbyChat());
+    }
+  });
 
   useEffect(() => {
     if (!userId) return;
@@ -49,10 +57,14 @@ export default function HomePage() {
         socket.onmessage = (event) => {
           const message = parseServerEvent(event.data);
           if (message.type === "chat.message" && message.message.scope === "lobby") {
-            setChat((current) => [...current.filter((item) => item.id !== message.message.id).slice(-19), message.message]);
+            queryClient.setQueryData<LobbyResponse>(queryKeys.lobby(), (current) => current ? {
+              ...current,
+              chat: [...current.chat.filter((item) => item.id !== message.message.id).slice(-19), message.message]
+            } : current);
           }
           if (message.type === "presence.changed") {
-            loadLobby().catch(() => setNotice("로비 지표를 갱신하지 못했습니다."));
+            invalidateExactQueries(queryClient, [queryKeys.lobby()])
+              .catch(() => setNotice("로비 지표를 갱신하지 못했습니다."));
           }
           if (message.type === "error") setNotice(message.message);
         };
@@ -73,7 +85,7 @@ export default function HomePage() {
       if (socket.readyState === WebSocket.OPEN || socket.readyState === WebSocket.CONNECTING) socket.close();
       if (socketRef.current === socket) socketRef.current = null;
     };
-  }, [loadLobby, userId]);
+  }, [queryClient, userId]);
 
   async function submitLobbyChat(event: React.FormEvent<HTMLFormElement>) {
     event.preventDefault();
@@ -84,8 +96,7 @@ export default function HomePage() {
       if (socket?.readyState === WebSocket.OPEN) {
         socket.send(JSON.stringify({ v: 1, type: "chat.send", scope: "lobby", roomId: null, body }));
       } else {
-        const message = await sendLobbyChat(body);
-        setChat((current) => [...current.slice(-19), message]);
+        await chatMutation.mutateAsync(body);
       }
       setChatInput("");
       setNotice("");
@@ -98,7 +109,7 @@ export default function HomePage() {
     return (
       <div className="min-h-screen bg-slate-50 p-4">
         <div className="mx-auto grid min-h-[calc(100vh-32px)] max-w-6xl items-center gap-6 lg:grid-cols-[420px_1fr]">
-          <LoginPanel onLogin={setMe} />
+          <LoginPanel />
           <section className="card hidden p-6 lg:block">
             <PongCanvas />
             <div className="mt-5 grid grid-cols-3 gap-3 text-center text-sm font-bold text-muted">
@@ -158,7 +169,7 @@ export default function HomePage() {
           <h2 className="flex items-center gap-2 text-lg font-black text-ink">
             <MessageCircle size={20} /> 로비 채팅
           </h2>
-          {notice ? <p className="mt-3 rounded-lg bg-amber-50 px-3 py-2 text-sm font-bold text-amber-700">{notice}</p> : null}
+          {notice || lobbyQuery.isError ? <p className="mt-3 rounded-lg bg-amber-50 px-3 py-2 text-sm font-bold text-amber-700">{notice || "서버 로비 정보를 불러오지 못했습니다."}</p> : null}
           <div className="mt-4 grid gap-3">
             {chat.length === 0 ? <div className="rounded-lg border border-dashed border-line p-3 text-sm font-semibold text-muted">아직 로비 채팅이 없습니다.</div> : null}
             {chat.map((message) => (
diff --git a/apps/web/src/components/LoginPanel.tsx b/apps/web/src/components/LoginPanel.tsx
index 6d7d1fa..d8159b0 100644
--- a/apps/web/src/components/LoginPanel.tsx
+++ b/apps/web/src/components/LoginPanel.tsx
@@ -1,13 +1,21 @@
 "use client";
 
 import { useState } from "react";
+import { useMutation, useQueryClient } from "@tanstack/react-query";
 import { devLogin } from "@/lib/api";
-import type { SessionUser } from "@pong-pong/shared";
+import { invalidateExactQueries, mutationInvalidations, queryKeys } from "@/lib/query";
 
-export function LoginPanel({ onLogin }: { onLogin: (user: SessionUser) => void }) {
+export function LoginPanel() {
+  const queryClient = useQueryClient();
   const [handle, setHandle] = useState("퐁마스터");
   const [displayName, setDisplayName] = useState("퐁마스터");
-  const [error, setError] = useState<string | null>(null);
+  const login = useMutation({
+    mutationFn: () => devLogin(handle, displayName),
+    onSuccess: async (user) => {
+      queryClient.setQueryData(queryKeys.me(), user);
+      await invalidateExactQueries(queryClient, mutationInvalidations.login());
+    }
+  });
 
   return (
     <section className="card grid gap-5 p-6">
@@ -25,19 +33,13 @@ export function LoginPanel({ onLogin }: { onLogin: (user: SessionUser) => void }
           <input className="focus-ring rounded-lg border border-line px-3 py-2" value={displayName} onChange={(event) => setDisplayName(event.target.value)} />
         </label>
       </div>
-      {error ? <p className="text-sm font-bold text-red-600">{error}</p> : null}
+      {login.isError ? <p className="text-sm font-bold text-red-600">API 서버에 연결하지 못했습니다.</p> : null}
       <button
         className="focus-ring rounded-lg bg-blue-600 px-4 py-3 text-sm font-black text-white hover:bg-blue-700"
-        onClick={async () => {
-          try {
-            setError(null);
-            onLogin(await devLogin(handle, displayName));
-          } catch {
-            setError("API 서버에 연결하지 못했습니다.");
-          }
-        }}
+        disabled={login.isPending}
+        onClick={() => login.mutate()}
       >
-        개발 로그인
+        {login.isPending ? "로그인 중" : "개발 로그인"}
       </button>
     </section>
   );


## `refactor(web): dashboard와 leaderboard를 query cache로 전환`

diff --git a/apps/web/src/app/dashboard/page.tsx b/apps/web/src/app/dashboard/page.tsx
index 5eaaa41..876c854 100644
--- a/apps/web/src/app/dashboard/page.tsx
+++ b/apps/web/src/app/dashboard/page.tsx
@@ -1,30 +1,23 @@
 "use client";
 
-import { useEffect, useState } from "react";
+import { useQuery } from "@tanstack/react-query";
 import { Flame, Target, Trophy, X } from "lucide-react";
-import type { DashboardSummary, MatchSummary } from "@pong-pong/shared";
+import type { MatchSummary } from "@pong-pong/shared";
 import { AppShell } from "@/components/AppShell";
 import { StatCard } from "@/components/StatCard";
-import { getDashboard } from "@/lib/api";
+import { dashboardQueryOptions } from "@/lib/query";
 
 export default function DashboardPage() {
-  const [dashboard, setDashboard] = useState<DashboardSummary | null>(null);
-  const [message, setMessage] = useState("대시보드를 불러오는 중입니다.");
-
-  useEffect(() => {
-    getDashboard()
-      .then((summary) => {
-        setDashboard(summary);
-        setMessage("");
-      })
-      .catch(() => setMessage("대시보드를 불러오려면 로그인 상태와 서버 연결을 확인해야 합니다."));
-  }, []);
+  const dashboardQuery = useQuery(dashboardQueryOptions());
+  const dashboard = dashboardQuery.data;
 
   if (!dashboard) {
     return (
       <AppShell>
         <h1 className="text-3xl font-black text-ink">내 대시보드</h1>
-        <p className="mt-4 rounded-lg border border-line bg-white p-4 text-sm font-bold text-muted">{message}</p>
+        <p className="mt-4 rounded-lg border border-line bg-white p-4 text-sm font-bold text-muted">
+          {dashboardQuery.isError ? "대시보드를 불러오려면 로그인 상태와 서버 연결을 확인해야 합니다." : "대시보드를 불러오는 중입니다."}
+        </p>
       </AppShell>
     );
   }
diff --git a/apps/web/src/app/leaderboard/page.tsx b/apps/web/src/app/leaderboard/page.tsx
index 507e5b3..1cd7da8 100644
--- a/apps/web/src/app/leaderboard/page.tsx
+++ b/apps/web/src/app/leaderboard/page.tsx
@@ -1,22 +1,17 @@
 "use client";
 
-import { useEffect, useState } from "react";
-import type { LeaderboardEntry } from "@pong-pong/shared";
+import { useQuery } from "@tanstack/react-query";
 import { AppShell } from "@/components/AppShell";
-import { getLeaderboard } from "@/lib/api";
+import { leaderboardQueryOptions } from "@/lib/query";
 
 export default function LeaderboardPage() {
-  const [entries, setEntries] = useState<LeaderboardEntry[]>([]);
-  const [message, setMessage] = useState("순위표를 불러오는 중입니다.");
-
-  useEffect(() => {
-    getLeaderboard()
-      .then((items) => {
-        setEntries(items);
-        setMessage("");
-      })
-      .catch(() => setMessage("순위표를 불러오지 못했습니다."));
-  }, []);
+  const leaderboardQuery = useQuery(leaderboardQueryOptions());
+  const entries = leaderboardQuery.data ?? [];
+  const message = leaderboardQuery.isError
+    ? "순위표를 불러오지 못했습니다."
+    : leaderboardQuery.isPending
+      ? "순위표를 불러오는 중입니다."
+      : "";
 
   return (
     <AppShell>


## `refactor(web): profile 조회를 query cache로 전환`

diff --git a/apps/web/src/app/profile/[handle]/page.tsx b/apps/web/src/app/profile/[handle]/page.tsx
index 4838aa8..78cde45 100644
--- a/apps/web/src/app/profile/[handle]/page.tsx
+++ b/apps/web/src/app/profile/[handle]/page.tsx
@@ -1,51 +1,45 @@
 "use client";
 
-import { useEffect, useState } from "react";
+import { use, useState } from "react";
+import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
 import { Share2, Target, Trophy, UserPlus, X } from "lucide-react";
-import type { MatchSummary, PublicUser } from "@pong-pong/shared";
 import { AppShell } from "@/components/AppShell";
 import { StatCard } from "@/components/StatCard";
-import { getProfile, requestFriend } from "@/lib/api";
+import { requestFriend } from "@/lib/api";
+import {
+  invalidateExactQueries,
+  mutationInvalidations,
+  profileQueryOptions
+} from "@/lib/query";
 
 export default function ProfilePage({ params }: { params: Promise<{ handle: string }> }) {
-  const [handle, setHandle] = useState("pongmaster42");
-  const [user, setUser] = useState<PublicUser | null>(null);
-  const [recentMatches, setRecentMatches] = useState<MatchSummary[]>([]);
-  const [message, setMessage] = useState("프로필 정보를 불러오는 중입니다.");
-
-  useEffect(() => {
-    params.then(({ handle: resolved }) => {
-      setHandle(resolved);
-      getProfile(resolved)
-        .then((profile) => {
-          setUser(profile.user);
-          setRecentMatches(profile.recentMatches);
-          setMessage("공개 프로필 정보를 표시합니다.");
-        })
-        .catch(() => {
-          setUser(null);
-          setRecentMatches([]);
-          setMessage("프로필 정보를 불러오지 못했습니다.");
-        });
-    });
-  }, [params]);
-
-  async function addFriend() {
-    try {
-      const friend = await requestFriend(handle);
-      setMessage(`${friend.user.displayName}에게 친구 요청을 보냈습니다.`);
-    } catch {
-      setMessage("친구 요청을 보내려면 로그인 상태와 대상 핸들을 확인해야 합니다.");
-    }
-  }
+  const { handle } = use(params);
+  const queryClient = useQueryClient();
+  const profileQuery = useQuery(profileQueryOptions(handle));
+  const [notice, setNotice] = useState("");
+  const user = profileQuery.data?.user ?? null;
+  const recentMatches = profileQuery.data?.recentMatches ?? [];
+  const message = notice || (profileQuery.isError
+    ? "프로필 정보를 불러오지 못했습니다."
+    : profileQuery.isPending
+      ? "프로필 정보를 불러오는 중입니다."
+      : "공개 프로필 정보를 표시합니다.");
+  const friendRequest = useMutation({
+    mutationFn: () => requestFriend(handle),
+    onSuccess: async (friend) => {
+      setNotice(`${friend.user.displayName}에게 친구 요청을 보냈습니다.`);
+      await invalidateExactQueries(queryClient, mutationInvalidations.friendRequest());
+    },
+    onError: () => setNotice("친구 요청을 보내려면 로그인 상태와 대상 핸들을 확인해야 합니다.")
+  });
 
   async function shareProfile() {
     try {
       const url = `${window.location.origin}/profile/${handle}`;
       await navigator.clipboard.writeText(url);
-      setMessage("프로필 공유 링크를 복사했습니다.");
+      setNotice("프로필 공유 링크를 복사했습니다.");
     } catch {
-      setMessage("프로필 공유 링크를 복사하지 못했습니다.");
+      setNotice("프로필 공유 링크를 복사하지 못했습니다.");
     }
   }
 
@@ -72,7 +66,7 @@ export default function ProfilePage({ params }: { params: Promise<{ handle: stri
             </div>
           </div>
           <div className="flex gap-3">
-            <button className="focus-ring rounded-lg border border-line px-4 py-3 text-sm font-black text-ink disabled:cursor-not-allowed disabled:bg-slate-50 disabled:text-muted" onClick={addFriend} disabled={user.isNpc}>
+            <button className="focus-ring rounded-lg border border-line px-4 py-3 text-sm font-black text-ink disabled:cursor-not-allowed disabled:bg-slate-50 disabled:text-muted" onClick={() => friendRequest.mutate()} disabled={user.isNpc || friendRequest.isPending}>
               <UserPlus size={18} className="mr-2 inline" />
               친구 추가
             </button>


