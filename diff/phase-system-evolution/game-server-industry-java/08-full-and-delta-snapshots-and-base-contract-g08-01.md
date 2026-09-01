# Full·Delta Snapshot과 Base Contract (G08)

## `feat: publish acknowledged full and delta snapshots`

diff --git a/TRACK.md b/TRACK.md
index a2bf87a..de9c85e 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -1,6 +1,6 @@
-# Java arena — through G07
+# Java arena — through G08
 
-Current thread: G07 (G01–G06 regressions retained). Phase: phase-1. Profile: realtime-core. Spec revision: `c1d62196ab76b55652f5d75a67514f8c6d8210ce`.
+Current thread: G08 (G01–G07 regressions retained). Phase: phase-1. Profile: realtime-core. Spec revision: `c1d62196ab76b55652f5d75a67514f8c6d8210ce`.
 G01–G05 retain their original spec trailers and verification. The user-authorized spec/profile transition changes procedure only; the game contract is unchanged. Scope remains G01–G14, with no G15+, external infrastructure or push.
 
 ## Frozen versions
@@ -28,6 +28,7 @@ The wrapper uses the locally installed Temurin path when JAVA_HOME is unset. On
 ./track scenario-run /absolute/path/to/G06.json /absolute/path/to/authority-evidence.json
 ./track scenario-run /absolute/path/to/G07.json /absolute/path/to/L1.json
 ./track scenario-run /absolute/path/to/G07.json /absolute/path/to/V.json --variant rejected-removed
