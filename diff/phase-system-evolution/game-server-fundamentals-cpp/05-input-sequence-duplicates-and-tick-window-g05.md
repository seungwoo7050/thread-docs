# Input Sequence, Duplicate와 Tick Window (G05)

## `G05: enforce input sequence identity and target tick window`

diff --git a/evidence/G05-runs.jsonl b/evidence/G05-runs.jsonl
new file mode 100644
index 0000000..c4c5905
--- /dev/null
+++ b/evidence/G05-runs.jsonl
@@ -0,0 +1,7 @@
+{"category":"compile","units":1,"label":"baseline-compile","argv":["clang++","-std=c++20","-O2","-Wall","-Wextra","-Wpedantic","-Werror","-fsanitize=thread","-g","-I","src","-I","/opt/homebrew/include","artifacts/g05/reproduce.cpp","src/game.cpp","src/transport.cpp","src/scenario.cpp","-o","artifacts/g05/reproduce"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","started_at":"2026-08-28T02:50:24.398524+00:00","duration_seconds":18.504448,"exit":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g05/baseline-compile.log"}
+{"category":"unit","units":1,"label":"baseline","argv":["env","TSAN_OPTIONS=halt_on_error=1","./artifacts/g05/reproduce","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G05.json","artifacts/g05/baseline.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","started_at":"2026-08-28T02:52:15.282777+00:00","duration_seconds":1.289838,"exit":1,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g05/baseline.log"}
+{"category":"compile","units":2,"label":"build","argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g05-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g05/track","ARENA_TSAN=ON","./track","build"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","started_at":"2026-08-28T03:00:15.416096+00:00","duration_seconds":43.396636,"exit":2,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g05/build.log"}
+{"category":"compile","units":1,"label":"build-retry","argv":["cmake","--build","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g05-tsan","--parallel","2"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","started_at":"2026-08-28T03:01:15.224342+00:00","duration_seconds":5.938933,"exit":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g05/build-retry.log"}
+{"category":"unit","units":1,"label":"unit","argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g05-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g05/track","TSAN_OPTIONS=halt_on_error=1","./track","unit-test"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","started_at":"2026-08-28T03:02:06.263905+00:00","duration_seconds":1.877621,"exit":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g05/unit.log"}
+{"category":"integration","units":1,"label":"integration","argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g05-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g05/track","TSAN_OPTIONS=halt_on_error=1","./track","integration-test"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","started_at":"2026-08-28T03:02:28.221960+00:00","duration_seconds":1.876848,"exit":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g05/integration.log"}
+{"category":"canonical","units":1,"label":"canonical","argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g05-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g05/track","TSAN_OPTIONS=halt_on_error=1","./track","scenario-run","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G05.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g05/canonical.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","started_at":"2026-08-28T03:03:07.964364+00:00","duration_seconds":0.39297,"exit":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g05/canonical.log"}
diff --git a/evidence/G05.md b/evidence/G05.md
new file mode 100644
index 0000000..a47150a
--- /dev/null
+++ b/evidence/G05.md
@@ -0,0 +1,42 @@
+# G05 — sequence identity and tick-window admission
+
+THREAD G05; BRANCH `track/fundamentals-cpp`; PROFILE `realtime-core`; ATTEMPT initial.
+SPEC_REVISION `5a6e4a2f8fc71d4be18c3279583bfc2558d5c232`.
+START `6572ed342edb5eb22cbcaa11b8d7d62ca37d4134`.
+Frozen input SHA-256 `971899b4bcca18c0087c085e0c0824bb5d678661c0b474f6b165d5723da47ba9`.
+
+## Preserved baseline
+
+All eight production source files byte-match START in `artifacts/g05/pre-change-production.json` (SHA-256 `f4dc3492fb1a88676745b7e115048ac7f3e19e0e7418fa9fa1e44e7e532a0d41`). The baseline links those unchanged sources and reuses the existing real-TCP setup/cleanup fixture. The exact resolved reproduction command was recorded before execution and production edits:
+
+```sh
+python3 artifacts/g05/run.py unit 1 baseline env TSAN_OPTIONS=halt_on_error=1 ./artifacts/g05/reproduce /Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G05.json artifacts/g05/baseline.json
+```
+
+Actual `FrameParser → Server::drain_mailbox → Room::input/tick` execution ACKed all ten attempts `[1,1,3,2,4,4,5,6,7,8]`, without duplicate identification or rejections. Seven observed positions were `[[10400,10000],[10400,9600],[10400,9200],[10400,8800],[10400,8400],[10400,8000],[10400,7600]]`. Expected new-constraint failure: exit1, 1.289838s. Existing pending bound64 was source-observed, not saturated in this baseline; that guarantee is NOT_REPRODUCED. All six descriptors closed and active cleanup counters were zero. Raw `baseline.json` SHA-256 `fe452a3f083df918778432735384bdac11d128a277970d48a3ed1b56c09e3058`. Main was notified before production edits and independently checked these observations.
+
+## Scope and verification
+
+Input identity uses exact uint64 sequences and typed known fields only; unknown fields remain ignored. Exact signed/unsigned target integers are classified against the next tick through next+4 without floating conversion or negative clamping. Accepted candidates remain bounded at64; only the highest accepted sequence due on a tick supplies its direction/TAG, and future candidates remain queued. A single last accepted payload per player supports duplicate/conflict checks; one current-tick applied sequence supports direct evidence. No rate limiting, replay, networking expansion or game-rule change.
+
+Earlier harnesses supply seq/target_tick at their original admission boundaries. Directions, actions, physical assertions and tick schedules stay unchanged. The G05 runner reads the actual shared file; common logical values come from real ACK/error, authoritative player and cleanup observations. Main specified role-keyed `scores` as evidence normalization.
+
+`G05-runs.jsonl` is the command ledger: exact argv, category, UTC time, duration, exit and raw path, including the expected baseline failure. Planned fixed verification uses TSan in `build-g05-tsan`: track build (configure+compile), complete unit suite, complete integration suite, and one post-change canonical invocation. Supplemental pure cases are exactly pending1..65, unsigned `[1,max,max,max-1]`, and seven isolated invalid integer forms. Caps: compile/configure8, unit4 including baseline, integration2, post canonical1; fault/load0.
+
+## Actual results
+
+| Ledger command | Exit | Observation / seconds |
+|---|---:|---|
+| build | 2 | Production core/executable compiled; one test string literal missed its closing quote; 43.396636s |
+| build-retry | 0 | Corrected only that literal; warnings-as-errors/TSan build complete; 5.938933s |
+| unit | 0 | 16/16, including all thirteen earlier units; 1.877621s |
+| integration | 0 | 3/3, unchanged authority and manual/monotonic process checks; 1.876848s |
+| canonical | 0 | Actual frozen ten-event/seven-tick input; 0.392970s |
+
+Actual accepted sequences `[1,3,4,6,7]`, duplicate `[1]`, and rejection order `2:INPUT_STALE, 4:SEQUENCE_CONFLICT, 5:INPUT_LATE, 8:INPUT_TOO_EARLY`. Applied sequences `[1,4,6,null,null,null,7]`; positions `[[10400,10000],[10000,10000],[10400,10000],[10800,10000],[11200,10000],[11600,10000],[11600,10400]]`. Last accepted7, bravo `[90000,90000]`, scores `{"alpha":0,"bravo":0}`, RUNNING after seven ticks. Raw tick1 retains accepted3/4 before selecting4; future7 remains pending through ticks2–5. Duplicate/rejected attempts leave the accepted identity and pending candidates unchanged.
+
+Pure probes observed all64 retained, 65th `INPUT_QUEUE_FULL`, selected64 and one400-unit movement; unsigned max `18446744073709551615` accepted/duplicated exactly, max−1 stale, one movement; all seven isolated invalid forms `MESSAGE_INVALID` with unchanged state. TSan reported no error. Canonical high water: pending2/64, mailbox1/512, control3/64, parser259/16388 bytes, connections2/512. Existing pure mailbox/parser regressions reached their exact512/16388 bounds. One accepted identity cache per player remains bounded historical data. All six canonical descriptors closed, active resource counts zero, `all_resources_released=true`; both integration child processes exited0, were reaped, and released rebindable ports.
+
+Final budget: compile/configure **4/8**, unit **2/4** including reproduction, integration **1/2**, post canonical **1/1**. The expected baseline exit1 and corrected test compilation exit2 are both preserved. Canonical SHA-256 `a82f334428a9271c90448a18ea1925ee8ac53ec266e7f461081e504ebf9799ad`; shared input SHA rechecked unchanged. UNRESOLVED: no known G05 failure; independent main verification/comparison pending.
+
+STATE_HASHES inactive until G07; NETWORK_FAULT_RUNS0; LOAD_RUNS0.
diff --git a/src/game.cpp b/src/game.cpp
index b184f64..565a46d 100644
--- a/src/game.cpp
+++ b/src/game.cpp
@@ -71,7 +71,7 @@ Player& Room::join(std::string player_id, std::string session_id, std::uint64_t
   const int slot = next_slot_++;
   Player player{std::move(player_id), std::move(session_id), connection_id, slot,
                 spawns[static_cast<std::size_t>(slot)][0], spawns[static_cast<std::size_t>(slot)][1],
-                Direction::stop, 0, -20, true, {}};
+                Direction::stop, 0, -20, true, {}, {}, {}};
   const auto key = player.id;
   auto [found, inserted] = players_.emplace(key, std::move(player));
   if (!inserted) throw std::logic_error("server generated duplicate player id");
@@ -79,16 +79,31 @@ Player& Room::join(std::string player_id, std::string session_id, std::uint64_t
   if (ready >= 2) transition("RUNNING");
   return found->second;
 }
-std::optional<std::string> Room::input(const std::string& player_id, Intent intent) {
+InputResult Room::input(const std::string& player_id, Input input) {
   assert_owner();
-  if (status_ != "RUNNING") return "ROOM_NOT_RUNNING";
+  if (status_ != "RUNNING") return {"ROOM_NOT_RUNNING", false};
   auto found = players_.find(player_id);
-  if (found == players_.end() || !found->second.connected) return "PLAYER_MISMATCH";
-  auto& queue = found->second.pending;
-  if (queue.size() == max_pending_inputs) return "INPUT_QUEUE_FULL";
-  queue.push_back(std::move(intent));
+  if (found == players_.end() || !found->second.connected) return {"PLAYER_MISMATCH", false};
+  if (input.seq == 0) return {"MESSAGE_INVALID", false};
+  // Normalize positive signed integers without truncating unsigned values.
+  if (const auto* tick = std::get_if<std::int64_t>(&input.target_tick); tick && *tick >= 0)
+    input.target_tick = static_cast<std::uint64_t>(*tick);
+  auto& player = found->second;
+  if (input.seq < player.last_accepted_seq()) return {"INPUT_STALE", false};
+  if (input.seq == player.last_accepted_seq()) {
+    if (input == *player.last_accepted_input) return {{}, true};
+    return {"SEQUENCE_CONFLICT", false};
+  }
+  const auto* tick = std::get_if<std::uint64_t>(&input.target_tick);
+  const auto next_tick = static_cast<std::uint64_t>(executed_ticks_);
+  if (tick == nullptr || *tick < next_tick) return {"INPUT_LATE", false};
+  if (*tick > next_tick + max_future_input_ticks) return {"INPUT_TOO_EARLY", false};
+  auto& queue = player.pending;
+  if (queue.size() == max_pending_inputs) return {"INPUT_QUEUE_FULL", false};
+  queue.push_back(input);
+  player.last_accepted_input = std::move(input);
   input_high_water_ = std::max(input_high_water_, queue.size());
-  return std::nullopt;
+  return {};
 }
 std::vector<ActionFailure> Room::tick() {
   assert_owner();
@@ -96,12 +111,21 @@ std::vector<ActionFailure> Room::tick() {
   if (status_ != "RUNNING") return failures;
   std::map<std::string, std::string> tags;
   for (auto& [id, player] : players_) {
+    player.applied_seq.reset();
     if (!player.connected) continue;
-    if (!player.pending.empty()) {
-      const Intent& intent = player.pending.back();
-      player.direction = intent.direction;
-      if (intent.tag_target) tags.emplace(id, *intent.tag_target);
-      player.pending.clear();
+    std::optional<Input> selected;
+    for (auto it = player.pending.begin(); it != player.pending.end();) {
+      if (std::get<std::uint64_t>(it->target_tick) == static_cast<std::uint64_t>(executed_ticks_)) {
+        if (!selected || it->seq > selected->seq) selected = *it;
+        it = player.pending.erase(it);
+      } else {
+        ++it;
+      }
+    }
+    if (selected) {
+      player.applied_seq = selected->seq;
+      player.direction = selected->intent.direction;
+      if (selected->intent.tag_target) tags.emplace(id, *selected->intent.tag_target);
     }
     switch (player.direction) {
       case Direction::stop: break;
@@ -144,6 +168,7 @@ void Room::leave(std::uint64_t connection_id) {
       player.connected = false;
       player.direction = Direction::stop;
       player.pending.clear();
+      player.applied_seq.reset();
     }
   }
 }
@@ -155,6 +180,7 @@ void Room::close() {
     player.connected = false;
     player.direction = Direction::stop;
     player.pending.clear();
+    player.applied_seq.reset();
   }
   transition("CLOSED");
 }
diff --git a/src/game.hpp b/src/game.hpp
index 1c267a5..819ddd7 100644
--- a/src/game.hpp
+++ b/src/game.hpp
@@ -5,6 +5,7 @@
 #include <optional>
 #include <string>
 #include <thread>
+#include <variant>
 #include <vector>
 #include <nlohmann/json.hpp>
 
@@ -19,6 +20,7 @@ inline constexpr std::size_t max_mailbox_messages = max_players * max_pending_in
 inline constexpr int session_ticks = 1'200;
 inline constexpr int tick_duration_ms = 50;
 inline constexpr int max_catch_up_ticks = 4;
+inline constexpr std::uint64_t max_future_input_ticks = 4;
 
 enum class Direction { stop, north, east, south, west };
 std::string direction_name(Direction direction);
@@ -29,6 +31,19 @@ Json error_message(const std::string& code, const std::string& description);
 struct Intent {
   Direction direction = Direction::stop;
   std::optional<std::string> tag_target;
+  bool operator==(const Intent&) const = default;
+};
+struct Input {
+  std::uint64_t seq = 0;
+  // JSON integers can be negative or span the whole unsigned64 range. Keep
+  // both exact until the owner classifies the target window.
+  std::variant<std::int64_t, std::uint64_t> target_tick = std::uint64_t{0};
+  Intent intent;
+  bool operator==(const Input&) const = default;
+};
+struct InputResult {
+  std::optional<std::string> error;
+  bool duplicate = false;
 };
 struct Player {
   std::string id;
@@ -41,7 +56,10 @@ struct Player {
   int score = 0;
   int last_tag_tick = -20;
   bool connected = true;
-  std::deque<Intent> pending;
+  std::deque<Input> pending;
+  std::optional<Input> last_accepted_input;
+  std::optional<std::uint64_t> applied_seq;
+  std::uint64_t last_accepted_seq() const { return last_accepted_input ? last_accepted_input->seq : 0; }
 };
 struct ActionFailure {
   std::uint64_t connection_id;
@@ -54,7 +72,7 @@ class Room {
   Room();
   void create(std::string id);
   Player& join(std::string player_id, std::string session_id, std::uint64_t connection_id);
-  std::optional<std::string> input(const std::string& player_id, Intent intent);
+  InputResult input(const std::string& player_id, Input input);
   std::vector<ActionFailure> tick();
   void leave(std::uint64_t connection_id);
   void close();
diff --git a/src/lifecycle_scenario.cpp b/src/lifecycle_scenario.cpp
index 771d30b..653a131 100644
--- a/src/lifecycle_scenario.cpp
+++ b/src/lifecycle_scenario.cpp
@@ -49,6 +49,10 @@ struct LifecycleFixture {
     value["session_id"] = peer.session;
     if (type != "CREATE_ROOM") value["room_id"] = room_id;
     if (type == "INPUT" || type == "LEAVE_ROOM") value["player_id"] = peer.player;
+    if (type == "INPUT") {
+      // Earlier fixed fixtures each send one legitimate input per player.
+      value["seq"] = 1; value["target_tick"] = server.room().executed_ticks(); value["owner_epoch"] = 0;
+    }
     return value;
   }
   void hello(const std::string& role) {
@@ -336,6 +340,115 @@ Json run_lifecycle_scenario(const Json& scenario) {
   evidence["result"] = "PASS";
   return evidence;
 }
+Json run_input_scenario(const Json& scenario) {
+  const auto check_input = [](bool condition, const std::string& text) {
+    if (!condition) throw std::runtime_error("G05: " + text);
+  };
+  check_input(scenario.at("thread") == "G05" && scenario.at("contract_version") == 1 && scenario.at("seed") == 7050 &&
+    scenario.at("clock").at("kind") == "manual" && scenario.at("clock").at("tick_duration_ms") == tick_duration_ms &&
+    scenario.at("clients") == Json::array({"alpha","bravo"}) && scenario.at("ticks") == 7 &&
+    scenario.at("events").size() == 10 && scenario.at("max_future_input_ticks") == max_future_input_ticks &&
+    scenario.at("pending_input_bound") == max_pending_inputs && scenario.at("socket_ceiling_ms") == 5000,
+    "fixed input scenario constants changed");
+  const auto pending_record = [](const Player& player) {
+    Json rows = Json::array();
+    for (const auto& input : player.pending) rows.push_back(Json{{"seq",input.seq},
+      {"target_tick",std::get<std::uint64_t>(input.target_tick)}, {"direction",direction_name(input.intent.direction)},
+      {"tag_target_player_id",input.intent.tag_target ? Json(*input.intent.tag_target) : Json(nullptr)}});
+    return rows;
+  };
+  const auto descriptors_before = Fd::live();
+  LifecycleFixture fixture(scenario.at("socket_ceiling_ms").get<int>()); fixture.setup(2);
+  auto& alpha = fixture.peers.at("alpha");
+  const auto& player = fixture.server.room().players().at(alpha.player);
+  Json logical{{"accepted_sequences",Json::array()}, {"duplicate_sequences",Json::array()}, {"rejections",Json::array()},
+    {"applied_sequences",Json::array()}, {"alpha_positions",Json::array()}};
+  Json evidence{{"scenario_id",scenario.at("scenario_id")}, {"thread","G05"}, {"contract_version",1},
+    {"transport","production/real-loopback-TCP/kqueue/manual-owner-drain"}, {"identities",fixture.identities()},
+    {"events",Json::array()}, {"ticks",Json::array()}, {"state_hashes","INACTIVE_UNTIL_G07"}};
+  Json directions = Json::array();
+  std::size_t consumed = 0;
+  for (int tick = 0; tick < scenario.at("ticks").get<int>(); ++tick) {
+    for (const auto& event : scenario.at("events")) {
+      if (event.at("before_tick") != tick) continue;
+      check_input(event.at("client") == "alpha" && event.at("owner_epoch") == 0, "fixed actor/epoch changed");
+      ++consumed;
+      auto request = fixture.request(event.at("client").get<std::string>(), "INPUT");
+      for (const char* key : {"seq","target_tick","direction","tag_target_player_id","owner_epoch"}) request[key] = event.at(key);
+      const auto before = fixture.server.room().view();
+      const auto pending_before = player.pending;
+      const auto last_before = player.last_accepted_input;
+      const auto last_seq_before = player.last_accepted_seq();
+      alpha.tcp->send(request);
+      Json response;
+      do { response = alpha.tcp->receive(fixture.server); }
+      while (response.at("type") != "INPUT_ACK" && response.at("type") != "ERROR");
+      check_input(fixture.server.room().view() == before && fixture.clock.now_ms == tick * tick_duration_ms,
+                  "admission must not simulate movement or change another player");
+      if (response.at("type") == "ERROR") {
+        logical["rejections"].push_back(Json{{"seq",event.at("seq")},{"code",response.at("code")}});
+        check_input(player.pending == pending_before && player.last_accepted_input == last_before,
+                    "rejected input changed pending inputs or accepted sequence identity");
+      } else {
+        check_input(response.at("accepted") == true && response.at("seq") == event.at("seq") &&
+          response.at("player_id") == alpha.player && response.at("tick") == tick &&
+          response.at("last_accepted_seq") == player.last_accepted_seq(), "wire ACK disagrees with authoritative admission");
+        if (response.at("code") == "DUPLICATE") {
+          logical["duplicate_sequences"].push_back(response.at("seq"));
+          check_input(player.pending == pending_before && player.last_accepted_input == last_before,
+                      "duplicate added another effect or advanced accepted identity");
+        } else {
+          check_input(response.at("code") == "ACCEPTED" && player.last_accepted_seq() == event.at("seq") &&
+                      player.pending.size() == pending_before.size() + 1, "unique accepted input was lost");
+          logical["accepted_sequences"].push_back(response.at("seq"));
+        }
+      }
+      evidence["events"].push_back(Json{{"before_tick",tick}, {"request",request}, {"response",response},
+        {"last_accepted_seq_before",last_seq_before}, {"last_accepted_seq_after",player.last_accepted_seq()},
+        {"pending_before",pending_before.size()}, {"pending_after",pending_record(player)}, {"physical_state",before}});
+    }
+    const auto pending_before_tick = pending_record(player);
+    fixture.server.advance_one_tick();
+    const Json applied = player.applied_seq ? Json(*player.applied_seq) : Json(nullptr);
+    logical["applied_sequences"].push_back(applied);
+    logical["alpha_positions"].push_back(Json::array({player.x,player.y}));
+    directions.push_back(direction_name(player.direction));
+    check_input(fixture.server.room().executed_ticks() == tick + 1 && fixture.clock.now_ms == (tick + 1) * tick_duration_ms,
+                "sequence gap stalled or changed the fixed simulation tick");
+    evidence["ticks"].push_back(Json{{"tick",tick}, {"applied_seq",applied}, {"pending_before",pending_before_tick},
+      {"pending_after",pending_record(player)}, {"state",fixture.server.room().view()}, {"simulation_ms",fixture.clock.now_ms}});
+  }
+  check_input(consumed == scenario.at("events").size(), "event fell outside the frozen timeline");
+  const auto& bravo = fixture.server.room().players().at(fixture.peers.at("bravo").player);
+  logical["executed_ticks"] = fixture.server.room().executed_ticks();
+  logical["final_last_accepted_seq"] = player.last_accepted_seq();
+  logical["bravo_position"] = Json::array({bravo.x,bravo.y});
+  logical["scores"] = Json{{"alpha",player.score},{"bravo",bravo.score}};
+  check_input(logical.at("accepted_sequences") == Json::array({1,3,4,6,7}) &&
+    logical.at("duplicate_sequences") == Json::array({1}) && logical.at("rejections") == Json::array({
+      Json{{"seq",2},{"code","INPUT_STALE"}}, Json{{"seq",4},{"code","SEQUENCE_CONFLICT"}},
+      Json{{"seq",5},{"code","INPUT_LATE"}}, Json{{"seq",8},{"code","INPUT_TOO_EARLY"}}}), "fixed admission matrix differs");
+  check_input(logical.at("applied_sequences") == Json::array({1,4,6,nullptr,nullptr,nullptr,7}) &&
+    directions == Json::array({"EAST","WEST","EAST","EAST","EAST","EAST","NORTH"}) &&
+    logical.at("alpha_positions") == Json::array({{10400,10000},{10000,10000},{10400,10000},{10800,10000},
+      {11200,10000},{11600,10000},{11600,10400}}), "highest sequence/due-tick effect differs");
+  check_input(logical.at("final_last_accepted_seq") == 7 && logical.at("bravo_position") == Json::array({90000,90000}) &&
+    logical.at("scores") == Json{{"alpha",0},{"bravo",0}} && fixture.server.room().status() == "RUNNING" &&
+    fixture.server.room().pending_count() == 0, "final identity, unrelated player, score, lifecycle or pending state differs");
+  evidence["directions"] = directions; evidence["executed_ticks"] = fixture.server.room().executed_ticks();
+  evidence["metrics"] = fixture.server.metrics();
+  const auto& metrics = evidence.at("metrics");
+  check_input(metrics.at("parser_buffer_high_water").get<std::size_t>() <= FrameParser::storage_bytes &&
+    metrics.at("input_per_player_high_water").get<std::size_t>() <= max_pending_inputs &&
+    metrics.at("outbound_control_high_water").get<std::size_t>() <= max_control_messages &&
+    metrics.at("mailbox_high_water").get<std::size_t>() <= max_mailbox_messages &&
+    metrics.at("connection_high_water") == 2, "fixed scenario resource bound exceeded");
+  evidence["cleanup"] = fixture.finish();
+  logical["all_resources_released"] = Fd::live() == descriptors_before;
+  check_input(logical.at("all_resources_released").get<bool>(), "input scenario retained resources");
+  evidence["logical"] = logical; evidence["result"] = "PASS";
+  return evidence;
+}
 
 Json run_clock_scenario(const Json& scenario) {
   const auto check_clock = [](bool condition, const std::string& text) {
diff --git a/src/scenario.cpp b/src/scenario.cpp
index de0a555..bda4ad1 100644
--- a/src/scenario.cpp
+++ b/src/scenario.cpp
@@ -15,6 +15,7 @@ struct Participant {
   std::string session_id;
   std::string player_id;
   int slot = -1;
+  std::uint64_t next_seq = 1;
 };
 Json for_player(const std::string& type, const Participant& participant, const std::string& room_id) {
   Json value = message(type);
@@ -284,6 +285,7 @@ Json run_scenario(const Json& scenario) {
   if (scenario.at("thread") == "G02") return run_framing_scenario(scenario);
   if (scenario.at("thread") == "G03") return run_lifecycle_scenario(scenario);
   if (scenario.at("thread") == "G04") return run_clock_scenario(scenario);
+  if (scenario.at("thread") == "G05") return run_input_scenario(scenario);
   require(scenario.at("contract_version") == 1 && scenario.at("thread") == "G01", "only G01 contract v1 is active");
   require(scenario.at("clock").at("kind") == "manual" && scenario.at("clock").at("tick_duration_ms") == tick_duration_ms,
           "G01 runner requires the fixed 50ms manual clock");
@@ -303,7 +305,7 @@ Json run_scenario(const Json& scenario) {
   for (const auto& role_value : scenario.at("clients")) {
     const auto role = role_value.get<std::string>();
     require(role.size() <= 64 && !clients.contains(role), "client role must be unique and bounded");
-    clients.emplace(role, Participant{std::make_unique<TcpClient>(server.port()), {}, {}, -1});
+    clients.emplace(role, Participant{std::make_unique<TcpClient>(server.port()), {}, {}, -1, 1});
   }
   for (const auto& step : scenario.at("setup")) {
     const auto role = step.at("client").get<std::string>();
@@ -335,6 +337,10 @@ Json run_scenario(const Json& scenario) {
       const auto role = input.at("client").get<std::string>();
       auto& participant = clients.at(role);
       Json request = for_player("INPUT", participant, room_id);
+      // G05 fields at the original G01 admission boundary; physical input and
+      // tick timing are unchanged.
+      request["seq"] = participant.next_seq++; request["target_tick"] = tick;
+      request["owner_epoch"] = 0;
       request["direction"] = input.at("direction");
       request["tag_target_player_id"] = nullptr;
       if (!input.at("tag_target").is_null()) {
diff --git a/src/scenario.hpp b/src/scenario.hpp
index 6cdb488..2f82733 100644
--- a/src/scenario.hpp
+++ b/src/scenario.hpp
@@ -8,4 +8,5 @@ Json run_scenario(const Json& scenario);
 Json run_lifecycle_scenario(const Json& scenario);
 Json run_mailbox_probe(std::size_t capacity);
 Json run_clock_scenario(const Json& scenario);
+Json run_input_scenario(const Json& scenario);
 }
diff --git a/src/transport.cpp b/src/transport.cpp
index 34ef148..fd1a607 100644
--- a/src/transport.cpp
+++ b/src/transport.cpp
@@ -72,12 +72,33 @@ std::string request_error(const Json& value) {
   if (type == "JOIN_ROOM") return {};
   if (!string_field("player_id")) return "MESSAGE_INVALID";
   if (type == "LEAVE_ROOM") return {};
-  if (!string_field("direction") || !value.contains("tag_target_player_id")) return "MESSAGE_INVALID";
-  const auto& target = value.at("tag_target_player_id");
-  if (!target.is_null() && !target.is_string()) return "MESSAGE_INVALID";
+  // Shared with owner admission and pure wire-to-domain integer probes.
+  (void)decode_input(value);
   return {};
 }
 }
+Input decode_input(const Json& value) {
+  const auto& sequence = value.at("seq");
+  const auto& target_tick = value.at("target_tick");
+  if (!sequence.is_number_integer() || !target_tick.is_number_integer() ||
+      (!sequence.is_number_unsigned() && sequence.get<std::int64_t>() <= 0))
+    throw std::invalid_argument("seq and target_tick must be exact integers; seq starts at1");
+  const auto seq = sequence.get<std::uint64_t>();
+  if (seq == 0) throw std::invalid_argument("seq starts at1");
+  const auto direction = parse_direction(value.at("direction").get<std::string>());
+  if (!direction) throw std::invalid_argument("direction is not a cardinal enum");
+  std::optional<std::string> tag;
+  if (!value.at("tag_target_player_id").is_null()) tag = value.at("tag_target_player_id").get<std::string>();
+  if (tag && tag->size() > 64) throw std::invalid_argument("identifier too long");
+  if (value.contains("owner_epoch") && (!value.at("owner_epoch").is_number_integer() || value.at("owner_epoch") != 0))
+    throw std::invalid_argument("owner_epoch is fixed at0 before G15");
+  Input input{seq, std::uint64_t{0}, Intent{*direction, std::move(tag)}};
+  if (target_tick.is_number_unsigned()) input.target_tick = target_tick.get<std::uint64_t>();
+  else input.target_tick = target_tick.get<std::int64_t>();
+  // Version/identities are validated separately, epoch is fixed, and unknown
+  // fields are ignored. Only these typed logical fields define a retry.
+  return input;
+}
 std::atomic<int> Fd::live_{0};
 Fd::Fd(int value) : value_(value) { if (value_ >= 0) ++live_; }
 Fd::~Fd() { reset(); }
@@ -404,19 +425,19 @@ void Server::handle(const Envelope& envelope) {
       room_.leave(id);
       Json state = room_.view(); state.update(message("SNAPSHOT")); queue(id, state); return;
     }
-    const auto direction = parse_direction(value.at("direction").get<std::string>());
-    if (!direction) { reject("MESSAGE_INVALID", "direction is not a cardinal enum"); return; }
-    std::optional<std::string> target;
-    const auto& tag = value.at("tag_target_player_id");
-    if (!tag.is_null()) target = tag.get<std::string>();
-    if (target && target->size() > 64) { reject("MESSAGE_INVALID", "identifier too long"); return; }
-    if (const auto error = room_.input(conn->player_id, Intent{*direction, target})) {
-      reject(*error, "input was not accepted"); return;
+    const auto input = decode_input(value);
+    const auto result = room_.input(conn->player_id, input);
+    if (result.error) {
+      reject(*result.error, "input was not accepted"); return;
     }
     Json reply = message("INPUT_ACK"); reply["player_id"] = conn->player_id; reply["accepted"] = true;
+    reply["seq"] = input.seq; reply["code"] = result.duplicate ? "DUPLICATE" : "ACCEPTED";
+    reply["last_accepted_seq"] = room_.players().at(conn->player_id).last_accepted_seq();
     reply["tick"] = room_.executed_ticks(); queue(id, std::move(reply));
   } catch (const Json::exception&) {
     reject("MESSAGE_INVALID", "required field missing or wrong type");
+  } catch (const std::invalid_argument&) {
+    reject("MESSAGE_INVALID", "invalid input field");
   }
 }
 void Server::drain_mailbox() {
diff --git a/src/transport.hpp b/src/transport.hpp
index fada46c..565b641 100644
--- a/src/transport.hpp
+++ b/src/transport.hpp
@@ -26,6 +26,7 @@ class Fd {
 
 std::vector<std::uint8_t> encode_frame(const Json& value);
 Json decode_complete_frame(std::span<const std::uint8_t> bytes);
+Input decode_input(const Json& value);
 
 enum class ParseState { need_more, message, message_error, terminal_frame_error, io_end };
 std::string parse_state_name(ParseState state);
diff --git a/tests/tests.cpp b/tests/tests.cpp
index d654c52..cf5caa9 100644
--- a/tests/tests.cpp
+++ b/tests/tests.cpp
@@ -6,6 +6,7 @@
 #include <filesystem>
 #include <functional>
 #include <iostream>
+#include <limits>
 #include <netinet/in.h>
 #include <poll.h>
 #include <spawn.h>
@@ -25,6 +26,11 @@ void populate(Room& room) {
   room.join("player-00", "session-00", 1);
   room.join("player-01", "session-01", 2);
 }
+std::optional<std::string> submit_next(Room& room, const std::string& player_id, Intent intent) {
+  // Supply newly active protocol fields at each original test boundary.
+  return room.input(player_id, Input{room.players().at(player_id).last_accepted_seq() + 1,
+    static_cast<std::uint64_t>(room.executed_ticks()), std::move(intent)}).error;
+}
 void lifecycle_and_duration() {
   Room room;
   room.create("unit-room");
@@ -42,21 +48,21 @@ void lifecycle_and_duration() {
   }
   room.tick();
   check(room.executed_ticks() == 1200 && room.view().at("tick") == 1199, "finished tick does not advance");
-  check(room.input("player-00", {}) == "ROOM_NOT_RUNNING", "finished input rejected");
+  check(submit_next(room, "player-00", {}) == "ROOM_NOT_RUNNING", "finished input rejected");
   room.close();
   check(room.lifecycle() == std::vector<std::string>({"LOBBY", "RUNNING", "FINISHED", "CLOSED"}), "complete lifecycle");
   check(room.pending_count() == 0, "close clears input");
 }
 void movement_is_integer_and_bounded() {
   Room room; populate(room);
-  room.input("player-00", {Direction::east, std::nullopt});
-  room.input("player-01", {Direction::south, std::nullopt});
+  submit_next(room, "player-00", {Direction::east, std::nullopt});
+  submit_next(room, "player-01", {Direction::south, std::nullopt});
   room.tick(); room.tick();
   check(room.players().at("player-00").x == 10800, "direction persists at 400 integer units per tick");
   check(room.players().at("player-01").y == 89200, "south moves negative y");
   for (int tick = 0; tick < 300; ++tick) room.tick();
   check(room.players().at("player-00").x == 100000 && room.players().at("player-01").y == 0, "arena clamps both bounds");
-  room.input("player-00", {Direction::stop, std::nullopt}); room.tick();
+  submit_next(room, "player-00", {Direction::stop, std::nullopt}); room.tick();
   check(room.players().at("player-00").x == 100000, "STOP stops movement");
 }
 void tag_uses_wide_distance_and_one_shot_intent() {
@@ -67,11 +73,11 @@ void tag_uses_wide_distance_and_one_shot_intent() {
   // Unit-only setup of exact arithmetic boundaries. The production network
   // never exposes position/score mutation to clients.
   actor.x = 0; actor.y = 0; target.x = 100000; target.y = 100000;
-  room.input(actor.id, {Direction::stop, target.id});
+  submit_next(room, actor.id, {Direction::stop, target.id});
   const auto failures = room.tick();
   check(failures.size() == 1 && actor.score == 0 && actor.last_tag_tick == -20, "20,000,000,000 squared distance rejected without overflow");
   actor.x = 50000; actor.y = 50000; target.x = 52500; target.y = 50000;
-  room.input(actor.id, {Direction::stop, target.id});
+  submit_next(room, actor.id, {Direction::stop, target.id});
   check(room.tick().empty() && actor.score == 1 && target.score == 0, "inclusive TAG range awards actor only");
   const int success_tick = actor.last_tag_tick;
   for (int tick = 0; tick < 40; ++tick) room.tick();
@@ -80,8 +86,8 @@ void tag_uses_wide_distance_and_one_shot_intent() {
 void input_capacity_is_explicit() {
   Room room; populate(room);
   for (std::size_t input = 0; input < max_pending_inputs; ++input)
-    check(!room.input("player-00", {Direction::east, std::nullopt}), "first 64 pending accepted");
-  check(room.input("player-00", {Direction::west, std::nullopt}) == "INPUT_QUEUE_FULL", "65th pending rejected explicitly");
+    check(!submit_next(room, "player-00", {Direction::east, std::nullopt}), "first 64 pending accepted");
+  check(submit_next(room, "player-00", {Direction::west, std::nullopt}) == "INPUT_QUEUE_FULL", "65th pending rejected explicitly");
   check(room.input_high_water() == 64 && room.pending_count() == 64, "bounded pending metric");
   room.tick();
   check(room.pending_count() == 0 && room.players().at("player-00").x == 10400, "one movement per tick despite many intents");
@@ -119,7 +125,7 @@ void foreign_thread_mutation_is_rejected() {
   Room room; populate(room);
   std::atomic<bool> rejected{false};
   std::thread foreign([&] {
-    try { room.input("player-00", {Direction::east, std::nullopt}); }
+    try { room.input("player-00", Input{1, std::uint64_t{0}, {Direction::east, std::nullopt}}); }
     catch (const std::logic_error&) { rejected.store(true); }
   });
   foreign.join();
@@ -271,6 +277,97 @@ void parser_maximum_and_transport_end() {
     {"parser_storage_bytes", FrameParser::storage_bytes}, {"partial_eof", ends}, {"clean_eof_code", clean_end.code},
     {"io_error_code", io_error.code}, {"retained_bytes", broken.buffered_bytes()}}.dump() << '\n';
 }
+std::string input_wire(const std::string& seq, const std::string& tick = "0") {
+  return R"({"v":1,"type":"INPUT","session_id":"session-00","room_id":"unit-room","player_id":"player-00","seq":)" + seq +
+    R"(,"target_tick":)" + tick + R"(,"direction":"EAST","tag_target_player_id":null,"owner_epoch":0})";
+}
+Input parsed_input(const std::string& seq) {
+  FrameParser parser;
+  const auto bytes = framed_text(input_wire(seq));
+  const auto parsed = parser.consume(bytes);
+  check(parsed.state == ParseState::message && parsed.consumed == bytes.size() && parser.buffered_bytes() == 0,
+        "fixed input reaches the production decoder through a complete frame");
+  check(parsed.value.at("seq").is_number_unsigned(), "positive sequence remained an exact unsigned JSON integer");
+  return decode_input(parsed.value);
+}
+void fixed_pending_sequence_bound() {
+  Room room; populate(room);
+  const auto& player = room.players().at("player-00");
+  Json accepted = Json::array(), queued = Json::array();
+  for (std::uint64_t seq = 1; seq <= 64; ++seq) {
+    const auto result = room.input(player.id, parsed_input(std::to_string(seq)));
+    check(!result.error && !result.duplicate && player.last_accepted_seq() == seq, "fixed unique input accepted through64");
+    accepted.push_back(player.last_accepted_seq());
+  }
+  const auto pending = player.pending;
+  const auto last = player.last_accepted_input;
+  const auto state = room.view();
+  const auto overflow = room.input(player.id, parsed_input("65"));
+  check(overflow.error == "INPUT_QUEUE_FULL" && player.pending == pending && player.last_accepted_input == last &&
+    room.view() == state && player.last_accepted_seq() == 64 && room.input_high_water() == 64,
+    "65th EAST candidate rejects without discarding an accepted input or advancing sequence");
+  for (const auto& input : player.pending) queued.push_back(input.seq);
+  check(queued == accepted && player.pending.size() == 64, "all64 accepted inputs remain auditable until the tick");
+  room.tick();
+  check(player.applied_seq == 64 && player.x == 10400 && player.y == 10000 && player.score == 0 && room.pending_count() == 0,
+        "highest of64 same-tick inputs has one selected movement");
+  std::cout << Json{{"G05_pending_bound",Json{{"accepted_sequences",accepted},{"queued_before_tick",queued},
+    {"rejection",Json{{"seq",65},{"code",*overflow.error}}},{"high_water",room.input_high_water()},
+    {"applied_seq",*player.applied_seq},{"alpha_position",Json::array({player.x,player.y})},
+    {"last_accepted_seq",player.last_accepted_seq()},{"pending_after_tick",room.pending_count()}}}}.dump() << '\n';
+  room.close();
+}
+void unsigned_sequence_identity() {
+  Room room; populate(room);
+  const auto& player = room.players().at("player-00");
+  const std::array<std::string,4> values{"1","18446744073709551615","18446744073709551615","18446744073709551614"};
+  const std::array<std::string,4> expected{"ACCEPTED","ACCEPTED","DUPLICATE","INPUT_STALE"};
+  Json rows = Json::array();
+  for (std::size_t index = 0; index < values.size(); ++index) {
+    const auto input = parsed_input(values[index]);
+    const auto before = room.view();
+    const auto pending = player.pending;
+    const auto last = player.last_accepted_input;
+    const auto result = room.input(player.id,input);
+    const std::string code = result.error ? *result.error : result.duplicate ? "DUPLICATE" : "ACCEPTED";
+    check(code == expected[index] && room.view() == before, "unsigned sequence admission preserves exact identity and physical state");
+    if (index >= 2) check(player.pending == pending && player.last_accepted_input == last, "duplicate/stale unsigned value changed accepted work");
+    rows.push_back(Json{{"seq",input.seq},{"code",code},{"last_accepted_seq",player.last_accepted_seq()},
+      {"pending",player.pending.size()}});
+  }
+  const auto maximum = std::numeric_limits<std::uint64_t>::max();
+  check(player.last_accepted_seq() == maximum && player.pending.size() == 2, "uint64 max accepted without signed or floating conversion");
+  room.tick();
+  check(player.applied_seq == maximum && player.x == 10400 && player.y == 10000 && room.pending_count() == 0,
+        "unsigned max duplicate did not add a second effect");
+  std::cout << Json{{"G05_unsigned_sequence",rows},{"applied_seq",*player.applied_seq},
+    {"alpha_position",Json::array({player.x,player.y})},{"pending_after_tick",room.pending_count()}}.dump() << '\n';
+  room.close();
+}
+void invalid_input_integer_types() {
+  const std::vector<std::pair<std::string,std::string>> values{
+    {"0","0"},{"-1","0"},{"1.0","0"},{R"("1")","0"},{"18446744073709551616","0"},{"1","0.0"},{"1",R"("0")"}};
+  Json rows = Json::array();
+  for (const auto& [seq,tick] : values) {
+    Room room; populate(room);
+    const auto state = room.view();
+    const auto& player = room.players().at("player-00");
+    const auto wire = input_wire(seq,tick);
+    const auto bytes = framed_text(wire);
+    FrameParser parser;
+    const auto parsed = parser.consume(bytes);
+    check(parsed.state == ParseState::message_error && parsed.code == "MESSAGE_INVALID" && parsed.consumed == bytes.size(),
+          "frozen invalid integer form is a recoverable MESSAGE_INVALID");
+    check(room.view() == state && room.pending_count() == 0 && player.last_accepted_seq() == 0 &&
+          !player.last_accepted_input && !player.applied_seq && parser.buffered_bytes() == 0,
+          "invalid parser input cannot mutate fresh room/player state");
+    rows.push_back(Json{{"wire",wire},{"code",parsed.code},{"state_unchanged",room.view() == state},
+      {"last_accepted_seq",player.last_accepted_seq()},{"pending",room.pending_count()},
+      {"parser_buffered_bytes",parser.buffered_bytes()}});
+    room.close();
+  }
+  std::cout << Json{{"G05_invalid_integer_types",rows}}.dump() << '\n';
+}
 void real_tcp_authority_and_cleanup() {
   const auto scenario = Json::parse(R"({
     "scenario_id":"G01-three-tick-authority-smoke","contract_version":1,"thread":"G01","seed":7050,
@@ -404,7 +501,10 @@ int main(int argc, char** argv) {
       {"G03_room_permission_matrix", room_permission_matrix_preserves_state},
       {"G03_production_mailbox_capacity_drain", production_mailbox_capacity_and_drain},
       {"G04_fixed_monotonic_schedule", fixed_monotonic_schedule},
-      {"G04_production_monotonic_adapter", production_monotonic_adapter}};
+      {"G04_production_monotonic_adapter", production_monotonic_adapter},
+      {"G05_fixed_pending_sequence_bound", fixed_pending_sequence_bound},
+      {"G05_unsigned_sequence_identity", unsigned_sequence_identity},
+      {"G05_invalid_input_integer_types", invalid_input_integer_types}};
   } else if (std::string(argv[1]) == "integration") {
     tests = {{"real_TCP_authority_and_cleanup", real_tcp_authority_and_cleanup}, {"standalone_process_shutdown", [&] {
       standalone_process_shutdown(std::filesystem::absolute(argv[0]).parent_path() / "arena"); }},
