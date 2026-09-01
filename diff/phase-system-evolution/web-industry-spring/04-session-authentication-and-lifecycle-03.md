## `고정 E04 반례와 검증 결과 보존`

diff --git a/TRACK.md b/TRACK.md
index 1279dd7..7aca3b4 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -2,7 +2,7 @@
 
 Spec revision: `0a006589477f8ae47bad3faa5510c999cff85ee4`
 
-E03 is a local development product: Next.js/React renders Monitor creation, editing, pause/activation, deletion and Check history. Spring MVC uses PostgreSQL as the authoritative store for Monitors and every completed CheckRun. A manual check is synchronous. There are no accounts, scheduler, workers, cache, broker, or production application containers.
+E04 is a local development product: Next.js/React renders login, logout, Monitor creation, editing, pause/activation, deletion and Check history. Spring Security authenticates browser sessions; PostgreSQL stores salted account passwords, Monitors and every completed CheckRun. A manual check is synchronous. There is no signup, ownership authorization, scheduler, worker, cache, broker, or production application container.
 
 ## Pinned toolchain
 
@@ -23,6 +23,7 @@ E03 is a local development product: Next.js/React renders Monitor creation, edit
 | Hibernate ORM | 6.6.53.Final | existing Spring Boot 3.5.16 BOM |
 | Flyway core / PostgreSQL module | 11.7.2 | existing Spring Boot 3.5.16 BOM |
 | PostgreSQL JDBC | 42.7.11 | existing Spring Boot 3.5.16 BOM |
+| Spring Security core / config / web / crypto | 6.5.11 | existing Spring Boot 3.5.16 BOM; starter-security 3.5.16 |
 
 Versions remain fixed for this evolution. The Spring parent pins build plugins and transitive dependency management. There is no application container build yet; the only container is the isolated development/test database.
 
@@ -34,8 +35,18 @@ Run all commands from the repository root using the pinned runtimes. Maven artif
 fnm use 24.19.0
 npm ci
 npm run db:up
+npm run api:package
 ```
 
+Before the first login, provide independent `E04_ALICE_PASSWORD` and
+`E04_BOB_PASSWORD` values through runtime secret environment input (24–72 UTF-8
+bytes each), then run `npm run bootstrap:users`. This fixed command prepares
+`alice-e04` and `bob-e04` in the configured database schema without printing a
+password or hash. Repeating with the same input is idempotent; conflicting
+existing credentials fail rather than silently resetting an account. Do not put
+secret values in shell history, command arguments, committed files, or logs.
+Verification generates its own independent random passwords automatically.
+
 Start each process in a separate terminal:
 
 ```sh
