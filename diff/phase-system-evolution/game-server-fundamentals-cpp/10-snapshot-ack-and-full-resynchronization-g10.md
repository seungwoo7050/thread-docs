# Snapshot ACK와 Full Resynchronization (G10)

## `feat(snapshot): validate monotonic acknowledgements and schedule full resync`

diff --git a/CMakeLists.txt b/CMakeLists.txt
index 3788f31..8323a7c 100644
--- a/CMakeLists.txt
+++ b/CMakeLists.txt
@@ -34,7 +34,7 @@ target_compile_options(arena PRIVATE -Wall -Wextra -Wpedantic -Werror)
 add_executable(arena_tests tests/tests.cpp tests/g09.cpp)
 target_link_libraries(arena_tests PRIVATE arena_test_core)
 target_compile_options(arena_tests PRIVATE -Wall -Wextra -Wpedantic -Werror)
-add_executable(arena_scenarios tests/scenario_main.cpp tests/g07.cpp tests/g09.cpp)
+add_executable(arena_scenarios tests/scenario_main.cpp tests/g07.cpp tests/g09.cpp tests/g10.cpp)
 target_link_libraries(arena_scenarios PRIVATE arena_test_core)
 target_compile_options(arena_scenarios PRIVATE -Wall -Wextra -Wpedantic -Werror)
 enable_testing()
