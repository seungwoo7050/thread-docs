## `fix: reject unpublished snapshot acknowledgement bases`

diff --git a/evidence/G08-repair1-command-ledger.jsonl b/evidence/G08-repair1-command-ledger.jsonl
new file mode 100644
index 0000000..685651e
--- /dev/null
+++ b/evidence/G08-repair1-command-ledger.jsonl
@@ -0,0 +1,11 @@
+{"kind": "activation", "thread": "G08", "profile": "realtime-core", "spec_revision": "c1d62196ab76b55652f5d75a67514f8c6d8210ce", "attempt": "repair1", "start": "319da6c31b766cc26e1342d36724687ddbed8770", "thread_start": "9f70abaf1c3419b58114d2f15c653f34f431936b", "budget": {"compile": 8, "unit_including_reproduction": 4, "integration": 2, "post_canonical": 1, "network_fault": 0, "load": 0}, "production_hash_manifest": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-repair1/reproduce-unit/production-hashes-before.json", "scenario_sha256": "d121973167461dddbb3ba0bd339ed26b09486cda8bbec642328ccc5cea9e578e"}
+{"kind": "resolved_before_execution", "pass": "baseline", "category": "unit-reproduction", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test", "--tests", "arena.SnapshotStreamTest.unknownFutureAcknowledgementCannotBecomeABaseLater"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-repair1/reproduce-unit", "reserved_ticks": 0, "resolved_at": "2026-08-28T04:46:36.189092+00:00"}
+{"kind": "resolved_before_execution", "pass": "build", "category": "build", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "build"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-repair1/verify-build", "reserved_ticks": 0, "resolved_at": "2026-08-28T04:46:36.189092+00:00"}
+{"kind": "resolved_before_execution", "pass": "unit", "category": "unit", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-repair1/verify-unit", "reserved_ticks": 0, "resolved_at": "2026-08-28T04:46:36.189092+00:00"}
+{"kind": "resolved_before_execution", "pass": "integration", "category": "integration", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "integration-test"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-repair1/verify-integration", "reserved_ticks": 0, "resolved_at": "2026-08-28T04:46:36.189092+00:00"}
+{"kind": "resolved_before_execution", "pass": "canonical", "category": "canonical", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "scenario-run", "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G08.json", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-repair1/canonical/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-repair1/canonical", "reserved_ticks": 196, "resolved_at": "2026-08-28T04:46:36.189092+00:00"}
+{"pass": "baseline", "category": "unit-reproduction", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test", "--tests", "arena.SnapshotStreamTest.unknownFutureAcknowledgementCannotBecomeABaseLater"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T04:46:46.493477+00:00", "duration_seconds": 4.655, "command_process_id": 76662, "exit_code": 1, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-repair1/reproduce-unit/stdout.log", "process_terminated": true, "compiler_tasks_executed": ["> Task :compileTestJava"], "xml": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-repair1/reproduce-unit/xml"}
+{"pass": "build", "category": "build", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "build"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T04:47:26.429692+00:00", "duration_seconds": 5.09, "command_process_id": 76921, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-repair1/verify-build/stdout.log", "process_terminated": true, "compiler_tasks_executed": ["> Task :compileJava", "> Task :compileTestJava"]}
+{"pass": "unit", "category": "unit", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T04:47:31.520629+00:00", "duration_seconds": 4.014, "command_process_id": 76959, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-repair1/verify-unit/stdout.log", "process_terminated": true, "compiler_tasks_executed": [], "xml": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-repair1/verify-unit/xml"}
+{"pass": "integration", "category": "integration", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "integration-test"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T04:47:35.537652+00:00", "duration_seconds": 4.724, "command_process_id": 77009, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-repair1/verify-integration/stdout.log", "process_terminated": true, "compiler_tasks_executed": [], "xml": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-repair1/verify-integration/xml"}
+{"pass": "canonical", "category": "canonical", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "scenario-run", "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G08.json", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-repair1/canonical/result.json"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T04:47:40.263074+00:00", "duration_seconds": 1.226, "command_process_id": 77052, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-repair1/canonical/stdout.log", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g08-repair1/canonical/result.json", "simulation_process_id": 77052, "executed_ticks": 196, "snapshot_counts": {"alpha": 99, "bravo": 99, "charlie": 99, "delta": 98}}
diff --git a/evidence/G08-repair1-verification.md b/evidence/G08-repair1-verification.md
new file mode 100644
index 0000000..27085a2
--- /dev/null
+++ b/evidence/G08-repair1-verification.md
@@ -0,0 +1,27 @@
+# G08 — bounded repair1
+
+Repair START `319da6c31b766cc26e1342d36724687ddbed8770`; original Thread START `9f70abaf1c3419b58114d2f15c653f34f431936b`. Profile `realtime-core`; spec revision `c1d62196ab76b55652f5d75a67514f8c6d8210ce`. This append preserves the initial commit and evidence.
+
+**Finding and reproduction:** a sequence ACKed before publication must not become a usable base merely because the server publishes it later. The fixed zero-tick regression published1, ACKed unknown2, then published2 and3 without another ACK. Actual START output was FULL1/null, FULL2/null, DELTA3/base2. One JUnit testcase failed on the third publication; errors/skips0. All23 production/build/fixture files matched START before and after reproduction. Raw output, XML, source snapshots and both manifests are under `runs/g08-repair1/reproduce-unit/`.
+
+**Correction:** `SnapshotStream.acknowledge` admits only a sequence already retained at ACK receipt; an unknown sequence clears the usable base. This changes one production assignment and its comment. It adds no hash, stale-watermark, explicit-resync or later-Thread policy. The new pure three-publication test releases stream/replay storage before assertions. The original33-publication retention fixture body, all network scenarios and existing acceptance assertions are unchanged.
+
+Exact argv, pre-resolved output paths, timestamps, process IDs, durations, compiler tasks and every exit are in `G08-repair1-command-ledger.jsonl`.
+
+| Command | Exit | Seconds | Actual result |
+|---|---:|---:|---|
+| `./track unit-test --tests arena.SnapshotStreamTest.unknownFutureAcknowledgementCannotBecomeABaseLater` | 1 expected | 4.655 | 0 ticks,3 publications; third incorrectly DELTA/base2 |
+| `./track build` | 0 | 5.090 | clean main/test compilation and installed artifact |
+| `./track unit-test` | 0 | 4.014 | 43 tests; errors/failures/skips0 |
+| `./track integration-test` | 0 | 4.724 | 4 tests; errors/failures/skips0 |
+| `./track scenario-run <main-G08.json> <repair1-result.json>` | 0 | 1.226 | one196-tick real TCP run, PID77052 |
+
+The post-change pure regression observed FULL1/FULL2/FULL3 with null bases, zero simulation ticks and retained/replay storage0 after close. The unchanged33-publication test observed retained IDs2..33, high-water32, unknown999 fallback, owner rejection and close count0. Shipping-fixture exclusion and PARANOID buffer checks remain in the passing full suite.
+
+The fixed scenario SHA remains `d121973167461dddbb3ba0bd339ed26b09486cda8bbec642328ccc5cea9e578e`. All196 canonical records/hashes and395 wire snapshots equal the initial post-change run. Counts remain alpha/bravo/charlie99 each and delta98. FULL sequences remain1,4,20,40,60,80; DELTA98 contains the score change and DELTA99 contains STOP plus `player-03` removal. Every client application was also checked from the raw delta/base data against the seven-field authoritative projection. Final hash is `2f41457f82749fe037faed4566ea907707c2ca65817081afd5af41fa1d9e2269`.
+
+Accepted replay bytes are identical to initial:118410 bytes, SHA-256 `06381b6292f4cd7530277e94a63a1a45f3b5d919428215c4e872d359183e58b6`. Retention ends at IDs68..99,32 per remaining client,96 total before shutdown and0 after. Pending input/mailbox/outbound high-waters are1/2/2; parser high-water225 bytes/capacity256. Cleanup has zero channels, connections, pending writes, mailbox entries, parser buffers/bytes, replay bytes, retained snapshots, clock backlog and live owned threads; timer stopped, owner/event loops terminated, client sockets released.
+
+Read-only raw audit: `runs/g08-repair1/read-only-audit.json`. Exact canonical output/replay/stdout: `runs/g08-repair1/canonical/`. A checksum manifest verifies that all31 initial raw/summary/ledger files remain byte-identical. No initial evidence was overwritten or discarded.
+
+Repair budget consumed:3 compiler tasks across2 compile-bearing commands/8;2 unit invocations including reproduction/4;1 integration/2;1 post canonical/1. Repair G08 simulation ticks196; both new pure probes use0 ticks. Initial counts remain separately preserved at3 compiler tasks,2 unit,1 integration and1 post canonical. No extra scenario, fault, load, profiler, external infrastructure, tags or push. Unresolved: none in this repair; main verification remains required.
diff --git a/src/main/java/arena/SnapshotStream.java b/src/main/java/arena/SnapshotStream.java
index 17ccd57..93ab778 100644
--- a/src/main/java/arena/SnapshotStream.java
+++ b/src/main/java/arena/SnapshotStream.java
@@ -48,8 +48,8 @@ final class SnapshotStream implements AutoCloseable {
     void acknowledge(long seq) {
         assertOwner();
         if (closed) throw new IllegalStateException("snapshot stream closed");
-        // An unknown base only affects the next scheduled publication, never the simulation.
-        acknowledged = seq;
+        // A future sequence must not become acknowledged merely by being published later.
+        acknowledged = retained.containsKey(seq) ? seq : 0;
     }
     int retainedCount() { assertOwner(); return retained.size(); }
     int highWater() { assertOwner(); return highWater; }
diff --git a/src/test/java/arena/SnapshotStreamTest.java b/src/test/java/arena/SnapshotStreamTest.java
index f99cbe3..1a2e03c 100644
--- a/src/test/java/arena/SnapshotStreamTest.java
+++ b/src/test/java/arena/SnapshotStreamTest.java
@@ -7,7 +7,7 @@ import java.util.concurrent.ExecutionException;
 import java.util.concurrent.FutureTask;
 import org.junit.jupiter.api.Test;
 
-/** The single permitted pure 33-snapshot retention fixture; no simulation ticks or TCP campaign. */
+/** Pure snapshot fixtures; no simulation ticks or TCP campaign. */
 final class SnapshotStreamTest {
     @Test void retainOnlyLast32ImmutableStatesAndReleaseOnClose() throws Exception {
         Room room = new Room("retention-room"); room.join("z-player"); room.join("a-player");
@@ -43,4 +43,35 @@ final class SnapshotStreamTest {
         observed.put("closed_retained", stream.retainedCount()).put("owner_rejected", true);
         System.out.println(observed);
     }
+
+    @Test void unknownFutureAcknowledgementCannotBecomeABaseLater() {
+        Room room = new Room("future-ack-room"); room.join("player-a"); room.join("player-b");
+        SnapshotStream stream = new SnapshotStream();
+        ObjectNode observed = Json.MAPPER.createObjectNode().put("probe", "G08 repair1 future ACK")
+                .put("process_id", ProcessHandle.current().pid()).put("unknown_ack_seq", 2);
+        var snapshots = observed.putArray("snapshots");
+        try {
+            snapshots.add(stream.next(room, false));
+            stream.acknowledge(2); // Sequence 2 has not been published or retained.
+            snapshots.add(stream.next(room, false));
+            snapshots.add(stream.next(room, false));
+            observed.put("retained_before_close", stream.retainedCount());
+        } finally {
+            stream.close(); room.close();
+        }
+        observed.put("executed_ticks", room.executedTicks()).put("closed_retained", stream.retainedCount())
+                .put("closed_replay_bytes", room.replayStoredBytes());
+        System.out.println(observed);
+        assertEquals(0, room.executedTicks()); assertEquals(0, stream.retainedCount()); assertEquals(0, room.replayStoredBytes());
+        assertEquals(3, snapshots.size());
+        for (int index = 0; index < snapshots.size(); index++) {
+            var snapshot = snapshots.get(index);
+            int expectedSequence = index + 1;
+            assertAll("publication " + expectedSequence,
+                    () -> assertEquals(expectedSequence, snapshot.path("snapshot_seq").asInt()),
+                    () -> assertEquals(-1, snapshot.path("tick").asInt()),
+                    () -> assertEquals("FULL", snapshot.path("kind").asText()),
+                    () -> assertTrue(snapshot.path("base_snapshot_seq").isNull()));
+        }
+    }
 }
