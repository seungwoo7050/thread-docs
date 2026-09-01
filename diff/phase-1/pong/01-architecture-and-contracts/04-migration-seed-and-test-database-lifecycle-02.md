## `refactor(db): migration과 seed CLI 연결`

diff --git a/apps/api/src/index.ts b/apps/api/src/index.ts
index f61f3b9..404b807 100644
--- a/apps/api/src/index.ts
+++ b/apps/api/src/index.ts
@@ -4,7 +4,9 @@ import { readEnv } from "./env";
 
 const env = readEnv();
 const repo = env.databaseUrl ? createPostgresRepository(env.databaseUrl) : createMemoryRepository();
-await repo.ensureSeedData();
+if (!env.databaseUrl) {
+  await repo.ensureSeedData();
+}
 
 const app = buildApp({ repo, webOrigin: env.webOrigin });
 app.addHook("onClose", async () => {
diff --git a/packages/db/package.json b/packages/db/package.json
index 99ff8d1..bd5d77c 100644
--- a/packages/db/package.json
+++ b/packages/db/package.json
@@ -10,7 +10,8 @@
     "build": "tsc --noEmit",
     "typecheck": "tsc --noEmit",
     "migrate": "tsx src/cli.ts migrate",
-    "seed": "tsx src/cli.ts seed",
+    "seed:dev": "tsx src/cli.ts seed:dev",
+    "seed:demo": "tsx src/cli.ts seed:demo",
     "memory-smoke": "tsx src/cli.ts memory-smoke",
     "test": "vitest run"
   },
diff --git a/packages/db/src/cli.ts b/packages/db/src/cli.ts
index 1398d25..30b7716 100644
--- a/packages/db/src/cli.ts
+++ b/packages/db/src/cli.ts
@@ -1,26 +1,35 @@
 import { createMemoryRepository, createPostgresRepository } from "./index";
+import { migrateDatabase } from "./migrator";
 
 const command = process.argv[2];
-const databaseUrl = process.env.DATABASE_URL;
 
-if (!databaseUrl) {
-  throw new Error("DATABASE_URL is required for database CLI commands");
-}
+if (command === "memory-smoke") {
+  const memory = createMemoryRepository();
+  await memory.ensureSeedData();
+  await memory.close();
+  console.log("ok");
+} else {
+  const databaseUrl = process.env.DATABASE_URL;
 
-const repo = createPostgresRepository(databaseUrl);
+  if (!databaseUrl) {
+    throw new Error("DATABASE_URL is required for database CLI commands");
+  }
 
-try {
-  if (command === "migrate" || command === "seed") {
-    await repo.ensureSeedData();
-    console.log(command === "migrate" ? "migrated" : "seeded");
-  } else if (command === "memory-smoke") {
-    const memory = createMemoryRepository();
-    await memory.ensureSeedData();
-    await memory.close();
-    console.log("ok");
+  if (command === "migrate") {
+    await migrateDatabase(databaseUrl);
+    console.log("migrated");
   } else {
-    throw new Error("Usage: pnpm --filter @pong-pong/db migrate|seed");
+    const repo = createPostgresRepository(databaseUrl);
+
+    try {
+      if (command === "seed:dev" || command === "seed:demo") {
+        await repo.ensureSeedData(command === "seed:dev" ? "development" : "demo");
+        console.log(command === "seed:dev" ? "development seed complete" : "demo seed complete");
+      } else {
+        throw new Error("Usage: pnpm --filter @pong-pong/db migrate|seed:dev|seed:demo|memory-smoke");
+      }
+    } finally {
+      await repo.close();
+    }
   }
-} finally {
-  await repo.close();
 }


## `fix(api): startup seed 생성을 제거`

diff --git a/apps/api/src/index.ts b/apps/api/src/index.ts
index 56f9f4d..ccfa61d 100644
--- a/apps/api/src/index.ts
+++ b/apps/api/src/index.ts
@@ -5,9 +5,6 @@ import { installGracefulShutdown } from "./gracefulShutdown.js";
 
 const env = readEnv();
 const repo = env.databaseUrl ? createPostgresRepository(env.databaseUrl) : createMemoryRepository();
-if (!env.databaseUrl) {
-  await repo.ensureSeedData();
-}
 
 const app = buildApp({
   repo,


## `feat(db): test database reset target guard 추가`

diff --git a/packages/db/src/testReset.ts b/packages/db/src/testReset.ts
new file mode 100644
index 0000000..4813584
--- /dev/null
+++ b/packages/db/src/testReset.ts
@@ -0,0 +1,62 @@
+const ISOLATED_TEST_SCHEMA = /^test_[a-f0-9]{32}$/;
+const DEDICATED_TEST_DATABASE = /^(?:test(?:_[a-z0-9][a-z0-9_-]*)?|[a-z0-9][a-z0-9_-]*_test)$/;
+
+export interface TestResetTarget {
+  databaseUrl: string;
+  databaseName: string;
+  schema: string;
+}
+
+export function resolveTestResetTarget(env: NodeJS.ProcessEnv): TestResetTarget {
+  if (env.NODE_ENV !== "test") {
+    throw new Error("reset:test requires NODE_ENV=test");
+  }
+  const databaseUrl = env.TEST_DATABASE_URL;
+  if (!databaseUrl) {
+    throw new Error("TEST_DATABASE_URL is required for reset:test");
+  }
+
+  let url: URL;
+  try {
+    url = new URL(databaseUrl);
+  } catch {
+    return unsafeTarget();
+  }
+  if (url.protocol !== "postgres:" && url.protocol !== "postgresql:") {
+    return unsafeTarget();
+  }
+
+  let databaseName: string;
+  try {
+    databaseName = decodeURIComponent(url.pathname.slice(1));
+  } catch {
+    return unsafeTarget();
+  }
+  if (!databaseName || databaseName.includes("/")) {
+    return unsafeTarget();
+  }
+
+  const optionValues = url.searchParams.getAll("options");
+  if (optionValues.length > 1) {
+    return unsafeTarget();
+  }
+  let schema = "public";
+  if (optionValues.length === 1) {
+    const match = /^-c search_path=(test_[a-f0-9]{32})$/.exec(optionValues[0]);
+    if (!match) return unsafeTarget();
+    schema = match[1];
+  }
+
+  if (schema === "public" && !DEDICATED_TEST_DATABASE.test(databaseName)) {
+    return unsafeTarget();
+  }
+  if (schema !== "public" && !ISOLATED_TEST_SCHEMA.test(schema)) {
+    return unsafeTarget();
+  }
+
+  return { databaseUrl, databaseName, schema };
+}
+
+function unsafeTarget(): never {
+  throw new Error("Unsafe test reset target");
+}


## `feat(db): test schema reset과 migration 실행 연결`

diff --git a/packages/db/package.json b/packages/db/package.json
index ca9bebc..6337fcc 100644
--- a/packages/db/package.json
+++ b/packages/db/package.json
@@ -19,6 +19,7 @@
     "migrate": "tsx src/cli.ts migrate",
     "seed:dev": "tsx src/cli.ts seed:dev",
     "seed:demo": "tsx src/cli.ts seed:demo",
+    "reset:test": "NODE_ENV=test tsx src/cli.ts reset:test",
     "migrate:prod": "node dist/cli.js migrate",
     "user:set-role": "tsx src/cli.ts user:set-role",
     "memory-smoke": "tsx src/cli.ts memory-smoke",
diff --git a/packages/db/src/cli.ts b/packages/db/src/cli.ts
index 572ce97..8edee92 100644
--- a/packages/db/src/cli.ts
+++ b/packages/db/src/cli.ts
@@ -1,5 +1,6 @@
 import { createMemoryRepository, createPostgresRepository } from "./index.js";
 import { migrateDatabase } from "./migrator.js";
+import { resetTestDatabase } from "./testReset.js";
 
 const command = process.argv[2];
 
@@ -8,6 +9,9 @@ if (command === "memory-smoke") {
   await memory.ensureSeedData();
   await memory.close();
   console.log("ok");
+} else if (command === "reset:test") {
+  const target = await resetTestDatabase();
+  console.log(`test schema reset: ${target.schema}`);
 } else {
   const databaseUrl = process.env.DATABASE_URL;
 
@@ -34,7 +38,7 @@ if (command === "memory-smoke") {
         const user = await repo.setUserRoleByHandle(handle, role);
         console.log(`${user.handle} role set to ${user.role}`);
       } else {
-        throw new Error("Usage: pnpm --filter @pong-pong/db migrate|seed:dev|seed:demo|user:set-role|memory-smoke");
+        throw new Error("Usage: pnpm --filter @pong-pong/db migrate|seed:dev|seed:demo|reset:test|user:set-role|memory-smoke");
       }
     } finally {
       await repo.close();
diff --git a/packages/db/src/testReset.ts b/packages/db/src/testReset.ts
index 4813584..d9dd250 100644
--- a/packages/db/src/testReset.ts
+++ b/packages/db/src/testReset.ts
@@ -1,3 +1,6 @@
+import { Pool } from "pg";
+import { migrateDatabase } from "./migrator.js";
+
 const ISOLATED_TEST_SCHEMA = /^test_[a-f0-9]{32}$/;
 const DEDICATED_TEST_DATABASE = /^(?:test(?:_[a-z0-9][a-z0-9_-]*)?|[a-z0-9][a-z0-9_-]*_test)$/;
 
@@ -57,6 +60,36 @@ export function resolveTestResetTarget(env: NodeJS.ProcessEnv): TestResetTarget
   return { databaseUrl, databaseName, schema };
 }
 
+export async function resetTestDatabase(
+  env: NodeJS.ProcessEnv = process.env
+): Promise<TestResetTarget> {
+  const target = resolveTestResetTarget(env);
+  const controlUrl = new URL(target.databaseUrl);
+  controlUrl.searchParams.delete("options");
+  const pool = new Pool({ connectionString: controlUrl.toString() });
+  const quotedSchema = `"${target.schema}"`;
+
+  try {
+    const client = await pool.connect();
+    try {
+      await client.query("begin");
+      await client.query(`drop schema if exists ${quotedSchema} cascade`);
+      await client.query(`create schema ${quotedSchema}`);
+      await client.query("commit");
+    } catch (error) {
+      await client.query("rollback").catch(() => undefined);
+      throw error;
+    } finally {
+      client.release();
+    }
+  } finally {
+    await pool.end();
+  }
+
+  await migrateDatabase(target.databaseUrl);
+  return target;
+}
+
 function unsafeTarget(): never {
   throw new Error("Unsafe test reset target");
 }


## `test(db): test database reset guard 검증`

diff --git a/packages/db/src/postgres.integration.test.ts b/packages/db/src/postgres.integration.test.ts
index 35dca98..14405bb 100644
--- a/packages/db/src/postgres.integration.test.ts
+++ b/packages/db/src/postgres.integration.test.ts
@@ -4,6 +4,7 @@ import { afterAll, beforeAll, describe, expect, it } from "vitest";
 import { Pool } from "pg";
 import { createPostgresRepository, type AppRepository } from "./index";
 import { migrateDatabase } from "./migrator";
+import { resetTestDatabase } from "./testReset";
 
 const POSTGRES_IMAGE = "postgres:16-alpine";
 const TEST_DATABASE = "pong_pong_test";
@@ -136,6 +137,41 @@ describe("PostgreSQL integration", () => {
     }, { migrate: false });
   });
 
+  it("resets only the selected isolated schema and reapplies migrations", async () => {
+    const siblingSchema = `test_${randomUUID().replaceAll("-", "")}`;
+    const pool = requireAdminPool();
+    await pool.query(`create schema ${quoteSchema(siblingSchema)}`);
+    await pool.query(`create table ${quoteSchema(siblingSchema)}.reset_guard_marker (id integer primary key)`);
+
+    try {
+      await withIsolatedDatabase(async ({ databaseUrl, openPool, openRepository }) => {
+        const repository = openRepository();
+        const isolatedPool = openPool();
+        await repository.upsertDevUser({
+          handle: "reset-test-user",
+          displayName: "Reset Test User"
+        });
+
+        await resetTestDatabase({ NODE_ENV: "test", TEST_DATABASE_URL: databaseUrl });
+
+        await expect(isolatedPool.query<{ count: number }>(
+          "select count(*)::integer as count from users"
+        )).resolves.toMatchObject({ rows: [{ count: 0 }] });
+        await expect(repository.checkReadiness()).resolves.toEqual({
+          database: "up",
+          migrations: "current"
+        });
+        const sibling = await pool.query<{ exists: boolean }>(
+          "select exists(select 1 from pg_tables where schemaname = $1 and tablename = 'reset_guard_marker') as exists",
+          [siblingSchema]
+        );
+        expect(sibling.rows[0]?.exists).toBe(true);
+      });
+    } finally {
+      await pool.query(`drop schema if exists ${quoteSchema(siblingSchema)} cascade`);
+    }
+  });
+
   it("keeps the demo seed limited to NPC accounts", async () => {
     await withIsolatedDatabase(async ({ openPool, openRepository }) => {
       const repository = openRepository();
diff --git a/packages/db/src/testReset.test.ts b/packages/db/src/testReset.test.ts
new file mode 100644
index 0000000..b9971f3
--- /dev/null
+++ b/packages/db/src/testReset.test.ts
@@ -0,0 +1,70 @@
+import { describe, expect, it } from "vitest";
+import { resolveTestResetTarget } from "./testReset";
+
+const ISOLATED_SCHEMA = `test_${"a".repeat(32)}`;
+const TEST_DATABASE_URL = "postgresql://pong:pong@localhost:5432/pong_pong_test";
+const APPLICATION_DATABASE_URL = "postgresql://pong:pong@localhost:5432/pong_pong";
+
+describe("test database reset guard", () => {
+  it("requires the test runtime and TEST_DATABASE_URL", () => {
+    expect(() => resolveTestResetTarget({
+      NODE_ENV: "development",
+      TEST_DATABASE_URL
+    })).toThrow("NODE_ENV=test");
+    expect(() => resolveTestResetTarget({
+      NODE_ENV: "test",
+      DATABASE_URL: TEST_DATABASE_URL
+    })).toThrow("TEST_DATABASE_URL");
+  });
+
+  it.each([
+    APPLICATION_DATABASE_URL,
+    "postgresql://pong:pong@localhost:5432/pong_pong_test_backup",
+    "postgresql://pong:pong@localhost:5432/contest"
+  ])("rejects a regular database without an isolated schema: %s", (databaseUrl) => {
+    expect(() => resolveTestResetTarget({
+      NODE_ENV: "test",
+      TEST_DATABASE_URL: databaseUrl
+    })).toThrow("Unsafe test reset target");
+  });
+
+  it.each([
+    "-c search_path=public,other",
+    "-c search_path=test_manual",
+    `-c search_path=${ISOLATED_SCHEMA},public`,
+    "-c statement_timeout=1000"
+  ])("rejects an ambiguous PostgreSQL options value: %s", (options) => {
+    const url = new URL(APPLICATION_DATABASE_URL);
+    url.searchParams.set("options", options);
+
+    expect(() => resolveTestResetTarget({
+      NODE_ENV: "test",
+      TEST_DATABASE_URL: url.toString()
+    })).toThrow("Unsafe test reset target");
+  });
+
+  it("allows the public schema only inside a clearly named test database", () => {
+    expect(resolveTestResetTarget({
+      NODE_ENV: "test",
+      TEST_DATABASE_URL
+    })).toEqual({
+      databaseUrl: TEST_DATABASE_URL,
+      databaseName: "pong_pong_test",
+      schema: "public"
+    });
+  });
+
+  it("allows one generated isolated schema without requiring a test database name", () => {
+    const url = new URL(APPLICATION_DATABASE_URL);
+    url.searchParams.set("options", `-c search_path=${ISOLATED_SCHEMA}`);
+
+    expect(resolveTestResetTarget({
+      NODE_ENV: "test",
+      TEST_DATABASE_URL: url.toString()
+    })).toEqual({
+      databaseUrl: url.toString(),
+      databaseName: "pong_pong",
+      schema: ISOLATED_SCHEMA
+    });
+  });
+});
