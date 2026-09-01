# 비회원 체험의 신원·자원·데이터 격리

## `feat(guest): signed guest session token 정의`

diff --git a/apps/api/src/guestAccess.ts b/apps/api/src/guestAccess.ts
new file mode 100644
index 0000000..464355c
--- /dev/null
+++ b/apps/api/src/guestAccess.ts
@@ -0,0 +1,109 @@
+import {
+  createHmac,
+  randomBytes,
+  randomInt,
+  randomUUID,
+  timingSafeEqual
+} from "node:crypto";
+import type { SessionUser } from "@pong-pong/shared";
+
+export const GUEST_SESSION_TTL_SECONDS = 2 * 60 * 60;
+
+export type GuestSessionUser = SessionUser & {
+  sessionKind: "guest";
+};
+
+type GuestPayload = {
+  v: 1;
+  user: GuestSessionUser;
+  ip: string;
+  expiresAtMs: number;
+};
+
+type GuestAccessOptions = {
+  secret: string;
+  clock?: () => number;
+};
+
+export class GuestAccess {
+  private readonly clock: () => number;
+
+  constructor(private readonly options: GuestAccessOptions) {
+    if (Buffer.byteLength(options.secret, "utf8") < 32) {
+      throw new Error("Guest session secret must be at least 32 bytes");
+    }
+    this.clock = options.clock ?? Date.now;
+  }
+
+  createSession(ip: string): {
+    user: GuestSessionUser;
+    cookieValue: string;
+    expiresInSeconds: number;
+  } {
+    const handleSuffix = randomBytes(6).toString("hex");
+    const user: GuestSessionUser = {
+      id: randomUUID(),
+      handle: `guest-${handleSuffix}`,
+      displayName: `게스트 ${randomInt(1_000, 10_000)}`,
+      avatarKey: "default",
+      role: "user",
+      status: "active",
+      rating: 1_200,
+      wins: 0,
+      losses: 0,
+      online: true,
+      isNpc: false,
+      email: null,
+      sessionKind: "guest"
+    };
+    const payload: GuestPayload = {
+      v: 1,
+      user,
+      ip,
+      expiresAtMs: this.clock() + (GUEST_SESSION_TTL_SECONDS * 1_000)
+    };
+    const encoded = Buffer.from(JSON.stringify(payload), "utf8").toString("base64url");
+    return {
+      user,
+      cookieValue: `${encoded}.${this.sign(encoded)}`,
+      expiresInSeconds: GUEST_SESSION_TTL_SECONDS
+    };
+  }
+
+  authenticate(cookieValue: string | undefined, expectedIp?: string): GuestSessionUser | null {
+    if (!cookieValue) return null;
+    const separator = cookieValue.lastIndexOf(".");
+    if (separator <= 0) return null;
+    const encoded = cookieValue.slice(0, separator);
+    const signature = cookieValue.slice(separator + 1);
+    if (!secureEqual(signature, this.sign(encoded))) return null;
+
+    try {
+      const payload = JSON.parse(Buffer.from(encoded, "base64url").toString("utf8")) as GuestPayload;
+      if (
+        payload.v !== 1
+        || payload.user?.sessionKind !== "guest"
+        || payload.user.role !== "user"
+        || payload.user.status !== "active"
+        || !Number.isFinite(payload.expiresAtMs)
+        || this.clock() >= payload.expiresAtMs
+        || (expectedIp !== undefined && payload.ip !== expectedIp)
+      ) {
+        return null;
+      }
+      return payload.user;
+    } catch {
+      return null;
+    }
+  }
+
+  private sign(payload: string): string {
+    return createHmac("sha256", this.options.secret).update(payload, "utf8").digest("base64url");
+  }
+}
+
+function secureEqual(left: string, right: string): boolean {
+  const leftBuffer = Buffer.from(left, "utf8");
+  const rightBuffer = Buffer.from(right, "utf8");
+  return leftBuffer.byteLength === rightBuffer.byteLength && timingSafeEqual(leftBuffer, rightBuffer);
+}


## `feat(guest): guest 요청 rate limit 추가`

