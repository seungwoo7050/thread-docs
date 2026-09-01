# TCP Control과 UDP Data Plane (G09)

## `feat(udp): add bounded realtime data and independent endpoint binding`

diff --git a/TRACK.md b/TRACK.md
index de9c85e..2dc477a 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -1,6 +1,6 @@
-# Java arena — through G08
+# Java arena — through G09
 
-Current thread: G08 (G01–G07 regressions retained). Phase: phase-1. Profile: realtime-core. Spec revision: `c1d62196ab76b55652f5d75a67514f8c6d8210ce`.
+Current thread: G09 (G01–G08 regressions retained). Phase: phase-1. Profile: realtime-core. Spec revision: `c1d62196ab76b55652f5d75a67514f8c6d8210ce`.
 G01–G05 retain their original spec trailers and verification. The user-authorized spec/profile transition changes procedure only; the game contract is unchanged. Scope remains G01–G14, with no G15+, external infrastructure or push.
 
 ## Frozen versions
@@ -29,6 +29,7 @@ The wrapper uses the locally installed Temurin path when JAVA_HOME is unset. On
 ./track scenario-run /absolute/path/to/G07.json /absolute/path/to/L1.json
 ./track scenario-run /absolute/path/to/G07.json /absolute/path/to/V.json --variant rejected-removed
 ./track scenario-run /absolute/path/to/G08.json /absolute/path/to/snapshot-evidence.json
