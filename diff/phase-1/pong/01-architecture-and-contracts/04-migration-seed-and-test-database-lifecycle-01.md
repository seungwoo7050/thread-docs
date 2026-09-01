# 마이그레이션·시드·테스트 데이터베이스 수명 주기

## `feat(db): 초기 PostgreSQL schema 정의`

diff --git a/packages/db/migrations/001_initial.sql b/packages/db/migrations/001_initial.sql
new file mode 100644
index 0000000..6ef6d91
--- /dev/null
+++ b/packages/db/migrations/001_initial.sql
@@ -0,0 +1,85 @@
+create extension if not exists pgcrypto;
+
+create table if not exists users (
+  id uuid primary key default gen_random_uuid(),
+  email text unique,
+  handle text not null unique,
+  display_name text not null,
+  avatar_key text not null default 'blue',
+  role text not null default 'user',
+  status text not null default 'active',
+  rating integer not null default 1200,
+  wins integer not null default 0,
+  losses integer not null default 0,
+  created_at timestamptz not null default now(),
+  banned_at timestamptz
+);
+
+create table if not exists sessions (
+  token text primary key,
+  user_id uuid not null references users(id) on delete cascade,
+  expires_at timestamptz not null,
+  created_at timestamptz not null default now()
+);
+
+create table if not exists friendships (
+  id uuid primary key default gen_random_uuid(),
+  requester_id uuid not null references users(id) on delete cascade,
+  addressee_id uuid not null references users(id) on delete cascade,
+  status text not null,
+  created_at timestamptz not null default now(),
+  updated_at timestamptz not null default now(),
+  unique (requester_id, addressee_id)
+);
+
+create table if not exists matches (
+  id uuid primary key default gen_random_uuid(),
+  mode text not null,
+  winner_id uuid references users(id),
+  loser_id uuid references users(id),
+  score_left integer not null,
+  score_right integer not null,
+  rating_delta integer not null default 16,
+  started_at timestamptz not null default now(),
+  ended_at timestamptz not null default now()
+);
+
+create table if not exists chat_messages (
+  id uuid primary key default gen_random_uuid(),
+  scope text not null,
+  room_id text,
+  sender_id uuid not null references users(id) on delete cascade,
+  body text not null,
+  created_at timestamptz not null default now()
+);
+
+create table if not exists tournaments (
+  id uuid primary key default gen_random_uuid(),
+  name text not null,
+  status text not null default 'open',
+  created_by uuid not null references users(id),
+  winner_id uuid references users(id),
+  capacity integer not null default 4,
+  created_at timestamptz not null default now()
+);
+
+create table if not exists tournament_entries (
+  id uuid primary key default gen_random_uuid(),
+  tournament_id uuid not null references tournaments(id) on delete cascade,
+  user_id uuid not null references users(id) on delete cascade,
+  seed integer not null,
+  created_at timestamptz not null default now(),
+  unique (tournament_id, user_id)
+);
+
+create table if not exists admin_actions (
+  id uuid primary key default gen_random_uuid(),
+  actor_id uuid references users(id),
+  target_user_id uuid references users(id),
+  action text not null,
+  reason text not null,
+  created_at timestamptz not null default now()
+);
+
+create index if not exists matches_ended_at_idx on matches (ended_at desc);
+create index if not exists chat_messages_scope_idx on chat_messages (scope, created_at desc);


## `feat(db): migration 실행 경계 구성`

diff --git a/packages/db/package.json b/packages/db/package.json
index 0089b9d..65043c4 100644
--- a/packages/db/package.json
+++ b/packages/db/package.json
@@ -3,6 +3,13 @@
   "version": "0.1.0",
   "private": true,
   "type": "module",
