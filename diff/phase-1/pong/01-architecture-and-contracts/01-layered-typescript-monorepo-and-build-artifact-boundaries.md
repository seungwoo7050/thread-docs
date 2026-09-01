# 계층형 TypeScript 모노레포와 빌드 산출물 경계

## `chore(workspace): pnpm 모노레포 경계 구성`

diff --git a/Makefile b/Makefile
new file mode 100644
index 0000000..878f0a9
--- /dev/null
+++ b/Makefile
@@ -0,0 +1,10 @@
+.PHONY: install typecheck build
+
+install:
+	pnpm install
+
+typecheck:
+	pnpm -r typecheck
+
+build:
+	pnpm -r build
diff --git a/package.json b/package.json
new file mode 100644
index 0000000..8e619ab
--- /dev/null
+++ b/package.json
@@ -0,0 +1,14 @@
+{
+  "name": "pong-pong",
+  "version": "0.1.0",
+  "private": true,
+  "packageManager": "pnpm@10.32.1",
+  "scripts": {
+    "build": "pnpm -r build",
+    "typecheck": "pnpm -r typecheck"
+  },
+  "devDependencies": {
+    "@types/node": "^22.15.3",
+    "typescript": "^5.8.3"
+  }
+}
diff --git a/pnpm-workspace.yaml b/pnpm-workspace.yaml
new file mode 100644
index 0000000..3ff5faa
--- /dev/null
+++ b/pnpm-workspace.yaml
@@ -0,0 +1,3 @@
+packages:
+  - "apps/*"
+  - "packages/*"
diff --git a/tsconfig.base.json b/tsconfig.base.json
new file mode 100644
index 0000000..93abfaa
--- /dev/null
+++ b/tsconfig.base.json
@@ -0,0 +1,14 @@
+{
+  "compilerOptions": {
+    "target": "ES2022",
+    "lib": ["ES2022", "DOM"],
+    "module": "ESNext",
+    "moduleResolution": "Bundler",
+    "resolveJsonModule": true,
+    "esModuleInterop": true,
+    "allowSyntheticDefaultImports": true,
+    "strict": true,
+    "skipLibCheck": true,
+    "forceConsistentCasingInFileNames": true
+  }
+}


## `chore(shared): 공유 패키지 경계 구성`

diff --git a/packages/shared/package.json b/packages/shared/package.json
new file mode 100644
index 0000000..610f892
--- /dev/null
+++ b/packages/shared/package.json
@@ -0,0 +1,20 @@
+{
+  "name": "@pong-pong/shared",
+  "version": "0.1.0",
+  "private": true,
+  "type": "module",
+  "exports": {
+    ".": "./src/index.ts"
+  },
+  "scripts": {
+    "build": "tsc --noEmit",
+    "typecheck": "tsc --noEmit"
+  },
+  "dependencies": {
+    "zod": "^3.24.4"
+  },
+  "devDependencies": {
+    "typescript": "^5.8.3",
+    "vitest": "^3.1.2"
+  }
+}
diff --git a/packages/shared/tsconfig.json b/packages/shared/tsconfig.json
new file mode 100644
index 0000000..0d7100c
--- /dev/null
+++ b/packages/shared/tsconfig.json
@@ -0,0 +1,7 @@
+{
+  "extends": "../../tsconfig.base.json",
+  "compilerOptions": {
+    "noEmit": true
+  },
+  "include": ["src/**/*.ts"]
+}
diff --git a/tsconfig.base.json b/tsconfig.base.json
index 93abfaa..ca7c0b4 100644
--- a/tsconfig.base.json
+++ b/tsconfig.base.json
@@ -9,6 +9,9 @@
     "allowSyntheticDefaultImports": true,
     "strict": true,
     "skipLibCheck": true,
-    "forceConsistentCasingInFileNames": true
+    "forceConsistentCasingInFileNames": true,
+    "paths": {
+      "@pong-pong/shared": ["./packages/shared/src/index.ts"]
+    }
   }
 }


