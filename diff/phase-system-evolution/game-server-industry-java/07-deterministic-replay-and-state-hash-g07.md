# Replay와 State Hash

## `feat: capture bounded replay and canonical tick hashes`

diff --git a/TRACK.md b/TRACK.md
index 80f9867..a2bf87a 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -1,6 +1,6 @@
-# Java arena — through G06
+# Java arena — through G07
 
-Current thread: G06 (G01–G05 regressions retained). Phase: phase-1. Profile: realtime-core. Spec revision: `c1d62196ab76b55652f5d75a67514f8c6d8210ce`.
+Current thread: G07 (G01–G06 regressions retained). Phase: phase-1. Profile: realtime-core. Spec revision: `c1d62196ab76b55652f5d75a67514f8c6d8210ce`.
 G01–G05 retain their original spec trailers and verification. The user-authorized spec/profile transition changes procedure only; the game contract is unchanged. Scope remains G01–G14, with no G15+, external infrastructure or push.
 
 ## Frozen versions
@@ -26,11 +26,13 @@ The wrapper uses the locally installed Temurin path when JAVA_HOME is unset. On
 ./track scenario-run /absolute/path/to/G04.json /absolute/path/to/clock-evidence.json
 ./track scenario-run /absolute/path/to/G05.json /absolute/path/to/sequence-evidence.json
 ./track scenario-run /absolute/path/to/G06.json /absolute/path/to/authority-evidence.json
