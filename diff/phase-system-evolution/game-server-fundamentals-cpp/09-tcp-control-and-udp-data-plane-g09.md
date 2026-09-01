# TCP Control과 UDP Data Plane (G09)

## `feat: add bounded UDP session data plane`

diff --git a/CMakeLists.txt b/CMakeLists.txt
index 39ea0d6..3788f31 100644
--- a/CMakeLists.txt
+++ b/CMakeLists.txt
@@ -31,10 +31,10 @@ endforeach()
 add_executable(arena src/main.cpp)
 target_link_libraries(arena PRIVATE arena_core)
 target_compile_options(arena PRIVATE -Wall -Wextra -Wpedantic -Werror)
-add_executable(arena_tests tests/tests.cpp)
-target_link_libraries(arena_tests PRIVATE arena_core)
+add_executable(arena_tests tests/tests.cpp tests/g09.cpp)
+target_link_libraries(arena_tests PRIVATE arena_test_core)
 target_compile_options(arena_tests PRIVATE -Wall -Wextra -Wpedantic -Werror)
-add_executable(arena_scenarios tests/scenario_main.cpp tests/g07.cpp)
+add_executable(arena_scenarios tests/scenario_main.cpp tests/g07.cpp tests/g09.cpp)
 target_link_libraries(arena_scenarios PRIVATE arena_test_core)
 target_compile_options(arena_scenarios PRIVATE -Wall -Wextra -Wpedantic -Werror)
 enable_testing()
