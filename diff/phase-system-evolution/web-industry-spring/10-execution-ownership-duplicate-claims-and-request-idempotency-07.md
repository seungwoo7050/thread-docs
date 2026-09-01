## `docs(execution): finish E10 guide and author verification`

diff --git a/TRACK.md b/TRACK.md
index a80303c..5a1ebce 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -4,7 +4,7 @@ Profile: `phase-1` (E01–E12 → E20 → profile E24; phase-2 deferred, E25 not
 
 Spec revision: `2ada57a71cd34fa2fae9809415c362a8bbfcdf02`
 
-E09 is a local development product: Next.js/React renders login, logout, Monitor lifecycle and Check history from request-scoped server data. Spring Security authenticates browser sessions; PostgreSQL stores salted account passwords, Monitors and Check execution intents/results. The API accepts a manual Check with202/QUEUED, and a separate worker executes it. Monitor and CheckRun access is scoped to the signed-in user. There is no signup, Redis, broker or production application container.
+E10 is a local development product: Next.js/React renders login, logout, Monitor lifecycle and Check history from request-scoped server data. Spring Security authenticates browser sessions; PostgreSQL stores salted account passwords, Monitors and Check execution intents/results. Manual requests have durable owner-scoped identities, and competing workers claim one execution owner before I/O. Monitor and CheckRun access is scoped to the signed-in user. There is no signup, Redis, broker or production application container.
 
 ## Pinned toolchain
 
@@ -80,8 +80,8 @@ The default connection is database `monitor`, local test identity `wse_industry`
 - Names are stripped of surrounding whitespace and must contain 1–100 UTF-16 code units. Interval is 1–86400 seconds inclusive; the JSON number `60.0` is the integer value 60, but the string `"60"` is invalid.
 - E03 also rejects a NUL character in a name before persistence, because PostgreSQL text cannot store it. Create and replacement use the same runtime validator, including the existing non-finite numeric rejection.
 - URL syntax must be absolute HTTP(S). The existing fixture policy then restricts it to the configured HTTP hostname and port, without credentials or fragments. This does not enable general public or HTTPS monitoring.
-- Successful responses contain exactly `{ "data": <payload> }`. Create returns201; Check acceptance returns202/QUEUED; reads return200. The existing CheckRun fields remain, with null execution/result fields before they have been observed.
-- API failures contain `{ "error": { "code": "...", "message": "..." } }`: INVALID_INPUT / 400, UNAUTHENTICATED / 401, FORBIDDEN / 403, NOT_FOUND / 404, or INTERNAL_ERROR / 500. Malformed JSON and IDs use INVALID_INPUT after authentication. Unexpected exception details are not returned.
+- Successful responses contain exactly `{ "data": <payload> }`. Create returns201; a new Check acceptance returns202/QUEUED; retransmission returns202 with that same execution's current state; reads return200. The existing CheckRun fields remain, with null execution/result fields before they have been observed.
+- API failures contain `{ "error": { "code": "...", "message": "..." } }`: INVALID_INPUT / 400, UNAUTHENTICATED / 401, FORBIDDEN / 403, NOT_FOUND / 404, CONFLICT / 409, or INTERNAL_ERROR / 500. Malformed JSON and IDs use INVALID_INPUT after authentication. Unexpected exception details are not returned.
 - Endpoint HTTP 503 is still an API success containing a FAILED CheckRun with HTTP_STATUS; timeout and connection failure retain null httpStatus. API errors never fabricate a CheckRun.
 - The browser validates the envelope and the displayed payload shape, selects errors by the stable code/status pair, and does not classify or display arbitrary server prose. Network or malformed responses use the INTERNAL_ERROR UI fallback without applying the mutation.
 
@@ -326,8 +326,8 @@ creation. A late tick creates only the current due slot, without historical
 catch-up. Repeating a slot is ignored, with a PostgreSQL unique scheduled-slot
 index as the durable guard. Disabled Monitors receive no scheduled intent.
 Distinct manual and scheduled intents may overlap in the queue; no Monitor-wide
-exclusion is imposed. The single E09 worker executes one at a time. Competing
-claims and recovery of a worker that stops while RUNNING are not implemented here.
+exclusion is imposed. E10 adds competing-worker claims below. Recovery of a worker
+that stops while RUNNING is not implemented.
 
 `npm run test:worker` owns an initially worker-free API and explicitly starts one
 separate JVM after the202 checkpoint. `npm run test:e2e` runs the earlier browser
@@ -337,3 +337,36 @@ uses the actual transaction at T0,T0,T+60. Neither harness mocks terminal result
 The E05 SQL audit retains both user and worker statements, classifying only its
 explicit worker setup scope separately; every user/store owner predicate and
 denied mutation assertion remains intact.
+
+## Request identity and execution ownership (E10)
+
+Manual Check POSTs require `Idempotency-Key`:1–128 printable ASCII characters,
+without whitespace. Missing or invalid values return400 / INVALID_INPUT. The
+authenticated user UUID and key identify one durable CheckRun. A concurrent
+insert uses PostgreSQL's unique identity constraint and `ON CONFLICT DO NOTHING`,
+then reads the current row. The semantic target is the Monitor ID; editing that
+Monitor does not redefine the request. Reusing the key for another owned Monitor
+returns409 / CONFLICT. A different key intentionally creates a new execution.
+
+The browser creates one key per intention, prevents a second submission while
+pending, and retains the key if it does not receive a valid acknowledgement.
+An explicit retry reuses it, including when the execution has already completed.
+After acknowledgement, the next click gets a new key. This page-local intention
+state contains no session credential and adds no automatic request retry.
+
+Workers select a QUEUED row with `FOR UPDATE SKIP LOCKED` and commit RUNNING plus
+their owner identity in one short transaction. Only that owner can write its
+terminal result. Outbound I/O still runs outside a transaction with the existing
+timeouts. Scheduled-slot insertion also uses the existing unique slot index with
+`ON CONFLICT DO NOTHING`; repeated ticks retain one intent. V7 adds internal
+identity/owner columns without changing old outcomes or inventing historical
+request keys. No claim lease, takeover or crash recovery is added.
+
+The E10 Java test starts two actual non-web JVMs that invoke the production
+CheckWorker once through a test-only entry point and PostgreSQL barriers; this
+is distinct from the normal scheduled CLI loop. It checks one held outbound
+request and a rejected completion from the losing process. The browser case
+forwards a real acceptance, loses its response, then checks same-key/current-ID
+retransmission and a new key after acknowledgement. The frozen non-ASCII Java
+request uses a literal-byte test transport because the selected JDK HTTP client
+changes that header before transmission. Production validation is unchanged.
diff --git a/evidence/phase-1/E10/author-attempt2/browser-results.json b/evidence/phase-1/E10/author-attempt2/browser-results.json
new file mode 100644
index 0000000..666b34b
--- /dev/null
+++ b/evidence/phase-1/E10/author-attempt2/browser-results.json
@@ -0,0 +1,23 @@
+{
+  "stats": {
+    "startTime": "2026-08-28T05:48:57.429Z",
+    "duration": 11424.776,
+    "expected": 1,
+    "skipped": 0,
+    "unexpected": 0,
+    "flaky": 0
+  },
+  "cases": [
+    {
+      "title": "E10 retains one manual intent after an accepted response is lost and creates a new key after acknowledgement",
+      "expectedStatus": "passed",
+      "results": [
+        {
+          "status": "passed",
+          "duration": 2190,
+          "retry": 0
+        }
+      ]
+    }
+  ]
+}
diff --git a/evidence/phase-1/E10/author-attempt2/invocations.jsonl b/evidence/phase-1/E10/author-attempt2/invocations.jsonl
new file mode 100644
index 0000000..05af9d1
--- /dev/null
+++ b/evidence/phase-1/E10/author-attempt2/invocations.jsonl
@@ -0,0 +1,8 @@
+{"name":"package-attempt2-1","command":"mvn -B -ntp -f backend/pom.xml -DskipTests package","startedAt":"2026-08-28T05:47:34.022Z","elapsedSeconds":2.581,"exitCode":0}
+{"name":"build-attempt2-1","command":"npm run build","startedAt":"2026-08-28T05:47:46.672Z","elapsedSeconds":8.027,"exitCode":0}
+{"name":"browser-intent-attempt2-1","command":"npm run test:e2e -- tests/browser/server-state.spec.ts --grep E10","startedAt":"2026-08-28T05:48:57.025Z","elapsedSeconds":11.841,"exitCode":0}
+{"name":"attempt2-database-inspection","command":"docker compose --project-name wse-industry --file compose.yaml exec --no-TTY postgres psql -U wse_industry -d monitor -tA -c \"SELECT schema_name FROM information_schema.schemata WHERE schema_name NOT LIKE 'pg_%' AND schema_name <> 'information_schema' ORDER BY schema_name\"","startedAt":"2026-08-28T05:49:46.663Z","elapsedSeconds":0.018097125,"exitCode":0,"result":"owned e04_browser and public"}
+{"name":"attempt2-listener-inspection","command":"lsof -nP -iTCP:4321-4325 -iTCP:4999 -sTCP:LISTEN","startedAt":"2026-08-28T05:49:46.881Z","elapsedSeconds":0.000003208,"exitCode":1,"result":"no matching listeners; expected lsof exit1"}
+{"name":"attempt2-process-inspection","command":"pgrep -fl 'java.*monitor-api-0[.]0[.]1[.]jar|dev[.]evolution[.]monitor[.]E10WorkerProcess'","startedAt":"2026-08-28T05:49:49.767Z","elapsedSeconds":0.000002833,"exitCode":1,"result":"no owned API/worker JVM; expected pgrep exit1"}
+{"name":"browser-intent-attempt2-cleanup","command":"node scripts/database.mjs drop e04_browser","startedAt":"2026-08-28T05:50:19.229Z","elapsedSeconds":0.17,"exitCode":0}
+{"name":"attempt2-final-database-inspection","command":"docker compose --project-name wse-industry --file compose.yaml exec --no-TTY postgres psql -U wse_industry -d monitor -tA -c \"SELECT schema_name FROM information_schema.schemata WHERE schema_name NOT LIKE 'pg_%' AND schema_name <> 'information_schema' ORDER BY schema_name\"","startedAt":"2026-08-28T05:50:19.406Z","elapsedSeconds":0.000004292,"exitCode":0,"result":"public only after explicit e04_browser cleanup"}
diff --git a/evidence/phase-1/E10/browser-intent.json b/evidence/phase-1/E10/browser-intent.json
new file mode 100644
index 0000000..df01797
--- /dev/null
+++ b/evidence/phase-1/E10/browser-intent.json
@@ -0,0 +1,30 @@
+{
+  "start": "3cc49f3d2a35055c92d0312fca6167c89dfadec5",
+  "fixtureSha256": "8628e42090fb5a71d9f6e4570742b670a63a1e9ce6d5d11603f9c6b03b693649",
+  "result": "PASS",
+  "heldAcceptance": {
+    "status": 202,
+    "clickEvents": 2,
+    "requests": 1,
+    "persistedRows": 1
+  },
+  "retransmission": {
+    "responseWasLostAfterAcceptance": true,
+    "sameKey": true,
+    "sameId": true,
+    "status": 202,
+    "currentState": "SUCCEEDED",
+    "requests": 2,
+    "persistedRows": 1,
+    "failureAndPendingCleared": true
+  },
+  "nextIntent": {
+    "differentKey": true,
+    "differentId": true,
+    "status": 202,
+    "totalRequests": 3,
+    "persistedRows": 2,
+    "otherMonitorRows": 0
+  },
+  "heldRoutesSettled": true
+}
diff --git a/evidence/phase-1/E10/cleanup.json b/evidence/phase-1/E10/cleanup.json
new file mode 100644
index 0000000..3f66362
--- /dev/null
+++ b/evidence/phase-1/E10/cleanup.json
@@ -0,0 +1,17 @@
+{
+  "databaseCheckedAt": "2026-08-28T05:50:19.406Z",
+  "schemas": [
+    "public"
+  ],
+  "beforeExplicitCleanup": [
+    "e04_browser",
+    "public"
+  ],
+  "cleanupCommand": "node scripts/database.mjs drop e04_browser",
+  "cleanupExitCode": 0,
+  "listenersCheckedAt": "2026-08-28T05:49:46.881Z",
+  "listenersOn4321Through4325And4999": [],
+  "processesCheckedAt": "2026-08-28T05:49:49.767Z",
+  "ownedApiOrWorkerJvms": 0,
+  "publicDataAndVolumePreserved": true
+}
diff --git a/evidence/phase-1/E10/verification.md b/evidence/phase-1/E10/verification.md
new file mode 100644
index 0000000..b8420a9
--- /dev/null
+++ b/evidence/phase-1/E10/verification.md
@@ -0,0 +1,106 @@
+# Phase-1 E10 author report — attempt2
+
+Branch `track/industry-spring`; original Thread START
+`3cc49f3d2a35055c92d0312fca6167c89dfadec5`.
+SPEC_REVISION `2ada57a71cd34fa2fae9809415c362a8bbfcdf02`.
+Resumed after root verified repair1 END
+`808394b4c26d3ee4bab9d6989ffe3e63f182354a`.
+Fixture SHA256 remains
+`8628e42090fb5a71d9f6e4570742b670a63a1e9ce6d5d11603f9c6b03b693649`.
+
+## Baseline, failure and repair history
+
+The single unchanged-product baseline returned two202/QUEUED responses with
+different IDs and two durable rows, while the real fixture counted zero calls.
+No second baseline or baseline worker race was run.
+
+Attempt1's targeted Maven gate ran21 tests:20 passed and the new E10 case failed
+on expected400 versus actual202 in its invalid-key loop. The first parallel
+request checkpoint had already proved two waiting inserts, one resulting ID/row
+and zero outbound calls. No competing worker had started. The original assertion
+did not label which invalid input failed; that historical limitation remains.
+All original raw output and partial evidence are preserved under `repair1/attempt1`.
+
+Root stopped authoring and assigned one fresh bounded repair. Its one fixed
+loopback diagnostic proved the selected JDK transport changed U+00E9 into byte3f;
+the alternate Simple transport sentc3a9 and was not used as a literal substitute.
+Only the frozen U+00E9 request now uses the literal e9/HTTP1.0 test adapter. The
+same five inputs, status/no-write assertions and production validator remained.
+The repair also labeled each invalid-key observation. Its one targeted Java case
+passed. Root independently reviewed the repair, its actual output and cleanup.
+See `repair1/result.json`; raw Maven/JUnit whitespace remains byte-exact.
+
+## Verified behavior
+
+The repaired real-PostgreSQL case observed two blocked same-owner/key inserts,
+then202 responses with one current ID/QUEUED row. Another target returned409;
+mutable Monitor fields did not change identity. All five invalid keys returned400
+without a write or outbound call. A128-character key was accepted as a new
+intent, and its replay did not create another row.
+
+Two actual non-web worker JVMs reached the database start gate. They used a
+test-only startup entry point calling the production CheckWorker once, not the
+normal scheduled CLI loop. Worker4659 won; worker4658 lost. While one /ok response
+was held, one RUNNING row retained its owner and null terminal outcome. The
+losing process's non-owner completion changed zero rows. Releasing200 produced
+one same-ID SUCCEEDED result and exactlyone outbound call; both exits were
+awaited. Current RUNNING/terminal retransmissions retained the same identity.
+Actual process and owner IDs are in the immutable repair gate artifacts.
+
+The original targeted gate also passed the existing owner/security matrix3,
+functional checks15, scheduler1 and migration1. Its SQL/transaction assertions
+remained active. The E05 two-owner setup used the same key for different owners
+while retaining its original two-row dataset. V6→V7 preserved seven original
+rows including queued metadata and prior migration checksums. Existing scheduled
+slot identity, terminal finished_at/id ordering, Origin/CSRF and CORS boundaries
+were retained; no additional author rerun of those passing cases occurred.
+
+`browser-intent.json` records the newly completed production Chromium case:
+two clicks while the first real acceptance response was held sent one request
+and created one row. Delivery was then aborted. Failure was visible and pending
+cleared. Explicit retry reused the key, returned202 with the same SUCCEEDED ID
+and left one row. A click after acknowledgement used a different key/ID and
+created one additional row. B retained zero rows. All held routes settled.
+
+## Actual invocations and cumulative budget
+
+| Invocation | Outcome | Elapsed |
+| --- | --- | --- |
+| Unchanged baseline | Expected counterexample / exit1 | 8.036s |
+| Attempt1 targeted Maven/package |21 tests;20 pass,1 fail; package stopped |15.919s |
+| Attempt1 typecheck | PASS |1.988s |
+| Repair1 transport diagnostic | PASS; six fixed captures, one invocation |1.925166s |
+| Repair1 targeted Java case | PASS; one test |13.472970s |
+| Attempt2 Maven package with tests skipped | PASS; actual repackaged artifact |2.581s |
+| Attempt2 production frontend build | PASS |8.027s |
+| Attempt2 targeted E10 browser case | PASS; one test |11.841s |
+
+The original typecheck was reused after matching all four changed frontend/browser
+source hashes against the preserved author bytes in `repair1/result.json` and
+confirming no frontend/config diff since adoption commit3a1adae. No product or
+test source was changed during the resumed author work. Packaging explicitly
+skipped tests; it was not an additional Java test pass.
+
+Cumulative author plus repair budget: baseline1; Maven invocations3 (two test
+gates and one package-only invocation); Java test executions22, passes21,
+failures1, with21 distinct tests having passing evidence; typecheck1; transport
+diagnostic1/six requests; competing-worker gate1/two JVMs; build1; browser1/one
+test; fresh repairs1 of2; load runs0; automatic retries0; parameter sweeps0;
+full author/root `npm run verify` invocations0 so far. Root's final full acceptance
+is pending and is not represented as an author pass.
+
+Original commands remain in `repair1/attempt1/invocations.jsonl`; repair commands
+remain in `repair1/invocations.jsonl`; the resumed commands and cleanup inspection
+outcomes are appended separately in `author-attempt2/invocations.jsonl`.
+
+## Cleanup and scope
+
+After the targeted browser command, the API/worker processes and ports were free.
+The disposable e04_browser schema remained, so the standard explicit cleanup
+removed only that schema. Final inspection showed public only. Public data and
+the PostgreSQL volume were preserved. See `cleanup.json` for actual observations.
+
+No old evidence, migration, dependency pin, main/spec/index/tag or history was
+changed during resumed authoring. No credentials/session/CSRF values or browser
+captures were added. No E11 lease, takeover, recovery or other future behavior
+was implemented. This candidate awaits root's independent final verification.
