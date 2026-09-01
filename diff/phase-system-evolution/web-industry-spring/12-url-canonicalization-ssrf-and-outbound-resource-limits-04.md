## `docs(e12): preserve repair evidence and outbound contract`

diff --git a/TRACK.md b/TRACK.md
index 881b7dc..3e39bf2 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -4,7 +4,7 @@ Profile: `phase-1` (E01–E12 → E20 → profile E24; phase-2 deferred, E25 not
 
 Spec revision: `2ada57a71cd34fa2fae9809415c362a8bbfcdf02`
 
-E10 is a local development product: Next.js/React renders login, logout, Monitor lifecycle and Check history from request-scoped server data. Spring Security authenticates browser sessions; PostgreSQL stores salted account passwords, Monitors and Check execution intents/results. Manual requests have durable owner-scoped identities, and competing workers claim one execution owner before I/O. Monitor and CheckRun access is scoped to the signed-in user. There is no signup, Redis, broker or production application container.
+E12 is a local development product: Next.js/React renders login, logout, Monitor lifecycle and Check history from request-scoped server data. Spring Security authenticates browser sessions; PostgreSQL stores salted account passwords, Monitors and Check execution intents/results. Manual requests have durable owner-scoped identities, and competing workers claim one execution owner before bounded HTTP/HTTPS I/O. Monitor and CheckRun access is scoped to the signed-in user. There is no signup, Redis, broker or production application container.
 
 ## Pinned toolchain
 
@@ -49,17 +49,24 @@ existing credentials fail rather than silently resetting an account. Do not put
 secret values in shell history, command arguments, committed files, or logs.
 Verification generates its own independent random passwords automatically.
 
-Start each process in a separate terminal:
+For the controlled local fixture demo, start each process in a separate terminal.
+The API and worker both need the explicit trusted test configuration shown here:
 
 ```sh
 npm run fixture
-npm run api:dev
-npm run worker
+ALLOW_TEST_FIXTURE=true npm run api:dev
+ALLOW_TEST_FIXTURE=true npm run worker
 npm run dev
 ```
 
 Open [Monitors](http://127.0.0.1:4323/monitors), sign in with a prepared account, then create `Fixture monitor` with URL `http://127.0.0.1:4321/ok`, interval `60`, enabled checked. Click **Run check** to accept a QUEUED execution; the worker progresses it through RUNNING to `SUCCEEDED / HTTP 200`. `/fail` yields `FAILED / HTTP 503`. Enabled Monitors also receive scheduled intents at their interval.
 
+`ALLOW_TEST_FIXTURE` defaults to false. The exception permits only the exact
+configured loopback HTTP fixture origin; a request cannot enable it. The browser's
+historical fixture helper copy is unchanged and describes this explicit demo
+configuration. The current API also accepts canonical public HTTP/HTTPS URLs;
+general monitoring does not require or enable the fixture exception.
+
 All defaults bind to `127.0.0.1`. Fixture port is 4321, API port 4322, UI port 4323, PostgreSQL port 15432. `FIXTURE_PORT`, `FIXTURE_ORIGIN`, and `API_PORT` can configure the server processes; `API_ORIGIN` configures the Next API proxy. The committed E01 tests use the fixed default ports; stop local servers before verification. Tests also reserve 4324 for a forbidden destination trap and 4325 for timeout/connection fixtures.
 
 The compose project is exclusively `wse-industry`, with its own bridge network and persistent volume. It uses explicitly nonsecret local test trust authentication and a loopback-only published port. An internal-only Docker network cannot provide the required published port on the verified Docker Desktop runtime. Never use this compose configuration for production or put unrelated data in it. `npm run db:down` stops the project but preserves data; `npm run db:up` restores it. `npm run db:destroy` explicitly deletes this project's disposable database volume.
@@ -68,10 +75,10 @@ The default connection is database `monitor`, local test identity `wse_industry`
 
 ## Check boundary
 
-- Only HTTP URLs matching the configured fixture hostname and explicit port are accepted, both on Monitor creation and at the outbound boundary. Credentials and fragments are rejected. A hostname alias is not treated as the configured host.
-- Checks send one GET with no request body, use no proxy, and never follow redirects. `/redirect` therefore records `FAILED / 302`, with no request to `/ok`.
-- Connect timeout is 1 second and response-header read timeout is 2 seconds. No response body is materialized or retained. This is a controlled fixture implementation, not general Internet monitoring or general SSRF defense.
-- `200..299` is `SUCCEEDED`. Other observed HTTP statuses are `FAILED / HTTP_STATUS`. No HTTP response produces a null status and `TIMEOUT` or `CONNECTION_FAILURE`; no synthetic status is invented.
+- Monitor URLs are canonical absolute HTTP/HTTPS URLs. Credentials, fragments, ambiguous numeric hosts and unsafe literal destinations are rejected without create-time DNS. The explicit fixture exception remains restricted to its exact host, port and HTTP scheme.
+- Every outbound hop validates all actual IPv4/IPv6 DNS answers and connects to a validated address without a second hostname lookup. Checks send GET with no request body or proxy and follow at most three revalidated redirects. `/redirect` now observes the final `/ok` response; the fixed `/redirect-outside` hop to loopback4324 is refused before connection.
+- Connect and read limits are each500ms, with one1500ms deadline covering DNS, connect, TLS and all redirects. Header storage is bounded at65536 bytes per hop; observation stops at final headers and consumes zero body bytes. Each runner has at most one I/O thread and no queued tasks.
+- Final `200..299` is `SUCCEEDED`; other final HTTP statuses are `FAILED / HTTP_STATUS`. Transport timeout or connection failure before final headers is `FAILED` with null status. A policy refusal after acceptance is `ABORTED` with null HTTP status, latency and failure reason, with only a fixed policy category logged. No intermediate or synthetic HTTP outcome is invented.
 - Monitors, queued intents and completed results survive API restarts. Enabled controls scheduling; a paused Monitor can still be checked manually.
 
 ## HTTP contract (E02)
@@ -79,7 +86,7 @@ The default connection is database `monitor`, local test identity `wse_industry`
 - Create input must be a JSON object with string name and URL, a numeric integer interval, and boolean enabled. Required fields cannot be null or omitted; scalar strings/numbers/booleans are not coerced into other types.
 - Names are stripped of surrounding whitespace and must contain 1–100 UTF-16 code units. Interval is 1–86400 seconds inclusive; the JSON number `60.0` is the integer value 60, but the string `"60"` is invalid.
 - E03 also rejects a NUL character in a name before persistence, because PostgreSQL text cannot store it. Create and replacement use the same runtime validator, including the existing non-finite numeric rejection.
-- URL syntax must be absolute HTTP(S). The existing fixture policy then restricts it to the configured HTTP hostname and port, without credentials or fragments. This does not enable general public or HTTPS monitoring.
+- URL syntax must be absolute HTTP(S). Canonicalization lowercases scheme/host, removes default ports, normalizes dot segments and uses `/` for an empty path, preserving the query. The current outbound policy above applies to general public destinations and the explicit local test exception.
 - Successful responses contain exactly `{ "data": <payload> }`. Create returns201; a new Check acceptance returns202/QUEUED; retransmission returns202 with that same execution's current state; reads return200. The existing CheckRun fields remain, with null execution/result fields before they have been observed.
 - API failures contain `{ "error": { "code": "...", "message": "..." } }`: INVALID_INPUT / 400, UNAUTHENTICATED / 401, FORBIDDEN / 403, NOT_FOUND / 404, CONFLICT / 409, or INTERNAL_ERROR / 500. Malformed JSON and IDs use INVALID_INPUT after authentication. Unexpected exception details are not returned.
 - Endpoint HTTP 503 is still an API success containing a FAILED CheckRun with HTTP_STATUS; timeout and connection failure retain null httpStatus. API errors never fabricate a CheckRun.
@@ -308,7 +315,7 @@ The existing article/history serves as detail; no routes or pagination were adde
 The API only validates ownership and commits a PostgreSQL QUEUED CheckRun before
 returning202. `npm run worker` starts the same packaged application with the
 `worker` profile and no HTTP server. It reads one queued execution, commits
-RUNNING, performs the existing fixture-only GET outside a transaction, then
+RUNNING, performs the outbound GET outside a transaction, then
 updates that same ID with its observed terminal result. The browser polls only
 its visible active execution identities and invalidates their terminal history
 when they finish; other form controls remain usable after acceptance.
@@ -394,7 +401,8 @@ ThreadPoolTaskScheduler stops periodic work and drains the current task before
 JPA is destroyed, with a3000ms lifecycle-phase bound in the worker profile. If
 the current operation cannot finish before shutdown, its uncommitted/unknown
 outcome remains recoverable through the lease; shutdown does not invent a result.
-The original1000ms connect/2000ms response timeouts remain unchanged.
+E11 retained the original1000ms connect/2000ms response timeouts. E12 replaces
+them with the bounded outbound contract above.
 
 The capped crash proof is opt-in:
 `mvn -B -ntp -f backend/pom.xml -Dtest=WorkerRecoveryTest -De11.process-proof=true test`.
@@ -406,3 +414,50 @@ silently repeat these fault scenarios. The separate browser regression seeds
 one expired RUNNING row and starts the normal worker to verify current view,
 cached history and reload with the same ABORTED ID. This seed is not a replacement
 for real-process crash evidence.
+
+## Outbound result and verification boundary (E12)
+
+Monitor creation stores the canonical URL without DNS or outbound I/O. Actual
+execution checks every resolved address and every redirect destination. The JDK
+connector uses that validated InetAddress; HTTPS keeps the original hostname for
+SNI and HTTPS endpoint identification. A deadline cancels the active socket and
+prevents a resolver that returns late from opening a connection. An uninterruptible
+resolver occupies the single I/O slot until it exits; another intent is rejected
+without adding a thread, queue entry or endpoint result.
+
+The following classifications describe failures; they do not schedule retries or
+change an existing terminal execution:
+
+| Observed condition | Classification and result |
+| --- | --- |
+| `TIMEOUT`, `CONNECTION_FAILURE` before final headers | Potentially retryable by a later explicit new execution; current result is `FAILED` with null HTTP status. |
+| HTTP503 in the retained endpoint regression | Transient endpoint failure that can be checked by a later new execution; current result remains `FAILED / 503 / HTTP_STATUS`. |
+| Invalid URL, unsafe address/answer, invalid redirect, redirect/header resource limit | Permanent policy refusal for that input/attempt. Input rejection is400 before enqueue; a refusal during execution is `ABORTED` with null endpoint fields. Only the fixed policy reason is logged. |
+| Executor rejection, interrupted service work or authoritative-store uncertainty | No fabricated terminal endpoint result and no internal retry. Durable ownership, fencing and recovery remain governed by E11. |
+
+Other final HTTP responses retain the status-only success rule. A final response
+needs no body-content assertion: an offered65537-byte body is closed after its
+headers, with zero body bytes consumed. Closing remaining I/O does not turn an
+observed200 into a failure. No automatic retry or terminal rollback was added.
+
+E12 evidence is under `evidence/phase-1/E12`. The unchanged baseline is retained:
+one public-shaped Monitor create was rejected400 with no durable rows or outbound
+call. Attempt1's28-method gate had27 passes and one timing-observer failure; its
+raw log, partial observations and all three JUnit reports are retained under
+`repair1/attempt1`. The historical run-return duration was not recorded, so cold
+serialization overhead remains a possible explanation, not a proven cause.
+
+Repair1 changes only observation timing. Existing execution/runner setup precedes
+the timer, return duration is captured immediately after `run()`, and blocked-DNS
+evidence is written before assertions; cleanup is recorded separately. The single
+authorized repaired `CheckRunnerTest` package gate passed11/11, including the
+unchanged `<1750ms` total bound and finite-capacity/late-I/O assertions. Unchanged
+functional16 and API-error1 results are reused with source hashes, not reported as
+reruns. The independent root acceptance gate is separate and pending in this
+author evidence.
+
+Public-address, unsafe-address and rebinding tests use deterministic resolver/connector
+stubs; actual HTTP I/O uses the controlled local4325 fixtures. The public IPv6 HTTPS case
+and TLS parameter assertions do not claim a live TLS handshake. No public network
+or metadata endpoint was contacted, and no baseline, E11 crash scenario, load run,
+parameter sweep or automatic retry was repeated during this repair.
diff --git a/evidence/phase-1/E12/repair1/attempt1/invocations.jsonl b/evidence/phase-1/E12/repair1/attempt1/invocations.jsonl
new file mode 100644
index 0000000..f7c99e0
--- /dev/null
+++ b/evidence/phase-1/E12/repair1/attempt1/invocations.jsonl
@@ -0,0 +1,2 @@
+{"command":"node scripts/e12-baseline.mjs","startedAt":"2026-08-28T06:55:52.514Z","elapsedSeconds":6.834,"exitCode":1}
+{"command":"mvn -B -ntp -f backend/pom.xml -Dtest=CheckRunnerTest,MonitorFunctionalTest,ApiErrorBoundaryTest -Dsurefire.skipAfterFailureCount=1 package","startedAt":"2026-08-28T07:15:17.851Z","elapsedSeconds":12.648,"exitCode":1,"signal":null}
diff --git a/evidence/phase-1/E12/repair1/attempt1/maven-console.txt b/evidence/phase-1/E12/repair1/attempt1/maven-console.txt
new file mode 100644
index 0000000..28c3e8e
--- /dev/null
+++ b/evidence/phase-1/E12/repair1/attempt1/maven-console.txt
@@ -0,0 +1,161 @@
+[INFO] Scanning for projects...
+[INFO] 
+[INFO] ---------------------< dev.evolution:monitor-api >----------------------
+[INFO] Building monitor-api 0.0.1
+[INFO]   from pom.xml
+[INFO] --------------------------------[ jar ]---------------------------------
+[INFO] 
+[INFO] --- enforcer:3.6.2:enforce (pinned-runtimes) @ monitor-api ---
+[INFO] Rule 0: org.apache.maven.enforcer.rules.version.RequireJavaVersion passed
+[INFO] Rule 1: org.apache.maven.enforcer.rules.version.RequireMavenVersion passed
+[INFO] 
+[INFO] --- resources:3.3.1:resources (default-resources) @ monitor-api ---
+[INFO] Copying 2 resources from src/main/resources to target/classes
+[INFO] Copying 8 resources from src/main/resources to target/classes
+[INFO] 
+[INFO] --- compiler:3.14.1:compile (default-compile) @ monitor-api ---
+[INFO] Recompiling the module because of changed source code.
+[INFO] Compiling 22 source files with javac [debug parameters release 21] to target/classes
+[INFO] 
+[INFO] --- resources:3.3.1:testResources (default-testResources) @ monitor-api ---
+[INFO] skip non existing resourceDirectory /private/tmp/web-systems-evolution-0a006589-industry-spring/backend/src/test/resources
+[INFO] 
+[INFO] --- compiler:3.14.1:testCompile (default-testCompile) @ monitor-api ---
+[INFO] Recompiling the module because of changed dependency.
+[INFO] Compiling 17 source files with javac [debug parameters release 21] to target/test-classes
+[INFO] 
+[INFO] --- surefire:3.5.6:test (default-test) @ monitor-api ---
+[INFO] Using auto detected provider org.apache.maven.surefire.junitplatform.JUnitPlatformProvider
+[INFO] 
+[INFO] -------------------------------------------------------
+[INFO]  T E S T S
+[INFO] -------------------------------------------------------
+[INFO] Running dev.evolution.monitor.CheckRunnerTest
+16:15:21.455 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:15:21.457 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:15:21.458 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:15:21.458 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:15:21.459 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:15:21.459 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:15:21.460 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:15:21.460 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:15:21.461 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+[WARNING] An attempt was made to cancel the current test run due to the configured skipAfterFailureCount of 1. However, the version of JUnit Platform on the runtime classpath does not support cancellation. Please update to 6.0.0 or later!
+16:15:23.301 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_ADDRESS
+16:15:24.841 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: REDIRECT_LIMIT
+[ERROR] Tests run: 11, Failures: 1, Errors: 0, Skipped: 0, Time elapsed: 4.015 s <<< FAILURE! -- in dev.evolution.monitor.CheckRunnerTest
+[ERROR] dev.evolution.monitor.CheckRunnerTest.uninterruptibleDnsRemainsBoundedAndCannotConnectAfterTheDeadline -- Time elapsed: 1.776 s <<< FAILURE!
+org.opentest4j.AssertionFailedError: expected: <true> but was: <false>
+	at org.junit.jupiter.api.AssertionFailureBuilder.build(AssertionFailureBuilder.java:151)
+	at org.junit.jupiter.api.AssertionFailureBuilder.buildAndThrow(AssertionFailureBuilder.java:132)
+	at org.junit.jupiter.api.AssertTrue.failNotTrue(AssertTrue.java:63)
+	at org.junit.jupiter.api.AssertTrue.assertTrue(AssertTrue.java:36)
+	at org.junit.jupiter.api.AssertTrue.assertTrue(AssertTrue.java:31)
+	at org.junit.jupiter.api.Assertions.assertTrue(Assertions.java:183)
+	at dev.evolution.monitor.CheckRunnerTest.uninterruptibleDnsRemainsBoundedAndCannotConnectAfterTheDeadline(CheckRunnerTest.java:279)
+	at java.base/java.lang.reflect.Method.invoke(Method.java:580)
+	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
+	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
+
+[INFO] Running dev.evolution.monitor.ApiErrorBoundaryTest
+Mockito is currently self-attaching to enable the inline-mock-maker. This will no longer work in future releases of the JDK. Please add Mockito as an agent to your build as described in Mockito's documentation: https://javadoc.io/doc/org.mockito/mockito-core/latest/org.mockito/org/mockito/Mockito.html#0.3
+OpenJDK 64-Bit Server VM warning: Sharing is only supported for boot loader classes because bootstrap classpath has been appended
+WARNING: A Java agent has been loaded dynamically (/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy-agent/1.17.8/byte-buddy-agent-1.17.8.jar)
+WARNING: If a serviceability tool is in use, please run with -XX:+EnableDynamicAgentLoading to hide this warning
+WARNING: If a serviceability tool is not in use, please run with -Djdk.instrument.traceUsage for more information
+WARNING: Dynamic loading of agents will be disallowed by default in a future release
+16:15:26.086 [main] INFO org.springframework.mock.web.MockServletContext -- Initializing Spring TestDispatcherServlet ''
+16:15:26.086 [main] INFO org.springframework.test.web.servlet.TestDispatcherServlet -- Initializing Servlet ''
+16:15:26.087 [main] INFO org.springframework.test.web.servlet.TestDispatcherServlet -- Completed initialization in 1 ms
+[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.766 s -- in dev.evolution.monitor.ApiErrorBoundaryTest
+[INFO] Running dev.evolution.monitor.MonitorFunctionalTest
+16:15:26.186 [main] INFO org.springframework.test.context.support.AnnotationConfigContextLoaderUtils -- Could not detect default configuration classes for test class [dev.evolution.monitor.MonitorFunctionalTest]: MonitorFunctionalTest does not declare any static, non-private, non-final, nested classes annotated with @Configuration.
+16:15:26.256 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper -- Found @SpringBootConfiguration dev.evolution.monitor.MonitorApplication for test class dev.evolution.monitor.MonitorFunctionalTest
+
+  .   ____          _            __ _ _
+ /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
+( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
+ \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
+  '  |____| .__|_| |_|_| |_\__, | / / / /
+ =========|_|==============|___/=/_/_/_/
+
+ :: Spring Boot ::               (v3.5.16)
+
+2026-08-28T16:15:26.500+09:00  INFO 1550 --- [monitor-api] [           main] d.e.monitor.MonitorFunctionalTest        : Starting MonitorFunctionalTest using Java 21.0.7 with PID 1550 (started by woopinbell in /private/tmp/web-systems-evolution-0a006589-industry-spring/backend)
+2026-08-28T16:15:26.501+09:00  INFO 1550 --- [monitor-api] [           main] d.e.monitor.MonitorFunctionalTest        : No active profile set, falling back to 1 default profile: "default"
+2026-08-28T16:15:26.839+09:00  INFO 1550 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT mode.
+2026-08-28T16:15:26.854+09:00  INFO 1550 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 9 ms. Found 0 JPA repository interfaces.
+2026-08-28T16:15:27.122+09:00  INFO 1550 --- [monitor-api] [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port 4322 (http)
+2026-08-28T16:15:27.131+09:00  INFO 1550 --- [monitor-api] [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
+2026-08-28T16:15:27.131+09:00  INFO 1550 --- [monitor-api] [           main] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.1.55]
+2026-08-28T16:15:27.168+09:00  INFO 1550 --- [monitor-api] [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
+2026-08-28T16:15:27.169+09:00  INFO 1550 --- [monitor-api] [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 660 ms
+2026-08-28T16:15:27.539+09:00  INFO 1550 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...
+2026-08-28T16:15:27.557+09:00  INFO 1550 --- [monitor-api] [           main] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Added connection org.postgresql.jdbc.PgConnection@3d1c8f35
+2026-08-28T16:15:27.558+09:00  INFO 1550 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.
+2026-08-28T16:15:27.581+09:00  INFO 1550 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)
+2026-08-28T16:15:27.616+09:00  INFO 1550 --- [monitor-api] [           main] o.f.c.i.s.JdbcTableSchemaHistory         : Schema history table "e04_functional"."flyway_schema_history" does not exist yet
+2026-08-28T16:15:27.619+09:00  INFO 1550 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 8 migrations (execution time 00:00.015s)
+2026-08-28T16:15:27.637+09:00  INFO 1550 --- [monitor-api] [           main] o.f.c.i.s.JdbcTableSchemaHistory         : Creating Schema History table "e04_functional"."flyway_schema_history" ...
+2026-08-28T16:15:27.672+09:00  INFO 1550 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema "e04_functional": << Empty Schema >>
+2026-08-28T16:15:27.676+09:00  INFO 1550 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e04_functional" to version "1 - create monitors"
+2026-08-28T16:15:27.696+09:00  INFO 1550 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e04_functional" to version "2 - create check runs"
+2026-08-28T16:15:27.713+09:00  INFO 1550 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e04_functional" to version "3 - create users"
+2026-08-28T16:15:27.745+09:00  INFO 1550 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e04_functional" to version "4 - require monitor ownership"
+2026-08-28T16:15:27.771+09:00  INFO 1550 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e04_functional" to version "5 - index check history"
+2026-08-28T16:15:27.785+09:00  INFO 1550 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e04_functional" to version "6 - queue check execution"
+2026-08-28T16:15:27.797+09:00  INFO 1550 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e04_functional" to version "7 - execution ownership and manual identity"
+2026-08-28T16:15:27.807+09:00  INFO 1550 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e04_functional" to version "8 - recover expired executions"
+2026-08-28T16:15:27.828+09:00  INFO 1550 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Successfully applied 8 migrations to schema "e04_functional", now at version v8 (execution time 00:00.062s)
+2026-08-28T16:15:27.882+09:00  INFO 1550 --- [monitor-api] [           main] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]
+2026-08-28T16:15:27.916+09:00  INFO 1550 --- [monitor-api] [           main] org.hibernate.Version                    : HHH000412: Hibernate ORM core version 6.6.53.Final
+2026-08-28T16:15:27.935+09:00  INFO 1550 --- [monitor-api] [           main] o.h.c.internal.RegionFactoryInitiator    : HHH000026: Second-level cache disabled
+2026-08-28T16:15:27.999+09:00  INFO 1550 --- [monitor-api] [           main] o.s.o.j.p.SpringPersistenceUnitInfo      : No LoadTimeWeaver setup: ignoring JPA class transformer
+2026-08-28T16:15:28.043+09:00  INFO 1550 --- [monitor-api] [           main] org.hibernate.orm.connections.pooling    : HHH10001005: Database info:
+	Database JDBC URL [Connecting through datasource 'HikariDataSource (HikariPool-1)']
+	Database driver: undefined/unknown
+	Database version: 17.11
+	Autocommit mode: undefined/unknown
+	Isolation level: undefined/unknown
+	Minimum pool size: undefined/unknown
+	Maximum pool size: undefined/unknown
+2026-08-28T16:15:28.389+09:00  INFO 1550 --- [monitor-api] [           main] o.h.e.t.j.p.i.JtaPlatformInitiator       : HHH000489: No JTA platform available (set 'hibernate.transaction.jta.platform' to enable JTA platform integration)
+2026-08-28T16:15:28.414+09:00  INFO 1550 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'
+2026-08-28T16:15:28.506+09:00  INFO 1550 --- [monitor-api] [           main] r$InitializeUserDetailsManagerConfigurer : Global AuthenticationManager configured with UserDetailsService bean with name userAccounts
+2026-08-28T16:15:28.770+09:00  INFO 1550 --- [monitor-api] [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 4322 (http) with context path '/'
+2026-08-28T16:15:28.776+09:00  INFO 1550 --- [monitor-api] [           main] d.e.monitor.MonitorFunctionalTest        : Started MonitorFunctionalTest in 2.468 seconds (process running for 7.658)
+2026-08-28T16:15:29.206+09:00  INFO 1550 --- [monitor-api] [0.1-4322-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
+2026-08-28T16:15:29.206+09:00  INFO 1550 --- [monitor-api] [0.1-4322-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
+2026-08-28T16:15:29.206+09:00  INFO 1550 --- [monitor-api] [0.1-4322-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 0 ms
+2026-08-28T16:15:29.941+09:00  INFO 1550 --- [monitor-api] [           main] dev.evolution.monitor.CheckRunner        : Check policy refused: UNSAFE_ADDRESS
+2026-08-28T16:15:30.043+09:00  INFO 1550 --- [monitor-api] [           main] o.s.b.w.e.tomcat.GracefulShutdown        : Commencing graceful shutdown. Waiting for active requests to complete
+2026-08-28T16:15:30.044+09:00  INFO 1550 --- [monitor-api] [tomcat-shutdown] o.s.b.w.e.tomcat.GracefulShutdown        : Graceful shutdown complete
+2026-08-28T16:15:30.046+09:00  INFO 1550 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Closing JPA EntityManagerFactory for persistence unit 'default'
+2026-08-28T16:15:30.047+09:00  INFO 1550 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown initiated...
+2026-08-28T16:15:30.048+09:00  INFO 1550 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown completed.
+[INFO] Tests run: 16, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 3.885 s -- in dev.evolution.monitor.MonitorFunctionalTest
+[INFO] 
+[INFO] Results:
+[INFO] 
+[ERROR] Failures: 
+[ERROR]   CheckRunnerTest.uninterruptibleDnsRemainsBoundedAndCannotConnectAfterTheDeadline:279 expected: <true> but was: <false>
+[INFO] 
+[ERROR] Tests run: 28, Failures: 1, Errors: 0, Skipped: 0
+[INFO] 
+[INFO] ------------------------------------------------------------------------
+[INFO] BUILD FAILURE
+[INFO] ------------------------------------------------------------------------
+[INFO] Total time:  11.591 s
+[INFO] Finished at: 2026-08-28T16:15:30+09:00
+[INFO] ------------------------------------------------------------------------
+[ERROR] Failed to execute goal org.apache.maven.plugins:maven-surefire-plugin:3.5.6:test (default-test) on project monitor-api: There are test failures.
+[ERROR] 
+[ERROR] See /private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/surefire-reports for the individual test results.
+[ERROR] See dump files (if any exist) [date].dump, [date]-jvmRun[N].dump and [date].dumpstream.
+[ERROR] -> [Help 1]
+[ERROR] 
+[ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.
+[ERROR] Re-run Maven using the -X switch to enable full debug logging.
+[ERROR] 
+[ERROR] For more information about the errors and possible solutions, please read the following articles:
+[ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/MojoFailureException
diff --git a/evidence/phase-1/E12/repair1/attempt1/outbound.json b/evidence/phase-1/E12/repair1/attempt1/outbound.json
new file mode 100644
index 0000000..68a7a33
--- /dev/null
+++ b/evidence/phase-1/E12/repair1/attempt1/outbound.json
@@ -0,0 +1,205 @@
+{
+  "fixtureSha256" : "5889fee87a5ec4506c701e6d509a5ce43af542a680502b7fd48bde44fa993ba1",
+  "observations" : [ {
+    "unsafeConnectorCalls" : 0,
+    "transport" : "numeric resolver stub",
+    "case" : "unsafe-answer-127.0.0.1",
+    "state" : "ABORTED",
+    "httpStatus" : null,
+    "failureReason" : null,
+    "elapsedMs" : 17
+  }, {
+    "unsafeConnectorCalls" : 0,
+    "transport" : "numeric resolver stub",
+    "case" : "unsafe-answer-::1",
+    "state" : "ABORTED",
+    "httpStatus" : null,
+    "failureReason" : null,
+    "elapsedMs" : 0
+  }, {
+    "unsafeConnectorCalls" : 0,
+    "transport" : "numeric resolver stub",
+    "case" : "unsafe-answer-10.0.0.1",
+    "state" : "ABORTED",
+    "httpStatus" : null,
+    "failureReason" : null,
+    "elapsedMs" : 0
+  }, {
+    "unsafeConnectorCalls" : 0,
+    "transport" : "numeric resolver stub",
+    "case" : "unsafe-answer-fc00::1",
+    "state" : "ABORTED",
+    "httpStatus" : null,
+    "failureReason" : null,
+    "elapsedMs" : 0
+  }, {
+    "unsafeConnectorCalls" : 0,
+    "transport" : "numeric resolver stub",
+    "case" : "unsafe-answer-169.254.169.254",
+    "state" : "ABORTED",
+    "httpStatus" : null,
+    "failureReason" : null,
+    "elapsedMs" : 0
+  }, {
+    "unsafeConnectorCalls" : 0,
+    "transport" : "numeric resolver stub",
+    "case" : "unsafe-answer-fe80::1",
+    "state" : "ABORTED",
+    "httpStatus" : null,
+    "failureReason" : null,
+    "elapsedMs" : 0
+  }, {
+    "unsafeConnectorCalls" : 0,
+    "transport" : "numeric resolver stub",
+    "case" : "unsafe-answer-::ffff:127.0.0.1",
+    "state" : "ABORTED",
+    "httpStatus" : null,
+    "failureReason" : null,
+    "elapsedMs" : 0
+  }, {
+    "unsafeConnectorCalls" : 0,
+    "transport" : "resolver stub",
+    "case" : "private.e12.test",
+    "state" : "ABORTED",
+    "httpStatus" : null,
+    "failureReason" : null,
+    "elapsedMs" : 0
+  }, {
+    "unsafeConnectorCalls" : 0,
+    "transport" : "resolver stub",
+    "case" : "mixed.e12.test",
+    "state" : "ABORTED",
+    "httpStatus" : null,
+    "failureReason" : null,
+    "elapsedMs" : 0
+  }, {
+    "dnsCalls" : 1,
+    "bodyBytesRead" : 0,
+    "logicalHost" : "public.e12.test",
+    "transport" : "connector stub; no live TLS",
+    "connectorCalls" : 1,
+    "connectedAddress" : "93.184.216.34",
+    "socketClosed" : true,
+    "case" : "validated-93.184.216.34",
+    "state" : "SUCCEEDED",
+    "httpStatus" : 200,
+    "failureReason" : null,
+    "elapsedMs" : 1
+  }, {
+    "dnsCalls" : 1,
+    "bodyBytesRead" : 0,
+    "logicalHost" : "public.e12.test",
+    "transport" : "connector stub; no live TLS",
+    "connectorCalls" : 1,
+    "connectedAddress" : "2606:4700:4700:0:0:0:0:1111",
+    "socketClosed" : true,
+    "case" : "validated-2606:4700:4700::1111",
+    "state" : "SUCCEEDED",
+    "httpStatus" : 200,
+    "failureReason" : null,
+    "elapsedMs" : 0
+  }, {
+    "liveTlsHandshakeTested" : false,
+    "case" : "TLS-configuration-only",
+    "sniHost" : "public.e12.test",
+    "endpointIdentification" : "HTTPS"
+  }, {
+    "port" : 4325,
+    "transport" : "actual local TCP",
+    "case" : "closed-local-port",
+    "state" : "FAILED",
+    "httpStatus" : null,
+    "failureReason" : "CONNECTION_FAILURE",
+    "elapsedMs" : 7
+  }, {
+    "connectorCalls" : 0,
+    "case" : "canonical-create-boundary",
+    "dnsCalls" : 0,
+    "canonicalUrl" : "http://public.e12.test/ok"
+  }, {
+    "transport" : "connector stub",
+    "socketClosed" : true,
+    "unsafeConnectorCalls" : 0,
+    "safeConnectorCalls" : 1,
+    "case" : "redirect-private",
+    "state" : "ABORTED",
+    "httpStatus" : null,
+    "failureReason" : null,
+    "elapsedMs" : 0
+  }, {
+    "case" : "service-uncertainty",
+    "terminalResultCreated" : false,
+    "automaticRetries" : 0,
+    "exceptionPropagated" : true
+  }, {
+    "transport" : "actual pinned local HTTP",
+    "paths" : [ "/ok" ],
+    "connectorCalls" : 1,
+    "inputBytesRead" : 38,
+    "allRawSocketsClosed" : true,
+    "bodyBytesRead" : 0,
+    "case" : "local-ok",
+    "state" : "SUCCEEDED",
+    "httpStatus" : 200,
+    "failureReason" : null,
+    "elapsedMs" : 4
+  }, {
+    "transport" : "actual pinned local HTTP",
+    "paths" : [ "/body" ],
+    "connectorCalls" : 1,
+    "inputBytesRead" : 42,
+    "allRawSocketsClosed" : true,
+    "bodyBytesOffered" : 65537,
+    "bodyBytesRead" : 0,
+    "case" : "local-body",
+    "state" : "SUCCEEDED",
+    "httpStatus" : 200,
+    "failureReason" : null,
+    "elapsedMs" : 1
+  }, {
+    "transport" : "actual pinned local HTTP",
+    "paths" : [ "/informational" ],
+    "connectorCalls" : 1,
+    "inputBytesRead" : 82,
+    "allRawSocketsClosed" : true,
+    "bodyBytesRead" : 0,
+    "case" : "local-informational",
+    "state" : "SUCCEEDED",
+    "httpStatus" : 200,
+    "failureReason" : null,
+    "elapsedMs" : 1
+  }, {
+    "transport" : "actual pinned local HTTP",
+    "paths" : [ "/trickle" ],
+    "connectorCalls" : 1,
+    "inputBytesRead" : 4,
+    "allRawSocketsClosed" : true,
+    "case" : "local-trickle",
+    "state" : "FAILED",
+    "httpStatus" : null,
+    "failureReason" : "TIMEOUT",
+    "elapsedMs" : 1503
+  }, {
+    "transport" : "actual pinned local HTTP",
+    "paths" : [ "/redirect/0", "/redirect/1", "/redirect/2", "/redirect/3" ],
+    "connectorCalls" : 4,
+    "inputBytesRead" : 256,
+    "allRawSocketsClosed" : true,
+    "case" : "local-redirect",
+    "state" : "ABORTED",
+    "httpStatus" : null,
+    "failureReason" : null,
+    "elapsedMs" : 9
+  }, {
+    "transport" : "actual local HTTP",
+    "responseHeadersSent" : false,
+    "requests" : 1,
+    "delayMs" : 2000,
+    "case" : "slow-headers",
+    "state" : "FAILED",
+    "httpStatus" : null,
+    "failureReason" : "TIMEOUT",
+    "elapsedMs" : 503
+  } ],
+  "externalNetworkUsed" : false
+}
diff --git a/evidence/phase-1/E12/repair1/attempt1/surefire-reports/TEST-dev.evolution.monitor.ApiErrorBoundaryTest.xml b/evidence/phase-1/E12/repair1/attempt1/surefire-reports/TEST-dev.evolution.monitor.ApiErrorBoundaryTest.xml
new file mode 100644
index 0000000..74c3fa3
--- /dev/null
+++ b/evidence/phase-1/E12/repair1/attempt1/surefire-reports/TEST-dev.evolution.monitor.ApiErrorBoundaryTest.xml
@@ -0,0 +1,77 @@
+<?xml version="1.0" encoding="UTF-8"?>
+<testsuite xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="https://maven.apache.org/surefire/maven-surefire-plugin/xsd/surefire-test-report.xsd" version="3.0.2" name="dev.evolution.monitor.ApiErrorBoundaryTest" time="0.766" tests="1" errors="0" skipped="0" failures="0" flakes="0">
+  <properties>
+    <property name="java.specification.version" value="21"/>
+    <property name="sun.jnu.encoding" value="UTF-8"/>
+    <property name="java.class.path" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/test-classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-web/3.5.16/spring-boot-starter-web-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter/3.5.16/spring-boot-starter-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot/3.5.16/spring-boot-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-autoconfigure/3.5.16/spring-boot-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-logging/3.5.16/spring-boot-starter-logging-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-classic/1.5.34/logback-classic-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-core/1.5.34/logback-core-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-to-slf4j/2.24.3/log4j-to-slf4j-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-api/2.24.3/log4j-api-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/jul-to-slf4j/2.0.18/jul-to-slf4j-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/annotation/jakarta.annotation-api/2.1.1/jakarta.annotation-api-2.1.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/yaml/snakeyaml/2.4/snakeyaml-2.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-json/3.5.16/spring-boot-starter-json-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-databind/2.21.4/jackson-databind-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-annotations/2.21/jackson-annotations-2.21.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-core/2.21.4/jackson-core-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jdk8/2.21.4/jackson-datatype-jdk8-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jsr310/2.21.4/jackson-datatype-jsr310-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/module/jackson-module-parameter-names/2.21.4/jackson-module-parameter-names-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-tomcat/3.5.16/spring-boot-starter-tomcat-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-core/10.1.55/tomcat-embed-core-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-el/10.1.55/tomcat-embed-el-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-websocket/10.1.55/tomcat-embed-websocket-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-web/6.2.19/spring-web-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-beans/6.2.19/spring-beans-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-observation/1.15.12/micrometer-observation-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-commons/1.15.12/micrometer-commons-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-webmvc/6.2.19/spring-webmvc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-context/6.2.19/spring-context-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-expression/6.2.19/spring-expression-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-data-jpa/3.5.16/spring-boot-starter-data-jpa-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-jdbc/3.5.16/spring-boot-starter-jdbc-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/zaxxer/HikariCP/6.3.3/HikariCP-6.3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jdbc/6.2.19/spring-jdbc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/orm/hibernate-core/6.6.53.Final/hibernate-core-6.6.53.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/persistence/jakarta.persistence-api/3.1.0/jakarta.persistence-api-3.1.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/transaction/jakarta.transaction-api/2.0.1/jakarta.transaction-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/jboss/logging/jboss-logging/3.6.3.Final/jboss-logging-3.6.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/common/hibernate-commons-annotations/7.0.3.Final/hibernate-commons-annotations-7.0.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/smallrye/jandex/3.2.0/jandex-3.2.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/classmate/1.7.3/classmate-1.7.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy/1.17.8/byte-buddy-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-runtime/4.0.9/jaxb-runtime-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-core/4.0.9/jaxb-core-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/eclipse/angus/angus-activation/2.0.3/angus-activation-2.0.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/txw2/4.0.9/txw2-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/sun/istack/istack-commons-runtime/4.1.2/istack-commons-runtime-4.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/inject/jakarta.inject-api/2.0.1/jakarta.inject-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/antlr/antlr4-runtime/4.13.2/antlr4-runtime-4.13.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-jpa/3.5.13/spring-data-jpa-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-commons/3.5.13/spring-data-commons-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-orm/6.2.19/spring-orm-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-tx/6.2.19/spring-tx-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/slf4j-api/2.0.18/slf4j-api-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aspects/6.2.19/spring-aspects-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/aspectj/aspectjweaver/1.9.25.1/aspectjweaver-1.9.25.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-security/3.5.16/spring-boot-starter-security-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aop/6.2.19/spring-aop-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-config/6.5.11/spring-security-config-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-core/6.5.11/spring-security-core-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-crypto/6.5.11/spring-security-crypto-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-web/6.5.11/spring-security-web-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-database-postgresql/11.7.2/flyway-database-postgresql-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-core/11.7.2/flyway-core-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/dataformat/jackson-dataformat-toml/2.21.4/jackson-dataformat-toml-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/postgresql/postgresql/42.7.11/postgresql-42.7.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-test/3.5.16/spring-boot-starter-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test/3.5.16/spring-boot-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test-autoconfigure/3.5.16/spring-boot-test-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/jayway/jsonpath/json-path/2.9.0/json-path-2.9.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/xml/bind/jakarta.xml.bind-api/4.0.5/jakarta.xml.bind-api-4.0.5.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/activation/jakarta.activation-api/2.1.4/jakarta.activation-api-2.1.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/json-smart/2.5.2/json-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/accessors-smart/2.5.2/accessors-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/ow2/asm/asm/9.7.1/asm-9.7.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/assertj/assertj-core/3.27.7/assertj-core-3.27.7.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/awaitility/awaitility/4.2.2/awaitility-4.2.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hamcrest/hamcrest/3.0/hamcrest-3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter/5.12.2/junit-jupiter-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-api/5.12.2/junit-jupiter-api-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/opentest4j/opentest4j/1.3.0/opentest4j-1.3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-commons/1.12.2/junit-platform-commons-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apiguardian/apiguardian-api/1.1.2/apiguardian-api-1.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-params/5.12.2/junit-jupiter-params-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-engine/5.12.2/junit-jupiter-engine-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-engine/1.12.2/junit-platform-engine-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-core/5.17.0/mockito-core-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy-agent/1.17.8/byte-buddy-agent-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/objenesis/objenesis/3.3/objenesis-3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-junit-jupiter/5.17.0/mockito-junit-jupiter-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/skyscreamer/jsonassert/1.5.3/jsonassert-1.5.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/vaadin/external/google/android-json/0.0.20131108.vaadin1/android-json-0.0.20131108.vaadin1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-core/6.2.19/spring-core-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jcl/6.2.19/spring-jcl-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-test/6.2.19/spring-test-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/xmlunit/xmlunit-core/2.10.4/xmlunit-core-2.10.4.jar:"/>
+    <property name="java.vm.vendor" value="Eclipse Adoptium"/>
+    <property name="sun.arch.data.model" value="64"/>
+    <property name="java.vendor.url" value="https://adoptium.net/"/>
+    <property name="user.timezone" value="Asia/Seoul"/>
+    <property name="os.name" value="Mac OS X"/>
+    <property name="java.vm.specification.version" value="21"/>
+    <property name="sun.java.launcher" value="SUN_STANDARD"/>
+    <property name="user.country" value="KR"/>
+    <property name="sun.boot.library.path" value="/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem/lib"/>
+    <property name="sun.java.command" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/surefire/surefirebooter-20260828161521074_3.jar /private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/surefire 2026-08-28T16-15-21_034-jvmRun1 surefire-20260828161521074_1tmp surefire_0-20260828161521074_2tmp"/>
+    <property name="http.nonProxyHosts" value="local|*.local|169.254/16|*.169.254/16"/>
+    <property name="jdk.debug" value="release"/>
+    <property name="test" value="CheckRunnerTest,MonitorFunctionalTest,ApiErrorBoundaryTest"/>
+    <property name="surefire.test.class.path" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/test-classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-web/3.5.16/spring-boot-starter-web-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter/3.5.16/spring-boot-starter-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot/3.5.16/spring-boot-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-autoconfigure/3.5.16/spring-boot-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-logging/3.5.16/spring-boot-starter-logging-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-classic/1.5.34/logback-classic-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-core/1.5.34/logback-core-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-to-slf4j/2.24.3/log4j-to-slf4j-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-api/2.24.3/log4j-api-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/jul-to-slf4j/2.0.18/jul-to-slf4j-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/annotation/jakarta.annotation-api/2.1.1/jakarta.annotation-api-2.1.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/yaml/snakeyaml/2.4/snakeyaml-2.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-json/3.5.16/spring-boot-starter-json-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-databind/2.21.4/jackson-databind-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-annotations/2.21/jackson-annotations-2.21.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-core/2.21.4/jackson-core-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jdk8/2.21.4/jackson-datatype-jdk8-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jsr310/2.21.4/jackson-datatype-jsr310-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/module/jackson-module-parameter-names/2.21.4/jackson-module-parameter-names-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-tomcat/3.5.16/spring-boot-starter-tomcat-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-core/10.1.55/tomcat-embed-core-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-el/10.1.55/tomcat-embed-el-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-websocket/10.1.55/tomcat-embed-websocket-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-web/6.2.19/spring-web-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-beans/6.2.19/spring-beans-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-observation/1.15.12/micrometer-observation-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-commons/1.15.12/micrometer-commons-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-webmvc/6.2.19/spring-webmvc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-context/6.2.19/spring-context-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-expression/6.2.19/spring-expression-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-data-jpa/3.5.16/spring-boot-starter-data-jpa-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-jdbc/3.5.16/spring-boot-starter-jdbc-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/zaxxer/HikariCP/6.3.3/HikariCP-6.3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jdbc/6.2.19/spring-jdbc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/orm/hibernate-core/6.6.53.Final/hibernate-core-6.6.53.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/persistence/jakarta.persistence-api/3.1.0/jakarta.persistence-api-3.1.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/transaction/jakarta.transaction-api/2.0.1/jakarta.transaction-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/jboss/logging/jboss-logging/3.6.3.Final/jboss-logging-3.6.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/common/hibernate-commons-annotations/7.0.3.Final/hibernate-commons-annotations-7.0.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/smallrye/jandex/3.2.0/jandex-3.2.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/classmate/1.7.3/classmate-1.7.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy/1.17.8/byte-buddy-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-runtime/4.0.9/jaxb-runtime-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-core/4.0.9/jaxb-core-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/eclipse/angus/angus-activation/2.0.3/angus-activation-2.0.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/txw2/4.0.9/txw2-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/sun/istack/istack-commons-runtime/4.1.2/istack-commons-runtime-4.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/inject/jakarta.inject-api/2.0.1/jakarta.inject-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/antlr/antlr4-runtime/4.13.2/antlr4-runtime-4.13.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-jpa/3.5.13/spring-data-jpa-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-commons/3.5.13/spring-data-commons-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-orm/6.2.19/spring-orm-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-tx/6.2.19/spring-tx-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/slf4j-api/2.0.18/slf4j-api-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aspects/6.2.19/spring-aspects-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/aspectj/aspectjweaver/1.9.25.1/aspectjweaver-1.9.25.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-security/3.5.16/spring-boot-starter-security-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aop/6.2.19/spring-aop-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-config/6.5.11/spring-security-config-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-core/6.5.11/spring-security-core-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-crypto/6.5.11/spring-security-crypto-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-web/6.5.11/spring-security-web-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-database-postgresql/11.7.2/flyway-database-postgresql-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-core/11.7.2/flyway-core-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/dataformat/jackson-dataformat-toml/2.21.4/jackson-dataformat-toml-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/postgresql/postgresql/42.7.11/postgresql-42.7.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-test/3.5.16/spring-boot-starter-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test/3.5.16/spring-boot-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test-autoconfigure/3.5.16/spring-boot-test-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/jayway/jsonpath/json-path/2.9.0/json-path-2.9.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/xml/bind/jakarta.xml.bind-api/4.0.5/jakarta.xml.bind-api-4.0.5.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/activation/jakarta.activation-api/2.1.4/jakarta.activation-api-2.1.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/json-smart/2.5.2/json-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/accessors-smart/2.5.2/accessors-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/ow2/asm/asm/9.7.1/asm-9.7.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/assertj/assertj-core/3.27.7/assertj-core-3.27.7.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/awaitility/awaitility/4.2.2/awaitility-4.2.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hamcrest/hamcrest/3.0/hamcrest-3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter/5.12.2/junit-jupiter-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-api/5.12.2/junit-jupiter-api-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/opentest4j/opentest4j/1.3.0/opentest4j-1.3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-commons/1.12.2/junit-platform-commons-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apiguardian/apiguardian-api/1.1.2/apiguardian-api-1.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-params/5.12.2/junit-jupiter-params-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-engine/5.12.2/junit-jupiter-engine-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-engine/1.12.2/junit-platform-engine-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-core/5.17.0/mockito-core-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy-agent/1.17.8/byte-buddy-agent-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/objenesis/objenesis/3.3/objenesis-3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-junit-jupiter/5.17.0/mockito-junit-jupiter-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/skyscreamer/jsonassert/1.5.3/jsonassert-1.5.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/vaadin/external/google/android-json/0.0.20131108.vaadin1/android-json-0.0.20131108.vaadin1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-core/6.2.19/spring-core-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jcl/6.2.19/spring-jcl-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-test/6.2.19/spring-test-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/xmlunit/xmlunit-core/2.10.4/xmlunit-core-2.10.4.jar:"/>
+    <property name="sun.cpu.endian" value="little"/>
+    <property name="user.home" value="/Users/woopinbell"/>
+    <property name="user.language" value="ko"/>
+    <property name="java.specification.vendor" value="Oracle Corporation"/>
+    <property name="java.version.date" value="2025-04-15"/>
+    <property name="java.home" value="/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"/>
+    <property name="surefire.skipAfterFailureCount" value="1"/>
+    <property name="file.separator" value="/"/>
+    <property name="basedir" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend"/>
+    <property name="java.vm.compressedOopsMode" value="Zero based"/>
+    <property name="line.separator" value="&#10;"/>
+    <property name="java.vm.specification.vendor" value="Oracle Corporation"/>
+    <property name="java.specification.name" value="Java Platform API Specification"/>
+    <property name="apple.awt.application.name" value="ForkedBooter"/>
+    <property name="surefire.real.class.path" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/surefire/surefirebooter-20260828161521074_3.jar"/>
+    <property name="sun.management.compiler" value="HotSpot 64-Bit Tiered Compilers"/>
+    <property name="ftp.nonProxyHosts" value="local|*.local|169.254/16|*.169.254/16"/>
+    <property name="java.runtime.version" value="21.0.7+6-LTS"/>
+    <property name="user.name" value="woopinbell"/>
+    <property name="stdout.encoding" value="UTF-8"/>
+    <property name="path.separator" value=":"/>
+    <property name="os.version" value="26.6.2"/>
+    <property name="java.runtime.name" value="OpenJDK Runtime Environment"/>
+    <property name="file.encoding" value="UTF-8"/>
+    <property name="java.vm.name" value="OpenJDK 64-Bit Server VM"/>
+    <property name="java.vendor.version" value="Temurin-21.0.7+6"/>
+    <property name="localRepository" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository"/>
+    <property name="java.vendor.url.bug" value="https://github.com/adoptium/adoptium-support/issues"/>
+    <property name="java.io.tmpdir" value="/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/"/>
+    <property name="java.version" value="21.0.7"/>
+    <property name="user.dir" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend"/>
+    <property name="os.arch" value="aarch64"/>
+    <property name="java.vm.specification.name" value="Java Virtual Machine Specification"/>
+    <property name="native.encoding" value="UTF-8"/>
+    <property name="java.library.path" value="/Users/woopinbell/Library/Java/Extensions:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java:."/>
+    <property name="java.vm.info" value="mixed mode, sharing"/>
+    <property name="stderr.encoding" value="UTF-8"/>
+    <property name="java.vendor" value="Eclipse Adoptium"/>
+    <property name="java.vm.version" value="21.0.7+6-LTS"/>
+    <property name="sun.io.unicode.encoding" value="UnicodeBig"/>
+    <property name="maven.repo.local" value=".m2/repository"/>
+    <property name="socksNonProxyHosts" value="local|*.local|169.254/16|*.169.254/16"/>
+    <property name="java.class.version" value="65.0"/>
+  </properties>
+  <testcase name="unexpectedFailureIsAnInternalErrorWithoutPrivateExceptionDetails" classname="dev.evolution.monitor.ApiErrorBoundaryTest" time="0.765">
+    <system-out><![CDATA[16:15:26.086 [main] INFO org.springframework.mock.web.MockServletContext -- Initializing Spring TestDispatcherServlet ''
+16:15:26.086 [main] INFO org.springframework.test.web.servlet.TestDispatcherServlet -- Initializing Servlet ''
+16:15:26.087 [main] INFO org.springframework.test.web.servlet.TestDispatcherServlet -- Completed initialization in 1 ms
+]]></system-out>
+    <system-err><![CDATA[Mockito is currently self-attaching to enable the inline-mock-maker. This will no longer work in future releases of the JDK. Please add Mockito as an agent to your build as described in Mockito's documentation: https://javadoc.io/doc/org.mockito/mockito-core/latest/org.mockito/org/mockito/Mockito.html#0.3
+WARNING: A Java agent has been loaded dynamically (/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy-agent/1.17.8/byte-buddy-agent-1.17.8.jar)
+WARNING: If a serviceability tool is in use, please run with -XX:+EnableDynamicAgentLoading to hide this warning
+WARNING: If a serviceability tool is not in use, please run with -Djdk.instrument.traceUsage for more information
+WARNING: Dynamic loading of agents will be disallowed by default in a future release
+]]></system-err>
+  </testcase>
+</testsuite>
\ No newline at end of file
diff --git a/evidence/phase-1/E12/repair1/attempt1/surefire-reports/TEST-dev.evolution.monitor.CheckRunnerTest.xml b/evidence/phase-1/E12/repair1/attempt1/surefire-reports/TEST-dev.evolution.monitor.CheckRunnerTest.xml
new file mode 100644
index 0000000..92efab3
--- /dev/null
+++ b/evidence/phase-1/E12/repair1/attempt1/surefire-reports/TEST-dev.evolution.monitor.CheckRunnerTest.xml
@@ -0,0 +1,106 @@
+<?xml version="1.0" encoding="UTF-8"?>
+<testsuite xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="https://maven.apache.org/surefire/maven-surefire-plugin/xsd/surefire-test-report.xsd" version="3.0.2" name="dev.evolution.monitor.CheckRunnerTest" time="4.015" tests="11" errors="0" skipped="0" failures="1" flakes="0">
+  <properties>
+    <property name="java.specification.version" value="21"/>
+    <property name="sun.jnu.encoding" value="UTF-8"/>
+    <property name="java.class.path" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/test-classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-web/3.5.16/spring-boot-starter-web-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter/3.5.16/spring-boot-starter-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot/3.5.16/spring-boot-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-autoconfigure/3.5.16/spring-boot-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-logging/3.5.16/spring-boot-starter-logging-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-classic/1.5.34/logback-classic-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-core/1.5.34/logback-core-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-to-slf4j/2.24.3/log4j-to-slf4j-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-api/2.24.3/log4j-api-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/jul-to-slf4j/2.0.18/jul-to-slf4j-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/annotation/jakarta.annotation-api/2.1.1/jakarta.annotation-api-2.1.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/yaml/snakeyaml/2.4/snakeyaml-2.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-json/3.5.16/spring-boot-starter-json-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-databind/2.21.4/jackson-databind-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-annotations/2.21/jackson-annotations-2.21.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-core/2.21.4/jackson-core-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jdk8/2.21.4/jackson-datatype-jdk8-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jsr310/2.21.4/jackson-datatype-jsr310-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/module/jackson-module-parameter-names/2.21.4/jackson-module-parameter-names-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-tomcat/3.5.16/spring-boot-starter-tomcat-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-core/10.1.55/tomcat-embed-core-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-el/10.1.55/tomcat-embed-el-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-websocket/10.1.55/tomcat-embed-websocket-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-web/6.2.19/spring-web-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-beans/6.2.19/spring-beans-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-observation/1.15.12/micrometer-observation-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-commons/1.15.12/micrometer-commons-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-webmvc/6.2.19/spring-webmvc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-context/6.2.19/spring-context-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-expression/6.2.19/spring-expression-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-data-jpa/3.5.16/spring-boot-starter-data-jpa-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-jdbc/3.5.16/spring-boot-starter-jdbc-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/zaxxer/HikariCP/6.3.3/HikariCP-6.3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jdbc/6.2.19/spring-jdbc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/orm/hibernate-core/6.6.53.Final/hibernate-core-6.6.53.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/persistence/jakarta.persistence-api/3.1.0/jakarta.persistence-api-3.1.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/transaction/jakarta.transaction-api/2.0.1/jakarta.transaction-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/jboss/logging/jboss-logging/3.6.3.Final/jboss-logging-3.6.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/common/hibernate-commons-annotations/7.0.3.Final/hibernate-commons-annotations-7.0.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/smallrye/jandex/3.2.0/jandex-3.2.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/classmate/1.7.3/classmate-1.7.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy/1.17.8/byte-buddy-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-runtime/4.0.9/jaxb-runtime-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-core/4.0.9/jaxb-core-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/eclipse/angus/angus-activation/2.0.3/angus-activation-2.0.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/txw2/4.0.9/txw2-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/sun/istack/istack-commons-runtime/4.1.2/istack-commons-runtime-4.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/inject/jakarta.inject-api/2.0.1/jakarta.inject-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/antlr/antlr4-runtime/4.13.2/antlr4-runtime-4.13.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-jpa/3.5.13/spring-data-jpa-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-commons/3.5.13/spring-data-commons-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-orm/6.2.19/spring-orm-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-tx/6.2.19/spring-tx-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/slf4j-api/2.0.18/slf4j-api-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aspects/6.2.19/spring-aspects-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/aspectj/aspectjweaver/1.9.25.1/aspectjweaver-1.9.25.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-security/3.5.16/spring-boot-starter-security-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aop/6.2.19/spring-aop-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-config/6.5.11/spring-security-config-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-core/6.5.11/spring-security-core-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-crypto/6.5.11/spring-security-crypto-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-web/6.5.11/spring-security-web-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-database-postgresql/11.7.2/flyway-database-postgresql-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-core/11.7.2/flyway-core-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/dataformat/jackson-dataformat-toml/2.21.4/jackson-dataformat-toml-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/postgresql/postgresql/42.7.11/postgresql-42.7.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-test/3.5.16/spring-boot-starter-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test/3.5.16/spring-boot-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test-autoconfigure/3.5.16/spring-boot-test-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/jayway/jsonpath/json-path/2.9.0/json-path-2.9.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/xml/bind/jakarta.xml.bind-api/4.0.5/jakarta.xml.bind-api-4.0.5.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/activation/jakarta.activation-api/2.1.4/jakarta.activation-api-2.1.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/json-smart/2.5.2/json-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/accessors-smart/2.5.2/accessors-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/ow2/asm/asm/9.7.1/asm-9.7.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/assertj/assertj-core/3.27.7/assertj-core-3.27.7.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/awaitility/awaitility/4.2.2/awaitility-4.2.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hamcrest/hamcrest/3.0/hamcrest-3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter/5.12.2/junit-jupiter-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-api/5.12.2/junit-jupiter-api-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/opentest4j/opentest4j/1.3.0/opentest4j-1.3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-commons/1.12.2/junit-platform-commons-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apiguardian/apiguardian-api/1.1.2/apiguardian-api-1.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-params/5.12.2/junit-jupiter-params-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-engine/5.12.2/junit-jupiter-engine-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-engine/1.12.2/junit-platform-engine-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-core/5.17.0/mockito-core-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy-agent/1.17.8/byte-buddy-agent-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/objenesis/objenesis/3.3/objenesis-3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-junit-jupiter/5.17.0/mockito-junit-jupiter-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/skyscreamer/jsonassert/1.5.3/jsonassert-1.5.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/vaadin/external/google/android-json/0.0.20131108.vaadin1/android-json-0.0.20131108.vaadin1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-core/6.2.19/spring-core-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jcl/6.2.19/spring-jcl-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-test/6.2.19/spring-test-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/xmlunit/xmlunit-core/2.10.4/xmlunit-core-2.10.4.jar:"/>
+    <property name="java.vm.vendor" value="Eclipse Adoptium"/>
+    <property name="sun.arch.data.model" value="64"/>
+    <property name="java.vendor.url" value="https://adoptium.net/"/>
+    <property name="user.timezone" value="Asia/Seoul"/>
+    <property name="os.name" value="Mac OS X"/>
+    <property name="java.vm.specification.version" value="21"/>
+    <property name="sun.java.launcher" value="SUN_STANDARD"/>
+    <property name="user.country" value="KR"/>
+    <property name="sun.boot.library.path" value="/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem/lib"/>
+    <property name="sun.java.command" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/surefire/surefirebooter-20260828161521074_3.jar /private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/surefire 2026-08-28T16-15-21_034-jvmRun1 surefire-20260828161521074_1tmp surefire_0-20260828161521074_2tmp"/>
+    <property name="http.nonProxyHosts" value="local|*.local|169.254/16|*.169.254/16"/>
+    <property name="jdk.debug" value="release"/>
+    <property name="test" value="CheckRunnerTest,MonitorFunctionalTest,ApiErrorBoundaryTest"/>
+    <property name="surefire.test.class.path" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/test-classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-web/3.5.16/spring-boot-starter-web-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter/3.5.16/spring-boot-starter-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot/3.5.16/spring-boot-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-autoconfigure/3.5.16/spring-boot-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-logging/3.5.16/spring-boot-starter-logging-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-classic/1.5.34/logback-classic-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-core/1.5.34/logback-core-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-to-slf4j/2.24.3/log4j-to-slf4j-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-api/2.24.3/log4j-api-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/jul-to-slf4j/2.0.18/jul-to-slf4j-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/annotation/jakarta.annotation-api/2.1.1/jakarta.annotation-api-2.1.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/yaml/snakeyaml/2.4/snakeyaml-2.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-json/3.5.16/spring-boot-starter-json-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-databind/2.21.4/jackson-databind-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-annotations/2.21/jackson-annotations-2.21.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-core/2.21.4/jackson-core-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jdk8/2.21.4/jackson-datatype-jdk8-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jsr310/2.21.4/jackson-datatype-jsr310-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/module/jackson-module-parameter-names/2.21.4/jackson-module-parameter-names-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-tomcat/3.5.16/spring-boot-starter-tomcat-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-core/10.1.55/tomcat-embed-core-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-el/10.1.55/tomcat-embed-el-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-websocket/10.1.55/tomcat-embed-websocket-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-web/6.2.19/spring-web-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-beans/6.2.19/spring-beans-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-observation/1.15.12/micrometer-observation-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-commons/1.15.12/micrometer-commons-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-webmvc/6.2.19/spring-webmvc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-context/6.2.19/spring-context-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-expression/6.2.19/spring-expression-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-data-jpa/3.5.16/spring-boot-starter-data-jpa-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-jdbc/3.5.16/spring-boot-starter-jdbc-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/zaxxer/HikariCP/6.3.3/HikariCP-6.3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jdbc/6.2.19/spring-jdbc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/orm/hibernate-core/6.6.53.Final/hibernate-core-6.6.53.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/persistence/jakarta.persistence-api/3.1.0/jakarta.persistence-api-3.1.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/transaction/jakarta.transaction-api/2.0.1/jakarta.transaction-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/jboss/logging/jboss-logging/3.6.3.Final/jboss-logging-3.6.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/common/hibernate-commons-annotations/7.0.3.Final/hibernate-commons-annotations-7.0.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/smallrye/jandex/3.2.0/jandex-3.2.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/classmate/1.7.3/classmate-1.7.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy/1.17.8/byte-buddy-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-runtime/4.0.9/jaxb-runtime-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-core/4.0.9/jaxb-core-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/eclipse/angus/angus-activation/2.0.3/angus-activation-2.0.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/txw2/4.0.9/txw2-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/sun/istack/istack-commons-runtime/4.1.2/istack-commons-runtime-4.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/inject/jakarta.inject-api/2.0.1/jakarta.inject-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/antlr/antlr4-runtime/4.13.2/antlr4-runtime-4.13.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-jpa/3.5.13/spring-data-jpa-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-commons/3.5.13/spring-data-commons-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-orm/6.2.19/spring-orm-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-tx/6.2.19/spring-tx-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/slf4j-api/2.0.18/slf4j-api-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aspects/6.2.19/spring-aspects-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/aspectj/aspectjweaver/1.9.25.1/aspectjweaver-1.9.25.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-security/3.5.16/spring-boot-starter-security-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aop/6.2.19/spring-aop-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-config/6.5.11/spring-security-config-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-core/6.5.11/spring-security-core-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-crypto/6.5.11/spring-security-crypto-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-web/6.5.11/spring-security-web-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-database-postgresql/11.7.2/flyway-database-postgresql-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-core/11.7.2/flyway-core-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/dataformat/jackson-dataformat-toml/2.21.4/jackson-dataformat-toml-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/postgresql/postgresql/42.7.11/postgresql-42.7.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-test/3.5.16/spring-boot-starter-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test/3.5.16/spring-boot-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test-autoconfigure/3.5.16/spring-boot-test-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/jayway/jsonpath/json-path/2.9.0/json-path-2.9.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/xml/bind/jakarta.xml.bind-api/4.0.5/jakarta.xml.bind-api-4.0.5.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/activation/jakarta.activation-api/2.1.4/jakarta.activation-api-2.1.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/json-smart/2.5.2/json-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/accessors-smart/2.5.2/accessors-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/ow2/asm/asm/9.7.1/asm-9.7.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/assertj/assertj-core/3.27.7/assertj-core-3.27.7.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/awaitility/awaitility/4.2.2/awaitility-4.2.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hamcrest/hamcrest/3.0/hamcrest-3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter/5.12.2/junit-jupiter-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-api/5.12.2/junit-jupiter-api-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/opentest4j/opentest4j/1.3.0/opentest4j-1.3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-commons/1.12.2/junit-platform-commons-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apiguardian/apiguardian-api/1.1.2/apiguardian-api-1.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-params/5.12.2/junit-jupiter-params-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-engine/5.12.2/junit-jupiter-engine-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-engine/1.12.2/junit-platform-engine-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-core/5.17.0/mockito-core-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy-agent/1.17.8/byte-buddy-agent-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/objenesis/objenesis/3.3/objenesis-3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-junit-jupiter/5.17.0/mockito-junit-jupiter-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/skyscreamer/jsonassert/1.5.3/jsonassert-1.5.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/vaadin/external/google/android-json/0.0.20131108.vaadin1/android-json-0.0.20131108.vaadin1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-core/6.2.19/spring-core-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jcl/6.2.19/spring-jcl-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-test/6.2.19/spring-test-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/xmlunit/xmlunit-core/2.10.4/xmlunit-core-2.10.4.jar:"/>
+    <property name="sun.cpu.endian" value="little"/>
+    <property name="user.home" value="/Users/woopinbell"/>
+    <property name="user.language" value="ko"/>
+    <property name="java.specification.vendor" value="Oracle Corporation"/>
+    <property name="java.version.date" value="2025-04-15"/>
+    <property name="java.home" value="/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"/>
+    <property name="surefire.skipAfterFailureCount" value="1"/>
+    <property name="file.separator" value="/"/>
+    <property name="basedir" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend"/>
+    <property name="java.vm.compressedOopsMode" value="Zero based"/>
+    <property name="line.separator" value="&#10;"/>
+    <property name="java.vm.specification.vendor" value="Oracle Corporation"/>
+    <property name="java.specification.name" value="Java Platform API Specification"/>
+    <property name="apple.awt.application.name" value="ForkedBooter"/>
+    <property name="surefire.real.class.path" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/surefire/surefirebooter-20260828161521074_3.jar"/>
+    <property name="sun.management.compiler" value="HotSpot 64-Bit Tiered Compilers"/>
+    <property name="ftp.nonProxyHosts" value="local|*.local|169.254/16|*.169.254/16"/>
+    <property name="java.runtime.version" value="21.0.7+6-LTS"/>
+    <property name="user.name" value="woopinbell"/>
+    <property name="stdout.encoding" value="UTF-8"/>
+    <property name="path.separator" value=":"/>
+    <property name="os.version" value="26.6.2"/>
+    <property name="java.runtime.name" value="OpenJDK Runtime Environment"/>
+    <property name="file.encoding" value="UTF-8"/>
+    <property name="java.vm.name" value="OpenJDK 64-Bit Server VM"/>
+    <property name="java.vendor.version" value="Temurin-21.0.7+6"/>
+    <property name="localRepository" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository"/>
+    <property name="java.vendor.url.bug" value="https://github.com/adoptium/adoptium-support/issues"/>
+    <property name="java.io.tmpdir" value="/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/"/>
+    <property name="java.version" value="21.0.7"/>
+    <property name="user.dir" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend"/>
+    <property name="os.arch" value="aarch64"/>
+    <property name="java.vm.specification.name" value="Java Virtual Machine Specification"/>
+    <property name="native.encoding" value="UTF-8"/>
+    <property name="java.library.path" value="/Users/woopinbell/Library/Java/Extensions:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java:."/>
+    <property name="java.vm.info" value="mixed mode, sharing"/>
+    <property name="stderr.encoding" value="UTF-8"/>
+    <property name="java.vendor" value="Eclipse Adoptium"/>
+    <property name="java.vm.version" value="21.0.7+6-LTS"/>
+    <property name="sun.io.unicode.encoding" value="UnicodeBig"/>
+    <property name="maven.repo.local" value=".m2/repository"/>
+    <property name="socksNonProxyHosts" value="local|*.local|169.254/16|*.169.254/16"/>
+    <property name="java.class.version" value="65.0"/>
+  </properties>
+  <testcase name="everyUnsafeLiteralAndActualDnsAnswerIsRefusedBeforeConnection" classname="dev.evolution.monitor.CheckRunnerTest" time="0.05">
+    <system-out><![CDATA[16:15:21.455 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:15:21.457 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:15:21.458 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:15:21.458 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:15:21.459 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:15:21.459 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:15:21.460 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:15:21.460 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:15:21.461 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+]]></system-out>
+  </testcase>
+  <testcase name="uninterruptibleDnsRemainsBoundedAndCannotConnectAfterTheDeadline" classname="dev.evolution.monitor.CheckRunnerTest" time="1.776">
+    <failure message="expected: &lt;true&gt; but was: &lt;false&gt;" type="org.opentest4j.AssertionFailedError"><![CDATA[org.opentest4j.AssertionFailedError: expected: <true> but was: <false>
+	at org.junit.jupiter.api.AssertionFailureBuilder.build(AssertionFailureBuilder.java:151)
+	at org.junit.jupiter.api.AssertionFailureBuilder.buildAndThrow(AssertionFailureBuilder.java:132)
+	at org.junit.jupiter.api.AssertTrue.failNotTrue(AssertTrue.java:63)
+	at org.junit.jupiter.api.AssertTrue.assertTrue(AssertTrue.java:36)
+	at org.junit.jupiter.api.AssertTrue.assertTrue(AssertTrue.java:31)
+	at org.junit.jupiter.api.Assertions.assertTrue(Assertions.java:183)
+	at dev.evolution.monitor.CheckRunnerTest.uninterruptibleDnsRemainsBoundedAndCannotConnectAfterTheDeadline(CheckRunnerTest.java:279)
+	at java.base/java.lang.reflect.Method.invoke(Method.java:580)
+	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
+	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
+]]></failure>
+  </testcase>
+  <testcase name="publicAnswersUseValidatedAddressesAndOriginalTlsHostnameWithoutSecondDns" classname="dev.evolution.monitor.CheckRunnerTest" time="0.016"/>
+  <testcase name="connectionFailureHasNoInventedHttpStatusOnTheWire" classname="dev.evolution.monitor.CheckRunnerTest" time="0.023"/>
+  <testcase name="canonicalUrlsPerformNoDnsOrConnectorWork" classname="dev.evolution.monitor.CheckRunnerTest" time="0.003"/>
+  <testcase name="onlyTwoHundredsAreSuccessful" classname="dev.evolution.monitor.CheckRunnerTest" time="0.001"/>
+  <testcase name="fixtureExceptionIsExplicitAndStillExactlyConfiguredHostPortAndScheme" classname="dev.evolution.monitor.CheckRunnerTest" time="0.003"/>
+  <testcase name="aRedirectToPrivateSpaceIsAbortedWithoutAnIntermediateHttpOutcome" classname="dev.evolution.monitor.CheckRunnerTest" time="0.002">
+    <system-out><![CDATA[16:15:23.301 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_ADDRESS
+]]></system-out>
+  </testcase>
+  <testcase name="authoritativeServiceUncertaintyDoesNotBecomeAnEndpointResult" classname="dev.evolution.monitor.CheckRunnerTest" time="0.001"/>
+  <testcase name="realLocalResponsesRespectFinalHeadersTotalDeadlineAndRedirectBudget" classname="dev.evolution.monitor.CheckRunnerTest" time="1.538">
+    <system-out><![CDATA[16:15:24.841 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: REDIRECT_LIMIT
+]]></system-out>
+  </testcase>
+  <testcase name="headerTimeoutHasNoInventedHttpStatusOnTheWire" classname="dev.evolution.monitor.CheckRunnerTest" time="0.515"/>
+</testsuite>
\ No newline at end of file
diff --git a/evidence/phase-1/E12/repair1/attempt1/surefire-reports/TEST-dev.evolution.monitor.MonitorFunctionalTest.xml b/evidence/phase-1/E12/repair1/attempt1/surefire-reports/TEST-dev.evolution.monitor.MonitorFunctionalTest.xml
new file mode 100644
index 0000000..2741dff
--- /dev/null
+++ b/evidence/phase-1/E12/repair1/attempt1/surefire-reports/TEST-dev.evolution.monitor.MonitorFunctionalTest.xml
@@ -0,0 +1,156 @@
+<?xml version="1.0" encoding="UTF-8"?>
+<testsuite xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="https://maven.apache.org/surefire/maven-surefire-plugin/xsd/surefire-test-report.xsd" version="3.0.2" name="dev.evolution.monitor.MonitorFunctionalTest" time="3.885" tests="16" errors="0" skipped="0" failures="0" flakes="0">
+  <properties>
+    <property name="java.specification.version" value="21"/>
+    <property name="sun.jnu.encoding" value="UTF-8"/>
+    <property name="java.class.path" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/test-classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-web/3.5.16/spring-boot-starter-web-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter/3.5.16/spring-boot-starter-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot/3.5.16/spring-boot-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-autoconfigure/3.5.16/spring-boot-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-logging/3.5.16/spring-boot-starter-logging-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-classic/1.5.34/logback-classic-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-core/1.5.34/logback-core-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-to-slf4j/2.24.3/log4j-to-slf4j-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-api/2.24.3/log4j-api-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/jul-to-slf4j/2.0.18/jul-to-slf4j-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/annotation/jakarta.annotation-api/2.1.1/jakarta.annotation-api-2.1.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/yaml/snakeyaml/2.4/snakeyaml-2.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-json/3.5.16/spring-boot-starter-json-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-databind/2.21.4/jackson-databind-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-annotations/2.21/jackson-annotations-2.21.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-core/2.21.4/jackson-core-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jdk8/2.21.4/jackson-datatype-jdk8-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jsr310/2.21.4/jackson-datatype-jsr310-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/module/jackson-module-parameter-names/2.21.4/jackson-module-parameter-names-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-tomcat/3.5.16/spring-boot-starter-tomcat-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-core/10.1.55/tomcat-embed-core-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-el/10.1.55/tomcat-embed-el-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-websocket/10.1.55/tomcat-embed-websocket-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-web/6.2.19/spring-web-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-beans/6.2.19/spring-beans-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-observation/1.15.12/micrometer-observation-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-commons/1.15.12/micrometer-commons-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-webmvc/6.2.19/spring-webmvc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-context/6.2.19/spring-context-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-expression/6.2.19/spring-expression-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-data-jpa/3.5.16/spring-boot-starter-data-jpa-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-jdbc/3.5.16/spring-boot-starter-jdbc-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/zaxxer/HikariCP/6.3.3/HikariCP-6.3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jdbc/6.2.19/spring-jdbc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/orm/hibernate-core/6.6.53.Final/hibernate-core-6.6.53.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/persistence/jakarta.persistence-api/3.1.0/jakarta.persistence-api-3.1.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/transaction/jakarta.transaction-api/2.0.1/jakarta.transaction-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/jboss/logging/jboss-logging/3.6.3.Final/jboss-logging-3.6.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/common/hibernate-commons-annotations/7.0.3.Final/hibernate-commons-annotations-7.0.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/smallrye/jandex/3.2.0/jandex-3.2.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/classmate/1.7.3/classmate-1.7.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy/1.17.8/byte-buddy-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-runtime/4.0.9/jaxb-runtime-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-core/4.0.9/jaxb-core-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/eclipse/angus/angus-activation/2.0.3/angus-activation-2.0.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/txw2/4.0.9/txw2-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/sun/istack/istack-commons-runtime/4.1.2/istack-commons-runtime-4.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/inject/jakarta.inject-api/2.0.1/jakarta.inject-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/antlr/antlr4-runtime/4.13.2/antlr4-runtime-4.13.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-jpa/3.5.13/spring-data-jpa-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-commons/3.5.13/spring-data-commons-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-orm/6.2.19/spring-orm-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-tx/6.2.19/spring-tx-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/slf4j-api/2.0.18/slf4j-api-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aspects/6.2.19/spring-aspects-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/aspectj/aspectjweaver/1.9.25.1/aspectjweaver-1.9.25.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-security/3.5.16/spring-boot-starter-security-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aop/6.2.19/spring-aop-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-config/6.5.11/spring-security-config-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-core/6.5.11/spring-security-core-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-crypto/6.5.11/spring-security-crypto-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-web/6.5.11/spring-security-web-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-database-postgresql/11.7.2/flyway-database-postgresql-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-core/11.7.2/flyway-core-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/dataformat/jackson-dataformat-toml/2.21.4/jackson-dataformat-toml-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/postgresql/postgresql/42.7.11/postgresql-42.7.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-test/3.5.16/spring-boot-starter-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test/3.5.16/spring-boot-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test-autoconfigure/3.5.16/spring-boot-test-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/jayway/jsonpath/json-path/2.9.0/json-path-2.9.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/xml/bind/jakarta.xml.bind-api/4.0.5/jakarta.xml.bind-api-4.0.5.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/activation/jakarta.activation-api/2.1.4/jakarta.activation-api-2.1.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/json-smart/2.5.2/json-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/accessors-smart/2.5.2/accessors-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/ow2/asm/asm/9.7.1/asm-9.7.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/assertj/assertj-core/3.27.7/assertj-core-3.27.7.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/awaitility/awaitility/4.2.2/awaitility-4.2.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hamcrest/hamcrest/3.0/hamcrest-3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter/5.12.2/junit-jupiter-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-api/5.12.2/junit-jupiter-api-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/opentest4j/opentest4j/1.3.0/opentest4j-1.3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-commons/1.12.2/junit-platform-commons-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apiguardian/apiguardian-api/1.1.2/apiguardian-api-1.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-params/5.12.2/junit-jupiter-params-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-engine/5.12.2/junit-jupiter-engine-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-engine/1.12.2/junit-platform-engine-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-core/5.17.0/mockito-core-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy-agent/1.17.8/byte-buddy-agent-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/objenesis/objenesis/3.3/objenesis-3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-junit-jupiter/5.17.0/mockito-junit-jupiter-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/skyscreamer/jsonassert/1.5.3/jsonassert-1.5.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/vaadin/external/google/android-json/0.0.20131108.vaadin1/android-json-0.0.20131108.vaadin1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-core/6.2.19/spring-core-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jcl/6.2.19/spring-jcl-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-test/6.2.19/spring-test-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/xmlunit/xmlunit-core/2.10.4/xmlunit-core-2.10.4.jar:"/>
+    <property name="java.vm.vendor" value="Eclipse Adoptium"/>
+    <property name="sun.arch.data.model" value="64"/>
+    <property name="java.vendor.url" value="https://adoptium.net/"/>
+    <property name="catalina.useNaming" value="false"/>
+    <property name="user.timezone" value="Asia/Seoul"/>
+    <property name="org.jboss.logging.provider" value="slf4j"/>
+    <property name="os.name" value="Mac OS X"/>
+    <property name="java.vm.specification.version" value="21"/>
+    <property name="APPLICATION_NAME" value="monitor-api"/>
+    <property name="sun.java.launcher" value="SUN_STANDARD"/>
+    <property name="user.country" value="KR"/>
+    <property name="sun.boot.library.path" value="/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem/lib"/>
+    <property name="sun.java.command" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/surefire/surefirebooter-20260828161521074_3.jar /private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/surefire 2026-08-28T16-15-21_034-jvmRun1 surefire-20260828161521074_1tmp surefire_0-20260828161521074_2tmp"/>
+    <property name="http.nonProxyHosts" value="local|*.local|169.254/16|*.169.254/16"/>
+    <property name="jdk.debug" value="release"/>
+    <property name="test" value="CheckRunnerTest,MonitorFunctionalTest,ApiErrorBoundaryTest"/>
+    <property name="surefire.test.class.path" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/test-classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-web/3.5.16/spring-boot-starter-web-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter/3.5.16/spring-boot-starter-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot/3.5.16/spring-boot-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-autoconfigure/3.5.16/spring-boot-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-logging/3.5.16/spring-boot-starter-logging-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-classic/1.5.34/logback-classic-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-core/1.5.34/logback-core-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-to-slf4j/2.24.3/log4j-to-slf4j-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-api/2.24.3/log4j-api-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/jul-to-slf4j/2.0.18/jul-to-slf4j-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/annotation/jakarta.annotation-api/2.1.1/jakarta.annotation-api-2.1.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/yaml/snakeyaml/2.4/snakeyaml-2.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-json/3.5.16/spring-boot-starter-json-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-databind/2.21.4/jackson-databind-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-annotations/2.21/jackson-annotations-2.21.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-core/2.21.4/jackson-core-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jdk8/2.21.4/jackson-datatype-jdk8-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jsr310/2.21.4/jackson-datatype-jsr310-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/module/jackson-module-parameter-names/2.21.4/jackson-module-parameter-names-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-tomcat/3.5.16/spring-boot-starter-tomcat-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-core/10.1.55/tomcat-embed-core-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-el/10.1.55/tomcat-embed-el-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-websocket/10.1.55/tomcat-embed-websocket-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-web/6.2.19/spring-web-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-beans/6.2.19/spring-beans-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-observation/1.15.12/micrometer-observation-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-commons/1.15.12/micrometer-commons-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-webmvc/6.2.19/spring-webmvc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-context/6.2.19/spring-context-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-expression/6.2.19/spring-expression-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-data-jpa/3.5.16/spring-boot-starter-data-jpa-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-jdbc/3.5.16/spring-boot-starter-jdbc-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/zaxxer/HikariCP/6.3.3/HikariCP-6.3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jdbc/6.2.19/spring-jdbc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/orm/hibernate-core/6.6.53.Final/hibernate-core-6.6.53.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/persistence/jakarta.persistence-api/3.1.0/jakarta.persistence-api-3.1.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/transaction/jakarta.transaction-api/2.0.1/jakarta.transaction-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/jboss/logging/jboss-logging/3.6.3.Final/jboss-logging-3.6.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/common/hibernate-commons-annotations/7.0.3.Final/hibernate-commons-annotations-7.0.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/smallrye/jandex/3.2.0/jandex-3.2.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/classmate/1.7.3/classmate-1.7.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy/1.17.8/byte-buddy-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-runtime/4.0.9/jaxb-runtime-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-core/4.0.9/jaxb-core-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/eclipse/angus/angus-activation/2.0.3/angus-activation-2.0.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/txw2/4.0.9/txw2-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/sun/istack/istack-commons-runtime/4.1.2/istack-commons-runtime-4.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/inject/jakarta.inject-api/2.0.1/jakarta.inject-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/antlr/antlr4-runtime/4.13.2/antlr4-runtime-4.13.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-jpa/3.5.13/spring-data-jpa-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-commons/3.5.13/spring-data-commons-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-orm/6.2.19/spring-orm-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-tx/6.2.19/spring-tx-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/slf4j-api/2.0.18/slf4j-api-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aspects/6.2.19/spring-aspects-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/aspectj/aspectjweaver/1.9.25.1/aspectjweaver-1.9.25.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-security/3.5.16/spring-boot-starter-security-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aop/6.2.19/spring-aop-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-config/6.5.11/spring-security-config-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-core/6.5.11/spring-security-core-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-crypto/6.5.11/spring-security-crypto-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-web/6.5.11/spring-security-web-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-database-postgresql/11.7.2/flyway-database-postgresql-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-core/11.7.2/flyway-core-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/dataformat/jackson-dataformat-toml/2.21.4/jackson-dataformat-toml-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/postgresql/postgresql/42.7.11/postgresql-42.7.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-test/3.5.16/spring-boot-starter-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test/3.5.16/spring-boot-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test-autoconfigure/3.5.16/spring-boot-test-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/jayway/jsonpath/json-path/2.9.0/json-path-2.9.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/xml/bind/jakarta.xml.bind-api/4.0.5/jakarta.xml.bind-api-4.0.5.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/activation/jakarta.activation-api/2.1.4/jakarta.activation-api-2.1.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/json-smart/2.5.2/json-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/accessors-smart/2.5.2/accessors-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/ow2/asm/asm/9.7.1/asm-9.7.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/assertj/assertj-core/3.27.7/assertj-core-3.27.7.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/awaitility/awaitility/4.2.2/awaitility-4.2.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hamcrest/hamcrest/3.0/hamcrest-3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter/5.12.2/junit-jupiter-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-api/5.12.2/junit-jupiter-api-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/opentest4j/opentest4j/1.3.0/opentest4j-1.3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-commons/1.12.2/junit-platform-commons-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apiguardian/apiguardian-api/1.1.2/apiguardian-api-1.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-params/5.12.2/junit-jupiter-params-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-engine/5.12.2/junit-jupiter-engine-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-engine/1.12.2/junit-platform-engine-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-core/5.17.0/mockito-core-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy-agent/1.17.8/byte-buddy-agent-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/objenesis/objenesis/3.3/objenesis-3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-junit-jupiter/5.17.0/mockito-junit-jupiter-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/skyscreamer/jsonassert/1.5.3/jsonassert-1.5.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/vaadin/external/google/android-json/0.0.20131108.vaadin1/android-json-0.0.20131108.vaadin1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-core/6.2.19/spring-core-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jcl/6.2.19/spring-jcl-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-test/6.2.19/spring-test-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/xmlunit/xmlunit-core/2.10.4/xmlunit-core-2.10.4.jar:"/>
+    <property name="sun.cpu.endian" value="little"/>
+    <property name="user.home" value="/Users/woopinbell"/>
+    <property name="user.language" value="ko"/>
+    <property name="java.specification.vendor" value="Oracle Corporation"/>
+    <property name="java.version.date" value="2025-04-15"/>
+    <property name="java.home" value="/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"/>
+    <property name="surefire.skipAfterFailureCount" value="1"/>
+    <property name="file.separator" value="/"/>
+    <property name="basedir" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend"/>
+    <property name="java.vm.compressedOopsMode" value="Zero based"/>
+    <property name="line.separator" value="&#10;"/>
+    <property name="java.vm.specification.vendor" value="Oracle Corporation"/>
+    <property name="java.specification.name" value="Java Platform API Specification"/>
+    <property name="FILE_LOG_CHARSET" value="UTF-8"/>
+    <property name="java.awt.headless" value="true"/>
+    <property name="apple.awt.application.name" value="ForkedBooter"/>
+    <property name="surefire.real.class.path" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/surefire/surefirebooter-20260828161521074_3.jar"/>
+    <property name="polyglot.engine.WarnInterpreterOnly" value="false"/>
+    <property name="sun.management.compiler" value="HotSpot 64-Bit Tiered Compilers"/>
+    <property name="ftp.nonProxyHosts" value="local|*.local|169.254/16|*.169.254/16"/>
+    <property name="java.runtime.version" value="21.0.7+6-LTS"/>
+    <property name="user.name" value="woopinbell"/>
+    <property name="stdout.encoding" value="UTF-8"/>
+    <property name="path.separator" value=":"/>
+    <property name="os.version" value="26.6.2"/>
+    <property name="java.runtime.name" value="OpenJDK Runtime Environment"/>
+    <property name="file.encoding" value="UTF-8"/>
+    <property name="java.vm.name" value="OpenJDK 64-Bit Server VM"/>
+    <property name="java.vendor.version" value="Temurin-21.0.7+6"/>
+    <property name="localRepository" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository"/>
+    <property name="java.vendor.url.bug" value="https://github.com/adoptium/adoptium-support/issues"/>
+    <property name="java.io.tmpdir" value="/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/"/>
+    <property name="catalina.home" value="/private/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/tomcat.4322.4578628019067252483"/>
+    <property name="com.zaxxer.hikari.pool_number" value="1"/>
+    <property name="java.version" value="21.0.7"/>
+    <property name="user.dir" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend"/>
+    <property name="os.arch" value="aarch64"/>
+    <property name="java.vm.specification.name" value="Java Virtual Machine Specification"/>
+    <property name="PID" value="1550"/>
+    <property name="CONSOLE_LOG_CHARSET" value="UTF-8"/>
+    <property name="catalina.base" value="/private/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/tomcat.4322.4578628019067252483"/>
+    <property name="native.encoding" value="UTF-8"/>
+    <property name="java.library.path" value="/Users/woopinbell/Library/Java/Extensions:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java:."/>
+    <property name="java.vm.info" value="mixed mode, sharing"/>
+    <property name="stderr.encoding" value="UTF-8"/>
+    <property name="java.vendor" value="Eclipse Adoptium"/>
+    <property name="java.vm.version" value="21.0.7+6-LTS"/>
+    <property name="sun.io.unicode.encoding" value="UnicodeBig"/>
+    <property name="maven.repo.local" value=".m2/repository"/>
+    <property name="socksNonProxyHosts" value="local|*.local|169.254/16|*.169.254/16"/>
+    <property name="java.class.version" value="65.0"/>
+    <property name="LOGGED_APPLICATION_NAME" value="[monitor-api] "/>
+  </properties>
+  <testcase name="workerOutboundRequestSeesCommittedRunningWithoutHoldingAStoreTransaction" classname="dev.evolution.monitor.MonitorFunctionalTest" time="0.133">
+    <system-out><![CDATA[16:15:26.186 [main] INFO org.springframework.test.context.support.AnnotationConfigContextLoaderUtils -- Could not detect default configuration classes for test class [dev.evolution.monitor.MonitorFunctionalTest]: MonitorFunctionalTest does not declare any static, non-private, non-final, nested classes annotated with @Configuration.
+16:15:26.256 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper -- Found @SpringBootConfiguration dev.evolution.monitor.MonitorApplication for test class dev.evolution.monitor.MonitorFunctionalTest
+
+  .   ____          _            __ _ _
+ /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
+( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
+ \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
+  '  |____| .__|_| |_|_| |_\__, | / / / /
+ =========|_|==============|___/=/_/_/_/
+
+ :: Spring Boot ::               (v3.5.16)
+
+2026-08-28T16:15:26.500+09:00  INFO 1550 --- [monitor-api] [           main] d.e.monitor.MonitorFunctionalTest        : Starting MonitorFunctionalTest using Java 21.0.7 with PID 1550 (started by woopinbell in /private/tmp/web-systems-evolution-0a006589-industry-spring/backend)
+2026-08-28T16:15:26.501+09:00  INFO 1550 --- [monitor-api] [           main] d.e.monitor.MonitorFunctionalTest        : No active profile set, falling back to 1 default profile: "default"
+2026-08-28T16:15:26.839+09:00  INFO 1550 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT mode.
+2026-08-28T16:15:26.854+09:00  INFO 1550 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 9 ms. Found 0 JPA repository interfaces.
+2026-08-28T16:15:27.122+09:00  INFO 1550 --- [monitor-api] [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port 4322 (http)
+2026-08-28T16:15:27.131+09:00  INFO 1550 --- [monitor-api] [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
+2026-08-28T16:15:27.131+09:00  INFO 1550 --- [monitor-api] [           main] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.1.55]
+2026-08-28T16:15:27.168+09:00  INFO 1550 --- [monitor-api] [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
+2026-08-28T16:15:27.169+09:00  INFO 1550 --- [monitor-api] [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 660 ms
+2026-08-28T16:15:27.539+09:00  INFO 1550 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...
+2026-08-28T16:15:27.557+09:00  INFO 1550 --- [monitor-api] [           main] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Added connection org.postgresql.jdbc.PgConnection@3d1c8f35
+2026-08-28T16:15:27.558+09:00  INFO 1550 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.
+2026-08-28T16:15:27.581+09:00  INFO 1550 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)
+2026-08-28T16:15:27.616+09:00  INFO 1550 --- [monitor-api] [           main] o.f.c.i.s.JdbcTableSchemaHistory         : Schema history table "e04_functional"."flyway_schema_history" does not exist yet
+2026-08-28T16:15:27.619+09:00  INFO 1550 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 8 migrations (execution time 00:00.015s)
+2026-08-28T16:15:27.637+09:00  INFO 1550 --- [monitor-api] [           main] o.f.c.i.s.JdbcTableSchemaHistory         : Creating Schema History table "e04_functional"."flyway_schema_history" ...
+2026-08-28T16:15:27.672+09:00  INFO 1550 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema "e04_functional": << Empty Schema >>
+2026-08-28T16:15:27.676+09:00  INFO 1550 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e04_functional" to version "1 - create monitors"
+2026-08-28T16:15:27.696+09:00  INFO 1550 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e04_functional" to version "2 - create check runs"
+2026-08-28T16:15:27.713+09:00  INFO 1550 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e04_functional" to version "3 - create users"
+2026-08-28T16:15:27.745+09:00  INFO 1550 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e04_functional" to version "4 - require monitor ownership"
+2026-08-28T16:15:27.771+09:00  INFO 1550 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e04_functional" to version "5 - index check history"
+2026-08-28T16:15:27.785+09:00  INFO 1550 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e04_functional" to version "6 - queue check execution"
+2026-08-28T16:15:27.797+09:00  INFO 1550 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e04_functional" to version "7 - execution ownership and manual identity"
+2026-08-28T16:15:27.807+09:00  INFO 1550 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e04_functional" to version "8 - recover expired executions"
+2026-08-28T16:15:27.828+09:00  INFO 1550 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Successfully applied 8 migrations to schema "e04_functional", now at version v8 (execution time 00:00.062s)
+2026-08-28T16:15:27.882+09:00  INFO 1550 --- [monitor-api] [           main] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]
+2026-08-28T16:15:27.916+09:00  INFO 1550 --- [monitor-api] [           main] org.hibernate.Version                    : HHH000412: Hibernate ORM core version 6.6.53.Final
+2026-08-28T16:15:27.935+09:00  INFO 1550 --- [monitor-api] [           main] o.h.c.internal.RegionFactoryInitiator    : HHH000026: Second-level cache disabled
+2026-08-28T16:15:27.999+09:00  INFO 1550 --- [monitor-api] [           main] o.s.o.j.p.SpringPersistenceUnitInfo      : No LoadTimeWeaver setup: ignoring JPA class transformer
+2026-08-28T16:15:28.043+09:00  INFO 1550 --- [monitor-api] [           main] org.hibernate.orm.connections.pooling    : HHH10001005: Database info:
+	Database JDBC URL [Connecting through datasource 'HikariDataSource (HikariPool-1)']
+	Database driver: undefined/unknown
+	Database version: 17.11
+	Autocommit mode: undefined/unknown
+	Isolation level: undefined/unknown
+	Minimum pool size: undefined/unknown
+	Maximum pool size: undefined/unknown
+2026-08-28T16:15:28.389+09:00  INFO 1550 --- [monitor-api] [           main] o.h.e.t.j.p.i.JtaPlatformInitiator       : HHH000489: No JTA platform available (set 'hibernate.transaction.jta.platform' to enable JTA platform integration)
+2026-08-28T16:15:28.414+09:00  INFO 1550 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'
+2026-08-28T16:15:28.506+09:00  INFO 1550 --- [monitor-api] [           main] r$InitializeUserDetailsManagerConfigurer : Global AuthenticationManager configured with UserDetailsService bean with name userAccounts
+2026-08-28T16:15:28.770+09:00  INFO 1550 --- [monitor-api] [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 4322 (http) with context path '/'
+2026-08-28T16:15:28.776+09:00  INFO 1550 --- [monitor-api] [           main] d.e.monitor.MonitorFunctionalTest        : Started MonitorFunctionalTest in 2.468 seconds (process running for 7.658)
+2026-08-28T16:15:29.206+09:00  INFO 1550 --- [monitor-api] [0.1-4322-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
+2026-08-28T16:15:29.206+09:00  INFO 1550 --- [monitor-api] [0.1-4322-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
+2026-08-28T16:15:29.206+09:00  INFO 1550 --- [monitor-api] [0.1-4322-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 0 ms
+]]></system-out>
+  </testcase>
+  <testcase name="successWireModelKeepsJsonPrimitivesAndExplicitNulls" classname="dev.evolution.monitor.MonitorFunctionalTest" time="0.035"/>
+  <testcase name="publicUrlsAreCanonicalAndDurableWithoutDnsOrCheckIoDuringCreateAndUpdate" classname="dev.evolution.monitor.MonitorFunctionalTest" time="0.12"/>
+  <testcase name="rejectsInvalidNameLengthAndUrlSyntax" classname="dev.evolution.monitor.MonitorFunctionalTest" time="0.02"/>
+  <testcase name="rejectsOverflowedNumericIntervalWithoutMutatingMonitors" classname="dev.evolution.monitor.MonitorFunctionalTest" time="0.028"/>
+  <testcase name="malformedJsonRootsAndMediaTypesUseInputErrorEnvelope" classname="dev.evolution.monitor.MonitorFunctionalTest" time="0.022"/>
+  <testcase name="rejectsWrongJsonTypesAndMissingFields" classname="dev.evolution.monitor.MonitorFunctionalTest" time="0.061"/>
+  <testcase name="postgresTextBoundaryRejectsNulWithoutCreationOrReplacement" classname="dev.evolution.monitor.MonitorFunctionalTest" time="0.038"/>
+  <testcase name="observed503IsAnEndpointFailure" classname="dev.evolution.monitor.MonitorFunctionalTest" time="0.032"/>
+  <testcase name="acceptsTrimmedNamesAndInclusiveIntegerValueBoundaries" classname="dev.evolution.monitor.MonitorFunctionalTest" time="0.021"/>
+  <testcase name="rejectsBlankNameAtRuntime" classname="dev.evolution.monitor.MonitorFunctionalTest" time="0.003"/>
+  <testcase name="redirectsCannotLeaveConfiguredFixture" classname="dev.evolution.monitor.MonitorFunctionalTest" time="0.024">
+    <system-out><![CDATA[2026-08-28T16:15:29.941+09:00  INFO 1550 --- [monitor-api] [           main] dev.evolution.monitor.CheckRunner        : Check policy refused: UNSAFE_ADDRESS
+]]></system-out>
+  </testcase>
+  <testcase name="createAndManuallyCheckSuccessfulMonitor" classname="dev.evolution.monitor.MonitorFunctionalTest" time="0.037"/>
+  <testcase name="rejectsNonFixtureDestinationWithoutOutboundRequest" classname="dev.evolution.monitor.MonitorFunctionalTest" time="0.005"/>
+  <testcase name="missingResourcesAreDifferentFromMalformedIds" classname="dev.evolution.monitor.MonitorFunctionalTest" time="0.015"/>
+  <testcase name="redirectsUseTheFinalValidatedFixtureResponse" classname="dev.evolution.monitor.MonitorFunctionalTest" time="0.028"/>
+</testsuite>
\ No newline at end of file
diff --git a/evidence/phase-1/E12/repair1/attempt1/surefire-reports/dev.evolution.monitor.ApiErrorBoundaryTest.txt b/evidence/phase-1/E12/repair1/attempt1/surefire-reports/dev.evolution.monitor.ApiErrorBoundaryTest.txt
new file mode 100644
index 0000000..e00cf63
--- /dev/null
+++ b/evidence/phase-1/E12/repair1/attempt1/surefire-reports/dev.evolution.monitor.ApiErrorBoundaryTest.txt
@@ -0,0 +1,4 @@
+-------------------------------------------------------------------------------
+Test set: dev.evolution.monitor.ApiErrorBoundaryTest
+-------------------------------------------------------------------------------
+Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.766 s -- in dev.evolution.monitor.ApiErrorBoundaryTest
diff --git a/evidence/phase-1/E12/repair1/attempt1/surefire-reports/dev.evolution.monitor.CheckRunnerTest.txt b/evidence/phase-1/E12/repair1/attempt1/surefire-reports/dev.evolution.monitor.CheckRunnerTest.txt
new file mode 100644
index 0000000..97ede44
--- /dev/null
+++ b/evidence/phase-1/E12/repair1/attempt1/surefire-reports/dev.evolution.monitor.CheckRunnerTest.txt
@@ -0,0 +1,17 @@
+-------------------------------------------------------------------------------
+Test set: dev.evolution.monitor.CheckRunnerTest
+-------------------------------------------------------------------------------
+Tests run: 11, Failures: 1, Errors: 0, Skipped: 0, Time elapsed: 4.015 s <<< FAILURE! -- in dev.evolution.monitor.CheckRunnerTest
+dev.evolution.monitor.CheckRunnerTest.uninterruptibleDnsRemainsBoundedAndCannotConnectAfterTheDeadline -- Time elapsed: 1.776 s <<< FAILURE!
+org.opentest4j.AssertionFailedError: expected: <true> but was: <false>
+	at org.junit.jupiter.api.AssertionFailureBuilder.build(AssertionFailureBuilder.java:151)
+	at org.junit.jupiter.api.AssertionFailureBuilder.buildAndThrow(AssertionFailureBuilder.java:132)
+	at org.junit.jupiter.api.AssertTrue.failNotTrue(AssertTrue.java:63)
+	at org.junit.jupiter.api.AssertTrue.assertTrue(AssertTrue.java:36)
+	at org.junit.jupiter.api.AssertTrue.assertTrue(AssertTrue.java:31)
+	at org.junit.jupiter.api.Assertions.assertTrue(Assertions.java:183)
+	at dev.evolution.monitor.CheckRunnerTest.uninterruptibleDnsRemainsBoundedAndCannotConnectAfterTheDeadline(CheckRunnerTest.java:279)
+	at java.base/java.lang.reflect.Method.invoke(Method.java:580)
+	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
+	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
+
diff --git a/evidence/phase-1/E12/repair1/attempt1/surefire-reports/dev.evolution.monitor.MonitorFunctionalTest.txt b/evidence/phase-1/E12/repair1/attempt1/surefire-reports/dev.evolution.monitor.MonitorFunctionalTest.txt
new file mode 100644
index 0000000..c72c551
--- /dev/null
+++ b/evidence/phase-1/E12/repair1/attempt1/surefire-reports/dev.evolution.monitor.MonitorFunctionalTest.txt
@@ -0,0 +1,4 @@
+-------------------------------------------------------------------------------
+Test set: dev.evolution.monitor.MonitorFunctionalTest
+-------------------------------------------------------------------------------
+Tests run: 16, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 3.885 s -- in dev.evolution.monitor.MonitorFunctionalTest
diff --git a/evidence/phase-1/E12/repair1/invocations.jsonl b/evidence/phase-1/E12/repair1/invocations.jsonl
new file mode 100644
index 0000000..360827e
--- /dev/null
+++ b/evidence/phase-1/E12/repair1/invocations.jsonl
@@ -0,0 +1,3 @@
+{"command":"node scripts/e12-baseline.mjs","startedAt":"2026-08-28T06:55:52.514Z","elapsedSeconds":6.834,"exitCode":1}
+{"command":"mvn -B -ntp -f backend/pom.xml -Dtest=CheckRunnerTest,MonitorFunctionalTest,ApiErrorBoundaryTest -Dsurefire.skipAfterFailureCount=1 package","startedAt":"2026-08-28T07:15:17.851Z","elapsedSeconds":12.648,"exitCode":1,"signal":null}
+{"command":"mvn -B -ntp -f backend/pom.xml -Dtest=CheckRunnerTest -Dsurefire.skipAfterFailureCount=1 package","executable":"/opt/homebrew/Cellar/maven/3.9.11/bin/mvn","javaHome":"/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem","startedAt":"2026-08-28T07:33:11.273537+00:00","attempt":2,"repair":1,"testSourceSha256":"85140678d0981514f0b5e6d78b1714294814a7603f2bc548470a0047bae349b9","rootSourceReview":"index/profiles/phase-1/verification/industry-spring/E12-artifacts/root-repair1-source-review.json","elapsedSeconds":7.853,"exitCode":0,"signal":null}
diff --git a/evidence/phase-1/E12/repair1/maven-console.txt b/evidence/phase-1/E12/repair1/maven-console.txt
new file mode 100644
index 0000000..b43abd3
--- /dev/null
+++ b/evidence/phase-1/E12/repair1/maven-console.txt
@@ -0,0 +1,62 @@
+[INFO] Scanning for projects...
+[INFO] 
+[INFO] ---------------------< dev.evolution:monitor-api >----------------------
+[INFO] Building monitor-api 0.0.1
+[INFO]   from pom.xml
+[INFO] --------------------------------[ jar ]---------------------------------
+[INFO] 
+[INFO] --- enforcer:3.6.2:enforce (pinned-runtimes) @ monitor-api ---
+[INFO] Rule 0: org.apache.maven.enforcer.rules.version.RequireJavaVersion passed
+[INFO] Rule 1: org.apache.maven.enforcer.rules.version.RequireMavenVersion passed
+[INFO] 
+[INFO] --- resources:3.3.1:resources (default-resources) @ monitor-api ---
+[INFO] Copying 2 resources from src/main/resources to target/classes
+[INFO] Copying 8 resources from src/main/resources to target/classes
+[INFO] 
+[INFO] --- compiler:3.14.1:compile (default-compile) @ monitor-api ---
+[INFO] Nothing to compile - all classes are up to date.
+[INFO] 
+[INFO] --- resources:3.3.1:testResources (default-testResources) @ monitor-api ---
+[INFO] skip non existing resourceDirectory /private/tmp/web-systems-evolution-0a006589-industry-spring/backend/src/test/resources
+[INFO] 
+[INFO] --- compiler:3.14.1:testCompile (default-testCompile) @ monitor-api ---
+[INFO] Recompiling the module because of changed source code.
+[INFO] Compiling 17 source files with javac [debug parameters release 21] to target/test-classes
+[INFO] 
+[INFO] --- surefire:3.5.6:test (default-test) @ monitor-api ---
+[INFO] Using auto detected provider org.apache.maven.surefire.junitplatform.JUnitPlatformProvider
+[INFO] 
+[INFO] -------------------------------------------------------
+[INFO]  T E S T S
+[INFO] -------------------------------------------------------
+[INFO] Running dev.evolution.monitor.CheckRunnerTest
+16:33:14.768 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:33:14.770 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:33:14.771 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:33:14.771 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:33:14.772 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:33:14.772 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:33:14.773 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:33:14.773 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:33:14.774 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:33:16.466 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_ADDRESS
+16:33:18.005 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: REDIRECT_LIMIT
+[INFO] Tests run: 11, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 3.842 s -- in dev.evolution.monitor.CheckRunnerTest
+[INFO] 
+[INFO] Results:
+[INFO] 
+[INFO] Tests run: 11, Failures: 0, Errors: 0, Skipped: 0
+[INFO] 
+[INFO] 
+[INFO] --- jar:3.4.2:jar (default-jar) @ monitor-api ---
+[INFO] Building jar: /private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/monitor-api-0.0.1.jar
+[INFO] 
+[INFO] --- spring-boot:3.5.16:repackage (repackage) @ monitor-api ---
+[INFO] Replacing main artifact /private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/monitor-api-0.0.1.jar with repackaged archive, adding nested dependencies in BOOT-INF/.
+[INFO] The original artifact has been renamed to /private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/monitor-api-0.0.1.jar.original
+[INFO] ------------------------------------------------------------------------
+[INFO] BUILD SUCCESS
+[INFO] ------------------------------------------------------------------------
+[INFO] Total time:  6.724 s
+[INFO] Finished at: 2026-08-28T16:33:19+09:00
+[INFO] ------------------------------------------------------------------------
diff --git a/evidence/phase-1/E12/repair1/outbound.json b/evidence/phase-1/E12/repair1/outbound.json
new file mode 100644
index 0000000..7f246f9
--- /dev/null
+++ b/evidence/phase-1/E12/repair1/outbound.json
@@ -0,0 +1,217 @@
+{
+  "fixtureSha256" : "5889fee87a5ec4506c701e6d509a5ce43af542a680502b7fd48bde44fa993ba1",
+  "observations" : [ {
+    "transport" : "numeric resolver stub",
+    "unsafeConnectorCalls" : 0,
+    "case" : "unsafe-answer-127.0.0.1",
+    "state" : "ABORTED",
+    "httpStatus" : null,
+    "failureReason" : null,
+    "elapsedMs" : 2
+  }, {
+    "transport" : "numeric resolver stub",
+    "unsafeConnectorCalls" : 0,
+    "case" : "unsafe-answer-::1",
+    "state" : "ABORTED",
+    "httpStatus" : null,
+    "failureReason" : null,
+    "elapsedMs" : 0
+  }, {
+    "transport" : "numeric resolver stub",
+    "unsafeConnectorCalls" : 0,
+    "case" : "unsafe-answer-10.0.0.1",
+    "state" : "ABORTED",
+    "httpStatus" : null,
+    "failureReason" : null,
+    "elapsedMs" : 0
+  }, {
+    "transport" : "numeric resolver stub",
+    "unsafeConnectorCalls" : 0,
+    "case" : "unsafe-answer-fc00::1",
+    "state" : "ABORTED",
+    "httpStatus" : null,
+    "failureReason" : null,
+    "elapsedMs" : 0
+  }, {
+    "transport" : "numeric resolver stub",
+    "unsafeConnectorCalls" : 0,
+    "case" : "unsafe-answer-169.254.169.254",
+    "state" : "ABORTED",
+    "httpStatus" : null,
+    "failureReason" : null,
+    "elapsedMs" : 0
+  }, {
+    "transport" : "numeric resolver stub",
+    "unsafeConnectorCalls" : 0,
+    "case" : "unsafe-answer-fe80::1",
+    "state" : "ABORTED",
+    "httpStatus" : null,
+    "failureReason" : null,
+    "elapsedMs" : 0
+  }, {
+    "transport" : "numeric resolver stub",
+    "unsafeConnectorCalls" : 0,
+    "case" : "unsafe-answer-::ffff:127.0.0.1",
+    "state" : "ABORTED",
+    "httpStatus" : null,
+    "failureReason" : null,
+    "elapsedMs" : 0
+  }, {
+    "transport" : "resolver stub",
+    "unsafeConnectorCalls" : 0,
+    "case" : "private.e12.test",
+    "state" : "ABORTED",
+    "httpStatus" : null,
+    "failureReason" : null,
+    "elapsedMs" : 0
+  }, {
+    "transport" : "resolver stub",
+    "unsafeConnectorCalls" : 0,
+    "case" : "mixed.e12.test",
+    "state" : "ABORTED",
+    "httpStatus" : null,
+    "failureReason" : null,
+    "elapsedMs" : 0
+  }, {
+    "resolverThreads" : 1,
+    "connectorCalls" : 0,
+    "transport" : "explicit resolver barrier",
+    "case" : "blocked-DNS",
+    "state" : "FAILED",
+    "httpStatus" : null,
+    "failureReason" : "TIMEOUT",
+    "elapsedMs" : 1505,
+    "queuedTasksAccepted" : 0,
+    "cleanupMs" : 0,
+    "actualIoThreadExited" : true
+  }, {
+    "bodyBytesRead" : 0,
+    "dnsCalls" : 1,
+    "socketClosed" : true,
+    "connectedAddress" : "93.184.216.34",
+    "connectorCalls" : 1,
+    "transport" : "connector stub; no live TLS",
+    "logicalHost" : "public.e12.test",
+    "case" : "validated-93.184.216.34",
+    "state" : "SUCCEEDED",
+    "httpStatus" : 200,
+    "failureReason" : null,
+    "elapsedMs" : 0
+  }, {
+    "bodyBytesRead" : 0,
+    "dnsCalls" : 1,
+    "socketClosed" : true,
+    "connectedAddress" : "2606:4700:4700:0:0:0:0:1111",
+    "connectorCalls" : 1,
+    "transport" : "connector stub; no live TLS",
+    "logicalHost" : "public.e12.test",
+    "case" : "validated-2606:4700:4700::1111",
+    "state" : "SUCCEEDED",
+    "httpStatus" : 200,
+    "failureReason" : null,
+    "elapsedMs" : 0
+  }, {
+    "sniHost" : "public.e12.test",
+    "case" : "TLS-configuration-only",
+    "liveTlsHandshakeTested" : false,
+    "endpointIdentification" : "HTTPS"
+  }, {
+    "transport" : "actual local TCP",
+    "port" : 4325,
+    "case" : "closed-local-port",
+    "state" : "FAILED",
+    "httpStatus" : null,
+    "failureReason" : "CONNECTION_FAILURE",
+    "elapsedMs" : 2
+  }, {
+    "case" : "canonical-create-boundary",
+    "connectorCalls" : 0,
+    "canonicalUrl" : "http://public.e12.test/ok",
+    "dnsCalls" : 0
+  }, {
+    "unsafeConnectorCalls" : 0,
+    "socketClosed" : true,
+    "transport" : "connector stub",
+    "safeConnectorCalls" : 1,
+    "case" : "redirect-private",
+    "state" : "ABORTED",
+    "httpStatus" : null,
+    "failureReason" : null,
+    "elapsedMs" : 0
+  }, {
+    "terminalResultCreated" : false,
+    "case" : "service-uncertainty",
+    "exceptionPropagated" : true,
+    "automaticRetries" : 0
+  }, {
+    "transport" : "actual pinned local HTTP",
+    "paths" : [ "/ok" ],
+    "connectorCalls" : 1,
+    "inputBytesRead" : 38,
+    "allRawSocketsClosed" : true,
+    "bodyBytesRead" : 0,
+    "case" : "local-ok",
+    "state" : "SUCCEEDED",
+    "httpStatus" : 200,
+    "failureReason" : null,
+    "elapsedMs" : 4
+  }, {
+    "transport" : "actual pinned local HTTP",
+    "paths" : [ "/body" ],
+    "connectorCalls" : 1,
+    "inputBytesRead" : 42,
+    "allRawSocketsClosed" : true,
+    "bodyBytesOffered" : 65537,
+    "bodyBytesRead" : 0,
+    "case" : "local-body",
+    "state" : "SUCCEEDED",
+    "httpStatus" : 200,
+    "failureReason" : null,
+    "elapsedMs" : 0
+  }, {
+    "transport" : "actual pinned local HTTP",
+    "paths" : [ "/informational" ],
+    "connectorCalls" : 1,
+    "inputBytesRead" : 82,
+    "allRawSocketsClosed" : true,
+    "bodyBytesRead" : 0,
+    "case" : "local-informational",
+    "state" : "SUCCEEDED",
+    "httpStatus" : 200,
+    "failureReason" : null,
+    "elapsedMs" : 0
+  }, {
+    "transport" : "actual pinned local HTTP",
+    "paths" : [ "/trickle" ],
+    "connectorCalls" : 1,
+    "inputBytesRead" : 4,
+    "allRawSocketsClosed" : true,
+    "case" : "local-trickle",
+    "state" : "FAILED",
+    "httpStatus" : null,
+    "failureReason" : "TIMEOUT",
+    "elapsedMs" : 1505
+  }, {
+    "transport" : "actual pinned local HTTP",
+    "paths" : [ "/redirect/0", "/redirect/1", "/redirect/2", "/redirect/3" ],
+    "connectorCalls" : 4,
+    "inputBytesRead" : 256,
+    "allRawSocketsClosed" : true,
+    "case" : "local-redirect",
+    "state" : "ABORTED",
+    "httpStatus" : null,
+    "failureReason" : null,
+    "elapsedMs" : 8
+  }, {
+    "transport" : "actual local HTTP",
+    "delayMs" : 2000,
+    "requests" : 1,
+    "responseHeadersSent" : false,
+    "case" : "slow-headers",
+    "state" : "FAILED",
+    "httpStatus" : null,
+    "failureReason" : "TIMEOUT",
+    "elapsedMs" : 502
+  } ],
+  "externalNetworkUsed" : false
+}
diff --git a/evidence/phase-1/E12/repair1/source-adoption.json b/evidence/phase-1/E12/repair1/source-adoption.json
new file mode 100644
index 0000000..ff4179c
--- /dev/null
+++ b/evidence/phase-1/E12/repair1/source-adoption.json
@@ -0,0 +1,133 @@
+{
+  "adoptionCommit": "595dd9980c8a36660ab641e79a9e8f91c1320a0f",
+  "verifiedSourceCommit": "d9b0f36904c4a86391793db83a19269e9b01f8f8",
+  "preservedHead": "d2fdc3c2c9d6e51ed53e389f3014432055f4972a",
+  "preservationManifest": "index/profiles/phase-1/preserved/industry-E12-attempt1/manifest.json",
+  "preservedLocalSnapshot": "output/phase-1/e12/repair1/attempt1",
+  "sourceFiles": {
+    "backend/src/main/java/dev/evolution/monitor/CheckRunner.java": {
+      "preservedSha256": "3c19ab3c35bfdbcf0587018b6ea0d033fb6352c948aa575ed1acebe2ce40db08",
+      "adoptedGitBlobSha256": "3c19ab3c35bfdbcf0587018b6ea0d033fb6352c948aa575ed1acebe2ce40db08",
+      "verifiedSourceSha256": "3c19ab3c35bfdbcf0587018b6ea0d033fb6352c948aa575ed1acebe2ce40db08",
+      "unchangedSinceAdoption": true
+    },
+    "backend/src/main/java/dev/evolution/monitor/MonitorController.java": {
+      "preservedSha256": "c5f387343b3e2ece2e0b3eef5d85e24cf639b5eeafa4f3c8cdc8923e21e2b119",
+      "adoptedGitBlobSha256": "c5f387343b3e2ece2e0b3eef5d85e24cf639b5eeafa4f3c8cdc8923e21e2b119",
+      "verifiedSourceSha256": "c5f387343b3e2ece2e0b3eef5d85e24cf639b5eeafa4f3c8cdc8923e21e2b119",
+      "unchangedSinceAdoption": true
+    },
+    "backend/src/main/resources/application.properties": {
+      "preservedSha256": "5e7e55fede67e84703fb8bdbf8e80ae7ff9a6748d3be6b1069fbf76cc3fc0a8d",
+      "adoptedGitBlobSha256": "5e7e55fede67e84703fb8bdbf8e80ae7ff9a6748d3be6b1069fbf76cc3fc0a8d",
+      "verifiedSourceSha256": "5e7e55fede67e84703fb8bdbf8e80ae7ff9a6748d3be6b1069fbf76cc3fc0a8d",
+      "unchangedSinceAdoption": true
+    },
+    "backend/src/test/java/dev/evolution/monitor/ApiErrorBoundaryTest.java": {
+      "preservedSha256": "ab0eca51905ffdaab051564988e1f1e4f1f2d62ca11ca8c28be4fa13ca4438ab",
+      "adoptedGitBlobSha256": "ab0eca51905ffdaab051564988e1f1e4f1f2d62ca11ca8c28be4fa13ca4438ab",
+      "verifiedSourceSha256": "ab0eca51905ffdaab051564988e1f1e4f1f2d62ca11ca8c28be4fa13ca4438ab",
+      "unchangedSinceAdoption": true
+    },
+    "backend/src/test/java/dev/evolution/monitor/CheckRunnerTest.java": {
+      "preservedSha256": "d31dcacf78e22fd096d98828b3c3494a574bc2325d3d97a16c08ae5163e3b6a9",
+      "adoptedGitBlobSha256": "d31dcacf78e22fd096d98828b3c3494a574bc2325d3d97a16c08ae5163e3b6a9",
+      "verifiedSourceSha256": "85140678d0981514f0b5e6d78b1714294814a7603f2bc548470a0047bae349b9",
+      "unchangedSinceAdoption": false
+    },
+    "backend/src/test/java/dev/evolution/monitor/E10WorkerProcess.java": {
+      "preservedSha256": "2a829b14dcfb40e2d1016f025922f81493f55e6a3f14ec65cde870a8c3c62e30",
+      "adoptedGitBlobSha256": "2a829b14dcfb40e2d1016f025922f81493f55e6a3f14ec65cde870a8c3c62e30",
+      "verifiedSourceSha256": "2a829b14dcfb40e2d1016f025922f81493f55e6a3f14ec65cde870a8c3c62e30",
+      "unchangedSinceAdoption": true
+    },
+    "backend/src/test/java/dev/evolution/monitor/MonitorFunctionalTest.java": {
+      "preservedSha256": "bacff4a53750f1290d3d8f990f94de6718fd11f5306edf91ea8d2f6db6ed22f3",
+      "adoptedGitBlobSha256": "bacff4a53750f1290d3d8f990f94de6718fd11f5306edf91ea8d2f6db6ed22f3",
+      "verifiedSourceSha256": "bacff4a53750f1290d3d8f990f94de6718fd11f5306edf91ea8d2f6db6ed22f3",
+      "unchangedSinceAdoption": true
+    },
+    "backend/src/test/java/dev/evolution/monitor/TestDatabase.java": {
+      "preservedSha256": "f47348cc0edd79c94fa4485fd33aeea2f63d1a3d7725446b484f570194c8adf0",
+      "adoptedGitBlobSha256": "f47348cc0edd79c94fa4485fd33aeea2f63d1a3d7725446b484f570194c8adf0",
+      "verifiedSourceSha256": "f47348cc0edd79c94fa4485fd33aeea2f63d1a3d7725446b484f570194c8adf0",
+      "unchangedSinceAdoption": true
+    },
+    "scripts/persistence-scenario.mjs": {
+      "preservedSha256": "8cbeba58ea1c13dd41415a75196a8020e9c7a75bf2f59b5d78cd7813e83abd39",
+      "adoptedGitBlobSha256": "8cbeba58ea1c13dd41415a75196a8020e9c7a75bf2f59b5d78cd7813e83abd39",
+      "verifiedSourceSha256": "8cbeba58ea1c13dd41415a75196a8020e9c7a75bf2f59b5d78cd7813e83abd39",
+      "unchangedSinceAdoption": true
+    },
+    "scripts/test-api.mjs": {
+      "preservedSha256": "e87e0ab0604fb0417baf84152dcf319dad29c9433c32219d0828f7baeccef9c1",
+      "adoptedGitBlobSha256": "e87e0ab0604fb0417baf84152dcf319dad29c9433c32219d0828f7baeccef9c1",
+      "verifiedSourceSha256": "e87e0ab0604fb0417baf84152dcf319dad29c9433c32219d0828f7baeccef9c1",
+      "unchangedSinceAdoption": true
+    },
+    "tests/browser/worker.spec.ts": {
+      "preservedSha256": "b0843f9f87d31fe30af281f09e3d0fb504327d5193c8c53eae6ff93657d78d5e",
+      "adoptedGitBlobSha256": "b0843f9f87d31fe30af281f09e3d0fb504327d5193c8c53eae6ff93657d78d5e",
+      "verifiedSourceSha256": "b0843f9f87d31fe30af281f09e3d0fb504327d5193c8c53eae6ff93657d78d5e",
+      "unchangedSinceAdoption": true
+    },
+    "backend/src/main/java/dev/evolution/monitor/OutboundUrl.java": {
+      "preservedSha256": "b9d3d71a9f806f7d0774c90234df5116c3f65464cc1bec59368dd7cf9a0edf81",
+      "adoptedGitBlobSha256": "b9d3d71a9f806f7d0774c90234df5116c3f65464cc1bec59368dd7cf9a0edf81",
+      "verifiedSourceSha256": "b9d3d71a9f806f7d0774c90234df5116c3f65464cc1bec59368dd7cf9a0edf81",
+      "unchangedSinceAdoption": true
+    }
+  },
+  "frozenInputs": {
+    "evidence/phase-1/E12/fixtures.md": "5889fee87a5ec4506c701e6d509a5ce43af542a680502b7fd48bde44fa993ba1",
+    "scripts/e12-baseline.mjs": "53f6da8f313709964f546e6051d219a93e2d05b23209b75b275b85e7fbef05b9"
+  },
+  "allTwelveAdoptedUnchanged": true,
+  "onlySourceChangedByRepair": "backend/src/test/java/dev/evolution/monitor/CheckRunnerTest.java",
+  "allOriginalRawOutputsRetained": {
+    "output/phase-1/e12/outbound.json": {
+      "sha256": "6d77322108c291c3703febebe351492c45399872104ad745862f504d7cbb9ea0",
+      "snapshot": "index/profiles/phase-1/preserved/industry-E12-attempt1/outputs/output/phase-1/e12/outbound.json"
+    },
+    "output/phase-1/e12/baseline-api.log": {
+      "sha256": "ae6378d38cff71844fddfa84f4fec4b4c6f9f49be79e0096a9de464f08dc5888",
+      "snapshot": "index/profiles/phase-1/preserved/industry-E12-attempt1/outputs/output/phase-1/e12/baseline-api.log"
+    },
+    "output/phase-1/e12/baseline.json": {
+      "sha256": "e4039232544254b57a868d93dffeaa12dc84e612949fa23001dfee94d351c5b1",
+      "snapshot": "index/profiles/phase-1/preserved/industry-E12-attempt1/outputs/output/phase-1/e12/baseline.json"
+    },
+    "output/phase-1/e12/author-maven.log": {
+      "sha256": "357d0c5e7399a46b3e09d4b906ac9ce8781e1b91d1bf6c1ddd99deff1c85c9de",
+      "snapshot": "index/profiles/phase-1/preserved/industry-E12-attempt1/outputs/output/phase-1/e12/author-maven.log"
+    },
+    "output/phase-1/e12/invocations.jsonl": {
+      "sha256": "5e42e3ca5f782523dc0e51dc989707d6679d1697e3391a08c170b8a9ad8e40eb",
+      "snapshot": "index/profiles/phase-1/preserved/industry-E12-attempt1/outputs/output/phase-1/e12/invocations.jsonl"
+    },
+    "backend/target/surefire-reports/TEST-dev.evolution.monitor.CheckRunnerTest.xml": {
+      "sha256": "e2346f4e82da3f2b0fe394278cc87ba9865e428f484941eb432c5c76b3b129c7",
+      "snapshot": "index/profiles/phase-1/preserved/industry-E12-attempt1/outputs/backend/target/surefire-reports/TEST-dev.evolution.monitor.CheckRunnerTest.xml"
+    },
+    "backend/target/surefire-reports/dev.evolution.monitor.CheckRunnerTest.txt": {
+      "sha256": "d770fb39d466eb1012eee9b3fb38ed45b6c303abfe225da50e1f2da042a2d360",
+      "snapshot": "index/profiles/phase-1/preserved/industry-E12-attempt1/outputs/backend/target/surefire-reports/dev.evolution.monitor.CheckRunnerTest.txt"
+    },
+    "backend/target/surefire-reports/TEST-dev.evolution.monitor.MonitorFunctionalTest.xml": {
+      "sha256": "c76d679abe0714adf1e2051f61e77f9670e04b0cdf1edb6a110ffbddcfc08f3a",
+      "snapshot": "index/profiles/phase-1/preserved/industry-E12-attempt1/outputs/backend/target/surefire-reports/TEST-dev.evolution.monitor.MonitorFunctionalTest.xml"
+    },
+    "backend/target/surefire-reports/dev.evolution.monitor.MonitorFunctionalTest.txt": {
+      "sha256": "6e1d1a0128d44026fcd6c87d912ff3abc1e3b45a2bb6ed4f144809439b05ddf4",
+      "snapshot": "index/profiles/phase-1/preserved/industry-E12-attempt1/outputs/backend/target/surefire-reports/dev.evolution.monitor.MonitorFunctionalTest.txt"
+    },
+    "backend/target/surefire-reports/TEST-dev.evolution.monitor.ApiErrorBoundaryTest.xml": {
+      "sha256": "1a27ecb7955b63aa4c8b510c6a70b0e150ac9113e261b04bc016218670e62dac",
+      "snapshot": "index/profiles/phase-1/preserved/industry-E12-attempt1/outputs/backend/target/surefire-reports/TEST-dev.evolution.monitor.ApiErrorBoundaryTest.xml"
+    },
+    "backend/target/surefire-reports/dev.evolution.monitor.ApiErrorBoundaryTest.txt": {
+      "sha256": "6393a5d19dee7ae995f77335553863484f05d73b811025c83e525183d10223f7",
+      "snapshot": "index/profiles/phase-1/preserved/industry-E12-attempt1/outputs/backend/target/surefire-reports/dev.evolution.monitor.ApiErrorBoundaryTest.txt"
+    }
+  }
+}
diff --git a/evidence/phase-1/E12/repair1/surefire-reports/TEST-dev.evolution.monitor.CheckRunnerTest.xml b/evidence/phase-1/E12/repair1/surefire-reports/TEST-dev.evolution.monitor.CheckRunnerTest.xml
new file mode 100644
index 0000000..b92f85a
--- /dev/null
+++ b/evidence/phase-1/E12/repair1/surefire-reports/TEST-dev.evolution.monitor.CheckRunnerTest.xml
@@ -0,0 +1,93 @@
+<?xml version="1.0" encoding="UTF-8"?>
+<testsuite xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="https://maven.apache.org/surefire/maven-surefire-plugin/xsd/surefire-test-report.xsd" version="3.0.2" name="dev.evolution.monitor.CheckRunnerTest" time="3.842" tests="11" errors="0" skipped="0" failures="0" flakes="0">
+  <properties>
+    <property name="java.specification.version" value="21"/>
+    <property name="sun.jnu.encoding" value="UTF-8"/>
+    <property name="java.class.path" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/test-classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-web/3.5.16/spring-boot-starter-web-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter/3.5.16/spring-boot-starter-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot/3.5.16/spring-boot-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-autoconfigure/3.5.16/spring-boot-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-logging/3.5.16/spring-boot-starter-logging-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-classic/1.5.34/logback-classic-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-core/1.5.34/logback-core-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-to-slf4j/2.24.3/log4j-to-slf4j-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-api/2.24.3/log4j-api-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/jul-to-slf4j/2.0.18/jul-to-slf4j-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/annotation/jakarta.annotation-api/2.1.1/jakarta.annotation-api-2.1.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/yaml/snakeyaml/2.4/snakeyaml-2.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-json/3.5.16/spring-boot-starter-json-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-databind/2.21.4/jackson-databind-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-annotations/2.21/jackson-annotations-2.21.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-core/2.21.4/jackson-core-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jdk8/2.21.4/jackson-datatype-jdk8-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jsr310/2.21.4/jackson-datatype-jsr310-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/module/jackson-module-parameter-names/2.21.4/jackson-module-parameter-names-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-tomcat/3.5.16/spring-boot-starter-tomcat-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-core/10.1.55/tomcat-embed-core-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-el/10.1.55/tomcat-embed-el-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-websocket/10.1.55/tomcat-embed-websocket-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-web/6.2.19/spring-web-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-beans/6.2.19/spring-beans-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-observation/1.15.12/micrometer-observation-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-commons/1.15.12/micrometer-commons-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-webmvc/6.2.19/spring-webmvc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-context/6.2.19/spring-context-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-expression/6.2.19/spring-expression-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-data-jpa/3.5.16/spring-boot-starter-data-jpa-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-jdbc/3.5.16/spring-boot-starter-jdbc-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/zaxxer/HikariCP/6.3.3/HikariCP-6.3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jdbc/6.2.19/spring-jdbc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/orm/hibernate-core/6.6.53.Final/hibernate-core-6.6.53.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/persistence/jakarta.persistence-api/3.1.0/jakarta.persistence-api-3.1.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/transaction/jakarta.transaction-api/2.0.1/jakarta.transaction-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/jboss/logging/jboss-logging/3.6.3.Final/jboss-logging-3.6.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/common/hibernate-commons-annotations/7.0.3.Final/hibernate-commons-annotations-7.0.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/smallrye/jandex/3.2.0/jandex-3.2.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/classmate/1.7.3/classmate-1.7.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy/1.17.8/byte-buddy-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-runtime/4.0.9/jaxb-runtime-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-core/4.0.9/jaxb-core-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/eclipse/angus/angus-activation/2.0.3/angus-activation-2.0.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/txw2/4.0.9/txw2-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/sun/istack/istack-commons-runtime/4.1.2/istack-commons-runtime-4.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/inject/jakarta.inject-api/2.0.1/jakarta.inject-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/antlr/antlr4-runtime/4.13.2/antlr4-runtime-4.13.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-jpa/3.5.13/spring-data-jpa-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-commons/3.5.13/spring-data-commons-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-orm/6.2.19/spring-orm-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-tx/6.2.19/spring-tx-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/slf4j-api/2.0.18/slf4j-api-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aspects/6.2.19/spring-aspects-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/aspectj/aspectjweaver/1.9.25.1/aspectjweaver-1.9.25.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-security/3.5.16/spring-boot-starter-security-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aop/6.2.19/spring-aop-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-config/6.5.11/spring-security-config-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-core/6.5.11/spring-security-core-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-crypto/6.5.11/spring-security-crypto-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-web/6.5.11/spring-security-web-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-database-postgresql/11.7.2/flyway-database-postgresql-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-core/11.7.2/flyway-core-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/dataformat/jackson-dataformat-toml/2.21.4/jackson-dataformat-toml-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/postgresql/postgresql/42.7.11/postgresql-42.7.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-test/3.5.16/spring-boot-starter-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test/3.5.16/spring-boot-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test-autoconfigure/3.5.16/spring-boot-test-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/jayway/jsonpath/json-path/2.9.0/json-path-2.9.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/xml/bind/jakarta.xml.bind-api/4.0.5/jakarta.xml.bind-api-4.0.5.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/activation/jakarta.activation-api/2.1.4/jakarta.activation-api-2.1.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/json-smart/2.5.2/json-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/accessors-smart/2.5.2/accessors-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/ow2/asm/asm/9.7.1/asm-9.7.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/assertj/assertj-core/3.27.7/assertj-core-3.27.7.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/awaitility/awaitility/4.2.2/awaitility-4.2.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hamcrest/hamcrest/3.0/hamcrest-3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter/5.12.2/junit-jupiter-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-api/5.12.2/junit-jupiter-api-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/opentest4j/opentest4j/1.3.0/opentest4j-1.3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-commons/1.12.2/junit-platform-commons-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apiguardian/apiguardian-api/1.1.2/apiguardian-api-1.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-params/5.12.2/junit-jupiter-params-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-engine/5.12.2/junit-jupiter-engine-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-engine/1.12.2/junit-platform-engine-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-core/5.17.0/mockito-core-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy-agent/1.17.8/byte-buddy-agent-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/objenesis/objenesis/3.3/objenesis-3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-junit-jupiter/5.17.0/mockito-junit-jupiter-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/skyscreamer/jsonassert/1.5.3/jsonassert-1.5.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/vaadin/external/google/android-json/0.0.20131108.vaadin1/android-json-0.0.20131108.vaadin1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-core/6.2.19/spring-core-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jcl/6.2.19/spring-jcl-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-test/6.2.19/spring-test-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/xmlunit/xmlunit-core/2.10.4/xmlunit-core-2.10.4.jar:"/>
+    <property name="java.vm.vendor" value="Eclipse Adoptium"/>
+    <property name="sun.arch.data.model" value="64"/>
+    <property name="java.vendor.url" value="https://adoptium.net/"/>
+    <property name="user.timezone" value="Asia/Seoul"/>
+    <property name="os.name" value="Mac OS X"/>
+    <property name="java.vm.specification.version" value="21"/>
+    <property name="sun.java.launcher" value="SUN_STANDARD"/>
+    <property name="user.country" value="KR"/>
+    <property name="sun.boot.library.path" value="/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem/lib"/>
+    <property name="sun.java.command" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/surefire/surefirebooter-20260828163314410_3.jar /private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/surefire 2026-08-28T16-33-14_371-jvmRun1 surefire-20260828163314410_1tmp surefire_0-20260828163314410_2tmp"/>
+    <property name="http.nonProxyHosts" value="local|*.local|169.254/16|*.169.254/16"/>
+    <property name="jdk.debug" value="release"/>
+    <property name="test" value="CheckRunnerTest"/>
+    <property name="surefire.test.class.path" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/test-classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-web/3.5.16/spring-boot-starter-web-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter/3.5.16/spring-boot-starter-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot/3.5.16/spring-boot-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-autoconfigure/3.5.16/spring-boot-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-logging/3.5.16/spring-boot-starter-logging-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-classic/1.5.34/logback-classic-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-core/1.5.34/logback-core-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-to-slf4j/2.24.3/log4j-to-slf4j-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-api/2.24.3/log4j-api-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/jul-to-slf4j/2.0.18/jul-to-slf4j-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/annotation/jakarta.annotation-api/2.1.1/jakarta.annotation-api-2.1.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/yaml/snakeyaml/2.4/snakeyaml-2.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-json/3.5.16/spring-boot-starter-json-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-databind/2.21.4/jackson-databind-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-annotations/2.21/jackson-annotations-2.21.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-core/2.21.4/jackson-core-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jdk8/2.21.4/jackson-datatype-jdk8-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jsr310/2.21.4/jackson-datatype-jsr310-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/module/jackson-module-parameter-names/2.21.4/jackson-module-parameter-names-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-tomcat/3.5.16/spring-boot-starter-tomcat-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-core/10.1.55/tomcat-embed-core-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-el/10.1.55/tomcat-embed-el-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-websocket/10.1.55/tomcat-embed-websocket-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-web/6.2.19/spring-web-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-beans/6.2.19/spring-beans-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-observation/1.15.12/micrometer-observation-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-commons/1.15.12/micrometer-commons-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-webmvc/6.2.19/spring-webmvc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-context/6.2.19/spring-context-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-expression/6.2.19/spring-expression-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-data-jpa/3.5.16/spring-boot-starter-data-jpa-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-jdbc/3.5.16/spring-boot-starter-jdbc-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/zaxxer/HikariCP/6.3.3/HikariCP-6.3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jdbc/6.2.19/spring-jdbc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/orm/hibernate-core/6.6.53.Final/hibernate-core-6.6.53.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/persistence/jakarta.persistence-api/3.1.0/jakarta.persistence-api-3.1.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/transaction/jakarta.transaction-api/2.0.1/jakarta.transaction-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/jboss/logging/jboss-logging/3.6.3.Final/jboss-logging-3.6.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/common/hibernate-commons-annotations/7.0.3.Final/hibernate-commons-annotations-7.0.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/smallrye/jandex/3.2.0/jandex-3.2.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/classmate/1.7.3/classmate-1.7.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy/1.17.8/byte-buddy-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-runtime/4.0.9/jaxb-runtime-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-core/4.0.9/jaxb-core-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/eclipse/angus/angus-activation/2.0.3/angus-activation-2.0.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/txw2/4.0.9/txw2-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/sun/istack/istack-commons-runtime/4.1.2/istack-commons-runtime-4.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/inject/jakarta.inject-api/2.0.1/jakarta.inject-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/antlr/antlr4-runtime/4.13.2/antlr4-runtime-4.13.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-jpa/3.5.13/spring-data-jpa-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-commons/3.5.13/spring-data-commons-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-orm/6.2.19/spring-orm-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-tx/6.2.19/spring-tx-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/slf4j-api/2.0.18/slf4j-api-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aspects/6.2.19/spring-aspects-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/aspectj/aspectjweaver/1.9.25.1/aspectjweaver-1.9.25.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-security/3.5.16/spring-boot-starter-security-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aop/6.2.19/spring-aop-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-config/6.5.11/spring-security-config-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-core/6.5.11/spring-security-core-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-crypto/6.5.11/spring-security-crypto-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-web/6.5.11/spring-security-web-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-database-postgresql/11.7.2/flyway-database-postgresql-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-core/11.7.2/flyway-core-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/dataformat/jackson-dataformat-toml/2.21.4/jackson-dataformat-toml-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/postgresql/postgresql/42.7.11/postgresql-42.7.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-test/3.5.16/spring-boot-starter-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test/3.5.16/spring-boot-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test-autoconfigure/3.5.16/spring-boot-test-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/jayway/jsonpath/json-path/2.9.0/json-path-2.9.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/xml/bind/jakarta.xml.bind-api/4.0.5/jakarta.xml.bind-api-4.0.5.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/activation/jakarta.activation-api/2.1.4/jakarta.activation-api-2.1.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/json-smart/2.5.2/json-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/accessors-smart/2.5.2/accessors-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/ow2/asm/asm/9.7.1/asm-9.7.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/assertj/assertj-core/3.27.7/assertj-core-3.27.7.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/awaitility/awaitility/4.2.2/awaitility-4.2.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hamcrest/hamcrest/3.0/hamcrest-3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter/5.12.2/junit-jupiter-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-api/5.12.2/junit-jupiter-api-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/opentest4j/opentest4j/1.3.0/opentest4j-1.3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-commons/1.12.2/junit-platform-commons-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apiguardian/apiguardian-api/1.1.2/apiguardian-api-1.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-params/5.12.2/junit-jupiter-params-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-engine/5.12.2/junit-jupiter-engine-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-engine/1.12.2/junit-platform-engine-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-core/5.17.0/mockito-core-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy-agent/1.17.8/byte-buddy-agent-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/objenesis/objenesis/3.3/objenesis-3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-junit-jupiter/5.17.0/mockito-junit-jupiter-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/skyscreamer/jsonassert/1.5.3/jsonassert-1.5.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/vaadin/external/google/android-json/0.0.20131108.vaadin1/android-json-0.0.20131108.vaadin1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-core/6.2.19/spring-core-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jcl/6.2.19/spring-jcl-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-test/6.2.19/spring-test-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/xmlunit/xmlunit-core/2.10.4/xmlunit-core-2.10.4.jar:"/>
+    <property name="sun.cpu.endian" value="little"/>
+    <property name="user.home" value="/Users/woopinbell"/>
+    <property name="user.language" value="ko"/>
+    <property name="java.specification.vendor" value="Oracle Corporation"/>
+    <property name="java.version.date" value="2025-04-15"/>
+    <property name="java.home" value="/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"/>
+    <property name="surefire.skipAfterFailureCount" value="1"/>
+    <property name="file.separator" value="/"/>
+    <property name="basedir" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend"/>
+    <property name="java.vm.compressedOopsMode" value="Zero based"/>
+    <property name="line.separator" value="&#10;"/>
+    <property name="java.vm.specification.vendor" value="Oracle Corporation"/>
+    <property name="java.specification.name" value="Java Platform API Specification"/>
+    <property name="apple.awt.application.name" value="ForkedBooter"/>
+    <property name="surefire.real.class.path" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/surefire/surefirebooter-20260828163314410_3.jar"/>
+    <property name="sun.management.compiler" value="HotSpot 64-Bit Tiered Compilers"/>
+    <property name="ftp.nonProxyHosts" value="local|*.local|169.254/16|*.169.254/16"/>
+    <property name="java.runtime.version" value="21.0.7+6-LTS"/>
+    <property name="user.name" value="woopinbell"/>
+    <property name="stdout.encoding" value="UTF-8"/>
+    <property name="path.separator" value=":"/>
+    <property name="os.version" value="26.6.2"/>
+    <property name="java.runtime.name" value="OpenJDK Runtime Environment"/>
+    <property name="file.encoding" value="UTF-8"/>
+    <property name="java.vm.name" value="OpenJDK 64-Bit Server VM"/>
+    <property name="java.vendor.version" value="Temurin-21.0.7+6"/>
+    <property name="localRepository" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository"/>
+    <property name="java.vendor.url.bug" value="https://github.com/adoptium/adoptium-support/issues"/>
+    <property name="java.io.tmpdir" value="/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/"/>
+    <property name="java.version" value="21.0.7"/>
+    <property name="user.dir" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend"/>
+    <property name="os.arch" value="aarch64"/>
+    <property name="java.vm.specification.name" value="Java Virtual Machine Specification"/>
+    <property name="native.encoding" value="UTF-8"/>
+    <property name="java.library.path" value="/Users/woopinbell/Library/Java/Extensions:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java:."/>
+    <property name="java.vm.info" value="mixed mode, sharing"/>
+    <property name="stderr.encoding" value="UTF-8"/>
+    <property name="java.vendor" value="Eclipse Adoptium"/>
+    <property name="java.vm.version" value="21.0.7+6-LTS"/>
+    <property name="sun.io.unicode.encoding" value="UnicodeBig"/>
+    <property name="maven.repo.local" value=".m2/repository"/>
+    <property name="socksNonProxyHosts" value="local|*.local|169.254/16|*.169.254/16"/>
+    <property name="java.class.version" value="65.0"/>
+  </properties>
+  <testcase name="everyUnsafeLiteralAndActualDnsAnswerIsRefusedBeforeConnection" classname="dev.evolution.monitor.CheckRunnerTest" time="0.048">
+    <system-out><![CDATA[16:33:14.768 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:33:14.770 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:33:14.771 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:33:14.771 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:33:14.772 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:33:14.772 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:33:14.773 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:33:14.773 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+16:33:14.774 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_DNS_ANSWER
+]]></system-out>
+  </testcase>
+  <testcase name="uninterruptibleDnsRemainsBoundedAndCannotConnectAfterTheDeadline" classname="dev.evolution.monitor.CheckRunnerTest" time="1.656"/>
+  <testcase name="publicAnswersUseValidatedAddressesAndOriginalTlsHostnameWithoutSecondDns" classname="dev.evolution.monitor.CheckRunnerTest" time="0.011"/>
+  <testcase name="connectionFailureHasNoInventedHttpStatusOnTheWire" classname="dev.evolution.monitor.CheckRunnerTest" time="0.012"/>
+  <testcase name="canonicalUrlsPerformNoDnsOrConnectorWork" classname="dev.evolution.monitor.CheckRunnerTest" time="0.002"/>
+  <testcase name="onlyTwoHundredsAreSuccessful" classname="dev.evolution.monitor.CheckRunnerTest" time="0.001"/>
+  <testcase name="fixtureExceptionIsExplicitAndStillExactlyConfiguredHostPortAndScheme" classname="dev.evolution.monitor.CheckRunnerTest" time="0.002"/>
+  <testcase name="aRedirectToPrivateSpaceIsAbortedWithoutAnIntermediateHttpOutcome" classname="dev.evolution.monitor.CheckRunnerTest" time="0.002">
+    <system-out><![CDATA[16:33:16.466 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: UNSAFE_ADDRESS
+]]></system-out>
+  </testcase>
+  <testcase name="authoritativeServiceUncertaintyDoesNotBecomeAnEndpointResult" classname="dev.evolution.monitor.CheckRunnerTest" time="0.002"/>
+  <testcase name="realLocalResponsesRespectFinalHeadersTotalDeadlineAndRedirectBudget" classname="dev.evolution.monitor.CheckRunnerTest" time="1.537">
+    <system-out><![CDATA[16:33:18.005 [main] INFO dev.evolution.monitor.CheckRunner -- Check policy refused: REDIRECT_LIMIT
+]]></system-out>
+  </testcase>
+  <testcase name="headerTimeoutHasNoInventedHttpStatusOnTheWire" classname="dev.evolution.monitor.CheckRunnerTest" time="0.512"/>
+</testsuite>
\ No newline at end of file
diff --git a/evidence/phase-1/E12/repair1/surefire-reports/dev.evolution.monitor.CheckRunnerTest.txt b/evidence/phase-1/E12/repair1/surefire-reports/dev.evolution.monitor.CheckRunnerTest.txt
new file mode 100644
index 0000000..8bdddf9
--- /dev/null
+++ b/evidence/phase-1/E12/repair1/surefire-reports/dev.evolution.monitor.CheckRunnerTest.txt
@@ -0,0 +1,4 @@
+-------------------------------------------------------------------------------
+Test set: dev.evolution.monitor.CheckRunnerTest
+-------------------------------------------------------------------------------
+Tests run: 11, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 3.842 s -- in dev.evolution.monitor.CheckRunnerTest
diff --git a/evidence/phase-1/E12/repair1/verification.json b/evidence/phase-1/E12/repair1/verification.json
new file mode 100644
index 0000000..8ca3fef
--- /dev/null
+++ b/evidence/phase-1/E12/repair1/verification.json
@@ -0,0 +1,727 @@
+{
+  "thread": "E12",
+  "profile": "phase-1",
+  "branch": "track/industry-spring",
+  "specRevision": "2ada57a71cd34fa2fae9809415c362a8bbfcdf02",
+  "attempt": 2,
+  "repair": 1,
+  "status": "AUTHOR_PASS_ROOT_ACCEPTANCE_PENDING",
+  "recordedAt": "2026-08-28T07:38:55.540635+00:00",
+  "start": "1b168ca405b0b69fd1409a68bf8d1e3f65ea23bd",
+  "verifiedSourceCommit": "d9b0f36904c4a86391793db83a19269e9b01f8f8",
+  "adoptionCommit": "595dd9980c8a36660ab641e79a9e8f91c1320a0f",
+  "rootSourceReview": "index/profiles/phase-1/verification/industry-spring/E12-artifacts/root-repair1-source-review.json",
+  "baseline": {
+    "executions": 1,
+    "result": "REPRODUCED",
+    "evidence": "../baseline.json",
+    "productUnchanged": true,
+    "httpStatus": 400,
+    "errorCode": "INVALID_INPUT",
+    "durableMonitors": 0,
+    "durableChecks": 0,
+    "outboundCalls": 0,
+    "workersStarted": 0,
+    "elapsedSeconds": 6.834
+  },
+  "attempt1": {
+    "command": {
+      "command": "mvn -B -ntp -f backend/pom.xml -Dtest=CheckRunnerTest,MonitorFunctionalTest,ApiErrorBoundaryTest -Dsurefire.skipAfterFailureCount=1 package",
+      "startedAt": "2026-08-28T07:15:17.851Z",
+      "elapsedSeconds": 12.648,
+      "exitCode": 1,
+      "signal": null
+    },
+    "suites": [
+      {
+        "name": "dev.evolution.monitor.CheckRunnerTest",
+        "tests": 11,
+        "failures": 1,
+        "errors": 0,
+        "skipped": 0,
+        "elapsedSeconds": 4.015,
+        "methods": [
+          {
+            "name": "everyUnsafeLiteralAndActualDnsAnswerIsRefusedBeforeConnection",
+            "classname": "dev.evolution.monitor.CheckRunnerTest",
+            "time": "0.05",
+            "failures": []
+          },
+          {
+            "name": "uninterruptibleDnsRemainsBoundedAndCannotConnectAfterTheDeadline",
+            "classname": "dev.evolution.monitor.CheckRunnerTest",
+            "time": "1.776",
+            "failures": [
+              "org.opentest4j.AssertionFailedError: expected: <true> but was: <false>\n\tat org.junit.jupiter.api.AssertionFailureBuilder.build(AssertionFailureBuilder.java:151)\n\tat org.junit.jupiter.api.AssertionFailureBuilder.buildAndThrow(AssertionFailureBuilder.java:132)\n\tat org.junit.jupiter.api.AssertTrue.failNotTrue(AssertTrue.java:63)\n\tat org.junit.jupiter.api.AssertTrue.assertTrue(AssertTrue.java:36)\n\tat org.junit.jupiter.api.AssertTrue.assertTrue(AssertTrue.java:31)\n\tat org.junit.jupiter.api.Assertions.assertTrue(Assertions.java:183)\n\tat dev.evolution.monitor.CheckRunnerTest.uninterruptibleDnsRemainsBoundedAndCannotConnectAfterTheDeadline(CheckRunnerTest.java:279)\n\tat java.base/java.lang.reflect.Method.invoke(Method.java:580)\n\tat java.base/java.util.ArrayList.forEach(ArrayList.java:1596)\n\tat java.base/java.util.ArrayList.forEach(ArrayList.java:1596)\n"
+            ]
+          },
+          {
+            "name": "publicAnswersUseValidatedAddressesAndOriginalTlsHostnameWithoutSecondDns",
+            "classname": "dev.evolution.monitor.CheckRunnerTest",
+            "time": "0.016",
+            "failures": []
+          },
+          {
+            "name": "connectionFailureHasNoInventedHttpStatusOnTheWire",
+            "classname": "dev.evolution.monitor.CheckRunnerTest",
+            "time": "0.023",
+            "failures": []
+          },
+          {
+            "name": "canonicalUrlsPerformNoDnsOrConnectorWork",
+            "classname": "dev.evolution.monitor.CheckRunnerTest",
+            "time": "0.003",
+            "failures": []
+          },
+          {
+            "name": "onlyTwoHundredsAreSuccessful",
+            "classname": "dev.evolution.monitor.CheckRunnerTest",
+            "time": "0.001",
+            "failures": []
+          },
+          {
+            "name": "fixtureExceptionIsExplicitAndStillExactlyConfiguredHostPortAndScheme",
+            "classname": "dev.evolution.monitor.CheckRunnerTest",
+            "time": "0.003",
+            "failures": []
+          },
+          {
+            "name": "aRedirectToPrivateSpaceIsAbortedWithoutAnIntermediateHttpOutcome",
+            "classname": "dev.evolution.monitor.CheckRunnerTest",
+            "time": "0.002",
+            "failures": []
+          },
+          {
+            "name": "authoritativeServiceUncertaintyDoesNotBecomeAnEndpointResult",
+            "classname": "dev.evolution.monitor.CheckRunnerTest",
+            "time": "0.001",
+            "failures": []
+          },
+          {
+            "name": "realLocalResponsesRespectFinalHeadersTotalDeadlineAndRedirectBudget",
+            "classname": "dev.evolution.monitor.CheckRunnerTest",
+            "time": "1.538",
+            "failures": []
+          },
+          {
+            "name": "headerTimeoutHasNoInventedHttpStatusOnTheWire",
+            "classname": "dev.evolution.monitor.CheckRunnerTest",
+            "time": "0.515",
+            "failures": []
+          }
+        ]
+      },
+      {
+        "name": "dev.evolution.monitor.MonitorFunctionalTest",
+        "tests": 16,
+        "failures": 0,
+        "errors": 0,
+        "skipped": 0,
+        "elapsedSeconds": 3.885,
+        "methods": [
+          {
+            "name": "workerOutboundRequestSeesCommittedRunningWithoutHoldingAStoreTransaction",
+            "classname": "dev.evolution.monitor.MonitorFunctionalTest",
+            "time": "0.133",
+            "failures": []
+          },
+          {
+            "name": "successWireModelKeepsJsonPrimitivesAndExplicitNulls",
+            "classname": "dev.evolution.monitor.MonitorFunctionalTest",
+            "time": "0.035",
+            "failures": []
+          },
+          {
+            "name": "publicUrlsAreCanonicalAndDurableWithoutDnsOrCheckIoDuringCreateAndUpdate",
+            "classname": "dev.evolution.monitor.MonitorFunctionalTest",
+            "time": "0.12",
+            "failures": []
+          },
+          {
+            "name": "rejectsInvalidNameLengthAndUrlSyntax",
+            "classname": "dev.evolution.monitor.MonitorFunctionalTest",
+            "time": "0.02",
+            "failures": []
+          },
+          {
+            "name": "rejectsOverflowedNumericIntervalWithoutMutatingMonitors",
+            "classname": "dev.evolution.monitor.MonitorFunctionalTest",
+            "time": "0.028",
+            "failures": []
+          },
+          {
+            "name": "malformedJsonRootsAndMediaTypesUseInputErrorEnvelope",
+            "classname": "dev.evolution.monitor.MonitorFunctionalTest",
+            "time": "0.022",
+            "failures": []
+          },
+          {
+            "name": "rejectsWrongJsonTypesAndMissingFields",
+            "classname": "dev.evolution.monitor.MonitorFunctionalTest",
+            "time": "0.061",
+            "failures": []
+          },
+          {
+            "name": "postgresTextBoundaryRejectsNulWithoutCreationOrReplacement",
+            "classname": "dev.evolution.monitor.MonitorFunctionalTest",
+            "time": "0.038",
+            "failures": []
+          },
+          {
+            "name": "observed503IsAnEndpointFailure",
+            "classname": "dev.evolution.monitor.MonitorFunctionalTest",
+            "time": "0.032",
+            "failures": []
+          },
+          {
+            "name": "acceptsTrimmedNamesAndInclusiveIntegerValueBoundaries",
+            "classname": "dev.evolution.monitor.MonitorFunctionalTest",
+            "time": "0.021",
+            "failures": []
+          },
+          {
+            "name": "rejectsBlankNameAtRuntime",
+            "classname": "dev.evolution.monitor.MonitorFunctionalTest",
+            "time": "0.003",
+            "failures": []
+          },
+          {
+            "name": "redirectsCannotLeaveConfiguredFixture",
+            "classname": "dev.evolution.monitor.MonitorFunctionalTest",
+            "time": "0.024",
+            "failures": []
+          },
+          {
+            "name": "createAndManuallyCheckSuccessfulMonitor",
+            "classname": "dev.evolution.monitor.MonitorFunctionalTest",
+            "time": "0.037",
+            "failures": []
+          },
+          {
+            "name": "rejectsNonFixtureDestinationWithoutOutboundRequest",
+            "classname": "dev.evolution.monitor.MonitorFunctionalTest",
+            "time": "0.005",
+            "failures": []
+          },
+          {
+            "name": "missingResourcesAreDifferentFromMalformedIds",
+            "classname": "dev.evolution.monitor.MonitorFunctionalTest",
+            "time": "0.015",
+            "failures": []
+          },
+          {
+            "name": "redirectsUseTheFinalValidatedFixtureResponse",
+            "classname": "dev.evolution.monitor.MonitorFunctionalTest",
+            "time": "0.028",
+            "failures": []
+          }
+        ]
+      },
+      {
+        "name": "dev.evolution.monitor.ApiErrorBoundaryTest",
+        "tests": 1,
+        "failures": 0,
+        "errors": 0,
+        "skipped": 0,
+        "elapsedSeconds": 0.766,
+        "methods": [
+          {
+            "name": "unexpectedFailureIsAnInternalErrorWithoutPrivateExceptionDetails",
+            "classname": "dev.evolution.monitor.ApiErrorBoundaryTest",
+            "time": "0.765",
+            "failures": []
+          }
+        ]
+      }
+    ],
+    "javaTestsExecuted": 28,
+    "passed": 27,
+    "failed": 1,
+    "diagnostic": "The failed <1750ms assertion occurred after assertNoResponse constructed and used an ObjectMapper. Whole-method time was1.776s; actual run-return duration was not retained. Serialization overhead is plausible but was not proven by that evidence. Capacity/late-I/O assertions were not reached in that method.",
+    "nativeFailFastLimitation": "Pinned JUnit Platform could not cancel the run for skipAfterFailureCount=1; the single invocation completed28 methods without a retry."
+  },
+  "repairGate": {
+    "command": {
+      "command": "mvn -B -ntp -f backend/pom.xml -Dtest=CheckRunnerTest -Dsurefire.skipAfterFailureCount=1 package",
+      "executable": "/opt/homebrew/Cellar/maven/3.9.11/bin/mvn",
+      "javaHome": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem",
+      "startedAt": "2026-08-28T07:33:11.273537+00:00",
+      "attempt": 2,
+      "repair": 1,
+      "testSourceSha256": "85140678d0981514f0b5e6d78b1714294814a7603f2bc548470a0047bae349b9",
+      "rootSourceReview": "index/profiles/phase-1/verification/industry-spring/E12-artifacts/root-repair1-source-review.json",
+      "elapsedSeconds": 7.853,
+      "exitCode": 0,
+      "signal": null
+    },
+    "suite": {
+      "name": "dev.evolution.monitor.CheckRunnerTest",
+      "tests": 11,
+      "failures": 0,
+      "errors": 0,
+      "skipped": 0,
+      "elapsedSeconds": 3.842,
+      "methods": [
+        {
+          "name": "everyUnsafeLiteralAndActualDnsAnswerIsRefusedBeforeConnection",
+          "classname": "dev.evolution.monitor.CheckRunnerTest",
+          "time": "0.048",
+          "failures": []
+        },
+        {
+          "name": "uninterruptibleDnsRemainsBoundedAndCannotConnectAfterTheDeadline",
+          "classname": "dev.evolution.monitor.CheckRunnerTest",
+          "time": "1.656",
+          "failures": []
+        },
+        {
+          "name": "publicAnswersUseValidatedAddressesAndOriginalTlsHostnameWithoutSecondDns",
+          "classname": "dev.evolution.monitor.CheckRunnerTest",
+          "time": "0.011",
+          "failures": []
+        },
+        {
+          "name": "connectionFailureHasNoInventedHttpStatusOnTheWire",
+          "classname": "dev.evolution.monitor.CheckRunnerTest",
+          "time": "0.012",
+          "failures": []
+        },
+        {
+          "name": "canonicalUrlsPerformNoDnsOrConnectorWork",
+          "classname": "dev.evolution.monitor.CheckRunnerTest",
+          "time": "0.002",
+          "failures": []
+        },
+        {
+          "name": "onlyTwoHundredsAreSuccessful",
+          "classname": "dev.evolution.monitor.CheckRunnerTest",
+          "time": "0.001",
+          "failures": []
+        },
+        {
+          "name": "fixtureExceptionIsExplicitAndStillExactlyConfiguredHostPortAndScheme",
+          "classname": "dev.evolution.monitor.CheckRunnerTest",
+          "time": "0.002",
+          "failures": []
+        },
+        {
+          "name": "aRedirectToPrivateSpaceIsAbortedWithoutAnIntermediateHttpOutcome",
+          "classname": "dev.evolution.monitor.CheckRunnerTest",
+          "time": "0.002",
+          "failures": []
+        },
+        {
+          "name": "authoritativeServiceUncertaintyDoesNotBecomeAnEndpointResult",
+          "classname": "dev.evolution.monitor.CheckRunnerTest",
+          "time": "0.002",
+          "failures": []
+        },
+        {
+          "name": "realLocalResponsesRespectFinalHeadersTotalDeadlineAndRedirectBudget",
+          "classname": "dev.evolution.monitor.CheckRunnerTest",
+          "time": "1.537",
+          "failures": []
+        },
+        {
+          "name": "headerTimeoutHasNoInventedHttpStatusOnTheWire",
+          "classname": "dev.evolution.monitor.CheckRunnerTest",
+          "time": "0.512",
+          "failures": []
+        }
+      ]
+    },
+    "sourceSha256": "85140678d0981514f0b5e6d78b1714294814a7603f2bc548470a0047bae349b9",
+    "changes": "Only CheckRunnerTest observation: prepare existing runner/execution fixtures before timing, capture run-return duration before assertion/serialization work, persist blocked-DNS observation before assertions, record cleanup separately. Original fixed bounds, resolver barrier, finite capacity and late-I/O assertions remain unchanged.",
+    "packagedArtifactSha256": "d8859afc7ca787b77a70d74a3a0e5925130a689f3720e0fcdf1b5fe641ba6f80"
+  },
+  "reusedPassingSuites": [
+    {
+      "name": "MonitorFunctionalTest",
+      "tests": 16,
+      "failures": 0,
+      "errors": 0,
+      "sourceSha256": "bacff4a53750f1290d3d8f990f94de6718fd11f5306edf91ea8d2f6db6ed22f3",
+      "report": "attempt1/surefire-reports/TEST-dev.evolution.monitor.MonitorFunctionalTest.xml",
+      "basis": "The 12-file adoption matches the failed-author preservation manifest. Only CheckRunnerTest changed afterward; product, these tests, adapters, dependency pins and configuration are unchanged. This is reuse, not a rerun."
+    },
+    {
+      "name": "ApiErrorBoundaryTest",
+      "tests": 1,
+      "failures": 0,
+      "errors": 0,
+      "sourceSha256": "ab0eca51905ffdaab051564988e1f1e4f1f2d62ca11ca8c28be4fa13ca4438ab",
+      "report": "attempt1/surefire-reports/TEST-dev.evolution.monitor.ApiErrorBoundaryTest.xml",
+      "basis": "The 12-file adoption matches the failed-author preservation manifest. Only CheckRunnerTest changed afterward; product, these tests, adapters, dependency pins and configuration are unchanged. This is reuse, not a rerun."
+    }
+  ],
+  "observations": {
+    "fixtureSha256": "5889fee87a5ec4506c701e6d509a5ce43af542a680502b7fd48bde44fa993ba1",
+    "observations": [
+      {
+        "transport": "numeric resolver stub",
+        "unsafeConnectorCalls": 0,
+        "case": "unsafe-answer-127.0.0.1",
+        "state": "ABORTED",
+        "httpStatus": null,
+        "failureReason": null,
+        "elapsedMs": 2
+      },
+      {
+        "transport": "numeric resolver stub",
+        "unsafeConnectorCalls": 0,
+        "case": "unsafe-answer-::1",
+        "state": "ABORTED",
+        "httpStatus": null,
+        "failureReason": null,
+        "elapsedMs": 0
+      },
+      {
+        "transport": "numeric resolver stub",
+        "unsafeConnectorCalls": 0,
+        "case": "unsafe-answer-10.0.0.1",
+        "state": "ABORTED",
+        "httpStatus": null,
+        "failureReason": null,
+        "elapsedMs": 0
+      },
+      {
+        "transport": "numeric resolver stub",
+        "unsafeConnectorCalls": 0,
+        "case": "unsafe-answer-fc00::1",
+        "state": "ABORTED",
+        "httpStatus": null,
+        "failureReason": null,
+        "elapsedMs": 0
+      },
+      {
+        "transport": "numeric resolver stub",
+        "unsafeConnectorCalls": 0,
+        "case": "unsafe-answer-169.254.169.254",
+        "state": "ABORTED",
+        "httpStatus": null,
+        "failureReason": null,
+        "elapsedMs": 0
+      },
+      {
+        "transport": "numeric resolver stub",
+        "unsafeConnectorCalls": 0,
+        "case": "unsafe-answer-fe80::1",
+        "state": "ABORTED",
+        "httpStatus": null,
+        "failureReason": null,
+        "elapsedMs": 0
+      },
+      {
+        "transport": "numeric resolver stub",
+        "unsafeConnectorCalls": 0,
+        "case": "unsafe-answer-::ffff:127.0.0.1",
+        "state": "ABORTED",
+        "httpStatus": null,
+        "failureReason": null,
+        "elapsedMs": 0
+      },
+      {
+        "transport": "resolver stub",
+        "unsafeConnectorCalls": 0,
+        "case": "private.e12.test",
+        "state": "ABORTED",
+        "httpStatus": null,
+        "failureReason": null,
+        "elapsedMs": 0
+      },
+      {
+        "transport": "resolver stub",
+        "unsafeConnectorCalls": 0,
+        "case": "mixed.e12.test",
+        "state": "ABORTED",
+        "httpStatus": null,
+        "failureReason": null,
+        "elapsedMs": 0
+      },
+      {
+        "resolverThreads": 1,
+        "connectorCalls": 0,
+        "transport": "explicit resolver barrier",
+        "case": "blocked-DNS",
+        "state": "FAILED",
+        "httpStatus": null,
+        "failureReason": "TIMEOUT",
+        "elapsedMs": 1505,
+        "queuedTasksAccepted": 0,
+        "cleanupMs": 0,
+        "actualIoThreadExited": true
+      },
+      {
+        "bodyBytesRead": 0,
+        "dnsCalls": 1,
+        "socketClosed": true,
+        "connectedAddress": "93.184.216.34",
+        "connectorCalls": 1,
+        "transport": "connector stub; no live TLS",
+        "logicalHost": "public.e12.test",
+        "case": "validated-93.184.216.34",
+        "state": "SUCCEEDED",
+        "httpStatus": 200,
+        "failureReason": null,
+        "elapsedMs": 0
+      },
+      {
+        "bodyBytesRead": 0,
+        "dnsCalls": 1,
+        "socketClosed": true,
+        "connectedAddress": "2606:4700:4700:0:0:0:0:1111",
+        "connectorCalls": 1,
+        "transport": "connector stub; no live TLS",
+        "logicalHost": "public.e12.test",
+        "case": "validated-2606:4700:4700::1111",
+        "state": "SUCCEEDED",
+        "httpStatus": 200,
+        "failureReason": null,
+        "elapsedMs": 0
+      },
+      {
+        "sniHost": "public.e12.test",
+        "case": "TLS-configuration-only",
+        "liveTlsHandshakeTested": false,
+        "endpointIdentification": "HTTPS"
+      },
+      {
+        "transport": "actual local TCP",
+        "port": 4325,
+        "case": "closed-local-port",
+        "state": "FAILED",
+        "httpStatus": null,
+        "failureReason": "CONNECTION_FAILURE",
+        "elapsedMs": 2
+      },
+      {
+        "case": "canonical-create-boundary",
+        "connectorCalls": 0,
+        "canonicalUrl": "http://public.e12.test/ok",
+        "dnsCalls": 0
+      },
+      {
+        "unsafeConnectorCalls": 0,
+        "socketClosed": true,
+        "transport": "connector stub",
+        "safeConnectorCalls": 1,
+        "case": "redirect-private",
+        "state": "ABORTED",
+        "httpStatus": null,
+        "failureReason": null,
+        "elapsedMs": 0
+      },
+      {
+        "terminalResultCreated": false,
+        "case": "service-uncertainty",
+        "exceptionPropagated": true,
+        "automaticRetries": 0
+      },
+      {
+        "transport": "actual pinned local HTTP",
+        "paths": [
+          "/ok"
+        ],
+        "connectorCalls": 1,
+        "inputBytesRead": 38,
+        "allRawSocketsClosed": true,
+        "bodyBytesRead": 0,
+        "case": "local-ok",
+        "state": "SUCCEEDED",
+        "httpStatus": 200,
+        "failureReason": null,
+        "elapsedMs": 4
+      },
+      {
+        "transport": "actual pinned local HTTP",
+        "paths": [
+          "/body"
+        ],
+        "connectorCalls": 1,
+        "inputBytesRead": 42,
+        "allRawSocketsClosed": true,
+        "bodyBytesOffered": 65537,
+        "bodyBytesRead": 0,
+        "case": "local-body",
+        "state": "SUCCEEDED",
+        "httpStatus": 200,
+        "failureReason": null,
+        "elapsedMs": 0
+      },
+      {
+        "transport": "actual pinned local HTTP",
+        "paths": [
+          "/informational"
+        ],
+        "connectorCalls": 1,
+        "inputBytesRead": 82,
+        "allRawSocketsClosed": true,
+        "bodyBytesRead": 0,
+        "case": "local-informational",
+        "state": "SUCCEEDED",
+        "httpStatus": 200,
+        "failureReason": null,
+        "elapsedMs": 0
+      },
+      {
+        "transport": "actual pinned local HTTP",
+        "paths": [
+          "/trickle"
+        ],
+        "connectorCalls": 1,
+        "inputBytesRead": 4,
+        "allRawSocketsClosed": true,
+        "case": "local-trickle",
+        "state": "FAILED",
+        "httpStatus": null,
+        "failureReason": "TIMEOUT",
+        "elapsedMs": 1505
+      },
+      {
+        "transport": "actual pinned local HTTP",
+        "paths": [
+          "/redirect/0",
+          "/redirect/1",
+          "/redirect/2",
+          "/redirect/3"
+        ],
+        "connectorCalls": 4,
+        "inputBytesRead": 256,
+        "allRawSocketsClosed": true,
+        "case": "local-redirect",
+        "state": "ABORTED",
+        "httpStatus": null,
+        "failureReason": null,
+        "elapsedMs": 8
+      },
+      {
+        "transport": "actual local HTTP",
+        "delayMs": 2000,
+        "requests": 1,
+        "responseHeadersSent": false,
+        "case": "slow-headers",
+        "state": "FAILED",
+        "httpStatus": null,
+        "failureReason": "TIMEOUT",
+        "elapsedMs": 502
+      }
+    ],
+    "externalNetworkUsed": false
+  },
+  "cleanup": {
+    "listenerRead": {
+      "command": "lsof -nP -iTCP:4321-4325 -sTCP:LISTEN",
+      "exitCode": 1,
+      "stdout": "",
+      "stderr": "",
+      "noListeners": true
+    },
+    "databaseUsedByRepairGate": false,
+    "priorRootDatabaseAndPortsAudit": "index/profiles/phase-1/verification/industry-spring/E12-artifacts/root-attempt1-cleanup.json",
+    "blockedDnsThreadExited": true
+  },
+  "verificationBudget": {
+    "baselineInvocations": 1,
+    "originalAuthorMavenInvocations": 1,
+    "repairedAuthorMavenInvocations": 1,
+    "rootAcceptanceInvocationsSoFar": 0,
+    "javaMethodsExecutedAcrossAuthorGates": 39,
+    "javaMethodsPassedAcrossAuthorGates": 38,
+    "javaMethodsFailedAcrossAuthorGates": 1,
+    "baselineAndAuthorGateWallSeconds": 27.335,
+    "freshRepairTasksUsed": 1,
+    "maximumFreshRepairTasks": 2,
+    "automaticRetries": 0,
+    "parameterSweeps": 0,
+    "loadRuns": 0,
+    "e11CrashReruns": 0
+  },
+  "limitations": [
+    "Root final relevant acceptance is pending and is not replaced by the author gate.",
+    "Public IPv4/IPv6 and DNS rebinding are deterministic resolver/connector fixtures; real I/O uses isolated local HTTP fixtures only.",
+    "TLS hostname and SNI configuration were verified, not a live public TLS handshake.",
+    "No causal claim is made about the unretained historical run-return duration or cold serialization cost."
+  ],
+  "artifactManifest": {
+    "attempt1/maven-console.txt": {
+      "sha256": "357d0c5e7399a46b3e09d4b906ac9ce8781e1b91d1bf6c1ddd99deff1c85c9de",
+      "bytes": 16711
+    },
+    "attempt1/outbound.json": {
+      "sha256": "6d77322108c291c3703febebe351492c45399872104ad745862f504d7cbb9ea0",
+      "bytes": 5692
+    },
+    "attempt1/invocations.jsonl": {
+      "sha256": "5e42e3ca5f782523dc0e51dc989707d6679d1697e3391a08c170b8a9ad8e40eb",
+      "bytes": 363
+    },
+    "attempt1/surefire-reports/TEST-dev.evolution.monitor.CheckRunnerTest.xml": {
+      "sha256": "e2346f4e82da3f2b0fe394278cc87ba9865e428f484941eb432c5c76b3b129c7",
+      "bytes": 37376
+    },
+    "attempt1/surefire-reports/dev.evolution.monitor.CheckRunnerTest.txt": {
+      "sha256": "d770fb39d466eb1012eee9b3fb38ed45b6c303abfe225da50e1f2da042a2d360",
+      "bytes": 1335
+    },
+    "attempt1/surefire-reports/TEST-dev.evolution.monitor.MonitorFunctionalTest.xml": {
+      "sha256": "c76d679abe0714adf1e2051f61e77f9670e04b0cdf1edb6a110ffbddcfc08f3a",
+      "bytes": 45272
+    },
+    "attempt1/surefire-reports/dev.evolution.monitor.MonitorFunctionalTest.txt": {
+      "sha256": "6e1d1a0128d44026fcd6c87d912ff3abc1e3b45a2bb6ed4f144809439b05ddf4",
+      "bytes": 337
+    },
+    "attempt1/surefire-reports/TEST-dev.evolution.monitor.ApiErrorBoundaryTest.xml": {
+      "sha256": "1a27ecb7955b63aa4c8b510c6a70b0e150ac9113e261b04bc016218670e62dac",
+      "bytes": 34959
+    },
+    "attempt1/surefire-reports/dev.evolution.monitor.ApiErrorBoundaryTest.txt": {
+      "sha256": "6393a5d19dee7ae995f77335553863484f05d73b811025c83e525183d10223f7",
+      "bytes": 334
+    },
+    "maven-console.txt": {
+      "sha256": "142784ce9063832ef11c36a47695faf607d119829fc6ff5e36e4844d070432a7",
+      "bytes": 3977
+    },
+    "outbound.json": {
+      "sha256": "c868ea7278305bb2375618ccd881a1b15a680a4f977b48d5762a411808a02e1d",
+      "bytes": 6018
+    },
+    "surefire-reports/TEST-dev.evolution.monitor.CheckRunnerTest.xml": {
+      "sha256": "13490753e7665aaab83024a8ccbe538b38823e4cdf25d5317e122f62a671b499",
+      "bytes": 36331
+    },
+    "surefire-reports/dev.evolution.monitor.CheckRunnerTest.txt": {
+      "sha256": "f75b9f6a2f89a5733928438cf481ba6f81358a5b4969bdf5300123ea64d748e5",
+      "bytes": 325
+    },
+    "invocations.jsonl": {
+      "sha256": "2c9684de335807f88ef3773f77e31b5097d32de764b0d2a5639e490b327d7f9d",
+      "bytes": 922
+    },
+    "source-adoption.json": {
+      "sha256": "2bcb30cc000571b62f390ef125209dddd19dcdbc472ca5156a027a33a71d293a",
+      "bytes": 8883
+    }
+  },
+  "repositoryWhitespaceInspection": {
+    "command": "git diff --cached --check",
+    "exitCode": 2,
+    "disposition": "Root approved exact raw evidence bytes unchanged; no code or test-criterion exception. Source and TRACK inspection was clean.",
+    "approval": "index/profiles/phase-1/verification/industry-spring/E12-artifacts/root-raw-evidence-whitespace-approval.json",
+    "rawArtifacts": [
+      {
+        "path": "evidence/phase-1/E12/repair1/attempt1/maven-console.txt",
+        "sha256": "357d0c5e7399a46b3e09d4b906ac9ce8781e1b91d1bf6c1ddd99deff1c85c9de"
+      },
+      {
+        "path": "evidence/phase-1/E12/repair1/attempt1/surefire-reports/dev.evolution.monitor.CheckRunnerTest.txt",
+        "sha256": "d770fb39d466eb1012eee9b3fb38ed45b6c303abfe225da50e1f2da042a2d360"
+      },
+      {
+        "path": "evidence/phase-1/E12/repair1/maven-console.txt",
+        "sha256": "142784ce9063832ef11c36a47695faf607d119829fc6ff5e36e4844d070432a7"
+      }
+    ]
+  }
+}
