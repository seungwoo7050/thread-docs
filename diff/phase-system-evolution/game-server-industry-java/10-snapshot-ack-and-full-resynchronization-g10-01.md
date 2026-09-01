# Snapshot ACK와 Full Resynchronization (G10)

## `feat: preserve snapshot ACK watermarks and schedule FULL recovery`

diff --git a/TRACK.md b/TRACK.md
index 2dc477a..16ad182 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -136,6 +136,12 @@ The canonical runner uses four ordinary unbound TCP joins with test-only identif
 
 Initial G09 full-unit verification failed during one fixture startup; the exact uncommitted tree and raw logs were preserved before repair1. The fresh repair reproduced UDP allocation coupling with one reserved endpoint and zero ticks, then changed only allocation/advertisement use and the directly observed constructor cleanup boundary. The unchanged eleven-case matrix, byte probe and added reserved-port regression pass in the45-test suite; integration4/4, canonical24 and offline24 also pass. See `evidence/G09-verification.md`, `evidence/G09-repair1-provenance.json` and the separate initial/repair command ledgers for failure provenance, checksums and consumption.
 
-G01 initial budget was build/compile <=8, unit suites <=4, integration suites <=2, canonical scenario <=1. Later Threads use their frozen active plans, including G07's explicit five-pass and negative-replay budget. G09 executes one fixed network-fault campaign; load remains0. No test sleep, microbenchmark, fuzzing, reconnect, many-room or distributed implementation is included.
+## G10 ACK and scheduled FULL recovery
+
+Each retained snapshot now includes its canonical hash. An authenticated ACK can advance the session watermark only to a newer retained sequence with a matching optional hash. Stale retained ACKs cannot lower it; unknown/expired ACKs and mismatched hashes leave it unchanged and latch FULL. An optional boolean `resync_required` also latches FULL. The next ordinary scheduled publication consumes that latch without adding a message, tick or sequence, resetting input state, or changing transport. Retention remains32 and close releases it.
+
+The fixed14-tick runner performs normal four-session joins/binds and drops only alpha delta2. Delta3 reconstructs from acknowledged base1; unknown999 causes FULL4, stale1 leaves DELTA5/base4, hash mismatch causes FULL6, and the deliberately missing client base6 causes FULL8 after an unapplied delta7. Actual ACKs never acknowledge missing snapshots. The existing pure33 retention fixture adds the expired-ACK assertion without another publication or live campaign. Baseline and post canonical records plus accepted artifacts are identical; the baseline-only test is archived, not repeated by the full suites. See `evidence/G10-command-ledger.jsonl` and `evidence/G10-verification.md` for exact commands, raw provenance, results and consumption.
+
+G01 initial budget was build/compile <=8, unit suites <=4, integration suites <=2, canonical scenario <=1. Later Threads use their frozen active plans, including G07's explicit five-pass and negative-replay budget. G09 and G10 each execute their fixed network-fault campaign; load remains0. No test sleep, microbenchmark, fuzzing, reconnect, many-room or distributed implementation is included.
 
 JVM concurrency evidence uses owner-confinement assertions plus real cross-thread Netty handoff, actual thread joins and shutdown assertions. No JVM race detector is installed; no sanitizer result is claimed.
