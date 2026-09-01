# 준비 상태·메트릭·데이터베이스 장애 관측

## `feat(db): migration set 상태 검사 추가`

diff --git a/packages/db/src/migrator.ts b/packages/db/src/migrator.ts
index 05aef2e..b2a014d 100644
--- a/packages/db/src/migrator.ts
+++ b/packages/db/src/migrator.ts
@@ -12,18 +12,20 @@ import {
 import { Pool } from "pg";
 import type { Database } from "./schema.js";
 
-const migrationsDirectory = fileURLToPath(
-  new URL("../migrations", import.meta.url)
-);
+const migrationDirectoryCandidates = [
+  fileURLToPath(new URL("./migrations", import.meta.url)),
+  fileURLToPath(new URL("../migrations", import.meta.url))
+];
 
 class SqlMigrationProvider implements MigrationProvider {
   async getMigrations(): Promise<Record<string, Migration>> {
-    const filenames = (await readdir(migrationsDirectory))
+    const { directory, filenames } = await findMigrationFiles();
+    const migrationFilenames = filenames
       .filter((filename) => extname(filename) === ".sql")
       .sort();
     const migrations = await Promise.all(
-      filenames.map(async (filename) => {
-        const statement = await readFile(join(migrationsDirectory, filename), "utf8");
+      migrationFilenames.map(async (filename) => {
+        const statement = await readFile(join(directory, filename), "utf8");
         return [
           basename(filename, ".sql"),
           {
@@ -39,6 +41,56 @@ class SqlMigrationProvider implements MigrationProvider {
   }
 }
 
+async function findMigrationFiles(): Promise<{ directory: string; filenames: string[] }> {
+  let lastError: unknown;
+  for (const directory of migrationDirectoryCandidates) {
+    try {
+      return { directory, filenames: await readdir(directory) };
+    } catch (error) {
+      lastError = error;
+    }
+  }
+  throw new Error("Bundled database migrations were not found", { cause: lastError });
+}
+
+export interface MigrationSetComparison {
+  status: "current" | "pending" | "diverged";
+  missing: string[];
+  unexpected: string[];
+}
+
+export function compareMigrationSets(
+  expectedNames: string[],
+  appliedNames: string[]
+): MigrationSetComparison {
+  const expected = new Set(expectedNames);
+  const applied = new Set(appliedNames);
+  const missing = expectedNames.filter((name) => !applied.has(name));
+  const unexpected = appliedNames.filter((name) => !expected.has(name));
+  return {
+    status: unexpected.length > 0 ? "diverged" : missing.length > 0 ? "pending" : "current",
+    missing,
+    unexpected
+  };
+}
+
+export async function inspectMigrationSet(
+  db: Kysely<Database>
+): Promise<MigrationSetComparison> {
+  const expectedNames = Object.keys(await new SqlMigrationProvider().getMigrations()).sort();
+  let appliedNames: string[];
+  try {
+    const applied = await sql<{ name: string }>`
+      select name from kysely_migration order by name
+    `.execute(db);
+    appliedNames = applied.rows.map((row) => row.name);
+  } catch (error) {
+    if (!isUndefinedTableError(error)) throw error;
+    appliedNames = [];
+  }
+  return compareMigrationSets(expectedNames, appliedNames);
+}
+
 export async function migrateDatabase(databaseUrl: string): Promise<void> {
   const pool = new Pool({ connectionString: databaseUrl });
   const db = new Kysely<Database>({ dialect: new PostgresDialect({ pool }) });
@@ -59,3 +111,7 @@ export async function migrateDatabase(databaseUrl: string): Promise<void> {
     await db.destroy();
   }
 }
+
+function isUndefinedTableError(error: unknown): boolean {
+  return typeof error === "object" && error !== null && "code" in error && error.code === "42P01";
+}


## `feat(db): repository readiness 경계 추가`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index 459e032..8857bce 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -39,6 +39,7 @@ import type {
   UserProjectionRow,
   UserRow
 } from "./schema.js";
+import { inspectMigrationSet } from "./migrator.js";
 
 export type { Database } from "./schema.js";
 
@@ -74,6 +75,11 @@ export interface CreateWsTicketInput {
 
 export type SeedProfile = "development" | "demo";
 
+export interface RepositoryReadiness {
+  database: "up";
+  migrations: "current" | "pending" | "diverged" | "not_applicable";
+}
+
 type NpcSeed = {
   handle: string;
   displayName: string;
@@ -123,6 +129,7 @@ export interface TournamentMatchRecord {
 
 export interface AppRepository {
   close(): Promise<void>;
+  checkReadiness(): Promise<RepositoryReadiness>;
   ensureSeedData(profile?: SeedProfile): Promise<void>;
   upsertDevUser(input: DevLoginInput): Promise<SessionUser>;
   createSession(userId: string): Promise<string>;
@@ -178,6 +185,15 @@ class PostgresRepository implements AppRepository {
     await this.pool.end().catch(() => undefined);
   }
 
+  async checkReadiness(): Promise<RepositoryReadiness> {
+    await sql<{ ok: number }>`select 1 as ok`.execute(this.db);
+    const migrationSet = await inspectMigrationSet(this.db);
+    return {
+      database: "up",
+      migrations: migrationSet.status
+    };
+  }
+
   async ensureSeedData(profile: SeedProfile = "development"): Promise<void> {
     if (profile === "development") {
       const players: DevLoginInput[] = [
@@ -907,6 +923,10 @@ class MemoryRepository implements AppRepository {
 
   async close(): Promise<void> {}
 
+  async checkReadiness(): Promise<RepositoryReadiness> {
+    return { database: "up", migrations: "not_applicable" };
+  }
+
   async ensureSeedData(profile: SeedProfile = "development"): Promise<void> {
     if (profile === "development") {
       for (const player of [


## `feat(ops): liveness와 readiness endpoint 추가`

diff --git a/apps/api/src/app.ts b/apps/api/src/app.ts
index 06c61c8..1ccaaf2 100644
--- a/apps/api/src/app.ts
+++ b/apps/api/src/app.ts
@@ -144,6 +144,37 @@ export function buildApp({
     service: "pong-pong-api"
   }));
 
+  app.get("/health/live", async () => parseOutput(http.liveHealthResponseSchema, {
+    status: "ok",
+    service: "pong-pong-api"
+  }));
+
+  app.get("/health/ready", async (request, reply) => {
+    try {
+      const repository = await repo.checkReadiness();
+      const ready = repository.database === "up"
+        && (repository.migrations === "current" || repository.migrations === "not_applicable");
+      const body = parseOutput(http.readyHealthResponseSchema, {
+        status: ready ? "ready" : "not_ready",
+        service: "pong-pong-api",
+        checks: {
+          lifecycle: "accepting",
+          database: repository.database,
+          migrations: repository.migrations
+        }
+      });
+      return reply.code(ready ? 200 : 503).send(body);
+    } catch (error) {
+      request.log.warn({ errorName: error instanceof Error ? error.name : "UnknownError" }, "readiness check failed");
+      const body = parseOutput(http.readyHealthResponseSchema, {
+        status: "not_ready",
+        service: "pong-pong-api",
+        checks: { lifecycle: "accepting", database: "down", migrations: "unknown" }
+      });
+      return reply.code(503).send(body);
+    }
+  });
+
   if (appMode === "development" || appMode === "test") {
     app.post("/auth/dev-login", async (request, reply) => {
       const body = parseInput(http.devLoginBodySchema, request.body);
diff --git a/packages/shared/src/http.ts b/packages/shared/src/http.ts
index 72b6ee4..b63d418 100644
--- a/packages/shared/src/http.ts
+++ b/packages/shared/src/http.ts
@@ -191,6 +191,19 @@ export const wsHandshakeQuerySchema = z.object({
 
 export const okResponseSchema = z.object({ ok: z.literal(true) });
 export const healthResponseSchema = z.object({ ok: z.literal(true), service: z.literal("pong-pong-api") });
+export const liveHealthResponseSchema = z.object({
+  status: z.literal("ok"),
+  service: z.literal("pong-pong-api")
+});
+export const readyHealthResponseSchema = z.object({
+  status: z.enum(["ready", "not_ready"]),
+  service: z.literal("pong-pong-api"),
+  checks: z.object({
+    lifecycle: z.enum(["accepting", "draining"]),
+    database: z.enum(["up", "down"]),
+    migrations: z.enum(["current", "pending", "diverged", "not_applicable", "unknown"])
+  }).strict()
+}).strict();
 export const userResponseSchema = z.object({ user: sessionUserSchema });
 export const guestAuthResponseSchema = z.object({
   user: sessionUserSchema,


## `feat(metrics): runtime gauge registry 추가`

diff --git a/apps/api/src/observability.ts b/apps/api/src/observability.ts
new file mode 100644
index 0000000..f6a8731
--- /dev/null
+++ b/apps/api/src/observability.ts
@@ -0,0 +1,54 @@
+import {
+  Gauge,
+  Registry,
+  collectDefaultMetrics
+} from "prom-client";
+
+interface LiveGameStats {
+  onlinePlayers: number;
+  queuedPlayers: number;
+  activeRooms: number;
+}
+
+export class ApiMetrics {
+  private readonly registry = new Registry();
+  private readonly connections = new Gauge({
+    name: "pong_pong_api_connections",
+    help: "Current websocket connection count",
+    registers: [this.registry]
+  });
+  private readonly queuedPlayers = new Gauge({
+    name: "pong_pong_api_queued_players",
+    help: "Current matchmaking queue size",
+    registers: [this.registry]
+  });
+  private readonly rooms = new Gauge({
+    name: "pong_pong_api_rooms",
+    help: "Current game room count",
+    registers: [this.registry]
+  });
+
+  constructor(private readonly readGameStats: () => LiveGameStats) {
+    collectDefaultMetrics({
+      register: this.registry,
+      prefix: "pong_pong_api_",
+      eventLoopMonitoringPrecision: 20
+    });
+  }
+
+  get contentType(): string {
+    return this.registry.contentType;
+  }
+
+  async scrape(): Promise<string> {
+    const stats = this.readGameStats();
+    this.connections.set(stats.onlinePlayers);
+    this.queuedPlayers.set(stats.queuedPlayers);
+    this.rooms.set(stats.activeRooms);
+    return this.registry.metrics();
+  }
+
+  close(): void {
+    this.registry.clear();
+  }
+}


## `feat(metrics): HTTP와 readiness 측정 추가`

diff --git a/apps/api/src/app.ts b/apps/api/src/app.ts
index 1ccaaf2..5feb6b3 100644
--- a/apps/api/src/app.ts
+++ b/apps/api/src/app.ts
@@ -26,6 +26,7 @@ import {
 import { createLoggerOptions } from "./requestLogging.js";
 import { readAppMode } from "./env.js";
 import { createRawWsTicket, hashWsTicket, WS_TICKET_TTL_SECONDS } from "./wsTicket.js";
+import { ApiMetrics } from "./observability.js";
 
 const WS_POLICY_VIOLATION = 1008;
 const WS_MESSAGE_TOO_BIG = 1009;
@@ -57,9 +58,23 @@ export function buildApp({
     trustProxy
   });
   const hub = new GameHub(repo);
+  const metrics = new ApiMetrics(() => hub.liveStats());
   const guests = appMode === "demo" ? guestAccess ?? new GuestAccess({ secret: sessionSecret }) : null;
   const getCurrentUser = (request: FastifyRequest) => currentUser(repo, request, guests, appMode === "demo");
 
+  app.addHook("onResponse", (request, reply, done) => {
+    metrics.observeRequest(
+      request.method,
+      request.routeOptions.url ?? "unmatched",
+      reply.statusCode,
+      reply.elapsedTime
+    );
+    done();
+  });
+  app.addHook("onClose", async () => {
+    metrics.close();
+  });
+
   installHttpErrorBoundary(app);
   app.register(cors, {
     origin: [webOrigin, "http://localhost:3000", "http://localhost:8080"],
@@ -150,6 +165,7 @@ export function buildApp({
   }));
 
   app.get("/health/ready", async (request, reply) => {
+    const startedAt = performance.now();
     try {
       const repository = await repo.checkReadiness();
       const ready = repository.database === "up"
@@ -163,6 +179,7 @@ export function buildApp({
           migrations: repository.migrations
         }
       });
+      metrics.observeReadiness(body.status, performance.now() - startedAt);
       return reply.code(ready ? 200 : 503).send(body);
     } catch (error) {
       request.log.warn({ errorName: error instanceof Error ? error.name : "UnknownError" }, "readiness check failed");
@@ -171,10 +188,16 @@ export function buildApp({
         service: "pong-pong-api",
         checks: { lifecycle: "accepting", database: "down", migrations: "unknown" }
       });
+      metrics.observeReadiness("not_ready", performance.now() - startedAt);
       return reply.code(503).send(body);
     }
   });
 
+  app.get("/metrics", async (_request, reply) => {
+    reply.header("content-type", metrics.contentType);
+    return reply.send(await metrics.scrape());
+  });
+
   if (appMode === "development" || appMode === "test") {
     app.post("/auth/dev-login", async (request, reply) => {
       const body = parseInput(http.devLoginBodySchema, request.body);
diff --git a/apps/api/src/observability.ts b/apps/api/src/observability.ts
index f6a8731..adb3d8c 100644
--- a/apps/api/src/observability.ts
+++ b/apps/api/src/observability.ts
@@ -1,5 +1,6 @@
 import {
   Gauge,
+  Histogram,
   Registry,
   collectDefaultMetrics
 } from "prom-client";
@@ -12,6 +13,20 @@ interface LiveGameStats {
 
 export class ApiMetrics {
   private readonly registry = new Registry();
+  private readonly requestDuration = new Histogram({
+    name: "pong_pong_api_http_request_duration_seconds",
+    help: "HTTP request duration in seconds",
+    labelNames: ["method", "route", "status_code"] as const,
+    buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5],
+    registers: [this.registry]
+  });
+  private readonly readinessDuration = new Histogram({
+    name: "pong_pong_api_readiness_check_duration_seconds",
+    help: "Repository readiness check duration in seconds",
+    labelNames: ["result"] as const,
+    buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5],
+    registers: [this.registry]
+  });
   private readonly connections = new Gauge({
     name: "pong_pong_api_connections",
     help: "Current websocket connection count",
@@ -40,6 +55,18 @@ export class ApiMetrics {
     return this.registry.contentType;
   }
 
+  observeRequest(method: string, route: string, statusCode: number, durationMs: number): void {
+    this.requestDuration.observe({
+      method,
+      route,
+      status_code: String(statusCode)
+    }, Math.max(0, durationMs) / 1_000);
+  }
+
+  observeReadiness(result: "ready" | "not_ready", durationMs: number): void {
+    this.readinessDuration.observe({ result }, Math.max(0, durationMs) / 1_000);
+  }
+
   async scrape(): Promise<string> {
     const stats = this.readGameStats();
     this.connections.set(stats.onlinePlayers);


## `feat(metrics): repository operation 측정 추가`

diff --git a/apps/api/src/app.ts b/apps/api/src/app.ts
index 5feb6b3..02cb42b 100644
--- a/apps/api/src/app.ts
+++ b/apps/api/src/app.ts
@@ -26,7 +26,7 @@ import {
 import { createLoggerOptions } from "./requestLogging.js";
 import { readAppMode } from "./env.js";
 import { createRawWsTicket, hashWsTicket, WS_TICKET_TTL_SECONDS } from "./wsTicket.js";
-import { ApiMetrics } from "./observability.js";
+import { ApiMetrics, instrumentRepository } from "./observability.js";
 
 const WS_POLICY_VIOLATION = 1008;
 const WS_MESSAGE_TOO_BIG = 1009;
@@ -46,7 +46,7 @@ export interface BuildAppOptions {
 }
 
 export function buildApp({
-  repo,
+  repo: sourceRepo,
   webOrigin,
   appMode = readAppMode(),
   guestAccess,
@@ -57,8 +57,11 @@ export function buildApp({
     logger: createLoggerOptions(process.env.LOG_LEVEL ?? "info"),
     trustProxy
   });
+  let readGameStats = () => ({ onlinePlayers: 0, queuedPlayers: 0, activeRooms: 0 });
+  const metrics = new ApiMetrics(() => readGameStats());
+  const repo = instrumentRepository(sourceRepo, metrics);
   const hub = new GameHub(repo);
-  const metrics = new ApiMetrics(() => hub.liveStats());
+  readGameStats = () => hub.liveStats();
   const guests = appMode === "demo" ? guestAccess ?? new GuestAccess({ secret: sessionSecret }) : null;
   const getCurrentUser = (request: FastifyRequest) => currentUser(repo, request, guests, appMode === "demo");
 
diff --git a/apps/api/src/observability.ts b/apps/api/src/observability.ts
index adb3d8c..97d46aa 100644
--- a/apps/api/src/observability.ts
+++ b/apps/api/src/observability.ts
@@ -4,6 +4,7 @@ import {
   Registry,
   collectDefaultMetrics
 } from "prom-client";
+import type { AppRepository } from "@pong-pong/db";
 
 interface LiveGameStats {
   onlinePlayers: number;
@@ -11,6 +12,43 @@ interface LiveGameStats {
   activeRooms: number;
 }
 
+const REPOSITORY_OPERATIONS = new Set([
+  "close",
+  "checkReadiness",
+  "ensureSeedData",
+  "upsertDevUser",
+  "createSession",
+  "getSessionUser",
+  "deleteSession",
+  "createWsTicket",
+  "consumeWsTicket",
+  "setUserRoleByHandle",
+  "getUserById",
+  "getUserByHandle",
+  "updateProfile",
+  "listOnlineUsers",
+  "listNpcOpponents",
+  "listLeaderboard",
+  "listRecentMatches",
+  "getDashboard",
+  "listFriends",
+  "requestFriend",
+  "acceptFriend",
+  "createMatch",
+  "finalizeMatch",
+  "listLobbyChat",
+  "createChatMessage",
+  "listTournaments",
+  "createTournament",
+  "joinTournament",
+  "getTournamentMatch",
+  "startTournamentMatch",
+  "completeTournamentMatch",
+  "listAdminUsers",
+  "listAdminActions",
+  "setUserBan"
+]);
+
 export class ApiMetrics {
   private readonly registry = new Registry();
   private readonly requestDuration = new Histogram({
@@ -27,6 +65,13 @@ export class ApiMetrics {
     buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5],
     registers: [this.registry]
   });
+  private readonly databaseOperationDuration = new Histogram({
+    name: "pong_pong_api_database_operation_duration_seconds",
+    help: "Repository operation duration in seconds",
+    labelNames: ["operation", "outcome"] as const,
+    buckets: [0.001, 0.0025, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5],
+    registers: [this.registry]
+  });
   private readonly connections = new Gauge({
     name: "pong_pong_api_connections",
     help: "Current websocket connection count",
@@ -67,6 +112,13 @@ export class ApiMetrics {
     this.readinessDuration.observe({ result }, Math.max(0, durationMs) / 1_000);
   }
 
+  observeDatabaseOperation(operation: string, outcome: "success" | "failure", durationMs: number): void {
+    this.databaseOperationDuration.observe({
+      operation: REPOSITORY_OPERATIONS.has(operation) ? operation : "other",
+      outcome
+    }, Math.max(0, durationMs) / 1_000);
+  }
+
   async scrape(): Promise<string> {
     const stats = this.readGameStats();
     this.connections.set(stats.onlinePlayers);
@@ -79,3 +131,35 @@ export class ApiMetrics {
     this.registry.clear();
   }
 }
+
+export function instrumentRepository(
+  repository: AppRepository,
+  metrics: ApiMetrics
+): AppRepository {
+  return new Proxy(repository, {
+    get(target, property) {
+      const value = Reflect.get(target, property, target);
+      if (typeof property !== "string" || typeof value !== "function") return value;
+      return (...args: unknown[]) => {
+        const startedAt = performance.now();
+        let result: unknown;
+        try {
+          result = Reflect.apply(value as (...methodArgs: unknown[]) => unknown, target, args);
+        } catch (error) {
+          metrics.observeDatabaseOperation(property, "failure", performance.now() - startedAt);
+          throw error;
+        }
+        return Promise.resolve(result).then(
+          (resolved) => {
+            metrics.observeDatabaseOperation(property, "success", performance.now() - startedAt);
+            return resolved;
+          },
+          (error) => {
+            metrics.observeDatabaseOperation(property, "failure", performance.now() - startedAt);
+            throw error;
+          }
+        );
+      };
+    }
+  }) as AppRepository;
+}


