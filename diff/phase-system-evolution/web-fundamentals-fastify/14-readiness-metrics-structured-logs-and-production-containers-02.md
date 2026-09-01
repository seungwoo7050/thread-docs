## `feat: adopt preserved E24 production operations`

diff --git a/.dockerignore b/.dockerignore
new file mode 100644
index 0000000..1ddf51e
--- /dev/null
+++ b/.dockerignore
@@ -0,0 +1,10 @@
+*
+!Dockerfile
+!package.json
+!package-lock.json
+!tsconfig.json
+!next.config.ts
+!app/
+!app/**
+!server/
+!server/**
diff --git a/.github/workflows/check.yml b/.github/workflows/check.yml
index 0ecd7ef..30c748f 100644
--- a/.github/workflows/check.yml
+++ b/.github/workflows/check.yml
@@ -10,7 +10,7 @@ permissions:
   contents: read
 
 jobs:
-  verify:
+  unit:
     runs-on: ubuntu-24.04
     timeout-minutes: 10
     env:
@@ -27,6 +27,17 @@ jobs:
       - run: npm run typecheck
       - name: Unit
         run: npm run test:unit
+  integration:
+    runs-on: ubuntu-24.04
+    timeout-minutes: 10
+    steps:
+      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
+      - uses: actions/setup-node@49933ea5288caeca8642d1e84afbd3f7d6820020 # v4.4.0
+        with:
+          node-version-file: .node-version
+          cache: npm
+      - run: npm install --global npm@11.17.0
+      - run: npm ci
       - name: Start isolated PostgreSQL
         run: npm run db:up
       - name: PostgreSQL migrations and persistence
@@ -35,16 +46,53 @@ jobs:
         run: npm run test:functional
       - name: Request identity and competing workers
         run: npm run test:ownership
-      - name: Worker crash recovery and shutdown
-        run: npm run test:recovery
+      # E11 crash and E20 plan cases retain explicit commands/frozen evidence;
+      # capped fault/plan scenarios are not repeated by every push.
       - name: Outbound safety and resource limits
         run: npm run test:outbound
+      - name: Worker lifecycle and scheduler
+        run: npm run test:execution
+      - name: Stop isolated PostgreSQL
+        if: always()
+        run: npm run db:down
+  browser:
+    runs-on: ubuntu-24.04
+    timeout-minutes: 10
+    env:
+      NEXT_TELEMETRY_DISABLED: '1'
+    steps:
+      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
+      - uses: actions/setup-node@49933ea5288caeca8642d1e84afbd3f7d6820020 # v4.4.0
+        with:
+          node-version-file: .node-version
+          cache: npm
+      - run: npm install --global npm@11.17.0
+      - run: npm ci
+      - run: npm run db:up
       - name: Install pinned Chromium
         run: npx playwright install --with-deps chromium
       - name: Production build and browser E2E
         run: npm run test:e2e
-      - name: Worker lifecycle and scheduler
-        run: npm run test:execution
+      - name: Stop isolated PostgreSQL
+        if: always()
+        run: npm run db:down
+  container:
+    runs-on: ubuntu-24.04
+    timeout-minutes: 20
+    steps:
+      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
+      - uses: actions/setup-node@49933ea5288caeca8642d1e84afbd3f7d6820020 # v4.4.0
+        with:
+          node-version-file: .node-version
+          cache: npm
+      - run: npm install --global npm@11.17.0
+      - run: npm ci
+      - run: npm run db:up
+      - run: npx playwright install --with-deps chromium
+      - name: Production images
+        run: docker compose -f compose.production.yaml build api frontend
+      - name: Production container smoke without the capped dependency stop
+        run: npm run test:container
       - name: Stop isolated PostgreSQL
         if: always()
         run: npm run db:down
diff --git a/Dockerfile b/Dockerfile
new file mode 100644
index 0000000..bca44e6
--- /dev/null
+++ b/Dockerfile
@@ -0,0 +1,33 @@
+FROM node:24.19.0-bookworm-slim@sha256:a9f5f7c91a432850b2a8a7797adf5eadb6c733ceed61167806cee7ea7fbc29df AS dependencies
+WORKDIR /app
+COPY package.json package-lock.json ./
+RUN npm ci --no-audit --no-fund --fetch-retries=0
+
+FROM dependencies AS web-build
+ENV NEXT_TELEMETRY_DISABLED=1
+ARG API_ORIGIN=http://api:4312
+ENV API_ORIGIN=$API_ORIGIN
+COPY next.config.ts tsconfig.json ./
+COPY app ./app
+COPY server ./server
+RUN npm run build
+
+FROM dependencies AS production-dependencies
+RUN npm prune --omit=dev --no-audit --no-fund
+
+FROM node:24.19.0-bookworm-slim@sha256:a9f5f7c91a432850b2a8a7797adf5eadb6c733ceed61167806cee7ea7fbc29df AS backend
+WORKDIR /app
+ENV NODE_ENV=production
+COPY --from=production-dependencies --chown=node:node /app/node_modules ./node_modules
+COPY --chown=node:node package.json ./
+COPY --chown=node:node server ./server
+USER node
+CMD ["node", "server/main.ts"]
+
+FROM node:24.19.0-bookworm-slim@sha256:a9f5f7c91a432850b2a8a7797adf5eadb6c733ceed61167806cee7ea7fbc29df AS frontend
+WORKDIR /app
+ENV NODE_ENV=production NEXT_TELEMETRY_DISABLED=1 HOSTNAME=0.0.0.0 PORT=4313 API_ORIGIN=http://api:4312
+COPY --from=web-build --chown=node:node /app/.next/standalone ./
+COPY --from=web-build --chown=node:node /app/.next/static ./.next/static
+USER node
+CMD ["node", "server.js"]
diff --git a/compose.production.yaml b/compose.production.yaml
new file mode 100644
index 0000000..3b3f2a0
--- /dev/null
+++ b/compose.production.yaml
@@ -0,0 +1,43 @@
+name: wse-fundamentals-e24
+services:
+  api:
+    image: wse-fundamentals-e24-backend:local
+    build:
+      context: .
+      target: backend
+    environment:
+      API_HOST: 0.0.0.0
+      DATABASE_URL: postgresql://monitor@postgres:5432/monitor
+      DATABASE_SCHEMA: ${DATABASE_SCHEMA:-public}
+    ports:
+      - "127.0.0.1:4312:4312"
+    networks: [default, postgres]
+    stop_grace_period: 5s
+  worker:
+    image: wse-fundamentals-e24-backend:local
+    command: ["node", "server/worker.ts"]
+    environment:
+      WORKER_HOST: 0.0.0.0
+      WORKER_HTTP_PORT: '4314'
+      DATABASE_URL: postgresql://monitor@postgres:5432/monitor
+      DATABASE_SCHEMA: ${DATABASE_SCHEMA:-public}
+    ports:
+      - "127.0.0.1:4314:4314"
+    networks: [default, postgres]
+    stop_grace_period: 5s
+  frontend:
+    image: wse-fundamentals-e24-frontend:local
+    build:
+      context: .
+      target: frontend
+      args:
+        API_ORIGIN: http://api:4312
+    environment:
+      API_ORIGIN: http://api:4312
+    ports:
+      - "127.0.0.1:4313:4313"
+    stop_grace_period: 30s
+networks:
+  postgres:
+    external: true
+    name: wse-fundamentals_default
diff --git a/evidence/phase-1/E24/compose.test.yaml b/evidence/phase-1/E24/compose.test.yaml
new file mode 100644
index 0000000..5b92de5
--- /dev/null
+++ b/evidence/phase-1/E24/compose.test.yaml
@@ -0,0 +1,31 @@
+# Test transport only. Production compose has no fixture or private-URL trust.
+services:
+  api:
+    environment:
+      NODE_ENV: test
+      WSE_TEST_FIXTURE_ORIGIN: http://127.0.0.1:4311
+  worker:
+    environment:
+      NODE_ENV: test
+      WSE_TEST_FIXTURE_ORIGIN: http://127.0.0.1:4311
+    networks: !reset []
+    ports: !reset []
+    network_mode: service:fixture
+    depends_on:
+      fixture:
+        condition: service_healthy
+  fixture:
+    image: wse-fundamentals-e24-backend:local
+    command: ["node", "evidence/phase-1/E24/fixture-server.mjs"]
+    volumes:
+      - ./test/fixture.ts:/app/test/fixture.ts:ro
+      - ./evidence/phase-1/E24/fixture-server.mjs:/app/evidence/phase-1/E24/fixture-server.mjs:ro
+    ports:
+      - "127.0.0.1:4311:4311"
+      - "127.0.0.1:4314:4314"
+    networks: [default, postgres]
+    healthcheck:
+      test: ["CMD", "node", "-e", "fetch('http://127.0.0.1:4311/e24/counts').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"]
+      interval: 1s
+      timeout: 2s
+      retries: 10
diff --git a/evidence/phase-1/E24/fixture-server.mjs b/evidence/phase-1/E24/fixture-server.mjs
new file mode 100644
index 0000000..a876a69
--- /dev/null
+++ b/evidence/phase-1/E24/fixture-server.mjs
@@ -0,0 +1,29 @@
+import { fixtureServer } from '../../../test/fixture.ts';
+
+const { server, calls } = fixtureServer();
+const handler = server.listeners('request')[0];
+server.removeAllListeners('request');
+let body = '';
+let bodyWrites = 0;
+server.on('request', async (request, response) => {
+  const path = new URL(request.url ?? '/', 'http://fixture.invalid').pathname;
+  if (path === '/e24/counts') {
+    response.setHeader('content-type', 'application/json');
+    response.end(JSON.stringify({ fail: calls.get('/fail') ?? 0, ok: calls.get('/ok') ?? 0, bodyWrites })); return;
+  }
+  if (path === '/e24/control' && request.method === 'POST') {
+    let input = '';
+    for await (const chunk of request) input += chunk;
+    const value = JSON.parse(input);
+    if (typeof value.body !== 'string' || value.body.length > 256) { response.writeHead(400); response.end(); return; }
+    body = value.body;
+    response.end('configured'); return;
+  }
+  request.url = path;
+  const end = response.end.bind(response);
+  response.end = () => { bodyWrites++; return end(body); };
+  const timer = setTimeout(() => handler.call(server, request, response), 250);
+  response.once('close', () => clearTimeout(timer));
+});
+server.listen(4311, '0.0.0.0');
+process.once('SIGTERM', () => { server.closeAllConnections(); server.close(() => process.exit(0)); });
diff --git a/next.config.ts b/next.config.ts
index b8e9929..f3f298c 100644
--- a/next.config.ts
+++ b/next.config.ts
@@ -1,6 +1,7 @@
 import type { NextConfig } from 'next';
 
 const config: NextConfig = {
+  output: 'standalone',
   turbopack: { root: process.cwd() },
   async rewrites() {
     return [{ source: '/api/:path*', destination: `${process.env.API_ORIGIN ?? 'http://127.0.0.1:4312'}/:path*` }];
diff --git a/package.json b/package.json
index ee56fe5..488def5 100644
--- a/package.json
+++ b/package.json
@@ -25,7 +25,8 @@
     "test:execution": "node --test test/execution.test.ts",
     "test:ownership": "node --test test/ownership.test.ts",
     "test:recovery": "node --test test/recovery.test.ts",
-    "test:outbound": "node --test test/outbound.test.ts"
+    "test:outbound": "node --test test/outbound.test.ts",
+    "test:container": "node test/container-smoke.mjs"
   },
   "dependencies": {
     "fastify": "5.12.1",
diff --git a/server/app.ts b/server/app.ts
index e331e1d..cada2a5 100644
--- a/server/app.ts
+++ b/server/app.ts
@@ -10,6 +10,7 @@ import { checkRunFromRow, monitorFromRow, monitorToValues, monitorViewFromRow }
 import type { CheckRunRow, MonitorRow, MonitorViewRow } from './mapping.ts';
 import { registerAuthentication } from './auth.ts';
 import { historyCursor, historyQuery } from './history.ts';
+import { Operations } from './operations.ts';
 
 const inputErrors = [
   errorCodes.FST_ERR_CTP_INVALID_JSON_BODY,
@@ -28,7 +29,8 @@ const monitorViewSql = `SELECT m.id, m.owner_user_id, m.name, m.url, m.interval_
       ORDER BY (finished_at IS NULL) DESC, COALESCE(finished_at, queued_at) DESC, id DESC LIMIT 1
   ) c ON true`;
 
-export function buildApp(testFixtureOrigin: string | undefined = undefined, database: DatabaseConfig = databaseConfig(), now: () => number = Date.now) {
+export function buildApp(testFixtureOrigin: string | undefined = undefined, database: DatabaseConfig = databaseConfig(), now: () => number = Date.now,
+  operations = new Operations('api')) {
   const handleError = (error: unknown, _request: FastifyRequest, reply: FastifyReply) => {
     const failure = error instanceof ApiError ? error
       : inputErrors.some((ErrorType) => error instanceof ErrorType)
@@ -37,8 +39,11 @@ export function buildApp(testFixtureOrigin: string | undefined = undefined, data
     const body: ApiFailure = { error: { code: failure.code, message: failure.message } };
     return reply.code(ERROR_STATUS[failure.code]).type('application/json').send(body);
   };
-  const app = Fastify({ logger: false, bodyLimit: 8_192, frameworkErrors: handleError });
-  const pool = databasePool(database);
+  const app = Fastify({ logger: false, bodyLimit: 8_192, frameworkErrors: handleError, requestIdHeader: false, genReqId: () => randomUUID() });
+  const pool = databasePool(database, () => operations.log('database_connection_lost'));
+  app.addHook('onResponse', async (request, reply) => {
+    operations.recordHttp(request.routeOptions.url, request.method, reply.statusCode, reply.elapsedTime, request.id);
+  });
   let poolClosed = false;
   async function closePool() {
     if (!poolClosed) { poolClosed = true; await pool.end(); }
@@ -54,6 +59,12 @@ export function buildApp(testFixtureOrigin: string | undefined = undefined, data
   registerAuthentication(app, pool, now);
 
   app.get('/health', async () => ({ data: { status: 'ok' } }));
+  app.get('/health/live', async () => ({ data: { status: 'ok' } }));
+  app.get('/health/ready', async (_request, reply) => {
+    const ready = await operations.ready(pool);
+    return reply.code(ready ? 200 : 503).send({ data: { status: ready ? 'ok' : 'unavailable' } });
+  });
+  app.get('/metrics', async (_request, reply) => reply.type('text/plain; version=0.0.4').send(await operations.metrics(pool)));
   app.get('/monitors', async (request) => {
     const result = await pool.query<MonitorViewRow>(`${monitorViewSql} WHERE m.owner_user_id = $1 ORDER BY m.created_at, m.id`, [request.user!.id]);
     return { data: result.rows.map(monitorViewFromRow) };
@@ -118,6 +129,7 @@ export function buildApp(testFixtureOrigin: string | undefined = undefined, data
         'SELECT * FROM check_runs WHERE request_user_id = $1 AND idempotency_key = $2', [userId, key])).rows[0];
       if (saved.monitor_id !== id) throw new ApiError('CONFLICT', 'Idempotency-Key was already used for another Monitor.');
       await client.query('COMMIT');
+      operations.log('check_accepted', { requestId: request.id, checkId: saved.id });
       return reply.code(202).send({ data: checkRunFromRow(saved) });
     } catch (error) { await client.query('ROLLBACK'); throw error; }
     finally { client.release(); }
diff --git a/server/auth.ts b/server/auth.ts
index c3e3a4a..f9922cf 100644
--- a/server/auth.ts
+++ b/server/auth.ts
@@ -59,7 +59,8 @@ export function registerAuthentication(app: FastifyInstance, pool: Pool, now: ()
     // This only controls browser response access; mutations still check CSRF.
     if (request.method === 'OPTIONS') throw forbidden();
     const route = request.routeOptions.url;
-    if (route === '/health' && (request.method === 'GET' || request.method === 'HEAD')) return;
+    if (['/health', '/health/live', '/health/ready', '/metrics'].includes(route ?? '') &&
+      (request.method === 'GET' || request.method === 'HEAD')) return;
     if (route === '/auth/login' && request.method === 'POST') {
       // Login has no authenticated session yet: reject missing/null/foreign
       // Origin before credentials or session replacement are processed.
diff --git a/server/database.ts b/server/database.ts
index e3c97d0..b665742 100644
--- a/server/database.ts
+++ b/server/database.ts
@@ -14,11 +14,15 @@ export function schemaIdentifier(schema: string): string {
   return `"${schema}"`;
 }
 
-export function databasePool(config: DatabaseConfig): Pool {
+export function databasePool(config: DatabaseConfig, onIdleError: () => void = () => {}): Pool {
   schemaIdentifier(config.schema);
-  return new Pool({
+  const pool = new Pool({
     connectionString: config.connectionString,
     options: `-c search_path=${config.schema} -c timezone=UTC`,
     connectionTimeoutMillis: 2_000,
   });
+  // A lost idle connection must not kill a live process. Awaited queries still
+  // fail normally; readiness and each transaction remain authoritative.
+  pool.on('error', onIdleError);
+  return pool;
 }
diff --git a/server/main.ts b/server/main.ts
index e9b5e9a..b4b81d2 100644
--- a/server/main.ts
+++ b/server/main.ts
@@ -1,9 +1,17 @@
 import { buildApp } from './app.ts';
 import { configuredTestFixtureOrigin } from './outbound.ts';
+import { Operations } from './operations.ts';
 
 const app = buildApp(configuredTestFixtureOrigin());
-await app.listen({ host: '127.0.0.1', port: Number(process.env.API_PORT ?? 4312) });
-console.log('Monitor API listening on loopback.');
+const operations = new Operations('api');
+await app.listen({ host: process.env.API_HOST ?? '127.0.0.1', port: Number(process.env.API_PORT ?? 4312) });
+operations.log('api_ready');
 for (const signal of ['SIGINT', 'SIGTERM'] as const) {
-  process.once(signal, async () => { await app.close(); process.exit(0); });
+  process.once(signal, async () => {
+    operations.log('api_stopping');
+    const deadline = setTimeout(() => { operations.log('api_shutdown_deadline'); process.exit(1); }, 3000);
+    await app.close();
+    clearTimeout(deadline);
+    process.exit(0);
+  });
 }
diff --git a/server/operations.ts b/server/operations.ts
new file mode 100644
index 0000000..e3c939e
--- /dev/null
+++ b/server/operations.ts
@@ -0,0 +1,97 @@
+import { randomUUID } from 'node:crypto';
+import { createServer } from 'node:http';
+import type { Pool } from 'pg';
+
+export const PROCESS_ID = randomUUID();
+const routes = new Set(['/health', '/health/live', '/health/ready', '/metrics', '/auth/login',
+  '/auth/session', '/auth/csrf', '/auth/logout', '/monitors', '/monitors/:id', '/monitors/:id/checks', '/checks/:id']);
+const methods = new Set(['GET', 'HEAD', 'POST', 'PUT', 'DELETE', 'PATCH', 'OPTIONS', 'CONNECT', 'TRACE']);
+type Fields = { requestId?: string; checkId?: string; workerId?: string; count?: number; message?: string };
+type HttpSample = { route: string; method: string; status: number; count: number; seconds: number; errors: number };
+
+// Explicit fields only: never pass a request, error, URL, headers or body here.
+export class Operations {
+  readonly role: 'api' | 'worker';
+  readonly write: (line: string) => void;
+  readonly http = new Map<string, HttpSample>();
+  active = 0;
+  claims = 0;
+  recoveryRuns = 0;
+  recovered = 0;
+
+  constructor(role: 'api' | 'worker', write = (line: string) => { process.stdout.write(line + '\n'); }) {
+    this.role = role;
+    this.write = write;
+  }
+
+  log(event: string, fields: Fields = {}) {
+    this.write(JSON.stringify({ time: new Date().toISOString(), role: this.role, processId: PROCESS_ID,
+      pid: process.pid, event, ...fields }));
+  }
+
+  recordHttp(route: string | undefined, method: string, status: number, milliseconds: number, requestId: string) {
+    const safeRoute = route && routes.has(route) ? route : 'unmatched';
+    const safeMethod = methods.has(method) ? method : 'OTHER';
+    const safeStatus = Number.isInteger(status) && status >= 100 && status <= 599 ? status : 500;
+    const key = JSON.stringify([safeRoute, safeMethod, safeStatus]);
+    const sample = this.http.get(key) ?? { route: safeRoute, method: safeMethod, status: safeStatus, count: 0, seconds: 0, errors: 0 };
+    sample.count++;
+    sample.seconds += Math.max(0, milliseconds) / 1000;
+    if (safeStatus >= 400) sample.errors++;
+    this.http.set(key, sample);
+    this.write(JSON.stringify({ time: new Date().toISOString(), role: this.role, processId: PROCESS_ID,
+      pid: process.pid, event: 'http_request', requestId, route: safeRoute, method: safeMethod,
+      status: safeStatus, durationMs: Math.max(0, milliseconds) }));
+  }
+
+  async ready(pool: Pool) {
+    try { await pool.query('SELECT 1'); return true; }
+    catch { return false; }
+  }
+
+  async metrics(pool: Pool) {
+    let age: number | undefined;
+    try {
+      const result = await pool.query<{ seconds: string }>(`SELECT COALESCE(
+        GREATEST(0, extract(epoch FROM clock_timestamp() - min(queued_at))), 0)::text AS seconds
+        FROM check_runs WHERE state = 'QUEUED'`);
+      age = Number(result.rows[0].seconds);
+    } catch { /* Unknown queue age is absent, never a fabricated zero. */ }
+    const lines: string[] = [];
+    for (const sample of this.http.values()) {
+      const labels = `role="${this.role}",route="${sample.route}",method="${sample.method}",status="${sample.status}"`;
+      lines.push(`http_request_duration_seconds_sum{${labels}} ${sample.seconds}`,
+        `http_request_duration_seconds_count{${labels}} ${sample.count}`, `http_errors_total{${labels}} ${sample.errors}`);
+    }
+    lines.push(`postgres_ready{role="${this.role}"} ${age === undefined ? 0 : 1}`);
+    if (age !== undefined) lines.push(`check_queue_age_seconds{role="${this.role}"} ${age}`);
+    if (this.role === 'worker') lines.push(`worker_active{role="worker"} ${this.active}`,
+      `worker_claims_total{role="worker"} ${this.claims}`, `worker_recovery_runs_total{role="worker"} ${this.recoveryRuns}`,
+      `worker_recovered_checks_total{role="worker"} ${this.recovered}`);
+    return lines.join('\n') + '\n';
+  }
+}
+
+export async function workerHealth(operations: Operations, pool: Pool, host: string, port: number) {
+  const server = createServer(async (request, response) => {
+    const began = performance.now();
+    const requestId = randomUUID();
+    const path = request.url?.split('?')[0];
+    const route = ['/health/live', '/health/ready', '/metrics'].includes(path ?? '') ? path : undefined;
+    response.once('finish', () => operations.recordHttp(route, request.method ?? 'OTHER', response.statusCode,
+      performance.now() - began, requestId));
+    response.setHeader('cache-control', 'no-store');
+    try {
+      if (!['GET', 'HEAD'].includes(request.method ?? '') || !route) { response.writeHead(404); response.end(); return; }
+      if (route === '/metrics') {
+        response.setHeader('content-type', 'text/plain; version=0.0.4');
+        response.end(await operations.metrics(pool)); return;
+      }
+      const ready = route === '/health/live' || await operations.ready(pool);
+      response.writeHead(ready ? 200 : 503, { 'content-type': 'application/json' });
+      response.end(JSON.stringify({ data: { status: ready ? 'ok' : 'unavailable' } }));
+    } catch { response.writeHead(500); response.end(); }
+  });
+  await new Promise<void>((resolve, reject) => { server.once('error', reject); server.listen(port, host, resolve); });
+  return server;
+}
diff --git a/server/worker.ts b/server/worker.ts
index fb15d9c..a0d6791 100644
--- a/server/worker.ts
+++ b/server/worker.ts
@@ -9,9 +9,11 @@ import { checkRunFromRow, monitorFromRow } from './mapping.ts';
 import type { TerminalCheckRun } from './model.ts';
 import type { CheckRunRow, MonitorRow } from './mapping.ts';
 import { verifySchema } from './schema.ts';
