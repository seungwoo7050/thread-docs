# E04 세션 인증과 수명주기

## `Freeze the E04 anonymous session counterexample`

diff --git a/evidence/E04/baseline-reproducer.mjs b/evidence/E04/baseline-reproducer.mjs
new file mode 100644
index 0000000..dcb8f42
--- /dev/null
+++ b/evidence/E04/baseline-reproducer.mjs
@@ -0,0 +1,23 @@
+// Run from the worktree root at the frozen START, before changing application code.
+import assert from 'node:assert/strict';
+import { writeFile } from 'node:fs/promises';
+import { buildApp } from '../../server/app.ts';
+import { resetTestSchema, dropTestSchema } from '../../test/database.ts';
+import scenario from './scenario.json' with { type: 'json' };
+
+const started = performance.now();
+const config = await resetTestSchema(scenario.baseline.schema);
+const app = buildApp(scenario.fixtureOrigin, config);
+try {
+  await app.listen({ host: '127.0.0.1', port: 4312 });
+  const response = await fetch(`${scenario.apiOrigin}${scenario.baseline.path}`);
+  const evidence = {
+    start: scenario.start, codeUnmodified: true, requests: 1, cookiePresent: false,
+    observedStatus: response.status, requiredStatus: scenario.baseline.requiredStatus,
+    result: response.status === 200 ? 'REPRODUCED' : 'NOT_REPRODUCED',
+    durationMs: Math.round(performance.now() - started),
+  };
+  await writeFile(new URL('./baseline.json', import.meta.url), JSON.stringify(evidence, null, 2) + '\n');
+  console.log(JSON.stringify(evidence));
+  assert.equal(response.status, 200, 'The unchanged START permits the anonymous collection request.');
+} finally { await app.close(); await dropTestSchema(config.schema); }
diff --git a/evidence/E04/baseline.json b/evidence/E04/baseline.json
new file mode 100644
index 0000000..ff2ec3f
--- /dev/null
+++ b/evidence/E04/baseline.json
@@ -0,0 +1,10 @@
+{
+  "start": "a2b51405f406a86b7d81b09c00905a25be05c8ec",
+  "codeUnmodified": true,
+  "requests": 1,
+  "cookiePresent": false,
+  "observedStatus": 200,
+  "requiredStatus": 401,
+  "result": "REPRODUCED",
+  "durationMs": 132
+}
diff --git a/evidence/E04/scenario.json b/evidence/E04/scenario.json
new file mode 100644
index 0000000..b23c7bf
--- /dev/null
+++ b/evidence/E04/scenario.json
@@ -0,0 +1,79 @@
+{
+  "thread": "E04",
+  "attempt": 1,
+  "specRevision": "0a006589477f8ae47bad3faa5510c999cff85ee4",
+  "start": "a2b51405f406a86b7d81b09c00905a25be05c8ec",
+  "frozenBeforeBaseline": true,
+  "fixtureOrigin": "http://127.0.0.1:4311",
+  "apiOrigin": "http://127.0.0.1:4312",
+  "browserOrigin": "http://127.0.0.1:4313",
+  "baseline": {
+    "schema": "e03_auth_baseline",
+    "method": "GET",
+    "path": "/monitors",
+    "cookie": "absent",
+    "requests": 1,
+    "requiredStatus": 401,
+    "requiredCode": "UNAUTHENTICATED"
+  },
+  "users": ["alice-e04", "bob-e04"],
+  "unknownUsername": "absent-e04",
+  "passwordGeneration": "Independent randomBytes(32).toString('base64url') at runtime for each user and wrong-password input; never printed or committed",
+  "passwordHash": { "algorithm": "scrypt", "N": 131072, "r": 8, "p": 1, "maxmem": 268435456, "saltBytes": 16, "keyBytes": 64 },
+  "session": {
+    "identifierBytes": 32,
+    "ttlMs": 3600000,
+    "clockStart": "2026-01-01T00:00:00.000Z",
+    "validCheckpointMs": 3599999,
+    "expiredCheckpointMs": 3600000,
+    "expiryControl": "Injected application clock, exactly expiresAt; no sleeps or TTL changes",
+    "cookie": { "name": "wse_session", "httpOnly": true, "sameSite": "Lax", "secure": false, "path": "/", "hostOnly": true, "maxAgeSeconds": 3600 }
+  },
+  "sequence": [
+    "prepare two users",
+    "anonymous collection returns 401 UNAUTHENTICATED",
+    "Alice valid login issues a session cookie",
+    "Alice wrong password returns 401 UNAUTHENTICATED without account existence detail",
+    "authenticated collection returns 200",
+    "successful login carrying the prior cookie changes the identifier and invalidates the prior identifier",
+    "logout invalidates the rotated cookie",
+    "a fresh session is valid at expiresAt minus 1ms and rejected exactly at expiresAt",
+    "Bob independently logs in"
+  ],
+  "additionalAssertions": {
+    "schema": "e04_sessions",
+    "missingId": "00000000-0000-4000-8000-000000000000",
+    "protectedRequests": [
+      ["GET", "/monitors"],
+      ["POST", "/monitors"],
+      ["GET", "/monitors/00000000-0000-4000-8000-000000000000"],
+      ["PUT", "/monitors/00000000-0000-4000-8000-000000000000"],
+      ["DELETE", "/monitors/00000000-0000-4000-8000-000000000000"],
+      ["GET", "/monitors/00000000-0000-4000-8000-000000000000/checks"],
+      ["POST", "/monitors/00000000-0000-4000-8000-000000000000/checks"],
+      ["GET", "/checks/00000000-0000-4000-8000-000000000000"],
+      ["GET", "/auth/session"],
+      ["POST", "/auth/logout"]
+    ],
+    "expectedMigrationFiles": ["001_monitors.sql", "002_check_runs.sql", "003_sessions.sql"],
+    "schemaMutations": [
+      { "name": "missing_password_hash", "sql": "ALTER TABLE users DROP COLUMN password_hash" },
+      { "name": "nullable_username", "sql": "ALTER TABLE users ALTER COLUMN username DROP NOT NULL" },
+      { "name": "missing_user_key", "sql": "ALTER TABLE users DROP CONSTRAINT users_pkey CASCADE" },
+      { "name": "missing_unique_username", "sql": "ALTER TABLE users DROP CONSTRAINT users_username_key" },
+      { "name": "unexpected_user_column", "sql": "ALTER TABLE users ADD COLUMN e04_required_extra text NOT NULL" },
+      { "name": "nullable_expiry", "sql": "ALTER TABLE sessions ALTER COLUMN expires_at DROP NOT NULL" },
+      { "name": "wrong_expiry_type", "sql": "ALTER TABLE sessions ALTER COLUMN expires_at TYPE timestamp(3) without time zone" },
+      { "name": "wrong_expiry_precision", "sql": "ALTER TABLE sessions ALTER COLUMN expires_at TYPE timestamptz(6)" },
+      { "name": "missing_session_key", "sql": "ALTER TABLE sessions DROP CONSTRAINT sessions_pkey" },
+      { "name": "missing_session_parent", "sql": "ALTER TABLE sessions DROP CONSTRAINT sessions_user_id_fkey" }
+    ],
+    "browser": "anonymous redirect; Alice wrong then valid login; HttpOnly cookie invisible to document.cookie; reload; logout; old-cookie API rejection; Bob login; empty browser credential storage",
+    "regression": "E01-E03 fixed Monitor, contract, persistence and lifecycle inputs remain unchanged; only authenticated setup and current migration expectation extend"
+  },
+  "evidencePolicy": "Record statuses, safe cookie flags, rotation/revocation booleans, password representation booleans, counts and durations only; no password, password hash, session identifier or CSRF value",
+  "retries": 0,
+  "loadRuns": 0,
+  "benchmarkRuns": 0,
+  "parameterSweeps": 0
+}


