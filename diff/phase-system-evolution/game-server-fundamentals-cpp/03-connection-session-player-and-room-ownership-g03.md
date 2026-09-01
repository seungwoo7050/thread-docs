# Connection, Session, Player와 Room Ownership (G03)

## `test(g03): verify identity lifecycle and mailbox ownership`

diff --git a/CMakeLists.txt b/CMakeLists.txt
index 2b35fc8..f2b8be3 100644
--- a/CMakeLists.txt
+++ b/CMakeLists.txt
@@ -18,7 +18,7 @@ if(ARENA_TSAN)
   add_link_options(-fsanitize=thread)
 endif()
 find_package(nlohmann_json 3.12.0 EXACT CONFIG REQUIRED)
-add_library(arena_core src/game.cpp src/transport.cpp src/scenario.cpp)
+add_library(arena_core src/game.cpp src/transport.cpp src/scenario.cpp src/lifecycle_scenario.cpp)
 target_include_directories(arena_core PUBLIC src)
 target_link_libraries(arena_core PUBLIC nlohmann_json::nlohmann_json)
 target_compile_options(arena_core PRIVATE -Wall -Wextra -Wpedantic -Werror)
diff --git a/TRACK.md b/TRACK.md
index 1d922a9..a752ce9 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -1,4 +1,4 @@
-# fundamentals-cpp — G02 bounded TCP framing
+# fundamentals-cpp — G03 identity and lifecycle ownership
 
 SPEC_REVISION: `5a6e4a2f8fc71d4be18c3279583bfc2558d5c232`
 
@@ -26,6 +26,7 @@ Run from this worktree; `track` resolves its own source directory. Build never r
 ./track integration-test
 ./track scenario-run /absolute/path/to/G01.json /absolute/path/to/evidence.json
 ./track scenario-run /absolute/path/to/G02.json /absolute/path/to/evidence.json