+  "exports": {
+    ".": "./src/index.ts"
+  },
+  "scripts": {
+    "build": "tsc --noEmit",
+    "typecheck": "tsc --noEmit"
+  },
   "dependencies": {
     "@pong-pong/shared": "workspace:*",
     "kysely": "^0.28.2",
diff --git a/packages/db/src/migrations.ts b/packages/db/src/migrations.ts
new file mode 100644
index 0000000..0967298
--- /dev/null
+++ b/packages/db/src/migrations.ts
@@ -0,0 +1,87 @@
+export const initialMigrationSql = `
+create extension if not exists pgcrypto;
+
+create table if not exists users (
+  id uuid primary key default gen_random_uuid(),
+  email text unique,
+  handle text not null unique,
+  display_name text not null,
+  avatar_key text not null default 'blue',
+  role text not null default 'user',
+  status text not null default 'active',
+  rating integer not null default 1200,
+  wins integer not null default 0,
+  losses integer not null default 0,
+  created_at timestamptz not null default now(),
+  banned_at timestamptz
+);
+
+create table if not exists sessions (
+  token text primary key,
+  user_id uuid not null references users(id) on delete cascade,
+  expires_at timestamptz not null,
+  created_at timestamptz not null default now()
+);
+
+create table if not exists friendships (
+  id uuid primary key default gen_random_uuid(),
+  requester_id uuid not null references users(id) on delete cascade,
+  addressee_id uuid not null references users(id) on delete cascade,
+  status text not null,
+  created_at timestamptz not null default now(),
+  updated_at timestamptz not null default now(),
+  unique (requester_id, addressee_id)
+);
+
+create table if not exists matches (
+  id uuid primary key default gen_random_uuid(),
+  mode text not null,
+  winner_id uuid references users(id),
+  loser_id uuid references users(id),
+  score_left integer not null,
+  score_right integer not null,
+  rating_delta integer not null default 16,
+  started_at timestamptz not null default now(),
+  ended_at timestamptz not null default now()
+);
+
+create table if not exists chat_messages (
+  id uuid primary key default gen_random_uuid(),
+  scope text not null,
+  room_id text,
+  sender_id uuid not null references users(id) on delete cascade,
+  body text not null,
+  created_at timestamptz not null default now()
+);
+
+create table if not exists tournaments (
+  id uuid primary key default gen_random_uuid(),
+  name text not null,
+  status text not null default 'open',
+  created_by uuid not null references users(id),
+  winner_id uuid references users(id),
+  capacity integer not null default 4,
+  created_at timestamptz not null default now()
+);
+
+create table if not exists tournament_entries (
+  id uuid primary key default gen_random_uuid(),
+  tournament_id uuid not null references tournaments(id) on delete cascade,
+  user_id uuid not null references users(id) on delete cascade,
+  seed integer not null,
+  created_at timestamptz not null default now(),
+  unique (tournament_id, user_id)
+);
+
+create table if not exists admin_actions (
+  id uuid primary key default gen_random_uuid(),
+  actor_id uuid references users(id),
+  target_user_id uuid references users(id),
+  action text not null,
+  reason text not null,
+  created_at timestamptz not null default now()
+);
+
+create index if not exists matches_ended_at_idx on matches (ended_at desc);
+create index if not exists chat_messages_scope_idx on chat_messages (scope, created_at desc);
+`;
diff --git a/tsconfig.base.json b/tsconfig.base.json
index ca7c0b4..e0198a8 100644
--- a/tsconfig.base.json
+++ b/tsconfig.base.json
@@ -11,7 +11,8 @@
     "skipLibCheck": true,
     "forceConsistentCasingInFileNames": true,
     "paths": {
-      "@pong-pong/shared": ["./packages/shared/src/index.ts"]
+      "@pong-pong/shared": ["./packages/shared/src/index.ts"],
+      "@pong-pong/db": ["./packages/db/src/index.ts"]
     }
   }
 }


## `feat(db): 데이터베이스 CLI 명령 연결`