## `chore(db): PostgreSQL 패키지 경계 구성`

diff --git a/packages/db/package.json b/packages/db/package.json
new file mode 100644
index 0000000..0089b9d
--- /dev/null
+++ b/packages/db/package.json
@@ -0,0 +1,17 @@
+{
+  "name": "@pong-pong/db",
+  "version": "0.1.0",
+  "private": true,
+  "type": "module",
+  "dependencies": {
+    "@pong-pong/shared": "workspace:*",
+    "kysely": "^0.28.2",
+    "pg": "^8.15.6"
+  },
+  "devDependencies": {
+    "@types/pg": "^8.11.14",
+    "tsx": "^4.19.2",
+    "typescript": "^5.8.3",
+    "vitest": "^3.1.2"
+  }
+}
diff --git a/packages/db/tsconfig.json b/packages/db/tsconfig.json
new file mode 100644
index 0000000..0d7100c
--- /dev/null
+++ b/packages/db/tsconfig.json
@@ -0,0 +1,7 @@
+{
+  "extends": "../../tsconfig.base.json",
+  "compilerOptions": {
+    "noEmit": true
+  },
+  "include": ["src/**/*.ts"]
+}


## `chore(api): Fastify 패키지 경계 구성`

diff --git a/apps/api/package.json b/apps/api/package.json
new file mode 100644
index 0000000..20cee3f
--- /dev/null
+++ b/apps/api/package.json
@@ -0,0 +1,28 @@
+{
+  "name": "@pong-pong/api",
+  "version": "0.1.0",
+  "private": true,
+  "type": "module",
+  "scripts": {
+    "build": "tsc --noEmit",
+    "typecheck": "tsc --noEmit",
+    "dev": "tsx watch src/index.ts",
+    "start": "tsx src/index.ts"
+  },
+  "dependencies": {
+    "@fastify/cookie": "^11.0.1",
+    "@fastify/cors": "^11.0.0",
+    "@fastify/websocket": "^11.0.1",
+    "@pong-pong/db": "workspace:*",
+    "@pong-pong/shared": "workspace:*",
+    "fastify": "^5.2.1",
+    "ws": "^8.18.0",
+    "zod": "^3.24.4"
+  },
+  "devDependencies": {
+    "@types/ws": "^8.5.13",
+    "tsx": "^4.19.2",
+    "typescript": "^5.8.3",
+    "vitest": "^3.1.2"
+  }
+}
diff --git a/apps/api/tsconfig.json b/apps/api/tsconfig.json
new file mode 100644
index 0000000..734b278
--- /dev/null
+++ b/apps/api/tsconfig.json
@@ -0,0 +1,8 @@
+{
+  "extends": "../../tsconfig.base.json",
+  "compilerOptions": {
+    "noEmit": true,
+    "types": ["node"]
+  },
+  "include": ["src/**/*.ts"]
+}


## `chore(web): Next.js runtime 경계 구성`

