# Reconnect, Resume Token과 Full Resync (G11)

## `feat(session): add bounded reconnect grace and rotating credentials`

diff --git a/CMakeLists.txt b/CMakeLists.txt
index 8323a7c..5e59071 100644
--- a/CMakeLists.txt
+++ b/CMakeLists.txt
@@ -34,7 +34,7 @@ target_compile_options(arena PRIVATE -Wall -Wextra -Wpedantic -Werror)
 add_executable(arena_tests tests/tests.cpp tests/g09.cpp)
 target_link_libraries(arena_tests PRIVATE arena_test_core)
 target_compile_options(arena_tests PRIVATE -Wall -Wextra -Wpedantic -Werror)
-add_executable(arena_scenarios tests/scenario_main.cpp tests/g07.cpp tests/g09.cpp tests/g10.cpp)
+add_executable(arena_scenarios tests/scenario_main.cpp tests/g07.cpp tests/g09.cpp tests/g10.cpp tests/g11.cpp)
 target_link_libraries(arena_scenarios PRIVATE arena_test_core)
 target_compile_options(arena_scenarios PRIVATE -Wall -Wextra -Wpedantic -Werror)
 enable_testing()
diff --git a/evidence/G11-runs.jsonl b/evidence/G11-runs.jsonl
new file mode 100644
index 0000000..8c569bc
--- /dev/null
+++ b/evidence/G11-runs.jsonl
@@ -0,0 +1,7 @@
+{"label":"baseline-compile","category":"compile","units":1,"ticks":0,"ceiling_seconds":180,"argv":["clang++","-std=c++20","-O2","-Wall","-Wextra","-Wpedantic","-Werror","-fsanitize=thread","-g","-DARENA_TEST_FIXTURES=1","-I","src","-I","tests","-I","/opt/homebrew/include","artifacts/g11/reproduce.cpp","src/game.cpp","src/transport.cpp","src/replay.cpp","src/snapshot.cpp","-o","artifacts/g11/reproduce"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g11/baseline-compile.log","started_at":"2026-08-28T06:27:55.865560+00:00","duration_seconds":21.261705,"exit":0,"wrapper_pid":49633,"child_pid":49662,"timed_out":false}
+{"label":"baseline","category":"unit","units":1,"ticks":202,"ceiling_seconds":120,"argv":["env","TSAN_OPTIONS=halt_on_error=1","./artifacts/g11/reproduce","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G11.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g11/baseline.json"],"expected_exit":1,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g11/baseline.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g11/baseline.json","started_at":"2026-08-28T06:29:48.372325+00:00","duration_seconds":1.476106,"exit":1,"wrapper_pid":51745,"child_pid":51754,"timed_out":false,"observed_ticks":202,"runtime_pid":51754}
+{"label":"build","category":"compile","units":2,"ticks":0,"ceiling_seconds":180,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g11-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g11/track","TSAN_OPTIONS=halt_on_error=1","ARENA_TSAN=ON","./track","build"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g11/build.log","started_at":"2026-08-28T06:43:54.018873+00:00","duration_seconds":42.788145,"exit":2,"wrapper_pid":71426,"child_pid":71435,"timed_out":false}
+{"label":"build-retry1","category":"compile","units":1,"ticks":0,"ceiling_seconds":180,"argv":["/opt/homebrew/bin/cmake","--build","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g11-tsan","--parallel","2"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g11/build-retry1.log","reason":"Fix two explicit string/Json comparisons in the test driver; preserve original configure/build failure.","started_at":"2026-08-28T06:46:00.036053+00:00","duration_seconds":5.357809,"exit":0,"wrapper_pid":73939,"child_pid":73948,"timed_out":false}
+{"label":"unit","category":"unit","units":1,"ticks":0,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g11-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g11/track","TSAN_OPTIONS=halt_on_error=1","./track","unit-test"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g11/unit.log","started_at":"2026-08-28T06:47:43.935553+00:00","duration_seconds":3.169699,"exit":0,"wrapper_pid":75824,"child_pid":75825,"timed_out":false}
+{"label":"integration","category":"integration","units":1,"ticks":0,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g11-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g11/track","TSAN_OPTIONS=halt_on_error=1","./track","integration-test"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g11/integration.log","started_at":"2026-08-28T06:47:47.174701+00:00","duration_seconds":1.438394,"exit":0,"wrapper_pid":75887,"child_pid":75888,"timed_out":false}
+{"label":"canonical","category":"canonical","units":1,"ticks":1008,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g11-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g11/track","TSAN_OPTIONS=halt_on_error=1","./track","scenario-run","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G11.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g11/canonical.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g11/canonical.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g11/canonical.json","started_at":"2026-08-28T06:47:48.659430+00:00","duration_seconds":3.366778,"exit":0,"wrapper_pid":75929,"child_pid":75930,"timed_out":false,"observed_ticks":1008,"runtime_pid":75936}
diff --git a/evidence/G11.md b/evidence/G11.md
new file mode 100644
index 0000000..143ea92
--- /dev/null
+++ b/evidence/G11.md
@@ -0,0 +1,28 @@
+# G11 — bounded reconnect grace and rotating credentials
+
+THREAD `G11`; BRANCH `track/fundamentals-cpp`; PROFILE `realtime-core`; initial attempt, repairs `0/2`. START `08192f170145d7f4e85facc3f5a14a5cbf48596e`; Spec-Revision `c1d62196ab76b55652f5d75a67514f8c6d8210ce`. Frozen fixture SHA-256 `f4b892f5155655d41b52f197c1174c7c41f7fe75cd3f75e6f0f5b2f8e8c261c7`.
+
+Exact commands, processes, ceilings, durations and exits: `evidence/G11-runs.jsonl`. Resolved command plans: `artifacts/g11/commands.initial.json` and append-only command expansion in `commands.json`. Actual unchanged START hashes for 17 source/build inputs: `artifacts/g11/pre-change-production.json`; baseline helper: `artifacts/g11/reproduce.cpp`.
+
+| Raw result | Exit | Bytes | SHA-256 |
+| --- | ---: | ---: | --- |
+| `artifacts/g11/baseline.json` | 1 | 371252 | `415e95bc8aa7a569c04b46d93ccbc952de8f86d6de1c201624b47d3e3caada63` |
+| `artifacts/g11/canonical.json` | 0 | 2035758 | `9708d1a5b0f14c3056e0d9bcea3da03937f769c1babdd31ff42a626b615c1a06` |
+
+Baseline used real unchanged G10 joins/binds, seven INPUTs, 202 ticks and 102 actual alpha snapshots. Neither join issued a resume credential; resume was explicitly unexercised. TCP close retained the original UDP socket and left Room state unchanged until owner drain, which committed LEFT/STOP. Position `(50400,50000)`, score 1 and last sequence 20 already survived; those guarantees were retained. All nine descriptors and active resources were released.
+
+Production now keeps one resume record per bounded Room player, uses a 200-tick owner deadline, rotates 128-bit resume/bind credentials, retires the provisional session on success and preserves player/session/input/snapshot identity. DISCONNECTED is visible and cannot move or TAG; only LEFT is omitted. Fresh bind publishes FULL from current canonical state without rewriting the historical tick hash. Existing movement/range/cooldown/sequence rules remain unchanged. G03 close expectations alone adopt DISCONNECTED; the offline adapter recognizes the new lifecycle events without an extra offline campaign.
+
+Post: TSan unit **24/24**, integration **4/4**, and exactly one three-case **1008-tick / 24-INPUT** run passed. Every case's first 202 canonical records/hashes equal baseline. Consumed R0 failed while DISCONNECTED, current R1 succeeded, consumed B2 failed, duplicate20 caused no movement, and the identical21 was rejected on the still-open original UDP endpoint then accepted on the current endpoint. Deadline402 was valid at next401; LEFT was already visible after tick401 before EXPIRED at next402.
+
+| Case | Ticks | Alpha final / sequence | Final state hash | Alpha/bravo snapshots and ACKs |
+| --- | ---: | --- | --- | --- |
+| immediate and reuse | 204 | `(50800,50000)`, EAST, 21 | `0cf3f07eff0b19d75584223b2e12ed95f1273e7ef03eb83ba8900a55cfda92d1` | 105 / 103 |
+| last valid | 402 | `(50400,50000)`, STOP, 20 | `5223cfe3b07eddb95694e1ad217414917875a65e31d033ddbd99c1c1fe6bc907` | 104 / 202 |
+| expired | 402 | `(50400,50000)`, LEFT, 20 | `a3d3d408cdd112d3e8f4dbf3faea8c1c835e01aa0fc5004056ee06b390ec5547` | 102 / 202 |
+
+Scores remain `{alpha:1,bravo:0}`. Immediate FULLs103/104 at still-tick201 hash current STOP/CONNECTED as `a92db48eb9822fd2a8d0c0865ee41e8386b3f201d62c8aa328d85c62f1f96291`; historical EAST201 remains `eb31b6f2fff56c9813e3a18e6ad5029419bea76694e8cf5bff134c69f57bcd3b`. Raw results retain every per-tick scalar/canonical/hash record, actual lifecycle boundaries, credential relationship booleans, compact ordinary publication/ACK rows and complete immediate FULL captures.
+
+Bounds: existing eleven-case UDP matrix unchanged; reused pure eight-player serializer measured all-connected **1186 bytes** and seven-DISCONNECTED/STOP plus one-CONNECTED **1200 bytes**. Generated public Room IDs shorten17→14 only; player IDs8, accepted1..64 identifiers and credential entropy remain unchanged. Live high-waters: mailbox/control/pending/attempt1, connections2, retention32, resume records2/8, UDP495 bytes, replay43610 bytes. Actual descriptor checks **17/12/11**; all active session/credential/grace/queue/buffer/socket/thread counters and descriptor deltas end at zero. No credential fields or raw32-hex credential values appear in baseline/post evidence.
+
+Budget: compile/configure **4/8**, unit **2/4** including baseline, integration **1/2**, post canonical **1/1**. Initial post build exit2 (two driver string/JSON comparison errors) is retained in `build.log`; explicit string reads and incremental `build-retry1` exit0 followed. Unit/integration/canonical logs share `artifacts/g11/`. No runtime reruns, timeouts, offline, packet-fault or load runs; suites do not invoke the G11 campaign. Static diff check passed; shipping binary contains no fixture/runner symbols, and only test targets define `ARENA_TEST_FIXTURES`. UNRESOLVED: none; no G12, tags or push.
diff --git a/src/game.cpp b/src/game.cpp
index 4f853af..5a8f969 100644
--- a/src/game.cpp
+++ b/src/game.cpp
@@ -79,9 +79,11 @@ Player& Room::join(std::string player_id, std::string session_id, std::uint64_t
   return found->second;
 }
 void Room::evaluate_start_condition() {
-  const auto joined = std::count_if(players_.begin(),players_.end(),[](const auto& pair) { return pair.second.connected; });
+  const auto joined = std::count_if(players_.begin(),players_.end(),[](const auto& pair) {
+    return pair.second.connected || pair.second.disconnect_deadline;
+  });
   const bool ready = std::all_of(players_.begin(),players_.end(),[](const auto& pair) {
-    return !pair.second.connected || pair.second.realtime_ready;
+    return (!pair.second.connected && !pair.second.disconnect_deadline) || (pair.second.connected && pair.second.realtime_ready);
   });
   if (status_ == "LOBBY" && joined >= 2 && ready) transition("RUNNING");
 }
@@ -182,6 +184,13 @@ std::vector<ActionFailure> Room::tick() {
       failures.push_back({actor.connection_id, actor_id});
     }
   }
+  // Expiry is visible at the upcoming boundary: disconnect at next N is
+  // resumable while next_tick < N+200, including before its final grace tick.
+  for (auto& [id,player] : players_) {
+    (void)id;
+    if (!player.connected && player.disconnect_deadline && executed_ticks_+1 >= *player.disconnect_deadline)
+      player.disconnect_deadline.reset();
+  }
   ++executed_ticks_;
   if (executed_ticks_ == session_ticks) transition("FINISHED");
   return failures;
@@ -197,10 +206,31 @@ void Room::leave(std::uint64_t connection_id) {
       player.applied_seq.reset();
       player.input_attempts = 0;
       player.realtime_ready = false;
+      player.disconnect_deadline.reset();
     }
   }
   evaluate_start_condition();
 }
+void Room::disconnect(std::uint64_t connection_id) {
+  assert_owner();
+  for (auto& [id,player] : players_) {
+    (void)id;
+    if (player.connection_id != connection_id || !player.connected) continue;
+    player.connected = false; player.direction = Direction::stop;
+    player.pending.clear(); player.applied_seq.reset(); player.input_attempts = 0; player.realtime_ready = false;
+    player.disconnect_deadline = executed_ticks_+reconnect_grace_ticks;
+  }
+}
+bool Room::reconnect(const std::string& player_id, std::uint64_t connection_id) {
+  assert_owner();
+  const auto found = players_.find(player_id);
+  if (found == players_.end() || status_ == "CLOSED") return false;
+  auto& player = found->second;
+  if (player.connected || !player.disconnect_deadline || executed_ticks_ >= *player.disconnect_deadline) return false;
+  player.connected = true; player.connection_id = connection_id; player.disconnect_deadline.reset();
+  player.direction = Direction::stop; player.realtime_ready = false;
+  return true;
+}
 void Room::close() {
   assert_owner();
   if (status_ == "ABSENT" || status_ == "CLOSED") return;
@@ -212,6 +242,7 @@ void Room::close() {
     player.applied_seq.reset();
     player.input_attempts = 0;
     player.realtime_ready = false;
+    player.disconnect_deadline.reset();
   }
   transition("CLOSED");
 }
@@ -226,7 +257,7 @@ Json Room::view() const {
   for (const auto& [id, player] : players_) {
     result["players"].push_back(Json{{"player_id", id}, {"slot", player.slot}, {"x", player.x}, {"y", player.y},
       {"direction", direction_name(player.direction)}, {"score", player.score},
-      {"connectivity", player.connected ? "CONNECTED" : "LEFT"}, {"last_tag_tick", player.last_tag_tick}});
+      {"connectivity", player.connectivity()}, {"last_tag_tick", player.last_tag_tick}});
   }
   return result;
 }
diff --git a/src/game.hpp b/src/game.hpp
index a6e27fb..e99aef5 100644
--- a/src/game.hpp
+++ b/src/game.hpp
@@ -24,6 +24,7 @@ inline constexpr int tick_duration_ms = 50;
 inline constexpr int max_catch_up_ticks = 4;
 inline constexpr std::uint64_t max_future_input_ticks = 4;
 inline constexpr std::size_t max_input_attempts_per_tick = 4;
+inline constexpr int reconnect_grace_ticks = 200;
 
 enum class Direction { stop, north, east, south, west };
 std::string direction_name(Direction direction);
@@ -64,6 +65,8 @@ struct Player {
   std::optional<std::uint64_t> applied_seq;
   std::size_t input_attempts = 0;
   bool realtime_ready = false;
+  std::optional<int> disconnect_deadline = std::nullopt;
+  std::string connectivity() const { return connected ? "CONNECTED" : disconnect_deadline ? "DISCONNECTED" : "LEFT"; }
   std::uint64_t last_accepted_seq() const { return last_accepted_input ? last_accepted_input->seq : 0; }
 };
 struct ActionFailure {
@@ -82,6 +85,8 @@ class Room {
   InputResult input(const std::string& player_id, Input input);
   std::vector<ActionFailure> tick();
   void leave(std::uint64_t connection_id);
+  void disconnect(std::uint64_t connection_id);
+  bool reconnect(const std::string& player_id, std::uint64_t connection_id);
   void close();
   Json view() const;
   const std::map<std::string, Player>& players() const { return players_; }
diff --git a/src/lifecycle_scenario.cpp b/src/lifecycle_scenario.cpp
index 1831cdb..3e2fef4 100644
--- a/src/lifecycle_scenario.cpp
+++ b/src/lifecycle_scenario.cpp
@@ -207,10 +207,11 @@ Json lifecycle_cell(const Json& scenario, const std::string& state, const std::s
     }
   }
   auto expected = before;
-  for (auto& player : expected["players"]) if (player.at("player_id") == actor_id) player["connectivity"] = "LEFT";
+  for (auto& player : expected["players"]) if (player.at("player_id") == actor_id)
+    player["connectivity"] = action == "CONNECTION_CLOSE" ? "DISCONNECTED" : "LEFT";
   const auto after = fixture.server.room().view();
   ensure(after == expected && fixture.clock.now_ms == before_clock && fixture.server.room().pending_count() == 0,
-         "leave/close must only mark alpha LEFT and preserve identity, slot, state, tick and other player");
+         "leave/close preserves identity, slot, state, tick and other player with the active connectivity transition");
   Json row{{"state", state}, {"action", action}, {"identities", identities}, {"before", before}, {"after", after},
     {"finished_messages", finished}, {"manual_clock_ms", fixture.clock.now_ms}, {"pending_inputs", fixture.server.room().pending_count()},
     {"cleanup", fixture.finish()}, {"result", "PASS"}};
diff --git a/src/replay.cpp b/src/replay.cpp
index c0a28a7..02ed941 100644
--- a/src/replay.cpp
+++ b/src/replay.cpp
@@ -32,7 +32,7 @@ std::string canonical_state(const Room& room) {
   for (const auto& [id, player] : room.players()) {
     record += "player=" + id + "|slot=" + decimal(player.slot) + "|x=" + decimal(player.x) +
       "|y=" + decimal(player.y) + "|dir=" + direction_name(player.direction) +
-      "|score=" + decimal(player.score) + "|conn=" + (player.connected ? "CONNECTED" : "LEFT") +
+      "|score=" + decimal(player.score) + "|conn=" + player.connectivity() +
       "|last_seq=" + decimal(player.last_accepted_seq()) + "|last_tag_tick=" + decimal(player.last_tag_tick) + "\n";
   }
   return record;
@@ -65,7 +65,7 @@ void ReplayLog::start(const Room& room) {
     throw std::logic_error("replay starts once at the initial RUNNING boundary");
   Json initial = Json::array();
   for (const auto& [id, player] : room.players()) initial.push_back(Json{{"player_id",id},{"slot",player.slot},
-    {"spawn",Json::array({player.x,player.y})},{"connectivity",player.connected ? "CONNECTED" : "LEFT"}});
+    {"spawn",Json::array({player.x,player.y})},{"connectivity",player.connectivity()}});
   Json artifact{{"contract_version",1},{"room_id",room.id()},{"owner_epoch",0},{"complete",true},{"executed_ticks",0},
     {"tick_duration_ms",tick_duration_ms},{"session_duration_ticks",session_ticks},
     {"initial_players",std::move(initial)},{"ticks",Json::array()}};
diff --git a/src/replay.hpp b/src/replay.hpp
index a5f4c90..4758606 100644
--- a/src/replay.hpp
+++ b/src/replay.hpp
@@ -15,6 +15,7 @@ class ReplayLog {
   void start(const Room& room);
   void accepted_input(const Room& room, const std::string& player_id);
   void left(const Room& room, const std::string& player_id, const std::string& kind);
+  void reconnected(const Room& room, const std::string& player_id) { left(room,player_id,"RECONNECT"); }
   void finish_tick(const Room& room);
   std::string serialize() const;
   const Json& artifact() const;
diff --git a/src/snapshot.cpp b/src/snapshot.cpp
index b55967d..6345f99 100644
--- a/src/snapshot.cpp
+++ b/src/snapshot.cpp
@@ -21,9 +21,9 @@ void SnapshotStream::acknowledge(std::uint64_t sequence, const std::optional<std
 Json SnapshotStream::publish(const Room& room, const std::string& state_hash) {
   Json players = Json::array();
   for (const auto& [id, player] : room.players()) {
-    if (!player.connected) continue;
+    if (!player.connected && !player.disconnect_deadline) continue;
     players.push_back(Json{{"player_id",id},{"slot",player.slot},{"x",player.x},{"y",player.y},
-      {"direction",direction_name(player.direction)},{"score",player.score},{"connectivity","CONNECTED"}});
+      {"direction",direction_name(player.direction)},{"score",player.score},{"connectivity",player.connectivity()}});
   }
   auto full = message("SNAPSHOT");
   full.update(Json{{"snapshot_seq",++sequence_},{"room_id",room.id()},{"tick",room.executed_ticks()-1},
diff --git a/src/snapshot.hpp b/src/snapshot.hpp
index 4c05989..4027b9e 100644
--- a/src/snapshot.hpp
+++ b/src/snapshot.hpp
@@ -11,6 +11,8 @@ class SnapshotStream {
   Json publish(const Room& room, const std::string& state_hash);
   void acknowledge(std::uint64_t sequence, const std::optional<std::string>& state_hash = std::nullopt, bool resync_required = false);
   void clear() { retained_.clear(); acknowledged_.reset(); resync_reason_.clear(); last_full_reason_.clear(); }
+  void reconnect_after(std::uint64_t sequence) { clear(); sequence_ = sequence; resync_reason_ = "RECONNECT"; }
+  std::uint64_t last_sequence() const { return sequence_; }
   std::size_t size() const { return retained_.size(); }
   std::size_t high_water() const { return high_water_; }
   Json metrics() const;
diff --git a/src/transport.cpp b/src/transport.cpp
index 2207950..89e57a1 100644
--- a/src/transport.cpp
+++ b/src/transport.cpp
@@ -77,13 +77,14 @@ std::string request_error(const Json& value) {
   const auto type = value.at("type").get<std::string>();
   if (type == "HELLO") return {};
   if (realtime_type(type) && type != "INPUT" && type != "SNAPSHOT_ACK") return {};
-  if (type != "CREATE_ROOM" && type != "JOIN_ROOM" && type != "LEAVE_ROOM" && !realtime_type(type))
+  if (type != "CREATE_ROOM" && type != "JOIN_ROOM" && type != "LEAVE_ROOM" && type != "RECONNECT" && !realtime_type(type))
     return "MESSAGE_TYPE_UNKNOWN";
   const auto string_field = [&](const char* name) { return value.contains(name) && value.at(name).is_string(); };
   if (!string_field("session_id")) return "MESSAGE_INVALID";
   if (type == "CREATE_ROOM") return {};
   if (!string_field("room_id")) return "MESSAGE_INVALID";
   if (type == "JOIN_ROOM") return {};
+  if (type == "RECONNECT") return string_field("resume_token") ? std::string{} : "MESSAGE_INVALID";
   if (!string_field("player_id")) return "MESSAGE_INVALID";
   if (type == "LEAVE_ROOM") return {};
   // Full INPUT validation is deferred until authenticated owner admission, so
@@ -268,9 +269,11 @@ std::string Server::new_id(const std::string& prefix, std::uint64_t number) cons
   if (prefix == "room" && fixture_room_id_) return *fixture_room_id_;
   if (prefix == "player" && !fixture_player_ids_.empty()) return fixture_player_ids_.at(static_cast<std::size_t>(number-1));
 #endif
-  // Room identity retains the process nonce. Player identity is room-scoped;
-  // at most8 joins need a one-digit suffix. R17/P8 keeps max8 FULL <=1200.
-  if (prefix == "room") return "r"+nonce_;
+  // Public Room identity needs three fewer bytes for the reachable G11
+  // seven-DISCONNECTED full. Session IDs and 128-bit credentials are unchanged.
+  // Player identity is room-scoped; at most8 joins need one suffix digit.
+  // R14/P8 keeps the seven-DISCONNECTED max8 FULL within1200 bytes.
+  if (prefix == "room") return "r"+nonce_.substr(0,13);
   if (prefix == "player") return "p"+nonce_.substr(0,6)+std::to_string(number);
   std::ostringstream out;
   out << prefix << '-' << nonce_ << '-' << std::setw(10) << std::setfill('0') << number;
@@ -475,6 +478,12 @@ void Server::bind_datagram(Connection& conn, const Envelope& envelope) {
   send_realtime(conn.id,reply);
   const auto before = room_.status(); room_.bind_realtime(conn.id);
   if (before == "LOBBY" && room_.status() == "RUNNING") start_room();
+  else if (conn.full_after_bind) {
+    // Connectivity changed between ticks; the cached completed-tick hash is
+    // historical. Capture current STOP/CONNECTED state without rewriting it.
+    publish_snapshot(conn,sha256(canonical_state(room_)));
+  }
+  conn.full_after_bind = false;
 }
 void Server::start_room() {
   replay_.start(room_);
@@ -486,16 +495,56 @@ void Server::publish_snapshots(const std::string& state_hash) {
   for (const auto& [fd, conn] : connections_) {
     (void)fd;
     const auto player = room_.players().find(conn.player_id);
-    if (player != room_.players().end() && player->second.connected) ids.push_back(conn.id);
+    if (player != room_.players().end() && player->second.connected && player->second.connection_id == conn.id &&
+        player->second.realtime_ready && conn.udp_endpoint) ids.push_back(conn.id);
   }
   for (const auto id : ids) {
     if (auto* conn = connection(id)) {
-      auto snapshot = conn->snapshots.publish(room_,state_hash);
-      snapshot_retention_high_water_ = std::max(snapshot_retention_high_water_,conn->snapshots.high_water());
-      send_realtime(id,snapshot);
+      publish_snapshot(*conn,state_hash);
     }
   }
 }
+void Server::publish_snapshot(Connection& conn, const std::string& state_hash) {
+  auto snapshot = conn.snapshots.publish(room_,state_hash);
+  snapshot_retention_high_water_ = std::max(snapshot_retention_high_water_,conn.snapshots.high_water());
+  if (const auto record = resume_.find(conn.player_id); record != resume_.end())
+    record->second.last_snapshot_seq = conn.snapshots.last_sequence();
+  send_realtime(conn.id,snapshot);
+}
+void Server::reconnect(Connection& conn, const Json& value) {
+  const auto reject = [&](const std::string& code) {
+    ++errors_[code]; queue(conn.id,error_message(code,"resume identity, credential or grace invalid"));
+  };
+  if (conn.session_id.empty() || !conn.player_id.empty() || value.at("room_id").get<std::string>() != room_.id()) {
+    reject("RECONNECT_INVALID"); return;
+  }
+  const auto session = value.at("session_id").get<std::string>();
+  const auto player = std::find_if(room_.players().begin(),room_.players().end(),[&](const auto& item) {
+    return item.second.session_id == session;
+  });
+  if (player == room_.players().end()) { reject("RECONNECT_INVALID"); return; }
+  const auto saved = resume_.find(player->first);
+  if (saved == resume_.end() || saved->second.token.empty() || value.at("resume_token").get<std::string>() != saved->second.token) {
+    reject("RECONNECT_INVALID"); return;
+  }
+  if (player->second.connected) { reject("RECONNECT_INVALID"); return; }
+  if (!player->second.disconnect_deadline || room_.executed_ticks() >= *player->second.disconnect_deadline) {
+    reject("RECONNECT_EXPIRED"); return;
+  }
+  const auto resume_token = new_bind_token(), bind_token = new_bind_token();
+  if (!room_.reconnect(player->first,conn.id)) { reject("RECONNECT_INVALID"); return; }
+  // Retire the HELLO provisional identity and credential. The stable session
+  // belongs only to this new connection; the old TCP/UDP binding is gone.
+  conn.session_id = session; conn.player_id = player->first; conn.udp_endpoint.reset();
+  conn.bind_token = bind_token; conn.token_issued_ms = monotonic_now_(); conn.full_after_bind = true;
+  conn.snapshots.reconnect_after(saved->second.last_snapshot_seq); saved->second.token = resume_token;
+  if (room_.status() == "RUNNING") replay_.reconnected(room_,conn.player_id);
+  auto reply = message("RECONNECTED");
+  reply.update(Json{{"session_id",session},{"room_id",room_.id()},{"player_id",conn.player_id},
+    {"last_accepted_seq",player->second.last_accepted_seq()},{"resume_token",resume_token},
+    {"udp_bind_token",bind_token},{"udp_port",udp_port_},{"owner_epoch",0}});
+  queue(conn.id,std::move(reply));
+}
 void Server::handle(const Envelope& envelope) {
   auto* conn = connection(envelope.connection_id);
   if (conn == nullptr) return;
@@ -529,6 +578,7 @@ void Server::handle(const Envelope& envelope) {
       reply["udp_bind_token"] = conn->bind_token; reply["udp_port"] = udp_port_;
       queue(id, std::move(reply)); return;
     }
+    if (type == "RECONNECT") { reconnect(*conn,value); return; }
     if (type != "CREATE_ROOM" && type != "JOIN_ROOM" && type != "LEAVE_ROOM" && type != "INPUT" && type != "SNAPSHOT_ACK" && type != "PING") {
       reject("MESSAGE_TYPE_UNKNOWN", "unknown message type"); return;
     }
@@ -550,9 +600,11 @@ void Server::handle(const Envelope& envelope) {
       }
       conn->player_id = new_id("player", next_player_++);
       const auto& player = room_.join(conn->player_id, conn->session_id, id);
+      const auto resume_token = new_bind_token(); resume_.emplace(player.id,ResumeRecord{resume_token});
       if (conn->udp_endpoint) room_.bind_realtime(id);
       Json reply = message("ROOM_JOINED"); reply["room_id"] = room_.id(); reply["player_id"] = player.id;
       reply["slot"] = player.slot; reply["status"] = room_.status();
+      reply["resume_token"] = resume_token;
       queue(id, std::move(reply));
       if (room_.status() == "RUNNING") start_room();
       return;
@@ -595,12 +647,11 @@ void Server::handle(const Envelope& envelope) {
 void Server::leave_room(std::uint64_t connection_id, const std::string& kind) {
   const auto previous_status = room_.status();
   std::string player_id;
-  if (room_.status() == "RUNNING") {
-    for (const auto& [id, player] : room_.players())
-      if (player.connection_id == connection_id && player.connected) player_id = id;
-  }
-  room_.leave(connection_id);
-  if (!player_id.empty()) replay_.left(room_,player_id,kind);
+  for (const auto& [id, player] : room_.players())
+    if (player.connection_id == connection_id && player.connected) player_id = id;
+  if (kind == "DISCONNECT") room_.disconnect(connection_id);
+  else { room_.leave(connection_id); resume_.erase(player_id); }
+  if (!player_id.empty() && previous_status == "RUNNING") replay_.left(room_,player_id,kind);
   if (auto* conn = connection(connection_id)) conn->snapshots.clear();
   if (previous_status == "LOBBY" && room_.status() == "RUNNING") start_room();
 }
@@ -659,11 +710,12 @@ void Server::advance_one_tick() {
 }
 Json Server::metrics() const {
   auto streams = Json::object();
-  std::size_t bound = 0, tokens = 0;
+  std::size_t bound = 0, tokens = 0, sessions = 0;
   for (const auto& [fd, conn] : connections_) {
     (void)fd;
     if (!conn.player_id.empty()) streams[conn.player_id] = conn.snapshots.metrics();
     bound += conn.udp_endpoint.has_value(); tokens += !conn.bind_token.empty();
+    sessions += !conn.session_id.empty();
   }
   return Json{{"received_messages", received_messages_}, {"sent_messages", sent_messages_},
     {"mailbox_high_water", mailbox_high_water_}, {"outbound_control_high_water", outbound_high_water_},
@@ -672,6 +724,7 @@ Json Server::metrics() const {
     {"udp_received_datagrams",received_datagrams_},{"udp_sent_datagrams",sent_datagrams_},
     {"udp_payload_high_water",datagram_high_water_},{"udp_outbound_high_water",outbound_datagram_high_water_},
     {"udp_receive_buffer_bytes",max_datagram_bytes+1},{"udp_bound_endpoints",bound},{"udp_bind_tokens",tokens},
+    {"active_sessions",sessions},{"resume_records",resume_.size()},{"resume_record_capacity",max_players},
     {"input_attempt_per_player_high_water", room_.input_attempt_high_water()},
     {"replay_bytes_high_water",replay_.high_water_bytes()},
     {"replay_capture_complete",replay_.complete()},{"replay_capture_error",replay_.failure()},
@@ -687,17 +740,19 @@ Json Server::metrics() const {
       {"operational_state", last_batch_.overloaded ? "OVERLOADED" : "NORMAL"}}}};
 }
 Json Server::cleanup() const {
-  std::size_t queued = 0, parser_buffered = 0, input_attempts = 0, retained_snapshots = 0, endpoints = 0, tokens = 0;
+  std::size_t queued = 0, parser_buffered = 0, input_attempts = 0, retained_snapshots = 0, endpoints = 0, tokens = 0, sessions = 0, grace = 0;
   for (const auto& [fd, conn] : connections_) {
     (void)fd; queued += conn.outbound.size(); parser_buffered += conn.parser.buffered_bytes();
     retained_snapshots += conn.snapshots.size();
     endpoints += conn.udp_endpoint.has_value(); tokens += !conn.bind_token.empty();
+    sessions += !conn.session_id.empty();
   }
-  for (const auto& [id, player] : room_.players()) { (void)id; input_attempts += player.input_attempts; }
+  for (const auto& [id, player] : room_.players()) { (void)id; input_attempts += player.input_attempts; grace += player.disconnect_deadline.has_value(); }
   return Json{{"server_connections", connections_.size()}, {"server_descriptors", owned_descriptors().size()},
     {"mailbox_messages", mailbox_.size()}, {"pending_inputs", room_.pending_count()}, {"outbound_messages", queued},
     {"input_attempts", input_attempts},
     {"retained_snapshots",retained_snapshots},
+    {"active_sessions",sessions},{"resume_records",resume_.size()},{"grace_deadlines",grace},
     {"udp_bound_endpoints",endpoints},{"udp_bind_tokens",tokens},{"udp_descriptors",datagram_.get() >= 0 ? 1 : 0},
     {"replay_bytes",replay_.bytes()},{"replay_pending_events",replay_.pending_events()},
     {"parser_buffered_bytes", parser_buffered}, {"parser_storage_bytes", connections_.size() * FrameParser::storage_bytes},
@@ -719,6 +774,7 @@ void Server::shutdown() {
   datagram_.reset();
   drain_mailbox();
   room_.close();
+  resume_.clear();
   replay_.clear();
   accumulator_.reset(0); last_batch_ = {};
   // Only transport flushing uses a wall deadline; no simulation runs here.
@@ -783,10 +839,13 @@ std::optional<Json> TcpClient::try_receive() {
   count = ::recv(fd_.get(), bytes.data(), total, 0);
   if (count != static_cast<ssize_t>(total)) throw std::runtime_error("client complete-frame read failed");
   Json value = decode_complete_frame(std::span(bytes).first(total));
-  if (value.at("type") == "WELCOME" && value.contains("udp_bind_token")) {
+  if ((value.at("type") == "WELCOME" || value.at("type") == "RECONNECTED") && value.contains("udp_bind_token")) {
     bind_token_ = value.at("udp_bind_token").get<std::string>(); udp_port_ = value.at("udp_port").get<std::uint16_t>();
     value.erase("udp_bind_token");
   }
+  if ((value.at("type") == "ROOM_JOINED" || value.at("type") == "RECONNECTED") && value.contains("resume_token")) {
+    resume_token_ = value.at("resume_token").get<std::string>(); value.erase("resume_token");
+  }
   if (observations_.size() == 4096) throw std::runtime_error("client observation bound exceeded");
   observations_.push_back(value);
   return value;
@@ -811,6 +870,10 @@ Json TcpClient::bind_request(const std::string& session_id, int owner_epoch) con
   auto value = message("UDP_BIND"); value["session_id"] = session_id;
   value["udp_bind_token"] = bind_token_; value["owner_epoch"] = owner_epoch; return value;
 }
+Json TcpClient::reconnect_request(const std::string& session_id, const std::string& room_id) const {
+  auto value = message("RECONNECT"); value.update(Json{{"session_id",session_id},{"room_id",room_id},{"resume_token",resume_token_}});
+  return value;
+}
 UdpClient::UdpClient(std::uint16_t server_port) : fd_(::socket(AF_INET,SOCK_DGRAM,0)) {
   if (fd_.get() < 0) system_failure("UDP client socket");
   nonblocking(fd_.get()); sockaddr_in local{}; local.sin_family = AF_INET; local.sin_addr.s_addr = htonl(INADDR_LOOPBACK);
diff --git a/src/transport.hpp b/src/transport.hpp
index 752f73e..7ce1200 100644
--- a/src/transport.hpp
+++ b/src/transport.hpp
@@ -100,7 +100,11 @@ class Server {
     std::string bind_token = {};
     std::int64_t token_issued_ms = 0;
     std::optional<sockaddr_in> udp_endpoint = {};
+    bool full_after_bind = false;
   };
+  // At most one record per bounded Room player, including expired players
+  // until Room teardown so their current credential gets EXPIRED, not reset.
+  struct ResumeRecord { std::string token; std::uint64_t last_snapshot_seq = 0; };
   struct Envelope {
     std::uint64_t connection_id; Json value; std::string parser_error;
     std::optional<sockaddr_in> udp_endpoint = {};
@@ -137,6 +141,8 @@ class Server {
   void broadcast(const Json& value);
   void start_room();
   void publish_snapshots(const std::string& state_hash);
+  void publish_snapshot(Connection& conn, const std::string& state_hash);
+  void reconnect(Connection& conn, const Json& value);
   void handle(const Envelope& envelope);
   void leave_room(std::uint64_t connection_id, const std::string& kind);
   std::int64_t read_monotonic();
@@ -158,6 +164,7 @@ class Server {
   std::set<std::uint64_t> disconnected_;
   Room room_;
   ReplayLog replay_;
+  std::map<std::string,ResumeRecord> resume_;
   std::string nonce_;
   std::uint64_t next_connection_ = 1;
   std::uint64_t next_player_ = 1;
@@ -194,16 +201,19 @@ class TcpClient {
   bool peer_closed() const;
   Json receive(Server& server);
   Json receive_type(Server& server, const std::string& type);
-  void close() { fd_.reset(); bind_token_.clear(); }
+  void close() { fd_.reset(); bind_token_.clear(); resume_token_.clear(); }
   int descriptor() const { return fd_.get(); }
   bool has_bind_token() const { return !bind_token_.empty(); }
+  bool has_resume_token() const { return !resume_token_.empty(); }
   std::uint16_t udp_port() const { return udp_port_; }
   Json bind_request(const std::string& session_id, int owner_epoch = 0) const;
+  Json reconnect_request(const std::string& session_id, const std::string& room_id) const;
   const std::vector<Json>& observations() const { return observations_; }
  private:
   Fd fd_;
   std::vector<Json> observations_;
   std::string bind_token_;
+  std::string resume_token_;
   std::uint16_t udp_port_ = 0;
 };
 // Real UDP test/CLI client; credentials remain in the TCP client's private
diff --git a/tests/g07.cpp b/tests/g07.cpp
index 287a796..faeccd4 100644
--- a/tests/g07.cpp
+++ b/tests/g07.cpp
@@ -431,6 +431,7 @@ ReplayRun verify_replay(const Json& artifact) {
   // The CLI's bounded parser has validated completeness and record framing.
   const int descriptors_before = Fd::live(); Room room; ReplayFixture::offline(room,artifact);
   ReplayRun run; run.records = Json::array(); Json hashes = Json::array(), actions = Json::array();
+  std::uint64_t next_connection = max_players+1;
   Json divergence = nullptr;
   for (const auto& tick : artifact.at("ticks")) {
     const int before_tick = room.executed_ticks();
@@ -445,8 +446,9 @@ ReplayRun verify_replay(const Json& artifact) {
         const auto result = admit_input(room,player_id,event);
         require(!result.error && !result.duplicate, "recorded accepted input failed production admission");
       } else {
-        require(event.at("kind") == "LEAVE_ROOM" || event.at("kind") == "DISCONNECT", "known lifecycle record");
-        room.leave(room.players().at(player_id).connection_id);
+        if (event.at("kind") == "DISCONNECT") room.disconnect(room.players().at(player_id).connection_id);
+        else if (event.at("kind") == "RECONNECT") require(room.reconnect(player_id,next_connection++),"recorded resume follows production grace");
+        else { require(event.at("kind") == "LEAVE_ROOM", "known lifecycle record"); room.leave(room.players().at(player_id).connection_id); }
       }
     }
     for (const auto& failure : room.tick()) actions.push_back(Json{{"tick",before_tick},
diff --git a/tests/g09.cpp b/tests/g09.cpp
index 1d9bd0d..c17804b 100644
--- a/tests/g09.cpp
+++ b/tests/g09.cpp
@@ -32,7 +32,7 @@ Json state(const Room& room) {
 }
 }
 struct UdpFixture {
-  static Json maximum_full(const std::string& room_id, const std::vector<std::string>& ids) {
+  static Json maximum_full(const std::string& room_id, const std::vector<std::string>& ids, bool disconnected = false) {
     // Only test-owned scalar state is set. No simulation or extra campaign is
     // used to measure the production full serializer at reachable field widths.
     Room room; room.create(room_id);
@@ -41,6 +41,7 @@ struct UdpFixture {
       p.direction = Direction::north; p.score = 60;
     }
     room.status_ = "FINISHED"; room.executed_ticks_ = 1200;
+    if (disconnected) for (std::size_t i = 0; i+1 < ids.size(); ++i) room.disconnect(i+1);
     SnapshotStream stream; auto full = stream.publish(room,sha256(canonical_state(room)));
     full["snapshot_seq"] = 601; return full;
   }
@@ -52,7 +53,7 @@ struct UdpFixture {
       const auto id = row.at("player_id").get<std::string>();
       need(identifier(id) && unique.insert(id).second,"ASCII1..64 unique fixture player identifier"); ids.push_back(id);
     }
-    (void)encode_datagram(maximum_full(room,ids)); // Refuse an oversized fixture before any state/transport change.
+    (void)encode_datagram(maximum_full(room,ids,true)); // Refuse an oversized fixture before any state/transport change.
     server.fixture_room_id_ = room; server.fixture_player_ids_ = std::move(ids);
   }
   static Json outbound_probe(Server& server, const std::string& player_id) {
@@ -61,6 +62,13 @@ struct UdpFixture {
     const auto room_id = server.new_id("room",1); const auto full = maximum_full(room_id,ids);
     const auto encoded = encode_datagram(full);
     need(full.at("players").size() == 8 && encoded.size() <= max_datagram_bytes,"normal max8 full fits exactly bounded UDP");
+    const auto mixed = maximum_full(room_id,ids,true); const auto mixed_bytes = encode_datagram(mixed);
+    need(mixed.at("players").size() == 8 && mixed_bytes.size() <= max_datagram_bytes,"seven DISCONNECTED remain visible within1200 bytes");
+    for (std::size_t i = 0; i < 8; ++i) {
+      const auto& p = mixed.at("players").at(i);
+      need(p.size() == 7 && p.at("connectivity") == (i < 7 ? "DISCONNECTED" : "CONNECTED") &&
+        p.at("direction") == (i < 7 ? "STOP" : "NORTH"),"unchanged seven-field projection includes disconnected STOP players");
+    }
     std::vector<std::string> long_ids; Json invalid{{"room_id",std::string(64,'R')},{"players",Json::array()}};
     for (int i = 0; i < 8; ++i) {
       long_ids.push_back(std::string(63,'P')+std::to_string(i)); invalid["players"].push_back(Json{{"player_id",long_ids.back()}});
@@ -78,6 +86,7 @@ struct UdpFixture {
     need(server.errors_["UDP_OUTBOUND_SIZE_INVALID"] == before+1 && server.sent_datagrams_ == sent,"real outbound size rejection emits no datagram");
     Json lengths = Json::array(); for (const auto& id : ids) lengths.push_back(id.size());
     return Json{{"players",8},{"room_id_length",room_id.size()},{"player_id_lengths",lengths},{"max_full_bytes",encoded.size()},
+      {"seven_disconnected_full_bytes",mixed_bytes.size()},{"seven_disconnected_full",mixed},
       {"oversized_fixture_rejected",rejected},{"outbound_size_rejections",1},{"sent_datagrams_delta",0},{"executed_ticks",0}};
   }
 };
diff --git a/tests/g11.cpp b/tests/g11.cpp
new file mode 100644
index 0000000..b9c5b47
--- /dev/null
+++ b/tests/g11.cpp
@@ -0,0 +1,313 @@
+#include "g11.hpp"
+#ifndef ARENA_TEST_FIXTURES
+#error G11 fixed identifiers are test-build only
+#endif
+#include <algorithm>
+#include <array>
+#include <chrono>
+#include <memory>
+#include <unistd.h>
+
+namespace arena {
+namespace {
+void resume_need(bool value, const std::string& text) { if (!value) throw std::runtime_error("G11: "+text); }
+Json resume_state(const Room& room) {
+  auto state = room.view(); state["owner_epoch"] = 0;
+  for (auto& row : state["players"]) {
+    const auto& p = room.players().at(row.at("player_id").get<std::string>());
+    row["last_seq"] = p.last_accepted_seq(); row["pending"] = p.pending.size();
+    row["applied_seq"] = p.applied_seq ? Json(*p.applied_seq) : Json(nullptr);
+    row["disconnect_deadline"] = p.disconnect_deadline ? Json(*p.disconnect_deadline) : Json(nullptr);
+  }
+  return state;
+}
+Json resume_visible(const Room& room) {
+  Json rows = Json::array();
+  for (const auto& [id,p] : room.players()) if (p.connected || p.disconnect_deadline)
+    rows.push_back(Json{{"player_id",id},{"slot",p.slot},{"x",p.x},{"y",p.y},{"score",p.score},
+      {"direction",direction_name(p.direction)},{"connectivity",p.connected ? "CONNECTED" : "DISCONNECTED"}});
+  return Json{{"room_id",room.id()},{"tick",room.executed_ticks()-1},{"owner_epoch",0},{"status",room.status()},{"players",rows}};
+}
+struct ResumePeer {
+  std::unique_ptr<TcpClient> tcp;
+  std::unique_ptr<UdpClient> udp;
+  std::vector<std::unique_ptr<UdpClient>> old_udp;
+  std::string session, player, role;
+  std::uint64_t latest = 0, snapshots = 0, acknowledgements = 0;
+  Json applied;
+};
+struct ResumeFixture {
+  int descriptors_before = Fd::live();
+  ManualClock clock;
+  Server server{clock,0,[this] { return clock.now_ms; }};
+  std::array<ResumePeer,2> peers;
+  std::size_t already_closed = 0;
+  Json records = Json::array(), hashes = Json::array(), inputs = Json::array(), boundaries = Json::array();
+  Json publications = Json::array(), immediate_fulls = Json::array(), joins = Json::array();
+  Json original_resume, current_resume, current_bind; // Credentials never enter evidence.
+  std::string room_id;
+  explicit ResumeFixture(const Json& scenario) {
+    inject_udp_fixture_ids(server,scenario);
+    for (std::size_t i = 0; i < peers.size(); ++i) {
+      auto& p = peers[i]; p.role = scenario.at("players").at(i).at("client").get<std::string>();
+      p.tcp = std::make_unique<TcpClient>(server.port()); p.tcp->send(message("HELLO"));
+      const auto welcome = p.tcp->receive_type(server,"WELCOME"); p.session = welcome.at("session_id").get<std::string>();
+      resume_need(p.tcp->has_bind_token() && !welcome.contains("udp_bind_token"),"private initial bind credential");
+      p.udp = std::make_unique<UdpClient>(server.udp_port());
+    }
+    auto create = message("CREATE_ROOM"); create["session_id"] = peers[0].session; peers[0].tcp->send(create);
+    room_id = peers[0].tcp->receive_type(server,"ROOM_CREATED").at("room_id").get<std::string>();
+    resume_need(scenario.at("room_id") == room_id,"ordinary fixed room allocation");
+    for (std::size_t i = 0; i < peers.size(); ++i) {
+      auto& p = peers[i]; auto join = message("JOIN_ROOM"); join.update(Json{{"session_id",p.session},{"room_id",room_id}});
+      p.tcp->send(join); const auto reply = p.tcp->receive_type(server,"ROOM_JOINED"); p.player = reply.at("player_id").get<std::string>();
+      resume_need(reply.at("status") == "LOBBY" && reply.at("slot") == i && p.player == scenario.at("players").at(i).at("player_id").get<std::string>() &&
+        p.tcp->has_resume_token() && !reply.contains("resume_token"),"ordinary join issues privately captured resume credential");
+      joins.push_back(Json{{"client",p.role},{"player_id",p.player},{"slot",i},{"status",reply.at("status")},{"resume_token_present",true}});
+    }
+    original_resume = peers[0].tcp->reconnect_request(peers[0].session,room_id); current_resume = original_resume;
+    for (std::size_t i = 0; i < peers.size(); ++i) {
+      auto& p = peers[i]; p.udp->bind(*p.tcp,server,p.session);
+      resume_need(server.room().status() == (i == 0 ? "LOBBY" : "RUNNING"),"normal all-joined UDP-ready start");
+    }
+    for (auto& p : peers) consume_snapshot(p,false);
+  }
+  template<class Predicate> void wait_for(Predicate ready, bool owner = true) {
+    const auto deadline = std::chrono::steady_clock::now()+std::chrono::seconds(5);
+    while (!ready() && std::chrono::steady_clock::now() < deadline) {
+      if (owner) server.pump(1); else server.poll_io(1);
+    }
+    resume_need(ready(),"socket/owner ceiling");
+  }
+  void consume_snapshot(ResumePeer& p, bool immediate) {
+    const auto wire = p.udp->receive_type(server,"SNAPSHOT"); const auto sequence = wire.at("snapshot_seq").get<std::uint64_t>();
+    const auto canonical = canonical_state(server.room()), hash = sha256(canonical); const auto authority = resume_visible(server.room());
+    resume_need(sequence == p.latest+1 && wire.at("room_id") == room_id && wire.at("tick") == server.room().executed_ticks()-1 &&
+      wire.at("owner_epoch") == 0 && wire.at("state_hash") == hash && encode_datagram(wire).size() <= max_datagram_bytes,
+      "actual monotonically sequenced snapshot has current canonical metadata");
+    std::map<std::string,Json> players; std::string status;
+    if (wire.at("kind") == "FULL") {
+      resume_need(wire.at("base_snapshot_seq").is_null() && wire.size() == 12,"full wire shape"); status = wire.at("status").get<std::string>();
+    } else {
+      resume_need(!immediate && wire.at("kind") == "DELTA" && wire.at("base_snapshot_seq") == p.latest && wire.size() == 11,
+        "healthy delta uses actual previous applied ACK"); status = p.applied.at("status").get<std::string>();
+      for (const auto& row : p.applied.at("players")) players.emplace(row.at("player_id").get<std::string>(),row);
+    }
+    for (const auto& id : wire.at("removed_player_ids")) players.erase(id.get<std::string>());
+    std::string previous;
+    for (const auto& row : wire.at("players")) {
+      const auto id = row.at("player_id").get<std::string>();
+      resume_need(row.size() == 7 && id > previous && (row.at("connectivity") == "CONNECTED" || row.at("connectivity") == "DISCONNECTED"),
+        "all seven visible fields, sorted IDs, no LEFT row"); previous = id; players[id] = row;
+    }
+    Json rows = Json::array(); for (const auto& [id,row] : players) { (void)id; rows.push_back(row); }
+    p.applied = Json{{"room_id",room_id},{"tick",wire.at("tick")},{"owner_epoch",0},{"status",status},{"players",rows}};
+    resume_need(p.applied == authority,"actual full/delta application includes DISCONNECTED and excludes only LEFT");
+    p.latest = sequence; ++p.snapshots;
+    if (immediate) {
+      const auto historical = server.replay().last_state_hash();
+      resume_need(wire.at("kind") == "FULL" && historical == hashes.back().get<std::string>() && hash != historical,
+        "fresh-bound full hashes current connectivity, never cached completed-tick state");
+      immediate_fulls.push_back(Json{{"next_tick",server.room().executed_ticks()},{"wire",wire},{"canonical_record",canonical},
+        {"state_hash",hash},{"state",resume_state(server.room())},{"client_projection",p.applied},
+        {"historical_tick_record",records.back()},{"historical_hash_unchanged",historical}});
+    }
+    auto ack = message("SNAPSHOT_ACK"); ack.update(Json{{"session_id",p.session},{"room_id",room_id},{"player_id",p.player},
+      {"snapshot_seq",p.latest},{"state_hash",hash},{"owner_epoch",0}});
+    const auto incoming = server.metrics().at("udp_received_datagrams").get<std::uint64_t>(); p.udp->send(ack);
+    wait_for([&] { return server.metrics().at("udp_received_datagrams") == incoming+1 && server.cleanup().at("mailbox_messages") == 0; });
+    resume_need(server.metrics().at("snapshot_streams").at(p.player).at("acknowledged_seq") == p.latest,"actual applied ACK is owned");
+    ++p.acknowledgements;
+    publications.push_back(Json{{"client",p.role},{"snapshot_seq",sequence},{"tick",wire.at("tick")},{"kind",wire.at("kind")},
+      {"base",wire.at("base_snapshot_seq")},{"state_hash",hash},{"visible_players",rows.size()},{"ack_owned",true},{"immediate",immediate}});
+  }
+  Json input_wire(const Json& event) const {
+    const auto i = event.at("client") == "alpha" ? 0U : 1U; const auto& p = peers[i]; auto value = message("INPUT");
+    value.update(Json{{"session_id",p.session},{"room_id",room_id},{"player_id",p.player},{"owner_epoch",event.at("owner_epoch")},
+      {"seq",event.at("seq")},{"target_tick",event.at("target_tick")},{"direction",event.at("direction")},
+      {"tag_target_player_id",event.at("tag_target_role").is_null() ? Json(nullptr) : Json(peers[1].player)}});
+    return value;
+  }
+  void input(const Json& wire, std::size_t index, UdpClient& endpoint, const std::string& expected, const std::string& label, bool probe = false) {
+    const auto before = resume_state(server.room()); endpoint.send(wire); Json reply;
+    if (expected == "ACCEPTED" || expected == "DUPLICATE") reply = peers[index].udp->receive_type(server,"INPUT_ACK");
+    else reply = peers[index].tcp->receive_type(server,"ERROR");
+    resume_need(reply.at("code") == expected,"actual input admission code"); const auto after = resume_state(server.room());
+    if (expected == "ACCEPTED" || expected == "DUPLICATE")
+      resume_need(reply.at("seq") == wire.at("seq") && reply.at("tick") == server.room().executed_ticks() &&
+        reply.at("last_accepted_seq") == server.room().players().at(peers[index].player).last_accepted_seq(),"actual ACK sequence and admission boundary");
+    if (expected != "ACCEPTED") resume_need(before == after,"rejected/duplicate input leaves authority and pending unchanged");
+    auto safe = wire; safe.erase("session_id"); safe["session_alias"] = peers[index].role;
+    Json row{{"before_tick",server.room().executed_ticks()},{"endpoint_alias",label},{"request",safe},{"response",reply}};
+    if (probe) { row["before"] = before; row["after"] = after; row["endpoint_open"] = !descriptor_closed(endpoint.descriptor()); }
+    inputs.push_back(row);
+  }
+  void step() {
+    const auto next = server.room().executed_ticks(); server.advance_one_tick();
+    resume_need(server.room().executed_ticks() == next+1 && clock.now_ms == (next+1)*50 && server.room().status() == "RUNNING",
+      "one uninterrupted actual Room tick");
+    const auto canonical = canonical_state(server.room()), hash = sha256(canonical);
+    resume_need(server.replay().last_state_hash() == hash,"actual replay closes the same authoritative tick");
+    records.push_back(Json{{"tick",next},{"state",resume_state(server.room())},{"canonical_record",canonical},{"state_hash",hash}}); hashes.push_back(hash);
+    if ((next+1)%2 == 0) for (auto& p : peers) if (server.room().players().at(p.player).connected) consume_snapshot(p,false);
+    for (auto& p : peers) {
+      if (p.tcp->descriptor() >= 0) resume_need(!p.tcp->try_receive(),"no unexpected control message");
+      resume_need(!p.udp->try_receive(),"no extra/early snapshot");
+      for (const auto& old : p.old_udp) resume_need(!old->try_receive(),"invalidated endpoint receives no later data");
+    }
+  }
+  void setup(const Json& scenario) {
+    for (int tick = 0; tick < 202; ++tick) {
+      for (const auto& event : scenario.at("setup_events")) if (event.at("before_tick") == tick) {
+        const auto i = event.at("client") == "alpha" ? 0U : 1U;
+        input(input_wire(event),i,*peers[i].udp,"ACCEPTED",peers[i].role+"_original");
+      }
+      step();
+    }
+    const auto& a = server.room().players().at(peers[0].player); const auto& b = server.room().players().at(peers[1].player);
+    resume_need(a.x == 50400 && a.y == 50000 && a.direction == Direction::east && a.score == 1 && a.last_accepted_seq() == 20 && a.last_tag_tick == 200 &&
+      b.x == 50000 && b.y == 50000 && b.direction == Direction::stop && b.score == 0 && b.last_accepted_seq() == 3,
+      "fixed seven-input202-tick setup");
+  }
+  void close_control(TcpClient& tcp) {
+    const auto before = resume_state(server.room()); const auto owned = server.owned_descriptors();
+    const auto connections = server.cleanup().at("server_connections").get<std::size_t>(); const int fd = tcp.descriptor();
+    tcp.close(); resume_need(descriptor_closed(fd),"actual client TCP closed"); ++already_closed;
+    wait_for([&] { return server.cleanup().at("server_connections") == connections-1; },false);
+    resume_need(resume_state(server.room()) == before,"transport callback never commits Room state");
+    const auto remaining = server.owned_descriptors();
+    for (const auto old : owned) if (std::find(remaining.begin(),remaining.end(),old) == remaining.end()) {
+      resume_need(descriptor_closed(old),"actual accepted TCP closed"); ++already_closed;
+    }
+    server.drain_mailbox();
+  }
+  void disconnect(const std::string& label) {
+    const auto before = resume_state(server.room()); const auto next = server.room().executed_ticks(); close_control(*peers[0].tcp);
+    const auto after = resume_state(server.room()); const auto& a = server.room().players().at(peers[0].player);
+    auto expected = before; expected["players"][0]["connectivity"] = "DISCONNECTED"; expected["players"][0]["direction"] = "STOP";
+    expected["players"][0]["pending"] = 0; expected["players"][0]["applied_seq"] = nullptr; expected["players"][0]["disconnect_deadline"] = 402;
+    resume_need(next == 202 && a.disconnect_deadline == 402 && after == expected && !descriptor_closed(peers[0].udp->descriptor()) &&
+      server.metrics().at("udp_bound_endpoints") == 1,"owner disconnect preserves stable state, stops input and sets deadline402");
+    boundaries.push_back(Json{{"kind","DISCONNECT"},{"label",label},{"next_tick",next},{"before",before},{"after",after},
+      {"old_udp_open",true},{"deadline",402},{"transport_callback_preserved_authority",true}});
+  }
+  void resume(const Json& credential, const std::string& expected, const std::string& label) {
+    const auto before = resume_state(server.room()); const auto old_connection = server.room().players().at(peers[0].player).connection_id;
+    auto candidate = std::make_unique<TcpClient>(server.port()); candidate->send(message("HELLO"));
+    const auto welcome = candidate->receive_type(server,"WELCOME"); const auto provisional = welcome.at("session_id").get<std::string>();
+    const auto provisional_bind = candidate->bind_request(provisional); candidate->send(credential); auto reply = candidate->receive(server);
+    const bool success = expected == "RECONNECTED";
+    resume_need(reply.at("type") == (success ? "RECONNECTED" : "ERROR") && (success || reply.at("code") == expected),"actual resume control result");
+    Json row{{"kind","RECONNECT"},{"token_alias",label},{"next_tick",server.room().executed_ticks()},{"code",expected},{"before",before}};
+    if (success) {
+      auto& p = peers[0]; const auto& model = server.room().players().at(p.player);
+      resume_need(reply.at("session_id") == p.session && reply.at("room_id") == room_id && reply.at("player_id") == p.player &&
+        reply.at("last_accepted_seq") == 20 && model.session_id == p.session && model.connection_id != old_connection &&
+        candidate->has_resume_token() && candidate->has_bind_token() && provisional != p.session,"stable identity and new connection, privately rotated credentials");
+      current_resume = candidate->reconnect_request(p.session,room_id); current_bind = candidate->bind_request(p.session);
+      const bool rotated = current_resume.at("resume_token") != credential.at("resume_token");
+      const bool fresh_bind = current_bind.at("udp_bind_token") != provisional_bind.at("udp_bind_token");
+      resume_need(rotated && fresh_bind && server.metrics().at("active_sessions") == 2 && server.metrics().at("udp_bound_endpoints") == 1,
+        "rotation retires provisional credential and waits for fresh observed endpoint");
+      p.tcp = std::move(candidate); p.old_udp.push_back(std::move(p.udp)); p.udp = std::make_unique<UdpClient>(server.udp_port());
+      auto expected_state = before; expected_state["players"][0]["connectivity"] = "CONNECTED"; expected_state["players"][0]["disconnect_deadline"] = nullptr;
+      resume_need(resume_state(server.room()) == expected_state,"resume never replays/reset accepted input, movement, score or tick");
+      reply.erase("session_id"); reply.erase("udp_port"); reply["session_alias"] = "alpha";
+      row["response"] = reply; row["resume_rotated"] = rotated; row["fresh_bind_credential"] = fresh_bind;
+      row["stable_session"] = true; row["new_connection"] = true; row["provisional_session_retired"] = true;
+    } else {
+      resume_need(resume_state(server.room()) == before && server.metrics().at("udp_bound_endpoints") == 1 &&
+        server.room().players().size() == 2,"invalid/expired resume leaves existing ownership, endpoint and scores untouched");
+      row["response"] = reply; close_control(*candidate); resume_need(resume_state(server.room()) == before,"retired rejected provisional session is not a player");
+    }
+    row["after"] = resume_state(server.room()); row["metrics"] = server.metrics(); boundaries.push_back(row);
+  }
+  void bind_resume() {
+    const auto before = resume_state(server.room()); auto& p = peers[0]; p.udp->send(current_bind);
+    const auto bound = p.udp->receive_type(server,"UDP_BOUND");
+    resume_need(bound.at("session_id") == p.session && bound.at("owner_epoch") == 0 && server.metrics().at("udp_bound_endpoints") == 2,
+      "fresh bind associates current stable session and endpoint");
+    consume_snapshot(p,true); resume_need(resume_state(server.room()) == before,"immediate bound full does not simulate");
+    for (const auto& old : p.old_udp) resume_need(!old->try_receive(),"no immediate full sent to invalidated endpoint");
+  }
+  void repeat_bind() {
+    const auto before = resume_state(server.room()); peers[0].udp->send(current_bind);
+    const auto reply = peers[0].tcp->receive_type(server,"ERROR");
+    resume_need(reply.at("code") == "UDP_BIND_INVALID" && resume_state(server.room()) == before && server.metrics().at("udp_bound_endpoints") == 2,
+      "consumed B2 rejected without moving current endpoint");
+    boundaries.push_back(Json{{"kind","UDP_BIND"},{"token_alias","B2_consumed"},{"code",reply.at("code")},
+      {"next_tick",server.room().executed_ticks()},{"before",before},{"after",resume_state(server.room())},{"endpoint_unchanged",true}});
+  }
+  Json finish() {
+    auto descriptors = server.owned_descriptors();
+    for (const auto& p : peers) {
+      if (p.tcp->descriptor() >= 0) descriptors.push_back(p.tcp->descriptor());
+      descriptors.push_back(p.udp->descriptor());
+      for (const auto& old : p.old_udp) descriptors.push_back(old->descriptor());
+    }
+    server.shutdown();
+    for (auto& p : peers) { p.tcp->close(); p.udp->close(); for (auto& old : p.old_udp) old->close(); }
+    for (const auto fd : descriptors) resume_need(descriptor_closed(fd),"actual final descriptor closure");
+    auto cleanup = server.cleanup(); for (const auto& [key,value] : cleanup.items()) { (void)key; resume_need(value == 0,"all active resources released"); }
+    resume_need(Fd::live() == descriptors_before,"no RAII descriptor leak"); cleanup["descriptor_checks"] = descriptors.size()+already_closed;
+    cleanup["all_descriptors_closed"] = true; cleanup["tracked_descriptor_delta"] = 0; return cleanup;
+  }
+};
+}
+Json run_resume_scenario(const Json& scenario) {
+  resume_need(scenario.at("thread") == "G11" && scenario.at("contract_version") == 1 && scenario.at("seed") == 7050 &&
+    scenario.at("clock").at("kind") == "manual" && scenario.at("clock").at("tick_duration_ms") == 50 && scenario.at("socket_ceiling_ms") == 5000 &&
+    scenario.at("players").size() == 2 && scenario.at("setup_events").size() == 7 && scenario.at("setup_execute_ticks") == 202 &&
+    scenario.at("cases").size() == 3 && scenario.at("disconnect").at("deadline_next_tick") == 402 && reconnect_grace_ticks == 200,
+    "frozen three-case dimensions");
+  Json cases = Json::array(), summary = Json::array(), descriptor_counts = Json::array(); std::size_t total_inputs = 0; int total_ticks = 0;
+  for (std::size_t index = 0; index < 3; ++index) {
+    const auto& cell = scenario.at("cases").at(index); ResumeFixture f(scenario); f.setup(scenario); f.disconnect("original_TCP");
+    if (index == 0) {
+      f.resume(f.original_resume,"RECONNECTED","R0"); f.bind_resume(); const auto r1 = f.current_resume;
+      f.disconnect("replacement_TCP"); f.resume(f.original_resume,"RECONNECT_INVALID","R0_consumed_while_DISCONNECTED");
+      f.resume(r1,"RECONNECTED","R1"); f.bind_resume(); f.repeat_bind();
+      const auto duplicate = f.input_wire(scenario.at("setup_events").back());
+      f.input(duplicate,0,*f.peers[0].udp,"DUPLICATE","current_B2",true); f.step();
+      const auto& stopped = f.server.room().players().at("player-00");
+      resume_need(stopped.x == 50400 && stopped.y == 50000 && stopped.direction == Direction::stop && stopped.last_accepted_seq() == 20,
+        "duplicate20 never repeats old EAST movement at202");
+      auto next = duplicate; next["seq"] = 21; next["target_tick"] = 203;
+      f.input(next,0,*f.peers[0].old_udp.front(),"UDP_BIND_INVALID","original_still_open",true);
+      f.input(next,0,*f.peers[0].udp,"ACCEPTED","current_B2",true); f.step();
+    } else {
+      const int count = cell.at("wait_grace_ticks").get<int>(); resume_need(count == (index == 1 ? 199 : 200),"frozen wait count");
+      for (int tick = 0; tick < count; ++tick) f.step();
+      const auto boundary = resume_state(f.server.room());
+      resume_need(f.server.room().executed_ticks() == (index == 1 ? 401 : 402) &&
+        boundary.at("players").at(0).at("connectivity") == (index == 1 ? "DISCONNECTED" : "LEFT"),"exact last-valid/expired boundary");
+      f.boundaries.push_back(Json{{"kind","GRACE_BOUNDARY"},{"next_tick",f.server.room().executed_ticks()},{"state",boundary}});
+      f.resume(f.original_resume,index == 1 ? "RECONNECTED" : "RECONNECT_EXPIRED","R0_current_unused");
+      if (index == 1) { f.bind_resume(); f.step(); }
+    }
+    const auto final = resume_state(f.server.room()); const auto& a = f.server.room().players().at("player-00"); const auto& b = f.server.room().players().at("player-01");
+    resume_need(f.server.room().executed_ticks() == cell.at("executed_ticks") && f.server.room().players().size() == 2 &&
+      a.x == (index == 0 ? 50800 : 50400) && a.y == 50000 && a.score == 1 && a.last_tag_tick == 200 && a.last_accepted_seq() == (index == 0 ? 21U : 20U) &&
+      a.direction == (index == 0 ? Direction::east : Direction::stop) && a.connectivity() == (index == 2 ? "LEFT" : "CONNECTED") &&
+      b.x == 50000 && b.y == 50000 && b.direction == Direction::stop && b.score == 0 && b.last_accepted_seq() == 3 && b.connected,
+      "final identity, movement, score, accepted sequence and connectivity");
+    Json counts = Json::object(); for (const auto& p : f.peers) counts[p.role] = Json{{"snapshots",p.snapshots},{"ACKs",p.acknowledgements},{"last_applied",p.latest}};
+    const auto metrics = f.server.metrics(); const auto journal = f.server.replay().serialize(); const auto lifecycle = f.server.replay().artifact().at("ticks");
+    Json lifecycle_events = Json::array(); for (const auto& tick : lifecycle) for (const auto& event : tick.at("events")) if (event.at("kind") != "INPUT") lifecycle_events.push_back(event);
+    Json logical{{"case",cell.at("name")},{"executed_ticks",f.server.room().executed_ticks()},
+      {"positions",Json{{"alpha",Json::array({a.x,a.y})},{"bravo",Json::array({b.x,b.y})}}},{"scores",Json{{"alpha",a.score},{"bravo",b.score}}},
+      {"last_seq",Json{{"alpha",a.last_accepted_seq()},{"bravo",b.last_accepted_seq()}}},
+      {"connectivity",Json{{"alpha",a.connectivity()},{"bravo",b.connectivity()}}},{"final_hash",f.hashes.back()},{"snapshot_counts",counts}};
+    const int executed = f.server.room().executed_ticks(); total_ticks += executed; total_inputs += f.inputs.size();
+    const auto cleanup = f.finish(); descriptor_counts.push_back(cleanup.at("descriptor_checks")); summary.push_back(logical);
+    cases.push_back(Json{{"case",cell.at("name")},{"result","PASS"},{"executed_ticks",executed},{"joins",f.joins},
+      {"inputs",f.inputs},{"boundaries",f.boundaries},{"tick_records",f.records},{"state_hashes",f.hashes},
+      {"immediate_fulls",f.immediate_fulls},{"publications",f.publications},{"snapshot_counts",counts},{"lifecycle_events",lifecycle_events},
+      {"journal_bytes",journal.size()},{"journal_hash",sha256(journal)},{"final_state",final},{"logical",logical},{"metrics",metrics},{"cleanup",cleanup}});
+  }
+  resume_need(total_ticks == 1008 && total_inputs == 24,"exactly one three-case1008-tick24-input campaign");
+  return Json{{"result","PASS"},{"thread","G11"},{"scenario_id",scenario.at("scenario_id")},{"process_id",::getpid()},
+    {"executed_ticks",total_ticks},{"input_attempts",total_inputs},{"cases",cases},{"logical_summary",summary},{"network_fault_runs",0},{"load_runs",0},
+    {"cleanup",Json{{"all_descriptors_closed",true},{"case_descriptor_checks",descriptor_counts},{"tracked_descriptor_delta",0},{"remaining_active_resources",0}}}};
+}
+}
diff --git a/tests/g11.hpp b/tests/g11.hpp
new file mode 100644
index 0000000..44a0cbf
--- /dev/null
+++ b/tests/g11.hpp
@@ -0,0 +1,5 @@
+#pragma once
+#include "g09.hpp"
+namespace arena {
+Json run_resume_scenario(const Json& scenario);
+}
diff --git a/tests/scenario_main.cpp b/tests/scenario_main.cpp
index 4e36e00..c588ccb 100644
--- a/tests/scenario_main.cpp
+++ b/tests/scenario_main.cpp
@@ -1,6 +1,7 @@
 #include "g07.hpp"
 #include "g09.hpp"
 #include "g10.hpp"
+#include "g11.hpp"
 #ifndef ARENA_TEST_FIXTURES
 #error Scenario fixture executable requires its separate test core
 #endif
@@ -35,7 +36,8 @@ int main(int argc, char** argv) {
       if (scenario.at("thread") != "G07" && scenario.at("thread") != "G09") {
         if (variant) throw std::invalid_argument("variant is only active for G07");
         const auto evidence = scenario.at("thread") == "G08" ? arena::run_snapshot_scenario(scenario) :
-          scenario.at("thread") == "G10" ? arena::run_ack_scenario(scenario) : arena::run_scenario(scenario);
+          scenario.at("thread") == "G10" ? arena::run_ack_scenario(scenario) :
+          scenario.at("thread") == "G11" ? arena::run_resume_scenario(scenario) : arena::run_scenario(scenario);
         arena::write_json_file(output,evidence);
         std::cout << arena::Json{{"result",evidence.at("result")},{"executed_ticks",evidence.at("executed_ticks")},
           {"scenario_id",evidence.at("scenario_id")},{"evidence",output.string()},{"cleanup",evidence.at("cleanup")}}.dump() << '\n';