diff --git a/evidence/G09-runs.jsonl b/evidence/G09-runs.jsonl
new file mode 100644
index 0000000..d698671
--- /dev/null
+++ b/evidence/G09-runs.jsonl
@@ -0,0 +1,9 @@
+{"label":"baseline-compile","category":"compile","units":1,"ticks":0,"ceiling_seconds":180,"argv":["clang++","-std=c++20","-O2","-Wall","-Wextra","-Wpedantic","-Werror","-fsanitize=thread","-g","-I","src","-I","/opt/homebrew/include","artifacts/g09/reproduce.cpp","src/game.cpp","src/transport.cpp","src/replay.cpp","src/snapshot.cpp","-o","artifacts/g09/reproduce"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g09/baseline-compile.log","started_at":"2026-08-28T04:58:31.177995+00:00","duration_seconds":14.865023,"exit":0,"wrapper_pid":83266,"child_pid":83275,"timed_out":false}
+{"label":"baseline","category":"unit","units":1,"ticks":1,"ceiling_seconds":120,"argv":["env","TSAN_OPTIONS=halt_on_error=1","./artifacts/g09/reproduce","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G09.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g09/baseline.json"],"expected_exit":1,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g09/baseline.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g09/baseline.json","started_at":"2026-08-28T05:00:50.713469+00:00","duration_seconds":0.989817,"exit":1,"wrapper_pid":84348,"child_pid":84357,"timed_out":false,"observed_ticks":1,"runtime_pid":84357}
+{"label":"build","category":"compile","units":2,"ticks":0,"ceiling_seconds":180,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g09-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g09/track","TSAN_OPTIONS=halt_on_error=1","ARENA_TSAN=ON","./track","build"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g09/build.log","started_at":"2026-08-28T05:26:22.384283+00:00","duration_seconds":7.455967,"exit":2,"wrapper_pid":96397,"child_pid":96406,"timed_out":false}
+{"label":"build-fix1","category":"compile","units":1,"ticks":0,"ceiling_seconds":180,"argv":["cmake","--build","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g09-tsan","--parallel","2"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g09/build-fix1.log","reason":"Fix compile-time std::string/JSON comparison; preserve original failed configure/build log.","started_at":"2026-08-28T05:27:22.075584+00:00","duration_seconds":29.877514,"exit":2,"wrapper_pid":97258,"child_pid":97267,"timed_out":false}
+{"label":"build-fix2","category":"compile","units":1,"ticks":0,"ceiling_seconds":180,"argv":["cmake","--build","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g09-tsan","--parallel","2"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g09/build-fix2.log","reason":"Fix test-only signed/unsigned comparison reported by -Werror; preserve prior failed builds.","started_at":"2026-08-28T05:28:16.918020+00:00","duration_seconds":5.730707,"exit":0,"wrapper_pid":97942,"child_pid":97951,"timed_out":false}
+{"label":"unit","category":"unit","units":1,"ticks":0,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g09-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g09/track","TSAN_OPTIONS=halt_on_error=1","./track","unit-test"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g09/unit.log","started_at":"2026-08-28T05:30:27.443474+00:00","duration_seconds":3.004742,"exit":0,"wrapper_pid":99233,"child_pid":99234,"timed_out":false}
+{"label":"integration","category":"integration","units":1,"ticks":0,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g09-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g09/track","TSAN_OPTIONS=halt_on_error=1","./track","integration-test"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g09/integration.log","started_at":"2026-08-28T05:30:30.518910+00:00","duration_seconds":1.726373,"exit":0,"wrapper_pid":99270,"child_pid":99272,"timed_out":false}
+{"label":"fault","category":"network_fault","units":1,"ticks":24,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g09-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g09/track","TSAN_OPTIONS=halt_on_error=1","./track","scenario-run","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G09.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g09/fault.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g09/fault.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g09/fault.json","started_at":"2026-08-28T05:30:32.325733+00:00","duration_seconds":1.143509,"exit":0,"wrapper_pid":99305,"child_pid":99306,"timed_out":false,"observed_ticks":24,"runtime_pid":99312}
+{"label":"offline","category":"offline","units":1,"ticks":24,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g09-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g09/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g09/fault.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g09/offline.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g09/offline.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g09/offline.json","started_at":"2026-08-28T05:30:33.555253+00:00","duration_seconds":0.468669,"exit":0,"wrapper_pid":99335,"child_pid":99336,"timed_out":false,"observed_ticks":24,"runtime_pid":99342}
diff --git a/evidence/G09.md b/evidence/G09.md
new file mode 100644
index 0000000..92a8862
--- /dev/null
+++ b/evidence/G09.md
@@ -0,0 +1,34 @@
+# G09 — bounded UDP data plane
+
+Initial attempt; phase-1 / realtime-core; branch `track/fundamentals-cpp`.
+SPEC_REVISION `c1d62196ab76b55652f5d75a67514f8c6d8210ce`; START `3cbfc0452ac8e53b576efd8ed24ab851bea42cbf`.
+Frozen main `index/scenarios/G09.json` SHA-256 `9519b687243a622f4e63675bb3690e8900bf7616337647f18de73e1bcc27edfa`.
+
+## Baseline and scope
+
+`artifacts/g09/pre-change-production.json` preserves14 actual Git START source hashes. `reproduce.cpp` linked unchanged G08 production code and exercised real HELLO/create/join/INPUT: WELCOME had no bind token; alpha joined LOBBY, bravo started RUNNING, later joins rejected; the first TCP INPUT was accepted and moved alpha to10400 after one tick. It stopped at that missing UDP/readiness prerequisite, not an unsupported command. Expected exit1; all10 descriptors closed. Raw `baseline.json` SHA-256 `cb9923446549332686f59c9040dc80fc86b108906faaf95808e5bfb7f05a47cc`. Main independently acknowledged the reproduction.
+
+Added only bounded UDP receive/send, one-time session credentials with5000ms expiry, observed-endpoint validation, TCP/UDP message mapping and min2/all-joined-ready start. Existing parser, owner mailbox, input ordering/rate and gameplay remain in use. Replay and snapshot implementation files are byte-identical to START. Earlier live harnesses now perform real UDP binding and transport mapping at their original physical boundaries; G02 still sends the same fragmented TCP INPUT and now expects WRONG_TRANSPORT after parsing. The G09 four-player runner uses ordinary joins and binds, with identifier allocation injected only in the separate test build. `nm` and compile flags confirmed no G09 fixture/proxy symbols in shipping `arena`; both cores use TSan. No dependencies or later-stage policies were added.
+
+## Actual verification
+
+`evidence/G09-runs.jsonl` is the exact command ledger: argv, category/units, raw paths, duration, exit and child/runtime PIDs. `artifacts/g09/commands.json` resolved each command before execution. The original matrix plan is preserved in `commands.before-matrix-packaging.json`; its standalone command is canceled and unused. The fixed11-case matrix ran once inside full integration alongside the previous3 tests. `verify.py` ran unit, integration, fault and offline sequentially with ceilings and stop on failure.
+
+| Pass | Actual result | Seconds / exit |
+| --- | --- | --- |
+| Baseline compile / reproduction | unchanged implementation,1 tick |14.865023 /0;0.989817 /1 expected |
+| Configure/build | failed string/JSON comparison |7.455967 /2 |
+| Incremental build1 | failed test signed/unsigned comparison |29.877514 /2 |
+| Incremental build2 | PASS |5.730707 /0 |
+| Full unit / integration |24/24;4/4 PASS |3.004742 /0;1.726373 /0 |
+| Fault canonical / offline |24 ticks each, PIDs99312/99342, PASS |1.143509 /0;0.468669 /0 |
+
+Matrix results:4999ms valid;5000ms expired; unknown/reused/other-session/epoch1/unbound/foreign-endpoint rejected without authority mutation or credential theft; valid INPUT overTCP WRONG_TRANSPORT;1200-byte INPUT ACCEPTED;1201-byte datagram processed and dropped with metric/no ACK. Actual max8 FULL with roomID17/playerID8 is1189 bytes. Oversized fixture injection and actual outbound send reject explicitly. Matrix executes zero Room ticks.
+
+Fault capture `artifacts/g09/fault.json` retains all24 admissions,13 original alpha snapshots, actual proxy delivery trace, client applications/ACKs and resource observations. Unique accepted sequences are1–24 except5/10; duplicate8 has no second effect; held10 rejects INPUT_STALE after11. Snapshot receipts `[1,2,3,4,6,7,8,8,9,11,10,12,13]`; duplicate8 and old10 send no ACK. Owner ACK stays4 after lost5 and9 while10 is held; FULL1 is followed by DELTAs against actual retained bases. Final alpha `[19600,10000]`,EAST,score0,lastseq24; others STOP at original spawns with score/lastseq0. All clients apply final sequence13.
+
+The actual accepted journal `fault.replay.json` is6192 bytes, SHA-256 `3345ec91e48b5c42672a909b2935eeb5c67f95a6277ff7c9df387bd453a2aa9b`. `fault.records.json` and `offline.records.json` are byte-identical (49956 bytes each; SHA-256 `0aee3a707544ec8faad953320fc4d15bf0b400f6a4872de7781583e77e0ca5a9`). All24 hash pairs match; hash-list digest, one hash plus LF, is `a20d7dac08fc5a7f702f8051c03bfeb84a0d88622c19504c23151928d2c50b98`. No all-input substitute or additional fault campaign was run.
+
+Actual fault-run high waters: connections4/512, mailbox4/512, control1/64, pending1/64, attempts2/4, parser106/16388, replay6192/4194304, snapshots13/32, inboundUDP211/1200, outboundUDP711/1200. Proxy holds at most1 packet in each direction and ends with0. UDP receive/send counts78/79. All16 descriptors closed; every active cleanup counter and FD delta0. No TSan reports or credential values in raw observations/artifacts.
+
+Budget consumed: compile/configure5/8 (two preserved failures), unit2/4 including baseline, integration1/2, fault canonical1/1 and offline1/1. G09 simulations49 ticks total including baseline; fixed matrix0; load0. All raw logs remain under `artifacts/g09/`, with child wrapper logs in `artifacts/g09/track/`. Root independent/cross gate pending; no progress tag, push, main/spec/index/threads edits or G10+ work.
diff --git a/src/game.cpp b/src/game.cpp
index f41c15f..4f853af 100644
--- a/src/game.cpp
+++ b/src/game.cpp
@@ -79,8 +79,19 @@ Player& Room::join(std::string player_id, std::string session_id, std::uint64_t
   return found->second;
 }
 void Room::evaluate_start_condition() {
-  const auto ready = std::count_if(players_.begin(), players_.end(), [](const auto& pair) { return pair.second.connected; });
-  if (ready >= 2) transition("RUNNING");
+  const auto joined = std::count_if(players_.begin(),players_.end(),[](const auto& pair) { return pair.second.connected; });
+  const bool ready = std::all_of(players_.begin(),players_.end(),[](const auto& pair) {
+    return !pair.second.connected || pair.second.realtime_ready;
+  });
+  if (status_ == "LOBBY" && joined >= 2 && ready) transition("RUNNING");
+}
+void Room::bind_realtime(std::uint64_t connection_id) {
+  assert_owner();
+  for (auto& [id,player] : players_) {
+    (void)id;
+    if (player.connection_id == connection_id && player.connected) player.realtime_ready = true;
+  }
+  evaluate_start_condition();
 }
 std::optional<std::string> Room::begin_input_attempt(const std::string& player_id) {
   assert_owner();
@@ -185,8 +196,10 @@ void Room::leave(std::uint64_t connection_id) {
       player.pending.clear();
       player.applied_seq.reset();
       player.input_attempts = 0;
+      player.realtime_ready = false;
     }
   }
+  evaluate_start_condition();
 }
 void Room::close() {
   assert_owner();
@@ -198,6 +211,7 @@ void Room::close() {
     player.pending.clear();
     player.applied_seq.reset();
     player.input_attempts = 0;
+    player.realtime_ready = false;
   }
   transition("CLOSED");
 }
diff --git a/src/game.hpp b/src/game.hpp
index 21f261f..a6e27fb 100644
--- a/src/game.hpp
+++ b/src/game.hpp
@@ -12,6 +12,8 @@
 namespace arena {
 using Json = nlohmann::json;
 inline constexpr std::size_t max_frame_bytes = 16'384;
+inline constexpr std::size_t max_datagram_bytes = 1'200;
+inline constexpr std::int64_t udp_bind_token_ttl_ms = 5'000;
 inline constexpr std::size_t max_connections = 512;
 inline constexpr std::size_t max_players = 8;
 inline constexpr std::size_t max_pending_inputs = 64;
@@ -61,6 +63,7 @@ struct Player {
   std::optional<Input> last_accepted_input;
   std::optional<std::uint64_t> applied_seq;
   std::size_t input_attempts = 0;
+  bool realtime_ready = false;
   std::uint64_t last_accepted_seq() const { return last_accepted_input ? last_accepted_input->seq : 0; }
 };
 struct ActionFailure {
@@ -74,6 +77,7 @@ class Room {
   Room();
   void create(std::string id);
   Player& join(std::string player_id, std::string session_id, std::uint64_t connection_id);
+  void bind_realtime(std::uint64_t connection_id);
   std::optional<std::string> begin_input_attempt(const std::string& player_id);
   InputResult input(const std::string& player_id, Input input);
   std::vector<ActionFailure> tick();
@@ -92,6 +96,7 @@ class Room {
   friend void initialize_shared_victim_fixture(Room& room);
 #ifdef ARENA_TEST_FIXTURES
   friend struct ReplayFixture;
+  friend struct UdpFixture;
 #endif
   void assert_owner() const;
   void evaluate_start_condition();
diff --git a/src/lifecycle_scenario.cpp b/src/lifecycle_scenario.cpp
index c2f4dc2..1831cdb 100644
--- a/src/lifecycle_scenario.cpp
+++ b/src/lifecycle_scenario.cpp
@@ -19,6 +19,7 @@ bool valid_issued_id(const std::string& value) {
 }
 struct Peer {
   std::unique_ptr<TcpClient> tcp;
+  std::unique_ptr<UdpClient> udp;
   std::string session;
   std::string player;
 };
@@ -61,6 +62,7 @@ struct LifecycleFixture {
     peer.tcp = std::make_unique<TcpClient>(server.port());
     peer.tcp->send(message("HELLO"));
     peer.session = peer.tcp->receive_type(server, "WELCOME").at("session_id").get<std::string>();
+    peer.udp = std::make_unique<UdpClient>(peer.tcp->udp_port()); peer.udp->bind(*peer.tcp,server,peer.session);
     ensure(valid_issued_id(peer.session), "issued session identifier violates ASCII/length contract");
   }
   void create() {
@@ -108,7 +110,7 @@ struct LifecycleFixture {
     const auto before = server.room().view();
     const auto pending = server.room().pending_count();
     auto& peer = peers.at(role);
-    peer.tcp->send(value);
+    if (value.at("type") == "INPUT") peer.udp->send(value); else peer.tcp->send(value);
     const auto response = peer.tcp->receive_type(server, "ERROR");
     ensure(response.at("code") == code, "identity/lifecycle rejection code");
     ensure(server.room().view() == before && server.room().pending_count() == pending,
@@ -121,13 +123,15 @@ struct LifecycleFixture {
     peer.tcp->close();
     ensure(descriptor_closed(fd), "closed client descriptor remains live");
     ++closed_client_checks;
+    const auto udp_fd = peer.udp->descriptor(); peer.udp->close();
+    ensure(descriptor_closed(udp_fd),"closed UDP client descriptor remains live"); ++closed_client_checks;
   }
   void drain_snapshots() {
     server.pump();
     wait_for([&] { return server.cleanup().at("outbound_messages") == 0; });
     for (auto& [role, peer] : peers) {
       (void)role;
-      while (auto value = peer.tcp->try_receive())
+      while (auto value = peer.udp->try_receive())
         ensure(value->at("type") == "SNAPSHOT" && value->contains("snapshot_seq"), "unexpected frame during snapshot drain");
     }
   }
@@ -136,9 +140,10 @@ struct LifecycleFixture {
     for (auto& [role, peer] : peers) {
       (void)role;
       if (peer.tcp->descriptor() >= 0) descriptors.push_back(peer.tcp->descriptor());
+      if (peer.udp->descriptor() >= 0) descriptors.push_back(peer.udp->descriptor());
     }
     server.shutdown();
-    for (auto& [role, peer] : peers) { (void)role; peer.tcp->close(); }
+    for (auto& [role, peer] : peers) { (void)role; peer.tcp->close(); peer.udp->close(); }
     for (int fd : descriptors) ensure(descriptor_closed(fd), "server/client descriptor survived cleanup");
     ensure(Fd::live() == descriptors_before, "RAII tracked descriptor leak");
     ensure(server.room().status() == "CLOSED" && server.room().players().size() <= max_players,
@@ -168,7 +173,7 @@ Json lifecycle_cell(const Json& scenario, const std::string& state, const std::s
       fixture.server.advance_one_tick();
       if ((tick + 1) % 2 == 0) for (auto& [role, peer] : fixture.peers) {
         (void)role;
-        const auto snapshot = peer.tcp->receive_type(fixture.server,"SNAPSHOT");
+        const auto snapshot = peer.udp->receive_type(fixture.server,"SNAPSHOT");
         ensure(snapshot.at("tick") == tick, "FINISHED setup preserves each real tick while draining snapshots");
       }
     }
@@ -263,7 +268,7 @@ Json run_lifecycle_scenario(const Json& scenario) {
          scenario.at("lifecycle_states") == Json::array({"LOBBY", "RUNNING", "FINISHED"}) &&
          scenario.at("lifecycle_actions") == Json::array({"LEAVE_ROOM", "CONNECTION_CLOSE"}), "fixed matrix changed");
   Json evidence{{"scenario_id", scenario.at("scenario_id")}, {"thread", "G03"}, {"contract_version", 1},
-    {"transport", "production/real-loopback-TCP/kqueue/manual-owner-drain"},
+    {"transport", "production/real-loopback-TCP-control/UDP-data/kqueue/manual-owner-drain"},
     {"identity_cases", Json::array()}, {"lifecycle_cases", Json::array()},
     {"state_hashes", "INACTIVE_UNTIL_G07"}};
   for (const auto& item : scenario.at("identity_cases")) {
@@ -309,7 +314,7 @@ Json run_lifecycle_scenario(const Json& scenario) {
     const auto before = fixture.server.room().view();
     auto input = fixture.request("alpha", "INPUT");
     input["direction"] = probe.at("direction"); input["tag_target_player_id"] = probe.at("tag_target_player_id");
-    alpha.tcp->send(input);
+    alpha.udp->send(input);
     fixture.wait_for([&] { return fixture.server.cleanup().at("mailbox_messages") == 1; }, false);
     ensure(fixture.server.room().view() == before && fixture.server.room().pending_count() == 0 && fixture.clock.now_ms == 0,
            "network callback mutated Room before owner release");
@@ -320,7 +325,7 @@ Json run_lifecycle_scenario(const Json& scenario) {
            fixture.server.cleanup().at("mailbox_messages") == 0, "owner drain must enqueue intent without simulating");
     owner["after_drain"] = fixture.server.room().view(); owner["pending_after_drain"] = fixture.server.room().pending_count();
     owner["mailbox_after_drain"] = fixture.server.cleanup().at("mailbox_messages");
-    const auto ack = alpha.tcp->receive_type(fixture.server, "INPUT_ACK");
+    const auto ack = alpha.udp->receive_type(fixture.server, "INPUT_ACK");
     ensure(ack.at("accepted") == true && ack.at("player_id") == alpha.player && ack.at("tick") == 0,
            "owner accepted input acknowledgement");
     fixture.server.advance_one_tick();
@@ -381,7 +386,7 @@ Json run_input_scenario(const Json& scenario) {
   Json logical{{"accepted_sequences",Json::array()}, {"duplicate_sequences",Json::array()}, {"rejections",Json::array()},
     {"applied_sequences",Json::array()}, {"alpha_positions",Json::array()}};
   Json evidence{{"scenario_id",scenario.at("scenario_id")}, {"thread","G05"}, {"contract_version",1},
-    {"transport","production/real-loopback-TCP/kqueue/manual-owner-drain"}, {"identities",fixture.identities()},
+    {"transport","production/real-loopback-TCP-control/UDP-data/kqueue/manual-owner-drain"}, {"identities",fixture.identities()},
     {"events",Json::array()}, {"ticks",Json::array()}, {"state_hashes","INACTIVE_UNTIL_G07"}};
   Json directions = Json::array();
   std::size_t consumed = 0;
@@ -396,10 +401,8 @@ Json run_input_scenario(const Json& scenario) {
       const auto pending_before = player.pending;
       const auto last_before = player.last_accepted_input;
       const auto last_seq_before = player.last_accepted_seq();
-      alpha.tcp->send(request);
-      Json response;
-      do { response = alpha.tcp->receive(fixture.server); }
-      while (response.at("type") != "INPUT_ACK" && response.at("type") != "ERROR");
+      alpha.udp->send(request);
+      const auto response = receive_input_result(*alpha.tcp,*alpha.udp,fixture.server);
       check_input(fixture.server.room().view() == before && fixture.clock.now_ms == tick * tick_duration_ms,
                   "admission must not simulate movement or change another player");
       if (response.at("type") == "ERROR") {
@@ -483,7 +486,7 @@ Json run_authority_scenario(const Json& scenario) {
   };
   Json logical{{"admissions",Json::array()},{"tag_events",Json::array()}};
   Json evidence{{"thread","G06"},{"scenario_id",scenario.at("scenario_id")},{"contract_version",1},
-    {"transport","production/real-loopback-TCP/kqueue/manual-owner-drain"},{"identities",fixture.identities()},
+    {"transport","production/real-loopback-TCP-control/UDP-data/kqueue/manual-owner-drain"},{"identities",fixture.identities()},
     {"events",Json::array()},{"ticks",Json::array()},{"tag_observations",Json::array()},{"state_hashes","INACTIVE_UNTIL_G07"}};
   std::size_t index = 0, accepted = 0;
   for (int tick = 0; tick < scenario.at("ticks").get<int>(); ++tick) {
@@ -505,10 +508,8 @@ Json run_authority_scenario(const Json& scenario) {
       const auto last_before = player.last_accepted_input;
       const auto last_seq_before = player.last_accepted_seq();
       const auto attempts_before = player.input_attempts;
-      peer.tcp->send(request);
-      Json response;
-      do { response = peer.tcp->receive(fixture.server); }
-      while (response.at("type") != "INPUT_ACK" && response.at("type") != "ERROR");
+      peer.udp->send(request);
+      const auto response = receive_input_result(*peer.tcp,*peer.udp,fixture.server);
       const auto code = response.at("code").get<std::string>();
       const std::string expected = index == 2 ? "MESSAGE_INVALID" : index == 9 ? "INPUT_RATE_EXCEEDED" : "ACCEPTED";
       check_authority(code == expected && fixture.server.room().view() == before,
@@ -638,8 +639,8 @@ Json run_clock_scenario(const Json& scenario) {
       auto input = fixture.request(role, "INPUT");
       input["direction"] = scenario.at("directions").at(role); input["tag_target_player_id"] = nullptr;
       auto& peer = fixture.peers.at(role);
-      peer.tcp->send(input);
-      const auto ack = peer.tcp->receive_type(fixture.server, "INPUT_ACK");
+      peer.udp->send(input);
+      const auto ack = peer.udp->receive_type(fixture.server, "INPUT_ACK");
       check_clock(ack.at("accepted") == true && ack.at("tick") == 0, "fixed intent did not reach tick zero");
     }
     Json logical{{"tick_counts", Json::array()}, {"cumulative_ticks", Json::array()}, {"remaining_ms", Json::array()},
diff --git a/src/scenario.cpp b/src/scenario.cpp
index 7e58e0d..9dbb85b 100644
--- a/src/scenario.cpp
+++ b/src/scenario.cpp
@@ -12,10 +12,12 @@ namespace {
 void require(bool value, const std::string& text) { if (!value) throw std::runtime_error(text); }
 struct Participant {
   std::unique_ptr<TcpClient> tcp;
+  std::unique_ptr<UdpClient> udp;
   std::string session_id;
   std::string player_id;
   int slot = -1;
   std::uint64_t next_seq = 1;
+  std::vector<std::string> lifecycle;
 };
 Json for_player(const std::string& type, const Participant& participant, const std::string& room_id) {
   Json value = message(type);
@@ -24,14 +26,10 @@ Json for_player(const std::string& type, const Participant& participant, const s
   if (type == "INPUT") value["player_id"] = participant.player_id;
   return value;
 }
-std::vector<std::string> observed_lifecycle(const TcpClient& client) {
-  std::vector<std::string> result;
-  for (const auto& observation : client.observations()) {
-    if (!observation.contains("status")) continue;
-    auto status = observation.at("status").get<std::string>();
-    if (result.empty() || result.back() != status) result.push_back(std::move(status));
-  }
-  return result;
+void observe_lifecycle(Participant& peer, const Json& value) {
+  if (!value.contains("status")) return;
+  auto status = value.at("status").get<std::string>();
+  if (peer.lifecycle.empty() || peer.lifecycle.back() != status) peer.lifecycle.push_back(std::move(status));
 }
 }
 Json read_json_file(const std::filesystem::path& path, std::size_t limit) {
@@ -186,8 +184,8 @@ Json run_framing_scenario(const Json& scenario) {
       if (value.at("type") == "HELLO")
         require(response.at("type") == "WELCOME" && response.at("session_id") == healthy_session, "fragmented HELLO failed");
       else
-        require(response.at("type") == "ERROR" && response.at("code") == "SESSION_INVALID",
-                "fixture identity should reach existing session validation after parsing");
+        require(response.at("type") == "ERROR" && response.at("code") == (value.at("type") == "INPUT" ? "WRONG_TRANSPORT" : "SESSION_INVALID"),
+                "fixed TCP bytes reach the active transport/session check after parsing");
       require(std::chrono::steady_clock::now() < deadline, "fragmentation socket deadline exceeded");
       row["response"] = response;
       row["parser_buffered_after"] = server.cleanup().at("parser_buffered_bytes");
@@ -301,12 +299,12 @@ Json run_scenario(const Json& scenario) {
   std::map<std::string, Participant> clients;
   std::string room_id;
   Json evidence{{"scenario_id", scenario.at("scenario_id")}, {"thread", "G01"}, {"contract_version", 1},
-                {"seed", scenario.at("seed")}, {"transport", "real-loopback-TCP/kqueue"}, {"clock", scenario.at("clock")},
+                {"seed", scenario.at("seed")}, {"transport", "real-loopback-TCP-control/UDP-data/kqueue"}, {"clock", scenario.at("clock")},
                 {"accepted_inputs", Json::array()}, {"setup_responses", Json::array()}};
   for (const auto& role_value : scenario.at("clients")) {
     const auto role = role_value.get<std::string>();
     require(role.size() <= 64 && !clients.contains(role), "client role must be unique and bounded");
-    clients.emplace(role, Participant{std::make_unique<TcpClient>(server.port()), {}, {}, -1, 1});
+    clients.emplace(role, Participant{std::make_unique<TcpClient>(server.port()), {}, {}, {}, -1, 1, {}});
   }
   for (const auto& step : scenario.at("setup")) {
     const auto role = step.at("client").get<std::string>();
@@ -316,8 +314,13 @@ Json run_scenario(const Json& scenario) {
     participant.tcp->send(for_player(type, participant, room_id));
     const std::string response_type = type == "HELLO" ? "WELCOME" : type == "CREATE_ROOM" ? "ROOM_CREATED" : "ROOM_JOINED";
     const auto response = participant.tcp->receive_type(server, response_type);
+    observe_lifecycle(participant,response);
     evidence["setup_responses"].push_back(Json{{"client", role}, {"response", response}});
-    if (type == "HELLO") participant.session_id = response.at("session_id").get<std::string>();
+    if (type == "HELLO") {
+      participant.session_id = response.at("session_id").get<std::string>();
+      participant.udp = std::make_unique<UdpClient>(participant.tcp->udp_port());
+      participant.udp->bind(*participant.tcp,server,participant.session_id);
+    }
     if (type == "CREATE_ROOM") room_id = response.at("room_id").get<std::string>();
     if (type == "JOIN_ROOM") {
       participant.player_id = response.at("player_id").get<std::string>();
@@ -327,7 +330,7 @@ Json run_scenario(const Json& scenario) {
   require(server.room().status() == "RUNNING", "setup did not start the real server room");
   for (auto& [role, participant] : clients) {
     (void)role;
-    const auto start = participant.tcp->receive_type(server,"SNAPSHOT");
+    const auto start = participant.udp->receive_type(server,"SNAPSHOT"); observe_lifecycle(participant,start);
     require(start.at("snapshot_seq") == 1 && start.at("tick") == -1 && start.at("kind") == "FULL" &&
       start.at("status") == "RUNNING", "real Room-start full snapshot");
   }
@@ -357,19 +360,19 @@ Json run_scenario(const Json& scenario) {
       // Optional untrusted fields enable an authority regression; the real
       // server ignores them. The canonical file has neither of these fields.
       for (const auto* field : {"position", "score"}) if (input.contains(field)) request[field] = input.at(field);
-      participant.tcp->send(request);
-      const auto acknowledgement = participant.tcp->receive_type(server, "INPUT_ACK");
+      participant.udp->send(request);
+      const auto acknowledgement = participant.udp->receive_type(server, "INPUT_ACK");
       require(acknowledgement.at("accepted") == true && acknowledgement.at("tick") == tick,
               "input acknowledgement did not establish the intended tick boundary");
       evidence["accepted_inputs"].push_back(Json{{"client", role}, {"before_tick", tick},
         {"direction", input.at("direction")}, {"tag_target", input.at("tag_target")}, {"ack", acknowledgement}});
     }
-    // Every intended input has crossed TCP and received an owner-phase ACK.
+    // Every intended input has crossed UDP and received an owner-phase ACK.
     server.advance_one_tick();
     require(server.room().executed_ticks() == tick + 1, "manual tick did not execute exactly once");
     if ((tick + 1) % 2 == 0) for (auto& [role, participant] : clients) {
       (void)role;
-      const auto snapshot = participant.tcp->receive_type(server,"SNAPSHOT");
+      const auto snapshot = participant.udp->receive_type(server,"SNAPSHOT"); observe_lifecycle(participant,snapshot);
       require(snapshot.at("tick") == tick, "periodic traffic drained at its actual tick boundary");
     }
   }
@@ -388,6 +391,7 @@ Json run_scenario(const Json& scenario) {
     require(evidence["players"].contains(role), "server final is missing a joined role");
     if (server.room().status() == "FINISHED") {
       auto client_final = participant.tcp->receive_type(server, "ROOM_FINISHED");
+      observe_lifecycle(participant,client_final);
       evidence["client_finished"][role] = client_final;
       client_final.erase("v"); client_final.erase("type");
       require(client_final == final, "TCP client final differs from authoritative room view");
@@ -398,20 +402,23 @@ Json run_scenario(const Json& scenario) {
   server.shutdown();
   evidence["client_lifecycle"] = Json::object();
   evidence["client_observations"] = Json::object();
+  evidence["client_udp_observations"] = Json::object();
   evidence["client_eof"] = Json::object();
   require(server.room().status() == "CLOSED", "owner did not close the Room");
   for (auto& [role, participant] : clients) {
     const auto deadline = std::chrono::steady_clock::now() + std::chrono::seconds(2);
     while (!participant.tcp->peer_closed() && std::chrono::steady_clock::now() < deadline) {
-      if (auto value = participant.tcp->try_receive())
-        require(value->at("type") == "SNAPSHOT" && value->contains("snapshot_seq"), "unexpected shutdown frame");
+      require(!participant.tcp->try_receive(), "unexpected shutdown control frame");
     }
     require(participant.tcp->peer_closed(), "client did not observe owner shutdown EOF");
     evidence["client_eof"][role] = true;
     closed_fds.push_back(participant.tcp->descriptor());
+    closed_fds.push_back(participant.udp->descriptor());
     participant.tcp->close();
-    evidence["client_lifecycle"][role] = observed_lifecycle(*participant.tcp);
+    participant.udp->close();
+    evidence["client_lifecycle"][role] = participant.lifecycle;
     evidence["client_observations"][role] = participant.tcp->observations();
+    evidence["client_udp_observations"][role] = participant.udp->observations();
   }
   bool all_closed = true;
   for (const int fd : closed_fds) all_closed = all_closed && descriptor_closed(fd);
diff --git a/src/transport.cpp b/src/transport.cpp
index 102236f..4433974 100644
--- a/src/transport.cpp
+++ b/src/transport.cpp
@@ -38,6 +38,19 @@ std::uint32_t payload_size(const std::uint8_t* data) {
          (static_cast<std::uint32_t>(data[2]) << 8U) | static_cast<std::uint32_t>(data[3]);
 }
 bool transient_io() { return errno == EAGAIN || errno == EWOULDBLOCK || errno == EINTR; }
+bool realtime_type(const std::string& type) {
+  return type == "UDP_BIND" || type == "UDP_BOUND" || type == "INPUT" || type == "INPUT_ACK" ||
+    type == "SNAPSHOT" || type == "SNAPSHOT_ACK" || type == "PING" || type == "PONG";
+}
+bool same_endpoint(const sockaddr_in& a, const sockaddr_in& b) {
+  return a.sin_family == b.sin_family && a.sin_addr.s_addr == b.sin_addr.s_addr && a.sin_port == b.sin_port;
+}
+std::string new_bind_token() {
+  std::random_device random; std::ostringstream out;
+  out << std::hex << std::setfill('0');
+  for (int word = 0; word < 4; ++word) out << std::setw(8) << random();
+  return out.str();
+}
 Json parse_object(std::span<const std::uint8_t> payload) {
   // nlohmann 3.12's lexer treats raw NUL as end-of-input. A framed payload
   // cannot have an early terminator that hides unchecked trailing bytes.
@@ -63,7 +76,8 @@ std::string request_error(const Json& value) {
   if (value.at("v") != 1) return "PROTOCOL_VERSION_UNSUPPORTED";
   const auto type = value.at("type").get<std::string>();
   if (type == "HELLO") return {};
-  if (type != "CREATE_ROOM" && type != "JOIN_ROOM" && type != "LEAVE_ROOM" && type != "INPUT" && type != "SNAPSHOT_ACK")
+  if (realtime_type(type) && type != "INPUT" && type != "SNAPSHOT_ACK") return {};
+  if (type != "CREATE_ROOM" && type != "JOIN_ROOM" && type != "LEAVE_ROOM" && !realtime_type(type))
     return "MESSAGE_TYPE_UNKNOWN";
   const auto string_field = [&](const char* name) { return value.contains(name) && value.at(name).is_string(); };
   if (!string_field("session_id")) return "MESSAGE_INVALID";
@@ -137,6 +151,16 @@ Json decode_complete_frame(std::span<const std::uint8_t> bytes) {
     throw std::length_error("exactly one bounded complete frame required");
   return parse_object(bytes.subspan(4));
 }
+std::vector<std::uint8_t> encode_datagram(const Json& value) {
+  const auto text = value.dump();
+  if (!value.is_object() || text.empty() || text.size() > max_datagram_bytes)
+    throw std::length_error("UDP object exceeds1200 bytes");
+  return {text.begin(),text.end()};
+}
+Json decode_datagram(std::span<const std::uint8_t> bytes) {
+  if (bytes.empty() || bytes.size() > max_datagram_bytes) throw std::length_error("bounded UDP object required");
+  return parse_object(bytes);
+}
 std::string parse_state_name(ParseState state) {
   switch (state) {
     case ParseState::need_more: return "need_more";
@@ -195,9 +219,10 @@ void PendingWrite::consume(std::size_t count) {
   offset += count;
 }
 Server::Server(ManualClock& clock, std::uint16_t port, MonotonicNow monotonic_now)
-    : clock_(clock), monotonic_now_(std::move(monotonic_now)), reactor_(::kqueue()), listener_(::socket(AF_INET, SOCK_STREAM, 0)) {
+    : clock_(clock), monotonic_now_(std::move(monotonic_now)), reactor_(::kqueue()), listener_(::socket(AF_INET, SOCK_STREAM, 0)),
+      datagram_(::socket(AF_INET,SOCK_DGRAM,0)) {
   if (!monotonic_now_) throw std::invalid_argument("monotonic clock source is required");
-  if (reactor_.get() == -1 || listener_.get() == -1) system_failure("create kqueue/listener");
+  if (reactor_.get() == -1 || listener_.get() == -1 || datagram_.get() == -1) system_failure("create kqueue/listener/UDP");
   std::random_device random;
   std::ostringstream nonce;
   nonce << std::hex << std::setfill('0') << std::setw(8) << random() << std::setw(8) << random();
@@ -215,7 +240,13 @@ Server::Server(ManualClock& clock, std::uint16_t port, MonotonicNow monotonic_no
   socklen_t size = sizeof(address);
   if (::getsockname(listener_.get(), reinterpret_cast<sockaddr*>(&address), &size) == -1) system_failure("getsockname");
   port_ = ntohs(address.sin_port);
+  nonblocking(datagram_.get()); address.sin_port = 0;
+  if (::bind(datagram_.get(),reinterpret_cast<sockaddr*>(&address),sizeof(address)) == -1) system_failure("bind UDP loopback");
+  size = sizeof(address);
+  if (::getsockname(datagram_.get(),reinterpret_cast<sockaddr*>(&address),&size) == -1) system_failure("UDP port");
+  udp_port_ = ntohs(address.sin_port);
   register_event(listener_.get(), EVFILT_READ, EV_ADD);
+  register_event(datagram_.get(),EVFILT_READ,EV_ADD);
 }
 Server::~Server() {
   // Destructors cannot report errors; explicit shutdown is required by callers
@@ -233,6 +264,14 @@ Server::Connection* Server::connection(std::uint64_t id) {
   return nullptr;
 }
 std::string Server::new_id(const std::string& prefix, std::uint64_t number) const {
+#ifdef ARENA_TEST_FIXTURES
+  if (prefix == "room" && fixture_room_id_) return *fixture_room_id_;
+  if (prefix == "player" && !fixture_player_ids_.empty()) return fixture_player_ids_.at(static_cast<std::size_t>(number-1));
+#endif
+  // Room identity retains the process nonce. Player identity is room-scoped;
+  // at most8 joins need a one-digit suffix. R17/P8 keeps max8 FULL <=1200.
+  if (prefix == "room") return "r"+nonce_;
+  if (prefix == "player") return "p"+nonce_.substr(0,6)+std::to_string(number);
   std::ostringstream out;
   out << prefix << '-' << nonce_ << '-' << std::setw(10) << std::setfill('0') << number;
   return out.str();
@@ -347,6 +386,7 @@ void Server::poll_io(int timeout_ms) {
     const auto& event = events[static_cast<std::size_t>(i)];
     const int fd = static_cast<int>(event.ident);
     if (fd == listener_.get()) { if (!stopping_) accept_ready(); continue; }
+    if (fd == datagram_.get()) { if (!stopping_) read_datagrams(); continue; }
     const auto found = connections_.find(fd);
     if (found == connections_.end() || found->second.id != reinterpret_cast<uintptr_t>(event.udata)) continue;
     if (event.flags & EV_ERROR) { end_transport(fd, true); continue; }
@@ -374,6 +414,68 @@ void Server::broadcast(const Json& value) {
   for (const auto& [fd, conn] : connections_) { (void)fd; if (!conn.player_id.empty()) ids.push_back(conn.id); }
   for (auto id : ids) queue(id, value);
 }
+void Server::read_datagrams() {
+  for (int work = 0; work < 64; ++work) {
+    std::array<std::uint8_t,max_datagram_bytes+1> bytes{}; sockaddr_in endpoint{};
+    iovec buffer{bytes.data(),bytes.size()}; msghdr header{};
+    header.msg_name = &endpoint; header.msg_namelen = sizeof(endpoint); header.msg_iov = &buffer; header.msg_iovlen = 1;
+    const auto count = ::recvmsg(datagram_.get(),&header,0);
+    if (count < 0) { if (!transient_io()) ++errors_["UDP_IO_ERROR"]; return; }
+    ++received_datagrams_;
+    if (count == 0 || count > static_cast<ssize_t>(max_datagram_bytes) || (header.msg_flags & MSG_TRUNC)) {
+      ++errors_["UDP_DATAGRAM_SIZE_INVALID"]; continue;
+    }
+    datagram_high_water_ = std::max(datagram_high_water_,static_cast<std::size_t>(count));
+    try {
+      auto value = decode_datagram(std::span(bytes).first(static_cast<std::size_t>(count)));
+      Connection* conn = nullptr;
+      const bool binding = value.contains("type") && value.at("type") == "UDP_BIND";
+      if (!binding) for (auto& [fd, candidate] : connections_) {
+        (void)fd;
+        if (candidate.udp_endpoint && same_endpoint(*candidate.udp_endpoint,endpoint)) { conn = &candidate; break; }
+      }
+      if (!conn && value.contains("session_id") && value.at("session_id").is_string()) {
+        for (auto& [fd, candidate] : connections_) {
+          (void)fd;
+          if (!candidate.session_id.empty() && value.at("session_id") == candidate.session_id) { conn = &candidate; break; }
+        }
+      }
+      if (!conn) { ++errors_["UDP_UNAUTHENTICATED_DROP"]; continue; }
+      if (const auto error = mailbox_.admit(Envelope{conn->id,std::move(value),{},endpoint},conn->pending_requests)) {
+        ++errors_[*error]; queue(conn->id,error_message(*error,"bounded UDP owner admission"));
+      }
+      mailbox_high_water_ = std::max(mailbox_high_water_,mailbox_.size());
+    } catch (const std::exception&) { ++errors_["UDP_MESSAGE_INVALID"]; }
+  }
+}
+void Server::send_realtime(std::uint64_t connection_id, const Json& value) {
+  const auto* conn = connection(connection_id);
+  if (!conn || !conn->udp_endpoint) { ++errors_["UDP_UNBOUND_SEND"]; return; }
+  try {
+    const auto bytes = encode_datagram(value);
+    outbound_datagram_high_water_ = std::max(outbound_datagram_high_water_,bytes.size());
+    const auto count = ::sendto(datagram_.get(),bytes.data(),bytes.size(),0,
+      reinterpret_cast<const sockaddr*>(&*conn->udp_endpoint),sizeof(sockaddr_in));
+    if (count == static_cast<ssize_t>(bytes.size())) ++sent_datagrams_;
+    else ++errors_["UDP_SEND_DROP"];
+  } catch (const std::length_error&) { ++errors_["UDP_OUTBOUND_SIZE_INVALID"]; }
+}
+void Server::bind_datagram(Connection& conn, const Envelope& envelope) {
+  const auto& value = envelope.value;
+  const bool valid = envelope.udp_endpoint && !conn.udp_endpoint && !conn.bind_token.empty() &&
+    value.contains("session_id") && value.at("session_id").is_string() && value.at("session_id") == conn.session_id &&
+    value.contains("udp_bind_token") && value.at("udp_bind_token").is_string() && value.at("udp_bind_token") == conn.bind_token &&
+    value.contains("owner_epoch") && value.at("owner_epoch").is_number_integer() && value.at("owner_epoch") == 0 &&
+    monotonic_now_()-conn.token_issued_ms < udp_bind_token_ttl_ms;
+  if (!valid) {
+    ++errors_["UDP_BIND_INVALID"]; queue(conn.id,error_message("UDP_BIND_INVALID","binding credential, lifetime or endpoint invalid")); return;
+  }
+  conn.udp_endpoint = envelope.udp_endpoint; conn.bind_token.clear();
+  auto reply = message("UDP_BOUND"); reply["session_id"] = conn.session_id; reply["owner_epoch"] = 0;
+  send_realtime(conn.id,reply);
+  const auto before = room_.status(); room_.bind_realtime(conn.id);
+  if (before == "LOBBY" && room_.status() == "RUNNING") start_room();
+}
 void Server::start_room() {
   replay_.start(room_);
   accumulator_.reset(read_monotonic());
@@ -390,7 +492,7 @@ void Server::publish_snapshots(const std::string& state_hash) {
     if (auto* conn = connection(id)) {
       auto snapshot = conn->snapshots.publish(room_,state_hash);
       snapshot_retention_high_water_ = std::max(snapshot_retention_high_water_,conn->snapshots.high_water());
-      queue(id,std::move(snapshot));
+      send_realtime(id,snapshot);
     }
   }
 }
@@ -411,12 +513,23 @@ void Server::handle(const Envelope& envelope) {
     }
     if (value.at("v") != 1) { reject("PROTOCOL_VERSION_UNSUPPORTED", "only protocol v1 is supported"); return; }
     const auto type = value.at("type").get<std::string>();
+    if ((!envelope.udp_endpoint && realtime_type(type)) || (envelope.udp_endpoint && !realtime_type(type))) {
+      reject("WRONG_TRANSPORT","message requires its contract transport"); return;
+    }
+    if (type == "UDP_BIND") { bind_datagram(*conn,envelope); return; }
+    if (envelope.udp_endpoint && (!conn->udp_endpoint || !same_endpoint(*conn->udp_endpoint,*envelope.udp_endpoint) ||
+        !value.contains("owner_epoch") || !value.at("owner_epoch").is_number_integer() || value.at("owner_epoch") != 0)) {
+      reject("UDP_BIND_INVALID","bound observed endpoint and standalone epoch required"); return;
+    }
     if (type == "HELLO") {
-      if (conn->session_id.empty()) conn->session_id = new_id("session", id);
+      if (conn->session_id.empty()) {
+        conn->session_id = new_id("session", id); conn->bind_token = new_bind_token(); conn->token_issued_ms = monotonic_now_();
+      }
       Json reply = message("WELCOME"); reply["session_id"] = conn->session_id;
+      reply["udp_bind_token"] = conn->bind_token; reply["udp_port"] = udp_port_;
       queue(id, std::move(reply)); return;
     }
-    if (type != "CREATE_ROOM" && type != "JOIN_ROOM" && type != "LEAVE_ROOM" && type != "INPUT" && type != "SNAPSHOT_ACK") {
+    if (type != "CREATE_ROOM" && type != "JOIN_ROOM" && type != "LEAVE_ROOM" && type != "INPUT" && type != "SNAPSHOT_ACK" && type != "PING") {
       reject("MESSAGE_TYPE_UNKNOWN", "unknown message type"); return;
     }
     if (conn->session_id.empty() || value.at("session_id").get<std::string>() != conn->session_id) {
@@ -437,6 +550,7 @@ void Server::handle(const Envelope& envelope) {
       }
       conn->player_id = new_id("player", next_player_++);
       const auto& player = room_.join(conn->player_id, conn->session_id, id);
+      if (conn->udp_endpoint) room_.bind_realtime(id);
       Json reply = message("ROOM_JOINED"); reply["room_id"] = room_.id(); reply["player_id"] = player.id;
       reply["slot"] = player.slot; reply["status"] = room_.status();
       queue(id, std::move(reply));
@@ -449,6 +563,10 @@ void Server::handle(const Envelope& envelope) {
     if (type == "LEAVE_ROOM") {
       leave_room(id,"LEAVE_ROOM"); return;
     }
+    if (type == "PING") {
+      auto reply = message("PONG"); reply["room_id"] = room_.id(); reply["player_id"] = conn->player_id;
+      reply["owner_epoch"] = 0; send_realtime(id,reply); return;
+    }
     if (type == "SNAPSHOT_ACK") {
       const auto& sequence = value.at("snapshot_seq");
       if (!sequence.is_number_integer() || (!sequence.is_number_unsigned() && sequence.get<std::int64_t>() < 0))
@@ -463,7 +581,7 @@ void Server::handle(const Envelope& envelope) {
     Json reply = message("INPUT_ACK"); reply["player_id"] = conn->player_id; reply["accepted"] = true;
     reply["seq"] = value.at("seq").get<std::uint64_t>(); reply["code"] = result.duplicate ? "DUPLICATE" : "ACCEPTED";
     reply["last_accepted_seq"] = room_.players().at(conn->player_id).last_accepted_seq();
-    reply["tick"] = room_.executed_ticks(); queue(id, std::move(reply));
+    reply["tick"] = room_.executed_ticks(); send_realtime(id,reply);
   } catch (const Json::exception&) {
     reject("MESSAGE_INVALID", "required field missing or wrong type");
   } catch (const std::invalid_argument&) {
@@ -471,6 +589,7 @@ void Server::handle(const Envelope& envelope) {
   }
 }
 void Server::leave_room(std::uint64_t connection_id, const std::string& kind) {
+  const auto previous_status = room_.status();
   std::string player_id;
   if (room_.status() == "RUNNING") {
     for (const auto& [id, player] : room_.players())
@@ -479,6 +598,7 @@ void Server::leave_room(std::uint64_t connection_id, const std::string& kind) {
   room_.leave(connection_id);
   if (!player_id.empty()) replay_.left(room_,player_id,kind);
   if (auto* conn = connection(connection_id)) conn->snapshots.clear();
+  if (previous_status == "LOBBY" && room_.status() == "RUNNING") start_room();
 }
 void Server::drain_mailbox() {
   // Room mutations happen after all ready I/O callbacks have completed.
@@ -535,14 +655,19 @@ void Server::advance_one_tick() {
 }
 Json Server::metrics() const {
   auto streams = Json::object();
+  std::size_t bound = 0, tokens = 0;
   for (const auto& [fd, conn] : connections_) {
     (void)fd;
     if (!conn.player_id.empty()) streams[conn.player_id] = conn.snapshots.metrics();
+    bound += conn.udp_endpoint.has_value(); tokens += !conn.bind_token.empty();
   }
   return Json{{"received_messages", received_messages_}, {"sent_messages", sent_messages_},
     {"mailbox_high_water", mailbox_high_water_}, {"outbound_control_high_water", outbound_high_water_},
     {"connection_high_water", connection_high_water_}, {"input_per_player_high_water", room_.input_high_water()},
     {"snapshot_retention_high_water",snapshot_retention_high_water_},{"snapshot_streams",streams},
+    {"udp_received_datagrams",received_datagrams_},{"udp_sent_datagrams",sent_datagrams_},
+    {"udp_payload_high_water",datagram_high_water_},{"udp_outbound_high_water",outbound_datagram_high_water_},
+    {"udp_receive_buffer_bytes",max_datagram_bytes+1},{"udp_bound_endpoints",bound},{"udp_bind_tokens",tokens},
     {"input_attempt_per_player_high_water", room_.input_attempt_high_water()},
     {"replay_bytes_high_water",replay_.high_water_bytes()},
     {"replay_capture_complete",replay_.complete()},{"replay_capture_error",replay_.failure()},
@@ -558,16 +683,18 @@ Json Server::metrics() const {
       {"operational_state", last_batch_.overloaded ? "OVERLOADED" : "NORMAL"}}}};
 }
 Json Server::cleanup() const {
-  std::size_t queued = 0, parser_buffered = 0, input_attempts = 0, retained_snapshots = 0;
+  std::size_t queued = 0, parser_buffered = 0, input_attempts = 0, retained_snapshots = 0, endpoints = 0, tokens = 0;
   for (const auto& [fd, conn] : connections_) {
     (void)fd; queued += conn.outbound.size(); parser_buffered += conn.parser.buffered_bytes();
     retained_snapshots += conn.snapshots.size();
+    endpoints += conn.udp_endpoint.has_value(); tokens += !conn.bind_token.empty();
   }
   for (const auto& [id, player] : room_.players()) { (void)id; input_attempts += player.input_attempts; }
   return Json{{"server_connections", connections_.size()}, {"server_descriptors", owned_descriptors().size()},
     {"mailbox_messages", mailbox_.size()}, {"pending_inputs", room_.pending_count()}, {"outbound_messages", queued},
     {"input_attempts", input_attempts},
     {"retained_snapshots",retained_snapshots},
+    {"udp_bound_endpoints",endpoints},{"udp_bind_tokens",tokens},{"udp_descriptors",datagram_.get() >= 0 ? 1 : 0},
     {"replay_bytes",replay_.bytes()},{"replay_pending_events",replay_.pending_events()},
     {"parser_buffered_bytes", parser_buffered}, {"parser_storage_bytes", connections_.size() * FrameParser::storage_bytes},
     {"worker_threads", 0}, {"timers", 0}, {"disconnect_notifications", disconnected_.size()},
@@ -577,6 +704,7 @@ std::vector<int> Server::owned_descriptors() const {
   std::vector<int> descriptors;
   if (reactor_.get() >= 0) descriptors.push_back(reactor_.get());
   if (listener_.get() >= 0) descriptors.push_back(listener_.get());
+  if (datagram_.get() >= 0) descriptors.push_back(datagram_.get());
   for (const auto& [fd, conn] : connections_) { (void)conn; descriptors.push_back(fd); }
   return descriptors;
 }
@@ -584,6 +712,7 @@ void Server::shutdown() {
   if (stopping_) return;
   stopping_ = true;
   listener_.reset();
+  datagram_.reset();
   drain_mailbox();
   room_.close();
   replay_.clear();
@@ -650,6 +779,10 @@ std::optional<Json> TcpClient::try_receive() {
   count = ::recv(fd_.get(), bytes.data(), total, 0);
   if (count != static_cast<ssize_t>(total)) throw std::runtime_error("client complete-frame read failed");
   Json value = decode_complete_frame(std::span(bytes).first(total));
+  if (value.at("type") == "WELCOME" && value.contains("udp_bind_token")) {
+    bind_token_ = value.at("udp_bind_token").get<std::string>(); udp_port_ = value.at("udp_port").get<std::uint16_t>();
+    value.erase("udp_bind_token");
+  }
   if (observations_.size() == 4096) throw std::runtime_error("client observation bound exceeded");
   observations_.push_back(value);
   return value;
@@ -670,4 +803,56 @@ Json TcpClient::receive_type(Server& server, const std::string& type) {
   }
   throw std::runtime_error("expected response absent within control bound");
 }
+Json TcpClient::bind_request(const std::string& session_id, int owner_epoch) const {
+  auto value = message("UDP_BIND"); value["session_id"] = session_id;
+  value["udp_bind_token"] = bind_token_; value["owner_epoch"] = owner_epoch; return value;
+}
+UdpClient::UdpClient(std::uint16_t server_port) : fd_(::socket(AF_INET,SOCK_DGRAM,0)) {
+  if (fd_.get() < 0) system_failure("UDP client socket");
+  nonblocking(fd_.get()); sockaddr_in local{}; local.sin_family = AF_INET; local.sin_addr.s_addr = htonl(INADDR_LOOPBACK);
+  if (::bind(fd_.get(),reinterpret_cast<sockaddr*>(&local),sizeof(local)) < 0) system_failure("UDP client bind");
+  server_.sin_family = AF_INET; server_.sin_addr.s_addr = htonl(INADDR_LOOPBACK); server_.sin_port = htons(server_port);
+}
+void UdpClient::send(const Json& value) { send_bytes(encode_datagram(value)); }
+void UdpClient::send_bytes(std::span<const std::uint8_t> bytes) {
+  const auto count = ::sendto(fd_.get(),bytes.data(),bytes.size(),0,reinterpret_cast<const sockaddr*>(&server_),sizeof(server_));
+  if (count != static_cast<ssize_t>(bytes.size())) system_failure("UDP client send");
+}
+std::optional<Json> UdpClient::try_receive() {
+  std::array<std::uint8_t,max_datagram_bytes+1> bytes{}; sockaddr_in source{}; socklen_t size = sizeof(source);
+  const auto count = ::recvfrom(fd_.get(),bytes.data(),bytes.size(),0,reinterpret_cast<sockaddr*>(&source),&size);
+  if (count < 0) { if (transient_io()) return std::nullopt; system_failure("UDP client receive"); }
+  if (!same_endpoint(source,server_)) throw std::runtime_error("UDP client unexpected sender");
+  auto value = decode_datagram(std::span(bytes).first(static_cast<std::size_t>(count)));
+  if (observations_.size() == 4096) throw std::runtime_error("UDP observation bound exceeded");
+  observations_.push_back(value); return value;
+}
+Json UdpClient::receive(Server& server) {
+  const auto deadline = std::chrono::steady_clock::now()+std::chrono::seconds(2);
+  do { server.pump(); if (auto value = try_receive()) return *value; } while (std::chrono::steady_clock::now() < deadline);
+  throw std::runtime_error("UDP response deadline exceeded");
+}
+Json UdpClient::receive_type(Server& server, const std::string& type) {
+  for (int count = 0; count < 64; ++count) { auto value = receive(server); if (value.at("type") == type) return value; }
+  throw std::runtime_error("expected UDP response absent within bound");
+}
+void UdpClient::bind(TcpClient& control, Server& server, const std::string& session_id) {
+  send(control.bind_request(session_id)); const auto bound = receive_type(server,"UDP_BOUND");
+  if (bound.at("session_id") != session_id || bound.at("owner_epoch") != 0) throw std::runtime_error("UDP binding identity");
+}
+Json receive_input_result(TcpClient& control, UdpClient& realtime, Server& server) {
+  const auto deadline = std::chrono::steady_clock::now()+std::chrono::seconds(2);
+  do {
+    server.pump();
+    if (auto value = control.try_receive()) {
+      if (value->at("type") != "ERROR") throw std::runtime_error("unexpected input control response");
+      return *value;
+    }
+    if (auto value = realtime.try_receive()) {
+      if (value->at("type") == "INPUT_ACK") return *value;
+      if (value->at("type") != "SNAPSHOT") throw std::runtime_error("unexpected input UDP response");
+    }
+  } while (std::chrono::steady_clock::now() < deadline);
+  throw std::runtime_error("input response deadline exceeded");
+}
 }
diff --git a/src/transport.hpp b/src/transport.hpp
index fcaf4c9..752f73e 100644
--- a/src/transport.hpp
+++ b/src/transport.hpp
@@ -7,6 +7,7 @@
 #include <functional>
 #include <span>
 #include <set>
+#include <netinet/in.h>
 
 namespace arena {
 class Fd {
@@ -27,6 +28,8 @@ class Fd {
 
 std::vector<std::uint8_t> encode_frame(const Json& value);
 Json decode_complete_frame(std::span<const std::uint8_t> bytes);
+std::vector<std::uint8_t> encode_datagram(const Json& value);
+Json decode_datagram(std::span<const std::uint8_t> bytes);
 Input decode_input(const Json& value);
 // The caller has attributed the request to its authenticated Room/player.
 InputResult admit_input(Room& room, const std::string& player_id, const Json& value);
@@ -72,6 +75,7 @@ class Server {
   Server(const Server&) = delete;
   Server& operator=(const Server&) = delete;
   std::uint16_t port() const { return port_; }
+  std::uint16_t udp_port() const { return udp_port_; }
   void poll_io(int timeout_ms = 0);
   void drain_mailbox();
   void pump(int timeout_ms = 0);
@@ -93,8 +97,14 @@ class Server {
     std::size_t pending_requests = 0;
     FrameParser parser;
     SnapshotStream snapshots;
+    std::string bind_token = {};
+    std::int64_t token_issued_ms = 0;
+    std::optional<sockaddr_in> udp_endpoint = {};
+  };
+  struct Envelope {
+    std::uint64_t connection_id; Json value; std::string parser_error;
+    std::optional<sockaddr_in> udp_endpoint = {};
   };
-  struct Envelope { std::uint64_t connection_id; Json value; std::string parser_error; };
   // The same bounded storage is used by the reactor and the pure ownership
   // regression. This extraction does not add a second admission policy.
   class Mailbox {
@@ -109,11 +119,17 @@ class Server {
   friend Json run_mailbox_probe(std::size_t capacity);
 #ifdef ARENA_TEST_FIXTURES
   friend struct ReplayFixture;
+  friend struct UdpFixture;
+  std::optional<std::string> fixture_room_id_;
+  std::vector<std::string> fixture_player_ids_;
 #endif
   Connection* connection(std::uint64_t id);
   void register_event(int fd, short filter, unsigned short flags, std::uint64_t connection_id = 0);
   void accept_ready();
   void read_ready(int fd);
+  void read_datagrams();
+  void send_realtime(std::uint64_t connection_id, const Json& value);
+  void bind_datagram(Connection& conn, const Envelope& envelope);
   void write_ready(int fd);
   void disconnect(int fd, const std::string& reason);
   void end_transport(int fd, bool io_error);
@@ -134,7 +150,9 @@ class Server {
   int catch_up_high_water_ = 0;
   Fd reactor_;
   Fd listener_;
+  Fd datagram_;
   std::uint16_t port_ = 0;
+  std::uint16_t udp_port_ = 0;
   std::map<int, Connection> connections_;
   Mailbox mailbox_;
   std::set<std::uint64_t> disconnected_;
@@ -157,6 +175,10 @@ class Server {
   std::uint64_t received_messages_ = 0;
   std::uint64_t sent_messages_ = 0;
   std::uint64_t partial_writes_ = 0;
+  std::uint64_t received_datagrams_ = 0;
+  std::uint64_t sent_datagrams_ = 0;
+  std::size_t datagram_high_water_ = 0;
+  std::size_t outbound_datagram_high_water_ = 0;
   std::map<std::string, std::uint64_t> errors_;
   bool stopping_ = false;
 };
@@ -172,12 +194,37 @@ class TcpClient {
   bool peer_closed() const;
   Json receive(Server& server);
   Json receive_type(Server& server, const std::string& type);
-  void close() { fd_.reset(); }
+  void close() { fd_.reset(); bind_token_.clear(); }
+  int descriptor() const { return fd_.get(); }
+  bool has_bind_token() const { return !bind_token_.empty(); }
+  std::uint16_t udp_port() const { return udp_port_; }
+  Json bind_request(const std::string& session_id, int owner_epoch = 0) const;
+  const std::vector<Json>& observations() const { return observations_; }
+ private:
+  Fd fd_;
+  std::vector<Json> observations_;
+  std::string bind_token_;
+  std::uint16_t udp_port_ = 0;
+};
+// Real UDP test/CLI client; credentials remain in the TCP client's private
+// storage and are removed from every returned/loggable WELCOME observation.
+class UdpClient {
+ public:
+  explicit UdpClient(std::uint16_t server_port);
+  void send(const Json& value);
+  void send_bytes(std::span<const std::uint8_t> bytes);
+  std::optional<Json> try_receive();
+  Json receive(Server& server);
+  Json receive_type(Server& server, const std::string& type);
+  void bind(TcpClient& control, Server& server, const std::string& session_id);
   int descriptor() const { return fd_.get(); }
+  void close() { fd_.reset(); }
   const std::vector<Json>& observations() const { return observations_; }
  private:
   Fd fd_;
+  sockaddr_in server_{};
   std::vector<Json> observations_;
 };
+Json receive_input_result(TcpClient& control, UdpClient& realtime, Server& server);
 bool descriptor_closed(int fd);
 }
diff --git a/tests/g07.cpp b/tests/g07.cpp
index 40ce17e..9c5fedf 100644
--- a/tests/g07.cpp
+++ b/tests/g07.cpp
@@ -22,6 +22,7 @@ bool identifier(const std::string& value) {
 }
 struct Peer {
   std::unique_ptr<TcpClient> tcp;
+  std::unique_ptr<UdpClient> udp;
   std::string session;
   std::string player;
 };
@@ -50,14 +51,6 @@ Json request(const Json& event, const std::map<std::string,Peer>& peers, const s
   }
   return value;
 }
-Json receive_control(TcpClient& peer, Server& server) {
-  for (int count = 0; count < 64; ++count) {
-    auto value = peer.receive(server);
-    if (value.at("type") != "SNAPSHOT") return value;
-    require(value.contains("snapshot_seq") && value.contains("state_hash"), "conformant snapshot during control demultiplexing");
-  }
-  throw std::runtime_error("control response absent within bounded snapshot drain");
-}
 void leave_barrier(Peer& peer, Server& server) {
   const auto tick = server.room().executed_ticks();
   const auto deadline = std::chrono::steady_clock::now() + std::chrono::seconds(5);
@@ -72,7 +65,7 @@ void drain_periodic(Server& server, const std::map<std::string,Peer>& peers) {
   require(server.cleanup().at("outbound_messages") == 0, "bounded periodic flush");
   for (const auto& [role,peer] : peers) {
     (void)role;
-    while (auto value = peer.tcp->try_receive())
+    while (auto value = peer.udp->try_receive())
       require(value->at("type") == "SNAPSHOT" && value->contains("snapshot_seq"), "unexpected frame during periodic drain");
   }
 }
@@ -97,6 +90,7 @@ struct ReplayFixture {
     const auto count = players.size();
     room.create(room_id);
     for (auto& player : players) {
+      player.realtime_ready = true; // Recorded initial RUNNING model, only in this test build.
       const auto id = player.id;
       require(room.players_.emplace(id,std::move(player)).second, "duplicate initial player");
     }
@@ -114,7 +108,9 @@ struct ReplayFixture {
       auto found = std::find_if(server.connections_.begin(),server.connections_.end(),
         [&](const auto& entry) { return entry.second.session_id == peer.session; });
       require(found != server.connections_.end() && found->second.player_id.empty(), "real server session binding");
-      auto& connection = found->second; connection.player_id = peer.player;
+      auto& connection = found->second;
+      require(connection.udp_endpoint.has_value(), "real UDP binding precedes the historical fixture start");
+      connection.player_id = peer.player;
       Player player; player.id = peer.player; player.session_id = connection.session_id;
       player.connection_id = connection.id; player.slot = item.at("slot").get<int>();
       require(identifier(player.id) && player.slot == static_cast<int>(players.size()), "frozen identifiers/ordered slots");
@@ -165,6 +161,7 @@ ReplayRun run_replay_scenario(const Json& scenario, bool rejected_removed) {
     const auto role = item.at("client").get<std::string>(); auto& peer = peers[role];
     peer.tcp = std::make_unique<TcpClient>(server.port()); peer.tcp->send(message("HELLO"));
     peer.session = peer.tcp->receive_type(server,"WELCOME").at("session_id").get<std::string>();
+    peer.udp = std::make_unique<UdpClient>(peer.tcp->udp_port()); peer.udp->bind(*peer.tcp,server,peer.session);
     peer.player = item.at("player_id").get<std::string>();
   }
   const auto bindings = ReplayFixture::live(server,scenario,peers);
@@ -180,8 +177,9 @@ ReplayRun run_replay_scenario(const Json& scenario, bool rejected_removed) {
       }
       const auto role = event.at("client").get<std::string>(); auto& peer = peers.at(role);
       const auto value = request(event,peers,server.room().id()); const auto before = observed_state(server.room());
-      peer.tcp->send(value); Json response = nullptr;
-      if (kind == "INPUT") response = receive_control(*peer.tcp,server);
+      if (event.at("kind") == "INPUT") peer.udp->send(value); else peer.tcp->send(value);
+      Json response = nullptr;
+      if (kind == "INPUT") response = receive_input_result(*peer.tcp,*peer.udp,server);
       else leave_barrier(peer,server);
       const auto after = observed_state(server.room());
       wire.push_back(Json{{"before_tick",tick},{"client",role},{"request",value},{"response",response},
@@ -203,7 +201,7 @@ ReplayRun run_replay_scenario(const Json& scenario, bool rejected_removed) {
     const auto hash = server.replay().artifact().at("ticks").back().at("state_hash").get<std::string>();
     hashes.push_back(hash); run.records.push_back(tick_record(server.room(),hash));
     if (tick == 199) {
-      const auto failure = receive_control(*peers.at("charlie").tcp,server);
+      const auto failure = receive_input_result(*peers.at("charlie").tcp,*peers.at("charlie").udp,server);
       require(failure.at("code") == "ACTION_REJECTED" && failure.at("tick") == 199 &&
         failure.at("player_id") == peers.at("charlie").player, "accepted charlie TAG action failure");
       actions.push_back(failure);
@@ -216,9 +214,9 @@ ReplayRun run_replay_scenario(const Json& scenario, bool rejected_removed) {
   require(run.replay_bytes.size() <= max_replay_bytes, "artifact byte bound");
   const auto final = observed_state(server.room()); const auto metrics = server.metrics();
   auto descriptors = server.owned_descriptors();
-  for (const auto& [role,peer] : peers) { (void)role; descriptors.push_back(peer.tcp->descriptor()); }
+  for (const auto& [role,peer] : peers) { (void)role; descriptors.push_back(peer.tcp->descriptor()); descriptors.push_back(peer.udp->descriptor()); }
   server.shutdown();
-  for (const auto& [role,peer] : peers) { (void)role; peer.tcp->close(); }
+  for (const auto& [role,peer] : peers) { (void)role; peer.tcp->close(); peer.udp->close(); }
   for (int fd : descriptors) require(descriptor_closed(fd), "descriptor survived shutdown");
   auto cleanup = server.cleanup();
   for (const auto& [key,value] : cleanup.items()) { (void)key; require(value == 0,"active resource survived shutdown"); }
@@ -251,6 +249,7 @@ Json run_snapshot_scenario(const Json& scenario) {
     const auto role = item.at("client").get<std::string>(); auto& peer = peers[role];
     peer.tcp = std::make_unique<TcpClient>(server.port()); peer.tcp->send(message("HELLO"));
     peer.session = peer.tcp->receive_type(server,"WELCOME").at("session_id").get<std::string>();
+    peer.udp = std::make_unique<UdpClient>(peer.tcp->udp_port()); peer.udp->bind(*peer.tcp,server,peer.session);
     peer.player = item.at("player_id").get<std::string>(); cadence[role] = Json::array();
   }
   const auto bindings = ReplayFixture::live(server,scenario,peers);
@@ -273,7 +272,7 @@ Json run_snapshot_scenario(const Json& scenario) {
     Json alpha_capture, acks = Json::array();
     for (auto& [role,peer] : peers) {
       if (!server.room().players().at(peer.player).connected) continue;
-      auto& replica = replicas[role]; const auto wire = peer.tcp->receive(server);
+      auto& replica = replicas[role]; const auto wire = peer.udp->receive(server);
       const bool full = sequence == 1 || sequence == 4 || sequence % 20 == 0;
       require(wire.at("v") == 1 && wire.at("type") == "SNAPSHOT" && wire.at("snapshot_seq").is_number_integer() &&
         wire.at("snapshot_seq") == ++replica.count && replica.count == sequence && wire.at("room_id") == server.room().id() &&
@@ -332,8 +331,8 @@ Json run_snapshot_scenario(const Json& scenario) {
       }
       require(ack_sequence.has_value(),"G08 actual applied sequence has ACK feedback");
       auto ack = message("SNAPSHOT_ACK"); ack.update(Json{{"session_id",peer.session},{"room_id",server.room().id()},
-        {"player_id",peer.player},{"snapshot_seq",*ack_sequence}});
-      peer.tcp->send(ack); acks.push_back(Json{{"client",role},{"request",ack}});
+        {"player_id",peer.player},{"snapshot_seq",*ack_sequence},{"owner_epoch",0}});
+      peer.udp->send(ack); acks.push_back(Json{{"client",role},{"request",ack}});
     }
     const auto all_acks_owned = [&] {
       const auto streams = server.metrics().at("snapshot_streams");
@@ -362,9 +361,10 @@ Json run_snapshot_scenario(const Json& scenario) {
       if (event.at("before_tick") != tick) continue;
       const auto role = event.at("client").get<std::string>(); auto& peer = peers.at(role);
       const auto value = request(event,peers,server.room().id()), before = observed_state(server.room());
-      peer.tcp->send(value); Json response = nullptr;
+      if (event.at("kind") == "INPUT") peer.udp->send(value); else peer.tcp->send(value);
+      Json response = nullptr;
       if (event.at("kind") == "INPUT") {
-        response = peer.tcp->receive(server);
+        response = receive_input_result(*peer.tcp,*peer.udp,server);
         require(response.at("type") == "INPUT_ACK" && response.at("code") == "ACCEPTED" &&
           response.at("seq") == event.at("seq") && response.at("tick") == tick,"G08 original actual input admission");
       } else leave_barrier(peer,server);
@@ -384,7 +384,7 @@ Json run_snapshot_scenario(const Json& scenario) {
     // No unscheduled fallback, tick0 publication or leave/close debug notice.
     server.pump();
     require(server.cleanup().at("outbound_messages") == 0,"G08 all scheduled publications drained");
-    for (const auto& [role,peer] : peers) { (void)role; require(!peer.tcp->try_receive(),"G08 unexpected out-of-cadence wire message"); }
+    for (const auto& [role,peer] : peers) { (void)role; require(!peer.tcp->try_receive() && !peer.udp->try_receive(),"G08 unexpected out-of-cadence wire message"); }
   }
   Json counts = Json::object(), final_applied = Json::object();
   for (const auto& [role,replica] : replicas) {
@@ -397,13 +397,13 @@ Json run_snapshot_scenario(const Json& scenario) {
   require(metrics.at("snapshot_retention_high_water") == 32 && metrics.at("snapshot_streams").at("player-03").at("retained_count") == 0,
           "G08 exact retention high-water and departed stream release");
   auto descriptors = server.owned_descriptors();
-  for (const auto& [role,peer] : peers) { (void)role; descriptors.push_back(peer.tcp->descriptor()); }
+  for (const auto& [role,peer] : peers) { (void)role; descriptors.push_back(peer.tcp->descriptor()); descriptors.push_back(peer.udp->descriptor()); }
   server.shutdown(); Json eof = Json::object();
   for (const auto& [role,peer] : peers) {
     const auto deadline = std::chrono::steady_clock::now() + std::chrono::seconds(5);
     while (!peer.tcp->peer_closed() && std::chrono::steady_clock::now() < deadline)
       require(!peer.tcp->try_receive(),"G08 shutdown emits no out-of-stream notice");
-    require(peer.tcp->peer_closed(),"G08 actual terminal TCP EOF"); eof[role] = true; peer.tcp->close();
+    require(peer.tcp->peer_closed(),"G08 actual terminal TCP EOF"); eof[role] = true; peer.tcp->close(); peer.udp->close();
   }
   for (int fd : descriptors) require(descriptor_closed(fd),"G08 actual descriptor closure");
   auto cleanup = server.cleanup();
diff --git a/tests/g09.cpp b/tests/g09.cpp
new file mode 100644
index 0000000..c1bc382
--- /dev/null
+++ b/tests/g09.cpp
@@ -0,0 +1,537 @@
+#include "g09.hpp"
+#ifndef ARENA_TEST_FIXTURES
+#error G09 fixture identifiers must not enter the shipping build
+#endif
+#include <algorithm>
+#include <array>
+#include <cerrno>
+#include <chrono>
+#include <fcntl.h>
+#include <memory>
+#include <set>
+#include <stdexcept>
+#include <sys/socket.h>
+#include <unistd.h>
+
+namespace arena {
+namespace {
+void need(bool condition, const std::string& text) { if (!condition) throw std::runtime_error("G09: "+text); }
+bool identifier(const std::string& value) {
+  return !value.empty() && value.size() <= 64 && std::all_of(value.begin(),value.end(),[](unsigned char c) {
+    return (c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z') || (c >= '0' && c <= '9') || c == '-' || c == '_';
+  });
+}
+Json state(const Room& room) {
+  auto value = room.view(); value["owner_epoch"] = 0;
+  for (auto& row : value["players"]) {
+    const auto& player = room.players().at(row.at("player_id").get<std::string>());
+    row["last_seq"] = player.last_accepted_seq(); row["pending"] = player.pending.size();
+    row["applied_seq"] = player.applied_seq ? Json(*player.applied_seq) : Json(nullptr);
+  }
+  return value;
+}
+}
+struct UdpFixture {
+  static Json maximum_full(const std::string& room_id, const std::vector<std::string>& ids) {
+    // Only test-owned scalar state is set. No simulation or extra campaign is
+    // used to measure the production full serializer at reachable field widths.
+    Room room; room.create(room_id);
+    for (std::size_t i = 0; i < ids.size(); ++i) {
+      auto& p = room.join(ids[i],"size-session",i+1); p.x = 100000; p.y = 100000;
+      p.direction = Direction::north; p.score = 60;
+    }
+    room.status_ = "FINISHED"; room.executed_ticks_ = 1200;
+    SnapshotStream stream; auto full = stream.publish(room,sha256(canonical_state(room)));
+    full["snapshot_seq"] = 601; return full;
+  }
+  static void identifiers(Server& server, const Json& scenario) {
+    need(server.room_.status() == "ABSENT" && server.connections_.empty(),"ID injection precedes all normal joins");
+    const auto room = scenario.at("room_id").get<std::string>(); std::vector<std::string> ids; std::set<std::string> unique;
+    need(identifier(room) && !scenario.at("players").empty() && scenario.at("players").size() <= max_players,"bounded fixture identifiers");
+    for (const auto& row : scenario.at("players")) {
+      const auto id = row.at("player_id").get<std::string>();
+      need(identifier(id) && unique.insert(id).second,"ASCII1..64 unique fixture player identifier"); ids.push_back(id);
+    }
+    (void)encode_datagram(maximum_full(room,ids)); // Refuse an oversized fixture before any state/transport change.
+    server.fixture_room_id_ = room; server.fixture_player_ids_ = std::move(ids);
+  }
+  static Json outbound_probe(Server& server, const std::string& player_id) {
+    std::vector<std::string> ids;
+    for (int i = 1; i <= 8; ++i) ids.push_back(server.new_id("player",static_cast<std::uint64_t>(i)));
+    const auto room_id = server.new_id("room",1); const auto full = maximum_full(room_id,ids);
+    const auto encoded = encode_datagram(full);
+    need(full.at("players").size() == 8 && encoded.size() <= max_datagram_bytes,"normal max8 full fits exactly bounded UDP");
+    std::vector<std::string> long_ids; Json invalid{{"room_id",std::string(64,'R')},{"players",Json::array()}};
+    for (int i = 0; i < 8; ++i) {
+      long_ids.push_back(std::string(63,'P')+std::to_string(i)); invalid["players"].push_back(Json{{"player_id",long_ids.back()}});
+    }
+    bool rejected = false;
+    {
+      ManualClock clock; Server candidate(clock);
+      try { identifiers(candidate,invalid); } catch (const std::length_error&) { rejected = true; }
+      need(rejected && !candidate.fixture_room_id_ && candidate.room_.status() == "ABSENT","oversized fixture is explicitly rejected before injection");
+      candidate.shutdown();
+    }
+    const auto before = server.errors_["UDP_OUTBOUND_SIZE_INVALID"], sent = server.sent_datagrams_;
+    const auto& player = server.room_.players().at(player_id);
+    server.send_realtime(player.connection_id,maximum_full(std::string(64,'R'),long_ids));
+    need(server.errors_["UDP_OUTBOUND_SIZE_INVALID"] == before+1 && server.sent_datagrams_ == sent,"real outbound size rejection emits no datagram");
+    Json lengths = Json::array(); for (const auto& id : ids) lengths.push_back(id.size());
+    return Json{{"players",8},{"room_id_length",room_id.size()},{"player_id_lengths",lengths},{"max_full_bytes",encoded.size()},
+      {"oversized_fixture_rejected",rejected},{"outbound_size_rejections",1},{"sent_datagrams_delta",0},{"executed_ticks",0}};
+  }
+};
+namespace {
+// A real loopback relay for alpha only. Bind credentials are forwarded as
+// opaque bytes and never retained in its trace. Only the two frozen streams
+// receive faults; each direction owns at most one held <=1200-byte datagram.
+class FaultProxy {
+ public:
+  explicit FaultProxy(std::uint16_t server_port) : fd_(::socket(AF_INET,SOCK_DGRAM,0)) {
+    need(fd_.get() >= 0,"proxy socket");
+    const int flags = ::fcntl(fd_.get(),F_GETFL,0);
+    need(flags >= 0 && ::fcntl(fd_.get(),F_SETFL,flags|O_NONBLOCK) == 0 &&
+      ::fcntl(fd_.get(),F_SETFD,FD_CLOEXEC) == 0,"bounded nonblocking proxy");
+    sockaddr_in local{}; local.sin_family = AF_INET; local.sin_addr.s_addr = htonl(INADDR_LOOPBACK);
+    need(::bind(fd_.get(),reinterpret_cast<sockaddr*>(&local),sizeof(local)) == 0,"proxy bind");
+    socklen_t size = sizeof(local);
+    need(::getsockname(fd_.get(),reinterpret_cast<sockaddr*>(&local),&size) == 0,"proxy port");
+    port_ = ntohs(local.sin_port); server_ = local; server_.sin_port = htons(server_port);
+  }
+  std::uint16_t port() const { return port_; }
+  int descriptor() const { return fd_.get(); }
+  void close() { for (auto& held : held_) held.reset(); fd_.reset(); }
+  int originals(int direction) const { return originals_.at(static_cast<std::size_t>(direction)); }
+  std::size_t forwarded(int direction) const { return delivered_.at(static_cast<std::size_t>(direction)).size(); }
+  const Json& deliveries(int direction) const { return delivered_.at(static_cast<std::size_t>(direction)); }
+  const Json& snapshots() const { return snapshots_; }
+  Json evidence() const {
+    return Json{{"trace",trace_},{"input_originals",originals_[0]},{"snapshot_originals",originals_[1]},
+      {"input_delivery_ordinals",delivered_[0]},{"snapshot_delivery_ordinals",delivered_[1]},
+      {"held_high_water_per_direction",high_water_},{"held_packets",static_cast<int>(held_[0].has_value())+static_cast<int>(held_[1].has_value())},
+      {"datagram_max_bytes",max_datagram_bytes},{"passthrough",passthrough_},{"token_value_recorded",false}};
+  }
+  void pump() {
+    for (int work = 0; work < 64; ++work) {
+      std::array<std::uint8_t,max_datagram_bytes+1> bytes{}; sockaddr_in source{}; socklen_t size = sizeof(source);
+      const auto count = ::recvfrom(fd_.get(),bytes.data(),bytes.size(),0,reinterpret_cast<sockaddr*>(&source),&size);
+      if (count < 0) { need(errno == EAGAIN || errno == EWOULDBLOCK || errno == EINTR,"proxy receive"); return; }
+      need(count > 0 && count <= static_cast<ssize_t>(max_datagram_bytes),"proxy bounded datagram");
+      const bool from_server = same(source,server_);
+      if (!from_server) { if (!client_) client_ = source; need(same(source,*client_),"one alpha endpoint per proxy"); }
+      need(client_.has_value(),"server reply follows actual client bind");
+      auto value = decode_datagram(std::span(bytes).first(static_cast<std::size_t>(count)));
+      const auto type = value.at("type").get<std::string>();
+      const int direction = from_server ? 1 : 0;
+      Packet packet{{bytes.begin(),bytes.begin()+count},from_server ? *client_ : server_,direction,0,0};
+      if ((!from_server && type == "INPUT") || (from_server && type == "SNAPSHOT")) {
+        packet.ordinal = ++originals_[static_cast<std::size_t>(direction)];
+        packet.tick = value.at(from_server ? "tick" : "target_tick").get<int>();
+        need(value.at(from_server ? "snapshot_seq" : "seq") == packet.ordinal &&
+          packet.ordinal <= (from_server ? 13 : 24),"original stream count excludes unrelated and duplicate packets");
+        if (from_server) snapshots_.push_back(value);
+        trace(packet,"original",false);
+        if (packet.ordinal == 5) { trace(packet,"drop",false); continue; }
+        if (packet.ordinal == 10) {
+          need(!held_[static_cast<std::size_t>(direction)],"one pending reorder datagram"); trace(packet,"hold",false);
+          held_[static_cast<std::size_t>(direction)] = std::move(packet); high_water_[static_cast<std::size_t>(direction)] = 1; continue;
+        }
+        forward(packet,false);
+        if (packet.ordinal == 8) forward(packet,true);
+        if (packet.ordinal == 11) {
+          auto& held = held_[static_cast<std::size_t>(direction)]; need(held.has_value(),"original10 held until11");
+          forward(*held,false); held.reset();
+        }
+      } else {
+        ++passthrough_[type]; send(packet);
+      }
+    }
+  }
+ private:
+  struct Packet { std::vector<std::uint8_t> bytes; sockaddr_in destination; int direction; int ordinal; int tick; };
+  static bool same(const sockaddr_in& a, const sockaddr_in& b) {
+    return a.sin_family == b.sin_family && a.sin_addr.s_addr == b.sin_addr.s_addr && a.sin_port == b.sin_port;
+  }
+  void trace(const Packet& packet, const std::string& action, bool duplicate) {
+    need(trace_.size() < 128,"fixed fault trace bound");
+    trace_.push_back(Json{{"stream",packet.direction == 0 ? "alpha_INPUT" : "alpha_SNAPSHOT"},
+      {"ordinal",packet.ordinal},{"tick",packet.tick},{"event",action},{"duplicate_copy",duplicate},{"bytes",packet.bytes.size()}});
+  }
+  void send(const Packet& packet) {
+    const auto count = ::sendto(fd_.get(),packet.bytes.data(),packet.bytes.size(),0,
+      reinterpret_cast<const sockaddr*>(&packet.destination),sizeof(packet.destination));
+    need(count == static_cast<ssize_t>(packet.bytes.size()),"proxy actual complete sendto");
+  }
+  void forward(const Packet& packet, bool duplicate) {
+    send(packet); delivered_[static_cast<std::size_t>(packet.direction)].push_back(packet.ordinal); trace(packet,"deliver",duplicate);
+  }
+  Fd fd_;
+  std::uint16_t port_ = 0;
+  sockaddr_in server_{};
+  std::optional<sockaddr_in> client_;
+  std::array<int,2> originals_{0,0}, high_water_{0,0};
+  std::array<std::optional<Packet>,2> held_;
+  std::array<Json,2> delivered_{Json::array(),Json::array()};
+  Json snapshots_ = Json::array(), trace_ = Json::array();
+  std::map<std::string,int> passthrough_;
+};
+struct Peer {
+  std::unique_ptr<TcpClient> tcp;
+  std::unique_ptr<UdpClient> udp;
+  std::string session;
+  std::string player;
+};
+struct Fixture {
+  int descriptors_before = Fd::live();
+  ManualClock clock;
+  std::int64_t now = 0;
+  Server server{clock,0,[this] { return now; }};
+  std::unique_ptr<FaultProxy> proxy;
+  std::vector<Peer> peers;
+  std::vector<std::unique_ptr<UdpClient>> extras;
+  std::string room;
+  Json joins = Json::array();
+  explicit Fixture(int count, bool bound, const Json* fixed = nullptr, bool faults = false) {
+    if (fixed) UdpFixture::identifiers(server,*fixed);
+    if (faults) proxy = std::make_unique<FaultProxy>(server.udp_port());
+    for (int i = 0; i < count; ++i) {
+      Peer peer; peer.tcp = std::make_unique<TcpClient>(server.port()); peer.tcp->send(message("HELLO"));
+      const auto welcome = peer.tcp->receive_type(server,"WELCOME"); peer.session = welcome.at("session_id").get<std::string>();
+      need(peer.tcp->has_bind_token() && !welcome.contains("udp_bind_token"),"real token privately captured, never placed in evidence");
+      peer.udp = std::make_unique<UdpClient>(proxy && i == 0 ? proxy->port() : peer.tcp->udp_port()); peers.push_back(std::move(peer));
+    }
+    auto create = message("CREATE_ROOM"); create["session_id"] = peers[0].session; peers[0].tcp->send(create);
+    room = peers[0].tcp->receive_type(server,"ROOM_CREATED").at("room_id").get<std::string>();
+    for (int i = 0; i < count; ++i) {
+      auto& peer = peers[static_cast<std::size_t>(i)]; auto join = message("JOIN_ROOM");
+      join["session_id"] = peer.session; join["room_id"] = room; peer.tcp->send(join);
+      const auto result = peer.tcp->receive_type(server,"ROOM_JOINED"); peer.player = result.at("player_id").get<std::string>();
+      need(result.at("status") == "LOBBY" && server.room().status() == "LOBBY" && result.at("slot") == i &&
+        server.room().executed_ticks() == 0,"ordinary unbound joins remain LOBBY, including third/fourth");
+      joins.push_back(Json{{"slot",i},{"player_id",peer.player},{"status",result.at("status")},{"udp_ready",false}});
+    }
+    if (bound) bind_all();
+  }
+  void bind_all() {
+    for (std::size_t i = 0; i < peers.size(); ++i) {
+      auto& peer = peers[i]; peer.udp->bind(*peer.tcp,server,peer.session);
+      need(server.room().status() == (peers.size() >= 2 && i+1 == peers.size() ? "RUNNING" : "LOBBY"),"unchanged min2 AND all joined UDP ready");
+    }
+    if (peers.size() >= 2) for (auto& peer : peers) {
+      const auto snapshot = peer.udp->receive_type(server,"SNAPSHOT");
+      need(snapshot.at("snapshot_seq") == 1 && snapshot.at("tick") == -1,"one normal start full over UDP");
+    }
+  }
+  Json input(std::size_t index = 0) const {
+    const auto& p = peers.at(index);
+    return Json{{"v",1},{"type","INPUT"},{"session_id",p.session},{"room_id",room},{"player_id",p.player},
+      {"owner_epoch",0},{"seq",1},{"target_tick",0},{"direction","EAST"},{"tag_target_player_id",nullptr}};
+  }
+  UdpClient& extra() { extras.push_back(std::make_unique<UdpClient>(server.udp_port())); return *extras.back(); }
+  void error(std::size_t peer, const std::string& code) {
+    const auto value = peers.at(peer).tcp->receive_type(server,"ERROR"); need(value.at("code") == code,"stable identified-session TCP error");
+  }
+  void ping(std::size_t index = 0) {
+    auto value = input(index); value["type"] = "PING"; peers[index].udp->send(value);
+    need(peers[index].udp->receive_type(server,"PONG").at("player_id") == peers[index].player,"original endpoint retains UDP PING/PONG");
+  }
+  Json finish() {
+    auto fds = server.owned_descriptors();
+    for (const auto& peer : peers) { fds.push_back(peer.tcp->descriptor()); fds.push_back(peer.udp->descriptor()); }
+    for (const auto& peer : extras) fds.push_back(peer->descriptor());
+    if (proxy) fds.push_back(proxy->descriptor());
+    server.shutdown(); for (auto& peer : peers) { peer.tcp->close(); peer.udp->close(); } for (auto& peer : extras) peer->close();
+    if (proxy) proxy->close();
+    for (const auto fd : fds) need(descriptor_closed(fd),"actual UDP/TCP/reactor descriptor release");
+    auto cleanup = server.cleanup(); for (const auto& [key,value] : cleanup.items()) { (void)key; need(value == 0,"active owner/credential/transport cleanup"); }
+    need(Fd::live() == descriptors_before,"tracked descriptor delta"); cleanup["descriptor_checks"] = fds.size();
+    cleanup["all_descriptors_closed"] = true; cleanup["tracked_descriptor_delta"] = Fd::live()-descriptors_before; return cleanup;
+  }
+};
+}
+Json run_udp_matrix() {
+  const std::array<std::string,11> names{"valid-token-before-deadline","expired-token-at-deadline","unknown-token",
+    "reused-consumed-token","other-session-token","INPUT-before-bind","INPUT-from-different-observed-endpoint",
+    "wrong-owner-epoch-bind","realtime-INPUT-over-TCP","valid-INPUT-exactly1200bytes","oversize-datagram1201bytes"};
+  Json rows = Json::array();
+  for (std::size_t index = 0; index < names.size(); ++index) {
+    const bool bound = index == 3 || index == 6 || index >= 8;
+    Fixture f(index == 4 || index == 5 || index >= 6 ? 2 : 1,bound); auto& peer = f.peers[0];
+    const auto before = state(f.server.room()); const auto tokens = f.server.metrics().at("udp_bind_tokens");
+    Json row{{"name",names[index]},{"before",before},{"endpoint_alias","original"},{"token_value_recorded",false}};
+    std::string code = "UDP_BIND_INVALID";
+    if (index <= 2 || index == 7) {
+      f.now = index == 0 ? 4999 : index == 1 ? 5000 : 0;
+      auto bind = peer.tcp->bind_request(peer.session,index == 7 ? 1 : 0);
+      if (index == 2) bind["udp_bind_token"] = "unknown-token";
+      peer.udp->send(bind);
+      if (index == 0) { need(peer.udp->receive_type(f.server,"UDP_BOUND").at("session_id") == peer.session,"4999 valid bind"); code = "UDP_BOUND"; }
+      else { f.error(0,code); need(f.server.metrics().at("udp_bind_tokens") == tokens,"failed bind never consumes a credential"); }
+      row["manual_ms_after_issue"] = f.now;
+    } else if (index == 3) {
+      f.extra().send(peer.tcp->bind_request(peer.session)); f.error(0,code); f.ping();
+      need(f.server.metrics().at("udp_bound_endpoints") == 1,"consumed token cannot move endpoint"); row["attempt_endpoint_alias"] = "alternate";
+    } else if (index == 4) {
+      f.peers[1].udp->send(peer.tcp->bind_request(f.peers[1].session)); f.error(1,code);
+      need(f.server.metrics().at("udp_bind_tokens") == tokens,"other-session token not consumed");
+      row["after_rejection"] = state(f.server.room());
+      need(row.at("after_rejection") == before,"other-session bind cannot mutate either player");
+      f.bind_all(); row["both_original_tokens_still_usable"] = true;
+    } else if (index == 5 || index == 6) {
+      if (index == 5) peer.udp->send(f.input()); else f.extra().send(f.input());
+      f.error(0,code); if (index == 6) f.ping();
+    } else if (index == 8) {
+      code = "WRONG_TRANSPORT"; peer.tcp->send(f.input()); f.error(0,code);
+    } else {
+      auto input = f.input(); input["padding"] = ""; const auto target = index == 9 ? 1200U : 1201U;
+      input["padding"] = std::string(target-input.dump().size(),'x'); const auto text = input.dump();
+      need(text.size() == target,"exact fixed UDP byte boundary");
+      const auto received = f.server.metrics().at("udp_received_datagrams").get<std::uint64_t>();
+      const auto sent = f.server.metrics().at("udp_sent_datagrams");
+      peer.udp->send_bytes(std::span(reinterpret_cast<const std::uint8_t*>(text.data()),text.size()));
+      if (index == 9) {
+        const auto result = peer.udp->receive_type(f.server,"INPUT_ACK"); code = result.at("code").get<std::string>();
+        need(code == "ACCEPTED" && f.server.room().players().at(peer.player).last_accepted_seq() == 1 && f.server.room().pending_count() == 1,
+          "exact1200 valid INPUT uses real admission");
+      } else {
+        const auto deadline = std::chrono::steady_clock::now()+std::chrono::seconds(5);
+        while (f.server.metrics().at("udp_received_datagrams") == received && std::chrono::steady_clock::now() < deadline) f.server.poll_io(1);
+        f.server.drain_mailbox();
+        need(f.server.metrics().at("udp_received_datagrams") == received+1 &&
+          f.server.metrics().at("errors").at("UDP_DATAGRAM_SIZE_INVALID") == 1 && f.server.metrics().at("udp_sent_datagrams") == sent &&
+          !peer.udp->try_receive() && !peer.tcp->try_receive(),"processed1201 drop metric and no response, not timeout inference");
+        code = "DROP_OVERSIZE";
+      }
+      row["bytes"] = target;
+    }
+    const auto after = state(f.server.room());
+    if (index != 4 && index != 9) need(before == after,"failed binding/transport cannot change authoritative state or sequence");
+    need(f.server.room().executed_ticks() == 0 && f.clock.now_ms == 0,"matrix never executes simulation");
+    row["code"] = code; row["after"] = after; row["metrics"] = f.server.metrics(); row["cleanup"] = f.finish(); rows.push_back(std::move(row));
+  }
+  Fixture f(2,true); auto outbound = UdpFixture::outbound_probe(f.server,f.peers[0].player);
+  need(!f.peers[0].udp->try_receive(),"oversized outbound never reaches client"); outbound["cleanup"] = f.finish();
+  return Json{{"result","PASS"},{"cases",rows},{"outbound_bound_probe",outbound},{"case_count",11},{"executed_ticks",0},
+    {"fault_runs",0},{"all_resources_released",Fd::live() == 0}};
+}
+
+ReplayRun run_udp_scenario(const Json& scenario) {
+  need(scenario.at("thread") == "G09" && scenario.at("contract_version") == 1 && scenario.at("seed") == 7050 &&
+    scenario.at("ticks") == 24 && scenario.at("players").size() == 4 && scenario.at("events").size() == 24 &&
+    scenario.at("clock").at("kind") == "manual" && scenario.at("clock").at("tick_duration_ms") == tick_duration_ms &&
+    scenario.at("udp_bind_token_ttl_ms") == udp_bind_token_ttl_ms && scenario.at("datagram_max_bytes") == max_datagram_bytes &&
+    scenario.at("socket_ceiling_ms") == 5000 && scenario.at("faults").at("drop_indices") == Json::array({5}) &&
+    scenario.at("faults").at("duplicate_indices") == Json::array({8}) &&
+    scenario.at("faults").at("swap_adjacent_indices") == Json::array({Json::array({10,11})}),"frozen24-tick dimensions and faults");
+  Fixture f(4,false,&scenario,true); auto& proxy = *f.proxy;
+  const auto wait_for = [&](const auto& predicate, const std::string& description) {
+    const auto deadline = std::chrono::steady_clock::now()+std::chrono::seconds(5);
+    do {
+      proxy.pump(); f.server.pump(); proxy.pump();
+      if (predicate()) return;
+    } while (std::chrono::steady_clock::now() < deadline);
+    throw std::runtime_error("G09 owner/socket barrier: "+description);
+  };
+  Json bindings = Json::array();
+  for (std::size_t i = 0; i < f.peers.size(); ++i) {
+    auto& peer = f.peers[i]; const auto& fixed = scenario.at("players").at(i);
+    need(fixed.at("player_id") == peer.player && scenario.at("room_id") == f.room,"test IDs enter only normal server allocation");
+    peer.udp->send(peer.tcp->bind_request(peer.session)); std::optional<Json> bound;
+    wait_for([&] { bound = peer.udp->try_receive(); return bound.has_value(); },"actual UDP_BOUND");
+    need(bound->at("type") == "UDP_BOUND" && bound->at("session_id") == peer.session && bound->at("owner_epoch") == 0 &&
+      f.server.room().status() == (i == 3 ? "RUNNING" : "LOBBY") && f.server.room().executed_ticks() == 0,
+      "ordinary four joins remain LOBBY until all four bind, with unchanged min2");
+    const auto& player = f.server.room().players().at(peer.player);
+    need(player.slot == fixed.at("slot") && player.x == fixed.at("spawn").at(0) && player.y == fixed.at("spawn").at(1) &&
+      player.realtime_ready,"normal join spawn and actual binding readiness");
+    bindings.push_back(Json{{"client",fixed.at("client")},{"player_id",peer.player},{"slot",player.slot},
+      {"session_matches_bound",true},{"token_present_in_WELCOME",peer.tcp->has_bind_token()},{"token_value_recorded",false},
+      {"endpoint_alias",i == 0 ? "alpha_proxy" : fixed.at("client").get<std::string>()},
+      {"room_status_after_bind",f.server.room().status()},{"tcp_descriptor",peer.tcp->descriptor()},{"udp_descriptor",peer.udp->descriptor()}});
+  }
+  need(f.server.metrics().at("udp_bind_tokens") == 0 && f.server.metrics().at("udp_bound_endpoints") == 4,
+    "all four one-time credentials consumed only by successful binds");
+  const auto initial = state(f.server.room());
+  const auto visible = [&] {
+    Json rows = Json::array();
+    for (const auto& [id,p] : f.server.room().players()) if (p.connected)
+      rows.push_back(Json{{"player_id",id},{"slot",p.slot},{"x",p.x},{"y",p.y},
+        {"direction",direction_name(p.direction)},{"score",p.score},{"connectivity","CONNECTED"}});
+    return Json{{"room_id",f.server.room().id()},{"tick",f.server.room().executed_ticks()-1},
+      {"owner_epoch",0},{"status",f.server.room().status()},{"players",rows}};
+  };
+  struct Replica {
+    std::uint64_t latest = 0;
+    std::map<std::uint64_t,Json> retained;
+    Json applied, received = Json::array(), feedback = Json::array();
+  };
+  std::array<Replica,4> replicas;
+  std::map<std::uint64_t,Json> authority;
+  std::map<std::uint64_t,std::string> hash_at;
+  Json captures = Json::array(), feedback = Json::array();
+  const auto capture = [&](int publication) {
+    const auto before = state(f.server.room()); const auto canonical = canonical_state(f.server.room());
+    authority[static_cast<std::uint64_t>(publication)] = visible(); hash_at[static_cast<std::uint64_t>(publication)] = sha256(canonical);
+    const auto validate = [&](const Json& wire) {
+      const auto sequence = wire.at("snapshot_seq").get<std::uint64_t>(); const bool full = sequence == 1;
+      need(wire.at("v") == 1 && wire.at("type") == "SNAPSHOT" && wire.at("owner_epoch") == 0 &&
+        wire.at("room_id") == f.room && wire.at("tick") == static_cast<int>(sequence)*2-3 &&
+        wire.at("state_hash") == hash_at.at(sequence) && wire.at("kind") == (full ? "FULL" : "DELTA") &&
+        wire.size() == (full ? 12U : 11U) && wire.at("removed_player_ids").empty() &&
+        wire.at("base_snapshot_seq").is_null() == full,"actual snapshot cadence, bounded shape and canonical hash");
+      std::string previous;
+      for (const auto& player : wire.at("players")) {
+        const auto id = player.at("player_id").get<std::string>();
+        need(id > previous && player.size() == 7 && player.contains("slot") && player.contains("x") && player.contains("y") &&
+          player.contains("direction") && player.contains("score") && player.at("connectivity") == "CONNECTED",
+          "sorted exact seven-field wire projection"); previous = id;
+      }
+    };
+    wait_for([&] { return proxy.originals(1) == publication; },"original alpha snapshot observed before fault");
+    const auto original = proxy.snapshots().at(static_cast<std::size_t>(publication-1)); validate(original);
+    need(original.at("base_snapshot_seq") == (publication == 1 ? Json(nullptr) : Json(replicas[0].latest)),
+      "alpha base is last actually applied ACK, including loss and held packet");
+    std::array<std::size_t,4> remaining{proxy.forwarded(1)-replicas[0].received.size(),1,1,1};
+    wait_for([&] {
+      for (std::size_t i = 0; i < f.peers.size(); ++i) {
+        auto& peer = f.peers[i]; auto& replica = replicas[i];
+        need(!peer.tcp->try_receive(),"snapshot feedback has no unexpected TCP control");
+        while (auto wire = peer.udp->try_receive()) {
+          need(remaining[i] > 0,"no extra or unscheduled snapshot datagram"); --remaining[i]; validate(*wire);
+          const auto sequence = wire->at("snapshot_seq").get<std::uint64_t>();
+          if (i == 0) need(sequence == proxy.deliveries(1).at(replica.received.size()),"actual client delivery matches successful proxy sends");
+          else need(sequence == static_cast<std::uint64_t>(publication),"unaffected client snapshot sequence");
+          replica.received.push_back(sequence);
+          if (sequence <= replica.latest) {
+            replica.feedback.push_back(Json{{"snapshot_seq",sequence},{"applied",false},{"ack_sent",false},
+              {"ignored_reason",sequence == replica.latest ? "DUPLICATE" : "OLD"}}); continue;
+          }
+          std::map<std::string,Json> players; std::string status;
+          if (wire->at("kind") == "FULL") status = wire->at("status").get<std::string>();
+          else {
+            const auto base = wire->at("base_snapshot_seq").get<std::uint64_t>();
+            need(replica.retained.contains(base) && base == replica.latest,"delta uses actual owned ACK base");
+            const auto& previous = replica.retained.at(base); status = previous.at("status").get<std::string>();
+            for (const auto& player : previous.at("players")) players.emplace(player.at("player_id").get<std::string>(),player);
+          }
+          for (const auto& player : wire->at("players")) {
+            const auto id = player.at("player_id").get<std::string>();
+            need(wire->at("kind") == "FULL" || !players.contains(id) || players.at(id) != player,"delta excludes unchanged player rows");
+            players[id] = player;
+          }
+          Json rows = Json::array(); for (const auto& [id,player] : players) { (void)id; rows.push_back(player); }
+          replica.applied = Json{{"room_id",f.room},{"tick",wire->at("tick")},{"owner_epoch",0},{"status",status},{"players",rows}};
+          need(replica.applied == authority.at(sequence),"actual client delta application equals separately observed authoritative projection");
+          if (replica.retained.size() == snapshot_retention) replica.retained.erase(replica.retained.begin());
+          replica.retained.emplace(sequence,replica.applied); replica.latest = sequence;
+          auto ack = message("SNAPSHOT_ACK"); ack.update(Json{{"session_id",peer.session},{"room_id",f.room},
+            {"player_id",peer.player},{"snapshot_seq",sequence},{"owner_epoch",0}}); peer.udp->send(ack);
+          replica.feedback.push_back(Json{{"snapshot_seq",sequence},{"applied",true},{"ack_sent",true}});
+        }
+      }
+      return std::all_of(remaining.begin(),remaining.end(),[](auto count) { return count == 0; });
+    },"all actual delivered snapshots read and applied or explicitly ignored");
+    const auto all_owned = [&] {
+      const auto streams = f.server.metrics().at("snapshot_streams");
+      for (std::size_t i = 0; i < f.peers.size(); ++i)
+        if (streams.at(f.peers[i].player).at("acknowledged_seq") != replicas[i].latest) return false;
+      return f.server.cleanup().at("mailbox_messages") == 0;
+    };
+    wait_for(all_owned,"real UDP ACK reaches owner before next publication");
+    need(state(f.server.room()) == before,"snapshot ACK never changes simulation/sequence");
+    const auto streams = f.server.metrics().at("snapshot_streams");
+    Json acks = Json::object();
+    for (std::size_t i = 0; i < f.peers.size(); ++i) {
+      const auto& stream = streams.at(f.peers[i].player);
+      need(stream.at("retained_count") == publication && stream.at("high_water") <= snapshot_retention,"real bounded materialization retention");
+      acks[scenario.at("players").at(i).at("client").get<std::string>()] = stream.at("acknowledged_seq");
+    }
+    feedback.push_back(Json{{"after_publication",publication},{"owner_acknowledged",acks}});
+    captures.push_back(Json{{"snapshot",original},{"canonical_record",canonical},{"authoritative_visible",authority.at(publication)},
+      {"alpha_applied_seq",replicas[0].latest},{"alpha_applied_visible",replicas[0].applied},{"retention",streams.at(f.peers[0].player)}});
+  };
+  capture(1);
+  ReplayRun run; run.records = Json::array(); Json hashes = Json::array(), events = Json::array(), accepted = Json::array(), duplicates = Json::array();
+  for (int tick = 0; tick < 24; ++tick) {
+    const auto& event = scenario.at("events").at(static_cast<std::size_t>(tick)); const int sequence = tick+1;
+    need(event.at("before_tick") == tick && event.at("target_tick") == tick && event.at("seq") == sequence &&
+      event.at("client") == "alpha" && event.at("direction") == "EAST" && event.at("tag_target_role").is_null() &&
+      event.at("owner_epoch") == 0,"each frozen event at original manual boundary");
+    auto input = f.input(); for (const auto* key : {"seq","target_tick","direction","owner_epoch"}) input[key] = event.at(key);
+    const auto before = state(f.server.room()); const auto forwarded_before = proxy.forwarded(0);
+    const auto ingress_before = f.server.metrics().at("udp_received_datagrams").get<std::uint64_t>();
+    f.peers[0].udp->send(input);
+    wait_for([&] { return proxy.originals(0) == sequence; },"original INPUT reaches fault proxy");
+    const auto forwarded = proxy.forwarded(0)-forwarded_before;
+    wait_for([&] { return f.server.metrics().at("udp_received_datagrams") == ingress_before+forwarded &&
+      f.server.cleanup().at("mailbox_messages") == 0; },"all successfully forwarded inputs consumed by real owner");
+    Json acks = Json::array(), errors = Json::array();
+    wait_for([&] {
+      while (auto reply = f.peers[0].udp->try_receive()) { need(reply->at("type") == "INPUT_ACK","input response uses UDP"); acks.push_back(*reply); }
+      while (auto reply = f.peers[0].tcp->try_receive()) { need(reply->at("type") == "ERROR","rejected admission uses TCP"); errors.push_back(*reply); }
+      need(acks.size()+errors.size() <= forwarded,"one response per actually forwarded INPUT");
+      return acks.size()+errors.size() == forwarded;
+    },"observed input ACK/errors for actual delivered packets");
+    const auto expected_forwards = sequence == 5 || sequence == 10 ? 0U : sequence == 8 || sequence == 11 ? 2U : 1U;
+    need(forwarded == expected_forwards && errors.size() == (sequence == 11 ? 1U : 0U),"frozen input fault cardinality");
+    for (const auto& ack : acks) {
+      need(ack.at("player_id") == f.peers[0].player && ack.at("seq") == sequence && ack.at("last_accepted_seq") == sequence &&
+        ack.at("accepted") == true && ack.at("tick") == tick,"actual accepted sequence and admission boundary");
+      if (ack.at("code") == "ACCEPTED") accepted.push_back(ack.at("seq"));
+      else { need(sequence == 8 && ack.at("code") == "DUPLICATE","only fixed identical duplicate"); duplicates.push_back(ack.at("seq")); }
+    }
+    if (!errors.empty()) need(errors.at(0).at("code") == "INPUT_STALE","held10 rejects after11, sequence precedence preserved");
+    need(f.clock.now_ms == tick*50 && f.server.room().executed_ticks() == tick,"manual tick unchanged by network/owner barriers");
+    Json delivered = Json::array(); for (std::size_t i = forwarded_before; i < proxy.forwarded(0); ++i) delivered.push_back(proxy.deliveries(0).at(i));
+    events.push_back(Json{{"before_tick",tick},{"original_seq",sequence},{"delivered_sequences",delivered},
+      {"acks",acks},{"errors",errors},{"before",before},{"after_admission",state(f.server.room())},
+      {"udp_ingress_delta",f.server.metrics().at("udp_received_datagrams").get<std::uint64_t>()-ingress_before}});
+    f.server.advance_one_tick(); f.now = f.clock.now_ms;
+    const auto& alpha = f.server.room().players().at("player-00");
+    need(f.server.room().executed_ticks() == tick+1 && f.clock.now_ms == (tick+1)*50 && f.server.room().status() == "RUNNING" &&
+      session_ticks == 1200 && alpha.x == 10000+400*(tick+1) && alpha.y == 10000 && alpha.direction == Direction::east &&
+      alpha.score == 0 && alpha.last_tag_tick == -20 && alpha.last_accepted_seq() == static_cast<std::uint64_t>(sequence == 5 || sequence == 10 ? sequence-1 : sequence),
+      "fixed movement persists through loss while last_seq reflects actual admissions");
+    const auto canonical = canonical_state(f.server.room()), hash = sha256(canonical);
+    need(f.server.replay().last_state_hash() == hash,"production replay hashes actual authority each tick");
+    run.records.push_back(Json{{"tick",tick},{"canonical_record",canonical},{"state_hash",hash},{"state",state(f.server.room())}}); hashes.push_back(hash);
+    if ((tick+1)%2 == 0) capture((tick+1)/2+1);
+  }
+  Json expected = Json::array(); for (int seq = 1; seq <= 24; ++seq) if (seq != 5 && seq != 10) expected.push_back(seq);
+  const auto received = Json::array({1,2,3,4,6,7,8,8,9,11,10,12,13});
+  need(accepted == expected && duplicates == Json::array({8}) && proxy.originals(0) == 24 && proxy.originals(1) == 13 &&
+    proxy.deliveries(1) == received && replicas[0].received == received && proxy.evidence().at("held_packets") == 0,
+    "exact22 accepted sequences, one duplicate and frozen snapshot delivery");
+  run.replay_bytes = f.server.replay().serialize(); const auto artifact = parse_replay_artifact(run.replay_bytes);
+  Json journal = Json::array(); for (const auto& tick : artifact.at("ticks")) for (const auto& event : tick.at("events")) {
+    need(event.at("kind") == "INPUT" && event.at("player_id") == "player-00" && event.at("before_tick") == event.at("target_tick"),
+      "replay records actual accepted input admission only"); journal.push_back(event.at("seq"));
+  }
+  need(journal == accepted && artifact.at("executed_ticks") == 24 && f.server.room().pending_count() == 0,"actual journal22 at closed24-tick export boundary");
+  Json final_applied = Json::object(), client_feedback = Json::object(), positions = Json::object(), scores = Json::object(), last_seq = Json::object();
+  for (std::size_t i = 0; i < f.peers.size(); ++i) {
+    const auto role = scenario.at("players").at(i).at("client").get<std::string>(); const auto& p = f.server.room().players().at(f.peers[i].player);
+    need(p.connected && p.pending.empty() && p.score == 0 && p.last_tag_tick == -20 &&
+      (i == 0 ? p.x == 19600 && p.y == 10000 && p.last_accepted_seq() == 24 :
+        p.x == scenario.at("players").at(i).at("spawn").at(0) && p.y == scenario.at("players").at(i).at("spawn").at(1) &&
+        p.direction == Direction::stop && p.last_accepted_seq() == 0),"independently observed fixed final player state");
+    need(replicas[i].latest == 13 && replicas[i].applied == visible() && replicas[i].received.size() == 13,"all clients converge on final visible state");
+    final_applied[role] = replicas[i].applied; client_feedback[role] = replicas[i].feedback;
+    positions[role] = Json::array({p.x,p.y}); scores[role] = p.score; last_seq[role] = p.last_accepted_seq();
+    need(!f.peers[i].udp->try_receive() && !f.peers[i].tcp->try_receive(),"no extra wire output after fixed campaign");
+  }
+  const auto final = state(f.server.room()), metrics = f.server.metrics(), faults = proxy.evidence();
+  need(metrics.at("errors").size() == 1 && metrics.at("errors").at("INPUT_STALE") == 1 &&
+    metrics.at("snapshot_retention_high_water") == 13 && metrics.at("udp_outbound_high_water") <= max_datagram_bytes,
+    "only expected stale error; bounded snapshot/UDP resources");
+  auto cleanup = f.finish(); cleanup["proxy_held_packets"] = proxy.evidence().at("held_packets");
+  run.evidence = Json{{"result","PASS"},{"mode","live"},{"thread","G09"},{"scenario_id",scenario.at("scenario_id")},
+    {"process_id",::getpid()},{"executed_ticks",24},{"normal_joins",f.joins},{"initial_bindings",bindings},
+    {"initial_state",initial},{"events",events},{"faults",faults},{"captures",captures},{"ack_feedback",feedback},
+    {"client_feedback",client_feedback},{"final_applied",final_applied},{"state_hashes",hashes},{"final_state",final},
+    {"logical",Json{{"accepted_sequences",accepted},{"duplicate_sequences",duplicates},{"stale_sequences",Json::array({10})},
+      {"snapshot_received_ordinals",received},{"final_positions",positions},{"scores",scores},{"last_seq",last_seq}}},
+    {"metrics",metrics},{"cleanup",cleanup},{"all_resources_released",true}};
+  return run;
+}
+}
diff --git a/tests/g09.hpp b/tests/g09.hpp
new file mode 100644
index 0000000..0f9b37c
--- /dev/null
+++ b/tests/g09.hpp
@@ -0,0 +1,6 @@
+#pragma once
+#include "g07.hpp"
+namespace arena {
+Json run_udp_matrix();
+ReplayRun run_udp_scenario(const Json& scenario);
+}
diff --git a/tests/scenario_main.cpp b/tests/scenario_main.cpp
index 65f4e17..6df326f 100644
--- a/tests/scenario_main.cpp
+++ b/tests/scenario_main.cpp
@@ -1,4 +1,5 @@
 #include "g07.hpp"
+#include "g09.hpp"
 #ifndef ARENA_TEST_FIXTURES
 #error Scenario fixture executable requires its separate test core
 #endif
@@ -30,7 +31,7 @@ int main(int argc, char** argv) {
       const bool variant = argc == 6 && std::string(argv[4]) == "--variant" && std::string(argv[5]) == "rejected-removed";
       if (argc != 4 && !variant) throw std::invalid_argument("unknown scenario variant");
       const auto scenario = arena::read_json_file(input);
-      if (scenario.at("thread") != "G07") {
+      if (scenario.at("thread") != "G07" && scenario.at("thread") != "G09") {
         if (variant) throw std::invalid_argument("variant is only active for G07");
         const auto evidence = scenario.at("thread") == "G08" ? arena::run_snapshot_scenario(scenario) : arena::run_scenario(scenario);
         arena::write_json_file(output,evidence);
@@ -38,7 +39,10 @@ int main(int argc, char** argv) {
           {"scenario_id",evidence.at("scenario_id")},{"evidence",output.string()},{"cleanup",evidence.at("cleanup")}}.dump() << '\n';
         return 0;
       }
-      run = arena::run_replay_scenario(scenario,variant);
+      if (scenario.at("thread") == "G09") {
+        if (variant) throw std::invalid_argument("variant is only active for G07");
+        run = arena::run_udp_scenario(scenario);
+      } else run = arena::run_replay_scenario(scenario,variant);
       const auto replay_path = sibling(output,".replay.json");
       if (replay_path.has_parent_path()) std::filesystem::create_directories(replay_path.parent_path());
       std::ofstream file(replay_path,std::ios::binary);
diff --git a/tests/tests.cpp b/tests/tests.cpp
index 6b61385..956afdb 100644
--- a/tests/tests.cpp
+++ b/tests/tests.cpp
@@ -1,4 +1,5 @@
 #include "scenario.hpp"
+#include "g09.hpp"
 #include <algorithm>
 #include <atomic>
 #include <cerrno>
@@ -30,7 +31,7 @@ void initialize_shared_victim_fixture(Room& room) {
   for (std::size_t index = 0; index < ids.size(); ++index) {
     Player player;
     player.id = ids[index]; player.session_id = "session-" + ids[index]; player.connection_id = index + 1;
-    player.slot = static_cast<int>(index); player.x = 50000; player.y = 50000;
+    player.slot = static_cast<int>(index); player.x = 50000; player.y = 50000; player.realtime_ready = true;
     const auto id = player.id;
     room.players_.emplace(id,std::move(player));
   }
@@ -41,10 +42,14 @@ void initialize_shared_victim_fixture(Room& room) {
 namespace {
 using namespace arena;
 void check(bool condition, const std::string& text) { if (!condition) throw std::runtime_error(text); }
+Player& ready_join(Room& room, std::string id, std::string session, std::uint64_t connection) {
+  auto& player = room.join(std::move(id),std::move(session),connection);
+  room.bind_realtime(connection); return player;
+}
 void populate(Room& room) {
   room.create("unit-room");
-  room.join("player-00", "session-00", 1);
-  room.join("player-01", "session-01", 2);
+  ready_join(room, "player-00", "session-00", 1);
+  ready_join(room, "player-01", "session-01", 2);
 }
 std::optional<std::string> submit_next(Room& room, const std::string& player_id, Intent intent) {
   // Supply newly active protocol fields at each original test boundary.
@@ -55,12 +60,12 @@ void lifecycle_and_duration() {
   Room room;
   room.create("unit-room");
   check(room.status() == "LOBBY", "create enters lobby");
-  const auto& first = room.join("player-00", "session-00", 1);
+  const auto& first = ready_join(room, "player-00", "session-00", 1);
   check(first.slot == 0 && first.x == 10000 && first.y == 10000 && first.last_tag_tick == -20, "first spawn constants");
   check(room.status() == "LOBBY", "one player does not start");
-  const auto& second = room.join("player-01", "session-01", 2);
+  const auto& second = ready_join(room, "player-01", "session-01", 2);
   check(second.slot == 1 && second.x == 90000 && second.y == 90000, "second spawn constants");
-  check(room.status() == "RUNNING", "two TCP-ready players start");
+  check(room.status() == "RUNNING", "two bound realtime-ready players start");
   for (int tick = 0; tick < session_ticks; ++tick) {
     room.tick();
     check(room.executed_ticks() == tick + 1, "exactly one authoritative tick");
@@ -88,8 +93,8 @@ void movement_is_integer_and_bounded() {
 void tag_uses_wide_distance_and_one_shot_intent() {
   Room room;
   room.create("unit-room");
-  auto& actor = room.join("player-00", "session-00", 1);
-  auto& target = room.join("player-01", "session-01", 2);
+  auto& actor = ready_join(room, "player-00", "session-00", 1);
+  auto& target = ready_join(room, "player-01", "session-01", 2);
   // Unit-only setup of exact arithmetic boundaries. The production network
   // never exposes position/score mutation to clients.
   actor.x = 0; actor.y = 0; target.x = 100000; target.y = 100000;
@@ -156,8 +161,8 @@ void room_permission_matrix_preserves_state() {
   for (const auto* state : {"LOBBY", "RUNNING", "FINISHED", "CLOSED"}) {
     Room room;
     room.create("unit-room");
-    room.join("player-00", "session-00", 1);
-    if (std::string(state) != "LOBBY") room.join("player-01", "session-01", 2);
+    ready_join(room, "player-00", "session-00", 1);
+    if (std::string(state) != "LOBBY") ready_join(room, "player-01", "session-01", 2);
     if (std::string(state) == "FINISHED") for (int tick = 0; tick < session_ticks; ++tick) room.tick();
     if (std::string(state) == "CLOSED") room.close();
     check(room.status() == state, "permission cell reached requested state");
@@ -167,7 +172,7 @@ void room_permission_matrix_preserves_state() {
     check(create_rejected && room.view() == before, "create cannot replace an existing room in any state");
     bool join_rejected = false;
     if (std::string(state) != "LOBBY") {
-      try { room.join("unused-player", "unused-session", 3); } catch (const std::logic_error&) { join_rejected = true; }
+      try { ready_join(room, "unused-player", "unused-session", 3); } catch (const std::logic_error&) { join_rejected = true; }
       check(join_rejected && room.view() == before, "new join outside LOBBY rejected without consuming a slot");
     }
     room.leave(1);
@@ -437,8 +442,8 @@ void four_fixed_movement_clamps() {
   Json rows = Json::array();
   for (const auto& item : cases) {
     Room room; room.create("clamp-room");
-    auto& actor = room.join("actor","actor-session",1);
-    room.join("target","target-session",2);
+    auto& actor = ready_join(room, "actor","actor-session",1);
+    ready_join(room, "target","target-session",2);
     actor.x = item.x; actor.y = item.y;
     const auto before = room.view();
     const auto result = admit_input(room,actor.id,unit_input(room,actor.id,1,item.direction));
@@ -455,8 +460,8 @@ void tag_range_and_membership_edges() {
   Json rows = Json::array();
   for (const auto distance : {2500,2501}) {
     Room room; room.create("range-room");
-    auto& actor = room.join("actor","actor-session",1);
-    auto& target = room.join("target","target-session",2);
+    auto& actor = ready_join(room, "actor","actor-session",1);
+    auto& target = ready_join(room, "target","target-session",2);
     actor.x = 50000; actor.y = 50000; target.x = 50000 + distance; target.y = 50000;
     const auto before = room.view();
     const auto result = admit_input(room,actor.id,unit_input(room,actor.id,1,"STOP",target.id));
@@ -473,8 +478,8 @@ void tag_range_and_membership_edges() {
   }
   for (const auto* name : {"actor LEFT","target LEFT","self-target","target in independent other-Room model"}) {
     Room room; room.create("edge-room");
-    auto& actor = room.join("actor","actor-session",1);
-    auto& target = room.join("target","target-session",2);
+    auto& actor = ready_join(room, "actor","actor-session",1);
+    auto& target = ready_join(room, "target","target-session",2);
     actor.x = target.x = 50000; actor.y = target.y = 50000;
     std::string target_id = target.id;
     std::unique_ptr<Room> other;
@@ -558,8 +563,8 @@ void canonical_hash_bytes_without_ticks() {
   check(sha256("") == "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855", "SHA-256 empty vector");
   check(sha256("abc") == "ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad", "SHA-256 abc vector");
   Room room; room.create("hash-room");
-  auto& z = room.join("player-z","session-z",1);
-  auto& a = room.join("player-A","session-A",2);
+  auto& z = ready_join(room, "player-z","session-z",1);
+  auto& a = ready_join(room, "player-A","session-A",2);
   const auto maximum = std::numeric_limits<std::uint64_t>::max();
   check(!room.input(z.id,Input{maximum,std::uint64_t{0},{Direction::east,std::nullopt}}).error,
         "exact maximum accepted sequence for serialization");
@@ -668,8 +673,8 @@ void real_tcp_authority_and_cleanup() {
   check(evidence.at("players").at("alpha").at("score") == 0, "TCP client forged score ignored");
   check(evidence.at("players").at("bravo").at("x") == 89600, "second real TCP client moved independently");
   check(evidence.at("executed_ticks") == 3 && evidence.at("manual_clock_ms") == 150, "injected clock drove actual server");
-  check(evidence.at("cleanup").at("descriptor_checks") == 6 && evidence.at("cleanup").at("all_descriptors_closed") == true,
-        "listener, kqueue, two accepted and two client descriptors closed");
+  check(evidence.at("cleanup").at("descriptor_checks") == 9 && evidence.at("cleanup").at("all_descriptors_closed") == true,
+        "TCP, UDP and kqueue descriptors all closed");
   check(evidence.at("lifecycle") == Json::array({"LOBBY","RUNNING","CLOSED"}), "full authoritative lifecycle preserved");
   check(evidence.at("client_lifecycle").at("alpha") == Json::array({"LOBBY","RUNNING"}) &&
     evidence.at("client_lifecycle").at("bravo") == Json::array({"RUNNING"}), "actual create/join/start wire lifecycle");
@@ -802,7 +807,8 @@ int main(int argc, char** argv) {
     tests = {{"real_TCP_authority_and_cleanup", real_tcp_authority_and_cleanup}, {"standalone_process_shutdown", [&] {
       standalone_process_shutdown(std::filesystem::absolute(argv[0]).parent_path() / "arena"); }},
       {"G04_standalone_monotonic_adapter", [&] {
-      standalone_process_shutdown(std::filesystem::absolute(argv[0]).parent_path() / "arena", "monotonic"); }}};
+      standalone_process_shutdown(std::filesystem::absolute(argv[0]).parent_path() / "arena", "monotonic"); }},
+      {"G09_fixed_UDP_binding_size_matrix", [] { std::cout << Json{{"G09_matrix",run_udp_matrix()}}.dump() << '\n'; }}};
   } else { std::cerr << "unknown suite\n"; return 2; }
   std::size_t passed = 0;
   for (const auto& [name, test] : tests) {