diff --git a/apps/web/next-env.d.ts b/apps/web/next-env.d.ts
new file mode 100644
index 0000000..830fb59
--- /dev/null
+++ b/apps/web/next-env.d.ts
@@ -0,0 +1,6 @@
+/// <reference types="next" />
+/// <reference types="next/image-types/global" />
+/// <reference path="./.next/types/routes.d.ts" />
+
+// NOTE: This file should not be edited
+// see https://nextjs.org/docs/app/api-reference/config/typescript for more information.
diff --git a/apps/web/next.config.mjs b/apps/web/next.config.mjs
new file mode 100644
index 0000000..a42e43b
--- /dev/null
+++ b/apps/web/next.config.mjs
@@ -0,0 +1,6 @@
+/** @type {import('next').NextConfig} */
+const nextConfig = {
+  transpilePackages: ["@pong-pong/shared"]
+};
+
+export default nextConfig;
diff --git a/apps/web/package.json b/apps/web/package.json
new file mode 100644
index 0000000..1b79534
--- /dev/null
+++ b/apps/web/package.json
@@ -0,0 +1,27 @@
+{
+  "name": "@pong-pong/web",
+  "version": "0.1.0",
+  "private": true,
+  "type": "module",
+  "scripts": {
+    "dev": "next dev --hostname 0.0.0.0 --port 3000",
+    "build": "next build",
+    "typecheck": "tsc --noEmit"
+  },
+  "dependencies": {
+    "@pong-pong/shared": "workspace:*",
+    "lucide-react": "^0.511.0",
+    "next": "^15.3.2",
+    "react": "^19.1.0",
+    "react-dom": "^19.1.0"
+  },
+  "devDependencies": {
+    "@types/react": "^19.1.3",
+    "@types/react-dom": "^19.1.3",
+    "autoprefixer": "^10.4.21",
+    "postcss": "^8.5.3",
+    "tailwindcss": "^3.4.17",
+    "typescript": "^5.8.3",
+    "vitest": "^3.1.2"
+  }
+}
diff --git a/apps/web/tsconfig.json b/apps/web/tsconfig.json
new file mode 100644
index 0000000..3867a3a
--- /dev/null
+++ b/apps/web/tsconfig.json
@@ -0,0 +1,36 @@
+{
+  "extends": "../../tsconfig.base.json",
+  "compilerOptions": {
+    "jsx": "preserve",
+    "noEmit": true,
+    "incremental": true,
+    "baseUrl": ".",
+    "paths": {
+      "@/*": [
+        "src/*"
+      ],
+      "@pong-pong/shared": [
+        "../../packages/shared/src/index.ts"
+      ],
+      "@pong-pong/db": [
+        "../../packages/db/src/index.ts"
+      ]
+    },
+    "plugins": [
+      {
+        "name": "next"
+      }
+    ],
+    "allowJs": true,
+    "isolatedModules": true
+  },
+  "include": [
+    "next-env.d.ts",
+    "src/**/*.ts",
+    "src/**/*.tsx",
+    ".next/types/**/*.ts"
+  ],
+  "exclude": [
+    "node_modules"
+  ]
+}


## `build(shared): production package artifact 구성`

diff --git a/packages/shared/package.json b/packages/shared/package.json
index cd78861..454c0f0 100644
--- a/packages/shared/package.json
+++ b/packages/shared/package.json
@@ -3,11 +3,18 @@
   "version": "0.1.0",
   "private": true,
   "type": "module",
+  "main": "./dist/index.js",
+  "types": "./dist/index.d.ts",
   "exports": {
-    ".": "./src/index.ts"
+    ".": {
+      "development": "./src/index.ts",
+      "types": "./dist/index.d.ts",
+      "import": "./dist/index.js",
+      "default": "./dist/index.js"
+    }
   },
   "scripts": {
-    "build": "tsc --noEmit",
+    "build": "tsc -p tsconfig.build.json",
     "typecheck": "tsc --noEmit",
     "test": "vitest run"
   },
diff --git a/packages/shared/src/index.ts b/packages/shared/src/index.ts
index 45a906e..482023e 100644
--- a/packages/shared/src/index.ts
+++ b/packages/shared/src/index.ts
@@ -1,3 +1,3 @@
-export * from "./http";
-export * from "./game";
-export * from "./ws";
+export * from "./http.js";
+export * from "./game.js";
+export * from "./ws.js";
diff --git a/packages/shared/src/ws.ts b/packages/shared/src/ws.ts
index 43efae3..3461ba9 100644
--- a/packages/shared/src/ws.ts
+++ b/packages/shared/src/ws.ts
@@ -1,6 +1,6 @@
 import { z } from "zod";
-import { chatMessageSchema } from "./http";
-import { gameFinishedSchema, gameSnapshotSchema, playerSideSchema } from "./game";
+import { chatMessageSchema } from "./http.js";
+import { gameFinishedSchema, gameSnapshotSchema, playerSideSchema } from "./game.js";
 
 const version = { v: z.literal(1) } as const;
 const roomIdSchema = z.string().min(1);