diff --git a/packages/db/package.json b/packages/db/package.json
index 65043c4..99ff8d1 100644
--- a/packages/db/package.json
+++ b/packages/db/package.json
@@ -8,7 +8,11 @@
   },
   "scripts": {
     "build": "tsc --noEmit",
-    "typecheck": "tsc --noEmit"
+    "typecheck": "tsc --noEmit",
+    "migrate": "tsx src/cli.ts migrate",
+    "seed": "tsx src/cli.ts seed",
+    "memory-smoke": "tsx src/cli.ts memory-smoke",
+    "test": "vitest run"
   },
   "dependencies": {
     "@pong-pong/shared": "workspace:*",
diff --git a/packages/db/src/cli.ts b/packages/db/src/cli.ts
new file mode 100644
index 0000000..1398d25
--- /dev/null
+++ b/packages/db/src/cli.ts
@@ -0,0 +1,26 @@
+import { createMemoryRepository, createPostgresRepository } from "./index";
+
+const command = process.argv[2];
+const databaseUrl = process.env.DATABASE_URL;
+
+if (!databaseUrl) {
+  throw new Error("DATABASE_URL is required for database CLI commands");
+}
+
+const repo = createPostgresRepository(databaseUrl);
+
+try {
+  if (command === "migrate" || command === "seed") {
+    await repo.ensureSeedData();
+    console.log(command === "migrate" ? "migrated" : "seeded");
+  } else if (command === "memory-smoke") {
+    const memory = createMemoryRepository();
+    await memory.ensureSeedData();
+    await memory.close();
+    console.log("ok");
+  } else {
+    throw new Error("Usage: pnpm --filter @pong-pong/db migrate|seed");
+  }
+} finally {
+  await repo.close();
+}


## `refactor(db): SQL migration lifecycle 분리`

diff --git a/packages/db/migrations/001_initial.sql b/packages/db/migrations/001_initial.sql
index 8c95263..c604d9d 100644
--- a/packages/db/migrations/001_initial.sql
+++ b/packages/db/migrations/001_initial.sql
@@ -11,10 +11,13 @@ create table if not exists users (
   rating integer not null default 1200,
   wins integer not null default 0,
   losses integer not null default 0,
+  is_npc boolean not null default false,
   created_at timestamptz not null default now(),
   banned_at timestamptz
 );
 
+alter table users add column if not exists is_npc boolean not null default false;
+
 create table if not exists sessions (
   token text primary key,
   user_id uuid not null references users(id) on delete cascade,
diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index 10f2f11..342ab5f 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -2,7 +2,6 @@ import { randomUUID } from "node:crypto";
 import { Kysely, PostgresDialect, sql } from "kysely";
 import { Pool } from "pg";
 import type { AdminActionSummary, ChatMessage, DashboardSummary, FriendSummary, LeaderboardEntry, MatchMode, MatchSummary, PublicUser, SessionUser, TournamentMatchSummary, TournamentSummary } from "@pong-pong/shared";
-import { initialMigrationSql } from "./migrations";
 import { toAdminActionSummary, toChatMessage, toFriendSummary, toMatchSummary, toPublicUser, toSessionUser, toTournamentMatchRecord, toTournamentMatchSummary, toTournamentSummary } from "./rowMappers";
 import type { AdminActionRow, ChatMessageRow, ChatMessageWithSenderRow, Database, FriendshipWithUserRow, MatchWithHandlesRow, MemoryUserRow, TournamentMatchRow, TournamentRow, TournamentWithCreatorRow, UserRow } from "./schema";
 
@@ -100,7 +99,6 @@ class PostgresRepository implements AppRepository {
   }
 
   async ensureSeedData(): Promise<void> {
-    await sql.raw(initialMigrationSql).execute(this.db);
     const players: DevLoginInput[] = [
       { handle: "spin-doctor", displayName: "스핀닥터", email: "spin@pong.local" },
       { handle: "paddle-pro", displayName: "패들프로", email: "paddle@pong.local" },
diff --git a/packages/db/src/migrations.ts b/packages/db/src/migrations.ts
deleted file mode 100644
index 72cf363..0000000
--- a/packages/db/src/migrations.ts
+++ /dev/null
@@ -1,109 +0,0 @@
-export const initialMigrationSql = `
-create extension if not exists pgcrypto;
-
-create table if not exists users (
-  id uuid primary key default gen_random_uuid(),
-  email text unique,
-  handle text not null unique,
-  display_name text not null,
-  avatar_key text not null default 'blue',
-  role text not null default 'user',
-  status text not null default 'active',
-  rating integer not null default 1200,
-  wins integer not null default 0,
-  losses integer not null default 0,
-  is_npc boolean not null default false,
-  created_at timestamptz not null default now(),
-  banned_at timestamptz
-);
-
-alter table users add column if not exists is_npc boolean not null default false;
-
-create table if not exists sessions (
-  token text primary key,
-  user_id uuid not null references users(id) on delete cascade,
-  expires_at timestamptz not null,
-  created_at timestamptz not null default now()
-);
-
-create table if not exists friendships (
-  id uuid primary key default gen_random_uuid(),
-  requester_id uuid not null references users(id) on delete cascade,
-  addressee_id uuid not null references users(id) on delete cascade,
-  status text not null,
-  created_at timestamptz not null default now(),
-  updated_at timestamptz not null default now(),
-  unique (requester_id, addressee_id)
-);
-
-create table if not exists matches (
-  id uuid primary key default gen_random_uuid(),
-  mode text not null,
-  winner_id uuid references users(id),
-  loser_id uuid references users(id),
-  score_left integer not null,
-  score_right integer not null,
-  rating_delta integer not null default 16,
-  started_at timestamptz not null default now(),
-  ended_at timestamptz not null default now()
-);
-
-create table if not exists chat_messages (
-  id uuid primary key default gen_random_uuid(),
-  scope text not null,
-  room_id text,
-  sender_id uuid not null references users(id) on delete cascade,
-  body text not null,
-  created_at timestamptz not null default now()
-);
-
-create table if not exists tournaments (
-  id uuid primary key default gen_random_uuid(),
-  name text not null,
-  status text not null default 'open',
-  created_by uuid not null references users(id),
-  winner_id uuid references users(id),
-  capacity integer not null default 4,
-  created_at timestamptz not null default now()
-);
-
-create table if not exists tournament_entries (
-  id uuid primary key default gen_random_uuid(),
-  tournament_id uuid not null references tournaments(id) on delete cascade,
-  user_id uuid not null references users(id) on delete cascade,
-  seed integer not null,
-  created_at timestamptz not null default now(),
-  unique (tournament_id, user_id)
-);
-
-create table if not exists tournament_matches (
-  id uuid primary key default gen_random_uuid(),
-  tournament_id uuid not null references tournaments(id) on delete cascade,
-  round text not null,
-  slot integer not null,
-  status text not null default 'ready',
-  left_user_id uuid references users(id),
-  right_user_id uuid references users(id),
-  winner_id uuid references users(id),
-  room_id text,
-  match_id uuid references matches(id),
-  score_left integer,
-  score_right integer,
-  created_at timestamptz not null default now(),
-  updated_at timestamptz not null default now(),
-  unique (tournament_id, round, slot)
-);
-
-create table if not exists admin_actions (
-  id uuid primary key default gen_random_uuid(),
-  actor_id uuid references users(id),
-  target_user_id uuid references users(id),
-  action text not null,
-  reason text not null,
-  created_at timestamptz not null default now()
-);
-
-create index if not exists matches_ended_at_idx on matches (ended_at desc);
-create index if not exists chat_messages_scope_idx on chat_messages (scope, created_at desc);
-create index if not exists tournament_matches_tournament_idx on tournament_matches (tournament_id, round, slot);
-`;
diff --git a/packages/db/src/migrator.ts b/packages/db/src/migrator.ts
new file mode 100644
index 0000000..e1c9c45
--- /dev/null
+++ b/packages/db/src/migrator.ts
@@ -0,0 +1,61 @@
+import { readFile, readdir } from "node:fs/promises";
+import { fileURLToPath } from "node:url";
+import { basename, extname, join } from "node:path";
+import {
+  Kysely,
+  Migrator,
+  PostgresDialect,
+  sql,
+  type Migration,
+  type MigrationProvider
+} from "kysely";
+import { Pool } from "pg";
+import type { Database } from "./schema";
+
+const migrationsDirectory = fileURLToPath(
+  new URL("../migrations", import.meta.url)
+);
+
+class SqlMigrationProvider implements MigrationProvider {
+  async getMigrations(): Promise<Record<string, Migration>> {
+    const filenames = (await readdir(migrationsDirectory))
+      .filter((filename) => extname(filename) === ".sql")
+      .sort();
+    const migrations = await Promise.all(
+      filenames.map(async (filename) => {
+        const statement = await readFile(join(migrationsDirectory, filename), "utf8");
+        return [
+          basename(filename, ".sql"),
+          {
+            up: async (db) => {
+              await sql.raw(statement).execute(db);
+            }
+          } satisfies Migration
+        ] as const;
+      })
+    );
+
+    return Object.fromEntries(migrations);
+  }
+}
+
+export async function migrateDatabase(databaseUrl: string): Promise<void> {
+  const pool = new Pool({ connectionString: databaseUrl });
+  const db = new Kysely<Database>({ dialect: new PostgresDialect({ pool }) });
+
+  try {
+    const migrator = new Migrator({
+      db,
+      provider: new SqlMigrationProvider()
+    });
+    const { error, results } = await migrator.migrateToLatest();
+
+    if (error) {
+      const failedMigration = results?.find((result) => result.status === "Error");
+      const suffix = failedMigration ? ` (${failedMigration.migrationName})` : "";
+      throw new Error(`Database migration failed${suffix}`, { cause: error });
+    }
+  } finally {
+    await db.destroy();
+  }
+}


## `feat(db): 환경별 seed profile 분리`

diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index 342ab5f..97a431a 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -13,6 +13,8 @@ export interface DevLoginInput {
   email?: string | null;
 }
 
+export type SeedProfile = "development" | "demo";
+
 type NpcSeed = { handle: string; displayName: string; rating: number; avatarKey: string };
 const NPC_PLAYERS: NpcSeed[] = [
   { handle: "npc-rally-1100", displayName: "AI 랠리 1100", rating: 1100, avatarKey: "green" },
@@ -47,7 +49,7 @@ export interface TournamentMatchRecord {
 
 export interface AppRepository {
   close(): Promise<void>;
-  ensureSeedData(): Promise<void>;
+  ensureSeedData(profile?: SeedProfile): Promise<void>;
   upsertDevUser(input: DevLoginInput): Promise<SessionUser>;
   createSession(userId: string): Promise<string>;
   getSessionUser(token: string | undefined): Promise<SessionUser | null>;
@@ -98,23 +100,27 @@ class PostgresRepository implements AppRepository {
     await this.pool.end().catch(() => undefined);
   }
 
-  async ensureSeedData(): Promise<void> {
-    const players: DevLoginInput[] = [
-      { handle: "spin-doctor", displayName: "스핀닥터", email: "spin@pong.local" },
-      { handle: "paddle-pro", displayName: "패들프로", email: "paddle@pong.local" },
-      { handle: "net-ninja", displayName: "네트닌자", email: "net@pong.local" },
-      { handle: "top-spin", displayName: "탑스핀", email: "top@pong.local" },
-      { handle: "admin", displayName: "운영자", email: "admin@pong.local" }
-    ];
-    for (const player of players) {
-      await this.upsertDevUser(player);
+  async ensureSeedData(profile: SeedProfile = "development"): Promise<void> {
+    if (profile === "development") {
+      const players: DevLoginInput[] = [
+        { handle: "spin-doctor", displayName: "스핀닥터", email: "spin@pong.local" },
+        { handle: "paddle-pro", displayName: "패들프로", email: "paddle@pong.local" },
+        { handle: "net-ninja", displayName: "네트닌자", email: "net@pong.local" },
+        { handle: "top-spin", displayName: "탑스핀", email: "top@pong.local" },
+        { handle: "admin", displayName: "운영자", email: "admin@pong.local" }
+      ];
+      for (const player of players) {
+        await this.upsertDevUser(player);
+      }
     }
     for (const npc of NPC_PLAYERS) await this.upsertNpc(npc);
-    await sql`update users set role = 'admin', rating = 1680 where handle = 'admin'`.execute(this.db);
-    await sql`update users set rating = 1723, wins = 32, losses = 11 where handle = 'spin-doctor'`.execute(this.db);
-    await sql`update users set rating = 1640, wins = 24, losses = 13 where handle = 'paddle-pro'`.execute(this.db);
-    await sql`update users set rating = 1512, wins = 18, losses = 15 where handle = 'net-ninja'`.execute(this.db);
-    await sql`update users set rating = 1450, wins = 15, losses = 17 where handle = 'top-spin'`.execute(this.db);
+    if (profile === "development") {
+      await sql`update users set role = 'admin', rating = 1680 where handle = 'admin'`.execute(this.db);
+      await sql`update users set rating = 1723, wins = 32, losses = 11 where handle = 'spin-doctor'`.execute(this.db);
+      await sql`update users set rating = 1640, wins = 24, losses = 13 where handle = 'paddle-pro'`.execute(this.db);
+      await sql`update users set rating = 1512, wins = 18, losses = 15 where handle = 'net-ninja'`.execute(this.db);
+      await sql`update users set rating = 1450, wins = 15, losses = 17 where handle = 'top-spin'`.execute(this.db);
+    }
   }
 
   async upsertDevUser(input: DevLoginInput): Promise<SessionUser> {
@@ -420,14 +426,16 @@ class MemoryRepository implements AppRepository {
 
   async close(): Promise<void> {}
 
-  async ensureSeedData(): Promise<void> {
-    for (const player of [
-      { handle: "spin-doctor", displayName: "스핀닥터", email: "spin@pong.local" },
-      { handle: "paddle-pro", displayName: "패들프로", email: "paddle@pong.local" },
-      { handle: "net-ninja", displayName: "네트닌자", email: "net@pong.local" },
-      { handle: "admin", displayName: "운영자", email: "admin@pong.local" }
-    ]) {
-      await this.upsertDevUser(player);
+  async ensureSeedData(profile: SeedProfile = "development"): Promise<void> {
+    if (profile === "development") {
+      for (const player of [
+        { handle: "spin-doctor", displayName: "스핀닥터", email: "spin@pong.local" },
+        { handle: "paddle-pro", displayName: "패들프로", email: "paddle@pong.local" },
+        { handle: "net-ninja", displayName: "네트닌자", email: "net@pong.local" },
+        { handle: "admin", displayName: "운영자", email: "admin@pong.local" }
+      ]) {
+        await this.upsertDevUser(player);
+      }
     }
     for (const npc of NPC_PLAYERS) {
       const existing = [...this.users.values()].find((user) => user.handle === npc.handle);


