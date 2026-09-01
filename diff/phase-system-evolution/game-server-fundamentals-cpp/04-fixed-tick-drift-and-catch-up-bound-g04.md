# Fixed Tick, Drift와 Catch-up Bound (G04)

## `feat(time): bound fixed ticks with monotonic accumulation`

diff --git a/TRACK.md b/TRACK.md
index a752ce9..bd6ecf8 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -1,4 +1,4 @@
-# fundamentals-cpp — G03 identity and lifecycle ownership
+# fundamentals-cpp — G04 fixed monotonic time
 
 SPEC_REVISION: `5a6e4a2f8fc71d4be18c3279583bfc2558d5c232`
 
@@ -27,6 +27,7 @@ Run from this worktree; `track` resolves its own source directory. Build never r
 ./track scenario-run /absolute/path/to/G01.json /absolute/path/to/evidence.json
 ./track scenario-run /absolute/path/to/G02.json /absolute/path/to/evidence.json
 ./track scenario-run /absolute/path/to/G03.json /absolute/path/to/evidence.json
+./track scenario-run /absolute/path/to/G04.json /absolute/path/to/evidence.json
 ./track replay-verify /absolute/path/to/replay.json /absolute/path/to/evidence.json
 ./track server /absolute/path/to/config.json
 ```
@@ -52,10 +53,10 @@ ARENA_BUILD_DIR="$PWD/build-tsan" TSAN_OPTIONS=halt_on_error=1 ./track unit-test
 CMake's normal `CXXFLAGS`/`LDFLAGS` also support independent instrumentation. The `ARENA_TSAN` option only adds compiler/linker flags.
 Network tests require permission to bind loopback sockets in restricted execution environments. No Internet is used.
 
-The server config accepts `listen_port` (0 chooses an OS-assigned port) and `clock: "manual"`.
+The server config accepts `listen_port` (0 chooses an OS-assigned port) and `clock: "monotonic"` (the default) or `clock: "manual"`.
 It prints a JSON `READY` line, services real TCP, and accepts local operator lines on stdin:
-`tick` advances one authoritative 50ms step if running; `state` prints current state; `stop`, EOF, SIGTERM or SIGINT closes the server.
-Operator commands are local process control, not extra wire protocol messages. A 10ms I/O wait does not advance simulation.
+`tick` advances one authoritative 50ms step in manual mode; `state` prints current state; `stop`, EOF, SIGTERM or SIGINT closes the server.
+Operator commands are local process control, not extra wire protocol messages. In monotonic mode each owner loop samples `std::chrono::steady_clock` through the same scheduler path used by the injected manual clock tests. The 10ms kqueue wait only bounds idle I/O waiting; elapsed monotonic time determines due fixed ticks.
 
 ## Current guarantee and ownership
 
@@ -96,7 +97,7 @@ Both clients observe LOBBY, RUNNING and CLOSED via actual TCP. The canonical 120
 | Pending inputs | 64 per player; 65th is INPUT_QUEUE_FULL, accepted inputs retained until tick |
 | Outbound control | 64 messages per connection, each bounded frame; overflow disconnect with CONTROL_BACKPRESSURE metric |
 | I/O iteration | 64 kqueue events, up to 64 accepts, up to 64 writes per ready connection |
-| Manual tick work | one explicit tick call; no accumulator or catch-up implemented |
+| Scheduler iteration | at most four 50ms ticks; unexecuted elapsed time remains in the integer accumulator and exposes OVERLOADED |
 | Client evidence | 4,096 received messages per client; overflow is a test failure |
 | Runner input/output | 1 MiB JSON input, 512 input commands, 32 setup commands; 4 MiB evidence output |
 | Operator input | 4,096 bytes; overflow terminates with explicit failure |
@@ -141,4 +142,12 @@ A nonexistent room reference is `ROOM_NOT_FOUND`; foreign session is `SESSION_IN
 
 The G03 canonical runner reads the actual scenario path, uses real issued identifiers, holds FINISHED at exactly 1200 manual ticks, and asserts unchanged state across rejection/leave/close. The ownership probe uses `poll_io` without consumer release, then `drain_mailbox` without a tick, then one explicit 50ms tick. The pure mailbox probe uses the production mailbox type, no sockets or rooms, and drains all accepted envelopes. Evidence and the one command ledger are `evidence/G03.md` and `evidence/G03-runs.jsonl`.
 