diff --git a/apps/api/src/guestAccess.ts b/apps/api/src/guestAccess.ts
index 464355c..74e28dd 100644
--- a/apps/api/src/guestAccess.ts
+++ b/apps/api/src/guestAccess.ts
@@ -8,6 +8,9 @@ import {
 import type { SessionUser } from "@pong-pong/shared";
 
 export const GUEST_SESSION_TTL_SECONDS = 2 * 60 * 60;
+export const DEFAULT_GUEST_CREATION_LIMIT_PER_MINUTE = 10;
+
+const CREATION_WINDOW_MS = 60_000;
 
 export type GuestSessionUser = SessionUser & {
   sessionKind: "guest";
@@ -23,16 +26,30 @@ type GuestPayload = {
 type GuestAccessOptions = {
   secret: string;
   clock?: () => number;
+  creationLimitPerMinute?: number;
 };
 
+export class GuestAccessError extends Error {
+  constructor(
+    readonly code: "guest_creation_rate_limited" | "guest_ticket_limit_reached",
+    message: string
+  ) {
+    super(message);
+    this.name = "GuestAccessError";
+  }
+}
+
 export class GuestAccess {
   private readonly clock: () => number;
+  private readonly creationLimitPerMinute: number;
+  private readonly creationsByIp = new Map<string, number[]>();
 
   constructor(private readonly options: GuestAccessOptions) {
     if (Buffer.byteLength(options.secret, "utf8") < 32) {
       throw new Error("Guest session secret must be at least 32 bytes");
     }
     this.clock = options.clock ?? Date.now;
+    this.creationLimitPerMinute = options.creationLimitPerMinute ?? DEFAULT_GUEST_CREATION_LIMIT_PER_MINUTE;
   }
 
   createSession(ip: string): {
@@ -40,6 +57,7 @@ export class GuestAccess {
     cookieValue: string;
     expiresInSeconds: number;
   } {
+    this.recordCreation(ip);
     const handleSuffix = randomBytes(6).toString("hex");
     const user: GuestSessionUser = {
       id: randomUUID(),
@@ -97,6 +115,16 @@ export class GuestAccess {
     }
   }
 
+  private recordCreation(ip: string): void {
+    const cutoff = this.clock() - CREATION_WINDOW_MS;
+    const recent = (this.creationsByIp.get(ip) ?? []).filter((createdAt) => createdAt > cutoff);
+    if (recent.length >= this.creationLimitPerMinute) {
+      throw new GuestAccessError("guest_creation_rate_limited", "게스트 생성 요청이 너무 많습니다. 잠시 후 다시 시도해주세요.");
+    }
+    recent.push(this.clock());
+    this.creationsByIp.set(ip, recent);
+  }
+
   private sign(payload: string): string {
     return createHmac("sha256", this.options.secret).update(payload, "utf8").digest("base64url");
   }


## `feat(guest): guest WebSocket ticket 발급 추가`

diff --git a/apps/api/src/guestAccess.ts b/apps/api/src/guestAccess.ts
index 74e28dd..9881df0 100644
--- a/apps/api/src/guestAccess.ts
+++ b/apps/api/src/guestAccess.ts
@@ -6,9 +6,11 @@ import {
   timingSafeEqual
 } from "node:crypto";
 import type { SessionUser } from "@pong-pong/shared";
+import { createRawWsTicket, hashWsTicket, WS_TICKET_TTL_SECONDS } from "./wsTicket.js";
 
 export const GUEST_SESSION_TTL_SECONDS = 2 * 60 * 60;
 export const DEFAULT_GUEST_CREATION_LIMIT_PER_MINUTE = 10;
+export const DEFAULT_GUEST_TICKET_LIMIT = 400;
 
 const CREATION_WINDOW_MS = 60_000;
 
@@ -27,6 +29,7 @@ type GuestAccessOptions = {
   secret: string;
   clock?: () => number;
   creationLimitPerMinute?: number;
+  ticketLimit?: number;
 };
 
 export class GuestAccessError extends Error {
@@ -42,7 +45,14 @@ export class GuestAccessError extends Error {
 export class GuestAccess {
   private readonly clock: () => number;
   private readonly creationLimitPerMinute: number;
+  private readonly ticketLimit: number;
   private readonly creationsByIp = new Map<string, number[]>();
+  private readonly tickets = new Map<string, {
+    user: GuestSessionUser;
+    expiresAtMs: number;
+    cleanupTimer: NodeJS.Timeout;
+  }>();
+  private readonly ticketHashByGuest = new Map<string, string>();
 
   constructor(private readonly options: GuestAccessOptions) {
     if (Buffer.byteLength(options.secret, "utf8") < 32) {
@@ -50,6 +60,12 @@ export class GuestAccess {
     }
     this.clock = options.clock ?? Date.now;
     this.creationLimitPerMinute = options.creationLimitPerMinute ?? DEFAULT_GUEST_CREATION_LIMIT_PER_MINUTE;
+    this.ticketLimit = options.ticketLimit ?? DEFAULT_GUEST_TICKET_LIMIT;
+  }
+
+  get activeTicketCount(): number {
+    this.pruneExpiredTickets();
+    return this.tickets.size;
   }
 
   createSession(ip: string): {
@@ -115,6 +131,46 @@ export class GuestAccess {
     }
   }
 
+  issueWsTicket(user: GuestSessionUser): string {
+    this.pruneExpiredTickets();
+    const previousHash = this.ticketHashByGuest.get(user.id);
+    if (previousHash) {
+      const previous = this.tickets.get(previousHash);
+      if (previous) clearTimeout(previous.cleanupTimer);
+      this.tickets.delete(previousHash);
+      this.ticketHashByGuest.delete(user.id);
+    }
+    if (this.tickets.size >= this.ticketLimit) {
+      throw new GuestAccessError(
+        "guest_ticket_limit_reached",
+        "게스트 연결 요청이 많습니다. 잠시 후 다시 시도해주세요."
+      );
+    }
+    const ticket = createRawWsTicket();
+    const ticketHash = hashWsTicket(ticket);
+    const expiresAtMs = this.clock() + (WS_TICKET_TTL_SECONDS * 1_000);
+    const cleanupTimer = setTimeout(() => this.deleteTicket(ticketHash), WS_TICKET_TTL_SECONDS * 1_000);
+    cleanupTimer.unref();
+    this.tickets.set(ticketHash, {
+      user,
+      expiresAtMs,
+      cleanupTimer
+    });
+    this.ticketHashByGuest.set(user.id, ticketHash);
+    return ticket;
+  }
+
+  consumeWsTicket(ticketHash: string): GuestSessionUser | null {
+    const stored = this.tickets.get(ticketHash);
+    if (stored) clearTimeout(stored.cleanupTimer);
+    this.tickets.delete(ticketHash);
+    if (stored && this.ticketHashByGuest.get(stored.user.id) === ticketHash) {
+      this.ticketHashByGuest.delete(stored.user.id);
+    }
+    if (!stored || this.clock() >= stored.expiresAtMs) return null;
+    return stored.user;
+  }
+
   private recordCreation(ip: string): void {
     const cutoff = this.clock() - CREATION_WINDOW_MS;
     const recent = (this.creationsByIp.get(ip) ?? []).filter((createdAt) => createdAt > cutoff);
@@ -125,6 +181,27 @@ export class GuestAccess {
     this.creationsByIp.set(ip, recent);
   }
 
+  private pruneExpiredTickets(): void {
+    const nowMs = this.clock();
+    for (const [ticketHash, ticket] of this.tickets) {
+      if (nowMs < ticket.expiresAtMs) continue;
+      clearTimeout(ticket.cleanupTimer);
+      this.tickets.delete(ticketHash);
+      if (this.ticketHashByGuest.get(ticket.user.id) === ticketHash) {
+        this.ticketHashByGuest.delete(ticket.user.id);
+      }
+    }
+  }
+
+  private deleteTicket(ticketHash: string): void {
+    const ticket = this.tickets.get(ticketHash);
+    if (!ticket) return;
+    this.tickets.delete(ticketHash);
+    if (this.ticketHashByGuest.get(ticket.user.id) === ticketHash) {
+      this.ticketHashByGuest.delete(ticket.user.id);
+    }
+  }
+
   private sign(payload: string): string {
     return createHmac("sha256", this.options.secret).update(payload, "utf8").digest("base64url");
   }


## `feat(guest): guest resource lease 수명주기 추가`

diff --git a/apps/api/src/guestAccess.ts b/apps/api/src/guestAccess.ts
index 9881df0..e7de311 100644
--- a/apps/api/src/guestAccess.ts
+++ b/apps/api/src/guestAccess.ts
@@ -10,6 +10,8 @@ import { createRawWsTicket, hashWsTicket, WS_TICKET_TTL_SECONDS } from "./wsTick
 
 export const GUEST_SESSION_TTL_SECONDS = 2 * 60 * 60;
 export const DEFAULT_GUEST_CREATION_LIMIT_PER_MINUTE = 10;
+export const DEFAULT_GUEST_CONNECTIONS_PER_IP = 4;
+export const DEFAULT_GUEST_CONNECTION_LIMIT = 200;
 export const DEFAULT_GUEST_TICKET_LIMIT = 400;
 
 const CREATION_WINDOW_MS = 60_000;
@@ -29,9 +31,15 @@ type GuestAccessOptions = {
   secret: string;
   clock?: () => number;
   creationLimitPerMinute?: number;
+  connectionsPerIp?: number;
+  connectionLimit?: number;
   ticketLimit?: number;
 };
 
+type ConnectionLease = {
+  release(): void;
+};
+
 export class GuestAccessError extends Error {
   constructor(
     readonly code: "guest_creation_rate_limited" | "guest_ticket_limit_reached",
@@ -45,6 +53,8 @@ export class GuestAccessError extends Error {
 export class GuestAccess {
   private readonly clock: () => number;
   private readonly creationLimitPerMinute: number;
+  private readonly connectionsPerIp: number;
+  private readonly connectionLimit: number;
   private readonly ticketLimit: number;
   private readonly creationsByIp = new Map<string, number[]>();
   private readonly tickets = new Map<string, {
@@ -53,6 +63,7 @@ export class GuestAccess {
     cleanupTimer: NodeJS.Timeout;
   }>();
   private readonly ticketHashByGuest = new Map<string, string>();
+  private readonly connections = new Map<string, { ip: string; leaseId: string }>();
 
   constructor(private readonly options: GuestAccessOptions) {
     if (Buffer.byteLength(options.secret, "utf8") < 32) {
@@ -60,9 +71,15 @@ export class GuestAccess {
     }
     this.clock = options.clock ?? Date.now;
     this.creationLimitPerMinute = options.creationLimitPerMinute ?? DEFAULT_GUEST_CREATION_LIMIT_PER_MINUTE;
+    this.connectionsPerIp = options.connectionsPerIp ?? DEFAULT_GUEST_CONNECTIONS_PER_IP;
+    this.connectionLimit = options.connectionLimit ?? DEFAULT_GUEST_CONNECTION_LIMIT;
     this.ticketLimit = options.ticketLimit ?? DEFAULT_GUEST_TICKET_LIMIT;
   }
 
+  get activeConnectionCount(): number {
+    return this.connections.size;
+  }
+
   get activeTicketCount(): number {
     this.pruneExpiredTickets();
     return this.tickets.size;
@@ -171,6 +188,27 @@ export class GuestAccess {
     return stored.user;
   }
 
+  acquireConnection(ip: string, guestId: string): ConnectionLease | null {
+    const current = this.connections.get(guestId);
+    const leaseId = randomUUID();
+    if (current) {
+      if (current.ip !== ip) {
+        const connectionsForIp = [...this.connections.values()]
+          .filter((connection) => connection.ip === ip).length;
+        if (connectionsForIp >= this.connectionsPerIp) return null;
+      }
+      this.connections.set(guestId, { ip, leaseId });
+      return this.lease(guestId, leaseId);
+    }
+
+    const connectionsForIp = [...this.connections.values()].filter((connection) => connection.ip === ip).length;
+    if (connectionsForIp >= this.connectionsPerIp || this.connections.size >= this.connectionLimit) {
+      return null;
+    }
+    this.connections.set(guestId, { ip, leaseId });
+    return this.lease(guestId, leaseId);
+  }
+
   private recordCreation(ip: string): void {
     const cutoff = this.clock() - CREATION_WINDOW_MS;
     const recent = (this.creationsByIp.get(ip) ?? []).filter((createdAt) => createdAt > cutoff);
@@ -181,6 +219,14 @@ export class GuestAccess {
     this.creationsByIp.set(ip, recent);
   }
 
+  private lease(guestId: string, leaseId: string): ConnectionLease {
+    return {
+      release: () => {
+        if (this.connections.get(guestId)?.leaseId === leaseId) this.connections.delete(guestId);
+      }
+    };
+  }
+
   private pruneExpiredTickets(): void {
     const nowMs = this.clock();
     for (const [ticketHash, ticket] of this.tickets) {


## `feat(guest): guest session과 WebSocket 인증 연결`

diff --git a/apps/api/src/app.ts b/apps/api/src/app.ts
index 7d84ba5..d085978 100644
--- a/apps/api/src/app.ts
+++ b/apps/api/src/app.ts
@@ -109,7 +109,14 @@ export function buildApp({
         return;
       }
 
-      repo.consumeWsTicket(hashWsTicket(parsedQuery.data.ticket))
+      const ticketHash = hashWsTicket(parsedQuery.data.ticket);
+      const guestUser = guests?.consumeWsTicket(ticketHash) ?? null;
+      const authenticated = guestUser
+        ? Promise.resolve(guestUser)
+        : appMode === "demo"
+          ? Promise.resolve(null)
+          : repo.consumeWsTicket(ticketHash);
+      authenticated
         .then((user) => {
           if (!user) {
             closeAuthentication(WS_POLICY_VIOLATION, "invalid websocket ticket");
@@ -118,6 +125,12 @@ export function buildApp({
           if (authenticationClosed || socket.readyState !== WebSocket.OPEN) {
             return;
           }
+          const lease = isGuestSession(user) ? guests?.acquireConnection(request.ip, user.id) : null;
+          if (isGuestSession(user) && !lease) {
+            closeAuthentication(WS_POLICY_VIOLATION, "guest connection limit exceeded");
+            return;
+          }
+          if (lease) socket.once("close", () => lease.release());
           socket.off("message", bufferPayload);
           hub.connect(socket as WebSocket, request.raw, user, pendingPayloads);
         })
@@ -146,24 +159,65 @@ export function buildApp({
     });
   }
 
+  if (appMode === "demo" && guests) {
+    app.post("/auth/guest", async (request, reply) => {
+      parseInput(http.emptyParamsSchema, request.body ?? {});
+      try {
+        const session = guests.createSession(request.ip);
+        reply.setCookie("pp_guest", session.cookieValue, {
+          path: "/",
+          sameSite: "lax",
+          httpOnly: true,
+          secure: true,
+          maxAge: GUEST_SESSION_TTL_SECONDS
+        });
+        return parseOutput(http.guestAuthResponseSchema, {
+          user: session.user,
+          guest: true,
+          expiresInSeconds: session.expiresInSeconds
+        });
+      } catch (error) {
+        if (error instanceof GuestAccessError) {
+          throw new ApiHttpError(429, error.code, error.message);
+        }
+        throw error;
+      }
+    });
+  }
+
   app.post("/auth/logout", async (request, reply) => {
-    await repo.deleteSession(readSessionToken(request));
+    if (!isGuestSession(await getCurrentUser(request))) {
+      await repo.deleteSession(readSessionToken(request));
+    }
     reply.clearCookie("pp_session", { path: "/" });
+    reply.clearCookie("pp_guest", { path: "/" });
     return parseOutput(http.okResponseSchema, { ok: true });
   });
 
   app.post("/auth/ws-ticket", async (request) => {
     parseInput(http.emptyParamsSchema, request.body ?? {});
-    const user = await currentUser(repo, request);
+    const user = await getCurrentUser(request);
     if (!user) unauthorized();
     if (!isActive(user)) suspended();
 
-    const ticket = createRawWsTicket();
-    await repo.createWsTicket({
-      userId: user.id,
-      ticketHash: hashWsTicket(ticket),
-      ttlSeconds: WS_TICKET_TTL_SECONDS
-    });
+    let ticket: string;
+    try {
+      ticket = isGuestSession(user) && guests
+        ? guests.issueWsTicket(user)
+        : createRawWsTicket();
+    } catch (error) {
+      if (error instanceof GuestAccessError) {
+        throw new ApiHttpError(429, error.code, error.message);
+      }
+      throw error;
+    }
+    if (!isGuestSession(user)) {
+      await repo.createWsTicket({
+        userId: user.id,
+        ticketHash: hashWsTicket(ticket),
+        ttlSeconds: WS_TICKET_TTL_SECONDS
+      });
+    }
     return parseOutput(http.wsTicketResponseSchema, {
       ticket,
       expiresInSeconds: WS_TICKET_TTL_SECONDS,
@@ -356,6 +410,16 @@ function isActive(user: SessionUser): boolean {
   return user.status === "active";
 }
 
+function isGuestSession(user: SessionUser | GuestSessionUser | null): user is GuestSessionUser {
+  return Boolean(user && "sessionKind" in user && user.sessionKind === "guest");
+}
+
+function requireRegistered(user: SessionUser | GuestSessionUser): void {
+  if (isGuestSession(user)) {
+    throw new ApiHttpError(403, "guest_feature_forbidden", "게스트 계정에서는 사용할 수 없는 기능입니다.");
+  }
+}
+
 function readAppMode(input = process.env): AppMode {
   if (input.APP_MODE === "demo") return "demo";
   if (input.NODE_ENV === "production") return "production";


## `feat(guest): guest 조회 범위와 lobby 격리`

diff --git a/apps/api/src/app.ts b/apps/api/src/app.ts
index d085978..09e1255 100644
--- a/apps/api/src/app.ts
+++ b/apps/api/src/app.ts
@@ -226,18 +226,19 @@ export function buildApp({
   });
 
   app.get("/me", async (request) => {
-    const user = await currentUser(repo, request);
+    const user = await getCurrentUser(request);
     if (!user) unauthorized();
     return parseOutput(http.userResponseSchema, { user });
   });
 
   app.get("/auth/me", async (request) => {
-    const user = await currentUser(repo, request);
+    const user = await getCurrentUser(request);
     if (!user) unauthorized();
     return parseOutput(http.userResponseSchema, { user });
   });
 
   app.get("/users/:id", async (request) => {
+    if (appMode === "demo") notFound("데모 모드에서는 제공하지 않는 기능입니다.");
     const { id } = parseInput(http.idParamsSchema, request.params);
     const user = await repo.getUserById(id);
     if (!user) notFound("사용자를 찾을 수 없습니다.");
@@ -245,19 +246,21 @@ export function buildApp({
   });
 
   app.get("/lobby", async (request) => {
-    const user = await currentUser(repo, request);
+    const user = await getCurrentUser(request);
+    const guest = isGuestSession(user);
     return parseOutput(http.lobbyResponseSchema, {
       me: user,
       onlinePlayers: hub.onlinePlayers(),
-      recentMatches: await repo.listRecentMatches(user?.id),
-      chat: await repo.listLobbyChat(),
+      recentMatches: appMode === "demo" || guest ? [] : await repo.listRecentMatches(user?.id),
+      chat: appMode === "demo" || guest ? [] : await repo.listLobbyChat(),
       stats: hub.liveStats()
     });
   });
 
   app.post("/chat/lobby", async (request) => {
-    const user = await currentUser(repo, request);
+    const user = await getCurrentUser(request);
     if (!user) unauthorized();
+    requireRegistered(user);
     if (!isActive(user)) suspended();
     const body = parseInput(http.chatBodySchema, request.body);
     return parseOutput(http.chatResponseSchema, {
@@ -270,17 +273,20 @@ export function buildApp({
     });
   });
 
-  app.get("/leaderboard", async () => parseOutput(http.leaderboardResponseSchema, {
-    entries: await repo.listLeaderboard()
-  }));
+  app.get("/leaderboard", async () => {
+    if (appMode === "demo") notFound("데모 모드에서는 제공하지 않는 기능입니다.");
+    return parseOutput(http.leaderboardResponseSchema, { entries: await repo.listLeaderboard() });
+  });
 
   app.get("/dashboard", async (request) => {
-    const user = await currentUser(repo, request);
+    const user = await getCurrentUser(request);
     if (!user) unauthorized();
+    requireRegistered(user);
     return parseOutput(http.dashboardSummarySchema, await repo.getDashboard(user.id));
   });
 
   app.get("/profile/:handle", async (request) => {
+    if (appMode === "demo") notFound("데모 모드에서는 제공하지 않는 기능입니다.");
     const { handle } = parseInput(http.handleParamsSchema, request.params);
     const user = await repo.getUserByHandle(handle);
     if (!user) notFound("프로필을 찾을 수 없습니다.");
@@ -331,9 +337,10 @@ export function buildApp({
     return parseOutput(http.friendResponseSchema, { friend: await repo.acceptFriend(user.id, id) });
   });
 
-  app.get("/tournaments", async () => parseOutput(http.tournamentsResponseSchema, {
-    tournaments: await repo.listTournaments()
-  }));
+  app.get("/tournaments", async () => {
+    if (appMode === "demo") notFound("데모 모드에서는 제공하지 않는 기능입니다.");
+    return parseOutput(http.tournamentsResponseSchema, { tournaments: await repo.listTournaments() });
+  });
 
   app.post("/tournaments", async (request) => {
     const user = await currentUser(repo, request);


