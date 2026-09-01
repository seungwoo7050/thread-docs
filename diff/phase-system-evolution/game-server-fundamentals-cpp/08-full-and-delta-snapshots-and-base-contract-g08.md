# Full·Delta Snapshot과 Base Contract (G08)

## `feat(snapshot): add acknowledged full and delta replication`

diff --git a/CMakeLists.txt b/CMakeLists.txt
index 4bb3fd6..39ea0d6 100644
--- a/CMakeLists.txt
+++ b/CMakeLists.txt
@@ -18,7 +18,7 @@ if(ARENA_TSAN)
   add_link_options(-fsanitize=thread)
 endif()
 find_package(nlohmann_json 3.12.0 EXACT CONFIG REQUIRED)
-set(ARENA_SOURCES src/game.cpp src/transport.cpp src/replay.cpp src/scenario.cpp src/lifecycle_scenario.cpp)
+set(ARENA_SOURCES src/game.cpp src/transport.cpp src/replay.cpp src/snapshot.cpp src/scenario.cpp src/lifecycle_scenario.cpp)
 add_library(arena_core ${ARENA_SOURCES})
 # Fixture friends and their definitions are absent from the shipping target.
 add_library(arena_test_core ${ARENA_SOURCES})
diff --git a/TRACK.md b/TRACK.md
index fe110c6..f81dd74 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -1,4 +1,4 @@
-# fundamentals-cpp — G07 deterministic replay and state hash
+# fundamentals-cpp — G08 acknowledged full/delta snapshots
 
 SPEC_REVISION: `c1d62196ab76b55652f5d75a67514f8c6d8210ce` (phase-1; earlier evidence retains its original revision)
 
@@ -30,6 +30,7 @@ Run from this worktree; `track` resolves its own source directory. Build never r
 ./track scenario-run /absolute/path/to/G04.json /absolute/path/to/evidence.json
 ./track scenario-run /absolute/path/to/G07.json /absolute/path/to/L1.json
 ./track scenario-run /absolute/path/to/G07.json /absolute/path/to/V.json --variant rejected-removed
+./track scenario-run /absolute/path/to/G08.json /absolute/path/to/evidence.json
 ./track replay-verify /absolute/path/to/replay.json /absolute/path/to/evidence.json
 ./track server /absolute/path/to/config.json
 ```
@@ -81,10 +82,9 @@ Clean EOF is `TRANSPORT_EOF`; partial header/payload EOF is `TRANSPORT_EOF_IN_FR
 - Mailbox: kqueue read callback produces bounded `Envelope` messages; `drain_mailbox` consumes them after I/O callbacks return. Only that owner phase or explicit tick mutates Room.
 - Serialization: each queued `PendingWrite` owns its byte vector and unsent offset until write completes. Nonblocking partial writes and EAGAIN retain the suffix.
 - Shutdown coordinator: `Server::shutdown`; it stops accepting, drains intent, closes Room, attempts bounded final control flushing, releases connections and kqueue, and clears queues.
-- No worker thread, OS tick timer, external process, distributed lease renewer or snapshot retention is allocated by the server.
+- No worker thread, OS tick timer, external process or distributed lease renewer is allocated by the server.
 
-`SNAPSHOT` is a minimal baseline state notice at join/start/close. It has no snapshot sequence, delta base, hash or periodic cadence. `ROOM_FINISHED` includes the same authoritative final view. Full/delta replication first activates in G08.
-Both clients observe LOBBY, RUNNING and CLOSED via actual TCP. The canonical 1200-tick run additionally observes FINISHED.
+G08 replaces the earlier debug state notices with contract-shaped sequenced snapshots. Existing `ROOM_CREATED`, `ROOM_JOINED` and `ROOM_FINISHED` responses retain their lifecycle status. Shutdown is observed through actual TCP EOF and authoritative CLOSED/cleanup checks; no new leave or shutdown message type is added.
 
 ## Explicit resource bounds
 
@@ -104,6 +104,7 @@ Both clients observe LOBBY, RUNNING and CLOSED via actual TCP. The canonical 120
 | Client evidence | 4,096 received messages per client; overflow is a test failure |
 | Runner input/output | 1 MiB JSON input, 512 input commands, 32 setup commands; 4 MiB evidence output |
 | Replay capture/intake | 4,194,304 bytes including final LF; overflow latches incomplete capture and refuses export, while gameplay/hash computation continue |
+| Snapshot retention | newest32 full materializations per connected player stream; missing/unknown/evicted acknowledged base selects the next scheduled FULL |
 | Operator input | 4,096 bytes; overflow terminates with explicit failure |
 | Error text | 160 bytes on wire; CLI failure text 256 bytes |
 | Shutdown flush | 500ms wall ceiling; timeout is explicit metric, never a simulation clock input |
@@ -118,7 +119,7 @@ This file fixes commands before the first build. Every wrapper command records e
 Human-readable actual results are in `evidence/G01.md`; generated binaries and detailed dumps stay ignored.
 
 The canonical scenario is supplied by main and read at runtime. The runner does not contain canonical positions/scores as output constants.
-It resolves role names to server-generated session/player IDs, sends complete frames over loopback, waits for owner-phase INPUT_ACK at each intended tick boundary, then advances the injected manual clock. It compares final TCP messages against the same production Room's view, checks client-observed CLOSED, and verifies actual descriptor invalidation using `fcntl(F_GETFD)`.
+It resolves role names to server-generated session/player IDs, sends complete frames over loopback, waits for owner-phase INPUT_ACK at each intended tick boundary, then advances the injected manual clock. It compares final TCP messages against the same production Room's view, checks client-observed EOF and authoritative CLOSED, and verifies actual descriptor invalidation using `fcntl(F_GETFD)`.
 
 ## Deliberate next constraints
 
@@ -154,7 +155,7 @@ G04 adds a small integer `FixedTickAccumulator` and `Server::run_scheduler`. The
 
 The production clock is `std::chrono::steady_clock`; the canonical runner injects a manual monotonic reader into the same Server path. The fixed deltas yield `[1,1,2,0,4,2]` ticks and `[0,0,25,25,50,0]` remaining milliseconds. A wall-clock reversal is recorded only as external evidence. The complete unit suite preserves prior regressions, and a separate monotonic-mode execution extends the existing standalone shutdown integration test to verify actual adapter reads in the CLI scheduler.
 
-G05 sequence/target-tick and G06 four-attempt/authority guarantees remain active with their preserved regressions. G08+ replication, UDP, reconnect, slow-consumer coalescing and many-room scheduling remain inactive.
+G05 sequence/target-tick and G06 four-attempt/authority guarantees remain active with their preserved regressions. UDP, reconnect, slow-consumer coalescing and many-room scheduling remain inactive.
 
 ## G07 replay boundary
 
@@ -163,3 +164,11 @@ The owner records every newly accepted canonical input at its original admission
 Replay files contain the initial player/slot/spawn mapping, contract/clock/session constants, accepted input and lifecycle events per tick, resulting hashes and the completed tick count. Offline verification reconstructs only the initial model, then uses the existing admission, leave and simulation functions; it never injects expected resulting state. Explicitly incomplete, malformed and oversized intake fails. Capture bytes and pending events are released at server shutdown.
 
 The G07 live fixture establishes four real TCP/session bindings before one unchanged owner start-condition evaluation in `arena_test_core`. Only that target defines `ARENA_TEST_FIXTURES`; `arena_core`/`arena` cannot link the bootstrap. Public joins still start at two and reject further RUNNING joins. Full suites retain earlier regressions and add only zero-tick G07 checks. The five200-tick live/replay/variant passes and separate38-tick negative probe are explicit commands, recorded in `evidence/G07-runs.jsonl`; results are in `evidence/G07.md`.