diff --git a/packages/shared/tsconfig.build.json b/packages/shared/tsconfig.build.json
new file mode 100644
index 0000000..d282dfd
--- /dev/null
+++ b/packages/shared/tsconfig.build.json
@@ -0,0 +1,16 @@
+{
+  "extends": "./tsconfig.json",
+  "compilerOptions": {
+    "declaration": true,
+    "declarationMap": true,
+    "module": "NodeNext",
+    "moduleResolution": "NodeNext",
+    "noEmit": false,
+    "outDir": "dist",
+    "paths": {},
+    "rootDir": "src",
+    "sourceMap": true
+  },
+  "include": ["src/**/*.ts"],
+  "exclude": ["src/**/*.test.ts"]
+}


## `build(db): production package artifact 구성`

diff --git a/packages/db/package.json b/packages/db/package.json
index e5e3e5e..ca9bebc 100644
--- a/packages/db/package.json
+++ b/packages/db/package.json
@@ -3,15 +3,23 @@
   "version": "0.1.0",
   "private": true,
   "type": "module",
+  "main": "./dist/index.js",
+  "types": "./dist/index.d.ts",
   "exports": {
-    ".": "./src/index.ts"
+    ".": {
+      "development": "./src/index.ts",
+      "types": "./dist/index.d.ts",
+      "import": "./dist/index.js",
+      "default": "./dist/index.js"
+    }
   },
   "scripts": {
-    "build": "tsc --noEmit",
+    "build": "tsc -p tsconfig.build.json && cp -R migrations dist/",
     "typecheck": "tsc --noEmit",
     "migrate": "tsx src/cli.ts migrate",
     "seed:dev": "tsx src/cli.ts seed:dev",
     "seed:demo": "tsx src/cli.ts seed:demo",
+    "migrate:prod": "node dist/cli.js migrate",
     "user:set-role": "tsx src/cli.ts user:set-role",
     "memory-smoke": "tsx src/cli.ts memory-smoke",
     "test": "vitest run --exclude \"**/*.integration.test.ts\"",
diff --git a/packages/db/src/cli.ts b/packages/db/src/cli.ts
index 66bbfda..572ce97 100644
--- a/packages/db/src/cli.ts
+++ b/packages/db/src/cli.ts
@@ -1,5 +1,5 @@
-import { createMemoryRepository, createPostgresRepository } from "./index";
-import { migrateDatabase } from "./migrator";
+import { createMemoryRepository, createPostgresRepository } from "./index.js";
+import { migrateDatabase } from "./migrator.js";
 
 const command = process.argv[2];
 
diff --git a/packages/db/src/index.ts b/packages/db/src/index.ts
index 1233c8a..459e032 100644
--- a/packages/db/src/index.ts
+++ b/packages/db/src/index.ts
@@ -25,7 +25,7 @@ import {
   toTournamentMatchRecord,
   toTournamentMatchSummary,
   toTournamentSummary
-} from "./rowMappers";
+} from "./rowMappers.js";
 import type {
   AdminActionRow,
   ChatMessageRow,
@@ -38,9 +38,9 @@ import type {
   TournamentWithCreatorRow,
   UserProjectionRow,
   UserRow
-} from "./schema";
+} from "./schema.js";
 