-Clock accumulator/catch-up (G04), input sequence/target tick (G05), abuse matrix (G06), replay/hash (G07), full/delta cadence (G08), UDP, reconnect, slow-consumer coalescing and many-room scheduling remain inactive.
+## G04 clock boundary
+
+The unchanged G03 explicit-step API executes one tick per operator call. Under the six fixed wake boundaries it executed six ticks, not the elapsed-time contract's ten. That pre-change behavior and its already-correct wall-clock isolation are preserved in `evidence/G04.md` and its raw source-hash manifest.
+
+G04 adds a small integer `FixedTickAccumulator` and `Server::run_scheduler`. The Room's RUNNING transition anchors the accumulator to the injected monotonic source. Each iteration consumes at most four fixed ticks and retains the rest; `remaining_ms`, `overloaded`, and the separate `NORMAL`/`OVERLOADED` operational metric expose the result without changing Room lifecycle. Scheduler backlog is released when the Room finishes or the server shuts down. No worker, timer registry, wall-clock dependency, overload terminal policy or new game rule is introduced.
+
+The production clock is `std::chrono::steady_clock`; the canonical runner injects a manual monotonic reader into the same Server path. The fixed deltas yield `[1,1,2,0,4,2]` ticks and `[0,0,25,25,50,0]` remaining milliseconds. A wall-clock reversal is recorded only as external evidence. The complete unit suite preserves prior regressions, and a separate monotonic-mode execution extends the existing standalone shutdown integration test to verify actual adapter reads in the CLI scheduler.
+
+Input sequence/target tick (G05), abuse matrix (G06), replay/hash (G07), full/delta cadence (G08), UDP, reconnect, slow-consumer coalescing and many-room scheduling remain inactive.
diff --git a/evidence/G04-runs.jsonl b/evidence/G04-runs.jsonl
new file mode 100644
index 0000000..452d54b
--- /dev/null
+++ b/evidence/G04-runs.jsonl
@@ -0,0 +1,6 @@
+{"category":"build","units":1,"label":"reproduce-compile","argv":["/usr/bin/clang++","-std=c++20","-O2","-Wall","-Wextra","-Wpedantic","-Werror","-fsanitize=thread","-g","-I","src","-I","/opt/homebrew/include","artifacts/g04/reproduce.cpp","src/game.cpp","src/transport.cpp","src/scenario.cpp","-o","artifacts/g04/reproduce-g03"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","started_at":"2026-08-28T02:23:11.693104+00:00","duration_seconds":17.605891,"exit":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g04/reproduce-compile.log"}
+{"category":"unit","units":1,"label":"reproduce-g03","argv":["env","TSAN_OPTIONS=halt_on_error=1","artifacts/g04/reproduce-g03","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G04.json","artifacts/g04/reproduction.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","started_at":"2026-08-28T02:26:30.727962+00:00","duration_seconds":0.807449,"exit":1,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g04/reproduce-g03.log"}
+{"category":"build","units":2,"label":"tsan-build","argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g04-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g04/track","ARENA_TSAN=ON","./track","build"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","started_at":"2026-08-28T02:36:23.195689+00:00","duration_seconds":23.294354,"exit":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g04/tsan-build.log"}
+{"category":"unit","units":1,"label":"tsan-unit","argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g04-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g04/track","TSAN_OPTIONS=halt_on_error=1","./track","unit-test"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","started_at":"2026-08-28T02:37:50.318993+00:00","duration_seconds":2.049393,"exit":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g04/tsan-unit.log"}
+{"category":"integration","units":1,"label":"tsan-integration","argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g04-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g04/track","TSAN_OPTIONS=halt_on_error=1","./track","integration-test"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","started_at":"2026-08-28T02:38:19.439448+00:00","duration_seconds":2.248878,"exit":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g04/tsan-integration.log"}
+{"category":"canonical","units":1,"label":"tsan-canonical","argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g04-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g04/track","TSAN_OPTIONS=halt_on_error=1","./track","scenario-run","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G04.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g04/canonical.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","started_at":"2026-08-28T02:38:54.062871+00:00","duration_seconds":0.413408,"exit":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g04/tsan-canonical.log"}
diff --git a/evidence/G04.md b/evidence/G04.md
new file mode 100644
index 0000000..3e59c59
--- /dev/null
+++ b/evidence/G04.md
@@ -0,0 +1,49 @@
+# G04 — fixed monotonic time and bounded catch-up
+
+THREAD G04; BRANCH `track/fundamentals-cpp`; PROFILE `realtime-core`; ATTEMPT initial.
+SPEC_REVISION `5a6e4a2f8fc71d4be18c3279583bfc2558d5c232`.
+START `9a77ab3b3ae1cdbae2cd586bf1119f06aa45519b`.
+Frozen G04 input SHA-256 `45a2fa3c767ac31f6f8550a70051b65b747452c3e8d16e77d09f790341173628`.
+
+## Baseline, resolved before execution
+
+The old entry point is `Server::advance_one_tick`, called by the production CLI's explicit `tick` operation. The baseline calls that unchanged operation once at each fixed wake boundary, recording the actual Room ticks/positions and external monotonic/wall readings. It does not implement an accumulator or assert that G03 already has an automatic timer. The existing real-TCP G03 setup/cleanup fixture is reused without alteration. The source manifest compares tracked production bytes with START before any production edit.
+
+```sh
+/usr/bin/clang++ -std=c++20 -O2 -Wall -Wextra -Wpedantic -Werror -fsanitize=thread -g -I src -I /opt/homebrew/include artifacts/g04/reproduce.cpp src/game.cpp src/transport.cpp src/scenario.cpp -o artifacts/g04/reproduce-g03
+env TSAN_OPTIONS=halt_on_error=1 artifacts/g04/reproduce-g03 /Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G04.json artifacts/g04/reproduction.json
+```
+
+The second command is one unit-category reproduction. Expected new-constraint counts are `[1,1,2,0,4,2]`; observed legacy counts are recorded separately. Wall-clock isolation already present is reported separately. Raw evidence and the pre-change manifest remain under `artifacts/g04`; exact argv/timing/exits are in `G04-runs.jsonl`.
+
+Ceilings: compile/configure 8; unit 4 including baseline; integration 2; one post canonical; fault/load 0.
+
+The baseline compile exited 0 (17.605891s); execution exited **1** (0.807449s), reporting eight new-contract iteration mismatches across the two cases. Actual tick counts were `[1,1,1,1,1,1]`, cumulative `[1,2,3,4,5,6]`, final alpha `(12400,10000)`. Both cases matched, so existing wall-clock isolation is NOT_REPRODUCED. This is missing elapsed-time scheduling, not a claim that G03's explicit-step operation was incorrect under its old contract. All 12 descriptors closed and both active cleanup sets were zero. `pre-change-production.json` records all eight source hashes matching START. Main was notified before production edits; no acknowledgement pause is required under the amended workflow.
+
+## Change and fixed verification
+
+Only the required monotonic accumulator/scheduler, production adapter and corresponding tests/runner are added. Gameplay rules remain unchanged. The real CLI defaults to monotonic scheduling; manual mode preserves earlier explicit-step regressions. The common `logical` result has two ordered case entries containing actual tick counts, cumulative ticks, remaining milliseconds, overload booleans and alpha positions; raw clock, adapter and cleanup evidence stays outside it.
+
+```sh
+env ARENA_BUILD_DIR=$PWD/build-g04-tsan ARENA_EVIDENCE_DIR=$PWD/artifacts/g04/track ARENA_TSAN=ON ./track build
+env ARENA_BUILD_DIR=$PWD/build-g04-tsan ARENA_EVIDENCE_DIR=$PWD/artifacts/g04/track TSAN_OPTIONS=halt_on_error=1 ./track unit-test
+env ARENA_BUILD_DIR=$PWD/build-g04-tsan ARENA_EVIDENCE_DIR=$PWD/artifacts/g04/track TSAN_OPTIONS=halt_on_error=1 ./track integration-test
+env ARENA_BUILD_DIR=$PWD/build-g04-tsan ARENA_EVIDENCE_DIR=$PWD/artifacts/g04/track TSAN_OPTIONS=halt_on_error=1 ./track scenario-run /Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G04.json $PWD/artifacts/g04/canonical.json
+```
+
+## Final verification
+
+| Execution | Exit | Result / duration |
+|---|---:|---|
+| TSan configure/build | 0 | frozen compiler/dependencies, warnings as errors; 23.294354s |
+| Complete unit suite | 0 | 13/13, all 11 earlier regressions preserved; 2.049393s |
+| Complete integration suite | 0 | 3/3, prior tests plus production monotonic CLI path; 2.248878s |
+| Actual frozen G04 canonical | 0 | both clock cases and cleanup pass; 0.413408s |
+
+Both actual logical rows contain ticks `[1,1,2,0,4,2]`, cumulative ticks `[1,2,4,4,8,10]`, remaining ms `[0,0,25,25,50,0]`, and overload flags `[false,false,false,false,true,false]`. Alpha positions are `[[10400,10000],[10800,10000],[11600,10000],[11600,10000],[13200,10000],[14000,10000]]`; bravo and all unrelated state stay unchanged. Raw fifth-iteration evidence reports OVERLOADED while gameplay remains RUNNING. Each case executes six scheduler iterations, seven monotonic reads including the start anchor, and a four-tick maximum. All 12 checked descriptors close, pending scheduler time and all active resource counters are zero, and `logical.all_resources_released` is true.
+
+The production adapter's two observed canonical readings were both 157439561ms (nondecreasing). The separate real CLI monotonic integration recorded two scheduler iterations/two adapter reads; manual mode recorded none. Both child processes exited 0, were reaped, and released rebindable listener ports. TSan reported no error. The expected baseline exit 1 is the only nonzero test command; no retries occurred.
+
+Final budget: compile/configure **3/8**, unit **2/4** including baseline, integration **1/2**, post canonical **1/1**. Raw artifacts remain in `artifacts/g04`: canonical SHA-256 `4e861611856e820884c8dfd1a63d0a1aa843fe514ebccd68315e014fb9173403`, baseline `3ed1e3c02cbdeff0a8e7b970ced46ccb9e719b8701ff2c58349eb36cdabc182e`, pre-change manifest `c9b668904d9a36a0d2666b9ce1eecf90693443f4ae4d507735dc94dbe88e3107`. The shared scenario SHA was rechecked unchanged.
+
+STATE_HASHES inactive until G07; NETWORK_FAULT_RUNS 0; LOAD_RUNS 0. UNRESOLVED: no known G04 failure; independent main verification/comparison pending. No G05+ business logic, dependency change or sustained-overload terminal policy was added.
diff --git a/server.example.json b/server.example.json
index 0a75404..cb1b573 100644
--- a/server.example.json
+++ b/server.example.json
@@ -1 +1 @@
-{"listen_port":0,"clock":"manual"}
+{"listen_port":0,"clock":"monotonic"}
diff --git a/src/game.cpp b/src/game.cpp
index 7bf6772..b184f64 100644
--- a/src/game.cpp
+++ b/src/game.cpp
@@ -1,11 +1,28 @@
 #include "game.hpp"
 #include <algorithm>
 #include <array>
+#include <chrono>
 #include <set>
 #include <stdexcept>
 #include <utility>
 
 namespace arena {
+std::int64_t monotonic_milliseconds() noexcept {
+  static_assert(std::chrono::steady_clock::is_steady);
+  return std::chrono::duration_cast<std::chrono::milliseconds>(std::chrono::steady_clock::now().time_since_epoch()).count();
+}
+void FixedTickAccumulator::reset(std::int64_t monotonic_ms) {
+  previous_ms_ = monotonic_ms;
+  remaining_ms_ = 0;
+}
+TickBatch FixedTickAccumulator::advance(std::int64_t monotonic_ms) {
+  if (monotonic_ms < previous_ms_) throw std::logic_error("monotonic clock moved backwards");
+  remaining_ms_ += monotonic_ms - previous_ms_;
+  previous_ms_ = monotonic_ms;
+  const auto ticks = static_cast<int>(std::min<std::int64_t>(remaining_ms_ / tick_duration_ms, max_catch_up_ticks));
+  remaining_ms_ -= static_cast<std::int64_t>(ticks) * tick_duration_ms;
+  return {ticks, remaining_ms_, remaining_ms_ >= tick_duration_ms};
+}
 std::string direction_name(Direction direction) {
   switch (direction) {
     case Direction::stop: return "STOP";
diff --git a/src/game.hpp b/src/game.hpp
index 2d2e3c5..1c267a5 100644
--- a/src/game.hpp
+++ b/src/game.hpp
@@ -18,6 +18,7 @@ inline constexpr std::size_t max_control_messages = 64;
 inline constexpr std::size_t max_mailbox_messages = max_players * max_pending_inputs;
 inline constexpr int session_ticks = 1'200;
 inline constexpr int tick_duration_ms = 50;
+inline constexpr int max_catch_up_ticks = 4;
 
 enum class Direction { stop, north, east, south, west };
 std::string direction_name(Direction direction);
@@ -78,9 +79,25 @@ class Room {
   std::size_t input_high_water_ = 0;
 };
 
-// G01 advances one explicit 50ms step. Accumulator/catch-up first activate in G04.
+// This is executed simulation time, separate from the scheduler's clock source.
 struct ManualClock {
   std::int64_t now_ms = 0;
   void advance_one() { now_ms += tick_duration_ms; }
 };
+
+std::int64_t monotonic_milliseconds() noexcept;
+struct TickBatch {
+  int ticks = 0;
+  std::int64_t remaining_ms = 0;
+  bool overloaded = false;
+};
+class FixedTickAccumulator {
+ public:
+  void reset(std::int64_t monotonic_ms);
+  TickBatch advance(std::int64_t monotonic_ms);
+  std::int64_t remaining_ms() const { return remaining_ms_; }
+ private:
+  std::int64_t previous_ms_ = 0;
+  std::int64_t remaining_ms_ = 0;
+};
 }
diff --git a/src/lifecycle_scenario.cpp b/src/lifecycle_scenario.cpp
index 7d0cfad..771d30b 100644
--- a/src/lifecycle_scenario.cpp
+++ b/src/lifecycle_scenario.cpp
@@ -25,13 +25,14 @@ struct Peer {
 struct LifecycleFixture {
   int descriptors_before = Fd::live();
   ManualClock clock;
-  Server server{clock};
+  Server server;
   std::map<std::string, Peer> peers;
   std::string room_id;
   int ceiling;
   std::size_t closed_client_checks = 0;
 
-  explicit LifecycleFixture(int socket_ceiling) : ceiling(socket_ceiling) {}
+  explicit LifecycleFixture(int socket_ceiling, Server::MonotonicNow monotonic_now = monotonic_milliseconds)
+      : server(clock, 0, std::move(monotonic_now)), ceiling(socket_ceiling) {}
   ~LifecycleFixture() {
     try { server.shutdown(); } catch (...) { /* explicit checks report failures */ }
   }
@@ -335,4 +336,87 @@ Json run_lifecycle_scenario(const Json& scenario) {
   evidence["result"] = "PASS";
   return evidence;
 }
+
+Json run_clock_scenario(const Json& scenario) {
+  const auto check_clock = [](bool condition, const std::string& text) {
+    if (!condition) throw std::runtime_error("G04: " + text);
+  };
+  check_clock(scenario.at("thread") == "G04" && scenario.at("contract_version") == 1 && scenario.at("seed") == 7050 &&
+    scenario.at("tick_duration_ms") == tick_duration_ms && scenario.at("max_catch_up_ticks") == max_catch_up_ticks,
+    "fixed contract constants changed");
+  check_clock(scenario.at("clients") == Json::array({"alpha", "bravo"}) && scenario.at("directions").at("alpha") == "EAST" &&
+    scenario.at("directions").at("bravo") == "STOP" && scenario.at("monotonic_deltas_ms") == Json::array({50,50,125,0,225,50}) &&
+    scenario.at("wall_clock_adjustment_ms_by_iteration") == Json::array({0,0,-3600000,0,0,0}) &&
+    scenario.at("wall_clock_initial_ms") == 1700000000000LL &&
+    scenario.at("cases") == Json::array({"monotonic-only", "same-monotonic-with-backward-wall-clock"}), "fixed clock fixture changed");
+  const auto descriptors_before = Fd::live();
+  const auto adapter_first = monotonic_milliseconds(), adapter_second = monotonic_milliseconds();
+  check_clock(adapter_second >= adapter_first, "production adapter is not monotonic");
+  Json evidence{{"scenario_id", scenario.at("scenario_id")}, {"thread", "G04"}, {"contract_version", 1},
+    {"state_hashes", "INACTIVE_UNTIL_G07"}, {"executed_ticks", 0}, {"case_evidence", Json::array()},
+    {"logical", Json{{"cases", Json::array()}}},
+    {"production_adapter", Json{{"source", "std::chrono::steady_clock"}, {"is_steady", std::chrono::steady_clock::is_steady},
+      {"readings_ms", Json::array({adapter_first, adapter_second})}}}};
+  std::size_t descriptor_checks = 0;
+  for (const auto& case_value : scenario.at("cases")) {
+    std::int64_t monotonic_ms = 0;
+    auto wall_ms = scenario.at("wall_clock_initial_ms").get<std::int64_t>();
+    LifecycleFixture fixture(5000, [&] { return monotonic_ms; }); fixture.setup(2);
+    const auto initial = fixture.server.room().view();
+    for (const auto& role_value : scenario.at("clients")) {
+      const auto role = role_value.get<std::string>();
+      auto input = fixture.request(role, "INPUT");
+      input["direction"] = scenario.at("directions").at(role); input["tag_target_player_id"] = nullptr;
+      auto& peer = fixture.peers.at(role);
+      peer.tcp->send(input);
+      const auto ack = peer.tcp->receive_type(fixture.server, "INPUT_ACK");
+      check_clock(ack.at("accepted") == true && ack.at("tick") == 0, "fixed intent did not reach tick zero");
+    }
+    Json logical{{"tick_counts", Json::array()}, {"cumulative_ticks", Json::array()}, {"remaining_ms", Json::array()},
+      {"overloaded", Json::array()}, {"alpha_positions", Json::array()}};
+    Json raw{{"case", case_value}, {"readings", Json::array()}, {"identities", fixture.identities()}};
+    for (std::size_t index = 0; index < scenario.at("monotonic_deltas_ms").size(); ++index) {
+      const auto delta = scenario.at("monotonic_deltas_ms").at(index).get<std::int64_t>();
+      monotonic_ms += delta; wall_ms += delta;
+      if (case_value == "same-monotonic-with-backward-wall-clock")
+        wall_ms += scenario.at("wall_clock_adjustment_ms_by_iteration").at(index).get<std::int64_t>();
+      const auto before_ticks = fixture.server.room().executed_ticks();
+      const auto batch = fixture.server.run_scheduler();
+      const auto ticks = fixture.server.room().executed_ticks();
+      check_clock(batch.ticks == ticks - before_ticks && batch.ticks <= max_catch_up_ticks, "scheduler exceeded bounded actual tick work");
+      auto expected = initial;
+      expected["executed_ticks"] = ticks; expected["tick"] = ticks - 1;
+      for (auto& player : expected["players"]) if (player.at("player_id") == fixture.peers.at("alpha").player) {
+        player["x"] = 10000 + ticks * 400; player["direction"] = ticks == 0 ? "STOP" : "EAST";
+      }
+      check_clock(fixture.server.room().view() == expected && fixture.clock.now_ms == ticks * tick_duration_ms,
+                  "fixed tick altered unrelated authoritative state or movement/time meaning");
+      const auto& alpha = fixture.server.room().players().at(fixture.peers.at("alpha").player);
+      logical["tick_counts"].push_back(batch.ticks); logical["cumulative_ticks"].push_back(ticks);
+      logical["remaining_ms"].push_back(batch.remaining_ms); logical["overloaded"].push_back(batch.overloaded);
+      logical["alpha_positions"].push_back(Json::array({alpha.x, alpha.y}));
+      const auto metrics = fixture.server.metrics();
+      check_clock(metrics.at("scheduler").at("operational_state") == (batch.overloaded ? "OVERLOADED" : "NORMAL") &&
+                  fixture.server.room().status() == "RUNNING", "overload must be separate from gameplay lifecycle");
+      raw["readings"].push_back(Json{{"monotonic_delta_ms", delta}, {"monotonic_ms", monotonic_ms}, {"wall_clock_ms", wall_ms},
+        {"simulation_ms", fixture.clock.now_ms}, {"state", fixture.server.room().view()}, {"scheduler", metrics.at("scheduler")}});
+    }
+    check_clock(logical.at("tick_counts") == Json::array({1,1,2,0,4,2}) &&
+      logical.at("cumulative_ticks") == Json::array({1,2,4,4,8,10}) && logical.at("remaining_ms") == Json::array({0,0,25,25,50,0}) &&
+      logical.at("overloaded") == Json::array({false,false,false,false,true,false}) && monotonic_ms == 500,
+      "fixed schedule lost elapsed time, advanced on zero delta, or failed to expose retained backlog");
+    evidence["executed_ticks"] = evidence.at("executed_ticks").get<int>() + fixture.server.room().executed_ticks();
+    raw["metrics"] = fixture.server.metrics(); raw["cleanup"] = fixture.finish();
+    descriptor_checks += raw.at("cleanup").at("descriptor_checks").get<std::size_t>();
+    evidence["case_evidence"].push_back(raw); evidence["logical"]["cases"].push_back(logical);
+  }
+  check_clock(evidence.at("logical").at("cases").at(0) == evidence.at("logical").at("cases").at(1),
+              "wall-clock reversal changed scheduler or simulation results");
+  evidence["logical"]["all_resources_released"] = Fd::live() == descriptors_before;
+  check_clock(evidence.at("logical").at("all_resources_released").get<bool>(), "fixed-clock scenario leaked resources");
+  evidence["cleanup"] = Json{{"descriptor_checks", descriptor_checks}, {"all_descriptors_closed", true},
+    {"tracked_descriptor_delta", Fd::live() - descriptors_before}, {"all_case_resource_counts_zero", true}};
+  evidence["result"] = "PASS";
+  return evidence;
+}
 }
diff --git a/src/main.cpp b/src/main.cpp
index fbc2c8b..37f9bef 100644
--- a/src/main.cpp
+++ b/src/main.cpp
@@ -12,19 +12,21 @@ namespace {
 volatile std::sig_atomic_t stop_requested = 0;
 void request_stop(int) { stop_requested = 1; }
 int serve(const arena::Json& config) {
-  if (config.value("clock", std::string("manual")) != "manual") throw std::runtime_error("G01 server supports manual clock only");
+  const auto clock_mode = config.value("clock", std::string("monotonic"));
+  if (clock_mode != "manual" && clock_mode != "monotonic") throw std::runtime_error("clock must be manual or monotonic");
   const auto port = config.value("listen_port", 0);
   if (port < 0 || port > 65535) throw std::runtime_error("listen_port outside range");
   arena::ManualClock clock;
   arena::Server server(clock, static_cast<std::uint16_t>(port));
   std::signal(SIGTERM, request_stop);
   std::signal(SIGINT, request_stop);
-  std::cout << arena::Json{{"status", "READY"}, {"port", server.port()}, {"clock", "manual"}}.dump() << std::endl;
+  std::cout << arena::Json{{"status", "READY"}, {"port", server.port()}, {"clock", clock_mode}}.dump() << std::endl;
   std::string operator_input;
   bool stopped = false;
   while (!stopped && !stop_requested) {
     server.pump(10);
     if (stop_requested) break;
+    if (clock_mode == "monotonic") server.run_scheduler();
     pollfd input{STDIN_FILENO, POLLIN, 0};
     if (::poll(&input, 1, 0) < 0) {
       if (errno == EINTR) continue;
@@ -44,12 +46,12 @@ int serve(const arena::Json& config) {
       operator_input.erase(0, end + 1);
       if (command == "stop") { stopped = true; break; }
       if (command == "state") { std::cout << server.room().view().dump() << std::endl; continue; }
-      if (command == "tick") { server.advance_one_tick(); continue; }
-      throw std::runtime_error("manual operator commands are tick, state, stop");
+      if (command == "tick" && clock_mode == "manual") { server.advance_one_tick(); continue; }
+      throw std::runtime_error("operator commands are state, stop, and tick in manual mode");
     }
   }
   server.shutdown();
-  std::cout << arena::Json{{"status", "STOPPED"}, {"cleanup", server.cleanup()}}.dump() << std::endl;
+  std::cout << arena::Json{{"status", "STOPPED"}, {"cleanup", server.cleanup()}, {"metrics", server.metrics()}}.dump() << std::endl;
   return 0;
 }
 }
diff --git a/src/scenario.cpp b/src/scenario.cpp
index 84ef195..de0a555 100644
--- a/src/scenario.cpp
+++ b/src/scenario.cpp
@@ -283,6 +283,7 @@ Json run_framing_scenario(const Json& scenario) {
 Json run_scenario(const Json& scenario) {
   if (scenario.at("thread") == "G02") return run_framing_scenario(scenario);
   if (scenario.at("thread") == "G03") return run_lifecycle_scenario(scenario);
+  if (scenario.at("thread") == "G04") return run_clock_scenario(scenario);
   require(scenario.at("contract_version") == 1 && scenario.at("thread") == "G01", "only G01 contract v1 is active");
   require(scenario.at("clock").at("kind") == "manual" && scenario.at("clock").at("tick_duration_ms") == tick_duration_ms,
           "G01 runner requires the fixed 50ms manual clock");
diff --git a/src/scenario.hpp b/src/scenario.hpp
index 9d1ebc1..6cdb488 100644
--- a/src/scenario.hpp
+++ b/src/scenario.hpp
@@ -7,4 +7,5 @@ void write_json_file(const std::filesystem::path& path, const Json& value);
 Json run_scenario(const Json& scenario);
 Json run_lifecycle_scenario(const Json& scenario);
 Json run_mailbox_probe(std::size_t capacity);
+Json run_clock_scenario(const Json& scenario);
 }
diff --git a/src/transport.cpp b/src/transport.cpp
index 523a217..34ef148 100644
--- a/src/transport.cpp
+++ b/src/transport.cpp
@@ -163,7 +163,9 @@ void PendingWrite::consume(std::size_t count) {
   if (count > remaining().size()) throw std::logic_error("write offset exceeds owned buffer");
   offset += count;
 }
-Server::Server(ManualClock& clock, std::uint16_t port) : clock_(clock), reactor_(::kqueue()), listener_(::socket(AF_INET, SOCK_STREAM, 0)) {
+Server::Server(ManualClock& clock, std::uint16_t port, MonotonicNow monotonic_now)
+    : clock_(clock), monotonic_now_(std::move(monotonic_now)), reactor_(::kqueue()), listener_(::socket(AF_INET, SOCK_STREAM, 0)) {
+  if (!monotonic_now_) throw std::invalid_argument("monotonic clock source is required");
   if (reactor_.get() == -1 || listener_.get() == -1) system_failure("create kqueue/listener");
   std::random_device random;
   std::ostringstream nonce;
@@ -390,6 +392,7 @@ void Server::handle(const Envelope& envelope) {
       queue(id, std::move(lobby));
       queue(id, std::move(reply));
       if (room_.status() == "RUNNING") {
+        accumulator_.reset(read_monotonic());
         Json state = room_.view(); state.update(message("SNAPSHOT")); broadcast(state);
       }
       return;
@@ -428,6 +431,30 @@ void Server::drain_mailbox() {
   }
 }
 void Server::pump(int timeout_ms) { poll_io(timeout_ms); drain_mailbox(); }
+std::int64_t Server::read_monotonic() {
+  const auto now = monotonic_now_();
+  ++monotonic_reads_;
+  return now;
+}
+TickBatch Server::run_scheduler() {
+  if (stopping_) return {};
+  drain_mailbox();
+  ++scheduler_iterations_;
+  const auto now = read_monotonic();
+  if (room_.status() != "RUNNING") {
+    accumulator_.reset(now); last_batch_ = {}; return last_batch_;
+  }
+  auto batch = accumulator_.advance(now);
+  const auto before = room_.executed_ticks();
+  for (int tick = 0; tick < batch.ticks; ++tick) advance_one_tick();
+  batch.ticks = room_.executed_ticks() - before;
+  if (room_.status() != "RUNNING") {
+    accumulator_.reset(now); batch.remaining_ms = 0; batch.overloaded = false;
+  }
+  catch_up_high_water_ = std::max(catch_up_high_water_, batch.ticks);
+  last_batch_ = batch;
+  return batch;
+}
 void Server::advance_one_tick() {
   if (stopping_) return;
   drain_mailbox();
@@ -439,6 +466,7 @@ void Server::advance_one_tick() {
     ++errors_["ACTION_REJECTED"]; queue(failure.connection_id, std::move(error));
   }
   if (room_.status() == "FINISHED") {
+    accumulator_.reset(0); last_batch_ = {};
     Json result = room_.view(); result.update(message("ROOM_FINISHED")); broadcast(result);
   }
 }
@@ -450,7 +478,11 @@ Json Server::metrics() const {
     {"parser_storage_bytes_per_connection", FrameParser::storage_bytes}, {"need_more_events", need_more_events_},
     {"message_error_events", message_error_events_}, {"terminal_frame_events", terminal_frame_events_},
     {"io_end_events", io_end_events_}, {"partial_eof_events", partial_eof_events_},
-    {"partial_writes", partial_writes_}, {"errors", errors_}};
+    {"partial_writes", partial_writes_}, {"errors", errors_},
+    {"scheduler", Json{{"iterations", scheduler_iterations_}, {"monotonic_reads", monotonic_reads_},
+      {"ticks_last_iteration", last_batch_.ticks}, {"catch_up_high_water", catch_up_high_water_},
+      {"remaining_ms", accumulator_.remaining_ms()}, {"overloaded", last_batch_.overloaded},
+      {"operational_state", last_batch_.overloaded ? "OVERLOADED" : "NORMAL"}}}};
 }
 Json Server::cleanup() const {
   std::size_t queued = 0, parser_buffered = 0;
@@ -460,7 +492,8 @@ Json Server::cleanup() const {
   return Json{{"server_connections", connections_.size()}, {"server_descriptors", owned_descriptors().size()},
     {"mailbox_messages", mailbox_.size()}, {"pending_inputs", room_.pending_count()}, {"outbound_messages", queued},
     {"parser_buffered_bytes", parser_buffered}, {"parser_storage_bytes", connections_.size() * FrameParser::storage_bytes},
-    {"worker_threads", 0}, {"timers", 0}, {"disconnect_notifications", disconnected_.size()}};
+    {"worker_threads", 0}, {"timers", 0}, {"disconnect_notifications", disconnected_.size()},
+    {"scheduler_pending_ms", accumulator_.remaining_ms()}};
 }
 std::vector<int> Server::owned_descriptors() const {
   std::vector<int> descriptors;
@@ -475,6 +508,7 @@ void Server::shutdown() {
   listener_.reset();
   drain_mailbox();
   room_.close();
+  accumulator_.reset(0); last_batch_ = {};
   if (room_.status() == "CLOSED") {
     Json state = room_.view(); state.update(message("SNAPSHOT")); broadcast(state);
   }
diff --git a/src/transport.hpp b/src/transport.hpp
index 016cd4e..fada46c 100644
--- a/src/transport.hpp
+++ b/src/transport.hpp
@@ -3,6 +3,7 @@
 #include <array>
 #include <atomic>
 #include <cstddef>
+#include <functional>
 #include <span>
 #include <set>
 
@@ -61,7 +62,8 @@ struct PendingWrite {
 
 class Server {
  public:
-  explicit Server(ManualClock& clock, std::uint16_t port = 0);
+  using MonotonicNow = std::function<std::int64_t()>;
+  explicit Server(ManualClock& clock, std::uint16_t port = 0, MonotonicNow monotonic_now = monotonic_milliseconds);
   ~Server();
   Server(const Server&) = delete;
   Server& operator=(const Server&) = delete;
@@ -70,6 +72,7 @@ class Server {
   void drain_mailbox();
   void pump(int timeout_ms = 0);
   void advance_one_tick();
+  TickBatch run_scheduler();
   void shutdown();
   const Room& room() const { return room_; }
   Json metrics() const;
@@ -108,8 +111,15 @@ class Server {
   void queue(std::uint64_t connection_id, Json value);
   void broadcast(const Json& value);
   void handle(const Envelope& envelope);
+  std::int64_t read_monotonic();
   std::string new_id(const std::string& prefix, std::uint64_t number) const;
   ManualClock& clock_;
+  MonotonicNow monotonic_now_;
+  FixedTickAccumulator accumulator_;
+  TickBatch last_batch_;
+  std::uint64_t scheduler_iterations_ = 0;
+  std::uint64_t monotonic_reads_ = 0;
+  int catch_up_high_water_ = 0;
   Fd reactor_;
   Fd listener_;
   std::uint16_t port_ = 0;
diff --git a/tests/tests.cpp b/tests/tests.cpp
index 434668e..d654c52 100644
--- a/tests/tests.cpp
+++ b/tests/tests.cpp
@@ -158,6 +158,35 @@ void room_permission_matrix_preserves_state() {
 void production_mailbox_capacity_and_drain() {
   std::cout << Json{{"G03_mailbox_probe", run_mailbox_probe(max_mailbox_messages)}}.dump() << '\n';
 }
+void fixed_monotonic_schedule() {
+  constexpr std::array<int, 6> deltas{50,50,125,0,225,50}, expected_ticks{1,1,2,0,4,2}, remaining{0,0,25,25,50,0};
+  Json cases = Json::array();
+  for (bool reverse_wall : {false, true}) {
+    FixedTickAccumulator accumulator;
+    accumulator.reset(0);
+    std::int64_t monotonic = 0, wall = 1700000000000LL;
+    int total = 0;
+    Json trace = Json::array();
+    for (std::size_t index = 0; index < deltas.size(); ++index) {
+      monotonic += deltas[index]; wall += deltas[index];
+      if (reverse_wall && index == 2) wall -= 3600000;
+      const auto batch = accumulator.advance(monotonic);
+      check(batch.ticks == expected_ticks[index] && batch.ticks <= max_catch_up_ticks &&
+        batch.remaining_ms == remaining[index] && batch.overloaded == (index == 4), "fixed accumulator/catch-up/backlog contract");
+      total += batch.ticks;
+      trace.push_back(Json{{"monotonic_ms", monotonic}, {"wall_ms", wall}, {"ticks", batch.ticks},
+        {"remaining_ms", batch.remaining_ms}, {"overloaded", batch.overloaded}});
+    }
+    check(total == 10 && accumulator.remaining_ms() == 0, "500ms retained exactly ten fixed ticks");
+    cases.push_back(trace);
+  }
+  std::cout << Json{{"G04_accumulator_cases", cases}}.dump() << '\n';
+}
+void production_monotonic_adapter() {
+  const auto first = monotonic_milliseconds(), second = monotonic_milliseconds();
+  check(std::chrono::steady_clock::is_steady && second >= first, "production adapter supplies monotonic milliseconds");
+  std::cout << Json{{"G04_adapter", "std::chrono::steady_clock"}, {"readings_ms", Json::array({first, second})}}.dump() << '\n';
+}
 std::vector<std::uint8_t> framed_text(const std::string& text) {
   const auto size = static_cast<std::uint32_t>(text.size());
   std::vector<std::uint8_t> frame{static_cast<std::uint8_t>(size >> 24U), static_cast<std::uint8_t>(size >> 16U),
@@ -266,9 +295,9 @@ void real_tcp_authority_and_cleanup() {
   }
   std::cout << Json{{"integration_evidence", evidence}}.dump() << '\n';
 }
-void standalone_process_shutdown(const std::filesystem::path& executable) {
+void standalone_process_shutdown(const std::filesystem::path& executable, const std::string& clock_mode = "manual") {
   const auto config = std::filesystem::temp_directory_path() / ("arena-g01-server-" + std::to_string(::getpid()) + ".json");
-  write_json_file(config, Json{{"listen_port", 0}, {"clock", "manual"}});
+  write_json_file(config, Json{{"listen_port", 0}, {"clock", clock_mode}});
   int control_pipe[2], output_pipe[2];
   check(::pipe(control_pipe) == 0 && ::pipe(output_pipe) == 0, "process pipe creation");
   Fd control_read(control_pipe[0]), control_write(control_pipe[1]), output_read(output_pipe[0]), output_write(output_pipe[1]);
@@ -304,6 +333,7 @@ void standalone_process_shutdown(const std::filesystem::path& executable) {
   check(output.find('\n') != std::string::npos, "child READY line within fixed process deadline");
   const auto ready_message = Json::parse(output.substr(0, output.find('\n')));
   check(ready_message.at("status") == "READY", "child process reported readiness");
+  check(ready_message.at("clock") == clock_mode, "child process selected requested clock adapter");
   const auto port = ready_message.at("port").get<std::uint16_t>();
   TcpClient client(port);
   client.send(message("HELLO"));
@@ -340,6 +370,12 @@ void standalone_process_shutdown(const std::filesystem::path& executable) {
     if (entry.at("status") == "STOPPED") {
       stopped = true;
       for (const auto& [key, count] : entry.at("cleanup").items()) { (void)key; check(count == 0, "standalone cleanup all zero"); }
+      if (clock_mode == "monotonic") {
+        const auto& scheduler = entry.at("metrics").at("scheduler");
+        check(scheduler.at("iterations").get<std::uint64_t>() > 0 &&
+              scheduler.at("monotonic_reads").get<std::uint64_t>() >= scheduler.at("iterations").get<std::uint64_t>(),
+              "real CLI scheduler used its production monotonic adapter");
+      }
     }
   }
   check(ready && stopped, "standalone readiness and explicit stop observed");
@@ -351,7 +387,7 @@ void standalone_process_shutdown(const std::filesystem::path& executable) {
   check(::bind(rebind.get(), reinterpret_cast<sockaddr*>(&address), sizeof(address)) == 0, "stopped listener port is rebindable");
   rebind.reset();
   std::cout << Json{{"process_exit", 0}, {"reaped", child_guard.reaped}, {"hello_welcome", true},
-    {"sigterm", true}, {"port_rebindable", true}, {"output", output}}.dump() << '\n';
+    {"sigterm", true}, {"port_rebindable", true}, {"clock", clock_mode}, {"output", output}}.dump() << '\n';
 }
 }
 int main(int argc, char** argv) {
@@ -366,10 +402,14 @@ int main(int argc, char** argv) {
       {"G02_strict_protocol_message_recovery", strict_protocol_and_message_recovery},
       {"G02_maximum_frame_transport_end", parser_maximum_and_transport_end},
       {"G03_room_permission_matrix", room_permission_matrix_preserves_state},
-      {"G03_production_mailbox_capacity_drain", production_mailbox_capacity_and_drain}};
+      {"G03_production_mailbox_capacity_drain", production_mailbox_capacity_and_drain},
+      {"G04_fixed_monotonic_schedule", fixed_monotonic_schedule},
+      {"G04_production_monotonic_adapter", production_monotonic_adapter}};
   } else if (std::string(argv[1]) == "integration") {
     tests = {{"real_TCP_authority_and_cleanup", real_tcp_authority_and_cleanup}, {"standalone_process_shutdown", [&] {
-      standalone_process_shutdown(std::filesystem::absolute(argv[0]).parent_path() / "arena"); }}};
+      standalone_process_shutdown(std::filesystem::absolute(argv[0]).parent_path() / "arena"); }},
+      {"G04_standalone_monotonic_adapter", [&] {
+      standalone_process_shutdown(std::filesystem::absolute(argv[0]).parent_path() / "arena", "monotonic"); }}};
   } else { std::cerr << "unknown suite\n"; return 2; }
   std::size_t passed = 0;
   for (const auto& [name, test] : tests) {
