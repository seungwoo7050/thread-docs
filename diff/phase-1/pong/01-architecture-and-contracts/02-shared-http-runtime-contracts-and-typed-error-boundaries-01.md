# 공유 HTTP 런타임 계약과 타입드 오류 경계

## `feat(shared): 사용자 HTTP runtime contract 정의`

diff --git a/packages/shared/src/http.ts b/packages/shared/src/http.ts
index 4cc1348..bc4fd31 100644
--- a/packages/shared/src/http.ts
+++ b/packages/shared/src/http.ts
@@ -1,26 +1,37 @@
-export type UserRole = "user" | "admin";
-export type UserStatus = "active" | "banned";
-export type FriendshipStatus = "pending" | "accepted";
-export type TournamentStatus = "open" | "running" | "finished";
-export type MatchMode = "queue" | "ai" | "tournament";
+import { z } from "zod";
 
-export interface PublicUser {
-  id: string;
-  handle: string;
-  displayName: string;
-  avatarKey: string;
-  role: UserRole;
-  status: UserStatus;
-  rating: number;
-  wins: number;
-  losses: number;
-  online: boolean;
-  isNpc: boolean;
-}
+export const userRoleSchema = z.enum(["user", "admin"]);
+export const userStatusSchema = z.enum(["active", "banned"]);
+export const friendshipStatusSchema = z.enum(["pending", "accepted"]);
+export const tournamentStatusSchema = z.enum(["open", "running", "finished"]);
+export const matchModeSchema = z.enum(["queue", "ai", "tournament"]);
 
