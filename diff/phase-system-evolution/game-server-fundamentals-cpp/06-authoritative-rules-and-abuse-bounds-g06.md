# Authoritative Rule과 Abuse Bound

## `G06: bound authenticated input validation attempts`

diff --git a/evidence/G06-runs.jsonl b/evidence/G06-runs.jsonl
new file mode 100644
index 0000000..97e8dfc
--- /dev/null
+++ b/evidence/G06-runs.jsonl
@@ -0,0 +1,6 @@
+{"category":"compile","units":1,"label":"baseline-compile","argv":["clang++","-std=c++20","-O2","-Wall","-Wextra","-Wpedantic","-Werror","-fsanitize=thread","-g","-I","src","-I","/opt/homebrew/include","artifacts/g06/reproduce.cpp","src/game.cpp","src/transport.cpp","src/scenario.cpp","-o","artifacts/g06/reproduce"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","started_at":"2026-08-28T03:21:11.006219+00:00","duration_seconds":16.73219,"exit":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g06/baseline-compile.log"}
+{"category":"unit","units":1,"label":"baseline","argv":["env","TSAN_OPTIONS=halt_on_error=1","./artifacts/g06/reproduce","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G06.json","artifacts/g06/baseline.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","started_at":"2026-08-28T03:22:19.554440+00:00","duration_seconds":0.885566,"exit":1,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g06/baseline.log"}
+{"category":"compile","units":2,"label":"build","argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g06-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g06/track","ARENA_TSAN=ON","./track","build"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","started_at":"2026-08-28T03:35:28.378453+00:00","duration_seconds":23.831165,"exit":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g06/build.log"}
+{"category":"unit","units":1,"label":"unit","argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g06-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g06/track","TSAN_OPTIONS=halt_on_error=1","./track","unit-test"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","started_at":"2026-08-28T03:36:55.784160+00:00","duration_seconds":0.921944,"exit":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g06/unit.log","ceiling_seconds":120}
+{"category":"integration","units":1,"label":"integration","argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g06-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g06/track","TSAN_OPTIONS=halt_on_error=1","./track","integration-test"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","started_at":"2026-08-28T03:36:56.707028+00:00","duration_seconds":1.409499,"exit":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g06/integration.log","ceiling_seconds":120}
+{"category":"canonical","units":1,"label":"canonical","argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g06-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g06/track","TSAN_OPTIONS=halt_on_error=1","./track","scenario-run","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G06.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g06/canonical.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","started_at":"2026-08-28T03:36:58.116893+00:00","duration_seconds":0.389579,"exit":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g06/canonical.log","ceiling_seconds":120}
diff --git a/evidence/G06.md b/evidence/G06.md
new file mode 100644
index 0000000..f702dd2
--- /dev/null
+++ b/evidence/G06.md
@@ -0,0 +1,37 @@
+# G06 — authoritative intent and four-attempt bound
+
+THREAD G06; PHASE phase-1; PROFILE `realtime-core`; BRANCH `track/fundamentals-cpp`; ATTEMPT initial.
+SPEC_REVISION `c1d62196ab76b55652f5d75a67514f8c6d8210ce`; START `7e4d8fa90d6eb5e0fc93e7d79fd426b936ac7890`.
+The user-authorized three-document spec/profile transition was read once; GAME_CONTRACT and G05 history/evidence remain unchanged. No G15+, external infrastructure, tags or push by this track.
+Frozen G06 input SHA-256 `8ca33010c24f31bdcfca54493b4868c2a89c682ba85f3795a4a8713f7ffb76df`.
+
+## Preserved reproduction
+
+All eight production files matched Git START before edits; manifest `artifacts/g06/pre-change-production.json` SHA-256 `e2fb44547639d62da524c6dec75f4a8a059ad8a1b5a117979ac77b501a8f9e44`. The exact command was resolved before execution:
+
+```sh
+python3 artifacts/g06/run.py unit 1 baseline env TSAN_OPTIONS=halt_on_error=1 ./artifacts/g06/reproduce /Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G06.json artifacts/g06/baseline.json
+```
+
+The helper linked unchanged G05 sources, reused actual issued identities/TCP/owner/manual-tick operations, and recorded18 admissions/221 states. Expected exit1 (0.885566s): fifth alpha seq8 at4 was ACCEPTED; its next-tick retry was SEQUENCE_CONFLICT. Pending high water5. Invalid direction already rejected; all six actual TAG outcomes, all221 movement/score/cooldown/time observations and final positions `[50000,50000]`, scores2/0, last sequences13/3, STOP/RUNNING matched. These exercised authority guarantees are NOT_REPRODUCED; no TAG/movement/range/cooldown algorithm was changed. All six descriptors closed and active cleanup counters were zero. Raw baseline SHA-256 `a1e0637758aa41e45f99a96152b683b673df54cc11d6decf7c5796e6ecf8ef9f`. Main was notified before production edits.
+
+## Narrow change and verification
+
+Authenticated owner admission now counts the first four attempts, including invalid intents, before calling the existing full input decoder. Fifth attempts return INPUT_RATE_EXCEEDED without reserving sequence or pending work; real simulation ticks reset the count. Parser framing/JSON/envelope/identity safety remains intact. Main explicitly approved moving the seven G05 invalid-integer probes through this production owner admission while preserving exact MESSAGE_INVALID, recovery, unchanged gameplay/sequence/pending and parser cleanup. Existing lower-level64/65 tests remain unchanged.
+
+Fixed pure additions cover four invalid attempts/fifth/retry, four clamps,2500/2501 range,LEFT actor/target,self-target, actual membership in an independent Room model, ASCII actor ordering for one shared victim, and full64-entry pending admission after rate activation. The three-player initialization function is defined only in tests; its narrow private friend does not change public join/start behavior or add server fixture injection. The G06 canonical reads the actual shared file and derives admissions, TAG outcomes, all221 resulting states and logical summary from production observations.
+
+`G06-runs.jsonl` records every command/category/UTC time/duration/exit/raw output. After a successful TSan build, `artifacts/g06/verify.py` runs complete unit, integration and one canonical command sequentially, with120s command ceilings and stop-on-failure. Caps: compile/configure8, unit4 including baseline, integration2, post canonical1; fault/load0.
+
+| Check | Actual result | Seconds | Exit |
+| --- | --- | ---: | ---: |
+| TSan configure/build | PASS | 23.831165 | 0 |
+| Complete unit suite | 21/21 PASS | 0.921944 | 0 |
+| Complete integration suite | 3/3 PASS | 1.409499 | 0 |
+| Frozen canonical | 221 ticks PASS | 0.389579 | 0 |
+
+Canonical observations: 16 accepted inputs; MESSAGE_INVALID at tick1 and INPUT_RATE_EXCEEDED at tick4; both rejected sequences remained reusable. TAG failed at ticks2,3,201,219 and succeeded at200,220. Final positions were alpha/bravo `[50000,50000]`, scores `{alpha:2,bravo:0}`, last sequences `{alpha:13,bravo:3}`, both STOP, room RUNNING. Raw result `artifacts/g06/canonical.json` SHA-256 `d4896582b60500f0ccca9e476eeb9bc21d71b56f4243fea91f1c5622e9f87a6b` includes all221 tick states and the common logical schema.
+
+Observed high waters / bounds: attempts4/4, pending inputs4/64, mailbox1/512, outbound control3/64, parser bytes330/16388. All six descriptors closed; all active cleanup counters (including input attempts) and file-descriptor delta were zero; `all_resources_released=true`. No TSan errors were reported. Actual budget: compile/configure3/8, unit2/4 including the expected failing reproduction, integration1/2, post canonical1/1; no retries or extra campaigns.
+
+STATE_HASHES inactive untilG07; NETWORK_FAULT_RUNS0; LOAD_RUNS0.
diff --git a/src/game.cpp b/src/game.cpp
index 565a46d..9b1d7ef 100644
--- a/src/game.cpp
+++ b/src/game.cpp
@@ -71,7 +71,7 @@ Player& Room::join(std::string player_id, std::string session_id, std::uint64_t
   const int slot = next_slot_++;
   Player player{std::move(player_id), std::move(session_id), connection_id, slot,
                 spawns[static_cast<std::size_t>(slot)][0], spawns[static_cast<std::size_t>(slot)][1],
-                Direction::stop, 0, -20, true, {}, {}, {}};
+                Direction::stop, 0, -20, true, {}, {}, {}, 0};
   const auto key = player.id;
   auto [found, inserted] = players_.emplace(key, std::move(player));
   if (!inserted) throw std::logic_error("server generated duplicate player id");
@@ -79,6 +79,17 @@ Player& Room::join(std::string player_id, std::string session_id, std::uint64_t
   if (ready >= 2) transition("RUNNING");
   return found->second;
 }
+std::optional<std::string> Room::begin_input_attempt(const std::string& player_id) {
+  assert_owner();
+  if (status_ != "RUNNING") return "ROOM_NOT_RUNNING";
+  auto found = players_.find(player_id);
+  if (found == players_.end() || !found->second.connected) return "PLAYER_MISMATCH";
+  auto& attempts = found->second.input_attempts;
+  if (attempts == max_input_attempts_per_tick) return "INPUT_RATE_EXCEEDED";
+  ++attempts;
+  input_attempt_high_water_ = std::max(input_attempt_high_water_, attempts);
+  return std::nullopt;
+}
 InputResult Room::input(const std::string& player_id, Input input) {
   assert_owner();
   if (status_ != "RUNNING") return {"ROOM_NOT_RUNNING", false};
@@ -111,6 +122,7 @@ std::vector<ActionFailure> Room::tick() {
   if (status_ != "RUNNING") return failures;
   std::map<std::string, std::string> tags;
   for (auto& [id, player] : players_) {
+    player.input_attempts = 0;
     player.applied_seq.reset();
     if (!player.connected) continue;
     std::optional<Input> selected;
@@ -169,6 +181,7 @@ void Room::leave(std::uint64_t connection_id) {
       player.direction = Direction::stop;
       player.pending.clear();
       player.applied_seq.reset();
+      player.input_attempts = 0;
     }
   }
 }
@@ -181,6 +194,7 @@ void Room::close() {
     player.direction = Direction::stop;
     player.pending.clear();
     player.applied_seq.reset();
+    player.input_attempts = 0;
   }
   transition("CLOSED");
 }
diff --git a/src/game.hpp b/src/game.hpp
index 819ddd7..e13569a 100644
--- a/src/game.hpp
+++ b/src/game.hpp
@@ -21,6 +21,7 @@ inline constexpr int session_ticks = 1'200;
 inline constexpr int tick_duration_ms = 50;
 inline constexpr int max_catch_up_ticks = 4;
 inline constexpr std::uint64_t max_future_input_ticks = 4;
+inline constexpr std::size_t max_input_attempts_per_tick = 4;
 
 enum class Direction { stop, north, east, south, west };
 std::string direction_name(Direction direction);
@@ -59,6 +60,7 @@ struct Player {
   std::deque<Input> pending;
   std::optional<Input> last_accepted_input;
   std::optional<std::uint64_t> applied_seq;
+  std::size_t input_attempts = 0;
   std::uint64_t last_accepted_seq() const { return last_accepted_input ? last_accepted_input->seq : 0; }
 };
 struct ActionFailure {
@@ -72,6 +74,7 @@ class Room {
   Room();
   void create(std::string id);
   Player& join(std::string player_id, std::string session_id, std::uint64_t connection_id);
+  std::optional<std::string> begin_input_attempt(const std::string& player_id);
   InputResult input(const std::string& player_id, Input input);
   std::vector<ActionFailure> tick();
   void leave(std::uint64_t connection_id);
@@ -84,7 +87,9 @@ class Room {
   int executed_ticks() const { return executed_ticks_; }
   std::size_t pending_count() const;
   std::size_t input_high_water() const { return input_high_water_; }
+  std::size_t input_attempt_high_water() const { return input_attempt_high_water_; }
  private:
+  friend void initialize_shared_victim_fixture(Room& room);
   void assert_owner() const;
   void transition(std::string status);
   std::thread::id owner_;
@@ -95,6 +100,7 @@ class Room {
   int next_slot_ = 0;
   int executed_ticks_ = 0;
   std::size_t input_high_water_ = 0;
+  std::size_t input_attempt_high_water_ = 0;
 };
 
 // This is executed simulation time, separate from the scheduler's clock source.
diff --git a/src/lifecycle_scenario.cpp b/src/lifecycle_scenario.cpp
index 653a131..30d7f59 100644
--- a/src/lifecycle_scenario.cpp
+++ b/src/lifecycle_scenario.cpp
@@ -450,6 +450,145 @@ Json run_input_scenario(const Json& scenario) {
   return evidence;
 }
 
+Json run_authority_scenario(const Json& scenario) {
+  const auto check_authority = [](bool condition, const std::string& text) {
+    if (!condition) throw std::runtime_error("G06: " + text);
+  };
+  check_authority(scenario.at("thread") == "G06" && scenario.at("contract_version") == 1 && scenario.at("seed") == 7050 &&
+    scenario.at("clock").at("kind") == "manual" && scenario.at("clock").at("tick_duration_ms") == tick_duration_ms &&
+    scenario.at("clients") == Json::array({"alpha","bravo"}) && scenario.at("events").size() == 18 &&
+    scenario.at("ticks") == 221 && scenario.at("socket_ceiling_ms") == 5000, "frozen authority scenario changed");
+  const auto descriptors_before = Fd::live();
+  LifecycleFixture fixture(scenario.at("socket_ceiling_ms").get<int>()); fixture.setup(2);
+  const auto role_of = [&](const std::string& id) {
+    for (const auto& [role, peer] : fixture.peers) if (peer.player == id) return role;
+    return id;
+  };
+  Json logical{{"admissions",Json::array()},{"tag_events",Json::array()}};
+  Json evidence{{"thread","G06"},{"scenario_id",scenario.at("scenario_id")},{"contract_version",1},
+    {"transport","production/real-loopback-TCP/kqueue/manual-owner-drain"},{"identities",fixture.identities()},
+    {"events",Json::array()},{"ticks",Json::array()},{"tag_observations",Json::array()},{"state_hashes","INACTIVE_UNTIL_G07"}};
+  std::size_t index = 0, accepted = 0;
+  for (int tick = 0; tick < scenario.at("ticks").get<int>(); ++tick) {
+    for (const auto& event : scenario.at("events")) {
+      if (event.at("before_tick") != tick) continue;
+      const auto role = event.at("client").get<std::string>();
+      auto& peer = fixture.peers.at(role);
+      const auto& player = fixture.server.room().players().at(peer.player);
+      auto request = fixture.request(role,"INPUT");
+      for (const auto* key : {"seq","target_tick","direction","owner_epoch"}) request[key] = event.at(key);
+      request["tag_target_player_id"] = nullptr;
+      if (!event.at("tag_target_role").is_null()) {
+        const auto target = event.at("tag_target_role").get<std::string>();
+        request["tag_target_player_id"] = fixture.peers.contains(target) ? fixture.peers.at(target).player : target;
+      }
+      if (event.contains("additional_ignored_fields")) request.update(event.at("additional_ignored_fields"));
+      const auto before = fixture.server.room().view();
+      const auto pending_before = player.pending;
+      const auto last_before = player.last_accepted_input;
+      const auto last_seq_before = player.last_accepted_seq();
+      const auto attempts_before = player.input_attempts;
+      peer.tcp->send(request);
+      Json response;
+      do { response = peer.tcp->receive(fixture.server); }
+      while (response.at("type") != "INPUT_ACK" && response.at("type") != "ERROR");
+      const auto code = response.at("code").get<std::string>();
+      const std::string expected = index == 2 ? "MESSAGE_INVALID" : index == 9 ? "INPUT_RATE_EXCEEDED" : "ACCEPTED";
+      check_authority(code == expected && fixture.server.room().view() == before,
+                      "admission code changed or input mutated physical state before the tick");
+      check_authority(player.input_attempts == std::min(attempts_before + 1,max_input_attempts_per_tick),
+                      "legitimate input attempt did not consume its bounded validation slot");
+      if (response.at("type") == "ERROR") {
+        check_authority(player.pending == pending_before && player.last_accepted_input == last_before,
+                        "rejected admission changed sequence identity or pending candidates");
+      } else {
+        check_authority(response.at("type") == "INPUT_ACK" && response.at("accepted") == true &&
+          response.at("seq") == event.at("seq") && response.at("player_id") == peer.player && response.at("tick") == tick &&
+          response.at("last_accepted_seq") == player.last_accepted_seq() && player.last_accepted_seq() == event.at("seq") &&
+          player.pending.size() == pending_before.size() + 1, "actual accepted input/ACK identity differs");
+        ++accepted;
+      }
+      logical["admissions"].push_back(Json{{"client",role},{"seq",event.at("seq")},{"before_tick",tick},{"code",code}});
+      evidence["events"].push_back(Json{{"request",request},{"response",response},{"before",before},{"after",fixture.server.room().view()},
+        {"last_accepted_before",last_seq_before},{"last_accepted_after",player.last_accepted_seq()},
+        {"pending_before",pending_before.size()},{"pending_after",player.pending.size()},
+        {"input_attempts_before",attempts_before},{"input_attempts_after",player.input_attempts}});
+      ++index;
+    }
+    std::map<std::string,std::deque<Input>> pending;
+    std::map<std::string,std::pair<int,int>> scores;
+    for (const auto& [role, peer] : fixture.peers) {
+      const auto& player = fixture.server.room().players().at(peer.player);
+      pending[role] = player.pending; scores[role] = {player.score,player.last_tag_tick};
+    }
+    fixture.server.advance_one_tick();
+    Json applied = Json::object();
+    for (const auto& [role, peer] : fixture.peers) {
+      const auto& player = fixture.server.room().players().at(peer.player);
+      check_authority(player.input_attempts == 0, "simulation boundary did not reset attempt budget");
+      applied[role] = player.applied_seq ? Json(*player.applied_seq) : Json(nullptr);
+      if (!player.applied_seq) continue;
+      const auto selected = std::find_if(pending.at(role).begin(),pending.at(role).end(),
+        [&](const auto& input) { return input.seq == *player.applied_seq; });
+      check_authority(selected != pending.at(role).end(), "actual applied sequence missing from pending candidates");
+      if (!selected->intent.tag_target) continue;
+      const bool success = player.score == scores.at(role).first + 1 && player.last_tag_tick == tick;
+      Json response = nullptr;
+      if (!success) {
+        response = peer.tcp->receive_type(fixture.server,"ERROR");
+        check_authority(response.at("code") == "ACTION_REJECTED" && response.at("player_id") == peer.player && response.at("tick") == tick &&
+          player.score == scores.at(role).first && player.last_tag_tick == scores.at(role).second,
+          "failed TAG lost its actual error or changed score/cooldown");
+      }
+      logical["tag_events"].push_back(Json{{"tick",tick},{"actor",role},{"target",role_of(*selected->intent.tag_target)},
+        {"result",success ? "TAG_SUCCESS" : "ACTION_REJECTED"}});
+      evidence["tag_observations"].push_back(Json{{"tick",tick},{"actor",peer.player},{"target",*selected->intent.tag_target},
+        {"score_before",scores.at(role).first},{"score_after",player.score},{"last_tag_before",scores.at(role).second},
+        {"last_tag_after",player.last_tag_tick},{"response",response}});
+    }
+    const auto& alpha = fixture.server.room().players().at(fixture.peers.at("alpha").player);
+    const auto& bravo = fixture.server.room().players().at(fixture.peers.at("bravo").player);
+    const int horizontal = std::min(tick + 1,100), vertical = std::clamp(tick - 99,0,100);
+    check_authority(alpha.x == 10000 + horizontal * 400 && alpha.y == 10000 + vertical * 400 &&
+      bravo.x == 90000 - horizontal * 400 && bravo.y == 90000 - vertical * 400 &&
+      alpha.score == (tick < 200 ? 0 : tick < 220 ? 1 : 2) && bravo.score == 0 &&
+      alpha.last_tag_tick == (tick < 200 ? -20 : tick < 220 ? 200 : 220) &&
+      fixture.clock.now_ms == (tick + 1) * tick_duration_ms && fixture.server.room().executed_ticks() == tick + 1,
+      "server movement, score, cooldown or fixed simulation clock differs at an actual tick");
+    evidence["ticks"].push_back(Json{{"tick",tick},{"state",fixture.server.room().view()},{"applied_sequences",applied},
+      {"pending_inputs",fixture.server.room().pending_count()},{"simulation_ms",fixture.clock.now_ms}});
+  }
+  check_authority(index == 18 && accepted == 16 && fixture.server.room().status() == "RUNNING" &&
+    fixture.server.room().pending_count() == 0, "fixed admission counts or terminal scenario state differs");
+  check_authority(logical.at("tag_events") == Json::array({
+    Json{{"tick",2},{"actor","alpha"},{"target","foreign-player"},{"result","ACTION_REJECTED"}},
+    Json{{"tick",3},{"actor","alpha"},{"target","bravo"},{"result","ACTION_REJECTED"}},
+    Json{{"tick",200},{"actor","alpha"},{"target","bravo"},{"result","TAG_SUCCESS"}},
+    Json{{"tick",201},{"actor","alpha"},{"target","bravo"},{"result","ACTION_REJECTED"}},
+    Json{{"tick",219},{"actor","alpha"},{"target","bravo"},{"result","ACTION_REJECTED"}},
+    Json{{"tick",220},{"actor","alpha"},{"target","bravo"},{"result","TAG_SUCCESS"}}}), "TAG action matrix differs");
+  logical["executed_ticks"] = fixture.server.room().executed_ticks(); logical["room_status"] = fixture.server.room().status();
+  for (const auto& [role, peer] : fixture.peers) {
+    const auto& player = fixture.server.room().players().at(peer.player);
+    check_authority(player.x == 50000 && player.y == 50000 && player.direction == Direction::stop &&
+      player.last_accepted_seq() == (role == "alpha" ? 13U : 3U), "final position/direction/sequence differs");
+    logical["final_positions"][role] = Json::array({player.x,player.y}); logical["scores"][role] = player.score;
+    logical["last_accepted_sequences"][role] = player.last_accepted_seq(); logical["final_directions"][role] = direction_name(player.direction);
+  }
+  evidence["metrics"] = fixture.server.metrics();
+  const auto& metrics = evidence.at("metrics");
+  check_authority(metrics.at("input_attempt_per_player_high_water") == 4 && metrics.at("input_per_player_high_water") == 4 &&
+    metrics.at("parser_buffer_high_water").get<std::size_t>() <= FrameParser::storage_bytes &&
+    metrics.at("mailbox_high_water").get<std::size_t>() <= max_mailbox_messages &&
+    metrics.at("outbound_control_high_water").get<std::size_t>() <= max_control_messages &&
+    metrics.at("connection_high_water") == 2, "rate or existing resource bound exceeded");
+  evidence["cleanup"] = fixture.finish();
+  logical["all_resources_released"] = Fd::live() == descriptors_before;
+  check_authority(logical.at("all_resources_released").get<bool>(), "authority scenario leaked resources");
+  evidence["logical"] = logical; evidence["executed_ticks"] = logical.at("executed_ticks"); evidence["result"] = "PASS";
+  return evidence;
+}
+
 Json run_clock_scenario(const Json& scenario) {
   const auto check_clock = [](bool condition, const std::string& text) {
     if (!condition) throw std::runtime_error("G04: " + text);
diff --git a/src/scenario.cpp b/src/scenario.cpp
index bda4ad1..b3b1f69 100644
--- a/src/scenario.cpp
+++ b/src/scenario.cpp
@@ -286,6 +286,7 @@ Json run_scenario(const Json& scenario) {
   if (scenario.at("thread") == "G03") return run_lifecycle_scenario(scenario);
   if (scenario.at("thread") == "G04") return run_clock_scenario(scenario);
   if (scenario.at("thread") == "G05") return run_input_scenario(scenario);
+  if (scenario.at("thread") == "G06") return run_authority_scenario(scenario);
   require(scenario.at("contract_version") == 1 && scenario.at("thread") == "G01", "only G01 contract v1 is active");
   require(scenario.at("clock").at("kind") == "manual" && scenario.at("clock").at("tick_duration_ms") == tick_duration_ms,
           "G01 runner requires the fixed 50ms manual clock");
diff --git a/src/scenario.hpp b/src/scenario.hpp
index 2f82733..7a7fcb4 100644
--- a/src/scenario.hpp
+++ b/src/scenario.hpp
@@ -9,4 +9,5 @@ Json run_lifecycle_scenario(const Json& scenario);
 Json run_mailbox_probe(std::size_t capacity);
 Json run_clock_scenario(const Json& scenario);
 Json run_input_scenario(const Json& scenario);
+Json run_authority_scenario(const Json& scenario);
 }
diff --git a/src/transport.cpp b/src/transport.cpp
index fd1a607..5b5d349 100644
--- a/src/transport.cpp
+++ b/src/transport.cpp
@@ -72,8 +72,8 @@ std::string request_error(const Json& value) {
   if (type == "JOIN_ROOM") return {};
   if (!string_field("player_id")) return "MESSAGE_INVALID";
   if (type == "LEAVE_ROOM") return {};
-  // Shared with owner admission and pure wire-to-domain integer probes.
-  (void)decode_input(value);
+  // Full INPUT validation is deferred until authenticated owner admission, so
+  // malformed attempts count and the fifth attempt never reaches the decoder.
   return {};
 }
 }
@@ -99,6 +99,16 @@ Input decode_input(const Json& value) {
   // fields are ignored. Only these typed logical fields define a retry.
   return input;
 }
+InputResult admit_input(Room& room, const std::string& player_id, const Json& value) {
+  if (const auto error = room.begin_input_attempt(player_id)) return {error, false};
+  try {
+    return room.input(player_id, decode_input(value));
+  } catch (const Json::exception&) {
+    return {"MESSAGE_INVALID", false};
+  } catch (const std::invalid_argument&) {
+    return {"MESSAGE_INVALID", false};
+  }
+}
 std::atomic<int> Fd::live_{0};
 Fd::Fd(int value) : value_(value) { if (value_ >= 0) ++live_; }
 Fd::~Fd() { reset(); }
@@ -425,13 +435,12 @@ void Server::handle(const Envelope& envelope) {
       room_.leave(id);
       Json state = room_.view(); state.update(message("SNAPSHOT")); queue(id, state); return;
     }
-    const auto input = decode_input(value);
-    const auto result = room_.input(conn->player_id, input);
+    const auto result = admit_input(room_, conn->player_id, value);
     if (result.error) {
       reject(*result.error, "input was not accepted"); return;
     }
     Json reply = message("INPUT_ACK"); reply["player_id"] = conn->player_id; reply["accepted"] = true;
-    reply["seq"] = input.seq; reply["code"] = result.duplicate ? "DUPLICATE" : "ACCEPTED";
+    reply["seq"] = value.at("seq").get<std::uint64_t>(); reply["code"] = result.duplicate ? "DUPLICATE" : "ACCEPTED";
     reply["last_accepted_seq"] = room_.players().at(conn->player_id).last_accepted_seq();
     reply["tick"] = room_.executed_ticks(); queue(id, std::move(reply));
   } catch (const Json::exception&) {
@@ -495,6 +504,7 @@ Json Server::metrics() const {
   return Json{{"received_messages", received_messages_}, {"sent_messages", sent_messages_},
     {"mailbox_high_water", mailbox_high_water_}, {"outbound_control_high_water", outbound_high_water_},
     {"connection_high_water", connection_high_water_}, {"input_per_player_high_water", room_.input_high_water()},
+    {"input_attempt_per_player_high_water", room_.input_attempt_high_water()},
     {"max_read_bytes", max_read_bytes_}, {"parser_buffer_high_water", parser_high_water_},
     {"parser_storage_bytes_per_connection", FrameParser::storage_bytes}, {"need_more_events", need_more_events_},
     {"message_error_events", message_error_events_}, {"terminal_frame_events", terminal_frame_events_},
@@ -506,12 +516,14 @@ Json Server::metrics() const {
       {"operational_state", last_batch_.overloaded ? "OVERLOADED" : "NORMAL"}}}};
 }
 Json Server::cleanup() const {
-  std::size_t queued = 0, parser_buffered = 0;
+  std::size_t queued = 0, parser_buffered = 0, input_attempts = 0;
   for (const auto& [fd, conn] : connections_) {
     (void)fd; queued += conn.outbound.size(); parser_buffered += conn.parser.buffered_bytes();
   }
+  for (const auto& [id, player] : room_.players()) { (void)id; input_attempts += player.input_attempts; }
   return Json{{"server_connections", connections_.size()}, {"server_descriptors", owned_descriptors().size()},
     {"mailbox_messages", mailbox_.size()}, {"pending_inputs", room_.pending_count()}, {"outbound_messages", queued},
+    {"input_attempts", input_attempts},
     {"parser_buffered_bytes", parser_buffered}, {"parser_storage_bytes", connections_.size() * FrameParser::storage_bytes},
     {"worker_threads", 0}, {"timers", 0}, {"disconnect_notifications", disconnected_.size()},
     {"scheduler_pending_ms", accumulator_.remaining_ms()}};
diff --git a/src/transport.hpp b/src/transport.hpp
index 565b641..dc56b58 100644
--- a/src/transport.hpp
+++ b/src/transport.hpp
@@ -27,6 +27,8 @@ class Fd {
 std::vector<std::uint8_t> encode_frame(const Json& value);
 Json decode_complete_frame(std::span<const std::uint8_t> bytes);
 Input decode_input(const Json& value);
+// The caller has attributed the request to its authenticated Room/player.
+InputResult admit_input(Room& room, const std::string& player_id, const Json& value);
 
 enum class ParseState { need_more, message, message_error, terminal_frame_error, io_end };
 std::string parse_state_name(ParseState state);
diff --git a/tests/tests.cpp b/tests/tests.cpp
index cf5caa9..7141dfb 100644
--- a/tests/tests.cpp
+++ b/tests/tests.cpp
@@ -1,4 +1,5 @@
 #include "scenario.hpp"
+#include <algorithm>
 #include <atomic>
 #include <cerrno>
 #include <chrono>
@@ -7,6 +8,7 @@
 #include <functional>
 #include <iostream>
 #include <limits>
+#include <memory>
 #include <netinet/in.h>
 #include <poll.h>
 #include <spawn.h>
@@ -18,6 +20,24 @@
 #include <unistd.h>
 
 extern char** environ;
+namespace arena {
+// Only this fixed pure-unit initialization is a Room friend. No server path
+// can select fixture IDs or bypass its unchanged two-player start lifecycle.
+void initialize_shared_victim_fixture(Room& room) {
+  room.assert_owner();
+  room.create("shared-victim-room");
+  const std::array<std::string,3> ids{"actor-z","target","actor-a"};
+  for (std::size_t index = 0; index < ids.size(); ++index) {
+    Player player;
+    player.id = ids[index]; player.session_id = "session-" + ids[index]; player.connection_id = index + 1;
+    player.slot = static_cast<int>(index); player.x = 50000; player.y = 50000;
+    const auto id = player.id;
+    room.players_.emplace(id,std::move(player));
+  }
+  room.next_slot_ = 3;
+  room.transition("RUNNING");
+}
+}
 namespace {
 using namespace arena;
 void check(bool condition, const std::string& text) { if (!condition) throw std::runtime_error(text); }
@@ -356,18 +376,184 @@ void invalid_input_integer_types() {
     const auto bytes = framed_text(wire);
     FrameParser parser;
     const auto parsed = parser.consume(bytes);
-    check(parsed.state == ParseState::message_error && parsed.code == "MESSAGE_INVALID" && parsed.consumed == bytes.size(),
-          "frozen invalid integer form is a recoverable MESSAGE_INVALID");
+    check(parsed.state == ParseState::message && parsed.consumed == bytes.size(), "complete framed input reaches owner validation");
+    const auto admission = admit_input(room,player.id,parsed.value);
+    check(admission.error == "MESSAGE_INVALID", "frozen invalid integer form is a recoverable MESSAGE_INVALID");
     check(room.view() == state && room.pending_count() == 0 && player.last_accepted_seq() == 0 &&
           !player.last_accepted_input && !player.applied_seq && parser.buffered_bytes() == 0,
-          "invalid parser input cannot mutate fresh room/player state");
-    rows.push_back(Json{{"wire",wire},{"code",parsed.code},{"state_unchanged",room.view() == state},
+          "invalid input cannot mutate fresh room/player state");
+    const auto next = parser.consume(encode_frame(message("HELLO")));
+    check(next.state == ParseState::message && next.value == message("HELLO") && parser.buffered_bytes() == 0 &&
+          player.input_attempts == 1, "owner schema rejection preserves parser recovery and counts its attempt");
+    rows.push_back(Json{{"wire",wire},{"code",*admission.error},{"state_unchanged",room.view() == state},
       {"last_accepted_seq",player.last_accepted_seq()},{"pending",room.pending_count()},
-      {"parser_buffered_bytes",parser.buffered_bytes()}});
+      {"parser_buffered_bytes",parser.buffered_bytes()},{"input_attempts",player.input_attempts},{"next_state",parse_state_name(next.state)}});
     room.close();
   }
   std::cout << Json{{"G05_invalid_integer_types",rows}}.dump() << '\n';
 }
+Json unit_input(const Room& room, const std::string& actor, std::uint64_t seq, const std::string& direction,
+                const std::optional<std::string>& target = std::nullopt) {
+  const auto& player = room.players().at(actor);
+  return Json{{"v",1},{"type","INPUT"},{"session_id",player.session_id},{"room_id",room.id()},
+    {"player_id",actor},{"seq",seq},{"target_tick",room.executed_ticks()},{"direction",direction},
+    {"tag_target_player_id",target ? Json(*target) : Json(nullptr)},{"owner_epoch",0}};
+}
+void first_four_attempts_include_invalid() {
+  Room room; populate(room);
+  const auto& player = room.players().at("player-00");
+  const auto before = room.view();
+  FrameParser parser;
+  Json rows = Json::array();
+  for (std::uint64_t seq = 1; seq <= 5; ++seq) {
+    const auto request = unit_input(room,player.id,seq,seq < 5 ? "NORTH_EAST" : "EAST");
+    const auto framed = parser.consume(encode_frame(request));
+    check(framed.state == ParseState::message, "rate probe framing reaches legitimate actor admission");
+    const auto result = admit_input(room,player.id,framed.value);
+    check(result.error == (seq < 5 ? "MESSAGE_INVALID" : "INPUT_RATE_EXCEEDED"), "first four attempts include malformed intent");
+    check(room.view() == before && room.pending_count() == 0 && player.last_accepted_seq() == 0 && !player.last_accepted_input &&
+      player.input_attempts == std::min<std::uint64_t>(seq,4) && parser.buffered_bytes() == 0,
+      "rejected attempt changed authoritative state/sequence/pending or exceeded validation bound");
+    rows.push_back(Json{{"seq",seq},{"code",*result.error},{"input_attempts",player.input_attempts},
+      {"last_accepted_seq",player.last_accepted_seq()},{"pending",room.pending_count()},{"state_unchanged",room.view() == before}});
+  }
+  room.tick();
+  check(player.input_attempts == 0 && room.input_attempt_high_water() == 4 && player.x == 10000 && player.y == 10000,
+        "one real tick resets attempts without applying rejected intent");
+  const auto retry = admit_input(room,player.id,unit_input(room,player.id,5,"EAST"));
+  check(!retry.error && !retry.duplicate && player.last_accepted_seq() == 5 && player.input_attempts == 1 && player.pending.size() == 1,
+        "rate rejection did not reserve seq5 across the next tick");
+  room.tick();
+  check(player.x == 10400 && player.y == 10000 && player.applied_seq == 5 && room.pending_count() == 0 && player.input_attempts == 0,
+        "next tick retry applies one server movement");
+  std::cout << Json{{"G06_first_four_attempts",rows},{"retry_code","ACCEPTED"},{"retry_seq",player.last_accepted_seq()},
+    {"alpha_position",Json::array({player.x,player.y})},{"attempt_high_water",room.input_attempt_high_water()}}.dump() << '\n';
+  room.close();
+}
+void four_fixed_movement_clamps() {
+  struct Case { int x; int y; const char* direction; int expected_x; int expected_y; };
+  const std::array<Case,4> cases{{{99900,50000,"EAST",100000,50000},{100,50000,"WEST",0,50000},
+    {50000,99900,"NORTH",50000,100000},{50000,100,"SOUTH",50000,0}}};
+  Json rows = Json::array();
+  for (const auto& item : cases) {
+    Room room; room.create("clamp-room");
+    auto& actor = room.join("actor","actor-session",1);
+    room.join("target","target-session",2);
+    actor.x = item.x; actor.y = item.y;
+    const auto before = room.view();
+    const auto result = admit_input(room,actor.id,unit_input(room,actor.id,1,item.direction));
+    check(!result.error && room.view() == before, "clamp intent admission does not move the player");
+    check(room.tick().empty() && actor.x == item.expected_x && actor.y == item.expected_y && actor.score == 0,
+          "fixed400 movement clamps at the exact arena edge");
+    rows.push_back(Json{{"before",Json::array({item.x,item.y})},{"direction",item.direction},
+      {"after",Json::array({actor.x,actor.y})},{"score",actor.score}});
+    room.close();
+  }
+  std::cout << Json{{"G06_four_clamps",rows}}.dump() << '\n';
+}
+void tag_range_and_membership_edges() {
+  Json rows = Json::array();
+  for (const auto distance : {2500,2501}) {
+    Room room; room.create("range-room");
+    auto& actor = room.join("actor","actor-session",1);
+    auto& target = room.join("target","target-session",2);
+    actor.x = 50000; actor.y = 50000; target.x = 50000 + distance; target.y = 50000;
+    const auto before = room.view();
+    const auto result = admit_input(room,actor.id,unit_input(room,actor.id,1,"STOP",target.id));
+    check(!result.error && room.view() == before, "range checked at target tick after accepted input");
+    const auto failures = room.tick();
+    const bool success = actor.score == 1 && actor.last_tag_tick == 0;
+    check(success == (distance == 2500) && failures.size() == (distance == 2500 ? 0U : 1U) && target.score == 0,
+          "inclusive2500 vs2501 TAG range boundary");
+    if (!success) check(actor.score == 0 && actor.last_tag_tick == -20 && failures.front().player_id == actor.id,
+                        "range failure leaves score/cooldown unchanged");
+    rows.push_back(Json{{"case","range"},{"distance",distance},{"result",success ? "TAG_SUCCESS" : "ACTION_REJECTED"},
+      {"score",actor.score},{"last_tag_tick",actor.last_tag_tick},{"last_accepted_seq",actor.last_accepted_seq()}});
+    room.close();
+  }
+  for (const auto* name : {"actor LEFT","target LEFT","self-target","target in independent other-Room model"}) {
+    Room room; room.create("edge-room");
+    auto& actor = room.join("actor","actor-session",1);
+    auto& target = room.join("target","target-session",2);
+    actor.x = target.x = 50000; actor.y = target.y = 50000;
+    std::string target_id = target.id;
+    std::unique_ptr<Room> other;
+    Json other_before = nullptr;
+    if (std::string(name) == "actor LEFT") room.leave(actor.connection_id);
+    if (std::string(name) == "target LEFT") room.leave(target.connection_id);
+    if (std::string(name) == "self-target") target_id = actor.id;
+    if (std::string(name) == "target in independent other-Room model") {
+      other = std::make_unique<Room>(); other->create("independent-room");
+      auto& foreign = other->join("foreign","foreign-session",3);
+      other->join("foreign-peer","foreign-peer-session",4);
+      foreign.x = 50000; foreign.y = 50000; target_id = foreign.id;
+      check(other->players().contains(target_id) && !room.players().contains(target_id), "actual foreign Room membership");
+      other_before = other->view();
+    }
+    const auto before = room.view();
+    const auto result = admit_input(room,actor.id,unit_input(room,actor.id,1,"STOP",target_id));
+    check(room.view() == before, "target-edge admission cannot mutate physical state");
+    std::string code;
+    if (std::string(name) == "actor LEFT") {
+      check(result.error == "PLAYER_MISMATCH" && room.pending_count() == 0 && actor.last_accepted_seq() == 0 && actor.input_attempts == 0,
+            "LEFT actor admission preserves identity, pending and validation budget");
+      code = *result.error;
+    } else {
+      check(!result.error && actor.last_accepted_seq() == 1 && actor.pending.size() == 1, "syntactically valid target is admitted");
+      const auto failures = room.tick();
+      check(failures.size() == 1 && failures.front().player_id == actor.id && failures.front().connection_id == actor.connection_id &&
+        actor.score == 0 && actor.last_tag_tick == -20 && actor.last_accepted_seq() == 1 && actor.pending.empty(),
+        "failed TAG emits actor failure, preserving score/cooldown without rolling back accepted sequence");
+      code = "ACTION_REJECTED";
+    }
+    check(target.score == 0 && actor.x == 50000 && actor.y == 50000, "rejected TAG changed unrelated target or STOP position");
+    if (other) check(other->view() == other_before && other->pending_count() == 0, "foreign Room was mutated");
+    rows.push_back(Json{{"case",name},{"result",code},{"before",before},{"after",room.view()},
+      {"last_accepted_seq",actor.last_accepted_seq()},{"pending",actor.pending.size()},
+      {"foreign_room_unchanged",other ? Json(other->view() == other_before) : Json(nullptr)}});
+    room.close(); if (other) other->close();
+  }
+  std::cout << Json{{"G06_TAG_range_membership",rows}}.dump() << '\n';
+}
+void shared_victim_ascii_order() {
+  Room room; initialize_shared_victim_fixture(room);
+  const auto& first = room.players().at("actor-a");
+  const auto& second = room.players().at("actor-z");
+  const auto& victim = room.players().at("target");
+  Json order = Json::array();
+  for (const auto& [id,player] : room.players()) { (void)player; order.push_back(id); }
+  for (const auto* actor : {"actor-z","actor-a"})
+    check(!admit_input(room,actor,unit_input(room,actor,1,"STOP",victim.id)).error, "both fixed actors admitted in reverse ASCII order");
+  const auto first_failures = room.tick();
+  check(first.score == 1 && first.last_tag_tick == 0 && second.score == 0 && second.last_tag_tick == -20 && victim.score == 0 &&
+    first_failures.size() == 1 && first_failures.front().player_id == second.id, "one victim per tick chooses ASCII-first actor");
+  const auto tick0 = room.view();
+  check(!admit_input(room,second.id,unit_input(room,second.id,2,"STOP",victim.id)).error, "losing actor retries on next tick");
+  check(room.tick().empty() && first.score == 1 && second.score == 1 && second.last_tag_tick == 1 && victim.score == 0,
+        "prior rejection did not consume cooldown; victim is eligible on the next tick");
+  std::cout << Json{{"G06_shared_victim",Json{{"insertion_order",Json::array({"actor-z","target","actor-a"})},
+    {"iteration_order",order},{"tick0",tick0},{"tick1",room.view()},{"tick0_winner",first.id},{"tick1_winner",second.id}}}}.dump() << '\n';
+  room.close();
+}
+void pending_bound_after_rate_activation() {
+  Room room; populate(room);
+  const auto& player = room.players().at("player-00");
+  for (std::uint64_t seq = 1; seq <= 64; ++seq)
+    check(!room.input(player.id,Input{seq,std::uint64_t{0},{Direction::east,std::nullopt}}).error, "existing lower-level queue prepopulation");
+  const auto pending = player.pending;
+  const auto last = player.last_accepted_input;
+  const auto before = room.view();
+  const auto result = admit_input(room,player.id,unit_input(room,player.id,65,"EAST"));
+  check(result.error == "INPUT_QUEUE_FULL" && player.pending == pending && player.last_accepted_input == last &&
+    room.view() == before && player.input_attempts == 1 && player.pending.size() == 64,
+    "rate activation preserves the64-entry lower-level bound and all accepted inputs");
+  room.tick();
+  check(player.applied_seq == 64 && player.x == 10400 && player.y == 10000 && player.pending.empty(), "retained highest candidate still applies once");
+  std::cout << Json{{"G06_pending_after_rate",Json{{"prepopulated",pending.size()},{"code",*result.error},
+    {"last_accepted_seq",player.last_accepted_seq()},{"applied_seq",*player.applied_seq},{"pending_after_tick",player.pending.size()},
+    {"alpha_position",Json::array({player.x,player.y})}}}}.dump() << '\n';
+  room.close();
+}
 void real_tcp_authority_and_cleanup() {
   const auto scenario = Json::parse(R"({
     "scenario_id":"G01-three-tick-authority-smoke","contract_version":1,"thread":"G01","seed":7050,
@@ -504,7 +690,12 @@ int main(int argc, char** argv) {
       {"G04_production_monotonic_adapter", production_monotonic_adapter},
       {"G05_fixed_pending_sequence_bound", fixed_pending_sequence_bound},
       {"G05_unsigned_sequence_identity", unsigned_sequence_identity},
-      {"G05_invalid_input_integer_types", invalid_input_integer_types}};
+      {"G05_invalid_input_integer_types", invalid_input_integer_types},
+      {"G06_first_four_attempts_include_invalid", first_four_attempts_include_invalid},
+      {"G06_four_movement_clamps", four_fixed_movement_clamps},
+      {"G06_TAG_range_membership", tag_range_and_membership_edges},
+      {"G06_shared_victim_ASCII_order", shared_victim_ascii_order},
+      {"G06_pending_bound_after_rate", pending_bound_after_rate_activation}};
   } else if (std::string(argv[1]) == "integration") {
     tests = {{"real_TCP_authority_and_cleanup", real_tcp_authority_and_cleanup}, {"standalone_process_shutdown", [&] {
       standalone_process_shutdown(std::filesystem::absolute(argv[0]).parent_path() / "arena"); }},
