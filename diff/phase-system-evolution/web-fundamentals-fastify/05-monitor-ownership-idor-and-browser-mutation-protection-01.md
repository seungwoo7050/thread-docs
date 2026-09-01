# E05 Monitor 소유권, IDOR와 브라우저 상태 변경 보호

## `Freeze the E05 cross-user Monitor disclosure`

diff --git a/evidence/E05/baseline-reproducer.mjs b/evidence/E05/baseline-reproducer.mjs
new file mode 100644
index 0000000..286ef68
--- /dev/null
+++ b/evidence/E05/baseline-reproducer.mjs
@@ -0,0 +1,67 @@
+// Execute exactly once at the frozen START before modifying any application/test code.
+import assert from 'node:assert/strict';
+import { createHash } from 'node:crypto';
+import { execFileSync } from 'node:child_process';
+import { readFile, writeFile } from 'node:fs/promises';
+import { buildApp } from '../../server/app.ts';
+import { databaseConfig, databasePool, schemaIdentifier } from '../../server/database.ts';
+import { migrate } from '../../server/migrate.ts';
+import { prepareTestUsers, cookieFromHeader } from '../../test/auth.ts';
+import { fixtureServer } from '../../test/fixture.ts';
+import scenario from './scenario.json' with { type: 'json' };
+
+assert.equal(execFileSync('git', ['rev-parse', 'HEAD'], { encoding: 'utf8' }).trim(), scenario.start);
+assert.equal(execFileSync('git', ['diff', '--name-only', 'HEAD', '--', 'server', 'test', 'app'], { encoding: 'utf8' }).trim(), '');
+const scenarioSha256 = createHash('sha256').update(await readFile(new URL('./scenario.json', import.meta.url))).digest('hex');
+const config = { ...databaseConfig(), schema: scenario.baseline.schema };
+assert.match(config.schema, /^e05_[a-z_]+$/);
+const admin = databasePool(config);
+const fixture = fixtureServer();
+const app = buildApp(scenario.fixtureOrigin, config);
+const started = performance.now();
+try {
+  // Binding refuses occupied ports before any HTTP request is sent.
+  await new Promise((resolve, reject) => { fixture.server.once('error', reject); fixture.server.listen(4311, '127.0.0.1', resolve); });
+  await admin.query(`CREATE SCHEMA ${schemaIdentifier(config.schema)}`);
+  await migrate(config);
+  await app.listen({ host: '127.0.0.1', port: 4312 });
+  const users = await prepareTestUsers(config);
+  const cookies = [];
+  const monitors = [];
+  for (let index = 0; index < users.length; index++) {
+    const login = await fetch(`${scenario.apiOrigin}/auth/login`, {
+      method: 'POST', headers: { 'content-type': 'application/json', origin: scenario.browserOrigin }, body: JSON.stringify(users[index]),
+    });
+    assert.equal(login.status, 200);
+    const cookie = cookieFromHeader(login.headers.get('set-cookie'));
+    cookies.push(cookie);
+    const created = await fetch(`${scenario.apiOrigin}/monitors`, {
+      method: 'POST', headers: { 'content-type': 'application/json', cookie }, body: JSON.stringify(scenario.monitors[index]),
+    });
+    assert.equal(created.status, 201);
+    const monitor = (await created.json()).data;
+    monitors.push(monitor);
+    const checked = await fetch(`${scenario.apiOrigin}/monitors/${monitor.id}/checks`, { method: 'POST', headers: { cookie } });
+    assert.equal(checked.status, 200);
+  }
+  const response = await fetch(`${scenario.apiOrigin}/monitors/${monitors[0].id}`, { headers: { cookie: cookies[1] } });
+  const body = await response.json();
+  const evidence = {
+    start: scenario.start, scenarioSha256, codeUnmodified: true,
+    setupMonitors: 2, setupCheckRuns: 2, ownOutboundCalls: Object.fromEntries(fixture.calls),
+    foreignReadRequests: 1, observedStatus: response.status,
+    foreignMonitorDisclosed: body.data?.id === monitors[0].id,
+    requiredStatus: scenario.baseline.requiredStatus, requiredCode: scenario.baseline.requiredCode,
+    result: response.status === 200 ? 'REPRODUCED' : 'NOT_REPRODUCED',
+    durationMs: Math.round(performance.now() - started),
+  };
+  await writeFile(new URL('./baseline.json', import.meta.url), JSON.stringify(evidence, null, 2) + '\n');
+  console.log(JSON.stringify(evidence));
+  assert.equal(response.status, 200, 'Unchanged START exposes Alice A to Bob.');
+} finally {
+  await app.close();
+  fixture.server.closeAllConnections();
+  await new Promise((resolve) => fixture.server.close(resolve));
+  await admin.query(`DROP SCHEMA IF EXISTS ${schemaIdentifier(config.schema)} CASCADE`);
+  await admin.end();
+}
diff --git a/evidence/E05/baseline.json b/evidence/E05/baseline.json
new file mode 100644
index 0000000..b3174fb
--- /dev/null
+++ b/evidence/E05/baseline.json
@@ -0,0 +1,18 @@
+{
+  "start": "c1bace0a0e2de598f11fc9f1fac32d459fc910a6",
+  "scenarioSha256": "a429c0c70afa8365656d64e54d158fcb51fd9a5d8dd6905b6ea5d58db73ac95f",
+  "codeUnmodified": true,
+  "setupMonitors": 2,
+  "setupCheckRuns": 2,
+  "ownOutboundCalls": {
+    "/ok": 1,
+    "/fail": 1
+  },
+  "foreignReadRequests": 1,
+  "observedStatus": 200,
+  "foreignMonitorDisclosed": true,
+  "requiredStatus": 404,
+  "requiredCode": "NOT_FOUND",
+  "result": "REPRODUCED",
+  "durationMs": 1913
+}
diff --git a/evidence/E05/scenario.json b/evidence/E05/scenario.json
new file mode 100644
index 0000000..7f0cb67
--- /dev/null
+++ b/evidence/E05/scenario.json
@@ -0,0 +1,52 @@
+{
+  "thread": "E05",
+  "attempt": 1,
+  "specRevision": "0a006589477f8ae47bad3faa5510c999cff85ee4",
+  "start": "c1bace0a0e2de598f11fc9f1fac32d459fc910a6",
+  "frozenBeforeBaseline": true,
+  "fixtureOrigin": "http://127.0.0.1:4311",
+  "apiOrigin": "http://127.0.0.1:4312",
+  "browserOrigin": "http://127.0.0.1:4313",
+  "foreignOrigin": "http://127.0.0.1:4999",
+  "guardPort": 4314,
+  "users": ["alice-e04", "bob-e04"],
+  "monitors": [
+    { "name": "Alice A", "url": "http://127.0.0.1:4311/ok", "interval": 60, "enabled": true },
+    { "name": "Bob B", "url": "http://127.0.0.1:4311/fail", "interval": 120, "enabled": true }
+  ],
+  "setup": "Prepare two runtime-only accounts; Alice creates A and runs one own check; Bob creates B and runs one own check.",
+  "baseline": {
+    "schema": "e05_ownership_baseline",
+    "method": "GET",
+    "path": "/monitors/<Alice A id>",
+    "session": "Bob",
+    "requests": 1,
+    "requiredStatus": 404,
+    "requiredCode": "NOT_FOUND",
+    "stop": "Stop after the first decisive Bob read; do not change START code or fixtures."
+  },
+  "missingId": "00000000-0000-4000-8000-000000000000",
+  "authorizationMatrix": [
+    "Anonymous: collection, create, detail, update, delete, nested history, manual check, direct CheckRun, session and logout -> 401 UNAUTHENTICATED.",
+    "Each owner's collection contains only its own Monitor and latest CheckRun; owner detail, history and direct CheckRun read -> 200.",
+    "Foreign and absent Monitor detail, history and direct CheckRun -> identical 404 NOT_FOUND categories and per-route bodies.",
+    "Foreign update, pause, resume, delete and manual check with valid CSRF -> 404 NOT_FOUND; all Monitor/CheckRun rows and outbound call counts unchanged after each request.",
+    "Owner create, update, pause, resume, manual check and delete with valid CSRF and allowed Origin succeed.",
+    "For every mutation including logout: authenticated missing CSRF, incorrect CSRF, another session's CSRF and foreign Origin -> 403 FORBIDDEN; rows, live session and outbound call counts unchanged.",
+    "CSRF alone cannot authenticate; session rotation and revocation invalidate old credentials.",
+    "Login rejects missing, null and foreign Origin; allowed Origin succeeds. Logout requires allowed Origin and session-bound CSRF.",
+    "Preflight from foreign Origin and credentialed cross-origin reads have no Access-Control-Allow-Origin or Access-Control-Allow-Credentials permission."
+  ],
+  "mutationInputs": {
+    "update": { "name": "Alice A edited", "url": "http://127.0.0.1:4311/ok", "interval": 60, "enabled": true },
+    "pause": { "name": "Alice A", "url": "http://127.0.0.1:4311/ok", "interval": 60, "enabled": false },
+    "resume": { "name": "Alice A", "url": "http://127.0.0.1:4311/ok", "interval": 60, "enabled": true }
+  },
+  "migration": "Append 004 only. Fresh schema succeeds. Existing unowned rows require an explicit operator owner designation; absent/invalid designation refuses atomically. Correct designation preserves every Monitor and CheckRun and creates non-null user ownership. Reject missing/wrong/nullable owner column and missing owner FK before listen.",
+  "browser": "Real authorized create/edit/pause/resume/check/history/delete/login/logout regressions; independent Alice and Bob contexts demonstrate isolation; browser requests carry CSRF and Origin but never log their values.",
+  "credentialPolicy": "Generate passwords, session and CSRF values only at runtime. Never print, write or commit values, password hashes, request traces, videos or credential screenshots.",
+  "previousScenarios": "E01-E04 product inputs and evidence remain unchanged; only auth/CSRF setup and latest migration expectations may extend.",
+  "loadRuns": 0,
+  "automaticRetries": 0,
+  "parameterSweeps": 0
+}


