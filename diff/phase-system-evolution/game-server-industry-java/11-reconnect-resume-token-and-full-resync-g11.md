# Reconnect, Resume Token과 Full Resync (G11)

## `feat(java): preserve sessions across bounded reconnect grace`

diff --git a/TRACK.md b/TRACK.md
index 8548adb..0d125b8 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -1,6 +1,6 @@
-# Java arena — through G10
+# Java arena — through G11
 
-Current thread: G10 (G01–G09 regressions retained). Phase: phase-1. Profile: realtime-core. Spec revision: `c1d62196ab76b55652f5d75a67514f8c6d8210ce`.
+Current thread: G11 (G01–G10 regressions retained). Phase: phase-1. Profile: realtime-core. Spec revision: `c1d62196ab76b55652f5d75a67514f8c6d8210ce`.
 G01–G05 retain their original spec trailers and verification. The user-authorized spec/profile transition changes procedure only; the game contract is unchanged. Scope remains G01–G14, with no G15+, external infrastructure or push.
 
 ## Frozen versions
@@ -30,6 +30,8 @@ The wrapper uses the locally installed Temurin path when JAVA_HOME is unset. On
 ./track scenario-run /absolute/path/to/G07.json /absolute/path/to/V.json --variant rejected-removed
 ./track scenario-run /absolute/path/to/G08.json /absolute/path/to/snapshot-evidence.json
 ./track scenario-run /absolute/path/to/G09.json /absolute/path/to/udp-evidence.json