+./track scenario-run /absolute/path/to/G07.json /absolute/path/to/L1.json
+./track scenario-run /absolute/path/to/G07.json /absolute/path/to/V.json --variant rejected-removed
 ./track replay-verify /absolute/path/to/replay /absolute/path/to/evidence
 ./track server config/server.json
 ```
 
-`replay-verify` exits 2 with NOT ACTIVATED until G07. Build does not execute tests or scenarios. Unit/integration tasks re-execute tests every invocation; compilation is reused after the explicit build. Reports are in `build/test-results/{test,integrationTest}` and `build/reports/tests/{test,integrationTest}`. Netty leak detection is PARANOID for all test JVMs. G01 verification remains in `evidence/G01-verification.md`. G02 commands, exits, durations and resource observations are recorded in `evidence/G02-verification.md` and `evidence/G02-observations.json`; raw stdout/XML copies are retained under ignored `evidence/runs/g02-initial/`.
+Build does not execute tests or scenarios. Unit/integration tasks re-execute tests every invocation; compilation is reused after the explicit build. Reports are in `build/test-results/{test,integrationTest}` and `build/reports/tests/{test,integrationTest}`. Netty leak detection is PARANOID for all test/tool JVMs. `server` uses only the installed production artifact. Scenario/replay tools use the separately compiled test runtime classpath written by the build, so G07's fixed-ID bootstrap cannot enter the shipping JAR. Each scenario/replay invocation launches one fresh JVM and executes one requested pass. G01 verification remains in `evidence/G01-verification.md`; earlier raw logs remain under ignored `evidence/runs/`.
 
 ## Ownership and bounds
 
@@ -98,8 +100,20 @@ The existing movement, clamp, TAG range/cooldown, connectivity and ASCII actor-o
 
 After the existing parser schema and transport identity checks, an owner-confined counter permits four INPUT attempts for each active Player between simulation ticks, including attempts rejected by direction or sequence admission. The fifth returns `INPUT_RATE_EXCEEDED` before those validations, without reserving its sequence. Counters saturate at four and reset on the next executed tick. Inactive/foreign identities do not consume another actor's allowance. Existing parser/lifecycle precedence and the independent lower-level 64-entry pending bound are preserved.
 
-The frozen 18-attempt/221-tick TCP scenario records real admission replies, TAG failures and successes, before/after state and all resulting ticks. Pure fixtures cover the prescribed clamps, range boundaries, LEFT/self/other-Room targets, shared-victim ordering, four invalid attempts and the retained 64-entry bound. No server ID injection or multi-room scheduling was added. See `evidence/G06-command-ledger.jsonl` and `evidence/G06-verification.md`. Replay/hash, UDP and later features remain inactive.
+The frozen 18-attempt/221-tick TCP scenario records real admission replies, TAG failures and successes, before/after state and all resulting ticks. Pure fixtures cover the prescribed clamps, range boundaries, LEFT/self/other-Room targets, shared-victim ordering, four invalid attempts and the retained 64-entry bound. No server ID injection or multi-room scheduling was added. See `evidence/G06-command-ledger.jsonl` and `evidence/G06-verification.md`.
 
-G01 initial budget: build/compile <=8, unit suites <=4, integration suites <=2, canonical scenario <=1; network-fault and load runs exactly zero. Main has its own separately frozen one-build/one-unit/one-integration/one-scenario verification budget. No test sleep, microbenchmark, fuzzing, replay, UDP, reconnect, many-room or distributed implementation is included.
+## G07 canonical state and replay
+
+The Room owner computes the exact GAME_CONTRACT canonical record after each executed tick, with ASCII Player order, integer decimal fields, CONNECTED/LEFT, owner_epoch0 and final LF, then hashes UTF-8 bytes with SHA-256. This does not add snapshot sequencing or replication. The independent replay log captures initial slot/spawn mapping, every accepted input at its original admission boundary (including superseded inputs), actual LEFT events once, and each resulting record/hash. Export returns an immutable byte copy through `ArenaServer.replayArtifact()` before shutdown.
+
+The JSON-lines artifact has HEADER, INPUT, LEFT and TICK records in owner order. INPUT records retain `before_tick` separately from `target_tick`. TICK records contain `tick`, `expected_hash` and `canonical_record`. Artifact storage is bounded to 4,194,304 bytes. If an append would exceed the bound, a latch makes export fail explicitly as incomplete; it neither changes gameplay nor stops per-tick hashing. Room close releases the backing storage, and server cleanup reports retained replay bytes0. No new wire error or Room termination policy is introduced.
+
+`ReplayFixture`, `ReplayScenario`, `ReplayVerifier` and `ReplayTool` exist only under `src/test/java`. Four actual TCP HELLO sessions are bound on the owner; three initial Player records are prepared before the existing fourth `Room.join` performs one unchanged start-condition evaluation. This is labeled fixture initialization, not evidence of public four-player joins. Normal two-player start and RUNNING join rejection are unchanged. The shipping JAR and an isolated production classloader are checked to exclude all fixture/tool classes.
+
+`scenario-run G07.json result.json` executes one200-tick live pass and writes `result.replay.jsonl` beside the result, refusing to overwrite an existing artifact. `--variant rejected-removed` removes only the manifest's four rejected INPUT attempts. It retains the real LEAVE and charlie's accepted TAG failure. `replay-verify artifact result.json` initializes the recorded roster in the test runtime, applies accepted events at their recorded admission boundaries, and uses the same Room ticks/hash code. It stops at the first mismatching expected hash/record, writes the actual divergent record and returns nonzero. Replay input bytes are bounded before parsing.
+
+G07 commands explicitly reserve L1/L2/O1/O2/V at200 ticks each, plus a separate38-tick negative invocation after changing only L1's expected hash37 in a disposable copy. No wrapper hides another pass. The reproduction-only200-tick test is archived and absent from the final suite; the three permanent G07 unit tests execute zero ticks and cover record format, immutable ownership/overflow/close release and production fixture exclusion. Prior stage regressions remain unchanged. Exact worker/root-ready argv and artifact paths are in `evidence/G07-command-ledger.jsonl`; outcomes are in `evidence/G07-verification.md`. G08+ features are inactive.
+
+G01 initial budget was build/compile <=8, unit suites <=4, integration suites <=2, canonical scenario <=1. Later Threads use their frozen active plans, including G07's explicit five-pass and negative-replay budget. Network-fault and load runs remain zero. No test sleep, microbenchmark, fuzzing, UDP, reconnect, many-room or distributed implementation is included.
 
 JVM concurrency evidence uses owner-confinement assertions plus real cross-thread Netty handoff, actual thread joins and shutdown assertions. No JVM race detector is installed; no sanitizer result is claimed.
diff --git a/build.gradle b/build.gradle
index aececfa..ccc9e41 100644
--- a/build.gradle
+++ b/build.gradle
@@ -31,7 +31,14 @@ tasks.withType(Test).configureEach {
     outputs.upToDateWhen { false }
     testLogging { events 'passed', 'skipped', 'failed'; exceptionFormat 'full' }
 }
-test { exclude '**/*IntegrationTest.class' }
+test {
+    exclude '**/*IntegrationTest.class'
+    dependsOn tasks.jar
+    systemProperty 'arena.productionJar', tasks.jar.archiveFile.get().asFile.absolutePath
+}
+tasks.register('writeTestRuntimeClasspath') {
+    doLast { layout.buildDirectory.file('test-runtime-classpath.txt').get().asFile.text = sourceSets.test.runtimeClasspath.asPath }
+}
 tasks.register('integrationTest', Test) {
     testClassesDirs = sourceSets.test.output.classesDirs
     classpath = sourceSets.test.runtimeClasspath
diff --git a/evidence/G07-command-ledger.jsonl b/evidence/G07-command-ledger.jsonl
new file mode 100644
index 0000000..ed231b8
--- /dev/null
+++ b/evidence/G07-command-ledger.jsonl
@@ -0,0 +1,20 @@
+{"kind":"resolved_before_execution","pass":"baseline","category":"unit-reproduction","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","unit-test","--tests","arena.G07BaselineTest"],"environment":{"ARENA_G07_SCENARIO":"/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G07.json","ARENA_G07_EVIDENCE":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/reproduce-unit/result.json"},"reserved_ticks":200,"resolved_at":"2026-08-28T03:49:47.305510+00:00","production_hash_manifest":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/reproduce-unit/production-hashes-before.json"}
+{"kind":"resolved_before_execution","pass":"L1","category":"positive-live","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","scenario-run","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G07.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/L1/result.json"],"environment":{},"reserved_ticks":200,"resolved_at":"2026-08-28T03:49:47.305542+00:00"}
+{"kind":"resolved_before_execution","pass":"L2","category":"positive-live-fresh-process","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","scenario-run","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G07.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/L2/result.json"],"environment":{},"reserved_ticks":200,"resolved_at":"2026-08-28T03:49:47.305552+00:00"}
+{"kind":"resolved_before_execution","pass":"O1","category":"positive-offline","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/L1/result.replay.jsonl","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/O1/result.json"],"environment":{},"reserved_ticks":200,"resolved_at":"2026-08-28T03:49:47.305558+00:00"}
+{"kind":"resolved_before_execution","pass":"O2","category":"positive-offline-fresh-process","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/L1/result.replay.jsonl","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/O2/result.json"],"environment":{},"reserved_ticks":200,"resolved_at":"2026-08-28T03:49:47.305565+00:00"}
+{"kind":"resolved_before_execution","pass":"V","category":"positive-live-rejected-removed","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","scenario-run","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G07.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/V/result.json","--variant","rejected-removed"],"environment":{},"reserved_ticks":200,"resolved_at":"2026-08-28T03:49:47.305571+00:00"}
+{"kind":"resolved_before_execution","pass":"negative","category":"unit-negative","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/negative/hash37.replay.jsonl","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/negative/result.json"],"environment":{},"reserved_ticks":38,"resolved_at":"2026-08-28T03:49:47.305577+00:00"}
+{"kind":"executed","pass":"baseline","category":"unit-reproduction","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","unit-test","--tests","arena.G07BaselineTest"],"environment":{"ARENA_G07_SCENARIO":"/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G07.json","ARENA_G07_EVIDENCE":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/reproduce-unit/result.json"},"started_at":"2026-08-28T03:52:55.580074+00:00","duration_seconds":6.207,"command_process_id":53638,"exit_code":1,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/reproduce-unit/stdout.log","xml":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/reproduce-unit/xml","simulation_process_id":53656,"executed_ticks":200,"result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/reproduce-unit/result.json"}
+{"kind":"resolved_before_execution","category":"build","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","build"],"environment":{},"resolved_at":"2026-08-28T04:03:55.479638+00:00"}
+{"kind":"executed","category":"build","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","build"],"environment":{},"started_at":"2026-08-28T04:03:55.480062+00:00","duration_seconds":6.957,"command_process_id":56912,"exit_code":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/verify-build/stdout.log"}
+{"kind":"resolved_before_execution","category":"unit","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","unit-test"],"environment":{},"resolved_at":"2026-08-28T04:04:02.438400+00:00"}
+{"kind":"executed","category":"unit","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","unit-test"],"environment":{},"started_at":"2026-08-28T04:04:02.438547+00:00","duration_seconds":4.759,"command_process_id":56962,"exit_code":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/verify-unit/stdout.log","xml":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/verify-unit/xml"}
+{"kind":"resolved_before_execution","category":"integration","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","integration-test"],"environment":{},"resolved_at":"2026-08-28T04:04:07.200206+00:00"}
+{"kind":"executed","category":"integration","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","integration-test"],"environment":{},"started_at":"2026-08-28T04:04:07.200277+00:00","duration_seconds":6.461,"command_process_id":56999,"exit_code":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/verify-integration/stdout.log","xml":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/verify-integration/xml"}
+{"kind":"executed","pass":"L1","category":"positive-live","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","scenario-run","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G07.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/L1/result.json"],"environment":{},"started_at":"2026-08-28T04:06:22.569442+00:00","duration_seconds":1.295,"command_process_id":57924,"exit_code":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/L1/stdout.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/L1/result.json","simulation_process_id":57924,"executed_ticks":200,"artifact_sha256":"4a951e5fb60ab3087556f810918eb8ab51792cf321439f04f11243b66ff54ca1","artifact_bytes":122682,"process_terminated":true}
+{"kind":"executed","pass":"L2","category":"positive-live-fresh-process","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","scenario-run","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G07.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/L2/result.json"],"environment":{},"started_at":"2026-08-28T04:06:23.871133+00:00","duration_seconds":1.222,"command_process_id":57929,"exit_code":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/L2/stdout.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/L2/result.json","simulation_process_id":57929,"executed_ticks":200,"artifact_sha256":"4a951e5fb60ab3087556f810918eb8ab51792cf321439f04f11243b66ff54ca1","artifact_bytes":122682,"process_terminated":true}
+{"kind":"executed","pass":"O1","category":"positive-offline","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/L1/result.replay.jsonl","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/O1/result.json"],"environment":{},"started_at":"2026-08-28T04:06:25.099936+00:00","duration_seconds":0.287,"command_process_id":57934,"exit_code":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/O1/stdout.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/O1/result.json","simulation_process_id":57934,"executed_ticks":200,"artifact_sha256":"4a951e5fb60ab3087556f810918eb8ab51792cf321439f04f11243b66ff54ca1","artifact_bytes":122682,"process_terminated":true}
+{"kind":"executed","pass":"O2","category":"positive-offline-fresh-process","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/L1/result.replay.jsonl","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/O2/result.json"],"environment":{},"started_at":"2026-08-28T04:06:25.387757+00:00","duration_seconds":0.249,"command_process_id":57939,"exit_code":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/O2/stdout.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/O2/result.json","simulation_process_id":57939,"executed_ticks":200,"artifact_sha256":"4a951e5fb60ab3087556f810918eb8ab51792cf321439f04f11243b66ff54ca1","artifact_bytes":122682,"process_terminated":true}
+{"kind":"executed","pass":"V","category":"positive-live-rejected-removed","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","scenario-run","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G07.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/V/result.json","--variant","rejected-removed"],"environment":{},"started_at":"2026-08-28T04:06:25.638063+00:00","duration_seconds":1.199,"command_process_id":57944,"exit_code":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/V/stdout.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/V/result.json","simulation_process_id":57944,"executed_ticks":200,"artifact_sha256":"4a951e5fb60ab3087556f810918eb8ab51792cf321439f04f11243b66ff54ca1","artifact_bytes":122682,"process_terminated":true}
+{"kind":"executed","pass":"negative","category":"unit-negative","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/negative/hash37.replay.jsonl","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/negative/result.json"],"environment":{},"started_at":"2026-08-28T04:06:26.848167+00:00","duration_seconds":0.238,"command_process_id":57950,"exit_code":1,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/negative/stdout.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/negative/result.json","mutation":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g07-initial/negative/mutation.json","simulation_process_id":57950,"executed_ticks":38,"first_divergent_tick":37,"process_terminated":true}
diff --git a/evidence/G07-verification.md b/evidence/G07-verification.md
new file mode 100644
index 0000000..984f307
--- /dev/null
+++ b/evidence/G07-verification.md
@@ -0,0 +1,29 @@
+# G07 — initial attempt
+
+START `eab6e6885f444b102adc58267773c7098147a651`; phase `phase-1`; profile `realtime-core`; spec `c1d62196ab76b55652f5d75a67514f8c6d8210ce`.
+Fixture SHA-256 `d8c80202888d2e3dbeb1a7c03c14a73e84899d7cc65546e1c58a8b3102c70796` is unchanged. Exact resolved/executed argv, process IDs, time, exit and raw paths are in `G07-command-ledger.jsonl`.
+
+**Reproduction:** all13 prior production sources matched START hashes. The test-only owner bootstrap bound four real TCP HELLO sessions before one unchanged Room start evaluation. The resolved `./track unit-test --tests arena.G07BaselineTest` executed exactly200 ticks on unchanged G06, exit1 (6.207s), one assertion failure. All17 admission codes, future admission98/target100, superseded alpha100, actual LEAVE175, TAG results and final state passed (NOT_REPRODUCED gameplay). Hashes/records were200 observed nulls and no artifact existed. Baseline sources, hashes,200 raw states, stdout and XML remain in `runs/g07-initial/reproduce-unit/`. Main was notified before production edits. The reproduction-only test was then removed; permanent G07 unit tests execute zero ticks, with no stateful skip.
+
+**Implementation:** the owner records exact canonical UTF-8/LF bytes and SHA-256 after each tick; journals initial mapping, accepted INPUTs at admission boundaries, real LEFT events once and resulting records/hashes. Fixed-ID live/offline setup and tool entry points are exclusively test-source classes. Normal two-player start/RUNNING join rejection, old test files, parser/clock core and dependency versions/locks are unchanged. Build changes only expose a separate test-tool classpath and the production-JAR exclusion test. Snapshot replication and G08+ remain inactive.
+
+**Verification:** clean build exit0 (6.957s); complete unit exit0 **41 tests** (4.759s); integration exit0 **4 tests** (6.461s); no skipped tests. Zero-tick format/capture tests observed immutable copies, overflow latched at218 stored bytes without appending the oversized record, explicit incomplete-export error and close bytes0. The installed production JAR and isolated production loader exclude ReplayFixture/ReplayScenario/ReplayVerifier/ReplayTool and the retired baseline test; the G07 fixture resource is absent from the shipping JAR. Its observed SHA-256 was `bf1cad272745f7a9a42ef685ab5689d821fd0fa2a0674938d9244abaeaec719b`.
+
+| Pass | PID | Ticks | Exit | Seconds |
+|---|---:|---:|---:|---:|
+| L1 | 57924 | 200 | 0 | 1.295 |
+| L2, fresh JVM | 57929 | 200 | 0 | 1.222 |
+| O1, immutable L1 | 57934 | 200 | 0 | 0.287 |
+| O2, fresh JVM/same L1 | 57939 | 200 | 0 | 0.249 |
+| V, only4 rejected INPUTs removed | 57944 | 200 | 0 | 1.199 |
+| Negative, only expected hash37 changed | 57950 | 38 | 1 expected | 0.238 |
+
+All five200-element hash arrays **and canonical-record arrays match**. The three live artifact byte sequences also match: **122682 bytes** (limit4194304), SHA-256 `4a951e5fb60ab3087556f810918eb8ab51792cf321439f04f11243b66ff54ca1`. Artifact content is one HEADER,13 INPUT, one LEFT and200 TICK records. Bravo seq2 records `before_tick98,target_tick100`; both alpha seq10/11 at100 remain; charlie's accepted seq3/TAG at199 remains despite ACTION_REJECTED. Canonical rejects are INPUT_LATE at10, INPUT_TOO_EARLY at11, SEQUENCE_CONFLICT at12 and INPUT_STALE at101. LEAVE175 appears once.
+
+Final tick199: Room RUNNING, epoch0; alpha/bravo/charlie `(50000,50000)` with NORTH/SOUTH/EAST, scores1/0/0, last sequences12/3/3; delta `(60400,50000)`, STOP, score0, last sequence3, LEFT. Alpha last_tag_tick199; others−20. Final hash: `e3c9b271e14da9708c26ea468b8c7a5293798a984f3880f77386f1f5e9799bf6`.
+
+Negative replay returned failure at **first divergent tick37**, executed **38 ticks**, and emitted the actual canonical record/hash matching L1 tick37. Only that expected-hash field changed; the mutation manifest and immutable copy are in `runs/g07-initial/negative/`. Original L1 bytes remain unchanged/read-only. Every child process terminated. Live cleanup reports zero channels, connections, pending writes/mailbox, parser buffers/bytes, recorder bytes, clock backlog and owned threads, with terminated owner/event loops and closed clients. Live high-water maxima: pending2, mailbox2, outbound1, parser bytes/capacity227/256. Offline cleanup reports CLOSED, pending0 and recorder bytes0, without creating a server.
+
+Raw per-pass result/hash/record paths: `runs/g07-initial/{L1,L2,O1,O2,V}/result.json`; L1 artifact `runs/g07-initial/L1/result.replay.jsonl`; negative `runs/g07-initial/negative/{result.json,mutation.json}`; aggregate equality and resource evidence `G07-result.json`. Each `scenario-run` is one pass; V adds `--variant rejected-removed`. Both offline invocations read the same L1 path. `replay-verify` negative is a separate command, not part of the full suite.
+
+Budget: **3 compiler tasks across2 compile-bearing commands /8**; **3 unit/probe invocations /4** (baseline, full suite, negative); **1 integration /2**; **5 positive executions /5**, exactly1000 ticks. Including baseline200 and negative38: **1238 G07-specific ticks**. Only expected baseline/negative failures occurred. Fault/load **0/0**. Unresolved: **none**. No push, G08+ or external infrastructure.
diff --git a/src/main/java/arena/ArenaServer.java b/src/main/java/arena/ArenaServer.java
index 283224a..9d7d2c1 100644
--- a/src/main/java/arena/ArenaServer.java
+++ b/src/main/java/arena/ArenaServer.java
@@ -69,6 +69,7 @@ public final class ArenaServer implements AutoCloseable {
     private long manualNanos;
     private volatile List<String> closedLifecycle = List.of();
     private volatile int closedInputHighWater;
+    private volatile int closedReplayBytes;
     private volatile ObjectNode closedClockMetrics = Json.MAPPER.createObjectNode().put("active", false)
             .put("accumulator_ns", 0L).put("max_ticks_per_iteration", 0);
 
@@ -342,6 +343,7 @@ public final class ArenaServer implements AutoCloseable {
         return call(() -> {
             ObjectNode result = Json.MAPPER.createObjectNode().put("manual_time_ns", manualNanos)
                     .put("pending_input_high_water", room == null ? 0 : room.inputHighWater())
+                    .put("replay_bytes", room == null ? 0 : room.replayStoredBytes())
                     .put("mailbox_high_water", mailboxHighWater.get()).put("outbound_high_water", outboundHighWater.get());
             result.set("clock", fixedClock == null ? closedClockMetrics.deepCopy() : fixedClock.view());
             result.set("parser", parserMetrics.view());
@@ -349,6 +351,15 @@ public final class ArenaServer implements AutoCloseable {
         });
     }
 
+    /** Immutable capture through the current owner boundary; contains no transport/session credentials. */
+    public byte[] replayArtifact() {
+        if (closing.get()) throw new IllegalStateException("capture requires a live owner");
+        return call(() -> {
+            if (room == null) throw new IllegalStateException("Room has not started");
+            return room.replayArtifact();
+        });
+    }
+
     public ObjectNode cleanup() {
         long liveThreads = ownedThreads.stream().filter(Thread::isAlive).count();
         ObjectNode result = Json.MAPPER.createObjectNode().put("open_channels", peers.size() + (listener.isOpen() ? 1 : 0))
@@ -357,6 +368,7 @@ public final class ArenaServer implements AutoCloseable {
                 .put("timer_alive", ticker != null && ticker.isAlive()).put("owner_terminated", owner.isTerminated())
                 .put("event_loops_terminated", acceptLoop.isTerminated() && ioLoop.isTerminated())
                 .put("pending_input_high_water", closedInputHighWater)
+                .put("replay_bytes", closedReplayBytes)
                 .put("mailbox_high_water", mailboxHighWater.get()).put("outbound_high_water", outboundHighWater.get());
         result.set("parser", parserMetrics.view());
         result.set("clock", closedClockMetrics.deepCopy());
@@ -382,6 +394,7 @@ public final class ArenaServer implements AutoCloseable {
             sessions.clear();
             if (room != null) {
                 room.close();
+                closedReplayBytes = room.replayStoredBytes();
                 closedLifecycle = room.lifecycle();
                 closedInputHighWater = room.inputHighWater();
             }
diff --git a/src/main/java/arena/ReplayLog.java b/src/main/java/arena/ReplayLog.java
new file mode 100644
index 0000000..0639dd2
--- /dev/null
+++ b/src/main/java/arena/ReplayLog.java
@@ -0,0 +1,62 @@
+package arena;
+
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import java.io.ByteArrayOutputStream;
+import java.nio.charset.StandardCharsets;
+import java.security.MessageDigest;
+import java.security.NoSuchAlgorithmException;
+import java.util.Collection;
+import java.util.Comparator;
+import java.util.HexFormat;
+
+/** Owner-confined capture. Overflow is explicit and cannot masquerade as a complete replay. */
+final class ReplayLog {
+    static final int MAX_BYTES = 4_194_304;
+    private ByteArrayOutputStream output = new ByteArrayOutputStream(4_096);
+    private boolean overflowed;
+
+    ReplayLog(String roomId, Collection<Room.Player> players) {
+        ObjectNode header = Json.MAPPER.createObjectNode().put("kind", "HEADER").put("contract_version", 1)
+                .put("room_id", roomId).put("owner_epoch", 0).put("status", "RUNNING");
+        var mapping = header.putArray("players");
+        players.stream().sorted(Comparator.comparingInt(player -> player.slot)).forEach(player -> {
+            ObjectNode entry = mapping.addObject().put("player_id", player.id).put("slot", player.slot);
+            entry.putArray("spawn").add(player.x).add(player.y);
+        });
+        append(header);
+    }
+
+    void accepted(int beforeTick, String playerId, Room.Intent intent) {
+        ObjectNode event = Json.MAPPER.createObjectNode().put("kind", "INPUT").put("before_tick", beforeTick)
+                .put("player_id", playerId).put("seq", intent.seq()).put("target_tick", intent.targetTick())
+                .put("direction", intent.direction().name()).put("owner_epoch", 0);
+        if (intent.target() == null) event.putNull("tag_target_player_id"); else event.put("tag_target_player_id", intent.target());
+        append(event);
+    }
+    void left(int beforeTick, String playerId) {
+        append(Json.MAPPER.createObjectNode().put("kind", "LEFT").put("before_tick", beforeTick).put("player_id", playerId));
+    }
+    void tick(int tick, String record, String hash) {
+        append(Json.MAPPER.createObjectNode().put("kind", "TICK").put("tick", tick)
+                .put("expected_hash", hash).put("canonical_record", record));
+    }
+    private void append(ObjectNode record) {
+        if (overflowed || output == null) return;
+        byte[] bytes = Json.bytes(record);
+        if (bytes.length >= MAX_BYTES - output.size()) { overflowed = true; return; }
+        output.writeBytes(bytes); output.write('\n');
+    }
+    byte[] bytes() {
+        if (output == null) throw new IllegalStateException("replay capture is closed");
+        if (overflowed) throw new IllegalStateException("replay artifact exceeded byte bound; capture incomplete");
+        return output.toByteArray();
+    }
+    void close() { output = null; }
+    int storedBytes() { return output == null ? 0 : output.size(); }
+    boolean overflowed() { return overflowed; }
+    static String hash(String canonicalRecord) {
+        try {
+            return HexFormat.of().formatHex(MessageDigest.getInstance("SHA-256").digest(canonicalRecord.getBytes(StandardCharsets.UTF_8)));
+        } catch (NoSuchAlgorithmException unavailable) { throw new IllegalStateException(unavailable); }
+    }
+}
diff --git a/src/main/java/arena/Room.java b/src/main/java/arena/Room.java
index 97839cc..55add01 100644
--- a/src/main/java/arena/Room.java
+++ b/src/main/java/arena/Room.java
@@ -62,6 +62,8 @@ final class Room {
     private int nextSlot;
     private int inputHighWater;
     private int validationHighWater;
+    private ReplayLog replay;
+    private String stateHash;
 
     Room(String id) { this.id = id; }
     void assertOwner() {
@@ -79,7 +81,10 @@ final class Room {
             throw new IllegalStateException("ROOM_NOT_JOINABLE");
         Player player = new Player(playerId, nextSlot++);
         players.put(playerId, player);
-        if (players.values().stream().filter(p -> p.connected).count() >= 2) transition(Status.RUNNING);
+        if (players.values().stream().filter(p -> p.connected).count() >= 2) {
+            transition(Status.RUNNING);
+            replay = new ReplayLog(id, players.values());
+        }
         return player;
     }
 
@@ -110,6 +115,7 @@ final class Room {
         player.pending.addLast(intent);
         player.lastAcceptedSeq = intent.seq();
         player.lastAcceptedIntent = intent;
+        replay.accepted(executedTicks, id, intent);
         inputHighWater = Math.max(inputHighWater, player.pending.size());
         return null;
     }
@@ -164,18 +170,23 @@ final class Room {
             }
         }
         if (++executedTicks == DURATION) transition(Status.FINISHED);
+        String record = canonicalRecord();
+        stateHash = ReplayLog.hash(record);
+        replay.tick(executedTicks - 1, record, stateHash);
         return rejected;
     }
 
     void left(String id) {
         assertOwner();
         Player player = players.get(id);
+        if (player != null && player.connected && replay != null) replay.left(executedTicks, id);
         if (player != null) { player.connected = false; player.direction = Direction.STOP; player.pending.clear(); }
     }
 
     void close() {
         assertOwner();
         for (Player player : players.values()) player.pending.clear();
+        if (replay != null) replay.close();
         if (status != Status.CLOSED) transition(Status.CLOSED);
     }
 
@@ -188,5 +199,23 @@ final class Room {
         return result;
     }
 
+    String canonicalRecord() {
+        assertOwner();
+        StringBuilder record = new StringBuilder().append("v=1|room=").append(id).append("|tick=").append(executedTicks - 1)
+                .append("|status=").append(status).append("|owner_epoch=0\n");
+        for (Player player : players.values()) record.append("player=").append(player.id).append("|slot=").append(player.slot)
+                .append("|x=").append(player.x).append("|y=").append(player.y).append("|dir=").append(player.direction)
+                .append("|score=").append(player.score).append("|conn=").append(player.connected ? "CONNECTED" : "LEFT")
+                .append("|last_seq=").append(player.lastAcceptedSeq).append("|last_tag_tick=").append(player.lastTagTick).append('\n');
+        return record.toString();
+    }
+    String stateHash() { assertOwner(); return stateHash; }
+    int replayStoredBytes() { assertOwner(); return replay == null ? 0 : replay.storedBytes(); }
+    byte[] replayArtifact() {
+        assertOwner();
+        if (replay == null) throw new IllegalStateException("Room has not started");
+        return replay.bytes();
+    }
+
     private void transition(Status next) { status = next; lifecycle.add(next.name()); }
 }
diff --git a/src/main/java/arena/ScenarioRunner.java b/src/main/java/arena/ScenarioRunner.java
index 2537429..359dd29 100644
--- a/src/main/java/arena/ScenarioRunner.java
+++ b/src/main/java/arena/ScenarioRunner.java
@@ -153,6 +153,7 @@ final class ScenarioRunner {
         if (cleanup.path("pending_input_high_water").asInt() > Room.INPUT_LIMIT) failures.add("input bound");
         if (cleanup.path("mailbox_high_water").asInt() > ArenaServer.MAILBOX_LIMIT) failures.add("mailbox bound");
         if (cleanup.path("outbound_high_water").asInt() > ArenaServer.OUTBOUND_LIMIT) failures.add("outbound bound");
+        if (cleanup.path("replay_bytes").asInt(-1) != 0) failures.add("replay storage cleanup");
         JsonNode clock = cleanup.path("clock");
         if (clock.path("active").asBoolean(true) || clock.path("accumulator_ns").asLong(-1) != 0
                 || clock.path("max_ticks_per_iteration").asInt(-1) < 0
diff --git a/src/test/java/arena/ReplayFixture.java b/src/test/java/arena/ReplayFixture.java
new file mode 100644
index 0000000..65eff70
--- /dev/null
+++ b/src/test/java/arena/ReplayFixture.java
@@ -0,0 +1,93 @@
+package arena;
+
+import com.fasterxml.jackson.databind.JsonNode;
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import java.io.IOException;
+import java.lang.reflect.Field;
+import java.util.Map;
+import java.util.TreeMap;
+import java.util.concurrent.Callable;
+import java.util.concurrent.ThreadPoolExecutor;
+import java.util.concurrent.TimeUnit;
+import java.util.function.LongSupplier;
+
+/** Test runtime only. Initial state setup is never represented as successful public joins. */
+final class ReplayFixture {
+    private ReplayFixture() { }
+
+    static ObjectNode bootstrap(ArenaServer server, ObjectNode initial, Map<String, TcpClient> clients) throws Exception {
+        return owned(server, () -> {
+            Room room = prepared(initial); set(server, "room", room);
+            Object registry = field(server, "sessions");
+            if (!(registry instanceof Map<?, ?> sessions) || sessions.size() != clients.size())
+                throw new IOException("four real HELLO session bindings required");
+            int bound = 0;
+            for (JsonNode player : initial.withArray("players")) {
+                TcpClient client = clients.get(player.path("client").asText());
+                boolean found = false;
+                for (Object session : sessions.values()) if (field(session, "id").equals(client.sessionId)) {
+                    set(session, "playerId", player.path("player_id").asText()); found = true; bound++;
+                }
+                if (!found) throw new IOException("real session missing");
+            }
+            start(room, initial);
+            set(server, "fixedClock", new FixedTickClock((LongSupplier) field(server, "monotonicNanos")));
+            ObjectNode observed = Json.MAPPER.createObjectNode().put("kind", "test-only owner initial state; not public joins")
+                    .put("bound_live_sessions", bound).put("normal_start_evaluations", 1);
+            observed.set("state", view(room)); return observed;
+        });
+    }
+
+    static Room offlineInitial(ObjectNode header) throws Exception {
+        Room room = prepared(header); start(room, header); return room;
+    }
+
+    private static Room prepared(ObjectNode initial) throws Exception {
+        var players = initial.withArray("players");
+        if (players.size() < 2 || players.size() > Room.SPAWNS.length) throw new IOException("initial roster bound");
+        Room room = new Room(Json.text(initial, "room_id"));
+        Object storage = field(room, "players");
+        if (!(storage instanceof TreeMap<?, ?>)) throw new IOException("unexpected Room player storage");
+        @SuppressWarnings("unchecked") Map<String, Room.Player> roster = (Map<String, Room.Player>) storage;
+        for (int slot = 0; slot < players.size(); slot++) {
+            JsonNode p = players.get(slot);
+            if (p.path("slot").asInt(-1) != slot || p.path("spawn").get(0).asInt() != Room.SPAWNS[slot][0]
+                    || p.path("spawn").get(1).asInt() != Room.SPAWNS[slot][1]) throw new IOException("initial slot/spawn mapping");
+            if (slot < players.size() - 1 && roster.put(p.path("player_id").asText(), new Room.Player(p.path("player_id").asText(), slot)) != null)
+                throw new IOException("duplicate fixture player");
+        }
+        set(room, "nextSlot", players.size() - 1); return room;
+    }
+
+    private static void start(Room room, ObjectNode initial) throws IOException {
+        JsonNode last = initial.withArray("players").get(initial.withArray("players").size() - 1);
+        room.join(last.path("player_id").asText());
+        if (room.status() != Room.Status.RUNNING) throw new IOException("unchanged start condition failed");
+    }
+
+    static ObjectNode snapshot(ArenaServer server) throws Exception { return owned(server, () -> view((Room) field(server, "room"))); }
+    static ObjectNode view(Room room) {
+        ObjectNode state = room.view("SNAPSHOT");
+        state.put("state_hash", room.stateHash());
+        for (JsonNode p : state.withArray("players")) {
+            Room.Player player = room.player(p.path("player_id").asText());
+            ((ObjectNode) p).put("last_tag_tick", player.lastTagTick).put("pending_inputs", player.pending.size());
+        }
+        return state;
+    }
+    static String canonicalRecord(ArenaServer server) throws Exception {
+        return owned(server, () -> ((Room) field(server, "room")).canonicalRecord());
+    }
+    static byte[] artifact(ArenaServer server) throws Exception {
+        return server.replayArtifact();
+    }
+    static <T> T owned(ArenaServer server, Callable<T> operation) throws Exception {
+        return ((ThreadPoolExecutor) field(server, "owner")).submit(operation).get(5, TimeUnit.SECONDS);
+    }
+    static Object field(Object object, String name) throws ReflectiveOperationException {
+        Field field = object.getClass().getDeclaredField(name); field.setAccessible(true); return field.get(object);
+    }
+    private static void set(Object object, String name, Object value) throws ReflectiveOperationException {
+        Field field = object.getClass().getDeclaredField(name); field.setAccessible(true); field.set(object, value);
+    }
+}
diff --git a/src/test/java/arena/ReplayFormatTest.java b/src/test/java/arena/ReplayFormatTest.java
new file mode 100644
index 0000000..703ed94
--- /dev/null
+++ b/src/test/java/arena/ReplayFormatTest.java
@@ -0,0 +1,57 @@
+package arena;
+
+import static org.junit.jupiter.api.Assertions.*;
+import java.math.BigInteger;
+import java.net.URL;
+import java.net.URLClassLoader;
+import java.nio.file.Path;
+import java.util.List;
+import java.util.jar.JarFile;
+import org.junit.jupiter.api.Test;
+
+/** Zero simulation ticks: the separately budgeted command invocations own all G07 campaigns. */
+final class ReplayFormatTest {
+    @Test void canonicalFieldOrderAsciiPlayersAndFinalLf() {
+        Room room = new Room("format-room"); room.join("z-player"); room.join("a-player");
+        String expected = "v=1|room=format-room|tick=-1|status=RUNNING|owner_epoch=0\n"
+                + "player=a-player|slot=1|x=90000|y=90000|dir=STOP|score=0|conn=CONNECTED|last_seq=0|last_tag_tick=-20\n"
+                + "player=z-player|slot=0|x=10000|y=10000|dir=STOP|score=0|conn=CONNECTED|last_seq=0|last_tag_tick=-20\n";
+        assertEquals(expected, room.canonicalRecord());
+        assertEquals("ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad", ReplayLog.hash("abc"));
+        assertNotEquals(ReplayLog.hash(expected), ReplayLog.hash(expected.stripTrailing()));
+        assertEquals(0, room.executedTicks()); room.close(); assertEquals(0, room.replayStoredBytes());
+    }
+
+    @Test void immutableCaptureExplicitOverflowAndCloseRelease() {
+        Room room = new Room("capture-room"); room.join("first"); room.join("second");
+        byte[] first = room.replayArtifact(), untouched = room.replayArtifact(); first[0] ^= 1;
+        assertArrayEquals(untouched, room.replayArtifact()); room.close();
+        assertEquals(0, room.replayStoredBytes()); assertThrows(IllegalStateException.class, room::replayArtifact);
+        ReplayLog log = new ReplayLog("bounded-room", List.of(new Room.Player("first", 0), new Room.Player("second", 1)));
+        int before = log.storedBytes();
+        log.accepted(0, "first", new Room.Intent(BigInteger.ONE, BigInteger.ZERO, Room.Direction.STOP, "x".repeat(ReplayLog.MAX_BYTES)));
+        assertTrue(log.overflowed()); assertEquals(before, log.storedBytes());
+        assertTrue(log.storedBytes() <= ReplayLog.MAX_BYTES);
+        IllegalStateException incomplete = assertThrows(IllegalStateException.class, log::bytes);
+        int afterOverflow = log.storedBytes();
+        log.close(); assertEquals(0, log.storedBytes()); assertThrows(IllegalStateException.class, log::bytes);
+        assertEquals(0, room.executedTicks());
+        System.out.println(Json.MAPPER.createObjectNode().put("probe", "G07 zero-tick capture")
+                .put("immutable_copy", true).put("before_bytes", before).put("after_overflow_bytes", afterOverflow)
+                .put("overflow_latched", log.overflowed()).put("export_error", incomplete.getMessage())
+                .put("closed_bytes", log.storedBytes()).put("executed_ticks", room.executedTicks()));
+    }
+
+    @Test void shippingJarAndProductionLoaderExcludeFixtureAndReplayTool() throws Exception {
+        Path path = Path.of(System.getProperty("arena.productionJar"));
+        try (JarFile jar = new JarFile(path.toFile());
+             URLClassLoader production = new URLClassLoader(new URL[] {path.toUri().toURL()}, ClassLoader.getPlatformClassLoader())) {
+            assertNotNull(jar.getJarEntry("arena/ArenaServer.class"));
+            for (String name : List.of("ReplayFixture", "ReplayScenario", "ReplayVerifier", "ReplayTool", "G07BaselineTest")) {
+                assertNull(jar.getJarEntry("arena/" + name + ".class"));
+                assertThrows(ClassNotFoundException.class, () -> Class.forName("arena." + name, false, production));
+            }
+        }
+        System.out.println("G07 production fixture disabled: shipping JAR and isolated production loader exclude all test bootstrap/tool classes");
+    }
+}
diff --git a/src/test/java/arena/ReplayScenario.java b/src/test/java/arena/ReplayScenario.java
new file mode 100644
index 0000000..e1d14c6
--- /dev/null
+++ b/src/test/java/arena/ReplayScenario.java
@@ -0,0 +1,146 @@
+package arena;
+
+import com.fasterxml.jackson.databind.JsonNode;
+import com.fasterxml.jackson.databind.node.ArrayNode;
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import java.io.DataInputStream;
+import java.io.IOException;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.security.MessageDigest;
+import java.util.HexFormat;
+import java.util.LinkedHashMap;
+import java.util.Map;
+
+/** Exactly one live 200-tick execution per invocation; no hidden repeat/offline campaign. */
+final class ReplayScenario {
+    static final String SHA256 = "d8c80202888d2e3dbeb1a7c03c14a73e84899d7cc65546e1c58a8b3102c70796";
+    record Observed(ObjectNode result, byte[] replay) { }
+    private ReplayScenario() { }
+
+    static Observed run(Path path, boolean rejectedRemoved) throws Exception {
+        byte[] bytes = Files.readAllBytes(path);
+        String hash = HexFormat.of().formatHex(MessageDigest.getInstance("SHA-256").digest(bytes));
+        if (!SHA256.equals(hash)) throw new IOException("G07 requires frozen scenario bytes");
+        ObjectNode scenario = Json.read(bytes);
+        ObjectNode result = Json.MAPPER.createObjectNode().put("thread", "G07").put("contract_version", 1)
+                .put("scenario_sha256", hash).put("process_id", ProcessHandle.current().pid())
+                .put("mode", rejectedRemoved ? "live-rejected-removed" : "live")
+                .put("network_fault_runs", 0).put("load_runs", 0);
+        ArrayNode admissions = result.putArray("admissions"), lifecycle = result.putArray("lifecycle_events");
+        ArrayNode tagEvents = result.putArray("tag_events"), raw = result.putArray("events"), ticks = result.putArray("ticks");
+        ArrayNode hashes = result.putArray("state_hashes"), records = result.putArray("canonical_records"), failures = result.putArray("failures");
+        Map<String, TcpClient> clients = new LinkedHashMap<>(); byte[] replay = null;
+        ArenaServer server = new ArenaServer("127.0.0.1", 0, true);
+        try {
+            for (JsonNode p : scenario.withArray("players")) {
+                TcpClient client = new TcpClient(server.port()); clients.put(p.path("client").asText(), client);
+                client.hello(); client.playerId = p.path("player_id").asText();
+            }
+            result.set("bootstrap", ReplayFixture.bootstrap(server, scenario, clients));
+            int eventIndex = 0, removed = 0;
+            for (int tick = 0; tick < scenario.path("ticks").asInt(); tick++) {
+                Map<String, String> acceptedTags = new LinkedHashMap<>();
+                while (eventIndex < scenario.withArray("events").size()
+                        && scenario.withArray("events").get(eventIndex).path("before_tick").asInt() == tick) {
+                    JsonNode event = scenario.withArray("events").get(eventIndex++);
+                    if (rejectedRemoved && !event.path("include_in_rejected_removed_variant").asBoolean()) { removed++; continue; }
+                    String role = event.path("client").asText(); TcpClient actor = clients.get(role);
+                    ObjectNode before = ReplayFixture.snapshot(server);
+                    ObjectNode request = actor.auth(event.path("kind").asText(), scenario.path("room_id").asText());
+                    if (event.path("kind").asText().equals("LEAVE_ROOM")) {
+                        actor.send(request); actor.expectClosed(); actor.close();
+                        ObjectNode after = ReplayFixture.snapshot(server);
+                        lifecycle.addObject().put("before_tick", tick).put("player_id", actor.playerId).put("kind", "LEFT");
+                        ObjectNode observed = raw.addObject(); observed.set("request", request); observed.set("before", before); observed.set("after", after);
+                        if (!player(after, actor.playerId).path("connectivity").asText().equals("LEFT")) failures.add("real LEAVE did not commit");
+                        continue;
+                    }
+                    for (String key : new String[] {"seq", "target_tick", "direction", "owner_epoch"}) request.set(key, event.path(key));
+                    String targetRole = event.path("tag_target_role").isNull() ? null : event.path("tag_target_role").asText();
+                    if (targetRole == null) request.putNull("tag_target_player_id");
+                    else request.put("tag_target_player_id", clients.get(targetRole).playerId);
+                    actor.send(request); ObjectNode response = response(actor), after = ReplayFixture.snapshot(server);
+                    String code = response.path("type").asText().equals("INPUT_ACK") ? response.path("status").asText() : response.path("code").asText();
+                    admissions.addObject().put("before_tick", tick).put("player_id", actor.playerId)
+                            .put("seq", event.path("seq").asInt()).put("target_tick", event.path("target_tick").asInt()).put("code", code);
+                    ObjectNode observed = raw.addObject(); observed.set("request", request); observed.set("response", response);
+                    observed.set("before", before); observed.set("after", after);
+                    if (!code.equals(event.path("expected_admission").asText())) failures.add("admission " + role + " seq" + event.path("seq") + ": " + code);
+                    if (!code.equals("ACCEPTED") && !before.equals(after)) failures.add("rejected input changed state");
+                    if (code.equals("ACCEPTED") && targetRole != null) acceptedTags.put(role, targetRole);
+                }
+                ObjectNode before = ReplayFixture.snapshot(server); server.advanceTicks(1);
+                ObjectNode after = ReplayFixture.snapshot(server); ticks.add(after);
+                JsonNode stateHash = after.get("state_hash"); hashes.add(stateHash == null ? Json.MAPPER.nullNode() : stateHash);
+                records.add(ReplayFixture.canonicalRecord(server));
+                for (var tag : acceptedTags.entrySet()) {
+                    TcpClient actor = clients.get(tag.getKey());
+                    int delta = player(after, actor.playerId).path("score").asInt() - player(before, actor.playerId).path("score").asInt();
+                    String outcome;
+                    if (delta == 1) outcome = "TAG_SUCCESS";
+                    else {
+                        ObjectNode response = response(actor); outcome = response.path("code").asText();
+                        result.set("last_tag_error", response);
+                        if (delta != 0 || !outcome.equals("ACTION_REJECTED")) failures.add("TAG result");
+                    }
+                    tagEvents.addObject().put("tick", after.path("tick").asInt()).put("actor", actor.playerId)
+                            .put("target", clients.get(tag.getValue()).playerId).put("result", outcome);
+                }
+            }
+            ObjectNode state = ReplayFixture.snapshot(server); result.set("final_state", state);
+            result.put("executed_ticks", state.path("executed_ticks").asInt()).put("removed_rejected_inputs", removed);
+            if (rejectedRemoved && removed != 4 || !rejectedRemoved && removed != 0) failures.add("variant filtering");
+            assertFinal(state, failures);
+            expect(tags(tagEvents), "[\"player-00:TAG_SUCCESS\",\"player-02:ACTION_REJECTED\"]", failures, "accepted TAG results");
+            for (JsonNode value : hashes) if (!value.isTextual() || !value.asText().matches("[0-9a-f]{64}")) {
+                failures.add("canonical per-tick hashes unavailable/invalid"); break;
+            }
+            for (JsonNode value : records) if (!value.isTextual() || !value.asText().endsWith("\n")) {
+                failures.add("canonical records unavailable/invalid"); break;
+            }
+            replay = ReplayFixture.artifact(server); result.put("replay_artifact_available", replay != null);
+            result.put("artifact_bytes", replay == null ? 0 : replay.length);
+            if (replay == null || replay.length > scenario.path("artifact_max_bytes").asInt()) failures.add("bounded replay artifact unavailable/invalid");
+            result.set("runtime_metrics", server.metrics());
+            server.close();
+            for (TcpClient client : clients.values()) if (!client.isClosed()) client.expectClosed();
+        } finally {
+            server.close(); for (TcpClient client : clients.values()) client.close();
+        }
+        ScenarioRunner.assertCleanup(server.cleanup()); result.set("cleanup", server.cleanup());
+        result.put("all_resources_released", clients.values().stream().allMatch(TcpClient::isClosed));
+        result.put("passed", failures.isEmpty()); return new Observed(result, replay);
+    }
+
+    static void assertFinal(ObjectNode state, ArrayNode failures) throws IOException {
+        String[] ids = {"player-00", "player-01", "player-02", "player-03"};
+        String[] directions = {"NORTH", "SOUTH", "EAST", "STOP"}; int[] seqs = {12, 3, 3, 3};
+        for (int i = 0; i < ids.length; i++) {
+            JsonNode p = player(state, ids[i]);
+            if (p.path("x").asInt() != (i == 3 ? 60400 : 50000) || p.path("y").asInt() != 50000
+                    || !p.path("direction").asText().equals(directions[i]) || p.path("score").asInt() != (i == 0 ? 1 : 0)
+                    || p.path("last_accepted_seq").asInt() != seqs[i] || p.path("last_tag_tick").asInt() != (i == 0 ? 199 : -20)
+                    || !p.path("connectivity").asText().equals(i == 3 ? "LEFT" : "CONNECTED")) failures.add("final player " + ids[i]);
+        }
+        if (state.path("executed_ticks").asInt() != 200 || state.path("tick").asInt() != 199
+                || !state.path("status").asText().equals("RUNNING")) failures.add("final clock/lifecycle");
+    }
+    static ObjectNode response(TcpClient client) throws Exception {
+        DataInputStream input = (DataInputStream) ReplayFixture.field(client, "input"); int length = input.readInt();
+        if (length < 1 || length > CompleteFrame.MAX_BYTES) throw new IOException("server frame bound");
+        byte[] bytes = new byte[length]; input.readFully(bytes); return Json.read(bytes);
+    }
+    static JsonNode player(ObjectNode state, String id) throws IOException {
+        for (JsonNode p : state.withArray("players")) if (p.path("player_id").asText().equals(id)) return p;
+        throw new IOException("missing authoritative player");
+    }
+    private static ArrayNode tags(ArrayNode events) {
+        ArrayNode result = Json.MAPPER.createArrayNode();
+        for (JsonNode event : events) result.add(event.path("actor").asText() + ":" + event.path("result").asText());
+        return result;
+    }
+    private static void expect(JsonNode actual, String expected, ArrayNode failures, String label) {
+        if (!actual.toString().equals(expected)) failures.add(label);
+    }
+}
diff --git a/src/test/java/arena/ReplayTool.java b/src/test/java/arena/ReplayTool.java
new file mode 100644
index 0000000..cb0731b
--- /dev/null
+++ b/src/test/java/arena/ReplayTool.java
@@ -0,0 +1,43 @@
+package arena;
+
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.nio.file.StandardOpenOption;
+import java.security.MessageDigest;
+import java.util.HexFormat;
+
+/** Test classpath command entry point. Each invocation executes one explicitly requested pass. */
+public final class ReplayTool {
+    private ReplayTool() { }
+
+    public static void main(String[] args) throws Exception {
+        if (args.length < 3) throw new IllegalArgumentException("scenario-run <scenario> <result> [--variant rejected-removed] | replay-verify <artifact> <result>");
+        Path input = Path.of(args[1]), output = Path.of(args[2]);
+        Files.createDirectories(output.toAbsolutePath().getParent());
+        ObjectNode result;
+        if (args[0].equals("scenario-run")) {
+            if (Files.size(input) > 65_536) throw new IllegalArgumentException("scenario byte bound");
+            ObjectNode scenario = Json.read(Files.readAllBytes(input));
+            if (scenario.path("thread").asText().equals("G07")) {
+                boolean variant = args.length == 5 && args[3].equals("--variant") && args[4].equals("rejected-removed");
+                if (args.length != 3 && !variant) throw new IllegalArgumentException("unknown G07 variant");
+                ReplayScenario.Observed observed = ReplayScenario.run(input, variant); result = observed.result();
+                Path artifact = output.resolveSibling(output.getFileName().toString().replaceFirst("\\.json$", "") + ".replay.jsonl");
+                if (observed.replay() != null) {
+                    Files.write(artifact, observed.replay(), StandardOpenOption.CREATE_NEW);
+                    result.put("replay_artifact_path", artifact.toAbsolutePath().toString()).put("artifact_sha256",
+                            HexFormat.of().formatHex(MessageDigest.getInstance("SHA-256").digest(observed.replay())));
+                }
+            } else {
+                if (args.length != 3) throw new IllegalArgumentException("variant is only active for G07");
+                result = ScenarioRunner.run(input);
+            }
+        } else if (args[0].equals("replay-verify") && args.length == 3) result = ReplayVerifier.run(input);
+        else throw new IllegalArgumentException("unsupported tool command");
+        Files.write(output, Json.MAPPER.writerWithDefaultPrettyPrinter().writeValueAsBytes(result));
+        boolean passed = !result.has("passed") || result.path("passed").asBoolean();
+        System.out.println(args[0] + " " + (passed ? "PASS" : "FAIL") + ": " + output);
+        if (!passed) System.exit(1);
+    }
+}
diff --git a/src/test/java/arena/ReplayVerifier.java b/src/test/java/arena/ReplayVerifier.java
new file mode 100644
index 0000000..6c4dc4b
--- /dev/null
+++ b/src/test/java/arena/ReplayVerifier.java
@@ -0,0 +1,90 @@
+package arena;
+
+import com.fasterxml.jackson.databind.node.ArrayNode;
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import java.io.IOException;
+import java.nio.ByteBuffer;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.security.MessageDigest;
+import java.util.HexFormat;
+
+/** Offline verification tool in the test runtime, reusing the real Room and its initial-state fixture. */
+final class ReplayVerifier {
+    private ReplayVerifier() { }
+
+    static ObjectNode run(Path artifact) throws Exception {
+        byte[] bytes;
+        try (var input = Files.newInputStream(artifact)) { bytes = input.readNBytes(ReplayLog.MAX_BYTES + 1); }
+        if (bytes.length == 0 || bytes.length > ReplayLog.MAX_BYTES || bytes[bytes.length - 1] != '\n')
+            throw new IOException("replay byte bound or incomplete final record");
+        ObjectNode result = Json.MAPPER.createObjectNode().put("thread", "G07").put("mode", "offline")
+                .put("process_id", ProcessHandle.current().pid()).put("artifact_bytes", bytes.length)
+                .put("artifact_sha256", HexFormat.of().formatHex(MessageDigest.getInstance("SHA-256").digest(bytes)));
+        ArrayNode hashes = result.putArray("state_hashes"), records = result.putArray("canonical_records");
+        ArrayNode admitted = result.putArray("accepted_inputs"), lifecycle = result.putArray("lifecycle_events");
+        ArrayNode rejectedTags = result.putArray("action_rejections");
+        Room room = null; int offset = 0; boolean passed = true;
+        try {
+            while (offset < bytes.length) {
+                int end = offset;
+                while (bytes[end] != '\n') end++;
+                ObjectNode entry = Json.read(ByteBuffer.wrap(bytes, offset, end - offset)); offset = end + 1;
+                String kind = Json.text(entry, "kind");
+                if (room == null) {
+                    if (!kind.equals("HEADER") || !entry.path("contract_version").isIntegralNumber()
+                            || entry.path("contract_version").asInt() != 1 || entry.path("owner_epoch").asInt(-1) != 0
+                            || !entry.path("status").asText().equals("RUNNING")) throw new IOException("replay header contract");
+                    room = ReplayFixture.offlineInitial(entry); result.set("initial_mapping", entry); continue;
+                }
+                switch (kind) {
+                    case "INPUT" -> {
+                        boundary(entry, room);
+                        String player = Json.text(entry, "player_id");
+                        if (!room.beginInputValidation(player)) throw new IOException("recorded INPUT exceeds validation bound");
+                        String target = entry.path("tag_target_player_id").isNull() ? null : Json.text(entry, "tag_target_player_id");
+                        Room.Intent intent = new Room.Intent(Json.sequence(entry), Json.targetTick(entry),
+                                Room.Direction.valueOf(Json.text(entry, "direction")), target);
+                        String code = room.accept(player, intent);
+                        if (code != null) throw new IOException("recorded accepted INPUT rejected: " + code);
+                        admitted.add(entry);
+                    }
+                    case "LEFT" -> {
+                        boundary(entry, room); String player = Json.text(entry, "player_id");
+                        if (room.player(player) == null || !room.player(player).connected) throw new IOException("invalid LEFT record");
+                        room.left(player); lifecycle.add(entry);
+                    }
+                    case "TICK" -> {
+                        if (!entry.path("tick").isIntegralNumber() || entry.path("tick").asInt(-1) != room.executedTicks()
+                                || room.executedTicks() >= Room.DURATION) throw new IOException("noncontiguous replay tick");
+                        for (Room.Rejection rejection : room.tick()) rejectedTags.addObject().put("tick", room.executedTicks() - 1)
+                                .put("player_id", rejection.playerId()).put("code", rejection.code());
+                        String actualRecord = room.canonicalRecord(), actualHash = room.stateHash();
+                        hashes.add(actualHash); records.add(actualRecord);
+                        if (!actualHash.equals(Json.text(entry, "expected_hash")) || !actualRecord.equals(Json.text(entry, "canonical_record"))) {
+                            result.put("first_divergent_tick", room.executedTicks() - 1).put("actual_hash", actualHash)
+                                    .put("expected_hash", entry.path("expected_hash").asText()).put("actual_record", actualRecord);
+                            passed = false;
+                        }
+                    }
+                    default -> throw new IOException("unknown replay record kind");
+                }
+                if (!passed) break;
+            }
+            if (room == null) throw new IOException("missing replay initial state");
+            result.put("executed_ticks", room.executedTicks()); result.set("final_state", ReplayFixture.view(room));
+            if (passed) result.putNull("first_divergent_tick");
+        } finally { if (room != null) room.close(); }
+        int pending = 0;
+        if (room != null) for (var p : room.view("SNAPSHOT").withArray("players")) pending += room.player(p.path("player_id").asText()).pending.size();
+        result.putObject("cleanup").put("pending_inputs", pending).put("replay_bytes", room == null ? 0 : room.replayStoredBytes())
+                .put("room_status", room == null ? "NONE" : room.status().name()).put("server_instances_created", 0);
+        boolean released = room != null && room.status() == Room.Status.CLOSED && pending == 0 && room.replayStoredBytes() == 0;
+        result.put("all_resources_released", released).put("passed", passed && released); return result;
+    }
+
+    private static void boundary(ObjectNode entry, Room room) throws IOException {
+        if (!entry.path("before_tick").isIntegralNumber() || entry.path("before_tick").asInt(-1) != room.executedTicks())
+            throw new IOException("event must retain its actual admission boundary");
+    }
+}
diff --git a/src/test/resources/G07.json b/src/test/resources/G07.json
new file mode 100644
index 0000000..d193105
--- /dev/null
+++ b/src/test/resources/G07.json
@@ -0,0 +1,312 @@
+{
+  "scenario_id": "G07-deterministic-replay",
+  "contract_version": 1,
+  "thread": "G07",
+  "seed": 7050,
+  "clock": {
+    "kind": "manual",
+    "tick_duration_ms": 50
+  },
+  "ticks": 200,
+  "room_id": "room-fixture",
+  "players": [
+    {
+      "client": "alpha",
+      "player_id": "player-00",
+      "slot": 0,
+      "spawn": [
+        10000,
+        10000
+      ]
+    },
+    {
+      "client": "bravo",
+      "player_id": "player-01",
+      "slot": 1,
+      "spawn": [
+        90000,
+        90000
+      ]
+    },
+    {
+      "client": "charlie",
+      "player_id": "player-02",
+      "slot": 2,
+      "spawn": [
+        10000,
+        90000
+      ]
+    },
+    {
+      "client": "delta",
+      "player_id": "player-03",
+      "slot": 3,
+      "spawn": [
+        90000,
+        10000
+      ]
+    }
+  ],
+  "initialization": "test-build-only owner fixture bootstrap with four ordered player records before one unchanged start-condition evaluation; normal live joins still start at two and reject RUNNING joins",
+  "events": [
+    {
+      "before_tick": 0,
+      "kind": "INPUT",
+      "client": "alpha",
+      "seq": 1,
+      "target_tick": 0,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0,
+      "expected_admission": "ACCEPTED",
+      "include_in_rejected_removed_variant": true
+    },
+    {
+      "before_tick": 0,
+      "kind": "INPUT",
+      "client": "bravo",
+      "seq": 1,
+      "target_tick": 0,
+      "direction": "WEST",
+      "tag_target_role": null,
+      "owner_epoch": 0,
+      "expected_admission": "ACCEPTED",
+      "include_in_rejected_removed_variant": true
+    },
+    {
+      "before_tick": 0,
+      "kind": "INPUT",
+      "client": "charlie",
+      "seq": 1,
+      "target_tick": 0,
+      "direction": "SOUTH",
+      "tag_target_role": null,
+      "owner_epoch": 0,
+      "expected_admission": "ACCEPTED",
+      "include_in_rejected_removed_variant": true
+    },
+    {
+      "before_tick": 0,
+      "kind": "INPUT",
+      "client": "delta",
+      "seq": 1,
+      "target_tick": 0,
+      "direction": "NORTH",
+      "tag_target_role": null,
+      "owner_epoch": 0,
+      "expected_admission": "ACCEPTED",
+      "include_in_rejected_removed_variant": true
+    },
+    {
+      "before_tick": 10,
+      "kind": "INPUT",
+      "client": "alpha",
+      "seq": 2,
+      "target_tick": 9,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0,
+      "expected_admission": "INPUT_LATE",
+      "include_in_rejected_removed_variant": false
+    },
+    {
+      "before_tick": 11,
+      "kind": "INPUT",
+      "client": "alpha",
+      "seq": 3,
+      "target_tick": 16,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0,
+      "expected_admission": "INPUT_TOO_EARLY",
+      "include_in_rejected_removed_variant": false
+    },
+    {
+      "before_tick": 12,
+      "kind": "INPUT",
+      "client": "alpha",
+      "seq": 1,
+      "target_tick": 12,
+      "direction": "WEST",
+      "tag_target_role": null,
+      "owner_epoch": 0,
+      "expected_admission": "SEQUENCE_CONFLICT",
+      "include_in_rejected_removed_variant": false
+    },
+    {
+      "before_tick": 98,
+      "kind": "INPUT",
+      "client": "bravo",
+      "seq": 2,
+      "target_tick": 100,
+      "direction": "SOUTH",
+      "tag_target_role": null,
+      "owner_epoch": 0,
+      "expected_admission": "ACCEPTED",
+      "include_in_rejected_removed_variant": true
+    },
+    {
+      "before_tick": 100,
+      "kind": "INPUT",
+      "client": "alpha",
+      "seq": 10,
+      "target_tick": 100,
+      "direction": "WEST",
+      "tag_target_role": null,
+      "owner_epoch": 0,
+      "expected_admission": "ACCEPTED",
+      "include_in_rejected_removed_variant": true
+    },
+    {
+      "before_tick": 100,
+      "kind": "INPUT",
+      "client": "alpha",
+      "seq": 11,
+      "target_tick": 100,
+      "direction": "NORTH",
+      "tag_target_role": null,
+      "owner_epoch": 0,
+      "expected_admission": "ACCEPTED",
+      "include_in_rejected_removed_variant": true
+    },
+    {
+      "before_tick": 100,
+      "kind": "INPUT",
+      "client": "charlie",
+      "seq": 2,
+      "target_tick": 100,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0,
+      "expected_admission": "ACCEPTED",
+      "include_in_rejected_removed_variant": true
+    },
+    {
+      "before_tick": 100,
+      "kind": "INPUT",
+      "client": "delta",
+      "seq": 2,
+      "target_tick": 100,
+      "direction": "WEST",
+      "tag_target_role": null,
+      "owner_epoch": 0,
+      "expected_admission": "ACCEPTED",
+      "include_in_rejected_removed_variant": true
+    },
+    {
+      "before_tick": 101,
+      "kind": "INPUT",
+      "client": "alpha",
+      "seq": 9,
+      "target_tick": 101,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0,
+      "expected_admission": "INPUT_STALE",
+      "include_in_rejected_removed_variant": false
+    },
+    {
+      "before_tick": 174,
+      "kind": "INPUT",
+      "client": "delta",
+      "seq": 3,
+      "target_tick": 174,
+      "direction": "STOP",
+      "tag_target_role": null,
+      "owner_epoch": 0,
+      "expected_admission": "ACCEPTED",
+      "include_in_rejected_removed_variant": true
+    },
+    {
+      "before_tick": 175,
+      "kind": "LEAVE_ROOM",
+      "client": "delta",
+      "include_in_rejected_removed_variant": true
+    },
+    {
+      "before_tick": 199,
+      "kind": "INPUT",
+      "client": "alpha",
+      "seq": 12,
+      "target_tick": 199,
+      "direction": "NORTH",
+      "tag_target_role": "bravo",
+      "owner_epoch": 0,
+      "expected_admission": "ACCEPTED",
+      "include_in_rejected_removed_variant": true
+    },
+    {
+      "before_tick": 199,
+      "kind": "INPUT",
+      "client": "bravo",
+      "seq": 3,
+      "target_tick": 199,
+      "direction": "SOUTH",
+      "tag_target_role": null,
+      "owner_epoch": 0,
+      "expected_admission": "ACCEPTED",
+      "include_in_rejected_removed_variant": true
+    },
+    {
+      "before_tick": 199,
+      "kind": "INPUT",
+      "client": "charlie",
+      "seq": 3,
+      "target_tick": 199,
+      "direction": "EAST",
+      "tag_target_role": "bravo",
+      "owner_epoch": 0,
+      "expected_admission": "ACCEPTED",
+      "include_in_rejected_removed_variant": true
+    }
+  ],
+  "variant": "remove only events with include_in_rejected_removed_variant=false, preserving all accepted payloads, original sequences, admission boundaries and lifecycle events",
+  "artifact_max_bytes": 4194304,
+  "socket_ceiling_ms": 5000,
+  "runs": {
+    "baseline": {
+      "category": "unit-reproduction",
+      "count": 1,
+      "ticks": 200
+    },
+    "post_change": [
+      {
+        "id": "L1",
+        "mode": "live",
+        "variant": "canonical",
+        "ticks": 200
+      },
+      {
+        "id": "L2",
+        "mode": "live-fresh-process",
+        "variant": "canonical",
+        "ticks": 200
+      },
+      {
+        "id": "O1",
+        "mode": "offline-replay",
+        "artifact": "L1",
+        "ticks": 200
+      },
+      {
+        "id": "O2",
+        "mode": "offline-replay-fresh-process",
+        "artifact": "same immutable L1 bytes",
+        "ticks": 200
+      },
+      {
+        "id": "V",
+        "mode": "live",
+        "variant": "rejected-removed",
+        "ticks": 200
+      }
+    ]
+  },
+  "negative_replay_probe": {
+    "case": "first hash mismatch",
+    "artifact": "copy of L1, alter only expected hash at tick37",
+    "expected_first_divergent_tick": 37,
+    "must_show_canonical_record": true,
+    "count": 1
+  }
+}
diff --git a/track b/track
index b5039cc..667e29a 100755
--- a/track
+++ b/track
@@ -8,10 +8,14 @@ fi
 command=${1:-help}
 if [ "$#" -gt 0 ]; then shift; fi
 case "$command" in
-    build) exec ./gradlew --offline --no-daemon clean classes testClasses installDist resolveLockedDependencies "$@" ;;
+    build) exec ./gradlew --offline --no-daemon clean classes testClasses installDist resolveLockedDependencies writeTestRuntimeClasspath "$@" ;;
     unit-test) exec ./gradlew --offline --no-daemon test "$@" ;;
     integration-test) exec ./gradlew --offline --no-daemon integrationTest "$@" ;;
-    scenario-run|server) exec ./build/install/arena-java/bin/arena-java "$command" "$@" ;;
-    replay-verify) echo 'replay-verify is not activated until G07' >&2; exit 2 ;;
+    scenario-run|replay-verify)
+        java_bin=java
+        if [ -n "${JAVA_HOME:-}" ]; then java_bin="$JAVA_HOME/bin/java"; fi
+        exec "$java_bin" -Dio.netty.leakDetection.level=paranoid -cp "$(cat build/test-runtime-classpath.txt)" arena.ReplayTool "$command" "$@"
+        ;;
+    server) exec ./build/install/arena-java/bin/arena-java "$command" "$@" ;;
     *) echo 'usage: ./track build | unit-test | integration-test | scenario-run <scenario.json> <evidence.json> | replay-verify <replay> <evidence> | server <config.json>' >&2; exit 2 ;;
 esac
