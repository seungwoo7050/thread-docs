# Fixed Tick, Drift와 Catch-up Bound (G04)

## `feat: add bounded monotonic fixed clock`

diff --git a/TRACK.md b/TRACK.md
index 2e3404f..3f9396a 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -1,6 +1,6 @@
-# Java arena — through G03
+# Java arena — through G04
 
-Current thread: G03 (G01/G02 regressions retained). Profile: realtime-core. Spec revision: `5a6e4a2f8fc71d4be18c3279583bfc2558d5c232`.
+Current thread: G04 (G01–G03 regressions retained). Profile: realtime-core. Spec revision: `5a6e4a2f8fc71d4be18c3279583bfc2558d5c232`.
 
 ## Frozen versions
 
@@ -22,6 +22,7 @@ The wrapper uses the locally installed Temurin path when JAVA_HOME is unset. On
 ./track scenario-run /absolute/path/to/G01.json /absolute/path/to/result.json
 ./track scenario-run /absolute/path/to/G02.json /absolute/path/to/framing-evidence.json
 ./track scenario-run /absolute/path/to/G03.json /absolute/path/to/identity-evidence.json
+./track scenario-run /absolute/path/to/G04.json /absolute/path/to/clock-evidence.json
 ./track replay-verify /absolute/path/to/replay /absolute/path/to/evidence
 ./track server config/server.json
 ```
@@ -40,7 +41,7 @@ Each player's pending input storage holds at most 64 intents and rejects overflo
 
 Both Netty event loops use explicit bounded task and tail queues (1,024 each), not an unbounded executor queue. Room commands use a one-thread `ThreadPoolExecutor` with `AbortPolicy`; overflow causes a terminal `INPUT_QUEUE_FULL` reply attempt. Each connection bounds outstanding writes to 64. The last slot is reserved as a `CONTROL_BACKPRESSURE` terminal reply. No snapshot retention or delta queue exists at G01. Parser error replies also pass through the same owner mailbox and bounded outbound path, preserving their order with preceding valid messages. Serialized outbound buffers transfer ownership to Netty on `writeAndFlush`; completion decrements an outstanding-buffer metric. Unit tests check actual inbound and outbound reference counts reach zero, including channel disposal. Snapshot cadence/coalescing remain later Threads.
 
-The manual clock advances an explicit 50ms per tick request with no sleeps or system-clock access. The TCP runner waits for INPUT_ACK sent after owner-side enqueue before advancing the clock. The standalone server uses a single 50ms timer thread with one wait and no delayed-task queue; G04 will replace its intentionally basic scheduling with an accumulator and bounded catch-up.
+The manual tick API advances its monotonic source by 50ms per requested step, with no sleeps or system-clock access. The TCP runner waits for INPUT_ACK sent after owner-side enqueue before advancing time. The standalone server still has one timer thread and one wait, with no delayed-task queue; each wake now asks the Room owner to sample `System.nanoTime` and process the fixed accumulator. The accumulator starts when the Room enters RUNNING, so pre-session idle time does not become game ticks.
 
 The calling main/test thread coordinates shutdown: stop/join the timer, close listener and client channels, drain the I/O callback boundary, close/clear owner state, shut down/join the owner and both event loops. No event loop blocks on another thread. Clients observe LOBBY/RUNNING/FINISHED from server replies and CLOSED from TCP EOF, while the server records its actual terminal lifecycle. Assertions require zero live channels, pending writes, parser buffers/allocated bytes, mailbox tasks and owned threads, terminated executors, stopped timer and locally closed client sockets. Unit tests inspect the actual cumulation reference count after release and reach exactly the maximum 16,388-byte capacity with a valid frame.
 
@@ -68,7 +69,17 @@ The actual unchanged G02 server passed the fixed G03 reproduction: **NOT_REPRODU
 
 G03 holds the real owner consumer with deterministic latches, sends exactly one TCP EAST input and observes unchanged state/pending input before release, one accepted pending intent after drain without a tick, then `(10400,10000)` after one manual tick. A separate pure production `enqueue` probe makes exactly 1,025 admissions with capacity 1,024 held: the last attempt emits terminal `INPUT_QUEUE_FULL`, admitted work drains exactly once, and the real reply ByteBuf reaches reference count zero. Reflection is confined to the scenario harness for observing the pre-existing private owner boundary; no Room mutation bypass or replacement server is introduced. Existing PARANOID parser, 64-input/control limits, foreign-owner and process-shutdown checks remain active.
 
-Exact commands and raw-output locations are in `evidence/G03-command-ledger.jsonl`; `evidence/G03-verification.md` contains the compact result and budget summary. G04 and later features remain inactive.
+Exact commands and raw-output locations are in `evidence/G03-command-ledger.jsonl`; `evidence/G03-verification.md` contains the compact result and budget summary.
+
+## G04 fixed monotonic clock
+
+The old `advanceTicks(1)` API was exercised once per frozen wake before production edits, matching the old timer's one-tick-per-wake callback. It produced six ticks and alpha x=12400 after the fixture's 500ms; the raw report labels that legacy API and unavailable accumulator/overload fields explicitly. The original production-file hashes, baseline harness source, failed assertion XML and output were preserved against START. Wall-clock isolation already held.
+
+`FixedTickClock` is one owner-confined accumulator, shared by the real timer, the existing manual-step API and an injected monotonic source used by the fixed scenario. Each iteration consumes at most four 50ms ticks and retains all remaining elapsed time. A full due tick remaining after the limit exposes `OVERLOADED` separately from the Room lifecycle; it is not a terminal game failure. Clock work is stopped and cleared during existing owner shutdown. No additional executor, queue, scheduler framework, game rule or dependency was added.
+
+For the fixed deltas 50,50,125,0,225,50ms, actual per-iteration ticks are 1,1,2,0,4,2; remaining time is 0,0,25,25,50,0ms. Both wall-clock cases use the same injected monotonic source and accumulator. Canonical evidence separates the comparable `logical` arrays from actual clock readings, production-adapter samples and cleanup counters. The existing real-timer integration test samples the production server's actual monotonic source and retains its executor/timer/socket cleanup assertions. No test sleeps or system wall-clock changes are used.
+
+See `evidence/G04-command-ledger.jsonl` and `evidence/G04-verification.md` for exact commands, baseline provenance, outcomes and budget. Sequence, replay, UDP, many-room scheduling and sustained-overload terminal behavior remain inactive.
 
 G01 initial budget: build/compile <=8, unit suites <=4, integration suites <=2, canonical scenario <=1; network-fault and load runs exactly zero. Main has its own separately frozen one-build/one-unit/one-integration/one-scenario verification budget. No test sleep, microbenchmark, fuzzing, replay, UDP, reconnect, many-room or distributed implementation is included.
 
diff --git a/evidence/G04-command-ledger.jsonl b/evidence/G04-command-ledger.jsonl
new file mode 100644
index 0000000..d58e674
--- /dev/null
+++ b/evidence/G04-command-ledger.jsonl
@@ -0,0 +1,6 @@
+{"kind":"resolved_before_baseline","category":"unit-reproduction","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","unit-test","--tests","arena.RoomTest.frozenG04ClockSchedule"],"environment":{"ARENA_G04_SCENARIO":"/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G04.json","ARENA_G04_EVIDENCE":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g04-initial/reproduce-unit/result.json"},"resolved_at":"2026-08-28T02:22:19.057152+00:00","production_hash_manifest":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g04-initial/reproduce-unit/production-hashes-before.json","baseline_api":"ArenaServer.advanceTicks(1) once per prescribed wake; the unchanged timer also enqueues tick() once per wake","output_directory":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g04-initial/reproduce-unit"}
+{"kind":"executed","category":"unit-reproduction","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","unit-test","--tests","arena.RoomTest.frozenG04ClockSchedule"],"environment":{"ARENA_G04_SCENARIO":"/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G04.json","ARENA_G04_EVIDENCE":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g04-initial/reproduce-unit/result.json"},"started_at":"2026-08-28T02:22:50.091309+00:00","duration_seconds":6.55,"exit_code":1,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g04-initial/reproduce-unit/stdout.log","xml":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g04-initial/reproduce-unit/xml"}
+{"kind":"executed","category":"build","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","build"],"environment":{},"started_at":"2026-08-28T02:28:43.201205+00:00","duration_seconds":8.534,"exit_code":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g04-initial/verify-build/stdout.log"}
+{"kind":"executed","category":"unit","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","unit-test"],"environment":{"ARENA_G04_SCENARIO":"/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G04.json","ARENA_G04_EVIDENCE":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g04-initial/verify-unit/result.json"},"started_at":"2026-08-28T02:28:51.737378+00:00","duration_seconds":5.236,"exit_code":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g04-initial/verify-unit/stdout.log","xml":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g04-initial/verify-unit/xml"}
+{"kind":"executed","category":"integration","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","integration-test"],"environment":{},"started_at":"2026-08-28T02:28:56.976119+00:00","duration_seconds":6.366,"exit_code":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g04-initial/verify-integration/stdout.log","xml":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g04-initial/verify-integration/xml"}
+{"kind":"executed","category":"canonical","cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java","argv":["./track","scenario-run","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G04.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/G04-result.json"],"environment":{},"started_at":"2026-08-28T02:29:03.344454+00:00","duration_seconds":1.23,"exit_code":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g04-initial/verify-canonical/stdout.log"}
diff --git a/evidence/G04-verification.md b/evidence/G04-verification.md
new file mode 100644
index 0000000..e4f44c0
--- /dev/null
+++ b/evidence/G04-verification.md
@@ -0,0 +1,16 @@
+# G04 — initial attempt
+
+START `4d3ecb5a67905c82b9ef7ecbf5b0b93c2b04e913`; profile `realtime-core`; spec `5a6e4a2f8fc71d4be18c3279583bfc2558d5c232`.
+Fixture SHA-256 `45a2fa3c767ac31f6f8550a70051b65b747452c3e8d16e77d09f790341173628` is unchanged.
+
+**Reproduction:** the resolved command `./track unit-test --tests arena.RoomTest.frozenG04ClockSchedule` ran the actual old `advanceTicks(1)` once per prescribed wake, matching the old timer callback. Exit **1**, 6.550s, one assertion failure. Both cases observed `[1,1,1,1,1,1]` ticks, total 6, alpha `(12400,10000)` and internal time 300ms against fixture elapsed 500ms. Accumulator/overload fields were explicitly unavailable, not invented. Wall-clock isolation and cleanup already passed. All nine prior main-source files matched START hashes before execution; manifest, baseline harness/test sources, raw JSON, log and XML are preserved in `runs/g04-initial/reproduce-unit/`. Main was notified before production edits.
+
+**Change:** one owner-confined 50ms accumulator; production `System.nanoTime` and injected manual sources share it. Maximum four ticks per iteration; remaining elapsed time is retained. OVERLOADED is a clock operational metric, separate from Room lifecycle. Existing manual tick calls retain their semantics through the accumulator. Shutdown clears clock work. No gameplay rule, dependency or future feature was added.
+
+**Verification:** exact commands/environment/time/output paths are in `G04-command-ledger.jsonl`. Clean build exit 0 (8.534s); full unit exit 0, **36 tests** (5.236s); integration exit 0, **4 tests** (6.366s); immutable main G04 canonical exit 0 (1.230s), output `G04-result.json`. No skipped tests or additional verification failures/retries. The existing real-timer integration test samples the live server's actual monotonic adapter and retains shutdown assertions.
+
+Both canonical cases: ticks `[1,1,2,0,4,2]`; cumulative `[1,2,4,4,8,10]`; remaining ms `[0,0,25,25,50,0]`; overload `[false,false,false,false,true,false]`. Alpha x positions `[10400,10800,11600,11600,13200,14000]`, y always 10000; bravo unchanged. Fifth iteration reports one due backlog tick while Room stays RUNNING. Raw adapter/readings/metrics remain outside the comparable `logical` object.
+
+Both canonical cleanups: zero channels, connections, pending writes/mailbox, parser bytes/buffers, clock accumulator and owned live threads; owner/event loops terminated, timer stopped, clients closed. Existing G01–G03 PARANOID/owner/bounds/shutdown regressions pass. Canonical high-water marks: catch-up **4**, pending input **1**, mailbox **1**, outbound **2**, parser bytes/capacity **231/256**.
+
+Budget: **4 compiler tasks across 2 compile-bearing commands / 8**, **2 unit / 4** including reproduction, **1 integration / 2**, **1 post-canonical / 1**. Fault/load **0/0**. State hashes `NOT_ACTIVATED_G07`. Unresolved: **none**.
diff --git a/src/main/java/arena/ArenaServer.java b/src/main/java/arena/ArenaServer.java
index bf51b78..18aa2a3 100644
--- a/src/main/java/arena/ArenaServer.java
+++ b/src/main/java/arena/ArenaServer.java
@@ -36,6 +36,7 @@ import java.util.concurrent.TimeoutException;
 import java.util.concurrent.atomic.AtomicBoolean;
 import java.util.concurrent.atomic.AtomicInteger;
 import java.util.concurrent.locks.LockSupport;
+import java.util.function.LongSupplier;
 
 /** A single simulation owner, with Netty used only for nonblocking transport. */
 public final class ArenaServer implements AutoCloseable {
@@ -57,14 +58,19 @@ public final class ArenaServer implements AutoCloseable {
     private final NioEventLoopGroup acceptLoop;
     private final NioEventLoopGroup ioLoop;
     private final boolean manual;
+    private final boolean internalManualClock;
+    private final LongSupplier monotonicNanos;
     private final Channel listener;
     private final Thread ticker;
     // The following fields, including the session registry, are exclusively room-owner state.
     private final Map<Peer, Session> sessions = new HashMap<>();
     private Room room;
+    private FixedTickClock fixedClock;
     private long manualNanos;
     private volatile List<String> closedLifecycle = List.of();
     private volatile int closedInputHighWater;
+    private volatile ObjectNode closedClockMetrics = Json.MAPPER.createObjectNode().put("active", false)
+            .put("accumulator_ns", 0L).put("max_ticks_per_iteration", 0);
 
     private static final class Session {
         final String id = "s-" + UUID.randomUUID();
@@ -109,7 +115,18 @@ public final class ArenaServer implements AutoCloseable {
     }
 
     public ArenaServer(String host, int port, boolean manual) {
+        this(host, port, manual, null);
+    }
+
+    ArenaServer(String host, int port, LongSupplier monotonicNanos) {
+        this(host, port, true, java.util.Objects.requireNonNull(monotonicNanos));
+    }
+
+    private ArenaServer(String host, int port, boolean manual, LongSupplier suppliedClock) {
         this.manual = manual;
+        internalManualClock = manual && suppliedClock == null;
+        monotonicNanos = suppliedClock != null ? suppliedClock
+                : (manual ? () -> manualNanos : FixedTickClock.systemMonotonicSource());
         owner = new ThreadPoolExecutor(1, 1, 0, TimeUnit.MILLISECONDS,
                 new ArrayBlockingQueue<>(MAILBOX_LIMIT), namedFactory("room"), new ThreadPoolExecutor.AbortPolicy());
         acceptLoop = loop("accept");
@@ -163,12 +180,11 @@ public final class ArenaServer implements AutoCloseable {
         if (manual) ticker = null;
         else {
             ticker = namedFactory("clock").newThread(() -> {
-                // Baseline timer has exactly one outstanding wait; no delayed-task queue.
-                // Accumulator, drift and catch-up behavior are deliberately deferred to G04.
+                // One wait, no delayed-task queue. The owner measures elapsed monotonic time.
                 while (!closing.get()) {
-                    LockSupport.parkNanos(TimeUnit.MILLISECONDS.toNanos(50));
+                    LockSupport.parkNanos(FixedTickClock.TICK_NANOS);
                     if (closing.get()) break;
-                    try { execute(this::tick); }
+                    try { execute(this::clockIteration); }
                     catch (RejectedExecutionException overload) { break; }
                 }
             });
@@ -235,7 +251,10 @@ public final class ArenaServer implements AutoCloseable {
                     Room.Player player = room.join(session.playerId);
                     peer.send(Json.message("ROOM_JOINED").put("room_id", room.id).put("player_id", player.id)
                             .put("slot", player.slot).put("status", room.status().name()));
-                    if (room.status() == Room.Status.RUNNING) broadcast(room.view("SNAPSHOT"));
+                    if (room.status() == Room.Status.RUNNING) {
+                        fixedClock = new FixedTickClock(monotonicNanos);
+                        broadcast(room.view("SNAPSHOT"));
+                    }
                 }
                 case "INPUT" -> {
                     if (!roomMatches(peer, message)) break;
@@ -287,11 +306,23 @@ public final class ArenaServer implements AutoCloseable {
         }
     }
 
-    /** Manual clock: each requested step represents exactly 50ms; it never reads wall time. */
+    private void clockIteration() {
+        if (fixedClock == null || room.status() != Room.Status.RUNNING || closing.get()) return;
+        int due = fixedClock.poll();
+        for (int i = 0; i < due; i++) tick();
+    }
+
+    /** Manual scheduler wake: read the injected monotonic source on the same owner path as the timer. */
+    void runClockIteration() {
+        if (!manual) throw new IllegalStateException("manual scheduler wake required");
+        call(() -> { clockIteration(); return null; });
+    }
+
+    /** Existing manual step API: each requested step advances the same accumulator by exactly 50ms. */
     public void advanceTicks(int ticks) {
-        if (!manual || ticks < 0 || ticks > Room.DURATION) throw new IllegalArgumentException("manual tick count");
+        if (!internalManualClock || ticks < 0 || ticks > Room.DURATION) throw new IllegalArgumentException("manual tick count");
         call(() -> {
-            for (int i = 0; i < ticks; i++) { manualNanos += 50_000_000L; tick(); }
+            for (int i = 0; i < ticks; i++) { manualNanos += FixedTickClock.TICK_NANOS; clockIteration(); }
             return null;
         });
     }
@@ -305,10 +336,14 @@ public final class ArenaServer implements AutoCloseable {
 
     ObjectNode metrics() {
         if (closing.get()) return cleanup();
-        return call(() -> Json.MAPPER.createObjectNode().put("manual_time_ns", manualNanos)
-                .put("pending_input_high_water", room == null ? 0 : room.inputHighWater())
-                .put("mailbox_high_water", mailboxHighWater.get()).put("outbound_high_water", outboundHighWater.get())
-                .set("parser", parserMetrics.view()));
+        return call(() -> {
+            ObjectNode result = Json.MAPPER.createObjectNode().put("manual_time_ns", manualNanos)
+                    .put("pending_input_high_water", room == null ? 0 : room.inputHighWater())
+                    .put("mailbox_high_water", mailboxHighWater.get()).put("outbound_high_water", outboundHighWater.get());
+            result.set("clock", fixedClock == null ? closedClockMetrics.deepCopy() : fixedClock.view());
+            result.set("parser", parserMetrics.view());
+            return result;
+        });
     }
 
     public ObjectNode cleanup() {
@@ -321,6 +356,7 @@ public final class ArenaServer implements AutoCloseable {
                 .put("pending_input_high_water", closedInputHighWater)
                 .put("mailbox_high_water", mailboxHighWater.get()).put("outbound_high_water", outboundHighWater.get());
         result.set("parser", parserMetrics.view());
+        result.set("clock", closedClockMetrics.deepCopy());
         var lifecycle = result.putArray("room_lifecycle");
         closedLifecycle.forEach(lifecycle::add);
         return result;
@@ -346,6 +382,7 @@ public final class ArenaServer implements AutoCloseable {
                 closedLifecycle = room.lifecycle();
                 closedInputHighWater = room.inputHighWater();
             }
+            if (fixedClock != null) { fixedClock.stop(); closedClockMetrics = fixedClock.view(); }
             return null;
         });
         owner.shutdown();
diff --git a/src/main/java/arena/ClockScenario.java b/src/main/java/arena/ClockScenario.java
new file mode 100644
index 0000000..d1064f3
--- /dev/null
+++ b/src/main/java/arena/ClockScenario.java
@@ -0,0 +1,126 @@
+package arena;
+
+import com.fasterxml.jackson.databind.JsonNode;
+import com.fasterxml.jackson.databind.node.ArrayNode;
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import java.io.IOException;
+import java.lang.reflect.Field;
+import java.util.List;
+import java.util.concurrent.ThreadPoolExecutor;
+import java.util.concurrent.TimeUnit;
+import java.util.concurrent.atomic.AtomicLong;
+
+/** Fixed G04 clock observations against the real server, with no simulation in the harness. */
+final class ClockScenario {
+    static final String SHA256 = "45a2fa3c767ac31f6f8550a70051b65b747452c3e8d16e77d09f790341173628";
+    private ClockScenario() { }
+
+    static ObjectNode run(ObjectNode scenario, String hash) throws Exception {
+        if (!SHA256.equals(hash)) throw new IOException("G04 requires its frozen scenario bytes");
+        ObjectNode result = Json.MAPPER.createObjectNode().put("thread", "G04").put("contract_version", 1)
+                .put("scenario_id", scenario.path("scenario_id").asText()).put("scenario_sha256", hash)
+                .put("state_hashes", "NOT_ACTIVATED_G07").put("network_fault_runs", 0).put("load_runs", 0);
+        ObjectNode logical = result.putObject("logical");
+        ArrayNode cases = logical.putArray("cases");
+        ArrayNode raw = result.putArray("raw_cases");
+        var productionSource = FixedTickClock.systemMonotonicSource();
+        long beforeSample = System.nanoTime();
+        long sample = productionSource.getAsLong();
+        long afterSample = System.nanoTime();
+        if (sample - beforeSample < 0 || afterSample - sample < 0) throw new IOException("production monotonic adapter");
+        result.putObject("production_adapter").put("source", "System.nanoTime").put("before_ns", beforeSample)
+                .put("sample_ns", sample).put("after_ns", afterSample).put("within_monotonic_sample_interval", true);
+        for (JsonNode name : scenario.withArray("cases")) runCase(scenario, name.asText(), cases.addObject(), raw.addObject());
+        logical.put("all_resources_released", true);
+        ArrayNode failures = result.putArray("failures");
+        JsonNode expectedCounts = Json.MAPPER.valueToTree(List.of(1, 1, 2, 0, 4, 2));
+        JsonNode expectedCumulative = Json.MAPPER.valueToTree(List.of(1, 2, 4, 4, 8, 10));
+        JsonNode expectedRemaining = Json.MAPPER.valueToTree(List.of(0, 0, 25, 25, 50, 0));
+        JsonNode expectedOverload = Json.MAPPER.valueToTree(List.of(false, false, false, false, true, false));
+        JsonNode expectedPositions = Json.MAPPER.valueToTree(List.of(List.of(10400, 10000), List.of(10800, 10000),
+                List.of(11600, 10000), List.of(11600, 10000), List.of(13200, 10000), List.of(14000, 10000)));
+        for (JsonNode cell : cases) {
+            for (var expected : List.of(new Expected("tick_counts", expectedCounts), new Expected("cumulative_ticks", expectedCumulative),
+                    new Expected("remaining_ms", expectedRemaining), new Expected("overloaded", expectedOverload),
+                    new Expected("alpha_positions", expectedPositions)))
+                if (!cell.path(expected.key()).equals(expected.value())) failures.add(cell.path("name").asText() + ": " + expected.key());
+        }
+        result.put("passed", failures.isEmpty());
+        return result;
+    }
+
+    private record Expected(String key, JsonNode value) { }
+
+    private static void runCase(ObjectNode scenario, String name, ObjectNode logical, ObjectNode raw) throws Exception {
+        logical.put("name", name);
+        raw.put("name", name).put("clock_api", "ArenaServer.runClockIteration with injected monotonic source")
+                .put("accumulator_available", true);
+        ArrayNode counts = logical.putArray("tick_counts");
+        ArrayNode cumulative = logical.putArray("cumulative_ticks");
+        ArrayNode remaining = logical.putArray("remaining_ms");
+        ArrayNode overloaded = logical.putArray("overloaded");
+        ArrayNode positions = logical.putArray("alpha_positions");
+        ArrayNode readings = raw.putArray("readings");
+        AtomicLong monotonicNanos = new AtomicLong();
+        ArenaServer server = new ArenaServer("127.0.0.1", 0, monotonicNanos::get);
+        TcpClient alpha = null;
+        TcpClient bravo = null;
+        try {
+            alpha = new TcpClient(server.port()); bravo = new TcpClient(server.port());
+            alpha.hello(); bravo.hello();
+            String roomId = alpha.createRoom();
+            alpha.join(roomId); bravo.join(roomId);
+            alpha.until("SNAPSHOT"); bravo.until("SNAPSHOT");
+            alpha.intent(roomId, scenario.path("directions").path("alpha").asText(), null);
+            bravo.intent(roomId, scenario.path("directions").path("bravo").asText(), null);
+            raw.putObject("issued_identifiers").put("room", roomId).put("alpha", alpha.playerId).put("bravo", bravo.playerId);
+            long monotonicMs = 0;
+            long wallMs = scenario.path("wall_clock_initial_ms").asLong();
+            int previous = 0;
+            int iteration = 0;
+            for (JsonNode delta : scenario.withArray("monotonic_deltas_ms")) {
+                monotonicMs += delta.asLong();
+                wallMs += delta.asLong();
+                if (name.equals("same-monotonic-with-backward-wall-clock"))
+                    wallMs += scenario.withArray("wall_clock_adjustment_ms_by_iteration").get(iteration).asLong();
+                monotonicNanos.set(TimeUnit.MILLISECONDS.toNanos(monotonicMs));
+                server.runClockIteration();
+                ObjectNode state = snapshot(server);
+                ObjectNode metrics = server.metrics();
+                JsonNode clock = metrics.path("clock");
+                int completed = state.path("executed_ticks").asInt();
+                if (completed - previous > scenario.path("max_catch_up_ticks").asInt()) throw new IOException("catch-up bound");
+                counts.add(completed - previous); cumulative.add(completed); previous = completed;
+                remaining.add(clock.path("remaining_ms").asInt(-1)); overloaded.add(clock.path("overloaded").asBoolean());
+                JsonNode actor = player(state, alpha.playerId);
+                positions.addArray().add(actor.path("x").asInt()).add(actor.path("y").asInt());
+                JsonNode other = player(state, bravo.playerId);
+                if (other.path("x").asInt() != 90_000 || other.path("y").asInt() != 90_000
+                        || !state.path("status").asText().equals("RUNNING"))
+                    throw new IOException("clock changed bravo or gameplay lifecycle");
+                ObjectNode reading = readings.addObject().put("iteration", iteration).put("monotonic_ms", monotonicMs).put("wall_ms", wallMs);
+                reading.set("state", state); reading.set("metrics", metrics);
+                iteration++;
+            }
+            server.close(); alpha.expectClosed(); bravo.expectClosed();
+        } finally {
+            try { if (alpha != null) alpha.close(); }
+            finally { try { if (bravo != null) bravo.close(); } finally { server.close(); } }
+        }
+        ScenarioRunner.assertCleanup(server.cleanup());
+        if (alpha == null || bravo == null || !alpha.isClosed() || !bravo.isClosed()) throw new IOException("client socket cleanup");
+        raw.set("cleanup", server.cleanup()); raw.put("client_sockets_closed", true);
+    }
+
+    private static ObjectNode snapshot(ArenaServer server) throws Exception {
+        ThreadPoolExecutor owner = (ThreadPoolExecutor) field(server, "owner");
+        return owner.submit(() -> ((Room) field(server, "room")).view("SNAPSHOT")).get(5, TimeUnit.SECONDS);
+    }
+    private static Object field(Object object, String name) throws ReflectiveOperationException {
+        Field field = object.getClass().getDeclaredField(name); field.setAccessible(true); return field.get(object);
+    }
+    private static JsonNode player(ObjectNode snapshot, String id) throws IOException {
+        for (JsonNode player : snapshot.withArray("players")) if (player.path("player_id").asText().equals(id)) return player;
+        throw new IOException("missing authoritative player");
+    }
+}
diff --git a/src/main/java/arena/FixedTickClock.java b/src/main/java/arena/FixedTickClock.java
new file mode 100644
index 0000000..7ba1b83
--- /dev/null
+++ b/src/main/java/arena/FixedTickClock.java
@@ -0,0 +1,48 @@
+package arena;
+
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import java.util.concurrent.TimeUnit;
+import java.util.function.LongSupplier;
+
+/** One Room-owner accumulator. Production and manual sources enter the same poll path. */
+final class FixedTickClock {
+    static final long TICK_NANOS = TimeUnit.MILLISECONDS.toNanos(50);
+    static final int MAX_CATCH_UP_TICKS = 4;
+    private final LongSupplier monotonicNanos;
+    private long previousNanos;
+    private long accumulatorNanos;
+    private int lastTicks;
+    private int maxTicks;
+    private boolean active = true;
+
+    FixedTickClock(LongSupplier monotonicNanos) {
+        this.monotonicNanos = monotonicNanos;
+        previousNanos = monotonicNanos.getAsLong();
+    }
+
+    static LongSupplier systemMonotonicSource() { return System::nanoTime; }
+
+    int poll() {
+        if (!active) throw new IllegalStateException("clock stopped");
+        long now = monotonicNanos.getAsLong();
+        long elapsed = now - previousNanos;
+        if (elapsed < 0) throw new IllegalStateException("monotonic source moved backwards");
+        previousNanos = now;
+        accumulatorNanos = Math.addExact(accumulatorNanos, elapsed);
+        lastTicks = (int) Math.min(accumulatorNanos / TICK_NANOS, MAX_CATCH_UP_TICKS);
+        accumulatorNanos -= lastTicks * TICK_NANOS;
+        maxTicks = Math.max(maxTicks, lastTicks);
+        return lastTicks;
+    }
+
+    void stop() { active = false; accumulatorNanos = 0; lastTicks = 0; }
+
+    ObjectNode view() {
+        boolean overloaded = accumulatorNanos >= TICK_NANOS;
+        return Json.MAPPER.createObjectNode().put("active", active).put("last_monotonic_ns", previousNanos)
+                .put("last_iteration_ticks", lastTicks).put("max_ticks_per_iteration", maxTicks)
+                .put("accumulator_ns", accumulatorNanos).put("remaining_ms", TimeUnit.NANOSECONDS.toMillis(accumulatorNanos))
+                .put("due_backlog_ticks", accumulatorNanos / TICK_NANOS).put("overloaded", overloaded)
+                .put("operational_state", active ? (overloaded ? "OVERLOADED" : "READY") : "STOPPED");
+    }
+}
diff --git a/src/main/java/arena/ScenarioRunner.java b/src/main/java/arena/ScenarioRunner.java
index 72c4737..ae803bc 100644
--- a/src/main/java/arena/ScenarioRunner.java
+++ b/src/main/java/arena/ScenarioRunner.java
@@ -28,6 +28,14 @@ final class ScenarioRunner {
             if (!result.path("passed").asBoolean()) throw new IOException("G03 assertions: " + result.path("failures"));
             return result;
         }
+        if (scenario.path("thread").asText().equals("G04")) {
+            try {
+                ObjectNode result = ClockScenario.run(scenario, sha256(scenarioBytes));
+                if (!result.path("passed").asBoolean()) throw new IOException("G04 assertions: " + result.path("failures"));
+                return result;
+            } catch (IOException failure) { throw failure; }
+            catch (Exception failure) { throw new IOException("G04 clock scenario failed", failure); }
+        }
         if (!scenario.path("thread").asText().equals("G01") || scenario.path("contract_version").asInt() != 1
                 || !scenario.path("clock").path("kind").asText().equals("manual")
                 || scenario.path("clock").path("tick_duration_ms").asInt() != 50
@@ -129,6 +137,11 @@ final class ScenarioRunner {
         if (cleanup.path("pending_input_high_water").asInt() > Room.INPUT_LIMIT) failures.add("input bound");
         if (cleanup.path("mailbox_high_water").asInt() > ArenaServer.MAILBOX_LIMIT) failures.add("mailbox bound");
         if (cleanup.path("outbound_high_water").asInt() > ArenaServer.OUTBOUND_LIMIT) failures.add("outbound bound");
+        JsonNode clock = cleanup.path("clock");
+        if (clock.path("active").asBoolean(true) || clock.path("accumulator_ns").asLong(-1) != 0
+                || clock.path("max_ticks_per_iteration").asInt(-1) < 0
+                || clock.path("max_ticks_per_iteration").asInt() > FixedTickClock.MAX_CATCH_UP_TICKS)
+            failures.add("clock work cleanup/bound");
         JsonNode parser = cleanup.path("parser");
         if (parser.path("live_buffers").asInt(-1) != 0 || parser.path("allocated_bytes").asInt(-1) != 0)
             failures.add("parser buffer cleanup");
diff --git a/src/test/java/arena/RoomTest.java b/src/test/java/arena/RoomTest.java
index 6ac211d..948dff8 100644
--- a/src/test/java/arena/RoomTest.java
+++ b/src/test/java/arena/RoomTest.java
@@ -12,6 +12,23 @@ import java.util.concurrent.TimeUnit;
 import org.junit.jupiter.api.Test;
 
 final class RoomTest {
+    @Test void frozenG04ClockSchedule() throws Exception {
+        byte[] bytes;
+        String scenarioPath = System.getenv("ARENA_G04_SCENARIO");
+        if (scenarioPath != null) bytes = Files.readAllBytes(Path.of(scenarioPath));
+        else try (var input = getClass().getResourceAsStream("/G04.json")) {
+            assertNotNull(input); bytes = input.readAllBytes();
+        }
+        String hash = HexFormat.of().formatHex(MessageDigest.getInstance("SHA-256").digest(bytes));
+        assertEquals(ClockScenario.SHA256, hash);
+        ObjectNode result = ClockScenario.run(Json.read(bytes), hash);
+        String evidencePath = System.getenv("ARENA_G04_EVIDENCE");
+        if (evidencePath != null)
+            Files.write(Path.of(evidencePath), Json.MAPPER.writerWithDefaultPrettyPrinter().writeValueAsBytes(result));
+        System.out.println("G04 clock " + result);
+        assertTrue(result.path("passed").asBoolean(), result.path("failures").toString());
+    }
+
     @Test void frozenG03IdentityLifecycle() throws Exception {
         byte[] bytes;
         String scenarioPath = System.getenv("ARENA_G03_SCENARIO");
diff --git a/src/test/java/arena/ServerIntegrationTest.java b/src/test/java/arena/ServerIntegrationTest.java
index 828d8cf..56a7df0 100644
--- a/src/test/java/arena/ServerIntegrationTest.java
+++ b/src/test/java/arena/ServerIntegrationTest.java
@@ -56,6 +56,15 @@ final class ServerIntegrationTest {
     @Test void realTimerAndBothEventLoopsTerminateWithoutSleep() throws Exception {
         ArenaServer server = new ArenaServer("127.0.0.1", 0, false);
         try (TcpClient client = new TcpClient(server.port())) {
+            var sourceField = ArenaServer.class.getDeclaredField("monotonicNanos");
+            sourceField.setAccessible(true);
+            var source = (java.util.function.LongSupplier) sourceField.get(server);
+            long before = System.nanoTime();
+            long sampled = source.getAsLong();
+            long after = System.nanoTime();
+            assertTrue(sampled - before >= 0 && after - sampled >= 0, "live production server uses monotonic adapter");
+            System.out.println("G04 production server adapter " + Json.MAPPER.createObjectNode()
+                    .put("before_ns", before).put("sample_ns", sampled).put("after_ns", after));
             client.hello();
             server.close();
             client.expectClosed();
diff --git a/src/test/resources/G04.json b/src/test/resources/G04.json
new file mode 100644
index 0000000..10efa44
--- /dev/null
+++ b/src/test/resources/G04.json
@@ -0,0 +1,37 @@
+{
+  "scenario_id": "G04-fixed-clock",
+  "contract_version": 1,
+  "thread": "G04",
+  "seed": 7050,
+  "clients": [
+    "alpha",
+    "bravo"
+  ],
+  "directions": {
+    "alpha": "EAST",
+    "bravo": "STOP"
+  },
+  "monotonic_deltas_ms": [
+    50,
+    50,
+    125,
+    0,
+    225,
+    50
+  ],
+  "wall_clock_initial_ms": 1700000000000,
+  "wall_clock_adjustment_ms_by_iteration": [
+    0,
+    0,
+    -3600000,
+    0,
+    0,
+    0
+  ],
+  "tick_duration_ms": 50,
+  "max_catch_up_ticks": 4,
+  "cases": [
+    "monotonic-only",
+    "same-monotonic-with-backward-wall-clock"
+  ]
+}