@@ -44,13 +55,13 @@ npm run api:dev
 npm run dev
 ```
 
-Open [Monitors](http://127.0.0.1:4323/monitors). Create `Fixture monitor` with URL `http://127.0.0.1:4321/ok`, interval `60`, enabled checked. Click **Run check** to observe `SUCCEEDED` and `HTTP 200`. `/fail` yields `FAILED` and `HTTP 503`.
+Open [Monitors](http://127.0.0.1:4323/monitors), sign in with a prepared account, then create `Fixture monitor` with URL `http://127.0.0.1:4321/ok`, interval `60`, enabled checked. Click **Run check** to observe `SUCCEEDED` and `HTTP 200`. `/fail` yields `FAILED` and `HTTP 503`.
 
 All defaults bind to `127.0.0.1`. Fixture port is 4321, API port 4322, UI port 4323, PostgreSQL port 15432. `FIXTURE_PORT`, `FIXTURE_ORIGIN`, and `API_PORT` can configure the server processes; `API_ORIGIN` configures the Next API proxy. The committed E01 tests use the fixed default ports; stop local servers before verification. Tests also reserve 4324 for a forbidden destination trap and 4325 for timeout/connection fixtures.
 
 The compose project is exclusively `wse-industry`, with its own bridge network and persistent volume. It uses explicitly nonsecret local test trust authentication and a loopback-only published port. An internal-only Docker network cannot provide the required published port on the verified Docker Desktop runtime. Never use this compose configuration for production or put unrelated data in it. `npm run db:down` stops the project but preserves data; `npm run db:up` restores it. `npm run db:destroy` explicitly deletes this project's disposable database volume.
 
-The default connection is database `monitor`, local test identity `wse_industry`, schema `public`. `DB_URL` (JDBC URL), `DB_USER`, `DB_PASSWORD`, and `DB_SCHEMA` support external runtime configuration. Never commit or log real credentials or put a password in a JDBC URL. Verification resets only explicitly named `e03_*` schemas, not the developer's `public` schema.
+The default connection is database `monitor`, local test identity `wse_industry`, schema `public`. `DB_URL` (JDBC URL), `DB_USER`, `DB_PASSWORD`, and `DB_SCHEMA` support external runtime configuration. Never commit or log real credentials or put a password in a JDBC URL. Current verification resets only explicitly named `e04_*` schemas, not the developer's `public` schema.
 
 ## Check boundary
 
@@ -67,13 +78,13 @@ The default connection is database `monitor`, local test identity `wse_industry`
 - E03 also rejects a NUL character in a name before persistence, because PostgreSQL text cannot store it. Create and replacement use the same runtime validator, including the existing non-finite numeric rejection.
 - URL syntax must be absolute HTTP(S). The existing fixture policy then restricts it to the configured HTTP hostname and port, without credentials or fragments. This does not enable general public or HTTPS monitoring.
 - Successful list/create/check responses contain exactly `{ "data": <payload> }`. Create returns 201; list and completed synchronous checks return 200. The existing MonitorView/CheckRun payload fields are preserved, including explicit nulls.
-- API failures contain `{ "error": { "code": "...", "message": "..." } }`: INVALID_INPUT / 400, NOT_FOUND / 404, or INTERNAL_ERROR / 500. Malformed JSON and IDs use INVALID_INPUT. Unexpected exception details are not returned.
+- API failures contain `{ "error": { "code": "...", "message": "..." } }`: INVALID_INPUT / 400, UNAUTHENTICATED / 401, FORBIDDEN / 403, NOT_FOUND / 404, or INTERNAL_ERROR / 500. Malformed JSON and IDs use INVALID_INPUT after authentication. Unexpected exception details are not returned.
 - Endpoint HTTP 503 is still an API success containing a FAILED CheckRun with HTTP_STATUS; timeout and connection failure retain null httpStatus. API errors never fabricate a CheckRun.
 - The browser validates the envelope and the displayed payload shape, selects errors by the stable code/status pair, and does not classify or display arbitrary server prose. Network or malformed responses use the INTERNAL_ERROR UI fallback without applying the mutation.
 
 ## PostgreSQL boundary (E03)
 
-- Flyway is the sole schema writer: V1 creates `monitors`, V2 creates `check_runs`. Repeated startup validates migration checksums and applies only pending migrations. Hibernate uses `ddl-auto=validate`, never create/update.
+- Flyway is the sole schema writer: unchanged V1 creates `monitors`, unchanged V2 creates `check_runs`, and appended V3 creates `users`. Repeated startup validates migration checksums and applies only pending migrations. Hibernate uses `ddl-auto=validate`, never create/update.
 - A startup metadata check supplements Hibernate validation by rejecting unmapped required columns without defaults. Missing mapped columns or incompatible insert requirements prevent the web server from becoming ready.
 - `MonitorEntity.fromDomain/toDomain` and `CheckRunEntity.fromDomain/toDomain` explicitly map canonical immutable records. Entities never reach JSON. UUIDs remain UUIDs; interval is PostgreSQL integer, enabled boolean, latency bigint, and nullable HTTP status/reason retain null rather than zero/empty text.
 - Timestamp values are truncated to microseconds before the first response and stored as `timestamp(6) with time zone`. Canonical Java Instant/JSON values remain UTC even in a non-UTC PostgreSQL session.
@@ -81,6 +92,39 @@ The default connection is database `monitor`, local test identity `wse_industry`
 - `GET /api/monitors/{id}` returns a MonitorView. `PUT /api/monitors/{id}` replaces the same four create fields, including enabled for pause/activation. `DELETE /api/monitors/{id}` returns `{ "data": null }`; PostgreSQL `ON DELETE CASCADE` removes all historical runs.
 - `GET /api/monitors/{id}/checks` returns the full history ordered by finishedAt descending, then ID descending. `GET /api/monitors/{id}/checks/{checkId}` returns a single historical result belonging to that Monitor. Deleted Monitor/history/run resources return 404/NOT_FOUND. Pagination is not introduced here.
 
+## Browser sessions (E04)
+
+- `POST /api/session/login` accepts JSON username/password, authenticates through
+  Spring Security's `DaoAuthenticationProvider`, and returns only the username.
+  Missing or invalid credentials use the same 401 / UNAUTHENTICATED message;
+  account existence and credential material are not returned.
+- `GET /api/session` reads the current username. `POST /api/session/logout`
+  invalidates the actual servlet session and clears the browser cookie. Even a
+  valid anonymous CSRF bootstrap session does not authenticate logout.
+- Successful login calls the container's session-ID rotation strategy, resets
+  CSRF state, and explicitly saves the new SecurityContext. The prior identifier
+  no longer authenticates. ProviderManager erases both password credentials and
+  the principal's password representation before session storage.
+- The HttpOnly `WSESESSION` cookie is SameSite=Lax, path `/`, and cookie-only
+  transport is enforced. Local development uses loopback HTTP, so Secure is false;
+  `SESSION_COOKIE_SECURE=true` is required when using TLS. No token enters frontend
+  local/session storage, login URLs, or normal API payloads.
+- Sessions live in the API process and are never serialized on restart. Their
+  absolute lifetime is exactly 3600000 ms after a successful login. A pre-context
+  filter rejects at the exact deadline, regardless of intervening reads. The
+  container also imposes a 3600-second idle limit. Restart requires login again.
+- Accounts use salted bcrypt cost 10; entities and hash fields are never returned
+  from controllers. Account reads/writes use the actual transaction proxy.
+- Spring Security's default session-backed CSRF protection remains enabled.
+  `GET /api/session/csrf` supplies its request token; the UI fetches a fresh token
+  before each mutation and sends it only in the indicated header. This token is
+  not a login credential. Missing authentication stays 401 even when CSRF rejects
+  first; an authenticated invalid-token request is 403. Additional E05 ownership,
+  cross-origin policy and authorization matrices are not introduced here.
+- The Security starter is the only new dependency: it directly supplies the
+  required filter boundary, credential verification, session strategies and
+  existing framework CSRF protection. All managed versions remain fixed.
+
 ## Verification
 
 ```sh
@@ -88,10 +132,16 @@ npx playwright install chromium
 npm run verify
 ```
 
-`verify` starts the isolated PostgreSQL project, runs Maven unit, real-HTTP functional and real-PostgreSQL integration tests, and packages the API. It then checks TypeScript, compiles Next for production, runs the fixed A,A,B process-restart/lifecycle scenario, and runs Chromium against the local development UI and packaged API. It does not retry failed tests. Command outcomes and elapsed times are appended to `output/verification/runs.jsonl`; the process-restart probe saves wire evidence in `output/e03`. Committed evidence is in `evidence/E01`, `evidence/E02`, and `evidence/E03`.
+`verify` starts the isolated PostgreSQL project, runs Maven unit, real-HTTP authentication/functional and real-PostgreSQL integration tests, and packages the API. It then checks TypeScript, compiles Next for production, runs the unchanged A,A,B process-restart/lifecycle product sequence with authentication setup, and runs Chromium against the local development UI and packaged API. It does not retry failed tests. Command outcomes and elapsed times are appended to `output/verification/runs.jsonl`; the process-restart probe saves only product wire evidence in `output/e03`. Committed evidence is in `evidence/E01` through `evidence/E04`.
 
 Tests create and remove isolated schemas for functional, browser, restart, mapping, migration, and incompatible-schema fixtures. The standard runner cleans up its browser/restart schemas even after a failure. The database remains available afterward; use `npm run db:down` to stop only this project. The Java tests explicitly close independent application contexts to verify restart persistence, capture actual generated SQL/transaction flags, check rollback and cascade, and assert startup rejection for both incompatible-schema cases.
 
+E04 additionally tests account schema incompatibilities, salted representation and
+credential erasure, a fixed one-hour clock boundary, actual old-cookie revocation,
+and real browser login/logout and session-loss recovery. Existing browser cases
+receive only auth setup; their product inputs and assertions remain unchanged.
+Credential-bearing traces, screenshots, videos and storage-state files are disabled.
+
 The CI workflow installs the exact toolchain and runs the same gates. No hosted CI run is claimed by local verification. The browser gate starts and stops its own processes and refuses existing servers. There are no load tests, benchmarks, or parameter sweeps.
 
 Individual commands:
@@ -114,3 +164,6 @@ npm run test:e2e
 - [Spring Boot 3.5 database initialization](https://docs.spring.io/spring-boot/3.5/how-to/data-initialization.html)
 - [Spring transaction proxy boundaries](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html)
 - [PostgreSQL 17 timestamp precision](https://www.postgresql.org/docs/17/datatype-datetime.html)
+- [Spring Security 6.5 session management](https://docs.spring.io/spring-security/reference/6.5/servlet/authentication/session-management.html)
+- [Spring Security 6.5 logout filter ordering](https://docs.spring.io/spring-security/reference/6.5/servlet/authentication/logout.html)
+- [Spring Security 6.5 CSRF defaults](https://docs.spring.io/spring-security/reference/6.5/servlet/exploits/csrf.html)
diff --git a/evidence/E04/authorized-regressions.json b/evidence/E04/authorized-regressions.json
new file mode 100644
index 0000000..1bd1206
--- /dev/null
+++ b/evidence/E04/authorized-regressions.json
@@ -0,0 +1,64 @@
+{
+  "result": "PASS: all historical records survive a new API process; lifecycle and NUL boundary hold",
+  "monitors": 2,
+  "completedChecks": 3,
+  "requestCount": 30,
+  "processes": [
+    {
+      "role": "fixture",
+      "pid": 6962,
+      "startedAt": "2026-08-28T00:55:30.366Z",
+      "logPath": "output/e03/fixed-fixture.log",
+      "readyAt": "2026-08-28T00:55:30.507Z",
+      "ownedStartupMarkerObserved": true,
+      "exitCode": 0,
+      "signal": null,
+      "exitedAt": "2026-08-28T00:55:45.931Z",
+      "exitAwaited": true
+    },
+    {
+      "role": "api-first",
+      "pid": 6974,
+      "startedAt": "2026-08-28T00:55:30.511Z",
+      "logPath": "output/e03/fixed-api-first.log",
+      "readyAt": "2026-08-28T00:55:34.772Z",
+      "ownedStartupMarkerObserved": true,
+      "exitCode": 143,
+      "signal": null,
+      "exitedAt": "2026-08-28T00:55:36.103Z",
+      "exitAwaited": true,
+      "exitObserved": true
+    },
+    {
+      "role": "api-fresh",
+      "pid": 7049,
+      "startedAt": "2026-08-28T00:55:36.108Z",
+      "logPath": "output/e03/fixed-api-fresh.log",
+      "readyAt": "2026-08-28T00:55:43.842Z",
+      "ownedStartupMarkerObserved": true,
+      "exitCode": 143,
+      "signal": null,
+      "exitedAt": "2026-08-28T00:55:45.923Z",
+      "exitAwaited": true
+    }
+  ],
+  "portChecks": [
+    {
+      "port": 4321,
+      "checkedAt": "2026-08-28T00:55:25.686Z",
+      "result": "free"
+    },
+    {
+      "port": 4322,
+      "checkedAt": "2026-08-28T00:55:25.692Z",
+      "result": "free"
+    },
+    {
+      "port": 4322,
+      "checkedAt": "2026-08-28T00:55:36.104Z",
+      "result": "free"
+    }
+  ],
+  "elapsedSeconds": 20.254,
+  "credentialsCaptured": false
+}
diff --git a/evidence/E04/baseline.json b/evidence/E04/baseline.json
new file mode 100644
index 0000000..8dfaa3e
--- /dev/null
+++ b/evidence/E04/baseline.json
@@ -0,0 +1,26 @@
+{
+  "start": "45711aaa58b311065e0cea423bc53d283e1d4fa9",
+  "fixtureSha256": "fa9d3babe27ae625665468994674ef7a1a6afdb5e7d4bfabcc7e261984ac2aa2",
+  "requests": [
+    {
+      "method": "GET",
+      "path": "/api/monitors",
+      "anonymous": true,
+      "status": 200,
+      "collectionCount": 0
+    }
+  ],
+  "startedAt": "2026-08-28T00:32:44.310Z",
+  "portFree": true,
+  "pid": 87577,
+  "ownStartupMarker": true,
+  "readyAt": "2026-08-28T00:32:58.427Z",
+  "requiredStatus": 401,
+  "requiredCode": "UNAUTHENTICATED",
+  "result": "REPRODUCED: anonymous collection read is authorized at unchanged START",
+  "exitAwaited": true,
+  "exitCode": 143,
+  "signal": null,
+  "schemaDropped": true,
+  "finishedAt": "2026-08-28T00:32:59.972Z"
+}
diff --git a/evidence/E04/browser-verification.json b/evidence/E04/browser-verification.json
new file mode 100644
index 0000000..1ebbe25
--- /dev/null
+++ b/evidence/E04/browser-verification.json
@@ -0,0 +1,24 @@
+{
+  "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:e2e",
+  "exitCode": 0,
+  "elapsedSeconds": 18.85,
+  "stats": {
+    "startTime": "2026-08-28T00:59:01.672Z",
+    "duration": 18152.035,
+    "expected": 12,
+    "skipped": 0,
+    "unexpected": 0,
+    "flaky": 0
+  },
+  "workers": 1,
+  "retries": 0,
+  "trace": "off",
+  "screenshot": "off",
+  "video": "off",
+  "automaticPageSnapshot": false,
+  "preserveOutput": "never",
+  "retainedPerTestFiles": 0,
+  "postRunExplicitSchemaCleanup": "e04_browser",
+  "remainingOwnedSchemas": 0,
+  "ownedPortListeners": 0
+}
diff --git a/evidence/E04/csrf-clarification.md b/evidence/E04/csrf-clarification.md
new file mode 100644
index 0000000..71d4f7c
--- /dev/null
+++ b/evidence/E04/csrf-clarification.md
@@ -0,0 +1,23 @@
+# E04 framework CSRF setup clarification
+
+Before the first implementation patch was applied, the approval reviewer rejected
+that patch because it disabled Spring Security CSRF protection for browser
+sessions. The entire rejected patch was unapplied. It was not retried or bypassed.
+
+The main agent clarified that retaining Spring Security's default CSRF protection
+and adding its minimum token transport is part of making E04 login/logout and
+authorized product requests work. E05 ownership, authorization matrices and
+additional CORS/CSRF hardening remain out of scope.
+
+The frozen `fixtures.md` remains byte-identical. All original Monitor inputs,
+session lifecycle assertions, 3600000 ms TTL and exact clock checkpoints remain
+unchanged. Authentication setup obtains a default session-backed CSRF token via
+`GET /api/session/csrf`, sends it in the indicated request header, and refreshes it
+after login rotates authentication state. No token, password, hash, session ID,
+credential URL, or sensitive browser trace is written to evidence.
+
+Unauthenticated protected requests retain 401 / UNAUTHENTICATED even when the
+default CSRF filter is the first denying filter. Authorized requests missing a
+valid token remain rejected by that filter with 403 / FORBIDDEN. The only added
+token failure check confirms preservation of the framework default; it is not an
+E05 cross-user, CORS, origin or CSRF attack matrix.
diff --git a/evidence/E04/first-verification.json b/evidence/E04/first-verification.json
new file mode 100644
index 0000000..cc7a350
--- /dev/null
+++ b/evidence/E04/first-verification.json
@@ -0,0 +1,78 @@
+{
+  "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run verify",
+  "exitCode": 1,
+  "elapsedSeconds": 88.87,
+  "javaTests": 30,
+  "javaFailures": 0,
+  "browserPassed": 11,
+  "browserFailed": 1,
+  "browserRetries": 0,
+  "failure": "New E04 alert locator also matched the Next route announcer; actual application alert had the expected UNAUTHENTICATED code and message.",
+  "correction": "Only scope the two new alert locators to main; preserve all actions, inputs, expected messages, fixed TTL and product code.",
+  "artifactAudit": {
+    "credentialValueMatches": 0,
+    "populatedPasswordSnapshotLines": 0
+  },
+  "gates": [
+    {
+      "command": "node scripts/database.mjs up",
+      "startedAt": "2026-08-28T00:54:54.050Z",
+      "elapsedSeconds": 0.807,
+      "exitCode": 0
+    },
+    {
+      "command": "mvn -B -ntp -f backend/pom.xml package",
+      "startedAt": "2026-08-28T00:54:54.858Z",
+      "elapsedSeconds": 16.765,
+      "exitCode": 0
+    },
+    {
+      "command": "npm run typecheck",
+      "startedAt": "2026-08-28T00:55:11.626Z",
+      "elapsedSeconds": 1.769,
+      "exitCode": 0
+    },
+    {
+      "command": "npm run build",
+      "startedAt": "2026-08-28T00:55:13.395Z",
+      "elapsedSeconds": 11.778,
+      "exitCode": 0
+    },
+    {
+      "command": "node scripts/persistence-isolation.mjs",
+      "startedAt": "2026-08-28T00:55:25.175Z",
+      "elapsedSeconds": 0.165,
+      "exitCode": 0
+    },
+    {
+      "command": "node scripts/database.mjs reset e04_restart",
+      "startedAt": "2026-08-28T00:55:25.341Z",
+      "elapsedSeconds": 0.288,
+      "exitCode": 0
+    },
+    {
+      "command": "node scripts/persistence-scenario.mjs fixed",
+      "startedAt": "2026-08-28T00:55:25.629Z",
+      "elapsedSeconds": 20.311,
+      "exitCode": 0
+    },
+    {
+      "command": "npm run test:e2e",
+      "startedAt": "2026-08-28T00:55:45.941Z",
+      "elapsedSeconds": 36.387,
+      "exitCode": 1
+    },
+    {
+      "command": "node scripts/database.mjs drop e04_restart",
+      "startedAt": "2026-08-28T00:56:22.328Z",
+      "elapsedSeconds": 0.256,
+      "exitCode": 0
+    },
+    {
+      "command": "node scripts/database.mjs drop e04_browser",
+      "startedAt": "2026-08-28T00:56:22.584Z",
+      "elapsedSeconds": 0.163,
+      "exitCode": 0
+    }
+  ]
+}
diff --git a/evidence/E04/fixtures.md b/evidence/E04/fixtures.md
new file mode 100644
index 0000000..3283d45
--- /dev/null
+++ b/evidence/E04/fixtures.md
@@ -0,0 +1,71 @@
+# E04 frozen session fixtures
+
+Branch: `track/industry-spring`.
+Spec revision: `0a006589477f8ae47bad3faa5510c999cff85ee4`.
+Unchanged START: `45711aaa58b311065e0cea423bc53d283e1d4fa9`.
+Attempt: 1. This file is frozen before any E04 baseline or expiry execution.
+
+## Decisive unchanged baseline
+
+Build the unchanged START. Refuse an occupied loopback API port 4322; create only
+the private `e04_baseline` PostgreSQL schema in `wse-industry`. Start an owned API
+process, require its own startup marker and live PID, then send exactly one
+anonymous `GET /api/monitors`. The existing result is expected to be 200 with an
+empty collection; the new required result is 401 / UNAUTHENTICATED. Readiness uses
+the owned process marker, not another collection request. Await the owned exit
+and remove only that schema. Do not repeat the baseline.
+
+## Fixed session lifecycle
+
+Prepare `alice-e04` and `bob-e04` with independent strong passwords generated at
+runtime. Passwords enter only through runtime secret input and never appear in
+source, command arguments, logs, evidence, browser traces, or screenshots.
+
+1. Anonymous collection GET returns 401 / UNAUTHENTICATED.
+2. Valid Alice login issues a browser session cookie. A wrong runtime password
+   returns the same 401 envelope as an unknown username, without account details.
+3. Alice's cookie authorizes the collection GET.
+4. Successful login with that existing cookie changes the identifier; the old
+   cookie returns 401 and the new cookie authorizes the collection.
+5. Logout revokes the credential and clears its browser cookie; replay is 401.
+6. A fresh Alice login expires deterministically at the exact fixed checkpoint
+   below. Invalid, missing, and expired cookies all return 401 / UNAUTHENTICATED.
+7. Bob logs in independently and receives an authorized session. No ownership
+   assertion or CSRF protection is introduced before E05.
+
+Session TTL is exactly **3600000 ms (one hour)**, an absolute limit from successful
+login, not sliding renewal by reads. The deterministic test clock starts at
+`2026-08-28T01:00:00.000Z`. At `2026-08-28T01:59:59.999Z` the new session still
+authorizes a read; at `2026-08-28T02:00:00.000Z` it returns 401. No sleep, search,
+retry, or changed timeout is used to discover expiry.
+
+## Required supplementary gates
+
+- Exercise the real Spring Security servlet filter boundary over HTTP for every
+  Monitor/history/check route and method, including malformed protected input.
+- Observe only status/count/boolean evidence for cookies and credentials. Check
+  HttpOnly, SameSite, cookie-only transport, no session ID in API payloads or
+  frontend storage, and the configured local HTTP versus secure-cookie policy.
+- Store only salted one-way password representations. Inspect matching,
+  non-plaintext storage and independent salts through booleans; never output the
+  stored hash. Suppress framework generated-password output by providing the
+  actual account service.
+- Append a migration for current account storage; preserve V1/V2 byte-for-byte.
+  Test fresh and V2 upgrade/repeat behavior, real transactions/SQL, and refusal
+  before readiness for missing mapped or extra required account columns.
+- Real Chromium login, invalid-login recovery, authenticated existing product
+  flows, logout and redirect/recovery after session loss. Disable credential
+  traces, screenshots and videos, including failure captures.
+- Preserve all E01-E03 product inputs/assertions and all committed evidence bytes.
+  Extend only auth setup and latest-migration expectations where required.
+- Private E04 schemas: `e04_baseline`, `e04_session`, `e04_users`,
+  `e04_missing_user_column`, `e04_extra_user_required`, `e04_functional`,
+  `e04_browser`, `e04_restart`, `e04_migrations`, `e04_mapping`,
+  `e04_missing_column`, `e04_extra_required`.
+- Existing process ports remain fixture 4321, API 4322, UI 4323, destination guard
+  4324, timeout 4325, PostgreSQL 15432. Refuse occupied fixed ports and await owned
+  child exits. Java integration contexts may keep their existing OS-assigned
+  loopback ports where already used.
+
+Budget: zero load runs, no automatic retries, no parameter sweeps, at most two
+separately dispatched repair tasks. Stop after the required gates pass.
diff --git a/evidence/E04/logout-baseline.json b/evidence/E04/logout-baseline.json
new file mode 100644
index 0000000..943a237
--- /dev/null
+++ b/evidence/E04/logout-baseline.json
@@ -0,0 +1,16 @@
+{
+  "test": "SessionAuthenticationTest#anonymousCsrfBootstrapIsNotAnAuthenticatedLogout",
+  "runtime": "Spring Security 6.5.11 / Java 21.0.7+6",
+  "processPid": 95335,
+  "ownedSchema": "e04_session",
+  "csrfBootstrapSucceeded": true,
+  "anonymousLogoutActualStatus": 200,
+  "requiredStatus": 401,
+  "requiredCode": "UNAUTHENTICATED",
+  "executions": 1,
+  "result": "REPRODUCED before logout success-handler correction",
+  "testsRun": 1,
+  "failures": 1,
+  "errors": 0,
+  "mavenElapsedSeconds": 6.074
+}
diff --git a/evidence/E04/logout-supplement.md b/evidence/E04/logout-supplement.md
new file mode 100644
index 0000000..487364c
--- /dev/null
+++ b/evidence/E04/logout-supplement.md
@@ -0,0 +1,16 @@
+# E04 anonymous CSRF bootstrap / logout supplement
+
+Frozen before its first execution, following the main agent's static filter-order
+review. No original fixture, product assertion, clock or TTL changes.
+
+Create an anonymous client against the actual Spring Security 6.5.11 servlet
+application in the owned `e04_session` schema. GET `/api/session/csrf`, retaining
+only its runtime cookie and token in memory. POST `/api/session/logout` with that
+anonymous cookie and the valid CSRF header. Required result: 401 with the normal
+UNAUTHENTICATED envelope. A CSRF bootstrap session is not authentication.
+
+Before any correction, execute only this focused case once. The static concern
+is that LogoutFilter precedes AuthorizationFilter and may invoke the existing
+unconditional success handler with no authenticated principal. Fix only after
+the actual result confirms this inference, then include the case in the required
+full regression gate. Evidence records statuses and booleans only.
diff --git a/evidence/E04/maven-dependencies.txt b/evidence/E04/maven-dependencies.txt
new file mode 100644
index 0000000..19e8c31
--- /dev/null
+++ b/evidence/E04/maven-dependencies.txt
@@ -0,0 +1,97 @@
+dev.evolution:monitor-api:jar:0.0.1
++- org.springframework.boot:spring-boot-starter-web:jar:3.5.16:compile
+|  +- org.springframework.boot:spring-boot-starter:jar:3.5.16:compile
+|  |  +- org.springframework.boot:spring-boot:jar:3.5.16:compile
+|  |  +- org.springframework.boot:spring-boot-autoconfigure:jar:3.5.16:compile
+|  |  +- org.springframework.boot:spring-boot-starter-logging:jar:3.5.16:compile
+|  |  |  +- ch.qos.logback:logback-classic:jar:1.5.34:compile
+|  |  |  |  \- ch.qos.logback:logback-core:jar:1.5.34:compile
+|  |  |  +- org.apache.logging.log4j:log4j-to-slf4j:jar:2.24.3:compile
+|  |  |  |  \- org.apache.logging.log4j:log4j-api:jar:2.24.3:compile
+|  |  |  \- org.slf4j:jul-to-slf4j:jar:2.0.18:compile
+|  |  +- jakarta.annotation:jakarta.annotation-api:jar:2.1.1:compile
+|  |  \- org.yaml:snakeyaml:jar:2.4:compile
+|  +- org.springframework.boot:spring-boot-starter-json:jar:3.5.16:compile
+|  |  +- com.fasterxml.jackson.core:jackson-databind:jar:2.21.4:compile
+|  |  |  +- com.fasterxml.jackson.core:jackson-annotations:jar:2.21:compile
+|  |  |  \- com.fasterxml.jackson.core:jackson-core:jar:2.21.4:compile
+|  |  +- com.fasterxml.jackson.datatype:jackson-datatype-jdk8:jar:2.21.4:compile
+|  |  +- com.fasterxml.jackson.datatype:jackson-datatype-jsr310:jar:2.21.4:compile
+|  |  \- com.fasterxml.jackson.module:jackson-module-parameter-names:jar:2.21.4:compile
+|  +- org.springframework.boot:spring-boot-starter-tomcat:jar:3.5.16:compile
+|  |  +- org.apache.tomcat.embed:tomcat-embed-core:jar:10.1.55:compile
+|  |  +- org.apache.tomcat.embed:tomcat-embed-el:jar:10.1.55:compile
+|  |  \- org.apache.tomcat.embed:tomcat-embed-websocket:jar:10.1.55:compile
+|  +- org.springframework:spring-web:jar:6.2.19:compile
+|  |  +- org.springframework:spring-beans:jar:6.2.19:compile
+|  |  \- io.micrometer:micrometer-observation:jar:1.15.12:compile
+|  |     \- io.micrometer:micrometer-commons:jar:1.15.12:compile
+|  \- org.springframework:spring-webmvc:jar:6.2.19:compile
+|     +- org.springframework:spring-context:jar:6.2.19:compile
+|     \- org.springframework:spring-expression:jar:6.2.19:compile
++- org.springframework.boot:spring-boot-starter-data-jpa:jar:3.5.16:compile
+|  +- org.springframework.boot:spring-boot-starter-jdbc:jar:3.5.16:compile
+|  |  +- com.zaxxer:HikariCP:jar:6.3.3:compile
+|  |  \- org.springframework:spring-jdbc:jar:6.2.19:compile
+|  +- org.hibernate.orm:hibernate-core:jar:6.6.53.Final:compile
+|  |  +- jakarta.persistence:jakarta.persistence-api:jar:3.1.0:compile
+|  |  +- jakarta.transaction:jakarta.transaction-api:jar:2.0.1:compile
+|  |  +- org.jboss.logging:jboss-logging:jar:3.6.3.Final:runtime
+|  |  +- org.hibernate.common:hibernate-commons-annotations:jar:7.0.3.Final:runtime
+|  |  +- io.smallrye:jandex:jar:3.2.0:runtime
+|  |  +- com.fasterxml:classmate:jar:1.7.3:runtime
+|  |  +- net.bytebuddy:byte-buddy:jar:1.17.8:runtime
+|  |  +- org.glassfish.jaxb:jaxb-runtime:jar:4.0.9:runtime
+|  |  |  \- org.glassfish.jaxb:jaxb-core:jar:4.0.9:runtime
+|  |  |     +- org.eclipse.angus:angus-activation:jar:2.0.3:runtime
+|  |  |     +- org.glassfish.jaxb:txw2:jar:4.0.9:runtime
+|  |  |     \- com.sun.istack:istack-commons-runtime:jar:4.1.2:runtime
+|  |  +- jakarta.inject:jakarta.inject-api:jar:2.0.1:runtime
+|  |  \- org.antlr:antlr4-runtime:jar:4.13.2:compile
+|  +- org.springframework.data:spring-data-jpa:jar:3.5.13:compile
+|  |  +- org.springframework.data:spring-data-commons:jar:3.5.13:compile
+|  |  +- org.springframework:spring-orm:jar:6.2.19:compile
+|  |  +- org.springframework:spring-tx:jar:6.2.19:compile
+|  |  \- org.slf4j:slf4j-api:jar:2.0.18:compile
+|  \- org.springframework:spring-aspects:jar:6.2.19:compile
+|     \- org.aspectj:aspectjweaver:jar:1.9.25.1:compile
++- org.springframework.boot:spring-boot-starter-security:jar:3.5.16:compile
+|  +- org.springframework:spring-aop:jar:6.2.19:compile
+|  +- org.springframework.security:spring-security-config:jar:6.5.11:compile
+|  |  \- org.springframework.security:spring-security-core:jar:6.5.11:compile
+|  |     \- org.springframework.security:spring-security-crypto:jar:6.5.11:compile
+|  \- org.springframework.security:spring-security-web:jar:6.5.11:compile
++- org.flywaydb:flyway-database-postgresql:jar:11.7.2:compile
+|  \- org.flywaydb:flyway-core:jar:11.7.2:compile
+|     \- com.fasterxml.jackson.dataformat:jackson-dataformat-toml:jar:2.21.4:compile
++- org.postgresql:postgresql:jar:42.7.11:runtime
+\- org.springframework.boot:spring-boot-starter-test:jar:3.5.16:test
+   +- org.springframework.boot:spring-boot-test:jar:3.5.16:test
+   +- org.springframework.boot:spring-boot-test-autoconfigure:jar:3.5.16:test
+   +- com.jayway.jsonpath:json-path:jar:2.9.0:test
+   +- jakarta.xml.bind:jakarta.xml.bind-api:jar:4.0.5:runtime
+   |  \- jakarta.activation:jakarta.activation-api:jar:2.1.4:runtime
+   +- net.minidev:json-smart:jar:2.5.2:test
+   |  \- net.minidev:accessors-smart:jar:2.5.2:test
+   |     \- org.ow2.asm:asm:jar:9.7.1:test
+   +- org.assertj:assertj-core:jar:3.27.7:test
+   +- org.awaitility:awaitility:jar:4.2.2:test
+   +- org.hamcrest:hamcrest:jar:3.0:test
+   +- org.junit.jupiter:junit-jupiter:jar:5.12.2:test
+   |  +- org.junit.jupiter:junit-jupiter-api:jar:5.12.2:test
+   |  |  +- org.opentest4j:opentest4j:jar:1.3.0:test
+   |  |  +- org.junit.platform:junit-platform-commons:jar:1.12.2:test
+   |  |  \- org.apiguardian:apiguardian-api:jar:1.1.2:test
+   |  +- org.junit.jupiter:junit-jupiter-params:jar:5.12.2:test
+   |  \- org.junit.jupiter:junit-jupiter-engine:jar:5.12.2:test
+   |     \- org.junit.platform:junit-platform-engine:jar:1.12.2:test
+   +- org.mockito:mockito-core:jar:5.17.0:test
+   |  +- net.bytebuddy:byte-buddy-agent:jar:1.17.8:test
+   |  \- org.objenesis:objenesis:jar:3.3:test
+   +- org.mockito:mockito-junit-jupiter:jar:5.17.0:test
+   +- org.skyscreamer:jsonassert:jar:1.5.3:test
+   |  \- com.vaadin.external.google:android-json:jar:0.0.20131108.vaadin1:test
+   +- org.springframework:spring-core:jar:6.2.19:compile
+   |  \- org.springframework:spring-jcl:jar:6.2.19:compile
+   +- org.springframework:spring-test:jar:6.2.19:test
+   \- org.xmlunit:xmlunit-core:jar:2.10.4:test
diff --git a/evidence/E04/secrecy-audit.json b/evidence/E04/secrecy-audit.json
new file mode 100644
index 0000000..8ef732c
--- /dev/null
+++ b/evidence/E04/secrecy-audit.json
@@ -0,0 +1,7 @@
+{
+  "fixtureSha256": "fa9d3babe27ae625665468994674ef7a1a6afdb5e7d4bfabcc7e261984ac2aa2",
+  "logCredentialPatternMatches": 0,
+  "evidenceCredentialPatternMatches": 0,
+  "generatedPasswordMessagePresent": false,
+  "priorFailurePasswordSnapshotValues": 0
+}
diff --git a/evidence/E04/session-lifecycle.json b/evidence/E04/session-lifecycle.json
new file mode 100644
index 0000000..2571c0b
--- /dev/null
+++ b/evidence/E04/session-lifecycle.json
@@ -0,0 +1,19 @@
+{
+  "anonymousStatus" : 401,
+  "httpOnlyCookie" : true,
+  "sameSiteLax" : true,
+  "cookieOnlyTracking" : true,
+  "anonymousSessionRotated" : true,
+  "invalidMissingAndOversizeLoginStatus" : 401,
+  "unknownAndWrongPasswordSameEnvelope" : true,
+  "reloginRotated" : true,
+  "oldIdentifierStatus" : 401,
+  "frameworkCsrfStillEnforced" : true,
+  "logoutCookieCleared" : true,
+  "revokedStatus" : 401,
+  "ttlMs" : 3600000,
+  "fixedStart" : "2026-08-28T01:00:00Z",
+  "beforeExpiryStatus" : 200,
+  "exactExpiryStatus" : 401,
+  "bobIndependentSessionStatus" : 200
+}
diff --git a/evidence/E04/user-persistence.json b/evidence/E04/user-persistence.json
new file mode 100644
index 0000000..0e9f79e
--- /dev/null
+++ b/evidence/E04/user-persistence.json
@@ -0,0 +1,3 @@
+{"users":2,"bcryptCost":10,"plaintextAtRest":false,"independentSalts":true,
+ "bootstrapRollback":true,"idempotentBootstrap":true,"reopenedContextAuthentication":true,
+ "authenticatedCredentialsErased":true,"allSqlTransactional":true,"v2UpgradeApplied":1,"repeatApplied":0}
diff --git a/evidence/E04/user-sql.txt b/evidence/E04/user-sql.txt
new file mode 100644
index 0000000..457853e
--- /dev/null
+++ b/evidence/E04/user-sql.txt
@@ -0,0 +1,16 @@
+Event[sql=select ue1_0.id,ue1_0.password_hash,ue1_0.username from e04_users.users ue1_0 where ue1_0.username=?, transaction=true, readOnly=false]
+Event[sql=insert into e04_users.users (password_hash,username,id) values (?,?,?), transaction=true, readOnly=false]
+Event[sql=select ue1_0.id,ue1_0.password_hash,ue1_0.username from e04_users.users ue1_0 where ue1_0.username=?, transaction=true, readOnly=false]
+Event[sql=insert into e04_users.users (password_hash,username,id) values (?,?,?), transaction=true, readOnly=false]
+Event[sql=select ue1_0.id,ue1_0.password_hash,ue1_0.username from e04_users.users ue1_0 where ue1_0.username=?, transaction=true, readOnly=false]
+Event[sql=insert into e04_users.users (password_hash,username,id) values (?,?,?), transaction=true, readOnly=false]
+Event[sql=select ue1_0.id,ue1_0.password_hash,ue1_0.username from e04_users.users ue1_0 where ue1_0.username=?, transaction=true, readOnly=false]
+Event[sql=insert into e04_users.users (password_hash,username,id) values (?,?,?), transaction=true, readOnly=false]
+Event[sql=select ue1_0.id,ue1_0.password_hash,ue1_0.username from e04_users.users ue1_0 where ue1_0.username=?, transaction=true, readOnly=true]
+Event[sql=select ue1_0.id,ue1_0.password_hash,ue1_0.username from e04_users.users ue1_0 where ue1_0.username=?, transaction=true, readOnly=false]
+Event[sql=select ue1_0.id,ue1_0.password_hash,ue1_0.username from e04_users.users ue1_0 where ue1_0.username=?, transaction=true, readOnly=false]
+Event[sql=select ue1_0.id,ue1_0.password_hash,ue1_0.username from e04_users.users ue1_0 where ue1_0.username=?, transaction=true, readOnly=true]
+Event[sql=select ue1_0.id,ue1_0.password_hash,ue1_0.username from e04_users.users ue1_0 where ue1_0.username=?, transaction=true, readOnly=true]
+Event[sql=select ue1_0.id,ue1_0.password_hash,ue1_0.username from e04_users.users ue1_0 where ue1_0.username=?, transaction=true, readOnly=true]
+Event[sql=select ue1_0.id,ue1_0.password_hash,ue1_0.username from e04_users.users ue1_0 where ue1_0.username=?, transaction=true, readOnly=true]
+Event[sql=select ue1_0.id,ue1_0.password_hash,ue1_0.username from e04_users.users ue1_0 where ue1_0.username=?, transaction=true, readOnly=true]
diff --git a/evidence/E04/verification.md b/evidence/E04/verification.md
new file mode 100644
index 0000000..f182973
--- /dev/null
+++ b/evidence/E04/verification.md
@@ -0,0 +1,130 @@
+# E04 actual verification
+
+Branch `track/industry-spring`, attempt 1.
+START `45711aaa58b311065e0cea423bc53d283e1d4fa9`.
+Spec revision `0a006589477f8ae47bad3faa5510c999cff85ee4`.
+
+Frozen fixture SHA-256:
+`fa9d3babe27ae625665468994674ef7a1a6afdb5e7d4bfabcc7e261984ac2aa2`.
+It was written before the first baseline and remains byte-identical. No old
+product input/assertion, E01–E03 committed evidence, V1/V2 migration or runtime
+version pin changed. Auth setup uses only independent runtime-generated passwords.
+
+## Unchanged baseline and setup
+
+`mvn -B -ntp -f backend/pom.xml -DskipTests package` built the unchanged START
+(exit 0, Maven 1.159 seconds). `fnm exec --using 24.19.0 node
+scripts/e04-baseline.mjs` then ran exactly once. The owned API PID 87577 logged its
+startup before the sole anonymous `GET /api/monitors`; response was 200 and count
+0, versus the required 401 / UNAUTHENTICATED. There was no collection readiness
+request. The API exit was awaited, and the private schema was dropped. See
+`baseline.json` for the actual timestamps and process evidence.
+
+The initial implementation patch that disabled CSRF was rejected and unapplied.
+Following the main agent's clarification, default Spring Security CSRF protection
+was retained with only the token transport needed by E04. See
+`csrf-clarification.md`; the rejection was not bypassed.
+
+Resolving the newly required Security starter initially failed at sandbox DNS
+before compilation (exit 1, Maven 0.594 seconds). The same compile command with
+approved network access succeeded (exit 0, Maven 4.779 seconds), without executing
+tests. `mvn -B -ntp -f backend/pom.xml dependency:tree
+-DoutputFile=target/e04-maven-dependencies.txt` succeeded (0.870 seconds).
+`maven-dependencies.txt` records starter 3.5.16 and Security core/config/web/crypto
+6.5.11 from the unchanged Boot BOM. No other dependency or runtime was upgraded.
+
+## Confirmed logout filter boundary
+
+The separately frozen `logout-supplement.md` case was executed once before its
+correction:
+
+`mvn -B -ntp -f backend/pom.xml
+'-Dtest=SessionAuthenticationTest#anonymousCsrfBootstrapIsNotAnAuthenticatedLogout'
+test` exited 1 (Maven 6.074 seconds). One actual HTTP test failed with 200 instead
+of 401: an anonymous CSRF bootstrap session reached the logout success handler.
+No other lifecycle/expiry test ran in this invocation. `logout-baseline.json`
+records the safe result. The handler now checks the Authentication provided by
+LogoutFilter; the full Java gate subsequently passed this same case. Default
+CSRF remained enabled throughout the applied implementation.
+
+## Required gates
+
+The full `/usr/bin/time -p fnm exec --using 24.19.0 npm run verify` invocation
+exited 1 after 88.87 seconds, solely at the new browser locator. Its complete
+per-command statuses/times are preserved in `first-verification.json`.
+
+| Gate | Actual result | Seconds |
+| --- | --- | ---: |
+| Own PostgreSQL setup | healthy, exit 0 | 0.807 |
+| Maven package | 30 tests, zero failures/errors; package built | 16.765 |
+| TypeScript | passed | 1.769 |
+| Next production build | passed, including `/login` and `/monitors` | 11.778 |
+| Occupied-port guard | refused before requests/children; wrapper passed | 0.165 |
+| Private restart schema setup | passed | 0.288 |
+| Original A,A,B process restart and lifecycle | passed with auth setup | 20.311 |
+| First Chromium invocation | 11 passed, 1 locator failure, retries 0 | 36.387 |
+| Both runner schema cleanup commands | passed | 0.419 |
+
+Java results: the original 24 cases still pass, plus 3 account-persistence cases
+and 3 session cases. Tests exercise the actual servlet filter chain, not mocked
+authentication annotations. Missing/invalid credentials fail before all protected
+Monitor/check/history methods, including malformed product input. The fixed
+two-user sequence proves login, generic wrong/unknown/missing credential errors,
+anonymous-session and authenticated-session identifier rotation, invalidation of
+the old identifier, logout cookie deletion, revocation, and independent Bob login.
+At exactly 3599999 ms the fresh Alice session still returns 200; at 3600000 ms it
+returns 401 without a sleep or time search. An authenticated request lacking the
+default CSRF token remains 403. See `session-lifecycle.json`.
+
+Real PostgreSQL tests verify V2→V3 upgrade, fresh/repeated migrations, rollback of
+flushed bootstrap inserts, idempotent bootstrap, two salted bcrypt cost-10 stored
+representations, authentication after closing and reopening the application,
+and erasure of password/hash material from authenticated principals. Captured
+SQL contains parameter placeholders only and proves actual write/read-only
+transactions. Both missing and extra-required account columns prevent web-server
+readiness. Existing Monitor schema, UTC/null mapping, transaction, cascade and
+NUL/non-finite input tests remain intact. See `user-persistence.json` and
+`user-sql.txt`.
+
+The original restart product sequence used distinct owned API PIDs 6974 and 7049,
+each with its own startup marker and awaited exit. Complete Monitor/CheckRun
+values survived restart; edits, pause, activation, deletion and NUL rejection
+passed. Credential headers and CSRF setup responses were never captured. Safe
+process/count evidence is in `authorized-regressions.json`; full product-only wire
+output remains in ignored `output/e03/fixed-persistence.json`.
+
+## Browser locator correction and final result
+
+The first E04 invalid-login assertion matched both the application's correct
+UNAUTHENTICATED alert and Next's route announcer. The main agent authorized a
+locator-only correction before final candidate reporting. The two new locators
+were scoped to `main`, using the existing E02 pattern. No product file, input,
+action, expected message, expiry fixture or threshold changed after the full gate.
+
+Pinned Playwright 1.62.1 uses `PLAYWRIGHT_NO_COPY_PROMPT` to skip automatic DOM/ARIA
+failure snapshots. That flag is now set, and `preserveOutput: 'never'` prevents
+retained per-test error-context artifacts; trace, screenshots and video remain
+off. The previous error context and logs were inspected without printing values:
+no populated password snapshot line or credential pattern was present.
+
+The directed browser-only invocation
+`/usr/bin/time -p fnm exec --using 24.19.0 npm run test:e2e` exited 0 in 18.85
+seconds: **12/12 passed**, one worker, zero retries. This includes real invalid-login
+recovery, HttpOnly/SameSite browser transport, login rotation and old-cookie
+rejection, logout/replay rejection, Bob login and UI recovery after revocation,
+plus all ten unchanged earlier product/error cases. No per-test artifacts were
+retained. See `browser-verification.json` and `secrecy-audit.json`.
+
+After the standalone browser invocation, Playwright had stopped all child
+listeners but left the private `e04_browser` schema. Explicit
+`fnm exec --using 24.19.0 node scripts/database.mjs drop e04_browser` removed it
+(exit 0). A final own-container SQL count confirmed zero `e04_*` schemas; ports
+4321–4325 had no listeners. PostgreSQL remains running for main verification;
+public data, the owned persistent volume and unrelated resources are preserved.
+The standard full runner retains its unconditional schema-cleanup fallback.
+
+Budget: one unchanged baseline; one focused logout counterexample; one full gate;
+one directed browser-only correction check. Zero load runs, automatic retries,
+parameter sweeps or separately dispatched repair tasks. Compile/dependency and
+cleanup operations are recorded separately. No hosted CI run is claimed.
+All required gates are confirmed; no additional scenarios were run afterward.
diff --git a/scripts/e04-baseline.mjs b/scripts/e04-baseline.mjs
new file mode 100644
index 0000000..6456d5b
--- /dev/null
+++ b/scripts/e04-baseline.mjs
@@ -0,0 +1,82 @@
+import assert from 'node:assert/strict';
+import { spawn, spawnSync } from 'node:child_process';
+import { once } from 'node:events';
+import { createHash } from 'node:crypto';
+import { closeSync, existsSync, mkdirSync, openSync, readFileSync, writeFileSync } from 'node:fs';
+import { createServer } from 'node:net';
+import { setTimeout as delay } from 'node:timers/promises';
+
+const start = '45711aaa58b311065e0cea423bc53d283e1d4fa9';
+const directory = 'output/e04';
+const output = `${directory}/baseline.json`;
+assert.ok(!existsSync(output), 'The unchanged baseline must not be repeated');
+assert.equal(spawnSync('git', ['rev-parse', 'HEAD'], { encoding: 'utf8' }).stdout.trim(), start);
+assert.equal(spawnSync('git', ['diff', 'HEAD', '--name-only'], { encoding: 'utf8' }).stdout.trim(), '');
+mkdirSync(directory, { recursive: true });
+const evidence = { start, fixtureSha256: createHash('sha256').update(readFileSync('evidence/E04/fixtures.md')).digest('hex'),
+  requests: [], startedAt: new Date().toISOString() };
+const probe = createServer();
+await new Promise((resolve, reject) => {
+  probe.once('error', () => reject(new Error('Refusing occupied API port 4322')));
+  probe.listen({ host: '127.0.0.1', port: 4322, exclusive: true }, () => probe.close(resolve));
+});
+evidence.portFree = true;
+function sql(query) {
+  const result = spawnSync('docker', ['compose', '--project-name', 'wse-industry', '--file', 'compose.yaml',
+    'exec', '--no-TTY', 'postgres', 'psql', '--username', 'wse_industry', '--dbname', 'monitor',
+    '--set', 'ON_ERROR_STOP=1', '--command', query], { encoding: 'utf8' });
+  assert.equal(result.status, 0, 'Owned database schema operation failed');
+}
+let api;
+let ownedSchema = false;
+let exitCode = 0;
+try {
+  sql('CREATE SCHEMA e04_baseline');
+  ownedSchema = true;
+  const logPath = `${directory}/baseline-api.log`;
+  const log = openSync(logPath, 'w');
+  api = spawn('java', ['-jar', 'backend/target/monitor-api-0.0.1.jar'], {
+    env: { ...process.env, DB_SCHEMA: 'e04_baseline' }, stdio: ['ignore', log, log],
+  });
+  closeSync(log);
+  evidence.pid = api.pid;
+  const alive = () => assert.ok(api.pid && api.exitCode === null && api.signalCode === null, 'Owned API exited');
+  api.once('error', () => { evidence.spawnError = true; });
+  const deadline = Date.now() + 30_000;
+  while (!readFileSync(logPath, 'utf8').includes('Started MonitorApplication')) {
+    alive();
+    assert.ok(Date.now() < deadline, 'Owned API did not start in bounded readiness interval');
+    await delay(100);
+  }
+  alive();
+  evidence.ownStartupMarker = true;
+  evidence.readyAt = new Date().toISOString();
+  const response = await fetch('http://127.0.0.1:4322/api/monitors', { signal: AbortSignal.timeout(5000) });
+  const body = await response.json();
+  evidence.requests.push({ method: 'GET', path: '/api/monitors', anonymous: true, status: response.status,
+    collectionCount: Array.isArray(body.data) ? body.data.length : null });
+  alive();
+  assert.equal(response.status, 200);
+  assert.deepEqual(body, { data: [] });
+  evidence.requiredStatus = 401;
+  evidence.requiredCode = 'UNAUTHENTICATED';
+  evidence.result = 'REPRODUCED: anonymous collection read is authorized at unchanged START';
+} catch (error) {
+  exitCode = 1;
+  evidence.result = `FAILED: ${error.message}`;
+} finally {
+  if (api && api.exitCode === null && api.signalCode === null) {
+    const exited = once(api, 'exit');
+    api.kill('SIGTERM');
+    const force = setTimeout(() => api.kill('SIGKILL'), 5000);
+    await exited;
+    clearTimeout(force);
+    evidence.exitAwaited = true;
+  }
+  if (api) Object.assign(evidence, { exitCode: api.exitCode, signal: api.signalCode });
+  if (ownedSchema) { sql('DROP SCHEMA e04_baseline CASCADE'); evidence.schemaDropped = true; }
+  evidence.finishedAt = new Date().toISOString();
+  writeFileSync(output, `${JSON.stringify(evidence, null, 2)}\n`);
+}
+console.log(JSON.stringify(evidence, null, 2));
+process.exitCode = exitCode;