-export type { Database } from "./schema";
+export type { Database } from "./schema.js";
 
 type MemoryFriendship = {
   id: string;
diff --git a/packages/db/src/migrator.ts b/packages/db/src/migrator.ts
index e1c9c45..05aef2e 100644
--- a/packages/db/src/migrator.ts
+++ b/packages/db/src/migrator.ts
@@ -10,7 +10,7 @@ import {
   type MigrationProvider
 } from "kysely";
 import { Pool } from "pg";
-import type { Database } from "./schema";
+import type { Database } from "./schema.js";
 
 const migrationsDirectory = fileURLToPath(
   new URL("../migrations", import.meta.url)
diff --git a/packages/db/src/rowMappers.ts b/packages/db/src/rowMappers.ts
index 435a09a..102c57b 100644
--- a/packages/db/src/rowMappers.ts
+++ b/packages/db/src/rowMappers.ts
@@ -16,7 +16,7 @@ import type {
   TournamentMatchRow,
   TournamentWithCreatorRow,
   UserProjectionRow
-} from "./schema";
+} from "./schema.js";
 
 export interface TournamentMatchRecordView {
   id: string;
diff --git a/packages/db/tsconfig.build.json b/packages/db/tsconfig.build.json
new file mode 100644
index 0000000..f732c7c
--- /dev/null
+++ b/packages/db/tsconfig.build.json
@@ -0,0 +1,16 @@
+{
+  "extends": "./tsconfig.json",
+  "compilerOptions": {
+    "declaration": true,
+    "declarationMap": true,
+    "module": "NodeNext",
+    "moduleResolution": "NodeNext",
+    "noEmit": false,
+    "outDir": "dist",
+    "paths": {},
+    "rootDir": "src",
+    "sourceMap": true
+  },
+  "include": ["src/**/*.ts"],
+  "exclude": ["src/**/*.test.ts", "src/**/*.integration.test.ts"]
+}


## `build(app): API와 Web production artifact 구성`

diff --git a/apps/api/package.json b/apps/api/package.json
index 5cd51b9..87a972a 100644
--- a/apps/api/package.json
+++ b/apps/api/package.json
@@ -4,10 +4,10 @@
   "private": true,
   "type": "module",
   "scripts": {
-    "build": "tsc --noEmit",
+    "build": "tsc -p tsconfig.build.json",
     "typecheck": "tsc --noEmit",
     "dev": "tsx watch src/index.ts",
-    "start": "tsx src/index.ts",
+    "start": "node dist/index.js",
     "test": "vitest run"
   },
   "dependencies": {
diff --git a/apps/api/src/game/pongAi.ts b/apps/api/src/game/pongAi.ts
index 4bf2a5a..1805a79 100644
--- a/apps/api/src/game/pongAi.ts
+++ b/apps/api/src/game/pongAi.ts
@@ -1,5 +1,5 @@
 import { BALL_RADIUS, GAME_HEIGHT, GAME_WIDTH, PADDLE_HEIGHT } from "@pong-pong/shared";
-import type { PaddleDirection, PongSimulationState } from "./pongSimulation";
+import type { PaddleDirection, PongSimulationState } from "./pongSimulation.js";
 
 interface AiProfile {
   reactionTicks: number;
diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index 306f017..d7bb212 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -14,13 +14,13 @@ import {
   type SessionUser,
   WINNING_SCORE
 } from "@pong-pong/shared";
-import { DEFAULT_TIMESTEP_MS } from "./game/fixedStepScheduler";
-import { ConnectionHeartbeat } from "./game/heartbeat";
-import { InputGate } from "./game/inputGate";
-import { HARD_BUFFERED_AMOUNT_BYTES, LatestSnapshotBuffer } from "./game/latestSnapshotBuffer";
-import { PongAi } from "./game/pongAi";
-import { PongSimulation, type PongSimulationState } from "./game/pongSimulation";
-import { RoomSession } from "./game/roomSession";
+import { DEFAULT_TIMESTEP_MS } from "./game/fixedStepScheduler.js";
+import { ConnectionHeartbeat } from "./game/heartbeat.js";
+import { InputGate } from "./game/inputGate.js";
+import { HARD_BUFFERED_AMOUNT_BYTES, LatestSnapshotBuffer } from "./game/latestSnapshotBuffer.js";
+import { PongAi } from "./game/pongAi.js";
+import { PongSimulation, type PongSimulationState } from "./game/pongSimulation.js";
+import { RoomSession } from "./game/roomSession.js";
 import { SharedRoomScheduler } from "./game/sharedRoomScheduler.js";
 import type { GuestSessionUser } from "./guestAccess.js";
 
diff --git a/apps/api/tsconfig.build.json b/apps/api/tsconfig.build.json
new file mode 100644
index 0000000..d282dfd
--- /dev/null
+++ b/apps/api/tsconfig.build.json
@@ -0,0 +1,16 @@
+{
+  "extends": "./tsconfig.json",
+  "compilerOptions": {
+    "declaration": true,
+    "declarationMap": true,
+    "module": "NodeNext",
+    "moduleResolution": "NodeNext",
+    "noEmit": false,
+    "outDir": "dist",
+    "paths": {},
+    "rootDir": "src",
+    "sourceMap": true
+  },
+  "include": ["src/**/*.ts"],
+  "exclude": ["src/**/*.test.ts"]
+}
diff --git a/apps/web/next.config.mjs b/apps/web/next.config.mjs
index a42e43b..8e74340 100644
--- a/apps/web/next.config.mjs
+++ b/apps/web/next.config.mjs
@@ -1,6 +1,17 @@
+import { fileURLToPath } from "node:url";
+
+const repositoryRoot = fileURLToPath(new URL("../..", import.meta.url));
+const sharedRuntime = fileURLToPath(new URL("../../packages/shared/dist/index.js", import.meta.url));
+
 /** @type {import('next').NextConfig} */
 const nextConfig = {
-  transpilePackages: ["@pong-pong/shared"]
+  output: "standalone",
+  outputFileTracingRoot: repositoryRoot,
+  transpilePackages: ["@pong-pong/shared"],
+  webpack(config) {
+    config.resolve.alias["@pong-pong/shared"] = sharedRuntime;
+    return config;
+  }
 };
 
 export default nextConfig;
diff --git a/apps/web/package.json b/apps/web/package.json
index e788f51..9d244be 100644
--- a/apps/web/package.json
+++ b/apps/web/package.json
@@ -4,10 +4,14 @@
   "private": true,
   "type": "module",
   "scripts": {
+    "predev": "pnpm --filter @pong-pong/shared build",
     "dev": "next dev --hostname 0.0.0.0 --port 3000",
+    "prebuild": "pnpm --filter @pong-pong/shared build",
     "build": "next build",
     "start": "next start --hostname 0.0.0.0 --port 3000",
+    "pretypecheck": "pnpm --filter @pong-pong/shared build",
     "typecheck": "tsc --noEmit",
+    "pretest": "pnpm --filter @pong-pong/shared build",
     "test": "vitest run"
   },
   "dependencies": {
diff --git a/apps/web/tsconfig.json b/apps/web/tsconfig.json
index 94a169d..345af57 100644
--- a/apps/web/tsconfig.json
+++ b/apps/web/tsconfig.json
@@ -8,12 +8,6 @@
     "paths": {
       "@/*": [
         "src/*"
-      ],
-      "@pong-pong/shared": [
-        "../../packages/shared/src/index.ts"
-      ],
-      "@pong-pong/db": [
-        "../../packages/db/src/index.ts"
       ]
     },
     "plugins": [
diff --git a/package.json b/package.json
index 2505cb5..bbeb1f5 100644
--- a/package.json
+++ b/package.json
@@ -8,13 +8,14 @@
     "pnpm": "10.32.1"
   },
   "scripts": {
-    "build": "pnpm -r build",
+    "build": "pnpm --filter @pong-pong/shared build && pnpm --filter @pong-pong/db build && pnpm --filter @pong-pong/api build && pnpm --filter @pong-pong/web build",
     "typecheck": "pnpm -r typecheck",
     "test": "pnpm unit",
     "unit": "pnpm -r test",
     "postgres-integration": "pnpm --filter @pong-pong/db postgres-integration",
     "smoke:http": "node tests/smoke-api.mjs",
     "smoke:ws": "node tests/smoke-ws.mjs",
+    "verify:build": "node tests/build-artifacts.mjs",
     "e2e": "playwright test",
     "test:e2e": "pnpm e2e",
     "dev": "pnpm --parallel --filter @pong-pong/api --filter @pong-pong/web dev"
