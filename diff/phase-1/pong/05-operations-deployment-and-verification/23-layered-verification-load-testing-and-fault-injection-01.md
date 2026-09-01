# 계층형 검증·부하 시험·장애 주입

## `test(smoke): HTTP API 실행 검사 추가`

diff --git a/tests/smoke-api.mjs b/tests/smoke-api.mjs
new file mode 100644
index 0000000..05a1e89
--- /dev/null
+++ b/tests/smoke-api.mjs
@@ -0,0 +1,27 @@
+const baseUrl = process.env.API_BASE_URL ?? "http://localhost:4000";
+
+const login = await request("/auth/dev-login", {
+  method: "POST",
+  body: JSON.stringify({ handle: "smoke", displayName: "스모크" })
+});
+
+await request("/me", { headers: { authorization: `Bearer ${login.token}` } });
+await request("/lobby", { headers: { authorization: `Bearer ${login.token}` } });
+await request("/leaderboard");
+await request("/dashboard", { headers: { authorization: `Bearer ${login.token}` } });
+
+console.log("api smoke ok");
+
+async function request(path, init = {}) {
+  const response = await fetch(`${baseUrl}${path}`, {
+    ...init,
+    headers: {
+      "content-type": "application/json",
+      ...init.headers
+    }
+  });
+  if (!response.ok) {
+    throw new Error(`${path} failed with ${response.status}: ${await response.text()}`);
+  }
+  return response.json();
+}


## `test(smoke): WebSocket 경기 실행 검사 추가`

diff --git a/Makefile b/Makefile
index 859ca3b..aa9c466 100644
--- a/Makefile
+++ b/Makefile
@@ -1,4 +1,4 @@
-.PHONY: install typecheck build test
+.PHONY: install typecheck build test smoke
 
 install:
 	pnpm install
@@ -11,3 +11,7 @@ build:
 
 test:
 	pnpm -r test
+
+smoke:
+	node tests/smoke-api.mjs
+	node tests/smoke-ws.mjs
diff --git a/tests/smoke-ws.mjs b/tests/smoke-ws.mjs
new file mode 100644
index 0000000..6e7697b
--- /dev/null
+++ b/tests/smoke-ws.mjs
@@ -0,0 +1,68 @@
+const baseUrl = process.env.API_BASE_URL ?? "http://localhost:4000";
+const wsUrl = process.env.WS_URL ?? "ws://localhost:4000/ws";
+
+const left = await login("left-smoke", "왼쪽");
+const right = await login("right-smoke", "오른쪽");
+
+const leftSocket = connect(left.token);
+const rightSocket = connect(right.token);
+const events = [];
+
+leftSocket.addEventListener("message", (event) => events.push(JSON.parse(event.data)));
+rightSocket.addEventListener("message", (event) => events.push(JSON.parse(event.data)));
+
+await opened(leftSocket);
+await opened(rightSocket);
+leftSocket.send(JSON.stringify({ type: "queue.join", mode: "queue" }));
+rightSocket.send(JSON.stringify({ type: "queue.join", mode: "queue" }));
+
+const matched = await waitFor(() => events.find((event) => event.type === "queue.matched"));
+leftSocket.send(JSON.stringify({ type: "game.ready", roomId: matched.roomId }));
+rightSocket.send(JSON.stringify({ type: "game.ready", roomId: matched.roomId }));
+leftSocket.send(JSON.stringify({ type: "chat.send", scope: "match", roomId: matched.roomId, body: "준비됐습니다." }));
+
+await waitFor(() => events.find((event) => event.type === "game.snapshot" && event.snapshot.phase === "playing"));
+await waitFor(() => events.find((event) => event.type === "chat.message"));
+
+leftSocket.close();
+rightSocket.close();
+console.log("websocket smoke ok");
+
+async function login(handle, displayName) {
+  const response = await fetch(`${baseUrl}/auth/dev-login`, {
+    method: "POST",
+    headers: { "content-type": "application/json" },
+    body: JSON.stringify({ handle, displayName })
+  });
+  if (!response.ok) throw new Error(`login failed: ${response.status}`);
+  return response.json();
+}
+
+function connect(token) {
+  return new WebSocket(`${wsUrl}?session=${token}`);
+}
+
+function opened(socket) {
+  if (socket.readyState === WebSocket.OPEN) return Promise.resolve();
+  return new Promise((resolve, reject) => {
+    socket.addEventListener("open", resolve, { once: true });
+    socket.addEventListener("error", reject, { once: true });
+  });
+}
+
+function waitFor(predicate) {
+  const startedAt = Date.now();
+  return new Promise((resolve, reject) => {
+    const timer = setInterval(() => {
+      const value = predicate();
+      if (value) {
+        clearInterval(timer);
+        resolve(value);
+      }
+      if (Date.now() - startedAt > 10_000) {
+        clearInterval(timer);
+        reject(new Error("timed out"));
+      }
+    }, 50);
+  });
+}


