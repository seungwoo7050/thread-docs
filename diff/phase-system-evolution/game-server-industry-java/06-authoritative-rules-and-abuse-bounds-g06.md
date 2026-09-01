# Authoritative Rule과 Abuse Bound

## `feat: bound per-tick input validation attempts`

diff --git a/TRACK.md b/TRACK.md
index 5830124..80f9867 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -1,6 +1,7 @@
-# Java arena — through G05
+# Java arena — through G06
 
-Current thread: G05 (G01–G04 regressions retained). Profile: realtime-core. Spec revision: `5a6e4a2f8fc71d4be18c3279583bfc2558d5c232`.
+Current thread: G06 (G01–G05 regressions retained). Phase: phase-1. Profile: realtime-core. Spec revision: `c1d62196ab76b55652f5d75a67514f8c6d8210ce`.
+G01–G05 retain their original spec trailers and verification. The user-authorized spec/profile transition changes procedure only; the game contract is unchanged. Scope remains G01–G14, with no G15+, external infrastructure or push.
 
 ## Frozen versions
 
@@ -24,6 +25,7 @@ The wrapper uses the locally installed Temurin path when JAVA_HOME is unset. On
 ./track scenario-run /absolute/path/to/G03.json /absolute/path/to/identity-evidence.json
 ./track scenario-run /absolute/path/to/G04.json /absolute/path/to/clock-evidence.json
 ./track scenario-run /absolute/path/to/G05.json /absolute/path/to/sequence-evidence.json
