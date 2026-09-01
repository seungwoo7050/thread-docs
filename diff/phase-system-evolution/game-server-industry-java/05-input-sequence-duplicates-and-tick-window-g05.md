# Input Sequence, Duplicate와 Tick Window (G05)

## `feat: validate sequence and tick input ordering`

diff --git a/TRACK.md b/TRACK.md
index 3f9396a..5830124 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -1,6 +1,6 @@
-# Java arena — through G04
+# Java arena — through G05
 
-Current thread: G04 (G01–G03 regressions retained). Profile: realtime-core. Spec revision: `5a6e4a2f8fc71d4be18c3279583bfc2558d5c232`.
+Current thread: G05 (G01–G04 regressions retained). Profile: realtime-core. Spec revision: `5a6e4a2f8fc71d4be18c3279583bfc2558d5c232`.
 
 ## Frozen versions
 
@@ -23,6 +23,7 @@ The wrapper uses the locally installed Temurin path when JAVA_HOME is unset. On
 ./track scenario-run /absolute/path/to/G02.json /absolute/path/to/framing-evidence.json
 ./track scenario-run /absolute/path/to/G03.json /absolute/path/to/identity-evidence.json
 ./track scenario-run /absolute/path/to/G04.json /absolute/path/to/clock-evidence.json
