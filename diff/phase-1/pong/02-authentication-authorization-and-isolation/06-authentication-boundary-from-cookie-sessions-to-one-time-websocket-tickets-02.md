## `fix(web): browser token 저장 제거`

diff --git a/apps/web/src/app/admin/page.tsx b/apps/web/src/app/admin/page.tsx
index b68287b..72035b7 100644
--- a/apps/web/src/app/admin/page.tsx
+++ b/apps/web/src/app/admin/page.tsx
@@ -4,7 +4,7 @@ import { useEffect, useState } from "react";
 import { Shield } from "lucide-react";
 import type { AdminActionSummary, PublicUser } from "@pong-pong/shared";
 import { AppShell } from "@/components/AppShell";
-import { apiFetch, getAdminActions, setUserStatus } from "@/lib/api";
+import { getAdminActions, getAdminUsers, setUserStatus } from "@/lib/api";
 
 export default function AdminPage() {
   const [users, setUsers] = useState<PublicUser[]>([]);
@@ -13,9 +13,9 @@ export default function AdminPage() {
   const [message, setMessage] = useState("운영자 계정으로 로그인하면 상태 변경이 저장됩니다.");
 
   useEffect(() => {
-    Promise.all([apiFetch<{ users: PublicUser[] }>("/admin/users"), getAdminActions()])
-      .then(([result, actionItems]) => {
-        setUsers(result.users);
+    Promise.all([getAdminUsers(), getAdminActions()])
+      .then(([userItems, actionItems]) => {
+        setUsers(userItems);
         setActions(actionItems);
         setMessage("사용자 목록과 감사 로그를 불러왔습니다.");
       })
diff --git a/apps/web/src/app/page.tsx b/apps/web/src/app/page.tsx
index 9626c6b..78a038c 100644
--- a/apps/web/src/app/page.tsx
+++ b/apps/web/src/app/page.tsx
@@ -7,7 +7,7 @@ import { AppShell } from "@/components/AppShell";
 import { LoginPanel } from "@/components/LoginPanel";
 import { PongCanvas } from "@/components/PongCanvas";
 import { StatCard } from "@/components/StatCard";
-import { getLobby, getMe, getToken, sendLobbyChat } from "@/lib/api";
+import { getLobby, getMe, requestWsTicket, sendLobbyChat } from "@/lib/api";
 
 const WS_URL = process.env.NEXT_PUBLIC_WS_URL ?? "ws://localhost:4000/ws";
 
@@ -36,24 +36,38 @@ export default function HomePage() {
   }, [loadLobby]);
 
   useEffect(() => {
-    const token = getToken();
-    if (!userId || !token) return;
-    const socket = new WebSocket(`${WS_URL}?session=${token}`);
-    socketRef.current = socket;
-    socket.onmessage = (event) => {
-      const message = JSON.parse(event.data) as ServerEvent;
-      if (message.type === "chat.message" && message.message.scope === "lobby") {
-        setChat((current) => [...current.filter((item) => item.id !== message.message.id).slice(-19), message.message]);
-      }
-      if (message.type === "presence.changed") {
-        loadLobby().catch(() => setNotice("로비 지표를 갱신하지 못했습니다."));
-      }
-      if (message.type === "error") setNotice(message.message);
-    };
-    socket.onclose = () => {
-      if (socketRef.current === socket) socketRef.current = null;
-    };
+    if (!userId) return;
+    let cancelled = false;
+    let socket: WebSocket | null = null;
+    const controller = new AbortController();
+
+    requestWsTicket(controller.signal)
+      .then(({ ticket, protocolVersion }) => {
+        if (cancelled) return;
+        socket = new WebSocket(`${WS_URL}?ticket=${encodeURIComponent(ticket)}&v=${protocolVersion}`);
+        socketRef.current = socket;
+        socket.onmessage = (event) => {
+          const message = JSON.parse(event.data) as ServerEvent;
+          if (message.type === "chat.message" && message.message.scope === "lobby") {
+            setChat((current) => [...current.filter((item) => item.id !== message.message.id).slice(-19), message.message]);
+          }
+          if (message.type === "presence.changed") {
+            loadLobby().catch(() => setNotice("로비 지표를 갱신하지 못했습니다."));
+          }
+          if (message.type === "error") setNotice(message.message);
+        };
+        socket.onclose = () => {
+          if (socketRef.current === socket) socketRef.current = null;
+        };
+      })
+      .catch(() => {
+        if (!cancelled) setNotice("실시간 연결을 준비하지 못했습니다.");
+      });
+
     return () => {
+      cancelled = true;
+      controller.abort();
+      if (!socket) return;
       socket.onclose = null;
       socket.onmessage = null;
       if (socket.readyState === WebSocket.OPEN || socket.readyState === WebSocket.CONNECTING) socket.close();
diff --git a/apps/web/src/app/play/page.tsx b/apps/web/src/app/play/page.tsx
index fa40368..5c961f2 100644
--- a/apps/web/src/app/play/page.tsx
+++ b/apps/web/src/app/play/page.tsx
@@ -5,7 +5,7 @@ import { MessageCircle, Pause, Play, Send, Signal, Users } from "lucide-react";
 import type { GameSnapshot, ServerEvent } from "@pong-pong/shared";
 import { AppShell } from "@/components/AppShell";
 import { PongCanvas } from "@/components/PongCanvas";
-import { getToken } from "@/lib/api";
+import { requestWsTicket } from "@/lib/api";
 
 const WS_URL = process.env.NEXT_PUBLIC_WS_URL ?? "ws://localhost:4000/ws";
 
@@ -16,6 +16,7 @@ export default function PlayPage() {
   const [messages, setMessages] = useState<string[]>([]);
   const [chatInput, setChatInput] = useState("");
   const socketRef = useRef<WebSocket | null>(null);
+  const ticketRequestRef = useRef<AbortController | null>(null);
   const directionRef = useRef<-1 | 0 | 1>(0);
 
   const score = useMemo(() => (snapshot ? `${snapshot.leftScore} - ${snapshot.rightScore}` : "경기 전"), [snapshot]);
@@ -94,19 +95,29 @@ export default function PlayPage() {
     openGameSocket("토너먼트 경기 상대 입장 대기 중", { type: "tournament.join", matchId });
   }
 
-  function openGameSocket(openStatus: string, payload: Record<string, unknown>) {
-    const token = getToken();
-    if (!token) {
-      setStatus("로그인 후 이용할 수 있습니다.");
-      return;
-    }
+  async function openGameSocket(openStatus: string, payload: Record<string, unknown>) {
     closeCurrentSocket();
     setRoomId(null);
     setSnapshot(null);
     setMessages([]);
     setChatInput("");
     directionRef.current = 0;
-    const socket = new WebSocket(`${WS_URL}?session=${token}`);
+    setStatus("실시간 연결 준비 중");
+    const controller = new AbortController();
+    ticketRequestRef.current = controller;
+    let ticketResponse;
+    try {
+      ticketResponse = await requestWsTicket(controller.signal);
+    } catch (error) {
+      if (!controller.signal.aborted) setStatus("로그인 후 이용할 수 있습니다.");
+      return;
+    } finally {
+      if (ticketRequestRef.current === controller) ticketRequestRef.current = null;
+    }
+    if (controller.signal.aborted) return;
+    const socket = new WebSocket(
+      `${WS_URL}?ticket=${encodeURIComponent(ticketResponse.ticket)}&v=${ticketResponse.protocolVersion}`
+    );
     socketRef.current = socket;
     socket.onopen = () => {
       setStatus(openStatus);
@@ -169,6 +180,8 @@ export default function PlayPage() {
   }
 
   function closeCurrentSocket() {
+    ticketRequestRef.current?.abort();
+    ticketRequestRef.current = null;
     const socket = socketRef.current;
     if (!socket) return;
     socket.onclose = null;
diff --git a/apps/web/src/lib/api.test.ts b/apps/web/src/lib/api.test.ts
index 18c1468..5ef44b8 100644
--- a/apps/web/src/lib/api.test.ts
+++ b/apps/web/src/lib/api.test.ts
@@ -1,18 +1,20 @@
 import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
-import type { SessionUser } from "@pong-pong/shared";
+import { okResponseSchema, type PublicUser, type SessionUser } from "@pong-pong/shared";
 import {
+  ApiError,
+  SESSION_EXPIRED_EVENT,
   apiFetch,
-  clearToken,
   devLogin,
   getLeaderboard,
   getMe,
-  getToken,
-  sendLobbyChat,
-  setToken
+  sendLobbyChat
 } from "./api";
 
-const sessionUser = {
-  id: "user-1",
+const USER_ID = "11111111-1111-4111-8111-111111111111";
+const ITEM_ID = "22222222-2222-4222-8222-222222222222";
+
+const publicUser = {
+  id: USER_ID,
   handle: "tester",
   displayName: "테스터",
   avatarKey: "avatar-1",
@@ -22,35 +24,14 @@ const sessionUser = {
   wins: 3,
   losses: 2,
   online: true,
-  isNpc: false,
+  isNpc: false
+} satisfies PublicUser;
+
+const sessionUser = {
+  ...publicUser,
   email: "tester@example.com"
 } satisfies SessionUser;
 
-function createStorage(): Storage {
-  const values = new Map<string, string>();
-
-  return {
-    get length() {
-      return values.size;
-    },
-    clear() {
-      values.clear();
-    },
-    getItem(key) {
-      return values.get(key) ?? null;
-    },
-    key(index) {
-      return [...values.keys()][index] ?? null;
-    },
-    removeItem(key) {
-      values.delete(key);
-    },
-    setItem(key, value) {
-      values.set(key, value);
-    }
-  };
-}
-
 function jsonResponse(body: unknown, init: ResponseInit = {}): Response {
   return new Response(JSON.stringify(body), {
     status: 200,
@@ -59,34 +40,11 @@ function jsonResponse(body: unknown, init: ResponseInit = {}): Response {
   });
 }
 
-describe("token storage", () => {
-  afterEach(() => {
-    vi.unstubAllGlobals();
-  });
-
-  it("returns null when rendered without a browser window", () => {
-    expect(getToken()).toBeNull();
-  });
-
-  it("stores, reads, and clears the browser token", () => {
-    vi.stubGlobal("window", { localStorage: createStorage() });
-
-    setToken("session-token");
-    expect(getToken()).toBe("session-token");
-
-    clearToken();
-    expect(getToken()).toBeNull();
-  });
-});
-
 describe("apiFetch", () => {
-  let storage: Storage;
   let fetchMock: ReturnType<typeof vi.fn>;
 
   beforeEach(() => {
-    storage = createStorage();
     fetchMock = vi.fn();
-    vi.stubGlobal("window", { localStorage: storage });
     vi.stubGlobal("fetch", fetchMock);
   });
 
@@ -94,44 +52,33 @@ describe("apiFetch", () => {
     vi.unstubAllGlobals();
   });
 
-  it("sends authenticated JSON requests with browser credentials", async () => {
-    storage.setItem("pong-pong-token", "session-token");
+  it("sends cookie-authenticated JSON requests without reading or adding a bearer token", async () => {
+    const getItem = vi.fn(() => "legacy-token");
+    vi.stubGlobal("window", { localStorage: { getItem } });
     fetchMock.mockResolvedValue(jsonResponse({ ok: true }));
 
     await expect(
-      apiFetch<{ ok: boolean }>("/resource", {
+      apiFetch("/resource", okResponseSchema, {
         method: "POST",
         headers: { "x-request-source": "web" },
         body: JSON.stringify({ value: 1 })
       })
     ).resolves.toEqual({ ok: true });
 
-    expect(fetchMock).toHaveBeenCalledOnce();
     const [url, init] = fetchMock.mock.calls[0] as [string, RequestInit];
     const headers = new Headers(init.headers);
     expect(url).toBe("http://localhost:4000/resource");
     expect(init).toMatchObject({ method: "POST", credentials: "include" });
-    expect(headers.get("authorization")).toBe("Bearer session-token");
+    expect(headers.has("authorization")).toBe(false);
     expect(headers.get("content-type")).toBe("application/json");
     expect(headers.get("x-request-source")).toBe("web");
-  });
-
-  it("does not add authentication or content type to a public GET request", async () => {
-    fetchMock.mockResolvedValue(jsonResponse({ ok: true }));
-
-    await apiFetch("/public");
-
-    const [, init] = fetchMock.mock.calls[0] as [string, RequestInit];
-    const headers = new Headers(init.headers);
-    expect(init.credentials).toBe("include");
-    expect(headers.has("authorization")).toBe(false);
-    expect(headers.has("content-type")).toBe(false);
+    expect(getItem).not.toHaveBeenCalled();
   });
 
   it("preserves an explicit content type", async () => {
     fetchMock.mockResolvedValue(jsonResponse({ ok: true }));
 
-    await apiFetch("/form", {
+    await apiFetch("/form", okResponseSchema, {
       method: "POST",
       headers: { "content-type": "text/plain" },
       body: "payload"
@@ -141,35 +88,65 @@ describe("apiFetch", () => {
     expect(new Headers(init.headers).get("content-type")).toBe("text/plain");
   });
 
-  it("throws the response body for a failed request", async () => {
-    fetchMock.mockResolvedValue(new Response("요청이 거절되었습니다.", { status: 403 }));
-
-    await expect(apiFetch("/forbidden")).rejects.toThrow("요청이 거절되었습니다.");
+  it("throws the structured API error returned by the server", async () => {
+    fetchMock.mockResolvedValue(jsonResponse({
+      error: {
+        code: "INVALID_REQUEST",
+        message: "입력값을 확인해주세요.",
+        requestId: "req-123",
+        fieldErrors: { body: ["메시지를 입력해주세요."] }
+      }
+    }, { status: 400 }));
+
+    await expect(apiFetch("/failure", okResponseSchema)).rejects.toMatchObject({
+      name: "ApiError",
+      status: 400,
+      code: "INVALID_REQUEST",
+      message: "입력값을 확인해주세요.",
+      requestId: "req-123",
+      fieldErrors: { body: ["메시지를 입력해주세요."] }
+    });
   });
 
-  it("uses the fallback message when a failed response has no body", async () => {
-    fetchMock.mockResolvedValue(new Response(null, { status: 500 }));
+  it("keeps malformed error responses inside the common ApiError boundary", async () => {
+    fetchMock.mockResolvedValue(new Response("gateway failure", {
+      status: 502,
+      statusText: "Bad Gateway",
+      headers: { "x-request-id": "gateway-1" }
+    }));
 
-    await expect(apiFetch("/failure")).rejects.toThrow("요청을 처리하지 못했습니다.");
+    await expect(apiFetch("/failure", okResponseSchema)).rejects.toEqual(
+      expect.objectContaining({
+        status: 502,
+        code: "HTTP_ERROR",
+        message: "Bad Gateway",
+        requestId: "gateway-1"
+      })
+    );
   });
 
-  it("clears the stored token after an unauthorized response", async () => {
-    storage.setItem("pong-pong-token", "expired-token");
-    fetchMock.mockResolvedValue(new Response("인증이 만료되었습니다.", { status: 401 }));
+  it("signals cookie expiration after a 401 response", async () => {
+    const dispatchEvent = vi.fn();
+    vi.stubGlobal("window", { dispatchEvent });
+    fetchMock.mockResolvedValue(jsonResponse({
+      error: {
+        code: "UNAUTHORIZED",
+        message: "로그인이 필요합니다.",
+        requestId: "req-401"
+      }
+    }, { status: 401 }));
 
-    await expect(apiFetch("/me")).rejects.toThrow("인증이 만료되었습니다.");
-    expect(storage.getItem("pong-pong-token")).toBeNull();
+    await expect(apiFetch("/me", okResponseSchema)).rejects.toBeInstanceOf(ApiError);
+    expect(dispatchEvent).toHaveBeenCalledOnce();
+    expect(dispatchEvent.mock.calls[0][0]).toMatchObject({ type: SESSION_EXPIRED_EVENT });
   });
 });
 
 describe("API endpoint helpers", () => {
-  let storage: Storage;
   let fetchMock: ReturnType<typeof vi.fn>;
 
   beforeEach(() => {
-    storage = createStorage();
     fetchMock = vi.fn();
-    vi.stubGlobal("window", { localStorage: storage });
     vi.stubGlobal("fetch", fetchMock);
   });
 
@@ -177,8 +154,8 @@ describe("API endpoint helpers", () => {
     vi.unstubAllGlobals();
   });
 
-  it("logs in with the current request shape and stores the returned token", async () => {
-    fetchMock.mockResolvedValue(jsonResponse({ user: sessionUser, token: "new-token" }));
+  it("logs in from a token-free user envelope", async () => {
+    fetchMock.mockResolvedValue(jsonResponse({ user: sessionUser }));
 
     await expect(devLogin("tester", "테스터")).resolves.toEqual(sessionUser);
 
@@ -186,16 +163,10 @@ describe("API endpoint helpers", () => {
     expect(url).toBe("http://localhost:4000/auth/dev-login");
     expect(init.method).toBe("POST");
     expect(init.body).toBe(JSON.stringify({ handle: "tester", displayName: "테스터" }));
-    expect(storage.getItem("pong-pong-token")).toBe("new-token");
+    expect(new Headers(init.headers).has("authorization")).toBe(false);
   });
 
-  it("does not request the current user without a stored token", async () => {
-    await expect(getMe()).resolves.toBeNull();
-    expect(fetchMock).not.toHaveBeenCalled();
-  });
-
-  it("returns the current user for a valid stored token", async () => {
-    storage.setItem("pong-pong-token", "session-token");
+  it("always requests the current cookie session", async () => {
     fetchMock.mockResolvedValue(jsonResponse({ user: sessionUser }));
 
     await expect(getMe()).resolves.toEqual(sessionUser);
@@ -205,24 +176,25 @@ describe("API endpoint helpers", () => {
     );
   });
 
-  it("returns null when the current-user request fails", async () => {
-    storage.setItem("pong-pong-token", "expired-token");
-    fetchMock.mockResolvedValue(new Response(null, { status: 401 }));
-
+  it("returns null only when the current cookie session is unauthorized", async () => {
+    fetchMock.mockResolvedValueOnce(jsonResponse({
+      error: { code: "UNAUTHORIZED", message: "로그인이 필요합니다.", requestId: "req-401" }
+    }, { status: 401 }));
     await expect(getMe()).resolves.toBeNull();
+
+    fetchMock.mockResolvedValueOnce(jsonResponse({
+      error: { code: "UNAVAILABLE", message: "잠시 후 다시 시도해주세요.", requestId: "req-503" }
+    }, { status: 503 }));
+    await expect(getMe()).rejects.toMatchObject({ status: 503, code: "UNAVAILABLE" });
   });
 
-  it("extracts endpoint payloads from their response envelopes", async () => {
-    const leaderboardEntry = {
-      rank: 1,
-      user: sessionUser,
-      winRate: 60
-    };
+  it("extracts endpoint payloads from their validated response envelopes", async () => {
+    const leaderboardEntry = { rank: 1, user: publicUser, winRate: 60 };
     const chatMessage = {
-      id: "message-1",
+      id: ITEM_ID,
       scope: "lobby",
       roomId: null,
-      sender: sessionUser,
+      sender: publicUser,
       body: "안녕하세요",
       createdAt: "2026-07-23T00:00:00.000Z"
     };
diff --git a/apps/web/src/lib/api.ts b/apps/web/src/lib/api.ts
index 39cd019..c015301 100644
--- a/apps/web/src/lib/api.ts
+++ b/apps/web/src/lib/api.ts
@@ -1,102 +1,199 @@
-import type { AdminActionSummary, ChatMessage, DashboardSummary, FriendSummary, LeaderboardEntry, LobbyResponse, MatchSummary, PublicUser, SessionUser, TournamentSummary } from "@pong-pong/shared";
+import {
+  adminActionsResponseSchema,
+  adminUsersResponseSchema,
+  apiErrorBodySchema,
+  chatResponseSchema,
+  dashboardSummarySchema,
+  friendResponseSchema,
+  leaderboardResponseSchema,
+  lobbyResponseSchema,
+  profileResponseSchema,
+  publicUserResponseSchema,
+  tournamentResponseSchema,
+  tournamentsResponseSchema,
+  userResponseSchema,
+  wsTicketResponseSchema,
+  type AdminActionSummary,
+  type ApiErrorBody,
+  type ChatMessage,
+  type DashboardSummary,
+  type FriendSummary,
+  type LeaderboardEntry,
+  type LobbyResponse,
+  type MatchSummary,
+  type PublicUser,
+  type SessionUser,
+  type TournamentSummary,
+  type WsTicketResponse
+} from "@pong-pong/shared";
 
 const API_BASE = process.env.NEXT_PUBLIC_API_BASE_URL ?? "http://localhost:4000";
 
-export function getToken(): string | null {
-  if (typeof window === "undefined") return null;
-  return window.localStorage.getItem("pong-pong-token");
-}
+export const SESSION_EXPIRED_EVENT = "pong-pong:session-expired";
 
-export function setToken(token: string): void {
-  window.localStorage.setItem("pong-pong-token", token);
-}
+type FieldErrors = ApiErrorBody["error"]["fieldErrors"];
+
+type ResponseSchema<T> = {
+  parse(value: unknown): T;
+};
 
-export function clearToken(): void {
-  window.localStorage.removeItem("pong-pong-token");
+export class ApiError extends Error {
+  override readonly name = "ApiError";
+
+  constructor(
+    readonly status: number,
+    readonly code: string,
+    message: string,
+    readonly requestId: string,
+    readonly fieldErrors?: FieldErrors
+  ) {
+    super(message);
+  }
 }
 
-export async function apiFetch<T>(path: string, init: RequestInit = {}): Promise<T> {
-  const token = getToken();
+export async function apiFetch<T>(
+  path: string,
+  schema: ResponseSchema<T>,
+  init: RequestInit = {}
+): Promise<T> {
   const headers = new Headers(init.headers);
   if (init.body && !headers.has("content-type")) {
     headers.set("content-type", "application/json");
   }
-  if (token) {
-    headers.set("authorization", `Bearer ${token}`);
-  }
   const response = await fetch(`${API_BASE}${path}`, {
     ...init,
     credentials: "include",
     headers
   });
   if (!response.ok) {
-    if (response.status === 401) clearToken();
-    throw new Error((await response.text()) || "요청을 처리하지 못했습니다.");
+    const error = await responseError(response);
+    if (response.status === 401) signalSessionExpired();
+    throw error;
   }
-  return response.json() as Promise<T>;
+  return schema.parse(await response.json());
 }
 
-export async function devLogin(handle: string, displayName: string): Promise<SessionUser> {
-  const result = await apiFetch<{ user: SessionUser; token: string }>("/auth/dev-login", {
+export async function devLogin(
+  handle: string,
+  displayName: string,
+  signal?: AbortSignal
+): Promise<SessionUser> {
+  const result = await apiFetch("/auth/dev-login", userResponseSchema, {
     method: "POST",
-    body: JSON.stringify({ handle, displayName })
+    body: JSON.stringify({ handle, displayName }),
+    signal
   });
-  setToken(result.token);
   return result.user;
 }
 
-export async function getMe(): Promise<SessionUser | null> {
-  if (!getToken()) return null;
+export async function getMe(signal?: AbortSignal): Promise<SessionUser | null> {
   try {
-    return (await apiFetch<{ user: SessionUser }>("/me")).user;
-  } catch {
-    return null;
+    return (await apiFetch("/me", userResponseSchema, { signal })).user;
+  } catch (error) {
+    if (error instanceof ApiError && error.status === 401) return null;
+    throw error;
   }
 }
 
-export async function getLobby(): Promise<LobbyResponse> {
-  return await apiFetch("/lobby");
+export async function getLobby(signal?: AbortSignal): Promise<LobbyResponse> {
+  return apiFetch("/lobby", lobbyResponseSchema, { signal });
 }
 
-export async function sendLobbyChat(body: string): Promise<ChatMessage> {
-  return (await apiFetch<{ message: ChatMessage }>("/chat/lobby", { method: "POST", body: JSON.stringify({ body }) })).message;
+export async function sendLobbyChat(body: string, signal?: AbortSignal): Promise<ChatMessage> {
+  return (await apiFetch("/chat/lobby", chatResponseSchema, {
+    method: "POST",
+    body: JSON.stringify({ body }),
+    signal
+  })).message;
 }
 
-export async function getDashboard(): Promise<DashboardSummary> {
-  return await apiFetch("/dashboard");
+export async function getDashboard(signal?: AbortSignal): Promise<DashboardSummary> {
+  return apiFetch("/dashboard", dashboardSummarySchema, { signal });
 }
 
-export async function getLeaderboard(): Promise<LeaderboardEntry[]> {
-  return (await apiFetch<{ entries: LeaderboardEntry[] }>("/leaderboard")).entries;
+export async function getLeaderboard(signal?: AbortSignal): Promise<LeaderboardEntry[]> {
+  return (await apiFetch("/leaderboard", leaderboardResponseSchema, { signal })).entries;
 }
 
-export async function getTournaments(): Promise<TournamentSummary[]> {
-  return (await apiFetch<{ tournaments: TournamentSummary[] }>("/tournaments")).tournaments;
+export async function getTournaments(signal?: AbortSignal): Promise<TournamentSummary[]> {
+  return (await apiFetch("/tournaments", tournamentsResponseSchema, { signal })).tournaments;
 }
 
-export async function createTournament(name: string): Promise<TournamentSummary> {
-  return (await apiFetch<{ tournament: TournamentSummary }>("/tournaments", { method: "POST", body: JSON.stringify({ name }) })).tournament;
+export async function createTournament(name: string, signal?: AbortSignal): Promise<TournamentSummary> {
+  return (await apiFetch("/tournaments", tournamentResponseSchema, {
+    method: "POST",
+    body: JSON.stringify({ name }),
+    signal
+  })).tournament;
 }
 
-export async function joinTournament(id: string): Promise<TournamentSummary> {
-  return (await apiFetch<{ tournament: TournamentSummary }>(`/tournaments/${id}/join`, { method: "POST" })).tournament;
+export async function joinTournament(id: string, signal?: AbortSignal): Promise<TournamentSummary> {
+  return (await apiFetch(`/tournaments/${id}/join`, tournamentResponseSchema, {
+    method: "POST",
+    signal
+  })).tournament;
 }
 
-export async function getProfile(handle: string): Promise<{ user: PublicUser; recentMatches: MatchSummary[] }> {
-  return apiFetch(`/profile/${handle}`);
+export async function getProfile(
+  handle: string,
+  signal?: AbortSignal
+): Promise<{ user: PublicUser; recentMatches: MatchSummary[] }> {
+  return apiFetch(`/profile/${handle}`, profileResponseSchema, { signal });
 }
 
-export async function requestFriend(handle: string): Promise<FriendSummary> {
-  return (await apiFetch<{ friend: FriendSummary }>("/friends/request", { method: "POST", body: JSON.stringify({ handle }) })).friend;
+export async function requestFriend(handle: string, signal?: AbortSignal): Promise<FriendSummary> {
+  return (await apiFetch("/friends/request", friendResponseSchema, {
+    method: "POST",
+    body: JSON.stringify({ handle }),
+    signal
+  })).friend;
 }
 
-export async function getAdminActions(): Promise<AdminActionSummary[]> {
-  return (await apiFetch<{ actions: AdminActionSummary[] }>("/admin/actions")).actions;
+export async function getAdminUsers(signal?: AbortSignal): Promise<PublicUser[]> {
+  return (await apiFetch("/admin/users", adminUsersResponseSchema, { signal })).users;
 }
 
-export async function setUserStatus(id: string, status: "active" | "banned", reason: string): Promise<PublicUser> {
-  return (await apiFetch<{ user: PublicUser }>(`/admin/users/${id}/status`, {
+export async function getAdminActions(signal?: AbortSignal): Promise<AdminActionSummary[]> {
+  return (await apiFetch("/admin/actions", adminActionsResponseSchema, { signal })).actions;
+}
+
+export async function setUserStatus(
+  id: string,
+  status: "active" | "banned",
+  reason: string,
+  signal?: AbortSignal
+): Promise<PublicUser> {
+  return (await apiFetch(`/admin/users/${id}/status`, publicUserResponseSchema, {
     method: "PATCH",
-    body: JSON.stringify({ status, reason })
+    body: JSON.stringify({ status, reason }),
+    signal
   })).user;
 }
+
+export async function requestWsTicket(signal?: AbortSignal): Promise<WsTicketResponse> {
+  return apiFetch("/auth/ws-ticket", wsTicketResponseSchema, { method: "POST", signal });
+}
+
+async function responseError(response: Response): Promise<ApiError> {
+  try {
+    const parsed = apiErrorBodySchema.safeParse(await response.json());
+    if (parsed.success) {
+      const { code, message, requestId, fieldErrors } = parsed.data.error;
+      return new ApiError(response.status, code, message, requestId, fieldErrors);
+    }
+  } catch {
+    // The fallback below keeps network-facing failures typed even if the server response is malformed.
+  }
+
+  return new ApiError(
+    response.status,
+    "HTTP_ERROR",
+    response.statusText || "요청을 처리하지 못했습니다.",
+    response.headers.get("x-request-id") ?? "unknown"
+  );
+}
+
+function signalSessionExpired(): void {
+  if (typeof window === "undefined" || typeof window.dispatchEvent !== "function") return;
+  window.dispatchEvent(new Event(SESSION_EXPIRED_EVENT));
+}