+./track scenario-run /absolute/path/to/G06.json /absolute/path/to/authority-evidence.json
 ./track replay-verify /absolute/path/to/replay /absolute/path/to/evidence
 ./track server config/server.json
 ```
@@ -38,7 +40,7 @@ Parser outcomes distinguish NEED_MORE_BYTES, COMPLETE_VALID_MESSAGE, MESSAGE_ERR
 
 Session registry and Room state belong to one dedicated room-owner thread. Network callbacks submit to its `ArrayBlockingQueue(1024)` and never mutate a Room. Each Room public operation checks the constructing owner thread; unit tests reject mutation from another thread. There is one room and at most eight accepted connections. Netty's channel ID is the process-local Connection identifier; separate server-generated UUIDs identify Session, Player and Room. No client chooses these IDs. The G03 runner observes actual issued values and checks the contract's ASCII/length form without fixed-ID injection.
 
-Each player's pending input storage holds at most 64 accepted intents and rejects overflow with `INPUT_QUEUE_FULL`. An owner tick removes only inputs for that tick, selects their highest accepted sequence, moves players in ASCII ID order, then evaluates one-shot TAG with 64-bit squared distance. Future inputs stay queued. Direction persists; TAG does not. Player data is integer only; unknown position/score fields are ignored. G06 rate validation remains inactive.
+Each player's pending input storage holds at most 64 accepted intents and rejects overflow with `INPUT_QUEUE_FULL`. An owner tick removes only inputs for that tick, selects their highest accepted sequence, moves players in ASCII ID order, then evaluates one-shot TAG with 64-bit squared distance. Future inputs stay queued. Direction persists; TAG does not. Player data is integer only; unknown position/score fields are ignored. The owner's G06 guard permits the first four attributed INPUT attempts between ticks to reach admission validation.
 
 Both Netty event loops use explicit bounded task and tail queues (1,024 each), not an unbounded executor queue. Room commands use a one-thread `ThreadPoolExecutor` with `AbortPolicy`; overflow causes a terminal `INPUT_QUEUE_FULL` reply attempt. Each connection bounds outstanding writes to 64. The last slot is reserved as a `CONTROL_BACKPRESSURE` terminal reply. No snapshot retention or delta queue exists at G01. Parser error replies also pass through the same owner mailbox and bounded outbound path, preserving their order with preceding valid messages. Serialized outbound buffers transfer ownership to Netty on `writeAndFlush`; completion decrements an outstanding-buffer metric. Unit tests check actual inbound and outbound reference counts reach zero, including channel disposal. Snapshot cadence/coalescing remain later Threads.
 
@@ -88,7 +90,15 @@ INPUT now requires an integer `seq` in 1..2^64−1 and an integer `target_tick`.
 
 Each Player holds its last accepted sequence and canonical intent. A lower sequence is `INPUT_STALE`; an equal sequence acknowledges `DUPLICATE` only for the same payload, otherwise `SEQUENCE_CONFLICT`. New sequences are checked against inclusive next-tick..next-tick+4 and then the existing 64-input bound. Every rejection preserves the last accepted sequence and accepted queue. Gaps do not block simulation. ACKs record accepted/superseded inputs; per-tick `applied_seq` and `last_accepted_seq` expose actual owner decisions without adding a replay system.
 
-The fixed ten-event/seven-tick scenario uses real TCP admission. Supplemental pure parser/Room tests cover exactly 64/65 admissions, unsigned maximum/duplicate/stale and the seven prescribed invalid numeric forms, with actual buffer disposal checks. Earlier harnesses add sequence and target tick at their original admission boundaries; directions, TAG actions, physical expectations, clock schedules and old assertions are preserved. See `evidence/G05-command-ledger.jsonl` and `evidence/G05-verification.md`. Rate limiting, replay/hash, UDP and later business features remain inactive.
+The fixed ten-event/seven-tick scenario uses real TCP admission. Supplemental pure parser/Room tests cover exactly 64/65 admissions, unsigned maximum/duplicate/stale and the seven prescribed invalid numeric forms, with actual buffer disposal checks. Earlier harnesses add sequence and target tick at their original admission boundaries; directions, TAG actions, physical expectations, clock schedules and old assertions are preserved. See `evidence/G05-command-ledger.jsonl` and `evidence/G05-verification.md`.
+
+## G06 authority and input validation bound
+
+The existing movement, clamp, TAG range/cooldown, connectivity and ASCII actor-order rules already passed the fixed baseline and were not rewritten. Invalid directions return `MESSAGE_INVALID` without committing sequence, movement or pending input. A valid foreign/absent TAG target is admitted, then emits `ACTION_REJECTED` at its target tick without a score/cooldown effect. Ignored client position, score and elapsed-time fields cannot alter the server's fixed movement.
+
+After the existing parser schema and transport identity checks, an owner-confined counter permits four INPUT attempts for each active Player between simulation ticks, including attempts rejected by direction or sequence admission. The fifth returns `INPUT_RATE_EXCEEDED` before those validations, without reserving its sequence. Counters saturate at four and reset on the next executed tick. Inactive/foreign identities do not consume another actor's allowance. Existing parser/lifecycle precedence and the independent lower-level 64-entry pending bound are preserved.
+
+The frozen 18-attempt/221-tick TCP scenario records real admission replies, TAG failures and successes, before/after state and all resulting ticks. Pure fixtures cover the prescribed clamps, range boundaries, LEFT/self/other-Room targets, shared-victim ordering, four invalid attempts and the retained 64-entry bound. No server ID injection or multi-room scheduling was added. See `evidence/G06-command-ledger.jsonl` and `evidence/G06-verification.md`. Replay/hash, UDP and later features remain inactive.
 
 G01 initial budget: build/compile <=8, unit suites <=4, integration suites <=2, canonical scenario <=1; network-fault and load runs exactly zero. Main has its own separately frozen one-build/one-unit/one-integration/one-scenario verification budget. No test sleep, microbenchmark, fuzzing, replay, UDP, reconnect, many-room or distributed implementation is included.
 
diff --git a/evidence/G06-command-ledger.jsonl b/evidence/G06-command-ledger.jsonl
new file mode 100644
index 0000000..981ee90
--- /dev/null
+++ b/evidence/G06-command-ledger.jsonl
@@ -0,0 +1,6 @@
+{"kind":"resolved_before_baseline","category":"unit-reproduction","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","unit-test","--tests","arena.RoomTest.frozenG06AuthorityAbuse"],"environment":{"ARENA_G06_SCENARIO":"/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G06.json","ARENA_G06_EVIDENCE":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g06-initial/reproduce-unit/result.json"},"resolved_at":"2026-08-28T03:19:20.092434+00:00","production_hash_manifest":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g06-initial/reproduce-unit/production-hashes-before.json","output_directory":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g06-initial/reproduce-unit"}
+{"kind":"executed","category":"unit-reproduction","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","unit-test","--tests","arena.RoomTest.frozenG06AuthorityAbuse"],"environment":{"ARENA_G06_SCENARIO":"/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G06.json","ARENA_G06_EVIDENCE":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g06-initial/reproduce-unit/result.json"},"started_at":"2026-08-28T03:24:09.321406+00:00","duration_seconds":6.064,"exit_code":1,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g06-initial/reproduce-unit/stdout.log","xml":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g06-initial/reproduce-unit/xml"}
+{"kind":"executed","category":"build","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","build"],"environment":{},"started_at":"2026-08-28T03:26:53.745677+00:00","duration_seconds":5.496,"exit_code":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g06-initial/verify-build/stdout.log"}
+{"kind":"executed","category":"unit","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","unit-test"],"environment":{"ARENA_G06_SCENARIO":"/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G06.json","ARENA_G06_EVIDENCE":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g06-initial/verify-unit/result.json"},"started_at":"2026-08-28T03:26:59.242482+00:00","duration_seconds":4.233,"exit_code":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g06-initial/verify-unit/stdout.log","xml":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g06-initial/verify-unit/xml"}
+{"kind":"executed","category":"integration","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","integration-test"],"environment":{},"started_at":"2026-08-28T03:27:03.477544+00:00","duration_seconds":4.744,"exit_code":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g06-initial/verify-integration/stdout.log","xml":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g06-initial/verify-integration/xml"}
+{"kind":"executed","category":"canonical","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","scenario-run","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G06.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/G06-result.json"],"environment":{},"started_at":"2026-08-28T03:27:08.223793+00:00","duration_seconds":1.2,"exit_code":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g06-initial/verify-canonical/stdout.log"}
diff --git a/evidence/G06-verification.md b/evidence/G06-verification.md
new file mode 100644
index 0000000..50bf1a1
--- /dev/null
+++ b/evidence/G06-verification.md
@@ -0,0 +1,20 @@
+# G06 — initial attempt
+
+START `8ba2f137f77fd19766c68f51f2df4cd242864dd7`; phase `phase-1`; profile `realtime-core`; spec `c1d62196ab76b55652f5d75a67514f8c6d8210ce`.
+Fixture SHA-256 `8ca33010c24f31bdcfca54493b4868c2a89c682ba85f3795a4a8713f7ffb76df` is unchanged. The user-authorized procedural spec transition preserves G01–G05 commits/evidence and the unchanged game contract.
+
+**Reproduction:** the resolved `./track unit-test --tests arena.RoomTest.frozenG06AuthorityAbuse` ran the actual old TCP/parser/owner/manual-clock path for all 18 attempts and 221 ticks. Exit **1**, 6.064s, one assertion failure. Alpha seq8 at tick4 was incorrectly accepted as the fifth attempt, then its tick5 retry returned `SEQUENCE_CONFLICT`. The pure four-invalid-direction fixture similarly admitted its fifth attempt and rejected the subsequent retry. Validation-counter evidence was explicitly unavailable before activation. All twelve prior main-source files matched START hashes; manifest, baseline harness/test copies, raw result/log/XML are preserved in `runs/g06-initial/reproduce-unit/`. Main was notified before production edits.
+
+**Already satisfied (NOT_REPRODUCED):** fixed movement and all four clamps; TAG distance2500/2501, cooldown at1/19/20 elapsed ticks, LEFT/self/other-Room isolation, ASCII shared-victim winner ordering, pending64/65th rejection, malformed direction isolation and clean shutdown. These production rules were not rewritten.
+
+**Change:** seventeen added core lines introduce an owner-confined, per-Player first-four validation guard after existing schema and transport identity checks. Attempts rejected by direction or sequence admission consume allowance; the fifth returns `INPUT_RATE_EXCEEDED` without committing sequence or pending input. The bounded counter saturates at four and resets on the next executed tick. Inactive/foreign actors cannot consume another Player's allowance. Existing parser/lifecycle precedence and the separate lower-level pending64 boundary remain intact. No dependency, prior test expectation, movement/TAG rule or future feature changed.
+
+**Verification:** exact argv/environment/timestamps/output paths are in `G06-command-ledger.jsonl`. Clean build exit0 (5.496s), full unit exit0 **38 tests** (4.233s), integration exit0 **4 tests** (4.744s), immutable main canonical exit0 (1.200s), output `G06-result.json`. No skipped tests or post-change failures/retries. Prior PARANOID buffer, ownership, clock, G05 sequence and process-shutdown regressions pass unchanged.
+
+Actual canonical: **16 ACCEPTED**; alpha seq2 at1 `MESSAGE_INVALID`, alpha seq8 at4 `INPUT_RATE_EXCEEDED`; retries at2 and5 accepted. TAG failures at2,3,201,219; successes at200,220. Executed221 ticks, final tick220, Room RUNNING; alpha/bravo both `(50000,50000)` and STOP, scores **2/0**, last accepted sequences **13/3**. Logical fields derive from actual replies/state; raw before/after state, 221 tick states, TAG error messages and owner counters are separate.
+
+Pure results: rate counters `[1,2,3,4,4]`, reset0, next-tick retry accepted; all prescribed clamp/range/LEFT/self/foreign cases pass; shared-victim winners `actor-a` then `actor-z`, with no cooldown committed for the losing attempt; 65th lower-level admission returns `INPUT_QUEUE_FULL` with all64 prior entries preserved and one selected movement. These state fixtures do not add a multi-room server or server-ID injection.
+
+Canonical high-water marks: input validation **4**, pending input **4**, mailbox/outbound **1/1**, parser bytes/capacity **343/512**. Shutdown: zero channels, connections, writes, mailbox, parser buffers/bytes, clock backlog and owned live threads; owner/event loops terminated, timer stopped, clients closed. Pure Room pending storage is cleared on close.
+
+Budget: **4 compiler tasks across 2 compile-bearing commands / 8**, **2 unit / 4** including reproduction, **1 integration / 2**, **1 post-canonical / 1**. Fault/load **0/0**. State hashes `NOT_ACTIVATED_G07`. Unresolved: **none**. G07 has not started; no G15+, external infrastructure or push.
diff --git a/src/main/java/arena/ArenaServer.java b/src/main/java/arena/ArenaServer.java
index ae89b77..283224a 100644
--- a/src/main/java/arena/ArenaServer.java
+++ b/src/main/java/arena/ArenaServer.java
@@ -261,6 +261,7 @@ public final class ArenaServer implements AutoCloseable {
                     if (session.playerId == null || !session.playerId.equals(Json.text(message, "player_id"))) {
                         peer.error("PLAYER_MISMATCH"); break;
                     }
+                    if (!room.beginInputValidation(session.playerId)) { peer.error("INPUT_RATE_EXCEEDED"); break; }
                     Room.Direction direction = Room.Direction.valueOf(Json.text(message, "direction"));
                     String target = message.path("tag_target_player_id").isNull() ? null : Json.text(message, "tag_target_player_id");
                     Room.Intent intent = new Room.Intent(Json.sequence(message), Json.targetTick(message), direction, target);
diff --git a/src/main/java/arena/AuthorityScenario.java b/src/main/java/arena/AuthorityScenario.java
new file mode 100644
index 0000000..6ba01a1
--- /dev/null
+++ b/src/main/java/arena/AuthorityScenario.java
@@ -0,0 +1,308 @@
+package arena;
+
+import com.fasterxml.jackson.databind.JsonNode;
+import com.fasterxml.jackson.databind.node.ArrayNode;
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import java.io.DataInputStream;
+import java.io.IOException;
+import java.lang.reflect.Field;
+import java.math.BigInteger;
+import java.util.ArrayList;
+import java.util.LinkedHashMap;
+import java.util.List;
+import java.util.Map;
+import java.util.TreeMap;
+import java.util.concurrent.ThreadPoolExecutor;
+import java.util.concurrent.TimeUnit;
+
+/** Frozen G06 observations from real admission/ticks and the prescribed pure state fixtures. */
+final class AuthorityScenario {
+    static final String SHA256 = "8ca33010c24f31bdcfca54493b4868c2a89c682ba85f3795a4a8713f7ffb76df";
+    private AuthorityScenario() { }
+
+    static ObjectNode run(ObjectNode scenario, String hash) throws Exception {
+        if (!SHA256.equals(hash)) throw new IOException("G06 requires frozen scenario bytes");
+        ObjectNode result = Json.MAPPER.createObjectNode().put("thread", "G06").put("contract_version", 1)
+                .put("scenario_sha256", hash).put("state_hashes", "NOT_ACTIVATED_G07")
+                .put("network_fault_runs", 0).put("load_runs", 0);
+        ObjectNode logical = result.putObject("logical");
+        ArrayNode admissions = logical.putArray("admissions"), tags = logical.putArray("tag_events");
+        ArrayNode events = result.putArray("events"), ticks = result.putArray("ticks");
+        ArrayNode tagResponses = result.putArray("tag_responses"), failures = result.putArray("failures");
+        ArenaServer server = new ArenaServer("127.0.0.1", 0, true);
+        TcpClient alpha = null, bravo = null;
+        try {
+            alpha = new TcpClient(server.port()); bravo = new TcpClient(server.port());
+            alpha.hello(); String room = alpha.createRoom(); alpha.join(room); bravo.hello(); bravo.join(room);
+            alpha.until("SNAPSHOT"); bravo.until("SNAPSHOT");
+            Map<String, TcpClient> clients = Map.of("alpha", alpha, "bravo", bravo);
+            Map<String, String> roles = Map.of(alpha.playerId, "alpha", bravo.playerId, "bravo");
+            result.putObject("issued_identifiers").put("room", room).put("alpha", alpha.playerId).put("bravo", bravo.playerId);
+            int eventIndex = 0;
+            for (int tick = 0; tick < scenario.path("ticks").asInt(); tick++) {
+                Map<String, String> selectedTags = new LinkedHashMap<>();
+                while (eventIndex < scenario.withArray("events").size()
+                        && scenario.withArray("events").get(eventIndex).path("before_tick").asInt() == tick) {
+                    JsonNode event = scenario.withArray("events").get(eventIndex++);
+                    String role = event.path("client").asText(); TcpClient client = clients.get(role);
+                    ObjectNode request = client.auth("INPUT", room);
+                    for (String key : List.of("seq", "target_tick", "direction", "owner_epoch")) request.set(key, event.path(key));
+                    String targetRole = event.path("tag_target_role").isNull() ? null : event.path("tag_target_role").asText();
+                    String target = targetRole == null ? null
+                            : (clients.containsKey(targetRole) ? clients.get(targetRole).playerId : targetRole);
+                    if (target == null) request.putNull("tag_target_player_id"); else request.put("tag_target_player_id", target);
+                    if (event.path("additional_ignored_fields").isObject())
+                        request.setAll((ObjectNode) event.path("additional_ignored_fields"));
+                    ObjectNode before = snapshot(server); client.send(request);
+                    ObjectNode response = response(client), after = snapshot(server);
+                    String code = response.path("type").asText().equals("INPUT_ACK")
+                            ? response.path("status").asText() : response.path("code").asText();
+                    admissions.addObject().put("client", role).put("seq", event.path("seq").asInt())
+                            .put("before_tick", tick).put("code", code);
+                    ObjectNode observed = events.addObject(); observed.set("request", request); observed.set("response", response);
+                    observed.set("before", before); observed.set("after", after);
+                    if (!code.equals("ACCEPTED") && !before.equals(after)) failures.add("rejected admission changed gameplay/pending");
+                    for (String id : roles.keySet()) {
+                        JsonNode p = player(before, id), q = player(after, id);
+                        for (String key : List.of("x", "y", "score", "direction", "last_tag_tick"))
+                            if (!p.path(key).equals(q.path(key))) failures.add("admission changed authoritative " + key);
+                    }
+                    String expected = tick == 1 ? "MESSAGE_INVALID"
+                            : (tick == 4 && event.path("seq").asInt() == 8 ? "INPUT_RATE_EXCEEDED" : "ACCEPTED");
+                    if (!code.equals(expected)) failures.add("admission " + role + "/" + event.path("seq") + " at " + tick + ": " + code);
+                    if (code.equals("ACCEPTED") && target != null) selectedTags.put(client.playerId, target);
+                }
+                ObjectNode beforeTick = snapshot(server); server.advanceTicks(1);
+                ObjectNode afterTick = snapshot(server); ticks.add(afterTick);
+                for (var tag : selectedTags.entrySet()) {
+                    JsonNode before = player(beforeTick, tag.getKey()), after = player(afterTick, tag.getKey());
+                    int delta = after.path("score").asInt() - before.path("score").asInt();
+                    String outcome;
+                    if (delta == 1) outcome = "TAG_SUCCESS";
+                    else {
+                        ObjectNode response = response(clients.get(roles.get(tag.getKey()))); tagResponses.add(response);
+                        outcome = response.path("code").asText();
+                        if (delta != 0 || !outcome.equals("ACTION_REJECTED")
+                                || !before.path("last_tag_tick").equals(after.path("last_tag_tick")))
+                            failures.add("rejected TAG changed score/cooldown or wrong event");
+                    }
+                    tags.addObject().put("tick", afterTick.path("tick").asInt()).put("actor", roles.get(tag.getKey()))
+                            .put("target", roles.getOrDefault(tag.getValue(), tag.getValue())).put("result", outcome);
+                }
+                int movedX = Math.min(tick + 1, 100) * 400, movedY = Math.clamp(tick - 99, 0, 100) * 400;
+                JsonNode a = player(afterTick, alpha.playerId), b = player(afterTick, bravo.playerId);
+                if (a.path("x").asInt() != 10000 + movedX || a.path("y").asInt() != 10000 + movedY
+                        || b.path("x").asInt() != 90000 - movedX || b.path("y").asInt() != 90000 - movedY)
+                    failures.add("fixed movement at " + tick);
+            }
+            ObjectNode state = snapshot(server); logical.put("executed_ticks", state.path("executed_ticks").asInt());
+            ObjectNode positions = logical.putObject("final_positions"), scores = logical.putObject("scores");
+            ObjectNode sequences = logical.putObject("last_accepted_sequences"), directions = logical.putObject("final_directions");
+            for (String role : List.of("alpha", "bravo")) {
+                JsonNode p = player(state, clients.get(role).playerId);
+                positions.putArray(role).add(p.path("x").asInt()).add(p.path("y").asInt());
+                scores.set(role, p.path("score")); sequences.set(role, p.path("last_accepted_seq")); directions.set(role, p.path("direction"));
+            }
+            logical.set("room_status", state.path("status")); result.set("runtime_metrics", server.metrics());
+            result.set("input_validation_high_water", validationHighWater(server));
+            if (result.path("input_validation_high_water").asInt(-1) != 4) failures.add("first-four validation high-water unavailable/wrong");
+            expect(tags, "[{\"tick\":2,\"actor\":\"alpha\",\"target\":\"foreign-player\",\"result\":\"ACTION_REJECTED\"},"
+                    + "{\"tick\":3,\"actor\":\"alpha\",\"target\":\"bravo\",\"result\":\"ACTION_REJECTED\"},"
+                    + "{\"tick\":200,\"actor\":\"alpha\",\"target\":\"bravo\",\"result\":\"TAG_SUCCESS\"},"
+                    + "{\"tick\":201,\"actor\":\"alpha\",\"target\":\"bravo\",\"result\":\"ACTION_REJECTED\"},"
+                    + "{\"tick\":219,\"actor\":\"alpha\",\"target\":\"bravo\",\"result\":\"ACTION_REJECTED\"},"
+                    + "{\"tick\":220,\"actor\":\"alpha\",\"target\":\"bravo\",\"result\":\"TAG_SUCCESS\"}]", failures, "TAG events");
+            expect(positions, "{\"alpha\":[50000,50000],\"bravo\":[50000,50000]}", failures, "final positions");
+            expect(scores, "{\"alpha\":2,\"bravo\":0}", failures, "scores");
+            expect(sequences, "{\"alpha\":13,\"bravo\":3}", failures, "last sequences");
+            expect(directions, "{\"alpha\":\"STOP\",\"bravo\":\"STOP\"}", failures, "directions");
+            if (state.path("executed_ticks").asInt() != 221 || state.path("tick").asInt() != 220
+                    || !state.path("status").asText().equals("RUNNING")) failures.add("final clock/lifecycle");
+            server.close(); alpha.expectClosed(); bravo.expectClosed();
+        } finally {
+            try { if (alpha != null) alpha.close(); }
+            finally { try { if (bravo != null) bravo.close(); } finally { server.close(); } }
+        }
+        ScenarioRunner.assertCleanup(server.cleanup()); result.set("cleanup", server.cleanup());
+        logical.put("all_resources_released", alpha != null && bravo != null && alpha.isClosed() && bravo.isClosed());
+        result.put("passed", failures.isEmpty()); return result;
+    }
+
+    static ObjectNode supplemental(ObjectNode scenario) throws Exception {
+        ObjectNode result = Json.MAPPER.createObjectNode(); ArrayNode failures = result.putArray("failures");
+        JsonNode fixed = scenario.path("supplemental_unit_probes");
+        ArrayNode clamps = result.putArray("clamp");
+        for (JsonNode cell : fixed.withArray("clamp")) {
+            Room room = runningRoom(); Room.Player actor = room.player("player-a");
+            actor.x = cell.path("x").asInt(); actor.y = cell.path("y").asInt();
+            ObjectNode observed = clamps.addObject(); observed.set("before", view(room));
+            String code = room.accept(actor.id, intent(1, 0, cell.path("direction").asText(), null));
+            room.tick(); observed.put("admission", admission(code)); observed.set("after", view(room));
+            if (code != null || actor.x != cell.path("expected").get(0).asInt() || actor.y != cell.path("expected").get(1).asInt())
+                failures.add("clamp " + cell.path("direction"));
+            close(room);
+        }
+        ArrayNode ranges = result.putArray("tag_range");
+        for (JsonNode distance : fixed.withArray("tag_range_units")) {
+            Room room = runningRoom(); colocate(room); room.player("player-b").x += distance.asInt();
+            ObjectNode observed = ranges.addObject().put("range_units", distance.asInt()); observed.set("before", view(room));
+            String code = room.accept("player-a", intent(1, 0, "STOP", "player-b"));
+            List<Room.Rejection> rejected = room.tick(); observed.put("admission", admission(code)); observed.set("after", view(room));
+            observed.set("action_rejections", rejections(rejected));
+            boolean success = distance.asInt() == 2500;
+            if (code != null || room.player("player-a").score != (success ? 1 : 0)
+                    || room.player("player-a").lastTagTick != (success ? 0 : -20)
+                    || (success ? !rejected.isEmpty() : !actionRejected(rejected, "player-a"))) failures.add("range boundary " + distance);
+            close(room);
+        }
+        ArrayNode rejectedTags = result.putArray("tag_rejections");
+        for (JsonNode name : fixed.withArray("tag_rejections")) {
+            Room room = runningRoom(); colocate(room); Room other = null; ObjectNode otherBefore = null;
+            String target = "player-b", label = name.asText();
+            if (label.equals("actor LEFT")) room.left("player-a");
+            else if (label.equals("target LEFT")) room.left("player-b");
+            else if (label.equals("self-target")) target = "player-a";
+            else {
+                other = new Room("independent-room"); other.join("foreign-a"); other.join("foreign-b");
+                target = other.player("foreign-a").id; otherBefore = view(other);
+            }
+            ObjectNode observed = rejectedTags.addObject().put("case", label); ObjectNode before = view(room); observed.set("before", before);
+            String code = room.accept("player-a", intent(1, 0, "STOP", target));
+            observed.put("admission", admission(code)); observed.set("after_admission", view(room));
+            if (code != null && !before.equals(view(room))) failures.add("LEFT admission mutated state");
+            List<Room.Rejection> rejected = room.tick(); observed.set("action_rejections", rejections(rejected)); observed.set("after", view(room));
+            if (room.player("player-a").score != 0 || room.player("player-a").lastTagTick != -20
+                    || (label.equals("actor LEFT") ? !"PLAYER_MISMATCH".equals(code) || !rejected.isEmpty()
+                    : code != null || !actionRejected(rejected, "player-a"))) failures.add("TAG rejection " + label);
+            if (other != null) {
+                observed.set("other_room_before", otherBefore); observed.set("other_room_after", view(other));
+                if (!otherBefore.equals(view(other)) || room.player(target) != null) failures.add("foreign membership isolation");
+                close(other);
+            }
+            close(room);
+        }
+        JsonNode shared = fixed.path("one_target_per_tick"); Room room = new Room("shared-target");
+        room.join(shared.path("player_ids").get(0).asText()); room.join(shared.path("player_ids").get(1).asText());
+        // Pure initialization only: a third player is inserted after automatic two-player RUNNING transition.
+        Field playersField = Room.class.getDeclaredField("players"); playersField.setAccessible(true);
+        Object players = playersField.get(room);
+        if (!(players instanceof TreeMap<?, ?>)) throw new IOException("unexpected Room player storage");
+        @SuppressWarnings("unchecked") Map<String, Room.Player> initializedPlayers = (Map<String, Room.Player>) players;
+        String third = shared.path("player_ids").get(2).asText(); initializedPlayers.put(third, new Room.Player(third, 2));
+        colocate(room); ObjectNode oneTarget = result.putObject("one_target_per_tick"); oneTarget.set("before", view(room));
+        for (JsonNode actor : shared.withArray("tick0_actors"))
+            if (room.accept(actor.asText(), intent(1, 0, "STOP", shared.path("target").asText())) != null) failures.add("shared target admission");
+        oneTarget.set("tick0_rejections", rejections(room.tick())); oneTarget.set("after_tick0", view(room));
+        String firstWinner = winner(room, 0); oneTarget.put("tick0_winner", firstWinner);
+        if (!firstWinner.equals(shared.path("expected_tick0_winner").asText()) || room.player("actor-z").lastTagTick != -20)
+            failures.add("ASCII actor order or losing cooldown");
+        for (JsonNode actor : shared.withArray("tick1_actors"))
+            if (room.accept(actor.asText(), intent(2, 1, "STOP", shared.path("target").asText())) != null) failures.add("shared target retry admission");
+        oneTarget.set("tick1_rejections", rejections(room.tick())); oneTarget.set("after_tick1", view(room));
+        String secondWinner = winner(room, 1); oneTarget.put("tick1_winner", secondWinner);
+        if (!secondWinner.equals(shared.path("expected_tick1_winner").asText()) || room.player("target").score != 0)
+            failures.add("losing actor retry");
+        close(room);
+
+        room = runningRoom(); ObjectNode rate = result.putObject("first_four_attempts"); ArrayNode attempts = rate.putArray("attempts");
+        for (JsonNode attempt : fixed.path("first_four_attempts").withArray("attempts")) {
+            ObjectNode before = view(room); String code = pureAdmission(room, attempt.path("seq").asInt(), 0, attempt.path("direction").asText());
+            ObjectNode observed = attempts.addObject().put("code", code); observed.set("before", before); observed.set("after", view(room));
+            observed.set("validation_attempts", optionalInteger(room.player("player-a"), "validationAttempts"));
+            String expected = attempt.path("seq").asInt() == 5 ? fixed.path("first_four_attempts").path("fifth_code").asText() : "MESSAGE_INVALID";
+            if (!code.equals(expected) || !before.equals(view(room))
+                    || observed.path("validation_attempts").asInt(-1) != Math.min(attempt.path("seq").asInt(), 4))
+                failures.add("first-four attempt " + attempt.path("seq") + ": " + code);
+        }
+        room.tick(); rate.set("after_tick0", view(room));
+        rate.set("validation_attempts_after_tick0", optionalInteger(room.player("player-a"), "validationAttempts"));
+        String retry = pureAdmission(room, fixed.path("first_four_attempts").path("next_tick_retry_seq").asInt(), 1, "EAST");
+        rate.put("next_tick_retry", retry); room.tick(); rate.set("after_tick1", view(room));
+        rate.set("validation_high_water", optionalInteger(room, "validationHighWater"));
+        if (!retry.equals("ACCEPTED") || room.player("player-a").x != 10400
+                || !room.player("player-a").lastAcceptedSeq.equals(BigInteger.valueOf(5))
+                || rate.path("validation_attempts_after_tick0").asInt(-1) != 0
+                || rate.path("validation_high_water").asInt(-1) != 4) failures.add("rate reset/sequence preservation");
+        close(room);
+
+        room = runningRoom(); JsonNode bound = fixed.path("pending_bound_after_rate_activation");
+        ObjectNode capacity = result.putObject("pending_bound");
+        for (int seq = 1; seq <= bound.path("prepopulate_existing_pending").asInt(); seq++)
+            if (room.accept("player-a", intent(seq, 0, "EAST", null)) != null) failures.add("lower admission prepopulate " + seq);
+        ObjectNode before = view(room); List<Room.Intent> entries = new ArrayList<>(room.player("player-a").pending);
+        String code = room.accept("player-a", intent(bound.path("next_unique_seq").asInt(), 0, "EAST", null));
+        capacity.put("code", admission(code)); capacity.set("before", before); capacity.set("after", view(room));
+        if (!bound.path("expected_code").asText().equals(code) || !before.equals(view(room))
+                || !entries.equals(new ArrayList<>(room.player("player-a").pending))) failures.add("pending64 preservation");
+        room.tick(); capacity.set("after_tick", view(room));
+        if (room.player("player-a").x != 10400 || !room.player("player-a").pending.isEmpty()) failures.add("pending64 selected effect");
+        close(room);
+        result.put("all_resources_released", true).put("passed", failures.isEmpty()); return result;
+    }
+
+    private static String pureAdmission(Room room, int seq, int tick, String direction) {
+        if (!room.beginInputValidation("player-a")) return "INPUT_RATE_EXCEEDED";
+        try { return admission(room.accept("player-a", intent(seq, tick, direction, null))); }
+        catch (IllegalArgumentException invalid) { return "MESSAGE_INVALID"; }
+    }
+    private static Room.Intent intent(int seq, int tick, String direction, String target) {
+        return new Room.Intent(BigInteger.valueOf(seq), BigInteger.valueOf(tick), Room.Direction.valueOf(direction), target);
+    }
+    private static String admission(String code) { return code == null ? "ACCEPTED" : code; }
+    private static Room runningRoom() { Room room = new Room("room-unit"); room.join("player-a"); room.join("player-b"); return room; }
+    private static void colocate(Room room) {
+        for (JsonNode p : room.view("SNAPSHOT").withArray("players")) { Room.Player player = room.player(p.path("player_id").asText()); player.x = 50000; player.y = 50000; }
+    }
+    private static String winner(Room room, int tick) {
+        for (JsonNode p : room.view("SNAPSHOT").withArray("players"))
+            if (room.player(p.path("player_id").asText()).lastTagTick == tick) return p.path("player_id").asText();
+        return "NONE";
+    }
+    private static boolean actionRejected(List<Room.Rejection> rejected, String actor) {
+        return rejected.size() == 1 && rejected.getFirst().playerId().equals(actor) && rejected.getFirst().code().equals("ACTION_REJECTED");
+    }
+    private static ArrayNode rejections(List<Room.Rejection> rejected) {
+        ArrayNode result = Json.MAPPER.createArrayNode();
+        for (Room.Rejection item : rejected) result.addObject().put("actor", item.playerId()).put("code", item.code());
+        return result;
+    }
+    private static ObjectNode view(Room room) {
+        ObjectNode state = room.view("SNAPSHOT");
+        for (JsonNode p : state.withArray("players")) {
+            Room.Player player = room.player(p.path("player_id").asText());
+            ((ObjectNode) p).put("pending_inputs", player.pending.size()).put("last_tag_tick", player.lastTagTick);
+        }
+        return state;
+    }
+    private static void close(Room room) throws IOException {
+        room.close();
+        for (JsonNode p : room.view("SNAPSHOT").withArray("players"))
+            if (!room.player(p.path("player_id").asText()).pending.isEmpty()) throw new IOException("pure pending cleanup");
+        if (room.status() != Room.Status.CLOSED) throw new IOException("pure Room cleanup");
+    }
+    private static ObjectNode snapshot(ArenaServer server) throws Exception {
+        return ((ThreadPoolExecutor) field(server, "owner")).submit(() -> view((Room) field(server, "room"))).get(5, TimeUnit.SECONDS);
+    }
+    private static JsonNode validationHighWater(ArenaServer server) throws Exception {
+        return ((ThreadPoolExecutor) field(server, "owner")).submit(() -> optionalInteger(field(server, "room"), "validationHighWater")).get(5, TimeUnit.SECONDS);
+    }
+    private static JsonNode optionalInteger(Object object, String name) throws ReflectiveOperationException {
+        try { return Json.MAPPER.valueToTree(field(object, name)); }
+        catch (NoSuchFieldException unavailableBeforeActivation) { return Json.MAPPER.nullNode(); }
+    }
+    private static Object field(Object object, String name) throws ReflectiveOperationException {
+        Field field = object.getClass().getDeclaredField(name); field.setAccessible(true); return field.get(object);
+    }
+    private static ObjectNode response(TcpClient client) throws Exception {
+        DataInputStream input = (DataInputStream) field(client, "input"); int length = input.readInt();
+        if (length < 1 || length > CompleteFrame.MAX_BYTES) throw new IOException("server frame bound");
+        byte[] bytes = new byte[length]; input.readFully(bytes); return Json.read(bytes);
+    }
+    private static JsonNode player(ObjectNode state, String id) throws IOException {
+        for (JsonNode p : state.withArray("players")) if (p.path("player_id").asText().equals(id)) return p;
+        throw new IOException("missing authoritative player");
+    }
+    private static void expect(JsonNode actual, String expected, ArrayNode failures, String label) {
+        if (!actual.toString().equals(expected)) failures.add(label);
+    }
+}
diff --git a/src/main/java/arena/Room.java b/src/main/java/arena/Room.java
index 195748c..97839cc 100644
--- a/src/main/java/arena/Room.java
+++ b/src/main/java/arena/Room.java
@@ -14,6 +14,7 @@ final class Room {
     static final int DURATION = 1_200;
     static final int INPUT_LIMIT = 64;
     static final int MAX_FUTURE_TICKS = 4;
+    static final int VALIDATION_LIMIT = 4;
     static final int[][] SPAWNS = {
         {10_000, 10_000}, {90_000, 90_000}, {10_000, 90_000}, {90_000, 10_000},
         {50_000, 10_000}, {50_000, 90_000}, {10_000, 50_000}, {90_000, 50_000}
@@ -37,6 +38,7 @@ final class Room {
         BigInteger lastAcceptedSeq = BigInteger.ZERO;
         Intent lastAcceptedIntent;
         BigInteger appliedSeq;
+        int validationAttempts;
 
         Player(String id, int slot) {
             this.id = id; this.slot = slot;
@@ -59,6 +61,7 @@ final class Room {
     private int executedTicks;
     private int nextSlot;
     private int inputHighWater;
+    private int validationHighWater;
 
     Room(String id) { this.id = id; }
     void assertOwner() {
@@ -80,6 +83,18 @@ final class Room {
         return player;
     }
 
+    /** Called after transport identity checks, before direction/sequence admission validation. */
+    boolean beginInputValidation(String id) {
+        assertOwner();
+        Player player = players.get(id);
+        // Preserve earlier lifecycle/schema error precedence; inactive actors consume no allowance.
+        if (status != Status.RUNNING || player == null || !player.connected) return true;
+        if (player.validationAttempts == VALIDATION_LIMIT) return false;
+        player.validationAttempts++;
+        validationHighWater = Math.max(validationHighWater, player.validationAttempts);
+        return true;
+    }
+
     String accept(String id, Intent intent) {
         assertOwner();
         if (status != Status.RUNNING) return "ROOM_NOT_RUNNING";
@@ -105,6 +120,7 @@ final class Room {
         TreeMap<String, String> tags = new TreeMap<>();
         BigInteger currentTick = BigInteger.valueOf(executedTicks);
         for (Player player : players.values()) {
+            player.validationAttempts = 0;
             Intent selected = null;
             player.appliedSeq = null;
             var pending = player.pending.iterator();
diff --git a/src/main/java/arena/ScenarioRunner.java b/src/main/java/arena/ScenarioRunner.java
index 79cb438..2537429 100644
--- a/src/main/java/arena/ScenarioRunner.java
+++ b/src/main/java/arena/ScenarioRunner.java
@@ -44,6 +44,14 @@ final class ScenarioRunner {
             } catch (IOException failure) { throw failure; }
             catch (Exception failure) { throw new IOException("G05 input scenario failed", failure); }
         }
+        if (scenario.path("thread").asText().equals("G06")) {
+            try {
+                ObjectNode result = AuthorityScenario.run(scenario, sha256(scenarioBytes));
+                if (!result.path("passed").asBoolean()) throw new IOException("G06 assertions: " + result.path("failures"));
+                return result;
+            } catch (IOException failure) { throw failure; }
+            catch (Exception failure) { throw new IOException("G06 authority scenario failed", failure); }
+        }
         if (!scenario.path("thread").asText().equals("G01") || scenario.path("contract_version").asInt() != 1
                 || !scenario.path("clock").path("kind").asText().equals("manual")
                 || scenario.path("clock").path("tick_duration_ms").asInt() != 50
diff --git a/src/test/java/arena/RoomTest.java b/src/test/java/arena/RoomTest.java
index f7e7a04..83ee60e 100644
--- a/src/test/java/arena/RoomTest.java
+++ b/src/test/java/arena/RoomTest.java
@@ -12,6 +12,22 @@ import java.util.concurrent.TimeUnit;
 import org.junit.jupiter.api.Test;
 
 final class RoomTest {
+    @Test void frozenG06AuthorityAbuse() throws Exception {
+        byte[] bytes;
+        String scenarioPath = System.getenv("ARENA_G06_SCENARIO");
+        if (scenarioPath != null) bytes = Files.readAllBytes(Path.of(scenarioPath));
+        else try (var input = getClass().getResourceAsStream("/G06.json")) { assertNotNull(input); bytes = input.readAllBytes(); }
+        String hash = HexFormat.of().formatHex(MessageDigest.getInstance("SHA-256").digest(bytes));
+        assertEquals(AuthorityScenario.SHA256, hash);
+        ObjectNode scenario = Json.read(bytes), result = AuthorityScenario.run(scenario, hash);
+        ObjectNode supplemental = AuthorityScenario.supplemental(scenario); result.set("supplemental", supplemental);
+        String output = System.getenv("ARENA_G06_EVIDENCE");
+        if (output != null) Files.write(Path.of(output), Json.MAPPER.writerWithDefaultPrettyPrinter().writeValueAsBytes(result));
+        System.out.println("G06 authority " + result.path("logical"));
+        assertTrue(result.path("passed").asBoolean() && supplemental.path("passed").asBoolean(),
+                result.path("failures") + " " + supplemental.path("failures"));
+    }
+
     @Test void frozenG05InputOrder() throws Exception {
         byte[] bytes;
         String scenarioPath = System.getenv("ARENA_G05_SCENARIO");
diff --git a/src/test/resources/G06.json b/src/test/resources/G06.json
new file mode 100644
index 0000000..a6ba2cf
--- /dev/null
+++ b/src/test/resources/G06.json
@@ -0,0 +1,291 @@
+{
+  "scenario_id": "G06-authority-abuse",
+  "contract_version": 1,
+  "thread": "G06",
+  "seed": 7050,
+  "clock": {
+    "kind": "manual",
+    "tick_duration_ms": 50
+  },
+  "clients": [
+    "alpha",
+    "bravo"
+  ],
+  "ticks": 221,
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
+      "before_tick": 0,
+      "client": "bravo",
+      "seq": 1,
+      "target_tick": 0,
+      "direction": "WEST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 1,
+      "client": "alpha",
+      "seq": 2,
+      "target_tick": 1,
+      "direction": "NORTH_EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 2,
+      "client": "alpha",
+      "seq": 2,
+      "target_tick": 2,
+      "direction": "EAST",
+      "tag_target_role": "foreign-player",
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 3,
+      "client": "alpha",
+      "seq": 3,
+      "target_tick": 3,
+      "direction": "EAST",
+      "tag_target_role": "bravo",
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 4,
+      "client": "alpha",
+      "seq": 4,
+      "target_tick": 4,
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
+      "before_tick": 4,
+      "client": "alpha",
+      "seq": 6,
+      "target_tick": 4,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 4,
+      "client": "alpha",
+      "seq": 7,
+      "target_tick": 4,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 4,
+      "client": "alpha",
+      "seq": 8,
+      "target_tick": 4,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 5,
+      "client": "alpha",
+      "seq": 8,
+      "target_tick": 5,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0,
+      "additional_ignored_fields": {
+        "position": {
+          "x": 999999,
+          "y": 999999
+        },
+        "score": 999,
+        "elapsed_time_ms": 999999
+      }
+    },
+    {
+      "before_tick": 100,
+      "client": "alpha",
+      "seq": 9,
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
+      "seq": 10,
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
+      "seq": 11,
+      "target_tick": 201,
+      "direction": "STOP",
+      "tag_target_role": "bravo",
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 219,
+      "client": "alpha",
+      "seq": 12,
+      "target_tick": 219,
+      "direction": "STOP",
+      "tag_target_role": "bravo",
+      "owner_epoch": 0
+    },
+    {
+      "before_tick": 220,
+      "client": "alpha",
+      "seq": 13,
+      "target_tick": 220,
+      "direction": "STOP",
+      "tag_target_role": "bravo",
+      "owner_epoch": 0
+    }
+  ],
+  "foreign_target": "canonical TCP case uses syntactically valid foreign-player absent from this Room; pure two-Room fixture proves actual other-Room membership",
+  "supplemental_unit_probes": {
+    "tag_range_units": [
+      2500,
+      2501
+    ],
+    "clamp": [
+      {
+        "x": 99900,
+        "y": 50000,
+        "direction": "EAST",
+        "expected": [
+          100000,
+          50000
+        ]
+      },
+      {
+        "x": 100,
+        "y": 50000,
+        "direction": "WEST",
+        "expected": [
+          0,
+          50000
+        ]
+      },
+      {
+        "x": 50000,
+        "y": 99900,
+        "direction": "NORTH",
+        "expected": [
+          50000,
+          100000
+        ]
+      },
+      {
+        "x": 50000,
+        "y": 100,
+        "direction": "SOUTH",
+        "expected": [
+          50000,
+          0
+        ]
+      }
+    ],
+    "tag_rejections": [
+      "actor LEFT",
+      "target LEFT",
+      "self-target",
+      "target in independent other-Room model"
+    ],
+    "one_target_per_tick": {
+      "player_ids": [
+        "actor-z",
+        "target",
+        "actor-a"
+      ],
+      "initialization": "pure state fixture, reverse insertion vs ASCII actor order; all in range, STOP, connected",
+      "target": "target",
+      "tick0_actors": [
+        "actor-z",
+        "actor-a"
+      ],
+      "tick1_actors": [
+        "actor-z"
+      ],
+      "expected_tick0_winner": "actor-a",
+      "expected_tick1_winner": "actor-z"
+    },
+    "first_four_attempts": {
+      "before_tick": 0,
+      "attempts": [
+        {
+          "seq": 1,
+          "direction": "NORTH_EAST"
+        },
+        {
+          "seq": 2,
+          "direction": "NORTH_EAST"
+        },
+        {
+          "seq": 3,
+          "direction": "NORTH_EAST"
+        },
+        {
+          "seq": 4,
+          "direction": "NORTH_EAST"
+        },
+        {
+          "seq": 5,
+          "direction": "EAST"
+        }
+      ],
+      "fifth_code": "INPUT_RATE_EXCEEDED",
+      "next_tick_retry_seq": 5
+    },
+    "pending_bound_after_rate_activation": {
+      "prepopulate_existing_pending": 64,
+      "next_unique_seq": 65,
+      "expected_code": "INPUT_QUEUE_FULL",
+      "preserve_all_previous_entries": true,
+      "test_boundary": "existing lower-level pending admission, not64 network attempts insideone tick"
+    }
+  },
+  "socket_ceiling_ms": 5000
+}
