## `test(guest): 격리된 guest session 경계 검증`

diff --git a/apps/api/src/gameHub.guest.test.ts b/apps/api/src/gameHub.guest.test.ts
new file mode 100644
index 0000000..478a273
--- /dev/null
+++ b/apps/api/src/gameHub.guest.test.ts
@@ -0,0 +1,206 @@
+import { EventEmitter } from "node:events";
+import type { IncomingMessage } from "node:http";
+import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
+import { WebSocket } from "ws";
+import { createMemoryRepository } from "@pong-pong/db";
+import { parseServerEvent, type ServerEvent, type SessionUser } from "@pong-pong/shared";
+import { GameHub } from "./gameHub.js";
+import type { GuestSessionUser } from "./guestAccess.js";
+
+describe("GameHub guest isolation", () => {
+  const repositories: Array<ReturnType<typeof createMemoryRepository>> = [];
+  const sockets: FakeSocket[] = [];
+
+  beforeEach(() => {
+    vi.useFakeTimers();
+    vi.setSystemTime(new Date("2026-01-01T00:00:00.000Z"));
+  });
+
+  afterEach(async () => {
+    for (const socket of sockets.splice(0)) socket.terminate();
+    vi.clearAllTimers();
+    vi.useRealTimers();
+    await Promise.all(repositories.splice(0).map((repository) => repository.close()));
+  });
+
+  it("never pairs a guest with a registered user", async () => {
+    const { hub } = setup();
+    const guestLeft = connect(hub, guest("guest-left", "게스트 1001"));
+    const registered = connect(hub, player("registered", "등록 사용자"));
+
+    guestLeft.receive({ v: 1, type: "queue.join", mode: "queue" });
+    registered.receive({ v: 1, type: "queue.join", mode: "queue" });
+    await flushEvents();
+    expect(guestLeft.latest("queue.matched")).toBeUndefined();
+    expect(registered.latest("queue.matched")).toBeUndefined();
+
+    const guestRight = connect(hub, guest("guest-right", "게스트 1002"));
+    guestRight.receive({ v: 1, type: "queue.join", mode: "queue" });
+    await flushEvents();
+
+    expect(guestLeft.latest("queue.matched")).toEqual(
+      expect.objectContaining({ opponent: "게스트 1002" })
+    );
+    expect(guestRight.latest("queue.matched")).toEqual(
+      expect.objectContaining({ opponent: "게스트 1001" })
+    );
+    expect(registered.latest("queue.matched")).toBeUndefined();
+  });
+
+  it("moves a waiting guest to an in-memory AI after six seconds", async () => {
+    const { hub, repository } = setup();
+    const listNpcs = vi.spyOn(repository, "listNpcOpponents");
+    const socket = connect(hub, guest("guest-fallback", "게스트 2001"));
+    socket.receive({ v: 1, type: "queue.join", mode: "queue" });
+
+    await vi.advanceTimersByTimeAsync(5_999);
+    expect(socket.latest("queue.matched")).toBeUndefined();
+    await vi.advanceTimersByTimeAsync(1);
+    await flushEvents();
+
+    expect(socket.latest("queue.matched")).toEqual(
+      expect.objectContaining({ opponent: "연습 AI" })
+    );
+    expect(listNpcs).not.toHaveBeenCalled();
+  });
+
+  it("rejects guest chat and tournament commands before repository access", async () => {
+    const { hub, repository } = setup();
+    const chat = vi.spyOn(repository, "createChatMessage");
+    const tournament = vi.spyOn(repository, "getTournamentMatch");
+    const socket = connect(hub, guest("guest-commands", "게스트 3001"));
+
+    socket.receive({ v: 1, type: "chat.send", scope: "lobby", body: "게스트 채팅" });
+    socket.receive({ v: 1, type: "tournament.join", matchId: "match-1" });
+    await flushEvents();
+
+    expect(socket.events().filter((event) => event.type === "error")).toEqual([
+      expect.objectContaining({ code: "forbidden" }),
+      expect.objectContaining({ code: "forbidden" })
+    ]);
+    expect(chat).not.toHaveBeenCalled();
+    expect(tournament).not.toHaveBeenCalled();
+  });
+
+  it("returns a transient result without finalizing or rating changes and remembers it for two minutes", async () => {
+    const { hub, repository } = setup();
+    const finalize = vi.spyOn(repository, "finalizeMatch");
+    const leftUser = guest("guest-result-left", "게스트 4001");
+    const rightUser = guest("guest-result-right", "게스트 4002");
+    const left = connect(hub, leftUser);
+    const right = connect(hub, rightUser);
+
+    left.receive({ v: 1, type: "queue.join", mode: "queue" });
+    right.receive({ v: 1, type: "queue.join", mode: "queue" });
+    await flushEvents();
+    const matched = right.latest("queue.matched");
+    if (matched?.type !== "queue.matched") throw new Error("expected a guest match");
+    left.receive({ v: 1, type: "game.ready", roomId: matched.roomId });
+    right.receive({ v: 1, type: "game.ready", roomId: matched.roomId });
+    await flushEvents();
+
+    left.terminate();
+    await vi.advanceTimersByTimeAsync(15_000);
+    await flushEvents();
+
+    expect(finalize).not.toHaveBeenCalled();
+    expect(right.latest("game.finished")).toEqual({
+      v: 1,
+      type: "game.finished",
+      result: {
+        roomId: matched.roomId,
+        matchId: null,
+        persisted: false,
+        winnerSide: "right",
+        leftScore: 0,
+        rightScore: 3,
+        ratingDelta: 0
+      }
+    });
+
+    const recovered = connect(hub, leftUser);
+    await flushEvents();
+    expect(recovered.latest("game.finished")).toEqual(right.latest("game.finished"));
+    recovered.terminate();
+
+    await vi.advanceTimersByTimeAsync(120_001);
+    const expired = connect(hub, leftUser);
+    await flushEvents();
+    expect(expired.latest("game.finished")).toBeUndefined();
+  });
+
+  function setup() {
+    const repository = createMemoryRepository();
+    repositories.push(repository);
+    return { repository, hub: new GameHub(repository) };
+  }
+
+  function connect(hub: GameHub, user: SessionUser | GuestSessionUser): FakeSocket {
+    const socket = new FakeSocket();
+    sockets.push(socket);
+    hub.connect(socket as unknown as WebSocket, {} as IncomingMessage, user);
+    return socket;
+  }
+});
+
+class FakeSocket extends EventEmitter {
+  readyState: number = WebSocket.OPEN;
+  bufferedAmount = 0;
+  private readonly payloads: string[] = [];
+
+  send(payload: string, callback?: (error?: Error) => void): void {
+    this.payloads.push(payload);
+    callback?.();
+  }
+
+  ping(): void {}
+
+  close(): void {
+    this.terminate();
+  }
+
+  terminate(): void {
+    if (this.readyState === WebSocket.CLOSED) return;
+    this.readyState = WebSocket.CLOSED;
+    this.emit("close");
+  }
+
+  receive(event: object): void {
+    this.emit("message", Buffer.from(JSON.stringify(event)));
+  }
+
+  latest(type: ServerEvent["type"]): ServerEvent | undefined {
+    return this.events().filter((event) => event.type === type).at(-1);
+  }
+
+  events(): ServerEvent[] {
+    return this.payloads.map((payload) => parseServerEvent(payload));
+  }
+}
+
+function guest(handle: string, displayName: string): GuestSessionUser {
+  return { ...player(handle, displayName), sessionKind: "guest" };
+}
+
+function player(handle: string, displayName: string): SessionUser {
+  return {
+    id: `${handle}-id`,
+    handle,
+    displayName,
+    avatarKey: "default",
+    role: "user",
+    status: "active",
+    rating: 1_200,
+    wins: 0,
+    losses: 0,
+    online: true,
+    isNpc: false,
+    email: null
+  };
+}
+
+async function flushEvents(): Promise<void> {
+  await Promise.resolve();
+  await Promise.resolve();
+  await Promise.resolve();
+}
diff --git a/apps/api/src/guest-demo.test.ts b/apps/api/src/guest-demo.test.ts
new file mode 100644
index 0000000..b59147e
--- /dev/null
+++ b/apps/api/src/guest-demo.test.ts
@@ -0,0 +1,204 @@
+import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
+import { WebSocket } from "ws";
+import { createMemoryRepository, type AppRepository } from "@pong-pong/db";
+import { buildApp } from "./app.js";
+import { GuestAccess } from "./guestAccess.js";
+
+type InjectResponse = {
+  statusCode: number;
+  headers: Record<string, string | string[] | number | undefined>;
+  json<T = unknown>(): T;
+};
+
+describe("guest demo HTTP boundary", () => {
+  let repo: AppRepository;
+  let app: ReturnType<typeof buildApp>;
+
+  beforeEach(async () => {
+    repo = createMemoryRepository();
+    app = buildApp({
+      repo,
+      webOrigin: "http://localhost:3000",
+      appMode: "demo",
+      guestAccess: new GuestAccess({ secret: "guest-demo-test-secret-that-is-long-enough" })
+    });
+    await app.ready();
+  });
+
+  afterEach(async () => {
+    await app.close();
+    await repo.close();
+    vi.restoreAllMocks();
+  });
+
+  it("creates a server-named guest without writing a user or session to the database", async () => {
+    const createSession = vi.spyOn(repo, "createSession");
+    const upsertUser = vi.spyOn(repo, "upsertDevUser");
+    const response = await app.inject({ method: "POST", url: "/auth/guest" });
+
+    expect(response.statusCode).toBe(200);
+    const body = response.json<{
+      user: { id: string; handle: string; displayName: string; role: string };
+      guest: boolean;
+      expiresInSeconds: number;
+    }>();
+    expect(body).toMatchObject({
+      user: {
+        handle: expect.stringMatching(/^guest-[a-f0-9]{12}$/),
+        displayName: expect.stringMatching(/^게스트 [0-9]{4}$/),
+        role: "user"
+      },
+      guest: true,
+      expiresInSeconds: 7_200
+    });
+    expect(await repo.getUserById(body.user.id)).toBeNull();
+    expect(createSession).not.toHaveBeenCalled();
+    expect(upsertUser).not.toHaveBeenCalled();
+
+    const cookieHeader = guestCookieHeader(response);
+    expect(cookieHeader).toContain("Max-Age=7200");
+    expect(cookieHeader).toContain("Path=/");
+    expect(cookieHeader).toContain("HttpOnly");
+    expect(cookieHeader).toContain("Secure");
+    expect(cookieHeader).toContain("SameSite=Lax");
+
+    const me = await app.inject({
+      method: "GET",
+      url: "/me",
+      headers: { cookie: guestCookie(response) }
+    });
+    expect(me.statusCode).toBe(200);
+    expect(me.json<{ user: { id: string } }>().user.id).toBe(body.user.id);
+  });
+
+  it("rejects guest input fields and limits creation to ten requests per IP per minute", async () => {
+    const withBody = await app.inject({
+      method: "POST",
+      url: "/auth/guest",
+      payload: { displayName: "직접 정한 이름" }
+    });
+    expectApiError(withBody, 400, "validation_failed");
+
+    for (let count = 0; count < 10; count += 1) {
+      const response = await app.inject({ method: "POST", url: "/auth/guest" });
+      expect(response.statusCode).toBe(200);
+    }
+    const limited = await app.inject({ method: "POST", url: "/auth/guest" });
+    expectApiError(limited, 429, "guest_creation_rate_limited");
+  });
+
+  it("does not expose guest login outside demo mode", async () => {
+    const developmentApp = buildApp({ repo, webOrigin: "http://localhost:3000", appMode: "test" });
+    await developmentApp.ready();
+    try {
+      const response = await developmentApp.inject({ method: "POST", url: "/auth/guest" });
+      expectApiError(response, 404, "not_found");
+    } finally {
+      await developmentApp.close();
+    }
+  });
+
+  it("blocks guest chat, profile, friend, tournament, and admin operations", async () => {
+    const login = await app.inject({ method: "POST", url: "/auth/guest" });
+    const cookie = guestCookie(login);
+    const createChat = vi.spyOn(repo, "createChatMessage");
+    const updateProfile = vi.spyOn(repo, "updateProfile");
+    const listFriends = vi.spyOn(repo, "listFriends");
+    const createTournament = vi.spyOn(repo, "createTournament");
+
+    const blocked = await Promise.all([
+      app.inject({ method: "POST", url: "/chat/lobby", headers: { cookie }, payload: { body: "안녕하세요" } }),
+      app.inject({ method: "PATCH", url: "/profile/me", headers: { cookie }, payload: { displayName: "변경" } }),
+      app.inject({ method: "GET", url: "/friends", headers: { cookie } }),
+      app.inject({ method: "POST", url: "/tournaments", headers: { cookie }, payload: { name: "게스트 대회" } })
+    ]);
+    for (const response of blocked) expectApiError(response, 403, "guest_feature_forbidden");
+
+    const admin = await app.inject({ method: "GET", url: "/admin/actions", headers: { cookie } });
+    expectApiError(admin, 404, "not_found");
+    expect(createChat).not.toHaveBeenCalled();
+    expect(updateProfile).not.toHaveBeenCalled();
+    expect(listFriends).not.toHaveBeenCalled();
+    expect(createTournament).not.toHaveBeenCalled();
+  });
+
+  it("issues a one-time websocket ticket from the guest cookie without database storage", async () => {
+    const address = await app.listen({ host: "127.0.0.1", port: 0 });
+    const wsBaseUrl = address.replace(/^http/, "ws");
+    const login = await app.inject({ method: "POST", url: "/auth/guest" });
+    const createTicket = vi.spyOn(repo, "createWsTicket");
+    const ticketResponse = await app.inject({
+      method: "POST",
+      url: "/auth/ws-ticket",
+      headers: { cookie: guestCookie(login) }
+    });
+    expect(ticketResponse.statusCode).toBe(200);
+    const { ticket } = ticketResponse.json<{ ticket: string }>();
+    expect(createTicket).not.toHaveBeenCalled();
+
+    const accepted = new WebSocket(`${wsBaseUrl}/ws?ticket=${ticket}&v=1`);
+    await onceOpen(accepted);
+    await expectStillOpen(accepted);
+    accepted.close(1000, "test complete");
+
+    const reused = new WebSocket(`${wsBaseUrl}/ws?ticket=${ticket}&v=1`);
+    await onceOpen(reused);
+    await expectClose(reused, 1008, "invalid websocket ticket");
+  });
+});
+
+function guestCookie(response: InjectResponse): string {
+  return guestCookieHeader(response).split(";", 1)[0];
+}
+
+function guestCookieHeader(response: InjectResponse): string {
+  const value = response.headers["set-cookie"];
+  const header = Array.isArray(value)
+    ? value.find((item) => item.startsWith("pp_guest="))
+    : typeof value === "string" ? value : undefined;
+  if (!header) throw new Error("pp_guest cookie was not set");
+  return header;
+}
+
+function expectApiError(response: InjectResponse, statusCode: number, code: string): void {
+  expect(response.statusCode).toBe(statusCode);
+  expect(response.json()).toEqual({
+    error: expect.objectContaining({ code, message: expect.any(String), requestId: expect.any(String) })
+  });
+}
+
+function onceOpen(socket: WebSocket): Promise<void> {
+  return new Promise((resolve, reject) => {
+    socket.once("open", resolve);
+    socket.once("error", reject);
+  });
+}
+
+function expectStillOpen(socket: WebSocket): Promise<void> {
+  return new Promise((resolve, reject) => {
+    const timer = setTimeout(resolve, 30);
+    socket.once("close", (code, reason) => {
+      clearTimeout(timer);
+      reject(new Error(`unexpected close: ${code} ${reason.toString("utf8")}`));
+    });
+  });
+}
+
+function expectClose(socket: WebSocket, expectedCode: number, expectedReason: string): Promise<void> {
+  return new Promise((resolve, reject) => {
+    const timer = setTimeout(() => reject(new Error("timed out waiting for close")), 2_000);
+    socket.once("close", (code, reason) => {
+      clearTimeout(timer);
+      try {
+        expect({ code, reason: reason.toString("utf8") }).toEqual({
+          code: expectedCode,
+          reason: expectedReason
+        });
+        resolve();
+      } catch (error) {
+        reject(error);
+      }
+    });
+    socket.once("error", reject);
+  });
+}
diff --git a/apps/api/src/guestAccess.test.ts b/apps/api/src/guestAccess.test.ts
new file mode 100644
index 0000000..436aa56
--- /dev/null
+++ b/apps/api/src/guestAccess.test.ts
@@ -0,0 +1,112 @@
+import { describe, expect, it } from "vitest";
+import { hashWsTicket } from "./wsTicket.js";
+import {
+  DEFAULT_GUEST_CONNECTION_LIMIT,
+  DEFAULT_GUEST_CONNECTIONS_PER_IP,
+  DEFAULT_GUEST_CREATION_LIMIT_PER_MINUTE,
+  GUEST_SESSION_TTL_SECONDS,
+  GuestAccess,
+  GuestAccessError
+} from "./guestAccess.js";
+
+describe("GuestAccess", () => {
+  it("authenticates an HMAC-signed session for two hours and rejects tampering", () => {
+    let nowMs = Date.parse("2026-01-01T00:00:00.000Z");
+    const access = createAccess({ clock: () => nowMs });
+    const created = access.createSession("203.0.113.10");
+
+    expect(created.expiresInSeconds).toBe(GUEST_SESSION_TTL_SECONDS);
+    expect(created.user).toMatchObject({
+      handle: expect.stringMatching(/^guest-[a-f0-9]{12}$/),
+      displayName: expect.stringMatching(/^게스트 [0-9]{4}$/),
+      sessionKind: "guest",
+      role: "user",
+      rating: 1_200
+    });
+    expect(access.authenticate(created.cookieValue)).toEqual(created.user);
+
+    const replacement = created.cookieValue.endsWith("a") ? "b" : "a";
+    const tampered = `${created.cookieValue.slice(0, -1)}${replacement}`;
+    expect(access.authenticate(tampered)).toBeNull();
+
+    nowMs += GUEST_SESSION_TTL_SECONDS * 1_000;
+    expect(access.authenticate(created.cookieValue)).toBeNull();
+  });
+
+  it("allows ten creations per IP in a rolling minute", () => {
+    let nowMs = 1_000_000;
+    const access = createAccess({ clock: () => nowMs });
+
+    for (let count = 0; count < DEFAULT_GUEST_CREATION_LIMIT_PER_MINUTE; count += 1) {
+      expect(access.createSession("203.0.113.20").user.sessionKind).toBe("guest");
+    }
+    expect(() => access.createSession("203.0.113.20")).toThrowError(
+      expect.objectContaining<Partial<GuestAccessError>>({ code: "guest_creation_rate_limited" })
+    );
+
+    nowMs += 60_000;
+    expect(access.createSession("203.0.113.20").user.sessionKind).toBe("guest");
+  });
+
+  it("stores only a one-time hash for 30-second websocket tickets", () => {
+    let nowMs = 2_000_000;
+    const access = createAccess({ clock: () => nowMs });
+    const guest = access.createSession("203.0.113.30").user;
+    const ticket = access.issueWsTicket(guest);
+
+    expect(ticket).toMatch(/^[A-Za-z0-9_-]{43}$/);
+    expect(access.consumeWsTicket(hashWsTicket(ticket))).toEqual(guest);
+    expect(access.consumeWsTicket(hashWsTicket(ticket))).toBeNull();
+
+    const expired = access.issueWsTicket(guest);
+    nowMs += 30_000;
+    expect(access.consumeWsTicket(hashWsTicket(expired))).toBeNull();
+  });
+
+  it("limits guest sockets per IP and across the process", () => {
+    expect(DEFAULT_GUEST_CONNECTIONS_PER_IP).toBe(4);
+    expect(DEFAULT_GUEST_CONNECTION_LIMIT).toBe(200);
+
+    const access = createAccess({ connectionsPerIp: 1, connectionLimit: 2 });
+    const first = access.createSession("198.51.100.1").user;
+    const second = access.createSession("198.51.100.2").user;
+    const third = access.createSession("198.51.100.3").user;
+    const sameIp = access.createSession("198.51.100.1").user;
+
+    const firstLease = access.acquireConnection("198.51.100.1", first.id);
+    expect(firstLease).not.toBeNull();
+    expect(access.acquireConnection("198.51.100.1", sameIp.id)).toBeNull();
+    const secondLease = access.acquireConnection("198.51.100.2", second.id);
+    expect(secondLease).not.toBeNull();
+    expect(access.acquireConnection("198.51.100.3", third.id)).toBeNull();
+
+    firstLease?.release();
+    const thirdLease = access.acquireConnection("198.51.100.3", third.id);
+    expect(thirdLease).not.toBeNull();
+    secondLease?.release();
+    thirdLease?.release();
+    expect(access.activeConnectionCount).toBe(0);
+  });
+
+  it("moves an existing guest lease without counting a replacement twice", () => {
+    const access = createAccess({ connectionsPerIp: 1, connectionLimit: 1 });
+    const guest = access.createSession("192.0.2.10").user;
+    const first = access.acquireConnection("192.0.2.10", guest.id);
+    const replacement = access.acquireConnection("192.0.2.10", guest.id);
+
+    expect(first).not.toBeNull();
+    expect(replacement).not.toBeNull();
+    expect(access.activeConnectionCount).toBe(1);
+    first?.release();
+    expect(access.activeConnectionCount).toBe(1);
+    replacement?.release();
+    expect(access.activeConnectionCount).toBe(0);
+  });
+});
+
+function createAccess(overrides: Partial<ConstructorParameters<typeof GuestAccess>[0]> = {}): GuestAccess {
+  return new GuestAccess({
+    secret: "guest-test-secret-that-is-long-enough",
+    ...overrides
+  });
+}
diff --git a/packages/shared/src/http.test.ts b/packages/shared/src/http.test.ts
index caec71d..6eb2230 100644
--- a/packages/shared/src/http.test.ts
+++ b/packages/shared/src/http.test.ts
@@ -3,6 +3,7 @@ import {
   apiErrorBodySchema,
   chatBodySchema,
   devLoginBodySchema,
+  guestAuthResponseSchema,
   idParamsSchema,
   profileUpdateBodySchema,
   sessionUserSchema,
@@ -79,6 +80,17 @@ describe("HTTP contracts", () => {
     expect(wsTicketResponseSchema.safeParse({ ...response, protocolVersion: 2 }).success).toBe(false);
   });
 
+  it("keeps the guest session lifetime explicit", () => {
+    const response = {
+      user: { ...user, handle: "guest-018f4af4", displayName: "게스트 7050", online: true },
+      guest: true,
+      expiresInSeconds: 7_200
+    } as const;
+
+    expect(guestAuthResponseSchema.parse(response)).toEqual(response);
+    expect(guestAuthResponseSchema.safeParse({ ...response, expiresInSeconds: 3_600 }).success).toBe(false);
+  });
+
   it("accepts only a one-time ticket and protocol v1 in websocket query parameters", () => {
     const query = { ticket: "a".repeat(43), v: "1" } as const;
 