diff --git a/evidence/G10-command-ledger.jsonl b/evidence/G10-command-ledger.jsonl
new file mode 100644
index 0000000..f59b531
--- /dev/null
+++ b/evidence/G10-command-ledger.jsonl
@@ -0,0 +1,12 @@
+{"kind": "activation", "thread": "G10", "profile": "realtime-core", "spec_revision": "c1d62196ab76b55652f5d75a67514f8c6d8210ce", "start": "a808121131f8c84ad5a2cc2e5b722d1a5a06dffa", "attempt": "initial", "budget": {"compile": 8, "unit_including_baseline": 4, "integration": 2, "post_canonical": 1, "baseline_fault_campaign": 1, "post_fault_campaign": 1, "load": 0}, "production_hash_manifest": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g10-initial/reproduce-unit/production-hashes-before.json"}
+{"kind": "resolved_before_execution", "pass": "baseline", "category": "unit-reproduction", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test", "--tests", "arena.G10BaselineTest"], "environment": {"ARENA_G10_SCENARIO": "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G10.json", "ARENA_G10_EVIDENCE": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g10-initial/reproduce-unit/result.json"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g10-initial/reproduce-unit", "reserved_g10_ticks": 14, "resolved_at": "2026-08-28T06:07:20.617388+00:00"}
+{"kind": "resolved_before_execution", "pass": "build", "category": "build", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "build"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g10-initial/verify-build", "reserved_g10_ticks": 0, "resolved_at": "2026-08-28T06:07:20.617496+00:00"}
+{"kind": "resolved_before_execution", "pass": "unit", "category": "unit-with-existing-pure33-retention", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g10-initial/verify-unit", "reserved_g10_ticks": 0, "resolved_at": "2026-08-28T06:07:20.617509+00:00"}
+{"kind": "resolved_before_execution", "pass": "integration", "category": "integration", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "integration-test"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g10-initial/verify-integration", "reserved_g10_ticks": 0, "resolved_at": "2026-08-28T06:07:20.617520+00:00"}
+{"kind": "resolved_before_execution", "pass": "canonical", "category": "canonical-fixed-drop", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "scenario-run", "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G10.json", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g10-initial/canonical/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g10-initial/canonical", "reserved_g10_ticks": 14, "resolved_at": "2026-08-28T06:07:20.617530+00:00"}
+{"pass": "baseline", "category": "unit-reproduction", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test", "--tests", "arena.G10BaselineTest"], "environment": {"ARENA_G10_SCENARIO": "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G10.json", "ARENA_G10_EVIDENCE": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g10-initial/reproduce-unit/result.json"}, "kind": "executed", "started_at": "2026-08-28T06:07:37.509501+00:00", "duration_seconds": 4.461, "command_process_id": 23058, "exit_code": 1, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g10-initial/reproduce-unit/stdout.log", "process_terminated": true, "compiler_tasks_executed": ["> Task :compileTestJava"], "xml": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g10-initial/reproduce-unit/xml", "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g10-initial/reproduce-unit/result.json", "simulation_process_id": 23084, "executed_ticks": 14}
+{"pass": "build", "category": "build", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "build"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T06:10:57.564756+00:00", "duration_seconds": 5.973, "command_process_id": 24595, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g10-initial/verify-build/stdout.log", "process_terminated": true, "compiler_tasks_executed": ["> Task :compileJava", "> Task :compileTestJava"]}
+{"pass": "unit", "category": "unit-with-existing-pure33-retention", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T06:11:03.538439+00:00", "duration_seconds": 5.259, "command_process_id": 24780, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g10-initial/verify-unit/stdout.log", "process_terminated": true, "compiler_tasks_executed": [], "xml": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g10-initial/verify-unit/xml"}
+{"pass": "integration", "category": "integration", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "integration-test"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T06:11:08.799820+00:00", "duration_seconds": 5.433, "command_process_id": 24836, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g10-initial/verify-integration/stdout.log", "process_terminated": true, "compiler_tasks_executed": [], "xml": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g10-initial/verify-integration/xml"}
+{"pass": "canonical", "category": "canonical-fixed-drop", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "scenario-run", "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G10.json", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g10-initial/canonical/result.json"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T06:11:46.123456+00:00", "duration_seconds": 1.193, "command_process_id": 25308, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g10-initial/canonical/stdout.log", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g10-initial/canonical/result.json", "simulation_process_id": 25308, "executed_ticks": 14}
+{"kind":"final_read_only_audit","at":"2026-08-28T06:17:54.903330+00:00","runtime_reruns":0,"post_suite_counts":{"verify-unit":{"tests":45,"failures":0,"errors":0,"skipped":0},"verify-integration":{"tests":4,"failures":0,"errors":0,"skipped":0}},"process_inspection":{"ids":[23058,23084,24595,24780,24822,24836,24893,24900,25308],"first_attempt":"sandbox PermissionError for ps; no ledger mutation","authorized_retry_argv":["ps","-p","23058,23084,24595,24780,24822,24836,24893,24900,25308","-o","pid=,comm="],"exit_code":1,"stdout":"","remaining_processes":[]},"integration_child":{"process_id":24900,"exit_code":143,"alive":false,"evidence":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g10-initial/verify-integration/xml/TEST-arena.ServerIntegrationTest.xml"},"canonical":{"executed_ticks":14,"observer_snapshots":8,"drop_count":1,"healthy_captures":24,"records_and_hashes_equal_baseline":true,"artifact_equal_baseline":true,"final_hash":"5af6ca04d0d1c1a4bbdfbd9452de2d1146426a4bdc793e1fa231d14c0a991c8c","result_sha256":"7906183f37986a80865769dd9c677efee413d3bd73eb63b945ecbcb679837115","artifact_sha256":"99e53ad4037cbbfade59867dd942ee0d0a8c3608d254b763608393de74832577","resources_released":true,"max_udp_bytes":711},"budget_consumed":{"compiler_bearing_commands":2,"compiler_tasks":3,"unit":2,"integration":1,"post_canonical":1,"baseline_fault_campaigns":1,"post_fault_campaigns":1,"g10_live_ticks":28,"pure_retention_snapshots":33,"offline":0,"load":0,"repairs":0},"changed_production_sha256":{"src/main/java/arena/ArenaServer.java":"ada43b2e290bcf87c085f97207c70a0e9e67c100cdec6ea21b8b35023241bdb3","src/main/java/arena/CompleteFrame.java":"b19dd530b2ffb998cad5145a7ff7b21da15e27429648bdcbd49763ed159ede33","src/main/java/arena/SnapshotStream.java":"e2734bfad0c0d426bce84f5dbc71651d2f77aac37221684c5b512adc862f5f82"}}
diff --git a/evidence/G10-verification.md b/evidence/G10-verification.md
new file mode 100644
index 0000000..d780bbb
--- /dev/null
+++ b/evidence/G10-verification.md
@@ -0,0 +1,43 @@
+# G10 verification — initial attempt
+
+Profile `realtime-core`; Spec `c1d62196ab76b55652f5d75a67514f8c6d8210ce`.
+START `a808121131f8c84ad5a2cc2e5b722d1a5a06dffa`.
+Frozen G10 input SHA-256: `8f8a9d8d55a092aab97a6923936c17d9adee5333e537488720efdb9e9f40afe2`.
+
+## Reproduction and change
+
+The resolved baseline ran the unchanged G09 server for all14 ticks using four ordinary TCP joins, test-only ID remapping, UDP binding and one real dropped delta2. All16 production source hashes matched START before and after. The archived harness, raw result/artifact and XML preserve the expected assertion failure: unknown ACK999 reset watermark3 to0, stale ACK1 lowered4 to1, and hash mismatch/client resync did not request FULL. Loss recovery at delta3/base1 already passed and was not rewritten. The baseline never fabricated an applied ACK for an unreconstructable snapshot.
+
+Production changes are confined to optional ACK field validation/routing and `SnapshotStream`: retain each published hash beside its bounded visible state, advance the watermark only for newer retained valid ACKs, and latch the next scheduled FULL for unknown/expired ACK, hash mismatch or explicit resync. Publication clears the latch; close releases all retained state and clears it. Tick timing, input admission, game state and transports are unchanged.
+
+## Observed verification
+
+| Check | Actual result |
+|---|---|
+| Baseline unit | Expected exit1;1 assertion failure,0 errors/skips;14 ticks |
+| Clean build | Exit0 |
+| Full unit suite |45 tests;0 failures/errors/skips |
+| Full integration suite |4 tests;0 failures/errors/skips |
+| Fixed post canonical | Exit0;14 ticks,8 observer snapshots,one actual delta2 drop |
+
+The post observer sequence is `FULL1, DELTA2/base1(drop), DELTA3/base1, FULL4, DELTA5/base4, FULL6, DELTA7/base6(unapplied), FULL8`. Unknown999 preserves watermark3; stale1 preserves4; the mismatched ACK5 preserves5. Removing only client base6 leaves server state and client last-applied6 unchanged; the client sends resync ACK6 without ACK7. FULL8 restores the replica and both watermarks reach8. The24 healthy-client captures and all special ACK wire objects/owner barriers are retained in the raw result.
+
+The existing pure33-snapshot unit fixture runs once with zero simulation ticks: retained IDs2..33, high-water32; expired ACK1 preserves watermark33 and latches FULL without generating snapshot34. Close leaves retention0 and the latch false. Owner, immutable-state, unpublished-future-ACK and earlier regressions remain active. The baseline-only test is archived and absent from the final suite; the shipping JAR excludes the G10 runner and fixture.
+
+All14 authoritative canonical records/hashes and the accepted-input artifact are byte-identical to baseline. Final alpha is `(15600,10000)`, EAST, score0, last-sequence1; other players remain at their spawns. Final hash: `5af6ca04d0d1c1a4bbdfbd9452de2d1146426a4bdc793e1fa231d14c0a991c8c`. Artifact SHA-256: `99e53ad4037cbbfade59867dd942ee0d0a8c3608d254b763608393de74832577`.
+
+Cleanup reports zero channels, connections, pending writes/mailbox work, live threads, parser/UDP buffers and allocated bytes, replay bytes and retained snapshots. Owner/event loops terminate; timer/clock stop; both proxy sockets close with zero pending packets. Maximum outbound UDP is711 bytes; all39 received datagrams dispatch with no invalid/oversize/unroutable/I/O failure.
+
+## Ledger and limits
+
+`G10-command-ledger.jsonl` is the single command record, including resolved argv, timestamps, exits, processes, compiler tasks and raw paths. Raw root: `/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g10-initial/`. Baseline provenance is in `reproduce-unit/`; post logs/XML in `verify-build/`, `verify-unit/`, `verify-integration/`; complete14 states, timeline, hashes and artifact in `canonical/result.json` and `canonical/result.replay.jsonl`.
+
+Consumption: two compiler-bearing commands/three compiler tasks of8; unit2/4 including reproduction; integration1/2; post canonical1/1. Exactly one baseline14-tick and one post14-tick fault campaign; pure retention33 once; offline0, load0, repairs0/2. No extra campaign is hidden in the full suites.
+
+Root can use the ledger's build/unit/integration commands and run the canonical once with a fresh output path:
+
+```sh
+./track scenario-run /Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G10.json /private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g10-root/canonical/result.json
+```
+
+Worker checks pass; independent post-change verification remains pending. No tags, push or next-stage work were performed.
diff --git a/src/main/java/arena/ArenaServer.java b/src/main/java/arena/ArenaServer.java
index 2b4fa2e..898918d 100644
--- a/src/main/java/arena/ArenaServer.java
+++ b/src/main/java/arena/ArenaServer.java
@@ -357,7 +357,9 @@ public final class ArenaServer implements AutoCloseable {
                     if (session.playerId == null || !session.playerId.equals(Json.text(message, "player_id"))) {
                         peer.error("PLAYER_MISMATCH"); break;
                     }
-                    session.snapshots.acknowledge(Json.snapshotSequence(message));
+                    session.snapshots.acknowledge(Json.snapshotSequence(message),
+                            message.has("state_hash") ? Json.text(message, "state_hash") : null,
+                            message.has("resync_required") && message.get("resync_required").booleanValue());
                 }
                 case "INPUT" -> {
                     if (!roomMatches(peer, message)) break;
diff --git a/src/main/java/arena/CompleteFrame.java b/src/main/java/arena/CompleteFrame.java
index 16b0789..0708b38 100644
--- a/src/main/java/arena/CompleteFrame.java
+++ b/src/main/java/arena/CompleteFrame.java
@@ -204,6 +204,8 @@ final class CompleteFrame extends SimpleChannelInboundHandler<ByteBuf> {
                     Json.identifier(message, "room_id");
                     Json.identifier(message, "player_id");
                     Json.snapshotSequence(message);
+                    if (message.has("state_hash") && !message.get("state_hash").isTextual()) return "MESSAGE_INVALID";
+                    if (message.has("resync_required") && !message.get("resync_required").isBoolean()) return "MESSAGE_INVALID";
                 }
                 case "INPUT" -> {
                     Json.identifier(message, "session_id");
diff --git a/src/main/java/arena/SnapshotStream.java b/src/main/java/arena/SnapshotStream.java
index 93ab778..df2dcc9 100644
--- a/src/main/java/arena/SnapshotStream.java
+++ b/src/main/java/arena/SnapshotStream.java
@@ -10,11 +10,13 @@ import java.util.TreeMap;
 final class SnapshotStream implements AutoCloseable {
     static final int RETENTION = 32;
     private static final List<String> VISIBLE_FIELDS = List.of("player_id", "slot", "x", "y", "direction", "score", "connectivity");
+    private record Retained(ArrayNode players, String hash) { }
     private final Thread owner = Thread.currentThread();
-    private final TreeMap<Long, ArrayNode> retained = new TreeMap<>();
+    private final TreeMap<Long, Retained> retained = new TreeMap<>();
     private long sequence;
     private long acknowledged;
     private int highWater;
+    private boolean resyncPending;
     private boolean closed;
 
     ObjectNode next(Room room, boolean forceFull) {
@@ -25,35 +27,45 @@ final class SnapshotStream implements AutoCloseable {
             if (player.path("connectivity").asText().equals("LEFT")) continue;
             ObjectNode visible = current.addObject(); for (String field : VISIBLE_FIELDS) visible.set(field, player.path(field));
         }
-        ArrayNode base = retained.get(acknowledged); sequence++;
-        boolean full = forceFull || sequence % 20 == 0 || base == null;
+        Retained base = retained.get(acknowledged); sequence++;
+        boolean full = forceFull || resyncPending || sequence % 20 == 0 || base == null;
+        String hash = ReplayLog.hash(room.canonicalRecord());
         ObjectNode message = Json.message("SNAPSHOT").put("snapshot_seq", sequence).put("room_id", room.id)
                 .put("tick", room.executedTicks() - 1).put("owner_epoch", 0).put("kind", full ? "FULL" : "DELTA")
-                .put("state_hash", ReplayLog.hash(room.canonicalRecord()));
+                .put("state_hash", hash);
         if (full) message.putNull("base_snapshot_seq").put("status", room.status().name());
         else message.put("base_snapshot_seq", acknowledged);
         ArrayNode changed = message.putArray("players"), removed = message.putArray("removed_player_ids");
         TreeMap<String, JsonNode> previous = new TreeMap<>();
-        if (!full) for (JsonNode player : base) previous.put(player.path("player_id").asText(), player);
+        if (!full) for (JsonNode player : base.players()) previous.put(player.path("player_id").asText(), player);
         for (JsonNode player : current) {
             JsonNode old = previous.remove(player.path("player_id").asText());
             if (!player.equals(old)) changed.add(player.deepCopy());
         }
         previous.keySet().forEach(removed::add);
         if (retained.size() == RETENTION) retained.pollFirstEntry();
-        retained.put(sequence, current); highWater = Math.max(highWater, retained.size());
+        retained.put(sequence, new Retained(current, hash)); highWater = Math.max(highWater, retained.size());
+        resyncPending = false;
         return message;
     }
 
     void acknowledge(long seq) {
+        acknowledge(seq, null, false);
+    }
+    void acknowledge(long seq, String reportedHash, boolean resyncRequired) {
         assertOwner();
         if (closed) throw new IllegalStateException("snapshot stream closed");
-        // A future sequence must not become acknowledged merely by being published later.
-        acknowledged = retained.containsKey(seq) ? seq : 0;
+        Retained snapshot = retained.get(seq);
+        // Unknown/expired ACKs and mismatch reports cannot become valid watermarks later.
+        if (snapshot == null || reportedHash != null && !reportedHash.equals(snapshot.hash())) {
+            resyncPending = true; return;
+        }
+        if (resyncRequired) resyncPending = true;
+        if (seq > acknowledged) acknowledged = seq;
     }
     int retainedCount() { assertOwner(); return retained.size(); }
     int highWater() { assertOwner(); return highWater; }
-    @Override public void close() { assertOwner(); retained.clear(); acknowledged = 0; closed = true; }
+    @Override public void close() { assertOwner(); retained.clear(); acknowledged = 0; resyncPending = false; closed = true; }
     private void assertOwner() {
         if (Thread.currentThread() != owner) throw new IllegalStateException("snapshot access outside owner");
     }
diff --git a/src/test/java/arena/AckScenario.java b/src/test/java/arena/AckScenario.java
new file mode 100644
index 0000000..a69bfed
--- /dev/null
+++ b/src/test/java/arena/AckScenario.java
@@ -0,0 +1,264 @@
+package arena;
+
+import com.fasterxml.jackson.databind.JsonNode;
+import com.fasterxml.jackson.databind.node.ArrayNode;
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import java.io.IOException;
+import java.net.DatagramPacket;
+import java.net.DatagramSocket;
+import java.net.InetSocketAddress;
+import java.nio.charset.StandardCharsets;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.util.Arrays;
+import java.util.LinkedHashMap;
+import java.util.List;
+import java.util.Map;
+import java.util.TreeMap;
+
+/** One frozen 14-tick pass over the real server; also observes unchanged G09 without new production APIs. */
+final class AckScenario {
+    static final String SHA256 = "8f8a9d8d55a092aab97a6923936c17d9adee5333e537488720efdb9e9f40afe2";
+    private AckScenario() { }
+
+    static ReplayScenario.Observed run(Path path) throws Exception {
+        byte[] bytes = Files.readAllBytes(path);
+        if (!SHA256.equals(UdpScenario.hash(bytes))) throw new IOException("frozen G10 scenario bytes required");
+        ObjectNode scenario = Json.read(bytes), result = Json.MAPPER.createObjectNode().put("thread", "G10")
+                .put("scenario_sha256", SHA256).put("process_id", ProcessHandle.current().pid()).put("fault_campaigns", 1).put("load_runs", 0)
+                .put("identity_observations", "fixed Room/Player IDs; generated session correlation is replaced by client alias; no credentials");
+        ArrayNode failures = result.putArray("failures"), states = result.putArray("ticks");
+        ArrayNode hashes = result.putArray("state_hashes"), records = result.putArray("canonical_records");
+        result.putArray("observer_timeline"); result.putArray("other_client_captures");
+        Map<String, TcpClient> clients = new LinkedHashMap<>(); Map<String, Replica> replicas = new LinkedHashMap<>();
+        ArenaServer server = new ArenaServer("127.0.0.1", 0, true); DropProxy proxy = null; byte[] artifact = null;
+        try {
+            for (JsonNode player : scenario.withArray("players")) {
+                String role = player.path("client").asText(); TcpClient client = new TcpClient(server.port());
+                clients.put(role, client); replicas.put(role, new Replica()); client.hello();
+            }
+            result.set("normal_joins", ReplayFixture.joinFixed(server, scenario, clients));
+            TcpClient alpha = clients.get("alpha");
+            proxy = new DropProxy(alpha.localUdpEndpoint(), alpha.udpDestination()); alpha.udpDestination(proxy.frontEndpoint());
+            alpha.send(alpha.binding()); proxy.clientPacket(); proxy.serverPacket(); alpha.until("UDP_BOUND");
+            for (var entry : clients.entrySet()) if (!entry.getKey().equals("alpha")) entry.getValue().bind();
+            result.set("initial_state", ReplayFixture.snapshot(server)); capture(server, scenario, clients, replicas, proxy, result, -1);
+            for (int tick = 0; tick < scenario.path("ticks").asInt(); tick++) {
+                for (JsonNode event : scenario.withArray("observer_timeline")) if (event.has("before_tick") && event.path("before_tick").asInt() == tick) {
+                    ObjectNode observation = result.withArray("observer_timeline").addObject().put("before_tick", tick);
+                    if (event.has("client_fault")) {
+                        Replica replica = replicas.get("alpha"); ObjectNode before = ReplayFixture.snapshot(server), streamBefore = stream(server, alpha);
+                        ArrayNode previous = replica.state.deepCopy(); long previousSequence = replica.sequence;
+                        observation.put("kind", "DISCARD_LOCAL_BASE6").set("server_before", before); observation.set("stream_before", streamBefore);
+                        observation.put("removed_only_base6", replica.retained.remove(6L) != null);
+                        observation.put("client_last_applied", replica.sequence).set("replica", replica.state.deepCopy());
+                        observation.set("client_retained_bases", replica.ids());
+                        ObjectNode after = ReplayFixture.snapshot(server); observation.set("server_after", after); observation.set("stream_after", stream(server, alpha));
+                        check(observation.path("removed_only_base6").asBoolean() && before.equals(after) && streamBefore.equals(stream(server, alpha))
+                                && previous.equals(replica.state) && previousSequence == replica.sequence && replica.sequence == 6, failures, "client base6 fault isolation");
+                    } else {
+                        String kind = event.path("send_ack").asLong() == 999 ? "UNKNOWN_ACK" : event.has("state_hash") ? "HASH_MISMATCH" : "STALE_ACK";
+                        observation.put("kind", kind); ObjectNode ack = alpha.auth("SNAPSHOT_ACK", scenario.path("room_id").asText());
+                        ack.set("snapshot_seq", event.path("send_ack")); ack.set("resync_required", event.path("resync_required"));
+                        if (event.has("state_hash")) ack.set("state_hash", event.path("state_hash"));
+                        observation.set("acknowledgement", sendAck(server, alpha, proxy, ack, false, failures));
+                        observation.put("client_last_applied", replicas.get("alpha").sequence).set("replica", replicas.get("alpha").state.deepCopy());
+                    }
+                }
+                for (JsonNode event : scenario.withArray("events")) if (event.path("before_tick").asInt() == tick) {
+                    ObjectNode request = alpha.auth("INPUT", scenario.path("room_id").asText());
+                    for (String field : List.of("seq", "target_tick", "direction", "owner_epoch")) request.set(field, event.path(field));
+                    request.putNull("tag_target_player_id"); ObjectNode admission = result.putObject("input_admission").put("before_tick", tick);
+                    admission.set("before", ReplayFixture.snapshot(server)); alpha.send(request); proxy.clientPacket(); proxy.serverPacket();
+                    ObjectNode response = alpha.until("INPUT_ACK"); admission.set("response", response); admission.set("after", ReplayFixture.snapshot(server));
+                    check(response.path("seq").asInt() == 1 && response.path("status").asText().equals("ACCEPTED"), failures, "fixed input admission");
+                }
+                server.advanceTicks(1); ObjectNode state = ReplayFixture.snapshot(server); states.add(state);
+                hashes.add(state.path("state_hash")); records.add(ReplayFixture.canonicalRecord(server)); checkAuthority(state, tick, failures);
+                if ((tick + 1) % 2 == 0) capture(server, scenario, clients, replicas, proxy, result, tick);
+            }
+            result.set("final_state", ReplayFixture.snapshot(server)); result.set("runtime_metrics", server.metrics());
+            result.set("final_stream", stream(server, alpha)); result.put("client_last_applied", replicas.get("alpha").sequence);
+            result.set("final_replica", replicas.get("alpha").state.deepCopy());
+            check(replicas.get("alpha").state.equals(SnapshotScenario.projection(ReplayFixture.snapshot(server))), failures, "final visible replica");
+            artifact = server.replayArtifact(); result.put("artifact_bytes", artifact.length);
+            check(proxy.snapshotOrdinal == 8 && proxy.drops == 1, failures, "exact eight publications and one dropped delta2");
+            server.close(); for (TcpClient client : clients.values()) client.expectClosed();
+        } catch (Exception failure) {
+            failures.add(failure.getClass().getName() + ": " + failure.getMessage());
+            java.io.StringWriter trace = new java.io.StringWriter(); failure.printStackTrace(new java.io.PrintWriter(trace)); result.put("execution_error", trace.toString());
+        } finally {
+            if (proxy != null) { result.set("fault_trace", proxy.view()); proxy.close(); result.set("proxy_cleanup", proxy.cleanup()); }
+            server.close(); for (TcpClient client : clients.values()) client.close();
+        }
+        result.put("executed_ticks", states.size()); ScenarioRunner.assertCleanup(server.cleanup()); result.set("cleanup", server.cleanup());
+        boolean released = clients.values().stream().allMatch(TcpClient::isClosed)
+                && result.path("proxy_cleanup").path("front_closed").asBoolean() && result.path("proxy_cleanup").path("back_closed").asBoolean();
+        result.put("all_resources_released", released).put("passed", failures.isEmpty() && released && states.size() == 14);
+        return new ReplayScenario.Observed(result, artifact);
+    }
+
+    private static void capture(ArenaServer server, ObjectNode scenario, Map<String, TcpClient> clients, Map<String, Replica> replicas,
+                                DropProxy proxy, ObjectNode result, int tick) throws Exception {
+        ArrayNode failures = result.withArray("failures"); ObjectNode authority = ReplayFixture.snapshot(server);
+        String canonical = ReplayFixture.canonicalRecord(server), hash = UdpScenario.hash(canonical.getBytes(StandardCharsets.UTF_8));
+        for (var entry : clients.entrySet()) {
+            String role = entry.getKey(); TcpClient client = entry.getValue(); boolean observer = role.equals("alpha"), delivered = true;
+            ObjectNode wire; int bytes;
+            if (observer) { DropProxy.Packet packet = proxy.serverPacket(); wire = packet.message(); delivered = packet.delivered(); bytes = packet.bytes(); }
+            else { wire = client.until("SNAPSHOT"); bytes = Json.bytes(wire).length; }
+            long sequence = wire.path("snapshot_seq").asLong(); JsonNode expected = null;
+            if (observer) for (JsonNode item : scenario.withArray("observer_timeline"))
+                if (item.path("snapshot_seq").asLong(-1) == sequence) expected = item;
+            if (observer && expected == null) throw new IOException("snapshot outside frozen observer timeline");
+            ObjectNode cell = result.withArray(observer ? "observer_timeline" : "other_client_captures").addObject().put("client", role)
+                    .put("seq", sequence).put("tick", tick).put("kind", wire.path("kind").asText()).put("bytes", bytes)
+                    .put("dropped", !delivered).put("delivered", delivered).put("canonical_record", canonical).put("canonical_hash", hash);
+            cell.set("base", wire.path("base_snapshot_seq")); cell.set("wire", wire); cell.set("authoritative_state", authority);
+            cell.set("stream_before", stream(server, client)); cell.put("server_ack_watermark_before", stream(server, client).path("acknowledged_seq").asLong());
+            check(wire.path("type").asText().equals("SNAPSHOT") && wire.path("tick").asInt(-2) == tick
+                    && wire.path("owner_epoch").asInt(-1) == 0 && wire.path("room_id").equals(scenario.path("room_id"))
+                    && wire.path("state_hash").asText().equals(hash) && bytes <= 1_200, failures, role + "/" + sequence + " snapshot metadata");
+            String expectedKind = observer ? expected.path("expect").asText() : sequence == 1 ? "FULL" : "DELTA";
+            long expectedBase = observer ? expected.path("expect_base").asLong() : sequence - 1;
+            check(wire.path("kind").asText().equals(expectedKind)
+                    && (expectedKind.equals("FULL") ? wire.path("base_snapshot_seq").isNull() : wire.path("base_snapshot_seq").asLong() == expectedBase),
+                    failures, role + "/" + sequence + " required kind/base");
+            Replica replica = replicas.get(role); long previousSequence = replica.sequence;
+            cell.put("client_last_applied_before", previousSequence); String application = "DROPPED";
+            if (delivered) {
+                if (observer) check(wire.equals(client.until("SNAPSHOT")), failures, "actual proxy/client snapshot equality");
+                application = replica.apply(wire);
+            }
+            boolean applied = application.equals("APPLIED"); cell.put("application", application).put("applied", applied)
+                    .put("client_last_applied", replica.sequence).set("replica", replica.state.deepCopy()); cell.set("client_retained_bases", replica.ids());
+            check(replica.sequence >= previousSequence, failures, "client last-applied decreased");
+            if (observer) check(applied == expected.path("apply").asBoolean(), failures, "observer application " + sequence);
+            if (applied) check(replica.state.equals(SnapshotScenario.projection(authority)), failures, role + "/" + sequence + " visible projection");
+            if (observer && sequence == 3) result.put("lost_delta_base1_recovery", applied && wire.path("base_snapshot_seq").asLong() == 1 ? "NOT_REPRODUCED" : "REPRODUCED");
+            String fallback = "NONE";
+            if (wire.path("kind").asText().equals("FULL")) fallback = switch ((int) sequence) {
+                case 1 -> "ROOM_START"; case 4 -> "UNKNOWN_ACK999"; case 6 -> "HASH_MISMATCH"; case 8 -> "CLIENT_MISSING_BASE6"; default -> "BASE_UNAVAILABLE";
+            };
+            cell.put("fallback_reason", fallback);
+            ObjectNode ack = null; boolean validAppliedAck = false;
+            if (observer && expected.has("send_ack")) {
+                ack = client.auth("SNAPSHOT_ACK", scenario.path("room_id").asText());
+                ack.set("snapshot_seq", expected.path("send_ack")); ack.set("resync_required", expected.path("resync_required"));
+            } else if (applied) {
+                ack = client.auth("SNAPSHOT_ACK", scenario.path("room_id").asText()).put("snapshot_seq", replica.sequence).put("state_hash", wire.path("state_hash").asText());
+                validAppliedAck = true;
+            }
+            cell.put("ack_sent", ack != null).put("applied_ack_sent", validAppliedAck);
+            if (ack != null) cell.set("acknowledgement", sendAck(server, client, observer ? proxy : null, ack, validAppliedAck, failures));
+            cell.set("stream_after", stream(server, client)); cell.put("server_ack_watermark_after", stream(server, client).path("acknowledged_seq").asLong());
+        }
+    }
+
+    private static ObjectNode sendAck(ArenaServer server, TcpClient client, DropProxy proxy, ObjectNode ack,
+                                      boolean validApplied, ArrayNode failures) throws Exception {
+        ObjectNode before = ReplayFixture.snapshot(server), streamBefore = stream(server, client);
+        ObjectNode observed = Json.MAPPER.createObjectNode().put("transport", "UDP").put("valid_applied_ack", validApplied);
+        ObjectNode safe = ack.deepCopy(); safe.remove("session_id"); safe.put("session_alias", client.playerId);
+        observed.set("wire", safe); observed.set("authority_before", before); observed.set("stream_before", streamBefore);
+        int received = ReplayFixture.udpReceived(server); client.send(ack); if (proxy != null) proxy.clientPacket();
+        observed.set("ingress_barrier", ReplayFixture.udpBarrier(server, received + 1));
+        ObjectNode after = ReplayFixture.snapshot(server), streamAfter = stream(server, client);
+        observed.set("authority_after", after); observed.set("stream_after", streamAfter);
+        long previous = streamBefore.path("acknowledged_seq").asLong(), next = streamAfter.path("acknowledged_seq").asLong();
+        check(before.equals(after), failures, "ACK changed simulation/input state");
+        check(next >= previous, failures, "server valid ACK watermark decreased " + previous + "->" + next);
+        check(next == (validApplied ? Math.max(previous, ack.path("snapshot_seq").asLong()) : previous), failures, "ACK adopted invalid/lower watermark");
+        return observed;
+    }
+
+    static ObjectNode stream(ArenaServer server, TcpClient client) throws Exception {
+        return ReplayFixture.owned(server, () -> {
+            for (Object session : ((Map<?, ?>) ReplayFixture.field(server, "sessions")).values())
+                if (ReplayFixture.field(session, "id").equals(client.sessionId)) return stream(ReplayFixture.field(session, "snapshots"));
+            throw new IOException("active session stream missing");
+        });
+    }
+    static ObjectNode stream(Object stream) throws Exception {
+        ObjectNode result = Json.MAPPER.createObjectNode().put("acknowledged_seq", (long) ReplayFixture.field(stream, "acknowledged"))
+                .put("published_sequence", (long) ReplayFixture.field(stream, "sequence"));
+        Map<?, ?> retained = (Map<?, ?>) ReplayFixture.field(stream, "retained"); result.put("retained_count", retained.size());
+        ArrayNode ids = result.putArray("retained_base_ids"); for (Object id : retained.keySet()) ids.add(((Number) id).longValue());
+        try { result.put("resync_pending", (boolean) ReplayFixture.field(stream, "resyncPending")).put("fallback_flag_available", true); }
+        catch (NoSuchFieldException oldImplementation) { result.put("fallback_flag_available", false); }
+        return result;
+    }
+    private static void checkAuthority(ObjectNode state, int tick, ArrayNode failures) throws IOException {
+        for (int slot = 0; slot < 4; slot++) {
+            JsonNode player = ReplayScenario.player(state, "player-0" + slot);
+            check(player.path("x").asInt() == (slot == 0 ? 10_000 + 400 * (tick + 1) : Room.SPAWNS[slot][0])
+                    && player.path("y").asInt() == Room.SPAWNS[slot][1] && player.path("score").asInt() == 0
+                    && player.path("direction").asText().equals(slot == 0 ? "EAST" : "STOP")
+                    && player.path("last_accepted_seq").asInt() == (slot == 0 ? 1 : 0)
+                    && player.path("connectivity").asText().equals("CONNECTED") && player.path("pending_inputs").asInt() == 0, failures, "authority " + slot + "/" + tick);
+        }
+        check(state.path("tick").asInt() == tick && state.path("status").asText().equals("RUNNING"), failures, "tick/status continuity");
+    }
+    private static void check(boolean condition, ArrayNode failures, String message) { if (!condition) failures.add(message); }
+
+    private static final class Replica {
+        final TreeMap<Long, ArrayNode> retained = new TreeMap<>(); ArrayNode state = Json.MAPPER.createArrayNode(); long sequence;
+        String apply(ObjectNode snapshot) throws IOException {
+            long next = snapshot.path("snapshot_seq").asLong(); if (next <= sequence) return "IGNORED_OLD";
+            TreeMap<String, JsonNode> players = new TreeMap<>();
+            if (snapshot.path("kind").asText().equals("DELTA")) {
+                ArrayNode base = retained.get(snapshot.path("base_snapshot_seq").asLong()); if (base == null) return "MISSING_BASE";
+                for (JsonNode player : base) players.put(player.path("player_id").asText(), player);
+            } else if (!snapshot.path("kind").asText().equals("FULL")) throw new IOException("unknown snapshot kind");
+            String previous = "";
+            for (JsonNode player : snapshot.withArray("players")) {
+                String id = player.path("player_id").asText();
+                if (id.compareTo(previous) <= 0 || player.size() != 7) throw new IOException("sorted seven-field row");
+                for (String field : SnapshotScenario.VISIBLE_FIELDS) if (!player.has(field)) throw new IOException("missing visible field");
+                if (player.equals(players.get(id))) throw new IOException("unchanged delta row"); players.put(id, player); previous = id;
+            }
+            previous = "";
+            for (JsonNode removed : snapshot.withArray("removed_player_ids")) {
+                if (removed.asText().compareTo(previous) <= 0 || players.remove(removed.asText()) == null) throw new IOException("sorted delta removal"); previous = removed.asText();
+            }
+            state = Json.MAPPER.createArrayNode(); players.values().forEach(state::add); sequence = next;
+            retained.put(sequence, state.deepCopy()); if (retained.size() > 32) retained.pollFirstEntry(); return "APPLIED";
+        }
+        ArrayNode ids() { ArrayNode ids = Json.MAPPER.createArrayNode(); retained.keySet().forEach(ids::add); return ids; }
+    }
+
+    /** A single actual alpha SNAPSHOT2 drop, with no pending packet, retry, timer or backgroundworker. */
+    private static final class DropProxy implements AutoCloseable {
+        record Packet(ObjectNode message, int bytes, boolean delivered) { }
+        final DatagramSocket front, back; final InetSocketAddress client, server;
+        final ArrayNode snapshots = Json.MAPPER.createArrayNode(); int snapshotOrdinal, drops, maximumBytes;
+        DropProxy(InetSocketAddress client, InetSocketAddress server) throws IOException {
+            this.client = client; this.server = server; front = new DatagramSocket(new InetSocketAddress("127.0.0.1", 0));
+            try { back = new DatagramSocket(new InetSocketAddress("127.0.0.1", 0)); }
+            catch (IOException failure) { front.close(); throw failure; }
+            front.setSoTimeout(5_000); back.setSoTimeout(5_000);
+        }
+        InetSocketAddress frontEndpoint() { return (InetSocketAddress) front.getLocalSocketAddress(); }
+        void clientPacket() throws IOException { packet(true); }
+        Packet serverPacket() throws IOException { return packet(false); }
+        private Packet packet(boolean fromClient) throws IOException {
+            byte[] storage = new byte[1_201]; DatagramPacket packet = new DatagramPacket(storage, storage.length);
+            (fromClient ? front : back).receive(packet);
+            if (!packet.getSocketAddress().equals(fromClient ? client : server) || packet.getLength() < 1 || packet.getLength() > 1_200)
+                throw new IOException("proxy endpoint/datagram bound");
+            byte[] bytes = Arrays.copyOf(storage, packet.getLength()); maximumBytes = Math.max(maximumBytes, bytes.length);
+            ObjectNode message = Json.read(bytes); boolean delivered = true;
+            if (!fromClient && message.path("type").asText().equals("SNAPSHOT")) {
+                snapshotOrdinal++; delivered = snapshotOrdinal != 2; if (!delivered) drops++;
+                snapshots.addObject().put("ordinal", snapshotOrdinal).put("snapshot_seq", message.path("snapshot_seq").asLong())
+                        .put("bytes", bytes.length).put("delivered", delivered).put("fault", delivered ? "PASS" : "DROP_ONCE");
+            }
+            if (delivered) (fromClient ? back : front).send(new DatagramPacket(bytes, bytes.length, fromClient ? server : client));
+            message.remove(List.of("udp_bind_token", "resume_token")); return new Packet(message, bytes.length, delivered);
+        }
+        ObjectNode view() { ObjectNode value = Json.MAPPER.createObjectNode().put("snapshot_originals", snapshotOrdinal).put("drop_count", drops)
+                    .put("pending_packets", 0).put("maximum_datagram_bytes", maximumBytes); value.set("snapshots", snapshots); return value; }
+        ObjectNode cleanup() { return Json.MAPPER.createObjectNode().put("front_closed", front.isClosed()).put("back_closed", back.isClosed()).put("pending_packets", 0).put("live_threads", 0); }
+        @Override public void close() { front.close(); back.close(); }
+    }
+}
diff --git a/src/test/java/arena/ReplayFormatTest.java b/src/test/java/arena/ReplayFormatTest.java
index 3cf4003..cedf7d6 100644
--- a/src/test/java/arena/ReplayFormatTest.java
+++ b/src/test/java/arena/ReplayFormatTest.java
@@ -49,7 +49,8 @@ final class ReplayFormatTest {
             assertNotNull(jar.getJarEntry("arena/ArenaServer.class"));
             assertNull(jar.getJarEntry("G08.json"));
             assertNull(jar.getJarEntry("G09.json"));
-            for (String name : List.of("ReplayFixture", "ReplayScenario", "ReplayVerifier", "ReplayTool", "G07BaselineTest", "SnapshotScenario", "G08BaselineTest", "UdpScenario", "UdpFaultProxy", "UdpBoundaryTest", "G09BaselineTest")) {
+            assertNull(jar.getJarEntry("G10.json"));
+            for (String name : List.of("ReplayFixture", "ReplayScenario", "ReplayVerifier", "ReplayTool", "G07BaselineTest", "SnapshotScenario", "G08BaselineTest", "UdpScenario", "UdpFaultProxy", "UdpBoundaryTest", "G09BaselineTest", "AckScenario", "G10BaselineTest")) {
                 assertNull(jar.getJarEntry("arena/" + name + ".class"));
                 assertThrows(ClassNotFoundException.class, () -> Class.forName("arena." + name, false, production));
             }
diff --git a/src/test/java/arena/ReplayTool.java b/src/test/java/arena/ReplayTool.java
index b73c16f..f36d477 100644
--- a/src/test/java/arena/ReplayTool.java
+++ b/src/test/java/arena/ReplayTool.java
@@ -20,11 +20,11 @@ public final class ReplayTool {
             if (Files.size(input) > 65_536) throw new IllegalArgumentException("scenario byte bound");
             ObjectNode scenario = Json.read(Files.readAllBytes(input));
             String thread = scenario.path("thread").asText();
-            if (thread.equals("G07") || thread.equals("G08") || thread.equals("G09")) {
+            if (thread.equals("G07") || thread.equals("G08") || thread.equals("G09") || thread.equals("G10")) {
                 boolean variant = args.length == 5 && args[3].equals("--variant") && args[4].equals("rejected-removed");
                 if (args.length != 3 && !(thread.equals("G07") && variant)) throw new IllegalArgumentException("unknown scenario variant");
                 ReplayScenario.Observed observed = thread.equals("G07") ? ReplayScenario.run(input, variant)
-                        : thread.equals("G08") ? SnapshotScenario.run(input) : UdpScenario.run(input);
+                        : thread.equals("G08") ? SnapshotScenario.run(input) : thread.equals("G09") ? UdpScenario.run(input) : AckScenario.run(input);
                 result = observed.result();
                 Path artifact = output.resolveSibling(output.getFileName().toString().replaceFirst("\\.json$", "") + ".replay.jsonl");
                 if (observed.replay() != null) {
diff --git a/src/test/java/arena/SnapshotStreamTest.java b/src/test/java/arena/SnapshotStreamTest.java
index 15eafa7..39e610b 100644
--- a/src/test/java/arena/SnapshotStreamTest.java
+++ b/src/test/java/arena/SnapshotStreamTest.java
@@ -34,10 +34,23 @@ final class SnapshotStreamTest {
                 .put("generated_snapshots", 33).put("high_water", stream.highWater()).put("executed_ticks", room.executedTicks())
                 .put("unknown_ack_seq", 999).put("last_kind", last.path("kind").asText());
         var ids = observed.putArray("retained_base_ids"); for (Object id : retained.keySet()) ids.add(((Number) id).longValue());
+        // G10 uses this same bounded33 fixture; no additional snapshot or live campaign.
+        stream.acknowledge(33);
+        ObjectNode beforeExpired = AckScenario.stream(stream);
+        stream.acknowledge(1);
+        ObjectNode afterExpired = AckScenario.stream(stream);
+        ObjectNode expired = observed.putObject("g10_expired_ack").put("expired_ack", 1);
+        expired.set("before", beforeExpired); expired.set("after", afterExpired);
+        assertEquals(33, beforeExpired.path("acknowledged_seq").asLong());
+        assertEquals(33, afterExpired.path("acknowledged_seq").asLong());
+        assertEquals(33, afterExpired.path("published_sequence").asLong());
+        assertEquals(32, afterExpired.path("retained_count").asInt());
+        assertTrue(afterExpired.path("resync_pending").asBoolean());
         FutureTask<Integer> foreign = new FutureTask<>(stream::retainedCount);
         Thread thread = new Thread(foreign, "g08-owner-check"); thread.start(); thread.join(5_000); assertFalse(thread.isAlive());
         assertInstanceOf(IllegalStateException.class, assertThrows(ExecutionException.class, foreign::get).getCause());
         stream.close(); assertEquals(0, stream.retainedCount()); assertTrue(retained.isEmpty());
+        assertFalse(AckScenario.stream(stream).path("resync_pending").asBoolean());
         assertThrows(IllegalStateException.class, () -> stream.next(room, false));
         room.close(); assertEquals(0, room.replayStoredBytes()); assertEquals(0, room.executedTicks());
         observed.put("closed_retained", stream.retainedCount()).put("owner_rejected", true);
diff --git a/src/test/resources/G10.json b/src/test/resources/G10.json
new file mode 100644
index 0000000..7ce2031
--- /dev/null
+++ b/src/test/resources/G10.json
@@ -0,0 +1,164 @@
+{
+  "scenario_id": "G10-ack-resynchronization",
+  "contract_version": 1,
+  "thread": "G10",
+  "seed": 7050,
+  "clock": {
+    "kind": "manual",
+    "tick_duration_ms": 50
+  },
+  "ticks": 14,
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
+  "initialization": "four ordinary TCP joins while all UDP-unbound, then bind all four using server-issued one-time tokens; fixed IDs only in test artifact",
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
+    }
+  ],
+  "observer": "alpha",
+  "other_clients": "apply and ACK latest valid snapshot continuously",
+  "observer_timeline": [
+    {
+      "snapshot_seq": 1,
+      "tick": -1,
+      "expect": "FULL",
+      "apply": true,
+      "ack": 1
+    },
+    {
+      "snapshot_seq": 2,
+      "tick": 1,
+      "expect": "DELTA",
+      "expect_base": 1,
+      "fault": "DROP_ONCE",
+      "apply": false,
+      "ack": null
+    },
+    {
+      "snapshot_seq": 3,
+      "tick": 3,
+      "expect": "DELTA",
+      "expect_base": 1,
+      "apply": true,
+      "ack": 3
+    },
+    {
+      "before_tick": 4,
+      "send_ack": 999,
+      "resync_required": false
+    },
+    {
+      "snapshot_seq": 4,
+      "tick": 5,
+      "expect": "FULL",
+      "expect_base": null,
+      "apply": true,
+      "ack": 4
+    },
+    {
+      "before_tick": 6,
+      "send_ack": 1,
+      "resync_required": false
+    },
+    {
+      "snapshot_seq": 5,
+      "tick": 7,
+      "expect": "DELTA",
+      "expect_base": 4,
+      "apply": true,
+      "ack": 5
+    },
+    {
+      "before_tick": 8,
+      "send_ack": 5,
+      "state_hash": "0000000000000000000000000000000000000000000000000000000000000000",
+      "resync_required": false
+    },
+    {
+      "snapshot_seq": 6,
+      "tick": 9,
+      "expect": "FULL",
+      "expect_base": null,
+      "apply": true,
+      "ack": 6
+    },
+    {
+      "before_tick": 10,
+      "client_fault": "discard local base6 only; keep last-applied sequence6; server unchanged"
+    },
+    {
+      "snapshot_seq": 7,
+      "tick": 11,
+      "expect": "DELTA",
+      "expect_base": 6,
+      "apply": false,
+      "send_ack": 6,
+      "resync_required": true
+    },
+    {
+      "snapshot_seq": 8,
+      "tick": 13,
+      "expect": "FULL",
+      "expect_base": null,
+      "apply": true,
+      "ack": 8
+    }
+  ],
+  "ack_fields": {
+    "snapshot_seq": "highest actually applied; exception is declared unknown999/old1 test input",
+    "state_hash": "optional reported hash; only the explicit mismatch report differs from known authoritative capture",
+    "resync_required": "optional boolean, true only for explicit missing-base report"
+  },
+  "supplemental_unit_probe": {
+    "retention": 32,
+    "generated_snapshots": 33,
+    "expired_ack": 1,
+    "expected": "force full without rolling back known accepted ACK watermark"
+  },
+  "socket_ceiling_ms": 5000
+}


