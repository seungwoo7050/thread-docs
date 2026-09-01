## `Authenticate API calls through expiring PostgreSQL sessions`

diff --git a/app/monitors/api.ts b/app/monitors/api.ts
index 12d8b79..48f47ff 100644
--- a/app/monitors/api.ts
+++ b/app/monitors/api.ts
@@ -2,10 +2,11 @@ import type { ApiErrorCode } from '../../server/model.ts';
 
 export const ERROR_MESSAGES: Record<ApiErrorCode, string> = {
   INVALID_INPUT: 'Check the monitor input and try again.',
+  UNAUTHENTICATED: 'Sign in to continue.',
   NOT_FOUND: 'The requested monitor or resource was not found.',
   INTERNAL_ERROR: 'The monitoring service could not complete the request.',
 };
-const ERROR_STATUS = { INVALID_INPUT: 400, NOT_FOUND: 404, INTERNAL_ERROR: 500 } as const;
+const ERROR_STATUS = { INVALID_INPUT: 400, UNAUTHENTICATED: 401, NOT_FOUND: 404, INTERNAL_ERROR: 500 } as const;
 
 export class RequestFailure extends Error {
   readonly code: ApiErrorCode;
@@ -29,7 +30,7 @@ export async function responseData<T>(response: Response): Promise<T> {
   if (!response.ok) {
     if (isObject(body) && isObject(body.error) && typeof body.error.message === 'string') {
       const code = body.error.code;
-      if ((code === 'INVALID_INPUT' || code === 'NOT_FOUND' || code === 'INTERNAL_ERROR') && ERROR_STATUS[code] === response.status) {
+      if ((code === 'INVALID_INPUT' || code === 'UNAUTHENTICATED' || code === 'NOT_FOUND' || code === 'INTERNAL_ERROR') && ERROR_STATUS[code] === response.status) {
         throw new RequestFailure(code);
       }
     }
diff --git a/server/app.ts b/server/app.ts
index 7f23a17..7e6be67 100644
--- a/server/app.ts
+++ b/server/app.ts
@@ -9,6 +9,7 @@ import type { DatabaseConfig } from './database.ts';
 import { verifySchema } from './schema.ts';
 import { checkRunFromRow, checkRunToValues, monitorFromRow, monitorToValues, monitorViewFromRow } from './mapping.ts';
 import type { CheckRunRow, MonitorRow, MonitorViewRow } from './mapping.ts';
+import { registerAuthentication } from './auth.ts';
 
 const inputErrors = [
   errorCodes.FST_ERR_CTP_INVALID_JSON_BODY,
@@ -26,7 +27,7 @@ const monitorViewSql = `SELECT m.id, m.name, m.url, m.interval_seconds, m.enable
     FROM check_runs WHERE monitor_id = m.id ORDER BY finished_at DESC, id DESC LIMIT 1
   ) c ON true`;
 
-export function buildApp(fixtureOrigin = DEFAULT_FIXTURE_ORIGIN, database: DatabaseConfig = databaseConfig()) {
+export function buildApp(fixtureOrigin = DEFAULT_FIXTURE_ORIGIN, database: DatabaseConfig = databaseConfig(), now: () => number = Date.now) {
   const handleError = (error: unknown, _request: FastifyRequest, reply: FastifyReply) => {
     const failure = error instanceof ApiError ? error
       : inputErrors.some((ErrorType) => error instanceof ErrorType)
@@ -49,6 +50,7 @@ export function buildApp(fixtureOrigin = DEFAULT_FIXTURE_ORIGIN, database: Datab
 
   app.setErrorHandler(handleError);
   app.setNotFoundHandler(async () => { throw new ApiError('NOT_FOUND', 'Resource not found.'); });
+  registerAuthentication(app, pool, now);
 
   app.get('/health', async () => ({ data: { status: 'ok' } }));
   app.get('/monitors', async () => {
diff --git a/server/auth.ts b/server/auth.ts
new file mode 100644
index 0000000..b2e25bf
--- /dev/null
+++ b/server/auth.ts
@@ -0,0 +1,89 @@
+import { createHash, randomBytes } from 'node:crypto';
+import type { FastifyInstance } from 'fastify';
+import type { Pool } from 'pg';
+import { ApiError, loginInput } from './contracts.ts';
+import { userFromRow } from './mapping.ts';
+import type { UserRow } from './mapping.ts';
+import type { User } from './model.ts';
+import { DUMMY_PASSWORD_HASH, verifyPassword } from './password.ts';
+
+declare module 'fastify' {
+  interface FastifyRequest { user: User | null }
+}
+
+export const SESSION_TTL_MS = 3_600_000;
+export const SESSION_COOKIE_NAME = 'wse_session';
+// This application only serves loopback HTTP. Secure is intentionally absent for
+// that transport; no public/production HTTP deployment is supported.
+const COOKIE_ATTRIBUTES = 'Path=/; HttpOnly; SameSite=Lax';
+export const CLEAR_SESSION_COOKIE = `${SESSION_COOKIE_NAME}=; ${COOKIE_ATTRIBUTES}; Max-Age=0`;
+
+export function sessionTokenFromCookie(header: string | undefined): string | null {
+  const matches = (header ?? '').split(';').map((part) => part.trim())
+    .filter((part) => part.startsWith(`${SESSION_COOKIE_NAME}=`));
+  if (matches.length !== 1) return null;
+  const value = matches[0].slice(SESSION_COOKIE_NAME.length + 1);
+  // Issued identifiers are exactly 32 random bytes encoded as base64url. Do not
+  // decode arbitrary cookie input or accept ambiguous duplicate cookie names.
+  return /^[A-Za-z0-9_-]{43}$/.test(value) ? value : null;
+}
+
+export function sessionTokenHash(token: string): string {
+  return createHash('sha256').update(token).digest('hex');
+}
+
+function unauthenticated() { return new ApiError('UNAUTHENTICATED', 'Authentication is required.'); }
+
+export function registerAuthentication(app: FastifyInstance, pool: Pool, now: () => number) {
+  app.decorateRequest('user', null);
+  app.addHook('onRequest', async (request, reply) => {
+    reply.header('cache-control', 'no-store');
+    const route = request.routeOptions.url;
+    if ((route === '/health' && (request.method === 'GET' || request.method === 'HEAD')) ||
+      (route === '/auth/login' && request.method === 'POST')) return;
+
+    const token = sessionTokenFromCookie(request.headers.cookie);
+    const result = token ? await pool.query<Pick<UserRow, 'id' | 'username'>>(`
+      SELECT u.id, u.username FROM sessions s JOIN users u ON u.id = s.user_id
+      WHERE s.token_hash = $1 AND s.expires_at > $2`, [sessionTokenHash(token), new Date(now())]) : null;
+    if (!result?.rows[0]) {
+      reply.header('set-cookie', CLEAR_SESSION_COOKIE);
+      throw unauthenticated();
+    }
+    request.user = userFromRow(result.rows[0]);
+  });
+
+  app.post<{ Body: unknown }>('/auth/login', async (request, reply) => {
+    const input = loginInput(request.body);
+    const result = await pool.query<UserRow>('SELECT id, username, password_hash FROM users WHERE username = $1', [input.username]);
+    const row = result.rows[0];
+    const passwordValid = await verifyPassword(input.password, row?.password_hash ?? DUMMY_PASSWORD_HASH);
+    if (!row || !passwordValid) throw unauthenticated();
+
+    const token = randomBytes(32).toString('base64url');
+    const previous = sessionTokenFromCookie(request.headers.cookie);
+    const createdAt = new Date(now());
+    const expiresAt = new Date(createdAt.getTime() + SESSION_TTL_MS);
+    const client = await pool.connect();
+    try {
+      // Revocation and replacement use one explicit connection and transaction.
+      await client.query('BEGIN');
+      if (previous) await client.query('DELETE FROM sessions WHERE token_hash = $1', [sessionTokenHash(previous)]);
+      await client.query('INSERT INTO sessions (token_hash, user_id, created_at, expires_at) VALUES ($1, $2, $3, $4)',
+        [sessionTokenHash(token), row.id, createdAt, expiresAt]);
+      await client.query('COMMIT');
+    } catch (error) { await client.query('ROLLBACK'); throw error; }
+    finally { client.release(); }
+    reply.header('set-cookie', `${SESSION_COOKIE_NAME}=${token}; ${COOKIE_ATTRIBUTES}; Max-Age=${SESSION_TTL_MS / 1000}`);
+    return { data: { user: userFromRow(row) } };
+  });
+
+  app.get('/auth/session', async (request) => ({ data: { user: request.user } }));
+
+  app.post('/auth/logout', async (request, reply) => {
+    const token = sessionTokenFromCookie(request.headers.cookie)!; // onRequest authenticated this exact cookie.
+    await pool.query('DELETE FROM sessions WHERE token_hash = $1', [sessionTokenHash(token)]);
+    reply.header('set-cookie', CLEAR_SESSION_COOKIE);
+    return { data: { loggedOut: true } };
+  });
+}
diff --git a/server/contracts.ts b/server/contracts.ts
index 0428a43..3c0c5a9 100644
--- a/server/contracts.ts
+++ b/server/contracts.ts
@@ -1,7 +1,7 @@
 import { fixtureUrl } from './check.ts';
 import type { ApiErrorCode, Monitor } from './model.ts';
 
-export const ERROR_STATUS = { INVALID_INPUT: 400, NOT_FOUND: 404, INTERNAL_ERROR: 500 } as const;
+export const ERROR_STATUS = { INVALID_INPUT: 400, UNAUTHENTICATED: 401, NOT_FOUND: 404, INTERNAL_ERROR: 500 } as const;
 
 export class ApiError extends Error {
   readonly code: ApiErrorCode;
@@ -42,3 +42,15 @@ export function monitorId(value: string): string {
   }
   return value.toLowerCase();
 }
+
+export function loginInput(value: unknown): { username: string; password: string } {
+  if (value === null || typeof value !== 'object' || Array.isArray(value)) {
+    throw new ApiError('INVALID_INPUT', 'A login JSON object is required.');
+  }
+  const { username, password } = value as Record<string, unknown>;
+  if (typeof username !== 'string' || !/^[a-z][a-z0-9_-]{2,63}$/.test(username) ||
+    typeof password !== 'string' || password.length < 1 || password.length > 1024) {
+    throw new ApiError('INVALID_INPUT', 'Username must be 3–64 lowercase letters, digits, underscores or hyphens, starting with a letter; password must be 1–1024 characters.');
+  }
+  return { username, password };
+}
diff --git a/server/mapping.ts b/server/mapping.ts
index f8fddbd..a475d5f 100644
--- a/server/mapping.ts
+++ b/server/mapping.ts
@@ -1,4 +1,4 @@
-import type { CheckRun, Monitor, MonitorView } from './model.ts';
+import type { CheckRun, Monitor, MonitorView, User } from './model.ts';
 
 // PostgreSQL rows use snake_case, integer/boolean columns, and pg Date objects.
 // API/domain models use camelCase and canonical UTC ISO strings, never raw rows.
@@ -12,6 +12,12 @@ export type CheckRunRow = {
   started_at: Date; finished_at: Date;
 };
 export type MonitorViewRow = MonitorRow & Omit<CheckRunRow, 'id'> & { check_id: string | null };
+export type UserRow = { id: string; username: string; password_hash: string };
+
+// Only these two public fields may cross the authentication response boundary.
+export function userFromRow(row: Pick<UserRow, 'id' | 'username'>): User {
+  return { id: row.id, username: row.username };
+}
 
 export function monitorFromRow(row: MonitorRow): Monitor {
   return {
diff --git a/server/migrate.ts b/server/migrate.ts
index 9d907e4..91c3c91 100644
--- a/server/migrate.ts
+++ b/server/migrate.ts
@@ -5,7 +5,7 @@ import { databaseConfig, databasePool, schemaIdentifier } from './database.ts';
 import type { DatabaseConfig } from './database.ts';
 
 export async function migrationFiles() {
-  const names = ['001_monitors.sql', '002_check_runs.sql'];
+  const names = ['001_monitors.sql', '002_check_runs.sql', '003_sessions.sql'];
   return Promise.all(names.map(async (version) => {
     const sql = await readFile(new URL(`./migrations/${version}`, import.meta.url), 'utf8');
     return { version, sql, checksum: createHash('sha256').update(sql).digest('hex') };
diff --git a/server/migrations/003_sessions.sql b/server/migrations/003_sessions.sql
new file mode 100644
index 0000000..e390c16
--- /dev/null
+++ b/server/migrations/003_sessions.sql
@@ -0,0 +1,12 @@
+CREATE TABLE users (
+  id uuid PRIMARY KEY,
+  username text NOT NULL UNIQUE CHECK (username ~ '^[a-z][a-z0-9_-]{2,63}$'),
+  password_hash text NOT NULL
+);
+
+CREATE TABLE sessions (
+  token_hash text PRIMARY KEY CHECK (token_hash ~ '^[0-9a-f]{64}$'),
+  user_id uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
+  created_at timestamptz(3) NOT NULL,
+  expires_at timestamptz(3) NOT NULL CHECK (expires_at > created_at)
+);
diff --git a/server/model.ts b/server/model.ts
index 03f1253..51fa3af 100644
--- a/server/model.ts
+++ b/server/model.ts
@@ -22,6 +22,8 @@ export type CheckRun = {
 
 export type MonitorView = Monitor & { latestCheck: CheckRun | null };
 
-export type ApiErrorCode = 'INVALID_INPUT' | 'NOT_FOUND' | 'INTERNAL_ERROR';
+export type User = { id: string; username: string };
+
+export type ApiErrorCode = 'INVALID_INPUT' | 'UNAUTHENTICATED' | 'NOT_FOUND' | 'INTERNAL_ERROR';
 export type ApiSuccess<T> = { data: T };
 export type ApiFailure = { error: { code: ApiErrorCode; message: string } };
diff --git a/server/password.ts b/server/password.ts
new file mode 100644
index 0000000..63dc87a
--- /dev/null
+++ b/server/password.ts
@@ -0,0 +1,28 @@
+import { randomBytes, scrypt, timingSafeEqual } from 'node:crypto';
+
+// A fixed password-storage cost, not a per-request or user-controlled parameter.
+export const SCRYPT_OPTIONS = { N: 131_072, r: 8, p: 1, maxmem: 256 * 1024 * 1024 } as const;
+const HASH_PREFIX = 'scrypt$131072$8$1$';
+const HASH_PATTERN = /^scrypt\$131072\$8\$1\$([0-9a-f]{32})\$([0-9a-f]{128})$/;
+
+function derive(password: string, salt: Buffer): Promise<Buffer> {
+  return new Promise((resolve, reject) => {
+    scrypt(password, salt, 64, SCRYPT_OPTIONS, (error, key) => error ? reject(error) : resolve(key));
+  });
+}
+
+export async function hashPassword(password: string): Promise<string> {
+  const salt = randomBytes(16);
+  const key = await derive(password, salt);
+  return `${HASH_PREFIX}${salt.toString('hex')}$${key.toString('hex')}`;
+}
+
+// Missing accounts still pay the same KDF cost; this is not a usable credential.
+export const DUMMY_PASSWORD_HASH = `${HASH_PREFIX}${randomBytes(16).toString('hex')}$${randomBytes(64).toString('hex')}`;
+
+export async function verifyPassword(password: string, encoded: string): Promise<boolean> {
+  const parts = HASH_PATTERN.exec(encoded);
+  if (!parts) throw new Error('Invalid stored password representation.');
+  const actual = await derive(password, Buffer.from(parts[1], 'hex'));
+  return timingSafeEqual(actual, Buffer.from(parts[2], 'hex'));
+}
diff --git a/server/schema.ts b/server/schema.ts
index 27f16b8..2eba2ff 100644
--- a/server/schema.ts
+++ b/server/schema.ts
@@ -12,6 +12,13 @@ const expectedColumns = {
     http_status: ['int4', 'YES'], latency_ms: ['int4', 'NO'], failure_reason: ['text', 'YES'],
     started_at: ['timestamptz', 'NO'], finished_at: ['timestamptz', 'NO'],
   },
+  users: {
+    id: ['uuid', 'NO'], username: ['text', 'NO'], password_hash: ['text', 'NO'],
+  },
+  sessions: {
+    token_hash: ['text', 'NO'], user_id: ['uuid', 'NO'],
+    created_at: ['timestamptz', 'NO'], expires_at: ['timestamptz', 'NO'],
+  },
 } as const;
 
 export async function verifySchema(pool: Pool): Promise<void> {
@@ -22,7 +29,7 @@ export async function verifySchema(pool: Pool): Promise<void> {
     const columns = await client.query<{
       table_name: string; column_name: string; udt_name: string; is_nullable: string; datetime_precision: number | null;
     }>(`SELECT table_name, column_name, udt_name, is_nullable, datetime_precision
-      FROM information_schema.columns WHERE table_schema = current_schema() AND table_name IN ('monitors', 'check_runs')`);
+      FROM information_schema.columns WHERE table_schema = current_schema() AND table_name IN ('monitors', 'check_runs', 'users', 'sessions')`);
     for (const column of columns.rows) {
       const fields = expectedColumns[column.table_name as keyof typeof expectedColumns];
       if (!Object.hasOwn(fields, column.column_name)) {
@@ -42,14 +49,20 @@ export async function verifySchema(pool: Pool): Promise<void> {
       SELECT rel.relname AS table_name, pg_get_constraintdef(con.oid) AS definition
       FROM pg_constraint con JOIN pg_class rel ON rel.oid = con.conrelid
       JOIN pg_namespace ns ON ns.oid = rel.relnamespace
-      WHERE ns.nspname = current_schema() AND rel.relname IN ('monitors', 'check_runs') AND con.contype IN ('p', 'f')`);
-    for (const table of ['monitors', 'check_runs']) {
-      if (!keys.rows.some((row) => row.table_name === table && row.definition === 'PRIMARY KEY (id)')) {
+      WHERE ns.nspname = current_schema() AND rel.relname IN ('monitors', 'check_runs', 'users', 'sessions') AND con.contype IN ('p', 'f', 'u')`);
+    for (const [table, column] of [['monitors', 'id'], ['check_runs', 'id'], ['users', 'id'], ['sessions', 'token_hash']]) {
+      if (!keys.rows.some((row) => row.table_name === table && row.definition === `PRIMARY KEY (${column})`)) {
         throw new Error(`Incompatible database schema: ${table} primary key.`);
       }
     }
     if (!keys.rows.some((row) => row.table_name === 'check_runs' && row.definition === 'FOREIGN KEY (monitor_id) REFERENCES monitors(id) ON DELETE CASCADE')) {
       throw new Error('Incompatible database schema: CheckRun parent deletion rule.');
     }
+    if (!keys.rows.some((row) => row.table_name === 'users' && row.definition === 'UNIQUE (username)')) {
+      throw new Error('Incompatible database schema: unique username.');
+    }
+    if (!keys.rows.some((row) => row.table_name === 'sessions' && row.definition === 'FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE')) {
+      throw new Error('Incompatible database schema: session user deletion rule.');
+    }
   } finally { client.release(); }
 }
diff --git a/test/auth.ts b/test/auth.ts
new file mode 100644
index 0000000..cecded1
--- /dev/null
+++ b/test/auth.ts
@@ -0,0 +1,54 @@
+import { randomBytes, randomUUID } from 'node:crypto';
+import type { FastifyInstance, InjectOptions } from 'fastify';
+import { databasePool } from '../server/database.ts';
+import type { DatabaseConfig } from '../server/database.ts';
+import { hashPassword } from '../server/password.ts';
+import { SESSION_COOKIE_NAME } from '../server/auth.ts';
+import scenario from '../evidence/E04/scenario.json' with { type: 'json' };
+
+export type TestCredentials = { username: string; password: string };
+
+// Passwords exist only in this test process. Never serialize these credentials,
+// request headers, login payloads, or session values into evidence or logs.
+export async function prepareTestUsers(config: DatabaseConfig): Promise<TestCredentials[]> {
+  const pool = databasePool(config);
+  const credentials: TestCredentials[] = [];
+  try {
+    for (const username of scenario.users) {
+      const password = randomBytes(32).toString('base64url');
+      const encoded = await hashPassword(password);
+      await pool.query('INSERT INTO users (id, username, password_hash) VALUES ($1, $2, $3)',
+        [randomUUID(), username, encoded]);
+      credentials.push({ username, password });
+    }
+    return credentials;
+  } finally { await pool.end(); }
+}
+
+export function cookieFromHeader(header: string | string[] | undefined): string {
+  const value = Array.isArray(header) ? header[0] : header;
+  const cookie = value?.split(';')[0];
+  if (!cookie?.startsWith(`${SESSION_COOKIE_NAME}=`)) throw new Error('Expected a session cookie without exposing its value.');
+  return cookie;
+}
+
+export async function loginForTest(app: FastifyInstance, credentials: TestCredentials): Promise<string> {
+  const response = await app.inject({ method: 'POST', url: '/auth/login', payload: credentials });
+  if (response.statusCode !== 200) throw new Error(`Fixture login failed with status ${response.statusCode}.`);
+  return cookieFromHeader(response.headers['set-cookie']);
+}
+
+export function authenticatedInject(app: FastifyInstance, cookie: string) {
+  return (input: string | InjectOptions) => {
+    const options = typeof input === 'string' ? { url: input } : input;
+    return app.inject({ ...options, headers: { ...options.headers, cookie } });
+  };
+}
+
+export function authenticatedFetch(cookie: string) {
+  return (url: string, options: RequestInit = {}) => {
+    const headers = new Headers(options.headers);
+    headers.set('cookie', cookie);
+    return fetch(url, { ...options, headers });
+  };
+}
diff --git a/test/contracts.test.ts b/test/contracts.test.ts
index 3790cac..e9c921e 100644
--- a/test/contracts.test.ts
+++ b/test/contracts.test.ts
@@ -1,19 +1,27 @@
 import { after, before, test } from 'node:test';
 import assert from 'node:assert/strict';
+import { randomBytes } from 'node:crypto';
+import { mkdir, writeFile } from 'node:fs/promises';
 import { buildApp } from '../server/app.ts';
 import { fixtureServer } from './fixture.ts';
+import { authenticatedFetch, cookieFromHeader, loginForTest, prepareTestUsers } from './auth.ts';
+import { SESSION_COOKIE_NAME, sessionTokenFromCookie, sessionTokenHash } from '../server/auth.ts';
+import { databasePool } from '../server/database.ts';
 import { dropTestSchema, resetTestSchema, testDatabaseConfig } from './database.ts';
 import type { ApiErrorCode, ApiSuccess, CheckRun, MonitorView } from '../server/model.ts';
 import scenario from '../evidence/E02/scenario.json' with { type: 'json' };
+import authScenario from '../evidence/E04/scenario.json' with { type: 'json' };
 
 const fixture = fixtureServer();
 const database = testDatabaseConfig('e03_contracts');
 const app = buildApp(scenario.fixtureOrigin, database);
 const boundaries = scenario.additionalBoundaries;
+let request: ReturnType<typeof authenticatedFetch>;
 app.get(boundaries.internalErrorRoute, async () => { throw new Error(boundaries.privateInternalMessage); });
 
 before(async () => {
   await resetTestSchema(database.schema);
+  request = authenticatedFetch(await loginForTest(app, (await prepareTestUsers(database))[0]));
   await new Promise<void>((resolve) => fixture.server.listen(4311, '127.0.0.1', resolve));
   await app.listen({ host: '127.0.0.1', port: 4312 });
 });
@@ -26,7 +34,7 @@ after(async () => {
 });
 
 test('E02 fixed counterexample: blank name is INVALID_INPUT', async () => {
-  const response = await fetch(`${scenario.apiOrigin}/monitors`, {
+  const response = await request(`${scenario.apiOrigin}/monitors`, {
     method: 'POST', headers: { 'content-type': 'application/json' },
     body: JSON.stringify({ ...scenario.monitor, name: scenario.fixedCounterexample.name }),
   });
@@ -38,7 +46,7 @@ test('E02 fixed counterexample: blank name is INVALID_INPUT', async () => {
 });
 
 function postMonitor(body: unknown) {
-  return fetch(`${scenario.apiOrigin}/monitors`, {
+  return request(`${scenario.apiOrigin}/monitors`, {
     method: 'POST', headers: { 'content-type': 'application/json' }, body: JSON.stringify(body),
   });
 }
@@ -79,16 +87,16 @@ function assertMonitor(monitor: MonitorView) {
 }
 
 test('E02 fixed malformed values are INVALID_INPUT and do not create Monitors', async () => {
-  const before = await success<MonitorView[]>(await fetch(`${scenario.apiOrigin}/monitors`), 200);
+  const before = await success<MonitorView[]>(await request(`${scenario.apiOrigin}/monitors`), 200);
   for (const override of scenario.requiredInvalidOverrides) {
     await assertFailure(await postMonitor({ ...scenario.monitor, ...override }), 400, 'INVALID_INPUT');
   }
-  const after = await success<MonitorView[]>(await fetch(`${scenario.apiOrigin}/monitors`), 200);
+  const after = await success<MonitorView[]>(await request(`${scenario.apiOrigin}/monitors`), 200);
   assert.deepEqual(after, before);
 });
 
 test('E02 validates required field types, trimmed name length and parsed numeric integers', async () => {
-  const before = await success<MonitorView[]>(await fetch(`${scenario.apiOrigin}/monitors`), 200);
+  const before = await success<MonitorView[]>(await request(`${scenario.apiOrigin}/monitors`), 200);
   const invalid = [
     ...boundaries.invalidOverrides.map((override) => ({ ...scenario.monitor, ...override })),
     ...boundaries.invalidNameRepetitions.map(({ unit, count }) => ({ ...scenario.monitor, name: unit.repeat(count) })),
@@ -96,7 +104,7 @@ test('E02 validates required field types, trimmed name length and parsed numeric
     ...boundaries.invalidBodies,
   ];
   for (const body of invalid) await assertFailure(await postMonitor(body), 400, 'INVALID_INPUT');
-  const after = await success<MonitorView[]>(await fetch(`${scenario.apiOrigin}/monitors`), 200);
+  const after = await success<MonitorView[]>(await request(`${scenario.apiOrigin}/monitors`), 200);
   assert.deepEqual(after, before);
 });
 
@@ -119,35 +127,35 @@ test('E02 accepts and serializes trimmed name and interval boundary values witho
 
 test('E02 missing Monitor and route are NOT_FOUND; malformed ID and path are INVALID_INPUT', async () => {
   const calls = [...fixture.calls];
-  await assertFailure(await fetch(`${scenario.apiOrigin}/monitors/${scenario.unknownMonitorId}/checks`, { method: 'POST' }), 404, 'NOT_FOUND');
-  await assertFailure(await fetch(`${scenario.apiOrigin}${boundaries.missingRoute}`), 404, 'NOT_FOUND');
-  await assertFailure(await fetch(`${scenario.apiOrigin}/monitors/${boundaries.malformedMonitorId}/checks`, { method: 'POST' }), 400, 'INVALID_INPUT');
-  await assertFailure(await fetch(`${scenario.apiOrigin}${boundaries.malformedRequestPath}`, { method: 'POST' }), 400, 'INVALID_INPUT');
+  await assertFailure(await request(`${scenario.apiOrigin}/monitors/${scenario.unknownMonitorId}/checks`, { method: 'POST' }), 404, 'NOT_FOUND');
+  await assertFailure(await request(`${scenario.apiOrigin}${boundaries.missingRoute}`), 404, 'NOT_FOUND');
+  await assertFailure(await request(`${scenario.apiOrigin}/monitors/${boundaries.malformedMonitorId}/checks`, { method: 'POST' }), 400, 'INVALID_INPUT');
+  await assertFailure(await request(`${scenario.apiOrigin}${boundaries.malformedRequestPath}`, { method: 'POST' }), 400, 'INVALID_INPUT');
   assert.deepEqual([...fixture.calls], calls);
 });
 
 test('E02 parser, media type and body size failures use the same INVALID_INPUT envelope', async () => {
   for (const body of [boundaries.malformedJson, boundaries.emptyJson, boundaries.oversizedBody.unit.repeat(boundaries.oversizedBody.count)]) {
-    await assertFailure(await fetch(`${scenario.apiOrigin}/monitors`, {
+    await assertFailure(await request(`${scenario.apiOrigin}/monitors`, {
       method: 'POST', headers: { 'content-type': 'application/json' }, body,
     }), 400, 'INVALID_INPUT');
   }
-  await assertFailure(await fetch(`${scenario.apiOrigin}/monitors`, {
+  await assertFailure(await request(`${scenario.apiOrigin}/monitors`, {
     method: 'POST', headers: { 'content-type': boundaries.unsupportedContentType }, body: boundaries.unsupportedBody,
   }), 400, 'INVALID_INPUT');
-  await assertFailure(await fetch(`${scenario.apiOrigin}/monitors`, { method: 'POST' }), 400, 'INVALID_INPUT');
+  await assertFailure(await request(`${scenario.apiOrigin}/monitors`, { method: 'POST' }), 400, 'INVALID_INPUT');
 });
 
 test('E02 unexpected API failures are INTERNAL_ERROR without private details', async () => {
-  await assertFailure(await fetch(`${scenario.apiOrigin}${boundaries.internalErrorRoute}`), 500, 'INTERNAL_ERROR');
-  assert.deepEqual(await success(await fetch(`${scenario.apiOrigin}/health`), 200), { status: 'ok' });
+  await assertFailure(await request(`${scenario.apiOrigin}${boundaries.internalErrorRoute}`), 500, 'INTERNAL_ERROR');
+  assert.deepEqual(await success(await request(`${scenario.apiOrigin}/health`), 200), { status: 'ok' });
 });
 
 test('E02 terminal CheckRun wire values distinguish observed HTTP failure from no response', async () => {
   for (const expected of scenario.terminalResults) {
     const monitor = await success<MonitorView>(await postMonitor({ ...scenario.monitor, url: `${scenario.fixtureOrigin}${expected.path}` }), 201);
     assertMonitor(monitor);
-    const result = await success<CheckRun>(await fetch(`${scenario.apiOrigin}/monitors/${monitor.id}/checks`, { method: 'POST' }), expected.apiStatus);
+    const result = await success<CheckRun>(await request(`${scenario.apiOrigin}/monitors/${monitor.id}/checks`, { method: 'POST' }), expected.apiStatus);
     assert.deepEqual(Object.keys(result).sort(), ['failureReason', 'finishedAt', 'httpStatus', 'id', 'latencyMs', 'monitorId', 'startedAt', 'state', 'trigger']);
     assert.match(result.id, /^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/);
     assert.equal(result.monitorId, monitor.id);
@@ -160,7 +168,130 @@ test('E02 terminal CheckRun wire values distinguish observed HTTP failure from n
     assertTimestamp(result.finishedAt);
     assert.ok(Date.parse(result.finishedAt) >= Date.parse(result.startedAt));
     assert.equal(fixture.calls.get(expected.path), 1);
-    const list = await success<MonitorView[]>(await fetch(`${scenario.apiOrigin}/monitors`), 200);
+    const list = await success<MonitorView[]>(await request(`${scenario.apiOrigin}/monitors`), 200);
     assert.deepEqual(list.find((item) => item.id === monitor.id)?.latestCheck, result);
   }
 });
+
+test('E04 fixed session sequence: anonymous, login, wrong password, rotation, logout, exact expiry and independent Bob', async () => {
+  const started = performance.now();
+  const config = await resetTestSchema(authScenario.additionalAssertions.schema);
+  const pool = databasePool(config);
+  let now = Date.parse(authScenario.session.clockStart);
+  const sessionApp = buildApp(authScenario.fixtureOrigin, config, () => now);
+  const observations: Record<string, unknown> = {};
+  const rejection = (response: Awaited<ReturnType<typeof sessionApp.inject>>) => {
+    assert.equal(response.statusCode, 401);
+    assert.deepEqual(response.json(), { error: { code: 'UNAUTHENTICATED', message: 'Authentication is required.' } });
+    return response.statusCode;
+  };
+  const get = (cookie?: string) => sessionApp.inject({ url: '/monitors', headers: cookie ? { cookie } : {} });
+  const login = (credentials: { username: string; password: string }, cookie?: string) => sessionApp.inject({
+    method: 'POST', url: '/auth/login', payload: credentials, headers: cookie ? { cookie } : {},
+  });
+  const acceptLogin = (response: Awaited<ReturnType<typeof sessionApp.inject>>, username: string) => {
+    assert.equal(response.statusCode, 200);
+    assert.equal(response.headers['cache-control'], 'no-store');
+    const data = response.json().data;
+    assert.deepEqual(Object.keys(data), ['user']);
+    assert.deepEqual(Object.keys(data.user).sort(), ['id', 'username']);
+    assert.equal(data.user.username, username);
+    const header = String(response.headers['set-cookie']);
+    const flags = {
+      name: SESSION_COOKIE_NAME, httpOnly: header.includes('; HttpOnly'), sameSite: /; SameSite=([^;]+)/.exec(header)?.[1],
+      secure: header.includes('; Secure'), path: /; Path=([^;]+)/.exec(header)?.[1],
+      hostOnly: !header.includes('; Domain='), maxAgeSeconds: Number(/; Max-Age=(\d+)/.exec(header)?.[1]),
+    };
+    assert.deepEqual(flags, authScenario.session.cookie);
+    observations.cookieFlags = flags;
+    return cookieFromHeader(response.headers['set-cookie']);
+  };
+  try {
+    const users = await prepareTestUsers(config);
+    const [alice, bob] = users;
+    assert.ok(alice.password !== bob.password);
+    const rows = (await pool.query<{ username: string; password_hash: string }>('SELECT username, password_hash FROM users ORDER BY username')).rows;
+    const passwordRepresentation = {
+      count: rows.length,
+      saltedScrypt: rows.every((row) => /^scrypt\$131072\$8\$1\$[0-9a-f]{32}\$[0-9a-f]{128}$/.test(row.password_hash)),
+      independentSalts: new Set(rows.map((row) => row.password_hash.split('$')[4])).size === 2,
+      noPlaintext: rows.every((row) => users.every((user) => !row.password_hash.includes(user.password))),
+    };
+    assert.deepEqual(passwordRepresentation, { count: 2, saltedScrypt: true, independentSalts: true, noPlaintext: true });
+    observations.passwordRepresentation = passwordRepresentation;
+
+    observations.anonymousStatus = rejection(await get());
+    const first = await login(alice);
+    const firstCookie = acceptLogin(first, alice.username);
+    observations.aliceLoginStatus = first.statusCode;
+    const wrong = await login({ ...alice, password: randomBytes(32).toString('base64url') });
+    observations.wrongPasswordStatus = rejection(wrong);
+    const authenticated = await get(firstCookie);
+    assert.equal(authenticated.statusCode, 200);
+    observations.authenticatedStatus = authenticated.statusCode;
+
+    const second = await login(alice, firstCookie);
+    const secondCookie = acceptLogin(second, alice.username);
+    assert.ok(secondCookie !== firstCookie, 'A successful login must replace the identifier.');
+    const oldStatus = rejection(await get(firstCookie));
+    assert.equal((await get(secondCookie)).statusCode, 200);
+    observations.rotation = { changed: true, oldStatus, newStatus: 200 };
+    const logout = await sessionApp.inject({ method: 'POST', url: '/auth/logout', headers: { cookie: secondCookie } });
+    assert.equal(logout.statusCode, 200);
+    assert.deepEqual(logout.json(), { data: { loggedOut: true } });
+    assert.ok(String(logout.headers['set-cookie']).includes('; Max-Age=0'));
+    observations.logout = { status: logout.statusCode, oldStatus: rejection(await get(secondCookie)), cleared: true };
+
+    const freshCookie = acceptLogin(await login(alice), alice.username);
+    now = Date.parse(authScenario.session.clockStart) + authScenario.session.validCheckpointMs;
+    const beforeExpiry = await get(freshCookie);
+    assert.equal(beforeExpiry.statusCode, 200);
+    now = Date.parse(authScenario.session.clockStart) + authScenario.session.expiredCheckpointMs;
+    observations.expiry = { beforeStatus: beforeExpiry.statusCode, atExpiryStatus: rejection(await get(freshCookie)), sleepMs: 0 };
+    const bobResponse = await login(bob);
+    const bobCookie = acceptLogin(bobResponse, bob.username);
+    observations.bobLoginStatus = bobResponse.statusCode;
+    const bobSession = await sessionApp.inject({ url: '/auth/session', headers: { cookie: bobCookie } });
+    assert.equal(bobSession.statusCode, 200);
+    assert.deepEqual(Object.keys(bobSession.json().data.user).sort(), ['id', 'username']);
+    assert.equal(bobSession.json().data.user.username, bob.username);
+
+    // Same externally visible rejection for an unknown account; a failed login
+    // carrying a valid cookie does not revoke or rotate that valid session.
+    const absent = await login({ username: authScenario.unknownUsername, password: randomBytes(32).toString('base64url') }, bobCookie);
+    rejection(absent);
+    assert.deepEqual(absent.json(), wrong.json());
+    assert.equal((await get(bobCookie)).statusCode, 200);
+    observations.accountExistenceHidden = true;
+
+    const stored = (await pool.query<{ token_hash: string; created_at: Date; expires_at: Date }>('SELECT token_hash, created_at, expires_at FROM sessions')).rows;
+    const token = sessionTokenFromCookie(bobCookie)!;
+    assert.ok(stored.some((row) => row.token_hash === sessionTokenHash(token)));
+    assert.ok(stored.every((row) => row.token_hash !== token && /^[0-9a-f]{64}$/.test(row.token_hash)));
+    assert.ok(stored.every((row) => row.expires_at.getTime() - row.created_at.getTime() === authScenario.session.ttlMs));
+    assert.equal((await pool.query('SELECT token_hash FROM sessions WHERE token_hash = ANY($1::text[])',
+      [[sessionTokenHash(sessionTokenFromCookie(firstCookie)!), sessionTokenHash(sessionTokenFromCookie(secondCookie)!)]])).rowCount, 0);
+    observations.sessionStorage = { hashOnly: true, finiteTtl: true, revokedRowsAbsent: true };
+
+    const freshApp = buildApp(authScenario.fixtureOrigin, config, () => now);
+    try {
+      assert.equal((await freshApp.inject({ url: '/monitors', headers: { cookie: bobCookie } })).statusCode, 200);
+      rejection(await freshApp.inject({ url: '/monitors', headers: { cookie: secondCookie } }));
+      observations.newInstance = { validStatus: 200, revokedStatus: 401 };
+    } finally { await freshApp.close(); }
+
+    let protectedRejections = 0;
+    const nonexistentCookie = `${SESSION_COOKIE_NAME}=${randomBytes(32).toString('base64url')}`;
+    for (const cookie of [undefined, nonexistentCookie, freshCookie, secondCookie]) {
+      for (const [method, url] of authScenario.additionalAssertions.protectedRequests) {
+        rejection(await sessionApp.inject({ method: method as 'GET' | 'POST' | 'PUT' | 'DELETE', url, headers: cookie ? { cookie } : {} }));
+        protectedRejections++;
+      }
+    }
+    observations.protectedRejections = protectedRejections;
+    assert.equal((await pool.query('SELECT count(*)::int AS count FROM monitors')).rows[0].count, 0);
+    assert.equal((await sessionApp.inject('/health')).statusCode, 200);
+    await mkdir('output/e04', { recursive: true });
+    await writeFile('output/e04/sessions.json', JSON.stringify({ ...observations, durationMs: Math.round(performance.now() - started) }, null, 2) + '\n');
+  } finally { await sessionApp.close(); await pool.end(); await dropTestSchema(config.schema); }
+});
diff --git a/test/database.ts b/test/database.ts
index 01562f8..1df6eeb 100644
--- a/test/database.ts
+++ b/test/database.ts
@@ -2,7 +2,7 @@ import { databaseConfig, databasePool, schemaIdentifier } from '../server/databa
 import { migrate } from '../server/migrate.ts';
 
 export function testDatabaseConfig(schema: string) {
-  if (!/^e03_[a-z_]+$/.test(schema)) throw new Error('Test cleanup is restricted to an explicit e03_ schema.');
+  if (!/^e0[34]_[a-z_]+$/.test(schema)) throw new Error('Test cleanup is restricted to an explicit e03_ or e04_ schema.');
   return { ...databaseConfig(), schema };
 }
 
diff --git a/test/functional.test.ts b/test/functional.test.ts
index daa27eb..bedc737 100644
--- a/test/functional.test.ts
+++ b/test/functional.test.ts
@@ -5,16 +5,21 @@ import { buildApp } from '../server/app.ts';
 import { checkMonitor, DEFAULT_FIXTURE_ORIGIN } from '../server/check.ts';
 import type { ApiSuccess, Monitor, MonitorView } from '../server/model.ts';
 import { fixtureServer } from './fixture.ts';
+import { authenticatedInject, loginForTest, prepareTestUsers } from './auth.ts';
 import { dropTestSchema, resetTestSchema, testDatabaseConfig } from './database.ts';
 
 const fixture = fixtureServer();
 const database = testDatabaseConfig('e03_functional');
 const app = buildApp(DEFAULT_FIXTURE_ORIGIN, database);
+let cookie: string;
+let inject: ReturnType<typeof authenticatedInject>;
 let guardCalls = 0;
 const guard = createServer((_request, response) => { guardCalls++; response.end('guard'); });
 
 before(async () => {
   await resetTestSchema(database.schema);
+  cookie = await loginForTest(app, (await prepareTestUsers(database))[0]);
+  inject = authenticatedInject(app, cookie);
   await new Promise<void>((resolve) => fixture.server.listen(4311, '127.0.0.1', resolve));
   await new Promise<void>((resolve) => guard.listen(4314, '127.0.0.1', resolve));
 });
@@ -30,7 +35,7 @@ after(async () => {
 });
 
 async function create(path: string): Promise<MonitorView> {
-  const response = await app.inject({ method: 'POST', url: '/monitors', payload: {
+  const response = await inject({ method: 'POST', url: '/monitors', payload: {
     name: 'Fixture monitor', url: `${DEFAULT_FIXTURE_ORIGIN}${path}`, interval: 60, enabled: true,
   } });
   assert.equal(response.statusCode, 201);
@@ -42,7 +47,7 @@ test('create a Monitor in PostgreSQL and synchronously observe GET /ok', async (
   assert.equal(monitor.interval, 60);
   assert.equal(monitor.enabled, true);
   assert.equal(monitor.latestCheck, null);
-  const response = await app.inject({ method: 'POST', url: `/monitors/${monitor.id}/checks` });
+  const response = await inject({ method: 'POST', url: `/monitors/${monitor.id}/checks` });
   assert.equal(response.statusCode, 200);
   const result = response.json().data;
   assert.equal(result.state, 'SUCCEEDED');
@@ -52,13 +57,13 @@ test('create a Monitor in PostgreSQL and synchronously observe GET /ok', async (
   assert.ok(result.latencyMs >= 0);
   assert.ok(Date.parse(result.finishedAt) >= Date.parse(result.startedAt));
   assert.equal(fixture.calls.get('/ok'), 1);
-  const list = (await app.inject('/monitors')).json<ApiSuccess<MonitorView[]>>().data;
+  const list = (await inject('/monitors')).json<ApiSuccess<MonitorView[]>>().data;
   assert.equal(list.find((item) => item.id === monitor.id)?.latestCheck?.id, result.id);
 });
 
 test('GET /fail is an observed endpoint failure with HTTP 503', async () => {
   const monitor = await create('/fail');
-  const result = (await app.inject({ method: 'POST', url: `/monitors/${monitor.id}/checks` })).json().data;
+  const result = (await inject({ method: 'POST', url: `/monitors/${monitor.id}/checks` })).json().data;
   assert.equal(result.state, 'FAILED');
   assert.equal(result.httpStatus, 503);
   assert.equal(result.failureReason, 'HTTP_STATUS');
@@ -68,7 +73,7 @@ test('GET /fail is an observed endpoint failure with HTTP 503', async () => {
 test('the check does not follow even a same-origin redirect', async () => {
   const previousOkCalls = fixture.calls.get('/ok');
   const monitor = await create('/redirect');
-  const result = (await app.inject({ method: 'POST', url: `/monitors/${monitor.id}/checks` })).json().data;
+  const result = (await inject({ method: 'POST', url: `/monitors/${monitor.id}/checks` })).json().data;
   assert.equal(result.state, 'FAILED');
   assert.equal(result.httpStatus, 302);
   assert.equal(fixture.calls.get('/ok'), previousOkCalls);
@@ -78,15 +83,15 @@ test('the check does not follow even a same-origin redirect', async () => {
 test('a non-fixture URL is rejected without contacting the controlled guard', async () => {
   const monitor = await create('/ok');
   const unsafe: Monitor = { ...monitor, url: 'http://127.0.0.1:4314/ok' };
-  const response = await app.inject({ method: 'POST', url: '/monitors', payload: unsafe });
+  const response = await inject({ method: 'POST', url: '/monitors', payload: unsafe });
   assert.equal(response.statusCode, 400);
   await assert.rejects(checkMonitor(unsafe, DEFAULT_FIXTURE_ORIGIN));
   assert.equal(guardCalls, 0);
 });
 
 test('another application instance reads the same persisted Monitors and latest checks', async () => {
-  const existing = (await app.inject('/monitors')).json();
+  const existing = (await inject('/monitors')).json();
   const fresh = buildApp(DEFAULT_FIXTURE_ORIGIN, database);
-  try { assert.deepEqual((await fresh.inject('/monitors')).json(), existing); }
+  try { assert.deepEqual((await fresh.inject({ url: '/monitors', headers: { cookie } })).json(), existing); }
   finally { await fresh.close(); }
 });
diff --git a/test/persistence.test.ts b/test/persistence.test.ts
index 7f9ed58..25791c1 100644
--- a/test/persistence.test.ts
+++ b/test/persistence.test.ts
@@ -9,6 +9,8 @@ import { monitorFromRow, checkRunFromRow, monitorToValues, checkRunToValues } fr
 import type { MonitorRow, CheckRunRow } from '../server/mapping.ts';
 import type { CheckRun, Monitor, MonitorView } from '../server/model.ts';
 import { fixtureServer } from './fixture.ts';
+import { authenticatedInject, loginForTest, prepareTestUsers } from './auth.ts';
+import authScenario from '../evidence/E04/scenario.json' with { type: 'json' };
 import { dropTestSchema, resetTestSchema } from './database.ts';
 import scenario from '../evidence/E03/scenario.json' with { type: 'json' };
 
@@ -30,10 +32,11 @@ async function failure(response: Response, status: number, code: string) {
   assert.deepEqual(Object.keys(body.error).sort(), ['code', 'message']);
   assert.equal(body.error.code, code);
 }
+let cookie: string;
 function request(path: string, method = 'GET', body?: unknown) {
   return fetch(`${scenario.apiOrigin}${path}`, {
-    method,
-    ...(body === undefined ? {} : { headers: { 'content-type': 'application/json' }, body: JSON.stringify(body) }),
+    method, headers: { cookie, ...(body === undefined ? {} : { 'content-type': 'application/json' }) },
+    ...(body === undefined ? {} : { body: JSON.stringify(body) }),
   });
 }
 function sortedById<T extends { id: string }>(values: T[]) {
@@ -47,9 +50,9 @@ test('E03 migration CLI applies the fresh chain and a repeated command changes n
   try {
     const options = { cwd: process.cwd(), env: { ...process.env, DATABASE_URL: config.connectionString, DATABASE_SCHEMA: config.schema }, timeout: 10_000 };
     const first = await execute(process.execPath, ['server/migrate.ts'], options);
-    assert.deepEqual(JSON.parse(first.stdout).applied, scenario.additionalAssertions.expectedMigrationFiles);
+    assert.deepEqual(JSON.parse(first.stdout).applied, authScenario.additionalAssertions.expectedMigrationFiles);
     const before = (await pool.query('SELECT version, checksum, applied_at FROM schema_migrations ORDER BY version')).rows;
-    assert.equal(before.length, 2);
+    assert.equal(before.length, authScenario.additionalAssertions.expectedMigrationFiles.length);
     for (const row of before) { assert.match(row.checksum, /^[0-9a-f]{64}$/); assert.ok(row.applied_at instanceof Date); }
     const second = await execute(process.execPath, ['server/migrate.ts'], options);
     assert.deepEqual(JSON.parse(second.stdout).applied, scenario.additionalAssertions.repeatMigrationExpectedApplied);
@@ -88,6 +91,7 @@ test('E03 A,A,B survive a fresh application; all Monitor lifecycle and CheckRun
   const fixture = fixtureServer();
   let app = buildApp(scenario.fixtureOrigin, config);
   try {
+    cookie = await loginForTest(app, (await prepareTestUsers(config))[0]);
     await new Promise<void>((resolve) => fixture.server.listen(4311, '127.0.0.1', resolve));
     await app.listen({ host: '127.0.0.1', port: 4312 });
     const monitors: MonitorView[] = [];
@@ -198,10 +202,11 @@ test('E03 canonical rows round-trip UUID, timezone, boolean, integer, zero and n
     const mappedCheck = checkRunFromRow(storedCheck);
     assert.deepEqual(mappedMonitor, { ...monitor, createdAt: input.expectedTimestamp, updatedAt: input.expectedTimestamp });
     assert.deepEqual(mappedCheck, { ...check, startedAt: input.expectedTimestamp, finishedAt: input.expectedTimestamp });
-    const response = await app.inject(`/monitors/${monitor.id}`);
+    const inject = authenticatedInject(app, await loginForTest(app, (await prepareTestUsers(config))[0]));
+    const response = await inject(`/monitors/${monitor.id}`);
     assert.equal(response.statusCode, 200);
     assert.deepEqual(response.json(), { data: { ...mappedMonitor, latestCheck: mappedCheck } });
-    assert.deepEqual((await app.inject(`/checks/${check.id}`)).json(), { data: mappedCheck });
+    assert.deepEqual((await inject(`/checks/${check.id}`)).json(), { data: mappedCheck });
     await record('mapping', { monitor: mappedMonitor, check: mappedCheck, sqlTimestampInput: input.timestamp, sqlTimestampRuntimeType: storedMonitor.created_at.constructor.name, durationMs: Math.round(performance.now() - started) });
   } finally { await app.close(); await pool.end(); await dropTestSchema(config.schema); }
 });
diff --git a/test/storage-contract.test.ts b/test/storage-contract.test.ts
index 0edd4f4..7cd8a0c 100644
--- a/test/storage-contract.test.ts
+++ b/test/storage-contract.test.ts
@@ -4,6 +4,7 @@ import { mkdir, writeFile } from 'node:fs/promises';
 import { buildApp } from '../server/app.ts';
 import { databasePool } from '../server/database.ts';
 import { resetTestSchema, dropTestSchema } from './database.ts';
+import { authenticatedInject, loginForTest, prepareTestUsers } from './auth.ts';
 import scenario from '../evidence/E03/supplemental-scenario.json' with { type: 'json' };
 
 test('E03 NUL name is INVALID_INPUT on create/update without PostgreSQL mutation', async () => {
@@ -13,18 +14,19 @@ test('E03 NUL name is INVALID_INPUT on create/update without PostgreSQL mutation
   const pool = databasePool(config);
   const app = buildApp(undefined, config);
   try {
-    const created = await app.inject({ method: 'POST', url: '/monitors', payload: input.input });
+    const inject = authenticatedInject(app, await loginForTest(app, (await prepareTestUsers(config))[0]));
+    const created = await inject({ method: 'POST', url: '/monitors', payload: input.input });
     assert.equal(created.statusCode, input.expectedStatus);
     assert.equal(created.json().error.code, input.expectedCode);
     const createdCount = (await pool.query('SELECT count(*)::int AS count FROM monitors')).rows[0].count;
     assert.equal(createdCount, input.expectedCreatedCount);
-    const valid = await app.inject({ method: 'POST', url: '/monitors', payload: scenario.extraRequiredColumn.probeInput });
+    const valid = await inject({ method: 'POST', url: '/monitors', payload: scenario.extraRequiredColumn.probeInput });
     assert.equal(valid.statusCode, 201);
     const monitor = valid.json().data;
-    const updated = await app.inject({ method: 'PUT', url: `/monitors/${monitor.id}`, payload: input.input });
+    const updated = await inject({ method: 'PUT', url: `/monitors/${monitor.id}`, payload: input.input });
     assert.equal(updated.statusCode, input.expectedStatus);
     assert.equal(updated.json().error.code, input.expectedCode);
-    assert.deepEqual((await app.inject(`/monitors/${monitor.id}`)).json().data, monitor);
+    assert.deepEqual((await inject(`/monitors/${monitor.id}`)).json().data, monitor);
     await mkdir('output/e03', { recursive: true });
     await writeFile('output/e03/supplemental-contract.json', JSON.stringify({ createStatus: created.statusCode, createBody: created.json(), createdCount, updateStatus: updated.statusCode, updateUnchanged: true, durationMs: Math.round(performance.now() - started) }, null, 2) + '\n');
   } finally { await app.close(); await pool.end(); await dropTestSchema(config.schema); }
diff --git a/test/storage-schema.test.ts b/test/storage-schema.test.ts
index 4924945..0054235 100644
--- a/test/storage-schema.test.ts
+++ b/test/storage-schema.test.ts
@@ -5,6 +5,7 @@ import { buildApp } from '../server/app.ts';
 import { databasePool } from '../server/database.ts';
 import { resetTestSchema, dropTestSchema } from './database.ts';
 import scenario from '../evidence/E03/supplemental-scenario.json' with { type: 'json' };
+import authScenario from '../evidence/E04/scenario.json' with { type: 'json' };
 
 test('E03 unexpected required Monitor column rejects startup before accepting valid requests', async () => {
   const started = performance.now();
@@ -26,3 +27,21 @@ test('E03 unexpected required Monitor column rejects startup before accepting va
     await writeFile('output/e03/supplemental-schema.json', JSON.stringify({ startupRejected: true, listening: false, message, durationMs: Math.round(performance.now() - started) }, null, 2) + '\n');
   } finally { await app.close(); await pool.end(); await dropTestSchema(config.schema); }
 });
+
+test('E04 authentication columns, nullability, types, precision, keys and user relation reject schema drift', async () => {
+  const started = performance.now();
+  const results: { mutation: string; startupRejected: boolean; listening: boolean }[] = [];
+  for (const mutation of authScenario.additionalAssertions.schemaMutations) {
+    const config = await resetTestSchema(`e04_${mutation.name}`);
+    const pool = databasePool(config);
+    const app = buildApp(undefined, config);
+    try {
+      await pool.query(mutation.sql);
+      await assert.rejects(app.listen({ host: '127.0.0.1', port: 4312 }), /Incompatible database schema/);
+      assert.equal(app.server.listening, false);
+      results.push({ mutation: mutation.name, startupRejected: true, listening: false });
+    } finally { await app.close(); await pool.end(); await dropTestSchema(config.schema); }
+  }
+  await mkdir('output/e04', { recursive: true });
+  await writeFile('output/e04/schema.json', JSON.stringify({ results, count: results.length, durationMs: Math.round(performance.now() - started) }, null, 2) + '\n');
+});
diff --git a/test/unit.test.ts b/test/unit.test.ts
index f28eb85..cddca38 100644
--- a/test/unit.test.ts
+++ b/test/unit.test.ts
@@ -1,9 +1,14 @@
 import test from 'node:test';
 import assert from 'node:assert/strict';
+import { randomBytes } from 'node:crypto';
 import { fixtureUrl, DEFAULT_FIXTURE_ORIGIN } from '../server/check.ts';
 import { ERROR_MESSAGES, failureCode, RequestFailure, responseData } from '../app/monitors/api.ts';
 import type { ApiErrorCode } from '../server/model.ts';
 import scenario from '../evidence/E02/scenario.json' with { type: 'json' };
+import authScenario from '../evidence/E04/scenario.json' with { type: 'json' };
+import { hashPassword, verifyPassword, SCRYPT_OPTIONS } from '../server/password.ts';
+import { loginInput } from '../server/contracts.ts';
+import { SESSION_COOKIE_NAME, SESSION_TTL_MS, sessionTokenFromCookie } from '../server/auth.ts';
 
 test('a path on the configured fixture origin is allowed', () => {
   assert.equal(fixtureUrl(`${DEFAULT_FIXTURE_ORIGIN}/ok`, DEFAULT_FIXTURE_ORIGIN).pathname, '/ok');
@@ -53,3 +58,47 @@ test('browser transport rejection has a stable fallback independent of exception
   assert.equal(failureCode(new TypeError(scenario.browserNetworkFailure)), 'INTERNAL_ERROR');
   assert.equal(failureCode(new Error(scenario.additionalBoundaries.privateInternalMessage)), 'INTERNAL_ERROR');
 });
+
+test('E04 salted scrypt hashes verify passwords without storing their plaintext', async () => {
+  const password = randomBytes(32).toString('base64url');
+  const wrong = randomBytes(32).toString('base64url');
+  const first = await hashPassword(password);
+  const second = await hashPassword(password);
+  const expected = authScenario.passwordHash;
+  assert.deepEqual(SCRYPT_OPTIONS, { N: expected.N, r: expected.r, p: expected.p, maxmem: expected.maxmem });
+  assert.ok(/^scrypt\$131072\$8\$1\$[0-9a-f]{32}\$[0-9a-f]{128}$/.test(first));
+  assert.ok(first !== password && !first.includes(password));
+  assert.ok(first !== second, 'Independent salts must produce different representations.');
+  assert.equal(await verifyPassword(password, first), true);
+  assert.equal(await verifyPassword(wrong, first), false);
+  await assert.rejects(verifyPassword(password, 'unsupported-representation'), /Invalid stored password representation/);
+});
+
+test('E04 login input is bounded at runtime and never trims or coerces a password', () => {
+  const password = randomBytes(32).toString('base64url');
+  const input = { username: authScenario.users[0], password: ` ${password} ` };
+  assert.ok(loginInput(input).password === input.password);
+  for (const invalid of [null, [], {}, { ...input, username: 123 }, { ...input, username: 'A' },
+    { ...input, password: 123 }, { ...input, password: '' }, { ...input, password: 'x'.repeat(1025) }]) {
+    assert.throws(() => loginInput(invalid), { name: 'ApiError', code: 'INVALID_INPUT' });
+  }
+});
+
+test('E04 a single bounded cookie identifier is required, including duplicate and malformed rejection', () => {
+  const token = randomBytes(authScenario.session.identifierBytes).toString('base64url');
+  const cookie = `${SESSION_COOKIE_NAME}=${token}`;
+  assert.equal(SESSION_TTL_MS, authScenario.session.ttlMs);
+  assert.ok(sessionTokenFromCookie(cookie) === token);
+  assert.ok(sessionTokenFromCookie(`unrelated=value; ${cookie}`) === token);
+  for (const input of [undefined, '', `${SESSION_COOKIE_NAME}=`, `${cookie}; ${cookie}`,
+    `${SESSION_COOKIE_NAME}=invalid`, `${SESSION_COOKIE_NAME}=${encodeURIComponent(' '.repeat(43))}`]) {
+    assert.equal(sessionTokenFromCookie(input), null);
+  }
+});
+
+test('E04 browser recognizes UNAUTHENTICATED only with HTTP 401 and ignores server prose', async () => {
+  await assert.rejects(responseData(Response.json({ error: { code: 'UNAUTHENTICATED', message: 'ignored' } }, { status: 401 })),
+    (error: unknown) => failureCode(error) === 'UNAUTHENTICATED');
+  await assert.rejects(responseData(Response.json({ error: { code: 'UNAUTHENTICATED', message: 'ignored' } }, { status: 400 })),
+    (error: unknown) => failureCode(error) === 'INTERNAL_ERROR');
+});