+./track scenario-run /absolute/path/to/G08.json /absolute/path/to/snapshot-evidence.json
 ./track replay-verify /absolute/path/to/replay /absolute/path/to/evidence
 ./track server config/server.json
 ```
@@ -112,7 +113,15 @@ The JSON-lines artifact has HEADER, INPUT, LEFT and TICK records in owner order.
 
 `scenario-run G07.json result.json` executes one200-tick live pass and writes `result.replay.jsonl` beside the result, refusing to overwrite an existing artifact. `--variant rejected-removed` removes only the manifest's four rejected INPUT attempts. It retains the real LEAVE and charlie's accepted TAG failure. `replay-verify artifact result.json` initializes the recorded roster in the test runtime, applies accepted events at their recorded admission boundaries, and uses the same Room ticks/hash code. It stops at the first mismatching expected hash/record, writes the actual divergent record and returns nonzero. Replay input bytes are bounded before parsing.
 
-G07 commands explicitly reserve L1/L2/O1/O2/V at200 ticks each, plus a separate38-tick negative invocation after changing only L1's expected hash37 in a disposable copy. No wrapper hides another pass. The reproduction-only200-tick test is archived and absent from the final suite; the three permanent G07 unit tests execute zero ticks and cover record format, immutable ownership/overflow/close release and production fixture exclusion. Prior stage regressions remain unchanged. Exact worker/root-ready argv and artifact paths are in `evidence/G07-command-ledger.jsonl`; outcomes are in `evidence/G07-verification.md`. G08+ features are inactive.
+G07 commands explicitly reserve L1/L2/O1/O2/V at200 ticks each, plus a separate38-tick negative invocation after changing only L1's expected hash37 in a disposable copy. No wrapper hides another pass. The reproduction-only200-tick test is archived and absent from the final suite; the three permanent G07 unit tests execute zero ticks and cover record format, immutable ownership/overflow/close release and production fixture exclusion. Prior stage regressions remain unchanged. Exact worker/root-ready argv and artifact paths are in `evidence/G07-command-ledger.jsonl`; outcomes are in `evidence/G07-verification.md`. G08+ features were inactive at that END.
+
+## G08 full/delta replication
+
+Each connected session owns a snapshot stream: Room-start FULL sequence1 at tick−1, then one publication after every two completed ticks. Sequence multiples of20, an absent acknowledged base, and session finish produce FULL; otherwise DELTA includes only changed seven-field player rows and sorted removed IDs. Visible rows contain `player_id`, `slot`, `x`, `y`, `direction`, `score`, and `connectivity`. LEFT rows stay in canonical state/replay but leave the visible projection. `state_hash` remains the hash of the complete canonical record, including retained LEFT and internal sequence/cooldown fields; a client cannot reconstruct that hash from the visible rows alone.
+
+Minimal authenticated TCP `SNAPSHOT_ACK` carries session/room/player identity and integer `snapshot_seq`. An unknown base only changes the next scheduled publication to FULL. Each owner-confined stream retains the latest32 immutable full visible states; disconnect and server shutdown release that storage. No new leave policy, wire message type, UDP, resync protocol or coalescing policy is introduced. Old tests consume the added snapshots with the existing64-frame bounds, without changing their simulation durations, INPUT boundaries or physical/lifecycle assertions.
+
+`scenario-run G08.json result.json` performs exactly one196-tick run with the shared test-only four-session bootstrap and writes the accepted-input replay beside the result. Real wire captures, ACKs, independent client application, full canonical records, retained base IDs and cleanup are retained in the result. HELLO/WELCOME observation markers drain frames without changing session/player identity or time. The full unit suite contains only one pure33-snapshot retention fixture (zero simulation ticks); the archived baseline and post-change196-tick campaign are not repeated there. Exact commands and raw locations are in `evidence/G08-command-ledger.jsonl`.
 
 G01 initial budget was build/compile <=8, unit suites <=4, integration suites <=2, canonical scenario <=1. Later Threads use their frozen active plans, including G07's explicit five-pass and negative-replay budget. Network-fault and load runs remain zero. No test sleep, microbenchmark, fuzzing, UDP, reconnect, many-room or distributed implementation is included.
 
diff --git a/evidence/G08-command-ledger.jsonl b/evidence/G08-command-ledger.jsonl
new file mode 100644
index 0000000..a5434aa
--- /dev/null
+++ b/evidence/G08-command-ledger.jsonl
@@ -0,0 +1,11 @@
+{"kind": "activation", "thread": "G08", "profile": "realtime-core", "spec_revision": "c1d62196ab76b55652f5d75a67514f8c6d8210ce", "start": "9f70abaf1c3419b58114d2f15c653f34f431936b", "attempt": "initial", "budget": {"compile": 8, "unit_including_reproduction": 4, "integration": 2, "post_canonical": 1, "network_fault": 0, "load": 0}, "production_hash_manifest": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-initial/reproduce-unit/production-hashes-before.json"}
+{"kind": "resolved_before_execution", "pass": "baseline", "category": "unit-reproduction", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test", "--tests", "arena.G08BaselineTest"], "environment": {"ARENA_G08_SCENARIO": "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G08.json", "ARENA_G08_EVIDENCE": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-initial/reproduce-unit/result.json"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-initial/reproduce-unit", "reserved_ticks": 196, "resolved_at": "2026-08-28T04:27:58.324558+00:00"}
+{"kind": "resolved_before_execution", "pass": "build", "category": "build", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "build"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-initial/verify-build", "reserved_ticks": 0, "resolved_at": "2026-08-28T04:27:58.324845+00:00"}
+{"kind": "resolved_before_execution", "pass": "unit", "category": "unit", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-initial/verify-unit", "reserved_ticks": 0, "resolved_at": "2026-08-28T04:27:58.324857+00:00"}
+{"kind": "resolved_before_execution", "pass": "integration", "category": "integration", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "integration-test"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-initial/verify-integration", "reserved_ticks": 0, "resolved_at": "2026-08-28T04:27:58.324866+00:00"}
+{"kind": "resolved_before_execution", "pass": "canonical", "category": "canonical", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "scenario-run", "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G08.json", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-initial/canonical/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-initial/canonical", "reserved_ticks": 196, "resolved_at": "2026-08-28T04:27:58.324878+00:00"}
+{"pass": "baseline", "category": "unit-reproduction", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test", "--tests", "arena.G08BaselineTest"], "environment": {"ARENA_G08_SCENARIO": "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G08.json", "ARENA_G08_EVIDENCE": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-initial/reproduce-unit/result.json"}, "kind": "executed", "started_at": "2026-08-28T04:28:14.318548+00:00", "duration_seconds": 5.248, "command_process_id": 68587, "exit_code": 1, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-initial/reproduce-unit/stdout.log", "process_terminated": true, "compiler_tasks_executed": ["> Task :compileTestJava"], "xml": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-initial/reproduce-unit/xml", "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-initial/reproduce-unit/result.json", "simulation_process_id": 68615, "executed_ticks": 196, "snapshot_counts": {"alpha": 1, "bravo": 1, "charlie": 1, "delta": 1}}
+{"pass": "build", "category": "build", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "build"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T04:38:05.781329+00:00", "duration_seconds": 6.491, "command_process_id": 71644, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-initial/verify-build/stdout.log", "process_terminated": true, "compiler_tasks_executed": ["> Task :compileJava", "> Task :compileTestJava"]}
+{"pass": "unit", "category": "unit", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T04:38:12.273937+00:00", "duration_seconds": 6.724, "command_process_id": 71697, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-initial/verify-unit/stdout.log", "process_terminated": true, "compiler_tasks_executed": [], "xml": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-initial/verify-unit/xml"}
+{"pass": "integration", "category": "integration", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "integration-test"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T04:38:19.001549+00:00", "duration_seconds": 6.441, "command_process_id": 71799, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-initial/verify-integration/stdout.log", "process_terminated": true, "compiler_tasks_executed": [], "xml": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-initial/verify-integration/xml"}
+{"pass": "canonical", "category": "canonical", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "scenario-run", "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G08.json", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-initial/canonical/result.json"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T04:39:04.500650+00:00", "duration_seconds": 1.195, "command_process_id": 72151, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-initial/canonical/stdout.log", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-initial/canonical/result.json", "simulation_process_id": 72151, "executed_ticks": 196, "snapshot_counts": {"alpha": 99, "bravo": 99, "charlie": 99, "delta": 98}}
diff --git a/evidence/G08-verification.md b/evidence/G08-verification.md
new file mode 100644
index 0000000..0cbcc52
--- /dev/null
+++ b/evidence/G08-verification.md
@@ -0,0 +1,28 @@
+# G08 — initial attempt
+
+START `9f70abaf1c3419b58114d2f15c653f34f431936b`; phase `phase-1`; profile `realtime-core`; spec `c1d62196ab76b55652f5d75a67514f8c6d8210ce`.
+Fixture SHA-256 `d121973167461dddbb3ba0bd339ed26b09486cda8bbec642328ccc5cea9e578e`. Exact pre-resolved and executed argv, environment, PID, duration, exit and raw paths are in `G08-command-ledger.jsonl`.
+
+**Reproduction:** all14 production source hashes matched Git START before and after the real196-tick run. The four real TCP/session bindings used the existing test-only owner bootstrap; one explicit owner call to the unchanged normal-start expression `broadcast(room.view("SNAPSHOT"))` supplied the actual old start frame. That bootstrap bridge is labeled separately, not called a missing production start feature. HELLO/WELCOME markers preserved the same session/player identity and clock. Each client received one old frame, with no sequence/kind; no ACK was fabricated. Actual intermediate publication was absent at391 client boundaries. Movement, TAG193, STOP194, real LEFT194 and all196 G07 hashes already passed. Baseline exit1 was one assertion failure, zero errors/skips. Sources, hashes, raw states/wire, replay, stdout and XML are preserved in `runs/g08-initial/reproduce-unit/`; the reproduction-only test was removed before the clean build. Main independently accepted the reproduction.
+
+**Change:** an owner-confined stream publishes start FULL1/tick−1, then every two completed ticks. It retains32 full visible bases, selects the acknowledged base, uses FULL at sequence multiples of20 or missing base, and reports changed seven-field player rows plus explicit removals. Minimal authenticated TCP ACK never advances time or publishes an extra snapshot. LEFT and internal sequence/cooldown fields stay in canonical state, but are absent from visible rows. Every wire `SNAPSHOT` has the new stream shape; finish uses the same publisher once. Room/gameplay, replay format/bytes, clock core and dependency/build files are unchanged. Old harnesses drain new snapshots without changing total ticks, input boundaries, physical assertions or64-frame bounds. Fixed-ID injection remains test-only.
+
+| Command | Exit | Seconds | Actual result |
+|---|---:|---:|---|
+| `./track unit-test --tests arena.G08BaselineTest` | 1 expected | 5.248 | 196tick real baseline; one assertion failure |
+| `./track build` | 0 | 6.491 | clean main/test compilation and shipping artifact |
+| `./track unit-test` | 0 | 6.724 | 42 tests, no errors/failures/skips |
+| `./track integration-test` | 0 | 6.441 | 4 tests, no errors/failures/skips |
+| `./track scenario-run <main-G08.json> <new-result.json>` | 0 | 1.195 | PID72151; one196-tick canonical |
+
+The single pure33-snapshot unit fixture executes zero game ticks. It observed IDs2..33, high-water32, unknown ACK999 causing FULL33, immutable retained data after emitted-frame mutation, foreign-owner rejection and close count0. Shipping JAR/isolated production loader exclude the fixture/tool classes and G08 resource. Observed installed JAR SHA-256: `2d9640c85b36ffd6b4160c3e73196f8310772d9d3538cdd4670932872894fe12`.
+
+**Actual canonical:** alpha/bravo/charlie each99 snapshots; delta98 before LEAVE;395 total captures. Alpha sequence1..99 has ticks−1,1,3,…195. FULL sequences are1,4,20,40,60,80; all other messages are DELTA. ACK1 selects base1 for2; ACK2 selects base2 for3; unknown999 gives FULL4/null at its normal boundary; remaining bases progress. All four streams have identical logical content over their common prefix. Each applied wire state equals the seven-field active-player projection; each metadata hash equals the separately captured complete canonical record, including LEFT. No client-side reconstruction of hidden canonical fields is claimed.
+
+DELTA98/tick193 records alpha `(87600,10000)`, EAST, score1. DELTA99/tick195 changes alpha to STOP and removes `player-03`; canonical state still contains delta LEFT. Room remains RUNNING. Final hash `2f41457f82749fe037faed4566ea907707c2ca65817081afd5af41fa1d9e2269`. All196 hashes and canonical records, and the exact replay bytes, equal the pre-change baseline. Replay has one HEADER,3 INPUT, one LEFT and196 TICK records:118410 bytes, SHA-256 `06381b6292f4cd7530277e94a63a1a45f3b5d919428215c4e872d359183e58b6`.
+
+Retention IDs at final capture are68..99 for each remaining client; per-client high-water32, total retained96 before close,0 after. Live input/mailbox/outbound high-waters are1/2/1; parser bytes/capacity225/256. Cleanup reports zero connections/channels, pending writes/mailbox, parser buffers/bytes, replay bytes, retained snapshots, clock backlog and owned threads; owner/event loops terminate and all client sockets close. PARANOID checks and prior regression assertions pass.
+
+Raw canonical evidence: `runs/g08-initial/canonical/{result.json,result.replay.jsonl,stdout.log}`. `result.json` contains every client capture, ACK, applied state, canonical record/hash, retained IDs and observation markers. Compact extracted logical content/resource comparison: `G08-result.json`. Root uses the same CLI and immutable main fixture with a fresh result path; the sibling replay artifact uses CREATE_NEW. No extra build/test is hidden in the tool.
+
+Budget:3 compiler tasks across2 compile-bearing commands /8;2 unit invocations including baseline /4;1 integration /2;1 post canonical /1. Four Gradle processes total. Pure retention33 snapshots/0ticks; baseline196 + post196 =392 G08-specific simulation ticks, excluding unchanged prior regressions. Only the expected baseline failed. A scoped helper performed read-only review with no executions or edits. Fault/load0/0. Unresolved: none. No G09+, external infrastructure, tags or push.
diff --git a/src/main/java/arena/ArenaServer.java b/src/main/java/arena/ArenaServer.java
index 9d7d2c1..818b1fe 100644
--- a/src/main/java/arena/ArenaServer.java
+++ b/src/main/java/arena/ArenaServer.java
@@ -70,11 +70,14 @@ public final class ArenaServer implements AutoCloseable {
     private volatile List<String> closedLifecycle = List.of();
     private volatile int closedInputHighWater;
     private volatile int closedReplayBytes;
+    private volatile int closedSnapshotCount;
+    private int snapshotRetentionHighWater;
     private volatile ObjectNode closedClockMetrics = Json.MAPPER.createObjectNode().put("active", false)
             .put("accumulator_ns", 0L).put("max_ticks_per_iteration", 0);
 
     private static final class Session {
         final String id = "s-" + UUID.randomUUID();
+        final SnapshotStream snapshots = new SnapshotStream();
         String playerId;
     }
 
@@ -253,10 +256,16 @@ public final class ArenaServer implements AutoCloseable {
                     peer.send(Json.message("ROOM_JOINED").put("room_id", room.id).put("player_id", player.id)
                             .put("slot", player.slot).put("status", room.status().name()));
                     if (room.status() == Room.Status.RUNNING) {
-                        fixedClock = new FixedTickClock(monotonicNanos);
-                        broadcast(room.view("SNAPSHOT"));
+                        startReplication();
                     }
                 }
+                case "SNAPSHOT_ACK" -> {
+                    if (!roomMatches(peer, message)) break;
+                    if (session.playerId == null || !session.playerId.equals(Json.text(message, "player_id"))) {
+                        peer.error("PLAYER_MISMATCH"); break;
+                    }
+                    session.snapshots.acknowledge(Json.snapshotSequence(message));
+                }
                 case "INPUT" -> {
                     if (!roomMatches(peer, message)) break;
                     if (session.playerId == null || !session.playerId.equals(Json.text(message, "player_id"))) {
@@ -290,7 +299,24 @@ public final class ArenaServer implements AutoCloseable {
 
     private void disconnect(Peer peer) {
         Session session = sessions.remove(peer);
-        if (session != null && session.playerId != null && room != null) room.left(session.playerId);
+        if (session != null) {
+            session.snapshots.close();
+            if (session.playerId != null && room != null) room.left(session.playerId);
+        }
+    }
+
+    private void startReplication() {
+        fixedClock = new FixedTickClock(monotonicNanos);
+        publishSnapshots(true);
+    }
+
+    private void publishSnapshots(boolean forceFull) {
+        for (var entry : sessions.entrySet()) {
+            Session session = entry.getValue();
+            if (session.playerId == null || !room.player(session.playerId).connected) continue;
+            entry.getKey().send(session.snapshots.next(room, forceFull));
+            snapshotRetentionHighWater = Math.max(snapshotRetentionHighWater, session.snapshots.highWater());
+        }
     }
 
     private void broadcast(ObjectNode message) {
@@ -304,8 +330,9 @@ public final class ArenaServer implements AutoCloseable {
                 if (rejection.playerId().equals(session.playerId)) peer.error(rejection.code());
             });
         }
+        if (room.executedTicks() % 2 == 0 || room.status() == Room.Status.FINISHED)
+            publishSnapshots(room.status() == Room.Status.FINISHED);
         if (room.status() == Room.Status.FINISHED) {
-            broadcast(room.view("SNAPSHOT"));
             broadcast(room.view("ROOM_FINISHED"));
         }
     }
@@ -344,6 +371,8 @@ public final class ArenaServer implements AutoCloseable {
             ObjectNode result = Json.MAPPER.createObjectNode().put("manual_time_ns", manualNanos)
                     .put("pending_input_high_water", room == null ? 0 : room.inputHighWater())
                     .put("replay_bytes", room == null ? 0 : room.replayStoredBytes())
+                    .put("retained_snapshots", sessions.values().stream().mapToInt(s -> s.snapshots.retainedCount()).sum())
+                    .put("snapshot_retention_high_water", snapshotRetentionHighWater)
                     .put("mailbox_high_water", mailboxHighWater.get()).put("outbound_high_water", outboundHighWater.get());
             result.set("clock", fixedClock == null ? closedClockMetrics.deepCopy() : fixedClock.view());
             result.set("parser", parserMetrics.view());
@@ -369,6 +398,7 @@ public final class ArenaServer implements AutoCloseable {
                 .put("event_loops_terminated", acceptLoop.isTerminated() && ioLoop.isTerminated())
                 .put("pending_input_high_water", closedInputHighWater)
                 .put("replay_bytes", closedReplayBytes)
+                .put("retained_snapshots", closedSnapshotCount).put("snapshot_retention_high_water", snapshotRetentionHighWater)
                 .put("mailbox_high_water", mailboxHighWater.get()).put("outbound_high_water", outboundHighWater.get());
         result.set("parser", parserMetrics.view());
         result.set("clock", closedClockMetrics.deepCopy());
@@ -391,6 +421,8 @@ public final class ArenaServer implements AutoCloseable {
         // Drain channel callbacks before final owner cleanup. No network thread waits for this owner.
         ioLoop.submit(() -> { }).syncUninterruptibly();
         call(() -> {
+            for (Session session : sessions.values()) session.snapshots.close();
+            closedSnapshotCount = sessions.values().stream().mapToInt(s -> s.snapshots.retainedCount()).sum();
             sessions.clear();
             if (room != null) {
                 room.close();
diff --git a/src/main/java/arena/AuthorityScenario.java b/src/main/java/arena/AuthorityScenario.java
index 6ba01a1..078b75e 100644
--- a/src/main/java/arena/AuthorityScenario.java
+++ b/src/main/java/arena/AuthorityScenario.java
@@ -3,7 +3,6 @@ package arena;
 import com.fasterxml.jackson.databind.JsonNode;
 import com.fasterxml.jackson.databind.node.ArrayNode;
 import com.fasterxml.jackson.databind.node.ObjectNode;
-import java.io.DataInputStream;
 import java.io.IOException;
 import java.lang.reflect.Field;
 import java.math.BigInteger;
@@ -89,6 +88,7 @@ final class AuthorityScenario {
                     tags.addObject().put("tick", afterTick.path("tick").asInt()).put("actor", roles.get(tag.getKey()))
                             .put("target", roles.getOrDefault(tag.getValue(), tag.getValue())).put("result", outcome);
                 }
+                if ((tick + 1) % 2 == 0) { alpha.until("SNAPSHOT"); bravo.until("SNAPSHOT"); }
                 int movedX = Math.min(tick + 1, 100) * 400, movedY = Math.clamp(tick - 99, 0, 100) * 400;
                 JsonNode a = player(afterTick, alpha.playerId), b = player(afterTick, bravo.playerId);
                 if (a.path("x").asInt() != 10000 + movedX || a.path("y").asInt() != 10000 + movedY
@@ -294,9 +294,7 @@ final class AuthorityScenario {
         Field field = object.getClass().getDeclaredField(name); field.setAccessible(true); return field.get(object);
     }
     private static ObjectNode response(TcpClient client) throws Exception {
-        DataInputStream input = (DataInputStream) field(client, "input"); int length = input.readInt();
-        if (length < 1 || length > CompleteFrame.MAX_BYTES) throw new IOException("server frame bound");
-        byte[] bytes = new byte[length]; input.readFully(bytes); return Json.read(bytes);
+        return client.control();
     }
     private static JsonNode player(ObjectNode state, String id) throws IOException {
         for (JsonNode p : state.withArray("players")) if (p.path("player_id").asText().equals(id)) return p;
diff --git a/src/main/java/arena/CompleteFrame.java b/src/main/java/arena/CompleteFrame.java
index 1a14981..c10386a 100644
--- a/src/main/java/arena/CompleteFrame.java
+++ b/src/main/java/arena/CompleteFrame.java
@@ -199,6 +199,12 @@ final class CompleteFrame extends SimpleChannelInboundHandler<ByteBuf> {
                     Json.text(message, "session_id");
                     Json.text(message, "room_id");
                 }
+                case "SNAPSHOT_ACK" -> {
+                    Json.text(message, "session_id");
+                    Json.text(message, "room_id");
+                    Json.text(message, "player_id");
+                    Json.snapshotSequence(message);
+                }
                 case "INPUT" -> {
                     Json.text(message, "session_id");
                     Json.text(message, "room_id");
diff --git a/src/main/java/arena/IdentityScenario.java b/src/main/java/arena/IdentityScenario.java
index 577c0fe..4540888 100644
--- a/src/main/java/arena/IdentityScenario.java
+++ b/src/main/java/arena/IdentityScenario.java
@@ -121,7 +121,9 @@ final class IdentityScenario {
         try {
             world.setup(scenario.path(state.toLowerCase(java.util.Locale.ROOT) + "_player_count").asInt());
             if (state.equals("FINISHED")) {
-                world.server.advanceTicks(scenario.path("finish_ticks").asInt());
+                for (int tick = 0; tick < scenario.path("finish_ticks").asInt(); tick += 2) {
+                    world.server.advanceTicks(2); world.alpha.until("SNAPSHOT"); world.bravo.until("SNAPSHOT");
+                }
                 out.put("alpha_finished_message", world.alpha.until("ROOM_FINISHED").path("status").asText());
                 out.put("bravo_finished_message", world.bravo.until("ROOM_FINISHED").path("status").asText());
             }
diff --git a/src/main/java/arena/Json.java b/src/main/java/arena/Json.java
index e82f208..5f8ee49 100644
--- a/src/main/java/arena/Json.java
+++ b/src/main/java/arena/Json.java
@@ -61,4 +61,11 @@ final class Json {
         if (value == null || !value.isIntegralNumber()) throw new IllegalArgumentException("target_tick must be an integer");
         return value.bigIntegerValue();
     }
+
+    static long snapshotSequence(ObjectNode object) {
+        JsonNode value = object.get("snapshot_seq");
+        if (value == null || !value.isIntegralNumber() || !value.canConvertToLong() || value.longValue() <= 0)
+            throw new IllegalArgumentException("snapshot_seq must be a positive integer");
+        return value.longValue();
+    }
 }
diff --git a/src/main/java/arena/ScenarioRunner.java b/src/main/java/arena/ScenarioRunner.java
index 359dd29..ff084d5 100644
--- a/src/main/java/arena/ScenarioRunner.java
+++ b/src/main/java/arena/ScenarioRunner.java
@@ -82,12 +82,13 @@ final class ScenarioRunner {
                     default -> throw new IOException("unsupported setup step");
                 }
             }
+            for (TcpClient client : clients.values()) client.until("SNAPSHOT");
             int currentTick = 0;
             int accepted = 0;
             for (JsonNode step : scenario.withArray("inputs")) {
                 int before = step.path("before_tick").asInt(-1);
                 if (before < currentTick || before >= Room.DURATION) throw new IOException("unordered scenario input");
-                server.advanceTicks(before - currentTick);
+                advanceAndDrain(server, clients, currentTick, before);
                 currentTick = before;
                 TcpClient actor = requiredClient(clients, step.path("client").asText());
                 String target = step.path("tag_target").isNull() ? null
@@ -96,7 +97,7 @@ final class ScenarioRunner {
                 actor.intent(roomId, step.path("direction").asText(), target, before);
                 accepted++;
             }
-            server.advanceTicks(scenario.path("ticks").asInt() - currentTick);
+            advanceAndDrain(server, clients, currentTick, scenario.path("ticks").asInt());
             for (TcpClient client : clients.values()) {
                 ObjectNode observed = client.until("ROOM_FINISHED");
                 if (!observed.path("status").asText().equals("FINISHED")
@@ -154,6 +155,8 @@ final class ScenarioRunner {
         if (cleanup.path("mailbox_high_water").asInt() > ArenaServer.MAILBOX_LIMIT) failures.add("mailbox bound");
         if (cleanup.path("outbound_high_water").asInt() > ArenaServer.OUTBOUND_LIMIT) failures.add("outbound bound");
         if (cleanup.path("replay_bytes").asInt(-1) != 0) failures.add("replay storage cleanup");
+        if (cleanup.path("retained_snapshots").asInt(-1) != 0) failures.add("snapshot retention cleanup");
+        if (cleanup.path("snapshot_retention_high_water").asInt() > SnapshotStream.RETENTION) failures.add("snapshot retention bound");
         JsonNode clock = cleanup.path("clock");
         if (clock.path("active").asBoolean(true) || clock.path("accumulator_ns").asLong(-1) != 0
                 || clock.path("max_ticks_per_iteration").asInt(-1) < 0
@@ -168,6 +171,13 @@ final class ScenarioRunner {
         if (!failures.isEmpty()) throw new IOException("cleanup failure: " + failures);
     }
 
+    private static void advanceAndDrain(ArenaServer server, Map<String, TcpClient> clients, int from, int to) throws IOException {
+        for (int tick = from; tick < to;) {
+            int count = Math.min(to - tick, 2 - tick % 2); server.advanceTicks(count); tick += count;
+            if (tick % 2 == 0) for (TcpClient client : clients.values()) client.until("SNAPSHOT");
+        }
+    }
+
     private static TcpClient requiredClient(Map<String, TcpClient> clients, String role) throws IOException {
         TcpClient client = clients.get(role);
         if (client == null) throw new IOException("unknown client role");
diff --git a/src/main/java/arena/SequenceScenario.java b/src/main/java/arena/SequenceScenario.java
index e72523f..0b37b67 100644
--- a/src/main/java/arena/SequenceScenario.java
+++ b/src/main/java/arena/SequenceScenario.java
@@ -5,7 +5,6 @@ import com.fasterxml.jackson.databind.node.ArrayNode;
 import com.fasterxml.jackson.databind.node.ObjectNode;
 import io.netty.buffer.ByteBuf;
 import io.netty.channel.embedded.EmbeddedChannel;
-import java.io.DataInputStream;
 import java.io.IOException;
 import java.lang.reflect.Field;
 import java.util.ArrayList;
@@ -60,6 +59,7 @@ final class SequenceScenario {
                     } else failures.add("unexpected input response");
                 }
                 server.advanceTicks(1);
+                if ((tick + 1) % 2 == 0) { alpha.until("SNAPSHOT"); bravo.until("SNAPSHOT"); }
                 ObjectNode state = snapshot(server); ticks.add(state);
                 JsonNode actor = player(state, alpha.playerId), other = player(state, bravo.playerId);
                 applied.add(actor.path("applied_seq").isMissingNode() ? Json.MAPPER.nullNode() : actor.path("applied_seq"));
@@ -181,9 +181,7 @@ final class SequenceScenario {
     }
 
     private static ObjectNode response(TcpClient client) throws Exception {
-        DataInputStream input = (DataInputStream) field(client, "input");
-        int length = input.readInt(); if (length < 1 || length > CompleteFrame.MAX_BYTES) throw new IOException("server frame bound");
-        byte[] bytes = new byte[length]; input.readFully(bytes); return Json.read(bytes);
+        return client.control();
     }
     private static ObjectNode snapshot(ArenaServer server) throws Exception {
         return ((ThreadPoolExecutor) field(server, "owner")).submit(() -> {
diff --git a/src/main/java/arena/SnapshotStream.java b/src/main/java/arena/SnapshotStream.java
new file mode 100644
index 0000000..17ccd57
--- /dev/null
+++ b/src/main/java/arena/SnapshotStream.java
@@ -0,0 +1,60 @@
+package arena;
+
+import com.fasterxml.jackson.databind.JsonNode;
+import com.fasterxml.jackson.databind.node.ArrayNode;
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import java.util.List;
+import java.util.TreeMap;
+
+/** One client stream, confined to the same Room owner as its session. */
+final class SnapshotStream implements AutoCloseable {
+    static final int RETENTION = 32;
+    private static final List<String> VISIBLE_FIELDS = List.of("player_id", "slot", "x", "y", "direction", "score", "connectivity");
+    private final Thread owner = Thread.currentThread();
+    private final TreeMap<Long, ArrayNode> retained = new TreeMap<>();
+    private long sequence;
+    private long acknowledged;
+    private int highWater;
+    private boolean closed;
+
+    ObjectNode next(Room room, boolean forceFull) {
+        assertOwner(); room.assertOwner();
+        if (closed) throw new IllegalStateException("snapshot stream closed");
+        ArrayNode current = Json.MAPPER.createArrayNode();
+        for (JsonNode player : room.view("SNAPSHOT").withArray("players")) {
+            if (player.path("connectivity").asText().equals("LEFT")) continue;
+            ObjectNode visible = current.addObject(); for (String field : VISIBLE_FIELDS) visible.set(field, player.path(field));
+        }
+        ArrayNode base = retained.get(acknowledged); sequence++;
+        boolean full = forceFull || sequence % 20 == 0 || base == null;
+        ObjectNode message = Json.message("SNAPSHOT").put("snapshot_seq", sequence).put("room_id", room.id)
+                .put("tick", room.executedTicks() - 1).put("owner_epoch", 0).put("kind", full ? "FULL" : "DELTA")
+                .put("state_hash", ReplayLog.hash(room.canonicalRecord()));
+        if (full) message.putNull("base_snapshot_seq").put("status", room.status().name());
+        else message.put("base_snapshot_seq", acknowledged);
+        ArrayNode changed = message.putArray("players"), removed = message.putArray("removed_player_ids");
+        TreeMap<String, JsonNode> previous = new TreeMap<>();
+        if (!full) for (JsonNode player : base) previous.put(player.path("player_id").asText(), player);
+        for (JsonNode player : current) {
+            JsonNode old = previous.remove(player.path("player_id").asText());
+            if (!player.equals(old)) changed.add(player.deepCopy());
+        }
+        previous.keySet().forEach(removed::add);
+        if (retained.size() == RETENTION) retained.pollFirstEntry();
+        retained.put(sequence, current); highWater = Math.max(highWater, retained.size());
+        return message;
+    }
+
+    void acknowledge(long seq) {
+        assertOwner();
+        if (closed) throw new IllegalStateException("snapshot stream closed");
+        // An unknown base only affects the next scheduled publication, never the simulation.
+        acknowledged = seq;
+    }
+    int retainedCount() { assertOwner(); return retained.size(); }
+    int highWater() { assertOwner(); return highWater; }
+    @Override public void close() { assertOwner(); retained.clear(); acknowledged = 0; closed = true; }
+    private void assertOwner() {
+        if (Thread.currentThread() != owner) throw new IllegalStateException("snapshot access outside owner");
+    }
+}
diff --git a/src/main/java/arena/TcpClient.java b/src/main/java/arena/TcpClient.java
index e133433..cf331d0 100644
--- a/src/main/java/arena/TcpClient.java
+++ b/src/main/java/arena/TcpClient.java
@@ -38,19 +38,34 @@ final class TcpClient implements AutoCloseable {
 
     ObjectNode until(String type) throws IOException {
         for (int messages = 0; messages < 64; messages++) {
-            int length = input.readInt();
-            if (length < 1 || length > CompleteFrame.MAX_BYTES) throw new IOException("bad server length");
-            byte[] payload = new byte[length];
-            input.readFully(payload);
-            ObjectNode message = Json.read(payload);
-            String status = message.path("status").asText();
-            if (List.of("LOBBY", "RUNNING", "FINISHED").contains(status)) observe(status);
+            ObjectNode message = readFrame();
+            if (message == null) throw new IOException("unexpected server EOF");
             if (message.path("type").asText().equals("ERROR")) throw new IOException("server error: " + message);
             if (message.path("type").asText().equals(type)) return message;
         }
         throw new IOException("response message ceiling exceeded");
     }
 
+    /** Added replication frames do not replace the original INPUT_ACK/ERROR observations. */
+    ObjectNode control() throws IOException {
+        for (int messages = 0; messages < 64; messages++) {
+            ObjectNode message = readFrame();
+            if (message == null) throw new IOException("unexpected server EOF");
+            if (!message.path("type").asText().equals("SNAPSHOT")) return message;
+        }
+        throw new IOException("response message ceiling exceeded");
+    }
+
+    private ObjectNode readFrame() throws IOException {
+        int first = input.read(); if (first == -1) return null;
+        int length = first << 24 | input.readUnsignedByte() << 16 | input.readUnsignedByte() << 8 | input.readUnsignedByte();
+        if (length < 1 || length > CompleteFrame.MAX_BYTES) throw new IOException("bad server length");
+        byte[] payload = new byte[length]; input.readFully(payload); ObjectNode message = Json.read(payload);
+        String status = message.path("status").asText();
+        if (List.of("LOBBY", "RUNNING", "FINISHED").contains(status)) observe(status);
+        return message;
+    }
+
     void hello() throws IOException {
         send(Json.message("HELLO"));
         sessionId = Json.text(until("WELCOME"), "session_id");
@@ -84,8 +99,12 @@ final class TcpClient implements AutoCloseable {
     }
 
     void expectClosed() throws IOException {
-        if (input.read() != -1) throw new IOException("expected EOF after graceful server close");
-        observe("CLOSED");
+        for (int messages = 0; messages < 64; messages++) {
+            ObjectNode message = readFrame();
+            if (message == null) { observe("CLOSED"); return; }
+            if (!message.path("type").asText().equals("SNAPSHOT")) throw new IOException("expected EOF after graceful server close: " + message);
+        }
+        throw new IOException("close message ceiling exceeded");
     }
 
     List<String> lifecycle() { return List.copyOf(lifecycle); }
diff --git a/src/test/java/arena/ReplayFixture.java b/src/test/java/arena/ReplayFixture.java
index 65eff70..9499baf 100644
--- a/src/test/java/arena/ReplayFixture.java
+++ b/src/test/java/arena/ReplayFixture.java
@@ -4,12 +4,12 @@ import com.fasterxml.jackson.databind.JsonNode;
 import com.fasterxml.jackson.databind.node.ObjectNode;
 import java.io.IOException;
 import java.lang.reflect.Field;
+import java.lang.reflect.Method;
 import java.util.Map;
 import java.util.TreeMap;
 import java.util.concurrent.Callable;
 import java.util.concurrent.ThreadPoolExecutor;
 import java.util.concurrent.TimeUnit;
-import java.util.function.LongSupplier;
 
 /** Test runtime only. Initial state setup is never represented as successful public joins. */
 final class ReplayFixture {
@@ -31,7 +31,7 @@ final class ReplayFixture {
                 if (!found) throw new IOException("real session missing");
             }
             start(room, initial);
-            set(server, "fixedClock", new FixedTickClock((LongSupplier) field(server, "monotonicNanos")));
+            Method start = ArenaServer.class.getDeclaredMethod("startReplication"); start.setAccessible(true); start.invoke(server);
             ObjectNode observed = Json.MAPPER.createObjectNode().put("kind", "test-only owner initial state; not public joins")
                     .put("bound_live_sessions", bound).put("normal_start_evaluations", 1);
             observed.set("state", view(room)); return observed;
diff --git a/src/test/java/arena/ReplayFormatTest.java b/src/test/java/arena/ReplayFormatTest.java
index 703ed94..a162475 100644
--- a/src/test/java/arena/ReplayFormatTest.java
+++ b/src/test/java/arena/ReplayFormatTest.java
@@ -47,7 +47,8 @@ final class ReplayFormatTest {
         try (JarFile jar = new JarFile(path.toFile());
              URLClassLoader production = new URLClassLoader(new URL[] {path.toUri().toURL()}, ClassLoader.getPlatformClassLoader())) {
             assertNotNull(jar.getJarEntry("arena/ArenaServer.class"));
-            for (String name : List.of("ReplayFixture", "ReplayScenario", "ReplayVerifier", "ReplayTool", "G07BaselineTest")) {
+            assertNull(jar.getJarEntry("G08.json"));
+            for (String name : List.of("ReplayFixture", "ReplayScenario", "ReplayVerifier", "ReplayTool", "G07BaselineTest", "SnapshotScenario", "G08BaselineTest")) {
                 assertNull(jar.getJarEntry("arena/" + name + ".class"));
                 assertThrows(ClassNotFoundException.class, () -> Class.forName("arena." + name, false, production));
             }
diff --git a/src/test/java/arena/ReplayScenario.java b/src/test/java/arena/ReplayScenario.java
index e1d14c6..376e4b3 100644
--- a/src/test/java/arena/ReplayScenario.java
+++ b/src/test/java/arena/ReplayScenario.java
@@ -3,7 +3,6 @@ package arena;
 import com.fasterxml.jackson.databind.JsonNode;
 import com.fasterxml.jackson.databind.node.ArrayNode;
 import com.fasterxml.jackson.databind.node.ObjectNode;
-import java.io.DataInputStream;
 import java.io.IOException;
 import java.nio.file.Files;
 import java.nio.file.Path;
@@ -38,6 +37,7 @@ final class ReplayScenario {
                 client.hello(); client.playerId = p.path("player_id").asText();
             }
             result.set("bootstrap", ReplayFixture.bootstrap(server, scenario, clients));
+            for (TcpClient client : clients.values()) client.until("SNAPSHOT");
             int eventIndex = 0, removed = 0;
             for (int tick = 0; tick < scenario.path("ticks").asInt(); tick++) {
                 Map<String, String> acceptedTags = new LinkedHashMap<>();
@@ -87,6 +87,7 @@ final class ReplayScenario {
                     tagEvents.addObject().put("tick", after.path("tick").asInt()).put("actor", actor.playerId)
                             .put("target", clients.get(tag.getValue()).playerId).put("result", outcome);
                 }
+                if ((tick + 1) % 2 == 0) for (TcpClient client : clients.values()) if (!client.isClosed()) client.until("SNAPSHOT");
             }
             ObjectNode state = ReplayFixture.snapshot(server); result.set("final_state", state);
             result.put("executed_ticks", state.path("executed_ticks").asInt()).put("removed_rejected_inputs", removed);
@@ -127,9 +128,7 @@ final class ReplayScenario {
                 || !state.path("status").asText().equals("RUNNING")) failures.add("final clock/lifecycle");
     }
     static ObjectNode response(TcpClient client) throws Exception {
-        DataInputStream input = (DataInputStream) ReplayFixture.field(client, "input"); int length = input.readInt();
-        if (length < 1 || length > CompleteFrame.MAX_BYTES) throw new IOException("server frame bound");
-        byte[] bytes = new byte[length]; input.readFully(bytes); return Json.read(bytes);
+        return client.control();
     }
     static JsonNode player(ObjectNode state, String id) throws IOException {
         for (JsonNode p : state.withArray("players")) if (p.path("player_id").asText().equals(id)) return p;
diff --git a/src/test/java/arena/ReplayTool.java b/src/test/java/arena/ReplayTool.java
index cb0731b..27222c0 100644
--- a/src/test/java/arena/ReplayTool.java
+++ b/src/test/java/arena/ReplayTool.java
@@ -19,10 +19,12 @@ public final class ReplayTool {
         if (args[0].equals("scenario-run")) {
             if (Files.size(input) > 65_536) throw new IllegalArgumentException("scenario byte bound");
             ObjectNode scenario = Json.read(Files.readAllBytes(input));
-            if (scenario.path("thread").asText().equals("G07")) {
+            String thread = scenario.path("thread").asText();
+            if (thread.equals("G07") || thread.equals("G08")) {
                 boolean variant = args.length == 5 && args[3].equals("--variant") && args[4].equals("rejected-removed");
-                if (args.length != 3 && !variant) throw new IllegalArgumentException("unknown G07 variant");
-                ReplayScenario.Observed observed = ReplayScenario.run(input, variant); result = observed.result();
+                if (args.length != 3 && !(thread.equals("G07") && variant)) throw new IllegalArgumentException("unknown scenario variant");
+                ReplayScenario.Observed observed = thread.equals("G07") ? ReplayScenario.run(input, variant) : SnapshotScenario.run(input);
+                result = observed.result();
                 Path artifact = output.resolveSibling(output.getFileName().toString().replaceFirst("\\.json$", "") + ".replay.jsonl");
                 if (observed.replay() != null) {
                     Files.write(artifact, observed.replay(), StandardOpenOption.CREATE_NEW);
diff --git a/src/test/java/arena/ServerIntegrationTest.java b/src/test/java/arena/ServerIntegrationTest.java
index 6afad9a..d663097 100644
--- a/src/test/java/arena/ServerIntegrationTest.java
+++ b/src/test/java/arena/ServerIntegrationTest.java
@@ -32,7 +32,9 @@ final class ServerIntegrationTest {
             alpha.send(attempt);
             alpha.until("INPUT_ACK");
             bravo.intent(room, "WEST", null, 0);
-            server.advanceTicks(1_200);
+            for (int tick = 0; tick < 1_200; tick += 2) {
+                server.advanceTicks(2); alpha.until("SNAPSHOT"); bravo.until("SNAPSHOT");
+            }
             ObjectNode first = alpha.until("ROOM_FINISHED");
             assertEquals(first, bravo.until("ROOM_FINISHED"));
             assertEquals(1_199, first.path("tick").asInt());
diff --git a/src/test/java/arena/SnapshotScenario.java b/src/test/java/arena/SnapshotScenario.java
new file mode 100644
index 0000000..bf8357c
--- /dev/null
+++ b/src/test/java/arena/SnapshotScenario.java
@@ -0,0 +1,242 @@
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
+import java.util.List;
+import java.util.Map;
+import java.util.TreeMap;
+
+/** One fixed 196-tick TCP run. HELLO markers only drain/observe the existing session stream. */
+final class SnapshotScenario {
+    static final String SHA256 = "d121973167461dddbb3ba0bd339ed26b09486cda8bbec642328ccc5cea9e578e";
+    static final List<String> VISIBLE_FIELDS = List.of("player_id", "slot", "x", "y", "direction", "score", "connectivity");
+    private SnapshotScenario() { }
+
+    static ReplayScenario.Observed run(Path path) throws Exception {
+        byte[] bytes = Files.readAllBytes(path);
+        if (!SHA256.equals(hash(bytes))) throw new IOException("G08 requires frozen scenario bytes");
+        ObjectNode scenario = Json.read(bytes), result = Json.MAPPER.createObjectNode().put("thread", "G08")
+                .put("contract_version", 1).put("scenario_sha256", SHA256).put("process_id", ProcessHandle.current().pid())
+                .put("network_fault_runs", 0).put("load_runs", 0);
+        ArrayNode failures = result.putArray("failures"), captures = result.putArray("wire_captures");
+        ArrayNode boundaries = result.putArray("boundaries"), markers = result.putArray("observation_markers");
+        ArrayNode events = result.putArray("events"), states = result.putArray("ticks");
+        ArrayNode hashes = result.putArray("state_hashes"), records = result.putArray("canonical_records");
+        Map<String, TcpClient> clients = new LinkedHashMap<>(); Map<String, Replica> replicas = new LinkedHashMap<>();
+        byte[] artifact = null; ArenaServer server = new ArenaServer("127.0.0.1", 0, true);
+        try {
+            for (JsonNode player : scenario.withArray("players")) {
+                String role = player.path("client").asText(); TcpClient client = new TcpClient(server.port());
+                clients.put(role, client); replicas.put(role, new Replica()); client.hello(); client.playerId = player.path("player_id").asText();
+            }
+            ObjectNode bootstrap = ReplayFixture.bootstrap(server, scenario, clients); result.set("bootstrap", bootstrap);
+            bootstrap.put("publication_bridge", "test-only invocation of the production startReplication helper used by normal JOIN");
+            captureBoundary(server, scenario, clients, replicas, result, -1);
+            int eventIndex = 0;
+            for (int tick = 0; tick < scenario.path("ticks").asInt(); tick++) {
+                while (eventIndex < scenario.withArray("events").size()
+                        && scenario.withArray("events").get(eventIndex).path("before_tick").asInt() == tick) {
+                    JsonNode event = scenario.withArray("events").get(eventIndex++);
+                    TcpClient client = clients.get(event.path("client").asText());
+                    ObjectNode request = client.auth(event.path("kind").asText(), scenario.path("room_id").asText());
+                    ObjectNode observed = events.addObject().put("before_tick", tick); observed.set("before", ReplayFixture.snapshot(server));
+                    if (event.path("kind").asText().equals("INPUT")) {
+                        for (String field : List.of("seq", "target_tick", "direction", "owner_epoch")) request.set(field, event.path(field));
+                        if (event.path("tag_target_role").isNull()) request.putNull("tag_target_player_id");
+                        else request.put("tag_target_player_id", clients.get(event.path("tag_target_role").asText()).playerId);
+                        client.send(request); ObjectNode response = read(client); observed.set("response", response);
+                        if (!response.path("type").asText().equals("INPUT_ACK") || !response.path("status").asText().equals("ACCEPTED"))
+                            failures.add("fixed INPUT not accepted at " + tick);
+                    } else {
+                        client.send(request); client.expectClosed(); client.close();
+                    }
+                    observed.set("request", request); observed.set("after", ReplayFixture.snapshot(server));
+                }
+                server.advanceTicks(1); ObjectNode state = ReplayFixture.snapshot(server); states.add(state);
+                hashes.add(state.path("state_hash")); records.add(ReplayFixture.canonicalRecord(server));
+                checkAuthority(state, tick, failures); captureBoundary(server, scenario, clients, replicas, result, tick);
+            }
+            result.put("executed_ticks", states.size()); result.set("final_state", ReplayFixture.snapshot(server));
+            ObjectNode counts = result.putObject("snapshot_counts");
+            for (var entry : replicas.entrySet()) {
+                counts.put(entry.getKey(), entry.getValue().count);
+                int expected = entry.getKey().equals("delta") ? 98 : scenario.path("expected_snapshot_count_for_remaining_clients").asInt();
+                if (entry.getValue().count != expected) failures.add("snapshot count " + entry.getKey() + ": " + entry.getValue().count);
+            }
+            long wrongCadence = 0;
+            for (JsonNode boundary : boundaries) for (JsonNode count : boundary.path("counts"))
+                if (count.asInt() != boundary.path("expected_each").asInt()) wrongCadence++;
+            result.put("cadence_mismatches", wrongCadence);
+            if (wrongCadence != 0) failures.add("observed publication cadence mismatches: " + wrongCadence);
+            result.put("captured_snapshots", captures.size()).put("observation_marker_count", markers.size());
+            artifact = server.replayArtifact(); result.put("artifact_bytes", artifact.length);
+            result.set("runtime_metrics", server.metrics()); result.set("final_retention", retention(server));
+            server.close(); for (TcpClient client : clients.values()) if (!client.isClosed()) client.expectClosed();
+        } catch (Exception failure) {
+            failures.add(failure.getClass().getName() + ": " + failure.getMessage());
+            java.io.StringWriter trace = new java.io.StringWriter(); failure.printStackTrace(new java.io.PrintWriter(trace));
+            result.put("execution_error", trace.toString()).put("executed_ticks", states.size());
+            ObjectNode counts = result.putObject("snapshot_counts"); replicas.forEach((role, replica) -> counts.put(role, replica.count));
+        } finally {
+            server.close(); for (TcpClient client : clients.values()) client.close();
+        }
+        ScenarioRunner.assertCleanup(server.cleanup()); result.set("cleanup", server.cleanup());
+        result.put("all_resources_released", clients.values().stream().allMatch(TcpClient::isClosed));
+        result.put("passed", failures.isEmpty()); return new ReplayScenario.Observed(result, artifact);
+    }
+
+    private static void captureBoundary(ArenaServer server, ObjectNode scenario, Map<String, TcpClient> clients,
+                                        Map<String, Replica> replicas, ObjectNode result, int tick) throws Exception {
+        ArrayNode failures = result.withArray("failures");
+        ObjectNode boundary = result.withArray("boundaries").addObject().put("tick", tick)
+                .put("expected_each", tick == -1 || (tick + 1) % 2 == 0 ? 1 : 0);
+        ObjectNode counts = boundary.putObject("counts");
+        ObjectNode authority = ReplayFixture.snapshot(server); String canonical = ReplayFixture.canonicalRecord(server);
+        for (var entry : clients.entrySet()) {
+            String role = entry.getKey(); TcpClient client = entry.getValue(); if (client.isClosed()) continue;
+            ArrayNode messages = marker(server, client, result, role, tick, "publication drain"); int count = 0;
+            for (JsonNode value : messages) {
+                if (!value.path("type").asText().equals("SNAPSHOT")) { failures.add("unexpected wire message: " + value); continue; }
+                count++; ObjectNode wire = (ObjectNode) value;
+                ObjectNode capture = result.withArray("wire_captures").addObject().put("client", role).put("observed_tick", tick);
+                capture.set("snapshot", wire); capture.set("authoritative_state", authority); capture.put("canonical_record", canonical)
+                        .put("canonical_hash", hash(canonical.getBytes(java.nio.charset.StandardCharsets.UTF_8)));
+                capture.set("retention", retention(server)); Replica replica = replicas.get(role); replica.count++;
+                if (!wire.path("snapshot_seq").isIntegralNumber() || !wire.path("kind").isTextual()) {
+                    capture.put("applied", false).put("ack_sent", false).put("reason", "actual old snapshot lacks sequence/kind; no fabricated ACK");
+                    failures.add("snapshot sequence/kind unavailable for " + role); continue;
+                }
+                replica.apply(wire); capture.set("client_visible_state", replica.state()); capture.put("applied", true);
+                if (!replica.state().equals(projection(authority))) failures.add("wire application differs from authoritative projection " + role + "/" + tick);
+                if (!wire.path("state_hash").asText().equals(capture.path("canonical_hash").asText())) failures.add("canonical metadata hash " + role + "/" + tick);
+                if (wire.path("tick").asInt(-2) != tick || wire.path("owner_epoch").asInt(-1) != 0
+                        || !wire.path("room_id").equals(scenario.path("room_id"))) failures.add("snapshot metadata " + role + "/" + tick);
+                long seq = wire.path("snapshot_seq").asLong(); boolean full = seq == 1 || seq == 4 || seq % 20 == 0;
+                if (!wire.path("kind").asText().equals(full ? "FULL" : "DELTA")
+                        || full && !wire.path("base_snapshot_seq").isNull()
+                        || !full && wire.path("base_snapshot_seq").asLong() != seq - 1) failures.add("base/kind " + role + "/" + seq);
+                ObjectNode ack = client.auth("SNAPSHOT_ACK", scenario.path("room_id").asText()).put("snapshot_seq", ackFor(scenario, seq));
+                client.send(ack); capture.set("ack", ack); capture.put("ack_sent", true);
+                ArrayNode feedback = marker(server, client, result, role, tick, "ACK observation drain"); capture.set("ack_feedback", feedback);
+                if (!feedback.isEmpty()) failures.add("ACK produced error or unscheduled publication " + role + "/" + seq);
+            }
+            counts.put(role, count);
+        }
+    }
+
+    private static ArrayNode marker(ArenaServer server, TcpClient client, ObjectNode result, String role, int tick, String phase) throws Exception {
+        ObjectNode marker = result.withArray("observation_markers").addObject().put("client", role).put("tick", tick).put("phase", phase);
+        ArrayNode messages = marker.putArray("wire_messages"); client.send(Json.message("HELLO"));
+        for (int i = 0; i < 64; i++) {
+            ObjectNode response = read(client);
+            if (response.path("type").asText().equals("WELCOME")) {
+                if (!client.sessionId.equals(response.path("session_id").asText())) throw new IOException("observation marker changed session");
+                marker.put("unchanged_session_id", client.sessionId).put("unchanged_player_id", client.playerId);
+                int observed = ReplayFixture.snapshot(server).path("tick").asInt(); marker.put("observed_tick", observed);
+                if (observed != tick) throw new IOException("observation marker advanced simulation");
+                return messages;
+            }
+            messages.add(response);
+        }
+        throw new IOException("observation marker frame ceiling");
+    }
+
+    private static long ackFor(ObjectNode scenario, long seq) throws IOException {
+        for (JsonNode schedule : scenario.withArray("ack_schedule")) {
+            if (schedule.path("after_snapshot_seq").asLong(-1) == seq) return schedule.path("send_ack").asLong();
+            JsonNode range = schedule.path("after_snapshot_seq_range");
+            if (range.isArray() && seq >= range.get(0).asLong() && seq <= range.get(1).asLong()) return seq;
+        }
+        throw new IOException("ACK outside frozen schedule");
+    }
+
+    static ArrayNode projection(ObjectNode authority) {
+        ArrayNode result = Json.MAPPER.createArrayNode();
+        for (JsonNode player : authority.withArray("players")) if (!player.path("connectivity").asText().equals("LEFT")) {
+            ObjectNode visible = result.addObject(); for (String field : VISIBLE_FIELDS) visible.set(field, player.path(field));
+        }
+        return result;
+    }
+
+    private static ObjectNode retention(ArenaServer server) throws Exception {
+        return ReplayFixture.owned(server, () -> {
+            ObjectNode result = Json.MAPPER.createObjectNode();
+            for (Object session : ((Map<?, ?>) ReplayFixture.field(server, "sessions")).values()) {
+                String player = (String) ReplayFixture.field(session, "playerId"); if (player == null) continue;
+                ObjectNode state = result.putObject(player);
+                try {
+                    Object snapshots = ReplayFixture.field(session, "snapshots");
+                    Map<?, ?> retained = (Map<?, ?>) ReplayFixture.field(snapshots, "retained");
+                    ArrayNode ids = state.putArray("retained_base_ids"); for (Object id : retained.keySet()) ids.add(((Number) id).longValue());
+                    state.put("count", retained.size()).put("high_water", (int) ReplayFixture.field(snapshots, "highWater"));
+                    state.put("acknowledged_seq", (long) ReplayFixture.field(snapshots, "acknowledged"));
+                    if (retained.size() > 32) throw new IOException("server retention bound");
+                } catch (NoSuchFieldException unavailableBeforeG08) { state.put("available", false); }
+            }
+            return result;
+        });
+    }
+
+    private static void checkAuthority(ObjectNode state, int tick, ArrayNode failures) throws IOException {
+        for (int slot = 0; slot < 4; slot++) {
+            JsonNode player = ReplayScenario.player(state, "player-0" + slot);
+            int x = slot == 0 ? 10000 + 400 * Math.min(tick + 1, 194) : Room.SPAWNS[slot][0];
+            if (player.path("x").asInt() != x || player.path("y").asInt() != Room.SPAWNS[slot][1]
+                    || player.path("score").asInt() != (slot == 0 && tick >= 193 ? 1 : 0)
+                    || !player.path("direction").asText().equals(slot == 0 && tick < 194 ? "EAST" : "STOP")
+                    || !player.path("connectivity").asText().equals(slot == 3 && tick >= 194 ? "LEFT" : "CONNECTED"))
+                failures.add("authoritative movement/TAG/lifecycle " + slot + "/" + tick);
+        }
+        if (state.path("tick").asInt() != tick || !state.path("status").asText().equals("RUNNING")) failures.add("fixed clock/lifecycle");
+    }
+
+    private static ObjectNode read(TcpClient client) throws Exception {
+        DataInputStream input = (DataInputStream) ReplayFixture.field(client, "input"); int length = input.readInt();
+        if (length < 1 || length > CompleteFrame.MAX_BYTES) throw new IOException("server frame bound");
+        byte[] bytes = new byte[length]; input.readFully(bytes); return Json.read(bytes);
+    }
+    private static String hash(byte[] bytes) throws Exception {
+        return HexFormat.of().formatHex(MessageDigest.getInstance("SHA-256").digest(bytes));
+    }
+
+    private static final class Replica {
+        private final TreeMap<Long, ArrayNode> retained = new TreeMap<>();
+        private ArrayNode state = Json.MAPPER.createArrayNode(); private long sequence; private int count;
+        void apply(ObjectNode snapshot) throws IOException {
+            long next = snapshot.path("snapshot_seq").asLong();
+            if (next != sequence + 1) throw new IOException("snapshot sequence discontinuity");
+            TreeMap<String, JsonNode> players = new TreeMap<>();
+            if (snapshot.path("kind").asText().equals("DELTA")) {
+                ArrayNode base = retained.get(snapshot.path("base_snapshot_seq").asLong());
+                if (base == null) throw new IOException("client missing delta base");
+                for (JsonNode player : base) players.put(player.path("player_id").asText(), player);
+            } else if (!snapshot.path("status").asText().equals("RUNNING")) throw new IOException("FULL lacks actual Room status");
+            String last = "";
+            for (JsonNode player : snapshot.withArray("players")) {
+                String id = player.path("player_id").asText();
+                if (id.compareTo(last) <= 0 || player.size() != VISIBLE_FIELDS.size()) throw new IOException("sorted seven-field player rows required");
+                for (String field : VISIBLE_FIELDS) if (!player.has(field)) throw new IOException("visible player field missing");
+                if (snapshot.path("kind").asText().equals("DELTA") && player.equals(players.get(id)))
+                    throw new IOException("delta included an unchanged player");
+                players.put(id, player); last = id;
+            }
+            last = "";
+            for (JsonNode removed : snapshot.withArray("removed_player_ids")) {
+                if (removed.asText().compareTo(last) <= 0) throw new IOException("removal order");
+                if (players.remove(removed.asText()) == null) throw new IOException("removal did not reference a base player");
+                last = removed.asText();
+            }
+            state = Json.MAPPER.createArrayNode(); players.values().forEach(state::add); sequence = next;
+            retained.put(sequence, state.deepCopy()); if (retained.size() > 32) retained.pollFirstEntry();
+        }
+        ArrayNode state() { return state; }
+    }
+}
diff --git a/src/test/java/arena/SnapshotStreamTest.java b/src/test/java/arena/SnapshotStreamTest.java
new file mode 100644
index 0000000..f99cbe3
--- /dev/null
+++ b/src/test/java/arena/SnapshotStreamTest.java
@@ -0,0 +1,46 @@
+package arena;
+
+import static org.junit.jupiter.api.Assertions.*;
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import java.util.Map;
+import java.util.concurrent.ExecutionException;
+import java.util.concurrent.FutureTask;
+import org.junit.jupiter.api.Test;
+
+/** The single permitted pure 33-snapshot retention fixture; no simulation ticks or TCP campaign. */
+final class SnapshotStreamTest {
+    @Test void retainOnlyLast32ImmutableStatesAndReleaseOnClose() throws Exception {
+        Room room = new Room("retention-room"); room.join("z-player"); room.join("a-player");
+        SnapshotStream stream = new SnapshotStream(); ObjectNode last = null;
+        for (int seq = 1; seq <= 33; seq++) {
+            if (seq > 1) stream.acknowledge(seq == 33 ? 999 : seq - 1);
+            last = stream.next(room, false);
+            assertEquals(seq, last.path("snapshot_seq").asInt());
+            boolean full = seq == 1 || seq == 20 || seq == 33;
+            assertEquals(full ? "FULL" : "DELTA", last.path("kind").asText());
+            assertEquals(full ? 2 : 0, last.withArray("players").size());
+            assertEquals(0, last.withArray("removed_player_ids").size());
+            assertEquals(Math.min(seq, 32), stream.retainedCount());
+            assertEquals(ReplayLog.hash(room.canonicalRecord()), last.path("state_hash").asText());
+            if (seq == 1) {
+                assertEquals("a-player", last.withArray("players").get(0).path("player_id").asText());
+                assertEquals(7, last.withArray("players").get(0).size());
+                ((ObjectNode) last.withArray("players").get(0)).put("x", -1); // Must not mutate the retained base.
+            }
+        }
+        Map<?, ?> retained = (Map<?, ?>) ReplayFixture.field(stream, "retained");
+        assertFalse(retained.containsKey(1L)); assertTrue(retained.containsKey(2L)); assertTrue(retained.containsKey(33L));
+        ObjectNode observed = Json.MAPPER.createObjectNode().put("probe", "G08 pure33 retention")
+                .put("generated_snapshots", 33).put("high_water", stream.highWater()).put("executed_ticks", room.executedTicks())
+                .put("unknown_ack_seq", 999).put("last_kind", last.path("kind").asText());
+        var ids = observed.putArray("retained_base_ids"); for (Object id : retained.keySet()) ids.add(((Number) id).longValue());
+        FutureTask<Integer> foreign = new FutureTask<>(stream::retainedCount);
+        Thread thread = new Thread(foreign, "g08-owner-check"); thread.start(); thread.join(5_000); assertFalse(thread.isAlive());
+        assertInstanceOf(IllegalStateException.class, assertThrows(ExecutionException.class, foreign::get).getCause());
+        stream.close(); assertEquals(0, stream.retainedCount()); assertTrue(retained.isEmpty());
+        assertThrows(IllegalStateException.class, () -> stream.next(room, false));
+        room.close(); assertEquals(0, room.replayStoredBytes()); assertEquals(0, room.executedTicks());
+        observed.put("closed_retained", stream.retainedCount()).put("owner_rejected", true);
+        System.out.println(observed);
+    }
+}
diff --git a/src/test/resources/G08.json b/src/test/resources/G08.json
new file mode 100644
index 0000000..16e3947
--- /dev/null
+++ b/src/test/resources/G08.json
@@ -0,0 +1,127 @@
+{
+  "scenario_id": "G08-snapshot-base",
+  "contract_version": 1,
+  "thread": "G08",
+  "seed": 7050,
+  "clock": {
+    "kind": "manual",
+    "tick_duration_ms": 50
+  },
+  "ticks": 196,
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
+  "initialization": "reuse G07 test-build-only four-player owner bootstrap; all realtime channels TCP-ready; normal public join semantics unchanged",
+  "events": [
+    {
+      "before_tick": 0,
+      "kind": "INPUT",
+      "client": "alpha",
+      "seq": 1,
+      "target_tick": 0,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 193,
+      "kind": "INPUT",
+      "client": "alpha",
+      "seq": 2,
+      "target_tick": 193,
+      "direction": "EAST",
+      "tag_target_role": "delta",
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 194,
+      "kind": "INPUT",
+      "client": "alpha",
+      "seq": 3,
+      "target_tick": 194,
+      "direction": "STOP",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 194,
+      "kind": "LEAVE_ROOM",
+      "client": "delta"
+    }
+  ],
+  "snapshot_cadence": {
+    "room_start_seq": 1,
+    "room_start_tick": -1,
+    "every_executed_ticks": 2,
+    "full_every_snapshot_seq_multiple": 20,
+    "retention": 32
+  },
+  "ack_schedule": [
+    {
+      "after_snapshot_seq": 1,
+      "send_ack": 1,
+      "clients": "all"
+    },
+    {
+      "after_snapshot_seq": 2,
+      "send_ack": 2,
+      "clients": "all"
+    },
+    {
+      "after_snapshot_seq": 3,
+      "send_ack": 999,
+      "clients": "all"
+    },
+    {
+      "after_snapshot_seq": 4,
+      "send_ack": 4,
+      "clients": "all"
+    },
+    {
+      "after_snapshot_seq_range": [
+        5,
+        99
+      ],
+      "send_ack": "actual applied sequence",
+      "clients": "all still connected"
+    }
+  ],
+  "expected_snapshot_count_for_remaining_clients": 99,
+  "socket_ceiling_ms": 5000
+}