## `test(e2e): 한국어 내비게이션과 캔버스 흐름 구성`

diff --git a/Makefile b/Makefile
index aa9c466..7ebd8ef 100644
--- a/Makefile
+++ b/Makefile
@@ -1,4 +1,4 @@
-.PHONY: install typecheck build test smoke
+.PHONY: install typecheck build test smoke e2e
 
 install:
 	pnpm install
@@ -15,3 +15,6 @@ test:
 smoke:
 	node tests/smoke-api.mjs
 	node tests/smoke-ws.mjs
+
+e2e:
+	pnpm test:e2e
diff --git a/package.json b/package.json
index 031dc85..146f658 100644
--- a/package.json
+++ b/package.json
@@ -6,9 +6,11 @@
   "scripts": {
     "build": "pnpm -r build",
     "typecheck": "pnpm -r typecheck",
-    "test": "pnpm -r test"
+    "test": "pnpm -r test",
+    "test:e2e": "playwright test"
   },
   "devDependencies": {
+    "@playwright/test": "^1.52.0",
     "@types/node": "^22.15.3",
     "typescript": "^5.8.3",
     "vitest": "^3.1.2"
diff --git a/playwright.config.ts b/playwright.config.ts
new file mode 100644
index 0000000..9a8c73c
--- /dev/null
+++ b/playwright.config.ts
@@ -0,0 +1,24 @@
+import { defineConfig, devices } from "@playwright/test";
+
+export default defineConfig({
+  testDir: "./tests/e2e",
+  timeout: 30_000,
+  expect: {
+    timeout: 10_000
+  },
+  use: {
+    baseURL: process.env.E2E_BASE_URL ?? "http://localhost:3000",
+    trace: "retain-on-failure",
+    screenshot: "only-on-failure"
+  },
+  projects: [
+    {
+      name: "chromium-desktop",
+      use: { ...devices["Desktop Chrome"], viewport: { width: 1448, height: 1086 } }
+    },
+    {
+      name: "chromium-mobile",
+      use: { ...devices["Pixel 7"] }
+    }
+  ]
+});
diff --git a/pnpm-lock.yaml b/pnpm-lock.yaml
index e86781a..15c1c67 100644
--- a/pnpm-lock.yaml
+++ b/pnpm-lock.yaml
@@ -8,6 +8,9 @@ importers:
 
   .:
     devDependencies:
+      '@playwright/test':
+        specifier: ^1.52.0
+        version: 1.59.1
       '@types/node':
         specifier: ^22.15.3
         version: 22.19.17
@@ -1883,7 +1886,6 @@ snapshots:
   '@playwright/test@1.59.1':
     dependencies:
       playwright: 1.59.1
-    optional: true
 
   '@rollup/rollup-android-arm-eabi@4.60.3':
     optional: true
@@ -2469,15 +2471,13 @@ snapshots:
 
   pirates@4.0.7: {}
 
-  playwright-core@1.59.1:
-    optional: true
+  playwright-core@1.59.1: {}
 
   playwright@1.59.1:
     dependencies:
       playwright-core: 1.59.1
     optionalDependencies:
       fsevents: 2.3.2
-    optional: true
 
   postcss-import@15.1.0(postcss@8.5.14):
     dependencies:
diff --git a/tests/e2e/pong-pong.spec.ts b/tests/e2e/pong-pong.spec.ts
new file mode 100644
index 0000000..aaa945c
--- /dev/null
+++ b/tests/e2e/pong-pong.spec.ts
@@ -0,0 +1,37 @@
+import { expect, test } from "@playwright/test";
+
+test("한국어 로비에서 로그인하고 주요 화면을 이동한다", async ({ page }) => {
+  await page.goto("/");
+  await expect(page.getByRole("heading", { name: "퐁퐁" })).toBeVisible();
+  await page.getByLabel("핸들").fill("tester");
+  await page.getByLabel("표시 이름").fill("테스터");
+  await page.getByRole("button", { name: "개발 로그인" }).click();
+
+  await expect(page.getByRole("link", { name: "빠른 매칭" })).toBeVisible();
+  await page.getByRole("link", { name: "대시보드" }).click();
+  await expect(page.getByRole("heading", { name: "내 대시보드" })).toBeVisible();
+  await page.getByRole("link", { name: "순위표" }).click();
+  await expect(page.getByRole("heading", { name: "순위표" })).toBeVisible();
+  await page.getByRole("link", { name: "토너먼트" }).click();
+  await expect(page.getByRole("heading", { name: "토너먼트" })).toBeVisible();
+});
+
+test("플레이 화면의 캔버스가 실제 픽셀을 그린다", async ({ page }) => {
+  await page.goto("/");
+  await page.getByLabel("핸들").fill("canvas");
+  await page.getByLabel("표시 이름").fill("캔버스");
+  await page.getByRole("button", { name: "개발 로그인" }).click();
+  await page.getByRole("link", { name: "경기" }).click();
+  await expect(page.getByRole("heading", { name: "경기장" })).toBeVisible();
+
+  const hasPaint = await page.locator("canvas").evaluate((canvas) => {
+    const ctx = (canvas as HTMLCanvasElement).getContext("2d");
+    if (!ctx) return false;
+    const data = ctx.getImageData(0, 0, (canvas as HTMLCanvasElement).width, (canvas as HTMLCanvasElement).height).data;
+    for (let index = 3; index < data.length; index += 4) {
+      if (data[index] !== 0) return true;
+    }
+    return false;
+  });
+  expect(hasPaint).toBe(true);
+});


## `ci(repo): typecheck·unit·build workflow 추가`

diff --git a/.github/workflows/ci.yml b/.github/workflows/ci.yml
new file mode 100644
index 0000000..1f3e746
--- /dev/null
+++ b/.github/workflows/ci.yml
@@ -0,0 +1,41 @@
+name: CI
+
+on:
+  push:
+  pull_request:
+
+permissions:
+  contents: read
+
+jobs:
+  verify:
+    name: Typecheck, unit tests, and build
+    runs-on: ubuntu-latest
+    timeout-minutes: 15
+
+    steps:
+      - name: Check out repository
+        uses: actions/checkout@v4
+
+      - name: Set up pnpm
+        uses: pnpm/action-setup@v4
+        with:
+          version: 10.32.1
+
+      - name: Set up Node.js
+        uses: actions/setup-node@v4
+        with:
+          node-version: 24.18.0
+          cache: pnpm
+
+      - name: Install dependencies
+        run: pnpm install --frozen-lockfile
+
+      - name: Typecheck
+        run: pnpm typecheck
+
+      - name: Run unit tests
+        run: pnpm unit
+
+      - name: Build
+        run: pnpm build


## `test(db): PostgreSQL integration 환경과 계약 추가`

diff --git a/packages/db/src/postgres.integration.test.ts b/packages/db/src/postgres.integration.test.ts
new file mode 100644
index 0000000..493e15f
--- /dev/null
+++ b/packages/db/src/postgres.integration.test.ts
@@ -0,0 +1,323 @@
+import { randomUUID } from "node:crypto";
+import { PostgreSqlContainer } from "@testcontainers/postgresql";
+import { afterAll, beforeAll, describe, expect, it } from "vitest";
+import { Pool } from "pg";
+import { createPostgresRepository, type AppRepository } from "./index";
+import { migrateDatabase } from "./migrator";
+
+const POSTGRES_IMAGE = "postgres:16-alpine";
+const TEST_DATABASE = "pong_pong_test";
+const TEST_USERNAME = "pong";
+const TEST_PASSWORD = "pong";
+
+type StartedPostgres = Awaited<ReturnType<PostgreSqlContainer["start"]>>;
+
+interface IsolatedDatabaseContext {
+  schema: string;
+  databaseUrl: string;
+  openPool(): Pool;
+  openRepository(): AppRepository;
+}
+
+let container: StartedPostgres | undefined;
+let adminPool: Pool | undefined;
+
+beforeAll(async () => {
+  container = await startPostgresContainer();
+  adminPool = new Pool({
+    connectionString: container.getConnectionUri(),
+    connectionTimeoutMillis: 5_000,
+    max: 2
+  });
+  await adminPool.query("select 1");
+});
+
+afterAll(async () => {
+  try {
+    await adminPool?.end();
+  } finally {
+    await container?.stop({ timeout: 10_000 });
+  }
+});
+
+describe("PostgreSQL integration", () => {
+  it("migrates an empty schema and leaves a repeated migration unchanged", async () => {
+    await withIsolatedDatabase(async ({ databaseUrl, openPool, schema }) => {
+      const pool = openPool();
+      const before = await pool.query<{ users: string | null }>(
+        "select to_regclass('users')::text as users"
+      );
+      expect(before.rows[0]?.users).toBeNull();
+
+      await migrateDatabase(databaseUrl);
+
+      const firstTables = await tableNames(pool, schema);
+      expect(firstTables).toEqual(expect.arrayContaining([
+        "admin_actions",
+        "chat_messages",
+        "friendships",
+        "matches",
+        "sessions",
+        "tournament_entries",
+        "tournament_matches",
+        "tournaments",
+        "users"
+      ]));
+      const firstMigrations = await appliedMigrations(pool);
+      expect(firstMigrations).toEqual(["001_initial"]);
+
+      await migrateDatabase(databaseUrl);
+
+      expect(await tableNames(pool, schema)).toEqual(firstTables);
+      expect(await appliedMigrations(pool)).toEqual(firstMigrations);
+    }, { migrate: false });
+  });
+
+  it("keeps the demo seed limited to NPC accounts", async () => {
+    await withIsolatedDatabase(async ({ openPool, openRepository }) => {
+      const repository = openRepository();
+      await repository.ensureSeedData("demo");
+      await repository.ensureSeedData("demo");
+
+      const users = await openPool().query<{
+        handle: string;
+        is_npc: boolean;
+        role: string;
+      }>("select handle, is_npc, role from users order by handle");
+
+      expect(users.rows).toHaveLength(4);
+      expect(users.rows.every((user) => user.is_npc)).toBe(true);
+      expect(users.rows.every((user) => user.role === "user")).toBe(true);
+      expect(users.rows.some((user) => user.handle === "admin")).toBe(false);
+    });
+  });
+
+  it("keeps development users and administrator data out of the demo seed", async () => {
+    await withIsolatedDatabase(async ({ openPool, openRepository }) => {
+      const repository = openRepository();
+      await repository.ensureSeedData("development");
+      await repository.ensureSeedData("development");
+
+      const users = await openPool().query<{
+        handle: string;
+        is_npc: boolean;
+        role: string;
+      }>("select handle, is_npc, role from users order by handle");
+      const players = users.rows.filter((user) => !user.is_npc);
+
+      expect(users.rows).toHaveLength(9);
+      expect(players.map((user) => user.handle)).toEqual([
+        "admin",
+        "net-ninja",
+        "paddle-pro",
+        "spin-doctor",
+        "top-spin"
+      ]);
+      expect(players.find((user) => user.handle === "admin")?.role).toBe("admin");
+    });
+  });
+
+  it("uses a fresh schema for each isolated database", async () => {
+    let firstSchema = "";
+
+    await withIsolatedDatabase(async ({ openRepository, schema }) => {
+      firstSchema = schema;
+      await openRepository().upsertDevUser({
+        handle: "schema-owner",
+        displayName: "Schema Owner"
+      });
+    });
+
+    await withIsolatedDatabase(async ({ openRepository, schema }) => {
+      expect(schema).not.toBe(firstSchema);
+      await expect(openRepository().getUserByHandle("schema-owner")).resolves.toBeNull();
+    });
+
+    expect(await schemaExists(firstSchema)).toBe(false);
+  });
+
+  it("drops the schema and closes tracked connections when a test callback fails", async () => {
+    let failedSchema = "";
+    let backendPid = 0;
+
+    await expect(withIsolatedDatabase(async ({ openPool, schema }) => {
+      failedSchema = schema;
+      const pool = openPool();
+      const backend = await pool.query<{ pid: number }>("select pg_backend_pid() as pid");
+      backendPid = backend.rows[0]?.pid ?? 0;
+      throw new Error("intentional integration failure");
+    })).rejects.toThrow("intentional integration failure");
+
+    expect(await schemaExists(failedSchema)).toBe(false);
+    const activeConnection = await requireAdminPool().query<{ active: boolean }>(
+      "select exists(select 1 from pg_stat_activity where pid = $1) as active",
+      [backendPid]
+    );
+    expect(activeConnection.rows[0]?.active).toBe(false);
+  });
+
+  it("stops a temporary container when its callback fails", async () => {
+    let stoppedConnectionUri = "";
+
+    await expect(withTemporaryPostgres(async (temporaryContainer) => {
+      stoppedConnectionUri = temporaryContainer.getConnectionUri();
+      const mappedPort = Number(new URL(stoppedConnectionUri).port);
+      expect(mappedPort).toBe(temporaryContainer.getPort());
+      expect(mappedPort).toBeGreaterThan(0);
+
+      const pool = new Pool({
+        connectionString: stoppedConnectionUri,
+        connectionTimeoutMillis: 2_000,
+        max: 1
+      });
+      try {
+        await pool.query("select 1");
+      } finally {
+        await pool.end();
+      }
+      throw new Error("intentional container failure");
+    })).rejects.toThrow("intentional container failure");
+
+    const stoppedPool = new Pool({
+      connectionString: stoppedConnectionUri,
+      connectionTimeoutMillis: 1_000,
+      max: 1
+    });
+    try {
+      await expect(stoppedPool.query("select 1")).rejects.toThrow();
+    } finally {
+      await stoppedPool.end();
+    }
+  });
+});
+
+async function startPostgresContainer(): Promise<StartedPostgres> {
+  return new PostgreSqlContainer(POSTGRES_IMAGE)
+    .withDatabase(TEST_DATABASE)
+    .withUsername(TEST_USERNAME)
+    .withPassword(TEST_PASSWORD)
+    .start();
+}
+
+async function withTemporaryPostgres<T>(
+  callback: (temporaryContainer: StartedPostgres) => Promise<T>
+): Promise<T> {
+  const temporaryContainer = await startPostgresContainer();
+  try {
+    return await callback(temporaryContainer);
+  } finally {
+    await temporaryContainer.stop({ timeout: 10_000 });
+  }
+}
+
+async function withIsolatedDatabase<T>(
+  callback: (context: IsolatedDatabaseContext) => Promise<T>,
+  options: { migrate?: boolean } = {}
+): Promise<T> {
+  const activeContainer = requireContainer();
+  const pool = requireAdminPool();
+  const schema = `test_${randomUUID().replaceAll("-", "")}`;
+  const quotedSchema = quoteSchema(schema);
+  const databaseUrl = withSearchPath(activeContainer.getConnectionUri(), schema);
+  const cleanupTasks: Array<() => Promise<void>> = [];
+  let callbackError: unknown;
+
+  await pool.query(`create schema ${quotedSchema}`);
+
+  const context: IsolatedDatabaseContext = {
+    schema,
+    databaseUrl,
+    openPool() {
+      const isolatedPool = new Pool({
+        connectionString: databaseUrl,
+        connectionTimeoutMillis: 5_000,
+        max: 2
+      });
+      cleanupTasks.push(() => isolatedPool.end());
+      return isolatedPool;
+    },
+    openRepository() {
+      const repository = createPostgresRepository(databaseUrl);
+      cleanupTasks.push(() => repository.close());
+      return repository;
+    }
+  };
+
+  try {
+    if (options.migrate !== false) {
+      await migrateDatabase(databaseUrl);
+    }
+    return await callback(context);
+  } catch (error) {
+    callbackError = error;
+    throw error;
+  } finally {
+    const cleanupErrors: unknown[] = [];
+    for (const cleanup of cleanupTasks.reverse()) {
+      try {
+        await cleanup();
+      } catch (error) {
+        cleanupErrors.push(error);
+      }
+    }
+    try {
+      await pool.query(`drop schema if exists ${quotedSchema} cascade`);
+    } catch (error) {
+      cleanupErrors.push(error);
+    }
+    if (callbackError === undefined && cleanupErrors.length > 0) {
+      throw new AggregateError(cleanupErrors, "Failed to clean up isolated PostgreSQL test resources");
+    }
+  }
+}
+
+async function tableNames(pool: Pool, schema: string): Promise<string[]> {
+  const result = await pool.query<{ tablename: string }>(
+    "select tablename from pg_tables where schemaname = $1 order by tablename",
+    [schema]
+  );
+  return result.rows.map((row) => row.tablename);
+}
+
+async function appliedMigrations(pool: Pool): Promise<string[]> {
+  const result = await pool.query<{ name: string }>(
+    "select name from kysely_migration order by name"
+  );
+  return result.rows.map((row) => row.name);
+}
+
+async function schemaExists(schema: string): Promise<boolean> {
+  const result = await requireAdminPool().query<{ exists: boolean }>(
+    "select exists(select 1 from pg_namespace where nspname = $1) as exists",
+    [schema]
+  );
+  return result.rows[0]?.exists ?? false;
+}
+
+function withSearchPath(databaseUrl: string, schema: string): string {
+  quoteSchema(schema);
+  const url = new URL(databaseUrl);
+  url.searchParams.set("options", `-c search_path=${schema}`);
+  return url.toString();
+}
+
+function quoteSchema(schema: string): string {
+  if (!/^test_[a-f0-9]{32}$/.test(schema)) {
+    throw new Error(`Unsafe test schema name: ${schema}`);
+  }
+  return `"${schema}"`;
+}
+
+function requireContainer(): StartedPostgres {
+  if (!container) {
+    throw new Error("PostgreSQL test container is not running");
+  }
+  return container;
+}
+
+function requireAdminPool(): Pool {
+  if (!adminPool) {
+    throw new Error("PostgreSQL admin pool is not connected");
+  }
+  return adminPool;
+}


