# E05 Monitor 소유권, IDOR와 브라우저 상태 변경 보호

## `test(authz): freeze the two-user IDOR counterexample`

diff --git a/evidence/E05/baseline.json b/evidence/E05/baseline.json
new file mode 100644
index 0000000..224d3cb
--- /dev/null
+++ b/evidence/E05/baseline.json
@@ -0,0 +1,73 @@
+{
+  "start": "c59551aeb8e5a351b7d875489578f31caea6f160",
+  "fixtureSha256": "03cd149112e7bed3613c2dab85c6f5c46d82e5037bef1f638abb0111f40e237d",
+  "startedAt": "2026-08-28T01:12:35.061Z",
+  "requests": [
+    {
+      "actor": "alice-e04",
+      "path": "/api/monitors",
+      "method": "POST",
+      "status": 201
+    },
+    {
+      "actor": "bob-e04",
+      "path": "/api/monitors",
+      "method": "POST",
+      "status": 201
+    },
+    {
+      "actor": "alice-e04",
+      "path": "/api/monitors/ef57cbb1-76b8-4e6d-b5dd-09992a026c85/checks",
+      "method": "POST",
+      "status": 200
+    },
+    {
+      "actor": "bob-e04",
+      "path": "/api/monitors/41ff9f8a-56f6-48f2-9c1f-b854a9f54b7b/checks",
+      "method": "POST",
+      "status": 200
+    },
+    {
+      "actor": "bob-e04",
+      "path": "/api/monitors/ef57cbb1-76b8-4e6d-b5dd-09992a026c85",
+      "method": "GET",
+      "status": 200
+    }
+  ],
+  "processes": [
+    {
+      "role": "fixture",
+      "pid": 21240,
+      "exitCode": 0,
+      "signal": null,
+      "exitAwaited": true
+    },
+    {
+      "role": "api",
+      "pid": 21247,
+      "exitCode": 143,
+      "signal": null,
+      "exitAwaited": true
+    }
+  ],
+  "dataset": {
+    "users": 2,
+    "monitors": 2,
+    "checks": 2,
+    "aliceMonitor": "ef57cbb1-76b8-4e6d-b5dd-09992a026c85",
+    "bobMonitor": "41ff9f8a-56f6-48f2-9c1f-b854a9f54b7b",
+    "aliceCheck": "485843ce-1cb2-470c-aa30-b5757aaa9115",
+    "bobCheck": "15ba1854-159f-41f0-ae55-3598f1d10190"
+  },
+  "counterexample": {
+    "requests": 1,
+    "status": 200,
+    "category": null,
+    "aliceIdentifierDisclosed": true,
+    "requiredStatus": 404,
+    "requiredCategory": "NOT_FOUND"
+  },
+  "result": "REPRODUCED",
+  "schemaDropped": true,
+  "finishedAt": "2026-08-28T01:12:43.571Z"
+}
diff --git a/evidence/E05/fixtures.md b/evidence/E05/fixtures.md
new file mode 100644
index 0000000..678e72b
--- /dev/null
+++ b/evidence/E05/fixtures.md
@@ -0,0 +1,51 @@
+# E05 frozen authorization scenario
+
+Frozen before any E05 product change or baseline request.
+Spec revision: `0a006589477f8ae47bad3faa5510c999cff85ee4`.
+Unchanged START: `c59551aeb8e5a351b7d875489578f31caea6f160`.
+Branch: `track/industry-spring`; attempt 1.
+
+## Baseline (exactly once)
+
+- Use only isolated `wse-industry` PostgreSQL on 15432 and schema `e05_baseline`.
+- Refuse occupied fixture 4321 and API 4322; await every owned child exit.
+- Bootstrap `alice-e04` and `bob-e04` using independent random runtime-only passwords.
+- Alice logs in and creates `Monitor A`, `http://127.0.0.1:4321/ok`, interval 60, enabled true.
+- Bob logs in and creates `Monitor B`, `http://127.0.0.1:4321/fail`, interval 120, enabled true.
+- Each user runs exactly one manual Check of their own Monitor.
+- Use existing session CSRF bootstrap for login and mutations; send Origin `http://127.0.0.1:4323`.
+- Bob performs exactly one GET of Alice's Monitor A identifier at unchanged START.
+- Record status/category and whether Alice's identifier was disclosed, not credential values.
+- Readiness deadline 30 seconds; request timeout 5 seconds; shutdown grace 5 seconds.
+- Do not rerun baseline or alter data/parameters based on its result.
+
+## Fixed post-change matrix
+
+Recreate the same two users, two Monitors and two CheckRuns. Authenticate every
+authorization probe and use valid CSRF and trusted Origin for mutation probes.
+
+1. Anonymous protected reads/mutations and anonymous-CSRF logout: 401 UNAUTHENTICATED.
+2. Each collection contains only the caller's Monitor; owner detail/latest/history/single CheckRun reads succeed.
+3. Foreign Monitor/detail/history/single CheckRun reads, foreign CheckRun under an owned Monitor, and fixed nonexistent UUID `00000000-0000-0000-0000-000000000000`: 404 NOT_FOUND with the same public body.
+4. Foreign and nonexistent PUT edit, pause, resume, DELETE and POST manual Check: 404 NOT_FOUND. After each request compare authoritative Monitor/CheckRun rows and outbound fixture request count against the before snapshot.
+5. Owner create/edit/pause/resume/manual Check/history/delete succeed with valid session CSRF and exact Origin `http://127.0.0.1:4323`. Deletion cascades only that owner's history.
+6. For authenticated POST create/check, PUT edit/pause/resume, DELETE and logout, missing/incorrect/cross-session CSRF and missing/null/foreign (`http://127.0.0.1:4999`) Origin: 403 FORBIDDEN with no authoritative mutation/outbound call; denied logout leaves the session authenticated.
+7. Login requires an anonymous session CSRF token and exact trusted Origin. Invalid login evidence does not authenticate or rotate the session. Authenticated invalid evidence is 403; absent authentication remains 401.
+8. Cross-origin credentialed preflight/request never receives `Access-Control-Allow-Origin` or `Access-Control-Allow-Credentials`. CORS and state-change validation are tested independently.
+9. Real isolated Alice/Bob browser contexts repeat collection/detail/history isolation, authorized lifecycle and logout. No traces, videos, screenshots of credentials or recorded network/session payloads.
+10. Successful logout invalidates the old cookie; CSRF material alone never authenticates. Existing exact one-hour expiry and session rotation/revocation regressions remain unchanged.
+
+## Ownership migration and framework evidence
+
+- Preserve V1/V2/V3 bytes. Fresh V4 startup must enforce a non-null owner foreign key.
+- Previous V3 fixture: two users, one existing Monitor and one historical CheckRun. Initial ownership migration refuses atomically while rows/history remain unchanged and no incomplete owner column remains.
+- An operator must explicitly designate a verified existing user for each existing Monitor while the application is stopped. Never choose the first user. Test unknown/unassigned owner refusal and successful explicit assignment; preserve historical identifiers and canonical data through V4 upgrade and repeated startup.
+- Inspect Hibernate-generated SQL for owner predicates and actual transaction state during SQL; verify public calls cross the Spring proxy, private helpers join those transactions, and outbound HTTP holds no transaction.
+- Preserve prior product fixtures and assertions; adapt only authenticated owner/Origin setup as required.
+
+## Budget and scope
+
+Zero load runs, automatic retries or parameter sweeps. One unchanged START baseline.
+Only deterministic contract/integration/browser/process verification. Record each
+gate invocation, including failures; stop once required gates pass. No E06+, no
+new runtime/dependency, no other track/main/spec/index/tags/history changes.