+./track scenario-run /absolute/path/to/G03.json /absolute/path/to/evidence.json
 ./track replay-verify /absolute/path/to/replay.json /absolute/path/to/evidence.json
 ./track server /absolute/path/to/config.json
 ```
@@ -122,4 +123,22 @@ Pure unit cases also validate the frozen supplemental protocol list, an exact 16
 
 G02 initial ceilings: 8 configure/compile invocations, 4 unit invocations including reproduction, 2 integration suites, at most 2 canonical runs (only one post-change run when reproduction is pure unit), zero network-fault/load runs. The actual G02 command ledger, retained failures and results are linked from `evidence/G02.md`. Builds/tests use the unchanged frozen toolchain and ordinary `ARENA_TSAN=ON` configuration. Each `./track build` consumes two conservative configure/compile units.
 
-Detailed identity/lifecycle matrices (G03), clock accumulator/catch-up (G04), input sequence/target tick (G05), abuse matrix (G06), replay/hash (G07), full/delta cadence (G08), UDP, reconnect, slow-consumer coalescing and many-room scheduling remain inactive.
+## G03 identity and lifecycle evidence
+
+The unchanged G02 implementation passed the fixed identity, six lifecycle cells and held-consumer owner reproduction. G03 adds focused regressions and a canonical runner, not a new identity system. The existing deque admission/take path is extracted into the private `Server::Mailbox` solely so a pure test can exercise the real 512 bound with exactly 513 attempts. Capacity, per-connection 64 bound, `INPUT_QUEUE_FULL`, FIFO order and owner dispatch remain unchanged.
+
+Connection generation identifies transport lifetime; HELLO creates its distinct opaque session identifier. A successful join commits a separate player identifier and stable slot. Room identifiers are separate opaque values. No resume credential is active. Network parsing/admission cannot mutate Room; `drain_mailbox` and explicit ticks belong to its recorded owner thread. The existing foreign-thread rejection test remains active.
+
+| Room state | CREATE_ROOM | JOIN_ROOM | Valid owner's LEAVE_ROOM / connection close |
+|---|---|---|---|
+| ABSENT | valid session creates LOBBY | ROOM_NOT_FOUND | no joined player |
+| LOBBY | ROOM_NOT_JOINABLE | valid new session and available slot accepted; duplicate rejected | player becomes LEFT |
+| RUNNING | ROOM_NOT_JOINABLE | ROOM_NOT_JOINABLE | player becomes LEFT; state/tick/other player preserved |
+| FINISHED | ROOM_NOT_JOINABLE | ROOM_NOT_JOINABLE | player becomes LEFT; final state/tick preserved |
+| CLOSED | no Room replacement | not joinable | terminal/idempotent cleanup |
+
+A nonexistent room reference is `ROOM_NOT_FOUND`; foreign session is `SESSION_INVALID`; an owned session with a foreign player is `PLAYER_MISMATCH`. Valid leave has no newly invented response/transport-close requirement. Explicit server shutdown releases all active connection/session, parser, mailbox and descriptor resources; bounded CLOSED player views remain for historical inspection.
+
+The G03 canonical runner reads the actual scenario path, uses real issued identifiers, holds FINISHED at exactly 1200 manual ticks, and asserts unchanged state across rejection/leave/close. The ownership probe uses `poll_io` without consumer release, then `drain_mailbox` without a tick, then one explicit 50ms tick. The pure mailbox probe uses the production mailbox type, no sockets or rooms, and drains all accepted envelopes. Evidence and the one command ledger are `evidence/G03.md` and `evidence/G03-runs.jsonl`.
+
+Clock accumulator/catch-up (G04), input sequence/target tick (G05), abuse matrix (G06), replay/hash (G07), full/delta cadence (G08), UDP, reconnect, slow-consumer coalescing and many-room scheduling remain inactive.
diff --git a/evidence/G03-runs.jsonl b/evidence/G03-runs.jsonl
new file mode 100644
index 0000000..b0da75e
--- /dev/null
+++ b/evidence/G03-runs.jsonl
@@ -0,0 +1,7 @@
+{"category":"inspection","units":0,"label":"core-before-baseline","argv":["git","diff","--","src/game.hpp","src/game.cpp","src/transport.hpp","src/transport.cpp"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","started_at":"2026-08-28T01:57:52.251502+00:00","duration_seconds":0.016067,"exit":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g03/core-before-baseline.log"}
+{"category":"build","units":1,"label":"reproduce-compile","argv":["/usr/bin/clang++","-std=c++20","-O2","-Wall","-Wextra","-Wpedantic","-Werror","-fsanitize=thread","-g","-I","src","-I","/opt/homebrew/include","src/game.cpp","src/transport.cpp","src/scenario.cpp","src/lifecycle_scenario.cpp","artifacts/g03/reproduce.cpp","-o","artifacts/g03/reproduce-g02"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","started_at":"2026-08-28T01:58:09.045491+00:00","duration_seconds":17.333043,"exit":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g03/reproduce-compile.log"}
+{"category":"unit","units":1,"label":"reproduce-g02","argv":["env","TSAN_OPTIONS=halt_on_error=1","artifacts/g03/reproduce-g02","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G03.json","artifacts/g03/reproduction.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","started_at":"2026-08-28T01:58:36.819751+00:00","duration_seconds":1.212492,"exit":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g03/reproduce-g02.log"}
+{"category":"build","units":2,"label":"tsan-build","argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g03-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g03/track","ARENA_TSAN=ON","./track","build"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","started_at":"2026-08-28T02:05:54.283619+00:00","duration_seconds":27.483441,"exit":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g03/tsan-build.log"}
+{"category":"unit","units":1,"label":"tsan-unit","argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g03-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g03/track","TSAN_OPTIONS=halt_on_error=1","./track","unit-test"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","started_at":"2026-08-28T02:07:10.349700+00:00","duration_seconds":1.207196,"exit":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g03/tsan-unit.log"}
+{"category":"integration","units":1,"label":"tsan-integration","argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g03-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g03/track","TSAN_OPTIONS=halt_on_error=1","./track","integration-test"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","started_at":"2026-08-28T02:07:33.624591+00:00","duration_seconds":1.215491,"exit":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g03/tsan-integration.log"}
+{"category":"canonical","units":1,"label":"tsan-canonical","argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g03-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g03/track","TSAN_OPTIONS=halt_on_error=1","./track","scenario-run","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G03.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g03/canonical.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","started_at":"2026-08-28T02:08:01.913587+00:00","duration_seconds":0.382231,"exit":0,"output":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g03/tsan-canonical.log"}
diff --git a/evidence/G03.md b/evidence/G03.md
new file mode 100644
index 0000000..0296e1d
--- /dev/null
+++ b/evidence/G03.md
@@ -0,0 +1,57 @@
+# G03 — identity, lifecycle and mutation ownership
+
+THREAD G03; BRANCH `track/fundamentals-cpp`; PROFILE `realtime-core`; ATTEMPT initial.
+SPEC_REVISION `5a6e4a2f8fc71d4be18c3279583bfc2558d5c232`.
+START `0702a00481384ef4a2983d9e62da00c8d07c1558`.
+
+Frozen input: `/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G03.json`;
+SHA-256 `d3cdc4dac5c0054847329dcf0b56b408ba5f30f95ca0e5f85a7da914fc3e0d62` (verified before work).
+
+## Reproduction command, fixed before execution
+
+Compile the new observation harness against unchanged G02 game/transport sources:
+
+```sh
+/usr/bin/clang++ -std=c++20 -O2 -Wall -Wextra -Wpedantic -Werror -fsanitize=thread -g -I src -I /opt/homebrew/include src/game.cpp src/transport.cpp src/scenario.cpp src/lifecycle_scenario.cpp artifacts/g03/reproduce.cpp -o artifacts/g03/reproduce-g02
+env TSAN_OPTIONS=halt_on_error=1 artifacts/g03/reproduce-g02 /Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G03.json artifacts/g03/reproduction.json
+```
+
+The second command is one unit-category reproduction invocation. It exercises real previous transport/parser/Room paths, not an unsupported G03 dispatch. Identity cases and six lifecycle cells use fresh servers and real issued identifiers; the owner probe holds the consumer by calling only `poll_io`, then explicitly drains without ticking. No core or header edits precede the main reproduction gate.
+
+The existing 512-message admission bound is source-observed before this run. Its exact pure 513-attempt saturation test awaits a minimal admission/test-access seam after main's gate acknowledgement; the missing seam is not a production defect.
+
+Ceilings: compile/configure 8; unit 4 including reproduction; integration 2; one post-change canonical (no pre-change canonical after unit reproduction); network fault 0; load 0. Exact commands, timestamps, exits, durations and raw output paths are recorded in `G03-runs.jsonl`.
+
+## Observed baseline
+
+The compile exited 0 (17.333043s). The unchanged-G02 reproduction exited 0 (1.212492s) with **NOT_REPRODUCED** for the exercised guarantees. Duplicate join returned `ROOM_NOT_JOINABLE` without consuming a slot; foreign-session/player inputs returned `SESSION_INVALID`/`PLAYER_MISMATCH` without changing state or pending input. All six independent lifecycle cells marked only alpha `LEFT`, preserving IDs, slots, positions, scores, tick and bravo. Both FINISHED cases observed `ROOM_FINISHED` at exactly 1200 manual ticks before their action.
+
+The held owner probe observed mailbox/pending counts 1/0 before release, 0/1 after drain with no tick, then x=10400/y=10000 and pending=0 after one 50ms tick. All 56 checked descriptors closed; all active cleanup counters and tracked descriptor delta were zero. TSan reported no error. The pure 513-admission probe remains unexecuted at this gate.
+
+Raw evidence: `artifacts/g03/reproduction.json` and `reproduce-g02.log`; the exact baseline harness is retained as `artifacts/g03/baseline-lifecycle_scenario.cpp`. `core-before-baseline.log` is an empty diff of game/transport sources. Budget consumed at gate: compile **1/8**, unit **1/4**, integration **0/2**, post canonical **0/1**, fault/load **0/0**.
+
+Main inspected the unchanged-core raw baseline and acknowledged the gate before the admission extraction. No new identity/lifecycle guarantee needed repair. The private production Mailbox now contains the same deque admission/take path, permitting the fixed pure test without constructing sockets/Rooms or duplicating its capacity policy. Existing tests are preserved; pure create/join/leave state and mailbox regressions are added.
+
+## Fixed post-gate verification commands
+
+```sh
+env ARENA_BUILD_DIR=$PWD/build-g03-tsan ARENA_EVIDENCE_DIR=$PWD/artifacts/g03/track ARENA_TSAN=ON ./track build
+env ARENA_BUILD_DIR=$PWD/build-g03-tsan ARENA_EVIDENCE_DIR=$PWD/artifacts/g03/track TSAN_OPTIONS=halt_on_error=1 ./track unit-test
+env ARENA_BUILD_DIR=$PWD/build-g03-tsan ARENA_EVIDENCE_DIR=$PWD/artifacts/g03/track TSAN_OPTIONS=halt_on_error=1 ./track integration-test
+env ARENA_BUILD_DIR=$PWD/build-g03-tsan ARENA_EVIDENCE_DIR=$PWD/artifacts/g03/track TSAN_OPTIONS=halt_on_error=1 ./track scenario-run /Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G03.json $PWD/artifacts/g03/canonical.json
+```
+
+## Final verification
+
+| Command | Exit | Observed result |
+|---|---:|---|
+| TSan configure/build | 0 | frozen compiler/dependency, warnings as errors; 27.483441s |
+| Complete unit suite | 0 | 11/11, including all nine prior tests; 1.207196s |
+| Complete integration suite | 0 | 2/2, real TCP authority plus SIGTERM/reaping/listener rebind; 1.215491s |
+| Actual frozen G03 canonical input | 0 | three identity cases, six lifecycle cells, owner and pure mailbox probes; 0.382231s |
+
+The canonical repeated the baseline identity/lifecycle/owner observations and checked 56 actual descriptor closures with all active cleanup counters zero. The pure production mailbox accepted 512 of exactly 513 attempts, rejected attempt 513 with `INPUT_QUEUE_FULL`, reached high-water 512, drained all 512 in order, and left all nine producer counters at zero. It allocated no transport or Room. TSan reported no error. No build or test execution failed or was retried.
+
+Final budget: compile/configure **3/8**, unit **2/4** including reproduction, integration **1/2**, post canonical **1/1**. Raw final evidence is `artifacts/g03/canonical.json` (SHA-256 `d90b3931682413eaf43e7e7dc1e4dd7207ccd558c9a6fbeb64a1971e0232cce5`); baseline JSON SHA-256 is `00a04877e2ca136cc6d0044afa10b81870ccd8b2f9deb7111128afd524d828a4`. The shared scenario checksum was rechecked unchanged after verification.
+
+STATE_HASHES inactive until G07; NETWORK_FAULT_RUNS 0; LOAD_RUNS 0. UNRESOLVED: no known G03 completion failure; independent main verification/comparison pending. No future-feature or dependency change. Execution SPEC_REVISION remains the assigned frozen SHA despite the separately authorized procedural main documentation update.
diff --git a/src/lifecycle_scenario.cpp b/src/lifecycle_scenario.cpp
new file mode 100644
index 0000000..7d0cfad
--- /dev/null
+++ b/src/lifecycle_scenario.cpp
@@ -0,0 +1,338 @@
+#include "scenario.hpp"
+#include <algorithm>
+#include <array>
+#include <chrono>
+#include <memory>
+#include <set>
+#include <stdexcept>
+
+namespace arena {
+namespace {
+void ensure(bool condition, const std::string& text) {
+  if (!condition) throw std::runtime_error("G03: " + text);
+}
+bool valid_issued_id(const std::string& value) {
+  return !value.empty() && value.size() <= 64 && std::all_of(value.begin(), value.end(), [](unsigned char ch) {
+    return (ch >= 'A' && ch <= 'Z') || (ch >= 'a' && ch <= 'z') ||
+           (ch >= '0' && ch <= '9') || ch == '_' || ch == '-';
+  });
+}
+struct Peer {
+  std::unique_ptr<TcpClient> tcp;
+  std::string session;
+  std::string player;
+};
+struct LifecycleFixture {
+  int descriptors_before = Fd::live();
+  ManualClock clock;
+  Server server{clock};
+  std::map<std::string, Peer> peers;
+  std::string room_id;
+  int ceiling;
+  std::size_t closed_client_checks = 0;
+
+  explicit LifecycleFixture(int socket_ceiling) : ceiling(socket_ceiling) {}
+  ~LifecycleFixture() {
+    try { server.shutdown(); } catch (...) { /* explicit checks report failures */ }
+  }
+  template<class Predicate> void wait_for(Predicate ready, bool consume = true) {
+    const auto deadline = std::chrono::steady_clock::now() + std::chrono::milliseconds(ceiling);
+    while (!ready() && std::chrono::steady_clock::now() < deadline) {
+      if (consume) server.pump(1); else server.poll_io(1);
+    }
+    ensure(ready(), "socket observation exceeded the frozen ceiling");
+  }
+  Json request(const std::string& role, const std::string& type) const {
+    const auto& peer = peers.at(role);
+    auto value = message(type);
+    value["session_id"] = peer.session;
+    if (type != "CREATE_ROOM") value["room_id"] = room_id;
+    if (type == "INPUT" || type == "LEAVE_ROOM") value["player_id"] = peer.player;
+    return value;
+  }
+  void hello(const std::string& role) {
+    auto& peer = peers[role];
+    ensure(!peer.tcp, "role was connected twice");
+    peer.tcp = std::make_unique<TcpClient>(server.port());
+    peer.tcp->send(message("HELLO"));
+    peer.session = peer.tcp->receive_type(server, "WELCOME").at("session_id").get<std::string>();
+    ensure(valid_issued_id(peer.session), "issued session identifier violates ASCII/length contract");
+  }
+  void create() {
+    auto& alpha = peers.at("alpha");
+    alpha.tcp->send(request("alpha", "CREATE_ROOM"));
+    const auto created = alpha.tcp->receive_type(server, "ROOM_CREATED");
+    room_id = created.at("room_id").get<std::string>();
+    ensure(valid_issued_id(room_id) && created.at("status") == "LOBBY", "create identity/state");
+  }
+  void join(const std::string& role, int expected_slot) {
+    auto& peer = peers.at(role);
+    peer.tcp->send(request(role, "JOIN_ROOM"));
+    const auto joined = peer.tcp->receive_type(server, "ROOM_JOINED");
+    peer.player = joined.at("player_id").get<std::string>();
+    const auto& player = server.room().players().at(peer.player);
+    ensure(valid_issued_id(peer.player) && player.slot == expected_slot && joined.at("slot") == expected_slot,
+           "stable join commit slot and identifier");
+    ensure(player.x == (expected_slot == 0 ? 10000 : 90000) &&
+           player.y == (expected_slot == 0 ? 10000 : 90000), "stable spawn coordinates");
+    ensure(player.session_id == peer.session && player.connection_id > 0 &&
+           peer.session != peer.player && peer.player != room_id && peer.session != room_id,
+           "transport/session/player/room identity separation");
+  }
+  void setup(int players) {
+    ensure(players == 1 || players == 2, "frozen player count changed");
+    hello("alpha"); create(); join("alpha", 0);
+    if (players == 2) { hello("bravo"); join("bravo", 1); }
+    ensure(server.room().status() == (players == 1 ? "LOBBY" : "RUNNING"), "setup lifecycle");
+    ensure(server.room().executed_ticks() == 0 && clock.now_ms == 0, "setup advanced simulation");
+  }
+  Json identities() const {
+    Json identities = Json::object();
+    std::set<std::string> issued{room_id};
+    std::set<std::uint64_t> connections;
+    for (const auto& [role, peer] : peers) {
+      const auto& player = server.room().players().at(peer.player);
+      ensure(issued.insert(peer.session).second && issued.insert(peer.player).second &&
+             connections.insert(player.connection_id).second, "issued identifiers were aliased");
+      identities[role] = Json{{"connection_id", player.connection_id}, {"session_id", peer.session},
+        {"player_id", peer.player}, {"slot", player.slot}, {"x", player.x}, {"y", player.y}};
+    }
+    return Json{{"room_id", room_id}, {"players", identities}};
+  }
+  Json rejected(const std::string& role, Json value, const std::string& code) {
+    const auto before = server.room().view();
+    const auto pending = server.room().pending_count();
+    auto& peer = peers.at(role);
+    peer.tcp->send(value);
+    const auto response = peer.tcp->receive_type(server, "ERROR");
+    ensure(response.at("code") == code, "identity/lifecycle rejection code");
+    ensure(server.room().view() == before && server.room().pending_count() == pending,
+           "rejected request changed authoritative state or pending intent");
+    return Json{{"request", value}, {"response", response}, {"unchanged_state", before}, {"pending_inputs", pending}};
+  }
+  void close_client(const std::string& role) {
+    auto& peer = peers.at(role);
+    const int fd = peer.tcp->descriptor();
+    peer.tcp->close();
+    ensure(descriptor_closed(fd), "closed client descriptor remains live");
+    ++closed_client_checks;
+  }
+  Json finish() {
+    auto descriptors = server.owned_descriptors();
+    for (auto& [role, peer] : peers) {
+      (void)role;
+      if (peer.tcp->descriptor() >= 0) descriptors.push_back(peer.tcp->descriptor());
+    }
+    server.shutdown();
+    for (auto& [role, peer] : peers) { (void)role; peer.tcp->close(); }
+    for (int fd : descriptors) ensure(descriptor_closed(fd), "server/client descriptor survived cleanup");
+    ensure(Fd::live() == descriptors_before, "RAII tracked descriptor leak");
+    ensure(server.room().status() == "CLOSED" && server.room().players().size() <= max_players,
+           "terminal room/historical view bound");
+    for (const auto& [id, player] : server.room().players()) {
+      (void)id;
+      ensure(!player.connected && player.pending.empty(), "terminal player remains active");
+    }
+    auto cleanup = server.cleanup();
+    for (const auto& [key, value] : cleanup.items()) { (void)key; ensure(value == 0, "owned resource survived shutdown"); }
+    cleanup["descriptor_checks"] = descriptors.size() + closed_client_checks;
+    cleanup["all_descriptors_closed"] = true;
+    cleanup["tracked_descriptor_delta"] = Fd::live() - descriptors_before;
+    cleanup["historical_players"] = server.room().players().size();
+    return cleanup;
+  }
+};
+Json lifecycle_cell(const Json& scenario, const std::string& state, const std::string& action) {
+  LifecycleFixture fixture(scenario.at("socket_ceiling_ms").get<int>());
+  const int count = scenario.at(state == "LOBBY" ? "lobby_player_count" :
+                                state == "RUNNING" ? "running_player_count" : "finished_player_count").get<int>();
+  fixture.setup(count);
+  Json finished = Json::object();
+  if (state == "FINISHED") {
+    for (int tick = 0; tick < scenario.at("finish_ticks").get<int>(); ++tick) fixture.server.advance_one_tick();
+    ensure(fixture.server.room().executed_ticks() == 1200 && fixture.clock.now_ms == 60000,
+           "FINISHED requires exactly 1200 manual ticks");
+    for (auto& [role, peer] : fixture.peers) {
+      auto response = peer.tcp->receive_type(fixture.server, "ROOM_FINISHED");
+      finished[role] = response;
+      response.erase("v"); response.erase("type");
+      ensure(response == fixture.server.room().view(), "ROOM_FINISHED differs from held authoritative state");
+    }
+  }
+  ensure(fixture.server.room().status() == state, "lifecycle cell not held in requested state");
+  const auto identities = fixture.identities();
+  const auto before = fixture.server.room().view();
+  const auto before_clock = fixture.clock.now_ms;
+  const auto actor_id = fixture.peers.at("alpha").player;
+  const auto descriptors_before_action = fixture.server.owned_descriptors();
+  if (action == "LEAVE_ROOM") {
+    fixture.peers.at("alpha").tcp->send(fixture.request("alpha", "LEAVE_ROOM"));
+  } else {
+    ensure(action == "CONNECTION_CLOSE", "unknown lifecycle action");
+    fixture.close_client("alpha");
+  }
+  fixture.wait_for([&] { return !fixture.server.room().players().at(actor_id).connected; });
+  const auto descriptors_after_action = fixture.server.owned_descriptors();
+  for (int fd : descriptors_before_action) {
+    if (std::find(descriptors_after_action.begin(), descriptors_after_action.end(), fd) == descriptors_after_action.end()) {
+      ensure(descriptor_closed(fd), "closed accepted descriptor remains live");
+      ++fixture.closed_client_checks;
+    }
+  }
+  auto expected = before;
+  for (auto& player : expected["players"]) if (player.at("player_id") == actor_id) player["connectivity"] = "LEFT";
+  const auto after = fixture.server.room().view();
+  ensure(after == expected && fixture.clock.now_ms == before_clock && fixture.server.room().pending_count() == 0,
+         "leave/close must only mark alpha LEFT and preserve identity, slot, state, tick and other player");
+  Json row{{"state", state}, {"action", action}, {"identities", identities}, {"before", before}, {"after", after},
+    {"finished_messages", finished}, {"manual_clock_ms", fixture.clock.now_ms}, {"pending_inputs", fixture.server.room().pending_count()},
+    {"cleanup", fixture.finish()}, {"result", "PASS"}};
+  return row;
+}
+}
+Json run_mailbox_probe(std::size_t capacity) {
+  ensure(capacity == max_mailbox_messages && capacity == 512, "pure mailbox capacity changed");
+  const int descriptors_before = Fd::live();
+  Server::Mailbox mailbox;
+  // Eight producer counters reach the unchanged per-connection limit. The
+  // final attempt uses a ninth counter so only the global 512 bound rejects it.
+  std::array<std::size_t, max_mailbox_messages / max_pending_inputs + 1> pending{};
+  std::size_t attempts = 0, accepted = 0, high_water = 0;
+  Json rejections = Json::array();
+  for (std::size_t index = 0; index <= capacity; ++index) {
+    const auto producer = index / max_pending_inputs;
+    Server::Envelope envelope{producer + 1, message("HELLO"), {}};
+    const auto error = mailbox.admit(std::move(envelope), pending.at(producer));
+    ++attempts;
+    if (index < capacity) {
+      ensure(!error, "mailbox rejected before capacity"); ++accepted;
+    } else {
+      ensure(error == "INPUT_QUEUE_FULL" && pending.at(producer) == 0,
+             "global mailbox overflow must reject without accepting or changing pending count");
+      rejections.push_back(Json{{"attempt", attempts}, {"code", *error}});
+    }
+    high_water = std::max(high_water, mailbox.size());
+    ensure(mailbox.size() <= capacity && pending.at(producer) <= max_pending_inputs, "mailbox/counter bound exceeded");
+  }
+  Json result{{"capacity", capacity}, {"admission_attempts", attempts}, {"accepted", accepted}, {"rejections", rejections},
+    {"high_water", high_water}, {"messages_before_release", mailbox.size()}, {"pending_requests_before_release", pending}};
+  std::size_t drained = 0;
+  while (mailbox.size() != 0) {
+    const auto envelope = mailbox.take();
+    const auto producer = drained / max_pending_inputs;
+    ensure(envelope.connection_id == producer + 1 && envelope.value == message("HELLO") && envelope.parser_error.empty(),
+           "production mailbox drain changed envelope order or content");
+    ensure(pending.at(producer) != 0, "mailbox consumer pending count underflow");
+    --pending.at(producer); ++drained;
+  }
+  ensure(drained == accepted && std::all_of(pending.begin(), pending.end(), [](auto value) { return value == 0; }),
+         "drained mailbox retained accepted work");
+  ensure(Fd::live() == descriptors_before, "pure mailbox test allocated a transport resource");
+  result["drained"] = drained; result["remaining_messages"] = mailbox.size(); result["pending_requests_after_drain"] = pending;
+  result["tracked_descriptor_delta"] = Fd::live() - descriptors_before; result["result"] = "PASS";
+  return result;
+}
+Json run_lifecycle_scenario(const Json& scenario) {
+  ensure(scenario.at("thread") == "G03" && scenario.at("contract_version") == 1 && scenario.at("seed") == 7050,
+         "scenario identity/version/seed changed");
+  ensure(scenario.at("clock").at("kind") == "manual" && scenario.at("clock").at("tick_duration_ms") == 50 &&
+         scenario.at("finish_ticks") == 1200 && scenario.at("socket_ceiling_ms") == 5000, "fixed clock or ceiling changed");
+  ensure(scenario.at("identity_cases").size() == 3 &&
+         scenario.at("lifecycle_states") == Json::array({"LOBBY", "RUNNING", "FINISHED"}) &&
+         scenario.at("lifecycle_actions") == Json::array({"LEAVE_ROOM", "CONNECTION_CLOSE"}), "fixed matrix changed");
+  Json evidence{{"scenario_id", scenario.at("scenario_id")}, {"thread", "G03"}, {"contract_version", 1},
+    {"transport", "production/real-loopback-TCP/kqueue/manual-owner-drain"},
+    {"identity_cases", Json::array()}, {"lifecycle_cases", Json::array()},
+    {"state_hashes", "INACTIVE_UNTIL_G07"}};
+  for (const auto& item : scenario.at("identity_cases")) {
+    LifecycleFixture fixture(scenario.at("socket_ceiling_ms").get<int>());
+    const auto name = item.at("name").get<std::string>();
+    ensure(item.at("clients") == Json::array({"alpha", "bravo"}), "identity roles changed");
+    Json row{{"name", name}};
+    if (name == "duplicate-join") {
+      ensure(item.at("steps") == Json::array({"alpha-hello-create-join", "alpha-join-again", "bravo-hello-join"}),
+             "duplicate join setup changed");
+      fixture.setup(1);
+      row["rejection"] = fixture.rejected("alpha", fixture.request("alpha", "JOIN_ROOM"), "ROOM_NOT_JOINABLE");
+      fixture.hello("bravo"); fixture.join("bravo", 1);
+      ensure(fixture.server.room().players().size() == 2 && fixture.server.room().status() == "RUNNING",
+             "duplicate join allocated a slot or prevented the next legitimate join");
+    } else {
+      ensure(name == "foreign-session" || name == "foreign-player", "unknown fixed identity case");
+      fixture.setup(2);
+      const auto sender = item.at("sender").get<std::string>();
+      auto request = fixture.request(sender, "INPUT");
+      request["session_id"] = fixture.peers.at(item.at("session_from").get<std::string>()).session;
+      request["player_id"] = fixture.peers.at(item.at("player_from").get<std::string>()).player;
+      request["direction"] = item.at("direction"); request["tag_target_player_id"] = nullptr;
+      row["rejection"] = fixture.rejected(sender, request, name == "foreign-session" ? "SESSION_INVALID" : "PLAYER_MISMATCH");
+    }
+    row["identities"] = fixture.identities();
+    row["after"] = fixture.server.room().view();
+    row["cleanup"] = fixture.finish(); row["result"] = "PASS";
+    evidence["identity_cases"].push_back(row);
+  }
+  for (const auto& state : scenario.at("lifecycle_states"))
+    for (const auto& action : scenario.at("lifecycle_actions"))
+      evidence["lifecycle_cases"].push_back(lifecycle_cell(scenario, state.get<std::string>(), action.get<std::string>()));
+  {
+    const auto& probe = scenario.at("owner_probe");
+    ensure(probe.at("clients") == Json::array({"alpha", "bravo"}) && probe.at("sender") == "alpha" &&
+           probe.at("direction") == "EAST" && probe.at("tag_target_player_id").is_null() &&
+           probe.at("target_state") == "RUNNING" && probe.at("before_tick") == 0 &&
+           probe.at("observe_before_consumer_release") == true && probe.at("drain_without_tick") == true &&
+           probe.at("execute_ticks_after_drain") == 1, "fixed owner probe changed");
+    LifecycleFixture fixture(scenario.at("socket_ceiling_ms").get<int>()); fixture.setup(2);
+    auto& alpha = fixture.peers.at("alpha");
+    const auto before = fixture.server.room().view();
+    auto input = fixture.request("alpha", "INPUT");
+    input["direction"] = probe.at("direction"); input["tag_target_player_id"] = probe.at("tag_target_player_id");
+    alpha.tcp->send(input);
+    fixture.wait_for([&] { return fixture.server.cleanup().at("mailbox_messages") == 1; }, false);
+    ensure(fixture.server.room().view() == before && fixture.server.room().pending_count() == 0 && fixture.clock.now_ms == 0,
+           "network callback mutated Room before owner release");
+    Json owner{{"before_release", fixture.server.room().view()}, {"mailbox_before_release", fixture.server.cleanup().at("mailbox_messages")},
+      {"pending_before_release", fixture.server.room().pending_count()}, {"input", input}};
+    fixture.server.drain_mailbox();
+    ensure(fixture.server.room().view() == before && fixture.server.room().pending_count() == 1 && fixture.clock.now_ms == 0 &&
+           fixture.server.cleanup().at("mailbox_messages") == 0, "owner drain must enqueue intent without simulating");
+    owner["after_drain"] = fixture.server.room().view(); owner["pending_after_drain"] = fixture.server.room().pending_count();
+    owner["mailbox_after_drain"] = fixture.server.cleanup().at("mailbox_messages");
+    const auto ack = alpha.tcp->receive_type(fixture.server, "INPUT_ACK");
+    ensure(ack.at("accepted") == true && ack.at("player_id") == alpha.player && ack.at("tick") == 0,
+           "owner accepted input acknowledgement");
+    fixture.server.advance_one_tick();
+    const auto& moved = fixture.server.room().players().at(alpha.player);
+    ensure(moved.x == 10400 && moved.y == 10000 && moved.direction == Direction::east && moved.score == 0 &&
+           fixture.server.room().executed_ticks() == 1 && fixture.clock.now_ms == 50 && fixture.server.room().pending_count() == 0,
+           "single manual tick did not apply owner intent exactly once");
+    auto expected_tick = before;
+    expected_tick["executed_ticks"] = 1; expected_tick["tick"] = 0;
+    for (auto& player : expected_tick["players"]) if (player.at("player_id") == alpha.player) {
+      player["x"] = 10400; player["direction"] = "EAST";
+    }
+    ensure(fixture.server.room().view() == expected_tick, "owner intent changed another player or unrelated state");
+    owner["ack"] = ack; owner["after_tick"] = fixture.server.room().view(); owner["manual_clock_ms"] = fixture.clock.now_ms;
+    owner["pending_after_tick"] = fixture.server.room().pending_count(); owner["cleanup"] = fixture.finish(); owner["result"] = "PASS";
+    evidence["owner_probe"] = owner;
+  }
+  ensure(scenario.at("mailbox_probe").at("capacity_by_track").at("fundamentals-cpp") == max_mailbox_messages,
+         "frozen mailbox capacity changed");
+  evidence["mailbox_probe"] = run_mailbox_probe(scenario.at("mailbox_probe").at("capacity_by_track").at("fundamentals-cpp").get<std::size_t>());
+  std::size_t descriptor_checks = 0;
+  int executed_ticks = 0;
+  for (const auto& row : evidence.at("identity_cases")) descriptor_checks += row.at("cleanup").at("descriptor_checks").get<std::size_t>();
+  for (const auto& row : evidence.at("lifecycle_cases")) {
+    descriptor_checks += row.at("cleanup").at("descriptor_checks").get<std::size_t>();
+    executed_ticks += row.at("after").at("executed_ticks").get<int>();
+  }
+  descriptor_checks += evidence.at("owner_probe").at("cleanup").at("descriptor_checks").get<std::size_t>();
+  executed_ticks += evidence.at("owner_probe").at("after_tick").at("executed_ticks").get<int>();
+  evidence["executed_ticks"] = executed_ticks;
+  evidence["cleanup"] = Json{{"descriptor_checks", descriptor_checks}, {"all_descriptors_closed", true},
+    {"live_tracked_descriptors", Fd::live()}, {"all_case_resource_counts_zero", true}};
+  ensure(Fd::live() == 0, "case resources leaked across independent fixtures");
+  evidence["result"] = "PASS";
+  return evidence;
+}
+}
diff --git a/src/scenario.cpp b/src/scenario.cpp
index ea8e94e..84ef195 100644
--- a/src/scenario.cpp
+++ b/src/scenario.cpp
@@ -282,6 +282,7 @@ Json run_framing_scenario(const Json& scenario) {
 }
 Json run_scenario(const Json& scenario) {
   if (scenario.at("thread") == "G02") return run_framing_scenario(scenario);
+  if (scenario.at("thread") == "G03") return run_lifecycle_scenario(scenario);
   require(scenario.at("contract_version") == 1 && scenario.at("thread") == "G01", "only G01 contract v1 is active");
   require(scenario.at("clock").at("kind") == "manual" && scenario.at("clock").at("tick_duration_ms") == tick_duration_ms,
           "G01 runner requires the fixed 50ms manual clock");
diff --git a/src/scenario.hpp b/src/scenario.hpp
index 21314f8..9d1ebc1 100644
--- a/src/scenario.hpp
+++ b/src/scenario.hpp
@@ -5,4 +5,6 @@ namespace arena {
 Json read_json_file(const std::filesystem::path& path, std::size_t limit = 1'048'576);
 void write_json_file(const std::filesystem::path& path, const Json& value);
 Json run_scenario(const Json& scenario);
+Json run_lifecycle_scenario(const Json& scenario);
+Json run_mailbox_probe(std::size_t capacity);
 }
diff --git a/src/transport.cpp b/src/transport.cpp
index 3eb23b8..523a217 100644
--- a/src/transport.cpp
+++ b/src/transport.cpp
@@ -237,6 +237,17 @@ void Server::end_transport(int fd, bool io_error) {
   if (!io_error && end.partial_frame) ++partial_eof_events_;
   disconnect(fd, io_error ? "TRANSPORT_IO_ERROR" : end.partial_frame ? end.code : "");
 }
+std::optional<std::string> Server::Mailbox::admit(Envelope envelope, std::size_t& pending_requests) {
+  if (entries_.size() == max_mailbox_messages || pending_requests == max_pending_inputs) return "INPUT_QUEUE_FULL";
+  ++pending_requests;
+  entries_.push_back(std::move(envelope));
+  return std::nullopt;
+}
+Server::Envelope Server::Mailbox::take() {
+  Envelope envelope = std::move(entries_.front());
+  entries_.pop_front();
+  return envelope;
+}
 void Server::read_ready(int fd) {
   auto found = connections_.find(fd);
   if (found == connections_.end()) return;
@@ -266,12 +277,12 @@ void Server::read_ready(int fd) {
     }
     if (parsed.state == ParseState::io_end) return;
     if (parsed.state == ParseState::message_error) ++message_error_events_;
-    if (mailbox_.size() == max_mailbox_messages || found->second.pending_requests == max_pending_inputs) {
-      queue(found->second.id, error_message("INPUT_QUEUE_FULL", "bounded transport mailbox is full"));
+    const auto error = mailbox_.admit({found->second.id, std::move(parsed.value), std::move(parsed.code)},
+                                      found->second.pending_requests);
+    if (error) {
+      queue(found->second.id, error_message(*error, "bounded transport mailbox is full"));
       continue;
     }
-    ++found->second.pending_requests;
-    mailbox_.push_back({found->second.id, std::move(parsed.value), std::move(parsed.code)});
     mailbox_high_water_ = std::max(mailbox_high_water_, mailbox_.size());
     ++received_messages_;
   }
@@ -411,7 +422,7 @@ void Server::drain_mailbox() {
   disconnected_.clear();
   const auto size = mailbox_.size();
   for (std::size_t i = 0; i < size; ++i) {
-    Envelope envelope = std::move(mailbox_.front()); mailbox_.pop_front();
+    Envelope envelope = mailbox_.take();
     if (auto* conn = connection(envelope.connection_id); conn != nullptr) --conn->pending_requests;
     handle(envelope);
   }
diff --git a/src/transport.hpp b/src/transport.hpp
index ed30026..016cd4e 100644
--- a/src/transport.hpp
+++ b/src/transport.hpp
@@ -86,6 +86,18 @@ class Server {
     FrameParser parser;
   };
   struct Envelope { std::uint64_t connection_id; Json value; std::string parser_error; };
+  // The same bounded storage is used by the reactor and the pure ownership
+  // regression. This extraction does not add a second admission policy.
+  class Mailbox {
+   public:
+    std::optional<std::string> admit(Envelope envelope, std::size_t& pending_requests);
+    Envelope take();
+    std::size_t size() const { return entries_.size(); }
+    void clear() { entries_.clear(); }
+   private:
+    std::deque<Envelope> entries_;
+  };
+  friend Json run_mailbox_probe(std::size_t capacity);
   Connection* connection(std::uint64_t id);
   void register_event(int fd, short filter, unsigned short flags, std::uint64_t connection_id = 0);
   void accept_ready();
@@ -102,7 +114,7 @@ class Server {
   Fd listener_;
   std::uint16_t port_ = 0;
   std::map<int, Connection> connections_;
-  std::deque<Envelope> mailbox_;
+  Mailbox mailbox_;
   std::set<std::uint64_t> disconnected_;
   Room room_;
   std::string nonce_;
diff --git a/tests/tests.cpp b/tests/tests.cpp
index 12ac70d..434668e 100644
--- a/tests/tests.cpp
+++ b/tests/tests.cpp
@@ -125,6 +125,39 @@ void foreign_thread_mutation_is_rejected() {
   foreign.join();
   check(rejected.load() && room.pending_count() == 0, "foreign thread cannot mutate the single owner room");
 }
+void room_permission_matrix_preserves_state() {
+  Json evidence = Json::array();
+  for (const auto* state : {"LOBBY", "RUNNING", "FINISHED", "CLOSED"}) {
+    Room room;
+    room.create("unit-room");
+    room.join("player-00", "session-00", 1);
+    if (std::string(state) != "LOBBY") room.join("player-01", "session-01", 2);
+    if (std::string(state) == "FINISHED") for (int tick = 0; tick < session_ticks; ++tick) room.tick();
+    if (std::string(state) == "CLOSED") room.close();
+    check(room.status() == state, "permission cell reached requested state");
+    const auto before = room.view();
+    bool create_rejected = false;
+    try { room.create("unused-room"); } catch (const std::logic_error&) { create_rejected = true; }
+    check(create_rejected && room.view() == before, "create cannot replace an existing room in any state");
+    bool join_rejected = false;
+    if (std::string(state) != "LOBBY") {
+      try { room.join("unused-player", "unused-session", 3); } catch (const std::logic_error&) { join_rejected = true; }
+      check(join_rejected && room.view() == before, "new join outside LOBBY rejected without consuming a slot");
+    }
+    room.leave(1);
+    auto expected = before;
+    for (auto& player : expected["players"]) if (player.at("player_id") == "player-00") player["connectivity"] = "LEFT";
+    check(room.view() == expected && room.pending_count() == 0, "leave preserves all other authoritative fields");
+    evidence.push_back(Json{{"state", state}, {"create_rejected", create_rejected},
+      {"join_rejected", join_rejected}, {"after_leave", room.view()}});
+    room.close();
+    check(room.status() == "CLOSED" && room.pending_count() == 0, "terminal cleanup after permission cell");
+  }
+  std::cout << Json{{"G03_room_permission_matrix", evidence}}.dump() << '\n';
+}
+void production_mailbox_capacity_and_drain() {
+  std::cout << Json{{"G03_mailbox_probe", run_mailbox_probe(max_mailbox_messages)}}.dump() << '\n';
+}
 std::vector<std::uint8_t> framed_text(const std::string& text) {
   const auto size = static_cast<std::uint32_t>(text.size());
   std::vector<std::uint8_t> frame{static_cast<std::uint8_t>(size >> 24U), static_cast<std::uint8_t>(size >> 16U),
@@ -331,7 +364,9 @@ int main(int argc, char** argv) {
       {"complete_frame_owned_partial_write", complete_frame_and_owned_write_buffer}, {"RAII_descriptor_ownership", descriptor_ownership},
       {"foreign_thread_mutation_rejected", foreign_thread_mutation_is_rejected},
       {"G02_strict_protocol_message_recovery", strict_protocol_and_message_recovery},
-      {"G02_maximum_frame_transport_end", parser_maximum_and_transport_end}};
+      {"G02_maximum_frame_transport_end", parser_maximum_and_transport_end},
+      {"G03_room_permission_matrix", room_permission_matrix_preserves_state},
+      {"G03_production_mailbox_capacity_drain", production_mailbox_capacity_and_drain}};
   } else if (std::string(argv[1]) == "integration") {
     tests = {{"real_TCP_authority_and_cleanup", real_tcp_authority_and_cleanup}, {"standalone_process_shutdown", [&] {
       standalone_process_shutdown(std::filesystem::absolute(argv[0]).parent_path() / "arena"); }}};
