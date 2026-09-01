# Connection, Session, Player와 Room Ownership (G03)

## `test: verify identity lifecycle and room ownership`

diff --git a/TRACK.md b/TRACK.md
index af98141..2e3404f 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -1,6 +1,6 @@
-# Java arena — through G02
+# Java arena — through G03
 
-Current thread: G02 (G01 baseline retained). Profile: realtime-core. Spec revision: `5a6e4a2f8fc71d4be18c3279583bfc2558d5c232`.
+Current thread: G03 (G01/G02 regressions retained). Profile: realtime-core. Spec revision: `5a6e4a2f8fc71d4be18c3279583bfc2558d5c232`.
 
 ## Frozen versions
 
@@ -21,6 +21,7 @@ The wrapper uses the locally installed Temurin path when JAVA_HOME is unset. On
 ./track integration-test    # real loopback tests, timer/executor cleanup and bounded CLI SIGTERM
 ./track scenario-run /absolute/path/to/G01.json /absolute/path/to/result.json
 ./track scenario-run /absolute/path/to/G02.json /absolute/path/to/framing-evidence.json
+./track scenario-run /absolute/path/to/G03.json /absolute/path/to/identity-evidence.json
 ./track replay-verify /absolute/path/to/replay /absolute/path/to/evidence
 ./track server config/server.json
 ```
@@ -33,7 +34,7 @@ Connection lifetime belongs to its non-sharable Netty channel handler. Each acce
 
 Parser outcomes distinguish NEED_MORE_BYTES, COMPLETE_VALID_MESSAGE, MESSAGE_ERROR, TERMINAL_FRAME_ERROR and IO_END. Transport end reasons distinguish clean EOF, partial EOF, framing close and I/O error. Length 0 or >16,384 disables further reads and attempts FRAME_SIZE_INVALID before closing. Complete malformed messages remain recoverable. Strict UTF-8 decoding, duplicate-key detection, object-root enforcement, trailing-token rejection and active G02 schema checks occur before Room handoff. Missing/wrongly typed v/type are MESSAGE_INVALID, integer v other than 1 is PROTOCOL_VERSION_UNSUPPORTED, and unknown types are MESSAGE_TYPE_UNKNOWN. Unknown fields on known messages remain ignored. No sequence, target tick or later message schema is activated.
 
-Session registry and Room state belong to one dedicated room-owner thread. Network callbacks submit to its `ArrayBlockingQueue(1024)` and never mutate a Room. Each Room public operation checks the constructing owner thread; unit tests reject mutation from another thread. There is one room and at most eight accepted connections. UUID identifiers are server-generated, distinct from connection objects, and not input authority. Detailed lifecycle and identity matrices remain G03 work.
+Session registry and Room state belong to one dedicated room-owner thread. Network callbacks submit to its `ArrayBlockingQueue(1024)` and never mutate a Room. Each Room public operation checks the constructing owner thread; unit tests reject mutation from another thread. There is one room and at most eight accepted connections. Netty's channel ID is the process-local Connection identifier; separate server-generated UUIDs identify Session, Player and Room. No client chooses these IDs. The G03 runner observes actual issued values and checks the contract's ASCII/length form without fixed-ID injection.
 
 Each player's pending input storage holds at most 64 intents and rejects overflow with `INPUT_QUEUE_FULL`. An owner tick drains that bounded storage, selects the latest pending direction/TAG, moves players in ASCII ID order, then evaluates one-shot TAG with 64-bit squared distance. Direction persists; TAG does not. No seq, target tick or rate-limit contract is activated. Player data is integer only; unknown position/score fields are ignored.
 
@@ -51,7 +52,23 @@ The G01 canonical runner reads all clients, setup steps, input boundaries, direc
 
 G02 reads the supplied scenario's messages, fragmentation matrix, coalescing indices, malformed bytes and socket deadline. Exact read boundaries run through the same production handler in Netty EmbeddedChannel; the four malformed cases run over real loopback TCP while a separate connection continues HELLO/WELCOME. The test resource is an exact, SHA-256-checked copy of main's frozen G02 scenario, not an independently adjusted fixture. The pre-change unit run demonstrated seven failures and three all-at-once passes before production edits. Partial EOF, strict JSON forms, maximum frame capacity and transport I/O classification have fixed supplemental unit coverage.
 
-G02 initial ceilings: build/compile <=8, unit <=4 including pre-change reproduction, integration <=2, and one post-change canonical run because reproduction used a unit suite. Actual compile-bearing commands: 2 (one reproduction test compilation and one clean build); unit invocations: 2, integration: 1, post-change canonical: 1. All outputs/XML were preserved before clean builds could overwrite them. Network-fault and load runs are zero. G03 identity/lifecycle matrices and every later feature remain inactive.
+G02 initial ceilings: build/compile <=8, unit <=4 including pre-change reproduction, integration <=2, and one post-change canonical run because reproduction used a unit suite. Actual compile-bearing commands: 2 (one reproduction test compilation and one clean build); unit invocations: 2, integration: 1, post-change canonical: 1. All outputs/XML were preserved before clean builds could overwrite them. Network-fault and load runs are zero.
+
+## G03 identity and lifecycle regression
+
+The actual unchanged G02 server passed the fixed G03 reproduction: **NOT_REPRODUCED**. G03 adds the shared scenario runner, focused tests and evidence; it does not refactor production identity or ownership classes. The immutable scenario SHA is checked in the runner and in the exact test resource. The baseline reads main's actual scenario path through `ARENA_G03_SCENARIO`; ordinary portable unit runs use the identical resource.
+
+| Operation | Permission and effect |
+|---|---|
+| Create | A valid session creates the absent single Room; an existing Room gives `ROOM_NOT_JOINABLE`. |
+| Join | A new valid session in LOBBY receives the next stable slot/spawn. Duplicate join and joins after RUNNING give `ROOM_NOT_JOINABLE`; the direct Room test covers RUNNING, FINISHED and CLOSED without state changes. |
+| Foreign identity INPUT | A foreign session gives `SESSION_INVALID`; own session plus another Player gives `PLAYER_MISMATCH`, with unchanged state and pending input. |
+| Leave or connection loss | In each of LOBBY, RUNNING and FINISHED, alpha becomes LEFT; identity, slot, position, score, tick and bravo remain unchanged. Reconnect is inactive. No voluntary-leave response/connection-close requirement is imposed. |
+| Shutdown | Registry, pending input/control/mailbox work, parser buffers, channels and owned threads/timer are released. A private bounded CLOSED model may remain for evidence; terminated ownership rejects further simulation. |
+
+G03 holds the real owner consumer with deterministic latches, sends exactly one TCP EAST input and observes unchanged state/pending input before release, one accepted pending intent after drain without a tick, then `(10400,10000)` after one manual tick. A separate pure production `enqueue` probe makes exactly 1,025 admissions with capacity 1,024 held: the last attempt emits terminal `INPUT_QUEUE_FULL`, admitted work drains exactly once, and the real reply ByteBuf reaches reference count zero. Reflection is confined to the scenario harness for observing the pre-existing private owner boundary; no Room mutation bypass or replacement server is introduced. Existing PARANOID parser, 64-input/control limits, foreign-owner and process-shutdown checks remain active.
+
+Exact commands and raw-output locations are in `evidence/G03-command-ledger.jsonl`; `evidence/G03-verification.md` contains the compact result and budget summary. G04 and later features remain inactive.
 
 G01 initial budget: build/compile <=8, unit suites <=4, integration suites <=2, canonical scenario <=1; network-fault and load runs exactly zero. Main has its own separately frozen one-build/one-unit/one-integration/one-scenario verification budget. No test sleep, microbenchmark, fuzzing, replay, UDP, reconnect, many-room or distributed implementation is included.
 
diff --git a/evidence/G03-command-ledger.jsonl b/evidence/G03-command-ledger.jsonl
new file mode 100644
index 0000000..ed8df09
--- /dev/null
+++ b/evidence/G03-command-ledger.jsonl
@@ -0,0 +1,6 @@
+{"kind":"resolved_before_baseline","category":"unit-reproduction","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","unit-test","--tests","arena.RoomTest.frozenG03IdentityLifecycle"],"environment":{"ARENA_G03_SCENARIO":"/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G03.json","ARENA_G03_EVIDENCE":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g03-initial/reproduce-unit/result.json"},"resolved_at":"2026-08-28T01:58:43.384506+00:00","output_directory":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g03-initial/reproduce-unit"}
+{"kind":"executed","category":"unit-reproduction","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","unit-test","--tests","arena.RoomTest.frozenG03IdentityLifecycle"],"environment":{"ARENA_G03_SCENARIO":"/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G03.json","ARENA_G03_EVIDENCE":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g03-initial/reproduce-unit/result.json"},"started_at":"2026-08-28T01:59:36.010233+00:00","duration_seconds":11.109,"exit_code":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g03-initial/reproduce-unit/stdout.log","xml":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g03-initial/reproduce-unit/xml"}
+{"kind":"executed","category":"build","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","build"],"environment":{},"started_at":"2026-08-28T02:04:44.454420+00:00","duration_seconds":18.324,"exit_code":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g03-initial/verify-build/stdout.log"}
+{"kind":"executed","category":"unit","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","unit-test"],"environment":{"ARENA_G03_SCENARIO":"/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G03.json","ARENA_G03_EVIDENCE":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g03-initial/verify-unit/result.json"},"started_at":"2026-08-28T02:05:02.780247+00:00","duration_seconds":9.846,"exit_code":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g03-initial/verify-unit/stdout.log","xml":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g03-initial/verify-unit/xml"}
+{"kind":"executed","category":"integration","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","integration-test"],"environment":{},"started_at":"2026-08-28T02:05:12.631640+00:00","duration_seconds":8.287,"exit_code":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g03-initial/verify-integration/stdout.log","xml":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g03-initial/verify-integration/xml"}
+{"kind":"executed","category":"canonical","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","scenario-run","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G03.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/G03-result.json"],"environment":{},"started_at":"2026-08-28T02:05:20.922232+00:00","duration_seconds":1.316,"exit_code":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g03-initial/verify-canonical/stdout.log"}
diff --git a/evidence/G03-verification.md b/evidence/G03-verification.md
new file mode 100644
index 0000000..8038103
--- /dev/null
+++ b/evidence/G03-verification.md
@@ -0,0 +1,21 @@
+# G03 — initial attempt
+
+Profile `realtime-core`; spec `5a6e4a2f8fc71d4be18c3279583bfc2558d5c232`.
+START `c85f0db9afeba06a75c275f3fe88cc0032faf5e7`.
+Frozen scenario SHA-256 `d3cdc4dac5c0054847329dcf0b56b408ba5f30f95ca0e5f85a7da914fc3e0d62`; the test resource has identical bytes.
+
+## Reproduction and scope
+
+**NOT_REPRODUCED.** The command resolved before baseline was `./track unit-test --tests arena.RoomTest.frozenG03IdentityLifecycle`, using `ARENA_G03_SCENARIO` for the actual immutable main scenario path. It ran the unchanged G02 core and passed, exit 0 (11.109s). Main inspected the actual source diff, raw JSON and XML before acknowledging the gate. Only scenario/test/evidence work followed; ArenaServer, Room, parser, runtime and dependency locks remain unchanged.
+
+The shared harness uses real loopback/parser/owner paths. Reflection only observes or holds the existing owner boundary. The fixed identity errors were `ROOM_NOT_JOINABLE`, `SESSION_INVALID` and `PLAYER_MISMATCH`, without state or pending-input changes. Slots/spawns remained 0/(10000,10000) and 1/(90000,90000). All six fresh lifecycle cells changed only alpha to LEFT; tick, identity, score, position and bravo remained unchanged. FINISHED was held after exactly 1,200 manual ticks. Voluntary leave has no required response or TCP-close assertion.
+
+The consumer-held EAST probe observed pending input 0 and unchanged Room state while one network intent waited; owner drain admitted one pending input without ticking; one manual tick produced (10400,10000), pending 0. The real Room rejected foreign-thread mutation. The pure production mailbox probe made exactly 1,025 admissions with unchanged capacity 1,024: 1,024 waited, none executed while held, and the overflow emitted terminal `INPUT_QUEUE_FULL`. Its actual ByteBuf reached refcount 0; exactly 1,024 admitted tasks executed after release and the queue emptied. This was not a load campaign.
+
+## Verification and bounds
+
+All exact argv, environments, timestamps, durations, exits and raw-output paths are in `G03-command-ledger.jsonl`. Baseline stdout/JSON/XML were preserved in `runs/g03-initial/reproduce-unit/` before the clean build. Final results: `./track build` exit 0 (18.324s); `./track unit-test` exit 0, **35 tests** (9.846s); `./track integration-test` exit 0, **4 tests** (8.287s); the canonical `./track scenario-run` of main's actual G03 file exit 0 (1.316s), output `G03-result.json`. No test errors/skips, failed verification commands or verification retries occurred.
+
+All 11 canonical server cleanups had zero channels, connections, sessions, pending writes/inputs/mailbox work, parser buffers/allocated bytes and live owned threads; owner/event loops terminated and no timer/client socket remained. A private bounded CLOSED model is allowed historical state; post-close simulation was rejected. Observed canonical high-water marks: mailbox **1024**, pending input **1**, outbound **2**, parser bytes/capacity **231/256**. Existing PARANOID reference-count checks, parser maximum **16388**, pending-input bound **64**, outbound bound **64**, foreign-owner rejection and real SIGTERM/listener-rebind assertions remain active.
+
+Budget: **4 compiler tasks across 2 compile-bearing commands / 8**, **2 unit / 4** including reproduction, **1 integration / 2**, **1 post-change canonical / 1**; no pre-change canonical. Fault/load runs **0/0**. State hashes `NOT_ACTIVATED_G07`. No G04 or later implementation. Unresolved: **none**.
diff --git a/src/main/java/arena/IdentityScenario.java b/src/main/java/arena/IdentityScenario.java
new file mode 100644
index 0000000..efdb232
--- /dev/null
+++ b/src/main/java/arena/IdentityScenario.java
@@ -0,0 +1,468 @@
+package arena;
+
+import com.fasterxml.jackson.databind.JsonNode;
+import com.fasterxml.jackson.databind.node.ArrayNode;
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import io.netty.buffer.ByteBuf;
+import io.netty.channel.Channel;
+import io.netty.channel.ChannelHandlerContext;
+import io.netty.channel.ChannelInboundHandlerAdapter;
+import io.netty.channel.embedded.EmbeddedChannel;
+import java.io.DataInputStream;
+import java.io.IOException;
+import java.lang.reflect.Field;
+import java.lang.reflect.InvocationTargetException;
+import java.net.InetSocketAddress;
+import java.net.Socket;
+import java.nio.ByteBuffer;
+import java.util.ArrayList;
+import java.util.HashSet;
+import java.util.List;
+import java.util.Map;
+import java.util.concurrent.Callable;
+import java.util.concurrent.CountDownLatch;
+import java.util.concurrent.ThreadPoolExecutor;
+import java.util.concurrent.TimeUnit;
+import java.util.concurrent.atomic.AtomicInteger;
+import java.util.concurrent.atomic.AtomicReference;
+
+/** G03 evidence harness. Reflection only observes/holds the unchanged production owner boundary. */
+final class IdentityScenario {
+    static final String SHA256 = "d3cdc4dac5c0054847329dcf0b56b408ba5f30f95ca0e5f85a7da914fc3e0d62";
+    private final ObjectNode scenario;
+    private final ArrayNode failures = Json.MAPPER.createArrayNode();
+
+    private IdentityScenario(ObjectNode scenario) { this.scenario = scenario; }
+
+    static ObjectNode run(ObjectNode scenario, String hash) throws IOException {
+        if (!SHA256.equals(hash) || !scenario.path("thread").asText().equals("G03"))
+            throw new IOException("G03 requires its frozen scenario bytes");
+        return new IdentityScenario(scenario).run(hash);
+    }
+
+    private ObjectNode run(String hash) {
+        ObjectNode evidence = Json.MAPPER.createObjectNode().put("thread", "G03").put("contract_version", 1)
+                .put("scenario_id", scenario.path("scenario_id").asText()).put("scenario_sha256", hash)
+                .put("state_hashes", "NOT_ACTIVATED_G07").put("network_fault_runs", 0).put("load_runs", 0);
+        ArrayNode identities = evidence.putArray("identity_cases");
+        for (JsonNode cell : scenario.withArray("identity_cases"))
+            collect(identities.addObject().put("name", cell.path("name").asText()), out -> identity(cell, out));
+        ArrayNode lifecycle = evidence.putArray("lifecycle_cases");
+        for (JsonNode state : scenario.withArray("lifecycle_states"))
+            for (JsonNode action : scenario.withArray("lifecycle_actions")) {
+                ObjectNode out = lifecycle.addObject().put("state", state.asText()).put("action", action.asText());
+                collect(out, value -> lifecycle(state.asText(), action.asText(), value));
+            }
+        collect(evidence.putObject("owner_probe"), this::ownerProbe);
+        collect(evidence.putObject("mailbox_probe"), this::mailboxProbe);
+        evidence.set("failures", failures);
+        evidence.put("passed", failures.isEmpty());
+        return evidence;
+    }
+
+    @FunctionalInterface private interface Probe { void run(ObjectNode out) throws Exception; }
+    private void collect(ObjectNode out, Probe probe) {
+        try { probe.run(out); }
+        catch (Exception failure) {
+            out.put("exception", failure.toString());
+            failures.add(out.path("name").asText(out.path("state").asText("probe")) + ": " + failure);
+        }
+    }
+    private void check(boolean ok, String description) { if (!ok) failures.add(description); }
+
+    private void identity(JsonNode cell, ObjectNode out) throws Exception {
+        World world = new World();
+        try {
+            String name = cell.path("name").asText();
+            world.setup(name.equals("duplicate-join") ? 1 : 2);
+            ObjectNode before = world.snapshot();
+            out.set("before", world.normalize(before));
+            Client sender;
+            ObjectNode request;
+            String expected;
+            if (name.equals("duplicate-join")) {
+                sender = world.alpha;
+                request = sender.auth("JOIN_ROOM", world.roomId);
+                expected = "ROOM_NOT_JOINABLE";
+            } else {
+                sender = world.client(cell.path("sender").asText());
+                request = sender.auth("INPUT", world.roomId)
+                        .put("session_id", world.client(cell.path("session_from").asText()).sessionId)
+                        .put("player_id", world.client(cell.path("player_from").asText()).playerId)
+                        .put("direction", cell.path("direction").asText()).putNull("tag_target_player_id");
+                expected = name.equals("foreign-session") ? "SESSION_INVALID" : "PLAYER_MISMATCH";
+            }
+            sender.send(request);
+            ObjectNode reply = sender.until("ERROR");
+            out.put("error_code", reply.path("code").asText());
+            check(reply.path("code").asText().equals(expected), name + " error code");
+            ObjectNode after = world.snapshot();
+            out.set("after_rejection", world.normalize(after));
+            check(before.equals(after), name + " changed authoritative state or pending input");
+            if (name.equals("duplicate-join")) {
+                world.joinBravo();
+                ObjectNode joined = world.snapshot();
+                out.set("after_bravo_join", world.normalize(joined));
+                check(joined.path("players").size() == 2, "duplicate join consumed an extra player");
+                check(world.player(joined, world.alpha).path("slot").asInt(-1) == 0, "alpha stable slot");
+                check(world.player(joined, world.bravo).path("slot").asInt(-1) == 1, "bravo stable slot");
+                check(world.player(joined, world.alpha).path("x").asInt() == 10_000
+                        && world.player(joined, world.alpha).path("y").asInt() == 10_000
+                        && world.player(joined, world.bravo).path("x").asInt() == 90_000
+                        && world.player(joined, world.bravo).path("y").asInt() == 90_000, "stable slot spawns");
+            }
+            world.recordIdentifiers(out);
+        } finally { world.finish(out); }
+    }
+
+    private void lifecycle(String state, String action, ObjectNode out) throws Exception {
+        World world = new World();
+        try {
+            world.setup(scenario.path(state.toLowerCase(java.util.Locale.ROOT) + "_player_count").asInt());
+            if (state.equals("FINISHED")) {
+                world.server.advanceTicks(scenario.path("finish_ticks").asInt());
+                out.put("alpha_finished_message", world.alpha.until("ROOM_FINISHED").path("status").asText());
+                out.put("bravo_finished_message", world.bravo.until("ROOM_FINISHED").path("status").asText());
+            }
+            ObjectNode before = world.snapshot();
+            check(before.path("status").asText().equals(state), "lifecycle setup " + state);
+            out.set("before", world.normalize(before));
+            // Existing-room create and repeated join reject without changing the cell's setup.
+            world.alpha.send(world.alpha.auth("CREATE_ROOM", null));
+            out.put("create_existing_code", world.alpha.until("ERROR").path("code").asText());
+            check(out.path("create_existing_code").asText().equals("ROOM_NOT_JOINABLE"), state + " create permission");
+            world.alpha.send(world.alpha.auth("JOIN_ROOM", world.roomId));
+            out.put("join_code", world.alpha.until("ERROR").path("code").asText());
+            check(out.path("join_code").asText().equals("ROOM_NOT_JOINABLE"), state + " join permission");
+            check(before.equals(world.snapshot()), state + " rejected create/join changed state");
+            Object peer = world.peer(world.alpha);
+            Channel channel = (Channel) field(peer, "channel");
+            CountDownLatch delivered = new CountDownLatch(1);
+            channel.eventLoop().submit(() -> {
+                CompleteFrame framing = channel.pipeline().get(CompleteFrame.class);
+                int messagesBefore = framing.state().path("messages").asInt();
+                String parser = channel.pipeline().context(framing).name();
+                channel.pipeline().addBefore(parser, "g03-lifecycle-observation", new ChannelInboundHandlerAdapter() {
+                    @Override public void channelRead(ChannelHandlerContext context, Object message) {
+                        context.fireChannelRead(message);
+                        if (action.equals("LEAVE_ROOM") && framing.state().path("messages").asInt() > messagesBefore)
+                            delivered.countDown();
+                    }
+                    @Override public void channelInactive(ChannelHandlerContext context) {
+                        context.fireChannelInactive();
+                        if (action.equals("CONNECTION_CLOSE")) delivered.countDown();
+                    }
+                });
+            }).syncUninterruptibly();
+            if (action.equals("LEAVE_ROOM")) world.alpha.send(world.alpha.auth("LEAVE_ROOM", world.roomId));
+            else world.alpha.close();
+            // Parser return or actual channelInactive queues the intent before the owner observation.
+            // Voluntary leave need not close the control connection or emit a particular response.
+            await(delivered);
+            if (action.equals("CONNECTION_CLOSE"))
+                check(!channel.isOpen(), "connection-close must close transport");
+            ObjectNode after = world.snapshot();
+            out.set("after", world.normalize(after));
+            ObjectNode expected = before.deepCopy();
+            ((ObjectNode) world.player(expected, world.alpha)).put("connectivity", "LEFT").put("direction", "STOP");
+            check(after.equals(expected), state + "/" + action + " must only mark alpha LEFT");
+            out.put("connection_closed", !channel.isOpen());
+            int remainingSessions = onOwner(world.server, () -> sessions(world.server).size());
+            out.put("sessions_after_action", remainingSessions);
+        } finally { world.finish(out); }
+    }
+
+    private void ownerProbe(ObjectNode out) throws Exception {
+        World world = new World();
+        CountDownLatch release = new CountDownLatch(1);
+        try {
+            world.setup(2);
+            Client sender = world.client(scenario.path("owner_probe").path("sender").asText());
+            Channel channel = (Channel) field(world.peer(sender), "channel");
+            CountDownLatch parsed = new CountDownLatch(1);
+            channel.eventLoop().submit(() -> {
+                CompleteFrame framing = channel.pipeline().get(CompleteFrame.class);
+                int messagesBefore = framing.state().path("messages").asInt();
+                String parser = channel.pipeline().context(framing).name();
+                channel.pipeline().addBefore(parser, "g03-observe-parser-return", new ChannelInboundHandlerAdapter() {
+                    @Override public void channelRead(ChannelHandlerContext context, Object message) {
+                        context.fireChannelRead(message);
+                        if (framing.state().path("messages").asInt() > messagesBefore) parsed.countDown();
+                    }
+                });
+            }).syncUninterruptibly();
+            CountDownLatch held = new CountDownLatch(1);
+            CountDownLatch observed = new CountDownLatch(1);
+            AtomicReference<ObjectNode> before = new AtomicReference<>();
+            AtomicReference<ObjectNode> beforeRelease = new AtomicReference<>();
+            var barrier = owner(world.server).submit(() -> {
+                before.set(world.snapshotOnOwner());
+                held.countDown();
+                await(parsed);
+                beforeRelease.set(world.snapshotOnOwner());
+                observed.countDown();
+                await(release);
+                return null;
+            });
+            await(held);
+            sender.send(sender.auth("INPUT", world.roomId)
+                    .put("direction", scenario.path("owner_probe").path("direction").asText())
+                    .putNull("tag_target_player_id"));
+            await(observed);
+            out.set("before", world.normalize(before.get()));
+            out.set("before_consumer_release", world.normalize(beforeRelease.get()));
+            out.put("queued_network_intents", owner(world.server).getQueue().size());
+            check(before.get().equals(beforeRelease.get()), "network callback mutated Room while consumer held");
+            check(out.path("queued_network_intents").asInt() == 1, "exactly one owner intent queued");
+            release.countDown();
+            barrier.get(5, TimeUnit.SECONDS);
+            out.put("ack", sender.until("INPUT_ACK").path("status").asText());
+            ObjectNode drained = world.snapshot();
+            out.set("after_drain_without_tick", world.normalize(drained));
+            check(drained.path("executed_ticks").asInt(-1) == 0 && drained.path("pending_inputs").asInt(-1) == 1,
+                    "owner drain must admit exactly one pending input without a tick");
+            check(world.player(drained, sender).path("x").asInt() == 10_000, "mailbox drain moved player");
+            world.server.advanceTicks(scenario.path("owner_probe").path("execute_ticks_after_drain").asInt());
+            ObjectNode ticked = world.snapshot();
+            out.set("after_tick", world.normalize(ticked));
+            check(world.player(ticked, sender).path("x").asInt() == 10_400
+                    && world.player(ticked, sender).path("y").asInt() == 10_000
+                    && ticked.path("pending_inputs").asInt(-1) == 0, "one owner tick movement/drain");
+            boolean rejected = false;
+            try { ((Room) field(world.server, "room")).tick(); }
+            catch (IllegalStateException foreignOwner) { rejected = true; }
+            out.put("foreign_mutation_rejected", rejected);
+            check(rejected && ticked.equals(world.snapshot()), "foreign owner rejection changed state");
+        } finally { release.countDown(); world.finish(out); }
+    }
+
+    private void mailboxProbe(ObjectNode out) throws Exception {
+        ArenaServer server = new ArenaServer("127.0.0.1", 0, true);
+        EmbeddedChannel channel = new EmbeddedChannel();
+        CountDownLatch release = new CountDownLatch(1);
+        try {
+            int capacity = scenario.path("mailbox_probe").path("capacity_by_track").path("industry-java").asInt();
+            check(ArenaServer.MAILBOX_LIMIT == capacity, "mailbox capacity changed");
+            CountDownLatch held = new CountDownLatch(1);
+            var barrier = owner(server).submit(() -> { held.countDown(); await(release); return null; });
+            await(held);
+            Class<?> peerType = Class.forName("arena.ArenaServer$Peer");
+            var constructor = peerType.getDeclaredConstructor(ArenaServer.class, Channel.class);
+            constructor.setAccessible(true);
+            Object peer = constructor.newInstance(server, channel);
+            var admission = ArenaServer.class.getDeclaredMethod("enqueue", peerType, Runnable.class);
+            admission.setAccessible(true);
+            AtomicInteger executed = new AtomicInteger();
+            CountDownLatch drained = new CountDownLatch(capacity);
+            for (int i = 0; i < capacity + 1; i++) {
+                Runnable command = () -> { executed.incrementAndGet(); drained.countDown(); };
+                invoke(admission, server, peer, command);
+            }
+            out.put("capacity", capacity).put("admission_attempts", capacity + 1)
+                    .put("queued_while_held", owner(server).getQueue().size())
+                    .put("executed_while_held", executed.get());
+            ByteBuf reply = channel.readOutbound();
+            if (reply == null) throw new IOException("mailbox overflow lacks explicit reply");
+            try {
+                int length = reply.readInt();
+                byte[] bytes = new byte[length];
+                reply.readBytes(bytes);
+                ObjectNode error = Json.read(bytes);
+                out.put("overflow_code", error.path("code").asText());
+                check(error.path("code").asText().equals("INPUT_QUEUE_FULL"), "mailbox overflow code");
+            } finally { reply.release(); }
+            out.put("overflow_buffer_ref_count", reply.refCnt()).put("overflow_connection_closed", !channel.isOpen());
+            check(reply.refCnt() == 0 && !channel.isOpen(), "mailbox overflow transport/buffer disposal");
+            check(owner(server).getQueue().size() == capacity && executed.get() == 0, "mailbox hold/bound");
+            Object extra = channel.readOutbound();
+            check(extra == null, "one overflow reply only");
+            io.netty.util.ReferenceCountUtil.release(extra);
+            release.countDown();
+            barrier.get(5, TimeUnit.SECONDS);
+            await(drained);
+            onOwner(server, () -> null);
+            out.put("executed_after_drain", executed.get()).put("queue_after_drain", owner(server).getQueue().size());
+            check(executed.get() == capacity && owner(server).getQueue().isEmpty(), "mailbox accepted work/drain");
+        } finally {
+            release.countDown();
+            channel.finishAndReleaseAll();
+            server.close();
+            cleanup(server, out);
+        }
+    }
+
+    private void cleanup(ArenaServer server, ObjectNode out) throws Exception {
+        ObjectNode cleanup = server.cleanup();
+        ScenarioRunner.assertCleanup(cleanup);
+        cleanup.put("sessions_remaining", sessions(server).size());
+        Room retained = (Room) field(server, "room");
+        cleanup.put("mutable_room_retained", retained != null);
+        int pending = 0;
+        if (retained != null) {
+            Map<?, ?> players = (Map<?, ?>) field(retained, "players");
+            for (Object value : players.values()) pending += ((Room.Player) value).pending.size();
+        }
+        cleanup.put("pending_inputs_remaining", pending);
+        check(sessions(server).isEmpty() && pending == 0, "terminal session/input cleanup");
+        if (retained != null) {
+            int ticks = (int) field(retained, "executedTicks");
+            int count = ((Map<?, ?>) field(retained, "players")).size();
+            cleanup.put("retained_room_status", field(retained, "status").toString()).put("retained_player_records", count);
+            boolean rejected = false;
+            try { server.advanceTicks(1); }
+            catch (java.util.concurrent.RejectedExecutionException closedOwner) { rejected = true; }
+            cleanup.put("post_close_simulation_rejected", rejected);
+            check(field(retained, "status") == Room.Status.CLOSED && count <= Room.SPAWNS.length
+                    && rejected && (int) field(retained, "executedTicks") == ticks,
+                    "closed historical state must be bounded and unable to simulate");
+        }
+        out.set("cleanup", cleanup);
+    }
+
+    private final class World {
+        final ArenaServer server = new ArenaServer("127.0.0.1", 0, true);
+        final List<Client> clients = new ArrayList<>();
+        final Client alpha;
+        final Client bravo;
+        String roomId;
+
+        World() throws IOException {
+            try {
+                alpha = new Client(server.port()); clients.add(alpha);
+                bravo = new Client(server.port()); clients.add(bravo);
+            } catch (IOException failure) { for (Client client : clients) client.close(); server.close(); throw failure; }
+        }
+        Client client(String role) throws IOException {
+            return switch (role) { case "alpha" -> alpha; case "bravo" -> bravo; default -> throw new IOException("unknown role"); };
+        }
+        void setup(int count) throws Exception {
+            alpha.hello();
+            alpha.send(alpha.auth("CREATE_ROOM", null));
+            roomId = Json.text(alpha.until("ROOM_CREATED"), "room_id");
+            alpha.join(roomId);
+            if (count == 2) joinBravo();
+            else if (count != 1) throw new IOException("unfrozen player count");
+        }
+        void joinBravo() throws IOException {
+            bravo.hello(); bravo.join(roomId);
+            alpha.until("SNAPSHOT"); bravo.until("SNAPSHOT");
+        }
+        Object peer(Client client) throws Exception {
+            return onOwner(server, () -> {
+                for (var entry : sessions(server).entrySet())
+                    if (field(entry.getValue(), "id").equals(client.sessionId)) return entry.getKey();
+                throw new IOException("missing server-issued session");
+            });
+        }
+        ObjectNode snapshot() throws Exception { return onOwner(server, this::snapshotOnOwner); }
+        ObjectNode snapshotOnOwner() throws ReflectiveOperationException {
+            Room room = (Room) field(server, "room");
+            ObjectNode view = room.view("SNAPSHOT");
+            int pending = 0;
+            for (Client client : clients) if (client.playerId != null) pending += room.player(client.playerId).pending.size();
+            return view.put("pending_inputs", pending);
+        }
+        JsonNode player(ObjectNode snapshot, Client client) throws IOException {
+            for (JsonNode value : snapshot.withArray("players"))
+                if (value.path("player_id").asText().equals(client.playerId)) return value;
+            throw new IOException("missing player in authority");
+        }
+        ObjectNode normalize(ObjectNode snapshot) throws IOException {
+            ObjectNode result = snapshot.deepCopy();
+            result.put("room_id", "room");
+            ArrayNode players = result.putArray("players");
+            for (String role : List.of("alpha", "bravo")) {
+                Client client = client(role);
+                if (client.playerId != null) {
+                    ObjectNode view = ((ObjectNode) player(snapshot, client)).deepCopy();
+                    view.remove("player_id"); view.put("role", role); players.add(view);
+                }
+            }
+            return result;
+        }
+        void recordIdentifiers(ObjectNode out) throws Exception {
+            ObjectNode identifiers = out.putObject("issued_identifiers");
+            identifiers.put("room", roomId);
+            for (String role : List.of("alpha", "bravo")) {
+                Client client = client(role);
+                identifiers.put(role + "_session", client.sessionId).put(role + "_player", client.playerId);
+                Channel channel = (Channel) field(peer(client), "channel");
+                identifiers.put(role + "_connection", channel.id().asLongText());
+            }
+            HashSet<String> seen = new HashSet<>();
+            identifiers.elements().forEachRemaining(id -> check(id.asText().matches("[A-Za-z0-9_-]{1,64}")
+                    && seen.add(id.asText()), "server-issued IDs must be valid and distinct"));
+        }
+        void finish(ObjectNode out) throws Exception {
+            for (Client client : clients) client.close();
+            server.close();
+            cleanup(server, out);
+            out.put("client_sockets_closed", clients.stream().allMatch(client -> client.socket.isClosed()));
+            check(out.path("client_sockets_closed").asBoolean(), "client socket cleanup");
+        }
+    }
+
+    private static final class Client implements AutoCloseable {
+        final Socket socket = new Socket();
+        final DataInputStream input;
+        String sessionId;
+        String playerId;
+        Client(int port) throws IOException {
+            try {
+                socket.connect(new InetSocketAddress("127.0.0.1", port), 5_000);
+                socket.setTcpNoDelay(true); socket.setSoTimeout(5_000);
+                input = new DataInputStream(socket.getInputStream());
+            } catch (IOException failure) { socket.close(); throw failure; }
+        }
+        void send(ObjectNode request) throws IOException {
+            byte[] payload = Json.bytes(request);
+            socket.getOutputStream().write(ByteBuffer.allocate(payload.length + 4).putInt(payload.length).put(payload).array());
+            socket.getOutputStream().flush();
+        }
+        ObjectNode until(String type) throws IOException {
+            for (int i = 0; i < 64; i++) {
+                int length = input.readInt();
+                if (length < 1 || length > CompleteFrame.MAX_BYTES) throw new IOException("server frame bound");
+                byte[] bytes = new byte[length]; input.readFully(bytes);
+                ObjectNode reply = Json.read(bytes);
+                if (reply.path("type").asText().equals(type)) return reply;
+                if (reply.path("type").asText().equals("ERROR")) throw new IOException("unexpected server error: " + reply);
+            }
+            throw new IOException("client response bound");
+        }
+        ObjectNode auth(String type, String room) {
+            ObjectNode request = Json.message(type).put("session_id", sessionId);
+            if (room != null) request.put("room_id", room);
+            if (playerId != null) request.put("player_id", playerId);
+            return request;
+        }
+        void hello() throws IOException { send(Json.message("HELLO")); sessionId = Json.text(until("WELCOME"), "session_id"); }
+        void join(String room) throws IOException { send(auth("JOIN_ROOM", room)); playerId = Json.text(until("ROOM_JOINED"), "player_id"); }
+        @Override public void close() throws IOException { socket.close(); }
+    }
+
+    private static Object field(Object object, String name) throws ReflectiveOperationException {
+        Field field = object.getClass().getDeclaredField(name);
+        field.setAccessible(true);
+        return field.get(object);
+    }
+    private static Map<?, ?> sessions(ArenaServer server) throws ReflectiveOperationException {
+        return (Map<?, ?>) field(server, "sessions");
+    }
+    private static ThreadPoolExecutor owner(ArenaServer server) throws ReflectiveOperationException {
+        return (ThreadPoolExecutor) field(server, "owner");
+    }
+    private static <T> T onOwner(ArenaServer server, Callable<T> action) throws Exception {
+        return owner(server).submit(action).get(5, TimeUnit.SECONDS);
+    }
+    private static Object invoke(java.lang.reflect.Method method, Object receiver, Object... arguments) throws Exception {
+        try { return method.invoke(receiver, arguments); }
+        catch (InvocationTargetException failure) {
+            if (failure.getCause() instanceof Exception cause) throw cause;
+            throw failure;
+        }
+    }
+    private static void await(CountDownLatch barrier) throws InterruptedException, IOException {
+        if (!barrier.await(5, TimeUnit.SECONDS)) throw new IOException("deterministic barrier deadline");
+    }
+}
diff --git a/src/main/java/arena/ScenarioRunner.java b/src/main/java/arena/ScenarioRunner.java
index 62a496c..72c4737 100644
--- a/src/main/java/arena/ScenarioRunner.java
+++ b/src/main/java/arena/ScenarioRunner.java
@@ -23,6 +23,11 @@ final class ScenarioRunner {
         ObjectNode scenario = Json.read(scenarioBytes);
         if (scenario.path("thread").asText().equals("G02"))
             return FramingScenario.run(scenario, sha256(scenarioBytes));
+        if (scenario.path("thread").asText().equals("G03")) {
+            ObjectNode result = IdentityScenario.run(scenario, sha256(scenarioBytes));
+            if (!result.path("passed").asBoolean()) throw new IOException("G03 assertions: " + result.path("failures"));
+            return result;
+        }
         if (!scenario.path("thread").asText().equals("G01") || scenario.path("contract_version").asInt() != 1
                 || !scenario.path("clock").path("kind").asText().equals("manual")
                 || scenario.path("clock").path("tick_duration_ms").asInt() != 50
diff --git a/src/test/java/arena/RoomTest.java b/src/test/java/arena/RoomTest.java
index bd409bb..6ac211d 100644
--- a/src/test/java/arena/RoomTest.java
+++ b/src/test/java/arena/RoomTest.java
@@ -1,12 +1,35 @@
 package arena;
 
 import static org.junit.jupiter.api.Assertions.*;
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.security.MessageDigest;
+import java.util.HexFormat;
 import java.util.concurrent.ExecutionException;
 import java.util.concurrent.FutureTask;
 import java.util.concurrent.TimeUnit;
 import org.junit.jupiter.api.Test;
 
 final class RoomTest {
+    @Test void frozenG03IdentityLifecycle() throws Exception {
+        byte[] bytes;
+        String scenarioPath = System.getenv("ARENA_G03_SCENARIO");
+        if (scenarioPath != null) bytes = Files.readAllBytes(Path.of(scenarioPath));
+        else try (var input = getClass().getResourceAsStream("/G03.json")) {
+            assertNotNull(input);
+            bytes = input.readAllBytes();
+        }
+        String hash = HexFormat.of().formatHex(MessageDigest.getInstance("SHA-256").digest(bytes));
+        assertEquals(IdentityScenario.SHA256, hash);
+        ObjectNode result = IdentityScenario.run(Json.read(bytes), hash);
+        String evidencePath = System.getenv("ARENA_G03_EVIDENCE");
+        if (evidencePath != null)
+            Files.write(Path.of(evidencePath), Json.MAPPER.writerWithDefaultPrettyPrinter().writeValueAsBytes(result));
+        System.out.println("G03 identity/lifecycle " + result);
+        assertTrue(result.path("passed").asBoolean(), result.path("failures").toString());
+    }
+
     private Room runningRoom() {
         Room room = new Room("room-unit");
         room.join("player-a");
@@ -63,4 +86,17 @@ final class RoomTest {
         assertFalse(foreign.isAlive());
         assertEquals(0, room.executedTicks());
     }
+
+    @Test void joinPermissionClosesAtRunningAndNeverReopens() {
+        Room room = runningRoom();
+        for (Room.Status state : new Room.Status[] {Room.Status.RUNNING, Room.Status.FINISHED, Room.Status.CLOSED}) {
+            if (state == Room.Status.FINISHED) for (int i = 0; i < Room.DURATION; i++) room.tick();
+            if (state == Room.Status.CLOSED) room.close();
+            assertEquals(state, room.status());
+            ObjectNode before = room.view("SNAPSHOT");
+            IllegalStateException rejected = assertThrows(IllegalStateException.class, () -> room.join("player-new"));
+            assertEquals("ROOM_NOT_JOINABLE", rejected.getMessage());
+            assertEquals(before, room.view("SNAPSHOT"));
+        }
+    }
 }
diff --git a/src/test/resources/G03.json b/src/test/resources/G03.json
new file mode 100644
index 0000000..bb83021
--- /dev/null
+++ b/src/test/resources/G03.json
@@ -0,0 +1,94 @@
+{
+  "scenario_id": "G03-identity-lifecycle",
+  "contract_version": 1,
+  "thread": "G03",
+  "seed": 7050,
+  "clock": {
+    "kind": "manual",
+    "tick_duration_ms": 50
+  },
+  "finish_ticks": 1200,
+  "identity_cases": [
+    {
+      "name": "duplicate-join",
+      "clients": [
+        "alpha",
+        "bravo"
+      ],
+      "steps": [
+        "alpha-hello-create-join",
+        "alpha-join-again",
+        "bravo-hello-join"
+      ]
+    },
+    {
+      "name": "foreign-session",
+      "clients": [
+        "alpha",
+        "bravo"
+      ],
+      "sender": "bravo",
+      "session_from": "alpha",
+      "player_from": "alpha",
+      "direction": "EAST"
+    },
+    {
+      "name": "foreign-player",
+      "clients": [
+        "alpha",
+        "bravo"
+      ],
+      "sender": "bravo",
+      "session_from": "bravo",
+      "player_from": "alpha",
+      "direction": "EAST"
+    }
+  ],
+  "lifecycle_states": [
+    "LOBBY",
+    "RUNNING",
+    "FINISHED"
+  ],
+  "lifecycle_actions": [
+    "LEAVE_ROOM",
+    "CONNECTION_CLOSE"
+  ],
+  "lobby_player_count": 1,
+  "running_player_count": 2,
+  "finished_player_count": 2,
+  "owner_check": "network intent becomes visible only at owner mailbox drain; foreign mutation rejected",
+  "socket_ceiling_ms": 5000,
+  "identity_setup": "Each identity case uses a fresh server. Foreign-session/player cases complete HELLO/create/join for alpha and HELLO/join for bravo in the same RUNNING room, without executing a tick, before the spoof message.",
+  "lifecycle_fixture_isolation": "Each of the six state/action cells uses a fresh server and fresh server-issued identities; actor is alpha. All direction values remain STOP; no INPUT is sent in lifecycle cells.",
+  "finished_observation": "Run exactly finish_ticks manual ticks, observe FINISHED and ROOM_FINISHED, then pause the manual clock before the leave/close action and before explicit server shutdown.",
+  "leave_fields": [
+    "session_id",
+    "room_id",
+    "player_id"
+  ],
+  "owner_probe": {
+    "clients": [
+      "alpha",
+      "bravo"
+    ],
+    "sender": "alpha",
+    "direction": "EAST",
+    "tag_target_player_id": null,
+    "target_state": "RUNNING",
+    "before_tick": 0,
+    "observe_before_consumer_release": true,
+    "drain_without_tick": true,
+    "execute_ticks_after_drain": 1
+  },
+  "mailbox_probe": {
+    "kind": "pure production mailbox test with consumer held; no network load campaign",
+    "capacity_by_track": {
+      "fundamentals-cpp": 512,
+      "industry-java": 1024
+    },
+    "admission_attempts": "exactly capacity plus one",
+    "overflow": "INPUT_QUEUE_FULL or documented connection rejection through the existing bounded path; never silently accepted",
+    "drain_and_cleanup": true
+  },
+  "terminal_cleanup": "Close all clients and server explicitly after observations; assert zero active connections, pending input/control/mailbox items, parser buffers, descriptors or owned threads/timers. Historical immutable player views may remain only as bounded evidence."
+}