+import { Operations, workerHealth } from './operations.ts';
 
 export const CHECK_LEASE_MS = 5000;
 export const SHUTDOWN_GRACE_MS = 3000;
+const operations = new Operations('worker');
 
 export async function scheduleDueChecks(pool: Pool, now = new Date()) {
   // The current interval slot is anchored to creation. Repeated ticks address
@@ -32,6 +34,9 @@ export async function recoverExpiredChecks(pool: Pool, now?: Date) {
     finished_at = COALESCE($1::timestamptz, clock_timestamp())
     WHERE state = 'RUNNING' AND lease_expires_at <= COALESCE($1::timestamptz, clock_timestamp())
     RETURNING *`, [now ?? null]);
+  operations.recoveryRuns++;
+  operations.recovered += recovered.rowCount ?? 0;
+  if (recovered.rowCount) operations.log('checks_recovered', { count: recovered.rowCount });
   return recovered.rows.map(checkRunFromRow);
 }
 
@@ -77,48 +82,65 @@ export async function runNextCheck(pool: Pool, testFixtureOrigin?: string, worke
   finally { client.release(); }
   if (!claimed) return null;
   // Commit ownership before outbound I/O; a competing worker skips this row.
-  const monitor = await pool.query<MonitorRow>('SELECT * FROM monitors WHERE id = $1', [claimed.monitor_id]);
-  if (!monitor.rows[0]) return null;
-  const execution = checkRunFromRow(claimed);
-  const observed = await checkMonitor(monitorFromRow(monitor.rows[0]), testFixtureOrigin, {
-    id: execution.id, trigger: execution.trigger, startedAt: execution.startedAt!,
-  });
-  return completeCheck(pool, workerId, observed);
+  operations.claims++;
+  operations.active++;
+  operations.log('check_claimed', { checkId: claimed.id, workerId });
+  try {
+    const monitor = await pool.query<MonitorRow>('SELECT * FROM monitors WHERE id = $1', [claimed.monitor_id]);
+    if (!monitor.rows[0]) return null;
+    const execution = checkRunFromRow(claimed);
+    const observed = await checkMonitor(monitorFromRow(monitor.rows[0]), testFixtureOrigin, {
+      id: execution.id, trigger: execution.trigger, startedAt: execution.startedAt!,
+    });
+    const completed = await completeCheck(pool, workerId, observed);
+    if (completed) operations.log('check_completed', { checkId: completed.id, workerId });
+    return completed;
+  } finally { operations.active--; }
 }
 
 if (process.argv[1] && import.meta.url === pathToFileURL(process.argv[1]).href) {
-  const pool = databasePool(databaseConfig());
+  const pool = databasePool(databaseConfig(), () => operations.log('database_connection_lost'));
   const workerId = randomUUID();
   const stopping = new AbortController();
   let shutdownDeadline: ReturnType<typeof setTimeout> | undefined;
   function stopClaims() {
     if (stopping.signal.aborted) return;
     stopping.abort();
-    console.log('Check worker stopping.');
+    operations.log('worker_stopping', { workerId, message: 'Check worker stopping.' });
     shutdownDeadline = setTimeout(() => {
       // A stuck operation is uncertain, not an endpoint failure. Its lease expires.
-      console.error('Check worker shutdown deadline reached.');
+      operations.log('worker_shutdown_deadline', { workerId });
       process.exit(1);
     }, SHUTDOWN_GRACE_MS);
   }
   process.on('SIGTERM', stopClaims);
+  let health: Awaited<ReturnType<typeof workerHealth>> | undefined;
   try {
     await verifySchema(pool);
-    console.log(`Check worker ready. ${workerId}`);
+    health = await workerHealth(operations, pool, process.env.WORKER_HOST ?? '127.0.0.1', Number(process.env.WORKER_HTTP_PORT ?? 4314));
+    operations.log('worker_ready', { workerId, message: `Check worker ready. ${workerId}` });
     while (!stopping.signal.aborted) {
       // Regression fixtures control scheduler time explicitly; normal operation
       // always schedules. There is no separate scheduler daemon or queue store.
-      if (!process.argv.includes('--no-schedule')) await scheduleDueChecks(pool);
-      if (stopping.signal.aborted) break;
-      const completed = await runNextCheck(pool, configuredTestFixtureOrigin(), workerId, stopping.signal);
-      if (stopping.signal.aborted) break;
-      await recoverExpiredChecks(pool);
+      let completed: Awaited<ReturnType<typeof runNextCheck>> = null;
+      try {
+        if (!process.argv.includes('--no-schedule')) await scheduleDueChecks(pool);
+        if (stopping.signal.aborted) break;
+        completed = await runNextCheck(pool, configuredTestFixtureOrigin(), workerId, stopping.signal);
+        if (stopping.signal.aborted) break;
+        await recoverExpiredChecks(pool);
+      } catch {
+        // Dependency polling does not requeue or retry a claimed HTTP check.
+        // Any uncertain RUNNING row retains its finite E11 lease for recovery.
+        operations.log('worker_iteration_failed', { workerId });
+      }
       if (!completed && !stopping.signal.aborted) await delay(250);
     }
   } catch {
-    console.error('Check worker stopped after an execution or database failure.');
+    operations.log('worker_start_failed', { workerId });
     process.exitCode = 1;
   } finally {
+    if (health) await new Promise<void>((resolve, reject) => health!.close(error => error ? reject(error) : resolve()));
     await pool.end();
     clearTimeout(shutdownDeadline);
     process.off('SIGTERM', stopClaims);
diff --git a/test/container-smoke.mjs b/test/container-smoke.mjs
new file mode 100644
index 0000000..e0314fd
--- /dev/null
+++ b/test/container-smoke.mjs
@@ -0,0 +1,333 @@
+import { execFile } from 'node:child_process';
+import { createHash } from 'node:crypto';
+import { promisify } from 'node:util';
+import { mkdir, readFile, writeFile } from 'node:fs/promises';
+import { setTimeout as delay } from 'node:timers/promises';
+import { chromium } from '@playwright/test';
+import { databaseConfig, databasePool, schemaIdentifier } from '../server/database.ts';
+import { migrate } from '../server/migrate.ts';
+import { cookieFromHeader, authenticatedFetch } from './auth.ts';
+import { requireFreePorts } from '../evidence/phase-1/E10/fixture.ts';
+import { insertOperationsFixture } from '../evidence/phase-1/E24/fixture.ts';
+import scenario from '../evidence/phase-1/E24/scenario.json' with { type: 'json' };
+
+const mode = process.argv[2] ?? 'smoke';
+const actor = process.argv[3] ?? 'ci';
+class SmokeFailure extends Error {}
+function check(value, message) { if (!value) throw new SmokeFailure(message); }
+check(['smoke', 'full'].includes(mode) && ['ci', 'author', 'root'].includes(actor), 'Explicit smoke/full and ci/author/root arguments required.');
+check(process.versions.node === scenario.runtime.node, 'Pinned host Node is required.');
+const output = 'output/phase-1/e24';
+await mkdir(output, { recursive: true });
+await writeFile(`${output}/${mode}-${actor}.started.json`, JSON.stringify({ mode, actor, at: new Date().toISOString() }) + '\n', { flag: 'wx' });
+const execute = promisify(execFile);
+const report = { mode, actor, result: 'NOT_RUN', commands: [], hashes: {}, postgresStops: 0, postgresRestores: 0, conditionPolls: 0 };
+for (const file of ['scenario.json', 'fixture.ts', 'baseline.mjs', 'compose.test.yaml', 'fixture-server.mjs']) {
+  report.hashes[file] = createHash('sha256').update(await readFile(`evidence/phase-1/E24/${file}`)).digest('hex');
+}
+report.hashes.observer = createHash('sha256').update(await readFile(new URL(import.meta.url))).digest('hex');
+const config = { ...databaseConfig(), schema: scenario.ownership.containerSchema };
+const databaseUrl = new URL(config.connectionString);
+check(databaseUrl.hostname === '127.0.0.1' && databaseUrl.port === '15431', 'Only the owned loopback PostgreSQL is permitted.');
+const pool = databasePool(config);
+const compose = ['compose', '--project-name', scenario.ownership.runtimeProject, '-f', 'compose.production.yaml', '-f', 'evidence/phase-1/E24/compose.test.yaml'];
+const pgCompose = ['compose', '--project-name', scenario.ownership.postgresProject, '-f', 'compose.yaml'];
+const api = 'http://127.0.0.1:4312';
+const frontend = 'http://127.0.0.1:4313';
+const worker = 'http://127.0.0.1:4314';
+const began = performance.now();
+let stage = 'preflight';
+let schemaOwned = false;
+let runtimeOwned = false;
+let needsRestore = false;
+let restoreAttempted = false;
+let browser;
+let fixture;
+async function docker(args) {
+  const started = performance.now();
+  try {
+    const result = await execute('docker', args, { env: { ...process.env, DATABASE_SCHEMA: config.schema },
+      timeout: scenario.runtime.observerExitDeadlineMs, maxBuffer: 8 * 1024 * 1024 });
+    report.commands.push({ args, exitCode: 0, durationMs: Math.round(performance.now() - started) });
+    return (args[0] === 'logs' ? result.stdout + '\n' + result.stderr : result.stdout).trim();
+  } catch (error) {
+    report.commands.push({ args, exitCode: error.code ?? null, durationMs: Math.round(performance.now() - started),
+      stdoutBytes: Buffer.byteLength(error.stdout ?? ''), stderrBytes: Buffer.byteLength(error.stderr ?? '') });
+    throw new SmokeFailure(`Docker ${args[0]} failed; raw output withheld from artifacts.`);
+  }
+}
+async function fetchAt(origin, path, options = {}) {
+  return fetch(new URL(path, origin), { ...options, signal: AbortSignal.timeout(10000) });
+}
+async function until(read, message) {
+  const deadline = Date.now() + 10000;
+  while (Date.now() < deadline) {
+    report.conditionPolls++;
+    const result = await read().catch(() => null);
+    if (result) return result;
+    await delay(25);
+  }
+  throw new SmokeFailure(message);
+}
+function samples(text) {
+  return text.trim().split('\n').filter(Boolean).map(line => {
+    const match = /^([a-z_]+)\{([^}]*)\} (.+)$/.exec(line);
+    check(match, 'Metrics must use the fixed scalar text format.');
+    return { name: match[1], labels: Object.fromEntries([...match[2].matchAll(/([a-z]+)="([^"]*)"/g)].map(m => [m[1], m[2]])), value: Number(match[3]), series: line.slice(0, line.lastIndexOf(' ')) };
+  });
+}
+function metric(text, name, labels = {}) {
+  return samples(text).filter(row => row.name === name && Object.entries(labels).every(([key, value]) => row.labels[key] === value))
+    .reduce((sum, row) => sum + row.value, 0);
+}
+async function metrics(origin) {
+  const response = await fetchAt(origin, scenario.paths.metrics);
+  check(response.status === 200, 'Operational metrics must remain reachable.');
+  return response.text();
+}
+async function roleHealth(expectedReady) {
+  const observed = [];
+  for (const origin of [api, worker]) {
+    const live = await fetchAt(origin, scenario.paths.live);
+    const ready = await fetchAt(origin, scenario.paths.ready);
+    check(live.status === 200 && ready.status === expectedReady, 'Role liveness/readiness status disagrees with PostgreSQL availability.');
+    observed.push({ live: live.status, ready: ready.status });
+  }
+  return observed;
+}
+async function restorePostgres() {
+  restoreAttempted = true;
+  report.postgresRestores++;
+  await docker([...pgCompose, 'start', '--wait', '--wait-timeout', '30', scenario.ownership.postgresService]);
+  needsRestore = false;
+}
+async function identities() {
+  const result = {};
+  for (const role of scenario.roles) {
+    const id = await docker([...compose, 'ps', '-q', role]);
+    check(/^[0-9a-f]+$/.test(id), 'Exactly one owned container must exist for each role.');
+    const state = JSON.parse(await docker(['inspect', '--format', '{"pid":{{.State.Pid}},"running":{{.State.Running}},"restarts":{{.RestartCount}},"image":{{json .Image}},"command":{{json .Config.Cmd}}}', id]));
+    result[role] = { id, ...state };
+  }
+  return result;
+}
+try {
+  await requireFreePorts();
+  check(await docker(['ps', '-aq', '--filter', `label=com.docker.compose.project=${scenario.ownership.runtimeProject}`]) === '', 'Runtime project is already occupied.');
+  check(await docker(['network', 'ls', '-q', '--filter', `label=com.docker.compose.project=${scenario.ownership.runtimeProject}`]) === '', 'Runtime network is already occupied.');
+  const namespaces = (await pool.query("SELECT nspname FROM pg_namespace WHERE nspname NOT LIKE 'pg_%' AND nspname <> 'information_schema' ORDER BY nspname")).rows;
+  check(namespaces.length === 1 && namespaces[0].nspname === 'public', 'PostgreSQL must be idle and exclusively available to this fixture.');
+  await pool.query(`CREATE SCHEMA ${schemaIdentifier(config.schema)}`); schemaOwned = true;
+  await migrate(config);
+  fixture = await insertOperationsFixture(pool);
+  stage = 'container startup';
+  runtimeOwned = true;
+  await docker([...compose, 'up', '-d', '--no-build', 'api', 'frontend', 'fixture']);
+  await until(async () => (await fetchAt(api, scenario.paths.ready)).status === 200, 'API did not become ready.');
+  await until(async () => (await fetchAt(frontend, '/login')).status === 200, 'Production frontend did not start.');
+  await until(async () => (await fetchAt(scenario.fixture.origin, scenario.fixture.counts)).status === 200, 'Owned fixture did not start.');
+  check((await fetchAt(scenario.fixture.origin, scenario.fixture.control, { method: 'POST', body: JSON.stringify({ body: fixture.body }) })).status === 200,
+    'Private fixture body must be configured before worker startup.');
+  const cookies = [];
+  for (const credentials of fixture.users) {
+    const response = await fetchAt(api, '/auth/login', { method: 'POST', headers: { origin: frontend, 'content-type': 'application/json' }, body: JSON.stringify(credentials) });
+    check(response.status === 200, 'Runtime account login failed.');
+    const cookie = cookieFromHeader(response.headers.get('set-cookie') ?? undefined);
+    cookies.push(cookie);
+    fixture.sentinels.push(cookie, cookie.split('=')[1]);
+  }
+  const alice = authenticatedFetch(cookies[0]);
+  const bob = authenticatedFetch(cookies[1]);
+  const csrfResponse = await fetchAt(api, '/auth/csrf', { headers: { cookie: cookies[0] } });
+  check(csrfResponse.status === 200, 'Runtime CSRF preparation failed.');
+  const csrf = (await csrfResponse.json()).data.csrfToken;
+  fixture.sentinels.push(csrf);
+  const manualHeaders = { cookie: cookies[0], origin: frontend, 'x-csrf-token': csrf, 'idempotency-key': fixture.manualKey };
+  stage = 'queue and actual worker';
+  const accepted = await fetchAt(api, `/monitors/${fixture.monitors[0].id}/checks`, { method: 'POST', headers: manualHeaders });
+  check(accepted.status === 202, 'The sole manual intent must be accepted with202.');
+  const queued = (await accepted.json()).data;
+  check(queued.state === 'QUEUED' && queued.httpStatus === null, 'Accepted work must initially be QUEUED with unknown HTTP status.');
+  await delay(100);
+  const queuedMetrics = await metrics(api);
+  check(metric(queuedMetrics, 'check_queue_age_seconds') > 0, 'Queue age must reflect the actual waiting intent.');
+  const startup = await Promise.allSettled([
+    docker([...compose, 'up', '-d', '--no-build', 'worker']),
+    until(async () => { const text = await metrics(worker); return metric(text, 'worker_active') === 1 ? text : null; }, 'Active worker gauge was not observed during the fixed header delay.'),
+  ]);
+  check(startup.every(result => result.status === 'fulfilled'), 'Worker startup or active-gauge observation failed.');
+  const terminal = await until(async () => {
+    const response = await alice(`${api}/checks/${queued.id}`);
+    if (response.status !== 200) return null;
+    const checkRun = (await response.json()).data;
+    return ['FAILED', 'SUCCEEDED', 'ABORTED'].includes(checkRun.state) ? checkRun : null;
+  }, 'The real worker did not finish the accepted intent.');
+  check(terminal.id === queued.id && terminal.state === 'FAILED' && terminal.httpStatus === 503, 'Same-ID worker result must be the observed fixture503.');
+  const idleWorkerMetrics = await until(async () => {
+    const text = await metrics(worker);
+    return metric(text, 'worker_active') === 0 && metric(text, 'worker_recovery_runs_total') > 0 ? text : null;
+  }, 'Worker did not return idle with actual recovery-operation observations.');
+  check(metric(idleWorkerMetrics, 'worker_claims_total') === 1 && metric(idleWorkerMetrics, 'worker_recovered_checks_total') === 0,
+    'Healthy smoke must contain exactly one claim and no recovered checks.');
+  report.healthUp = await roleHealth(200);
+  report.worker = { accepted202: true, queuedUnknownStatus: true, positiveQueueAge: true, activeObserved: true, sameIdTerminal503: true,
+    claims: 1, recovered: 0, recoveryScans: metric(idleWorkerMetrics, 'worker_recovery_runs_total') };
+  stage = 'container browser and owner boundary';
+  for (const [request, own] of [[alice, fixture.monitors[0]], [bob, fixture.monitors[1]]]) {
+    const response = await request(`${api}/monitors`);
+    check(response.status === 200, 'Owned collection read failed.');
+    const data = (await response.json()).data;
+    check(data.length === 1 && data[0].id === own.id, 'Collection must contain only its session owner monitor.');
+  }
+  check((await bob(`${api}/monitors/${fixture.monitors[0].id}/checks`)).status === 404, 'Foreign history must remain hidden.');
+  browser = await chromium.launch({ headless: true });
+  const context = await browser.newContext();
+  const page = await context.newPage();
+  page.setDefaultTimeout(10000);
+  const browserErrors = [];
+  page.on('pageerror', error => browserErrors.push(error.message));
+  page.on('console', message => { if (message.type() === 'error') browserErrors.push(message.text()); });
+  await page.goto(`${frontend}/login`);
+  await page.getByLabel('Username', { exact: true }).fill(fixture.users[0].username);
+  await page.getByLabel('Password', { exact: true }).fill(fixture.users[0].password);
+  await page.getByRole('button', { name: 'Sign in', exact: true }).click();
+  await page.waitForURL(`${frontend}/monitors`);
+  await page.getByRole('article', { name: fixture.monitors[0].name, exact: true }).waitFor();
+  await page.getByRole('button', { name: 'View history', exact: true }).click();
+  await page.locator('tbody tr').nth(3).waitFor();
+  check(await page.locator('tbody tr').count() === 4, 'Production detail history must contain the three fixed failures and real manual result.');
+  const html = await page.content();
+  check(html.includes(fixture.monitors[0].name) && !html.includes(fixture.monitors[1].name), 'Production browser must preserve owner isolation.');
+  const asset = await page.locator('script[src^="/_next/static/"]').first().getAttribute('src');
+  check(asset, 'Production static asset link is missing.');
+  const assetResponse = await fetchAt(frontend, asset);
+  check(assetResponse.status === 200 && (assetResponse.headers.get('content-type') ?? '').includes('javascript'), 'Actual production static JavaScript must be served.');
+  const serverRoute = await fetchAt(frontend, '/api/auth/session', { headers: { cookie: cookies[0] } });
+  check(serverRoute.status === 200 && (await serverRoute.json()).data.user.id === scenario.dataset.userIds[0], 'Production server proxy route must reach the authenticated API.');
+  check(browserErrors.length === 0, 'Production browser reported a console or hydration error.');
+  for (const cookie of await context.cookies()) fixture.sentinels.push(cookie.value);
+  report.browser = { version: browser.version(), login: true, list: true, detailRows: 4, ownerIsolation: true,
+    staticStatus: assetResponse.status, serverRouteStatus: serverRoute.status, consoleErrors: 0, screenshots: 0, traces: 0 };
+  await browser.close(); browser = undefined;
+  stage = 'metric cardinality';
+  const before = await metrics(api);
+  for (let n = 1; n <= 10; n++) {
+    const response = await alice(`${api}/monitors/24000000-0000-4000-b000-${String(n).padStart(12, '0')}/checks`);
+    check(response.status === 404, 'Each fixed missing resource must return404.');
+  }
+  const after = await metrics(api);
+  check(JSON.stringify(samples(before).map(row => row.series).sort()) === JSON.stringify(samples(after).map(row => row.series).sort()),
+    'Distinct resource IDs must not create metric label series.');
+  const failedLabels = { route: '/monitors/:id/checks', method: 'GET', status: '404' };
+  check(metric(after, 'http_errors_total', failedLabels) - metric(before, 'http_errors_total', failedLabels) === 10,
+    'Actual missing-resource HTTP error count must increase by10.');
+  report.cardinality = { before: samples(before).length, after: samples(after).length, distinctMissingIds: 10, errorCountDelta: 10 };
+  const baselineContainers = await identities();
+  const authority = async () => JSON.stringify({
+    checks: (await pool.query('SELECT row_to_json(c) AS row FROM check_runs c ORDER BY id')).rows,
+    monitors: (await pool.query('SELECT row_to_json(m) AS row FROM monitors m ORDER BY id')).rows,
+  });
+  const beforeAuthority = await authority();
+  const initialCalls = await (await fetchAt(scenario.fixture.origin, scenario.fixture.counts)).json();
+  check(initialCalls.fail === 1 && initialCalls.ok === 0 && initialCalls.bodyWrites === 1, 'Exactly one controlled HTTP request/body write is permitted.');
+  if (mode === 'full') {
+    stage = 'owned PostgreSQL stop and restore';
+    check((await pool.query("SELECT count(*)::int AS count FROM check_runs WHERE state IN ('QUEUED','RUNNING')")).rows[0].count === 0, 'PostgreSQL must stop only at the fixed idle boundary.');
+    needsRestore = true; report.postgresStops++;
+    await docker([...pgCompose, 'stop', '-t', '5', scenario.ownership.postgresService]);
+    report.healthDown = await roleHealth(503);
+    const downMetrics = [await metrics(api), await metrics(worker)];
+    check(downMetrics.every(text => !text.includes('check_queue_age_seconds') && metric(text, 'postgres_ready') === 0),
+      'Unavailable PostgreSQL must not produce a false queue-age sample.');
+    const rejected = await fetchAt(api, `/monitors/${fixture.monitors[0].id}/checks`, {
+      method: 'POST', headers: { ...manualHeaders, 'idempotency-key': fixture.unavailableKey },
+    });
+    check(rejected.status >= 500, 'Authority outage must reject the one unsafe manual submission.');
+    check(JSON.stringify(await (await fetchAt(scenario.fixture.origin, scenario.fixture.counts)).json()) === JSON.stringify(initialCalls), 'No outbound request is permitted while authority is unavailable.');
+    await restorePostgres();
+    await until(async () => (await fetchAt(api, scenario.paths.ready)).status === 200 && (await fetchAt(worker, scenario.paths.ready)).status === 200,
+      'Roles did not regain authority readiness after the single restore.');
+    report.healthRestored = await roleHealth(200);
+    check(await authority() === beforeAuthority, 'PostgreSQL restore must preserve every authoritative CheckRun.');
+    check((await pool.query('SELECT count(*)::int AS count FROM check_runs')).rows[0].count === 5, 'Rejected outage intent must not create a row.');
+    check((await alice(`${api}/monitors/${fixture.monitors[0].id}/checks`)).status === 200 &&
+      (await bob(`${api}/monitors/${fixture.monitors[0].id}/checks`)).status === 404, 'Restored session and owner boundaries must remain authoritative.');
+    report.outage = { rejectedStatus: rejected.status, authoritativeRows: 5, preserved: true, noNewOutbound: true };
+  }
+  const restoredContainers = await identities();
+  check(scenario.roles.every(role => baselineContainers[role].pid === restoredContainers[role].pid && restoredContainers[role].running && restoredContainers[role].restarts === 0),
+    'Dependency loss must not restart or replace a role process.');
+  const finalMetrics = [await metrics(api), await metrics(worker)];
+  check(metric(finalMetrics[1], 'worker_claims_total') === 1, 'Outage must not create another claim.');
+  for (const text of finalMetrics) for (const sample of samples(text)) {
+    check(Number.isFinite(sample.value) && Object.keys(sample.labels).every(key => scenario.metrics.labels.includes(key)), 'Metric samples/label names must remain bounded.');
+    check(!Object.values(sample.labels).some(value => /[0-9a-f]{8}-[0-9a-f-]{27}/.test(value)), 'Resource/process IDs must not appear in metric labels.');
+  }
+  stage = 'actual runtime UID and signal boundary';
+  report.runtime = {};
+  report.signals = {};
+  for (const role of scenario.roles) {
+    const id = restoredContainers[role].id;
+    const script = 'const fs=require("node:fs");const s=fs.readFileSync("/proc/1/status","utf8");console.log(JSON.stringify({uid:Number(/Uid:\\s+(\\d+)/.exec(s)[1]),executable:fs.readlinkSync("/proc/1/exe"),actualCommandLine:fs.readFileSync("/proc/1/cmdline","utf8").split("\\0").filter(Boolean),node:process.versions.node,nodeEnv:process.env.NODE_ENV,buildId:process.env.NODE_ENV==="production"&&fs.existsSync(".next/BUILD_ID")?fs.readFileSync(".next/BUILD_ID","utf8").trim():null}));';
+    const runtime = JSON.parse(await docker(['exec', id, 'node', '-e', script]));
+    check(runtime.uid === scenario.runtime.uid && runtime.node === scenario.runtime.node && runtime.executable.endsWith('/node') &&
+      JSON.stringify(restoredContainers[role].command) === JSON.stringify(scenario.runtime.commands[role]),
+      'Real PID1 must run the pinned Node command as non-root.');
+    if (role === 'frontend') check(runtime.nodeEnv === 'production' && runtime.buildId, 'Frontend must execute a production build artifact.');
+    report.runtime[role] = { ...runtime, configuredCommand: restoredContainers[role].command,
+      image: restoredContainers[role].image, hostPid: restoredContainers[role].pid, restarts: 0 };
+    const stoppedAt = performance.now();
+    await docker(['kill', '--signal=SIGTERM', id]);
+    const exitCode = Number(await docker(['wait', id]));
+    const elapsedMs = Math.round(performance.now() - stoppedAt);
+    check(exitCode === 0 && elapsedMs <= scenario.runtime.shutdownMs[role], 'Actual role SIGTERM exit must be clean and within its frozen bound.');
+    report.signals[role] = { signal: 'SIGTERM', exitCode, elapsedMs, forced: false };
+  }
+  stage = 'safe structured logs';
+  const logs = {};
+  for (const role of scenario.roles) logs[role] = await docker(['logs', restoredContainers[role].id]);
+  const joined = [...Object.values(logs), ...finalMetrics].join('\n');
+  const leaks = fixture.sentinels.filter(value => joined.includes(value)).length;
+  check(leaks === 0, 'Runtime sentinel/credential appeared in logs or metrics.');
+  const apiLogs = logs.api.split('\n').filter(Boolean).map(line => JSON.parse(line));
+  const workerLogs = logs.worker.split('\n').filter(Boolean).map(line => JSON.parse(line));
+  check([...apiLogs, ...workerLogs].every(line => line.processId && line.pid === 1 && ['api', 'worker'].includes(line.role)), 'Every API/worker log must carry process correlation.');
+  const admission = apiLogs.find(line => line.event === 'check_accepted' && line.checkId === queued.id);
+  check(admission?.requestId && apiLogs.some(line => line.event === 'http_request' && line.requestId === admission.requestId && line.status === 202) &&
+    workerLogs.some(line => line.event === 'check_claimed' && line.checkId === queued.id) &&
+    workerLogs.some(line => line.event === 'check_completed' && line.checkId === queued.id), 'Request and worker events must correlate through the same accepted check.');
+  check(apiLogs.some(line => line.event === 'api_stopping') && workerLogs.some(line => line.event === 'worker_stopping'), 'Actual process signal handlers must be observed.');
+  report.logging = { apiJsonLines: apiLogs.length, workerJsonLines: workerLogs.length, correlatedCheck: true,
+    signalHandlersObserved: true, sentinelLeaks: leaks, scannedRuntimeSentinels: fixture.sentinels.length, rawLogsRetained: false };
+  // Persist metric text only after both bounded labels and all runtime secrets
+  // have been checked; an earlier failure must not write a sensitive capture.
+  report.metrics = finalMetrics;
+  report.result = 'PASS';
+} catch (error) {
+  report.result = 'FAILED';
+  report.failure = { stage, assertion: error instanceof SmokeFailure ? error.message : 'Unexpected observer/runtime error; private exception detail withheld.', kind: error.name };
+  process.exitCode = 1;
+} finally {
+  const cleanupErrors = [];
+  async function cleanup(name, action) {
+    try { await action(); return true; }
+    catch (error) {
+      cleanupErrors.push({ step: name, kind: error.name });
+      report.result = 'FAILED'; process.exitCode = 1;
+      report.failure ??= { stage: 'cleanup', assertion: `Owned ${name} cleanup failed; private exception detail withheld.`, kind: error.name };
+      return false;
+    }
+  }
+  if (browser) await cleanup('browser', () => browser.close());
+  if (needsRestore && !restoreAttempted) await cleanup('postgres restore', restorePostgres);
+  const runtimeRemoved = runtimeOwned && await cleanup('runtime', () => docker([...compose, 'down']));
+  const schemaDropped = schemaOwned && !needsRestore && (!runtimeOwned || runtimeRemoved) &&
+    await cleanup('schema', () => pool.query(`DROP SCHEMA ${schemaIdentifier(config.schema)} CASCADE`));
+  await cleanup('pool', () => pool.end());
+  const portsFree = await cleanup('ports', requireFreePorts);
+  report.cleanup = { schemaDropped, runtimeRemoved, postgresRestored: !needsRestore, portsFree, errors: cleanupErrors };
+  report.durationMs = Math.round(performance.now() - began);
+  try { await writeFile(`${output}/${mode}-${actor}.json`, JSON.stringify(report, null, 2) + '\n', { flag: 'wx' }); }
+  catch { report.result = 'FAILED'; process.exitCode = 1; report.artifactWriteFailed = true; }
+  console.log(JSON.stringify({ result: report.result, mode, actor, durationMs: report.durationMs, failure: report.failure, cleanup: report.cleanup }));
+}
diff --git a/test/unit.test.ts b/test/unit.test.ts
index 3f095b2..a3067e7 100644
--- a/test/unit.test.ts
+++ b/test/unit.test.ts
@@ -14,6 +14,26 @@ import historyScenario from '../evidence/E07/scenario.json' with { type: 'json'
 import { emptyMonitors, historyLocation } from '../app/monitors/initial-state.ts';
 import { e08RscTransport } from './e08-rsc-transport.mjs';
 import renderingScenario from '../evidence/E08/scenario.json' with { type: 'json' };
+import type { Pool } from 'pg';
+import { Operations } from '../server/operations.ts';
+
+test('E24 operational series normalize inputs and omit unavailable queue age', async () => {
+  const logs: string[] = [];
+  const operations = new Operations('api', line => logs.push(line));
+  for (let n = 0; n < 10; n++) operations.recordHttp(`/private/${n}`, `UNKNOWN${n}`, 404, 2, `generated-${n}`);
+  const pool = { query: async () => ({ rows: [{ seconds: '0.25' }] }) } as unknown as Pool;
+  const metrics = await operations.metrics(pool);
+  assert.match(metrics, /http_errors_total\{role="api",route="unmatched",method="OTHER",status="404"\} 10/);
+  assert.match(metrics, /http_request_duration_seconds_count.* 10/);
+  assert.match(metrics, /check_queue_age_seconds\{role="api"\} 0.25/);
+  assert.equal(operations.http.size, 1);
+  assert.ok(logs.every(line => JSON.parse(line).processId && !line.includes('/private/')));
+  const down = { query: async () => { throw new Error('private connection detail'); } } as unknown as Pool;
+  assert.equal(await operations.ready(down), false);
+  const unavailable = await operations.metrics(down);
+  assert.match(unavailable, /postgres_ready\{role="api"\} 0/);
+  assert.ok(!unavailable.includes('check_queue_age_seconds') && !unavailable.includes('private connection detail'));
+});
 
 test('E10 manual identity is bounded printable ASCII without whitespace or coercion', () => {
   for (const key of ['manual-intent-e10-1', '!', '~', 'x'.repeat(128)]) assert.equal(idempotencyKey(key), key);
diff --git a/test/worker.ts b/test/worker.ts
index 699bce9..dc6085e 100644
--- a/test/worker.ts
+++ b/test/worker.ts
@@ -11,7 +11,7 @@ import { DEFAULT_FIXTURE_ORIGIN } from '../server/outbound.ts';
 export async function startTestWorker(config: DatabaseConfig) {
   const child = spawn(process.execPath, ['server/worker.ts', '--no-schedule'], {
     env: { ...process.env, DATABASE_URL: config.connectionString, DATABASE_SCHEMA: config.schema,
-      NODE_ENV: 'test', WSE_TEST_FIXTURE_ORIGIN: DEFAULT_FIXTURE_ORIGIN },
+      NODE_ENV: 'test', WSE_TEST_FIXTURE_ORIGIN: DEFAULT_FIXTURE_ORIGIN, WORKER_HTTP_PORT: '0' },
     stdio: ['ignore', 'pipe', 'pipe'],
   });
   const exited = new Promise<{ code: number | null; signal: string | null }>((resolve) => child.once('close', (code, signal) => resolve({ code, signal })));