diff --git a/evidence/G10-runs.jsonl b/evidence/G10-runs.jsonl
new file mode 100644
index 0000000..2c53506
--- /dev/null
+++ b/evidence/G10-runs.jsonl
@@ -0,0 +1,6 @@
+{"label":"baseline-compile","category":"compile","units":1,"ticks":0,"ceiling_seconds":180,"argv":["clang++","-std=c++20","-O2","-Wall","-Wextra","-Wpedantic","-Werror","-fsanitize=thread","-g","-DARENA_TEST_FIXTURES=1","-I","src","-I","tests","-I","/opt/homebrew/include","artifacts/g10/reproduce.cpp","src/game.cpp","src/transport.cpp","src/replay.cpp","src/snapshot.cpp","-o","artifacts/g10/reproduce"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g10/baseline-compile.log","started_at":"2026-08-28T06:05:09.913276+00:00","duration_seconds":17.749821,"exit":0,"wrapper_pid":21743,"child_pid":21752,"timed_out":false}
+{"label":"baseline","category":"unit","units":1,"ticks":14,"ceiling_seconds":120,"argv":["env","TSAN_OPTIONS=halt_on_error=1","./artifacts/g10/reproduce","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G10.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g10/baseline.json"],"expected_exit":1,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g10/baseline.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g10/baseline.json","started_at":"2026-08-28T06:06:39.862305+00:00","duration_seconds":0.997522,"exit":1,"wrapper_pid":22374,"child_pid":22383,"timed_out":false,"observed_ticks":14,"runtime_pid":22383}
+{"label":"build","category":"compile","units":2,"ticks":0,"ceiling_seconds":180,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g10-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g10/track","TSAN_OPTIONS=halt_on_error=1","ARENA_TSAN=ON","./track","build"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g10/build.log","started_at":"2026-08-28T06:10:59.264999+00:00","duration_seconds":39.424652,"exit":0,"wrapper_pid":24609,"child_pid":24619,"timed_out":false}
+{"label":"unit","category":"unit","units":1,"ticks":0,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g10-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g10/track","TSAN_OPTIONS=halt_on_error=1","./track","unit-test"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g10/unit.log","started_at":"2026-08-28T06:12:38.824076+00:00","duration_seconds":2.4436,"exit":0,"wrapper_pid":25892,"child_pid":25893,"timed_out":false}
+{"label":"integration","category":"integration","units":1,"ticks":0,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g10-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g10/track","TSAN_OPTIONS=halt_on_error=1","./track","integration-test"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g10/integration.log","started_at":"2026-08-28T06:12:41.354830+00:00","duration_seconds":1.422374,"exit":0,"wrapper_pid":25932,"child_pid":25933,"timed_out":false}
+{"label":"canonical","category":"canonical","units":1,"ticks":14,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g10-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g10/track","TSAN_OPTIONS=halt_on_error=1","./track","scenario-run","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G10.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g10/canonical.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g10/canonical.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g10/canonical.json","started_at":"2026-08-28T06:12:42.876790+00:00","duration_seconds":0.881118,"exit":0,"wrapper_pid":25944,"child_pid":25945,"timed_out":false,"observed_ticks":14,"runtime_pid":25951}
diff --git a/evidence/G10.md b/evidence/G10.md
new file mode 100644
index 0000000..bbb5425
--- /dev/null
+++ b/evidence/G10.md
@@ -0,0 +1,24 @@
+# G10 — ACK validation and scheduled full resynchronization
+
+START `c5723975b6aac999e7240b7a5630c04c8a70bd12`; profile `realtime-core`; Spec-Revision `c1d62196ab76b55652f5d75a67514f8c6d8210ce`. Frozen G10 fixture SHA-256: `8f8a9d8d55a092aab97a6923936c17d9adee5333e537488720efdb9e9f40afe2`.
+
+Exact commands, ceilings, processes, durations and exits: `evidence/G10-runs.jsonl`. Resolved before execution: `artifacts/g10/commands.json`. The 15 unchanged START source hashes are in `artifacts/g10/pre-change-production.json`; the preserved baseline helper and driver are `artifacts/g10/reproduce.cpp` and `driver.inc`. The committed G10 driver retains that exact driver body, using ordinary four-player joins/binds and only the existing test-build ID allocator.
+
+| Actual raw result | Exit | Bytes | SHA-256 |
+| --- | ---: | ---: | --- |
+| `artifacts/g10/baseline.json` | 1 | 211962 | `b1c3236e89dc2118c1630c0f116b7b4573e03bd69c1f2c6b75d2cdcfe7b5774a` |
+| `artifacts/g10/canonical.json` | 0 | 219147 | `a0f9e92d7942267481d42981c76c2af62b604056f86bc17f3332bc5918f80473` |
+
+Baseline: actual G09 core ran all 14 ticks and eight publications. Loss recovery via DELTA 3/base 1 already worked (`NOT_REPRODUCED`). Eleven observed ACK/resync policy violations included adopting unknown 999, rolling back to retained 1, ignoring hash mismatch and ignoring missing-base feedback. No gameplay failure was invented.
+
+Production changes only validate retained ACK/hash, preserve the highest valid watermark, and latch a bounded reason for the next scheduled FULL. The existing authenticated UDP owner path reads the two optional fields. The G08 harness now observes actual ingress/owner completion instead of expecting unknown 999 to become a valid watermark; its physical expectations are unchanged.
+
+Post result: `PASS`, no violations. Actual observer sequence: `1 FULL; 2 DELTA/base1 DROP; 3 DELTA/base1 apply; 4 FULL UNKNOWN_ACK; 5 DELTA/base4; 6 FULL HASH_MISMATCH; 7 DELTA/base6 refuse; 8 FULL RESYNC_REQUIRED`. Applied sequence and server watermark finish at 8. Special ACKs preserve watermarks `3→3`, `4→4`, `5→5`, `6→6`; every ACK and client-only base discard preserves authoritative state. Raw results contain all wire objects, ACK requests with session aliases, before/after states, replica states, 14 canonical records and hashes.
+
+All 14 post authoritative records/hashes equal baseline. Final alpha `(15600,10000)`, EAST, accepted sequence 1, score 0; other players remain at spawn with score 0. Final hash `5af6ca04d0d1c1a4bbdfbd9452de2d1146426a4bdc793e1fa231d14c0a991c8c`. SHA-256 of newline-joined 14 hashes with final newline: `648e8803100ebe259201a845574104d805e5be3e941a1c805bfa7a4009c9cbf8`.
+
+The existing zero-tick unit's exact 33 publications retain 2–33 (high-water 32); expired ACK1 leaves ACK33 and latches `ACK_OUTSIDE_RETENTION`. No extra publication/campaign. TSan build, unit **24/24**, integration **4/4** (existing 11-case UDP matrix once), and canonical **14 ticks** passed. Logs: `artifacts/g10/{build,unit,integration,canonical}.log`. Live high-waters: mailbox 1, pending/attempt 1, retention 8, outbound UDP 711 bytes. All 16 actual descriptors closed; every active resource counter and tracked descriptor delta is zero. Raw baseline/post contain no string token fields.
+
+Budget consumed: compile/configure **3/8** (baseline compile 1 + configure/build 2), unit **2/4** (baseline + full suite), integration **1/2**, post canonical **1/1**. Exactly one baseline and one post fault campaign, each 14 ticks; offline/load 0. No compiler failures, timeouts, runtime reruns or repairs. Prior regression simulations remain in standard suites; neither suite runs the G10 campaign.
+
+Static checks: `git diff --check` passed; gameplay/replay files remain byte-identical to START; shipping binary has no fixture/observer symbols and no `ARENA_TEST_FIXTURES` definition. No dependencies, future-stage behavior, progress tags or push.
diff --git a/src/snapshot.cpp b/src/snapshot.cpp
index 65dfc44..b55967d 100644
--- a/src/snapshot.cpp
+++ b/src/snapshot.cpp
@@ -2,13 +2,21 @@
 #include <algorithm>
 
 namespace arena {
-void SnapshotStream::acknowledge(std::uint64_t sequence) {
-  acknowledged_ = sequence;
-  // A sequence not yet sent cannot become an acknowledged base merely by
-  // being published later; only this actual feedback establishes a base.
-  acknowledged_known_ = std::any_of(retained_.begin(),retained_.end(),[&](const auto& entry) {
+void SnapshotStream::acknowledge(std::uint64_t sequence, const std::optional<std::string>& state_hash, bool resync_required) {
+  const auto snapshot = std::find_if(retained_.begin(),retained_.end(),[&](const auto& entry) {
     return entry.at("snapshot_seq") == sequence;
   });
+  if (snapshot == retained_.end()) {
+    resync_reason_ = sequence == 0 || sequence > sequence_ ? "UNKNOWN_ACK" : "ACK_OUTSIDE_RETENTION";
+    return;
+  }
+  if (state_hash && snapshot->at("state_hash") != *state_hash) {
+    resync_reason_ = "HASH_MISMATCH"; return;
+  }
+  if (resync_required) resync_reason_ = "RESYNC_REQUIRED";
+  // Only a retained publication with matching optional hash can advance the
+  // watermark. A late retained ACK cannot roll the selected base backwards.
+  if (!acknowledged_ || sequence > *acknowledged_) acknowledged_ = sequence;
 }
 Json SnapshotStream::publish(const Room& room, const std::string& state_hash) {
   Json players = Json::array();
@@ -23,9 +31,15 @@ Json SnapshotStream::publish(const Room& room, const std::string& state_hash) {
     {"players",players},{"removed_player_ids",Json::array()},{"status",room.status()}});
   auto wire = full;
   const auto base = std::find_if(retained_.begin(),retained_.end(),[&](const auto& entry) {
-    return acknowledged_known_ && acknowledged_ && entry.at("snapshot_seq") == *acknowledged_;
+    return acknowledged_ && entry.at("snapshot_seq") == *acknowledged_;
   });
-  if (sequence_ % 20 != 0 && room.status() == "RUNNING" && base != retained_.end()) {
+  if (!resync_reason_.empty()) last_full_reason_ = resync_reason_;
+  else if (sequence_ == 1) last_full_reason_ = "ROOM_START";
+  else if (sequence_ % 20 == 0) last_full_reason_ = "PERIODIC";
+  else if (room.status() != "RUNNING") last_full_reason_ = "ROOM_STATUS";
+  else if (base == retained_.end()) last_full_reason_ = "NO_RETAINED_BASE";
+  else last_full_reason_.clear();
+  if (last_full_reason_.empty()) {
     wire["kind"] = "DELTA"; wire["base_snapshot_seq"] = *acknowledged_;
     wire.erase("status"); wire["players"] = Json::array();
     for (const auto& player : players) {
@@ -42,12 +56,14 @@ Json SnapshotStream::publish(const Room& room, const std::string& state_hash) {
   if (retained_.size() == snapshot_retention) retained_.pop_front();
   retained_.push_back(std::move(full));
   high_water_ = std::max(high_water_,retained_.size());
+  resync_reason_.clear(); // The required full is consumed only at publication.
   return wire;
 }
 Json SnapshotStream::metrics() const {
   auto ids = Json::array();
   for (const auto& entry : retained_) ids.push_back(entry.at("snapshot_seq"));
   return Json{{"last_seq",sequence_},{"acknowledged_seq",acknowledged_ ? Json(*acknowledged_) : Json(nullptr)},
-    {"retained_ids",ids},{"retained_count",retained_.size()},{"high_water",high_water_}};
+    {"retained_ids",ids},{"retained_count",retained_.size()},{"high_water",high_water_},
+    {"resync_pending",!resync_reason_.empty()},{"resync_reason",resync_reason_},{"last_full_reason",last_full_reason_}};
 }
 }
diff --git a/src/snapshot.hpp b/src/snapshot.hpp
index 0f8e076..4c05989 100644
--- a/src/snapshot.hpp
+++ b/src/snapshot.hpp
@@ -9,15 +9,16 @@ inline constexpr std::size_t snapshot_retention = 32;
 class SnapshotStream {
  public:
   Json publish(const Room& room, const std::string& state_hash);
-  void acknowledge(std::uint64_t sequence);
-  void clear() { retained_.clear(); acknowledged_.reset(); acknowledged_known_ = false; }
+  void acknowledge(std::uint64_t sequence, const std::optional<std::string>& state_hash = std::nullopt, bool resync_required = false);
+  void clear() { retained_.clear(); acknowledged_.reset(); resync_reason_.clear(); last_full_reason_.clear(); }
   std::size_t size() const { return retained_.size(); }
   std::size_t high_water() const { return high_water_; }
   Json metrics() const;
  private:
   std::uint64_t sequence_ = 0;
   std::optional<std::uint64_t> acknowledged_;
-  bool acknowledged_known_ = false;
+  std::string resync_reason_;
+  std::string last_full_reason_;
   std::deque<Json> retained_;
   std::size_t high_water_ = 0;
 };
diff --git a/src/transport.cpp b/src/transport.cpp
index 4433974..2207950 100644
--- a/src/transport.cpp
+++ b/src/transport.cpp
@@ -571,7 +571,11 @@ void Server::handle(const Envelope& envelope) {
       const auto& sequence = value.at("snapshot_seq");
       if (!sequence.is_number_integer() || (!sequence.is_number_unsigned() && sequence.get<std::int64_t>() < 0))
         throw std::invalid_argument("snapshot_seq must be an exact nonnegative integer");
-      conn->snapshots.acknowledge(sequence.get<std::uint64_t>()); return;
+      std::optional<std::string> state_hash;
+      if (value.contains("state_hash")) state_hash = value.at("state_hash").get<std::string>();
+      if (value.contains("resync_required") && !value.at("resync_required").is_boolean())
+        throw std::invalid_argument("resync_required must be boolean");
+      conn->snapshots.acknowledge(sequence.get<std::uint64_t>(),state_hash,value.value("resync_required",false)); return;
     }
     const auto result = admit_input(room_, conn->player_id, value);
     if (result.error) {
diff --git a/tests/g07.cpp b/tests/g07.cpp
index 9c5fedf..287a796 100644
--- a/tests/g07.cpp
+++ b/tests/g07.cpp
@@ -270,6 +270,7 @@ Json run_snapshot_scenario(const Json& scenario) {
     const auto authority = visible();
     if (server.room().executed_ticks() != 0) require(server.replay().last_state_hash() == hash,"G08 canonical replay/hash equality");
     Json alpha_capture, acks = Json::array();
+    const auto incoming_before = server.metrics().at("udp_received_datagrams").get<std::uint64_t>();
     for (auto& [role,peer] : peers) {
       if (!server.room().players().at(peer.player).connected) continue;
       auto& replica = replicas[role]; const auto wire = peer.udp->receive(server);
@@ -332,13 +333,21 @@ Json run_snapshot_scenario(const Json& scenario) {
       require(ack_sequence.has_value(),"G08 actual applied sequence has ACK feedback");
       auto ack = message("SNAPSHOT_ACK"); ack.update(Json{{"session_id",peer.session},{"room_id",server.room().id()},
         {"player_id",peer.player},{"snapshot_seq",*ack_sequence},{"owner_epoch",0}});
-      peer.udp->send(ack); acks.push_back(Json{{"client",role},{"request",ack}});
+      const auto stream = server.metrics().at("snapshot_streams").at(peer.player);
+      auto expected_watermark = stream.at("acknowledged_seq");
+      const bool retained = std::any_of(stream.at("retained_ids").begin(),stream.at("retained_ids").end(),
+        [&](const auto& id) { return id == *ack_sequence; });
+      if (retained && (expected_watermark.is_null() || Json(*ack_sequence) > expected_watermark)) expected_watermark = *ack_sequence;
+      peer.udp->send(ack); acks.push_back(Json{{"client",role},{"request",ack},{"expected_valid_watermark",expected_watermark}});
     }
     const auto all_acks_owned = [&] {
       const auto streams = server.metrics().at("snapshot_streams");
-      return std::all_of(acks.begin(),acks.end(),[&](const auto& row) {
+      // Unknown999 no longer becomes a valid watermark. Actual ingress and
+      // owner completion still prove that its fallback request was consumed.
+      return server.metrics().at("udp_received_datagrams") == incoming_before+acks.size() && server.cleanup().at("mailbox_messages") == 0 &&
+        std::all_of(acks.begin(),acks.end(),[&](const auto& row) {
         return streams.at(row.at("request").at("player_id").template get<std::string>()).at("acknowledged_seq") ==
-          row.at("request").at("snapshot_seq");
+          row.at("expected_valid_watermark");
       });
     };
     const auto deadline = std::chrono::steady_clock::now() + std::chrono::seconds(5);
diff --git a/tests/g09.cpp b/tests/g09.cpp
index c1bc382..1d9bd0d 100644
--- a/tests/g09.cpp
+++ b/tests/g09.cpp
@@ -249,6 +249,7 @@ struct Fixture {
   }
 };
 }
+void inject_udp_fixture_ids(Server& server, const Json& scenario) { UdpFixture::identifiers(server,scenario); }
 Json run_udp_matrix() {
   const std::array<std::string,11> names{"valid-token-before-deadline","expired-token-at-deadline","unknown-token",
     "reused-consumed-token","other-session-token","INPUT-before-bind","INPUT-from-different-observed-endpoint",
diff --git a/tests/g09.hpp b/tests/g09.hpp
index 0f9b37c..9a5b5f0 100644
--- a/tests/g09.hpp
+++ b/tests/g09.hpp
@@ -1,6 +1,8 @@
 #pragma once
 #include "g07.hpp"
 namespace arena {
+// The allocator is compiled only into the test executable, never shipping core.
+void inject_udp_fixture_ids(Server& server, const Json& scenario);
 Json run_udp_matrix();
 ReplayRun run_udp_scenario(const Json& scenario);
 }
diff --git a/tests/g10.cpp b/tests/g10.cpp
new file mode 100644
index 0000000..42cfaa3
--- /dev/null
+++ b/tests/g10.cpp
@@ -0,0 +1,273 @@
+#include "g10.hpp"
+#ifndef ARENA_TEST_FIXTURES
+#error G10 observer fixture is test-build only
+#endif
+#include <algorithm>
+#include <array>
+#include <cerrno>
+#include <chrono>
+#include <fcntl.h>
+#include <memory>
+#include <sys/socket.h>
+#include <unistd.h>
+
+namespace arena {
+namespace {
+void ack_need(bool value, const std::string& description) { if (!value) throw std::runtime_error("G10: "+description); }
+Json ack_state(const Room& room) {
+  auto value = room.view(); value["owner_epoch"] = 0;
+  for (auto& row : value["players"]) {
+    const auto& p = room.players().at(row.at("player_id").get<std::string>());
+    row["last_seq"] = p.last_accepted_seq(); row["pending"] = p.pending.size();
+    row["applied_seq"] = p.applied_seq ? Json(*p.applied_seq) : Json(nullptr);
+  }
+  return value;
+}
+Json ack_visible(const Room& room) {
+  Json players = Json::array();
+  for (const auto& [id,p] : room.players()) if (p.connected)
+    players.push_back(Json{{"player_id",id},{"slot",p.slot},{"x",p.x},{"y",p.y},
+      {"direction",direction_name(p.direction)},{"score",p.score},{"connectivity","CONNECTED"}});
+  return Json{{"room_id",room.id()},{"tick",room.executed_ticks()-1},{"owner_epoch",0},{"status",room.status()},{"players",players}};
+}
+class AckDropProxy {
+ public:
+  explicit AckDropProxy(std::uint16_t server_port) : fd_(::socket(AF_INET,SOCK_DGRAM,0)) {
+    ack_need(fd_.get() >= 0,"drop proxy socket"); const int flags = ::fcntl(fd_.get(),F_GETFL,0);
+    ack_need(flags >= 0 && ::fcntl(fd_.get(),F_SETFL,flags|O_NONBLOCK) == 0 && ::fcntl(fd_.get(),F_SETFD,FD_CLOEXEC) == 0,"nonblocking proxy");
+    sockaddr_in local{}; local.sin_family = AF_INET; local.sin_addr.s_addr = htonl(INADDR_LOOPBACK);
+    ack_need(::bind(fd_.get(),reinterpret_cast<sockaddr*>(&local),sizeof(local)) == 0,"proxy bind");
+    socklen_t size = sizeof(local); ack_need(::getsockname(fd_.get(),reinterpret_cast<sockaddr*>(&local),&size) == 0,"proxy port");
+    port_ = ntohs(local.sin_port); server_ = local; server_.sin_port = htons(server_port);
+  }
+  std::uint16_t port() const { return port_; }
+  int descriptor() const { return fd_.get(); }
+  void close() { fd_.reset(); }
+  const Json& snapshots() const { return snapshots_; }
+  const Json& trace() const { return trace_; }
+  int delivered() const { return delivered_; }
+  void pump() {
+    for (int work = 0; work < 64; ++work) {
+      std::array<std::uint8_t,max_datagram_bytes+1> bytes{}; sockaddr_in source{}; socklen_t size = sizeof(source);
+      const auto count = ::recvfrom(fd_.get(),bytes.data(),bytes.size(),0,reinterpret_cast<sockaddr*>(&source),&size);
+      if (count < 0) { ack_need(errno == EAGAIN || errno == EWOULDBLOCK || errno == EINTR,"proxy receive"); return; }
+      ack_need(count > 0 && count <= static_cast<ssize_t>(max_datagram_bytes),"proxy bounded input");
+      const bool response = same(source,server_);
+      if (!response) { if (!client_) client_ = source; ack_need(same(source,*client_),"one alpha proxy endpoint"); }
+      ack_need(client_.has_value(),"observed client before response");
+      const auto value = decode_datagram(std::span(bytes).first(static_cast<std::size_t>(count)));
+      const bool snapshot = response && value.at("type") == "SNAPSHOT";
+      if (snapshot) {
+        ack_need(snapshots_.size() < 8 && value.at("snapshot_seq") == snapshots_.size()+1,"only eight original alpha snapshots");
+        snapshots_.push_back(value);
+        if (snapshots_.size() == 2) { trace_.push_back(Json{{"snapshot_seq",2},{"event","DROP_ONCE"},{"bytes",count}}); continue; }
+      }
+      const auto& destination = response ? *client_ : server_;
+      ack_need(::sendto(fd_.get(),bytes.data(),static_cast<std::size_t>(count),0,
+        reinterpret_cast<const sockaddr*>(&destination),sizeof(destination)) == count,"actual proxy forwarding");
+      if (snapshot) { ++delivered_; trace_.push_back(Json{{"snapshot_seq",value.at("snapshot_seq")},{"event","DELIVER"},{"bytes",count}}); }
+      // UDP_BIND is forwarded but its credential-bearing object is never retained.
+    }
+  }
+ private:
+  static bool same(const sockaddr_in& a, const sockaddr_in& b) {
+    return a.sin_family == b.sin_family && a.sin_addr.s_addr == b.sin_addr.s_addr && a.sin_port == b.sin_port;
+  }
+  Fd fd_; std::uint16_t port_ = 0; sockaddr_in server_{}; std::optional<sockaddr_in> client_;
+  int delivered_ = 0; Json snapshots_ = Json::array(), trace_ = Json::array();
+};
+struct AckPeer {
+  std::unique_ptr<TcpClient> tcp;
+  std::unique_ptr<UdpClient> udp;
+  std::string session, player, role;
+  std::uint64_t latest = 0;
+  std::map<std::uint64_t,Json> retained;
+  Json applied;
+};
+}
+Json run_ack_scenario(const Json& scenario) {
+  ack_need(scenario.at("thread") == "G10" && scenario.at("contract_version") == 1 && scenario.at("seed") == 7050 &&
+    scenario.at("ticks") == 14 && scenario.at("players").size() == 4 && scenario.at("events").size() == 1 &&
+    scenario.at("observer") == "alpha" && scenario.at("clock").at("kind") == "manual" &&
+    scenario.at("clock").at("tick_duration_ms") == 50 && scenario.at("socket_ceiling_ms") == 5000,"frozen G10 dimensions");
+  const int fd_before = Fd::live(); ManualClock clock; Server server(clock,0,[&] { return clock.now_ms; });
+  inject_udp_fixture_ids(server,scenario); AckDropProxy proxy(server.udp_port()); std::array<AckPeer,4> peers;
+  Json failures = Json::array(), ack_trace = Json::array(), observer = Json::array(), joins = Json::array(), bindings = Json::array(), local_faults = Json::array();
+  const auto expect = [&](bool value, const std::string& guarantee, const Json& actual = nullptr) {
+    if (!value) failures.push_back(Json{{"guarantee",guarantee},{"actual",actual}});
+  };
+  const auto wait_for = [&](const auto& ready, const std::string& description) {
+    const auto deadline = std::chrono::steady_clock::now()+std::chrono::seconds(5);
+    do { proxy.pump(); server.pump(); proxy.pump(); if (ready()) return; } while (std::chrono::steady_clock::now() < deadline);
+    throw std::runtime_error("G10 socket/owner barrier: "+description);
+  };
+  for (std::size_t i = 0; i < peers.size(); ++i) {
+    auto& p = peers[i]; p.role = scenario.at("players").at(i).at("client").get<std::string>();
+    p.tcp = std::make_unique<TcpClient>(server.port()); p.tcp->send(message("HELLO"));
+    const auto welcome = p.tcp->receive_type(server,"WELCOME"); p.session = welcome.at("session_id").get<std::string>();
+    ack_need(p.tcp->has_bind_token() && !welcome.contains("udp_bind_token"),"private token capture/redaction");
+    p.udp = std::make_unique<UdpClient>(i == 0 ? proxy.port() : server.udp_port());
+  }
+  auto create = message("CREATE_ROOM"); create["session_id"] = peers[0].session; peers[0].tcp->send(create);
+  const auto room_id = peers[0].tcp->receive_type(server,"ROOM_CREATED").at("room_id").get<std::string>();
+  ack_need(scenario.at("room_id") == room_id,"normal fixed room allocation");
+  for (std::size_t i = 0; i < peers.size(); ++i) {
+    auto& p = peers[i]; auto join = message("JOIN_ROOM"); join["session_id"] = p.session; join["room_id"] = room_id;
+    p.tcp->send(join); const auto result = p.tcp->receive_type(server,"ROOM_JOINED"); p.player = result.at("player_id").get<std::string>();
+    ack_need(result.at("status") == "LOBBY" && server.room().status() == "LOBBY" && result.at("slot") == i &&
+      scenario.at("players").at(i).at("player_id") == p.player && server.room().executed_ticks() == 0,"four ordinary unbound joins");
+    joins.push_back(Json{{"client",p.role},{"player_id",p.player},{"slot",i},{"status",result.at("status")}});
+  }
+  for (std::size_t i = 0; i < peers.size(); ++i) {
+    auto& p = peers[i]; p.udp->send(p.tcp->bind_request(p.session)); std::optional<Json> reply;
+    wait_for([&] { reply = p.udp->try_receive(); return reply.has_value(); },"UDP_BOUND");
+    ack_need(reply->at("type") == "UDP_BOUND" && reply->at("session_id") == p.session && reply->at("owner_epoch") == 0 &&
+      server.room().status() == (i == 3 ? "RUNNING" : "LOBBY"),"normal all-joined-ready start");
+    bindings.push_back(Json{{"client",p.role},{"player_id",p.player},{"session_matches_bound",true},{"token_value_recorded",false},
+      {"endpoint_alias",i == 0 ? "alpha_proxy" : p.role},{"status_after_bind",server.room().status()}});
+  }
+  const auto initial = ack_state(server.room());
+  const auto watermark = [&](const AckPeer& p) { return server.metrics().at("snapshot_streams").at(p.player).at("acknowledged_seq"); };
+  const auto send_ack = [&](AckPeer& p, Json fields, const std::string& cause) {
+    const auto before = ack_state(server.room()), watermark_before = watermark(p);
+    auto wire = message("SNAPSHOT_ACK"); wire.update(fields);
+    wire.update(Json{{"session_id",p.session},{"room_id",room_id},{"player_id",p.player},{"owner_epoch",0}});
+    const auto ingress = server.metrics().at("udp_received_datagrams").get<std::uint64_t>(); p.udp->send(wire);
+    wait_for([&] { return server.metrics().at("udp_received_datagrams") == ingress+1 && server.cleanup().at("mailbox_messages") == 0; },"actual ACK owner receipt");
+    const auto after = ack_state(server.room()), watermark_after = watermark(p);
+    ack_need(before == after,"ACK must not reset simulation or input sequence");
+    expect(watermark_before.is_null() || (!watermark_after.is_null() && watermark_after >= watermark_before),"server ACK watermark monotonic",
+      Json{{"cause",cause},{"before",watermark_before},{"after",watermark_after}});
+    if (cause != "APPLIED") expect(watermark_after == watermark_before,"invalid/lower/resync report does not adopt another watermark",
+      Json{{"cause",cause},{"before",watermark_before},{"after",watermark_after}});
+    else expect(watermark_after == fields.at("snapshot_seq"),"valid retained applied ACK accepted",watermark_after);
+    wire.erase("session_id"); wire["session_alias"] = p.role;
+    ack_trace.push_back(Json{{"client",p.role},{"before_tick",server.room().executed_ticks()},{"cause",cause},{"sent",wire},
+      {"watermark_before",watermark_before},{"watermark_after",watermark_after},{"client_last_applied",p.latest},
+      {"before",before},{"after",after},{"stream_after",server.metrics().at("snapshot_streams").at(p.player)}});
+  };
+  const auto apply = [&](AckPeer& p, const Json& wire, const Json& visible) {
+    const auto sequence = wire.at("snapshot_seq").get<std::uint64_t>(); if (sequence <= p.latest) return false;
+    std::map<std::string,Json> players; std::string status;
+    if (wire.at("kind") == "FULL") status = wire.at("status").get<std::string>();
+    else {
+      ack_need(wire.at("kind") == "DELTA","known snapshot kind"); const auto base = wire.at("base_snapshot_seq").get<std::uint64_t>();
+      if (!p.retained.contains(base)) return false;
+      const auto& saved = p.retained.at(base); status = saved.at("status").get<std::string>();
+      for (const auto& row : saved.at("players")) players.emplace(row.at("player_id").get<std::string>(),row);
+    }
+    for (const auto& id : wire.at("removed_player_ids")) players.erase(id.get<std::string>());
+    std::string previous;
+    for (const auto& row : wire.at("players")) {
+      const auto id = row.at("player_id").get<std::string>();
+      ack_need(id > previous && row.size() == 7 && row.contains("slot") && row.contains("x") && row.contains("y") &&
+        row.contains("direction") && row.contains("score") && row.at("connectivity") == "CONNECTED","seven visible fields sorted by player ID");
+      players[id] = row; previous = id;
+    }
+    Json rows = Json::array(); for (const auto& [id,row] : players) { (void)id; rows.push_back(row); }
+    p.applied = Json{{"room_id",room_id},{"tick",wire.at("tick")},{"owner_epoch",0},{"status",status},{"players",rows}};
+    ack_need(p.applied == visible,"actual FULL/DELTA reconstruction equals independent visible authority");
+    if (p.retained.size() == snapshot_retention) p.retained.erase(p.retained.begin());
+    p.retained[sequence] = p.applied; p.latest = sequence; return true;
+  };
+  bool loss_recovered = false;
+  const auto capture = [&](int sequence) {
+    const auto& timeline = scenario.at("observer_timeline");
+    const auto rule = std::find_if(timeline.begin(),timeline.end(),[&](const auto& item) { return item.contains("snapshot_seq") && item.at("snapshot_seq") == sequence; });
+    ack_need(rule != timeline.end(),"every publication is in frozen observer timeline");
+    const auto physical = ack_state(server.room()), visible = ack_visible(server.room());
+    const auto canonical = canonical_state(server.room()), hash = sha256(canonical);
+    wait_for([&] { return proxy.snapshots().size() == static_cast<std::size_t>(sequence); },"original snapshot observed at proxy");
+    Json alpha;
+    for (std::size_t i = 0; i < peers.size(); ++i) {
+      auto& p = peers[i]; const bool dropped = i == 0 && rule->value("fault",std::string()) == "DROP_ONCE";
+      const auto ack_before = watermark(p); const auto last_before = p.latest; Json wire;
+      if (dropped) wire = proxy.snapshots().at(static_cast<std::size_t>(sequence-1));
+      else {
+        std::optional<Json> value; wait_for([&] { value = p.udp->try_receive(); return value.has_value(); },"delivered snapshot"); wire = *value;
+      }
+      ack_need(wire.at("v") == 1 && wire.at("type") == "SNAPSHOT" && wire.at("snapshot_seq") == sequence &&
+        wire.at("tick") == sequence*2-3 && wire.at("room_id") == room_id && wire.at("owner_epoch") == 0 &&
+        wire.at("state_hash") == hash && wire.size() == (wire.at("kind") == "FULL" ? 12U : 11U),"actual wire cadence and independently checked canonical metadata");
+      const bool applied = !dropped && apply(p,wire,visible); bool sent = false;
+      if (applied) { send_ack(p,Json{{"snapshot_seq",p.latest},{"state_hash",wire.at("state_hash")}},"APPLIED"); sent = true; }
+      else if (!dropped && i == 0 && rule->value("resync_required",false)) {
+        send_ack(p,Json{{"snapshot_seq",p.latest},{"resync_required",true}},"MISSING_BASE"); sent = true;
+      }
+      ack_need(p.latest >= last_before,"client last-applied sequence never rolls back");
+      if (i == 0) {
+        expect(wire.at("kind") == rule->at("expect"),"observer snapshot kind",Json{{"seq",sequence},{"wire",wire}});
+        expect(wire.at("base_snapshot_seq") == (rule->contains("expect_base") ? rule->at("expect_base") : Json(nullptr)),
+          "observer retained base",Json{{"seq",sequence},{"base",wire.at("base_snapshot_seq")}});
+        expect(applied == rule->at("apply").get<bool>(),"observer actual application",Json{{"seq",sequence},{"applied",applied}});
+        if (sequence == 3) loss_recovered = applied && wire.at("base_snapshot_seq") == 1;
+        const auto stream = server.metrics().at("snapshot_streams").at(p.player);
+        alpha = Json{{"snapshot_seq",sequence},{"tick",wire.at("tick")},{"kind",wire.at("kind")},{"base",wire.at("base_snapshot_seq")},
+          {"wire",wire},{"dropped",dropped},{"received",!dropped},{"applied",applied},{"ack_sent",sent},
+          {"watermark_before",ack_before},{"watermark_after",watermark(p)},{"client_last_before",last_before},{"client_last_applied",p.latest},
+          {"replica",p.applied},{"authoritative_visible",visible},{"canonical_record",canonical},{"state_hash",hash},
+          {"fallback_reason",stream.value("last_full_reason",Json(nullptr))},{"stream",stream}};
+      } else ack_need(applied && p.latest == static_cast<std::uint64_t>(sequence),"unaffected clients continuously apply/ACK latest");
+    }
+    ack_need(physical == ack_state(server.room()),"publication/feedback never simulates"); observer.push_back(alpha);
+    for (auto& p : peers) ack_need(!p.tcp->try_receive() && !p.udp->try_receive(),"no extra control or out-of-cadence datagram");
+  };
+  capture(1); Json records = Json::array(), hashes = Json::array(), inputs = Json::array();
+  for (int tick = 0; tick < 14; ++tick) {
+    for (const auto& special : scenario.at("observer_timeline")) if (special.contains("before_tick") && special.at("before_tick") == tick) {
+      if (special.contains("send_ack")) {
+        Json fields{{"snapshot_seq",special.at("send_ack")},{"resync_required",special.at("resync_required")}};
+        if (special.contains("state_hash")) fields["state_hash"] = special.at("state_hash");
+        const auto cause = special.contains("state_hash") ? "HASH_MISMATCH" : special.at("send_ack") == 999 ? "UNKNOWN_ACK" : "STALE_ACK";
+        send_ack(peers[0],fields,cause);
+      } else {
+        ack_need(tick == 10 && special.contains("client_fault") && peers[0].latest == 6,"fixed client-only missing-base event");
+        const auto before = ack_state(server.room()), ack = watermark(peers[0]); const auto erased = peers[0].retained.erase(6);
+        ack_need(erased == 1 && peers[0].latest == 6 && ack_state(server.room()) == before && watermark(peers[0]) == ack,
+          "discard local base6 only, preserve server and client watermark");
+        local_faults.push_back(Json{{"before_tick",tick},{"discarded_base",6},{"client_last_applied",peers[0].latest},
+          {"watermark_before",ack},{"watermark_after",watermark(peers[0])},{"before",before},{"after",ack_state(server.room())}});
+      }
+    }
+    if (tick == 0) {
+      const auto& event = scenario.at("events").at(0); auto& p = peers[0];
+      ack_need(event.at("before_tick") == 0 && event.at("kind") == "INPUT" && event.at("client") == "alpha" && event.at("seq") == 1 &&
+        event.at("target_tick") == 0 && event.at("direction") == "EAST" && event.at("tag_target_role").is_null() && event.at("owner_epoch") == 0,"one frozen INPUT");
+      auto input = message("INPUT"); input.update(Json{{"session_id",p.session},{"room_id",room_id},{"player_id",p.player},
+        {"seq",event.at("seq")},{"target_tick",event.at("target_tick")},{"direction",event.at("direction")},{"owner_epoch",0},{"tag_target_player_id",nullptr}});
+      p.udp->send(input); std::optional<Json> response;
+      wait_for([&] { response = p.udp->try_receive(); return response.has_value(); },"real INPUT_ACK");
+      ack_need(response->at("type") == "INPUT_ACK" && response->at("seq") == 1 && response->at("code") == "ACCEPTED" && response->at("tick") == 0,"actual single input admission");
+      input.erase("session_id"); input["session_alias"] = p.role; inputs.push_back(Json{{"request",input},{"response",*response}});
+    }
+    server.advance_one_tick(); const auto& alpha = server.room().players().at("player-00");
+    ack_need(server.room().executed_ticks() == tick+1 && clock.now_ms == (tick+1)*50 && server.room().status() == "RUNNING" &&
+      alpha.x == 10000+400*(tick+1) && alpha.y == 10000 && alpha.direction == Direction::east && alpha.score == 0 &&
+      alpha.last_accepted_seq() == 1 && alpha.pending.empty() && session_ticks == 1200,"unchanged fourteen-tick physical/sequence continuity");
+    const auto canonical = canonical_state(server.room()), hash = sha256(canonical);
+    ack_need(server.replay().last_state_hash() == hash,"real replay hash matches canonical scalar state");
+    hashes.push_back(hash); records.push_back(Json{{"tick",tick},{"canonical_record",canonical},{"state_hash",hash},{"state",ack_state(server.room())}});
+    if ((tick+1)%2 == 0) capture((tick+1)/2+1);
+  }
+  expect(peers[0].latest == 8 && peers[0].applied == ack_visible(server.room()),"observer final full resynchronization",peers[0].latest);
+  ack_need(proxy.snapshots().size() == 8 && proxy.delivered() == 7 && observer.size() == 8 && local_faults.size() == 1,
+    "exactly one dropped snapshot and one client base discard");
+  const auto final = ack_state(server.room()), metrics = server.metrics(); auto fds = server.owned_descriptors(); fds.push_back(proxy.descriptor());
+  for (const auto& p : peers) { fds.push_back(p.tcp->descriptor()); fds.push_back(p.udp->descriptor()); }
+  server.shutdown(); proxy.close(); for (auto& p : peers) { p.tcp->close(); p.udp->close(); }
+  for (const auto fd : fds) ack_need(descriptor_closed(fd),"actual descriptor release");
+  auto cleanup = server.cleanup(); for (const auto& [key,value] : cleanup.items()) { (void)key; ack_need(value == 0,"active resource cleanup"); }
+  ack_need(Fd::live() == fd_before,"tracked descriptors released"); cleanup["descriptor_checks"] = fds.size();
+  cleanup["all_descriptors_closed"] = true; cleanup["tracked_descriptor_delta"] = Fd::live()-fd_before; cleanup["proxy_held_packets"] = 0;
+  Json logical = Json::array();
+  for (const auto& row : observer) {
+    Json item; for (const auto* key : {"snapshot_seq","tick","kind","base","dropped","applied","ack_sent","watermark_before","watermark_after","client_last_applied","fallback_reason"}) item[key] = row.at(key);
+    logical.push_back(item);
+  }
+  return Json{{"result",failures.empty() ? "PASS" : "FAIL"},{"thread","G10"},{"scenario_id",scenario.at("scenario_id")},
+    {"process_id",::getpid()},{"executed_ticks",14},{"initial_state",initial},{"normal_joins",joins},{"initial_bindings",bindings},
+    {"inputs",inputs},{"observer_timeline",observer},{"ack_trace",ack_trace},{"local_faults",local_faults},{"fault_trace",proxy.trace()},
+    {"logical_timeline",logical},{"tick_records",records},{"state_hashes",hashes},{"final_state",final},{"violations",failures},
+    {"loss_delta3_base1_recovery",loss_recovered},{"network_fault_runs",1},{"metrics",metrics},{"cleanup",cleanup},{"all_resources_released",true}};
+}
+}
diff --git a/tests/g10.hpp b/tests/g10.hpp
new file mode 100644
index 0000000..2047cd2
--- /dev/null
+++ b/tests/g10.hpp
@@ -0,0 +1,5 @@
+#pragma once
+#include "g09.hpp"
+namespace arena {
+Json run_ack_scenario(const Json& scenario);
+}
diff --git a/tests/scenario_main.cpp b/tests/scenario_main.cpp
index 6df326f..4e36e00 100644
--- a/tests/scenario_main.cpp
+++ b/tests/scenario_main.cpp
@@ -1,5 +1,6 @@
 #include "g07.hpp"
 #include "g09.hpp"
+#include "g10.hpp"
 #ifndef ARENA_TEST_FIXTURES
 #error Scenario fixture executable requires its separate test core
 #endif
@@ -33,11 +34,12 @@ int main(int argc, char** argv) {
       const auto scenario = arena::read_json_file(input);
       if (scenario.at("thread") != "G07" && scenario.at("thread") != "G09") {
         if (variant) throw std::invalid_argument("variant is only active for G07");
-        const auto evidence = scenario.at("thread") == "G08" ? arena::run_snapshot_scenario(scenario) : arena::run_scenario(scenario);
+        const auto evidence = scenario.at("thread") == "G08" ? arena::run_snapshot_scenario(scenario) :
+          scenario.at("thread") == "G10" ? arena::run_ack_scenario(scenario) : arena::run_scenario(scenario);
         arena::write_json_file(output,evidence);
         std::cout << arena::Json{{"result",evidence.at("result")},{"executed_ticks",evidence.at("executed_ticks")},
           {"scenario_id",evidence.at("scenario_id")},{"evidence",output.string()},{"cleanup",evidence.at("cleanup")}}.dump() << '\n';
-        return 0;
+        return evidence.at("result") == "PASS" ? 0 : 1;
       }
       if (scenario.at("thread") == "G09") {
         if (variant) throw std::invalid_argument("variant is only active for G07");
diff --git a/tests/tests.cpp b/tests/tests.cpp
index 956afdb..f7b490b 100644
--- a/tests/tests.cpp
+++ b/tests/tests.cpp
@@ -651,6 +651,13 @@ void snapshot_retention_without_ticks() {
   auto expected_ids = Json::array(); for (int seq = 2; seq <= 33; ++seq) expected_ids.push_back(seq);
   check(stream.metrics().at("retained_ids") == expected_ids && canonical_state(room) == record && room.executed_ticks() == 0,
         "33 publications evict only oldest1 without simulation");
+  const auto before_expired = stream.metrics(); stream.acknowledge(1);
+  check(before_expired.at("acknowledged_seq") == 33 && stream.metrics().at("acknowledged_seq") == 33 &&
+    stream.metrics().at("resync_pending") == true && stream.metrics().at("resync_reason") == "ACK_OUTSIDE_RETENTION" &&
+    stream.metrics().at("retained_ids") == expected_ids && stream.size() == 32 && canonical_state(room) == record && room.executed_ticks() == 0,
+    "G10 expired ACK1 forces next scheduled full without rolling back watermark33 or simulating");
+  std::cout << Json{{"G10_expired_ACK_probe",Json{{"publications",33},{"executed_ticks",0},
+    {"before",before_expired},{"after",stream.metrics()}}}}.dump() << '\n';
   const auto retained = stream.metrics(); stream.clear(); room.close();
   check(stream.size() == 0 && stream.metrics().at("acknowledged_seq").is_null(), "retained state and ACK released");
   std::cout << Json{{"G08_zero_tick_retention",Json{{"captures",captures},{"before_clear",retained},
