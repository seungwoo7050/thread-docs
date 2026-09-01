## `fix(guest): 체험 환경의 runtime 복구 제한`

diff --git a/apps/api/src/app.ts b/apps/api/src/app.ts
index a65c4c5..06c61c8 100644
--- a/apps/api/src/app.ts
+++ b/apps/api/src/app.ts
@@ -24,6 +24,7 @@ import {
   type GuestSessionUser
 } from "./guestAccess.js";
 import { createLoggerOptions } from "./requestLogging.js";
+import { readAppMode } from "./env.js";
 import { createRawWsTicket, hashWsTicket, WS_TICKET_TTL_SECONDS } from "./wsTicket.js";
 
 const WS_POLICY_VIOLATION = 1008;
@@ -203,7 +204,7 @@ export function buildApp({
     let ticket: string;
     try {
       ticket = isGuestSession(user) && guests
-        ? guests.issueWsTicket(user)
+        ? guests.issueWsTicket(user, request.ip)
         : createRawWsTicket();
     } catch (error) {
       if (error instanceof GuestAccessError) {
@@ -436,13 +437,6 @@ function requireRegistered(user: SessionUser | GuestSessionUser): void {
   }
 }
 
-function readAppMode(input = process.env): AppMode {
-  if (input.APP_MODE === "demo") return "demo";
-  if (input.NODE_ENV === "production") return "production";
-  if (input.NODE_ENV === "test") return "test";
-  return "development";
-}
-
 function useSecureCookies(mode: AppMode): boolean {
   return mode === "production" || mode === "demo";
 }
diff --git a/apps/api/src/env.ts b/apps/api/src/env.ts
index e54b1ba..2308263 100644
--- a/apps/api/src/env.ts
+++ b/apps/api/src/env.ts
@@ -26,8 +26,13 @@ export function readEnv(input = process.env): ApiEnv {
   };
 }
 
-function readAppMode(input: NodeJS.ProcessEnv): ApiEnv["appMode"] {
-  if (input.APP_MODE === "demo") return "demo";
+export function readAppMode(input: NodeJS.ProcessEnv = process.env): ApiEnv["appMode"] {
+  if (input.APP_MODE !== undefined) {
+    if (["development", "test", "production", "demo"].includes(input.APP_MODE)) {
+      return input.APP_MODE as ApiEnv["appMode"];
+    }
+    throw new Error(`APP_MODE must be development, test, production, or demo: ${input.APP_MODE}`);
+  }
   if (input.NODE_ENV === "production") return "production";
   if (input.NODE_ENV === "test") return "test";
   return "development";
diff --git a/apps/api/src/guestAccess.test.ts b/apps/api/src/guestAccess.test.ts
index 6a921bd..95afec2 100644
--- a/apps/api/src/guestAccess.test.ts
+++ b/apps/api/src/guestAccess.test.ts
@@ -52,13 +52,13 @@ describe("GuestAccess", () => {
     let nowMs = 2_000_000;
     const access = createAccess({ clock: () => nowMs });
     const guest = access.createSession("203.0.113.30").user;
-    const ticket = access.issueWsTicket(guest);
+    const ticket = access.issueWsTicket(guest, "203.0.113.30");
 
     expect(ticket).toMatch(/^[A-Za-z0-9_-]{43}$/);
     expect(access.consumeWsTicket(hashWsTicket(ticket))).toEqual(guest);
     expect(access.consumeWsTicket(hashWsTicket(ticket))).toBeNull();
 
-    const expired = access.issueWsTicket(guest);
+    const expired = access.issueWsTicket(guest, "203.0.113.30");
     nowMs += 30_000;
     expect(access.consumeWsTicket(hashWsTicket(expired))).toBeNull();
   });
@@ -70,20 +70,20 @@ describe("GuestAccess", () => {
     const secondGuest = access.createSession("203.0.113.32").user;
     const thirdGuest = access.createSession("203.0.113.33").user;
 
-    const replaced = access.issueWsTicket(firstGuest);
-    const replacement = access.issueWsTicket(firstGuest);
+    const replaced = access.issueWsTicket(firstGuest, "203.0.113.31");
+    const replacement = access.issueWsTicket(firstGuest, "203.0.113.31");
     expect(access.activeTicketCount).toBe(1);
     expect(access.consumeWsTicket(hashWsTicket(replaced))).toBeNull();
 
-    access.issueWsTicket(firstGuest);
-    access.issueWsTicket(secondGuest);
+    access.issueWsTicket(firstGuest, "203.0.113.31");
+    access.issueWsTicket(secondGuest, "203.0.113.32");
     expect(access.activeTicketCount).toBe(2);
-    expect(() => access.issueWsTicket(thirdGuest)).toThrowError(
+    expect(() => access.issueWsTicket(thirdGuest, "203.0.113.33")).toThrowError(
       expect.objectContaining<Partial<GuestAccessError>>({ code: "guest_ticket_limit_reached" })
     );
 
     nowMs += 30_000;
-    expect(access.issueWsTicket(thirdGuest)).toMatch(/^[A-Za-z0-9_-]{43}$/);
+    expect(access.issueWsTicket(thirdGuest, "203.0.113.33")).toMatch(/^[A-Za-z0-9_-]{43}$/);
     expect(access.activeTicketCount).toBe(1);
     expect(access.consumeWsTicket(hashWsTicket(replacement))).toBeNull();
   });
diff --git a/apps/api/src/guestAccess.ts b/apps/api/src/guestAccess.ts
index e7de311..289f36e 100644
--- a/apps/api/src/guestAccess.ts
+++ b/apps/api/src/guestAccess.ts
@@ -13,6 +13,9 @@ export const DEFAULT_GUEST_CREATION_LIMIT_PER_MINUTE = 10;
 export const DEFAULT_GUEST_CONNECTIONS_PER_IP = 4;
 export const DEFAULT_GUEST_CONNECTION_LIMIT = 200;
 export const DEFAULT_GUEST_TICKET_LIMIT = 400;
+export const DEFAULT_GUEST_TRACKED_IP_LIMIT = 10_000;
+export const DEFAULT_GUEST_TICKETS_PER_IP = 4;
+export const DEFAULT_GUEST_TICKET_ISSUE_LIMIT_PER_MINUTE = 30;
 
 const CREATION_WINDOW_MS = 60_000;
 
@@ -34,6 +37,9 @@ type GuestAccessOptions = {
   connectionsPerIp?: number;
   connectionLimit?: number;
   ticketLimit?: number;
+  trackedIpLimit?: number;
+  ticketsPerIp?: number;
+  ticketIssueLimitPerMinute?: number;
 };
 
 type ConnectionLease = {
@@ -42,7 +48,12 @@ type ConnectionLease = {
 
 export class GuestAccessError extends Error {
   constructor(
-    readonly code: "guest_creation_rate_limited" | "guest_ticket_limit_reached",
+    readonly code:
+      | "guest_creation_rate_limited"
+      | "guest_creation_capacity_reached"
+      | "guest_ticket_limit_reached"
+      | "guest_ticket_ip_limit_reached"
+      | "guest_ticket_rate_limited",
     message: string
   ) {
     super(message);
@@ -56,9 +67,14 @@ export class GuestAccess {
   private readonly connectionsPerIp: number;
   private readonly connectionLimit: number;
   private readonly ticketLimit: number;
-  private readonly creationsByIp = new Map<string, number[]>();
+  private readonly trackedIpLimit: number;
+  private readonly ticketsPerIp: number;
+  private readonly ticketIssueLimitPerMinute: number;
+  private readonly creationsByIp = new Map<string, RollingWindow>();
+  private readonly ticketIssuesByIp = new Map<string, RollingWindow>();
   private readonly tickets = new Map<string, {
     user: GuestSessionUser;
+    ip: string;
     expiresAtMs: number;
     cleanupTimer: NodeJS.Timeout;
   }>();
@@ -74,6 +90,10 @@ export class GuestAccess {
     this.connectionsPerIp = options.connectionsPerIp ?? DEFAULT_GUEST_CONNECTIONS_PER_IP;
     this.connectionLimit = options.connectionLimit ?? DEFAULT_GUEST_CONNECTION_LIMIT;
     this.ticketLimit = options.ticketLimit ?? DEFAULT_GUEST_TICKET_LIMIT;
+    this.trackedIpLimit = options.trackedIpLimit ?? DEFAULT_GUEST_TRACKED_IP_LIMIT;
+    this.ticketsPerIp = options.ticketsPerIp ?? DEFAULT_GUEST_TICKETS_PER_IP;
+    this.ticketIssueLimitPerMinute = options.ticketIssueLimitPerMinute
+      ?? DEFAULT_GUEST_TICKET_ISSUE_LIMIT_PER_MINUTE;
   }
 
   get activeConnectionCount(): number {
@@ -81,10 +101,13 @@ export class GuestAccess {
   }
 
   get activeTicketCount(): number {
-    this.pruneExpiredTickets();
     return this.tickets.size;
   }
 
+  get trackedCreationIpCount(): number {
+    return this.creationsByIp.size;
+  }
+
   createSession(ip: string): {
     user: GuestSessionUser;
     cookieValue: string;
@@ -148,14 +171,19 @@ export class GuestAccess {
     }
   }
 
-  issueWsTicket(user: GuestSessionUser): string {
+  issueWsTicket(user: GuestSessionUser, ip: string): string {
     this.pruneExpiredTickets();
+    this.recordTicketIssue(ip);
     const previousHash = this.ticketHashByGuest.get(user.id);
     if (previousHash) {
-      const previous = this.tickets.get(previousHash);
-      if (previous) clearTimeout(previous.cleanupTimer);
-      this.tickets.delete(previousHash);
-      this.ticketHashByGuest.delete(user.id);
+      this.deleteTicket(previousHash);
+    }
+    const pendingForIp = [...this.tickets.values()].filter((ticket) => ticket.ip === ip).length;
+    if (pendingForIp >= this.ticketsPerIp) {
+      throw new GuestAccessError(
+        "guest_ticket_ip_limit_reached",
+        "이 네트워크의 게스트 연결 요청이 많습니다. 잠시 후 다시 시도해주세요."
+      );
     }
     if (this.tickets.size >= this.ticketLimit) {
       throw new GuestAccessError(
@@ -170,6 +198,7 @@ export class GuestAccess {
     cleanupTimer.unref();
     this.tickets.set(ticketHash, {
       user,
+      ip,
       expiresAtMs,
       cleanupTimer
     });
@@ -210,13 +239,27 @@ export class GuestAccess {
   }
 
   private recordCreation(ip: string): void {
-    const cutoff = this.clock() - CREATION_WINDOW_MS;
-    const recent = (this.creationsByIp.get(ip) ?? []).filter((createdAt) => createdAt > cutoff);
-    if (recent.length >= this.creationLimitPerMinute) {
-      throw new GuestAccessError("guest_creation_rate_limited", "게스트 생성 요청이 너무 많습니다. 잠시 후 다시 시도해주세요.");
-    }
-    recent.push(this.clock());
-    this.creationsByIp.set(ip, recent);
+    this.recordWindowEvent({
+      store: this.creationsByIp,
+      key: ip,
+      limit: this.creationLimitPerMinute,
+      capacityCode: "guest_creation_capacity_reached",
+      rateCode: "guest_creation_rate_limited",
+      capacityMessage: "게스트 생성 요청을 추적할 수 있는 네트워크 수를 초과했습니다.",
+      rateMessage: "게스트 생성 요청이 너무 많습니다. 잠시 후 다시 시도해주세요."
+    });
+  }
+
+  private recordTicketIssue(ip: string): void {
+    this.recordWindowEvent({
+      store: this.ticketIssuesByIp,
+      key: ip,
+      limit: this.ticketIssueLimitPerMinute,
+      capacityCode: "guest_ticket_rate_limited",
+      rateCode: "guest_ticket_rate_limited",
+      capacityMessage: "게스트 연결 요청이 많습니다. 잠시 후 다시 시도해주세요.",
+      rateMessage: "게스트 연결 요청이 너무 잦습니다. 잠시 후 다시 시도해주세요."
+    });
   }
 
   private lease(guestId: string, leaseId: string): ConnectionLease {
@@ -239,6 +282,44 @@ export class GuestAccess {
     }
   }
 
+  private recordWindowEvent(options: {
+    store: Map<string, RollingWindow>;
+    key: string;
+    limit: number;
+    capacityCode: GuestAccessError["code"];
+    rateCode: GuestAccessError["code"];
+    capacityMessage: string;
+    rateMessage: string;
+  }): void {
+    const nowMs = this.clock();
+    this.pruneWindows(options.store, nowMs);
+    const existing = options.store.get(options.key);
+    const recent = (existing?.timestamps ?? []).filter((createdAt) => createdAt > nowMs - CREATION_WINDOW_MS);
+    if (!existing && options.store.size >= this.trackedIpLimit) {
+      throw new GuestAccessError(options.capacityCode, options.capacityMessage);
+    }
+    if (recent.length >= options.limit) {
+      throw new GuestAccessError(options.rateCode, options.rateMessage);
+    }
+    if (existing) clearTimeout(existing.cleanupTimer);
+    recent.push(nowMs);
+    const expiresAtMs = nowMs + CREATION_WINDOW_MS;
+    const cleanupTimer = setTimeout(() => {
+      const current = options.store.get(options.key);
+      if (current?.expiresAtMs === expiresAtMs) options.store.delete(options.key);
+    }, CREATION_WINDOW_MS);
+    cleanupTimer.unref();
+    options.store.set(options.key, { timestamps: recent, expiresAtMs, cleanupTimer });
+  }
+
+  private pruneWindows(store: Map<string, RollingWindow>, nowMs: number): void {
+    for (const [key, window] of store) {
+      if (nowMs < window.expiresAtMs) continue;
+      clearTimeout(window.cleanupTimer);
+      store.delete(key);
+    }
+  }
+
   private deleteTicket(ticketHash: string): void {
     const ticket = this.tickets.get(ticketHash);
     if (!ticket) return;
@@ -253,6 +334,12 @@ export class GuestAccess {
   }
 }
 
+type RollingWindow = {
+  timestamps: number[];
+  expiresAtMs: number;
+  cleanupTimer: NodeJS.Timeout;
+};
+
 function secureEqual(left: string, right: string): boolean {
   const leftBuffer = Buffer.from(left, "utf8");
   const rightBuffer = Buffer.from(right, "utf8");


## `test(guest): 체험 환경의 복구 경계 검증`

diff --git a/apps/api/src/env.test.ts b/apps/api/src/env.test.ts
index dff081c..6fa8172 100644
--- a/apps/api/src/env.test.ts
+++ b/apps/api/src/env.test.ts
@@ -38,4 +38,15 @@ describe("readEnv", () => {
     expect(readEnv({ TRUST_PROXY: "1" }).trustProxy).toBe(true);
     expect(readEnv({ TRUST_PROXY: "0" }).trustProxy).toBe(false);
   });
+
+  it("honors an explicit production app mode without relying on NODE_ENV", () => {
+    const env = readEnv({
+      APP_MODE: "production",
+      SESSION_SECRET: "0123456789abcdef0123456789abcdef"
+    });
+
+    expect(env.appMode).toBe("production");
+    expect(readEnv({ APP_MODE: "test" }).appMode).toBe("test");
+    expect(() => readEnv({ APP_MODE: "staging" })).toThrow("APP_MODE");
+  });
 });
diff --git a/apps/api/src/guestAccess.test.ts b/apps/api/src/guestAccess.test.ts
index 95afec2..c33fc66 100644
--- a/apps/api/src/guestAccess.test.ts
+++ b/apps/api/src/guestAccess.test.ts
@@ -1,4 +1,4 @@
-import { describe, expect, it } from "vitest";
+import { afterEach, describe, expect, it, vi } from "vitest";
 import { hashWsTicket } from "./wsTicket.js";
 import {
   DEFAULT_GUEST_CONNECTION_LIMIT,
@@ -10,6 +10,8 @@ import {
 } from "./guestAccess.js";
 
 describe("GuestAccess", () => {
+  afterEach(() => vi.useRealTimers());
+
   it("authenticates an HMAC-signed session for two hours and rejects tampering", () => {
     let nowMs = Date.parse("2026-01-01T00:00:00.000Z");
     const access = createAccess({ clock: () => nowMs });
@@ -48,6 +50,33 @@ describe("GuestAccess", () => {
     expect(access.createSession("203.0.113.20").user.sessionKind).toBe("guest");
   });
 
+  it("removes expired creation windows for inactive IP addresses", () => {
+    let nowMs = 1_500_000;
+    const access = createAccess({ clock: () => nowMs });
+    access.createSession("203.0.113.21");
+    access.createSession("203.0.113.22");
+    expect(access.trackedCreationIpCount).toBe(2);
+
+    nowMs += 60_000;
+    access.createSession("203.0.113.23");
+    expect(access.trackedCreationIpCount).toBe(1);
+  });
+
+  it("cleans inactive IP windows on time and bounds the tracked IP store", async () => {
+    vi.useFakeTimers();
+    vi.setSystemTime(new Date("2026-01-01T00:00:00.000Z"));
+    const access = createAccess({ trackedIpLimit: 2 });
+    access.createSession("203.0.113.24");
+    access.createSession("203.0.113.25");
+    expect(() => access.createSession("203.0.113.26")).toThrowError(
+      expect.objectContaining<Partial<GuestAccessError>>({ code: "guest_creation_capacity_reached" })
+    );
+
+    await vi.advanceTimersByTimeAsync(60_000);
+    expect(access.trackedCreationIpCount).toBe(0);
+    expect(access.createSession("203.0.113.26").user.sessionKind).toBe("guest");
+  });
+
   it("stores only a one-time hash for 30-second websocket tickets", () => {
     let nowMs = 2_000_000;
     const access = createAccess({ clock: () => nowMs });
@@ -88,6 +117,30 @@ describe("GuestAccess", () => {
     expect(access.consumeWsTicket(hashWsTicket(replacement))).toBeNull();
   });
 
+  it("cleans tickets on time and limits pending tickets and issuance per IP", async () => {
+    vi.useFakeTimers();
+    vi.setSystemTime(new Date("2026-01-01T00:00:00.000Z"));
+    const pending = createAccess({ ticketsPerIp: 1 });
+    const first = pending.createSession("198.51.100.31").user;
+    const second = pending.createSession("198.51.100.31").user;
+    pending.issueWsTicket(first, "198.51.100.31");
+    expect(() => pending.issueWsTicket(second, "198.51.100.31")).toThrowError(
+      expect.objectContaining<Partial<GuestAccessError>>({ code: "guest_ticket_ip_limit_reached" })
+    );
+
+    await vi.advanceTimersByTimeAsync(30_000);
+    expect(pending.activeTicketCount).toBe(0);
+    expect(pending.issueWsTicket(second, "198.51.100.31")).toMatch(/^[A-Za-z0-9_-]{43}$/);
+
+    const rate = createAccess({ ticketIssueLimitPerMinute: 2 });
+    const guest = rate.createSession("198.51.100.32").user;
+    rate.issueWsTicket(guest, "198.51.100.32");
+    rate.issueWsTicket(guest, "198.51.100.32");
+    expect(() => rate.issueWsTicket(guest, "198.51.100.32")).toThrowError(
+      expect.objectContaining<Partial<GuestAccessError>>({ code: "guest_ticket_rate_limited" })
+    );
+  });
+
   it("limits guest sockets per IP and across the process", () => {
     expect(DEFAULT_GUEST_CONNECTIONS_PER_IP).toBe(4);
     expect(DEFAULT_GUEST_CONNECTION_LIMIT).toBe(200);
diff --git a/apps/web/src/game/GameSocketClient.test.ts b/apps/web/src/game/GameSocketClient.test.ts
index 84398f9..dd6dcd9 100644
--- a/apps/web/src/game/GameSocketClient.test.ts
+++ b/apps/web/src/game/GameSocketClient.test.ts
@@ -1,4 +1,4 @@
-import { describe, expect, it, vi } from "vitest";
+import { afterEach, describe, expect, it, vi } from "vitest";
 import type { WsTicketResponse } from "@pong-pong/shared";
 import {
   GameSocketClient,
@@ -13,6 +13,8 @@ const ticket = {
 } satisfies WsTicketResponse;
 
 describe("GameSocketClient", () => {
+  afterEach(() => vi.useRealTimers());
+
   it("cancels an unused one-time ticket request before starting another connection", async () => {
     const signals: AbortSignal[] = [];
     const ticketProvider = vi.fn((signal?: AbortSignal) => new Promise<WsTicketResponse>((resolve, reject) => {
@@ -95,6 +97,39 @@ describe("GameSocketClient", () => {
       { v: 1, type: "game.input", roomId: "room-1", inputSeq: 3, direction: 1 }
     ]);
   });
+
+  it("uses a fresh ticket to reconnect without sending the original queue command again", async () => {
+    vi.useFakeTimers();
+    const sockets: FakeSocket[] = [];
+    const ticketProvider = vi.fn(async () => ticket);
+    const onClosed = vi.fn(() => true);
+    const client = new GameSocketClient({
+      url: "ws://localhost:4000/ws",
+      ticketProvider,
+      socketFactory: (url) => {
+        const socket = new FakeSocket(url);
+        sockets.push(socket);
+        return socket;
+      }
+    });
+
+    await client.connect(
+      { v: 1, type: "queue.join", mode: "queue" },
+      handlers({ onClosed })
+    );
+    sockets[0].open();
+    expect(sockets[0].sent).toHaveLength(1);
+
+    sockets[0].close();
+    await vi.advanceTimersByTimeAsync(1_000);
+    expect(onClosed).toHaveBeenCalledOnce();
+    expect(ticketProvider).toHaveBeenCalledTimes(2);
+    expect(sockets).toHaveLength(2);
+
+    sockets[1].open();
+    expect(sockets[1].sent).toEqual([]);
+    client.close();
+  });
 });
 
 class FakeSocket implements GameWebSocket {
diff --git a/apps/web/src/game/gameConnection.test.ts b/apps/web/src/game/gameConnection.test.ts
index 95041d8..419e453 100644
--- a/apps/web/src/game/gameConnection.test.ts
+++ b/apps/web/src/game/gameConnection.test.ts
@@ -3,6 +3,7 @@ import type { GameSnapshot } from "@pong-pong/shared";
 import {
   gameConnectionReducer,
   initialGameConnectionState,
+  canStartNewMatch,
   type GameConnectionStatus
 } from "./gameConnection";
 
@@ -105,6 +106,18 @@ describe("gameConnectionReducer", () => {
       messages: []
     });
   });
+
+  it("does not start another queue command while a room is reconnecting", () => {
+    expect(canStartNewMatch({
+      ...initialGameConnectionState,
+      status: "reconnecting",
+      roomId: "room-1"
+    })).toBe(false);
+    expect(canStartNewMatch({
+      ...initialGameConnectionState,
+      status: "finished"
+    })).toBe(true);
+  });
 });
 
 function snapshot(sequence: number, phase: GameSnapshot["state"]["phase"]): GameSnapshot {
diff --git a/apps/web/src/lib/demoPolicy.test.ts b/apps/web/src/lib/demoPolicy.test.ts
index 64dd734..b8cc6c5 100644
--- a/apps/web/src/lib/demoPolicy.test.ts
+++ b/apps/web/src/lib/demoPolicy.test.ts
@@ -2,6 +2,8 @@ import { describe, expect, it } from "vitest";
 import {
   createNavigation,
   demoLobbyPresentation,
+  formatTransientResultNotice,
+  shouldResumeGameFromLobby,
   isDemoRestrictedPath
 } from "./demoPolicy";
 
@@ -45,4 +47,18 @@ describe("guest demo presentation policy", () => {
     expect(isDemoRestrictedPath("/")).toBe(false);
     expect(isDemoRestrictedPath("/play")).toBe(false);
   });
+
+  it("labels recovered guest results as transient", () => {
+    expect(formatTransientResultNotice({
+      persisted: false,
+      leftScore: 1,
+      rightScore: 3
+    })).toBe("임시 경기 종료: 1 - 3 · 전적에 저장되지 않았습니다.");
+  });
+
+  it("moves an accidentally recovered room back to the game screen", () => {
+    expect(shouldResumeGameFromLobby({ type: "queue.matched" })).toBe(true);
+    expect(shouldResumeGameFromLobby({ type: "game.snapshot" })).toBe(true);
+    expect(shouldResumeGameFromLobby({ type: "game.finished" })).toBe(false);
+  });
 });
