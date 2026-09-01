## `docs: record E11 recovery proof and verification budget`

diff --git a/TRACK.md b/TRACK.md
index 5a1ebce..881b7dc 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -326,8 +326,8 @@ creation. A late tick creates only the current due slot, without historical
 catch-up. Repeating a slot is ignored, with a PostgreSQL unique scheduled-slot
 index as the durable guard. Disabled Monitors receive no scheduled intent.
 Distinct manual and scheduled intents may overlap in the queue; no Monitor-wide
-exclusion is imposed. E10 adds competing-worker claims below. Recovery of a worker
-that stops while RUNNING is not implemented.
+exclusion is imposed. E10 adds competing-worker claims and E11 adds finite lease
+recovery below.
 
 `npm run test:worker` owns an initially worker-free API and explicitly starts one
 separate JVM after the202 checkpoint. `npm run test:e2e` runs the earlier browser
@@ -360,7 +360,7 @@ terminal result. Outbound I/O still runs outside a transaction with the existing
 timeouts. Scheduled-slot insertion also uses the existing unique slot index with
 `ON CONFLICT DO NOTHING`; repeated ticks retain one intent. V7 adds internal
 identity/owner columns without changing old outcomes or inventing historical
-request keys. No claim lease, takeover or crash recovery is added.
+request keys. E11 extends this ownership with finite lease recovery below.
 
 The E10 Java test starts two actual non-web JVMs that invoke the production
 CheckWorker once through a test-only entry point and PostgreSQL barriers; this
@@ -370,3 +370,39 @@ forwards a real acceptance, loses its response, then checks same-key/current-ID
 retransmission and a new key after acknowledgement. The frozen non-ASCII Java
 request uses a literal-byte test transport because the selected JDK HTTP client
 changes that header before transmission. Production validation is unchanged.