+./track scenario-run /absolute/path/to/G05.json /absolute/path/to/sequence-evidence.json
 ./track replay-verify /absolute/path/to/replay /absolute/path/to/evidence
 ./track server config/server.json
 ```
@@ -37,7 +38,7 @@ Parser outcomes distinguish NEED_MORE_BYTES, COMPLETE_VALID_MESSAGE, MESSAGE_ERR
 
 Session registry and Room state belong to one dedicated room-owner thread. Network callbacks submit to its `ArrayBlockingQueue(1024)` and never mutate a Room. Each Room public operation checks the constructing owner thread; unit tests reject mutation from another thread. There is one room and at most eight accepted connections. Netty's channel ID is the process-local Connection identifier; separate server-generated UUIDs identify Session, Player and Room. No client chooses these IDs. The G03 runner observes actual issued values and checks the contract's ASCII/length form without fixed-ID injection.
 
-Each player's pending input storage holds at most 64 intents and rejects overflow with `INPUT_QUEUE_FULL`. An owner tick drains that bounded storage, selects the latest pending direction/TAG, moves players in ASCII ID order, then evaluates one-shot TAG with 64-bit squared distance. Direction persists; TAG does not. No seq, target tick or rate-limit contract is activated. Player data is integer only; unknown position/score fields are ignored.
+Each player's pending input storage holds at most 64 accepted intents and rejects overflow with `INPUT_QUEUE_FULL`. An owner tick removes only inputs for that tick, selects their highest accepted sequence, moves players in ASCII ID order, then evaluates one-shot TAG with 64-bit squared distance. Future inputs stay queued. Direction persists; TAG does not. Player data is integer only; unknown position/score fields are ignored. G06 rate validation remains inactive.
 
 Both Netty event loops use explicit bounded task and tail queues (1,024 each), not an unbounded executor queue. Room commands use a one-thread `ThreadPoolExecutor` with `AbortPolicy`; overflow causes a terminal `INPUT_QUEUE_FULL` reply attempt. Each connection bounds outstanding writes to 64. The last slot is reserved as a `CONTROL_BACKPRESSURE` terminal reply. No snapshot retention or delta queue exists at G01. Parser error replies also pass through the same owner mailbox and bounded outbound path, preserving their order with preceding valid messages. Serialized outbound buffers transfer ownership to Netty on `writeAndFlush`; completion decrements an outstanding-buffer metric. Unit tests check actual inbound and outbound reference counts reach zero, including channel disposal. Snapshot cadence/coalescing remain later Threads.
 
@@ -79,7 +80,15 @@ The old `advanceTicks(1)` API was exercised once per frozen wake before producti
 
 For the fixed deltas 50,50,125,0,225,50ms, actual per-iteration ticks are 1,1,2,0,4,2; remaining time is 0,0,25,25,50,0ms. Both wall-clock cases use the same injected monotonic source and accumulator. Canonical evidence separates the comparable `logical` arrays from actual clock readings, production-adapter samples and cleanup counters. The existing real-timer integration test samples the production server's actual monotonic source and retains its executor/timer/socket cleanup assertions. No test sleeps or system wall-clock changes are used.
 
-See `evidence/G04-command-ledger.jsonl` and `evidence/G04-verification.md` for exact commands, baseline provenance, outcomes and budget. Sequence, replay, UDP, many-room scheduling and sustained-overload terminal behavior remain inactive.
+See `evidence/G04-command-ledger.jsonl` and `evidence/G04-verification.md` for exact commands, baseline provenance, outcomes and budget.
+
+## G05 input sequence and target tick
+
+INPUT now requires an integer `seq` in 1..2^64−1 and an integer `target_tick`. `BigInteger` preserves unsigned values without floating-point conversion or signed truncation. Negative target ticks remain integers and fail the late-window check. Identity and lifecycle checks precede Room admission; parser type checks precede owner handoff. Duplicate identity compares only active logical payload fields, so ignored forward-compatible fields cannot create a conflict.
+
+Each Player holds its last accepted sequence and canonical intent. A lower sequence is `INPUT_STALE`; an equal sequence acknowledges `DUPLICATE` only for the same payload, otherwise `SEQUENCE_CONFLICT`. New sequences are checked against inclusive next-tick..next-tick+4 and then the existing 64-input bound. Every rejection preserves the last accepted sequence and accepted queue. Gaps do not block simulation. ACKs record accepted/superseded inputs; per-tick `applied_seq` and `last_accepted_seq` expose actual owner decisions without adding a replay system.
+
+The fixed ten-event/seven-tick scenario uses real TCP admission. Supplemental pure parser/Room tests cover exactly 64/65 admissions, unsigned maximum/duplicate/stale and the seven prescribed invalid numeric forms, with actual buffer disposal checks. Earlier harnesses add sequence and target tick at their original admission boundaries; directions, TAG actions, physical expectations, clock schedules and old assertions are preserved. See `evidence/G05-command-ledger.jsonl` and `evidence/G05-verification.md`. Rate limiting, replay/hash, UDP and later business features remain inactive.
 
 G01 initial budget: build/compile <=8, unit suites <=4, integration suites <=2, canonical scenario <=1; network-fault and load runs exactly zero. Main has its own separately frozen one-build/one-unit/one-integration/one-scenario verification budget. No test sleep, microbenchmark, fuzzing, replay, UDP, reconnect, many-room or distributed implementation is included.
 
diff --git a/evidence/G05-command-ledger.jsonl b/evidence/G05-command-ledger.jsonl
new file mode 100644
index 0000000..a37f08e
--- /dev/null
+++ b/evidence/G05-command-ledger.jsonl
@@ -0,0 +1,6 @@
+{"kind":"resolved_before_baseline","category":"unit-reproduction","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","unit-test","--tests","arena.RoomTest.frozenG05InputOrder"],"environment":{"ARENA_G05_SCENARIO":"/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G05.json","ARENA_G05_EVIDENCE":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g05-initial/reproduce-unit/result.json"},"resolved_at":"2026-08-28T02:49:01.089137+00:00","production_hash_manifest":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g05-initial/reproduce-unit/production-hashes-before.json","output_directory":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g05-initial/reproduce-unit"}
+{"kind":"executed","category":"unit-reproduction","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","unit-test","--tests","arena.RoomTest.frozenG05InputOrder"],"environment":{"ARENA_G05_SCENARIO":"/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G05.json","ARENA_G05_EVIDENCE":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g05-initial/reproduce-unit/result.json"},"started_at":"2026-08-28T02:49:41.501983+00:00","duration_seconds":5.327,"exit_code":1,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g05-initial/reproduce-unit/stdout.log","xml":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g05-initial/reproduce-unit/xml"}
+{"kind":"executed","category":"build","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","build"],"environment":{},"started_at":"2026-08-28T02:55:03.036132+00:00","duration_seconds":5.535,"exit_code":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g05-initial/verify-build/stdout.log"}
+{"kind":"executed","category":"unit","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","unit-test"],"environment":{"ARENA_G05_SCENARIO":"/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G05.json","ARENA_G05_EVIDENCE":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g05-initial/verify-unit/result.json"},"started_at":"2026-08-28T02:55:08.572898+00:00","duration_seconds":4.381,"exit_code":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g05-initial/verify-unit/stdout.log","xml":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g05-initial/verify-unit/xml"}
+{"kind":"executed","category":"integration","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","integration-test"],"environment":{},"started_at":"2026-08-28T02:55:12.956946+00:00","duration_seconds":4.571,"exit_code":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g05-initial/verify-integration/stdout.log","xml":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g05-initial/verify-integration/xml"}
+{"kind":"executed","category":"canonical","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","scenario-run","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G05.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/G05-result.json"],"environment":{},"started_at":"2026-08-28T02:55:17.530030+00:00","duration_seconds":1.214,"exit_code":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g05-initial/verify-canonical/stdout.log"}
diff --git a/evidence/G05-verification.md b/evidence/G05-verification.md
new file mode 100644
index 0000000..229841b
--- /dev/null
+++ b/evidence/G05-verification.md
@@ -0,0 +1,18 @@
+# G05 — initial attempt
+
+START `7ad47d60096e57383e95966571b7c220fa3e27e6`; profile `realtime-core`; spec `5a6e4a2f8fc71d4be18c3279583bfc2558d5c232`.
+Fixture SHA-256 `971899b4bcca18c0087c085e0c0824bb5d678661c0b474f6b165d5723da47ba9` is unchanged.
+
+**Reproduction:** the resolved command `./track unit-test --tests arena.RoomTest.frozenG05InputOrder` ran the real old parser, TCP admission and seven manual ticks. Exit **1**, 5.327s, one assertion failure. All ten attempts were accepted, no duplicate/stale/conflict/window rejection existed, and alpha ended at `(10400,7600)`. All seven invalid numeric forms were accepted. Sequence/application fields were explicitly unavailable rather than fabricated. The existing 64-input bound, 65th rejection, one movement and resource cleanup already passed (NOT_REPRODUCED). All eleven prior main-source files matched START hashes before execution; manifest, baseline harness/test sources, raw JSON, log and XML remain in `runs/g05-initial/reproduce-unit/`. Main was notified before production edits.
+
+**Change:** exact integer parsing preserves unsigned64 sequences, rejects invalid numeric forms, and leaves negative integer target ticks for the late-window check. Player admission compares the last accepted canonical intent, checks the inclusive next-tick..next-tick+4 window, then checks the existing pending bound. Rejections and duplicates preserve accepted state. Each tick selects the highest sequence for that tick, retaining future inputs. Only active logical payload fields define duplicate identity; unknown fields remain ignored. Prior harnesses add required input fields at their original boundaries; physical expectations, TAG rules, clock behavior, ownership and cleanup assertions remain unchanged. No dependency, rate limiter, replay/hash or later feature was added.
+
+**Verification:** exact commands/environment/time/output paths are in `G05-command-ledger.jsonl`. Clean build exit 0 (5.535s); full unit exit 0, **37 tests** (4.381s); integration exit 0, **4 tests** (4.571s); immutable main G05 canonical exit 0 (1.214s), output `G05-result.json`. No skipped tests or post-change failures/retries. Existing G01–G04 PARANOID/parser/owner/clock/shutdown regressions pass.
+
+Actual canonical: accepted `[1,3,4,6,7]`; duplicate `[1]`; rejections `(2,INPUT_STALE)`, `(4,SEQUENCE_CONFLICT)`, `(5,INPUT_LATE)`, `(8,INPUT_TOO_EARLY)`. Applied `[1,4,6,null,null,null,7]`; positions `[[10400,10000],[10000,10000],[10400,10000],[10800,10000],[11200,10000],[11600,10000],[11600,10400]]`. Seven ticks, last accepted 7, bravo `(90000,90000)`, scores 0, Room RUNNING before shutdown. Raw ACKs, authoritative states and resource metrics remain outside the comparable `logical` object.
+
+Supplemental unit results: exactly 64 accepted inputs, attempt 65 `INPUT_QUEUE_FULL`, last accepted 64, selected sequence 64 and one movement to `(10400,10000)`; unsigned maximum `18446744073709551615` remains exact in accepted and applied state, with outcomes `[ACCEPTED,ACCEPTED,DUPLICATE,INPUT_STALE]`; all seven prescribed invalid forms return `MESSAGE_INVALID` with state/pending unchanged. Actual inbound/outbound/cumulation disposal checks pass.
+
+Canonical cleanup: zero open channels, connections, pending writes/mailbox, parser buffers/bytes, clock accumulator and owned live threads; owner/event loops terminated, timer stopped, clients closed. High-water marks: pending input **2**, mailbox **1**, outbound **1**, parser bytes/capacity **272/512**. Pure pending probe reaches the fixed capacity **64** without discarding prior accepted input.
+
+Budget: **4 compiler tasks across 2 compile-bearing commands / 8**, **2 unit / 4** including reproduction, **1 integration / 2**, **1 post-canonical / 1**. Fault/load **0/0**. State hashes `NOT_ACTIVATED_G07`. Unresolved: **none**.
diff --git a/src/main/java/arena/ArenaServer.java b/src/main/java/arena/ArenaServer.java
index 18aa2a3..ae89b77 100644
--- a/src/main/java/arena/ArenaServer.java
+++ b/src/main/java/arena/ArenaServer.java
@@ -263,9 +263,11 @@ public final class ArenaServer implements AutoCloseable {
                     }
                     Room.Direction direction = Room.Direction.valueOf(Json.text(message, "direction"));
                     String target = message.path("tag_target_player_id").isNull() ? null : Json.text(message, "tag_target_player_id");
-                    String code = room.accept(session.playerId, new Room.Intent(direction, target));
-                    if (code != null) peer.error(code);
-                    else peer.send(Json.message("INPUT_ACK").put("status", "ACCEPTED"));
+                    Room.Intent intent = new Room.Intent(Json.sequence(message), Json.targetTick(message), direction, target);
+                    String code = room.accept(session.playerId, intent);
+                    if (code == null || code.equals("DUPLICATE"))
+                        peer.send(Json.message("INPUT_ACK").put("status", code == null ? "ACCEPTED" : code).put("seq", intent.seq()));
+                    else peer.error(code);
                 }
                 case "LEAVE_ROOM" -> {
                     if (!roomMatches(peer, message)) break;
diff --git a/src/main/java/arena/ClockScenario.java b/src/main/java/arena/ClockScenario.java
index d1064f3..7b5c52e 100644
--- a/src/main/java/arena/ClockScenario.java
+++ b/src/main/java/arena/ClockScenario.java
@@ -71,8 +71,8 @@ final class ClockScenario {
             String roomId = alpha.createRoom();
             alpha.join(roomId); bravo.join(roomId);
             alpha.until("SNAPSHOT"); bravo.until("SNAPSHOT");
-            alpha.intent(roomId, scenario.path("directions").path("alpha").asText(), null);
-            bravo.intent(roomId, scenario.path("directions").path("bravo").asText(), null);
+            alpha.intent(roomId, scenario.path("directions").path("alpha").asText(), null, 0);
+            bravo.intent(roomId, scenario.path("directions").path("bravo").asText(), null, 0);
             raw.putObject("issued_identifiers").put("room", roomId).put("alpha", alpha.playerId).put("bravo", bravo.playerId);
             long monotonicMs = 0;
             long wallMs = scenario.path("wall_clock_initial_ms").asLong();
diff --git a/src/main/java/arena/CompleteFrame.java b/src/main/java/arena/CompleteFrame.java
index f7fec31..1a14981 100644
--- a/src/main/java/arena/CompleteFrame.java
+++ b/src/main/java/arena/CompleteFrame.java
@@ -191,7 +191,7 @@ final class CompleteFrame extends SimpleChannelInboundHandler<ByteBuf> {
         if (!version.bigIntegerValue().equals(BigInteger.ONE)) return "PROTOCOL_VERSION_UNSUPPORTED";
         String type = typeNode.textValue();
         try {
-            // Only schemas already active at G02. Sequence, target tick and future messages are not implemented.
+            // Validate active schemas before owner handoff; future message families remain inactive.
             switch (type) {
                 case "HELLO" -> { }
                 case "CREATE_ROOM" -> Json.text(message, "session_id");
@@ -204,6 +204,8 @@ final class CompleteFrame extends SimpleChannelInboundHandler<ByteBuf> {
                     Json.text(message, "room_id");
                     Json.text(message, "player_id");
                     Json.text(message, "direction");
+                    Json.sequence(message);
+                    Json.targetTick(message);
                     JsonNode target = message.get("tag_target_player_id");
                     if (target == null || !(target.isNull() || target.isTextual())) return "MESSAGE_INVALID";
                 }
diff --git a/src/main/java/arena/IdentityScenario.java b/src/main/java/arena/IdentityScenario.java
index efdb232..577c0fe 100644
--- a/src/main/java/arena/IdentityScenario.java
+++ b/src/main/java/arena/IdentityScenario.java
@@ -87,6 +87,7 @@ final class IdentityScenario {
             } else {
                 sender = world.client(cell.path("sender").asText());
                 request = sender.auth("INPUT", world.roomId)
+                        .put("seq", 1).put("target_tick", 0).put("owner_epoch", 0)
                         .put("session_id", world.client(cell.path("session_from").asText()).sessionId)
                         .put("player_id", world.client(cell.path("player_from").asText()).playerId)
                         .put("direction", cell.path("direction").asText()).putNull("tag_target_player_id");
@@ -206,6 +207,7 @@ final class IdentityScenario {
             });
             await(held);
             sender.send(sender.auth("INPUT", world.roomId)
+                    .put("seq", 1).put("target_tick", 0).put("owner_epoch", 0)
                     .put("direction", scenario.path("owner_probe").path("direction").asText())
                     .putNull("tag_target_player_id"));
             await(observed);
diff --git a/src/main/java/arena/Json.java b/src/main/java/arena/Json.java
index e534af8..e82f208 100644
--- a/src/main/java/arena/Json.java
+++ b/src/main/java/arena/Json.java
@@ -9,6 +9,7 @@ import com.fasterxml.jackson.databind.JsonNode;
 import com.fasterxml.jackson.databind.ObjectMapper;
 import com.fasterxml.jackson.databind.node.ObjectNode;
 import java.io.IOException;
+import java.math.BigInteger;
 import java.nio.ByteBuffer;
 import java.nio.charset.CodingErrorAction;
 import java.nio.charset.StandardCharsets;
@@ -46,4 +47,18 @@ final class Json {
         if (value == null || !value.isTextual()) throw new IllegalArgumentException(field + " must be text");
         return value.textValue();
     }
+
+    static BigInteger sequence(ObjectNode object) {
+        JsonNode value = object.get("seq");
+        if (value == null || !value.isIntegralNumber()) throw new IllegalArgumentException("seq must be an unsigned integer");
+        BigInteger seq = value.bigIntegerValue();
+        if (seq.signum() <= 0 || seq.bitLength() > 64) throw new IllegalArgumentException("seq range");
+        return seq;
+    }
+
+    static BigInteger targetTick(ObjectNode object) {
+        JsonNode value = object.get("target_tick");
+        if (value == null || !value.isIntegralNumber()) throw new IllegalArgumentException("target_tick must be an integer");
+        return value.bigIntegerValue();
+    }
 }
diff --git a/src/main/java/arena/Room.java b/src/main/java/arena/Room.java
index 925d2a1..195748c 100644
--- a/src/main/java/arena/Room.java
+++ b/src/main/java/arena/Room.java
@@ -2,6 +2,7 @@ package arena;
 
 import com.fasterxml.jackson.databind.node.ArrayNode;
 import com.fasterxml.jackson.databind.node.ObjectNode;
+import java.math.BigInteger;
 import java.util.ArrayDeque;
 import java.util.ArrayList;
 import java.util.HashSet;
@@ -12,13 +13,15 @@ import java.util.TreeMap;
 final class Room {
     static final int DURATION = 1_200;
     static final int INPUT_LIMIT = 64;
+    static final int MAX_FUTURE_TICKS = 4;
     static final int[][] SPAWNS = {
         {10_000, 10_000}, {90_000, 90_000}, {10_000, 90_000}, {90_000, 10_000},
         {50_000, 10_000}, {50_000, 90_000}, {10_000, 50_000}, {90_000, 50_000}
     };
     enum Direction { STOP, NORTH, EAST, SOUTH, WEST }
     enum Status { LOBBY, RUNNING, FINISHED, CLOSED }
-    record Intent(Direction direction, String target) { }
+    // Only active logical payload fields define duplicate identity; unknown JSON fields are ignored.
+    record Intent(BigInteger seq, BigInteger targetTick, Direction direction, String target) { }
     record Rejection(String playerId, String code) { }
 
     static final class Player {
@@ -31,6 +34,9 @@ final class Room {
         int lastTagTick = -20;
         Direction direction = Direction.STOP;
         boolean connected = true;
+        BigInteger lastAcceptedSeq = BigInteger.ZERO;
+        Intent lastAcceptedIntent;
+        BigInteger appliedSeq;
 
         Player(String id, int slot) {
             this.id = id; this.slot = slot;
@@ -40,6 +46,7 @@ final class Room {
         ObjectNode view() {
             return Json.MAPPER.createObjectNode().put("player_id", id).put("slot", slot)
                     .put("x", x).put("y", y).put("direction", direction.name()).put("score", score)
+                    .put("last_accepted_seq", lastAcceptedSeq).put("applied_seq", appliedSeq)
                     .put("connectivity", connected ? "CONNECTED" : "LEFT");
         }
     }
@@ -78,8 +85,16 @@ final class Room {
         if (status != Status.RUNNING) return "ROOM_NOT_RUNNING";
         Player player = players.get(id);
         if (player == null || !player.connected) return "PLAYER_MISMATCH";
+        if (intent.seq().signum() <= 0 || intent.seq().bitLength() > 64) return "MESSAGE_INVALID";
+        int comparison = intent.seq().compareTo(player.lastAcceptedSeq);
+        if (comparison < 0) return "INPUT_STALE";
+        if (comparison == 0) return intent.equals(player.lastAcceptedIntent) ? "DUPLICATE" : "SEQUENCE_CONFLICT";
+        if (intent.targetTick().compareTo(BigInteger.valueOf(executedTicks)) < 0) return "INPUT_LATE";
+        if (intent.targetTick().compareTo(BigInteger.valueOf(executedTicks + MAX_FUTURE_TICKS)) > 0) return "INPUT_TOO_EARLY";
         if (player.pending.size() == INPUT_LIMIT) return "INPUT_QUEUE_FULL";
         player.pending.addLast(intent);
+        player.lastAcceptedSeq = intent.seq();
+        player.lastAcceptedIntent = intent;
         inputHighWater = Math.max(inputHighWater, player.pending.size());
         return null;
     }
@@ -88,11 +103,20 @@ final class Room {
         assertOwner();
         if (status != Status.RUNNING) return List.of();
         TreeMap<String, String> tags = new TreeMap<>();
+        BigInteger currentTick = BigInteger.valueOf(executedTicks);
         for (Player player : players.values()) {
             Intent selected = null;
-            while (!player.pending.isEmpty()) selected = player.pending.removeFirst();
+            player.appliedSeq = null;
+            var pending = player.pending.iterator();
+            while (pending.hasNext()) {
+                Intent candidate = pending.next();
+                if (!candidate.targetTick().equals(currentTick)) continue;
+                if (selected == null || candidate.seq().compareTo(selected.seq()) > 0) selected = candidate;
+                pending.remove();
+            }
             if (!player.connected) continue;
             if (selected != null) {
+                player.appliedSeq = selected.seq();
                 player.direction = selected.direction();
                 if (selected.target() != null) tags.put(player.id, selected.target());
             }
diff --git a/src/main/java/arena/ScenarioRunner.java b/src/main/java/arena/ScenarioRunner.java
index ae803bc..79cb438 100644
--- a/src/main/java/arena/ScenarioRunner.java
+++ b/src/main/java/arena/ScenarioRunner.java
@@ -36,6 +36,14 @@ final class ScenarioRunner {
             } catch (IOException failure) { throw failure; }
             catch (Exception failure) { throw new IOException("G04 clock scenario failed", failure); }
         }
+        if (scenario.path("thread").asText().equals("G05")) {
+            try {
+                ObjectNode result = SequenceScenario.run(scenario, sha256(scenarioBytes));
+                if (!result.path("passed").asBoolean()) throw new IOException("G05 assertions: " + result.path("failures"));
+                return result;
+            } catch (IOException failure) { throw failure; }
+            catch (Exception failure) { throw new IOException("G05 input scenario failed", failure); }
+        }
         if (!scenario.path("thread").asText().equals("G01") || scenario.path("contract_version").asInt() != 1
                 || !scenario.path("clock").path("kind").asText().equals("manual")
                 || scenario.path("clock").path("tick_duration_ms").asInt() != 50
@@ -77,7 +85,7 @@ final class ScenarioRunner {
                 String target = step.path("tag_target").isNull() ? null
                         : requiredClient(clients, step.path("tag_target").asText()).playerId;
                 // INPUT_ACK is emitted only after the owner queues the input. This is the barrier.
-                actor.intent(roomId, step.path("direction").asText(), target);
+                actor.intent(roomId, step.path("direction").asText(), target, before);
                 accepted++;
             }
             server.advanceTicks(scenario.path("ticks").asInt() - currentTick);
diff --git a/src/main/java/arena/SequenceScenario.java b/src/main/java/arena/SequenceScenario.java
new file mode 100644
index 0000000..e72523f
--- /dev/null
+++ b/src/main/java/arena/SequenceScenario.java
@@ -0,0 +1,206 @@
+package arena;
+
+import com.fasterxml.jackson.databind.JsonNode;
+import com.fasterxml.jackson.databind.node.ArrayNode;
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import io.netty.buffer.ByteBuf;
+import io.netty.channel.embedded.EmbeddedChannel;
+import java.io.DataInputStream;
+import java.io.IOException;
+import java.lang.reflect.Field;
+import java.util.ArrayList;
+import java.util.List;
+import java.util.concurrent.ThreadPoolExecutor;
+import java.util.concurrent.TimeUnit;
+
+/** G05 fixture observations from production transport, admission and authoritative state. */
+final class SequenceScenario {
+    static final String SHA256 = "971899b4bcca18c0087c085e0c0824bb5d678661c0b474f6b165d5723da47ba9";
+    private SequenceScenario() { }
+
+    static ObjectNode run(ObjectNode scenario, String hash) throws Exception {
+        if (!SHA256.equals(hash)) throw new IOException("G05 requires frozen scenario bytes");
+        ObjectNode result = Json.MAPPER.createObjectNode().put("thread", "G05").put("contract_version", 1)
+                .put("scenario_sha256", hash).put("state_hashes", "NOT_ACTIVATED_G07")
+                .put("network_fault_runs", 0).put("load_runs", 0);
+        ObjectNode logical = result.putObject("logical");
+        ArrayNode accepted = logical.putArray("accepted_sequences"), duplicates = logical.putArray("duplicate_sequences");
+        ArrayNode rejected = logical.putArray("rejections"), applied = logical.putArray("applied_sequences");
+        ArrayNode positions = logical.putArray("alpha_positions"), events = result.putArray("events"), ticks = result.putArray("ticks");
+        ArrayNode failures = result.putArray("failures");
+        ArenaServer server = new ArenaServer("127.0.0.1", 0, true);
+        TcpClient alpha = null, bravo = null;
+        try {
+            alpha = new TcpClient(server.port()); bravo = new TcpClient(server.port());
+            alpha.hello(); String room = alpha.createRoom(); alpha.join(room); bravo.hello(); bravo.join(room);
+            alpha.until("SNAPSHOT"); bravo.until("SNAPSHOT");
+            int eventIndex = 0;
+            for (int tick = 0; tick < scenario.path("ticks").asInt(); tick++) {
+                while (eventIndex < scenario.withArray("events").size()
+                        && scenario.withArray("events").get(eventIndex).path("before_tick").asInt() == tick) {
+                    JsonNode event = scenario.withArray("events").get(eventIndex++);
+                    ObjectNode request = alpha.auth("INPUT", room);
+                    for (String key : List.of("seq", "target_tick", "direction", "tag_target_player_id", "owner_epoch"))
+                        request.set(key, event.path(key).deepCopy());
+                    ObjectNode before = snapshot(server);
+                    alpha.send(request);
+                    ObjectNode response = response(alpha);
+                    ObjectNode after = snapshot(server);
+                    ObjectNode observed = events.addObject().put("before_tick", tick);
+                    observed.set("request", request); observed.set("response", response); observed.set("after", after);
+                    if (response.path("type").asText().equals("ERROR")) {
+                        ObjectNode rejection = rejected.addObject(); rejection.set("seq", event.path("seq"));
+                        rejection.put("code", response.path("code").asText());
+                        if (!before.equals(after)) failures.add("rejected input changed state/pending queue");
+                    } else if (response.path("type").asText().equals("INPUT_ACK")) {
+                        if (response.path("status").asText().equals("DUPLICATE")) {
+                            duplicates.add(event.path("seq"));
+                            if (!before.equals(after)) failures.add("duplicate input changed state/pending queue");
+                        } else accepted.add(event.path("seq"));
+                    } else failures.add("unexpected input response");
+                }
+                server.advanceTicks(1);
+                ObjectNode state = snapshot(server); ticks.add(state);
+                JsonNode actor = player(state, alpha.playerId), other = player(state, bravo.playerId);
+                applied.add(actor.path("applied_seq").isMissingNode() ? Json.MAPPER.nullNode() : actor.path("applied_seq"));
+                positions.addArray().add(actor.path("x").asInt()).add(actor.path("y").asInt());
+                if (other.path("x").asInt() != 90000 || other.path("y").asInt() != 90000
+                        || !state.path("status").asText().equals("RUNNING")) failures.add("bravo/lifecycle changed");
+            }
+            ObjectNode finalState = snapshot(server);
+            JsonNode actor = player(finalState, alpha.playerId), other = player(finalState, bravo.playerId);
+            logical.put("executed_ticks", finalState.path("executed_ticks").asInt());
+            logical.set("final_last_accepted_seq", actor.path("last_accepted_seq").isMissingNode()
+                    ? Json.MAPPER.nullNode() : actor.path("last_accepted_seq"));
+            logical.putArray("bravo_position").add(other.path("x").asInt()).add(other.path("y").asInt());
+            logical.putObject("scores").put("alpha", actor.path("score").asInt()).put("bravo", other.path("score").asInt());
+            result.set("runtime_metrics", server.metrics());
+            server.close(); alpha.expectClosed(); bravo.expectClosed();
+        } finally {
+            try { if (alpha != null) alpha.close(); }
+            finally { try { if (bravo != null) bravo.close(); } finally { server.close(); } }
+        }
+        ScenarioRunner.assertCleanup(server.cleanup()); result.set("cleanup", server.cleanup());
+        logical.put("all_resources_released", alpha != null && bravo != null && alpha.isClosed() && bravo.isClosed());
+        expect(accepted, "[1,3,4,6,7]", failures, "accepted sequences");
+        expect(duplicates, "[1]", failures, "duplicate sequences");
+        expect(rejected, "[{\"seq\":2,\"code\":\"INPUT_STALE\"},{\"seq\":4,\"code\":\"SEQUENCE_CONFLICT\"},"
+                + "{\"seq\":5,\"code\":\"INPUT_LATE\"},{\"seq\":8,\"code\":\"INPUT_TOO_EARLY\"}]", failures, "rejections");
+        expect(applied, "[1,4,6,null,null,null,7]", failures, "applied sequences");
+        expect(positions, "[[10400,10000],[10000,10000],[10400,10000],[10800,10000],[11200,10000],[11600,10000],[11600,10400]]", failures, "positions");
+        if (!logical.path("final_last_accepted_seq").toString().equals("7")) failures.add("last accepted sequence");
+        result.put("passed", failures.isEmpty()); return result;
+    }
+
+    static ObjectNode supplemental(ObjectNode scenario) throws Exception {
+        ObjectNode result = Json.MAPPER.createObjectNode();
+        ArrayNode failures = result.putArray("failures");
+        JsonNode fixed = scenario.path("supplemental_unit_probes");
+        ObjectNode bound = result.putObject("pending_bound");
+        try (Pure probe = new Pure()) {
+            ArrayNode outcomes = bound.putArray("outcomes");
+            int first = fixed.path("pending_bound").path("sequences").path("first").asInt();
+            int last = fixed.path("pending_bound").path("sequences").path("last").asInt();
+            for (int seq = first; seq <= last; seq++) outcomes.add(probe.send(input().put("seq", seq)));
+            bound.put("pending_before_tick", probe.room.player("player-a").pending.size());
+            for (int i = 0; i < 64; i++) if (!outcomes.get(i).asText().equals("ACCEPTED")) failures.add("pending admission " + i);
+            if (!outcomes.get(64).asText().equals("INPUT_QUEUE_FULL") || bound.path("pending_before_tick").asInt() != 64)
+                failures.add("pending bound");
+            JsonNode lastAccepted = probe.room.player("player-a").view().path("last_accepted_seq");
+            bound.set("last_accepted_before_tick", lastAccepted.isMissingNode() ? Json.MAPPER.nullNode() : lastAccepted);
+            if (!lastAccepted.toString().equals("64")) failures.add("queue rejection advanced/missing last sequence");
+            probe.room.tick(); bound.set("after_tick", probe.room.view("SNAPSHOT"));
+            if (probe.room.player("player-a").x != 10400 || !probe.room.player("player-a").pending.isEmpty()
+                    || !probe.room.player("player-a").view().path("applied_seq").toString().equals("64"))
+                failures.add("one bounded selected effect");
+        }
+        ObjectNode unsigned = result.putObject("unsigned_sequence");
+        try (Pure probe = new Pure()) {
+            ArrayNode outcomes = unsigned.putArray("outcomes");
+            for (JsonNode seq : fixed.path("unsigned_sequence").withArray("sequence_values")) {
+                ObjectNode request = input(); request.set("seq", seq); outcomes.add(probe.send(request));
+            }
+            expect(outcomes, "[\"ACCEPTED\",\"ACCEPTED\",\"DUPLICATE\",\"INPUT_STALE\"]", failures, "unsigned sequence outcomes");
+            unsigned.set("before_tick", probe.room.view("SNAPSHOT")); probe.room.tick();
+            unsigned.set("after_tick", probe.room.view("SNAPSHOT"));
+            if (!probe.room.player("player-a").view().path("last_accepted_seq").toString().equals("18446744073709551615"))
+                failures.add("unsigned max truncated/missing");
+            if (probe.room.player("player-a").x != 10400
+                    || !probe.room.player("player-a").view().path("applied_seq").toString().equals("18446744073709551615"))
+                failures.add("unsigned selected effect");
+        }
+        ArrayNode invalid = result.putArray("invalid_numeric_cases");
+        for (String key : List.of("seq", "target_tick")) {
+            String values = key.equals("seq") ? "invalid_seq_values" : "invalid_target_tick_values";
+            for (JsonNode value : fixed.withArray(values)) try (Pure probe = new Pure()) {
+                ObjectNode before = probe.room.view("SNAPSHOT");
+                ObjectNode request = input(); request.set(key, value);
+                String code = probe.send(request);
+                ObjectNode cell = invalid.addObject().put("field", key).put("code", code);
+                cell.set("value", value); cell.set("after", probe.room.view("SNAPSHOT"));
+                cell.put("pending", probe.room.player("player-a").pending.size());
+                if (!code.equals("MESSAGE_INVALID") || !before.equals(probe.room.view("SNAPSHOT"))
+                        || !probe.room.player("player-a").pending.isEmpty()) failures.add("invalid " + key + " " + value);
+            }
+        }
+        result.put("passed", failures.isEmpty()).put("all_parser_buffers_released", true);
+        return result;
+    }
+
+    private static ObjectNode input() {
+        return Json.message("INPUT").put("session_id", "session-unit").put("room_id", "room-unit").put("player_id", "player-a")
+                .put("seq", 1).put("target_tick", 0).put("direction", "EAST").putNull("tag_target_player_id");
+    }
+    private static final class Pure implements AutoCloseable {
+        final Room room = new Room("room-unit");
+        final List<String> outcomes = new ArrayList<>();
+        final CompleteFrame parser = new CompleteFrame(message -> {
+            String code = room.accept("player-a", new Room.Intent(Json.sequence(message), Json.targetTick(message),
+                    Room.Direction.valueOf(Json.text(message, "direction")), null));
+            outcomes.add(code == null ? "ACCEPTED" : code);
+        });
+        final EmbeddedChannel channel = new EmbeddedChannel(parser);
+        Pure() { room.join("player-a"); room.join("player-b"); }
+        String send(ObjectNode request) throws IOException {
+            outcomes.clear(); ByteBuf inbound = CompleteFrame.encode(request); channel.writeInbound(inbound);
+            if (inbound.refCnt() != 0) throw new IOException("pure parser inbound leak");
+            if (!channel.isOpen()) throw new IOException("numeric rejection unexpectedly closed channel");
+            ByteBuf response;
+            while ((response = channel.readOutbound()) != null) try {
+                int length = response.readInt(); byte[] bytes = new byte[length]; response.readBytes(bytes);
+                outcomes.add(Json.read(bytes).path("code").asText());
+            } finally { response.release(); }
+            if (outcomes.size() != 1) throw new IOException("pure input response count");
+            return outcomes.getFirst();
+        }
+        @Override public void close() throws IOException {
+            ByteBuf owned = parser.ownedBuffer(); channel.finishAndReleaseAll(); room.close();
+            if (owned != null && owned.refCnt() != 0 || parser.bufferedBytes() != 0 || !room.player("player-a").pending.isEmpty())
+                throw new IOException("pure parser/Room cleanup");
+        }
+    }
+
+    private static ObjectNode response(TcpClient client) throws Exception {
+        DataInputStream input = (DataInputStream) field(client, "input");
+        int length = input.readInt(); if (length < 1 || length > CompleteFrame.MAX_BYTES) throw new IOException("server frame bound");
+        byte[] bytes = new byte[length]; input.readFully(bytes); return Json.read(bytes);
+    }
+    private static ObjectNode snapshot(ArenaServer server) throws Exception {
+        return ((ThreadPoolExecutor) field(server, "owner")).submit(() -> {
+            Room room = (Room) field(server, "room"); ObjectNode view = room.view("SNAPSHOT");
+            int pending = 0;
+            for (JsonNode player : view.withArray("players")) pending += room.player(player.path("player_id").asText()).pending.size();
+            return view.put("pending_inputs", pending);
+        }).get(5, TimeUnit.SECONDS);
+    }
+    private static Object field(Object object, String name) throws ReflectiveOperationException {
+        Field field = object.getClass().getDeclaredField(name); field.setAccessible(true); return field.get(object);
+    }
+    private static JsonNode player(ObjectNode snapshot, String id) throws IOException {
+        for (JsonNode player : snapshot.withArray("players")) if (player.path("player_id").asText().equals(id)) return player;
+        throw new IOException("missing authoritative player");
+    }
+    private static void expect(JsonNode actual, String expected, ArrayNode failures, String label) {
+        if (!actual.toString().equals(expected)) failures.add(label);
+    }
+}
diff --git a/src/main/java/arena/TcpClient.java b/src/main/java/arena/TcpClient.java
index f2d8887..e133433 100644
--- a/src/main/java/arena/TcpClient.java
+++ b/src/main/java/arena/TcpClient.java
@@ -17,6 +17,7 @@ final class TcpClient implements AutoCloseable {
     private final List<String> lifecycle = new ArrayList<>();
     String sessionId;
     String playerId;
+    private long nextSequence = 1;
 
     TcpClient(int port) throws IOException {
         try {
@@ -73,8 +74,9 @@ final class TcpClient implements AutoCloseable {
         return request;
     }
 
-    void intent(String roomId, String direction, String targetId) throws IOException {
-        ObjectNode request = auth("INPUT", roomId).put("direction", direction);
+    void intent(String roomId, String direction, String targetId, int targetTick) throws IOException {
+        ObjectNode request = auth("INPUT", roomId).put("direction", direction)
+                .put("seq", nextSequence++).put("target_tick", targetTick).put("owner_epoch", 0);
         if (targetId == null) request.putNull("tag_target_player_id");
         else request.put("tag_target_player_id", targetId);
         send(request);
diff --git a/src/test/java/arena/RoomTest.java b/src/test/java/arena/RoomTest.java
index 948dff8..f7e7a04 100644
--- a/src/test/java/arena/RoomTest.java
+++ b/src/test/java/arena/RoomTest.java
@@ -12,6 +12,22 @@ import java.util.concurrent.TimeUnit;
 import org.junit.jupiter.api.Test;
 
 final class RoomTest {
+    @Test void frozenG05InputOrder() throws Exception {
+        byte[] bytes;
+        String scenarioPath = System.getenv("ARENA_G05_SCENARIO");
+        if (scenarioPath != null) bytes = Files.readAllBytes(Path.of(scenarioPath));
+        else try (var input = getClass().getResourceAsStream("/G05.json")) { assertNotNull(input); bytes = input.readAllBytes(); }
+        String hash = HexFormat.of().formatHex(MessageDigest.getInstance("SHA-256").digest(bytes));
+        assertEquals(SequenceScenario.SHA256, hash);
+        ObjectNode scenario = Json.read(bytes), result = SequenceScenario.run(scenario, hash);
+        ObjectNode supplemental = SequenceScenario.supplemental(scenario); result.set("supplemental", supplemental);
+        String output = System.getenv("ARENA_G05_EVIDENCE");
+        if (output != null) Files.write(Path.of(output), Json.MAPPER.writerWithDefaultPrettyPrinter().writeValueAsBytes(result));
+        System.out.println("G05 input ordering " + result);
+        assertTrue(result.path("passed").asBoolean() && supplemental.path("passed").asBoolean(),
+                result.path("failures") + " " + supplemental.path("failures"));
+    }
+
     @Test void frozenG04ClockSchedule() throws Exception {
         byte[] bytes;
         String scenarioPath = System.getenv("ARENA_G04_SCENARIO");
@@ -54,9 +70,14 @@ final class RoomTest {
         return room;
     }
 
+    private Room.Intent intent(Room room, Room.Direction direction, String target) {
+        return new Room.Intent(room.player("player-a").lastAcceptedSeq.add(java.math.BigInteger.ONE),
+                java.math.BigInteger.valueOf(room.executedTicks()), direction, target);
+    }
+
     @Test void serverMovementIsIntegerAndClamped() {
         Room room = runningRoom();
-        assertNull(room.accept("player-a", new Room.Intent(Room.Direction.EAST, null)));
+        assertNull(room.accept("player-a", intent(room, Room.Direction.EAST, null)));
         room.tick();
         assertEquals(10_400, room.player("player-a").x);
         assertEquals(10_000, room.player("player-a").y);
@@ -67,13 +88,13 @@ final class RoomTest {
 
     @Test void tagUsesWideSquaredDistanceAndIsOneShot() {
         Room room = runningRoom();
-        room.accept("player-a", new Room.Intent(Room.Direction.STOP, "player-b"));
+        room.accept("player-a", intent(room, Room.Direction.STOP, "player-b"));
         assertEquals("ACTION_REJECTED", room.tick().getFirst().code());
         assertEquals(0, room.player("player-a").score);
         // Deterministic unit setup of state owned by this thread, no transport shortcut in scenarios.
         room.player("player-b").x = 10_000;
         room.player("player-b").y = 10_000;
-        room.accept("player-a", new Room.Intent(Room.Direction.STOP, "player-b"));
+        room.accept("player-a", intent(room, Room.Direction.STOP, "player-b"));
         assertTrue(room.tick().isEmpty());
         assertEquals(1, room.player("player-a").score);
         for (int i = 0; i < 40; i++) room.tick();
@@ -83,8 +104,8 @@ final class RoomTest {
     @Test void inputStorageHasHardBoundAndDrains() {
         Room room = runningRoom();
         for (int i = 0; i < Room.INPUT_LIMIT; i++)
-            assertNull(room.accept("player-a", new Room.Intent(Room.Direction.STOP, null)));
-        assertEquals("INPUT_QUEUE_FULL", room.accept("player-a", new Room.Intent(Room.Direction.EAST, null)));
+            assertNull(room.accept("player-a", intent(room, Room.Direction.STOP, null)));
+        assertEquals("INPUT_QUEUE_FULL", room.accept("player-a", intent(room, Room.Direction.EAST, null)));
         assertEquals(64, room.player("player-a").pending.size());
         room.tick();
         assertTrue(room.player("player-a").pending.isEmpty());
diff --git a/src/test/java/arena/ServerIntegrationTest.java b/src/test/java/arena/ServerIntegrationTest.java
index 56a7df0..6afad9a 100644
--- a/src/test/java/arena/ServerIntegrationTest.java
+++ b/src/test/java/arena/ServerIntegrationTest.java
@@ -27,10 +27,11 @@ final class ServerIntegrationTest {
             String room = alpha.createRoom();
             alpha.join(room); bravo.join(room);
             ObjectNode attempt = alpha.auth("INPUT", room).put("direction", "EAST")
+                    .put("seq", 1).put("target_tick", 0).put("owner_epoch", 0)
                     .put("position", 999_999).put("score", 999).putNull("tag_target_player_id");
             alpha.send(attempt);
             alpha.until("INPUT_ACK");
-            bravo.intent(room, "WEST", null);
+            bravo.intent(room, "WEST", null, 0);
             server.advanceTicks(1_200);
             ObjectNode first = alpha.until("ROOM_FINISHED");
             assertEquals(first, bravo.until("ROOM_FINISHED"));
diff --git a/src/test/resources/G05.json b/src/test/resources/G05.json
new file mode 100644
index 0000000..bc1568a
--- /dev/null
+++ b/src/test/resources/G05.json
@@ -0,0 +1,151 @@
+{
+  "scenario_id": "G05-input-order",
+  "contract_version": 1,
+  "thread": "G05",
+  "seed": 7050,
+  "clock": {
+    "kind": "manual",
+    "tick_duration_ms": 50
+  },
+  "clients": [
+    "alpha",
+    "bravo"
+  ],
+  "setup": "alpha HELLO/create/join, bravo HELLO/join; both RUNNING at next_tick0, server-issued identities",
+  "ticks": 7,
+  "max_future_input_ticks": 4,
+  "pending_input_bound": 64,
+  "events": [
+    {
+      "before_tick": 0,
+      "client": "alpha",
+      "seq": 1,
+      "target_tick": 0,
+      "direction": "EAST",
+      "tag_target_player_id": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 0,
+      "client": "alpha",
+      "seq": 1,
+      "target_tick": 0,
+      "direction": "EAST",
+      "tag_target_player_id": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 1,
+      "client": "alpha",
+      "seq": 3,
+      "target_tick": 1,
+      "direction": "SOUTH",
+      "tag_target_player_id": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 1,
+      "client": "alpha",
+      "seq": 2,
+      "target_tick": 1,
+      "direction": "NORTH",
+      "tag_target_player_id": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 1,
+      "client": "alpha",
+      "seq": 4,
+      "target_tick": 1,
+      "direction": "WEST",
+      "tag_target_player_id": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 1,
+      "client": "alpha",
+      "seq": 4,
+      "target_tick": 1,
+      "direction": "SOUTH",
+      "tag_target_player_id": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 2,
+      "client": "alpha",
+      "seq": 5,
+      "target_tick": 1,
+      "direction": "NORTH",
+      "tag_target_player_id": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 2,
+      "client": "alpha",
+      "seq": 6,
+      "target_tick": 2,
+      "direction": "EAST",
+      "tag_target_player_id": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 2,
+      "client": "alpha",
+      "seq": 7,
+      "target_tick": 6,
+      "direction": "NORTH",
+      "tag_target_player_id": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 2,
+      "client": "alpha",
+      "seq": 8,
+      "target_tick": 7,
+      "direction": "SOUTH",
+      "tag_target_player_id": null,
+      "owner_epoch": 0
+    }
+  ],
+  "supplemental_unit_probes": {
+    "pending_bound": {
+      "fresh_room": true,
+      "client": "alpha",
+      "before_tick": 0,
+      "sequences": {
+        "first": 1,
+        "last": 65
+      },
+      "target_tick": 0,
+      "direction": "EAST",
+      "tag_target_player_id": null,
+      "capacity": 64
+    },
+    "unsigned_sequence": {
+      "fresh_room": true,
+      "before_tick": 0,
+      "sequence_values": [
+        1,
+        18446744073709551615,
+        18446744073709551615,
+        18446744073709551614
+      ],
+      "target_tick": 0,
+      "direction": "EAST",
+      "tag_target_player_id": null
+    },
+    "invalid_seq_values": [
+      0,
+      -1,
+      1.0,
+      "1",
+      18446744073709551616
+    ],
+    "invalid_target_tick_values": [
+      0.0,
+      "0"
+    ],
+    "invalid_case_isolation": "one fresh room/player per invalid value; otherwise valid INPUT fields"
+  },
+  "socket_ceiling_ms": 5000
+}
