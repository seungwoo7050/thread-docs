## `build(docker): production API image 구성`

diff --git a/.dockerignore b/.dockerignore
new file mode 100644
index 0000000..5fd82f2
--- /dev/null
+++ b/.dockerignore
@@ -0,0 +1,14 @@
+.git
+.github
+node_modules
+**/node_modules
+**/.next
+**/dist
+coverage
+playwright-report
+test-results
+output
+.env
+.env.*
+*.log
+README.md
diff --git a/apps/api/Dockerfile b/apps/api/Dockerfile
new file mode 100644
index 0000000..b6fc947
--- /dev/null
+++ b/apps/api/Dockerfile
@@ -0,0 +1,39 @@
+FROM node:24.18.0-bookworm-slim AS dependencies
+
+RUN corepack enable && corepack prepare pnpm@10.32.1 --activate
+WORKDIR /app
+COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./
+COPY apps/api/package.json apps/api/package.json
+COPY packages/db/package.json packages/db/package.json
+COPY packages/shared/package.json packages/shared/package.json
+RUN pnpm install --frozen-lockfile
+
+FROM dependencies AS builder
+
+COPY tsconfig.base.json ./
+COPY apps/api apps/api
+COPY packages/db packages/db
+COPY packages/shared packages/shared
+RUN pnpm --filter @pong-pong/shared build \
+  && pnpm --filter @pong-pong/db build \
+  && pnpm --filter @pong-pong/api build
+
+FROM node:24.18.0-bookworm-slim AS runner
+
+ENV NODE_ENV=production
+WORKDIR /app
+COPY --from=builder /app/node_modules ./node_modules
+COPY --from=builder /app/package.json ./package.json
+COPY --from=builder /app/apps/api/package.json ./apps/api/package.json
+COPY --from=builder /app/apps/api/node_modules ./apps/api/node_modules
+COPY --from=builder /app/apps/api/dist ./apps/api/dist
+COPY --from=builder /app/packages/db/package.json ./packages/db/package.json
+COPY --from=builder /app/packages/db/node_modules ./packages/db/node_modules
+COPY --from=builder /app/packages/db/dist ./packages/db/dist
+COPY --from=builder /app/packages/shared/package.json ./packages/shared/package.json
+COPY --from=builder /app/packages/shared/node_modules ./packages/shared/node_modules
+COPY --from=builder /app/packages/shared/dist ./packages/shared/dist
+USER node
+
+EXPOSE 4000
+CMD ["node", "apps/api/dist/index.js"]


## `build(docker): production Web image 구성`

diff --git a/apps/web/Dockerfile b/apps/web/Dockerfile
new file mode 100644
index 0000000..f4d218f
--- /dev/null
+++ b/apps/web/Dockerfile
@@ -0,0 +1,35 @@
+FROM node:24.18.0-bookworm-slim AS dependencies
+
+RUN corepack enable && corepack prepare pnpm@10.32.1 --activate
+WORKDIR /app
+COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./
+COPY apps/web/package.json apps/web/package.json
+COPY packages/shared/package.json packages/shared/package.json
+RUN pnpm install --frozen-lockfile
+
+FROM dependencies AS builder
+
+ARG NEXT_PUBLIC_API_BASE_URL=http://localhost:8080/api
+ARG NEXT_PUBLIC_WS_URL=ws://localhost:8080/ws
+ARG NEXT_PUBLIC_APP_MODE=production
+ENV NEXT_PUBLIC_API_BASE_URL=$NEXT_PUBLIC_API_BASE_URL
+ENV NEXT_PUBLIC_WS_URL=$NEXT_PUBLIC_WS_URL
+ENV NEXT_PUBLIC_APP_MODE=$NEXT_PUBLIC_APP_MODE
+COPY tsconfig.base.json ./
+COPY apps/web apps/web
+COPY packages/shared packages/shared
+RUN pnpm --filter @pong-pong/shared build \
+  && pnpm --filter @pong-pong/web build
+
+FROM node:24.18.0-bookworm-slim AS runner
+
+ENV NODE_ENV=production
+ENV HOSTNAME=0.0.0.0
+ENV PORT=3000
+WORKDIR /app
+USER node
+COPY --from=builder --chown=node:node /app/apps/web/.next/standalone ./
+COPY --from=builder --chown=node:node /app/apps/web/.next/static ./apps/web/.next/static
+
+EXPOSE 3000
+CMD ["node", "apps/web/server.js"]