+
+## G08 snapshot boundary
+
+Each Connection owns its sequence, latest ACK and32 immutable full materializations. Room start sends FULL sequence1 at tick-1. Every two completed simulation ticks publishes the next snapshot; sequences divisible by20 and the final FINISHED snapshot are FULL. A retained acknowledged base produces a DELTA with only changed player rows and explicit sorted removals. Unknown ACKs cannot become valid retroactively. TCP ACKs use the existing authenticated session/room/player envelope and an integer `snapshot_seq`; there is no additional resync protocol in this stage.
+
+Visible rows contain exactly `player_id`, `slot`, `x`, `y`, `direction`, `score`, `connectivity`, sorted by player ID and excluding LEFT records. Snapshot metadata carries the full canonical server hash, including internal sequence/cooldown and historical LEFT records. The client application check compares only the visible projection; it does not pretend those seven fields can reconstruct the complete canonical hash.
+
+The frozen G08 command executes196 real ticks once, captures99 snapshots for every remaining client, applies actual FULL/DELTA messages and sends the specified ACKs. Raw alpha captures include applied/authoritative visible state, canonical records/hashes and retained base IDs; all clients' cadence and feedback are retained. Earlier harnesses drain new periodic traffic at existing simulation boundaries and preserve gameplay, owner-order, lifecycle and cleanup assertions. Full unit adds one33-publication, zero-tick retention probe, not another canonical campaign. Evidence and exact command counts are in `evidence/G08.md` and `evidence/G08-runs.jsonl`.
diff --git a/evidence/G08-runs.jsonl b/evidence/G08-runs.jsonl
new file mode 100644
index 0000000..72f4d45
--- /dev/null
+++ b/evidence/G08-runs.jsonl
@@ -0,0 +1,6 @@
+{"label":"baseline-compile","category":"compile","units":1,"ticks":0,"ceiling_seconds":180,"argv":["clang++","-std=c++20","-O2","-Wall","-Wextra","-Wpedantic","-Werror","-fsanitize=thread","-g","-DARENA_TEST_FIXTURES=1","-I","src","-I","/opt/homebrew/include","artifacts/g08/reproduce.cpp","src/game.cpp","src/transport.cpp","src/replay.cpp","-o","artifacts/g08/reproduce"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g08/baseline-compile.log","started_at":"2026-08-28T04:28:16.730935+00:00","duration_seconds":13.223736,"exit":0,"wrapper_pid":68602,"child_pid":68611,"timed_out":false}
+{"label":"baseline","category":"unit","units":1,"ticks":196,"expected_exit":1,"argv":["env","TSAN_OPTIONS=halt_on_error=1","./artifacts/g08/reproduce","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G08.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g08/baseline.json"],"ceiling_seconds":120,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g08/baseline.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g08/baseline.json","started_at":"2026-08-28T04:28:48.602328+00:00","duration_seconds":0.980657,"exit":1,"wrapper_pid":69034,"child_pid":69043,"timed_out":false,"observed_ticks":196,"runtime_pid":69043}
+{"label":"build","category":"compile","units":2,"ticks":0,"ceiling_seconds":180,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g08-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g08/track","TSAN_OPTIONS=halt_on_error=1","ARENA_TSAN=ON","./track","build"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g08/build.log","started_at":"2026-08-28T04:39:44.332281+00:00","duration_seconds":32.009569,"exit":0,"wrapper_pid":72243,"child_pid":72252,"timed_out":false}
+{"label":"unit","category":"unit","units":1,"ticks":0,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g08-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g08/track","TSAN_OPTIONS=halt_on_error=1","./track","unit-test"],"expected_exit":0,"ceiling_seconds":120,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g08/unit.log","started_at":"2026-08-28T04:40:45.551968+00:00","duration_seconds":2.525793,"exit":0,"wrapper_pid":72709,"child_pid":72710,"timed_out":false}
+{"label":"integration","category":"integration","units":1,"ticks":0,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g08-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g08/track","TSAN_OPTIONS=halt_on_error=1","./track","integration-test"],"expected_exit":0,"ceiling_seconds":120,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g08/integration.log","started_at":"2026-08-28T04:40:48.148770+00:00","duration_seconds":1.363124,"exit":0,"wrapper_pid":72723,"child_pid":72724,"timed_out":false}
+{"label":"canonical","category":"canonical","units":1,"ticks":196,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g08-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g08/track","TSAN_OPTIONS=halt_on_error=1","./track","scenario-run","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G08.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g08/canonical.json"],"expected_exit":0,"ceiling_seconds":120,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g08/canonical.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g08/canonical.json","started_at":"2026-08-28T04:40:49.575635+00:00","duration_seconds":1.412812,"exit":0,"wrapper_pid":72744,"child_pid":72745,"timed_out":false,"observed_ticks":196,"runtime_pid":72751}
diff --git a/evidence/G08.md b/evidence/G08.md
new file mode 100644
index 0000000..89b2d3d
--- /dev/null
+++ b/evidence/G08.md
@@ -0,0 +1,30 @@
+# G08 — acknowledged full/delta snapshots
+
+Initial attempt; phase-1 / realtime-core; branch `track/fundamentals-cpp`.
+SPEC_REVISION `c1d62196ab76b55652f5d75a67514f8c6d8210ce`; START `0d5fbc4d2c6569841cfd21b0c045a491b1e6e787`.
+Frozen main `index/scenarios/G08.json` SHA-256 `d121973167461dddbb3ba0bd339ed26b09486cda8bbec642328ccc5cea9e578e`.
+
+## Baseline and scope
+
+`artifacts/g08/pre-change-production.json` records12 files byte-matched against actual START before and after baseline execution. `commands.json` resolved every argv/output before execution. `reproduce.cpp` used the unchanged G07 test-only roster bootstrap and real gameplay/transport/replay code:196 ticks, all4 TCP events, no periodic messages, only delta's legacy LEAVE snapshot. Bootstrap start omission is separately labelled, not a production defect; no ACK was fabricated. Expected exit1; raw `baseline.json` SHA-256 `f25d882526e78c3a24c28486c7c116d846bb728c5c060525302ba562639693be`. Main was notified before production edits.
+
+Added only per-connection full/delta sequencing,32 materialized bases, authenticated integer ACK and owner publication. Removed unsequenced debug SNAPSHOT notices; existing lifecycle responses, owner state and actual EOF preserve earlier assertions. Gameplay/replay source files remain byte-identical to START. The same test-only four-peer fixture calls normal start publication; `nm`/build flags confirm no fixture or G08 runner in shipping `arena`, and fixture definition only in `arena_test_core`. No dependency or future-stage policy changes.
+
+## Actual verification
+
+`evidence/G08-runs.jsonl` is the exact ledger (argv, category, counts, raw paths, duration, exit and process IDs). `artifacts/g08/verify.py` ran the post-build commands once each, sequentially with fixed ceilings and stop on failure. Existing regressions remain; the sole new unit fixture publishes33 snapshots with zero Room ticks.
+
+| Pass | Result | Seconds / exit |
+| --- | --- | --- |
+| Baseline compile / reproduction | unchanged real implementation,196 ticks | 13.223736 /0;0.980657 /1 expected |
+| TSan configure/build | PASS | 32.009569 /0 |
+| Full unit / integration | 24/24;3/3 PASS | 2.525793 /0;1.363124 /0 |
+| Post canonical |196 ticks, PID72751, PASS |1.412812 /0 |
+
+Raw `artifacts/g08/canonical.json` (1069712 bytes; SHA-256 `07e411cb8ba88872cb8093534d207a9a0ba540faa764162e4efbb418083ec577`) retains all99 alpha captures, actual applied/authoritative visible states, full canonical records, ACK feedback, every client's cadence/retention and196 tick records. Baseline and final196 canonical records/hashes are identical: gameplay/hash guarantees NOT_REPRODUCED. Hash-list digest (one hash plus LF) `0a1c36fdda715ba5643d4119e3295d35413ec61d08b8e76eb8b5f913e5248d4f`.
+
+Counts alpha/bravo/charlie99, delta98. Start FULL1/tick-1; DELTA2/base1; DELTA3/base2; unknown ACK999 causes scheduled FULL4. FULL sequences `[1,4,20,40,60,80]`. Seq98/tick193 carries alpha score1 at `[87600,10000]`; seq99/tick195 is DELTA/base98 with alpha STOP and removal `["player-03"]`. Canonical state retains delta LEFT. Final retained IDs68–99, high-water32; departed delta retains0. Snapshot hashes equal full canonical hashes, while client application equals only the seven-field visible projection.
+
+Actual resource high waters: connections4/512, mailbox1/512, outbound1/64, pending1/64, attempts1/4, parser222/16388, replay21497/4194304, retention32/32. Received403/sent402 messages. Four actual EOF observations; all10 descriptors closed; every active cleanup counter and FD delta0. No TSan reports.
+
+Budget consumed: compile/configure3/8, unit2/4 including baseline, integration1/2, post canonical1/1. G08 simulation392 ticks including baseline; new pure fixture0 ticks. Fault0/load0; unexpected failures0. Raw build/unit/integration/canonical logs remain under `artifacts/g08/`; wrapper child commands under `artifacts/g08/track/`. Root independent/cross gate is pending at submission; no tag, push, spec/index/threads or G09+ work.
diff --git a/src/lifecycle_scenario.cpp b/src/lifecycle_scenario.cpp
index 30d7f59..c2f4dc2 100644
--- a/src/lifecycle_scenario.cpp
+++ b/src/lifecycle_scenario.cpp
@@ -122,6 +122,15 @@ struct LifecycleFixture {
     ensure(descriptor_closed(fd), "closed client descriptor remains live");
     ++closed_client_checks;
   }
+  void drain_snapshots() {
+    server.pump();
+    wait_for([&] { return server.cleanup().at("outbound_messages") == 0; });
+    for (auto& [role, peer] : peers) {
+      (void)role;
+      while (auto value = peer.tcp->try_receive())
+        ensure(value->at("type") == "SNAPSHOT" && value->contains("snapshot_seq"), "unexpected frame during snapshot drain");
+    }
+  }
   Json finish() {
     auto descriptors = server.owned_descriptors();
     for (auto& [role, peer] : peers) {
@@ -154,7 +163,15 @@ Json lifecycle_cell(const Json& scenario, const std::string& state, const std::s
   fixture.setup(count);
   Json finished = Json::object();
   if (state == "FINISHED") {
-    for (int tick = 0; tick < scenario.at("finish_ticks").get<int>(); ++tick) fixture.server.advance_one_tick();
+    fixture.drain_snapshots();
+    for (int tick = 0; tick < scenario.at("finish_ticks").get<int>(); ++tick) {
+      fixture.server.advance_one_tick();
+      if ((tick + 1) % 2 == 0) for (auto& [role, peer] : fixture.peers) {
+        (void)role;
+        const auto snapshot = peer.tcp->receive_type(fixture.server,"SNAPSHOT");
+        ensure(snapshot.at("tick") == tick, "FINISHED setup preserves each real tick while draining snapshots");
+      }
+    }
     ensure(fixture.server.room().executed_ticks() == 1200 && fixture.clock.now_ms == 60000,
            "FINISHED requires exactly 1200 manual ticks");
     for (auto& [role, peer] : fixture.peers) {
@@ -557,6 +574,7 @@ Json run_authority_scenario(const Json& scenario) {
       "server movement, score, cooldown or fixed simulation clock differs at an actual tick");
     evidence["ticks"].push_back(Json{{"tick",tick},{"state",fixture.server.room().view()},{"applied_sequences",applied},
       {"pending_inputs",fixture.server.room().pending_count()},{"simulation_ms",fixture.clock.now_ms}});
+    fixture.drain_snapshots();
   }
   check_authority(index == 18 && accepted == 16 && fixture.server.room().status() == "RUNNING" &&
     fixture.server.room().pending_count() == 0, "fixed admission counts or terminal scenario state differs");
diff --git a/src/scenario.cpp b/src/scenario.cpp
index e82d307..7e58e0d 100644
--- a/src/scenario.cpp
+++ b/src/scenario.cpp
@@ -325,6 +325,12 @@ Json run_scenario(const Json& scenario) {
     }
   }
   require(server.room().status() == "RUNNING", "setup did not start the real server room");
+  for (auto& [role, participant] : clients) {
+    (void)role;
+    const auto start = participant.tcp->receive_type(server,"SNAPSHOT");
+    require(start.at("snapshot_seq") == 1 && start.at("tick") == -1 && start.at("kind") == "FULL" &&
+      start.at("status") == "RUNNING", "real Room-start full snapshot");
+  }
   std::size_t consumed_inputs = 0;
   int previous_tick = -1;
   for (const auto& input : scenario.at("inputs")) {
@@ -361,6 +367,11 @@ Json run_scenario(const Json& scenario) {
     // Every intended input has crossed TCP and received an owner-phase ACK.
     server.advance_one_tick();
     require(server.room().executed_ticks() == tick + 1, "manual tick did not execute exactly once");
+    if ((tick + 1) % 2 == 0) for (auto& [role, participant] : clients) {
+      (void)role;
+      const auto snapshot = participant.tcp->receive_type(server,"SNAPSHOT");
+      require(snapshot.at("tick") == tick, "periodic traffic drained at its actual tick boundary");
+    }
   }
   require(consumed_inputs == scenario.at("inputs").size(), "scenario inputs were not all consumed");
   const Json final = server.room().view();
@@ -387,13 +398,16 @@ Json run_scenario(const Json& scenario) {
   server.shutdown();
   evidence["client_lifecycle"] = Json::object();
   evidence["client_observations"] = Json::object();
+  evidence["client_eof"] = Json::object();
+  require(server.room().status() == "CLOSED", "owner did not close the Room");
   for (auto& [role, participant] : clients) {
-    bool closed_seen = false;
-    for (int pending = 0; pending < 64; ++pending) {
-      const auto state = participant.tcp->receive_type(server, "SNAPSHOT");
-      if (state.at("status") == "CLOSED") { closed_seen = true; break; }
+    const auto deadline = std::chrono::steady_clock::now() + std::chrono::seconds(2);
+    while (!participant.tcp->peer_closed() && std::chrono::steady_clock::now() < deadline) {
+      if (auto value = participant.tcp->try_receive())
+        require(value->at("type") == "SNAPSHOT" && value->contains("snapshot_seq"), "unexpected shutdown frame");
     }
-    require(closed_seen, "client did not observe CLOSED before EOF");
+    require(participant.tcp->peer_closed(), "client did not observe owner shutdown EOF");
+    evidence["client_eof"][role] = true;
     closed_fds.push_back(participant.tcp->descriptor());
     participant.tcp->close();
     evidence["client_lifecycle"][role] = observed_lifecycle(*participant.tcp);
diff --git a/src/snapshot.cpp b/src/snapshot.cpp
new file mode 100644
index 0000000..65dfc44
--- /dev/null
+++ b/src/snapshot.cpp
@@ -0,0 +1,53 @@
+#include "snapshot.hpp"
+#include <algorithm>
+
+namespace arena {
+void SnapshotStream::acknowledge(std::uint64_t sequence) {
+  acknowledged_ = sequence;
+  // A sequence not yet sent cannot become an acknowledged base merely by
+  // being published later; only this actual feedback establishes a base.
+  acknowledged_known_ = std::any_of(retained_.begin(),retained_.end(),[&](const auto& entry) {
+    return entry.at("snapshot_seq") == sequence;
+  });
+}
+Json SnapshotStream::publish(const Room& room, const std::string& state_hash) {
+  Json players = Json::array();
+  for (const auto& [id, player] : room.players()) {
+    if (!player.connected) continue;
+    players.push_back(Json{{"player_id",id},{"slot",player.slot},{"x",player.x},{"y",player.y},
+      {"direction",direction_name(player.direction)},{"score",player.score},{"connectivity","CONNECTED"}});
+  }
+  auto full = message("SNAPSHOT");
+  full.update(Json{{"snapshot_seq",++sequence_},{"room_id",room.id()},{"tick",room.executed_ticks()-1},
+    {"owner_epoch",0},{"kind","FULL"},{"base_snapshot_seq",nullptr},{"state_hash",state_hash},
+    {"players",players},{"removed_player_ids",Json::array()},{"status",room.status()}});
+  auto wire = full;
+  const auto base = std::find_if(retained_.begin(),retained_.end(),[&](const auto& entry) {
+    return acknowledged_known_ && acknowledged_ && entry.at("snapshot_seq") == *acknowledged_;
+  });
+  if (sequence_ % 20 != 0 && room.status() == "RUNNING" && base != retained_.end()) {
+    wire["kind"] = "DELTA"; wire["base_snapshot_seq"] = *acknowledged_;
+    wire.erase("status"); wire["players"] = Json::array();
+    for (const auto& player : players) {
+      const auto previous = std::find_if(base->at("players").begin(),base->at("players").end(),
+        [&](const auto& item) { return item.at("player_id") == player.at("player_id"); });
+      if (previous == base->at("players").end() || *previous != player) wire["players"].push_back(player);
+    }
+    for (const auto& previous : base->at("players")) {
+      if (std::none_of(players.begin(),players.end(),[&](const auto& item) {
+            return item.at("player_id") == previous.at("player_id"); }))
+        wire["removed_player_ids"].push_back(previous.at("player_id"));
+    }
+  }
+  if (retained_.size() == snapshot_retention) retained_.pop_front();
+  retained_.push_back(std::move(full));
+  high_water_ = std::max(high_water_,retained_.size());
+  return wire;
+}
+Json SnapshotStream::metrics() const {
+  auto ids = Json::array();
+  for (const auto& entry : retained_) ids.push_back(entry.at("snapshot_seq"));
+  return Json{{"last_seq",sequence_},{"acknowledged_seq",acknowledged_ ? Json(*acknowledged_) : Json(nullptr)},
+    {"retained_ids",ids},{"retained_count",retained_.size()},{"high_water",high_water_}};
+}
+}
diff --git a/src/snapshot.hpp b/src/snapshot.hpp
new file mode 100644
index 0000000..0f8e076
--- /dev/null
+++ b/src/snapshot.hpp
@@ -0,0 +1,24 @@
+#pragma once
+#include "game.hpp"
+
+namespace arena {
+inline constexpr std::size_t snapshot_retention = 32;
+
+// Owned by one Connection and used only by the Room owner. Every retained
+// entry is a full materialization, even when its wire representation is delta.
+class SnapshotStream {
+ public:
+  Json publish(const Room& room, const std::string& state_hash);
+  void acknowledge(std::uint64_t sequence);
+  void clear() { retained_.clear(); acknowledged_.reset(); acknowledged_known_ = false; }
+  std::size_t size() const { return retained_.size(); }
+  std::size_t high_water() const { return high_water_; }
+  Json metrics() const;
+ private:
+  std::uint64_t sequence_ = 0;
+  std::optional<std::uint64_t> acknowledged_;
+  bool acknowledged_known_ = false;
+  std::deque<Json> retained_;
+  std::size_t high_water_ = 0;
+};
+}
diff --git a/src/transport.cpp b/src/transport.cpp
index d907e44..102236f 100644
--- a/src/transport.cpp
+++ b/src/transport.cpp
@@ -63,7 +63,7 @@ std::string request_error(const Json& value) {
   if (value.at("v") != 1) return "PROTOCOL_VERSION_UNSUPPORTED";
   const auto type = value.at("type").get<std::string>();
   if (type == "HELLO") return {};
-  if (type != "CREATE_ROOM" && type != "JOIN_ROOM" && type != "LEAVE_ROOM" && type != "INPUT")
+  if (type != "CREATE_ROOM" && type != "JOIN_ROOM" && type != "LEAVE_ROOM" && type != "INPUT" && type != "SNAPSHOT_ACK")
     return "MESSAGE_TYPE_UNKNOWN";
   const auto string_field = [&](const char* name) { return value.contains(name) && value.at(name).is_string(); };
   if (!string_field("session_id")) return "MESSAGE_INVALID";
@@ -247,7 +247,7 @@ void Server::accept_ready() {
     socket_options(fd.get());
     const int raw = fd.get();
     const auto id = next_connection_++;
-    connections_.emplace(raw, Connection{std::move(fd), id, {}, {}, {}, 0, {}});
+    connections_.emplace(raw, Connection{std::move(fd), id, {}, {}, {}, 0, {}, {}});
     register_event(raw, EVFILT_READ, EV_ADD, id);
     register_event(raw, EVFILT_WRITE, EV_ADD | EV_DISABLE, id);
     connection_high_water_ = std::max(connection_high_water_, connections_.size());
@@ -374,6 +374,26 @@ void Server::broadcast(const Json& value) {
   for (const auto& [fd, conn] : connections_) { (void)fd; if (!conn.player_id.empty()) ids.push_back(conn.id); }
   for (auto id : ids) queue(id, value);
 }
+void Server::start_room() {
+  replay_.start(room_);
+  accumulator_.reset(read_monotonic());
+  publish_snapshots(sha256(canonical_state(room_)));
+}
+void Server::publish_snapshots(const std::string& state_hash) {
+  std::vector<std::uint64_t> ids;
+  for (const auto& [fd, conn] : connections_) {
+    (void)fd;
+    const auto player = room_.players().find(conn.player_id);
+    if (player != room_.players().end() && player->second.connected) ids.push_back(conn.id);
+  }
+  for (const auto id : ids) {
+    if (auto* conn = connection(id)) {
+      auto snapshot = conn->snapshots.publish(room_,state_hash);
+      snapshot_retention_high_water_ = std::max(snapshot_retention_high_water_,conn->snapshots.high_water());
+      queue(id,std::move(snapshot));
+    }
+  }
+}
 void Server::handle(const Envelope& envelope) {
   auto* conn = connection(envelope.connection_id);
   if (conn == nullptr) return;
@@ -396,7 +416,7 @@ void Server::handle(const Envelope& envelope) {
       Json reply = message("WELCOME"); reply["session_id"] = conn->session_id;
       queue(id, std::move(reply)); return;
     }
-    if (type != "CREATE_ROOM" && type != "JOIN_ROOM" && type != "LEAVE_ROOM" && type != "INPUT") {
+    if (type != "CREATE_ROOM" && type != "JOIN_ROOM" && type != "LEAVE_ROOM" && type != "INPUT" && type != "SNAPSHOT_ACK") {
       reject("MESSAGE_TYPE_UNKNOWN", "unknown message type"); return;
     }
     if (conn->session_id.empty() || value.at("session_id").get<std::string>() != conn->session_id) {
@@ -415,26 +435,25 @@ void Server::handle(const Envelope& envelope) {
       if (room_.status() != "LOBBY" || !conn->player_id.empty() || room_.players().size() == max_players) {
         reject("ROOM_NOT_JOINABLE", "room is not joinable"); return;
       }
-      Json lobby = room_.view(); lobby.update(message("SNAPSHOT"));
       conn->player_id = new_id("player", next_player_++);
       const auto& player = room_.join(conn->player_id, conn->session_id, id);
       Json reply = message("ROOM_JOINED"); reply["room_id"] = room_.id(); reply["player_id"] = player.id;
       reply["slot"] = player.slot; reply["status"] = room_.status();
-      queue(id, std::move(lobby));
       queue(id, std::move(reply));
-      if (room_.status() == "RUNNING") {
-        replay_.start(room_);
-        accumulator_.reset(read_monotonic());
-        Json state = room_.view(); state.update(message("SNAPSHOT")); broadcast(state);
-      }
+      if (room_.status() == "RUNNING") start_room();
       return;
     }
     if (conn->player_id.empty() || value.at("player_id").get<std::string>() != conn->player_id) {
       reject("PLAYER_MISMATCH", "player must belong to this connection"); return;
     }
     if (type == "LEAVE_ROOM") {
-      leave_room(id,"LEAVE_ROOM");
-      Json state = room_.view(); state.update(message("SNAPSHOT")); queue(id, state); return;
+      leave_room(id,"LEAVE_ROOM"); return;
+    }
+    if (type == "SNAPSHOT_ACK") {
+      const auto& sequence = value.at("snapshot_seq");
+      if (!sequence.is_number_integer() || (!sequence.is_number_unsigned() && sequence.get<std::int64_t>() < 0))
+        throw std::invalid_argument("snapshot_seq must be an exact nonnegative integer");
+      conn->snapshots.acknowledge(sequence.get<std::uint64_t>()); return;
     }
     const auto result = admit_input(room_, conn->player_id, value);
     if (result.error) {
@@ -459,6 +478,7 @@ void Server::leave_room(std::uint64_t connection_id, const std::string& kind) {
   }
   room_.leave(connection_id);
   if (!player_id.empty()) replay_.left(room_,player_id,kind);
+  if (auto* conn = connection(connection_id)) conn->snapshots.clear();
 }
 void Server::drain_mailbox() {
   // Room mutations happen after all ready I/O callbacks have completed.
@@ -507,15 +527,22 @@ void Server::advance_one_tick() {
     ++errors_["ACTION_REJECTED"]; queue(failure.connection_id, std::move(error));
   }
   replay_.finish_tick(room_);
+  if (room_.executed_ticks() % 2 == 0) publish_snapshots(replay_.last_state_hash());
   if (room_.status() == "FINISHED") {
     accumulator_.reset(0); last_batch_ = {};
     Json result = room_.view(); result.update(message("ROOM_FINISHED")); broadcast(result);
   }
 }
 Json Server::metrics() const {
+  auto streams = Json::object();
+  for (const auto& [fd, conn] : connections_) {
+    (void)fd;
+    if (!conn.player_id.empty()) streams[conn.player_id] = conn.snapshots.metrics();
+  }
   return Json{{"received_messages", received_messages_}, {"sent_messages", sent_messages_},
     {"mailbox_high_water", mailbox_high_water_}, {"outbound_control_high_water", outbound_high_water_},
     {"connection_high_water", connection_high_water_}, {"input_per_player_high_water", room_.input_high_water()},
+    {"snapshot_retention_high_water",snapshot_retention_high_water_},{"snapshot_streams",streams},
     {"input_attempt_per_player_high_water", room_.input_attempt_high_water()},
     {"replay_bytes_high_water",replay_.high_water_bytes()},
     {"replay_capture_complete",replay_.complete()},{"replay_capture_error",replay_.failure()},
@@ -531,14 +558,16 @@ Json Server::metrics() const {
       {"operational_state", last_batch_.overloaded ? "OVERLOADED" : "NORMAL"}}}};
 }
 Json Server::cleanup() const {
-  std::size_t queued = 0, parser_buffered = 0, input_attempts = 0;
+  std::size_t queued = 0, parser_buffered = 0, input_attempts = 0, retained_snapshots = 0;
   for (const auto& [fd, conn] : connections_) {
     (void)fd; queued += conn.outbound.size(); parser_buffered += conn.parser.buffered_bytes();
+    retained_snapshots += conn.snapshots.size();
   }
   for (const auto& [id, player] : room_.players()) { (void)id; input_attempts += player.input_attempts; }
   return Json{{"server_connections", connections_.size()}, {"server_descriptors", owned_descriptors().size()},
     {"mailbox_messages", mailbox_.size()}, {"pending_inputs", room_.pending_count()}, {"outbound_messages", queued},
     {"input_attempts", input_attempts},
+    {"retained_snapshots",retained_snapshots},
     {"replay_bytes",replay_.bytes()},{"replay_pending_events",replay_.pending_events()},
     {"parser_buffered_bytes", parser_buffered}, {"parser_storage_bytes", connections_.size() * FrameParser::storage_bytes},
     {"worker_threads", 0}, {"timers", 0}, {"disconnect_notifications", disconnected_.size()},
@@ -559,9 +588,6 @@ void Server::shutdown() {
   room_.close();
   replay_.clear();
   accumulator_.reset(0); last_batch_ = {};
-  if (room_.status() == "CLOSED") {
-    Json state = room_.view(); state.update(message("SNAPSHOT")); broadcast(state);
-  }
   // Only transport flushing uses a wall deadline; no simulation runs here.
   const auto deadline = std::chrono::steady_clock::now() + std::chrono::milliseconds(500);
   for (;;) {
diff --git a/src/transport.hpp b/src/transport.hpp
index 3e36529..fcaf4c9 100644
--- a/src/transport.hpp
+++ b/src/transport.hpp
@@ -1,5 +1,6 @@
 #pragma once
 #include "replay.hpp"
+#include "snapshot.hpp"
 #include <array>
 #include <atomic>
 #include <cstddef>
@@ -91,6 +92,7 @@ class Server {
     std::deque<PendingWrite> outbound;
     std::size_t pending_requests = 0;
     FrameParser parser;
+    SnapshotStream snapshots;
   };
   struct Envelope { std::uint64_t connection_id; Json value; std::string parser_error; };
   // The same bounded storage is used by the reactor and the pure ownership
@@ -117,6 +119,8 @@ class Server {
   void end_transport(int fd, bool io_error);
   void queue(std::uint64_t connection_id, Json value);
   void broadcast(const Json& value);
+  void start_room();
+  void publish_snapshots(const std::string& state_hash);
   void handle(const Envelope& envelope);
   void leave_room(std::uint64_t connection_id, const std::string& kind);
   std::int64_t read_monotonic();
@@ -142,6 +146,7 @@ class Server {
   std::size_t mailbox_high_water_ = 0;
   std::size_t outbound_high_water_ = 0;
   std::size_t connection_high_water_ = 0;
+  std::size_t snapshot_retention_high_water_ = 0;
   std::size_t max_read_bytes_ = 0;
   std::size_t parser_high_water_ = 0;
   std::uint64_t need_more_events_ = 0;
diff --git a/tests/g07.cpp b/tests/g07.cpp
index fe51db4..40ce17e 100644
--- a/tests/g07.cpp
+++ b/tests/g07.cpp
@@ -3,6 +3,7 @@
 #error G07 roster bootstrap must never be compiled into the shipping artifact
 #endif
 #include <algorithm>
+#include <chrono>
 #include <memory>
 #include <set>
 #include <stdexcept>
@@ -49,6 +50,32 @@ Json request(const Json& event, const std::map<std::string,Peer>& peers, const s
   }
   return value;
 }
+Json receive_control(TcpClient& peer, Server& server) {
+  for (int count = 0; count < 64; ++count) {
+    auto value = peer.receive(server);
+    if (value.at("type") != "SNAPSHOT") return value;
+    require(value.contains("snapshot_seq") && value.contains("state_hash"), "conformant snapshot during control demultiplexing");
+  }
+  throw std::runtime_error("control response absent within bounded snapshot drain");
+}
+void leave_barrier(Peer& peer, Server& server) {
+  const auto tick = server.room().executed_ticks();
+  const auto deadline = std::chrono::steady_clock::now() + std::chrono::seconds(5);
+  while (server.room().players().at(peer.player).connected && std::chrono::steady_clock::now() < deadline) server.pump(1);
+  require(!server.room().players().at(peer.player).connected && server.room().executed_ticks() == tick,
+          "actual owner LEAVE commit without a simulation tick or invented response");
+}
+void drain_periodic(Server& server, const std::map<std::string,Peer>& peers) {
+  server.pump();
+  const auto deadline = std::chrono::steady_clock::now() + std::chrono::seconds(5);
+  while (server.cleanup().at("outbound_messages") != 0 && std::chrono::steady_clock::now() < deadline) server.pump(1);
+  require(server.cleanup().at("outbound_messages") == 0, "bounded periodic flush");
+  for (const auto& [role,peer] : peers) {
+    (void)role;
+    while (auto value = peer.tcp->try_receive())
+      require(value->at("type") == "SNAPSHOT" && value->contains("snapshot_seq"), "unexpected frame during periodic drain");
+  }
+}
 void final_fixture_state(const Room& room) {
   const auto& a = room.players().at("player-00"); const auto& b = room.players().at("player-01");
   const auto& c = room.players().at("player-02"); const auto& d = room.players().at("player-03");
@@ -98,8 +125,7 @@ struct ReplayFixture {
     }
     // All four transport/session/player bindings precede the one start call.
     initialize(server.room_,scenario.at("room_id").get<std::string>(),std::move(players));
-    server.replay_.start(server.room_);
-    server.accumulator_.reset(server.read_monotonic());
+    server.start_room();
     return bindings;
   }
   static void offline(Room& room, const Json& artifact) {
@@ -154,7 +180,10 @@ ReplayRun run_replay_scenario(const Json& scenario, bool rejected_removed) {
       }
       const auto role = event.at("client").get<std::string>(); auto& peer = peers.at(role);
       const auto value = request(event,peers,server.room().id()); const auto before = observed_state(server.room());
-      peer.tcp->send(value); const auto response = peer.tcp->receive(server); const auto after = observed_state(server.room());
+      peer.tcp->send(value); Json response = nullptr;
+      if (kind == "INPUT") response = receive_control(*peer.tcp,server);
+      else leave_barrier(peer,server);
+      const auto after = observed_state(server.room());
       wire.push_back(Json{{"before_tick",tick},{"client",role},{"request",value},{"response",response},
         {"before",before},{"after",after}});
       if (kind == "INPUT") {
@@ -164,7 +193,7 @@ ReplayRun run_replay_scenario(const Json& scenario, bool rejected_removed) {
         admissions.push_back(Json{{"client",role},{"seq",event.at("seq")},{"before_tick",tick},
           {"target_tick",event.at("target_tick")},{"code",code}});
       } else {
-        require(kind == "LEAVE_ROOM" && response.at("type") == "SNAPSHOT" &&
+        require(kind == "LEAVE_ROOM" && response.is_null() &&
           !server.room().players().at(peer.player).connected, "real LEAVE_ROOM lifecycle");
         lifecycle.push_back(Json{{"before_tick",tick},{"kind",kind},{"player_id",peer.player},{"client",role},{"connectivity","LEFT"}});
       }
@@ -174,11 +203,12 @@ ReplayRun run_replay_scenario(const Json& scenario, bool rejected_removed) {
     const auto hash = server.replay().artifact().at("ticks").back().at("state_hash").get<std::string>();
     hashes.push_back(hash); run.records.push_back(tick_record(server.room(),hash));
     if (tick == 199) {
-      const auto failure = peers.at("charlie").tcp->receive(server);
+      const auto failure = receive_control(*peers.at("charlie").tcp,server);
       require(failure.at("code") == "ACTION_REJECTED" && failure.at("tick") == 199 &&
         failure.at("player_id") == peers.at("charlie").player, "accepted charlie TAG action failure");
       actions.push_back(failure);
     }
+    drain_periodic(server,peers);
   }
   require(removed == (rejected_removed ? 4U : 0U) && admissions.size() == (rejected_removed ? 13U : 17U) &&
     lifecycle.size() == 1, "exact variant/event count");
@@ -204,6 +234,190 @@ ReplayRun run_replay_scenario(const Json& scenario, bool rejected_removed) {
   return run;
 }
 
+Json run_snapshot_scenario(const Json& scenario) {
+  require(scenario.at("thread") == "G08" && scenario.at("contract_version") == 1 && scenario.at("seed") == 7050 &&
+    scenario.at("ticks") == 196 && scenario.at("players").size() == 4 && scenario.at("events").size() == 4 &&
+    scenario.at("clock").at("kind") == "manual" && scenario.at("clock").at("tick_duration_ms") == 50 &&
+    scenario.at("socket_ceiling_ms") == 5000 && scenario.at("expected_snapshot_count_for_remaining_clients") == 99 &&
+    scenario.at("snapshot_cadence") == Json{{"room_start_seq",1},{"room_start_tick",-1},{"every_executed_ticks",2},
+      {"full_every_snapshot_seq_multiple",20},{"retention",32}}, "G08 frozen dimensions");
+  const int descriptors_before = Fd::live(); ManualClock clock; Server server(clock);
+  std::map<std::string,Peer> peers;
+  struct Replica { std::map<std::uint64_t,Json> retained; Json applied; int count = 0; };
+  std::map<std::string,Replica> replicas;
+  Json cadence = Json::object(), captures = Json::array(), feedback = Json::array(), events = Json::array();
+  Json tick_records = Json::array(), hashes = Json::array();
+  for (const auto& item : scenario.at("players")) {
+    const auto role = item.at("client").get<std::string>(); auto& peer = peers[role];
+    peer.tcp = std::make_unique<TcpClient>(server.port()); peer.tcp->send(message("HELLO"));
+    peer.session = peer.tcp->receive_type(server,"WELCOME").at("session_id").get<std::string>();
+    peer.player = item.at("player_id").get<std::string>(); cadence[role] = Json::array();
+  }
+  const auto bindings = ReplayFixture::live(server,scenario,peers);
+  const auto initial = observed_state(server.room());
+  const auto visible = [&] {
+    Json players = Json::array();
+    // Independent expected projection from authoritative scalar fields, not
+    // the production snapshot serializer or the replica's reconstructed rows.
+    for (const auto& [id,p] : server.room().players()) if (p.connected)
+      players.push_back(Json{{"player_id",id},{"slot",p.slot},{"x",p.x},{"y",p.y},
+        {"direction",direction_name(p.direction)},{"score",p.score},{"connectivity","CONNECTED"}});
+    return Json{{"room_id",server.room().id()},{"tick",server.room().executed_ticks()-1},
+      {"owner_epoch",0},{"status",server.room().status()},{"players",players}};
+  };
+  auto capture = [&](int sequence) {
+    const auto state_before = observed_state(server.room());
+    const auto canonical = canonical_state(server.room()), hash = sha256(canonical);
+    const auto authority = visible();
+    if (server.room().executed_ticks() != 0) require(server.replay().last_state_hash() == hash,"G08 canonical replay/hash equality");
+    Json alpha_capture, acks = Json::array();
+    for (auto& [role,peer] : peers) {
+      if (!server.room().players().at(peer.player).connected) continue;
+      auto& replica = replicas[role]; const auto wire = peer.tcp->receive(server);
+      const bool full = sequence == 1 || sequence == 4 || sequence % 20 == 0;
+      require(wire.at("v") == 1 && wire.at("type") == "SNAPSHOT" && wire.at("snapshot_seq").is_number_integer() &&
+        wire.at("snapshot_seq") == ++replica.count && replica.count == sequence && wire.at("room_id") == server.room().id() &&
+        wire.at("tick") == sequence*2-3 && wire.at("tick") == server.room().executed_ticks()-1 &&
+        wire.at("owner_epoch") == 0 && wire.at("state_hash") == hash && wire.size() == (full ? 12U : 11U),
+        "G08 exact wire shape, sequence, cadence and full canonical hash metadata");
+      require(wire.at("kind") == (full ? "FULL" : "DELTA") &&
+        wire.at("base_snapshot_seq") == (full ? Json(nullptr) : Json(sequence-1)), "G08 acknowledged base/fallback/mandatory full");
+      std::map<std::string,Json> applied_players; std::string status;
+      if (full) {
+        require(wire.at("removed_player_ids").empty(),"G08 FULL has no removals");
+        status = wire.at("status").get<std::string>();
+      } else {
+        const auto base = wire.at("base_snapshot_seq").get<std::uint64_t>();
+        require(replica.retained.contains(base),"G08 client owns the actual delta base");
+        status = replica.retained.at(base).at("status").get<std::string>();
+        for (const auto& player : replica.retained.at(base).at("players"))
+          applied_players.emplace(player.at("player_id").get<std::string>(),player);
+      }
+      std::string previous;
+      for (const auto& removed : wire.at("removed_player_ids")) {
+        const auto id = removed.get<std::string>();
+        require(id > previous && applied_players.erase(id) == 1,"G08 sorted explicit removal of an existing visible player");
+        previous = id;
+      }
+      previous.clear();
+      for (const auto& player : wire.at("players")) {
+        const auto id = player.at("player_id").get<std::string>();
+        require(id > previous && player.size() == 7 && player.contains("slot") && player.contains("x") && player.contains("y") &&
+          player.contains("direction") && player.contains("score") && player.at("connectivity") == "CONNECTED",
+          "G08 sorted exact seven-field visible player rows");
+        require(full || !applied_players.contains(id) || applied_players.at(id) != player,"G08 delta omits unchanged rows");
+        applied_players[id] = player; previous = id;
+      }
+      Json players = Json::array(); for (const auto& [id,player] : applied_players) { (void)id; players.push_back(player); }
+      replica.applied = Json{{"room_id",wire.at("room_id")},{"tick",wire.at("tick")},{"owner_epoch",wire.at("owner_epoch")},
+        {"status",status},{"players",players}};
+      require(replica.applied == authority,"G08 actual FULL/DELTA application equals independently captured visible state");
+      if (replica.retained.size() == snapshot_retention) replica.retained.erase(replica.retained.begin());
+      replica.retained.emplace(static_cast<std::uint64_t>(sequence),replica.applied);
+      cadence[role].push_back(Json{{"snapshot_seq",sequence},{"tick",wire.at("tick")},{"kind",wire.at("kind")},
+        {"base_snapshot_seq",wire.at("base_snapshot_seq")}});
+      if (role == "alpha") alpha_capture = Json{{"snapshot",wire},{"applied_visible",replica.applied},
+        {"authoritative_visible",authority},{"canonical_record",canonical},{"state_hash",hash}};
+      std::optional<std::uint64_t> ack_sequence;
+      for (const auto& rule : scenario.at("ack_schedule")) {
+        const bool applies = rule.contains("after_snapshot_seq") ? rule.at("after_snapshot_seq") == sequence :
+          sequence >= rule.at("after_snapshot_seq_range").at(0).get<int>() && sequence <= rule.at("after_snapshot_seq_range").at(1).get<int>();
+        if (!applies) continue;
+        require(!ack_sequence,"G08 exactly one frozen ACK rule");
+        require(rule.at("clients") == "all" || rule.at("clients") == "all still connected","G08 frozen ACK recipient scope");
+        if (rule.at("send_ack").is_string()) {
+          require(rule.at("send_ack") == "actual applied sequence","G08 ACK uses actual client application");
+          ack_sequence = wire.at("snapshot_seq").get<std::uint64_t>();
+        } else ack_sequence = rule.at("send_ack").get<std::uint64_t>();
+      }
+      require(ack_sequence.has_value(),"G08 actual applied sequence has ACK feedback");
+      auto ack = message("SNAPSHOT_ACK"); ack.update(Json{{"session_id",peer.session},{"room_id",server.room().id()},
+        {"player_id",peer.player},{"snapshot_seq",*ack_sequence}});
+      peer.tcp->send(ack); acks.push_back(Json{{"client",role},{"request",ack}});
+    }
+    const auto all_acks_owned = [&] {
+      const auto streams = server.metrics().at("snapshot_streams");
+      return std::all_of(acks.begin(),acks.end(),[&](const auto& row) {
+        return streams.at(row.at("request").at("player_id").template get<std::string>()).at("acknowledged_seq") ==
+          row.at("request").at("snapshot_seq");
+      });
+    };
+    const auto deadline = std::chrono::steady_clock::now() + std::chrono::seconds(5);
+    while (!all_acks_owned() && std::chrono::steady_clock::now() < deadline) server.pump(1);
+    require(all_acks_owned() && observed_state(server.room()) == state_before,"G08 real owner ACK receipt never changes simulation");
+    const auto streams = server.metrics().at("snapshot_streams");
+    for (auto& ack : acks) {
+      const auto& stream = streams.at(ack.at("request").at("player_id").get<std::string>());
+      auto retained_ids = Json::array(); for (int seq = std::max(1,sequence-31); seq <= sequence; ++seq) retained_ids.push_back(seq);
+      require(stream.at("retained_ids") == retained_ids && stream.at("retained_count") == retained_ids.size() &&
+        stream.at("high_water") <= snapshot_retention,"G08 actual per-client materialized retention IDs and bound");
+      ack["observed_acknowledged_seq"] = stream.at("acknowledged_seq");
+    }
+    alpha_capture["retention"] = streams.at(peers.at("alpha").player); captures.push_back(std::move(alpha_capture));
+    feedback.push_back(Json{{"after_snapshot_seq",sequence},{"feedback",acks}});
+  };
+  capture(1);
+  for (int tick = 0; tick < 196; ++tick) {
+    for (const auto& event : scenario.at("events")) {
+      if (event.at("before_tick") != tick) continue;
+      const auto role = event.at("client").get<std::string>(); auto& peer = peers.at(role);
+      const auto value = request(event,peers,server.room().id()), before = observed_state(server.room());
+      peer.tcp->send(value); Json response = nullptr;
+      if (event.at("kind") == "INPUT") {
+        response = peer.tcp->receive(server);
+        require(response.at("type") == "INPUT_ACK" && response.at("code") == "ACCEPTED" &&
+          response.at("seq") == event.at("seq") && response.at("tick") == tick,"G08 original actual input admission");
+      } else leave_barrier(peer,server);
+      require(clock.now_ms == tick*50 && server.room().executed_ticks() == tick,"G08 original admission/leave boundary");
+      events.push_back(Json{{"before_tick",tick},{"client",role},{"request",value},{"response",response},
+        {"before",before},{"after",observed_state(server.room())}});
+    }
+    server.advance_one_tick();
+    const auto& alpha = server.room().players().at("player-00"); const auto& delta = server.room().players().at("player-03");
+    require(server.room().executed_ticks() == tick+1 && clock.now_ms == (tick+1)*50 && session_ticks == 1200 &&
+      server.room().status() == "RUNNING" && alpha.x == 10000+std::min(tick+1,194)*400 && alpha.y == 10000 &&
+      alpha.score == (tick >= 193 ? 1 : 0) && alpha.last_tag_tick == (tick >= 193 ? 193 : -20) &&
+      alpha.direction == (tick >= 194 ? Direction::stop : Direction::east) && alpha.last_accepted_seq() == (tick >= 194 ? 3U : tick >= 193 ? 2U : 1U) &&
+      delta.connected == (tick < 194) && delta.x == 90000 && delta.y == 10000,"G08 unchanged authoritative movement/TAG/leave/duration");
+    const auto hash = server.replay().last_state_hash(); hashes.push_back(hash); tick_records.push_back(tick_record(server.room(),hash));
+    if ((tick+1)%2 == 0) capture((tick+1)/2+1);
+    // No unscheduled fallback, tick0 publication or leave/close debug notice.
+    server.pump();
+    require(server.cleanup().at("outbound_messages") == 0,"G08 all scheduled publications drained");
+    for (const auto& [role,peer] : peers) { (void)role; require(!peer.tcp->try_receive(),"G08 unexpected out-of-cadence wire message"); }
+  }
+  Json counts = Json::object(), final_applied = Json::object();
+  for (const auto& [role,replica] : replicas) {
+    counts[role] = replica.count; final_applied[role] = replica.applied;
+    require(replica.count == (role == "delta" ? 98 : 99),"G08 exact per-client publication count");
+  }
+  require(events.size() == 4 && captures.size() == 99 && captures.back().at("snapshot").at("removed_player_ids") == Json::array({"player-03"}),
+          "G08 every frozen event and explicit visible removal");
+  const auto final = observed_state(server.room()), metrics = server.metrics();
+  require(metrics.at("snapshot_retention_high_water") == 32 && metrics.at("snapshot_streams").at("player-03").at("retained_count") == 0,
+          "G08 exact retention high-water and departed stream release");
+  auto descriptors = server.owned_descriptors();
+  for (const auto& [role,peer] : peers) { (void)role; descriptors.push_back(peer.tcp->descriptor()); }
+  server.shutdown(); Json eof = Json::object();
+  for (const auto& [role,peer] : peers) {
+    const auto deadline = std::chrono::steady_clock::now() + std::chrono::seconds(5);
+    while (!peer.tcp->peer_closed() && std::chrono::steady_clock::now() < deadline)
+      require(!peer.tcp->try_receive(),"G08 shutdown emits no out-of-stream notice");
+    require(peer.tcp->peer_closed(),"G08 actual terminal TCP EOF"); eof[role] = true; peer.tcp->close();
+  }
+  for (int fd : descriptors) require(descriptor_closed(fd),"G08 actual descriptor closure");
+  auto cleanup = server.cleanup();
+  for (const auto& [key,value] : cleanup.items()) { (void)key; require(value == 0,"G08 all active resources released"); }
+  require(Fd::live() == descriptors_before,"G08 tracked descriptor delta");
+  cleanup["descriptor_checks"] = descriptors.size(); cleanup["all_descriptors_closed"] = true;
+  cleanup["tracked_descriptor_delta"] = Fd::live()-descriptors_before;
+  return Json{{"result","PASS"},{"thread","G08"},{"scenario_id",scenario.at("scenario_id")},{"process_id",::getpid()},
+    {"executed_ticks",196},{"initial_bindings",bindings},{"initial_state",initial},{"events",events},
+    {"captures",captures},{"ack_feedback",feedback},{"client_cadence",cadence},{"snapshot_counts",counts},
+    {"tick_records",tick_records},{"state_hashes",hashes},{"final_applied",final_applied},{"final_state",final},
+    {"metrics",metrics},{"cleanup",cleanup},{"client_eof",eof},{"all_resources_released",true}};
+}
+
 ReplayRun verify_replay(const Json& artifact) {
   // The CLI's bounded parser has validated completeness and record framing.
   const int descriptors_before = Fd::live(); Room room; ReplayFixture::offline(room,artifact);
diff --git a/tests/g07.hpp b/tests/g07.hpp
index 3572c52..51ac821 100644
--- a/tests/g07.hpp
+++ b/tests/g07.hpp
@@ -9,4 +9,5 @@ struct ReplayRun {
 };
 ReplayRun run_replay_scenario(const Json& scenario, bool rejected_removed);
 ReplayRun verify_replay(const Json& artifact);
+Json run_snapshot_scenario(const Json& scenario);
 }
diff --git a/tests/scenario_main.cpp b/tests/scenario_main.cpp
index 1198f7e..65f4e17 100644
--- a/tests/scenario_main.cpp
+++ b/tests/scenario_main.cpp
@@ -32,7 +32,7 @@ int main(int argc, char** argv) {
       const auto scenario = arena::read_json_file(input);
       if (scenario.at("thread") != "G07") {
         if (variant) throw std::invalid_argument("variant is only active for G07");
-        const auto evidence = arena::run_scenario(scenario);
+        const auto evidence = scenario.at("thread") == "G08" ? arena::run_snapshot_scenario(scenario) : arena::run_scenario(scenario);
         arena::write_json_file(output,evidence);
         std::cout << arena::Json{{"result",evidence.at("result")},{"executed_ticks",evidence.at("executed_ticks")},
           {"scenario_id",evidence.at("scenario_id")},{"evidence",output.string()},{"cleanup",evidence.at("cleanup")}}.dump() << '\n';
diff --git a/tests/tests.cpp b/tests/tests.cpp
index 6622039..6b61385 100644
--- a/tests/tests.cpp
+++ b/tests/tests.cpp
@@ -616,6 +616,41 @@ void replay_storage_and_packaging_without_ticks() {
     {"high_water_bytes",high_water},{"bound",max_replay_bytes},{"failure",log.failure()},
     {"export_refused",refused},{"executed_ticks",0},{"released_bytes",log.bytes()}}}}.dump() << '\n';
 }
+void snapshot_retention_without_ticks() {
+  Room room; populate(room); SnapshotStream stream;
+  const auto record = canonical_state(room), hash = sha256(record);
+  Json captures = Json::array();
+  for (int seq = 1; seq <= 33; ++seq) {
+    const auto snapshot = stream.publish(room,hash);
+    check(snapshot.at("snapshot_seq") == seq && snapshot.at("tick") == -1 && snapshot.at("state_hash") == hash,
+          "snapshot sequence and full canonical hash metadata");
+    const bool full = seq <= 4 || seq == 20;
+    check(snapshot.at("kind") == (full ? "FULL" : "DELTA") &&
+      snapshot.at("base_snapshot_seq") == (full ? Json(nullptr) : Json(seq-1)), "only ACK known when received is a base; twentieth full");
+    if (full) {
+      check(snapshot.at("status") == "RUNNING" && snapshot.at("players").size() == 2, "full visible state");
+      for (const auto& player : snapshot.at("players"))
+        check(player.size() == 7 && player.contains("player_id") && player.contains("slot") && player.contains("x") &&
+          player.contains("y") && player.contains("direction") && player.contains("score") && player.contains("connectivity"),
+          "exact seven visible player fields");
+    } else check(snapshot.at("players").empty(), "unchanged players absent from delta");
+    check(snapshot.at("removed_player_ids").empty() && encode_frame(snapshot).size() <= max_frame_bytes+4,
+          "snapshot packaging is bounded");
+    check(stream.size() == std::min(static_cast<std::size_t>(seq),snapshot_retention) && stream.high_water() <= 32,
+          "materialized retention never exceeds32");
+    captures.push_back(Json{{"snapshot_seq",seq},{"kind",snapshot.at("kind")},{"base_snapshot_seq",snapshot.at("base_snapshot_seq")},
+      {"retained_ids",stream.metrics().at("retained_ids")}});
+    if (seq == 1) stream.acknowledge(3); // Unknown now; publishing3 must not validate this old ACK.
+    else if (seq >= 4) stream.acknowledge(static_cast<std::uint64_t>(seq));
+  }
+  auto expected_ids = Json::array(); for (int seq = 2; seq <= 33; ++seq) expected_ids.push_back(seq);
+  check(stream.metrics().at("retained_ids") == expected_ids && canonical_state(room) == record && room.executed_ticks() == 0,
+        "33 publications evict only oldest1 without simulation");
+  const auto retained = stream.metrics(); stream.clear(); room.close();
+  check(stream.size() == 0 && stream.metrics().at("acknowledged_seq").is_null(), "retained state and ACK released");
+  std::cout << Json{{"G08_zero_tick_retention",Json{{"captures",captures},{"before_clear",retained},
+    {"after_clear",stream.metrics()},{"publications",33},{"executed_ticks",0}}}}.dump() << '\n';
+}
 void real_tcp_authority_and_cleanup() {
   const auto scenario = Json::parse(R"({
     "scenario_id":"G01-three-tick-authority-smoke","contract_version":1,"thread":"G01","seed":7050,
@@ -635,9 +670,11 @@ void real_tcp_authority_and_cleanup() {
   check(evidence.at("executed_ticks") == 3 && evidence.at("manual_clock_ms") == 150, "injected clock drove actual server");
   check(evidence.at("cleanup").at("descriptor_checks") == 6 && evidence.at("cleanup").at("all_descriptors_closed") == true,
         "listener, kqueue, two accepted and two client descriptors closed");
-  for (const auto* role : {"alpha", "bravo"}) {
-    check(evidence.at("client_lifecycle").at(role) == Json::array({"LOBBY", "RUNNING", "CLOSED"}), "lifecycle observed over TCP");
-  }
+  check(evidence.at("lifecycle") == Json::array({"LOBBY","RUNNING","CLOSED"}), "full authoritative lifecycle preserved");
+  check(evidence.at("client_lifecycle").at("alpha") == Json::array({"LOBBY","RUNNING"}) &&
+    evidence.at("client_lifecycle").at("bravo") == Json::array({"RUNNING"}), "actual create/join/start wire lifecycle");
+  for (const auto* role : {"alpha", "bravo"})
+    check(evidence.at("client_eof").at(role) == true, "each TCP client observed owner shutdown EOF");
   std::cout << Json{{"integration_evidence", evidence}}.dump() << '\n';
 }
 void standalone_process_shutdown(const std::filesystem::path& executable, const std::string& clock_mode = "manual") {
@@ -759,7 +796,8 @@ int main(int argc, char** argv) {
       {"G06_shared_victim_ASCII_order", shared_victim_ascii_order},
       {"G06_pending_bound_after_rate", pending_bound_after_rate_activation},
       {"G07_zero_tick_canonical_SHA256", canonical_hash_bytes_without_ticks},
-      {"G07_zero_tick_storage_packaging", replay_storage_and_packaging_without_ticks}};
+      {"G07_zero_tick_storage_packaging", replay_storage_and_packaging_without_ticks},
+      {"G08_zero_tick_33_snapshot_retention", snapshot_retention_without_ticks}};
   } else if (std::string(argv[1]) == "integration") {
     tests = {{"real_TCP_authority_and_cleanup", real_tcp_authority_and_cleanup}, {"standalone_process_shutdown", [&] {
       standalone_process_shutdown(std::filesystem::absolute(argv[0]).parent_path() / "arena"); }},
