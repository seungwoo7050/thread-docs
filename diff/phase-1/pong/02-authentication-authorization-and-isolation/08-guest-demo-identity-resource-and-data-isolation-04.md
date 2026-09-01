## `test(guest): 위조 client address 거부`

diff --git a/apps/api/src/guest-demo.test.ts b/apps/api/src/guest-demo.test.ts
index b59147e..80837ef 100644
--- a/apps/api/src/guest-demo.test.ts
+++ b/apps/api/src/guest-demo.test.ts
@@ -87,6 +87,73 @@ describe("guest demo HTTP boundary", () => {
     expectApiError(limited, 429, "guest_creation_rate_limited");
   });
 
+  it("applies guest creation limits to the forwarded client IP behind the trusted proxy", async () => {
+    const proxied = buildApp({
+      repo,
+      webOrigin: "http://localhost:3000",
+      appMode: "demo",
+      trustProxy: true,
+      guestAccess: new GuestAccess({
+        secret: "guest-demo-test-secret-that-is-long-enough",
+        creationLimitPerMinute: 1
+      })
+    });
+    await proxied.ready();
+    try {
+      const first = await proxied.inject({
+        method: "POST",
+        url: "/auth/guest",
+        headers: { "x-forwarded-for": "203.0.113.41" }
+      });
+      const otherClient = await proxied.inject({
+        method: "POST",
+        url: "/auth/guest",
+        headers: { "x-forwarded-for": "203.0.113.42" }
+      });
+      const repeated = await proxied.inject({
+        method: "POST",
+        url: "/auth/guest",
+        headers: { "x-forwarded-for": "203.0.113.41" }
+      });
+
+      expect(first.statusCode).toBe(200);
+      expect(otherClient.statusCode).toBe(200);
+      expectApiError(repeated, 429, "guest_creation_rate_limited");
+    } finally {
+      await proxied.close();
+    }
+  });
+
+  it("ignores forwarded client addresses when no trusted proxy is configured", async () => {
+    const direct = buildApp({
+      repo,
+      webOrigin: "http://localhost:3000",
+      appMode: "demo",
+      guestAccess: new GuestAccess({
+        secret: "guest-demo-test-secret-that-is-long-enough",
+        creationLimitPerMinute: 1
+      })
+    });
+    await direct.ready();
+    try {
+      const first = await direct.inject({
+        method: "POST",
+        url: "/auth/guest",
+        headers: { "x-forwarded-for": "203.0.113.51" }
+      });
+      const spoofed = await direct.inject({
+        method: "POST",
+        url: "/auth/guest",
+        headers: { "x-forwarded-for": "203.0.113.52" }
+      });
+
+      expect(first.statusCode).toBe(200);
+      expectApiError(spoofed, 429, "guest_creation_rate_limited");
+    } finally {
+      await direct.close();
+    }
+  });
+
   it("does not expose guest login outside demo mode", async () => {
     const developmentApp = buildApp({ repo, webOrigin: "http://localhost:3000", appMode: "test" });
     await developmentApp.ready();
@@ -122,6 +189,36 @@ describe("guest demo HTTP boundary", () => {
     expect(createTournament).not.toHaveBeenCalled();
   });
 
+  it("does not expose registered data through public demo reads", async () => {
+    const listChat = vi.spyOn(repo, "listLobbyChat");
+    const listMatches = vi.spyOn(repo, "listRecentMatches");
+    const listLeaderboard = vi.spyOn(repo, "listLeaderboard");
+    const getProfile = vi.spyOn(repo, "getUserByHandle");
+    const listTournaments = vi.spyOn(repo, "listTournaments");
+
+    const lobby = await app.inject({ method: "GET", url: "/lobby" });
+    expect(lobby.statusCode).toBe(200);
+    expect(lobby.json()).toMatchObject({
+      me: null,
+      onlinePlayers: [],
+      recentMatches: [],
+      chat: []
+    });
+
+    const hidden = await Promise.all([
+      app.inject({ method: "GET", url: "/leaderboard" }),
+      app.inject({ method: "GET", url: "/profile/registered-user" }),
+      app.inject({ method: "GET", url: "/tournaments" })
+    ]);
+    for (const response of hidden) expectApiError(response, 404, "not_found");
+
+    expect(listChat).not.toHaveBeenCalled();
+    expect(listMatches).not.toHaveBeenCalled();
+    expect(listLeaderboard).not.toHaveBeenCalled();
+    expect(getProfile).not.toHaveBeenCalled();
+    expect(listTournaments).not.toHaveBeenCalled();
+  });
+
   it("issues a one-time websocket ticket from the guest cookie without database storage", async () => {
     const address = await app.listen({ host: "127.0.0.1", port: 0 });
     const wsBaseUrl = address.replace(/^http/, "ws");


## `feat(web): guest login API와 middleware 연결`

diff --git a/apps/web/src/lib/api.ts b/apps/web/src/lib/api.ts
index c015301..57ac0d0 100644
--- a/apps/web/src/lib/api.ts
+++ b/apps/web/src/lib/api.ts
@@ -5,6 +5,7 @@ import {
   chatResponseSchema,
   dashboardSummarySchema,
   friendResponseSchema,
+  guestAuthResponseSchema,
   leaderboardResponseSchema,
   lobbyResponseSchema,
   profileResponseSchema,
@@ -18,6 +19,7 @@ import {
   type ChatMessage,
   type DashboardSummary,
   type FriendSummary,
+  type GuestAuthResponse,
   type LeaderboardEntry,
   type LobbyResponse,
   type MatchSummary,
@@ -86,6 +88,13 @@ export async function devLogin(
   return result.user;
 }
 
+export async function guestLogin(signal?: AbortSignal): Promise<GuestAuthResponse> {
+  return apiFetch("/auth/guest", guestAuthResponseSchema, {
+    method: "POST",
+    signal
+  });
+}
+
 export async function getMe(signal?: AbortSignal): Promise<SessionUser | null> {
   try {
     return (await apiFetch("/me", userResponseSchema, { signal })).user;
diff --git a/apps/web/src/middleware.ts b/apps/web/src/middleware.ts
new file mode 100644
index 0000000..6f15dce
--- /dev/null
+++ b/apps/web/src/middleware.ts
@@ -0,0 +1,19 @@
+import { NextResponse, type NextRequest } from "next/server";
+import { isDemoMode, isDemoRestrictedPath } from "./lib/demoPolicy";
+
+export function middleware(request: NextRequest) {
+  if (isDemoMode() && isDemoRestrictedPath(request.nextUrl.pathname)) {
+    return new NextResponse("Not Found", { status: 404 });
+  }
+  return NextResponse.next();
+}
+
+export const config = {
+  matcher: [
+    "/dashboard/:path*",
+    "/leaderboard/:path*",
+    "/tournaments/:path*",
+    "/profile/:path*",
+    "/admin/:path*"
+  ]
+};


## `feat(web): guest play presentation 적용`

diff --git a/apps/web/src/app/play/page.tsx b/apps/web/src/app/play/page.tsx
index 12a6baf..389087b 100644
--- a/apps/web/src/app/play/page.tsx
+++ b/apps/web/src/app/play/page.tsx
@@ -3,11 +3,13 @@
 import { useCallback, useEffect, useMemo, useRef, useState } from "react";
 import { ArrowDown, ArrowUp, MessageCircle, Pause, Play, Send, Signal, Users } from "lucide-react";
 import { AppShell } from "@/components/AppShell";
+import { demoLobbyPresentation, isDemoMode } from "@/lib/demoPolicy";
 import { PongCanvas } from "@/components/PongCanvas";
 import { directionForKey, isEditableTarget } from "@/game/gameInput";
 import { useGameConnection } from "@/game/useGameConnection";
 
 export default function PlayPage() {
+  const demoMode = isDemoMode();
   const {
     state,
     connectQueue,
@@ -199,7 +201,7 @@ export default function PlayPage() {
             <p className="mt-4 text-2xl font-black text-ink">{opponentName}</p>
             <p className="mt-2 text-sm font-semibold text-muted">{opponent?.ai ? "AI 상대입니다. 서버 경기 장면 기준으로 상태가 갱신됩니다." : "서버 경기 장면 기준으로 상태가 갱신됩니다."}</p>
           </div>
-          <div className="card p-5">
+          {!demoMode || demoLobbyPresentation.showMatchChat ? <div className="card p-5">
             <h2 className="flex items-center gap-2 text-lg font-black text-ink">
               <MessageCircle size={20} /> 매치 채팅
             </h2>
@@ -225,7 +227,7 @@ export default function PlayPage() {
                 <Send size={18} />
               </button>
             </form>
-          </div>
+          </div> : null}
         </aside>
       </div>
     </AppShell>


## `test(guest): 체험 기능 오용 방지 검증`

diff --git a/apps/api/src/env.test.ts b/apps/api/src/env.test.ts
index 6bb6586..dff081c 100644
--- a/apps/api/src/env.test.ts
+++ b/apps/api/src/env.test.ts
@@ -22,4 +22,20 @@ describe("readEnv", () => {
     expect(env.databaseUrl).toBeNull();
     expect(env.webOrigin).toBe("http://localhost:3000");
   });
+
+  it("requires an explicit strong session secret in demo and production modes", () => {
+    expect(() => readEnv({ APP_MODE: "demo" })).toThrow("SESSION_SECRET");
+    expect(() => readEnv({ NODE_ENV: "production" })).toThrow("SESSION_SECRET");
+    expect(() => readEnv({ APP_MODE: "demo", SESSION_SECRET: "too-short" })).toThrow("SESSION_SECRET");
+
+    expect(readEnv({
+      APP_MODE: "demo",
+      SESSION_SECRET: "0123456789abcdef0123456789abcdef"
+    })).toMatchObject({ appMode: "demo", trustProxy: false });
+  });
+
+  it("enables proxy address parsing only when explicitly configured", () => {
+    expect(readEnv({ TRUST_PROXY: "1" }).trustProxy).toBe(true);
+    expect(readEnv({ TRUST_PROXY: "0" }).trustProxy).toBe(false);
+  });
 });
diff --git a/apps/api/src/gameHub.guest.test.ts b/apps/api/src/gameHub.guest.test.ts
index 478a273..421c51e 100644
--- a/apps/api/src/gameHub.guest.test.ts
+++ b/apps/api/src/gameHub.guest.test.ts
@@ -117,6 +117,7 @@ describe("GameHub guest isolation", () => {
         ratingDelta: 0
       }
     });
+    expect(hub.retainedGuestResultCount).toBe(2);
 
     const recovered = connect(hub, leftUser);
     await flushEvents();
@@ -124,6 +125,7 @@ describe("GameHub guest isolation", () => {
     recovered.terminate();
 
     await vi.advanceTimersByTimeAsync(120_001);
+    expect(hub.retainedGuestResultCount).toBe(0);
     const expired = connect(hub, leftUser);
     await flushEvents();
     expect(expired.latest("game.finished")).toBeUndefined();
diff --git a/apps/api/src/guest-demo.test.ts b/apps/api/src/guest-demo.test.ts
index 80837ef..186940b 100644
--- a/apps/api/src/guest-demo.test.ts
+++ b/apps/api/src/guest-demo.test.ts
@@ -124,36 +124,6 @@ describe("guest demo HTTP boundary", () => {
     }
   });
 
-  it("ignores forwarded client addresses when no trusted proxy is configured", async () => {
-    const direct = buildApp({
-      repo,
-      webOrigin: "http://localhost:3000",
-      appMode: "demo",
-      guestAccess: new GuestAccess({
-        secret: "guest-demo-test-secret-that-is-long-enough",
-        creationLimitPerMinute: 1
-      })
-    });
-    await direct.ready();
-    try {
-      const first = await direct.inject({
-        method: "POST",
-        url: "/auth/guest",
-        headers: { "x-forwarded-for": "203.0.113.51" }
-      });
-      const spoofed = await direct.inject({
-        method: "POST",
-        url: "/auth/guest",
-        headers: { "x-forwarded-for": "203.0.113.52" }
-      });
-
-      expect(first.statusCode).toBe(200);
-      expectApiError(spoofed, 429, "guest_creation_rate_limited");
-    } finally {
-      await direct.close();
-    }
-  });
-
   it("does not expose guest login outside demo mode", async () => {
     const developmentApp = buildApp({ repo, webOrigin: "http://localhost:3000", appMode: "test" });
     await developmentApp.ready();
diff --git a/apps/api/src/guestAccess.test.ts b/apps/api/src/guestAccess.test.ts
index 436aa56..6a921bd 100644
--- a/apps/api/src/guestAccess.test.ts
+++ b/apps/api/src/guestAccess.test.ts
@@ -63,6 +63,31 @@ describe("GuestAccess", () => {
     expect(access.consumeWsTicket(hashWsTicket(expired))).toBeNull();
   });
 
+  it("keeps one outstanding ticket per guest and bounds the process ticket store", () => {
+    let nowMs = 3_000_000;
+    const access = createAccess({ clock: () => nowMs, ticketLimit: 2 });
+    const firstGuest = access.createSession("203.0.113.31").user;
+    const secondGuest = access.createSession("203.0.113.32").user;
+    const thirdGuest = access.createSession("203.0.113.33").user;
+
+    const replaced = access.issueWsTicket(firstGuest);
+    const replacement = access.issueWsTicket(firstGuest);
+    expect(access.activeTicketCount).toBe(1);
+    expect(access.consumeWsTicket(hashWsTicket(replaced))).toBeNull();
+
+    access.issueWsTicket(firstGuest);
+    access.issueWsTicket(secondGuest);
+    expect(access.activeTicketCount).toBe(2);
+    expect(() => access.issueWsTicket(thirdGuest)).toThrowError(
+      expect.objectContaining<Partial<GuestAccessError>>({ code: "guest_ticket_limit_reached" })
+    );
+
+    nowMs += 30_000;
+    expect(access.issueWsTicket(thirdGuest)).toMatch(/^[A-Za-z0-9_-]{43}$/);
+    expect(access.activeTicketCount).toBe(1);
+    expect(access.consumeWsTicket(hashWsTicket(replacement))).toBeNull();
+  });
+
   it("limits guest sockets per IP and across the process", () => {
     expect(DEFAULT_GUEST_CONNECTIONS_PER_IP).toBe(4);
     expect(DEFAULT_GUEST_CONNECTION_LIMIT).toBe(200);
@@ -102,6 +127,23 @@ describe("GuestAccess", () => {
     replacement?.release();
     expect(access.activeConnectionCount).toBe(0);
   });
+
+  it("does not move a replacement connection into an IP that is already full", () => {
+    const access = createAccess({ connectionsPerIp: 1, connectionLimit: 3 });
+    const first = access.createSession("192.0.2.20").user;
+    const second = access.createSession("192.0.2.21").user;
+    const firstLease = access.acquireConnection("192.0.2.20", first.id);
+    const secondLease = access.acquireConnection("192.0.2.21", second.id);
+
+    expect(firstLease).not.toBeNull();
+    expect(secondLease).not.toBeNull();
+    expect(access.acquireConnection("192.0.2.20", second.id)).toBeNull();
+    expect(access.activeConnectionCount).toBe(2);
+
+    firstLease?.release();
+    expect(access.acquireConnection("192.0.2.20", second.id)).not.toBeNull();
+    secondLease?.release();
+  });
 });
 
 function createAccess(overrides: Partial<ConstructorParameters<typeof GuestAccess>[0]> = {}): GuestAccess {
diff --git a/apps/web/src/lib/demoPolicy.test.ts b/apps/web/src/lib/demoPolicy.test.ts
new file mode 100644
index 0000000..64dd734
--- /dev/null
+++ b/apps/web/src/lib/demoPolicy.test.ts
@@ -0,0 +1,48 @@
+import { describe, expect, it } from "vitest";
+import {
+  createNavigation,
+  demoLobbyPresentation,
+  isDemoRestrictedPath
+} from "./demoPolicy";
+
+describe("guest demo presentation policy", () => {
+  it("keeps only lobby and play navigation in demo mode", () => {
+    expect(createNavigation(true, "/profile/guest-1").map((item) => item.id)).toEqual([
+      "lobby",
+      "play"
+    ]);
+    expect(createNavigation(false, "/profile/player-1").map((item) => item.id)).toEqual([
+      "lobby",
+      "play",
+      "dashboard",
+      "leaderboard",
+      "tournaments",
+      "profile",
+      "admin"
+    ]);
+  });
+
+  it("hides persisted progress, ranking links, and chat from the guest lobby", () => {
+    expect(demoLobbyPresentation).toMatchObject({
+      showPersistedProgress: false,
+      showLeaderboardLink: false,
+      showLobbyChat: false,
+      showMatchChat: false
+    });
+    expect(demoLobbyPresentation.description).not.toMatch(/전적|순위.*갱신|저장/);
+  });
+
+  it("blocks direct navigation to registered-account pages in demo mode", () => {
+    for (const path of [
+      "/dashboard",
+      "/leaderboard",
+      "/tournaments",
+      "/profile/registered-user",
+      "/admin/users"
+    ]) {
+      expect(isDemoRestrictedPath(path)).toBe(true);
+    }
+    expect(isDemoRestrictedPath("/")).toBe(false);
+    expect(isDemoRestrictedPath("/play")).toBe(false);
+  });
+});