+
+## Worker crash recovery and shutdown (E11)
+
+A claim now commits a5000ms lease together with RUNNING and its owner. Each
+normal worker tick first closes expired RUNNING executions as ABORTED. This
+means the service could not establish an endpoint result: HTTP status, latency
+and failure reason remain null. The old ID is retained, never put back into the
+queue or automatically retried. An explicit new intention/key creates a new
+CheckRun; the old key continues to read the old terminal execution.
+
+Completion takes the same ID/Monitor/owner's row lock inside its transaction,
+then checks the lease against PostgreSQL clock_timestamp() at the UPDATE. It
+does not reuse a clock value sampled before a possible lock wait. An expired,
+recovered, already-terminal or wrong-owner completion changes no row. V8 keeps
+V1–V7 and all old outcomes/identities intact, assigning only existing RUNNING
+rows their started_at+5000ms lease. Terminal history still uses finished_at/id
+descending. Browser polling treats ABORTED as terminal and refreshes cached
+history; it never presents an invented endpoint failure.
+
+On ContextClosedEvent the worker stops admitting claims. The existing Spring
+ThreadPoolTaskScheduler stops periodic work and drains the current task before
+JPA is destroyed, with a3000ms lifecycle-phase bound in the worker profile. If
+the current operation cannot finish before shutdown, its uncommitted/unknown
+outcome remains recoverable through the lease; shutdown does not invent a result.
+The original1000ms connect/2000ms response timeouts remain unchanged.
+
+The capped crash proof is opt-in:
+`mvn -B -ntp -f backend/pom.xml -Dtest=WorkerRecoveryTest -De11.process-proof=true test`.
+It uses four actual JVM SIGKILL checkpoints and one actual scheduled-worker
+SIGTERM case. Its before-commit gate runs after the real UPDATE, inside the
+explicit transaction with autoCommit=false; killed database sessions must
+disappear and their write roll back before recovery. Default full tests do not
+silently repeat these fault scenarios. The separate browser regression seeds
+one expired RUNNING row and starts the normal worker to verify current view,
+cached history and reload with the same ABORTED ID. This seed is not a replacement
+for real-process crash evidence.
diff --git a/evidence/phase-1/E11/README.md b/evidence/phase-1/E11/README.md
new file mode 100644
index 0000000..07ce386
--- /dev/null
+++ b/evidence/phase-1/E11/README.md
@@ -0,0 +1,93 @@
+# E11 author evidence — attempt1
+
+Profile phase-1; START d51673b78cd4702584741e12f80c15af9f34cd4d;
+Spec-Revision 2ada57a71cd34fa2fae9809415c362a8bbfcdf02.
+
+`fixtures.md` was frozen before the single unchanged-product baseline. SHA256:
+38362668a1c4ae3f544cdf42f941df978a587e615cc1f40e93b86a56dd520003.
+`baseline.json` preserves the expected counterexample: after an actual worker
+SIGKILL and replacement normal worker, the identical row remained RUNNING at
+the fixed observation time, with no HTTP outcome and exactlyone fixture request.
+The baseline took13.661s and exited1. It was not rerun.
+
+## Changed behavior and actual proof
+
+The production changes add one finite5000ms lease, atomic expired-to-ABORTED
+recovery, a row-lock-then-database-clock completion fence, and a claim-admission
+stop at ContextClosedEvent. The existing Spring scheduler drains its in-flight
+task under the worker profile's3000ms lifecycle phase bound. No retry, lease
+renewal, replacement endpoint policy, executor framework or dependency was added.
+The frontend only recognizes ABORTED and invalidates its terminal history.
+
+The installed Spring6.2.19 bytecode was inspected with javap before process gates:
+ExecutorConfigurationSupport.onApplicationEvent initiates scheduler shutdown;
+its SmartLifecycle stop waits for active execution, and scheduled task cancellation
+uses cancel(false). No unsupported lifecycle method was assumed. The actual
+SIGTERM result below verifies the configured path; the3000ms deadline is distinct
+from the30-second JVM startup/observation watchdog.
+
+`recovery.json` records four distinct test-entry JVMs invoking the real
+CheckWorker once, without the scheduled loop. A test-only BeanPostProcessor adds
+the before-I/O gate and appends the before-commit advice *inside* the existing
+transaction advisor. These adapters do not perform alternate claims or writes.
+The first three real SIGKILLs produced ABORTED at exact lease expiry; outbound
+counts were0,1,1. Before expiry and on repeat recovery the update count was0.
+The fourth retained its committed SUCCEEDED200 row and one request. Completion
+with each recorded dead owner's identity changed0 rows; original request keys
+retained their original terminal identity. The stale attempt ran in the test
+coordinator using that identity, not in a process already dead.
+
+At the before-commit checkpoint the real runner had observed200, the real UPDATE
+had changed one row, the transaction was active and JDBC autoCommit was false
+(database PID45116, transaction5075). An independent read still saw RUNNING.
+After SIGKILL, the actual JVM exit and removal of its database sessions preceded
+recovery, and the uncommitted row matched its prior state. No pending autocommit
+write was released after death. All four worker exits were137 and awaited.
+
+The fifth JVM uses the **normal worker profile and scheduled loop**, with only
+a test-entry ContextClosedEvent listener for observation. One actual SIGTERM
+stopped claims, drained the held200, and exited143 in65ms. The second row stayed
+byte-identical QUEUED and /fail requests stayed0. The listener did not stop or
+signal the process. There were no cleanup SIGKILLs. The deadline-overrun branch
+was not a second process scenario; the fixed required drain case is the measured
+SIGTERM result, while finite lease recovery covers unknown unfinished work.
+
+`migration.json` adds a V7→V8 leg to the existing migration test: the seven old
+terminal rows plus one RUNNING/one QUEUED row keep every original field, key,
+owner and timestamp. Only the RUNNING row gains started_at+5000ms. Prior migration
+checksums are unchanged, and repeat migration applies0 changes. The additional
+legacy rows use IDs ending011/012, timestamp2026-08-28T00:00:01Z, keys
+e11-upgrade-running/e11-upgrade-queued and existing owner; they do not change
+any earlier migration fixture or its recorded evidence.
+
+`browser.json` is the one targeted production Chromium case. It seeds an expired
+RUNNING row for a real accepted ID (start2026-08-01T00:00:00Z, expiry+5000ms),
+then starts the actual packaged normal worker. The page first showed RUNNING
+and cached empty terminal history; recovery changed the same ID to ABORTED,
+with null HTTP/latency, one refreshed history row and the same result on reload.
+This is an explicit database setup adapter, not an additional crash checkpoint.
+
+## Invocations, budget and limitations
+
+`verification.json` contains every author baseline/gate/cleanup invocation,
+actual durations and final reviewed-source hashes. `author-output.json` stores
+the actual console and worker logs as lossless UTF-8 JSON strings with original
+SHA256 hashes; no old raw evidence was reformatted or changed.
+
+- Baseline1, expected counterexample1, unexpected failures0, repairs0/2.
+- Narrow Maven package1:2 tests passed,19.091s wall /18.070s Maven.
+- Typecheck1 passed1.690s; production build1 passed9.951s.
+- Targeted Chromium1 passed14.987s wall (case6.0s; Playwright report14.2s).
+- Explicit standalone browser schema cleanup1 passed0.172s.
+- Author process checkpoint invocations: four SIGKILLs and one in-flight SIGTERM.
+  The baseline's one SIGKILL and cleanup signals are recorded separately.
+- No author full-suite run, load run, automatic retry or parameter sweep.
+
+The root explicitly reserved one more independent invocation of each of the
+four crash checkpoints and the SIGTERM case. The process proof is opt-in via
+e11.process-proof=true, preventing accidental repetition by default full tests.
+The root's final complete affected regression and source/scope audit remain
+pending at this author candidate; they are not claimed here as already passed.
+There was no Next stream-closed message in this targeted browser output; ordinary
+NO_COLOR/FORCE_COLOR warnings are preserved. Earlier E10 observations remain in
+their original evidence without being relabelled as E11 results.
diff --git a/evidence/phase-1/E11/author-output.json b/evidence/phase-1/E11/author-output.json
new file mode 100644
index 0000000..281924c
--- /dev/null
+++ b/evidence/phase-1/E11/author-output.json
@@ -0,0 +1,54 @@
+{
+  "output/phase-1/e11/baseline-api.log": {
+    "sha256": "8dfef1f525b89040ee5f9331d0b4d15d44ccd43a3c4a40ee53780bb10aa10163",
+    "utf8": "\n  .   ____          _            __ _ _\n /\\\\ / ___'_ __ _ _(_)_ __  __ _ \\ \\ \\ \\\n( ( )\\___ | '_ | '_| | '_ \\/ _` | \\ \\ \\ \\\n \\\\/  ___)| |_)| | | | | || (_| |  ) ) ) )\n  '  |____| .__|_| |_|_| |_\\__, | / / / /\n =========|_|==============|___/=/_/_/_/\n\n :: Spring Boot ::               (v3.5.16)\n\n2026-08-28T15:12:40.689+09:00  INFO 25917 --- [monitor-api] [           main] d.evolution.monitor.MonitorApplication   : Starting MonitorApplication v0.0.1 using Java 21.0.7 with PID 25917 (/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/monitor-api-0.0.1.jar started by woopinbell in /private/tmp/web-systems-evolution-0a006589-industry-spring)\n2026-08-28T15:12:40.691+09:00  INFO 25917 --- [monitor-api] [           main] d.evolution.monitor.MonitorApplication   : No active profile set, falling back to 1 default profile: \"default\"\n2026-08-28T15:12:41.131+09:00  INFO 25917 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT mode.\n2026-08-28T15:12:41.144+09:00  INFO 25917 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 7 ms. Found 0 JPA repository interfaces.\n2026-08-28T15:12:41.481+09:00  INFO 25917 --- [monitor-api] [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port 4322 (http)\n2026-08-28T15:12:41.490+09:00  INFO 25917 --- [monitor-api] [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]\n2026-08-28T15:12:41.490+09:00  INFO 25917 --- [monitor-api] [           main] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.1.55]\n2026-08-28T15:12:41.504+09:00  INFO 25917 --- [monitor-api] [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext\n2026-08-28T15:12:41.505+09:00  INFO 25917 --- [monitor-api] [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 775 ms\n2026-08-28T15:12:41.640+09:00  INFO 25917 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...\n2026-08-28T15:12:41.700+09:00  INFO 25917 --- [monitor-api] [           main] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Added connection org.postgresql.jdbc.PgConnection@1e00bfe2\n2026-08-28T15:12:41.701+09:00  INFO 25917 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.\n2026-08-28T15:12:41.718+09:00  INFO 25917 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)\n2026-08-28T15:12:41.759+09:00  INFO 25917 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 7 migrations (execution time 00:00.025s)\n2026-08-28T15:12:41.779+09:00  INFO 25917 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema \"e04_restart\": 7\n2026-08-28T15:12:41.780+09:00  INFO 25917 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Schema \"e04_restart\" is up to date. No migration necessary.\n2026-08-28T15:12:41.831+09:00  INFO 25917 --- [monitor-api] [           main] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]\n2026-08-28T15:12:41.862+09:00  INFO 25917 --- [monitor-api] [           main] org.hibernate.Version                    : HHH000412: Hibernate ORM core version 6.6.53.Final\n2026-08-28T15:12:41.885+09:00  INFO 25917 --- [monitor-api] [           main] o.h.c.internal.RegionFactoryInitiator    : HHH000026: Second-level cache disabled\n2026-08-28T15:12:42.032+09:00  INFO 25917 --- [monitor-api] [           main] o.s.o.j.p.SpringPersistenceUnitInfo      : No LoadTimeWeaver setup: ignoring JPA class transformer\n2026-08-28T15:12:42.071+09:00  INFO 25917 --- [monitor-api] [           main] org.hibernate.orm.connections.pooling    : HHH10001005: Database info:\n\tDatabase JDBC URL [Connecting through datasource 'HikariDataSource (HikariPool-1)']\n\tDatabase driver: undefined/unknown\n\tDatabase version: 17.11\n\tAutocommit mode: undefined/unknown\n\tIsolation level: undefined/unknown\n\tMinimum pool size: undefined/unknown\n\tMaximum pool size: undefined/unknown\n2026-08-28T15:12:42.446+09:00  INFO 25917 --- [monitor-api] [           main] o.h.e.t.j.p.i.JtaPlatformInitiator       : HHH000489: No JTA platform available (set 'hibernate.transaction.jta.platform' to enable JTA platform integration)\n2026-08-28T15:12:42.469+09:00  INFO 25917 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'\n2026-08-28T15:12:42.569+09:00  INFO 25917 --- [monitor-api] [           main] r$InitializeUserDetailsManagerConfigurer : Global AuthenticationManager configured with UserDetailsService bean with name userAccounts\n2026-08-28T15:12:42.858+09:00  INFO 25917 --- [monitor-api] [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 4322 (http) with context path '/'\n2026-08-28T15:12:42.864+09:00  INFO 25917 --- [monitor-api] [           main] d.evolution.monitor.MonitorApplication   : Started MonitorApplication in 2.41 seconds (process running for 2.677)\n2026-08-28T15:12:42.907+09:00  INFO 25917 --- [monitor-api] [0.1-4322-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'\n2026-08-28T15:12:42.908+09:00  INFO 25917 --- [monitor-api] [0.1-4322-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'\n2026-08-28T15:12:42.908+09:00  INFO 25917 --- [monitor-api] [0.1-4322-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 0 ms\n2026-08-28T15:12:50.697+09:00  INFO 25917 --- [monitor-api] [ionShutdownHook] o.s.b.w.e.tomcat.GracefulShutdown        : Commencing graceful shutdown. Waiting for active requests to complete\n2026-08-28T15:12:50.698+09:00  INFO 25917 --- [monitor-api] [tomcat-shutdown] o.s.b.w.e.tomcat.GracefulShutdown        : Graceful shutdown complete\n2026-08-28T15:12:50.701+09:00  INFO 25917 --- [monitor-api] [ionShutdownHook] j.LocalContainerEntityManagerFactoryBean : Closing JPA EntityManagerFactory for persistence unit 'default'\n2026-08-28T15:12:50.703+09:00  INFO 25917 --- [monitor-api] [ionShutdownHook] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown initiated...\n2026-08-28T15:12:50.704+09:00  INFO 25917 --- [monitor-api] [ionShutdownHook] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown completed.\n"
+  },
+  "output/phase-1/e11/baseline-first-worker.log": {
+    "sha256": "a71c517c908cc9a46b126004658d6001c9b8d02c3f2dfdb54181354840b83f71",
+    "utf8": "\n  .   ____          _            __ _ _\n /\\\\ / ___'_ __ _ _(_)_ __  __ _ \\ \\ \\ \\\n( ( )\\___ | '_ | '_| | '_ \\/ _` | \\ \\ \\ \\\n \\\\/  ___)| |_)| | | | | || (_| |  ) ) ) )\n  '  |____| .__|_| |_|_| |_\\__, | / / / /\n =========|_|==============|___/=/_/_/_/\n\n :: Spring Boot ::               (v3.5.16)\n\n2026-08-28T15:12:43.912+09:00  INFO 25969 --- [monitor-api] [           main] d.evolution.monitor.MonitorApplication   : Starting MonitorApplication v0.0.1 using Java 21.0.7 with PID 25969 (/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/monitor-api-0.0.1.jar started by woopinbell in /private/tmp/web-systems-evolution-0a006589-industry-spring)\n2026-08-28T15:12:43.914+09:00  INFO 25969 --- [monitor-api] [           main] d.evolution.monitor.MonitorApplication   : The following 1 profile is active: \"worker\"\n2026-08-28T15:12:44.185+09:00  INFO 25969 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT mode.\n2026-08-28T15:12:44.207+09:00  INFO 25969 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 11 ms. Found 0 JPA repository interfaces.\n2026-08-28T15:12:44.479+09:00  INFO 25969 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...\n2026-08-28T15:12:44.550+09:00  INFO 25969 --- [monitor-api] [           main] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Added connection org.postgresql.jdbc.PgConnection@7c52fc81\n2026-08-28T15:12:44.551+09:00  INFO 25969 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.\n2026-08-28T15:12:44.567+09:00  INFO 25969 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)\n2026-08-28T15:12:44.602+09:00  INFO 25969 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 7 migrations (execution time 00:00.018s)\n2026-08-28T15:12:44.621+09:00  INFO 25969 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema \"e04_restart\": 7\n2026-08-28T15:12:44.623+09:00  INFO 25969 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Schema \"e04_restart\" is up to date. No migration necessary.\n2026-08-28T15:12:44.672+09:00  INFO 25969 --- [monitor-api] [           main] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]\n2026-08-28T15:12:44.698+09:00  INFO 25969 --- [monitor-api] [           main] org.hibernate.Version                    : HHH000412: Hibernate ORM core version 6.6.53.Final\n2026-08-28T15:12:44.714+09:00  INFO 25969 --- [monitor-api] [           main] o.h.c.internal.RegionFactoryInitiator    : HHH000026: Second-level cache disabled\n2026-08-28T15:12:44.853+09:00  INFO 25969 --- [monitor-api] [           main] o.s.o.j.p.SpringPersistenceUnitInfo      : No LoadTimeWeaver setup: ignoring JPA class transformer\n2026-08-28T15:12:44.898+09:00  INFO 25969 --- [monitor-api] [           main] org.hibernate.orm.connections.pooling    : HHH10001005: Database info:\n\tDatabase JDBC URL [Connecting through datasource 'HikariDataSource (HikariPool-1)']\n\tDatabase driver: undefined/unknown\n\tDatabase version: 17.11\n\tAutocommit mode: undefined/unknown\n\tIsolation level: undefined/unknown\n\tMinimum pool size: undefined/unknown\n\tMaximum pool size: undefined/unknown\n2026-08-28T15:12:45.254+09:00  INFO 25969 --- [monitor-api] [           main] o.h.e.t.j.p.i.JtaPlatformInitiator       : HHH000489: No JTA platform available (set 'hibernate.transaction.jta.platform' to enable JTA platform integration)\n2026-08-28T15:12:45.278+09:00  INFO 25969 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'\n2026-08-28T15:12:45.475+09:00  INFO 25969 --- [monitor-api] [           main] d.evolution.monitor.MonitorApplication   : Started MonitorApplication in 1.785 seconds (process running for 2.03)\n"
+  },
+  "output/phase-1/e11/baseline-replacement-worker.log": {
+    "sha256": "1e4d0c536c47224eae44a5b8a4d1bdd26ab63ea1f4c9f4d5cb1769406174710e",
+    "utf8": "\n  .   ____          _            __ _ _\n /\\\\ / ___'_ __ _ _(_)_ __  __ _ \\ \\ \\ \\\n( ( )\\___ | '_ | '_| | '_ \\/ _` | \\ \\ \\ \\\n \\\\/  ___)| |_)| | | | | || (_| |  ) ) ) )\n  '  |____| .__|_| |_|_| |_\\__, | / / / /\n =========|_|==============|___/=/_/_/_/\n\n :: Spring Boot ::               (v3.5.16)\n\n2026-08-28T15:12:46.163+09:00  INFO 25995 --- [monitor-api] [           main] d.evolution.monitor.MonitorApplication   : Starting MonitorApplication v0.0.1 using Java 21.0.7 with PID 25995 (/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/monitor-api-0.0.1.jar started by woopinbell in /private/tmp/web-systems-evolution-0a006589-industry-spring)\n2026-08-28T15:12:46.164+09:00  INFO 25995 --- [monitor-api] [           main] d.evolution.monitor.MonitorApplication   : The following 1 profile is active: \"worker\"\n2026-08-28T15:12:46.442+09:00  INFO 25995 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT mode.\n2026-08-28T15:12:46.462+09:00  INFO 25995 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 9 ms. Found 0 JPA repository interfaces.\n2026-08-28T15:12:46.713+09:00  INFO 25995 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...\n2026-08-28T15:12:46.781+09:00  INFO 25995 --- [monitor-api] [           main] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Added connection org.postgresql.jdbc.PgConnection@bbd4791\n2026-08-28T15:12:46.782+09:00  INFO 25995 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.\n2026-08-28T15:12:46.798+09:00  INFO 25995 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)\n2026-08-28T15:12:46.833+09:00  INFO 25995 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 7 migrations (execution time 00:00.018s)\n2026-08-28T15:12:46.851+09:00  INFO 25995 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema \"e04_restart\": 7\n2026-08-28T15:12:46.853+09:00  INFO 25995 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Schema \"e04_restart\" is up to date. No migration necessary.\n2026-08-28T15:12:46.900+09:00  INFO 25995 --- [monitor-api] [           main] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]\n2026-08-28T15:12:46.925+09:00  INFO 25995 --- [monitor-api] [           main] org.hibernate.Version                    : HHH000412: Hibernate ORM core version 6.6.53.Final\n2026-08-28T15:12:46.941+09:00  INFO 25995 --- [monitor-api] [           main] o.h.c.internal.RegionFactoryInitiator    : HHH000026: Second-level cache disabled\n2026-08-28T15:12:47.078+09:00  INFO 25995 --- [monitor-api] [           main] o.s.o.j.p.SpringPersistenceUnitInfo      : No LoadTimeWeaver setup: ignoring JPA class transformer\n2026-08-28T15:12:47.125+09:00  INFO 25995 --- [monitor-api] [           main] org.hibernate.orm.connections.pooling    : HHH10001005: Database info:\n\tDatabase JDBC URL [Connecting through datasource 'HikariDataSource (HikariPool-1)']\n\tDatabase driver: undefined/unknown\n\tDatabase version: 17.11\n\tAutocommit mode: undefined/unknown\n\tIsolation level: undefined/unknown\n\tMinimum pool size: undefined/unknown\n\tMaximum pool size: undefined/unknown\n2026-08-28T15:12:47.513+09:00  INFO 25995 --- [monitor-api] [           main] o.h.e.t.j.p.i.JtaPlatformInitiator       : HHH000489: No JTA platform available (set 'hibernate.transaction.jta.platform' to enable JTA platform integration)\n2026-08-28T15:12:47.531+09:00  INFO 25995 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'\n2026-08-28T15:12:47.698+09:00  INFO 25995 --- [monitor-api] [           main] d.evolution.monitor.MonitorApplication   : Started MonitorApplication in 1.742 seconds (process running for 1.983)\n2026-08-28T15:12:50.669+09:00  INFO 25995 --- [monitor-api] [ionShutdownHook] j.LocalContainerEntityManagerFactoryBean : Closing JPA EntityManagerFactory for persistence unit 'default'\n2026-08-28T15:12:50.671+09:00  INFO 25995 --- [monitor-api] [ionShutdownHook] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown initiated...\n2026-08-28T15:12:50.672+09:00  INFO 25995 --- [monitor-api] [ionShutdownHook] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown completed.\n"
+  },
+  "output/phase-1/e11/author-java.log": {
+    "sha256": "184f62489e9a2c2b9fd5a99355afe51a74f16515e2b13f6fa424201d6caf1a4d",
+    "utf8": "[INFO] Scanning for projects...\n[INFO] \n[INFO] ---------------------< dev.evolution:monitor-api >----------------------\n[INFO] Building monitor-api 0.0.1\n[INFO]   from pom.xml\n[INFO] --------------------------------[ jar ]---------------------------------\n[INFO] \n[INFO] --- enforcer:3.6.2:enforce (pinned-runtimes) @ monitor-api ---\n[INFO] Rule 0: org.apache.maven.enforcer.rules.version.RequireJavaVersion passed\n[INFO] Rule 1: org.apache.maven.enforcer.rules.version.RequireMavenVersion passed\n[INFO] \n[INFO] --- resources:3.3.1:resources (default-resources) @ monitor-api ---\n[INFO] Copying 2 resources from src/main/resources to target/classes\n[INFO] Copying 8 resources from src/main/resources to target/classes\n[INFO] \n[INFO] --- compiler:3.14.1:compile (default-compile) @ monitor-api ---\n[INFO] Recompiling the module because of changed source code.\n[INFO] Compiling 21 source files with javac [debug parameters release 21] to target/classes\n[INFO] \n[INFO] --- resources:3.3.1:testResources (default-testResources) @ monitor-api ---\n[INFO] skip non existing resourceDirectory /private/tmp/web-systems-evolution-0a006589-industry-spring/backend/src/test/resources\n[INFO] \n[INFO] --- compiler:3.14.1:testCompile (default-testCompile) @ monitor-api ---\n[INFO] Recompiling the module because of changed dependency.\n[INFO] Compiling 17 source files with javac [debug parameters release 21] to target/test-classes\n[INFO] \n[INFO] --- surefire:3.5.6:test (default-test) @ monitor-api ---\n[INFO] Using auto detected provider org.apache.maven.surefire.junitplatform.JUnitPlatformProvider\n[INFO] \n[INFO] -------------------------------------------------------\n[INFO]  T E S T S\n[INFO] -------------------------------------------------------\n[INFO] Running dev.evolution.monitor.WorkerRecoveryTest\n15:24:17.109 [main] INFO org.springframework.test.context.support.AnnotationConfigContextLoaderUtils -- Could not detect default configuration classes for test class [dev.evolution.monitor.WorkerRecoveryTest]: WorkerRecoveryTest does not declare any static, non-private, non-final, nested classes annotated with @Configuration.\n15:24:17.192 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper -- Found @SpringBootConfiguration dev.evolution.monitor.MonitorApplication for test class dev.evolution.monitor.WorkerRecoveryTest\n\n  .   ____          _            __ _ _\n /\\\\ / ___'_ __ _ _(_)_ __  __ _ \\ \\ \\ \\\n( ( )\\___ | '_ | '_| | '_ \\/ _` | \\ \\ \\ \\\n \\\\/  ___)| |_)| | | | | || (_| |  ) ) ) )\n  '  |____| .__|_| |_|_| |_\\__, | / / / /\n =========|_|==============|___/=/_/_/_/\n\n :: Spring Boot ::               (v3.5.16)\n\n2026-08-28T15:24:17.500+09:00  INFO 45386 --- [monitor-api] [           main] d.evolution.monitor.WorkerRecoveryTest   : Starting WorkerRecoveryTest using Java 21.0.7 with PID 45386 (started by woopinbell in /private/tmp/web-systems-evolution-0a006589-industry-spring/backend)\n2026-08-28T15:24:17.501+09:00  INFO 45386 --- [monitor-api] [           main] d.evolution.monitor.WorkerRecoveryTest   : No active profile set, falling back to 1 default profile: \"default\"\n2026-08-28T15:24:17.769+09:00  INFO 45386 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT mode.\n2026-08-28T15:24:17.783+09:00  INFO 45386 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 9 ms. Found 0 JPA repository interfaces.\n2026-08-28T15:24:18.094+09:00  INFO 45386 --- [monitor-api] [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port 4322 (http)\n2026-08-28T15:24:18.103+09:00  INFO 45386 --- [monitor-api] [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]\n2026-08-28T15:24:18.104+09:00  INFO 45386 --- [monitor-api] [           main] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.1.55]\n2026-08-28T15:24:18.125+09:00  INFO 45386 --- [monitor-api] [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext\n2026-08-28T15:24:18.126+09:00  INFO 45386 --- [monitor-api] [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 616 ms\n2026-08-28T15:24:18.254+09:00  INFO 45386 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...\n2026-08-28T15:24:18.272+09:00  INFO 45386 --- [monitor-api] [           main] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Added connection org.postgresql.jdbc.PgConnection@21c75084\n2026-08-28T15:24:18.273+09:00  INFO 45386 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.\n2026-08-28T15:24:18.292+09:00  INFO 45386 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)\n2026-08-28T15:24:18.318+09:00  INFO 45386 --- [monitor-api] [           main] o.f.c.i.s.JdbcTableSchemaHistory         : Schema history table \"e11_recovery\".\"flyway_schema_history\" does not exist yet\n2026-08-28T15:24:18.320+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 8 migrations (execution time 00:00.011s)\n2026-08-28T15:24:18.330+09:00  INFO 45386 --- [monitor-api] [           main] o.f.c.i.s.JdbcTableSchemaHistory         : Creating Schema History table \"e11_recovery\".\"flyway_schema_history\" ...\n2026-08-28T15:24:18.364+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema \"e11_recovery\": << Empty Schema >>\n2026-08-28T15:24:18.368+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema \"e11_recovery\" to version \"1 - create monitors\"\n2026-08-28T15:24:18.385+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema \"e11_recovery\" to version \"2 - create check runs\"\n2026-08-28T15:24:18.399+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema \"e11_recovery\" to version \"3 - create users\"\n2026-08-28T15:24:18.409+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema \"e11_recovery\" to version \"4 - require monitor ownership\"\n2026-08-28T15:24:18.426+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema \"e11_recovery\" to version \"5 - index check history\"\n2026-08-28T15:24:18.436+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema \"e11_recovery\" to version \"6 - queue check execution\"\n2026-08-28T15:24:18.448+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema \"e11_recovery\" to version \"7 - execution ownership and manual identity\"\n2026-08-28T15:24:18.457+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema \"e11_recovery\" to version \"8 - recover expired executions\"\n2026-08-28T15:24:18.468+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Successfully applied 8 migrations to schema \"e11_recovery\", now at version v8 (execution time 00:00.038s)\n2026-08-28T15:24:18.515+09:00  INFO 45386 --- [monitor-api] [           main] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]\n2026-08-28T15:24:18.547+09:00  INFO 45386 --- [monitor-api] [           main] org.hibernate.Version                    : HHH000412: Hibernate ORM core version 6.6.53.Final\n2026-08-28T15:24:18.563+09:00  INFO 45386 --- [monitor-api] [           main] o.h.c.internal.RegionFactoryInitiator    : HHH000026: Second-level cache disabled\n2026-08-28T15:24:18.687+09:00  INFO 45386 --- [monitor-api] [           main] o.s.o.j.p.SpringPersistenceUnitInfo      : No LoadTimeWeaver setup: ignoring JPA class transformer\n2026-08-28T15:24:18.732+09:00  INFO 45386 --- [monitor-api] [           main] org.hibernate.orm.connections.pooling    : HHH10001005: Database info:\n\tDatabase JDBC URL [Connecting through datasource 'HikariDataSource (HikariPool-1)']\n\tDatabase driver: undefined/unknown\n\tDatabase version: 17.11\n\tAutocommit mode: undefined/unknown\n\tIsolation level: undefined/unknown\n\tMinimum pool size: undefined/unknown\n\tMaximum pool size: undefined/unknown\n2026-08-28T15:24:19.105+09:00  INFO 45386 --- [monitor-api] [           main] o.h.e.t.j.p.i.JtaPlatformInitiator       : HHH000489: No JTA platform available (set 'hibernate.transaction.jta.platform' to enable JTA platform integration)\n2026-08-28T15:24:19.123+09:00  INFO 45386 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'\n2026-08-28T15:24:19.201+09:00  INFO 45386 --- [monitor-api] [           main] r$InitializeUserDetailsManagerConfigurer : Global AuthenticationManager configured with UserDetailsService bean with name userAccounts\n2026-08-28T15:24:19.503+09:00  INFO 45386 --- [monitor-api] [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 4322 (http) with context path '/'\n2026-08-28T15:24:19.508+09:00  INFO 45386 --- [monitor-api] [           main] d.evolution.monitor.WorkerRecoveryTest   : Started WorkerRecoveryTest in 2.254 seconds (process running for 2.741)\nMockito is currently self-attaching to enable the inline-mock-maker. This will no longer work in future releases of the JDK. Please add Mockito as an agent to your build as described in Mockito's documentation: https://javadoc.io/doc/org.mockito/mockito-core/latest/org.mockito/org/mockito/Mockito.html#0.3\nOpenJDK 64-Bit Server VM warning: Sharing is only supported for boot loader classes because bootstrap classpath has been appended\nWARNING: A Java agent has been loaded dynamically (/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy-agent/1.17.8/byte-buddy-agent-1.17.8.jar)\nWARNING: If a serviceability tool is in use, please run with -XX:+EnableDynamicAgentLoading to hide this warning\nWARNING: If a serviceability tool is not in use, please run with -Djdk.instrument.traceUsage for more information\nWARNING: Dynamic loading of agents will be disallowed by default in a future release\n2026-08-28T15:24:20.258+09:00  INFO 45386 --- [monitor-api] [0.1-4322-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'\n2026-08-28T15:24:20.258+09:00  INFO 45386 --- [monitor-api] [0.1-4322-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'\n2026-08-28T15:24:20.259+09:00  INFO 45386 --- [monitor-api] [0.1-4322-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 1 ms\n2026-08-28T15:24:31.078+09:00  INFO 45386 --- [monitor-api] [           main] o.s.b.w.e.tomcat.GracefulShutdown        : Commencing graceful shutdown. Waiting for active requests to complete\n2026-08-28T15:24:31.079+09:00  INFO 45386 --- [monitor-api] [tomcat-shutdown] o.s.b.w.e.tomcat.GracefulShutdown        : Graceful shutdown complete\n2026-08-28T15:24:31.082+09:00  INFO 45386 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Closing JPA EntityManagerFactory for persistence unit 'default'\n2026-08-28T15:24:31.083+09:00  INFO 45386 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown initiated...\n2026-08-28T15:24:31.084+09:00  INFO 45386 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown completed.\n[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 14.05 s -- in dev.evolution.monitor.WorkerRecoveryTest\n[INFO] Running dev.evolution.monitor.HistoryIndexMigrationTest\n2026-08-28T15:24:31.123+09:00  INFO 45386 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)\n2026-08-28T15:24:31.133+09:00  INFO 45386 --- [monitor-api] [           main] o.f.c.i.s.JdbcTableSchemaHistory         : Schema history table \"e07_index_upgrade\".\"flyway_schema_history\" does not exist yet\n2026-08-28T15:24:31.134+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 8 migrations (execution time 00:00.004s)\n2026-08-28T15:24:31.140+09:00  INFO 45386 --- [monitor-api] [           main] o.f.c.i.s.JdbcTableSchemaHistory         : Creating Schema History table \"e07_index_upgrade\".\"flyway_schema_history\" ...\n2026-08-28T15:24:31.153+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema \"e07_index_upgrade\": << Empty Schema >>\n2026-08-28T15:24:31.156+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema \"e07_index_upgrade\" to version \"1 - create monitors\"\n2026-08-28T15:24:31.168+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema \"e07_index_upgrade\" to version \"2 - create check runs\"\n2026-08-28T15:24:31.178+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema \"e07_index_upgrade\" to version \"3 - create users\"\n2026-08-28T15:24:31.187+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema \"e07_index_upgrade\" to version \"4 - require monitor ownership\"\n2026-08-28T15:24:31.196+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Successfully applied 4 migrations to schema \"e07_index_upgrade\", now at version v4 (execution time 00:00.013s)\n2026-08-28T15:24:31.316+09:00  INFO 45386 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)\n2026-08-28T15:24:31.327+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 8 migrations (execution time 00:00.005s)\n2026-08-28T15:24:31.338+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema \"e07_index_upgrade\": 4\n2026-08-28T15:24:31.341+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema \"e07_index_upgrade\" to version \"5 - index check history\"\n2026-08-28T15:24:31.351+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Successfully applied 1 migration to schema \"e07_index_upgrade\", now at version v5 (execution time 00:00.004s)\n2026-08-28T15:24:31.368+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 8 migrations (execution time 00:00.004s)\n2026-08-28T15:24:31.414+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 8 migrations (execution time 00:00.003s)\n2026-08-28T15:24:31.422+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema \"e07_index_upgrade\": 5\n2026-08-28T15:24:31.423+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Schema \"e07_index_upgrade\" is up to date. No migration necessary.\n2026-08-28T15:24:31.463+09:00  INFO 45386 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)\n2026-08-28T15:24:31.472+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 8 migrations (execution time 00:00.003s)\n2026-08-28T15:24:31.481+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema \"e07_index_upgrade\": 5\n2026-08-28T15:24:31.484+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema \"e07_index_upgrade\" to version \"6 - queue check execution\"\n2026-08-28T15:24:31.496+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Successfully applied 1 migration to schema \"e07_index_upgrade\", now at version v6 (execution time 00:00.006s)\n2026-08-28T15:24:31.510+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 8 migrations (execution time 00:00.003s)\n2026-08-28T15:24:31.520+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema \"e07_index_upgrade\": 6\n2026-08-28T15:24:31.521+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Schema \"e07_index_upgrade\" is up to date. No migration necessary.\n2026-08-28T15:24:31.561+09:00  INFO 45386 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)\n2026-08-28T15:24:31.570+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 8 migrations (execution time 00:00.004s)\n2026-08-28T15:24:31.579+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema \"e07_index_upgrade\": 6\n2026-08-28T15:24:31.581+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema \"e07_index_upgrade\" to version \"7 - execution ownership and manual identity\"\n2026-08-28T15:24:31.591+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Successfully applied 1 migration to schema \"e07_index_upgrade\", now at version v7 (execution time 00:00.004s)\n2026-08-28T15:24:31.605+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 8 migrations (execution time 00:00.003s)\n2026-08-28T15:24:31.614+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema \"e07_index_upgrade\": 7\n2026-08-28T15:24:31.615+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Schema \"e07_index_upgrade\" is up to date. No migration necessary.\n2026-08-28T15:24:31.653+09:00  INFO 45386 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)\n2026-08-28T15:24:31.662+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 8 migrations (execution time 00:00.004s)\n2026-08-28T15:24:31.673+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema \"e07_index_upgrade\": 7\n2026-08-28T15:24:31.676+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema \"e07_index_upgrade\" to version \"8 - recover expired executions\"\n2026-08-28T15:24:31.689+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Successfully applied 1 migration to schema \"e07_index_upgrade\", now at version v8 (execution time 00:00.007s)\n2026-08-28T15:24:31.706+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 8 migrations (execution time 00:00.004s)\n2026-08-28T15:24:31.716+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema \"e07_index_upgrade\": 8\n2026-08-28T15:24:31.717+09:00  INFO 45386 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Schema \"e07_index_upgrade\" is up to date. No migration necessary.\n[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.657 s -- in dev.evolution.monitor.HistoryIndexMigrationTest\n[INFO] \n[INFO] Results:\n[INFO] \n[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0\n[INFO] \n[INFO] \n[INFO] --- jar:3.4.2:jar (default-jar) @ monitor-api ---\n[INFO] Building jar: /private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/monitor-api-0.0.1.jar\n[INFO] \n[INFO] --- spring-boot:3.5.16:repackage (repackage) @ monitor-api ---\n[INFO] Replacing main artifact /private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/monitor-api-0.0.1.jar with repackaged archive, adding nested dependencies in BOOT-INF/.\n[INFO] The original artifact has been renamed to /private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/monitor-api-0.0.1.jar.original\n[INFO] ------------------------------------------------------------------------\n[INFO] BUILD SUCCESS\n[INFO] ------------------------------------------------------------------------\n[INFO] Total time:  18.070 s\n[INFO] Finished at: 2026-08-28T15:24:32+09:00\n[INFO] ------------------------------------------------------------------------\n"
+  },
+  "output/phase-1/e11/author-typecheck.log": {
+    "sha256": "2e2cf148badf4fe2a73749b44b300bdcf32682947e228fd9c498c31cf63989e0",
+    "utf8": "\n> industry-spring-monitor@0.0.1 typecheck\n> next typegen && tsc --noEmit\n\nGenerating route types...\n✓ Types generated successfully\n"
+  },
+  "output/phase-1/e11/author-build.log": {
+    "sha256": "a5cbcdcc07c52d0a277439cb914d168aa335259721d35033e2c716523fee5bb8",
+    "utf8": "\n> industry-spring-monitor@0.0.1 build\n> next build --webpack\n\n▲ Next.js 16.3.3 (webpack)\n✓ Running next.config.mjs took 6ms\n\n  Creating an optimized production build ...\n✓ Compiled successfully in 863ms\n  Running TypeScript ...\n  Finished TypeScript in 971ms ...\n  Collecting page data using 6 workers ...\n  Generating static pages using 6 workers (0/4) ...\n  Generating static pages using 6 workers (1/4) \r\n  Generating static pages using 6 workers (2/4) \r\n  Generating static pages using 6 workers (3/4) \r\n✓ Generating static pages using 6 workers (4/4) in 608ms\n  Finalizing page optimization ...\n  Collecting build traces ...\n\nRoute (app)\n┌ ○ /\n├ ○ /_not-found\n├ ○ /login\n└ ƒ /monitors\n\n\n○  (Static)   prerendered as static content\nƒ  (Dynamic)  server-rendered on demand\n\n"
+  },
+  "output/phase-1/e11/author-browser.log": {
+    "sha256": "2829ca5cb561ae44ae7c2dcaaeabe1dc159ca2dfbac10edb3fefb7060b9b9fe2",
+    "utf8": "[WebServer] (node:47962) Warning: The 'NO_COLOR' env is ignored due to the 'FORCE_COLOR' env being set.\n[WebServer] (Use `node --trace-warnings ...` to show where the warning was created)\n[WebServer] (node:47963) Warning: The 'NO_COLOR' env is ignored due to the 'FORCE_COLOR' env being set.\n[WebServer] (Use `node --trace-warnings ...` to show where the warning was created)\n[WebServer] (node:47964) Warning: The 'NO_COLOR' env is ignored due to the 'FORCE_COLOR' env being set.\n[WebServer] (Use `node --trace-warnings ...` to show where the warning was created)\n[WebServer] NOTICE:  schema \"e04_browser\" does not exist, skipping\n[WebServer] (node:48127) Warning: The 'NO_COLOR' env is ignored due to the 'FORCE_COLOR' env being set.\n[WebServer] (Use `node --trace-warnings ...` to show where the warning was created)\n\nRunning 1 test using 1 worker\n\n(node:48128) Warning: The 'NO_COLOR' env is ignored due to the 'FORCE_COLOR' env being set.\n(Use `node --trace-warnings ...` to show where the warning was created)\n  ✓  1 tests/browser/worker.spec.ts:135:5 › an expired unknown execution becomes ABORTED in the current view and cached history (6.0s)\n\n  1 passed (14.2s)\n"
+  },
+  "output/phase-1/e11/browser-worker.log": {
+    "sha256": "c2a4a4dd99b1af45e5d1bdaf8c8704c0e5fd849dd44d00a20fe982a430a0920a",
+    "utf8": "\n  .   ____          _            __ _ _\n /\\\\ / ___'_ __ _ _(_)_ __  __ _ \\ \\ \\ \\\n( ( )\\___ | '_ | '_| | '_ \\/ _` | \\ \\ \\ \\\n \\\\/  ___)| |_)| | | | | || (_| |  ) ) ) )\n  '  |____| .__|_| |_|_| |_\\__, | / / / /\n =========|_|==============|___/=/_/_/_/\n\n :: Spring Boot ::               (v3.5.16)\n\n2026-08-28T15:26:32.036+09:00  INFO 48183 --- [monitor-api] [           main] d.evolution.monitor.MonitorApplication   : Starting MonitorApplication v0.0.1 using Java 21.0.7 with PID 48183 (/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/monitor-api-0.0.1.jar started by woopinbell in /private/tmp/web-systems-evolution-0a006589-industry-spring)\n2026-08-28T15:26:32.038+09:00  INFO 48183 --- [monitor-api] [           main] d.evolution.monitor.MonitorApplication   : The following 1 profile is active: \"worker\"\n2026-08-28T15:26:32.424+09:00  INFO 48183 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT mode.\n2026-08-28T15:26:32.449+09:00  INFO 48183 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 9 ms. Found 0 JPA repository interfaces.\n2026-08-28T15:26:32.753+09:00  INFO 48183 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...\n2026-08-28T15:26:32.836+09:00  INFO 48183 --- [monitor-api] [           main] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Added connection org.postgresql.jdbc.PgConnection@7beae796\n2026-08-28T15:26:32.837+09:00  INFO 48183 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.\n2026-08-28T15:26:32.855+09:00  INFO 48183 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)\n2026-08-28T15:26:32.909+09:00  INFO 48183 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 8 migrations (execution time 00:00.029s)\n2026-08-28T15:26:32.930+09:00  INFO 48183 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema \"e04_browser\": 8\n2026-08-28T15:26:32.933+09:00  INFO 48183 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Schema \"e04_browser\" is up to date. No migration necessary.\n2026-08-28T15:26:32.998+09:00  INFO 48183 --- [monitor-api] [           main] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]\n2026-08-28T15:26:33.062+09:00  INFO 48183 --- [monitor-api] [           main] org.hibernate.Version                    : HHH000412: Hibernate ORM core version 6.6.53.Final\n2026-08-28T15:26:33.095+09:00  INFO 48183 --- [monitor-api] [           main] o.h.c.internal.RegionFactoryInitiator    : HHH000026: Second-level cache disabled\n2026-08-28T15:26:33.275+09:00  INFO 48183 --- [monitor-api] [           main] o.s.o.j.p.SpringPersistenceUnitInfo      : No LoadTimeWeaver setup: ignoring JPA class transformer\n2026-08-28T15:26:33.362+09:00  INFO 48183 --- [monitor-api] [           main] org.hibernate.orm.connections.pooling    : HHH10001005: Database info:\n\tDatabase JDBC URL [Connecting through datasource 'HikariDataSource (HikariPool-1)']\n\tDatabase driver: undefined/unknown\n\tDatabase version: 17.11\n\tAutocommit mode: undefined/unknown\n\tIsolation level: undefined/unknown\n\tMinimum pool size: undefined/unknown\n\tMaximum pool size: undefined/unknown\n2026-08-28T15:26:34.215+09:00  INFO 48183 --- [monitor-api] [           main] o.h.e.t.j.p.i.JtaPlatformInitiator       : HHH000489: No JTA platform available (set 'hibernate.transaction.jta.platform' to enable JTA platform integration)\n2026-08-28T15:26:34.242+09:00  INFO 48183 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'\n2026-08-28T15:26:34.536+09:00  INFO 48183 --- [monitor-api] [           main] d.evolution.monitor.MonitorApplication   : Started MonitorApplication in 2.861 seconds (process running for 3.475)\n2026-08-28T15:26:35.449+09:00  INFO 48183 --- [monitor-api] [ionShutdownHook] j.LocalContainerEntityManagerFactoryBean : Closing JPA EntityManagerFactory for persistence unit 'default'\n2026-08-28T15:26:35.454+09:00  INFO 48183 --- [monitor-api] [ionShutdownHook] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown initiated...\n2026-08-28T15:26:35.456+09:00  INFO 48183 --- [monitor-api] [ionShutdownHook] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown completed.\n"
+  },
+  "backend/target/e11-workers/before-io/process.log": {
+    "sha256": "0b920051e44437086ee621fdf58f2b6723dcc06d79c51e48710a5e197768e363",
+    "utf8": "2026-08-28T15:24:20.915+09:00  INFO 45428 --- [monitor-api] [           main] dev.evolution.monitor.E11WorkerProcess   : Starting E11WorkerProcess using Java 21.0.7 with PID 45428 (/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/test-classes started by woopinbell in /private/tmp/web-systems-evolution-0a006589-industry-spring/backend)\n2026-08-28T15:24:20.916+09:00  INFO 45428 --- [monitor-api] [           main] dev.evolution.monitor.E11WorkerProcess   : No active profile set, falling back to 1 default profile: \"default\"\n2026-08-28T15:24:21.162+09:00  INFO 45428 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT mode.\n2026-08-28T15:24:21.176+09:00  INFO 45428 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 10 ms. Found 0 JPA repository interfaces.\n2026-08-28T15:24:21.416+09:00  INFO 45428 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...\n2026-08-28T15:24:21.474+09:00  INFO 45428 --- [monitor-api] [           main] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Added connection org.postgresql.jdbc.PgConnection@1e236278\n2026-08-28T15:24:21.475+09:00  INFO 45428 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.\n2026-08-28T15:24:21.489+09:00  INFO 45428 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)\n2026-08-28T15:24:21.520+09:00  INFO 45428 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 8 migrations (execution time 00:00.016s)\n2026-08-28T15:24:21.538+09:00  INFO 45428 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema \"e11_recovery\": 8\n2026-08-28T15:24:21.540+09:00  INFO 45428 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Schema \"e11_recovery\" is up to date. No migration necessary.\n2026-08-28T15:24:21.586+09:00  INFO 45428 --- [monitor-api] [           main] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]\n2026-08-28T15:24:21.603+09:00  INFO 45428 --- [monitor-api] [           main] org.hibernate.Version                    : HHH000412: Hibernate ORM core version 6.6.53.Final\n2026-08-28T15:24:21.615+09:00  INFO 45428 --- [monitor-api] [           main] o.h.c.internal.RegionFactoryInitiator    : HHH000026: Second-level cache disabled\n2026-08-28T15:24:21.724+09:00  INFO 45428 --- [monitor-api] [           main] o.s.o.j.p.SpringPersistenceUnitInfo      : No LoadTimeWeaver setup: ignoring JPA class transformer\n2026-08-28T15:24:21.761+09:00  INFO 45428 --- [monitor-api] [           main] org.hibernate.orm.connections.pooling    : HHH10001005: Database info:\n\tDatabase JDBC URL [Connecting through datasource 'HikariDataSource (HikariPool-1)']\n\tDatabase driver: undefined/unknown\n\tDatabase version: 17.11\n\tAutocommit mode: undefined/unknown\n\tIsolation level: undefined/unknown\n\tMinimum pool size: undefined/unknown\n\tMaximum pool size: undefined/unknown\n2026-08-28T15:24:22.125+09:00  INFO 45428 --- [monitor-api] [           main] o.h.e.t.j.p.i.JtaPlatformInitiator       : HHH000489: No JTA platform available (set 'hibernate.transaction.jta.platform' to enable JTA platform integration)\n2026-08-28T15:24:22.163+09:00  INFO 45428 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'\n2026-08-28T15:24:22.338+09:00  INFO 45428 --- [monitor-api] [           main] dev.evolution.monitor.E11WorkerProcess   : Started E11WorkerProcess in 1.63 seconds (process running for 1.776)\n"
+  },
+  "backend/target/e11-workers/during-io/process.log": {
+    "sha256": "189c0247c9efa6ff2c971d05e182786e700fe97879e04ffec8bc27e2560a8da5",
+    "utf8": "2026-08-28T15:24:22.944+09:00  INFO 45450 --- [monitor-api] [           main] dev.evolution.monitor.E11WorkerProcess   : Starting E11WorkerProcess using Java 21.0.7 with PID 45450 (/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/test-classes started by woopinbell in /private/tmp/web-systems-evolution-0a006589-industry-spring/backend)\n2026-08-28T15:24:22.945+09:00  INFO 45450 --- [monitor-api] [           main] dev.evolution.monitor.E11WorkerProcess   : No active profile set, falling back to 1 default profile: \"default\"\n2026-08-28T15:24:23.195+09:00  INFO 45450 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT mode.\n2026-08-28T15:24:23.209+09:00  INFO 45450 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 9 ms. Found 0 JPA repository interfaces.\n2026-08-28T15:24:23.444+09:00  INFO 45450 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...\n2026-08-28T15:24:23.499+09:00  INFO 45450 --- [monitor-api] [           main] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Added connection org.postgresql.jdbc.PgConnection@2db4ad1\n2026-08-28T15:24:23.500+09:00  INFO 45450 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.\n2026-08-28T15:24:23.514+09:00  INFO 45450 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)\n2026-08-28T15:24:23.546+09:00  INFO 45450 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 8 migrations (execution time 00:00.016s)\n2026-08-28T15:24:23.564+09:00  INFO 45450 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema \"e11_recovery\": 8\n2026-08-28T15:24:23.566+09:00  INFO 45450 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Schema \"e11_recovery\" is up to date. No migration necessary.\n2026-08-28T15:24:23.612+09:00  INFO 45450 --- [monitor-api] [           main] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]\n2026-08-28T15:24:23.629+09:00  INFO 45450 --- [monitor-api] [           main] org.hibernate.Version                    : HHH000412: Hibernate ORM core version 6.6.53.Final\n2026-08-28T15:24:23.641+09:00  INFO 45450 --- [monitor-api] [           main] o.h.c.internal.RegionFactoryInitiator    : HHH000026: Second-level cache disabled\n2026-08-28T15:24:23.752+09:00  INFO 45450 --- [monitor-api] [           main] o.s.o.j.p.SpringPersistenceUnitInfo      : No LoadTimeWeaver setup: ignoring JPA class transformer\n2026-08-28T15:24:23.788+09:00  INFO 45450 --- [monitor-api] [           main] org.hibernate.orm.connections.pooling    : HHH10001005: Database info:\n\tDatabase JDBC URL [Connecting through datasource 'HikariDataSource (HikariPool-1)']\n\tDatabase driver: undefined/unknown\n\tDatabase version: 17.11\n\tAutocommit mode: undefined/unknown\n\tIsolation level: undefined/unknown\n\tMinimum pool size: undefined/unknown\n\tMaximum pool size: undefined/unknown\n2026-08-28T15:24:24.119+09:00  INFO 45450 --- [monitor-api] [           main] o.h.e.t.j.p.i.JtaPlatformInitiator       : HHH000489: No JTA platform available (set 'hibernate.transaction.jta.platform' to enable JTA platform integration)\n2026-08-28T15:24:24.142+09:00  INFO 45450 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'\n2026-08-28T15:24:24.290+09:00  INFO 45450 --- [monitor-api] [           main] dev.evolution.monitor.E11WorkerProcess   : Started E11WorkerProcess in 1.548 seconds (process running for 1.702)\n"
+  },
+  "backend/target/e11-workers/before-commit/process.log": {
+    "sha256": "80ab899690185bda060ccc3364f1cf47fbbba9794db7706c34624f3b78c905c5",
+    "utf8": "2026-08-28T15:24:24.898+09:00  INFO 45471 --- [monitor-api] [           main] dev.evolution.monitor.E11WorkerProcess   : Starting E11WorkerProcess using Java 21.0.7 with PID 45471 (/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/test-classes started by woopinbell in /private/tmp/web-systems-evolution-0a006589-industry-spring/backend)\n2026-08-28T15:24:24.900+09:00  INFO 45471 --- [monitor-api] [           main] dev.evolution.monitor.E11WorkerProcess   : No active profile set, falling back to 1 default profile: \"default\"\n2026-08-28T15:24:25.299+09:00  INFO 45471 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT mode.\n2026-08-28T15:24:25.314+09:00  INFO 45471 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 10 ms. Found 0 JPA repository interfaces.\n2026-08-28T15:24:25.684+09:00  INFO 45471 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...\n2026-08-28T15:24:25.751+09:00  INFO 45471 --- [monitor-api] [           main] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Added connection org.postgresql.jdbc.PgConnection@1e236278\n2026-08-28T15:24:25.751+09:00  INFO 45471 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.\n2026-08-28T15:24:25.766+09:00  INFO 45471 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)\n2026-08-28T15:24:25.804+09:00  INFO 45471 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 8 migrations (execution time 00:00.019s)\n2026-08-28T15:24:25.819+09:00  INFO 45471 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema \"e11_recovery\": 8\n2026-08-28T15:24:25.822+09:00  INFO 45471 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Schema \"e11_recovery\" is up to date. No migration necessary.\n2026-08-28T15:24:25.902+09:00  INFO 45471 --- [monitor-api] [           main] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]\n2026-08-28T15:24:25.925+09:00  INFO 45471 --- [monitor-api] [           main] org.hibernate.Version                    : HHH000412: Hibernate ORM core version 6.6.53.Final\n2026-08-28T15:24:25.941+09:00  INFO 45471 --- [monitor-api] [           main] o.h.c.internal.RegionFactoryInitiator    : HHH000026: Second-level cache disabled\n2026-08-28T15:24:26.064+09:00  INFO 45471 --- [monitor-api] [           main] o.s.o.j.p.SpringPersistenceUnitInfo      : No LoadTimeWeaver setup: ignoring JPA class transformer\n2026-08-28T15:24:26.106+09:00  INFO 45471 --- [monitor-api] [           main] org.hibernate.orm.connections.pooling    : HHH10001005: Database info:\n\tDatabase JDBC URL [Connecting through datasource 'HikariDataSource (HikariPool-1)']\n\tDatabase driver: undefined/unknown\n\tDatabase version: 17.11\n\tAutocommit mode: undefined/unknown\n\tIsolation level: undefined/unknown\n\tMinimum pool size: undefined/unknown\n\tMaximum pool size: undefined/unknown\n2026-08-28T15:24:26.441+09:00  INFO 45471 --- [monitor-api] [           main] o.h.e.t.j.p.i.JtaPlatformInitiator       : HHH000489: No JTA platform available (set 'hibernate.transaction.jta.platform' to enable JTA platform integration)\n2026-08-28T15:24:26.464+09:00  INFO 45471 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'\n2026-08-28T15:24:26.699+09:00  INFO 45471 --- [monitor-api] [           main] dev.evolution.monitor.E11WorkerProcess   : Started E11WorkerProcess in 2.032 seconds (process running for 2.205)\n"
+  },
+  "backend/target/e11-workers/after-commit/process.log": {
+    "sha256": "c155eedde1e3a95cf0e62f99f1fc9d277cd7cd5b6b7faec8472d6359caee85e0",
+    "utf8": "2026-08-28T15:24:27.288+09:00  INFO 45492 --- [monitor-api] [           main] dev.evolution.monitor.E11WorkerProcess   : Starting E11WorkerProcess using Java 21.0.7 with PID 45492 (/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/test-classes started by woopinbell in /private/tmp/web-systems-evolution-0a006589-industry-spring/backend)\n2026-08-28T15:24:27.289+09:00  INFO 45492 --- [monitor-api] [           main] dev.evolution.monitor.E11WorkerProcess   : No active profile set, falling back to 1 default profile: \"default\"\n2026-08-28T15:24:27.551+09:00  INFO 45492 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT mode.\n2026-08-28T15:24:27.565+09:00  INFO 45492 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 10 ms. Found 0 JPA repository interfaces.\n2026-08-28T15:24:27.792+09:00  INFO 45492 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...\n2026-08-28T15:24:27.884+09:00  INFO 45492 --- [monitor-api] [           main] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Added connection org.postgresql.jdbc.PgConnection@41a374be\n2026-08-28T15:24:27.884+09:00  INFO 45492 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.\n2026-08-28T15:24:27.902+09:00  INFO 45492 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)\n2026-08-28T15:24:27.944+09:00  INFO 45492 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 8 migrations (execution time 00:00.021s)\n2026-08-28T15:24:27.964+09:00  INFO 45492 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema \"e11_recovery\": 8\n2026-08-28T15:24:27.967+09:00  INFO 45492 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Schema \"e11_recovery\" is up to date. No migration necessary.\n2026-08-28T15:24:28.027+09:00  INFO 45492 --- [monitor-api] [           main] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]\n2026-08-28T15:24:28.049+09:00  INFO 45492 --- [monitor-api] [           main] org.hibernate.Version                    : HHH000412: Hibernate ORM core version 6.6.53.Final\n2026-08-28T15:24:28.062+09:00  INFO 45492 --- [monitor-api] [           main] o.h.c.internal.RegionFactoryInitiator    : HHH000026: Second-level cache disabled\n2026-08-28T15:24:28.187+09:00  INFO 45492 --- [monitor-api] [           main] o.s.o.j.p.SpringPersistenceUnitInfo      : No LoadTimeWeaver setup: ignoring JPA class transformer\n2026-08-28T15:24:28.225+09:00  INFO 45492 --- [monitor-api] [           main] org.hibernate.orm.connections.pooling    : HHH10001005: Database info:\n\tDatabase JDBC URL [Connecting through datasource 'HikariDataSource (HikariPool-1)']\n\tDatabase driver: undefined/unknown\n\tDatabase version: 17.11\n\tAutocommit mode: undefined/unknown\n\tIsolation level: undefined/unknown\n\tMinimum pool size: undefined/unknown\n\tMaximum pool size: undefined/unknown\n2026-08-28T15:24:28.542+09:00  INFO 45492 --- [monitor-api] [           main] o.h.e.t.j.p.i.JtaPlatformInitiator       : HHH000489: No JTA platform available (set 'hibernate.transaction.jta.platform' to enable JTA platform integration)\n2026-08-28T15:24:28.564+09:00  INFO 45492 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'\n2026-08-28T15:24:28.731+09:00  INFO 45492 --- [monitor-api] [           main] dev.evolution.monitor.E11WorkerProcess   : Started E11WorkerProcess in 1.642 seconds (process running for 1.799)\n"
+  },
+  "backend/target/e11-workers/shutdown/process.log": {
+    "sha256": "3217e6d1884039bf7a9afde44cacc7e0c602a275c0f585daf50234a2d2f9f755",
+    "utf8": "2026-08-28T15:24:29.264+09:00  INFO 45513 --- [monitor-api] [           main] dev.evolution.monitor.E11WorkerProcess   : Starting E11WorkerProcess using Java 21.0.7 with PID 45513 (/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/test-classes started by woopinbell in /private/tmp/web-systems-evolution-0a006589-industry-spring/backend)\n2026-08-28T15:24:29.265+09:00  INFO 45513 --- [monitor-api] [           main] dev.evolution.monitor.E11WorkerProcess   : The following 1 profile is active: \"worker\"\n2026-08-28T15:24:29.519+09:00  INFO 45513 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT mode.\n2026-08-28T15:24:29.533+09:00  INFO 45513 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 9 ms. Found 0 JPA repository interfaces.\n2026-08-28T15:24:29.775+09:00  INFO 45513 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...\n2026-08-28T15:24:29.833+09:00  INFO 45513 --- [monitor-api] [           main] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Added connection org.postgresql.jdbc.PgConnection@3bc4ef12\n2026-08-28T15:24:29.834+09:00  INFO 45513 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.\n2026-08-28T15:24:29.849+09:00  INFO 45513 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)\n2026-08-28T15:24:29.881+09:00  INFO 45513 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 8 migrations (execution time 00:00.016s)\n2026-08-28T15:24:29.899+09:00  INFO 45513 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema \"e11_recovery\": 8\n2026-08-28T15:24:29.901+09:00  INFO 45513 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Schema \"e11_recovery\" is up to date. No migration necessary.\n2026-08-28T15:24:29.952+09:00  INFO 45513 --- [monitor-api] [           main] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]\n2026-08-28T15:24:29.971+09:00  INFO 45513 --- [monitor-api] [           main] org.hibernate.Version                    : HHH000412: Hibernate ORM core version 6.6.53.Final\n2026-08-28T15:24:29.984+09:00  INFO 45513 --- [monitor-api] [           main] o.h.c.internal.RegionFactoryInitiator    : HHH000026: Second-level cache disabled\n2026-08-28T15:24:30.095+09:00  INFO 45513 --- [monitor-api] [           main] o.s.o.j.p.SpringPersistenceUnitInfo      : No LoadTimeWeaver setup: ignoring JPA class transformer\n2026-08-28T15:24:30.130+09:00  INFO 45513 --- [monitor-api] [           main] org.hibernate.orm.connections.pooling    : HHH10001005: Database info:\n\tDatabase JDBC URL [Connecting through datasource 'HikariDataSource (HikariPool-1)']\n\tDatabase driver: undefined/unknown\n\tDatabase version: 17.11\n\tAutocommit mode: undefined/unknown\n\tIsolation level: undefined/unknown\n\tMinimum pool size: undefined/unknown\n\tMaximum pool size: undefined/unknown\n2026-08-28T15:24:30.451+09:00  INFO 45513 --- [monitor-api] [           main] o.h.e.t.j.p.i.JtaPlatformInitiator       : HHH000489: No JTA platform available (set 'hibernate.transaction.jta.platform' to enable JTA platform integration)\n2026-08-28T15:24:30.471+09:00  INFO 45513 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'\n2026-08-28T15:24:30.635+09:00  INFO 45513 --- [monitor-api] [           main] dev.evolution.monitor.E11WorkerProcess   : Started E11WorkerProcess in 1.567 seconds (process running for 1.719)\n2026-08-28T15:24:30.964+09:00  INFO 45513 --- [monitor-api] [ionShutdownHook] j.LocalContainerEntityManagerFactoryBean : Closing JPA EntityManagerFactory for persistence unit 'default'\n2026-08-28T15:24:30.965+09:00  INFO 45513 --- [monitor-api] [ionShutdownHook] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown initiated...\n2026-08-28T15:24:30.966+09:00  INFO 45513 --- [monitor-api] [ionShutdownHook] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown completed.\n"
+  }
+}
diff --git a/evidence/phase-1/E11/browser.json b/evidence/phase-1/E11/browser.json
new file mode 100644
index 0000000..6a115a9
--- /dev/null
+++ b/evidence/phase-1/E11/browser.json
@@ -0,0 +1,23 @@
+{
+  "fixtureSha256": "38362668a1c4ae3f544cdf42f941df978a587e615cc1f40e93b86a56dd520003",
+  "setup": "explicit expired RUNNING fixture; actual normal worker recovery; no additional crash checkpoint",
+  "result": "PASS",
+  "beforeRecovery": {
+    "currentState": "RUNNING",
+    "sameAcceptedId": true,
+    "cachedTerminalHistoryRows": 0
+  },
+  "afterRecovery": {
+    "state": "ABORTED",
+    "sameId": true,
+    "httpStatus": null,
+    "latencyMs": null,
+    "terminalHistoryRows": 1,
+    "pollingInvalidatedCachedEmptyHistory": true,
+    "reloadRetainedTerminal": true,
+    "actualWorkerPid": 48183
+  },
+  "cleanup": {
+    "workerExitAwaited": true
+  }
+}
diff --git a/evidence/phase-1/E11/cleanup.json b/evidence/phase-1/E11/cleanup.json
new file mode 100644
index 0000000..ca3195c
--- /dev/null
+++ b/evidence/phase-1/E11/cleanup.json
@@ -0,0 +1,17 @@
+{
+  "ports": [
+    4321,
+    4322,
+    4323,
+    4324,
+    4325
+  ],
+  "listenerCheckExitCode": 1,
+  "listeners": "",
+  "schemaCheckExitCode": 0,
+  "schemas": [
+    "public"
+  ],
+  "ownedWorkerExits": "awaited in baseline/process/browser evidence",
+  "result": "PASS"
+}
diff --git a/evidence/phase-1/E11/migration.json b/evidence/phase-1/E11/migration.json
new file mode 100644
index 0000000..6129c2d
--- /dev/null
+++ b/evidence/phase-1/E11/migration.json
@@ -0,0 +1,3 @@
+{"result":"PASS","upgradeFrom":7,"upgradeTo":8,"migrationsExecuted":1,"repeatMigrations":0,
+ "sevenHistoricalRowsPlusRunningAndQueuedUnchanged":true,"priorMigrationChecksumsUnchanged":true,
+ "onlyExistingRunningRowReceives5000msLease":true,"requestAndWorkerIdentitiesUnchanged":true}
diff --git a/evidence/phase-1/E11/recovery.json b/evidence/phase-1/E11/recovery.json
new file mode 100644
index 0000000..68a9563
--- /dev/null
+++ b/evidence/phase-1/E11/recovery.json
@@ -0,0 +1,136 @@
+{
+  "fixtureSha256" : "38362668a1c4ae3f544cdf42f941df978a587e615cc1f40e93b86a56dd520003",
+  "checkpoints" : [ {
+    "checkpoint" : "before-io",
+    "barrier" : {
+      "claimProxyReturned" : true,
+      "checkpoint" : "before-io",
+      "transactionActive" : false
+    },
+    "databaseSessionsGoneBeforeRecovery" : true,
+    "checkId" : "f84946f1-a13e-4e08-ba79-5f4ee820a7c0",
+    "leaseMilliseconds" : 5000,
+    "terminalState" : "ABORTED",
+    "outboundRequests" : 0,
+    "preExpiryChanges" : 0,
+    "expiryChanges" : 1,
+    "repeatRecoveryChanges" : 0,
+    "deadOwnerWriteChangedRow" : false,
+    "originalIdentitySameTerminal" : true
+  }, {
+    "checkpoint" : "during-io",
+    "barrier" : {
+      "checkpoint" : "during-io",
+      "responseWithheld" : true
+    },
+    "databaseSessionsGoneBeforeRecovery" : true,
+    "checkId" : "a457db9c-57c6-46bc-8f19-07ea2e4fccd1",
+    "leaseMilliseconds" : 5000,
+    "terminalState" : "ABORTED",
+    "outboundRequests" : 1,
+    "preExpiryChanges" : 0,
+    "expiryChanges" : 1,
+    "repeatRecoveryChanges" : 0,
+    "deadOwnerWriteChangedRow" : false,
+    "originalIdentitySameTerminal" : true
+  }, {
+    "checkpoint" : "before-commit",
+    "barrier" : {
+      "checkpoint" : "before-commit",
+      "observedHttpStatus" : 200,
+      "updateChangedRow" : true,
+      "transactionActive" : true,
+      "jdbcAutoCommit" : false,
+      "databasePid" : 45116,
+      "transactionId" : 5075
+    },
+    "databaseSessionsGoneBeforeRecovery" : true,
+    "checkId" : "ffba074a-2225-4fa9-bd11-ebc8afd40f49",
+    "leaseMilliseconds" : 5000,
+    "terminalState" : "ABORTED",
+    "outboundRequests" : 1,
+    "preExpiryChanges" : 0,
+    "expiryChanges" : 1,
+    "repeatRecoveryChanges" : 0,
+    "deadOwnerWriteChangedRow" : false,
+    "originalIdentitySameTerminal" : true
+  }, {
+    "checkpoint" : "after-commit",
+    "barrier" : {
+      "checkpoint" : "after-commit",
+      "executeNextReturned" : true
+    },
+    "databaseSessionsGoneBeforeRecovery" : true,
+    "checkId" : "e5788f51-3266-4662-8d0e-d4a5f786a716",
+    "leaseMilliseconds" : 5000,
+    "terminalState" : "SUCCEEDED",
+    "outboundRequests" : 1,
+    "preExpiryChanges" : 0,
+    "expiryChanges" : 0,
+    "repeatRecoveryChanges" : 0,
+    "deadOwnerWriteChangedRow" : false,
+    "originalIdentitySameTerminal" : true
+  } ],
+  "processes" : [ {
+    "pid" : 45428,
+    "ownerId" : "7f07a4b2-d9f5-4d07-beee-bb2a5e3621a3",
+    "entry" : "test-only startup adapter; production CheckWorker once",
+    "signal" : "SIGKILL",
+    "exitCode" : 137,
+    "exitAwaited" : true
+  }, {
+    "pid" : 45450,
+    "ownerId" : "d26d9ec0-1bdb-4f14-bfb6-dda85aaf9e8b",
+    "entry" : "test-only startup adapter; production CheckWorker once",
+    "signal" : "SIGKILL",
+    "exitCode" : 137,
+    "exitAwaited" : true
+  }, {
+    "pid" : 45471,
+    "ownerId" : "3f96ce12-af59-4aa9-8f24-7bf9c794f189",
+    "entry" : "test-only startup adapter; production CheckWorker once",
+    "signal" : "SIGKILL",
+    "exitCode" : 137,
+    "exitAwaited" : true
+  }, {
+    "pid" : 45492,
+    "ownerId" : "75a9dbf8-857e-4578-9efb-09a0b711f1f2",
+    "entry" : "test-only startup adapter; production CheckWorker once",
+    "signal" : "SIGKILL",
+    "exitCode" : 137,
+    "exitAwaited" : true
+  }, {
+    "pid" : 45513,
+    "ownerId" : "baab4c83-e12a-49d1-abb8-91d13c21627d",
+    "entry" : "normal scheduled worker profile",
+    "signal" : "SIGTERM",
+    "exitCode" : 143,
+    "exitAwaited" : true
+  } ],
+  "result" : "PASS",
+  "shutdown" : {
+    "deadlineMilliseconds" : 3000,
+    "closingObservation" : {
+      "claimsStopped" : true,
+      "shutdownPhaseTimeout" : "3s"
+    },
+    "httpStatus" : 200,
+    "secondRowUnchangedQueued" : true,
+    "elapsedMilliseconds" : 65,
+    "secondOutboundRequests" : 0,
+    "inFlightState" : "SUCCEEDED",
+    "signalCount" : 1,
+    "actualExitAwaited" : true
+  },
+  "identityAndHistory" : {
+    "abortedHistoryRows" : 3,
+    "terminalHistoryRows" : 5,
+    "newIntentNewQueuedId" : true,
+    "originalTerminalUnchanged" : true
+  },
+  "cleanup" : {
+    "allChildExitsAwaited" : true,
+    "cleanupSigkills" : 0,
+    "fixtureReleaseSettled" : true
+  }
+}
diff --git a/evidence/phase-1/E11/verification.json b/evidence/phase-1/E11/verification.json
new file mode 100644
index 0000000..858c1ca
--- /dev/null
+++ b/evidence/phase-1/E11/verification.json
@@ -0,0 +1,126 @@
+{
+  "thread": "E11",
+  "profile": "phase-1",
+  "attempt": 1,
+  "start": "d51673b78cd4702584741e12f80c15af9f34cd4d",
+  "specRevision": "2ada57a71cd34fa2fae9809415c362a8bbfcdf02",
+  "fixtureSha256": "38362668a1c4ae3f544cdf42f941df978a587e615cc1f40e93b86a56dd520003",
+  "runtimeLauncher": "fnm exec --using 24.19.0",
+  "invocations": [
+    {
+      "command": "node scripts/e11-baseline.mjs",
+      "startedAt": "2026-08-28T06:12:37.228Z",
+      "elapsedSeconds": 13.661,
+      "exitCode": 1
+    },
+    {
+      "command": "fnm exec --using 24.19.0 mvn -B -ntp -f backend/pom.xml -Dtest=WorkerRecoveryTest,HistoryIndexMigrationTest -De11.process-proof=true package",
+      "startedAt": "2026-08-28T06:24:13.565Z",
+      "elapsedSeconds": 19.091,
+      "exitCode": 0,
+      "signal": null,
+      "log": "output/phase-1/e11/author-java.log"
+    },
+    {
+      "command": "fnm exec --using 24.19.0 npm run typecheck",
+      "startedAt": "2026-08-28T06:25:40.341Z",
+      "elapsedSeconds": 1.69,
+      "exitCode": 0,
+      "signal": null,
+      "log": "output/phase-1/e11/author-typecheck.log"
+    },
+    {
+      "command": "fnm exec --using 24.19.0 npm run build",
+      "startedAt": "2026-08-28T06:25:42.032Z",
+      "elapsedSeconds": 9.951,
+      "exitCode": 0,
+      "signal": null,
+      "log": "output/phase-1/e11/author-build.log"
+    },
+    {
+      "command": "E09_MANUAL_WORKER=1 fnm exec --using 24.19.0 npm exec -- playwright test tests/browser/worker.spec.ts --grep \"an expired unknown execution\"",
+      "startedAt": "2026-08-28T06:26:20.606Z",
+      "elapsedSeconds": 14.987,
+      "exitCode": 0,
+      "signal": null,
+      "log": "output/phase-1/e11/author-browser.log"
+    },
+    {
+      "command": "node scripts/database.mjs drop e04_browser",
+      "purpose": "owned standalone browser cleanup",
+      "startedAt": "2026-08-28T06:26:35.596Z",
+      "elapsedSeconds": 0.172,
+      "exitCode": 0
+    }
+  ],
+  "java": [
+    {
+      "name": "WorkerRecoveryTest",
+      "tests": 1,
+      "failures": 0,
+      "errors": 0,
+      "skipped": 0,
+      "time": 14.051
+    },
+    {
+      "name": "HistoryIndexMigrationTest",
+      "tests": 1,
+      "failures": 0,
+      "errors": 0,
+      "skipped": 0,
+      "time": 0.657
+    }
+  ],
+  "browserStats": {
+    "startTime": "2026-08-28T06:26:21.343Z",
+    "duration": 14216.259,
+    "expected": 1,
+    "skipped": 0,
+    "unexpected": 0,
+    "flaky": 0
+  },
+  "reviewedSourceSha256": {
+    "app/monitors/api.ts": "78796b2ea67b4d67188dbc4e401774984cb08140ab711450d288e173771d24e7",
+    "app/monitors/use-monitor-state.ts": "5396e2c39b86ef9801424ff42bf1abc9ef4f628fcd97b9259984bc0465cad24c",
+    "backend/src/main/java/dev/evolution/monitor/CheckQueue.java": "b29c5a373c43a6e073fd535c4853c3130f59cbae608d99a3166f91a0a4b05c40",
+    "backend/src/main/java/dev/evolution/monitor/CheckRunEntity.java": "d960343ecbb9d95a0e45c5427792f6d596d3f709e49a7dea4b4a557ad7ac0fae",
+    "backend/src/main/java/dev/evolution/monitor/CheckWorker.java": "0f4e628a30acb77a910ed7b7d81abe28e82d0b7e747c2462796811dddcca65d3",
+    "backend/src/main/java/dev/evolution/monitor/WorkerProcess.java": "b315a44ded29b5cd6c76177036b180c4e87db4260f98c53c13f5033a73d8e674",
+    "backend/src/main/resources/application-worker.properties": "2e081d57d2684b70b61f0d55a5d79020290d1df662ad610164cd4797432876f3",
+    "backend/src/main/resources/db/migration/V8__recover_expired_executions.sql": "52881acc201da4096c737bd7b25c8a03a97cee51f0464415e79633fb07a09240",
+    "backend/src/test/java/dev/evolution/monitor/E11WorkerProcess.java": "fbbca9762e2cf8cea6a0a5a5ad843b855013b9349f6414694e91f18646d9274d",
+    "backend/src/test/java/dev/evolution/monitor/WorkerRecoveryTest.java": "b5300f5767d14ba787f2b1f3d63ccd2125561a628c68421958f68b0fb8f96cd1",
+    "backend/src/test/java/dev/evolution/monitor/HistoryIndexMigrationTest.java": "2dc9e3b94e5fcec5709db586392a7580b091c1fda16a873370eadd7840fcd532",
+    "backend/src/test/java/dev/evolution/monitor/TestDatabase.java": "b69f5b6ca19be714ab8d359705d10fd96b41cd839071a0a802f28d276dd408e0",
+    "tests/browser/worker.spec.ts": "db4311c34e29d6f9915b7da204ba75fec440a51a21f27742615b0ab95ca9aa58"
+  },
+  "packageArtifactSha256": "5fb3a43c8b824667520cc1b00ffb9d32384ea66c02420417bbe98feb412357eb",
+  "budget": {
+    "baseline": 1,
+    "expectedBaselineCounterexamples": 1,
+    "unexpectedFailures": 0,
+    "mavenPackageGate": 1,
+    "javaTests": 2,
+    "typecheck": 1,
+    "productionBuild": 1,
+    "targetedChromium": 1,
+    "authorFullVerification": 0,
+    "authorCrashSigkills": 4,
+    "authorInflightSigterm": 1,
+    "baselineSigkill": 1,
+    "rootCrashSigkillsReserved": 4,
+    "rootInflightSigtermReserved": 1,
+    "repairsUsed": 0,
+    "repairLimit": 2,
+    "loadRuns": 0,
+    "automaticRetries": 0,
+    "parameterSweeps": 0,
+    "elapsedSecondsIncludingBaselineAndExplicitBrowserCleanup": 59.552
+  },
+  "observations": {
+    "nextStreamClosedMessages": 0,
+    "colorEnvironmentWarning": true
+  },
+  "cleanup": "PASS",
+  "rootIndependentFinalVerification": "pending"
+}