+./track scenario-run /absolute/path/to/G10.json /absolute/path/to/ack-evidence.json
+./track scenario-run /absolute/path/to/G11.json /absolute/path/to/reconnect-evidence.json
 ./track replay-verify /absolute/path/to/replay /absolute/path/to/evidence
 ./track server config/server.json
 ```
@@ -142,6 +144,12 @@ Each retained snapshot now includes its canonical hash. An authenticated ACK can
 
 The fixed14-tick runner performs normal four-session joins/binds and drops only alpha delta2. Delta3 reconstructs from acknowledged base1; unknown999 causes FULL4, stale1 leaves DELTA5/base4, hash mismatch causes FULL6, and the deliberately missing client base6 causes FULL8 after an unapplied delta7. Actual ACKs never acknowledge missing snapshots. The existing pure33 retention fixture adds the expired-ACK assertion without another publication or live campaign. Baseline and post canonical records plus accepted artifacts are identical; the baseline-only test is archived, not repeated by the full suites. See `evidence/G10-command-ledger.jsonl` and `evidence/G10-verification.md` for exact commands, raw provenance, results and consumption.
 
-G01 initial budget was build/compile <=8, unit suites <=4, integration suites <=2, canonical scenario <=1. Later Threads use their frozen active plans, including G07's explicit five-pass and negative-replay budget. G09 and G10 each execute their fixed network-fault campaign; load remains0. No test sleep, microbenchmark, fuzzing, reconnect, many-room or distributed implementation is included.
+## G11 disconnect grace and credential rotation
+
+TCP loss now makes the existing Player DISCONNECTED/STOP, clears pending work and old UDP/snapshot ownership, and preserves identity, position, score and accepted sequence. The owner retains the session in a registry bounded by the Room's eight slots. Its deadline is the next tick plus200; after tick401 executes, a disconnect at next202 is LEFT. Explicit LEAVE remains LEFT. A valid current resume credential attaches a new HELLO connection to that same session and rotates both credentials. A fresh one-time UDP bind is required before realtime authority returns; its immediate FULL hashes the actual current state without advancing the clock. Consumed credentials and the original UDP endpoint cannot regain authority. Room close releases the recoverable registry and both credential kinds.
+
+The test-only G11 runner uses normal joins and the existing fixed-ID fixture/client. One invocation runs exactly204+402+402 ticks, recording each canonical state/hash plus boundary controls and three immediate FULLs; ordinary snapshot/ACK traffic is counted without repeating entire states. The existing zero-tick eight-player serializer fixture now includes seven DISCONNECTED players. Generated Player IDs use seven characters so that its full envelope fits exactly1,200 bytes; accepted identifier widths and credential entropy are unchanged. See `evidence/G11-command-ledger.jsonl` and `evidence/G11-verification.md` for the preserved baseline, compile diagnostic and passing checks.
+
+G01 initial budget was build/compile <=8, unit suites <=4, integration suites <=2, canonical scenario <=1. Later Threads use their frozen active plans, including G07's explicit five-pass and negative-replay budget. G09 and G10 each execute their fixed network-fault campaign; load remains0. No test sleep, microbenchmark, fuzzing, many-room or distributed implementation is included.
 
 JVM concurrency evidence uses owner-confinement assertions plus real cross-thread Netty handoff, actual thread joins and shutdown assertions. No JVM race detector is installed; no sanitizer result is claimed.
diff --git a/evidence/G11-command-ledger.jsonl b/evidence/G11-command-ledger.jsonl
new file mode 100644
index 0000000..6955f8c
--- /dev/null
+++ b/evidence/G11-command-ledger.jsonl
@@ -0,0 +1,15 @@
+{"kind": "activation", "thread": "G11", "profile": "realtime-core", "spec_revision": "c1d62196ab76b55652f5d75a67514f8c6d8210ce", "start": "9f3c943bbf57b66e7bcd03c2bbf580d7dce549b9", "attempt": "initial", "budget": {"compile": 8, "unit_including_baseline": 4, "integration": 2, "post_canonical": 1, "post_case_ticks": [204, 402, 402], "offline": 0, "fault": 0, "load": 0}, "production_hash_manifest": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-initial/reproduce-unit/production-hashes-before.json"}
+{"kind": "resolved_before_execution", "pass": "baseline", "category": "unit-reproduction-fixed202-setup", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test", "--tests", "arena.G11BaselineTest"], "environment": {"ARENA_G11_SCENARIO": "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G11.json", "ARENA_G11_EVIDENCE": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-initial/reproduce-unit/result.json"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-initial/reproduce-unit", "reserved_g11_ticks": 202, "resolved_at": "2026-08-28T06:34:41.960387+00:00"}
+{"kind": "resolved_before_execution", "pass": "build", "category": "build", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "build"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-initial/verify-build", "reserved_g11_ticks": 0, "resolved_at": "2026-08-28T06:34:41.960737+00:00"}
+{"kind": "resolved_before_execution", "pass": "unit", "category": "full-unit-existing-pure8-serializer", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-initial/verify-unit", "reserved_g11_ticks": 0, "resolved_at": "2026-08-28T06:34:41.960756+00:00"}
+{"kind": "resolved_before_execution", "pass": "integration", "category": "integration", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "integration-test"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-initial/verify-integration", "reserved_g11_ticks": 0, "resolved_at": "2026-08-28T06:34:41.960771+00:00"}
+{"kind": "resolved_before_execution", "pass": "canonical", "category": "canonical-three-case-lifecycle", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "scenario-run", "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G11.json", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-initial/canonical/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-initial/canonical", "reserved_g11_ticks": 1008, "resolved_at": "2026-08-28T06:34:41.960785+00:00", "artifact_paths": ["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-initial/canonical/result.immediate-and-credential-reuse.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-initial/canonical/result.last-valid-boundary.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-initial/canonical/result.expired-boundary.replay.jsonl"]}
+{"pass": "baseline", "category": "unit-reproduction-fixed202-setup", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test", "--tests", "arena.G11BaselineTest"], "environment": {"ARENA_G11_SCENARIO": "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G11.json", "ARENA_G11_EVIDENCE": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-initial/reproduce-unit/result.json"}, "kind": "executed", "started_at": "2026-08-28T06:34:54.209539+00:00", "duration_seconds": 5.445, "command_process_id": 57885, "exit_code": 1, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-initial/reproduce-unit/stdout.log", "process_terminated": true, "compiler_tasks_executed": ["> Task :compileTestJava"], "xml": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-initial/reproduce-unit/xml", "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-initial/reproduce-unit/result.json", "simulation_process_id": 57977, "executed_ticks": 202}
+{"pass": "build", "category": "build", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "build"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T06:54:56.748093+00:00", "duration_seconds": 4.592, "command_process_id": 81524, "exit_code": 1, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-initial/verify-build/stdout.log", "process_terminated": true, "compiler_tasks_executed": ["> Task :compileJava", "> Task :compileTestJava FAILED"]}
+{"kind": "resolved_before_execution", "pass": "build2", "category": "build-compile-only-harness-correction", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "build"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-initial/verify-build-2", "reserved_g11_ticks": 0, "resolved_at": "2026-08-28T06:56:33.443598+00:00", "reason": "remove unused AutoCloseable interface/Override; explicit finally cleanup remains; production unchanged", "prior_failure": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-initial/verify-build/stdout.log", "failed_harness": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-initial/verify-build/ReconnectScenario.java.failed"}
+{"pass": "build2", "category": "build-compile-only-harness-correction", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "build"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T06:56:41.561229+00:00", "duration_seconds": 4.87, "command_process_id": 82487, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-initial/verify-build-2/stdout.log", "process_terminated": true, "compiler_tasks_executed": ["> Task :compileJava", "> Task :compileTestJava"]}
+{"pass": "unit", "category": "full-unit-existing-pure8-serializer", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T06:56:46.432430+00:00", "duration_seconds": 3.896, "command_process_id": 82525, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-initial/verify-unit/stdout.log", "process_terminated": true, "compiler_tasks_executed": [], "xml": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-initial/verify-unit/xml"}
+{"pass": "integration", "category": "integration", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "integration-test"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T06:56:50.331290+00:00", "duration_seconds": 4.544, "command_process_id": 82565, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-initial/verify-integration/stdout.log", "process_terminated": true, "compiler_tasks_executed": [], "xml": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-initial/verify-integration/xml"}
+{"pass": "canonical", "category": "canonical-three-case-lifecycle", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "scenario-run", "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G11.json", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-initial/canonical/result.json"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T07:00:14.525794+00:00", "duration_seconds": 1.318, "command_process_id": 85560, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-initial/canonical/stdout.log", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-initial/canonical/result.json", "simulation_process_id": 85560, "executed_ticks": 1008}
+{"kind":"final_read_only_audit","at":"2026-08-28T07:03:14.147949+00:00","runtime_reruns":0,"post_suites":{"verify-unit":{"tests":45,"failures":0,"errors":0,"skipped":0},"verify-integration":{"tests":4,"failures":0,"errors":0,"skipped":0}},"observed_test_worker_pids":[82563,82607],"integration_children":[{"process_id":82651,"xml":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-initial/verify-integration/xml/TEST-arena.ServerIntegrationTest.xml"}],"cases":[{"name":"immediate-and-credential-reuse","ticks":204,"input_attempts":10,"traffic":{"alpha":{"snapshots":105,"acks":105,"full":8,"delta":97,"disconnected_visible":0,"left_removed":0,"max_datagram_bytes":494,"last_sequence":105},"bravo":{"snapshots":103,"acks":103,"full":6,"delta":97,"disconnected_visible":0,"left_removed":0,"max_datagram_bytes":494,"last_sequence":103}},"final_alpha":{"player_id":"player-00","slot":0,"x":50800,"y":50000,"direction":"EAST","score":1,"last_accepted_seq":21,"applied_seq":21,"connectivity":"CONNECTED","last_tag_tick":200,"pending_inputs":0},"final_hash":"0cf3f07eff0b19d75584223b2e12ed95f1273e7ef03eb83ba8900a55cfda92d1","immediate_full_ticks":[201,201],"artifact_sha256":"02ca683a30f1f1c0c38eb51c3048d8153c7e9a087f833fd0ce4668b2c533d4f8","artifact_bytes":82645,"journal_counts":{"HEADER":1,"INPUT":8,"TICK":204,"DISCONNECTED":2,"RECONNECTED":2},"resources_released":true,"setup_records_equal_baseline":true},{"name":"last-valid-boundary","ticks":402,"input_attempts":7,"traffic":{"alpha":{"snapshots":104,"acks":104,"full":7,"delta":97,"disconnected_visible":0,"left_removed":0,"max_datagram_bytes":494,"last_sequence":104},"bravo":{"snapshots":202,"acks":202,"full":11,"delta":191,"disconnected_visible":99,"left_removed":0,"max_datagram_bytes":495,"last_sequence":202}},"final_alpha":{"player_id":"player-00","slot":0,"x":50400,"y":50000,"direction":"STOP","score":1,"last_accepted_seq":20,"applied_seq":null,"connectivity":"CONNECTED","last_tag_tick":200,"pending_inputs":0},"final_hash":"5223cfe3b07eddb95694e1ad217414917875a65e31d033ddbd99c1c1fe6bc907","immediate_full_ticks":[400],"artifact_sha256":"f04b1199fdc52d08e679e6d9e4a52646b609c5cc20cc83cb6d1b655396382ee9","artifact_bytes":161766,"journal_counts":{"HEADER":1,"INPUT":7,"TICK":402,"DISCONNECTED":1,"RECONNECTED":1},"resources_released":true,"setup_records_equal_baseline":true},{"name":"expired-boundary","ticks":402,"input_attempts":7,"traffic":{"alpha":{"snapshots":102,"acks":102,"full":6,"delta":96,"disconnected_visible":0,"left_removed":0,"max_datagram_bytes":494,"last_sequence":102},"bravo":{"snapshots":202,"acks":202,"full":11,"delta":191,"disconnected_visible":99,"left_removed":1,"max_datagram_bytes":495,"last_sequence":202}},"final_alpha":{"player_id":"player-00","slot":0,"x":50400,"y":50000,"direction":"STOP","score":1,"last_accepted_seq":20,"applied_seq":null,"connectivity":"LEFT","last_tag_tick":200,"pending_inputs":0},"final_hash":"a3d3d408cdd112d3e8f4dbf3faea8c1c835e01aa0fc5004056ee06b390ec5547","immediate_full_ticks":[],"artifact_sha256":"cc1f252cecbef0aa5a94ae08632af1efbd7386c36595a88c20b9468e757c3992","artifact_bytes":161696,"journal_counts":{"HEADER":1,"INPUT":7,"TICK":402,"DISCONNECTED":1},"resources_released":true,"setup_records_equal_baseline":true}],"result_bytes":1370881,"result_sha256":"2582d5e65d5f778dc7647ac57e1b298d66af44e174345f4b645f7bbe5a003109","budget_consumed":{"compiler_bearing_commands":3,"compiler_tasks":5,"unit_including_baseline":2,"integration":1,"post_canonical":1,"baseline_ticks":202,"post_ticks":1008,"offline":0,"fault":0,"load":0,"repairs":0},"compile_failure_preserved":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-initial/verify-build/stdout.log"}
+{"kind":"precommit_documentation_audit","at":"2026-08-28T07:07:24.930018+00:00","tested_source_files_unchanged":41,"source_hashes":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-initial/tested-source-hashes.json","fixture_sha256":"f4b892f5155655d41b52f197c1174c7c41f7fe75cd3f75e6f0f5b2f8e8c261c7","runtime_reruns":0,"contiguous_footer_parsed":["Thread: G11","Profile: realtime-core","Spec-Revision: c1d62196ab76b55652f5d75a67514f8c6d8210ce"]}
diff --git a/evidence/G11-verification.md b/evidence/G11-verification.md
new file mode 100644
index 0000000..402a57e
--- /dev/null
+++ b/evidence/G11-verification.md
@@ -0,0 +1,43 @@
+# G11 verification — Java initial attempt
+
+START: `9f3c943bbf57b66e7bcd03c2bbf580d7dce549b9`. Profile: `realtime-core`. Spec: `c1d62196ab76b55652f5d75a67514f8c6d8210ce`. Frozen G11 fixture SHA-256: `f4b892f5155655d41b52f197c1174c7c41f7fe75cd3f75e6f0f5b2f8e8c261c7`.
+
+Exact argv, environments, process IDs, compiler tasks and output/artifact paths are in [the command ledger](G11-command-ledger.jsonl). Raw root: `/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-initial/`.
+
+## Preserved reproduction
+
+`./track unit-test --tests arena.G11BaselineTest` exercised unchanged G10 production through seven accepted INPUTs and202 ticks. JOIN issued no resume credential; actual TCP loss while the UDP socket remained open made alpha LEFT/STOP. Resume was reported unavailable, not attempted with fabricated credentials. All16 production hashes matched START before and after. The complete baseline harness, raw202 states/records/hashes, replay, XML and zero-resource cleanup remain under `reproduce-unit/`. Its single intended assertion failed: exit1, one test, one failure, zero errors/skips.
+
+## Change and checks
+
+The owner now preserves the existing session/player for a200-tick disconnect grace, clears pending work and old transport ownership, rotates private resume/bind credentials, and requires a fresh UDP bind. Explicit LEAVE stays LEFT. Disconnect preserves `applied_seq` until the next tick; later normal ticks determine it as before. Deadline402 expires after401. Immediate FULL uses current CONNECTED/STOP state at the existing tick, including when that tick's historical record was EAST or DISCONNECTED. No dependency, gameplay rate/range/cooldown, tick schedule or identifier input-width policy changed.
+
+| Invocation | Actual result |
+|---|---|
+| First post build | Exit1: test harness `AutoCloseable.close()` triggered `-Xlint:try` under `-Werror`; raw diagnostic and failed source preserved. |
+| Second post build | Exit0 after removing only the unused interface/override; explicit cleanup unchanged. |
+| Full unit | 45 passed, zero failures/errors/skips. |
+| Integration | 4 passed, zero failures/errors/skips; real CLI shutdown child recorded. |
+| One G11 canonical | Exit0; three cases,1,008 ticks,24 INPUT attempts; all cleanup assertions passed. |
+
+No post-change runtime failure or rerun occurred. The baseline-only test is archived and absent from final suites; those suites do not repeat the G11 campaign.
+
+| Case | Ticks | Final alpha | Actual alpha/bravo snapshots and ACKs | Immediate FULL ticks |
+|---|---:|---|---|---|
+| immediate-and-credential-reuse | 204 | CONNECTED/EAST, x50800, seq21 | 105/103, each ACKed | 201,201 |
+| last-valid-boundary | 402 | CONNECTED/STOP, x50400, seq20 | 104/202, each ACKed | 400 |
+| expired-boundary | 402 | LEFT/STOP, x50400, seq20 | 102/202, each ACKed | none |
+
+All cases finish alpha at y50000, score1, last TAG200; bravo remains `(50000,50000)`, STOP/CONNECTED, score0. Each case's first202 canonical records/hashes exactly match the baseline. The result retains all1,008 raw states, LF-terminated records and hashes; all three immediate FULLs match their current-state records and differ from the historical same-tick hash. Credential reuse, rotation, duplicate20, old-endpoint rejection, valid21, last-valid401 and expired402 are actual control/UDP observations. Routine totals are818 snapshots and818 ACKs. The three accepted journals remain beside `canonical/result.json`; no G11 offline run is claimed.
+
+The existing pure serializer probe executes zero ticks: eight connected players serialize to1,186 bytes; seven DISCONNECTED/STOP plus one CONNECTED/NORTH serialize to exactly1,200. Generated Room IDs remain22 characters; Player IDs are7. Credentials retain their original entropy, and incoming IDs still allow64 bytes. The oversized injected fixture serializes to1,684 bytes and is explicitly rejected. No player omission or fragmentation is used.
+
+Every case closes all client sockets and clears private client credentials. Server cleanup reports zero channels, sessions/recoverable sessions, credentials, retained snapshots/replay bytes, pending writes/mailbox work and live parser/UDP buffers or allocated bytes; timer stopped and owner/event loops terminated. The result SHA-256 is `2582d5e65d5f778dc7647ac57e1b298d66af44e174345f4b645f7bbe5a003109`; the ledger records artifact hashes and child processes.
+
+Consumption: **5/8 compiler tasks** across three compiler-bearing commands, **2/4 unit** including reproduction, **1/2 integration**, **1/1 post canonical**, repairs0/2; offline/fault/load0. Dedicated simulation: baseline202 and post1,008 ticks. Worker verification passed; immutable-END root verification is separate.
+
+Root-ready canonical command, using a fresh destination:
+
+```sh
+./track scenario-run /Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G11.json /private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g11-root/canonical/result.json
+```
diff --git a/src/main/java/arena/ArenaServer.java b/src/main/java/arena/ArenaServer.java
index 898918d..2aed916 100644
--- a/src/main/java/arena/ArenaServer.java
+++ b/src/main/java/arena/ArenaServer.java
@@ -25,6 +25,7 @@ import java.security.SecureRandom;
 import java.util.ArrayList;
 import java.util.Base64;
 import java.util.HashMap;
+import java.util.HashSet;
 import java.util.List;
 import java.util.Map;
 import java.util.Set;
@@ -74,6 +75,8 @@ public final class ArenaServer implements AutoCloseable {
     private final Thread ticker;
     // The following fields, including the session registry, are exclusively room-owner state.
     private final Map<Peer, Session> sessions = new HashMap<>();
+    // Only successful joins enter this registry: bounded by the Room's eight stable slots.
+    private final Map<String, Session> recoverableSessions = new HashMap<>();
     private Room room;
     private FixedTickClock fixedClock;
     private long manualNanos;
@@ -81,6 +84,10 @@ public final class ArenaServer implements AutoCloseable {
     private volatile int closedInputHighWater;
     private volatile int closedReplayBytes;
     private volatile int closedSnapshotCount;
+    private volatile int closedActiveSessions;
+    private volatile int closedRecoverableSessions;
+    private volatile int closedResumeCredentials;
+    private volatile int closedBindCredentials;
     private int snapshotRetentionHighWater;
     private volatile ObjectNode closedClockMetrics = Json.MAPPER.createObjectNode().put("active", false)
             .put("accumulator_ns", 0L).put("max_ticks_per_iteration", 0);
@@ -88,12 +95,15 @@ public final class ArenaServer implements AutoCloseable {
     private static final class Session {
         final String id = "s-" + UUID.randomUUID();
         final SnapshotStream snapshots = new SnapshotStream();
-        final long bindIssuedAt;
+        long bindIssuedAt;
         String bindToken = opaque(24);
+        String resumeToken;
         InetSocketAddress endpoint;
         String playerId;
+        boolean reconnectFullPending;
         Session(long now) { bindIssuedAt = now; }
-        void release() { snapshots.close(); bindToken = null; endpoint = null; }
+        void disconnected() { snapshots.invalidateBase(); bindToken = null; endpoint = null; reconnectFullPending = false; }
+        void release() { snapshots.close(); bindToken = null; resumeToken = null; endpoint = null; reconnectFullPending = false; }
     }
 
     private final class Peer {
@@ -301,6 +311,7 @@ public final class ArenaServer implements AutoCloseable {
                 peer.send(welcome);
                 return;
             }
+            if (type.equals("RECONNECT")) { reconnect(peer, message); return; }
             Session session = sessions.get(peer);
             if (session == null || !session.id.equals(Json.text(message, "session_id"))) {
                 peer.error(type.equals("UDP_BIND") ? "UDP_BIND_INVALID" : "SESSION_INVALID"); return;
@@ -321,14 +332,15 @@ public final class ArenaServer implements AutoCloseable {
                         peer.error("ROOM_NOT_JOINABLE"); break;
                     }
                     String playerId;
-                    do { playerId = opaque(6); } while (room.player(playerId) != null);
+                    do { playerId = opaque(5); } while (room.player(playerId) != null);
                     Room.Player player;
                     try { player = room.join(playerId); }
                     catch (IllegalStateException notJoinable) { peer.error("ROOM_NOT_JOINABLE"); break; }
                     session.playerId = player.id;
+                    session.resumeToken = opaque(24); recoverableSessions.put(session.id, session);
                     if (session.endpoint != null) room.ready(player.id);
                     peer.send(Json.message("ROOM_JOINED").put("room_id", room.id).put("player_id", player.id)
-                            .put("slot", player.slot).put("status", room.status().name()));
+                            .put("slot", player.slot).put("status", room.status().name()).put("resume_token", session.resumeToken));
                     if (room.status() == Room.Status.RUNNING) {
                         startReplication();
                     }
@@ -341,7 +353,11 @@ public final class ArenaServer implements AutoCloseable {
                     session.endpoint = endpoint; session.bindToken = null;
                     if (session.playerId != null) room.ready(session.playerId);
                     peer.realtime(Json.message("UDP_BOUND").put("owner_epoch", 0), endpoint);
-                    if (room != null) startReplication();
+                    if (session.reconnectFullPending) {
+                        session.reconnectFullPending = false;
+                        if (fixedClock == null && room.status() == Room.Status.RUNNING) startReplication();
+                        else publishSnapshot(peer, session, true);
+                    } else if (room != null) startReplication();
                 }
                 case "PING" -> {
                     if (!roomMatches(peer, message)) break;
@@ -392,11 +408,32 @@ public final class ArenaServer implements AutoCloseable {
         return true;
     }
 
+    private void reconnect(Peer peer, ObjectNode message) {
+        Session provisional = sessions.get(peer);
+        Session existing = recoverableSessions.get(Json.text(message, "session_id"));
+        if (provisional == null || provisional.playerId != null || existing == null || room == null
+                || !room.id.equals(Json.text(message, "room_id")) || existing.resumeToken == null
+                || !existing.resumeToken.equals(Json.text(message, "resume_token"))) {
+            peer.error("RECONNECT_INVALID"); return;
+        }
+        String code = room.reconnect(existing.playerId);
+        if (code != null) { peer.error(code); return; }
+        provisional.release(); sessions.put(peer, existing);
+        existing.resumeToken = opaque(24); existing.bindToken = opaque(24);
+        existing.bindIssuedAt = monotonicNanos.getAsLong(); existing.reconnectFullPending = true;
+        peer.send(Json.message("RECONNECTED").put("session_id", existing.id).put("room_id", room.id)
+                .put("player_id", existing.playerId).put("last_accepted_seq", room.player(existing.playerId).lastAcceptedSeq)
+                .put("resume_token", existing.resumeToken).put("udp_bind_token", existing.bindToken).put("udp_port", udpPort()));
+    }
+
     private void disconnect(Peer peer) {
         Session session = sessions.remove(peer);
         if (session != null) {
-            session.release();
-            if (session.playerId != null && room != null) room.left(session.playerId);
+            if (session.playerId != null && room != null) {
+                room.disconnect(session.playerId);
+                if (room.player(session.playerId).connectivity == Room.Connectivity.DISCONNECTED) session.disconnected();
+                else { session.release(); recoverableSessions.remove(session.id); }
+            } else session.release();
             if (room != null) startReplication();
         }
     }
@@ -410,13 +447,17 @@ public final class ArenaServer implements AutoCloseable {
     private void publishSnapshots(boolean forceFull) {
         for (var entry : sessions.entrySet()) {
             Session session = entry.getValue();
-            if (session.playerId == null || !room.player(session.playerId).connected) continue;
+            if (session.playerId == null || !room.player(session.playerId).connected()) continue;
             if (session.endpoint == null) continue;
-            entry.getKey().realtime(session.snapshots.next(room, forceFull), session.endpoint);
-            snapshotRetentionHighWater = Math.max(snapshotRetentionHighWater, session.snapshots.highWater());
+            publishSnapshot(entry.getKey(), session, forceFull);
         }
     }
 
+    private void publishSnapshot(Peer peer, Session session, boolean forceFull) {
+        peer.realtime(session.snapshots.next(room, forceFull), session.endpoint);
+        snapshotRetentionHighWater = Math.max(snapshotRetentionHighWater, session.snapshots.highWater());
+    }
+
     private void broadcast(ObjectNode message) {
         for (var entry : sessions.entrySet()) if (entry.getValue().playerId != null) entry.getKey().send(message);
     }
@@ -476,6 +517,8 @@ public final class ArenaServer implements AutoCloseable {
             result.set("parser", parserMetrics.view());
             result.set("udp", udpMetrics.view());
             result.put("udp_bindings", sessions.values().stream().filter(s -> s.endpoint != null).count());
+            result.put("active_sessions", sessions.size()).put("recoverable_sessions", recoverableSessions.size())
+                    .put("resume_credentials", recoverableSessions.values().stream().filter(s -> s.resumeToken != null).count());
             return result;
         });
     }
@@ -498,6 +541,8 @@ public final class ArenaServer implements AutoCloseable {
                 .put("event_loops_terminated", acceptLoop.isTerminated() && ioLoop.isTerminated())
                 .put("pending_input_high_water", closedInputHighWater)
                 .put("replay_bytes", closedReplayBytes)
+                .put("active_sessions", closedActiveSessions).put("recoverable_sessions", closedRecoverableSessions)
+                .put("resume_credentials", closedResumeCredentials).put("bind_credentials", closedBindCredentials)
                 .put("retained_snapshots", closedSnapshotCount).put("snapshot_retention_high_water", snapshotRetentionHighWater)
                 .put("mailbox_high_water", mailboxHighWater.get()).put("outbound_high_water", outboundHighWater.get());
         result.set("parser", parserMetrics.view());
@@ -523,9 +568,12 @@ public final class ArenaServer implements AutoCloseable {
         // Drain channel callbacks before final owner cleanup. No network thread waits for this owner.
         ioLoop.submit(() -> { }).syncUninterruptibly();
         call(() -> {
-            for (Session session : sessions.values()) session.release();
-            closedSnapshotCount = sessions.values().stream().mapToInt(s -> s.snapshots.retainedCount()).sum();
-            sessions.clear();
+            Set<Session> allSessions = new HashSet<>(sessions.values()); allSessions.addAll(recoverableSessions.values());
+            for (Session session : allSessions) session.release();
+            closedSnapshotCount = allSessions.stream().mapToInt(s -> s.snapshots.retainedCount()).sum();
+            sessions.clear(); recoverableSessions.clear(); closedActiveSessions = sessions.size(); closedRecoverableSessions = recoverableSessions.size();
+            closedResumeCredentials = (int) allSessions.stream().filter(s -> s.resumeToken != null).count();
+            closedBindCredentials = (int) allSessions.stream().filter(s -> s.bindToken != null).count();
             if (room != null) {
                 room.close();
                 closedReplayBytes = room.replayStoredBytes();
diff --git a/src/main/java/arena/CompleteFrame.java b/src/main/java/arena/CompleteFrame.java
index 0708b38..7df744c 100644
--- a/src/main/java/arena/CompleteFrame.java
+++ b/src/main/java/arena/CompleteFrame.java
@@ -199,6 +199,11 @@ final class CompleteFrame extends SimpleChannelInboundHandler<ByteBuf> {
                     Json.identifier(message, "session_id");
                     Json.identifier(message, "room_id");
                 }
+                case "RECONNECT" -> {
+                    Json.identifier(message, "session_id");
+                    Json.identifier(message, "room_id");
+                    Json.text(message, "resume_token");
+                }
                 case "SNAPSHOT_ACK" -> {
                     Json.identifier(message, "session_id");
                     Json.identifier(message, "room_id");
@@ -227,7 +232,7 @@ final class CompleteFrame extends SimpleChannelInboundHandler<ByteBuf> {
                     Json.identifier(message, "room_id");
                     Json.identifier(message, "player_id");
                 }
-                case "UDP_BOUND", "INPUT_ACK", "SNAPSHOT", "PONG", "WELCOME", "ROOM_CREATED", "ROOM_JOINED", "ROOM_FINISHED", "ERROR" -> { }
+                case "UDP_BOUND", "INPUT_ACK", "SNAPSHOT", "PONG", "WELCOME", "ROOM_CREATED", "ROOM_JOINED", "ROOM_FINISHED", "RECONNECTED", "ERROR" -> { }
                 default -> { return "MESSAGE_TYPE_UNKNOWN"; }
             }
         } catch (IllegalArgumentException invalid) { return "MESSAGE_INVALID"; }
diff --git a/src/main/java/arena/IdentityScenario.java b/src/main/java/arena/IdentityScenario.java
index 7d2b8cd..e342833 100644
--- a/src/main/java/arena/IdentityScenario.java
+++ b/src/main/java/arena/IdentityScenario.java
@@ -164,8 +164,9 @@ final class IdentityScenario {
             ObjectNode after = world.snapshot();
             out.set("after", world.normalize(after));
             ObjectNode expected = before.deepCopy();
-            ((ObjectNode) world.player(expected, world.alpha)).put("connectivity", "LEFT").put("direction", "STOP");
-            check(after.equals(expected), state + "/" + action + " must only mark alpha LEFT");
+            String connectivity = action.equals("CONNECTION_CLOSE") ? "DISCONNECTED" : "LEFT";
+            ((ObjectNode) world.player(expected, world.alpha)).put("connectivity", connectivity).put("direction", "STOP");
+            check(after.equals(expected), state + "/" + action + " must only stop alpha and set " + connectivity);
             out.put("connection_closed", !channel.isOpen());
             int remainingSessions = onOwner(world.server, () -> sessions(world.server).size());
             out.put("sessions_after_action", remainingSessions);
@@ -301,6 +302,9 @@ final class IdentityScenario {
         }
         cleanup.put("pending_inputs_remaining", pending);
         check(sessions(server).isEmpty() && pending == 0, "terminal session/input cleanup");
+        check(((Map<?, ?>) field(server, "recoverableSessions")).isEmpty()
+                && cleanup.path("resume_credentials").asInt() == 0 && cleanup.path("bind_credentials").asInt() == 0,
+                "terminal reconnect resource cleanup");
         if (retained != null) {
             int ticks = (int) field(retained, "executedTicks");
             int count = ((Map<?, ?>) field(retained, "players")).size();
diff --git a/src/main/java/arena/ReplayLog.java b/src/main/java/arena/ReplayLog.java
index 0639dd2..4082b8f 100644
--- a/src/main/java/arena/ReplayLog.java
+++ b/src/main/java/arena/ReplayLog.java
@@ -34,7 +34,10 @@ final class ReplayLog {
         append(event);
     }
     void left(int beforeTick, String playerId) {
-        append(Json.MAPPER.createObjectNode().put("kind", "LEFT").put("before_tick", beforeTick).put("player_id", playerId));
+        lifecycle("LEFT", beforeTick, playerId);
+    }
+    void lifecycle(String kind, int beforeTick, String playerId) {
+        append(Json.MAPPER.createObjectNode().put("kind", kind).put("before_tick", beforeTick).put("player_id", playerId));
     }
     void tick(int tick, String record, String hash) {
         append(Json.MAPPER.createObjectNode().put("kind", "TICK").put("tick", tick)
diff --git a/src/main/java/arena/Room.java b/src/main/java/arena/Room.java
index accf9cf..62e12c6 100644
--- a/src/main/java/arena/Room.java
+++ b/src/main/java/arena/Room.java
@@ -15,12 +15,14 @@ final class Room {
     static final int INPUT_LIMIT = 64;
     static final int MAX_FUTURE_TICKS = 4;
     static final int VALIDATION_LIMIT = 4;
+    static final int RECONNECT_GRACE_TICKS = 200;
     static final int[][] SPAWNS = {
         {10_000, 10_000}, {90_000, 90_000}, {10_000, 90_000}, {90_000, 10_000},
         {50_000, 10_000}, {50_000, 90_000}, {10_000, 50_000}, {90_000, 50_000}
     };
     enum Direction { STOP, NORTH, EAST, SOUTH, WEST }
     enum Status { LOBBY, RUNNING, FINISHED, CLOSED }
+    enum Connectivity { CONNECTED, DISCONNECTED, LEFT }
     // Only active logical payload fields define duplicate identity; unknown JSON fields are ignored.
     record Intent(BigInteger seq, BigInteger targetTick, Direction direction, String target) { }
     record Rejection(String playerId, String code) { }
@@ -34,7 +36,8 @@ final class Room {
         int score;
         int lastTagTick = -20;
         Direction direction = Direction.STOP;
-        boolean connected = true;
+        Connectivity connectivity = Connectivity.CONNECTED;
+        int disconnectDeadline = -1;
         boolean realtimeReady;
         BigInteger lastAcceptedSeq = BigInteger.ZERO;
         Intent lastAcceptedIntent;
@@ -45,12 +48,13 @@ final class Room {
             this.id = id; this.slot = slot;
             x = SPAWNS[slot][0]; y = SPAWNS[slot][1];
         }
+        boolean connected() { return connectivity == Connectivity.CONNECTED; }
 
         ObjectNode view() {
             return Json.MAPPER.createObjectNode().put("player_id", id).put("slot", slot)
                     .put("x", x).put("y", y).put("direction", direction.name()).put("score", score)
                     .put("last_accepted_seq", lastAcceptedSeq).put("applied_seq", appliedSeq)
-                    .put("connectivity", connected ? "CONNECTED" : "LEFT");
+                    .put("connectivity", connectivity.name());
         }
     }
 
@@ -88,14 +92,14 @@ final class Room {
     void ready(String id) {
         assertOwner();
         Player player = players.get(id);
-        if (player == null || !player.connected) throw new IllegalArgumentException("PLAYER_MISMATCH");
+        if (player == null || !player.connected()) throw new IllegalArgumentException("PLAYER_MISMATCH");
         player.realtimeReady = true;
         startIfReady();
     }
 
     private void startIfReady() {
-        if (status == Status.LOBBY && players.values().stream().filter(p -> p.connected).count() >= 2
-                && players.values().stream().filter(p -> p.connected).allMatch(p -> p.realtimeReady)) {
+        if (status == Status.LOBBY && players.values().stream().filter(p -> p.connectivity != Connectivity.LEFT).count() >= 2
+                && players.values().stream().filter(p -> p.connectivity != Connectivity.LEFT).allMatch(p -> p.connected() && p.realtimeReady)) {
             transition(Status.RUNNING);
             replay = new ReplayLog(id, players.values());
         }
@@ -106,7 +110,7 @@ final class Room {
         assertOwner();
         Player player = players.get(id);
         // Preserve earlier lifecycle/schema error precedence; inactive actors consume no allowance.
-        if (status != Status.RUNNING || player == null || !player.connected) return true;
+        if (status != Status.RUNNING || player == null || !player.connected()) return true;
         if (player.validationAttempts == VALIDATION_LIMIT) return false;
         player.validationAttempts++;
         validationHighWater = Math.max(validationHighWater, player.validationAttempts);
@@ -117,7 +121,7 @@ final class Room {
         assertOwner();
         if (status != Status.RUNNING) return "ROOM_NOT_RUNNING";
         Player player = players.get(id);
-        if (player == null || !player.connected) return "PLAYER_MISMATCH";
+        if (player == null || !player.connected()) return "PLAYER_MISMATCH";
         if (intent.seq().signum() <= 0 || intent.seq().bitLength() > 64) return "MESSAGE_INVALID";
         int comparison = intent.seq().compareTo(player.lastAcceptedSeq);
         if (comparison < 0) return "INPUT_STALE";
@@ -149,7 +153,7 @@ final class Room {
                 if (selected == null || candidate.seq().compareTo(selected.seq()) > 0) selected = candidate;
                 pending.remove();
             }
-            if (!player.connected) continue;
+            if (!player.connected()) continue;
             if (selected != null) {
                 player.appliedSeq = selected.seq();
                 player.direction = selected.direction();
@@ -172,7 +176,7 @@ final class Room {
             Player target = players.get(tag.getValue());
             long dx = target == null ? 0 : (long) actor.x - target.x;
             long dy = target == null ? 0 : (long) actor.y - target.y;
-            if (target == null || actor == target || !actor.connected || !target.connected
+            if (target == null || actor == target || !actor.connected() || !target.connected()
                     || executedTicks - actor.lastTagTick < 20 || tagged.contains(target.id)
                     || dx * dx + dy * dy > 2_500L * 2_500L) {
                 rejected.add(new Rejection(actor.id, "ACTION_REJECTED"));
@@ -182,7 +186,12 @@ final class Room {
                 tagged.add(target.id);
             }
         }
-        if (++executedTicks == DURATION) transition(Status.FINISHED);
+        ++executedTicks;
+        for (Player player : players.values()) if (player.connectivity == Connectivity.DISCONNECTED
+                && executedTicks >= player.disconnectDeadline) {
+            player.connectivity = Connectivity.LEFT; player.pending.clear();
+        }
+        if (executedTicks == DURATION) transition(Status.FINISHED);
         String record = canonicalRecord();
         stateHash = ReplayLog.hash(record);
         replay.tick(executedTicks - 1, record, stateHash);
@@ -192,11 +201,32 @@ final class Room {
     void left(String id) {
         assertOwner();
         Player player = players.get(id);
-        if (player != null && player.connected && replay != null) replay.left(executedTicks, id);
-        if (player != null) { player.connected = false; player.direction = Direction.STOP; player.pending.clear(); }
+        if (player != null && player.connectivity != Connectivity.LEFT && replay != null) replay.left(executedTicks, id);
+        if (player != null) {
+            player.connectivity = Connectivity.LEFT; player.disconnectDeadline = -1; player.realtimeReady = false;
+            player.direction = Direction.STOP; player.pending.clear();
+        }
         startIfReady();
     }
 
+    void disconnect(String id) {
+        assertOwner(); Player player = players.get(id);
+        if (player == null || !player.connected()) return;
+        player.connectivity = Connectivity.DISCONNECTED; player.direction = Direction.STOP; player.realtimeReady = false; player.pending.clear();
+        player.disconnectDeadline = executedTicks + RECONNECT_GRACE_TICKS;
+        if (replay != null) replay.lifecycle("DISCONNECTED", executedTicks, id);
+    }
+
+    String reconnect(String id) {
+        assertOwner(); Player player = players.get(id);
+        if (player == null) return "RECONNECT_INVALID";
+        if (player.disconnectDeadline >= 0 && executedTicks >= player.disconnectDeadline) return "RECONNECT_EXPIRED";
+        if (player.connectivity != Connectivity.DISCONNECTED) return "RECONNECT_INVALID";
+        player.connectivity = Connectivity.CONNECTED; player.disconnectDeadline = -1;
+        if (replay != null) replay.lifecycle("RECONNECTED", executedTicks, id);
+        return null;
+    }
+
     void close() {
         assertOwner();
         for (Player player : players.values()) player.pending.clear();
@@ -219,7 +249,7 @@ final class Room {
                 .append("|status=").append(status).append("|owner_epoch=0\n");
         for (Player player : players.values()) record.append("player=").append(player.id).append("|slot=").append(player.slot)
                 .append("|x=").append(player.x).append("|y=").append(player.y).append("|dir=").append(player.direction)
-                .append("|score=").append(player.score).append("|conn=").append(player.connected ? "CONNECTED" : "LEFT")
+                .append("|score=").append(player.score).append("|conn=").append(player.connectivity)
                 .append("|last_seq=").append(player.lastAcceptedSeq).append("|last_tag_tick=").append(player.lastTagTick).append('\n');
         return record.toString();
     }
diff --git a/src/main/java/arena/SnapshotStream.java b/src/main/java/arena/SnapshotStream.java
index df2dcc9..f6ec31d 100644
--- a/src/main/java/arena/SnapshotStream.java
+++ b/src/main/java/arena/SnapshotStream.java
@@ -65,6 +65,11 @@ final class SnapshotStream implements AutoCloseable {
     }
     int retainedCount() { assertOwner(); return retained.size(); }
     int highWater() { assertOwner(); return highWater; }
+    void invalidateBase() {
+        assertOwner();
+        if (closed) throw new IllegalStateException("snapshot stream closed");
+        retained.clear(); acknowledged = 0; resyncPending = true;
+    }
     @Override public void close() { assertOwner(); retained.clear(); acknowledged = 0; resyncPending = false; closed = true; }
     private void assertOwner() {
         if (Thread.currentThread() != owner) throw new IllegalStateException("snapshot access outside owner");
diff --git a/src/main/java/arena/TcpClient.java b/src/main/java/arena/TcpClient.java
index 59f0dfd..9f58e12 100644
--- a/src/main/java/arena/TcpClient.java
+++ b/src/main/java/arena/TcpClient.java
@@ -29,6 +29,7 @@ final class TcpClient implements AutoCloseable {
     private final List<String> lifecycle = new ArrayList<>();
     private InetSocketAddress udpDestination;
     private String bindToken;
+    private String resumeToken;
     private boolean eof;
     String sessionId;
     String playerId;
@@ -149,8 +150,10 @@ final class TcpClient implements AutoCloseable {
     }
 
     private void observe(ObjectNode message) throws IOException {
-        if (message.path("type").asText().equals("WELCOME")) {
+        String type = message.path("type").asText();
+        if (type.equals("WELCOME") || type.equals("RECONNECTED")) {
             sessionId = Json.text(message, "session_id");
+            if (type.equals("RECONNECTED")) { playerId = Json.text(message, "player_id"); roomId = Json.text(message, "room_id"); }
             if (udpDestination == null) {
                 var port = message.path("udp_port");
                 if (!port.isIntegralNumber() || !port.canConvertToInt() || port.asInt() < 1 || port.asInt() > 65_535)
@@ -160,6 +163,10 @@ final class TcpClient implements AutoCloseable {
             if (message.has("udp_bind_token")) bindToken = Json.text(message, "udp_bind_token");
             message.put("udp_bind_token_present", message.has("udp_bind_token")); message.remove("udp_bind_token");
         }
+        if (type.equals("ROOM_JOINED") || type.equals("RECONNECTED")) {
+            if (message.has("resume_token")) resumeToken = Json.text(message, "resume_token");
+            message.put("resume_token_present", message.has("resume_token"));
+        }
         // Credentials remain only in memory and are never returned to evidence or exception formatting.
         message.remove("resume_token");
         String status = message.path("status").asText();
@@ -179,6 +186,22 @@ final class TcpClient implements AutoCloseable {
         playerId = Json.text(response, "player_id"); roomId = id; return response;
     }
 
+    ObjectNode reconnectFrom(TcpClient previous) throws IOException {
+        if (previous.resumeToken == null) throw new IOException("no resume credential available");
+        sendTcp(Json.message("RECONNECT").put("session_id", previous.sessionId).put("room_id", previous.roomId)
+                .put("resume_token", previous.resumeToken));
+        return control();
+    }
+
+    boolean resumeRotatedFrom(TcpClient previous) {
+        return resumeToken != null && previous.resumeToken != null && !resumeToken.equals(previous.resumeToken);
+    }
+    boolean bindRotatedFrom(TcpClient previous) {
+        return bindToken != null && previous.bindToken != null && !bindToken.equals(previous.bindToken);
+    }
+    void closeTcp() throws IOException { tcp.close(); eof = true; }
+    boolean tcpClosed() { return !tcp.isOpen(); }
+
     ObjectNode auth(String type, String id) {
         ObjectNode request = Json.message(type);
         if (sessionId != null) request.put("session_id", sessionId);
@@ -210,7 +233,7 @@ final class TcpClient implements AutoCloseable {
         if (lifecycle.isEmpty() || !lifecycle.getLast().equals(state)) lifecycle.add(state);
     }
     @Override public void close() {
-        bindToken = null; received.clear();
+        bindToken = null; resumeToken = null; received.clear();
         try { tcp.close(); udp.close(); selector.close(); }
         catch (IOException failure) { throw new UncheckedIOException(failure); }
     }
diff --git a/src/test/java/arena/AckScenario.java b/src/test/java/arena/AckScenario.java
index a69bfed..fd3504a 100644
--- a/src/test/java/arena/AckScenario.java
+++ b/src/test/java/arena/AckScenario.java
@@ -201,7 +201,7 @@ final class AckScenario {
     }
     private static void check(boolean condition, ArrayNode failures, String message) { if (!condition) failures.add(message); }
 
-    private static final class Replica {
+    static final class Replica {
         final TreeMap<Long, ArrayNode> retained = new TreeMap<>(); ArrayNode state = Json.MAPPER.createArrayNode(); long sequence;
         String apply(ObjectNode snapshot) throws IOException {
             long next = snapshot.path("snapshot_seq").asLong(); if (next <= sequence) return "IGNORED_OLD";
diff --git a/src/test/java/arena/ReconnectScenario.java b/src/test/java/arena/ReconnectScenario.java
new file mode 100644
index 0000000..a5dcecf
--- /dev/null
+++ b/src/test/java/arena/ReconnectScenario.java
@@ -0,0 +1,321 @@
+package arena;
+
+import com.fasterxml.jackson.databind.JsonNode;
+import com.fasterxml.jackson.databind.node.ArrayNode;
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import java.io.IOException;
+import java.nio.channels.DatagramChannel;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.security.MessageDigest;
+import java.util.ArrayList;
+import java.util.HexFormat;
+import java.util.LinkedHashMap;
+import java.util.List;
+import java.util.Map;
+import java.util.concurrent.TimeUnit;
+
+/** One frozen three-case lifecycle pass, using the existing TCP/UDP client and owner fixture. */
+final class ReconnectScenario {
+    static final String SHA = "f4b892f5155655d41b52f197c1174c7c41f7fe75cd3f75e6f0f5b2f8e8c261c7";
+    record Observed(ObjectNode result, List<byte[]> artifacts) { }
+
+    static Observed run(Path path) throws Exception {
+        byte[] bytes = Files.readAllBytes(path);
+        require(HexFormat.of().formatHex(MessageDigest.getInstance("SHA-256").digest(bytes)).equals(SHA), "frozen G11 scenario changed");
+        ObjectNode fixed = Json.read(bytes), result = Json.MAPPER.createObjectNode().put("thread", "G11").put("scenario_sha256", SHA)
+                .put("process_id", ProcessHandle.current().pid()).put("fault_runs", 0).put("load_runs", 0);
+        ArrayNode cases = result.putArray("cases"), failures = result.putArray("failures"); List<byte[]> artifacts = new ArrayList<>();
+        int totalTicks = 0, totalInputs = 0; boolean released = true;
+        for (JsonNode definition : fixed.withArray("cases")) {
+            ObjectNode cell = cases.addObject().put("name", definition.path("name").asText()); Case run = null; byte[] artifact = null;
+            try {
+                run = new Case(fixed, cell); run.setup(); run.lifecycle(definition);
+                cell.set("final_capture", capture(run.server)); cell.set("runtime_metrics", run.server.metrics());
+                artifact = run.server.replayArtifact(); cell.put("passed", true);
+            } catch (Exception failure) { failure(cell, failures, failure); }
+            finally {
+                if (run != null) {
+                    if (artifact == null) try { artifact = run.server.replayArtifact(); } catch (Exception unavailable) { cell.put("artifact_unavailable", true); }
+                    try { run.close(); } catch (Exception failure) { failure(cell, failures, failure); }
+                    cell.put("executed_ticks", run.ticks.size()); totalTicks += run.ticks.size(); totalInputs += run.inputs.size();
+                    released &= cell.path("all_resources_released").asBoolean();
+                } else released = false;
+            }
+            artifacts.add(artifact);
+            if (!cell.path("passed").asBoolean()) break;
+        }
+        result.put("executed_ticks", totalTicks).put("input_attempts", totalInputs).put("all_resources_released", released)
+                .put("passed", failures.isEmpty() && released && cases.size() == 3 && totalTicks == 1_008 && totalInputs == 24);
+        return new Observed(result, artifacts);
+    }
+
+    private static void failure(ObjectNode cell, ArrayNode failures, Exception failure) {
+        cell.put("passed", false); failures.add(cell.path("name").asText() + ": " + failure.getClass().getSimpleName() + ": " + failure.getMessage());
+        java.io.StringWriter trace = new java.io.StringWriter(); failure.printStackTrace(new java.io.PrintWriter(trace)); cell.put("execution_error", trace.toString());
+    }
+
+    private static final class Case {
+        final ObjectNode fixed, result;
+        final ArenaServer server = new ArenaServer("127.0.0.1", 0, true);
+        final Map<String, TcpClient> clients = new LinkedHashMap<>();
+        final List<TcpClient> allClients = new ArrayList<>();
+        final Map<TcpClient, String> connectionIds = new LinkedHashMap<>();
+        final Map<String, AckScenario.Replica> replicas = new LinkedHashMap<>();
+        final ArrayNode ticks, records, hashes, inputs, boundaries;
+        final ObjectNode traffic;
+        int successfulReconnects;
+
+        Case(ObjectNode fixed, ObjectNode result) {
+            this.fixed = fixed; this.result = result; result.put("process_id", ProcessHandle.current().pid());
+            ticks = result.putArray("ticks"); records = result.putArray("canonical_records"); hashes = result.putArray("state_hashes");
+            inputs = result.putArray("input_admissions"); boundaries = result.putArray("boundaries"); traffic = result.putObject("traffic");
+        }
+
+        void setup() throws Exception {
+            for (JsonNode player : fixed.withArray("players")) {
+                String role = player.path("client").asText(); clients.put(role, open()); replicas.put(role, new AckScenario.Replica());
+                traffic.putObject(role).put("snapshots", 0).put("acks", 0).put("full", 0).put("delta", 0)
+                        .put("disconnected_visible", 0).put("left_removed", 0).put("max_datagram_bytes", 0);
+            }
+            ObjectNode joined = ReplayFixture.joinFixed(server, fixed, clients); result.set("normal_joins", joined);
+            for (JsonNode join : joined.withArray("joins")) require(join.path("response").path("resume_token_present").asBoolean(), "join resume credential absent");
+            for (TcpClient client : clients.values()) client.bind();
+            for (var entry : clients.entrySet()) consume(entry.getKey(), entry.getValue(), false);
+            int setupTicks = fixed.path("setup_execute_ticks").asInt();
+            for (int tick = 0; tick < setupTicks; tick++) {
+                for (JsonNode event : fixed.withArray("setup_events")) if (event.path("before_tick").asInt() == tick) {
+                    TcpClient client = clients.get(event.path("client").asText()); ObjectNode command = input(client, event);
+                    ObjectNode observed = inputs.addObject().put("client", event.path("client").asText()).put("before_tick", tick);
+                    observed.set("logical_input", event.deepCopy()); client.send(command); ObjectNode reply = client.until("INPUT_ACK"); observed.set("reply", reply);
+                    require(reply.path("status").asText().equals("ACCEPTED")
+                            && reply.path("seq").bigIntegerValue().equals(command.path("seq").bigIntegerValue()), "fixed setup admission");
+                }
+                step();
+            }
+            ObjectNode state = ReplayFixture.snapshot(server); checkPlayer(state, "player-00", 50_400, "EAST", "CONNECTED", 20, 1, 200);
+            checkPlayer(state, "player-01", 50_000, "STOP", "CONNECTED", 3, 0, -20);
+            require(executed() == 202, "setup tick continuity");
+        }
+
+        void lifecycle(JsonNode definition) throws Exception {
+            TcpClient original = clients.get("alpha"); disconnect(original, "original");
+            String name = definition.path("name").asText();
+            if (name.equals("immediate-and-credential-reuse")) {
+                TcpClient first = resume(original, "R0-to-R1"); disconnect(first, "first-replacement");
+                rejectedResume(original, "RECONNECT_INVALID", "reuse-R0-while-disconnected");
+                TcpClient current = resume(first, "R1-to-R2");
+                ObjectNode repeatedBind = boundaries.addObject().put("kind", "REUSE_BIND_B2"); ObjectNode before = capture(server);
+                current.send(current.binding()); ObjectNode rejected = current.control(); repeatedBind.set("reply", rejected); repeatedBind.set("before", before);
+                ObjectNode after = capture(server); repeatedBind.set("after", after);
+                require(rejected.path("code").asText().equals("UDP_BIND_INVALID") && sameAuthority(before, after), "consumed bind rejection changed authority");
+                require(before.path("recovery").equals(after.path("recovery")), "consumed bind changed endpoint or credentials");
+                ObjectNode duplicate = input(current, fixed.withArray("setup_events").get(6));
+                probeInput(current, current, duplicate, "current", "DUPLICATE"); step();
+                checkPlayer(ReplayFixture.snapshot(server), "player-00", 50_400, "STOP", "CONNECTED", 20, 1, 200);
+                ObjectNode fresh = current.auth("INPUT", current.roomId).put("seq", 21).put("target_tick", 203).put("direction", "EAST").putNull("tag_target_player_id");
+                require(original.tcpClosed() && ((DatagramChannel) ReplayFixture.field(original, "udp")).isOpen()
+                        && !original.localUdpEndpoint().equals(current.localUdpEndpoint()), "old endpoint must be a live distinct UDP socket");
+                probeInput(original, current, fresh, "original-retired", "UDP_BIND_INVALID");
+                probeInput(current, current, fresh, "current", "ACCEPTED"); step();
+                checkPlayer(ReplayFixture.snapshot(server), "player-00", 50_800, "EAST", "CONNECTED", 21, 1, 200);
+            } else {
+                int wait = definition.path("wait_grace_ticks").asInt(); for (int i = 0; i < wait; i++) step();
+                int expectedBoundary = definition.path("reconnect_next_tick").asInt(); require(executed() == expectedBoundary, "fixed grace boundary");
+                if (name.equals("last-valid-boundary")) {
+                    checkPlayer(ReplayFixture.snapshot(server), "player-00", 50_400, "STOP", "DISCONNECTED", 20, 1, 200);
+                    resume(original, "last-valid-R0-to-R1"); step();
+                    checkPlayer(ReplayFixture.snapshot(server), "player-00", 50_400, "STOP", "CONNECTED", 20, 1, 200);
+                } else {
+                    require(name.equals("expired-boundary"), "unknown fixed case");
+                    checkPlayer(ReplayFixture.snapshot(server), "player-00", 50_400, "STOP", "LEFT", 20, 1, 200);
+                    rejectedResume(original, "RECONNECT_EXPIRED", "current-unused-R0-after-expiry");
+                }
+            }
+            require(executed() == definition.path("executed_ticks").asInt(), "case tick count");
+            checkPlayer(ReplayFixture.snapshot(server), "player-01", 50_000, "STOP", "CONNECTED", 3, 0, -20);
+            int expectedAlpha = name.equals("immediate-and-credential-reuse") ? 105 : name.equals("last-valid-boundary") ? 104 : 102;
+            require(traffic.path("alpha").path("snapshots").asInt() == expectedAlpha
+                    && traffic.path("bravo").path("snapshots").asInt() == executed() / 2 + 1, "exact publication counts");
+            for (String role : List.of("alpha", "bravo")) require(traffic.path(role).path("acks").equals(traffic.path(role).path("snapshots")), "one actual ACK per applied snapshot");
+            result.put("successful_reconnects", successfulReconnects);
+        }
+
+        TcpClient open() throws Exception {
+            TcpClient client = new TcpClient(server.port()); allClients.add(client); client.hello();
+            connectionIds.put(client, ReplayFixture.owned(server, () -> {
+                for (var entry : ((Map<?, ?>) ReplayFixture.field(server, "sessions")).entrySet())
+                    if (ReplayFixture.field(entry.getValue(), "id").equals(client.sessionId))
+                        return ((io.netty.channel.Channel) ReplayFixture.field(entry.getKey(), "channel")).id().asLongText();
+                throw new IOException("HELLO connection missing");
+            }));
+            return client;
+        }
+        int executed() throws Exception { return ReplayFixture.snapshot(server).path("executed_ticks").asInt(); }
+
+        void disconnect(TcpClient client, String source) throws Exception {
+            ObjectNode event = boundaries.addObject().put("kind", "TCP_CLOSE").put("connection_role", source); ObjectNode before = capture(server);
+            client.closeTcp(); clients.remove("alpha"); ObjectNode after = awaitDisconnect(server, "player-00");
+            event.set("before", before); event.set("after", after); event.put("tcp_closed", client.tcpClosed())
+                    .put("udp_socket_still_open", ((DatagramChannel) ReplayFixture.field(client, "udp")).isOpen());
+            ObjectNode expected = ((ObjectNode) before.path("state")).deepCopy();
+            ((ObjectNode) ReplayScenario.player(expected, "player-00")).put("direction", "STOP").put("connectivity", "DISCONNECTED");
+            require(expected.equals(after.path("state")) && after.path("next_tick").asInt() == 202, "disconnect preserved state and tick");
+            JsonNode recovery = after.path("recovery").path("player-00");
+            require(recovery.path("deadline_next_tick").asInt() == fixed.path("disconnect").path("deadline_next_tick").asInt()
+                    && !recovery.path("endpoint_bound").asBoolean() && recovery.path("retained_snapshots").asInt() == 0,
+                    "disconnect grace/transport release");
+        }
+
+        TcpClient resume(TcpClient previous, String label) throws Exception {
+            TcpClient replacement = open(); String provisional = replacement.sessionId; ObjectNode before = capture(server);
+            ObjectNode response = replacement.reconnectFrom(previous), after = capture(server);
+            ObjectNode event = boundaries.addObject().put("kind", "RECONNECT").put("credential_transition", label);
+            event.set("reply", safe(response)); event.set("before", before); event.set("after", after);
+            boolean stable = replacement.sessionId.equals(previous.sessionId) && replacement.playerId.equals(previous.playerId) && replacement.roomId.equals(previous.roomId);
+            boolean newConnection = !connectionIds.get(replacement).equals(connectionIds.get(previous));
+            boolean retired = ReplayFixture.owned(server, () -> {
+                for (Object session : ((Map<?, ?>) ReplayFixture.field(server, "sessions")).values()) if (ReplayFixture.field(session, "id").equals(provisional)) return false;
+                return !((Map<?, ?>) ReplayFixture.field(server, "recoverableSessions")).containsKey(provisional);
+            });
+            event.put("stable_identity", stable).put("new_connection", newConnection).put("provisional_session_retired", retired)
+                    .put("resume_rotated", replacement.resumeRotatedFrom(previous)).put("bind_rotated", replacement.bindRotatedFrom(previous));
+            require(response.path("type").asText().equals("RECONNECTED") && response.path("last_accepted_seq").asInt() == 20
+                    && stable && newConnection && retired && replacement.resumeRotatedFrom(previous) && replacement.bindRotatedFrom(previous), "resume identity/credential guarantees");
+            require(before.path("next_tick").equals(after.path("next_tick")) && after.path("active_sessions").asInt() == 2
+                    && after.path("recoverable_sessions").asInt() == 2, "resume must retire provisional session without ticking");
+            checkPlayer((ObjectNode) after.path("state"), "player-00", 50_400, "STOP", "CONNECTED", 20, 1, 200);
+            clients.put("alpha", replacement); successfulReconnects++;
+            ObjectNode bound = replacement.bind(); ObjectNode full = consume("alpha", replacement, true);
+            ObjectNode binding = boundaries.addObject().put("kind", "FRESH_UDP_BIND").put("endpoint_role", "current-replacement");
+            binding.set("reply", bound); binding.set("immediate_full", full);
+            require(full.path("wire").path("kind").asText().equals("FULL") && full.path("wire").path("base_snapshot_seq").isNull()
+                    && full.path("next_tick").equals(after.path("next_tick")), "first fresh-bound publication must be immediate FULL");
+            int historical = full.path("wire").path("tick").asInt();
+            boolean currentHash = !full.path("wire").path("state_hash").equals(hashes.get(historical));
+            full.put("differs_from_historical_tick_hash", currentHash); require(currentHash, "immediate capture must not reuse historical tick hash");
+            return replacement;
+        }
+
+        void rejectedResume(TcpClient previous, String expected, String label) throws Exception {
+            TcpClient attempted = open(); ObjectNode before = capture(server);
+            boolean currentCredential = ReplayFixture.owned(server, () -> {
+                Object session = ((Map<?, ?>) ReplayFixture.field(server, "recoverableSessions")).get(previous.sessionId);
+                return ReplayFixture.field(session, "resumeToken").equals(ReplayFixture.field(previous, "resumeToken"));
+            });
+            ObjectNode response = attempted.reconnectFrom(previous), after = capture(server);
+            ObjectNode event = boundaries.addObject().put("kind", "RECONNECT_REJECTED").put("attempt", label).put("credential_is_current", currentCredential);
+            event.set("reply", safe(response)); event.set("before", before); event.set("after", after);
+            require(response.path("code").asText().equals(expected) && sameAuthority(before, after)
+                    && before.path("recovery").equals(after.path("recovery")), "rejected resume changed state or binding");
+            require(currentCredential == expected.equals("RECONNECT_EXPIRED"), "reuse/expiry must isolate the intended credential condition");
+            attempted.close(); awaitSessions(server, 1);
+        }
+
+        void probeInput(TcpClient sender, TcpClient controlRecipient, ObjectNode command, String endpoint, String expected) throws Exception {
+            ObjectNode before = capture(server), event = inputs.addObject().put("client", "alpha").put("before_tick", executed()).put("endpoint_role", endpoint);
+            event.set("logical_input", safe(command)); event.set("before", before); sender.send(command);
+            ObjectNode reply = expected.equals("UDP_BIND_INVALID") ? controlRecipient.control() : controlRecipient.until("INPUT_ACK");
+            ObjectNode after = capture(server); event.set("reply", reply); event.set("after", after);
+            require(reply.path(expected.equals("UDP_BIND_INVALID") ? "code" : "status").asText().equals(expected), "input probe result");
+            if (!expected.equals("ACCEPTED")) require(sameAuthority(before, after), "duplicate or stale endpoint changed authority");
+            else require(ReplayScenario.player((ObjectNode) after.path("state"), "player-00").path("pending_inputs").asInt() == 1, "fresh accepted input was not pending");
+        }
+
+        void step() throws Exception {
+            server.advanceTicks(1); ObjectNode state = ReplayFixture.snapshot(server); String record = ReplayFixture.canonicalRecord(server);
+            require(state.path("tick").asInt() == ticks.size() && state.path("state_hash").asText().equals(ReplayLog.hash(record)), "tick record/hash continuity");
+            ticks.add(state); records.add(record); hashes.add(state.path("state_hash"));
+            if (state.path("executed_ticks").asInt() % 2 == 0) for (var entry : clients.entrySet()) consume(entry.getKey(), entry.getValue(), false);
+        }
+
+        ObjectNode consume(String role, TcpClient client, boolean boundary) throws Exception {
+            ObjectNode wire = client.until("SNAPSHOT"), state = ReplayFixture.snapshot(server); String current = ReplayFixture.canonicalRecord(server);
+            AckScenario.Replica replica = replicas.get(role); String applied = replica.apply(wire); int bytes = Json.bytes(wire).length;
+            require(applied.equals("APPLIED") && replica.state.equals(SnapshotScenario.projection(state))
+                    && wire.path("tick").asInt() == state.path("tick").asInt() && wire.path("state_hash").asText().equals(ReplayLog.hash(current))
+                    && bytes <= 1_200, "snapshot projection/current hash");
+            ObjectNode count = (ObjectNode) traffic.path(role); increment(count, "snapshots"); increment(count, wire.path("kind").asText().equals("FULL") ? "full" : "delta");
+            count.put("last_sequence", replica.sequence).put("max_datagram_bytes", Math.max(count.path("max_datagram_bytes").asInt(), bytes));
+            for (JsonNode player : replica.state) if (player.path("connectivity").asText().equals("DISCONNECTED")) increment(count, "disconnected_visible");
+            if (!wire.withArray("removed_player_ids").isEmpty()) increment(count, "left_removed");
+            ObjectNode ack = client.auth("SNAPSHOT_ACK", client.roomId).put("snapshot_seq", replica.sequence).put("state_hash", wire.path("state_hash").asText());
+            int received = ReplayFixture.udpReceived(server); client.send(ack); ReplayFixture.udpBarrier(server, received + 1); increment(count, "acks");
+            if (boundary) {
+                ObjectNode observed = capture(server); observed.set("wire", wire); observed.set("ack", safe(ack)); observed.set("client_projection", replica.state.deepCopy());
+                return observed;
+            }
+            return wire;
+        }
+
+        void close() throws Exception {
+            Exception first = null;
+            for (TcpClient client : allClients) try { client.close(); } catch (Exception failure) { if (first == null) first = failure; }
+            try { server.close(); } catch (Exception failure) { if (first == null) first = failure; }
+            ObjectNode cleanup = server.cleanup(); result.set("cleanup", cleanup);
+            boolean closed = allClients.stream().allMatch(TcpClient::isClosed), credentialsReleased = true;
+            for (TcpClient client : allClients) credentialsReleased &= ReplayFixture.field(client, "bindToken") == null && ReplayFixture.field(client, "resumeToken") == null;
+            result.put("all_client_sockets_closed", closed).put("client_credentials_released", credentialsReleased);
+            ScenarioRunner.assertCleanup(cleanup);
+            boolean empty = ((Map<?, ?>) ReplayFixture.field(server, "sessions")).isEmpty() && ((Map<?, ?>) ReplayFixture.field(server, "recoverableSessions")).isEmpty();
+            result.put("all_resources_released", closed && credentialsReleased && empty && cleanup.path("resume_credentials").asInt() == 0
+                    && cleanup.path("bind_credentials").asInt() == 0 && cleanup.path("retained_snapshots").asInt() == 0 && cleanup.path("replay_bytes").asInt() == 0);
+            if (first != null) throw first;
+        }
+    }
+
+    private static ObjectNode input(TcpClient client, JsonNode event) {
+        ObjectNode command = client.auth("INPUT", client.roomId).put("seq", event.path("seq").bigIntegerValue())
+                .put("target_tick", event.path("target_tick").bigIntegerValue()).put("direction", event.path("direction").asText());
+        JsonNode target = event.path("tag_target_role");
+        if (target.isNull()) command.putNull("tag_target_player_id"); else command.put("tag_target_player_id", target.asText().equals("alpha") ? "player-00" : "player-01");
+        return command;
+    }
+    private static ObjectNode capture(ArenaServer server) throws Exception {
+        return ReplayFixture.owned(server, () -> {
+            Room room = (Room) ReplayFixture.field(server, "room"); String record = room.canonicalRecord();
+            ObjectNode result = Json.MAPPER.createObjectNode().put("next_tick", room.executedTicks()).put("current_canonical_record", record)
+                    .put("current_canonical_hash", ReplayLog.hash(record)); result.set("state", ReplayFixture.view(room));
+            Map<?, ?> sessions = (Map<?, ?>) ReplayFixture.field(server, "sessions"), recoverable = (Map<?, ?>) ReplayFixture.field(server, "recoverableSessions");
+            result.put("active_sessions", sessions.size()).put("recoverable_sessions", recoverable.size()); ObjectNode recovery = result.putObject("recovery");
+            for (Object session : recoverable.values()) {
+                String player = (String) ReplayFixture.field(session, "playerId"); SnapshotStream stream = (SnapshotStream) ReplayFixture.field(session, "snapshots");
+                recovery.putObject(player).put("deadline_next_tick", room.player(player).disconnectDeadline)
+                        .put("endpoint_bound", ReplayFixture.field(session, "endpoint") != null).put("resume_credential_present", ReplayFixture.field(session, "resumeToken") != null)
+                        .put("bind_credential_present", ReplayFixture.field(session, "bindToken") != null).put("retained_snapshots", stream.retainedCount())
+                        .put("active_connection", sessions.containsValue(session));
+            }
+            return result;
+        });
+    }
+    private static ObjectNode awaitDisconnect(ArenaServer server, String playerId) throws Exception {
+        long deadline = System.nanoTime() + TimeUnit.SECONDS.toNanos(5);
+        while (true) {
+            ObjectNode observed = capture(server);
+            if (!ReplayScenario.player((ObjectNode) observed.path("state"), playerId).path("connectivity").asText().equals("CONNECTED")) return observed;
+            if (System.nanoTime() >= deadline) throw new IOException("owner disconnect barrier ceiling");
+        }
+    }
+    private static void awaitSessions(ArenaServer server, int count) throws Exception {
+        long deadline = System.nanoTime() + TimeUnit.SECONDS.toNanos(5);
+        while (capture(server).path("active_sessions").asInt() != count)
+            if (System.nanoTime() >= deadline) throw new IOException("owner session-close barrier ceiling");
+    }
+    private static ObjectNode safe(ObjectNode message) {
+        ObjectNode safe = message.deepCopy(); safe.remove(List.of("udp_bind_token", "resume_token"));
+        if (safe.has("session_id")) { safe.remove("session_id"); safe.put("session_alias", "alpha-original"); }
+        return safe;
+    }
+    private static boolean sameAuthority(ObjectNode before, ObjectNode after) {
+        return before.path("state").equals(after.path("state")) && before.path("current_canonical_record").equals(after.path("current_canonical_record"));
+    }
+    private static void checkPlayer(ObjectNode state, String id, int x, String direction, String connectivity, int seq, int score, int lastTag) throws IOException {
+        JsonNode player = ReplayScenario.player(state, id);
+        require(player.path("x").asInt() == x && player.path("y").asInt() == 50_000 && player.path("direction").asText().equals(direction)
+                && player.path("connectivity").asText().equals(connectivity) && player.path("last_accepted_seq").asInt() == seq
+                && player.path("score").asInt() == score && player.path("last_tag_tick").asInt() == lastTag && player.path("pending_inputs").asInt() == 0,
+                "fixed player state " + id);
+    }
+    private static void increment(ObjectNode value, String field) { value.put(field, value.path(field).asInt() + 1); }
+    private static void require(boolean condition, String message) throws IOException { if (!condition) throw new IOException(message); }
+}
diff --git a/src/test/java/arena/ReplayFormatTest.java b/src/test/java/arena/ReplayFormatTest.java
index cedf7d6..186224b 100644
--- a/src/test/java/arena/ReplayFormatTest.java
+++ b/src/test/java/arena/ReplayFormatTest.java
@@ -50,7 +50,8 @@ final class ReplayFormatTest {
             assertNull(jar.getJarEntry("G08.json"));
             assertNull(jar.getJarEntry("G09.json"));
             assertNull(jar.getJarEntry("G10.json"));
-            for (String name : List.of("ReplayFixture", "ReplayScenario", "ReplayVerifier", "ReplayTool", "G07BaselineTest", "SnapshotScenario", "G08BaselineTest", "UdpScenario", "UdpFaultProxy", "UdpBoundaryTest", "G09BaselineTest", "AckScenario", "G10BaselineTest")) {
+            assertNull(jar.getJarEntry("G11.json"));
+            for (String name : List.of("ReplayFixture", "ReplayScenario", "ReplayVerifier", "ReplayTool", "G07BaselineTest", "SnapshotScenario", "G08BaselineTest", "UdpScenario", "UdpFaultProxy", "UdpBoundaryTest", "G09BaselineTest", "AckScenario", "G10BaselineTest", "ReconnectScenario", "G11BaselineTest")) {
                 assertNull(jar.getJarEntry("arena/" + name + ".class"));
                 assertThrows(ClassNotFoundException.class, () -> Class.forName("arena." + name, false, production));
             }
diff --git a/src/test/java/arena/ReplayTool.java b/src/test/java/arena/ReplayTool.java
index f36d477..9e94f67 100644
--- a/src/test/java/arena/ReplayTool.java
+++ b/src/test/java/arena/ReplayTool.java
@@ -32,6 +32,17 @@ public final class ReplayTool {
                     result.put("replay_artifact_path", artifact.toAbsolutePath().toString()).put("artifact_sha256",
                             HexFormat.of().formatHex(MessageDigest.getInstance("SHA-256").digest(observed.replay())));
                 }
+            } else if (thread.equals("G11")) {
+                if (args.length != 3) throw new IllegalArgumentException("G11 has one fixed three-case pass");
+                ReconnectScenario.Observed observed = ReconnectScenario.run(input); result = observed.result();
+                for (int i = 0; i < observed.artifacts().size(); i++) {
+                    byte[] bytes = observed.artifacts().get(i); if (bytes == null) continue;
+                    ObjectNode cell = (ObjectNode) result.withArray("cases").get(i);
+                    Path artifact = output.resolveSibling(output.getFileName().toString().replaceFirst("\\.json$", "") + "." + cell.path("name").asText() + ".replay.jsonl");
+                    Files.write(artifact, bytes, StandardOpenOption.CREATE_NEW);
+                    cell.put("replay_artifact_path", artifact.toAbsolutePath().toString()).put("artifact_bytes", bytes.length)
+                            .put("artifact_sha256", HexFormat.of().formatHex(MessageDigest.getInstance("SHA-256").digest(bytes)));
+                }
             } else {
                 if (args.length != 3) throw new IllegalArgumentException("variant is only active for G07");
                 result = ScenarioRunner.run(input);
diff --git a/src/test/java/arena/ReplayVerifier.java b/src/test/java/arena/ReplayVerifier.java
index 6c4dc4b..65be62b 100644
--- a/src/test/java/arena/ReplayVerifier.java
+++ b/src/test/java/arena/ReplayVerifier.java
@@ -51,9 +51,19 @@ final class ReplayVerifier {
                     }
                     case "LEFT" -> {
                         boundary(entry, room); String player = Json.text(entry, "player_id");
-                        if (room.player(player) == null || !room.player(player).connected) throw new IOException("invalid LEFT record");
+                        if (room.player(player) == null || !room.player(player).connected()) throw new IOException("invalid LEFT record");
                         room.left(player); lifecycle.add(entry);
                     }
+                    case "DISCONNECTED" -> {
+                        boundary(entry, room); String player = Json.text(entry, "player_id");
+                        if (room.player(player) == null || !room.player(player).connected()) throw new IOException("invalid disconnect record");
+                        room.disconnect(player); lifecycle.add(entry);
+                    }
+                    case "RECONNECTED" -> {
+                        boundary(entry, room); String player = Json.text(entry, "player_id");
+                        if (room.reconnect(player) != null) throw new IOException("invalid reconnect record");
+                        lifecycle.add(entry);
+                    }
                     case "TICK" -> {
                         if (!entry.path("tick").isIntegralNumber() || entry.path("tick").asInt(-1) != room.executedTicks()
                                 || room.executedTicks() >= Room.DURATION) throw new IOException("noncontiguous replay tick");
diff --git a/src/test/java/arena/UdpBoundaryTest.java b/src/test/java/arena/UdpBoundaryTest.java
index 1705517..53f3c08 100644
--- a/src/test/java/arena/UdpBoundaryTest.java
+++ b/src/test/java/arena/UdpBoundaryTest.java
@@ -178,11 +178,28 @@ final class UdpBoundaryTest {
             }
             result.put("generated_room_id_length", world.room.length()); ArrayNode lengths = result.putArray("generated_player_id_lengths");
             for (TcpClient client : world.clients) lengths.add(client.playerId.length());
-            ObjectNode maximum = widthFixture(world.room, world.clients.stream().map(client -> client.playerId).toList());
+            List<String> identifiers = world.clients.stream().map(client -> client.playerId).toList();
+            ObjectNode maximum = widthFixture(world.room, identifiers, 0);
             byte[] maximumBytes = UdpData.bytes(maximum);
             result.put("worst_visible_envelope_bytes", maximumBytes.length).set("worst_visible_envelope", maximum);
+            // G11 extends this same zero-tick probe without another network World or matrix case.
+            ObjectNode disconnected = widthFixture(world.room, identifiers, 7);
+            byte[] disconnectedBytes = UdpData.bytes(disconnected);
+            require(disconnected.path("kind").asText().equals("FULL") && disconnected.withArray("players").size() == 8
+                    && disconnected.withArray("removed_player_ids").isEmpty() && disconnectedBytes.length <= 1_200,
+                    "seven-disconnected eight-player FULL bound");
+            for (int index = 0; index < identifiers.size(); index++) {
+                JsonNode player = ReplayScenario.player(disconnected, identifiers.get(index));
+                require(player.size() == 7, "disconnected projection field count");
+                for (String field : List.of("player_id", "slot", "x", "y", "direction", "score", "connectivity"))
+                    require(player.has(field), "disconnected projection field " + field);
+                require(player.path("connectivity").asText().equals(index < 7 ? "DISCONNECTED" : "CONNECTED")
+                        && player.path("direction").asText().equals(index < 7 ? "STOP" : "NORTH"),
+                        "disconnected projection state");
+            }
+            result.put("seven_disconnected_envelope_bytes", disconnectedBytes.length).set("seven_disconnected_envelope", disconnected);
             List<String> wide = new ArrayList<>(); for (int i = 0; i < 8; i++) wide.add("p" + "x".repeat(62) + i);
-            ObjectNode oversized = widthFixture("r".repeat(64), wide); int oversizedBytes = Json.bytes(oversized).length;
+            ObjectNode oversized = widthFixture("r".repeat(64), wide, 0); int oversizedBytes = Json.bytes(oversized).length;
             boolean rejected = false;
             try { UdpData.bytes(oversized); } catch (IllegalArgumentException expected) { rejected = true; }
             require(rejected && oversizedBytes > 1_200, "maximum-identifier fixture must explicitly reject");
@@ -205,12 +222,13 @@ final class UdpBoundaryTest {
         } finally { world.close(); result.set("cleanup", world.cleanup()); }
     }
 
-    private static ObjectNode widthFixture(String roomId, List<String> identifiers) {
+    private static ObjectNode widthFixture(String roomId, List<String> identifiers, int disconnectedPlayers) {
         Room room = new Room(roomId); SnapshotStream stream = new SnapshotStream();
         try {
             for (String id : identifiers) room.join(id);
             for (String id : identifiers) { Room.Player player = room.player(id); player.x = 100_000; player.y = 100_000; player.direction = Room.Direction.NORTH; player.score = 10; }
             for (String id : identifiers) room.ready(id);
+            for (int index = 0; index < disconnectedPlayers; index++) room.disconnect(identifiers.get(index));
             // Pure serializer width fixture; no simulation/clock iteration is invoked.
             return stream.next(room, true).put("tick", 1_199).put("snapshot_seq", 601).put("status", "FINISHED");
         } finally { stream.close(); room.close(); }
@@ -269,7 +287,10 @@ final class UdpBoundaryTest {
             for (Object session : retainedSessions) released &= ReplayFixture.field(session, "bindToken") == null && ReplayFixture.field(session, "endpoint") == null;
             cleanup.put("client_and_binding_resources_released", released);
             cleanup.put("session_registry_size", ((Map<?, ?>) ReplayFixture.field(server, "sessions")).size());
-            require(released && cleanup.path("session_registry_size").asInt() == 0, "client/binding cleanup"); return cleanup;
+            require(released && cleanup.path("session_registry_size").asInt() == 0, "client/binding cleanup");
+            require(cleanup.path("recoverable_sessions").asInt(-1) == 0 && cleanup.path("resume_credentials").asInt(-1) == 0,
+                    "reconnect credential cleanup");
+            return cleanup;
         }
         @Override public void close() { server.close(); for (TcpClient client : clients) client.close(); }
     }
diff --git a/src/test/resources/G11.json b/src/test/resources/G11.json
new file mode 100644
index 0000000..b53638e
--- /dev/null
+++ b/src/test/resources/G11.json
@@ -0,0 +1,152 @@
+{
+  "scenario_id": "G11-resume-grace",
+  "contract_version": 1,
+  "thread": "G11",
+  "seed": 7050,
+  "clock": {
+    "kind": "manual",
+    "tick_duration_ms": 50
+  },
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
+    }
+  ],
+  "setup_events": [
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
+      "before_tick": 0,
+      "client": "bravo",
+      "seq": 1,
+      "target_tick": 0,
+      "direction": "WEST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 100,
+      "client": "alpha",
+      "seq": 2,
+      "target_tick": 100,
+      "direction": "NORTH",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 100,
+      "client": "bravo",
+      "seq": 2,
+      "target_tick": 100,
+      "direction": "SOUTH",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 200,
+      "client": "alpha",
+      "seq": 3,
+      "target_tick": 200,
+      "direction": "STOP",
+      "tag_target_role": "bravo",
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 200,
+      "client": "bravo",
+      "seq": 3,
+      "target_tick": 200,
+      "direction": "STOP",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 201,
+      "client": "alpha",
+      "seq": 20,
+      "target_tick": 201,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    }
+  ],
+  "setup_execute_ticks": 202,
+  "disconnect": {
+    "client": "alpha",
+    "owner_commit_before_tick": 202,
+    "keep_old_udp_socket_open": true,
+    "expected_connectivity": "DISCONNECTED",
+    "expected_direction": "STOP",
+    "grace_ticks": 200,
+    "deadline_next_tick": 402
+  },
+  "cases": [
+    {
+      "name": "immediate-and-credential-reuse",
+      "first_reconnect_next_tick": 202,
+      "actions": [
+        "HELLO then RECONNECT withR0; expectsameidentity/score/lastseq20 androtationR1",
+        "bindfreshB1; firstsnapshotFULL beforeanytick",
+        "close replacementTCP beforetick202; ownercommitsDISCONNECTED again",
+        "HELLO then tryoldR0 whileDISCONNECTED/withinGrace; expectRECONNECT_INVALID",
+        "HELLO then reconnectwithcurrentR1; expectsuccess androtationR2",
+        "bindfreshB2; firstsnapshotFULL; repeatconsumedB2 expectUDP_BIND_INVALID",
+        "sendoriginalseq20/EAST/target201 fromcurrentendpoint; expectDUPLICATE",
+        "execute tick202; alphaSTOP position[50400,50000] unchanged",
+        "beforetick203 sendseq21/EAST/target203 fromoriginaloldUDPendpoint; expectUDP_BIND_INVALID/noauthoritychange",
+        "sendidenticalseq21command fromcurrentboundendpoint; expectACCEPTED",
+        "execute tick203; alphaposition[50800,50000],lastseq21,score1"
+      ],
+      "executed_ticks": 204
+    },
+    {
+      "name": "last-valid-boundary",
+      "wait_grace_ticks": 199,
+      "reconnect_next_tick": 401,
+      "expect": "RECONNECTED sameidentity/score/position/lastseq20; freshbind thenFULL",
+      "execute_after_reconnect": [
+        401
+      ],
+      "executed_ticks": 402
+    },
+    {
+      "name": "expired-boundary",
+      "wait_grace_ticks": 200,
+      "reconnect_next_tick": 402,
+      "expect": "RECONNECT_EXPIRED; alreadyLEFT/STOP; no newplayer",
+      "executed_ticks": 402
+    }
+  ],
+  "reconnect_wire": {
+    "v": 1,
+    "type": "RECONNECT",
+    "session_id": "existing session",
+    "room_id": "existing room",
+    "resume_token": "current opaque one-time token"
+  },
+  "socket_ceiling_ms": 5000,
+  "credential_evidence": "private equality/rotation comparisons and codes only; never log/store token bytes"
+}