## `build(docker): Caddy reverse proxy 구성`

diff --git a/Caddy.Dockerfile b/Caddy.Dockerfile
new file mode 100644
index 0000000..3d02dbb
--- /dev/null
+++ b/Caddy.Dockerfile
@@ -0,0 +1,3 @@
+FROM caddy:2-alpine
+
+COPY Caddyfile /etc/caddy/Caddyfile
diff --git a/Caddyfile b/Caddyfile
index f3d17e9..74e1ced 100644
--- a/Caddyfile
+++ b/Caddyfile
@@ -1,13 +1,18 @@
 :8080 {
-	handle_path /api/* {
-		reverse_proxy api:4000
-	}
+	route {
+		@internalMetrics path /api/metrics
+		respond @internalMetrics 404
 
-	handle /ws {
-		reverse_proxy api:4000
-	}
+		handle_path /api/* {
+			reverse_proxy api:4000
+		}
+
+		handle /ws {
+			reverse_proxy api:4000
+		}
 
-	handle {
-		reverse_proxy web:3000
+		handle {
+			reverse_proxy web:3000
+		}
 	}
 }


## `build(docker): production container lifecycle 구성`

diff --git a/docker-compose.yml b/docker-compose.yml
index 8e17e7e..28f846c 100644
--- a/docker-compose.yml
+++ b/docker-compose.yml
@@ -4,9 +4,7 @@ services:
     environment:
       POSTGRES_DB: pong_pong
       POSTGRES_USER: pong
-      POSTGRES_PASSWORD: pong
-    ports:
-      - "5432:5432"
+      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?POSTGRES_PASSWORD must be set}
     healthcheck:
       test: ["CMD-SHELL", "pg_isready -U pong -d pong_pong"]
       interval: 5s
@@ -15,52 +13,73 @@ services:
     volumes:
       - pong-pong-db:/var/lib/postgresql/data
 
-  api:
-    image: node:24.18.0-bookworm-slim
-    working_dir: /app
-    command: sh -c "corepack enable && corepack prepare pnpm@10.32.1 --activate && pnpm install --frozen-lockfile && pnpm --filter @pong-pong/api start"
+  migrate:
+    image: pong-pong-api:local
+    build:
+      context: .
+      dockerfile: apps/api/Dockerfile
+    command: ["node", "packages/db/dist/cli.js", "migrate"]
     environment:
-      DATABASE_URL: postgres://pong:pong@db:5432/pong_pong
-      SESSION_SECRET: dev-session-secret
-      API_PORT: 4000
-      WEB_ORIGIN: http://localhost:8080
-    volumes:
-      - .:/app
-      - api-node-modules:/app/node_modules
+      DATABASE_URL: postgres://pong:${POSTGRES_PASSWORD:?POSTGRES_PASSWORD must be set}@db:5432/pong_pong
     depends_on:
       db:
         condition: service_healthy
-    ports:
-      - "4000:4000"
+    restart: "no"
 
-  web:
-    image: node:24.18.0-bookworm-slim
-    working_dir: /app
-    command: sh -c "corepack enable && corepack prepare pnpm@10.32.1 --activate && pnpm install --frozen-lockfile && pnpm --filter @pong-pong/web build && pnpm --filter @pong-pong/web start"
+  api:
+    image: pong-pong-api:local
+    build:
+      context: .
+      dockerfile: apps/api/Dockerfile
     environment:
-      NEXT_PUBLIC_API_BASE_URL: http://localhost:8080/api
-      NEXT_PUBLIC_WS_URL: ws://localhost:8080/ws
-    volumes:
-      - .:/app
-      - web-node-modules:/app/node_modules
-      - web-next:/app/apps/web/.next
+      APP_MODE: ${APP_MODE:-production}
+      TRUST_PROXY: "1"
+      DATABASE_URL: postgres://pong:${POSTGRES_PASSWORD:?POSTGRES_PASSWORD must be set}@db:5432/pong_pong
+      SESSION_SECRET: ${SESSION_SECRET:?SESSION_SECRET must be set}
+      API_PORT: 4000
+      WEB_ORIGIN: ${PUBLIC_ORIGIN:-http://localhost:8080}
     depends_on:
-      - api
-    ports:
-      - "3000:3000"
+      migrate:
+        condition: service_completed_successfully
+    healthcheck:
+      test: ["CMD", "node", "-e", "fetch('http://127.0.0.1:4000/health/ready').then(r=>{if(!r.ok)process.exit(1)}).catch(()=>process.exit(1))"]
+      interval: 10s
+      timeout: 3s
+      retries: 12
+    expose:
+      - "4000"
+
+  web:
+    image: pong-pong-web:local
+    build:
+      context: .
+      dockerfile: apps/web/Dockerfile
+      args:
+        NEXT_PUBLIC_API_BASE_URL: ${PUBLIC_ORIGIN:-http://localhost:8080}/api
+        NEXT_PUBLIC_WS_URL: ${PUBLIC_WS_URL:-ws://localhost:8080/ws}
+        NEXT_PUBLIC_APP_MODE: ${APP_MODE:-production}
+    depends_on:
+      api:
+        condition: service_healthy
+    healthcheck:
+      test: ["CMD", "node", "-e", "fetch('http://127.0.0.1:3000').then(r=>{if(!r.ok)process.exit(1)}).catch(()=>process.exit(1))"]
+      interval: 10s
+      timeout: 3s
+      retries: 12
+    expose:
+      - "3000"
 
   caddy:
-    image: caddy:2-alpine
+    build:
+      context: .
+      dockerfile: Caddy.Dockerfile
+    depends_on:
+      api:
+        condition: service_healthy
+      web:
+        condition: service_healthy
     ports:
       - "8080:8080"
-    volumes:
-      - ./Caddyfile:/etc/caddy/Caddyfile:ro
-    depends_on:
-      - web
-      - api
 
 volumes:
   pong-pong-db:
-  api-node-modules:
-  web-node-modules:
-  web-next:


## `test(docker): production container contract 검증`

diff --git a/tests/docker-production.test.mjs b/tests/docker-production.test.mjs
new file mode 100644
index 0000000..7b93e88
--- /dev/null
+++ b/tests/docker-production.test.mjs
@@ -0,0 +1,60 @@
+import { readFileSync } from "node:fs";
+import { resolve } from "node:path";
+import { spawnSync } from "node:child_process";
+import test from "node:test";
+import assert from "node:assert/strict";
+
+const root = resolve(import.meta.dirname, "..");
+
+test("production compose exposes only Caddy and runs migration once", () => {
+  const result = spawnSync("docker", ["compose", "config", "--format", "json"], {
+    cwd: root,
+    encoding: "utf8",
+    env: {
+      ...process.env,
+      POSTGRES_PASSWORD: "compose-contract-password",
+      SESSION_SECRET: "compose-contract-session-secret-32-bytes"
+    }
+  });
+  assert.equal(result.status, 0, result.stderr || result.stdout);
+
+  const config = JSON.parse(result.stdout);
+  const services = config.services;
+  assert.deepEqual(Object.keys(services).sort(), ["api", "caddy", "db", "migrate", "web"]);
+  assert.deepEqual(Object.entries(services)
+    .filter(([, service]) => Array.isArray(service.ports) && service.ports.length > 0)
+    .map(([name]) => name), ["caddy"]);
+  assert.deepEqual(services.migrate.command, ["node", "packages/db/dist/cli.js", "migrate"]);
+  assert.equal(services.migrate.restart, "no");
+  assert.equal(services.api.depends_on.migrate.condition, "service_completed_successfully");
+
+  for (const [name, service] of Object.entries(services)) {
+    for (const volume of service.volumes ?? []) {
+      assert.notEqual(volume.type, "bind", `${name} must not use a source bind mount`);
+    }
+  }
+});
+
+test("production images pin Node and run application processes as non-root", () => {
+  for (const fileName of ["apps/api/Dockerfile", "apps/web/Dockerfile"]) {
+    const source = read(fileName);
+    assert.match(source, /FROM node:24\.18\.0-bookworm-slim/);
+    assert.match(source, /^USER node$/m);
+    assert.doesNotMatch(source, /^CMD .*\b(?:pnpm|npm)\b/m);
+  }
+});
+
+test("compose requires secrets and keeps metrics behind the internal API network", () => {
+  const compose = read("docker-compose.yml");
+  const caddy = read("Caddyfile");
+
+  assert.match(compose, /POSTGRES_PASSWORD: \$\{POSTGRES_PASSWORD:\?/);
+  assert.match(compose, /SESSION_SECRET: \$\{SESSION_SECRET:\?/);
+  assert.match(compose, /\/health\/ready/);
+  assert.match(caddy, /@internalMetrics path \/api\/metrics/);
+  assert.match(caddy, /respond @internalMetrics 404/);
+});
+
+function read(fileName) {
+  return readFileSync(resolve(root, fileName), "utf8");
+}


## `fix(config): production에서 영속 저장소 요구`

diff --git a/apps/api/src/env.ts b/apps/api/src/env.ts
index 2308263..2d05fb6 100644
--- a/apps/api/src/env.ts
+++ b/apps/api/src/env.ts
@@ -16,9 +16,13 @@ export function readEnv(input = process.env): ApiEnv {
   ) {
     throw new Error("SESSION_SECRET must be at least 32 bytes in demo and production modes");
   }
+  const databaseUrl = input.DATABASE_URL ?? null;
+  if (appMode === "production" && !databaseUrl) {
+    throw new Error("DATABASE_URL is required in production mode");
+  }
   return {
     port: Number(input.API_PORT ?? 4000),
-    databaseUrl: input.DATABASE_URL ?? null,
+    databaseUrl,
     webOrigin: input.WEB_ORIGIN ?? "http://localhost:3000",
     sessionSecret: configuredSecret ?? "dev-session-secret",
     appMode,


## `fix(runtime): container 종료 유예를 room drain과 정렬`

diff --git a/docker-compose.yml b/docker-compose.yml
index 28f846c..8217134 100644
--- a/docker-compose.yml
+++ b/docker-compose.yml
@@ -48,6 +48,7 @@ services:
       retries: 12
     expose:
       - "4000"
+    stop_grace_period: 70s
 
   web:
     image: pong-pong-web:local


## `test(docker): API 종료 유예 계약 검증`

diff --git a/tests/docker-production.test.mjs b/tests/docker-production.test.mjs
index 5ca74ba..0356044 100644
--- a/tests/docker-production.test.mjs
+++ b/tests/docker-production.test.mjs
@@ -28,6 +28,7 @@ test("production compose exposes only Caddy and runs migration once", () => {
   assert.deepEqual(services.migrate.command, ["node", "packages/db/dist/cli.js", "migrate"]);
   assert.equal(services.migrate.restart, "no");
   assert.equal(services.api.depends_on.migrate.condition, "service_completed_successfully");
+  assert.ok(parseDurationSeconds(services.api.stop_grace_period) >= 60);
 
   for (const [name, service] of Object.entries(services)) {
     for (const volume of service.volumes ?? []) {
@@ -63,3 +64,10 @@ function read(fileName) {
 function escapeRegExp(value) {
   return value.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
 }
+
+function parseDurationSeconds(value) {
+  assert.equal(typeof value, "string");
+  const match = /^(?:(\d+)m)?(?:(\d+)s)?$/.exec(value);
+  assert.ok(match, `unsupported duration: ${value}`);
+  return Number(match[1] ?? 0) * 60 + Number(match[2] ?? 0);
+}