+./track scenario-run /absolute/path/to/G09.json /absolute/path/to/udp-evidence.json
 ./track replay-verify /absolute/path/to/replay /absolute/path/to/evidence
 ./track server config/server.json
 ```
@@ -123,6 +124,18 @@ Minimal authenticated TCP `SNAPSHOT_ACK` carries session/room/player identity an
 
 `scenario-run G08.json result.json` performs exactly one196-tick run with the shared test-only four-session bootstrap and writes the accepted-input replay beside the result. Real wire captures, ACKs, independent client application, full canonical records, retained base IDs and cleanup are retained in the result. HELLO/WELCOME observation markers drain frames without changing session/player identity or time. The full unit suite contains only one pure33-snapshot retention fixture (zero simulation ticks); the archived baseline and post-change196-tick campaign are not repeated there. Exact commands and raw locations are in `evidence/G08-command-ledger.jsonl`.
 
-G01 initial budget was build/compile <=8, unit suites <=4, integration suites <=2, canonical scenario <=1. Later Threads use their frozen active plans, including G07's explicit five-pass and negative-replay budget. Network-fault and load runs remain zero. No test sleep, microbenchmark, fuzzing, UDP, reconnect, many-room or distributed implementation is included.
+## G09 TCP control and UDP data
+
+The stage notes above describe their original ENDs. G09 retains TCP control and moves UDP_BIND/UDP_BOUND, INPUT/INPUT_ACK, SNAPSHOT/SNAPSHOT_ACK and PING/PONG to one bounded datagram channel. The server independently binds UDP port0 and advertises the actual `udp_port` in WELCOME; the ordinary client and fault proxy consume that advertisement. TCP's requested port is not a claim on the UDP port namespace. The existing constructor failure cleanup now catches checked bind failures as well as runtime exceptions and preserves the cause when reporting startup failure.
+
+One owner-confined session stores the server-issued one-time bind credential, its monotonic issue time and the observed UDP endpoint. TTL is 5,000ms, expiring at `now >= issued + TTL`. Binding can precede or follow join; the Room starts only when the minimum player count and every joined player's UDP readiness hold. Invalid, expired, reused, foreign-session, foreign-endpoint and nonzero-epoch attempts cannot change another binding or Room state. Identified failures use stable TCP ERROR; unroutable packets drop with a metric. TCP realtime messages return WRONG_TRANSPORT. No reconnect or distributed epoch management is active.
+
+The Netty UDP handler is not sharable and sends validated intent through the existing bounded owner mailbox. A 1,201-byte receive buffer detects over-limit datagrams; payloads above1,200 drop with an explicit metric before owner admission. Outbound serialization also rejects oversize with a metric and never truncates, fragments or omits players. Generated Room IDs have22 ASCII characters and Player IDs8; the eight-player worst visible FULL envelope is1,194bytes. Input identifiers retain the contract's1..64ASCII allowance; an oversized maximum-ID test fixture is explicitly rejected. Credentials remain in memory and are removed from client observations before logging or artifact creation.
+
+The canonical runner uses four ordinary unbound TCP joins with test-only identifier remapping, then four valid binds. Its single24-tick campaign drops original5, duplicates8 and delivers11before10 independently on alpha INPUT and SNAPSHOT streams. Only original messages increment fault indices; one pending packet is the bound, with no sleep or additional live campaign in the full suite. The actual accepted journal contains22 INPUT records and is verified by one24-tick offline pass. Both passes produce the same24 canonical records and hashes. The client applies11of13 received snapshots, ignoring duplicate8 and old10, without changing server authority.
+
+Initial G09 full-unit verification failed during one fixture startup; the exact uncommitted tree and raw logs were preserved before repair1. The fresh repair reproduced UDP allocation coupling with one reserved endpoint and zero ticks, then changed only allocation/advertisement use and the directly observed constructor cleanup boundary. The unchanged eleven-case matrix, byte probe and added reserved-port regression pass in the45-test suite; integration4/4, canonical24 and offline24 also pass. See `evidence/G09-verification.md`, `evidence/G09-repair1-provenance.json` and the separate initial/repair command ledgers for failure provenance, checksums and consumption.
+
+G01 initial budget was build/compile <=8, unit suites <=4, integration suites <=2, canonical scenario <=1. Later Threads use their frozen active plans, including G07's explicit five-pass and negative-replay budget. G09 executes one fixed network-fault campaign; load remains0. No test sleep, microbenchmark, fuzzing, reconnect, many-room or distributed implementation is included.
 
 JVM concurrency evidence uses owner-confinement assertions plus real cross-thread Netty handoff, actual thread joins and shutdown assertions. No JVM race detector is installed; no sanitizer result is claimed.
diff --git a/evidence/G09-command-ledger.jsonl b/evidence/G09-command-ledger.jsonl
new file mode 100644
index 0000000..3aac994
--- /dev/null
+++ b/evidence/G09-command-ledger.jsonl
@@ -0,0 +1,11 @@
+{"kind": "activation", "thread": "G09", "profile": "realtime-core", "spec_revision": "c1d62196ab76b55652f5d75a67514f8c6d8210ce", "start": "f38e98caee608007e86ca87067f001c63dc15be3", "attempt": "initial", "budget": {"compile": 8, "unit_including_baseline_and_matrix": 4, "integration": 2, "post_fault_canonical": 1, "accepted_journal_replay": 1, "load": 0}, "production_hash_manifest": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-initial/reproduce-unit/production-hashes-before.json"}
+{"kind": "resolved_before_execution", "pass": "baseline", "category": "unit-reproduction", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test", "--tests", "arena.G09BaselineTest"], "environment": {"ARENA_G09_SCENARIO": "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G09.json", "ARENA_G09_EVIDENCE": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-initial/reproduce-unit/result.json"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-initial/reproduce-unit", "reserved_ticks": 1, "resolved_at": "2026-08-28T05:00:53.222934+00:00"}
+{"kind": "resolved_before_execution", "pass": "build", "category": "build", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "build"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-initial/verify-build", "reserved_ticks": 0, "resolved_at": "2026-08-28T05:00:53.223067+00:00"}
+{"kind": "resolved_before_execution", "pass": "unit", "category": "unit-with-fixed-boundary-matrix", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-initial/verify-unit", "reserved_ticks": 0, "resolved_at": "2026-08-28T05:00:53.223079+00:00"}
+{"kind": "resolved_before_execution", "pass": "integration", "category": "integration", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "integration-test"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-initial/verify-integration", "reserved_ticks": 0, "resolved_at": "2026-08-28T05:00:53.223089+00:00"}
+{"kind": "resolved_before_execution", "pass": "canonical", "category": "canonical-fault", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "scenario-run", "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G09.json", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-initial/canonical/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-initial/canonical", "reserved_ticks": 24, "resolved_at": "2026-08-28T05:00:53.223097+00:00"}
+{"kind": "resolved_before_execution", "pass": "replay", "category": "offline-accepted-journal", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-initial/canonical/result.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-initial/replay/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-initial/replay", "reserved_ticks": 24, "resolved_at": "2026-08-28T05:00:53.223107+00:00"}
+{"pass": "baseline", "category": "unit-reproduction", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test", "--tests", "arena.G09BaselineTest"], "environment": {"ARENA_G09_SCENARIO": "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G09.json", "ARENA_G09_EVIDENCE": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-initial/reproduce-unit/result.json"}, "kind": "executed", "started_at": "2026-08-28T05:01:07.369579+00:00", "duration_seconds": 5.397, "command_process_id": 84462, "exit_code": 1, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-initial/reproduce-unit/stdout.log", "process_terminated": true, "compiler_tasks_executed": ["> Task :compileTestJava"], "xml": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-initial/reproduce-unit/xml", "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-initial/reproduce-unit/result.json", "simulation_process_id": 84589, "executed_ticks": 1}
+{"kind": "resolved_artifact_before_execution", "pass": "unit", "matrix_source": "src/test/java/arena/UdpBoundaryTest.java", "fixture_resource": "src/test/resources/G09.json", "emitted_result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/build/g09-matrix-result.json", "archived_result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-initial/verify-unit/matrix.json", "matrix_cases": 11, "actual_simulation_tick_calls": 0, "canonical_source": "src/test/java/arena/UdpScenario.java", "fault_proxy_source": "src/test/java/arena/UdpFaultProxy.java", "resolved_at": "2026-08-28T05:32:13.086807+00:00"}
+{"pass": "build", "category": "build", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "build"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T05:32:29.426800+00:00", "duration_seconds": 5.833, "command_process_id": 919, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-initial/verify-build/stdout.log", "process_terminated": true, "compiler_tasks_executed": ["> Task :compileJava", "> Task :compileTestJava"]}
+{"pass": "unit", "category": "unit-with-fixed-boundary-matrix", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T05:32:35.260881+00:00", "duration_seconds": 4.915, "command_process_id": 1000, "exit_code": 1, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-initial/verify-unit/stdout.log", "process_terminated": true, "compiler_tasks_executed": [], "xml": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-initial/verify-unit/xml", "matrix_result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-initial/verify-unit/matrix.json", "simulation_process_id": 1027, "matrix_cases": 11, "matrix_executed_ticks": 0}
diff --git a/evidence/G09-repair1-command-ledger.jsonl b/evidence/G09-repair1-command-ledger.jsonl
new file mode 100644
index 0000000..72eb6ab
--- /dev/null
+++ b/evidence/G09-repair1-command-ledger.jsonl
@@ -0,0 +1,15 @@
+{"kind": "activation", "thread": "G09", "attempt": "repair1", "profile": "realtime-core", "spec_revision": "c1d62196ab76b55652f5d75a67514f8c6d8210ce", "start": "f38e98caee608007e86ca87067f001c63dc15be3", "budget": {"compile": 8, "unit": 4, "integration": 2, "canonical_fault_24tick": 1, "offline_accepted_journal_24tick": 1, "load": 0}, "preservation_verified": {"working_files": 23, "raw_files": 21, "tracked_diff_sha256": "8ac3505a429d23668c1dd2113e0b4094817c382d083bba23b26da4773e03e8c5", "manifest_sha256": "d5631a4855c50345fa6ceb899ecd03ea829087320f07eceb9eb8e34ee233ab97"}, "initial_consumption": {"baseline_unit": 1, "post_build": 1, "post_unit": 1, "actual_compile_tasks": 3, "integration": 0, "canonical": 0, "offline": 0, "load": 0}, "production_hash_manifest": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/reproduce-port/production-hashes-before.json", "baseline_test_source_sha256": "35d55f9ca969a4a3bb3d3c1400f043080597b161a0e4a8b740b500ae6055e271", "runner_sha256": "9999984275d18d2a6ef3f726eeade90f3c12412518dae26fd2263423209b8762"}
+{"kind": "resolved_before_execution", "pass": "reproduction", "category": "unit-reproduction", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test", "--tests", "arena.UdpBoundaryTest.reservedUdpPortDoesNotBlockTcpStartup"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/reproduce-port", "reserved_g09_ticks": 0, "resolved_at": "2026-08-28T05:41:34.057797+00:00"}
+{"kind": "resolved_before_execution", "pass": "build", "category": "build", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "build"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/verify-build", "reserved_g09_ticks": 0, "resolved_at": "2026-08-28T05:41:34.057797+00:00"}
+{"kind": "resolved_before_execution", "pass": "unit", "category": "unit-with-fixed-boundary-matrix", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/verify-unit", "reserved_g09_ticks": 0, "resolved_at": "2026-08-28T05:41:34.057797+00:00"}
+{"kind": "resolved_before_execution", "pass": "integration", "category": "integration", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "integration-test"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/verify-integration", "reserved_g09_ticks": 0, "resolved_at": "2026-08-28T05:41:34.057797+00:00"}
+{"kind": "resolved_before_execution", "pass": "canonical", "category": "canonical-fault", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "scenario-run", "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G09.json", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/canonical/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/canonical", "reserved_g09_ticks": 24, "resolved_at": "2026-08-28T05:41:34.057797+00:00"}
+{"kind": "resolved_before_execution", "pass": "replay", "category": "offline-accepted-journal", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/canonical/result.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/replay/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/replay", "reserved_g09_ticks": 24, "resolved_at": "2026-08-28T05:41:34.057797+00:00"}
+{"pass": "reproduction", "category": "unit-reproduction", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test", "--tests", "arena.UdpBoundaryTest.reservedUdpPortDoesNotBlockTcpStartup"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T05:41:48.345659+00:00", "finished_at": "2026-08-28T05:41:53.961462+00:00", "duration_seconds": 5.616, "command_process_id": 6987, "exit_code": 1, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/reproduce-port/stdout.log", "process_terminated": true, "process_ceiling_seconds": 120, "timed_out": false, "compiler_tasks_executed": ["> Task :compileTestJava"], "xml": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/reproduce-port/xml", "unit_or_integration_counts": {"tests": 1, "failures": 1, "errors": 0, "skipped": 0}, "port-allocation": {"path": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/reproduce-port/port-allocation.json", "process_id": 7122, "executed_ticks": 0, "passed": false}, "production_unchanged": true, "stdout_sha256": "73c0b952ce08855b9d7e723de3d77be0ce3c6fa5a80fd738de7ac774d9c20f54"}
+{"kind": "limited_repair", "timestamp": "2026-08-28T05:44:10.610645+00:00", "changed_from_preserved_initial": {"src/main/java/arena/ArenaServer.java": {"before_sha256": "dbbbba3343630b38d001b21e0f03d4f88c6997445a31eb7df47133eac6dff88c", "after_sha256": "4ed51b3fdd274b97598593df0b233c43cb7da42cfdea365a5973a7f3362dcf04"}, "src/main/java/arena/TcpClient.java": {"before_sha256": "09e23cfb9408df7dbfe6ed04dae7411c4f02356b1404d35e07fc244938fba357", "after_sha256": "7fd1bb1f570f8e159f03a6168d79491e4811ce6e29ea89a4f2e75c41698ced22"}, "src/test/java/arena/UdpBoundaryTest.java": {"before_sha256": "413d0604038adcaa780948eb3c13373e239209d4046a9e814b50852a77f19d90", "after_sha256": "35d55f9ca969a4a3bb3d3c1400f043080597b161a0e4a8b740b500ae6055e271"}, "src/test/java/arena/UdpScenario.java": {"before_sha256": "ee83601c856141bf64fdd6e3de375356b721c79e362105a4834f06abfa278ddc", "after_sha256": "8f4eab27daf72146392cecc98a8bed7b6633d9a6f15445b9fd117ede002b7200"}}, "repair_patch": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/repair.diff", "repair_patch_sha256": "054c2eec3940968fc9c905e7c0624acdbe98e4966288bae4b6fc060247c17d7f", "original_eleven_case_and_byte_probe_unchanged": true, "original_raw_files_unchanged": 21, "constructor_cleanup_scope": "root explicitly included existing cleanup catch(Exception) for observed BindException; retain original stack and two live event-loop threads until failed JVM exit", "additional_test_or_scenario_runs": 0}
+{"pass": "build", "category": "build", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "build"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T05:44:19.132616+00:00", "finished_at": "2026-08-28T05:44:25.218912+00:00", "duration_seconds": 6.086, "command_process_id": 8628, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/verify-build/stdout.log", "process_terminated": true, "process_ceiling_seconds": 120, "timed_out": false, "compiler_tasks_executed": ["> Task :compileJava", "> Task :compileTestJava"], "stdout_sha256": "684b1b9e91a57ec8bb0d5a91e6bc27b354cd5591e553e45432343bdb2ca5858a"}
+{"pass": "unit", "category": "unit-with-fixed-boundary-matrix", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T05:44:32.901930+00:00", "finished_at": "2026-08-28T05:44:37.092276+00:00", "duration_seconds": 4.19, "command_process_id": 8743, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/verify-unit/stdout.log", "process_terminated": true, "process_ceiling_seconds": 120, "timed_out": false, "compiler_tasks_executed": [], "xml": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/verify-unit/xml", "unit_or_integration_counts": {"tests": 45, "failures": 0, "errors": 0, "skipped": 0}, "port-allocation": {"path": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/verify-unit/port-allocation.json", "process_id": 8778, "executed_ticks": 0, "passed": true}, "matrix": {"path": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/verify-unit/matrix.json", "process_id": 8778, "executed_ticks": 0, "passed": true}, "stdout_sha256": "87caf2bc94ef075e14f1ea9384b37e0b17690aa45802ba6087233ac320ff12f2"}
+{"pass": "integration", "category": "integration", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "integration-test"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T05:45:00.604412+00:00", "finished_at": "2026-08-28T05:45:06.047749+00:00", "duration_seconds": 5.443, "command_process_id": 9207, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/verify-integration/stdout.log", "process_terminated": true, "process_ceiling_seconds": 120, "timed_out": false, "compiler_tasks_executed": [], "xml": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/verify-integration/xml", "unit_or_integration_counts": {"tests": 4, "failures": 0, "errors": 0, "skipped": 0}, "stdout_sha256": "b81bb3390f4865ef54ab200b23643c0291e78a1a1f17b82b8977f410b7f1c74a"}
+{"pass": "canonical", "category": "canonical-fault", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "scenario-run", "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G09.json", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/canonical/result.json"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T05:45:20.507905+00:00", "finished_at": "2026-08-28T05:45:21.709341+00:00", "duration_seconds": 1.201, "command_process_id": 9430, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/canonical/stdout.log", "process_terminated": true, "process_ceiling_seconds": 120, "timed_out": false, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/canonical/result.json", "simulation_process_id": 9430, "executed_ticks": 24, "passed": true, "stdout_sha256": "a896dae754fffa95e7692e718de97b65390852a09414397d0d5b26e11d86c710"}
+{"pass": "replay", "category": "offline-accepted-journal", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/canonical/result.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/replay/result.json"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T05:45:37.300416+00:00", "finished_at": "2026-08-28T05:45:37.653727+00:00", "duration_seconds": 0.353, "command_process_id": 9576, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/replay/stdout.log", "process_terminated": true, "process_ceiling_seconds": 120, "timed_out": false, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/replay/result.json", "simulation_process_id": 9576, "executed_ticks": 24, "passed": true, "stdout_sha256": "de89927693dd07939ef65ecd5a11ac613710c1a9122060d963bae8bc04dc49a6"}
+{"kind": "process-exit-inspection", "category": "read-only-no-runtime-pass", "timestamp": "2026-08-28T05:48:16.193501+00:00", "argv": ["ps", "-p", "7122,8778,9249,9250,9430,9576", "-o", "pid=,comm="], "exit_code": 1, "stdout": "", "stderr": "", "expected_exit_code_when_all_absent": 1, "all_recorded_jvms_absent": true}
diff --git a/evidence/G09-repair1-provenance.json b/evidence/G09-repair1-provenance.json
new file mode 100644
index 0000000..b3d334c
--- /dev/null
+++ b/evidence/G09-repair1-provenance.json
@@ -0,0 +1,618 @@
+{
+  "thread": "G09",
+  "profile": "realtime-core",
+  "spec_revision": "c1d62196ab76b55652f5d75a67514f8c6d8210ce",
+  "attempt": "repair1",
+  "branch": "track/industry-java",
+  "start": "f38e98caee608007e86ca87067f001c63dc15be3",
+  "status": "WORKER_VERIFICATION_PASS_ROOT_GATE_PENDING",
+  "raw_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1",
+  "scenario_sha256": "9519b687243a622f4e63675bb3690e8900bf7616337647f18de73e1bcc27edfa",
+  "initial_preservation": {
+    "manifest_path": "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/evidence/G09/industry-java/initial/failed-working-tree/manifest.json",
+    "manifest_sha256": "d5631a4855c50345fa6ceb899ecd03ea829087320f07eceb9eb8e34ee233ab97",
+    "working_tree_files": 23,
+    "raw_files": 21,
+    "tracked_diff_sha256": "8ac3505a429d23668c1dd2113e0b4094817c382d083bba23b26da4773e03e8c5",
+    "original_ledger_sha256": "b4da02df311b2c1d85986f393ef76863ecc510eca4e49fc2380e33f9f61528c3",
+    "consumption": {
+      "baseline_unit": 1,
+      "post_build": 1,
+      "post_unit": 1,
+      "actual_compile_tasks": 3,
+      "integration": 0,
+      "canonical": 0,
+      "offline": 0,
+      "load": 0
+    },
+    "full_unit_counts": {
+      "tests": 44,
+      "failures": 1,
+      "errors": 0,
+      "skipped": 0
+    },
+    "failure": "expired-token-at-deadline failed during fixture startup: Address already in use",
+    "initial_stack_unavailable": true
+  },
+  "repair_reproduction": {
+    "command": [
+      "./track",
+      "unit-test",
+      "--tests",
+      "arena.UdpBoundaryTest.reservedUdpPortDoesNotBlockTcpStartup"
+    ],
+    "exit_code": 1,
+    "executed_ticks": 0,
+    "exception": "java.net.BindException",
+    "observed_bind_path": "DatagramChannelImpl.bind -> NioDatagramChannel.doBind",
+    "reservation_established": true,
+    "reservation_closed": true,
+    "arena_threads_alive_before_failed_jvm_exit": 2,
+    "production_unchanged": true,
+    "stack_path": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/reproduce-port/port-allocation.json"
+  },
+  "limited_fix": {
+    "production": [
+      "ArenaServer UDP bind(host,0), independent of TCP requested/assigned port",
+      "TcpClient initial WELCOME udp_port consumption; no inferred TCP=UDP endpoint",
+      "existing ArenaServer startup cleanup catches Exception and preserves checked BindException cause"
+    ],
+    "test_harness": "UdpScenario fault proxy uses the client advertised endpoint; original eleven-case matrix and byte probe unchanged",
+    "additional_regression": "same zero-tick reserved UDP method, run only before production edit and in full unit",
+    "cleanup_scope_authorized_by_root": true,
+    "post_failure_cleanup_campaigns": 0,
+    "failure_cleanup_path_validation": "static check against captured checked BindException; permitted post runtime exercises successful startup and normal shutdown"
+  },
+  "verification": [
+    {
+      "pass": "reproduction",
+      "argv": [
+        "./track",
+        "unit-test",
+        "--tests",
+        "arena.UdpBoundaryTest.reservedUdpPortDoesNotBlockTcpStartup"
+      ],
+      "exit_code": 1,
+      "compiler_tasks": [
+        "> Task :compileTestJava"
+      ],
+      "test_counts": {
+        "tests": 1,
+        "failures": 1,
+        "errors": 0,
+        "skipped": 0
+      },
+      "executed_g09_ticks": 0
+    },
+    {
+      "pass": "build",
+      "argv": [
+        "./track",
+        "build"
+      ],
+      "exit_code": 0,
+      "compiler_tasks": [
+        "> Task :compileJava",
+        "> Task :compileTestJava"
+      ],
+      "test_counts": null,
+      "executed_g09_ticks": 0
+    },
+    {
+      "pass": "unit",
+      "argv": [
+        "./track",
+        "unit-test"
+      ],
+      "exit_code": 0,
+      "compiler_tasks": [],
+      "test_counts": {
+        "tests": 45,
+        "failures": 0,
+        "errors": 0,
+        "skipped": 0
+      },
+      "executed_g09_ticks": 0
+    },
+    {
+      "pass": "integration",
+      "argv": [
+        "./track",
+        "integration-test"
+      ],
+      "exit_code": 0,
+      "compiler_tasks": [],
+      "test_counts": {
+        "tests": 4,
+        "failures": 0,
+        "errors": 0,
+        "skipped": 0
+      },
+      "executed_g09_ticks": 0
+    },
+    {
+      "pass": "canonical",
+      "argv": [
+        "./track",
+        "scenario-run",
+        "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G09.json",
+        "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/canonical/result.json"
+      ],
+      "exit_code": 0,
+      "compiler_tasks": [],
+      "test_counts": null,
+      "executed_g09_ticks": 24
+    },
+    {
+      "pass": "replay",
+      "argv": [
+        "./track",
+        "replay-verify",
+        "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/canonical/result.replay.jsonl",
+        "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/replay/result.json"
+      ],
+      "exit_code": 0,
+      "compiler_tasks": [],
+      "test_counts": null,
+      "executed_g09_ticks": 24
+    }
+  ],
+  "repair_budget_consumed": {
+    "compiler_tasks": 3,
+    "unit_including_reproduction": 2,
+    "integration": 1,
+    "canonical_fault_campaign": 1,
+    "canonical_ticks": 24,
+    "offline_pass": 1,
+    "offline_ticks": 24,
+    "load": 0
+  },
+  "matrix": {
+    "cases": [
+      {
+        "name": "valid-token-before-deadline",
+        "passed": true,
+        "response": "UDP_BOUND"
+      },
+      {
+        "name": "expired-token-at-deadline",
+        "passed": true,
+        "response": "UDP_BIND_INVALID"
+      },
+      {
+        "name": "unknown-token",
+        "passed": true,
+        "response": "UDP_BIND_INVALID"
+      },
+      {
+        "name": "reused-consumed-token",
+        "passed": true,
+        "response": "UDP_BIND_INVALID"
+      },
+      {
+        "name": "other-session-token",
+        "passed": true,
+        "response": "UDP_BIND_INVALID"
+      },
+      {
+        "name": "INPUT-before-bind",
+        "passed": true,
+        "response": "UDP_BIND_INVALID"
+      },
+      {
+        "name": "INPUT-from-different-observed-endpoint",
+        "passed": true,
+        "response": "UDP_BIND_INVALID"
+      },
+      {
+        "name": "wrong-owner-epoch-bind",
+        "passed": true,
+        "response": "UDP_BIND_INVALID"
+      },
+      {
+        "name": "realtime-INPUT-over-TCP",
+        "passed": true,
+        "response": "WRONG_TRANSPORT"
+      },
+      {
+        "name": "valid-INPUT-exactly1200bytes",
+        "passed": true,
+        "response": "INPUT_ACK"
+      },
+      {
+        "name": "oversize-datagram1201bytes",
+        "passed": true,
+        "response": "PONG"
+      }
+    ],
+    "executed_ticks": 0,
+    "maximum_eight_player_envelope_bytes": 1194,
+    "oversized_fixture_bytes": 1684,
+    "oversized_fixture_rejected": true
+  },
+  "post_reserved_udp": {
+    "passed": true,
+    "executed_ticks": 0,
+    "requested_tcp_number_preserved": true,
+    "udp_allocated_independently": true,
+    "welcome_advertises_actual_udp_port": true,
+    "udp_bound": true,
+    "arena_threads_alive_after_test": 0
+  },
+  "canonical": {
+    "accepted_sequences": [
+      1,
+      2,
+      3,
+      4,
+      6,
+      7,
+      8,
+      9,
+      11,
+      12,
+      13,
+      14,
+      15,
+      16,
+      17,
+      18,
+      19,
+      20,
+      21,
+      22,
+      23,
+      24
+    ],
+    "duplicate_ack_count": 1,
+    "stale_error_count": 1,
+    "received_snapshot_ordinals": [
+      1,
+      2,
+      3,
+      4,
+      6,
+      7,
+      8,
+      8,
+      9,
+      11,
+      10,
+      12,
+      13
+    ],
+    "snapshot_counts": {
+      "alpha": {
+        "received": 13,
+        "applied": 11,
+        "highest_applied": 13
+      },
+      "bravo": {
+        "received": 13,
+        "applied": 13,
+        "highest_applied": 13
+      },
+      "charlie": {
+        "received": 13,
+        "applied": 13,
+        "highest_applied": 13
+      },
+      "delta": {
+        "received": 13,
+        "applied": 13,
+        "highest_applied": 13
+      }
+    },
+    "final_state": {
+      "v": 1,
+      "type": "SNAPSHOT",
+      "room_id": "room-fixture",
+      "status": "RUNNING",
+      "tick": 23,
+      "executed_ticks": 24,
+      "players": [
+        {
+          "player_id": "player-00",
+          "slot": 0,
+          "x": 19600,
+          "y": 10000,
+          "direction": "EAST",
+          "score": 0,
+          "last_accepted_seq": 24,
+          "applied_seq": 24,
+          "connectivity": "CONNECTED",
+          "last_tag_tick": -20,
+          "pending_inputs": 0
+        },
+        {
+          "player_id": "player-01",
+          "slot": 1,
+          "x": 90000,
+          "y": 90000,
+          "direction": "STOP",
+          "score": 0,
+          "last_accepted_seq": 0,
+          "applied_seq": null,
+          "connectivity": "CONNECTED",
+          "last_tag_tick": -20,
+          "pending_inputs": 0
+        },
+        {
+          "player_id": "player-02",
+          "slot": 2,
+          "x": 10000,
+          "y": 90000,
+          "direction": "STOP",
+          "score": 0,
+          "last_accepted_seq": 0,
+          "applied_seq": null,
+          "connectivity": "CONNECTED",
+          "last_tag_tick": -20,
+          "pending_inputs": 0
+        },
+        {
+          "player_id": "player-03",
+          "slot": 3,
+          "x": 90000,
+          "y": 10000,
+          "direction": "STOP",
+          "score": 0,
+          "last_accepted_seq": 0,
+          "applied_seq": null,
+          "connectivity": "CONNECTED",
+          "last_tag_tick": -20,
+          "pending_inputs": 0
+        }
+      ],
+      "state_hash": "b4b832ff24af230655c92bb461c85aceddd2835799a53dc4b05b966f6b9d7b14"
+    },
+    "artifact_sha256": "3c33f5374e59854b8295a00fd3fece7c4352db6829193781f50aeb2bed847076",
+    "artifact_bytes": 17919,
+    "state_hashes": [
+      "c633c02fe7e23da335c52ff62b99f14e393092e4c6ccc95bb6f62c31fab5a5bc",
+      "6d4564c285b4cf10b4e2a77fb75ee5b371940ce88ae34f690f529173a668b602",
+      "1a5928d31981635ace53771142d528e7ac294a9df00ffcbb9142bc23f19c30aa",
+      "f09981fee9a215fe29ba2352e59b81925cbf3f59b0a74690a887e0576377c6fd",
+      "0adcae3305923a4b918abdd7f209b50380c05f6c5f11f521efcd745cbb38a171",
+      "1d96d955e014b20f479ca6884722c83931404f83eb893544d7ccd959e79a8b2f",
+      "b4754fb4815f94d2cbb4186d9ca473a3890e5d7414042c26c58ee30ec008d362",
+      "674858d4e80df6dbf33a071c06a8915cd663634ba066116d85877e2115c91e09",
+      "d1132263c29c614a35cff16ee7c3ddfe9f973824481410310ae8844ae983557f",
+      "29782a939bff971cd33da0eb939cd8a45f309da6f341f6fccdf6b0366c0a5f00",
+      "6ff8cb480cff3ad5f1279078abc5b40b3f0639cbd7c9c8eac5e36f4f9b8e3813",
+      "e0f0728a4276e1aa64933cc7387ec24a8dfa235315251f48bcdf418b91f03079",
+      "b7ac15bceb1c63a2418abf78abdda5efb08f7b0188df5915fd00fb0467685544",
+      "079b20f055c3e40a8a538023a9b0f6e01356565c6f7c04bc2c351ce3234682b0",
+      "de198f69249dbc1d0fc2069da0bf893cd5979162c8313a7ed77e565fed362ccb",
+      "b7a89bb5726f4ee96b8c8832b347996ea9c455da99b2734495b6b9d5a7265e9d",
+      "eb60bbe170b78c3761afeefb4a82e58c581bf07987d6d55b0481c0b39a2d8a54",
+      "a5862b473f974940f0f7e100c51a1136465e629f13c314bc86d290362640b3be",
+      "991fc67339422bd33792cc54ce7a52ca072a505eb5a3177f0a4b05991fdbefe6",
+      "edeb55ce8d684d212391b97d9a91e1167c784d81a0f1757c56fb5206d3223dc7",
+      "96fb5bc6fdf2d82355ec668181483a1ae1634945892753719438c4b066687831",
+      "bc67cfe5eb96d74f55c063017872c838552cede0e02fcfaac15889e16a07176b",
+      "d10796bdc28dec96e7475e944cef6b7a5ec90c8e0ab171dbdd8f6ce06bb69e0a",
+      "b4b832ff24af230655c92bb461c85aceddd2835799a53dc4b05b966f6b9d7b14"
+    ],
+    "hash_sequence_sha256": "a20d7dac08fc5a7f702f8051c03bfeb84a0d88622c19504c23151928d2c50b98",
+    "offline_hashes_equal": true,
+    "offline_canonical_records_equal": true
+  },
+  "resource_evidence": {
+    "canonical_cleanup": {
+      "open_channels": 0,
+      "connections": 0,
+      "pending_writes": 0,
+      "mailbox_remaining": 0,
+      "live_threads": 0,
+      "timer_alive": false,
+      "owner_terminated": true,
+      "event_loops_terminated": true,
+      "pending_input_high_water": 1,
+      "replay_bytes": 0,
+      "retained_snapshots": 0,
+      "snapshot_retention_high_water": 13,
+      "mailbox_high_water": 2,
+      "outbound_high_water": 2,
+      "parser": {
+        "live_buffers": 0,
+        "allocated_bytes": 0,
+        "buffer_high_water": 109,
+        "capacity_high_water": 128,
+        "message_errors": 0,
+        "framing_errors": 0,
+        "partial_eofs": 0,
+        "io_errors": 0
+      },
+      "udp": {
+        "received": 78,
+        "dispatched": 78,
+        "oversize_dropped": 0,
+        "invalid": 0,
+        "unroutable": 0,
+        "mailbox_dropped": 0,
+        "outbound_oversize_rejected": 0,
+        "live_buffers": 0,
+        "allocated_bytes": 0,
+        "input_high_water": 214,
+        "output_high_water": 711,
+        "io_errors": 0
+      },
+      "clock": {
+        "active": false,
+        "last_monotonic_ns": 1200000000,
+        "last_iteration_ticks": 0,
+        "max_ticks_per_iteration": 1,
+        "accumulator_ns": 0,
+        "remaining_ms": 0,
+        "due_backlog_ticks": 0,
+        "overloaded": false,
+        "operational_state": "STOPPED"
+      },
+      "room_lifecycle": [
+        "LOBBY",
+        "RUNNING",
+        "CLOSED"
+      ]
+    },
+    "proxy_cleanup": {
+      "front_socket_closed": true,
+      "back_socket_closed": true,
+      "pending_packets": 0,
+      "live_threads": 0
+    },
+    "all_resources_released": true,
+    "failed_and_post_jvms_absent": true,
+    "runtime_high_water": {
+      "pending_inputs": 1,
+      "snapshot_retention_per_client": 13,
+      "datagram_in": 214,
+      "datagram_out": 711
+    }
+  },
+  "raw_checksums": {
+    "canonical/result.json": {
+      "sha256": "5e7140da263594dfeb2af659f0dbbfc120f96e565efbcda96bded39c19e42808",
+      "bytes": 272324
+    },
+    "canonical/result.replay.jsonl": {
+      "sha256": "3c33f5374e59854b8295a00fd3fece7c4352db6829193781f50aeb2bed847076",
+      "bytes": 17919
+    },
+    "canonical/stdout.log": {
+      "sha256": "a896dae754fffa95e7692e718de97b65390852a09414397d0d5b26e11d86c710",
+      "bytes": 143
+    },
+    "process-exit-inspection.json": {
+      "sha256": "f7c47bc56b3a247f30bbb16663b736b0a4a8c9fdced1c94770757c9d9f7e83b5",
+      "bytes": 364
+    },
+    "production-hashes-verified.json": {
+      "sha256": "c116dd5c5277dafc016d6f232d076fc991941585cc7ff5c1d37deb931f17e4b1",
+      "bytes": 11126
+    },
+    "repair.diff": {
+      "sha256": "054c2eec3940968fc9c905e7c0624acdbe98e4966288bae4b6fc060247c17d7f",
+      "bytes": 7979
+    },
+    "replay/result.json": {
+      "sha256": "791a7ca6e308990e768a77041fffd9ac39be33048d8e2cc28043bbbbc0920f3e",
+      "bytes": 20067
+    },
+    "replay/stdout.log": {
+      "sha256": "de89927693dd07939ef65ecd5a11ac613710c1a9122060d963bae8bc04dc49a6",
+      "bytes": 141
+    },
+    "reproduce-port/UdpBoundaryTest.java.before-production-edit": {
+      "sha256": "35d55f9ca969a4a3bb3d3c1400f043080597b161a0e4a8b740b500ae6055e271",
+      "bytes": 20159
+    },
+    "reproduce-port/port-allocation.json": {
+      "sha256": "2ce045c35407ce2f27a47b023b114852faded39d1947833acb0a8b6433f7042e",
+      "bytes": 2419
+    },
+    "reproduce-port/production-hashes-after.json": {
+      "sha256": "82e325d5f37ddf2c20521b8ed47b6f6a68136e1ac37160b7de1795c392b2a0fe",
+      "bytes": 9337
+    },
+    "reproduce-port/production-hashes-before.json": {
+      "sha256": "0c12e9c0fd6312ac1cdd7f9f6cb6a8b68c146de7c937f8f78b343407875e2531",
+      "bytes": 7153
+    },
+    "reproduce-port/stdout.log": {
+      "sha256": "73c0b952ce08855b9d7e723de3d77be0ce3c6fa5a80fd738de7ac774d9c20f54",
+      "bytes": 3103
+    },
+    "reproduce-port/xml/TEST-arena.UdpBoundaryTest.xml": {
+      "sha256": "b948a0c76139e5451c77bb370e2885b819235308547b606efeedde29ecfcf94d",
+      "bytes": 4852
+    },
+    "reproduce-port/xml/binary/output.bin": {
+      "sha256": "d82e8dfe0c6c5e1ac976570def58739ea0e50c4a2a16d243f6ccc05df5c395af",
+      "bytes": 2364
+    },
+    "reproduce-port/xml/binary/output.bin.idx": {
+      "sha256": "316e6cf5e786731386664e737d7235d2c405e5b86067adebd99883f412db69c0",
+      "bytes": 36
+    },
+    "reproduce-port/xml/binary/results.bin": {
+      "sha256": "6e1c821e8427728afaebf4cd45820dbcae995491fcdff92533966988d325572b",
+      "bytes": 2173
+    },
+    "run_commands.py": {
+      "sha256": "9999984275d18d2a6ef3f726eeade90f3c12412518dae26fd2263423209b8762",
+      "bytes": 4395
+    },
+    "verify-build/stdout.log": {
+      "sha256": "684b1b9e91a57ec8bb0d5a91e6bc27b354cd5591e553e45432343bdb2ca5858a",
+      "bytes": 616
+    },
+    "verify-integration/stdout.log": {
+      "sha256": "b81bb3390f4865ef54ab200b23643c0291e78a1a1f17b82b8977f410b7f1c74a",
+      "bytes": 929
+    },
+    "verify-integration/xml/TEST-arena.ServerIntegrationTest.xml": {
+      "sha256": "908f8d9499dcc92b21a38a16c54943b1588855fd2fc23a042be084f8ab1c2f47",
+      "bytes": 4290
+    },
+    "verify-integration/xml/binary/output.bin": {
+      "sha256": "8e9de0e0f227220f522eaa02218355f1a264c251f2d763f180bb85b0df58d12b",
+      "bytes": 3473
+    },
+    "verify-integration/xml/binary/output.bin.idx": {
+      "sha256": "0d0d8315329b019bd3cf1703a217edc99cffec84eb594fca9d84d35a5ae8c311",
+      "bytes": 102
+    },
+    "verify-integration/xml/binary/results.bin": {
+      "sha256": "7fef8703905364f071cedc9f75055f2f160d3cdea17bd379b47a360e7b420454",
+      "bytes": 567
+    },
+    "verify-unit/matrix.json": {
+      "sha256": "be636a699ee52d1764de1fe0da4fdf8da11c2907678a20a3ed3fe26819efc7f0",
+      "bytes": 58732
+    },
+    "verify-unit/port-allocation.json": {
+      "sha256": "07cdad73eea446fe22dbd786219ed08f092c9e611e3f794f54a1ad0edc68ec07",
+      "bytes": 2739
+    },
+    "verify-unit/stdout.log": {
+      "sha256": "87caf2bc94ef075e14f1ea9384b37e0b17690aa45802ba6087233ac320ff12f2",
+      "bytes": 3682
+    },
+    "verify-unit/xml/TEST-arena.CompleteFrameTest.xml": {
+      "sha256": "d2da45477f98adb726f648670928244987abacf7431198d5b247d75f9204ec14",
+      "bytes": 9483
+    },
+    "verify-unit/xml/TEST-arena.ReplayFormatTest.xml": {
+      "sha256": "8ee2b36608a72f492f6a54823f1d03d00d482e546e5c09fb32a2fb87e7d4c290",
+      "bytes": 1017
+    },
+    "verify-unit/xml/TEST-arena.RoomTest.xml": {
+      "sha256": "3bf8fb74a0ed03ae5a76ca4fc89b056e69f881cf08c379bec0f97c6f06170c65",
+      "bytes": 65963
+    },
+    "verify-unit/xml/TEST-arena.SnapshotStreamTest.xml": {
+      "sha256": "91221975c359b4e14b9d362b2871810091fdc08782c39976d89ff491937363a4",
+      "bytes": 2487
+    },
+    "verify-unit/xml/TEST-arena.UdpBoundaryTest.xml": {
+      "sha256": "c8955408a9c91723a132c40c33daf07b1bc47d4e7e553ef0f62244fac537f86e",
+      "bytes": 40211
+    },
+    "verify-unit/xml/binary/output.bin": {
+      "sha256": "d163e2aab9e6c23c93a1cb2f0c47b37004db9c65ab45e32cc460259723ee3ce0",
+      "bytes": 113523
+    },
+    "verify-unit/xml/binary/output.bin.idx": {
+      "sha256": "48c570c38a119a3f2acf6306758055aaa2084cbf08a395a6ca13fff732669d53",
+      "bytes": 1199
+    },
+    "verify-unit/xml/binary/results.bin": {
+      "sha256": "4807363d1e160bf43830be0b30fb233aae6063636559b2ed6854399fa69efc4c",
+      "bytes": 3632
+    }
+  },
+  "unresolved": [
+    "Independent orchestrator verification and cross-track acceptance are pending."
+  ]
+}
diff --git a/evidence/G09-verification.md b/evidence/G09-verification.md
new file mode 100644
index 0000000..8749619
--- /dev/null
+++ b/evidence/G09-verification.md
@@ -0,0 +1,33 @@
+# G09 verification — initial attempt and bounded repair1
+
+Profile: realtime-core. Spec revision: `c1d62196ab76b55652f5d75a67514f8c6d8210ce`.
+Parent START: `f38e98caee608007e86ca87067f001c63dc15be3`. The submitting G09 commit contains the passing implementation; no failing intermediate commit or history rewrite was made.
+
+The canonical input remains `index/scenarios/G09.json`, SHA-256 `9519b687243a622f4e63675bb3690e8900bf7616337647f18de73e1bcc27edfa`. No token, fixture, fault index, tick schedule, byte bound, readiness rule or acceptance threshold was changed during repair.
+
+## Provenance and reproduction
+
+The initial attempt's original ledger is preserved byte-for-byte in `G09-command-ledger.jsonl`. Its complete failed tree was archived by the orchestrator at `/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/evidence/G09/industry-java/initial/failed-working-tree/`. Before editing, repair1 verified all23 archived/current changed files, all21 archived/original raw files and the exact tracked patch checksum.
+
+The initial expired-token cell reported `Address already in use` during server construction, before any expiry observation. Its captured exception lacked a stack. Repair1 therefore made no claim that the expiry rule was defective: the sole pre-change regression reserved one real loopback UDP endpoint and requested the same number for TCP. The unchanged constructor failed at `DatagramChannelImpl.bind` / `NioDatagramChannel.doBind` with `java.net.BindException`. This is an actual UDP bind failure, not an unsupported command or fabricated observation. Production source/build/config hashes remained unchanged across this zero-tick run.
+
+That failed constructor left two event-loop threads alive inside its disposable test JVM; the original `catch (RuntimeException)` did not intercept Netty's checked `BindException`. The failed JVM subsequently exited, confirmed by the process inspection. The orchestrator explicitly included the directly related cleanup boundary in repair scope. The existing catch now handles `Exception`, executes the same cleanup and preserves the original cause. This catch change is source-verified against the captured exception; no extra post-fix failure campaign was run. Normal cleanup is covered by the permitted regression, matrix, integration and canonical checks.
+
+## Verification and immutable evidence
+
+`G09-repair1-command-ledger.jsonl` records every resolved and executed argv, timestamp, duration, exit, compiler task, raw path and failure for repair1. The raw directory is `/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g09-repair1/`. Raw logs, XML, accepted replay and complete canonical records remain outside the commit.
+
+| Repair1 command | Result |
+|---|---|
+| `./track unit-test --tests arena.UdpBoundaryTest.reservedUdpPortDoesNotBlockTcpStartup` | Expected pre-change failure; zero ticks; UDP bind stack preserved |
+| `./track build` | PASS; clean production/test compile |
+| `./track unit-test` | PASS;45 tests, no failures/errors/skips; original11-case matrix and added regression |
+| `./track integration-test` | PASS;4 tests, normal1200-tick session, timer and CLI SIGTERM cleanup |
+| `./track scenario-run` with the unchanged absolute G09 input | PASS; one24-tick fault campaign |
+| `./track replay-verify` on that campaign's actual accepted journal | PASS; one24-tick offline pass |
+
+The compact machine-readable `G09-repair1-provenance.json` contains exact input/artifact/raw checksums, all24 state hashes, full command arguments, matrix outcomes, final state, budget consumption and resource results. Live and offline canonical records and hash arrays are identical. The final hash is `b4b832ff24af230655c92bb461c85aceddd2835799a53dc4b05b966f6b9d7b14`.
+
+Initial consumption remains compiler tasks3, unit2 (one baseline and one failing full suite), integration0, canonical0, offline0. Repair1 consumes compiler tasks3/8, unit2/4 including its failing reproduction, integration1/2, canonical1/1 and offline1/1. Both attempts have load0. The fixed matrix runs once in each full-unit attempt; neither suite duplicates the24-tick fault campaign. Unused allowances were not spent.
+
+Worker verification passed. Independent orchestrator verification and cross-track acceptance remain pending. No tags, push, future-stage implementation or external infrastructure were performed by this repair.
diff --git a/src/main/java/arena/ArenaServer.java b/src/main/java/arena/ArenaServer.java
index 818b1fe..2b4fa2e 100644
--- a/src/main/java/arena/ArenaServer.java
+++ b/src/main/java/arena/ArenaServer.java
@@ -1,7 +1,9 @@
 package arena;
 
 import com.fasterxml.jackson.databind.node.ObjectNode;
+import io.netty.bootstrap.Bootstrap;
 import io.netty.bootstrap.ServerBootstrap;
+import io.netty.buffer.Unpooled;
 import io.netty.channel.Channel;
 import io.netty.channel.ChannelHandlerContext;
 import io.netty.channel.ChannelInboundHandlerAdapter;
@@ -11,13 +13,17 @@ import io.netty.channel.DefaultSelectStrategyFactory;
 import io.netty.channel.FixedRecvByteBufAllocator;
 import io.netty.channel.nio.NioEventLoopGroup;
 import io.netty.channel.socket.SocketChannel;
+import io.netty.channel.socket.DatagramPacket;
+import io.netty.channel.socket.nio.NioDatagramChannel;
 import io.netty.channel.socket.nio.NioServerSocketChannel;
 import io.netty.util.concurrent.DefaultEventExecutorChooserFactory;
 import io.netty.util.concurrent.RejectedExecutionHandlers;
 import io.netty.util.concurrent.ThreadPerTaskExecutor;
 import java.net.InetSocketAddress;
 import java.nio.channels.spi.SelectorProvider;
+import java.security.SecureRandom;
 import java.util.ArrayList;
+import java.util.Base64;
 import java.util.HashMap;
 import java.util.List;
 import java.util.Map;
@@ -44,6 +50,8 @@ public final class ArenaServer implements AutoCloseable {
     static final int EVENT_LOOP_QUEUE_LIMIT = 1_024;
     static final int CONNECTION_LIMIT = 8;
     static final int OUTBOUND_LIMIT = 64;
+    static final long UDP_BIND_TTL_NANOS = TimeUnit.MILLISECONDS.toNanos(5_000);
+    private static final SecureRandom RANDOM = new SecureRandom();
     private static final AtomicInteger SERVER_IDS = new AtomicInteger();
     private final String threadPrefix = "arena-" + SERVER_IDS.incrementAndGet() + "-";
     private final Set<Peer> peers = ConcurrentHashMap.newKeySet();
@@ -51,6 +59,7 @@ public final class ArenaServer implements AutoCloseable {
     private final AtomicInteger connections = new AtomicInteger();
     private final AtomicInteger pendingWrites = new AtomicInteger();
     private final CompleteFrame.Metrics parserMetrics = new CompleteFrame.Metrics();
+    private final UdpData.Metrics udpMetrics = new UdpData.Metrics();
     private final AtomicInteger outboundHighWater = new AtomicInteger();
     private final AtomicInteger mailboxHighWater = new AtomicInteger();
     private final AtomicBoolean closing = new AtomicBoolean();
@@ -61,6 +70,7 @@ public final class ArenaServer implements AutoCloseable {
     private final boolean internalManualClock;
     private final LongSupplier monotonicNanos;
     private final Channel listener;
+    private final Channel udpListener;
     private final Thread ticker;
     // The following fields, including the session registry, are exclusively room-owner state.
     private final Map<Peer, Session> sessions = new HashMap<>();
@@ -78,7 +88,12 @@ public final class ArenaServer implements AutoCloseable {
     private static final class Session {
         final String id = "s-" + UUID.randomUUID();
         final SnapshotStream snapshots = new SnapshotStream();
+        final long bindIssuedAt;
+        String bindToken = opaque(24);
+        InetSocketAddress endpoint;
         String playerId;
+        Session(long now) { bindIssuedAt = now; }
+        void release() { snapshots.close(); bindToken = null; endpoint = null; }
     }
 
     private final class Peer {
@@ -116,6 +131,28 @@ public final class ArenaServer implements AutoCloseable {
             });
         }
         void error(String code) { send(CompleteFrame.error(code, code)); }
+
+        void realtime(ObjectNode message, InetSocketAddress endpoint) {
+            if (!channel.isActive()) return;
+            byte[] bytes;
+            try { bytes = UdpData.bytes(message); }
+            catch (IllegalArgumentException tooLarge) { udpMetrics.outboundOversize.incrementAndGet(); return; }
+            int count = outbound.incrementAndGet();
+            if (count >= OUTBOUND_LIMIT) {
+                outbound.decrementAndGet(); send(CompleteFrame.error("CONTROL_BACKPRESSURE", "control message bound reached"), true); return;
+            }
+            outboundHighWater.accumulateAndGet(count, Math::max); udpMetrics.outputHighWater.accumulateAndGet(bytes.length, Math::max);
+            pendingWrites.incrementAndGet();
+            udpListener.writeAndFlush(new DatagramPacket(Unpooled.wrappedBuffer(bytes), endpoint)).addListener(result -> {
+                pendingWrites.decrementAndGet(); outbound.decrementAndGet();
+                if (!result.isSuccess()) { udpMetrics.ioErrors.incrementAndGet(); channel.close(); }
+            });
+        }
+    }
+
+    private static String opaque(int bytes) {
+        byte[] value = new byte[bytes]; RANDOM.nextBytes(value);
+        return Base64.getUrlEncoder().withoutPadding().encodeToString(value);
     }
 
     public ArenaServer(String host, int port, boolean manual) {
@@ -135,7 +172,7 @@ public final class ArenaServer implements AutoCloseable {
                 new ArrayBlockingQueue<>(MAILBOX_LIMIT), namedFactory("room"), new ThreadPoolExecutor.AbortPolicy());
         acceptLoop = loop("accept");
         ioLoop = loop("io");
-        Channel bound = null;
+        Channel bound = null, datagramBound = null;
         try {
             FixedRecvByteBufAllocator receive = new FixedRecvByteBufAllocator(CompleteFrame.MAX_BYTES + 4);
             receive.maxMessagesPerRead(1);
@@ -173,14 +210,23 @@ public final class ArenaServer implements AutoCloseable {
                                     (error, terminal) -> enqueue(peer, () -> peer.send(error, terminal)), parserMetrics));
                         }
                     }).bind(host, port).syncUninterruptibly().channel();
-        } catch (RuntimeException failure) {
+            datagramBound = new Bootstrap().group(ioLoop).channel(NioDatagramChannel.class)
+                    .option(ChannelOption.RCVBUF_ALLOCATOR, new FixedRecvByteBufAllocator(UdpData.MAX_BYTES + 1))
+                    .handler(new UdpData(packet -> {
+                        try {
+                            execute(() -> handleUdp(packet)); udpMetrics.dispatched.incrementAndGet();
+                        } catch (RejectedExecutionException full) { udpMetrics.mailboxDrops.incrementAndGet(); }
+                    }, udpMetrics)).bind(host, 0).syncUninterruptibly().channel();
+        } catch (Exception failure) {
+            if (datagramBound != null) datagramBound.close().syncUninterruptibly();
             if (bound != null) bound.close().syncUninterruptibly();
             owner.shutdownNow();
             acceptLoop.shutdownGracefully(0, 5, TimeUnit.SECONDS).syncUninterruptibly();
             ioLoop.shutdownGracefully(0, 5, TimeUnit.SECONDS).syncUninterruptibly();
-            throw failure;
+            throw new IllegalStateException("listener binding failed", failure);
         }
         listener = bound;
+        udpListener = datagramBound;
         if (manual) ticker = null;
         else {
             ticker = namedFactory("clock").newThread(() -> {
@@ -213,6 +259,7 @@ public final class ArenaServer implements AutoCloseable {
     }
 
     public int port() { return ((InetSocketAddress) listener.localAddress()).getPort(); }
+    public int udpPort() { return ((InetSocketAddress) udpListener.localAddress()).getPort(); }
 
     private void execute(Runnable runnable) {
         owner.execute(runnable);
@@ -227,23 +274,45 @@ public final class ArenaServer implements AutoCloseable {
     }
 
     private void handle(Peer peer, ObjectNode message) {
+        handleCommand(peer, message, null);
+    }
+
+    private void handleUdp(UdpData.Inbound packet) {
+        Peer peer = null;
+        // An established observed endpoint identifies the actual sender before claimed session identity.
+        for (var entry : sessions.entrySet()) if (packet.sender().equals(entry.getValue().endpoint)) peer = entry.getKey();
+        if (peer == null) for (var entry : sessions.entrySet())
+            if (entry.getValue().id.equals(packet.message().path("session_id").asText())) peer = entry.getKey();
+        if (peer == null) { udpMetrics.unroutable.incrementAndGet(); return; }
+        if (packet.validationError() != null) { udpMetrics.invalid.incrementAndGet(); peer.error(packet.validationError()); return; }
+        handleCommand(peer, packet.message(), packet.sender());
+    }
+
+    private void handleCommand(Peer peer, ObjectNode message, InetSocketAddress endpoint) {
         if (closing.get() || !peer.channel.isActive()) return;
         try {
             if (message.path("v").asInt(-1) != 1) { peer.error("PROTOCOL_VERSION_UNSUPPORTED"); return; }
             String type = Json.text(message, "type");
-            if (type.equals("HELLO")) {
-                Session session = sessions.computeIfAbsent(peer, ignored -> new Session());
-                peer.send(Json.message("WELCOME").put("session_id", session.id));
+            if ((endpoint != null) != UdpData.REALTIME.contains(type)) { peer.error("WRONG_TRANSPORT"); return; }
+            if (type.equals("HELLO") && endpoint == null) {
+                Session session = sessions.computeIfAbsent(peer, ignored -> new Session(monotonicNanos.getAsLong()));
+                ObjectNode welcome = Json.message("WELCOME").put("session_id", session.id).put("udp_port", udpPort());
+                if (session.bindToken != null) welcome.put("udp_bind_token", session.bindToken);
+                peer.send(welcome);
                 return;
             }
             Session session = sessions.get(peer);
             if (session == null || !session.id.equals(Json.text(message, "session_id"))) {
-                peer.error("SESSION_INVALID"); return;
+                peer.error(type.equals("UDP_BIND") ? "UDP_BIND_INVALID" : "SESSION_INVALID"); return;
+            }
+            if (endpoint != null && !type.equals("UDP_BIND")
+                    && (!endpoint.equals(session.endpoint) || !Json.epochZero(message))) {
+                peer.error("UDP_BIND_INVALID"); return;
             }
             switch (type) {
                 case "CREATE_ROOM" -> {
                     if (room != null) { peer.error("ROOM_NOT_JOINABLE"); break; }
-                    room = new Room("r-" + UUID.randomUUID());
+                    room = new Room(opaque(16));
                     peer.send(Json.message("ROOM_CREATED").put("room_id", room.id).put("status", "LOBBY"));
                 }
                 case "JOIN_ROOM" -> {
@@ -251,14 +320,38 @@ public final class ArenaServer implements AutoCloseable {
                     if (session.playerId != null || room.status() != Room.Status.LOBBY) {
                         peer.error("ROOM_NOT_JOINABLE"); break;
                     }
-                    session.playerId = "p-" + UUID.randomUUID();
-                    Room.Player player = room.join(session.playerId);
+                    String playerId;
+                    do { playerId = opaque(6); } while (room.player(playerId) != null);
+                    Room.Player player;
+                    try { player = room.join(playerId); }
+                    catch (IllegalStateException notJoinable) { peer.error("ROOM_NOT_JOINABLE"); break; }
+                    session.playerId = player.id;
+                    if (session.endpoint != null) room.ready(player.id);
                     peer.send(Json.message("ROOM_JOINED").put("room_id", room.id).put("player_id", player.id)
                             .put("slot", player.slot).put("status", room.status().name()));
                     if (room.status() == Room.Status.RUNNING) {
                         startReplication();
                     }
                 }
+                case "UDP_BIND" -> {
+                    if (session.endpoint != null || session.bindToken == null || !session.bindToken.equals(Json.text(message, "udp_bind_token"))
+                            || monotonicNanos.getAsLong() - session.bindIssuedAt >= UDP_BIND_TTL_NANOS || !Json.epochZero(message)) {
+                        peer.error("UDP_BIND_INVALID"); break;
+                    }
+                    session.endpoint = endpoint; session.bindToken = null;
+                    if (session.playerId != null) room.ready(session.playerId);
+                    peer.realtime(Json.message("UDP_BOUND").put("owner_epoch", 0), endpoint);
+                    if (room != null) startReplication();
+                }
+                case "PING" -> {
+                    if (!roomMatches(peer, message)) break;
+                    if (session.playerId == null || !session.playerId.equals(Json.text(message, "player_id"))) { peer.error("PLAYER_MISMATCH"); break; }
+                    var nonce = message.get("ping_id");
+                    if (nonce != null && (!nonce.isIntegralNumber() || !nonce.canConvertToLong())) { peer.error("MESSAGE_INVALID"); break; }
+                    ObjectNode pong = Json.message("PONG").put("owner_epoch", 0);
+                    if (nonce != null) pong.put("ping_id", nonce.longValue());
+                    peer.realtime(pong, endpoint);
+                }
                 case "SNAPSHOT_ACK" -> {
                     if (!roomMatches(peer, message)) break;
                     if (session.playerId == null || !session.playerId.equals(Json.text(message, "player_id"))) {
@@ -277,7 +370,7 @@ public final class ArenaServer implements AutoCloseable {
                     Room.Intent intent = new Room.Intent(Json.sequence(message), Json.targetTick(message), direction, target);
                     String code = room.accept(session.playerId, intent);
                     if (code == null || code.equals("DUPLICATE"))
-                        peer.send(Json.message("INPUT_ACK").put("status", code == null ? "ACCEPTED" : code).put("seq", intent.seq()));
+                        peer.realtime(Json.message("INPUT_ACK").put("status", code == null ? "ACCEPTED" : code).put("seq", intent.seq()), endpoint);
                     else peer.error(code);
                 }
                 case "LEAVE_ROOM" -> {
@@ -300,12 +393,14 @@ public final class ArenaServer implements AutoCloseable {
     private void disconnect(Peer peer) {
         Session session = sessions.remove(peer);
         if (session != null) {
-            session.snapshots.close();
+            session.release();
             if (session.playerId != null && room != null) room.left(session.playerId);
+            if (room != null) startReplication();
         }
     }
 
     private void startReplication() {
+        if (fixedClock != null || room.status() != Room.Status.RUNNING || closing.get()) return;
         fixedClock = new FixedTickClock(monotonicNanos);
         publishSnapshots(true);
     }
@@ -314,7 +409,8 @@ public final class ArenaServer implements AutoCloseable {
         for (var entry : sessions.entrySet()) {
             Session session = entry.getValue();
             if (session.playerId == null || !room.player(session.playerId).connected) continue;
-            entry.getKey().send(session.snapshots.next(room, forceFull));
+            if (session.endpoint == null) continue;
+            entry.getKey().realtime(session.snapshots.next(room, forceFull), session.endpoint);
             snapshotRetentionHighWater = Math.max(snapshotRetentionHighWater, session.snapshots.highWater());
         }
     }
@@ -376,6 +472,8 @@ public final class ArenaServer implements AutoCloseable {
                     .put("mailbox_high_water", mailboxHighWater.get()).put("outbound_high_water", outboundHighWater.get());
             result.set("clock", fixedClock == null ? closedClockMetrics.deepCopy() : fixedClock.view());
             result.set("parser", parserMetrics.view());
+            result.set("udp", udpMetrics.view());
+            result.put("udp_bindings", sessions.values().stream().filter(s -> s.endpoint != null).count());
             return result;
         });
     }
@@ -391,7 +489,7 @@ public final class ArenaServer implements AutoCloseable {
 
     public ObjectNode cleanup() {
         long liveThreads = ownedThreads.stream().filter(Thread::isAlive).count();
-        ObjectNode result = Json.MAPPER.createObjectNode().put("open_channels", peers.size() + (listener.isOpen() ? 1 : 0))
+        ObjectNode result = Json.MAPPER.createObjectNode().put("open_channels", peers.size() + (listener.isOpen() ? 1 : 0) + (udpListener.isOpen() ? 1 : 0))
                 .put("connections", connections.get()).put("pending_writes", pendingWrites.get())
                 .put("mailbox_remaining", owner.getQueue().size()).put("live_threads", liveThreads)
                 .put("timer_alive", ticker != null && ticker.isAlive()).put("owner_terminated", owner.isTerminated())
@@ -401,6 +499,7 @@ public final class ArenaServer implements AutoCloseable {
                 .put("retained_snapshots", closedSnapshotCount).put("snapshot_retention_high_water", snapshotRetentionHighWater)
                 .put("mailbox_high_water", mailboxHighWater.get()).put("outbound_high_water", outboundHighWater.get());
         result.set("parser", parserMetrics.view());
+        result.set("udp", udpMetrics.view());
         result.set("clock", closedClockMetrics.deepCopy());
         var lifecycle = result.putArray("room_lifecycle");
         closedLifecycle.forEach(lifecycle::add);
@@ -416,12 +515,13 @@ public final class ArenaServer implements AutoCloseable {
             if (ticker.isAlive()) throw new IllegalStateException("clock failed to stop");
         }
         listener.close().syncUninterruptibly();
+        udpListener.close().syncUninterruptibly();
         List<Peer> closingPeers = new ArrayList<>(peers);
         for (Peer peer : closingPeers) peer.channel.close().syncUninterruptibly();
         // Drain channel callbacks before final owner cleanup. No network thread waits for this owner.
         ioLoop.submit(() -> { }).syncUninterruptibly();
         call(() -> {
-            for (Session session : sessions.values()) session.snapshots.close();
+            for (Session session : sessions.values()) session.release();
             closedSnapshotCount = sessions.values().stream().mapToInt(s -> s.snapshots.retainedCount()).sum();
             sessions.clear();
             if (room != null) {
diff --git a/src/main/java/arena/AuthorityScenario.java b/src/main/java/arena/AuthorityScenario.java
index 078b75e..a33de10 100644
--- a/src/main/java/arena/AuthorityScenario.java
+++ b/src/main/java/arena/AuthorityScenario.java
@@ -33,6 +33,7 @@ final class AuthorityScenario {
         try {
             alpha = new TcpClient(server.port()); bravo = new TcpClient(server.port());
             alpha.hello(); String room = alpha.createRoom(); alpha.join(room); bravo.hello(); bravo.join(room);
+            alpha.bind(); bravo.bind();
             alpha.until("SNAPSHOT"); bravo.until("SNAPSHOT");
             Map<String, TcpClient> clients = Map.of("alpha", alpha, "bravo", bravo);
             Map<String, String> roles = Map.of(alpha.playerId, "alpha", bravo.playerId, "bravo");
@@ -164,6 +165,7 @@ final class AuthorityScenario {
             else if (label.equals("self-target")) target = "player-a";
             else {
                 other = new Room("independent-room"); other.join("foreign-a"); other.join("foreign-b");
+                other.ready("foreign-a"); other.ready("foreign-b");
                 target = other.player("foreign-a").id; otherBefore = view(other);
             }
             ObjectNode observed = rejectedTags.addObject().put("case", label); ObjectNode before = view(room); observed.set("before", before);
@@ -183,12 +185,13 @@ final class AuthorityScenario {
         }
         JsonNode shared = fixed.path("one_target_per_tick"); Room room = new Room("shared-target");
         room.join(shared.path("player_ids").get(0).asText()); room.join(shared.path("player_ids").get(1).asText());
-        // Pure initialization only: a third player is inserted after automatic two-player RUNNING transition.
+        // Pure initialization only: prepare the fixed third player before every player becomes ready.
         Field playersField = Room.class.getDeclaredField("players"); playersField.setAccessible(true);
         Object players = playersField.get(room);
         if (!(players instanceof TreeMap<?, ?>)) throw new IOException("unexpected Room player storage");
         @SuppressWarnings("unchecked") Map<String, Room.Player> initializedPlayers = (Map<String, Room.Player>) players;
         String third = shared.path("player_ids").get(2).asText(); initializedPlayers.put(third, new Room.Player(third, 2));
+        for (JsonNode player : shared.withArray("player_ids")) room.ready(player.asText());
         colocate(room); ObjectNode oneTarget = result.putObject("one_target_per_tick"); oneTarget.set("before", view(room));
         for (JsonNode actor : shared.withArray("tick0_actors"))
             if (room.accept(actor.asText(), intent(1, 0, "STOP", shared.path("target").asText())) != null) failures.add("shared target admission");
@@ -249,7 +252,10 @@ final class AuthorityScenario {
         return new Room.Intent(BigInteger.valueOf(seq), BigInteger.valueOf(tick), Room.Direction.valueOf(direction), target);
     }
     private static String admission(String code) { return code == null ? "ACCEPTED" : code; }
-    private static Room runningRoom() { Room room = new Room("room-unit"); room.join("player-a"); room.join("player-b"); return room; }
+    private static Room runningRoom() {
+        Room room = new Room("room-unit"); room.join("player-a"); room.join("player-b");
+        room.ready("player-a"); room.ready("player-b"); return room;
+    }
     private static void colocate(Room room) {
         for (JsonNode p : room.view("SNAPSHOT").withArray("players")) { Room.Player player = room.player(p.path("player_id").asText()); player.x = 50000; player.y = 50000; }
     }
diff --git a/src/main/java/arena/ClockScenario.java b/src/main/java/arena/ClockScenario.java
index 7b5c52e..b1f88e3 100644
--- a/src/main/java/arena/ClockScenario.java
+++ b/src/main/java/arena/ClockScenario.java
@@ -70,6 +70,7 @@ final class ClockScenario {
             alpha.hello(); bravo.hello();
             String roomId = alpha.createRoom();
             alpha.join(roomId); bravo.join(roomId);
+            alpha.bind(); bravo.bind();
             alpha.until("SNAPSHOT"); bravo.until("SNAPSHOT");
             alpha.intent(roomId, scenario.path("directions").path("alpha").asText(), null, 0);
             bravo.intent(roomId, scenario.path("directions").path("bravo").asText(), null, 0);
diff --git a/src/main/java/arena/CompleteFrame.java b/src/main/java/arena/CompleteFrame.java
index c10386a..16b0789 100644
--- a/src/main/java/arena/CompleteFrame.java
+++ b/src/main/java/arena/CompleteFrame.java
@@ -183,7 +183,7 @@ final class CompleteFrame extends SimpleChannelInboundHandler<ByteBuf> {
                 .put("buffer_ref_count", accumulated == null ? 0 : accumulated.refCnt());
     }
 
-    private static String protocolError(ObjectNode message) {
+    static String protocolError(ObjectNode message) {
         JsonNode version = message.get("v");
         JsonNode typeNode = message.get("type");
         if (version == null || !version.isIntegralNumber() || typeNode == null || !typeNode.isTextual())
@@ -194,27 +194,38 @@ final class CompleteFrame extends SimpleChannelInboundHandler<ByteBuf> {
             // Validate active schemas before owner handoff; future message families remain inactive.
             switch (type) {
                 case "HELLO" -> { }
-                case "CREATE_ROOM" -> Json.text(message, "session_id");
+                case "CREATE_ROOM" -> Json.identifier(message, "session_id");
                 case "JOIN_ROOM", "LEAVE_ROOM" -> {
-                    Json.text(message, "session_id");
-                    Json.text(message, "room_id");
+                    Json.identifier(message, "session_id");
+                    Json.identifier(message, "room_id");
                 }
                 case "SNAPSHOT_ACK" -> {
-                    Json.text(message, "session_id");
-                    Json.text(message, "room_id");
-                    Json.text(message, "player_id");
+                    Json.identifier(message, "session_id");
+                    Json.identifier(message, "room_id");
+                    Json.identifier(message, "player_id");
                     Json.snapshotSequence(message);
                 }
                 case "INPUT" -> {
-                    Json.text(message, "session_id");
-                    Json.text(message, "room_id");
-                    Json.text(message, "player_id");
+                    Json.identifier(message, "session_id");
+                    Json.identifier(message, "room_id");
+                    Json.identifier(message, "player_id");
                     Json.text(message, "direction");
                     Json.sequence(message);
                     Json.targetTick(message);
                     JsonNode target = message.get("tag_target_player_id");
                     if (target == null || !(target.isNull() || target.isTextual())) return "MESSAGE_INVALID";
+                    if (target.isTextual()) Json.identifier(message, "tag_target_player_id");
                 }
+                case "UDP_BIND" -> {
+                    Json.identifier(message, "session_id");
+                    Json.text(message, "udp_bind_token");
+                }
+                case "PING" -> {
+                    Json.identifier(message, "session_id");
+                    Json.identifier(message, "room_id");
+                    Json.identifier(message, "player_id");
+                }
+                case "UDP_BOUND", "INPUT_ACK", "SNAPSHOT", "PONG", "WELCOME", "ROOM_CREATED", "ROOM_JOINED", "ROOM_FINISHED", "ERROR" -> { }
                 default -> { return "MESSAGE_TYPE_UNKNOWN"; }
             }
         } catch (IllegalArgumentException invalid) { return "MESSAGE_INVALID"; }
diff --git a/src/main/java/arena/IdentityScenario.java b/src/main/java/arena/IdentityScenario.java
index 4540888..7d2b8cd 100644
--- a/src/main/java/arena/IdentityScenario.java
+++ b/src/main/java/arena/IdentityScenario.java
@@ -7,14 +7,11 @@ import io.netty.buffer.ByteBuf;
 import io.netty.channel.Channel;
 import io.netty.channel.ChannelHandlerContext;
 import io.netty.channel.ChannelInboundHandlerAdapter;
+import io.netty.channel.EventLoopGroup;
 import io.netty.channel.embedded.EmbeddedChannel;
-import java.io.DataInputStream;
 import java.io.IOException;
 import java.lang.reflect.Field;
 import java.lang.reflect.InvocationTargetException;
-import java.net.InetSocketAddress;
-import java.net.Socket;
-import java.nio.ByteBuffer;
 import java.util.ArrayList;
 import java.util.HashSet;
 import java.util.List;
@@ -177,23 +174,14 @@ final class IdentityScenario {
 
     private void ownerProbe(ObjectNode out) throws Exception {
         World world = new World();
+        CountDownLatch parsed = new CountDownLatch(1);
         CountDownLatch release = new CountDownLatch(1);
         try {
             world.setup(2);
             Client sender = world.client(scenario.path("owner_probe").path("sender").asText());
-            Channel channel = (Channel) field(world.peer(sender), "channel");
-            CountDownLatch parsed = new CountDownLatch(1);
-            channel.eventLoop().submit(() -> {
-                CompleteFrame framing = channel.pipeline().get(CompleteFrame.class);
-                int messagesBefore = framing.state().path("messages").asInt();
-                String parser = channel.pipeline().context(framing).name();
-                channel.pipeline().addBefore(parser, "g03-observe-parser-return", new ChannelInboundHandlerAdapter() {
-                    @Override public void channelRead(ChannelHandlerContext context, Object message) {
-                        context.fireChannelRead(message);
-                        if (framing.state().path("messages").asInt() > messagesBefore) parsed.countDown();
-                    }
-                });
-            }).syncUninterruptibly();
+            EventLoopGroup ioLoop = (EventLoopGroup) field(world.server, "ioLoop");
+            UdpData.Metrics udpMetrics = (UdpData.Metrics) field(world.server, "udpMetrics");
+            int dispatchedBefore = ioLoop.submit((Callable<Integer>) udpMetrics.dispatched::get).get(5, TimeUnit.SECONDS);
             CountDownLatch held = new CountDownLatch(1);
             CountDownLatch observed = new CountDownLatch(1);
             AtomicReference<ObjectNode> before = new AtomicReference<>();
@@ -212,6 +200,11 @@ final class IdentityScenario {
                     .put("seq", 1).put("target_tick", 0).put("owner_epoch", 0)
                     .put("direction", scenario.path("owner_probe").path("direction").asText())
                     .putNull("tag_target_player_id"));
+            // Observe the actual UDP dispatch after its event-loop callback; no extra owner/network command.
+            int dispatchedAfter = awaitUdpDispatch(ioLoop, udpMetrics, dispatchedBefore);
+            out.put("udp_dispatched_inputs", dispatchedAfter - dispatchedBefore);
+            check(dispatchedAfter - dispatchedBefore == 1, "exactly one UDP INPUT dispatched");
+            parsed.countDown();
             await(observed);
             out.set("before", world.normalize(before.get()));
             out.set("before_consumer_release", world.normalize(beforeRelease.get()));
@@ -237,7 +230,7 @@ final class IdentityScenario {
             catch (IllegalStateException foreignOwner) { rejected = true; }
             out.put("foreign_mutation_rejected", rejected);
             check(rejected && ticked.equals(world.snapshot()), "foreign owner rejection changed state");
-        } finally { release.countDown(); world.finish(out); }
+        } finally { parsed.countDown(); release.countDown(); world.finish(out); }
     }
 
     private void mailboxProbe(ObjectNode out) throws Exception {
@@ -349,6 +342,7 @@ final class IdentityScenario {
         }
         void joinBravo() throws IOException {
             bravo.hello(); bravo.join(roomId);
+            alpha.bind(); bravo.bind();
             alpha.until("SNAPSHOT"); bravo.until("SNAPSHOT");
         }
         Object peer(Client client) throws Exception {
@@ -401,48 +395,29 @@ final class IdentityScenario {
             for (Client client : clients) client.close();
             server.close();
             cleanup(server, out);
-            out.put("client_sockets_closed", clients.stream().allMatch(client -> client.socket.isClosed()));
+            out.put("client_sockets_closed", clients.stream().allMatch(Client::isClosed));
             check(out.path("client_sockets_closed").asBoolean(), "client socket cleanup");
         }
     }
 
     private static final class Client implements AutoCloseable {
-        final Socket socket = new Socket();
-        final DataInputStream input;
+        final TcpClient transport;
         String sessionId;
         String playerId;
-        Client(int port) throws IOException {
-            try {
-                socket.connect(new InetSocketAddress("127.0.0.1", port), 5_000);
-                socket.setTcpNoDelay(true); socket.setSoTimeout(5_000);
-                input = new DataInputStream(socket.getInputStream());
-            } catch (IOException failure) { socket.close(); throw failure; }
-        }
-        void send(ObjectNode request) throws IOException {
-            byte[] payload = Json.bytes(request);
-            socket.getOutputStream().write(ByteBuffer.allocate(payload.length + 4).putInt(payload.length).put(payload).array());
-            socket.getOutputStream().flush();
-        }
+        Client(int port) throws IOException { transport = new TcpClient(port); }
+        void send(ObjectNode request) throws IOException { transport.send(request); }
         ObjectNode until(String type) throws IOException {
-            for (int i = 0; i < 64; i++) {
-                int length = input.readInt();
-                if (length < 1 || length > CompleteFrame.MAX_BYTES) throw new IOException("server frame bound");
-                byte[] bytes = new byte[length]; input.readFully(bytes);
-                ObjectNode reply = Json.read(bytes);
-                if (reply.path("type").asText().equals(type)) return reply;
-                if (reply.path("type").asText().equals("ERROR")) throw new IOException("unexpected server error: " + reply);
-            }
-            throw new IOException("client response bound");
-        }
-        ObjectNode auth(String type, String room) {
-            ObjectNode request = Json.message(type).put("session_id", sessionId);
-            if (room != null) request.put("room_id", room);
-            if (playerId != null) request.put("player_id", playerId);
-            return request;
+            if (!type.equals("ERROR")) return transport.until(type);
+            ObjectNode reply = transport.control();
+            if (!reply.path("type").asText().equals("ERROR")) throw new IOException("expected identity/control ERROR");
+            return reply;
         }
-        void hello() throws IOException { send(Json.message("HELLO")); sessionId = Json.text(until("WELCOME"), "session_id"); }
-        void join(String room) throws IOException { send(auth("JOIN_ROOM", room)); playerId = Json.text(until("ROOM_JOINED"), "player_id"); }
-        @Override public void close() throws IOException { socket.close(); }
+        ObjectNode auth(String type, String room) { return transport.auth(type, room); }
+        void hello() throws IOException { transport.hello(); sessionId = transport.sessionId; }
+        void join(String room) throws IOException { transport.join(room); playerId = transport.playerId; }
+        void bind() throws IOException { transport.bind(); }
+        boolean isClosed() { return transport.isClosed(); }
+        @Override public void close() { transport.close(); }
     }
 
     private static Object field(Object object, String name) throws ReflectiveOperationException {
@@ -459,6 +434,15 @@ final class IdentityScenario {
     private static <T> T onOwner(ArenaServer server, Callable<T> action) throws Exception {
         return owner(server).submit(action).get(5, TimeUnit.SECONDS);
     }
+    private static int awaitUdpDispatch(EventLoopGroup ioLoop, UdpData.Metrics metrics, int before) throws Exception {
+        long deadline = System.nanoTime() + TimeUnit.SECONDS.toNanos(5);
+        while (true) {
+            long remaining = deadline - System.nanoTime();
+            if (remaining <= 0) throw new IOException("UDP dispatch observation deadline");
+            int dispatched = ioLoop.submit((Callable<Integer>) metrics.dispatched::get).get(remaining, TimeUnit.NANOSECONDS);
+            if (dispatched > before) return dispatched;
+        }
+    }
     private static Object invoke(java.lang.reflect.Method method, Object receiver, Object... arguments) throws Exception {
         try { return method.invoke(receiver, arguments); }
         catch (InvocationTargetException failure) {
diff --git a/src/main/java/arena/Json.java b/src/main/java/arena/Json.java
index 5f8ee49..a68a6f7 100644
--- a/src/main/java/arena/Json.java
+++ b/src/main/java/arena/Json.java
@@ -48,6 +48,17 @@ final class Json {
         return value.textValue();
     }
 
+    static String identifier(ObjectNode object, String field) {
+        String id = text(object, field);
+        if (!id.matches("[A-Za-z0-9_-]{1,64}")) throw new IllegalArgumentException("invalid identifier");
+        return id;
+    }
+
+    static boolean epochZero(ObjectNode object) {
+        JsonNode value = object.get("owner_epoch");
+        return value != null && value.isIntegralNumber() && value.bigIntegerValue().signum() == 0;
+    }
+
     static BigInteger sequence(ObjectNode object) {
         JsonNode value = object.get("seq");
         if (value == null || !value.isIntegralNumber()) throw new IllegalArgumentException("seq must be an unsigned integer");
diff --git a/src/main/java/arena/Room.java b/src/main/java/arena/Room.java
index 55add01..accf9cf 100644
--- a/src/main/java/arena/Room.java
+++ b/src/main/java/arena/Room.java
@@ -35,6 +35,7 @@ final class Room {
         int lastTagTick = -20;
         Direction direction = Direction.STOP;
         boolean connected = true;
+        boolean realtimeReady;
         BigInteger lastAcceptedSeq = BigInteger.ZERO;
         Intent lastAcceptedIntent;
         BigInteger appliedSeq;
@@ -81,11 +82,23 @@ final class Room {
             throw new IllegalStateException("ROOM_NOT_JOINABLE");
         Player player = new Player(playerId, nextSlot++);
         players.put(playerId, player);
-        if (players.values().stream().filter(p -> p.connected).count() >= 2) {
+        return player;
+    }
+
+    void ready(String id) {
+        assertOwner();
+        Player player = players.get(id);
+        if (player == null || !player.connected) throw new IllegalArgumentException("PLAYER_MISMATCH");
+        player.realtimeReady = true;
+        startIfReady();
+    }
+
+    private void startIfReady() {
+        if (status == Status.LOBBY && players.values().stream().filter(p -> p.connected).count() >= 2
+                && players.values().stream().filter(p -> p.connected).allMatch(p -> p.realtimeReady)) {
             transition(Status.RUNNING);
             replay = new ReplayLog(id, players.values());
         }
-        return player;
     }
 
     /** Called after transport identity checks, before direction/sequence admission validation. */
@@ -181,6 +194,7 @@ final class Room {
         Player player = players.get(id);
         if (player != null && player.connected && replay != null) replay.left(executedTicks, id);
         if (player != null) { player.connected = false; player.direction = Direction.STOP; player.pending.clear(); }
+        startIfReady();
     }
 
     void close() {
diff --git a/src/main/java/arena/ScenarioRunner.java b/src/main/java/arena/ScenarioRunner.java
index ff084d5..e7c4f04 100644
--- a/src/main/java/arena/ScenarioRunner.java
+++ b/src/main/java/arena/ScenarioRunner.java
@@ -82,6 +82,7 @@ final class ScenarioRunner {
                     default -> throw new IOException("unsupported setup step");
                 }
             }
+            for (TcpClient client : clients.values()) client.bind();
             for (TcpClient client : clients.values()) client.until("SNAPSHOT");
             int currentTick = 0;
             int accepted = 0;
@@ -168,6 +169,14 @@ final class ScenarioRunner {
         if (parser.path("buffer_high_water").asInt() > CompleteFrame.BUFFER_BOUND
                 || parser.path("capacity_high_water").asInt() > CompleteFrame.BUFFER_BOUND)
             failures.add("parser bound");
+        JsonNode udp = cleanup.path("udp");
+        if (udp.path("live_buffers").asInt(-1) != 0 || udp.path("allocated_bytes").asInt(-1) != 0)
+            failures.add("UDP buffer cleanup");
+        int inputHighWater = udp.path("input_high_water").asInt(-1);
+        int outputHighWater = udp.path("output_high_water").asInt(-1);
+        if (inputHighWater < 0 || inputHighWater > UdpData.MAX_BYTES + 1
+                || outputHighWater < 0 || outputHighWater > UdpData.MAX_BYTES)
+            failures.add("UDP datagram bound");
         if (!failures.isEmpty()) throw new IOException("cleanup failure: " + failures);
     }
 
diff --git a/src/main/java/arena/SequenceScenario.java b/src/main/java/arena/SequenceScenario.java
index 0b37b67..abbd65a 100644
--- a/src/main/java/arena/SequenceScenario.java
+++ b/src/main/java/arena/SequenceScenario.java
@@ -32,6 +32,7 @@ final class SequenceScenario {
         try {
             alpha = new TcpClient(server.port()); bravo = new TcpClient(server.port());
             alpha.hello(); String room = alpha.createRoom(); alpha.join(room); bravo.hello(); bravo.join(room);
+            alpha.bind(); bravo.bind();
             alpha.until("SNAPSHOT"); bravo.until("SNAPSHOT");
             int eventIndex = 0;
             for (int tick = 0; tick < scenario.path("ticks").asInt(); tick++) {
@@ -160,7 +161,10 @@ final class SequenceScenario {
             outcomes.add(code == null ? "ACCEPTED" : code);
         });
         final EmbeddedChannel channel = new EmbeddedChannel(parser);
-        Pure() { room.join("player-a"); room.join("player-b"); }
+        Pure() {
+            room.join("player-a"); room.join("player-b");
+            room.ready("player-a"); room.ready("player-b");
+        }
         String send(ObjectNode request) throws IOException {
             outcomes.clear(); ByteBuf inbound = CompleteFrame.encode(request); channel.writeInbound(inbound);
             if (inbound.refCnt() != 0) throw new IOException("pure parser inbound leak");
diff --git a/src/main/java/arena/TcpClient.java b/src/main/java/arena/TcpClient.java
index cf331d0..59f0dfd 100644
--- a/src/main/java/arena/TcpClient.java
+++ b/src/main/java/arena/TcpClient.java
@@ -1,119 +1,217 @@
 package arena;
 
 import com.fasterxml.jackson.databind.node.ObjectNode;
-import java.io.DataInputStream;
+import java.io.EOFException;
 import java.io.IOException;
 import java.io.UncheckedIOException;
 import java.net.InetSocketAddress;
 import java.net.Socket;
 import java.nio.ByteBuffer;
+import java.nio.channels.DatagramChannel;
+import java.nio.channels.SelectionKey;
+import java.nio.channels.Selector;
+import java.nio.channels.SocketChannel;
+import java.util.ArrayDeque;
 import java.util.ArrayList;
 import java.util.List;
+import java.util.concurrent.TimeUnit;
+import java.util.function.Predicate;
 
-/** Blocking CLI client; never used on a Netty event loop or simulation-owner thread. */
+/** Bounded CLI client for TCP control and UDP data; never runs on a server owner/event loop. */
 final class TcpClient implements AutoCloseable {
-    private final Socket socket = new Socket();
-    private final DataInputStream input;
+    private final SocketChannel tcp;
+    private final Socket socket;
+    private final DatagramChannel udp;
+    private final Selector selector;
+    private final ByteBuffer tcpBytes = ByteBuffer.allocate(CompleteFrame.MAX_BYTES + 4);
+    private final ByteBuffer udpBytes = ByteBuffer.allocate(UdpData.MAX_BYTES + 1);
+    private final ArrayDeque<ObjectNode> received = new ArrayDeque<>();
     private final List<String> lifecycle = new ArrayList<>();
+    private InetSocketAddress udpDestination;
+    private String bindToken;
+    private boolean eof;
     String sessionId;
     String playerId;
+    String roomId;
     private long nextSequence = 1;
 
     TcpClient(int port) throws IOException {
+        tcp = SocketChannel.open(); socket = tcp.socket(); udp = DatagramChannel.open(); selector = Selector.open();
         try {
-            socket.connect(new InetSocketAddress("127.0.0.1", port), 5_000);
-            socket.setTcpNoDelay(true);
-            socket.setSoTimeout(5_000);
-            input = new DataInputStream(socket.getInputStream());
-        } catch (IOException failure) { socket.close(); throw failure; }
+            socket.connect(new InetSocketAddress("127.0.0.1", port), 5_000); socket.setTcpNoDelay(true);
+            tcp.configureBlocking(false); tcp.register(selector, SelectionKey.OP_READ);
+            udp.bind(new InetSocketAddress("127.0.0.1", 0)); udp.configureBlocking(false); udp.register(selector, SelectionKey.OP_READ);
+        } catch (IOException failure) { close(); throw failure; }
     }
 
     void send(ObjectNode message) throws IOException {
+        if (UdpData.REALTIME.contains(message.path("type").asText())) sendUdp(message);
+        else sendTcp(message);
+    }
+
+    void sendTcp(ObjectNode message) throws IOException {
         byte[] payload = Json.bytes(message);
-        if (payload.length > CompleteFrame.MAX_BYTES) throw new IOException("client frame too large");
-        byte[] frame = ByteBuffer.allocate(4 + payload.length).putInt(payload.length).put(payload).array();
-        socket.getOutputStream().write(frame);
-        socket.getOutputStream().flush();
+        if (payload.length < 1 || payload.length > CompleteFrame.MAX_BYTES) throw new IOException("client frame too large");
+        ByteBuffer bytes = ByteBuffer.allocate(4 + payload.length).putInt(payload.length).put(payload); bytes.flip();
+        long deadline = deadline();
+        while (bytes.hasRemaining()) if (tcp.write(bytes) == 0) awaitWritable(tcp.keyFor(selector), deadline);
+    }
+
+    void sendUdp(ObjectNode message) throws IOException {
+        if (udpDestination == null) throw new IOException("WELCOME has not advertised a UDP endpoint");
+        ByteBuffer bytes = ByteBuffer.wrap(UdpData.bytes(message)); long deadline = deadline();
+        while (udp.send(bytes, udpDestination) == 0) awaitWritable(udp.keyFor(selector), deadline);
     }
 
+    // A client may address a local datagram proxy; this does not change server admission.
+    void udpDestination(InetSocketAddress endpoint) { udpDestination = endpoint; }
+    InetSocketAddress udpDestination() { return udpDestination; }
+    InetSocketAddress localUdpEndpoint() throws IOException { return (InetSocketAddress) udp.getLocalAddress(); }
+
+    ObjectNode binding() {
+        if (bindToken == null) throw new IllegalStateException("WELCOME has no binding credential");
+        return Json.message("UDP_BIND").put("session_id", sessionId).put("udp_bind_token", bindToken).put("owner_epoch", 0);
+    }
+
+    ObjectNode bind() throws IOException { send(binding()); return until("UDP_BOUND"); }
+
     ObjectNode until(String type) throws IOException {
-        for (int messages = 0; messages < 64; messages++) {
-            ObjectNode message = readFrame();
-            if (message == null) throw new IOException("unexpected server EOF");
-            if (message.path("type").asText().equals("ERROR")) throw new IOException("server error: " + message);
-            if (message.path("type").asText().equals(type)) return message;
-        }
-        throw new IOException("response message ceiling exceeded");
+        ObjectNode message = matching(value -> value.equals(type) || value.equals("ERROR"));
+        if (message.path("type").asText().equals("ERROR")) throw new IOException("server error: " + message.path("code").asText());
+        return message;
     }
 
-    /** Added replication frames do not replace the original INPUT_ACK/ERROR observations. */
+    /** Replication frames do not replace the original INPUT_ACK/ERROR observations. */
     ObjectNode control() throws IOException {
-        for (int messages = 0; messages < 64; messages++) {
-            ObjectNode message = readFrame();
-            if (message == null) throw new IOException("unexpected server EOF");
-            if (!message.path("type").asText().equals("SNAPSHOT")) return message;
+        return matching(value -> !value.equals("SNAPSHOT"));
+    }
+
+    private ObjectNode matching(Predicate<String> wanted) throws IOException {
+        long deadline = deadline();
+        while (true) {
+            var messages = received.iterator();
+            while (messages.hasNext()) {
+                ObjectNode message = messages.next();
+                if (wanted.test(message.path("type").asText())) { messages.remove(); return message; }
+            }
+            if (eof) throw new IOException("unexpected server EOF");
+            // TCP control and UDP publication can become readable in either order.
+            pump(deadline);
         }
-        throw new IOException("response message ceiling exceeded");
     }
 
-    private ObjectNode readFrame() throws IOException {
-        int first = input.read(); if (first == -1) return null;
-        int length = first << 24 | input.readUnsignedByte() << 16 | input.readUnsignedByte() << 8 | input.readUnsignedByte();
-        if (length < 1 || length > CompleteFrame.MAX_BYTES) throw new IOException("bad server length");
-        byte[] payload = new byte[length]; input.readFully(payload); ObjectNode message = Json.read(payload);
+    ObjectNode read() throws IOException {
+        long deadline = deadline();
+        while (received.isEmpty() && !eof) pump(deadline);
+        return received.pollFirst();
+    }
+
+    private static long deadline() { return System.nanoTime() + TimeUnit.SECONDS.toNanos(5); }
+
+    private void awaitWritable(SelectionKey key, long deadline) throws IOException {
+        key.interestOps(SelectionKey.OP_READ | SelectionKey.OP_WRITE);
+        try { pump(deadline); } finally { if (key.isValid()) key.interestOps(SelectionKey.OP_READ); }
+    }
+
+    private void pump(long deadline) throws IOException {
+        long remaining = deadline - System.nanoTime();
+        if (remaining <= 0) throw new IOException("client socket ceiling exceeded");
+        selector.select(Math.max(1, TimeUnit.NANOSECONDS.toMillis(remaining)));
+        var keys = selector.selectedKeys().iterator();
+        while (keys.hasNext()) {
+            SelectionKey key = keys.next(); keys.remove();
+            if (!key.isValid() || !key.isReadable()) continue;
+            if (key.channel() == tcp) readTcp(); else readUdp();
+        }
+    }
+
+    private void readTcp() throws IOException {
+        int count = tcp.read(tcpBytes);
+        if (count == -1) {
+            if (tcpBytes.position() != 0) throw new EOFException("truncated server frame");
+            eof = true; tcp.keyFor(selector).cancel(); return;
+        }
+        tcpBytes.flip();
+        while (tcpBytes.remaining() >= 4) {
+            tcpBytes.mark(); int length = tcpBytes.getInt();
+            if (length < 1 || length > CompleteFrame.MAX_BYTES) throw new IOException("bad server length");
+            if (tcpBytes.remaining() < length) { tcpBytes.reset(); break; }
+            byte[] payload = new byte[length]; tcpBytes.get(payload); observe(Json.read(payload));
+        }
+        tcpBytes.compact();
+    }
+
+    private void readUdp() throws IOException {
+        udpBytes.clear(); var sender = udp.receive(udpBytes);
+        if (sender == null || !sender.equals(udpDestination)) return;
+        if (udpBytes.position() < 1 || udpBytes.position() > UdpData.MAX_BYTES) throw new IOException("bad server datagram length");
+        udpBytes.flip(); observe(Json.read(udpBytes));
+    }
+
+    private void observe(ObjectNode message) throws IOException {
+        if (message.path("type").asText().equals("WELCOME")) {
+            sessionId = Json.text(message, "session_id");
+            if (udpDestination == null) {
+                var port = message.path("udp_port");
+                if (!port.isIntegralNumber() || !port.canConvertToInt() || port.asInt() < 1 || port.asInt() > 65_535)
+                    throw new IOException("WELCOME has no valid UDP port");
+                udpDestination = new InetSocketAddress(socket.getInetAddress(), port.asInt());
+            }
+            if (message.has("udp_bind_token")) bindToken = Json.text(message, "udp_bind_token");
+            message.put("udp_bind_token_present", message.has("udp_bind_token")); message.remove("udp_bind_token");
+        }
+        // Credentials remain only in memory and are never returned to evidence or exception formatting.
+        message.remove("resume_token");
         String status = message.path("status").asText();
         if (List.of("LOBBY", "RUNNING", "FINISHED").contains(status)) observe(status);
-        return message;
+        if (received.size() >= 64) throw new IOException("client response queue bound exceeded");
+        received.addLast(message);
     }
 
-    void hello() throws IOException {
-        send(Json.message("HELLO"));
-        sessionId = Json.text(until("WELCOME"), "session_id");
-    }
+    ObjectNode hello() throws IOException { send(Json.message("HELLO")); return until("WELCOME"); }
 
     String createRoom() throws IOException {
-        send(auth("CREATE_ROOM", null));
-        return Json.text(until("ROOM_CREATED"), "room_id");
+        send(auth("CREATE_ROOM", null)); return Json.text(until("ROOM_CREATED"), "room_id");
     }
 
-    void join(String roomId) throws IOException {
-        send(auth("JOIN_ROOM", roomId));
-        playerId = Json.text(until("ROOM_JOINED"), "player_id");
+    ObjectNode join(String id) throws IOException {
+        send(auth("JOIN_ROOM", id)); ObjectNode response = until("ROOM_JOINED");
+        playerId = Json.text(response, "player_id"); roomId = id; return response;
     }
 
-    ObjectNode auth(String type, String roomId) {
+    ObjectNode auth(String type, String id) {
         ObjectNode request = Json.message(type);
         if (sessionId != null) request.put("session_id", sessionId);
-        if (roomId != null) request.put("room_id", roomId);
+        if (id != null) request.put("room_id", id);
         if (playerId != null) request.put("player_id", playerId);
+        if (UdpData.REALTIME.contains(type)) request.put("owner_epoch", 0);
         return request;
     }
 
-    void intent(String roomId, String direction, String targetId, int targetTick) throws IOException {
-        ObjectNode request = auth("INPUT", roomId).put("direction", direction)
-                .put("seq", nextSequence++).put("target_tick", targetTick).put("owner_epoch", 0);
-        if (targetId == null) request.putNull("tag_target_player_id");
-        else request.put("tag_target_player_id", targetId);
-        send(request);
-        until("INPUT_ACK");
+    void intent(String id, String direction, String targetId, int targetTick) throws IOException {
+        ObjectNode request = auth("INPUT", id).put("direction", direction)
+                .put("seq", nextSequence++).put("target_tick", targetTick);
+        if (targetId == null) request.putNull("tag_target_player_id"); else request.put("tag_target_player_id", targetId);
+        send(request); until("INPUT_ACK");
     }
 
     void expectClosed() throws IOException {
         for (int messages = 0; messages < 64; messages++) {
-            ObjectNode message = readFrame();
+            ObjectNode message = read();
             if (message == null) { observe("CLOSED"); return; }
-            if (!message.path("type").asText().equals("SNAPSHOT")) throw new IOException("expected EOF after graceful server close: " + message);
+            if (!message.path("type").asText().equals("SNAPSHOT")) throw new IOException("expected EOF after graceful server close");
         }
         throw new IOException("close message ceiling exceeded");
     }
 
     List<String> lifecycle() { return List.copyOf(lifecycle); }
-    boolean isClosed() { return socket.isClosed(); }
+    boolean isClosed() { return !tcp.isOpen() && !udp.isOpen() && !selector.isOpen(); }
     private void observe(String state) {
         if (lifecycle.isEmpty() || !lifecycle.getLast().equals(state)) lifecycle.add(state);
     }
     @Override public void close() {
-        try { socket.close(); }
+        bindToken = null; received.clear();
+        try { tcp.close(); udp.close(); selector.close(); }
         catch (IOException failure) { throw new UncheckedIOException(failure); }
     }
 }
diff --git a/src/main/java/arena/UdpData.java b/src/main/java/arena/UdpData.java
new file mode 100644
index 0000000..32087a2
--- /dev/null
+++ b/src/main/java/arena/UdpData.java
@@ -0,0 +1,56 @@
+package arena;
+
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import io.netty.channel.ChannelHandlerContext;
+import io.netty.channel.SimpleChannelInboundHandler;
+import io.netty.channel.socket.DatagramPacket;
+import java.io.IOException;
+import java.net.InetSocketAddress;
+import java.util.Set;
+import java.util.concurrent.atomic.AtomicInteger;
+import java.util.function.Consumer;
+
+/** Bounded datagrams; no buffer crosses the event-loop/owner boundary. */
+final class UdpData extends SimpleChannelInboundHandler<DatagramPacket> {
+    static final int MAX_BYTES = 1_200;
+    static final Set<String> REALTIME = Set.of("UDP_BIND", "UDP_BOUND", "INPUT", "INPUT_ACK", "SNAPSHOT", "SNAPSHOT_ACK", "PING", "PONG");
+    record Inbound(InetSocketAddress sender, ObjectNode message, String validationError) { }
+    static final class Metrics {
+        final AtomicInteger received = new AtomicInteger(), dispatched = new AtomicInteger(), oversize = new AtomicInteger();
+        final AtomicInteger invalid = new AtomicInteger(), unroutable = new AtomicInteger(), mailboxDrops = new AtomicInteger();
+        final AtomicInteger outboundOversize = new AtomicInteger(), liveBuffers = new AtomicInteger(), allocatedBytes = new AtomicInteger();
+        final AtomicInteger inputHighWater = new AtomicInteger(), outputHighWater = new AtomicInteger(), ioErrors = new AtomicInteger();
+        ObjectNode view() {
+            return Json.MAPPER.createObjectNode().put("received", received.get()).put("dispatched", dispatched.get())
+                    .put("oversize_dropped", oversize.get()).put("invalid", invalid.get()).put("unroutable", unroutable.get())
+                    .put("mailbox_dropped", mailboxDrops.get()).put("outbound_oversize_rejected", outboundOversize.get())
+                    .put("live_buffers", liveBuffers.get()).put("allocated_bytes", allocatedBytes.get())
+                    .put("input_high_water", inputHighWater.get()).put("output_high_water", outputHighWater.get()).put("io_errors", ioErrors.get());
+        }
+    }
+    private final Consumer<Inbound> receiver;
+    private final Metrics metrics;
+    UdpData(Consumer<Inbound> receiver, Metrics metrics) { super(false); this.receiver = receiver; this.metrics = metrics; }
+
+    @Override protected void channelRead0(ChannelHandlerContext context, DatagramPacket packet) {
+        int capacity = packet.content().capacity(); metrics.liveBuffers.incrementAndGet(); metrics.allocatedBytes.addAndGet(capacity);
+        try {
+            int length = packet.content().readableBytes(); metrics.received.incrementAndGet(); metrics.inputHighWater.accumulateAndGet(length, Math::max);
+            if (length > MAX_BYTES) { metrics.oversize.incrementAndGet(); return; }
+            ObjectNode message;
+            try { message = Json.read(packet.content().nioBuffer()); }
+            catch (IOException invalid) { metrics.invalid.incrementAndGet(); return; }
+            receiver.accept(new Inbound(packet.sender(), message, CompleteFrame.protocolError(message)));
+        } finally {
+            boolean released = packet.release(); metrics.liveBuffers.decrementAndGet(); metrics.allocatedBytes.addAndGet(-capacity);
+            if (!released) throw new IllegalStateException("UDP input buffer has another owner");
+        }
+    }
+    @Override public void exceptionCaught(ChannelHandlerContext context, Throwable cause) { metrics.ioErrors.incrementAndGet(); }
+
+    static byte[] bytes(ObjectNode message) {
+        byte[] bytes = Json.bytes(message);
+        if (bytes.length == 0 || bytes.length > MAX_BYTES) throw new IllegalArgumentException("UDP datagram exceeds 1200-byte bound");
+        return bytes;
+    }
+}
diff --git a/src/test/java/arena/ReplayFixture.java b/src/test/java/arena/ReplayFixture.java
index 9499baf..f495560 100644
--- a/src/test/java/arena/ReplayFixture.java
+++ b/src/test/java/arena/ReplayFixture.java
@@ -4,38 +4,51 @@ import com.fasterxml.jackson.databind.JsonNode;
 import com.fasterxml.jackson.databind.node.ObjectNode;
 import java.io.IOException;
 import java.lang.reflect.Field;
-import java.lang.reflect.Method;
+import io.netty.channel.nio.NioEventLoopGroup;
 import java.util.Map;
-import java.util.TreeMap;
 import java.util.concurrent.Callable;
 import java.util.concurrent.ThreadPoolExecutor;
 import java.util.concurrent.TimeUnit;
 
-/** Test runtime only. Initial state setup is never represented as successful public joins. */
+/** Test runtime only. Identifiers may be fixed; joins and readiness use the real transports. */
 final class ReplayFixture {
     private ReplayFixture() { }
 
     static ObjectNode bootstrap(ArenaServer server, ObjectNode initial, Map<String, TcpClient> clients) throws Exception {
-        return owned(server, () -> {
-            Room room = prepared(initial); set(server, "room", room);
-            Object registry = field(server, "sessions");
-            if (!(registry instanceof Map<?, ?> sessions) || sessions.size() != clients.size())
-                throw new IOException("four real HELLO session bindings required");
-            int bound = 0;
-            for (JsonNode player : initial.withArray("players")) {
-                TcpClient client = clients.get(player.path("client").asText());
+        ObjectNode observed = joinFixed(server, initial, clients);
+        for (TcpClient client : clients.values()) client.bind();
+        observed.put("bound_live_sessions", clients.size()).put("normal_start_evaluations", 1);
+        observed.set("state", snapshot(server)); return observed;
+    }
+
+    static ObjectNode joinFixed(ArenaServer server, ObjectNode initial, Map<String, TcpClient> clients) throws Exception {
+        ObjectNode observed = Json.MAPPER.createObjectNode().put("kind", "normal TCP joins; test-build-only identifier remapping");
+        var joins = observed.putArray("joins"); String fixedRoom = Json.identifier(initial, "room_id");
+        clients.values().iterator().next().createRoom();
+        owned(server, () -> { set(field(server, "room"), "id", fixedRoom); return null; });
+        for (JsonNode fixture : initial.withArray("players")) {
+            String role = fixture.path("client").asText(); TcpClient client = clients.get(role);
+            ObjectNode joined = client.join(fixedRoom); joins.addObject().put("client", role).set("response", joined);
+            if (!joined.path("status").asText().equals("LOBBY")) throw new IOException("unbound ordinary join started Room");
+            String generated = client.playerId, fixed = fixture.path("player_id").asText();
+            owned(server, () -> {
+                Room room = (Room) field(server, "room"); Room.Player player = room.player(generated);
+                if (player.slot != fixture.path("slot").asInt() || player.x != fixture.path("spawn").get(0).asInt()
+                        || player.y != fixture.path("spawn").get(1).asInt()) throw new IOException("normal spawn differs from fixed roster");
+                @SuppressWarnings("unchecked") Map<String, Room.Player> roster = (Map<String, Room.Player>) field(room, "players");
+                roster.remove(generated); set(player, "id", fixed);
+                if (roster.put(fixed, player) != null) throw new IOException("duplicate fixed player");
                 boolean found = false;
-                for (Object session : sessions.values()) if (field(session, "id").equals(client.sessionId)) {
-                    set(session, "playerId", player.path("player_id").asText()); found = true; bound++;
+                for (Object session : ((Map<?, ?>) field(server, "sessions")).values()) if (field(session, "id").equals(client.sessionId)) {
+                    set(session, "playerId", fixed); found = true;
                 }
-                if (!found) throw new IOException("real session missing");
-            }
-            start(room, initial);
-            Method start = ArenaServer.class.getDeclaredMethod("startReplication"); start.setAccessible(true); start.invoke(server);
-            ObjectNode observed = Json.MAPPER.createObjectNode().put("kind", "test-only owner initial state; not public joins")
-                    .put("bound_live_sessions", bound).put("normal_start_evaluations", 1);
-            observed.set("state", view(room)); return observed;
-        });
+                if (!found) throw new IOException("normal join session missing");
+                return null;
+            });
+            client.playerId = fixed;
+        }
+        observed.set("unbound_state", snapshot(server));
+        return observed;
     }
 
     static Room offlineInitial(ObjectNode header) throws Exception {
@@ -46,22 +59,17 @@ final class ReplayFixture {
         var players = initial.withArray("players");
         if (players.size() < 2 || players.size() > Room.SPAWNS.length) throw new IOException("initial roster bound");
         Room room = new Room(Json.text(initial, "room_id"));
-        Object storage = field(room, "players");
-        if (!(storage instanceof TreeMap<?, ?>)) throw new IOException("unexpected Room player storage");
-        @SuppressWarnings("unchecked") Map<String, Room.Player> roster = (Map<String, Room.Player>) storage;
         for (int slot = 0; slot < players.size(); slot++) {
             JsonNode p = players.get(slot);
             if (p.path("slot").asInt(-1) != slot || p.path("spawn").get(0).asInt() != Room.SPAWNS[slot][0]
                     || p.path("spawn").get(1).asInt() != Room.SPAWNS[slot][1]) throw new IOException("initial slot/spawn mapping");
-            if (slot < players.size() - 1 && roster.put(p.path("player_id").asText(), new Room.Player(p.path("player_id").asText(), slot)) != null)
-                throw new IOException("duplicate fixture player");
+            room.join(p.path("player_id").asText());
         }
-        set(room, "nextSlot", players.size() - 1); return room;
+        return room;
     }
 
     private static void start(Room room, ObjectNode initial) throws IOException {
-        JsonNode last = initial.withArray("players").get(initial.withArray("players").size() - 1);
-        room.join(last.path("player_id").asText());
+        for (JsonNode player : initial.withArray("players")) room.ready(player.path("player_id").asText());
         if (room.status() != Room.Status.RUNNING) throw new IOException("unchanged start condition failed");
     }
 
@@ -84,6 +92,18 @@ final class ReplayFixture {
     static <T> T owned(ArenaServer server, Callable<T> operation) throws Exception {
         return ((ThreadPoolExecutor) field(server, "owner")).submit(operation).get(5, TimeUnit.SECONDS);
     }
+    static int udpReceived(ArenaServer server) throws Exception { return ((UdpData.Metrics) field(server, "udpMetrics")).received.get(); }
+    static ObjectNode udpBarrier(ArenaServer server, int minimumReceived) throws Exception {
+        UdpData.Metrics metrics = (UdpData.Metrics) field(server, "udpMetrics");
+        NioEventLoopGroup loop = (NioEventLoopGroup) field(server, "ioLoop");
+        long deadline = System.nanoTime() + TimeUnit.SECONDS.toNanos(5);
+        while (metrics.received.get() < minimumReceived) {
+            if (System.nanoTime() >= deadline) throw new IOException("UDP ingress barrier ceiling");
+            loop.submit(() -> { }).get(5, TimeUnit.SECONDS);
+        }
+        loop.submit(() -> { }).get(5, TimeUnit.SECONDS);
+        return owned(server, metrics::view);
+    }
     static Object field(Object object, String name) throws ReflectiveOperationException {
         Field field = object.getClass().getDeclaredField(name); field.setAccessible(true); return field.get(object);
     }
diff --git a/src/test/java/arena/ReplayFormatTest.java b/src/test/java/arena/ReplayFormatTest.java
index a162475..3cf4003 100644
--- a/src/test/java/arena/ReplayFormatTest.java
+++ b/src/test/java/arena/ReplayFormatTest.java
@@ -12,7 +12,7 @@ import org.junit.jupiter.api.Test;
 /** Zero simulation ticks: the separately budgeted command invocations own all G07 campaigns. */
 final class ReplayFormatTest {
     @Test void canonicalFieldOrderAsciiPlayersAndFinalLf() {
-        Room room = new Room("format-room"); room.join("z-player"); room.join("a-player");
+        Room room = new Room("format-room"); room.join("z-player"); room.join("a-player"); room.ready("z-player"); room.ready("a-player");
         String expected = "v=1|room=format-room|tick=-1|status=RUNNING|owner_epoch=0\n"
                 + "player=a-player|slot=1|x=90000|y=90000|dir=STOP|score=0|conn=CONNECTED|last_seq=0|last_tag_tick=-20\n"
                 + "player=z-player|slot=0|x=10000|y=10000|dir=STOP|score=0|conn=CONNECTED|last_seq=0|last_tag_tick=-20\n";
@@ -23,7 +23,7 @@ final class ReplayFormatTest {
     }
 
     @Test void immutableCaptureExplicitOverflowAndCloseRelease() {
-        Room room = new Room("capture-room"); room.join("first"); room.join("second");
+        Room room = new Room("capture-room"); room.join("first"); room.join("second"); room.ready("first"); room.ready("second");
         byte[] first = room.replayArtifact(), untouched = room.replayArtifact(); first[0] ^= 1;
         assertArrayEquals(untouched, room.replayArtifact()); room.close();
         assertEquals(0, room.replayStoredBytes()); assertThrows(IllegalStateException.class, room::replayArtifact);
@@ -48,7 +48,8 @@ final class ReplayFormatTest {
              URLClassLoader production = new URLClassLoader(new URL[] {path.toUri().toURL()}, ClassLoader.getPlatformClassLoader())) {
             assertNotNull(jar.getJarEntry("arena/ArenaServer.class"));
             assertNull(jar.getJarEntry("G08.json"));
-            for (String name : List.of("ReplayFixture", "ReplayScenario", "ReplayVerifier", "ReplayTool", "G07BaselineTest", "SnapshotScenario", "G08BaselineTest")) {
+            assertNull(jar.getJarEntry("G09.json"));
+            for (String name : List.of("ReplayFixture", "ReplayScenario", "ReplayVerifier", "ReplayTool", "G07BaselineTest", "SnapshotScenario", "G08BaselineTest", "UdpScenario", "UdpFaultProxy", "UdpBoundaryTest", "G09BaselineTest")) {
                 assertNull(jar.getJarEntry("arena/" + name + ".class"));
                 assertThrows(ClassNotFoundException.class, () -> Class.forName("arena." + name, false, production));
             }
diff --git a/src/test/java/arena/ReplayTool.java b/src/test/java/arena/ReplayTool.java
index 27222c0..b73c16f 100644
--- a/src/test/java/arena/ReplayTool.java
+++ b/src/test/java/arena/ReplayTool.java
@@ -20,10 +20,11 @@ public final class ReplayTool {
             if (Files.size(input) > 65_536) throw new IllegalArgumentException("scenario byte bound");
             ObjectNode scenario = Json.read(Files.readAllBytes(input));
             String thread = scenario.path("thread").asText();
-            if (thread.equals("G07") || thread.equals("G08")) {
+            if (thread.equals("G07") || thread.equals("G08") || thread.equals("G09")) {
                 boolean variant = args.length == 5 && args[3].equals("--variant") && args[4].equals("rejected-removed");
                 if (args.length != 3 && !(thread.equals("G07") && variant)) throw new IllegalArgumentException("unknown scenario variant");
-                ReplayScenario.Observed observed = thread.equals("G07") ? ReplayScenario.run(input, variant) : SnapshotScenario.run(input);
+                ReplayScenario.Observed observed = thread.equals("G07") ? ReplayScenario.run(input, variant)
+                        : thread.equals("G08") ? SnapshotScenario.run(input) : UdpScenario.run(input);
                 result = observed.result();
                 Path artifact = output.resolveSibling(output.getFileName().toString().replaceFirst("\\.json$", "") + ".replay.jsonl");
                 if (observed.replay() != null) {
diff --git a/src/test/java/arena/RoomTest.java b/src/test/java/arena/RoomTest.java
index 83ee60e..710e719 100644
--- a/src/test/java/arena/RoomTest.java
+++ b/src/test/java/arena/RoomTest.java
@@ -83,6 +83,8 @@ final class RoomTest {
         Room room = new Room("room-unit");
         room.join("player-a");
         room.join("player-b");
+        room.ready("player-a");
+        room.ready("player-b");
         return room;
     }
 
diff --git a/src/test/java/arena/ServerIntegrationTest.java b/src/test/java/arena/ServerIntegrationTest.java
index d663097..0b7bc80 100644
--- a/src/test/java/arena/ServerIntegrationTest.java
+++ b/src/test/java/arena/ServerIntegrationTest.java
@@ -26,6 +26,8 @@ final class ServerIntegrationTest {
             alpha.hello(); bravo.hello();
             String room = alpha.createRoom();
             alpha.join(room); bravo.join(room);
+            alpha.bind(); bravo.bind();
+            alpha.until("SNAPSHOT"); bravo.until("SNAPSHOT");
             ObjectNode attempt = alpha.auth("INPUT", room).put("direction", "EAST")
                     .put("seq", 1).put("target_tick", 0).put("owner_epoch", 0)
                     .put("position", 999_999).put("score", 999).putNull("tag_target_player_id");
@@ -107,7 +109,12 @@ final class ServerIntegrationTest {
                 assertEquals(143, child.exitValue(), "normal JVM SIGTERM exit");
                 client.expectClosed();
             }
-            ScenarioRunner.assertCleanup(Json.read(Files.readAllBytes(cleanup)));
+            ObjectNode observedCleanup = Json.read(Files.readAllBytes(cleanup));
+            ScenarioRunner.assertCleanup(observedCleanup);
+            ObjectNode observedChild = Json.MAPPER.createObjectNode().put("process_id", ProcessHandle.current().pid())
+                    .put("child_process_id", child.pid()).put("child_exit_code", child.exitValue()).put("child_alive", child.isAlive());
+            observedChild.set("ready", ready); observedChild.set("cleanup", observedCleanup);
+            System.out.println("G09 standalone child " + observedChild);
             try (ServerSocket released = new ServerSocket()) {
                 released.setReuseAddress(true);
                 released.bind(new InetSocketAddress("127.0.0.1", port));
diff --git a/src/test/java/arena/SnapshotScenario.java b/src/test/java/arena/SnapshotScenario.java
index bf8357c..1a1b2e0 100644
--- a/src/test/java/arena/SnapshotScenario.java
+++ b/src/test/java/arena/SnapshotScenario.java
@@ -3,7 +3,6 @@ package arena;
 import com.fasterxml.jackson.databind.JsonNode;
 import com.fasterxml.jackson.databind.node.ArrayNode;
 import com.fasterxml.jackson.databind.node.ObjectNode;
-import java.io.DataInputStream;
 import java.io.IOException;
 import java.nio.file.Files;
 import java.nio.file.Path;
@@ -14,7 +13,7 @@ import java.util.List;
 import java.util.Map;
 import java.util.TreeMap;
 
-/** One fixed 196-tick TCP run. HELLO markers only drain/observe the existing session stream. */
+/** One fixed 196-tick run. UDP PING markers only drain/observe the existing bound session stream. */
 final class SnapshotScenario {
     static final String SHA256 = "d121973167461dddbb3ba0bd339ed26b09486cda8bbec642328ccc5cea9e578e";
     static final List<String> VISIBLE_FIELDS = List.of("player_id", "slot", "x", "y", "direction", "score", "connectivity");
@@ -38,7 +37,7 @@ final class SnapshotScenario {
                 clients.put(role, client); replicas.put(role, new Replica()); client.hello(); client.playerId = player.path("player_id").asText();
             }
             ObjectNode bootstrap = ReplayFixture.bootstrap(server, scenario, clients); result.set("bootstrap", bootstrap);
-            bootstrap.put("publication_bridge", "test-only invocation of the production startReplication helper used by normal JOIN");
+            bootstrap.put("publication_bridge", "ordinary joins and real UDP binds invoke production readiness/publication");
             captureBoundary(server, scenario, clients, replicas, result, -1);
             int eventIndex = 0;
             for (int tick = 0; tick < scenario.path("ticks").asInt(); tick++) {
@@ -134,11 +133,14 @@ final class SnapshotScenario {
 
     private static ArrayNode marker(ArenaServer server, TcpClient client, ObjectNode result, String role, int tick, String phase) throws Exception {
         ObjectNode marker = result.withArray("observation_markers").addObject().put("client", role).put("tick", tick).put("phase", phase);
-        ArrayNode messages = marker.putArray("wire_messages"); client.send(Json.message("HELLO"));
+        ArrayNode messages = marker.putArray("wire_messages"); long nonce = result.withArray("observation_markers").size();
+        int received = ReplayFixture.udpReceived(server);
+        client.send(client.auth("PING", client.roomId).put("ping_id", nonce));
         for (int i = 0; i < 64; i++) {
             ObjectNode response = read(client);
-            if (response.path("type").asText().equals("WELCOME")) {
-                if (!client.sessionId.equals(response.path("session_id").asText())) throw new IOException("observation marker changed session");
+            if (response.path("type").asText().equals("PONG")) {
+                if (response.path("ping_id").asLong() != nonce) throw new IOException("observation marker correlation");
+                ReplayFixture.udpBarrier(server, received + 1);
                 marker.put("unchanged_session_id", client.sessionId).put("unchanged_player_id", client.playerId);
                 int observed = ReplayFixture.snapshot(server).path("tick").asInt(); marker.put("observed_tick", observed);
                 if (observed != tick) throw new IOException("observation marker advanced simulation");
@@ -199,9 +201,7 @@ final class SnapshotScenario {
     }
 
     private static ObjectNode read(TcpClient client) throws Exception {
-        DataInputStream input = (DataInputStream) ReplayFixture.field(client, "input"); int length = input.readInt();
-        if (length < 1 || length > CompleteFrame.MAX_BYTES) throw new IOException("server frame bound");
-        byte[] bytes = new byte[length]; input.readFully(bytes); return Json.read(bytes);
+        ObjectNode value = client.read(); if (value == null) throw new IOException("unexpected server EOF"); return value;
     }
     private static String hash(byte[] bytes) throws Exception {
         return HexFormat.of().formatHex(MessageDigest.getInstance("SHA-256").digest(bytes));
diff --git a/src/test/java/arena/SnapshotStreamTest.java b/src/test/java/arena/SnapshotStreamTest.java
index 1a2e03c..15eafa7 100644
--- a/src/test/java/arena/SnapshotStreamTest.java
+++ b/src/test/java/arena/SnapshotStreamTest.java
@@ -10,7 +10,7 @@ import org.junit.jupiter.api.Test;
 /** Pure snapshot fixtures; no simulation ticks or TCP campaign. */
 final class SnapshotStreamTest {
     @Test void retainOnlyLast32ImmutableStatesAndReleaseOnClose() throws Exception {
-        Room room = new Room("retention-room"); room.join("z-player"); room.join("a-player");
+        Room room = new Room("retention-room"); room.join("z-player"); room.join("a-player"); room.ready("z-player"); room.ready("a-player");
         SnapshotStream stream = new SnapshotStream(); ObjectNode last = null;
         for (int seq = 1; seq <= 33; seq++) {
             if (seq > 1) stream.acknowledge(seq == 33 ? 999 : seq - 1);
@@ -45,7 +45,7 @@ final class SnapshotStreamTest {
     }
 
     @Test void unknownFutureAcknowledgementCannotBecomeABaseLater() {
-        Room room = new Room("future-ack-room"); room.join("player-a"); room.join("player-b");
+        Room room = new Room("future-ack-room"); room.join("player-a"); room.join("player-b"); room.ready("player-a"); room.ready("player-b");
         SnapshotStream stream = new SnapshotStream();
         ObjectNode observed = Json.MAPPER.createObjectNode().put("probe", "G08 repair1 future ACK")
                 .put("process_id", ProcessHandle.current().pid()).put("unknown_ack_seq", 2);
diff --git a/src/test/java/arena/UdpBoundaryTest.java b/src/test/java/arena/UdpBoundaryTest.java
new file mode 100644
index 0000000..1705517
--- /dev/null
+++ b/src/test/java/arena/UdpBoundaryTest.java
@@ -0,0 +1,276 @@
+package arena;
+
+import static org.junit.jupiter.api.Assertions.assertTrue;
+import com.fasterxml.jackson.databind.JsonNode;
+import com.fasterxml.jackson.databind.node.ArrayNode;
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import java.io.IOException;
+import java.lang.reflect.Method;
+import java.net.InetSocketAddress;
+import java.nio.ByteBuffer;
+import java.nio.channels.DatagramChannel;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.util.ArrayList;
+import java.util.List;
+import java.util.Map;
+import java.util.concurrent.TimeUnit;
+import java.util.concurrent.atomic.AtomicLong;
+import org.junit.jupiter.api.Test;
+
+/** The frozen eleven-case boundary matrix runs once here, with zero simulation ticks. */
+final class UdpBoundaryTest {
+    @Test void reservedUdpPortDoesNotBlockTcpStartup() throws Exception {
+        ObjectNode result = Json.MAPPER.createObjectNode().put("thread", "G09").put("attempt", "repair1")
+                .put("kind", "one reserved UDP endpoint with the same requested TCP number")
+                .put("process_id", ProcessHandle.current().pid()).put("executed_ticks", 0);
+        ArenaServer server = null; TcpClient client = null; DatagramChannel reservation = null;
+        try {
+            reservation = DatagramChannel.open(); reservation.bind(new InetSocketAddress("127.0.0.1", 0));
+            int reservedPort = ((InetSocketAddress) reservation.getLocalAddress()).getPort();
+            result.put("udp_reservation_established", true).put("server_constructor_started", true);
+            server = new ArenaServer("127.0.0.1", reservedPort, true);
+            result.put("server_started", true).put("requested_tcp_number_preserved", server.port() == reservedPort)
+                    .put("udp_allocated_independently", server.udpPort() != reservedPort);
+            require(server.port() == reservedPort && server.udpPort() != reservedPort, "TCP/UDP allocation coupling");
+            client = new TcpClient(server.port()); ObjectNode welcome = client.hello();
+            result.put("welcome_advertises_actual_udp_port", welcome.path("udp_port").asInt() == server.udpPort());
+            require(result.path("welcome_advertises_actual_udp_port").asBoolean(), "WELCOME UDP endpoint advertisement");
+            // No destination override: the ordinary client must consume WELCOME's endpoint.
+            ObjectNode bound = client.bind(); result.set("response", bound);
+            require(bound.path("type").asText().equals("UDP_BOUND"), "advertised UDP endpoint bind");
+            ObjectNode metrics = server.metrics(); result.set("runtime_metrics", metrics);
+            require(metrics.path("manual_time_ns").asLong() == 0 && !metrics.path("clock").path("active").asBoolean(),
+                    "port-allocation regression executed simulation time");
+            result.put("passed", true);
+        } catch (Exception failure) {
+            result.put("passed", false).put("failure_type", failure.getClass().getName());
+            java.io.StringWriter trace = new java.io.StringWriter(); failure.printStackTrace(new java.io.PrintWriter(trace));
+            result.put("failure_stack", trace.toString());
+            throw failure;
+        } finally {
+            if (server != null) { server.close(); result.set("cleanup", server.cleanup()); ScenarioRunner.assertCleanup(server.cleanup()); }
+            if (client != null) { client.close(); require(client.isClosed(), "regression client cleanup"); }
+            if (reservation != null) { reservation.close(); result.put("udp_reservation_closed", !reservation.isOpen()); }
+            result.put("arena_threads_alive_after_test", Thread.getAllStackTraces().keySet().stream()
+                    .filter(thread -> thread.getName().startsWith("arena-") && thread.isAlive()).count());
+            Files.write(Path.of("build/g09-port-allocation-result.json"),
+                    Json.MAPPER.writerWithDefaultPrettyPrinter().writeValueAsBytes(result));
+            System.out.println(result);
+        }
+    }
+
+    @Test void fixedBindingTransportAndByteMatrix() throws Exception {
+        byte[] bytes;
+        try (var input = getClass().getResourceAsStream("/G09.json")) {
+            if (input == null) throw new IOException("missing frozen G09 resource"); bytes = input.readAllBytes();
+        }
+        require(UdpScenario.SHA256.equals(UdpScenario.hash(bytes)), "frozen matrix bytes");
+        ObjectNode fixture = Json.read(bytes), result = Json.MAPPER.createObjectNode().put("thread", "G09")
+                .put("kind", "fixed boundary matrix").put("process_id", ProcessHandle.current().pid()).put("executed_ticks", 0);
+        ArrayNode cases = result.putArray("cases"), failures = result.putArray("failures");
+        for (JsonNode definition : fixture.withArray("fixed_matrix")) {
+            ObjectNode cell = cases.addObject().put("name", definition.path("name").asText());
+            try { runCase(definition.path("name").asText(), cell); cell.put("passed", true); }
+            catch (Exception failure) { cell.put("passed", false); failures.add(cell.path("name").asText() + ": " + failure.getMessage()); }
+        }
+        try { result.set("outbound_bound_probe", outboundBound()); }
+        catch (Exception failure) { failures.add("outbound bound: " + failure.getMessage()); }
+        result.put("passed", failures.isEmpty());
+        Files.write(Path.of("build/g09-matrix-result.json"), Json.MAPPER.writerWithDefaultPrettyPrinter().writeValueAsBytes(result));
+        System.out.println(result);
+        assertTrue(failures.isEmpty(), failures.toString());
+    }
+
+    private static void runCase(String name, ObjectNode cell) throws Exception {
+        World world = new World(2);
+        try {
+            TcpClient alpha = world.clients.get(0), bravo = world.clients.get(1);
+            if (name.equals("valid-token-before-deadline")) {
+                world.clock.set(TimeUnit.MILLISECONDS.toNanos(4_999));
+                ObjectNode response = alpha.bind(); cell.set("response", response); bravo.bind();
+                cell.put("bind_before_room_join", true);
+                world.joinAll(); world.drainStart();
+                require(response.path("type").asText().equals("UDP_BOUND"), "4999ms token rejected");
+                require(ReplayFixture.snapshot(world.server).path("status").asText().equals("RUNNING"), "bound session JOIN readiness");
+                cell.put("manual_ms_after_issue", 4_999); cell.set("after", ReplayFixture.snapshot(world.server));
+                cell.set("binding", world.binding(alpha)); return;
+            }
+            world.joinAll();
+            boolean running = List.of("reused-consumed-token", "INPUT-from-different-observed-endpoint", "realtime-INPUT-over-TCP",
+                    "valid-INPUT-exactly1200bytes", "oversize-datagram1201bytes").contains(name);
+            if (running) { alpha.bind(); bravo.bind(); world.drainStart(); }
+            ObjectNode before = ReplayFixture.snapshot(world.server), bindingBefore = world.binding(alpha);
+            cell.set("before", before); cell.set("binding_before", bindingBefore);
+            ObjectNode response;
+            switch (name) {
+                case "expired-token-at-deadline" -> {
+                    world.clock.set(TimeUnit.MILLISECONDS.toNanos(5_000)); alpha.send(alpha.binding()); response = alpha.control();
+                    cell.put("manual_ms_after_issue", 5_000);
+                }
+                case "unknown-token" -> {
+                    alpha.send(alpha.binding().put("udp_bind_token", "not-an-issued-credential")); response = alpha.control();
+                }
+                case "reused-consumed-token" -> {
+                    world.foreign(alpha.binding()); response = alpha.control();
+                }
+                case "other-session-token" -> {
+                    ObjectNode attempt = alpha.binding(); attempt.set("udp_bind_token", bravo.binding().path("udp_bind_token"));
+                    alpha.send(attempt); response = alpha.control();
+                }
+                case "INPUT-before-bind" -> { alpha.send(input(alpha, world.room)); response = alpha.control(); }
+                case "INPUT-from-different-observed-endpoint" -> { world.foreign(input(alpha, world.room)); response = alpha.control(); }
+                case "wrong-owner-epoch-bind" -> { alpha.send(alpha.binding().put("owner_epoch", 1)); response = alpha.control(); }
+                case "realtime-INPUT-over-TCP" -> { alpha.sendTcp(input(alpha, world.room)); response = alpha.control(); }
+                case "valid-INPUT-exactly1200bytes" -> {
+                    ObjectNode request = padded(input(alpha, world.room), 1_200);
+                    cell.put("actual_datagram_bytes", Json.bytes(request).length); alpha.send(request); response = alpha.control();
+                }
+                case "oversize-datagram1201bytes" -> {
+                    byte[] request = Json.bytes(padded(input(alpha, world.room), 1_201));
+                    int received = ReplayFixture.udpReceived(world.server);
+                    ObjectNode metrics = (ObjectNode) world.server.metrics().path("udp");
+                    DatagramChannel channel = (DatagramChannel) ReplayFixture.field(alpha, "udp");
+                    int sent = channel.send(ByteBuffer.wrap(request), world.endpoint()); require(sent == 1_201, "incomplete oversize send");
+                    ObjectNode processed = ReplayFixture.udpBarrier(world.server, received + 1);
+                    require(processed.path("oversize_dropped").asInt() == metrics.path("oversize_dropped").asInt() + 1, "oversize metric");
+                    require(processed.path("dispatched").asInt() == metrics.path("dispatched").asInt(), "oversize owner dispatch");
+                    cell.put("actual_datagram_bytes", sent).set("ingress_barrier", processed);
+                    alpha.send(alpha.auth("PING", world.room)); response = alpha.control();
+                    require(response.path("type").asText().equals("PONG"), "oversize emitted an ACK/error before PONG barrier");
+                    cell.put("input_ack_count", 0); break;
+                }
+                default -> throw new IOException("unknown fixed matrix case");
+            }
+            cell.set("response", response); ObjectNode after = ReplayFixture.snapshot(world.server); cell.set("after", after);
+            cell.set("binding_after", world.binding(alpha));
+            if (name.equals("valid-INPUT-exactly1200bytes")) {
+                require(response.path("type").asText().equals("INPUT_ACK") && response.path("status").asText().equals("ACCEPTED"), "1200-byte admission");
+                JsonNode player = ReplayScenario.player(after, alpha.playerId);
+                require(player.path("last_accepted_seq").asInt() == 1 && player.path("pending_inputs").asInt() == 1, "1200-byte pending intent");
+                require(player.path("x").asInt() == 10_000 && player.path("y").asInt() == 10_000, "admission simulated a tick");
+            } else {
+                require(before.equals(after), "rejected datagram changed Room/sequence");
+                if (!name.equals("oversize-datagram1201bytes")) {
+                    String expected = name.equals("realtime-INPUT-over-TCP") ? "WRONG_TRANSPORT" : "UDP_BIND_INVALID";
+                    require(response.path("type").asText().equals("ERROR") && response.path("code").asText().equals(expected), "wrong stable error code");
+                }
+                require(bindingBefore.equals(world.binding(alpha)), "rejection changed established endpoint/token");
+            }
+            if (name.equals("other-session-token")) {
+                ObjectNode valid = bravo.bind(); cell.put("other_session_token_still_usable", valid.path("type").asText().equals("UDP_BOUND"));
+                require(cell.path("other_session_token_still_usable").asBoolean(), "other-session token was consumed");
+            }
+            require(after.path("tick").asInt() == -1, "boundary matrix executed ticks");
+        } finally {
+            cell.set("runtime_metrics", world.server.metrics()); world.close(); cell.set("cleanup", world.cleanup());
+        }
+    }
+
+    private static ObjectNode outboundBound() throws Exception {
+        World world = new World(8); ObjectNode result = Json.MAPPER.createObjectNode().put("actual_tick_calls", 0).put("probe", "pure serializer field widths plus actual outbound rejection");
+        try {
+            world.joinAll(); for (TcpClient client : world.clients) client.bind();
+            ArrayNode wire = result.putArray("initial_full_bytes");
+            for (TcpClient client : world.clients) {
+                ObjectNode snapshot = client.until("SNAPSHOT"); int size = Json.bytes(snapshot).length;
+                require(snapshot.withArray("players").size() == 8 && size <= 1_200, "normal eight-player FULL bound"); wire.add(size);
+            }
+            result.put("generated_room_id_length", world.room.length()); ArrayNode lengths = result.putArray("generated_player_id_lengths");
+            for (TcpClient client : world.clients) lengths.add(client.playerId.length());
+            ObjectNode maximum = widthFixture(world.room, world.clients.stream().map(client -> client.playerId).toList());
+            byte[] maximumBytes = UdpData.bytes(maximum);
+            result.put("worst_visible_envelope_bytes", maximumBytes.length).set("worst_visible_envelope", maximum);
+            List<String> wide = new ArrayList<>(); for (int i = 0; i < 8; i++) wide.add("p" + "x".repeat(62) + i);
+            ObjectNode oversized = widthFixture("r".repeat(64), wide); int oversizedBytes = Json.bytes(oversized).length;
+            boolean rejected = false;
+            try { UdpData.bytes(oversized); } catch (IllegalArgumentException expected) { rejected = true; }
+            require(rejected && oversizedBytes > 1_200, "maximum-identifier fixture must explicitly reject");
+            for (String id : wide) Json.identifier(Json.MAPPER.createObjectNode().put("id", id), "id");
+            Json.identifier(Json.MAPPER.createObjectNode().put("id", "r".repeat(64)), "id");
+            TcpClient alpha = world.clients.get(0);
+            ReplayFixture.owned(world.server, () -> {
+                for (var entry : ((Map<?, ?>) ReplayFixture.field(world.server, "sessions")).entrySet())
+                    if (ReplayFixture.field(entry.getValue(), "id").equals(alpha.sessionId)) {
+                        Method send = entry.getKey().getClass().getDeclaredMethod("realtime", ObjectNode.class, InetSocketAddress.class);
+                        send.setAccessible(true); send.invoke(entry.getKey(), oversized, alpha.localUdpEndpoint());
+                    }
+                return null;
+            });
+            alpha.send(alpha.auth("PING", world.room)); require(alpha.control().path("type").asText().equals("PONG"), "oversize output escaped bound");
+            require(world.server.metrics().path("udp").path("outbound_oversize_rejected").asInt() == 1, "outbound rejection metric");
+            result.put("oversized_fixture_bytes", oversizedBytes).put("oversized_fixture_rejected", true)
+                    .put("incoming_64byte_identifiers_valid", true).put("outbound_oversize_rejected", 1);
+            result.set("runtime_metrics", world.server.metrics()); return result;
+        } finally { world.close(); result.set("cleanup", world.cleanup()); }
+    }
+
+    private static ObjectNode widthFixture(String roomId, List<String> identifiers) {
+        Room room = new Room(roomId); SnapshotStream stream = new SnapshotStream();
+        try {
+            for (String id : identifiers) room.join(id);
+            for (String id : identifiers) { Room.Player player = room.player(id); player.x = 100_000; player.y = 100_000; player.direction = Room.Direction.NORTH; player.score = 10; }
+            for (String id : identifiers) room.ready(id);
+            // Pure serializer width fixture; no simulation/clock iteration is invoked.
+            return stream.next(room, true).put("tick", 1_199).put("snapshot_seq", 601).put("status", "FINISHED");
+        } finally { stream.close(); room.close(); }
+    }
+
+    private static ObjectNode input(TcpClient client, String room) {
+        return client.auth("INPUT", room).put("seq", 1).put("target_tick", 0).put("direction", "EAST").putNull("tag_target_player_id");
+    }
+    private static ObjectNode padded(ObjectNode request, int length) throws IOException {
+        request.put("padding", ""); int extra = length - Json.bytes(request).length;
+        require(extra >= 0, "padding fixture length"); request.put("padding", "x".repeat(extra));
+        require(Json.bytes(request).length == length, "exact byte fixture"); return request;
+    }
+    private static void require(boolean condition, String message) throws IOException { if (!condition) throw new IOException(message); }
+
+    private static final class World implements AutoCloseable {
+        final AtomicLong clock = new AtomicLong();
+        final ArenaServer server = new ArenaServer("127.0.0.1", 0, clock::get);
+        final List<TcpClient> clients = new ArrayList<>();
+        final List<Object> retainedSessions = new ArrayList<>();
+        String room;
+        World(int count) throws Exception {
+            try {
+                for (int i = 0; i < count; i++) { TcpClient client = new TcpClient(server.port()); clients.add(client); client.hello(); }
+                retainedSessions.addAll(ReplayFixture.owned(server, () -> new ArrayList<>(((Map<?, ?>) ReplayFixture.field(server, "sessions")).values())));
+            } catch (Exception failure) { close(); throw failure; }
+        }
+        void joinAll() throws Exception {
+            room = clients.get(0).createRoom();
+            for (TcpClient client : clients) client.join(room);
+        }
+        void drainStart() throws IOException { for (TcpClient client : clients) client.until("SNAPSHOT"); }
+        InetSocketAddress endpoint() { return new InetSocketAddress("127.0.0.1", server.udpPort()); }
+        void foreign(ObjectNode request) throws Exception {
+            int received = ReplayFixture.udpReceived(server);
+            try (DatagramChannel channel = DatagramChannel.open()) {
+                channel.bind(new InetSocketAddress("127.0.0.1", 0)); byte[] bytes = UdpData.bytes(request);
+                require(channel.send(ByteBuffer.wrap(bytes), endpoint()) == bytes.length, "foreign endpoint atomic send");
+                ReplayFixture.udpBarrier(server, received + 1);
+            }
+        }
+        ObjectNode binding(TcpClient client) throws Exception {
+            return ReplayFixture.owned(server, () -> {
+                for (Object session : ((Map<?, ?>) ReplayFixture.field(server, "sessions")).values())
+                    if (ReplayFixture.field(session, "id").equals(client.sessionId)) {
+                        Object endpoint = ReplayFixture.field(session, "endpoint");
+                        return Json.MAPPER.createObjectNode().put("token_present", ReplayFixture.field(session, "bindToken") != null)
+                                .put("endpoint_bound", endpoint != null).put("endpoint_matches_original", client.localUdpEndpoint().equals(endpoint));
+                    }
+                throw new IOException("session missing");
+            });
+        }
+        ObjectNode cleanup() throws Exception {
+            ObjectNode cleanup = server.cleanup(); ScenarioRunner.assertCleanup(cleanup);
+            boolean released = clients.stream().allMatch(TcpClient::isClosed);
+            for (Object session : retainedSessions) released &= ReplayFixture.field(session, "bindToken") == null && ReplayFixture.field(session, "endpoint") == null;
+            cleanup.put("client_and_binding_resources_released", released);
+            cleanup.put("session_registry_size", ((Map<?, ?>) ReplayFixture.field(server, "sessions")).size());
+            require(released && cleanup.path("session_registry_size").asInt() == 0, "client/binding cleanup"); return cleanup;
+        }
+        @Override public void close() { server.close(); for (TcpClient client : clients) client.close(); }
+    }
+}
diff --git a/src/test/java/arena/UdpFaultProxy.java b/src/test/java/arena/UdpFaultProxy.java
new file mode 100644
index 0000000..e7faa6f
--- /dev/null
+++ b/src/test/java/arena/UdpFaultProxy.java
@@ -0,0 +1,109 @@
+package arena;
+
+import com.fasterxml.jackson.databind.JsonNode;
+import com.fasterxml.jackson.databind.node.ArrayNode;
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import java.io.IOException;
+import java.net.DatagramPacket;
+import java.net.DatagramSocket;
+import java.net.InetSocketAddress;
+import java.util.Arrays;
+import java.util.List;
+
+/** Synchronous test-only proxy. Each call consumes one known datagram; no thread, timer or sleep. */
+final class UdpFaultProxy implements AutoCloseable {
+    record Transit(String type, ObjectNode original, int bytes, List<Integer> deliveredOrdinals) { }
+    private final DatagramSocket front;
+    private final DatagramSocket back;
+    private final InetSocketAddress client;
+    private final InetSocketAddress server;
+    private final Stream inputs = new Stream("INPUT"), snapshots = new Stream("SNAPSHOT");
+    private final ArrayNode controls = Json.MAPPER.createArrayNode();
+    private int pendingHighWater;
+    private int byteHighWater;
+
+    private static final class Stream {
+        final String type;
+        final ArrayNode originals = Json.MAPPER.createArrayNode(), deliveries = Json.MAPPER.createArrayNode();
+        int ordinal;
+        byte[] held;
+        int heldOrdinal;
+        Stream(String type) { this.type = type; }
+    }
+
+    UdpFaultProxy(InetSocketAddress client, InetSocketAddress server, JsonNode faults) throws IOException {
+        if (!faults.path("drop_indices").toString().equals("[5]") || !faults.path("duplicate_indices").toString().equals("[8]")
+                || !faults.path("swap_adjacent_indices").toString().equals("[[10,11]]")) throw new IOException("fixed G09 fault indices required");
+        this.client = client; this.server = server;
+        front = new DatagramSocket(new InetSocketAddress("127.0.0.1", 0));
+        try { back = new DatagramSocket(new InetSocketAddress("127.0.0.1", 0)); }
+        catch (IOException failure) { front.close(); throw failure; }
+        front.setSoTimeout(5_000); back.setSoTimeout(5_000);
+    }
+
+    InetSocketAddress clientDestination() { return (InetSocketAddress) front.getLocalSocketAddress(); }
+    InetSocketAddress serverSource() { return (InetSocketAddress) back.getLocalSocketAddress(); }
+    Transit clientPacket() throws IOException { return forward(true); }
+    Transit serverPacket() throws IOException { return forward(false); }
+
+    private Transit forward(boolean fromClient) throws IOException {
+        byte[] storage = new byte[UdpData.MAX_BYTES + 1]; DatagramPacket packet = new DatagramPacket(storage, storage.length);
+        (fromClient ? front : back).receive(packet);
+        if (!packet.getSocketAddress().equals(fromClient ? client : server)) throw new IOException("proxy observed unexpected endpoint");
+        if (packet.getLength() < 1 || packet.getLength() > UdpData.MAX_BYTES) throw new IOException("proxy datagram bound");
+        byte[] bytes = Arrays.copyOf(storage, packet.getLength()); byteHighWater = Math.max(byteHighWater, bytes.length);
+        ObjectNode message = Json.read(bytes); String type = Json.text(message, "type");
+        Stream stream = fromClient ? inputs : snapshots;
+        ObjectNode safe = message.deepCopy(); safe.remove(List.of("udp_bind_token", "resume_token"));
+        if (!type.equals(stream.type)) {
+            send(fromClient, bytes);
+            controls.addObject().put("direction", fromClient ? "client_to_server" : "server_to_client")
+                    .put("type", type).put("bytes", bytes.length);
+            return new Transit(type, safe, bytes.length, List.of(0));
+        }
+        int ordinal = ++stream.ordinal;
+        ObjectNode observation = stream.originals.addObject().put("ordinal", ordinal).put("bytes", bytes.length);
+        if (fromClient) {
+            safe.remove("session_id"); safe.put("session_alias", "alpha-session");
+            observation.set("input", safe);
+        } else observation.set("snapshot", safe);
+        List<Integer> delivered;
+        if (ordinal == 5) { observation.put("fault", "drop"); delivered = List.of(); }
+        else if (ordinal == 8) {
+            observation.put("fault", "duplicate"); deliver(stream, fromClient, bytes, ordinal); deliver(stream, fromClient, bytes, ordinal);
+            delivered = List.of(ordinal, ordinal);
+        } else if (ordinal == 10) {
+            if (pendingPackets() != 0) throw new IOException("proxy pending packet bound");
+            stream.held = bytes; stream.heldOrdinal = ordinal; pendingHighWater = Math.max(pendingHighWater, pendingPackets());
+            observation.put("fault", "hold_until_11"); delivered = List.of();
+        } else if (ordinal == 11) {
+            if (stream.held == null || stream.heldOrdinal != 10) throw new IOException("swap lacks original10");
+            observation.put("fault", "deliver_11_then_10"); deliver(stream, fromClient, bytes, ordinal);
+            deliver(stream, fromClient, stream.held, stream.heldOrdinal); stream.held = null; stream.heldOrdinal = 0;
+            delivered = List.of(11, 10);
+        } else { observation.put("fault", "pass"); deliver(stream, fromClient, bytes, ordinal); delivered = List.of(ordinal); }
+        ArrayNode order = observation.putArray("delivered_ordinals"); delivered.forEach(order::add);
+        observation.put("pending_packets_after", pendingPackets());
+        return new Transit(type, safe, bytes.length, delivered);
+    }
+
+    private void deliver(Stream stream, boolean fromClient, byte[] bytes, int ordinal) throws IOException {
+        send(fromClient, bytes); stream.deliveries.add(ordinal);
+    }
+    private void send(boolean fromClient, byte[] bytes) throws IOException {
+        (fromClient ? back : front).send(new DatagramPacket(bytes, bytes.length, fromClient ? server : client));
+    }
+    private int pendingPackets() { return (inputs.held == null ? 0 : 1) + (snapshots.held == null ? 0 : 1); }
+    ObjectNode view() {
+        ObjectNode result = Json.MAPPER.createObjectNode().put("input_original_count", inputs.ordinal).put("snapshot_original_count", snapshots.ordinal)
+                .put("pending_packets", pendingPackets()).put("pending_packet_high_water", pendingHighWater).put("datagram_byte_high_water", byteHighWater);
+        result.set("input_originals", inputs.originals); result.set("input_delivery_order", inputs.deliveries);
+        result.set("snapshot_originals", snapshots.originals); result.set("snapshot_delivery_order", snapshots.deliveries);
+        result.set("unaffected_control_datagrams", controls); return result;
+    }
+    ObjectNode cleanup() {
+        return Json.MAPPER.createObjectNode().put("front_socket_closed", front.isClosed()).put("back_socket_closed", back.isClosed())
+                .put("pending_packets", pendingPackets()).put("live_threads", 0);
+    }
+    @Override public void close() { inputs.held = null; snapshots.held = null; front.close(); back.close(); }
+}
diff --git a/src/test/java/arena/UdpScenario.java b/src/test/java/arena/UdpScenario.java
new file mode 100644
index 0000000..65fdf90
--- /dev/null
+++ b/src/test/java/arena/UdpScenario.java
@@ -0,0 +1,229 @@
+package arena;
+
+import com.fasterxml.jackson.databind.JsonNode;
+import com.fasterxml.jackson.databind.node.ArrayNode;
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import java.io.IOException;
+import java.net.InetSocketAddress;
+import java.nio.charset.StandardCharsets;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.security.MessageDigest;
+import java.util.ArrayList;
+import java.util.HexFormat;
+import java.util.LinkedHashMap;
+import java.util.List;
+import java.util.Map;
+import java.util.TreeMap;
+
+/** Exactly one frozen 24-tick fault campaign. Full suites invoke only UdpBoundaryTest. */
+final class UdpScenario {
+    static final String SHA256 = "9519b687243a622f4e63675bb3690e8900bf7616337647f18de73e1bcc27edfa";
+    private static final List<Integer> SNAPSHOT_ORDER = List.of(1, 2, 3, 4, 6, 7, 8, 8, 9, 11, 10, 12, 13);
+    private UdpScenario() { }
+
+    static ReplayScenario.Observed run(Path path) throws Exception {
+        byte[] bytes = Files.readAllBytes(path); require(SHA256.equals(hash(bytes)), "frozen G09 scenario bytes");
+        ObjectNode scenario = Json.read(bytes), result = Json.MAPPER.createObjectNode().put("thread", "G09")
+                .put("contract_version", 1).put("scenario_sha256", SHA256).put("process_id", ProcessHandle.current().pid())
+                .put("fault_campaigns", 1).put("load_runs", 0);
+        ArrayNode failures = result.putArray("failures"), ticks = result.putArray("ticks");
+        ArrayNode hashes = result.putArray("state_hashes"), records = result.putArray("canonical_records");
+        ArrayNode admissions = result.putArray("admissions"), bindings = result.putArray("bindings");
+        result.putArray("published_snapshots"); result.putArray("wire_captures"); result.putArray("alpha_received_snapshot_ordinals");
+        Map<String, TcpClient> clients = new LinkedHashMap<>(); Map<String, Replica> replicas = new LinkedHashMap<>();
+        List<Integer> accepted = new ArrayList<>(); int duplicateAcks = 0, staleErrors = 0;
+        ArenaServer server = new ArenaServer("127.0.0.1", 0, true); UdpFaultProxy proxy = null; byte[] artifact = null;
+        try {
+            for (JsonNode player : scenario.withArray("players")) {
+                String role = player.path("client").asText(); TcpClient client = new TcpClient(server.port());
+                clients.put(role, client); replicas.put(role, new Replica());
+                ObjectNode welcome = client.hello(); require(welcome.path("udp_bind_token_present").asBoolean(), "WELCOME missing server-issued token");
+            }
+            require(clients.values().stream().map(client -> client.sessionId).distinct().count() == 4, "four distinct sessions");
+            result.put("server_issued_credentials_present", true).put("distinct_session_count", 4);
+            result.set("normal_joins", ReplayFixture.joinFixed(server, scenario, clients));
+            TcpClient alpha = clients.get("alpha");
+            proxy = new UdpFaultProxy(alpha.localUdpEndpoint(), alpha.udpDestination(), scenario.path("faults"));
+            alpha.udpDestination(proxy.clientDestination());
+            for (var entry : clients.entrySet()) {
+                TcpClient client = entry.getValue(); ObjectNode response;
+                if (entry.getKey().equals("alpha")) {
+                    client.send(client.binding()); require(proxy.clientPacket().type().equals("UDP_BIND"), "bind proxy mapping");
+                    require(proxy.serverPacket().type().equals("UDP_BOUND"), "bound response transport"); response = client.until("UDP_BOUND");
+                } else response = client.bind();
+                ObjectNode binding = bindings.addObject().put("client", entry.getKey()).put("session_alias", entry.getKey() + "-session")
+                        .put("endpoint_alias", entry.getKey().equals("alpha") ? "alpha-proxy-server-endpoint" : entry.getKey() + "-endpoint");
+                binding.set("response", response);
+                InetSocketAddress expected = entry.getKey().equals("alpha") ? proxy.serverSource() : client.localUdpEndpoint();
+                binding.set("observed", binding(server, client, expected));
+                String status = ReplayFixture.snapshot(server).path("status").asText(); binding.put("room_status", status);
+                require(status.equals(bindings.size() < 4 ? "LOBBY" : "RUNNING"), "min2 AND all joined ready");
+            }
+            ObjectNode initial = ReplayFixture.snapshot(server); result.set("initial_state", initial);
+            capture(server, scenario, clients, replicas, proxy, result, -1);
+            int eventIndex = 0;
+            for (int tick = 0; tick < 24; tick++) {
+                JsonNode event = scenario.withArray("events").get(eventIndex++);
+                require(event.path("before_tick").asInt() == tick && event.path("seq").asInt() == tick + 1, "fixed input boundary");
+                ObjectNode request = alpha.auth("INPUT", scenario.path("room_id").asText());
+                for (String field : List.of("seq", "target_tick", "direction", "owner_epoch")) request.set(field, event.path(field));
+                request.putNull("tag_target_player_id");
+                ObjectNode admission = admissions.addObject().put("before_tick", tick).put("original_ordinal", tick + 1);
+                admission.set("before", ReplayFixture.snapshot(server));
+                int received = ReplayFixture.udpReceived(server); alpha.send(request); UdpFaultProxy.Transit transit = proxy.clientPacket();
+                require(transit.type().equals("INPUT"), "input proxy mapping");
+                ArrayNode delivered = admission.putArray("delivered_ordinals"); transit.deliveredOrdinals().forEach(delivered::add);
+                admission.set("ingress_barrier", ReplayFixture.udpBarrier(server, received + transit.deliveredOrdinals().size()));
+                int udpAcks = transit.deliveredOrdinals().size() - (transit.deliveredOrdinals().contains(10) ? 1 : 0);
+                for (int i = 0; i < udpAcks; i++) require(proxy.serverPacket().type().equals("INPUT_ACK"), "INPUT_ACK UDP transport");
+                ArrayNode responses = admission.putArray("responses");
+                for (int ignored : transit.deliveredOrdinals()) {
+                    ObjectNode response = alpha.control(); responses.add(response);
+                    if (response.path("type").asText().equals("INPUT_ACK")) {
+                        int seq = response.path("seq").asInt();
+                        if (response.path("status").asText().equals("ACCEPTED")) accepted.add(seq);
+                        else { require(seq == 8 && response.path("status").asText().equals("DUPLICATE"), "unexpected duplicate ACK"); duplicateAcks++; }
+                    } else {
+                        require(tick == 10 && response.path("code").asText().equals("INPUT_STALE"), "held10 must retain stale-before-window precedence"); staleErrors++;
+                    }
+                }
+                admission.set("after", ReplayFixture.snapshot(server));
+                server.advanceTicks(1); ObjectNode state = ReplayFixture.snapshot(server);
+                ticks.add(state); hashes.add(state.path("state_hash")); records.add(ReplayFixture.canonicalRecord(server));
+                checkState(state, tick);
+                if ((tick + 1) % 2 == 0) capture(server, scenario, clients, replicas, proxy, result, tick);
+            }
+            List<Integer> expected = new ArrayList<>(); for (int seq = 1; seq <= 24; seq++) if (seq != 5 && seq != 10) expected.add(seq);
+            require(accepted.equals(expected) && duplicateAcks == 1 && staleErrors == 1, "fixed authority admissions");
+            ArrayNode acceptedSeqs = result.putArray("accepted_sequences"); accepted.forEach(acceptedSeqs::add);
+            result.put("unique_accepted_count", accepted.size()).put("duplicate_ack_count", duplicateAcks).put("stale_error_count", staleErrors);
+            require(integers(result.withArray("alpha_received_snapshot_ordinals")).equals(SNAPSHOT_ORDER), "actual snapshot receive order");
+            ObjectNode faultTrace = proxy.view(); result.set("fault_trace", faultTrace);
+            require(faultTrace.path("input_original_count").asInt() == 24 && faultTrace.path("snapshot_original_count").asInt() == 13
+                    && faultTrace.path("pending_packets").asInt() == 0 && faultTrace.path("pending_packet_high_water").asInt() == 1, "fixed proxy bounds/counts");
+            ObjectNode counts = result.putObject("snapshot_counts");
+            for (var entry : replicas.entrySet()) counts.putObject(entry.getKey()).put("received", entry.getValue().received)
+                    .put("applied", entry.getValue().applied).put("highest_applied", entry.getValue().sequence);
+            result.set("final_state", ReplayFixture.snapshot(server)); artifact = server.replayArtifact();
+            result.put("artifact_bytes", artifact.length); ArrayNode journal = result.putArray("accepted_journal");
+            for (String line : new String(artifact, StandardCharsets.UTF_8).split("\n")) {
+                ObjectNode entry = Json.read(line.getBytes(StandardCharsets.UTF_8)); if (entry.path("kind").asText().equals("INPUT")) journal.add(entry);
+            }
+            require(journal.size() == 22, "actual accepted journal count");
+            result.set("runtime_metrics", server.metrics());
+            server.close(); for (TcpClient client : clients.values()) client.expectClosed();
+        } catch (Exception failure) {
+            failures.add(failure.getClass().getName() + ": " + failure.getMessage());
+            java.io.StringWriter trace = new java.io.StringWriter(); failure.printStackTrace(new java.io.PrintWriter(trace));
+            result.put("execution_error", trace.toString());
+        } finally {
+            if (proxy != null) { result.set("fault_trace", proxy.view()); proxy.close(); result.set("proxy_cleanup", proxy.cleanup()); }
+            server.close(); for (TcpClient client : clients.values()) client.close();
+        }
+        result.put("executed_ticks", ticks.size()); result.set("cleanup", server.cleanup()); ScenarioRunner.assertCleanup(server.cleanup());
+        boolean released = clients.values().stream().allMatch(TcpClient::isClosed)
+                && result.path("proxy_cleanup").path("front_socket_closed").asBoolean()
+                && result.path("proxy_cleanup").path("back_socket_closed").asBoolean()
+                && result.path("proxy_cleanup").path("pending_packets").asInt(-1) == 0;
+        result.put("all_resources_released", released).put("passed", failures.isEmpty() && released && ticks.size() == 24);
+        return new ReplayScenario.Observed(result, artifact);
+    }
+
+    private static void capture(ArenaServer server, ObjectNode scenario, Map<String, TcpClient> clients, Map<String, Replica> replicas,
+                                UdpFaultProxy proxy, ObjectNode result, int tick) throws Exception {
+        ObjectNode authority = ReplayFixture.snapshot(server); String canonicalHash = hash(ReplayFixture.canonicalRecord(server).getBytes(StandardCharsets.UTF_8));
+        for (var entry : clients.entrySet()) {
+            String role = entry.getKey(); TcpClient client = entry.getValue(); ObjectNode original; int copies, bytes;
+            if (role.equals("alpha")) {
+                UdpFaultProxy.Transit transit = proxy.serverPacket(); require(transit.type().equals("SNAPSHOT"), "snapshot UDP transport");
+                original = transit.original(); copies = transit.deliveredOrdinals().size(); bytes = transit.bytes();
+            } else { original = client.until("SNAPSHOT"); copies = 1; bytes = Json.bytes(original).length; }
+            long sequence = original.path("snapshot_seq").asLong();
+            require(bytes <= 1_200 && original.path("tick").asInt(-2) == tick && original.path("owner_epoch").asInt(-1) == 0
+                    && original.path("room_id").equals(scenario.path("room_id")) && original.path("state_hash").asText().equals(canonicalHash), "snapshot metadata/bytes/hash");
+            long expectedBase = role.equals("alpha") && (sequence == 6 || sequence == 11) ? sequence - 2 : sequence - 1;
+            require(sequence == (tick == -1 ? 1 : (tick + 3) / 2), "snapshot publication ordinal");
+            require(original.path("kind").asText().equals(sequence == 1 ? "FULL" : "DELTA")
+                    && (sequence == 1 ? original.path("base_snapshot_seq").isNull() : original.path("base_snapshot_seq").asLong() == expectedBase), "only actual applied ACK may select base");
+            ObjectNode publication = result.withArray("published_snapshots").addObject().put("client", role).put("tick", tick)
+                    .put("snapshot_seq", sequence).put("bytes", bytes).put("delivered_copies", copies).put("canonical_hash", canonicalHash);
+            publication.set("base_snapshot_seq", original.path("base_snapshot_seq")); publication.set("kind", original.path("kind"));
+            for (int i = 0; i < copies; i++) {
+                ObjectNode wire = role.equals("alpha") ? client.until("SNAPSHOT") : original;
+                Replica replica = replicas.get(role); boolean applied = replica.apply(wire);
+                ObjectNode capture = result.withArray("wire_captures").addObject().put("client", role).put("received_at_tick", tick)
+                        .put("applied", applied).put("highest_applied", replica.sequence); capture.set("snapshot", wire);
+                capture.set("client_visible_state", replica.state.deepCopy());
+                if (role.equals("alpha")) result.withArray("alpha_received_snapshot_ordinals").add(wire.path("snapshot_seq"));
+                if (applied) {
+                    require(replica.state.equals(SnapshotScenario.projection(authority)), "applied snapshot differs from actual projection");
+                    int received = ReplayFixture.udpReceived(server);
+                    ObjectNode ack = client.auth("SNAPSHOT_ACK", scenario.path("room_id").asText()).put("snapshot_seq", replica.sequence);
+                    client.send(ack); if (role.equals("alpha")) require(proxy.clientPacket().type().equals("SNAPSHOT_ACK"), "ACK unaffected by fault counter");
+                    capture.put("ack_sent", true).put("ack_snapshot_seq", replica.sequence);
+                    ReplayFixture.udpBarrier(server, received + 1);
+                    long acknowledged = acknowledged(server, client); capture.put("server_acknowledged_seq", acknowledged);
+                    require(acknowledged == replica.sequence, "actual ACK owner observation");
+                } else capture.put("ack_sent", false);
+            }
+        }
+    }
+
+    private static ObjectNode binding(ArenaServer server, TcpClient client, InetSocketAddress expected) throws Exception {
+        return ReplayFixture.owned(server, () -> {
+            Object session = session(server, client); boolean endpoint = expected.equals(ReplayFixture.field(session, "endpoint"));
+            boolean consumed = ReplayFixture.field(session, "bindToken") == null;
+            require(endpoint && consumed, "observed endpoint/session one-time binding");
+            return Json.MAPPER.createObjectNode().put("endpoint_matches", endpoint).put("token_consumed", consumed);
+        });
+    }
+    private static long acknowledged(ArenaServer server, TcpClient client) throws Exception {
+        return ReplayFixture.owned(server, () -> (long) ReplayFixture.field(ReplayFixture.field(session(server, client), "snapshots"), "acknowledged"));
+    }
+    private static Object session(ArenaServer server, TcpClient client) throws Exception {
+        for (Object session : ((Map<?, ?>) ReplayFixture.field(server, "sessions")).values())
+            if (ReplayFixture.field(session, "id").equals(client.sessionId)) return session;
+        throw new IOException("session owner binding missing");
+    }
+    private static void checkState(ObjectNode state, int tick) throws IOException {
+        int last = tick == 4 ? 4 : tick == 9 ? 9 : tick + 1;
+        for (int slot = 0; slot < 4; slot++) {
+            JsonNode player = ReplayScenario.player(state, "player-0" + slot);
+            require(player.path("x").asInt() == (slot == 0 ? 10_000 + 400 * (tick + 1) : Room.SPAWNS[slot][0])
+                    && player.path("y").asInt() == Room.SPAWNS[slot][1] && player.path("score").asInt() == 0
+                    && player.path("direction").asText().equals(slot == 0 ? "EAST" : "STOP")
+                    && player.path("connectivity").asText().equals("CONNECTED") && player.path("last_accepted_seq").asInt() == (slot == 0 ? last : 0)
+                    && player.path("pending_inputs").asInt() == 0, "fixed authority state " + slot + "/" + tick);
+        }
+        require(state.path("tick").asInt() == tick && state.path("status").asText().equals("RUNNING"), "manual tick/lifecycle");
+    }
+    private static List<Integer> integers(ArrayNode values) { List<Integer> result = new ArrayList<>(); values.forEach(value -> result.add(value.asInt())); return result; }
+    static String hash(byte[] bytes) throws Exception { return HexFormat.of().formatHex(MessageDigest.getInstance("SHA-256").digest(bytes)); }
+    private static void require(boolean condition, String message) throws IOException { if (!condition) throw new IOException(message); }
+
+    private static final class Replica {
+        final TreeMap<Long, ArrayNode> retained = new TreeMap<>();
+        ArrayNode state = Json.MAPPER.createArrayNode(); long sequence; int received, applied;
+        boolean apply(ObjectNode wire) throws IOException {
+            received++; long next = wire.path("snapshot_seq").asLong(); if (next <= sequence) return false;
+            TreeMap<String, JsonNode> players = new TreeMap<>();
+            if (wire.path("kind").asText().equals("DELTA")) {
+                ArrayNode base = retained.get(wire.path("base_snapshot_seq").asLong()); require(base != null, "client missing acknowledged delta base");
+                for (JsonNode player : base) players.put(player.path("player_id").asText(), player);
+            } else require(wire.path("kind").asText().equals("FULL") && wire.path("status").asText().equals("RUNNING"), "full snapshot status");
+            String previous = "";
+            for (JsonNode player : wire.withArray("players")) {
+                String id = player.path("player_id").asText(); require(id.compareTo(previous) > 0 && player.size() == 7, "sorted seven-field player row");
+                for (String field : SnapshotScenario.VISIBLE_FIELDS) require(player.has(field), "missing visible field");
+                require(!player.equals(players.get(id)), "unchanged delta player"); players.put(id, player); previous = id;
+            }
+            previous = "";
+            for (JsonNode id : wire.withArray("removed_player_ids")) {
+                require(id.asText().compareTo(previous) > 0 && players.remove(id.asText()) != null, "sorted delta removal"); previous = id.asText();
+            }
+            state = Json.MAPPER.createArrayNode(); players.values().forEach(state::add); sequence = next; applied++;
+            retained.put(sequence, state.deepCopy()); if (retained.size() > 32) retained.pollFirstEntry(); return true;
+        }
+    }
+}
diff --git a/src/test/resources/G09.json b/src/test/resources/G09.json
new file mode 100644
index 0000000..7edd04d
--- /dev/null
+++ b/src/test/resources/G09.json
@@ -0,0 +1,363 @@
+{
+  "scenario_id": "G09-udp-data-plane",
+  "contract_version": 1,
+  "thread": "G09",
+  "seed": 7050,
+  "clock": {
+    "kind": "manual",
+    "tick_duration_ms": 50
+  },
+  "ticks": 24,
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
+  "initialization": "ordinary four TCP joins while every player UDP-unbound, then four valid binds; fixed IDs available only in test build, no roster bootstrap needed",
+  "events": [
+    {
+      "before_tick": 0,
+      "client": "alpha",
+      "seq": 1,
+      "target_tick": 0,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 1,
+      "client": "alpha",
+      "seq": 2,
+      "target_tick": 1,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 2,
+      "client": "alpha",
+      "seq": 3,
+      "target_tick": 2,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 3,
+      "client": "alpha",
+      "seq": 4,
+      "target_tick": 3,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 4,
+      "client": "alpha",
+      "seq": 5,
+      "target_tick": 4,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 5,
+      "client": "alpha",
+      "seq": 6,
+      "target_tick": 5,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 6,
+      "client": "alpha",
+      "seq": 7,
+      "target_tick": 6,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 7,
+      "client": "alpha",
+      "seq": 8,
+      "target_tick": 7,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 8,
+      "client": "alpha",
+      "seq": 9,
+      "target_tick": 8,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 9,
+      "client": "alpha",
+      "seq": 10,
+      "target_tick": 9,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 10,
+      "client": "alpha",
+      "seq": 11,
+      "target_tick": 10,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 11,
+      "client": "alpha",
+      "seq": 12,
+      "target_tick": 11,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 12,
+      "client": "alpha",
+      "seq": 13,
+      "target_tick": 12,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 13,
+      "client": "alpha",
+      "seq": 14,
+      "target_tick": 13,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 14,
+      "client": "alpha",
+      "seq": 15,
+      "target_tick": 14,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 15,
+      "client": "alpha",
+      "seq": 16,
+      "target_tick": 15,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 16,
+      "client": "alpha",
+      "seq": 17,
+      "target_tick": 16,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 17,
+      "client": "alpha",
+      "seq": 18,
+      "target_tick": 17,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 18,
+      "client": "alpha",
+      "seq": 19,
+      "target_tick": 18,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 19,
+      "client": "alpha",
+      "seq": 20,
+      "target_tick": 19,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 20,
+      "client": "alpha",
+      "seq": 21,
+      "target_tick": 20,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 21,
+      "client": "alpha",
+      "seq": 22,
+      "target_tick": 21,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 22,
+      "client": "alpha",
+      "seq": 23,
+      "target_tick": 22,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 23,
+      "client": "alpha",
+      "seq": 24,
+      "target_tick": 23,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    }
+  ],
+  "faults": {
+    "counting": "one-based original message ordinal independently for alpha INPUT client-to-server and alpha SNAPSHOT server-to-client; duplicate copies do not increment; include startFULL as snapshot1",
+    "drop_indices": [
+      5
+    ],
+    "duplicate_indices": [
+      8
+    ],
+    "swap_adjacent_indices": [
+      [
+        10,
+        11
+      ]
+    ],
+    "swap_delivery": "hold original10 until11 exists, then deliver11 followedby10; no actual sleep",
+    "unaffected": [
+      "otherclient streams",
+      "INPUT_ACK",
+      "SNAPSHOT_ACK",
+      "UDP_BIND",
+      "UDP_BOUND",
+      "TCPcontrol"
+    ]
+  },
+  "udp_bind_token_ttl_ms": 5000,
+  "token_expiry": "now >= issued_at+5000",
+  "fixed_matrix": [
+    {
+      "name": "valid-token-before-deadline",
+      "manual_ms_after_issue": 4999,
+      "expect": "UDP_BOUND"
+    },
+    {
+      "name": "expired-token-at-deadline",
+      "manual_ms_after_issue": 5000,
+      "expect": "UDP_BIND_INVALID"
+    },
+    {
+      "name": "unknown-token",
+      "expect": "UDP_BIND_INVALID"
+    },
+    {
+      "name": "reused-consumed-token",
+      "expect": "UDP_BIND_INVALID; preserve established binding"
+    },
+    {
+      "name": "other-session-token",
+      "expect": "UDP_BIND_INVALID; do not consume valid other-session token"
+    },
+    {
+      "name": "INPUT-before-bind",
+      "expect": "UDP_BIND_INVALID; no sequence/state mutation"
+    },
+    {
+      "name": "INPUT-from-different-observed-endpoint",
+      "expect": "UDP_BIND_INVALID; no sequence/state mutation"
+    },
+    {
+      "name": "wrong-owner-epoch-bind",
+      "owner_epoch": 1,
+      "expect": "UDP_BIND_INVALID; standalone expected epoch0 only"
+    },
+    {
+      "name": "realtime-INPUT-over-TCP",
+      "expect": "WRONG_TRANSPORT; no sequence/state mutation"
+    },
+    {
+      "name": "valid-INPUT-exactly1200bytes",
+      "bytes": 1200,
+      "expect": "ACCEPTED"
+    },
+    {
+      "name": "oversize-datagram1201bytes",
+      "bytes": 1201,
+      "expect": "silentdrop+metric; no ACK or sequence/state mutation"
+    }
+  ],
+  "datagram_max_bytes": 1200,
+  "outbound_bound_probe": {
+    "players": 8,
+    "fields": [
+      "player_id",
+      "slot",
+      "x",
+      "y",
+      "direction",
+      "score",
+      "connectivity"
+    ],
+    "generated_id_lengths": "must jointly fit full envelope and eight player records within1200",
+    "oversized_test_fixture": "reject explicitly; never truncate/drop players/fragment/compress"
+  },
+  "replay": "one24-tick offline replay of actual accepted journal, not all-input reference",
+  "socket_ceiling_ms": 5000
+}