-export interface SessionUser extends PublicUser {
-  email: string | null;
-}
+export type UserRole = z.infer<typeof userRoleSchema>;
+export type UserStatus = z.infer<typeof userStatusSchema>;
+export type FriendshipStatus = z.infer<typeof friendshipStatusSchema>;
+export type TournamentStatus = z.infer<typeof tournamentStatusSchema>;
+export type MatchMode = z.infer<typeof matchModeSchema>;
+
+export const publicUserSchema = z.object({
+  id: z.string().uuid(),
+  handle: z.string().min(1),
+  displayName: z.string().min(1),
+  avatarKey: z.string(),
+  role: userRoleSchema,
+  status: userStatusSchema,
+  rating: z.number().int(),
+  wins: z.number().int().nonnegative(),
+  losses: z.number().int().nonnegative(),
+  online: z.boolean(),
+  isNpc: z.boolean()
+});
+
+export const sessionUserSchema = publicUserSchema.extend({
+  email: z.string().email().nullable()
+});
+
+export type PublicUser = z.infer<typeof publicUserSchema>;
+export type SessionUser = z.infer<typeof sessionUserSchema>;
 
 export interface MatchSummary {
   id: string;


## `feat(shared): 경기·대시보드 runtime contract 정의`

diff --git a/packages/shared/src/http.ts b/packages/shared/src/http.ts
index bc4fd31..74afba8 100644
--- a/packages/shared/src/http.ts
+++ b/packages/shared/src/http.ts
@@ -33,29 +33,35 @@ export const sessionUserSchema = publicUserSchema.extend({
 export type PublicUser = z.infer<typeof publicUserSchema>;
 export type SessionUser = z.infer<typeof sessionUserSchema>;
 
-export interface MatchSummary {
-  id: string;
-  mode: MatchMode;
-  opponentHandle: string;
-  result: "win" | "loss";
-  scoreLeft: number;
-  scoreRight: number;
-  ratingDelta: number;
-  endedAt: string;
-}
+export const matchSummarySchema = z.object({
+  id: z.string().uuid(),
+  mode: matchModeSchema,
+  opponentHandle: z.string().min(1),
+  result: z.enum(["win", "loss"]),
+  scoreLeft: z.number().int().nonnegative(),
+  scoreRight: z.number().int().nonnegative(),
+  ratingDelta: z.number().int(),
+  endedAt: z.string().datetime()
+});
 
-export interface DashboardSummary {
-  me: SessionUser;
-  recentMatches: MatchSummary[];
-  winRate: number;
-  bestStreak: number;
-}
+export type MatchSummary = z.infer<typeof matchSummarySchema>;
 
-export interface LeaderboardEntry {
-  rank: number;
-  user: PublicUser;
-  winRate: number;
-}
+export const dashboardSummarySchema = z.object({
+  me: sessionUserSchema,
+  recentMatches: z.array(matchSummarySchema),
+  winRate: z.number().min(0).max(100),
+  bestStreak: z.number().int().nonnegative()
+});
+
+export type DashboardSummary = z.infer<typeof dashboardSummarySchema>;
+
+export const leaderboardEntrySchema = z.object({
+  rank: z.number().int().positive(),
+  user: publicUserSchema,
+  winRate: z.number().min(0).max(100)
+});
+
+export type LeaderboardEntry = z.infer<typeof leaderboardEntrySchema>;
 
 export interface FriendSummary {
   id: string;


## `feat(shared): 친구·채팅·로비 runtime contract 정의`

diff --git a/packages/shared/src/http.ts b/packages/shared/src/http.ts
index 74afba8..2412f29 100644
--- a/packages/shared/src/http.ts
+++ b/packages/shared/src/http.ts
@@ -63,36 +63,44 @@ export const leaderboardEntrySchema = z.object({
 
 export type LeaderboardEntry = z.infer<typeof leaderboardEntrySchema>;
 
-export interface FriendSummary {
-  id: string;
-  user: PublicUser;
-  status: FriendshipStatus;
-}
+export const friendSummarySchema = z.object({
+  id: z.string().uuid(),
+  user: publicUserSchema,
+  status: friendshipStatusSchema
+});
 
-export interface ChatMessage {
-  id: string;
-  scope: "lobby" | "match";
-  roomId: string | null;
-  sender: PublicUser;
-  body: string;
-  createdAt: string;
-}
+export type FriendSummary = z.infer<typeof friendSummarySchema>;
 
-export interface LobbyStats {
-  onlinePlayers: number;
-  playingPlayers: number;
-  queuedPlayers: number;
-  activeRooms: number;
-  averageWaitSeconds: number | null;
-}
+export const chatMessageSchema = z.object({
+  id: z.string().uuid(),
+  scope: z.enum(["lobby", "match"]),
+  roomId: z.string().uuid().nullable(),
+  sender: publicUserSchema,
+  body: z.string().min(1).max(240),
+  createdAt: z.string().datetime()
+});
 
-export interface LobbyResponse {
-  me: SessionUser | null;
-  onlinePlayers: PublicUser[];
-  recentMatches: MatchSummary[];
-  chat: ChatMessage[];
-  stats: LobbyStats;
-}
+export type ChatMessage = z.infer<typeof chatMessageSchema>;
+
+export const lobbyStatsSchema = z.object({
+  onlinePlayers: z.number().int().nonnegative(),
+  playingPlayers: z.number().int().nonnegative(),
+  queuedPlayers: z.number().int().nonnegative(),
+  activeRooms: z.number().int().nonnegative(),
+  averageWaitSeconds: z.number().nonnegative().nullable()
+});
+
+export type LobbyStats = z.infer<typeof lobbyStatsSchema>;
+
+export const lobbyResponseSchema = z.object({
+  me: sessionUserSchema.nullable(),
+  onlinePlayers: z.array(publicUserSchema),
+  recentMatches: z.array(matchSummarySchema),
+  chat: z.array(chatMessageSchema),
+  stats: lobbyStatsSchema
+});
+
+export type LobbyResponse = z.infer<typeof lobbyResponseSchema>;
 
 export interface TournamentSummary {
   id: string;


## `feat(shared): 토너먼트·관리 runtime contract 정의`

diff --git a/packages/shared/src/http.ts b/packages/shared/src/http.ts
index 2412f29..75f84cc 100644
--- a/packages/shared/src/http.ts
+++ b/packages/shared/src/http.ts
@@ -102,38 +102,44 @@ export const lobbyResponseSchema = z.object({
 
 export type LobbyResponse = z.infer<typeof lobbyResponseSchema>;
 
-export interface TournamentSummary {
-  id: string;
-  name: string;
-  status: TournamentStatus;
-  createdBy: PublicUser;
-  playerCount: number;
-  capacity: number;
-  winner: PublicUser | null;
-  entries: PublicUser[];
-  matches: TournamentMatchSummary[];
-}
-
-export interface TournamentMatchSummary {
-  id: string;
-  tournamentId: string;
-  round: "semifinal" | "final";
-  slot: number;
-  status: "pending" | "ready" | "running" | "finished";
-  left: PublicUser | null;
-  right: PublicUser | null;
-  winner: PublicUser | null;
-  scoreLeft: number | null;
-  scoreRight: number | null;
-  roomId: string | null;
-  matchId: string | null;
-}
-
-export interface AdminActionSummary {
-  id: string;
-  actor: PublicUser | null;
-  target: PublicUser | null;
-  action: "ban" | "unban";
-  reason: string;
-  createdAt: string;
-}
+export const tournamentMatchSummarySchema = z.object({
+  id: z.string().uuid(),
+  tournamentId: z.string().uuid(),
+  round: z.enum(["semifinal", "final"]),
+  slot: z.number().int().nonnegative(),
+  status: z.enum(["pending", "ready", "running", "finished"]),
+  left: publicUserSchema.nullable(),
+  right: publicUserSchema.nullable(),
+  winner: publicUserSchema.nullable(),
+  scoreLeft: z.number().int().nonnegative().nullable(),
+  scoreRight: z.number().int().nonnegative().nullable(),
+  roomId: z.string().uuid().nullable(),
+  matchId: z.string().uuid().nullable()
+});
+
+export type TournamentMatchSummary = z.infer<typeof tournamentMatchSummarySchema>;
+
+export const tournamentSummarySchema = z.object({
+  id: z.string().uuid(),
+  name: z.string().min(1),
+  status: tournamentStatusSchema,
+  createdBy: publicUserSchema,
+  playerCount: z.number().int().nonnegative(),
+  capacity: z.number().int().positive(),
+  winner: publicUserSchema.nullable(),
+  entries: z.array(publicUserSchema),
+  matches: z.array(tournamentMatchSummarySchema)
+});
+
+export type TournamentSummary = z.infer<typeof tournamentSummarySchema>;
+
+export const adminActionSummarySchema = z.object({
+  id: z.string().uuid(),
+  actor: publicUserSchema.nullable(),
+  target: publicUserSchema.nullable(),
+  action: z.enum(["ban", "unban"]),
+  reason: z.string(),
+  createdAt: z.string().datetime()
+});
+
+export type AdminActionSummary = z.infer<typeof adminActionSummarySchema>;


## `feat(shared): HTTP 요청·오류 schema 정의`

diff --git a/packages/shared/src/http.ts b/packages/shared/src/http.ts
index 75f84cc..f2c0e6f 100644
--- a/packages/shared/src/http.ts
+++ b/packages/shared/src/http.ts
@@ -143,3 +143,42 @@ export const adminActionSummarySchema = z.object({
 });
 
 export type AdminActionSummary = z.infer<typeof adminActionSummarySchema>;
+
+export const apiErrorBodySchema = z.object({
+  error: z.object({
+    code: z.string().min(1),
+    message: z.string().min(1),
+    requestId: z.string().min(1),
+    fieldErrors: z.record(z.array(z.string())).optional()
+  })
+});
+
+export type ApiErrorBody = z.infer<typeof apiErrorBodySchema>;
+
+export const emptyParamsSchema = z.object({}).strict();
+export const idParamsSchema = z.object({ id: z.string().uuid() }).strict();
+export const handleParamsSchema = z.object({ handle: z.string().min(1).max(64) }).strict();
+
+export const devLoginBodySchema = z.object({
+  handle: z.string().trim().min(2).max(24).regex(/^[a-z0-9][a-z0-9-]*$/),
+  displayName: z.string().trim().min(1).max(40),
+  email: z.string().trim().email().optional()
+}).strict();
+
+export const chatBodySchema = z.object({ body: z.string().trim().min(1).max(240) }).strict();
+export const profileUpdateBodySchema = z.object({
+  displayName: z.string().trim().min(1).max(40).optional(),
+  avatarKey: z.string().trim().min(1).max(120).optional()
+}).strict().refine((body) => body.displayName !== undefined || body.avatarKey !== undefined, {
+  message: "변경할 프로필 값을 입력해주세요."
+});
+export const friendRequestBodySchema = z.object({ handle: z.string().trim().min(1).max(64) }).strict();
+export const tournamentCreateBodySchema = z.object({ name: z.string().trim().min(1).max(80) }).strict();
+export const adminBanBodySchema = z.object({
+  banned: z.boolean().optional(),
+  reason: z.string().trim().min(1).max(240).optional()
+}).strict();
+export const adminStatusBodySchema = z.object({
+  status: userStatusSchema,
+  reason: z.string().trim().min(1).max(240).optional()
+}).strict();


## `feat(shared): HTTP 응답 runtime contract 정의`

diff --git a/packages/shared/src/http.ts b/packages/shared/src/http.ts
index f2c0e6f..58290e6 100644
--- a/packages/shared/src/http.ts
+++ b/packages/shared/src/http.ts
@@ -182,3 +182,27 @@ export const adminStatusBodySchema = z.object({
   status: userStatusSchema,
   reason: z.string().trim().min(1).max(240).optional()
 }).strict();
+
+export const okResponseSchema = z.object({ ok: z.literal(true) });
+export const healthResponseSchema = z.object({ ok: z.literal(true), service: z.literal("pong-pong-api") });
+export const userResponseSchema = z.object({ user: sessionUserSchema });
+export const publicUserResponseSchema = z.object({ user: publicUserSchema });
+export const profileResponseSchema = z.object({ user: publicUserSchema, recentMatches: z.array(matchSummarySchema) });
+export const ownProfileResponseSchema = z.object({ profile: sessionUserSchema });
+export const friendsResponseSchema = z.object({ friends: z.array(friendSummarySchema) });
+export const friendResponseSchema = z.object({ friend: friendSummarySchema });
+export const chatResponseSchema = z.object({ message: chatMessageSchema });
+export const leaderboardResponseSchema = z.object({ entries: z.array(leaderboardEntrySchema) });
+export const tournamentsResponseSchema = z.object({ tournaments: z.array(tournamentSummarySchema) });
+export const tournamentResponseSchema = z.object({ tournament: tournamentSummarySchema });
+export const adminUsersResponseSchema = z.object({ users: z.array(publicUserSchema) });
+export const adminActionsResponseSchema = z.object({ actions: z.array(adminActionSummarySchema) });
+export const wsTicketResponseSchema = z.object({
+  ticket: z.string().min(32),
+  expiresInSeconds: z.literal(30),
+  protocolVersion: z.literal(1)
+});
+
+export type DevLoginBody = z.infer<typeof devLoginBodySchema>;
+export type ProfileUpdateBody = z.infer<typeof profileUpdateBodySchema>;
+export type WsTicketResponse = z.infer<typeof wsTicketResponseSchema>;


## `feat(api): typed HTTP 오류 boundary 추가`

diff --git a/apps/api/src/httpBoundary.ts b/apps/api/src/httpBoundary.ts
new file mode 100644
index 0000000..264815f
--- /dev/null
+++ b/apps/api/src/httpBoundary.ts
@@ -0,0 +1,91 @@
+import type { FastifyReply, FastifyRequest } from "fastify";
+import { apiErrorBodySchema, type ApiErrorBody } from "@pong-pong/shared";
+import type { ZodType } from "zod";
+
+export class ApiHttpError extends Error {
+  constructor(
+    readonly statusCode: number,
+    readonly code: string,
+    message: string,
+    readonly fieldErrors?: Record<string, string[]>
+  ) {
+    super(message);
+    this.name = "ApiHttpError";
+  }
+}
+
+export function parseInput<T>(schema: ZodType<T>, input: unknown): T {
+  const result = schema.safeParse(input);
+  if (result.success) return result.data;
+
+  const fieldErrors: Record<string, string[]> = {};
+  for (const issue of result.error.issues) {
+    const field = issue.path.join(".") || "request";
+    (fieldErrors[field] ??= []).push(issue.message);
+  }
+  throw new ApiHttpError(
+    400,
+    "validation_failed",
+    "입력값을 확인해주세요.",
+    Object.keys(fieldErrors).length > 0 ? fieldErrors : undefined
+  );
+}
+
+export function parseOutput<T>(schema: ZodType<T>, output: unknown): T {
+  const result = schema.safeParse(output);
+  if (result.success) return result.data;
+
+  throw new Error("HTTP response contract validation failed", { cause: result.error });
+}
+
+export function sendApiError(
+  reply: FastifyReply,
+  request: FastifyRequest,
+  statusCode: number,
+  code: string,
+  message: string,
+  fieldErrors?: Record<string, string[]>
+): FastifyReply {
+  const body: ApiErrorBody = {
+    error: {
+      code,
+      message,
+      requestId: String(request.id),
+      ...(fieldErrors ? { fieldErrors } : {})
+    }
+  };
+
+  return reply.code(statusCode).send(apiErrorBodySchema.parse(body));
+}
+
+export function installHttpErrorBoundary(app: import("fastify").FastifyInstance): void {
+  app.setNotFoundHandler((request, reply) => {
+    sendApiError(reply, request, 404, "not_found", "요청한 경로를 찾을 수 없습니다.");
+  });
+
+  app.setErrorHandler((error, request, reply) => {
+    if (error instanceof ApiHttpError) {
+      sendApiError(reply, request, error.statusCode, error.code, error.message, error.fieldErrors);
+      return;
+    }
+
+    request.log.error({ err: error }, "request failed");
+    sendApiError(reply, request, 500, "internal_error", "요청을 처리하지 못했습니다.");
+  });
+}
+
+export function unauthorized(): never {
+  throw new ApiHttpError(401, "authentication_required", "로그인이 필요합니다.");
+}
+
+export function suspended(): never {
+  throw new ApiHttpError(403, "account_suspended", "정지된 계정은 이 작업을 수행할 수 없습니다.");
+}
+
+export function forbidden(): never {
+  throw new ApiHttpError(403, "admin_required", "운영자 권한이 필요합니다.");
+}
+
+export function notFound(message: string): never {
+  throw new ApiHttpError(404, "not_found", message);
+}


## `feat(api): 인증·사용자 HTTP contract 적용`

diff --git a/apps/api/src/app.ts b/apps/api/src/app.ts
index d66c99a..fdf3d62 100644
--- a/apps/api/src/app.ts
+++ b/apps/api/src/app.ts
@@ -3,9 +3,17 @@ import cors from "@fastify/cors";
 import websocket from "@fastify/websocket";
 import Fastify, { type FastifyReply, type FastifyRequest } from "fastify";
 import type { AppRepository } from "@pong-pong/db";
+import * as http from "@pong-pong/shared";
 import type { SessionUser } from "@pong-pong/shared";
 import type { WebSocket } from "ws";
 import { GameHub } from "./gameHub";
+import {
+  installHttpErrorBoundary,
+  notFound,
+  parseInput,
+  parseOutput,
+  unauthorized as raiseUnauthorized
+} from "./httpBoundary";
 
 export interface BuildAppOptions {
   repo: AppRepository;
@@ -16,11 +24,12 @@ export function buildApp({ repo, webOrigin }: BuildAppOptions) {
   const app = Fastify({ logger: { level: process.env.LOG_LEVEL ?? "info" } });
   const hub = new GameHub(repo);
 
+  installHttpErrorBoundary(app);
   app.register(cors, {
     origin: [webOrigin, "http://localhost:3000", "http://localhost:8080"],
     credentials: true,
     methods: ["GET", "POST", "PATCH", "DELETE", "OPTIONS"],
-    allowedHeaders: ["content-type", "authorization"]
+    allowedHeaders: ["content-type", "authorization", "x-request-id"]
   });
   app.register(cookie);
   app.register(async (realtime) => {
@@ -46,15 +55,14 @@ export function buildApp({ repo, webOrigin }: BuildAppOptions) {
     });
   });
 
-  app.get("/health", async () => ({ ok: true, service: "pong-pong-api" }));
+  app.get("/health", async () => parseOutput(http.healthResponseSchema, {
+    ok: true,
+    service: "pong-pong-api"
+  }));
 
   app.post("/auth/dev-login", async (request, reply) => {
-    const body = request.body as { handle?: string; displayName?: string; email?: string };
-    const user = await repo.upsertDevUser({
-      handle: body.handle ?? "player",
-      displayName: body.displayName ?? body.handle ?? "플레이어",
-      email: body.email
-    });
+    const body = parseInput(http.devLoginBodySchema, request.body);
+    const user = await repo.upsertDevUser(body);
     const token = await repo.createSession(user.id);
     reply.setCookie("pp_session", token, {
       path: "/",
@@ -62,32 +70,32 @@ export function buildApp({ repo, webOrigin }: BuildAppOptions) {
       httpOnly: true,
       maxAge: 60 * 60 * 24 * 14
     });
-    return { user, token };
+    return parseOutput(http.userResponseSchema, { user });
   });
 
   app.post("/auth/logout", async (request, reply) => {
     await repo.deleteSession(readSessionToken(request));
     reply.clearCookie("pp_session", { path: "/" });
-    return { ok: true };
+    return parseOutput(http.okResponseSchema, { ok: true });
   });
 
-  app.get("/me", async (request, reply) => {
+  app.get("/me", async (request) => {
     const user = await currentUser(repo, request);
-    if (!user) return unauthorized(reply);
-    return { user };
+    if (!user) raiseUnauthorized();
+    return parseOutput(http.userResponseSchema, { user });
   });
 
-  app.get("/auth/me", async (request, reply) => {
+  app.get("/auth/me", async (request) => {
     const user = await currentUser(repo, request);
-    if (!user) return unauthorized(reply);
-    return { user };
+    if (!user) raiseUnauthorized();
+    return parseOutput(http.userResponseSchema, { user });
   });
 
-  app.get("/users/:id", async (request, reply) => {
-    const { id } = request.params as { id: string };
+  app.get("/users/:id", async (request) => {
+    const { id } = parseInput(http.idParamsSchema, request.params);
     const user = await repo.getUserById(id);
-    if (!user) return reply.code(404).send({ message: "not_found" });
-    return { user };
+    if (!user) notFound("사용자를 찾을 수 없습니다.");
+    return parseOutput(http.publicUserResponseSchema, { user });
   });
 
   app.get("/lobby", async (request) => {
@@ -127,32 +135,29 @@ export function buildApp({ repo, webOrigin }: BuildAppOptions) {
     return await repo.getDashboard(user.id);
   });
 
-  app.get("/profile/:handle", async (request, reply) => {
-    const { handle } = request.params as { handle: string };
+  app.get("/profile/:handle", async (request) => {
+    const { handle } = parseInput(http.handleParamsSchema, request.params);
     const user = await repo.getUserByHandle(handle);
-    if (!user) return reply.code(404).send({ message: "프로필을 찾을 수 없습니다." });
-    return {
+    if (!user) notFound("프로필을 찾을 수 없습니다.");
+    return parseOutput(http.profileResponseSchema, {
       user,
       recentMatches: await repo.listRecentMatches(user.id)
-    };
+    });
   });
 
-  app.get("/profile/me", async (request, reply) => {
+  app.get("/profile/me", async (request) => {
     const user = await currentUser(repo, request);
-    if (!user) return unauthorized(reply);
-    return { profile: user };
+    if (!user) raiseUnauthorized();
+    return parseOutput(http.ownProfileResponseSchema, { profile: user });
   });
 
-  app.patch("/profile/me", async (request, reply) => {
+  app.patch("/profile/me", async (request) => {
     const user = await currentUser(repo, request);
-    if (!user) return unauthorized(reply);
-    const body = request.body as { displayName?: string; avatarKey?: string };
-    return {
-      profile: await repo.updateProfile(user.id, {
-        displayName: body.displayName,
-        avatarKey: body.avatarKey
-      })
-    };
+    if (!user) raiseUnauthorized();
+    const body = parseInput(http.profileUpdateBodySchema, request.body);
+    return parseOutput(http.ownProfileResponseSchema, {
+      profile: await repo.updateProfile(user.id, body)
+    });
   });
 
   app.get("/friends", async (request, reply) => {


