# Slow Consumer와 Bounded Outbound Work (G12)

## `feat(cpp): bound and coalesce outbound snapshots`

diff --git a/CMakeLists.txt b/CMakeLists.txt
index 5e59071..5a056ed 100644
--- a/CMakeLists.txt
+++ b/CMakeLists.txt
@@ -31,10 +31,10 @@ endforeach()
 add_executable(arena src/main.cpp)
 target_link_libraries(arena PRIVATE arena_core)
 target_compile_options(arena PRIVATE -Wall -Wextra -Wpedantic -Werror)
-add_executable(arena_tests tests/tests.cpp tests/g09.cpp)
+add_executable(arena_tests tests/tests.cpp tests/g09.cpp tests/g12_queue.cpp)
 target_link_libraries(arena_tests PRIVATE arena_test_core)
 target_compile_options(arena_tests PRIVATE -Wall -Wextra -Wpedantic -Werror)
-add_executable(arena_scenarios tests/scenario_main.cpp tests/g07.cpp tests/g09.cpp tests/g10.cpp tests/g11.cpp)
+add_executable(arena_scenarios tests/scenario_main.cpp tests/g07.cpp tests/g09.cpp tests/g10.cpp tests/g11.cpp tests/g12.cpp tests/g12_queue.cpp)
 target_link_libraries(arena_scenarios PRIVATE arena_test_core)
 target_compile_options(arena_scenarios PRIVATE -Wall -Wextra -Wpedantic -Werror)
 enable_testing()
diff --git a/evidence/G12-runs.jsonl b/evidence/G12-runs.jsonl
new file mode 100644
index 0000000..7da1c03
--- /dev/null
+++ b/evidence/G12-runs.jsonl
@@ -0,0 +1,7 @@
+{"label":"baseline-compile","category":"compile","units":1,"ticks":0,"ceiling_seconds":180,"argv":["clang++","-std=c++20","-O2","-Wall","-Wextra","-Wpedantic","-Werror","-fsanitize=thread","-g","-DARENA_TEST_FIXTURES=1","-I","src","-I","tests","-I","/opt/homebrew/include","artifacts/g12/reproduce.cpp","src/game.cpp","src/transport.cpp","src/replay.cpp","src/snapshot.cpp","-o","artifacts/g12/reproduce"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g12/baseline-compile.log","started_at":"2026-08-28T07:16:41.147216+00:00","duration_seconds":24.580405,"exit":0,"wrapper_pid":2568,"child_pid":2579,"timed_out":false}
+{"label":"baseline","category":"unit","units":1,"ticks":100,"ceiling_seconds":120,"argv":["env","TSAN_OPTIONS=halt_on_error=1","./artifacts/g12/reproduce","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G12.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g12/baseline.json"],"expected_exit":1,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g12/baseline.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g12/baseline.json","started_at":"2026-08-28T07:18:31.487856+00:00","duration_seconds":1.742619,"exit":1,"wrapper_pid":4825,"child_pid":4834,"timed_out":false,"observed_ticks":100,"runtime_pid":4834}
+{"label":"build","category":"compile","units":2,"ticks":0,"ceiling_seconds":180,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g12-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g12/track","TSAN_OPTIONS=halt_on_error=1","ARENA_TSAN=ON","./track","build"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g12/build.log","started_at":"2026-08-28T07:30:21.519979+00:00","duration_seconds":48.00355,"exit":0,"wrapper_pid":16253,"child_pid":16262,"timed_out":false}
+{"label":"unit","category":"unit","units":1,"ticks":0,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g12-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g12/track","TSAN_OPTIONS=halt_on_error=1","./track","unit-test"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g12/unit.log","started_at":"2026-08-28T07:31:35.304142+00:00","duration_seconds":5.246566,"exit":0,"wrapper_pid":17479,"child_pid":17494,"timed_out":false}
+{"label":"integration","category":"integration","units":1,"ticks":0,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g12-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g12/track","TSAN_OPTIONS=halt_on_error=1","./track","integration-test"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g12/integration.log","started_at":"2026-08-28T07:31:40.644158+00:00","duration_seconds":1.48375,"exit":0,"wrapper_pid":17679,"child_pid":17694,"timed_out":false}
+{"label":"canonical","category":"canonical","units":1,"ticks":100,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g12-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g12/track","TSAN_OPTIONS=halt_on_error=1","./track","scenario-run","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G12.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g12/canonical.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g12/canonical.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g12/canonical.json","started_at":"2026-08-28T07:31:42.220937+00:00","duration_seconds":2.135193,"exit":0,"wrapper_pid":17715,"child_pid":17716,"timed_out":false,"observed_ticks":100,"runtime_pid":17722}
+{"label":"reference","category":"reference","units":1,"ticks":100,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g12-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g12/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g12/canonical.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g12/reference.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g12/reference.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g12/reference.json","started_at":"2026-08-28T07:31:44.409684+00:00","duration_seconds":0.432742,"exit":0,"wrapper_pid":17743,"child_pid":17744,"timed_out":false,"observed_ticks":100,"runtime_pid":17750}
diff --git a/evidence/G12.md b/evidence/G12.md
new file mode 100644
index 0000000..2d27d61
--- /dev/null
+++ b/evidence/G12.md
@@ -0,0 +1,31 @@
+# G12 — bounded outbound snapshots
+
+START `16756acc88aa655335c22b96a694ed0ea01563d4`; profile `realtime-core`; spec `c1d62196ab76b55652f5d75a67514f8c6d8210ce`.
+Fixture SHA256 `b5350ef5bfb9fa93cbde1fe0fd30079e6115fe26988514be59d63d9dee6bc6f3`.
+
+## Reproduction and change
+
+`artifacts/g12/commands.json` resolved every command before execution. `source-manifest.json` and `baseline-preservation.json` retain17 actual START file comparisons. The unchanged implementation ran100 ticks with one bravo INPUT and a selective test-only `EAGAIN` send/sendto boundary:50 actual `UDP_SEND_DROP`, no retained/coalesced UDP snapshots, healthy51 snapshots each,15 descriptors closed. The absent UDP pre-dequeue queue is recorded explicitly; the65th-attempt control boundary was source-observed only. Baseline exit1 was expected.
+
+The production path now owns one FULL and one DELTA buffer, releases superseded snapshots, and retries nonblocking sends without an additional transport queue. The64th control attempt substitutes a bounded terminal report, tries one nonblocking flush, then closes; unsent controls are never counted delivered. Gameplay, snapshot retention, G11 grace and other UDP reply semantics are unchanged. The readiness callback and fixed fixture remain test-build only.
+
+## Actual checks
+
+TSan build, unit26, integration4, live100 and separate accepted-journal replay100 passed. Budget: compile3/8 (baseline1 + configure/build2), unit2/4 (baseline + full suite including exactly two new zero-tick probes), integration1/2, live1/1, reference1/1; fault/load/profiler0. No post failure or rerun. Exact argv, processes, durations and logs: `evidence/G12-runs.jsonl`; nested build/process logs: `artifacts/g12/track/`.
+
+Alpha generated51 snapshots including initial FULL1, sent only that initial snapshot, coalesced49, and ended with one pending FULL (seq51,713 bytes) and no DELTA. Hold occupancy never exceeded FULL1/DELTA1; alpha retained-byte high-water1072. Global owned buffers peaked at3/1785 bytes, including temporary encoding ownership; the synchronous INPUT_ACK transfer peaked at1/123 bytes.502 readiness callbacks included351 not-ready returns. Healthy bravo/charlie/delta each applied and ACKed51. All100 canonical records and hashes equal both the unchanged baseline and the separate offline process; final hash `5d7f602c8e4eb781e28b0875e44cdda38df451fb27434a04b8963b6d6bc9f958`.
+
+The control probe created/released64 buffers, sent0, and closed at the64th attempt within capacity64. The mixed probe preserved both TCP controls and discarded exactly3 snapshot buffers; all6 allocated buffers were released. Alpha TCP close released its remaining outbound buffer before owner disconnect; deadline300 and DISCONNECTED/STOP were retained. Shutdown released218/218 live-run buffers and all15 descriptors; all active cleanup counters0. Pure probes closed5/9 descriptors. Shipping symbols contain no G12 fixture/driver/probe, and raw results contain no credential fields.
+
+## Raw artifacts
+
+All paths below are under `artifacts/g12/`. `reproduce.cpp` preserves the exact baseline wrapper; `unit.log` contains both complete pure-probe records. Live readiness, queue ownership, healthy wire/ACK/application and cleanup evidence are in `canonical.json`; per-tick state/canonical bytes are in the records files.
+
+| File | Bytes | SHA256 |
+| --- | ---: | --- |
+| baseline.json | 499192 | `8a2b1c0c90bbca2ba5785f03ec9b78cc6a9028f75440a62b0f66bfbbfc39158b` |
+| canonical.json | 992428 | `87972c31c7e07e534f1e76c92b5d184e606f1d219070d4d6ed189b9a7a0e975c` |
+| canonical.records.json | 224114 | `21f58c1165e7c880afe6daf7879349dc96fa5fb86f49e17581d500581e220136` |
+| canonical.replay.json | 11052 | `5c009287d23e881629cfcd757eb31a308ff8a7a6428d6e2d08b3a55df9fffcff` |
+| reference.json | 9277 | `5978b33164a7ab4cfc703323a1b9c6bf104afc911c08577f9e41cb80391c6c91` |
+| reference.records.json | 208514 | `c4dfe04f4e01c40869997f0ce30a6058b6a27062572017c4516a0baeba21ec03` |
diff --git a/src/transport.cpp b/src/transport.cpp
index 89e57a1..23a2f70 100644
--- a/src/transport.cpp
+++ b/src/transport.cpp
@@ -215,6 +215,27 @@ ParseResult FrameParser::finish(bool io_error) {
     io_error ? "TRANSPORT_IO_ERROR" : partial ? "TRANSPORT_EOF_IN_FRAME" : "TRANSPORT_EOF", partial};
   return *terminal_;
 }
+PendingWrite::PendingWrite(std::vector<std::uint8_t> value, std::size_t start, OutboundMemory* memory)
+    : bytes(std::move(value)), offset(start), memory_(memory) {
+  if (memory_) {
+    buffer_id = ++memory_->created; ++memory_->buffers; memory_->bytes += bytes.capacity();
+    memory_->high_water_buffers = std::max(memory_->high_water_buffers,memory_->buffers);
+    memory_->high_water_bytes = std::max(memory_->high_water_bytes,memory_->bytes);
+  }
+}
+PendingWrite::~PendingWrite() { release(); }
+PendingWrite::PendingWrite(PendingWrite&& other) noexcept
+    : bytes(std::move(other.bytes)), offset(other.offset), buffer_id(other.buffer_id), memory_(std::exchange(other.memory_,nullptr)) {}
+PendingWrite& PendingWrite::operator=(PendingWrite&& other) noexcept {
+  if (this != &other) {
+    release(); bytes = std::move(other.bytes); offset = other.offset; buffer_id = other.buffer_id;
+    memory_ = std::exchange(other.memory_,nullptr);
+  }
+  return *this;
+}
+void PendingWrite::release() {
+  if (memory_) { --memory_->buffers; memory_->bytes -= bytes.capacity(); ++memory_->released; memory_ = nullptr; }
+}
 void PendingWrite::consume(std::size_t count) {
   if (count > remaining().size()) throw std::logic_error("write offset exceeds owned buffer");
   offset += count;
@@ -248,6 +269,7 @@ Server::Server(ManualClock& clock, std::uint16_t port, MonotonicNow monotonic_no
   udp_port_ = ntohs(address.sin_port);
   register_event(listener_.get(), EVFILT_READ, EV_ADD);
   register_event(datagram_.get(),EVFILT_READ,EV_ADD);
+  register_event(datagram_.get(),EVFILT_WRITE,EV_ADD | EV_DISABLE);
 }
 Server::~Server() {
   // Destructors cannot report errors; explicit shutdown is required by callers
@@ -303,6 +325,7 @@ void Server::disconnect(int fd, const std::string& reason) {
   // Closing a descriptor removes its kqueue registrations. The generation id
   // in udata prevents old events from touching a subsequently reused fd.
   connections_.erase(found);
+  update_udp_write_interest();
 }
 void Server::end_transport(int fd, bool io_error) {
   const auto found = connections_.find(fd);
@@ -368,6 +391,7 @@ void Server::write_ready(int fd) {
   auto& writes = found->second.outbound;
   // Bounded per-ready work, preserving every unsent suffix in owned storage.
   for (int work = 0; work < 64 && !writes.empty(); ++work) {
+    if (!outbound_ready(found->second,false)) return;
     auto& write = writes.front();
     const auto bytes = write.remaining();
     const auto sent = ::send(fd, bytes.data(), bytes.size(), 0);
@@ -389,7 +413,16 @@ void Server::poll_io(int timeout_ms) {
     const auto& event = events[static_cast<std::size_t>(i)];
     const int fd = static_cast<int>(event.ident);
     if (fd == listener_.get()) { if (!stopping_) accept_ready(); continue; }
-    if (fd == datagram_.get()) { if (!stopping_) read_datagrams(); continue; }
+    if (fd == datagram_.get()) {
+      if (!stopping_) {
+        if (event.filter == EVFILT_READ) read_datagrams();
+        else {
+          for (auto& [peer_fd, conn] : connections_) { (void)peer_fd; write_datagrams(conn); }
+          update_udp_write_interest();
+        }
+      }
+      continue;
+    }
     const auto found = connections_.find(fd);
     if (found == connections_.end() || found->second.id != reinterpret_cast<uintptr_t>(event.udata)) continue;
     if (event.flags & EV_ERROR) { end_transport(fd, true); continue; }
@@ -403,13 +436,16 @@ void Server::poll_io(int timeout_ms) {
 void Server::queue(std::uint64_t connection_id, Json value) {
   auto* conn = connection(connection_id);
   if (conn == nullptr) return;
-  if (conn->outbound.size() == max_control_messages) {
-    disconnect(conn->fd.get(), "CONTROL_BACKPRESSURE");
-    return;
-  }
-  conn->outbound.push_back(PendingWrite{encode_frame(value), 0});
+  // The64th attempt is the terminal report, never a65th allocation. As with
+  // framing termination, attempt one nonblocking flush and then close. Unsent
+  // cancelled controls are released, never counted as successfully delivered.
+  const bool terminal = conn->outbound.size() == max_control_messages-1;
+  if (terminal) value = error_message("CONTROL_BACKPRESSURE","bounded control queue reached capacity");
+  conn->outbound.emplace_back(encode_frame(value),0,&outbound_memory_);
   outbound_high_water_ = std::max(outbound_high_water_, conn->outbound.size());
+  observe_outbound(*conn);
   register_event(conn->fd.get(), EVFILT_WRITE, EV_ENABLE, conn->id);
+  if (terminal) { const int fd = conn->fd.get(); write_ready(fd); disconnect(fd,"CONTROL_BACKPRESSURE"); }
 }
 void Server::broadcast(const Json& value) {
   std::vector<std::uint64_t> ids;
@@ -452,17 +488,79 @@ void Server::read_datagrams() {
   }
 }
 void Server::send_realtime(std::uint64_t connection_id, const Json& value) {
-  const auto* conn = connection(connection_id);
+  auto* conn = connection(connection_id);
   if (!conn || !conn->udp_endpoint) { ++errors_["UDP_UNBOUND_SEND"]; return; }
   try {
-    const auto bytes = encode_datagram(value);
-    outbound_datagram_high_water_ = std::max(outbound_datagram_high_water_,bytes.size());
-    const auto count = ::sendto(datagram_.get(),bytes.data(),bytes.size(),0,
+    PendingWrite buffer(encode_datagram(value),0,&outbound_memory_);
+    outbound_datagram_high_water_ = std::max(outbound_datagram_high_water_,buffer.bytes.size());
+    if (value.at("type") == "SNAPSHOT") {
+      ++conn->snapshots_generated;
+      const auto supersede = [&](auto& pending) { if (pending) { pending.reset(); ++conn->snapshots_coalesced; } };
+      if (value.at("kind") == "FULL") {
+        supersede(conn->pending_full); supersede(conn->pending_delta); conn->pending_full.emplace(std::move(buffer));
+      } else {
+        supersede(conn->pending_delta); conn->pending_delta.emplace(std::move(buffer));
+      }
+      observe_outbound(*conn); write_datagrams(*conn); update_udp_write_interest(); return;
+    }
+    // Other UDP responses keep their existing explicit best-effort semantics;
+    // only snapshots have replication replacement rules and retained slots.
+    if (!outbound_ready(*conn,true)) { ++errors_["UDP_SEND_DROP"]; return; }
+    const auto count = ::sendto(datagram_.get(),buffer.bytes.data(),buffer.bytes.size(),0,
       reinterpret_cast<const sockaddr*>(&*conn->udp_endpoint),sizeof(sockaddr_in));
-    if (count == static_cast<ssize_t>(bytes.size())) ++sent_datagrams_;
+    if (count == static_cast<ssize_t>(buffer.bytes.size())) ++sent_datagrams_;
     else ++errors_["UDP_SEND_DROP"];
   } catch (const std::length_error&) { ++errors_["UDP_OUTBOUND_SIZE_INVALID"]; }
 }
+bool Server::outbound_ready(const Connection& conn, bool datagram) const {
+#ifdef ARENA_TEST_FIXTURES
+  if (fixture_outbound_ready_) return fixture_outbound_ready_(conn.id,datagram);
+#endif
+  (void)conn; (void)datagram; return true;
+}
+void Server::write_datagrams(Connection& conn) {
+  if (!conn.udp_endpoint) return;
+  // A socket send borrows the slot's vector. EAGAIN retains that same owned
+  // buffer: no hidden transport queue or copied in-flight snapshot exists.
+  for (auto* pending : {&conn.pending_full,&conn.pending_delta}) {
+    if (!*pending) continue;
+    if (!outbound_ready(conn,true)) return;
+    const auto& bytes = (*pending)->bytes;
+    const auto sent = ::sendto(datagram_.get(),bytes.data(),bytes.size(),0,
+      reinterpret_cast<const sockaddr*>(&*conn.udp_endpoint),sizeof(sockaddr_in));
+    if (sent < 0 && transient_io()) return;
+    if (sent == static_cast<ssize_t>(bytes.size())) { ++sent_datagrams_; ++conn.snapshots_sent; }
+    else ++errors_["UDP_SEND_DROP"];
+    pending->reset();
+  }
+}
+void Server::update_udp_write_interest() {
+  if (datagram_.get() < 0) { udp_write_enabled_ = false; return; }
+  const bool pending = std::any_of(connections_.begin(),connections_.end(),[](const auto& item) {
+    return item.second.pending_full || item.second.pending_delta;
+  });
+  if (pending == udp_write_enabled_) return;
+  udp_write_enabled_ = pending;
+  register_event(datagram_.get(),EVFILT_WRITE,pending ? EV_ENABLE : EV_DISABLE);
+}
+Json Server::outbound_state(const Connection& conn) const {
+  std::size_t bytes = 0; for (const auto& control : conn.outbound) bytes += control.bytes.capacity();
+  if (conn.pending_full) bytes += conn.pending_full->bytes.capacity();
+  if (conn.pending_delta) bytes += conn.pending_delta->bytes.capacity();
+  return Json{{"control",conn.outbound.size()},{"full",conn.pending_full ? 1 : 0},{"delta",conn.pending_delta ? 1 : 0},
+    {"retained_bytes",bytes},{"control_high_water",conn.control_high_water},{"full_high_water",conn.full_high_water},
+    {"delta_high_water",conn.delta_high_water},{"bytes_high_water",conn.outbound_bytes_high_water},
+    {"snapshots_generated",conn.snapshots_generated},{"snapshots_sent",conn.snapshots_sent},{"snapshots_coalesced",conn.snapshots_coalesced}};
+}
+void Server::observe_outbound(Connection& conn) {
+  conn.control_high_water = std::max(conn.control_high_water,conn.outbound.size());
+  conn.full_high_water = std::max(conn.full_high_water,static_cast<std::size_t>(conn.pending_full.has_value()));
+  conn.delta_high_water = std::max(conn.delta_high_water,static_cast<std::size_t>(conn.pending_delta.has_value()));
+  std::size_t bytes = 0; for (const auto& control : conn.outbound) bytes += control.bytes.capacity();
+  if (conn.pending_full) bytes += conn.pending_full->bytes.capacity();
+  if (conn.pending_delta) bytes += conn.pending_delta->bytes.capacity();
+  conn.outbound_bytes_high_water = std::max(conn.outbound_bytes_high_water,bytes);
+}
 void Server::bind_datagram(Connection& conn, const Envelope& envelope) {
   const auto& value = envelope.value;
   const bool valid = envelope.udp_endpoint && !conn.udp_endpoint && !conn.bind_token.empty() &&
@@ -652,7 +750,9 @@ void Server::leave_room(std::uint64_t connection_id, const std::string& kind) {
   if (kind == "DISCONNECT") room_.disconnect(connection_id);
   else { room_.leave(connection_id); resume_.erase(player_id); }
   if (!player_id.empty() && previous_status == "RUNNING") replay_.left(room_,player_id,kind);
-  if (auto* conn = connection(connection_id)) conn->snapshots.clear();
+  if (auto* conn = connection(connection_id)) {
+    conn->snapshots.clear(); conn->pending_full.reset(); conn->pending_delta.reset(); update_udp_write_interest();
+  }
   if (previous_status == "LOBBY" && room_.status() == "RUNNING") start_room();
 }
 void Server::drain_mailbox() {
@@ -709,11 +809,14 @@ void Server::advance_one_tick() {
   }
 }
 Json Server::metrics() const {
-  auto streams = Json::object();
-  std::size_t bound = 0, tokens = 0, sessions = 0;
+  auto streams = Json::object(), outbound = Json::object();
+  std::size_t bound = 0, tokens = 0, sessions = 0, queued_buffers = 0, queued_bytes = 0;
   for (const auto& [fd, conn] : connections_) {
     (void)fd;
-    if (!conn.player_id.empty()) streams[conn.player_id] = conn.snapshots.metrics();
+    const auto queued = outbound_state(conn);
+    if (!conn.player_id.empty()) { streams[conn.player_id] = conn.snapshots.metrics(); outbound[conn.player_id] = queued; }
+    queued_buffers += conn.outbound.size()+conn.pending_full.has_value()+conn.pending_delta.has_value();
+    queued_bytes += queued.at("retained_bytes").get<std::size_t>();
     bound += conn.udp_endpoint.has_value(); tokens += !conn.bind_token.empty();
     sessions += !conn.session_id.empty();
   }
@@ -721,6 +824,10 @@ Json Server::metrics() const {
     {"mailbox_high_water", mailbox_high_water_}, {"outbound_control_high_water", outbound_high_water_},
     {"connection_high_water", connection_high_water_}, {"input_per_player_high_water", room_.input_high_water()},
     {"snapshot_retention_high_water",snapshot_retention_high_water_},{"snapshot_streams",streams},
+    {"outbound_streams",outbound},{"outbound_buffers",outbound_memory_.buffers},{"outbound_retained_bytes",outbound_memory_.bytes},
+    {"outbound_buffer_high_water",outbound_memory_.high_water_buffers},{"outbound_bytes_high_water",outbound_memory_.high_water_bytes},
+    {"outbound_buffers_created",outbound_memory_.created},{"outbound_buffers_released",outbound_memory_.released},
+    {"transport_owned_pending_buffers",outbound_memory_.buffers-queued_buffers},{"transport_owned_pending_bytes",outbound_memory_.bytes-queued_bytes},
     {"udp_received_datagrams",received_datagrams_},{"udp_sent_datagrams",sent_datagrams_},
     {"udp_payload_high_water",datagram_high_water_},{"udp_outbound_high_water",outbound_datagram_high_water_},
     {"udp_receive_buffer_bytes",max_datagram_bytes+1},{"udp_bound_endpoints",bound},{"udp_bind_tokens",tokens},
@@ -742,7 +849,7 @@ Json Server::metrics() const {
 Json Server::cleanup() const {
   std::size_t queued = 0, parser_buffered = 0, input_attempts = 0, retained_snapshots = 0, endpoints = 0, tokens = 0, sessions = 0, grace = 0;
   for (const auto& [fd, conn] : connections_) {
-    (void)fd; queued += conn.outbound.size(); parser_buffered += conn.parser.buffered_bytes();
+    (void)fd; queued += conn.outbound.size()+conn.pending_full.has_value()+conn.pending_delta.has_value(); parser_buffered += conn.parser.buffered_bytes();
     retained_snapshots += conn.snapshots.size();
     endpoints += conn.udp_endpoint.has_value(); tokens += !conn.bind_token.empty();
     sessions += !conn.session_id.empty();
@@ -750,6 +857,7 @@ Json Server::cleanup() const {
   for (const auto& [id, player] : room_.players()) { (void)id; input_attempts += player.input_attempts; grace += player.disconnect_deadline.has_value(); }
   return Json{{"server_connections", connections_.size()}, {"server_descriptors", owned_descriptors().size()},
     {"mailbox_messages", mailbox_.size()}, {"pending_inputs", room_.pending_count()}, {"outbound_messages", queued},
+    {"outbound_buffers",outbound_memory_.buffers},{"outbound_retained_bytes",outbound_memory_.bytes},
     {"input_attempts", input_attempts},
     {"retained_snapshots",retained_snapshots},
     {"active_sessions",sessions},{"resume_records",resume_.size()},{"grace_deadlines",grace},
diff --git a/src/transport.hpp b/src/transport.hpp
index 7ce1200..2426203 100644
--- a/src/transport.hpp
+++ b/src/transport.hpp
@@ -60,11 +60,25 @@ class FrameParser {
   std::size_t expected_ = 4;
   std::optional<ParseResult> terminal_;
 };
+struct OutboundMemory {
+  std::size_t buffers = 0, bytes = 0, high_water_buffers = 0, high_water_bytes = 0;
+  std::uint64_t created = 0, released = 0;
+};
 struct PendingWrite {
+  explicit PendingWrite(std::vector<std::uint8_t> value, std::size_t offset = 0, OutboundMemory* memory = nullptr);
+  ~PendingWrite();
+  PendingWrite(PendingWrite&& other) noexcept;
+  PendingWrite& operator=(PendingWrite&& other) noexcept;
+  PendingWrite(const PendingWrite&) = delete;
+  PendingWrite& operator=(const PendingWrite&) = delete;
   std::vector<std::uint8_t> bytes;
   std::size_t offset = 0;
+  std::uint64_t buffer_id = 0;
   std::span<const std::uint8_t> remaining() const { return std::span(bytes).subspan(offset); }
   void consume(std::size_t count);
+ private:
+  void release();
+  OutboundMemory* memory_ = nullptr;
 };
 
 class Server {
@@ -101,6 +115,9 @@ class Server {
     std::int64_t token_issued_ms = 0;
     std::optional<sockaddr_in> udp_endpoint = {};
     bool full_after_bind = false;
+    std::optional<PendingWrite> pending_full = {}, pending_delta = {};
+    std::uint64_t snapshots_generated = 0, snapshots_sent = 0, snapshots_coalesced = 0;
+    std::size_t control_high_water = 0, full_high_water = 0, delta_high_water = 0, outbound_bytes_high_water = 0;
   };
   // At most one record per bounded Room player, including expired players
   // until Room teardown so their current credential gets EXPIRED, not reset.
@@ -124,8 +141,10 @@ class Server {
 #ifdef ARENA_TEST_FIXTURES
   friend struct ReplayFixture;
   friend struct UdpFixture;
+  friend struct OutboundFixture;
   std::optional<std::string> fixture_room_id_;
   std::vector<std::string> fixture_player_ids_;
+  std::function<bool(std::uint64_t,bool)> fixture_outbound_ready_;
 #endif
   Connection* connection(std::uint64_t id);
   void register_event(int fd, short filter, unsigned short flags, std::uint64_t connection_id = 0);
@@ -133,6 +152,11 @@ class Server {
   void read_ready(int fd);
   void read_datagrams();
   void send_realtime(std::uint64_t connection_id, const Json& value);
+  void write_datagrams(Connection& conn);
+  void update_udp_write_interest();
+  bool outbound_ready(const Connection& conn, bool datagram) const;
+  Json outbound_state(const Connection& conn) const;
+  void observe_outbound(Connection& conn);
   void bind_datagram(Connection& conn, const Envelope& envelope);
   void write_ready(int fd);
   void disconnect(int fd, const std::string& reason);
@@ -159,6 +183,7 @@ class Server {
   Fd datagram_;
   std::uint16_t port_ = 0;
   std::uint16_t udp_port_ = 0;
+  OutboundMemory outbound_memory_;
   std::map<int, Connection> connections_;
   Mailbox mailbox_;
   std::set<std::uint64_t> disconnected_;
@@ -186,6 +211,7 @@ class Server {
   std::uint64_t sent_datagrams_ = 0;
   std::size_t datagram_high_water_ = 0;
   std::size_t outbound_datagram_high_water_ = 0;
+  bool udp_write_enabled_ = false;
   std::map<std::string, std::uint64_t> errors_;
   bool stopping_ = false;
 };
diff --git a/tests/g12.cpp b/tests/g12.cpp
new file mode 100644
index 0000000..65e2500
--- /dev/null
+++ b/tests/g12.cpp
@@ -0,0 +1,200 @@
+#include "g12.hpp"
+#include <algorithm>
+#include <array>
+#include <chrono>
+#include <memory>
+#include <unistd.h>
+
+namespace arena {
+namespace {
+void slow_need(bool value, const std::string& text) { if (!value) throw std::runtime_error("G12: "+text); }
+Json slow_state(const Room& room) {
+  auto state = room.view(); state["owner_epoch"] = 0;
+  for (auto& row : state["players"]) {
+    const auto& p = room.players().at(row.at("player_id").get<std::string>());
+    row["last_seq"] = p.last_accepted_seq(); row["pending"] = p.pending.size();
+    row["applied_seq"] = p.applied_seq ? Json(*p.applied_seq) : Json(nullptr);
+    row["disconnect_deadline"] = p.disconnect_deadline ? Json(*p.disconnect_deadline) : Json(nullptr);
+  }
+  return state;
+}
+Json slow_visible(const Room& room) {
+  Json rows = Json::array();
+  for (const auto& [id,p] : room.players()) if (p.connected || p.disconnect_deadline)
+    rows.push_back(Json{{"player_id",id},{"slot",p.slot},{"x",p.x},{"y",p.y},{"direction",direction_name(p.direction)},
+      {"score",p.score},{"connectivity",p.connectivity()}});
+  return Json{{"room_id",room.id()},{"tick",room.executed_ticks()-1},{"owner_epoch",0},{"status",room.status()},{"players",rows}};
+}
+struct SlowPeer {
+  std::unique_ptr<TcpClient> tcp;
+  std::unique_ptr<UdpClient> udp;
+  std::string role, player, session;
+  std::uint64_t connection = 0, latest = 0;
+  Json applied, sequences = Json::array(), publications = Json::array();
+};
+}
+ReplayRun run_backpressure_scenario(const Json& scenario) {
+  slow_need(scenario.at("thread") == "G12" && scenario.at("ticks") == 100 && scenario.at("players").size() == 4 &&
+    scenario.at("events").size() == 1 && scenario.at("seed") == 7050 && scenario.at("clock").at("tick_duration_ms") == 50,
+    "frozen100-tick four-player fixture");
+  const auto& event = scenario.at("events").at(0);
+  slow_need(event.at("before_tick") == 0 && event.at("client") == "bravo" && event.at("seq") == 1 && event.at("target_tick") == 0 &&
+    event.at("direction") == "EAST" && event.at("tag_target_role").is_null() && event.at("owner_epoch") == 0,"one frozen bravo UDP input");
+  const int descriptors_before = Fd::live(); ManualClock clock; Server server(clock,0,[&] { return clock.now_ms; });
+  inject_udp_fixture_ids(server,scenario); std::array<SlowPeer,4> peers; Json joins = Json::array();
+  for (std::size_t i = 0; i < peers.size(); ++i) {
+    auto& p = peers[i]; p.role = scenario.at("players").at(i).at("client").get<std::string>();
+    p.tcp = std::make_unique<TcpClient>(server.port()); p.tcp->send(message("HELLO"));
+    const auto welcome = p.tcp->receive_type(server,"WELCOME"); p.session = welcome.at("session_id").get<std::string>();
+    slow_need(p.tcp->has_bind_token() && !welcome.contains("udp_bind_token"),"private bind credential");
+    p.connection = OutboundFixture::connection_id(server,p.session); p.udp = std::make_unique<UdpClient>(server.udp_port());
+  }
+  auto create = message("CREATE_ROOM"); create["session_id"] = peers[0].session; peers[0].tcp->send(create);
+  const auto room_id = peers[0].tcp->receive_type(server,"ROOM_CREATED").at("room_id").get<std::string>();
+  slow_need(room_id == scenario.at("room_id").get<std::string>(),"normal fixed Room allocator");
+  for (std::size_t i = 0; i < peers.size(); ++i) {
+    auto& p = peers[i]; auto join = message("JOIN_ROOM"); join.update(Json{{"session_id",p.session},{"room_id",room_id}}); p.tcp->send(join);
+    const auto reply = p.tcp->receive_type(server,"ROOM_JOINED"); p.player = reply.at("player_id").get<std::string>();
+    const auto& fixed = scenario.at("players").at(i); const auto& model = server.room().players().at(p.player);
+    slow_need(reply.at("status") == "LOBBY" && server.room().status() == "LOBBY" && reply.at("slot") == i &&
+      p.player == fixed.at("player_id").get<std::string>() && model.x == fixed.at("spawn").at(0) && model.y == fixed.at("spawn").at(1) &&
+      p.tcp->has_resume_token(),"ordinary four unbound joins with contract spawn and existing resume credentials");
+    joins.push_back(Json{{"client",p.role},{"player_id",p.player},{"slot",i},{"status","LOBBY"},{"resume_token_present",true}});
+  }
+  for (std::size_t i = 0; i < peers.size(); ++i) {
+    peers[i].udp->bind(*peers[i].tcp,server,peers[i].session);
+    slow_need(server.room().status() == (i+1 == peers.size() ? "RUNNING" : "LOBBY"),"normal all-joined UDP ready barrier");
+  }
+  const auto wait_for = [&](const auto& ready, bool owner = true) {
+    const auto deadline = std::chrono::steady_clock::now()+std::chrono::seconds(5);
+    while (!ready() && std::chrono::steady_clock::now() < deadline) { if (owner) server.pump(); else server.poll_io(); }
+    slow_need(ready(),"bounded socket/owner barrier");
+  };
+  const auto consume = [&](SlowPeer& p) {
+    const auto wire = p.udp->receive_type(server,"SNAPSHOT"); const auto sequence = wire.at("snapshot_seq").get<std::uint64_t>();
+    const auto canonical = canonical_state(server.room()), hash = sha256(canonical); const auto authority = slow_visible(server.room());
+    slow_need(sequence == p.latest+1 && wire.at("tick") == server.room().executed_ticks()-1 && wire.at("state_hash") == hash &&
+      wire.at("room_id") == room_id && wire.at("owner_epoch") == 0 && encode_datagram(wire).size() <= max_datagram_bytes,
+      "healthy sequential snapshot with actual canonical capture metadata");
+    std::map<std::string,Json> players; std::string status;
+    if (wire.at("kind") == "FULL") {
+      slow_need(wire.at("base_snapshot_seq").is_null() && wire.size() == 12,"full contract wire shape"); status = wire.at("status").get<std::string>();
+    } else {
+      slow_need(wire.at("kind") == "DELTA" && wire.at("base_snapshot_seq") == p.latest && wire.size() == 11,"delta from actual applied ACK");
+      status = p.applied.at("status").get<std::string>();
+      for (const auto& row : p.applied.at("players")) players.emplace(row.at("player_id").get<std::string>(),row);
+    }
+    for (const auto& id : wire.at("removed_player_ids")) players.erase(id.get<std::string>());
+    std::string previous;
+    for (const auto& row : wire.at("players")) {
+      const auto id = row.at("player_id").get<std::string>();
+      slow_need(row.size() == 7 && id > previous && row.at("connectivity") == "CONNECTED","sorted seven-field visible projection");
+      previous = id; players[id] = row;
+    }
+    Json rows = Json::array(); for (const auto& [id,row] : players) { (void)id; rows.push_back(row); }
+    p.applied = Json{{"room_id",room_id},{"tick",wire.at("tick")},{"owner_epoch",0},{"status",status},{"players",rows}};
+    slow_need(p.applied == authority,"healthy full/delta replica at its capture tick"); p.latest = sequence; p.sequences.push_back(sequence);
+    auto ack = message("SNAPSHOT_ACK"); ack.update(Json{{"session_id",p.session},{"room_id",room_id},{"player_id",p.player},
+      {"owner_epoch",0},{"snapshot_seq",sequence},{"state_hash",hash}});
+    const auto before = server.metrics().at("udp_received_datagrams").get<std::uint64_t>(); p.udp->send(ack);
+    wait_for([&] { return server.metrics().at("udp_received_datagrams") == before+1 && server.cleanup().at("mailbox_messages") == 0; });
+    slow_need(server.metrics().at("snapshot_streams").at(p.player).at("acknowledged_seq") == sequence,"actual apply/ACK owner commit");
+    ack.erase("session_id"); ack["session_alias"] = p.role;
+    p.publications.push_back(Json{{"wire",wire},{"replica_visible",p.applied},{"projection_equal",true},{"canonical_hash_equal",true},
+      {"ack",ack},{"ack_owned",true}});
+  };
+  for (auto& p : peers) consume(p);
+  for (auto& p : peers) slow_need(!p.tcp->try_receive(),"bootstrap controls drained before readiness hold");
+  const auto initial = slow_state(server.room()); Json readiness = Json::array(), pending = Json::array();
+  std::size_t not_ready = 0, transport_high_water = 0, transport_bytes_high_water = 0;
+  OutboundFixture::gate(server,[&](std::uint64_t id, bool datagram) {
+    const auto actual = OutboundFixture::inspect(server,id); const bool ready = id != peers[0].connection;
+    const auto& usage = actual.at("memory");
+    transport_high_water = std::max(transport_high_water,usage.at("transport_owned_pending_buffers").get<std::size_t>());
+    transport_bytes_high_water = std::max(transport_bytes_high_water,usage.at("transport_owned_pending_bytes").get<std::size_t>());
+    slow_need(actual.at("control") <= 64 && actual.at("full") <= 1 && actual.at("delta") <= 1 && readiness.size() < 4096,
+      "bounded real queue at every readiness transition");
+    const auto peer = std::find_if(peers.begin(),peers.end(),[&](const auto& p) { return p.connection == id; });
+    slow_need(peer != peers.end(),"actual bound fixture connection");
+    Json row{{"client",peer->role},{"next_tick",server.room().executed_ticks()},{"channel",datagram ? "UDP" : "TCP"},
+      {"ready",ready},{"control",actual.at("control")},{"full",actual.at("full")},{"delta",actual.at("delta")},
+      {"retained_bytes",actual.at("retained_bytes")},{"memory",usage}};
+    for (const auto* kind : {"pending_full","pending_delta"}) {
+      row[kind] = nullptr;
+      if (!actual.at(kind).is_null()) row[kind] = Json{{"buffer_id",actual.at(kind).at("buffer_id")},
+        {"snapshot_seq",actual.at(kind).at("wire").at("snapshot_seq")},{"capacity_bytes",actual.at(kind).at("capacity_bytes")}};
+    }
+    readiness.push_back(std::move(row)); if (!ready) ++not_ready; return ready;
+  });
+  auto input = message("INPUT"); input.update(Json{{"session_id",peers[1].session},{"room_id",room_id},{"player_id",peers[1].player},
+    {"seq",event.at("seq")},{"target_tick",event.at("target_tick")},{"direction",event.at("direction")},
+    {"tag_target_player_id",nullptr},{"owner_epoch",0}});
+  peers[1].udp->send(input); const auto accepted = peers[1].udp->receive_type(server,"INPUT_ACK");
+  slow_need(accepted.at("code") == "ACCEPTED" && accepted.at("seq") == 1 && accepted.at("tick") == 0,"one actual accepted INPUT");
+  input.erase("session_id"); input["session_alias"] = "bravo";
+  ReplayRun run; run.records = Json::array(); Json hashes = Json::array();
+  for (int tick = 0; tick < 100; ++tick) {
+    server.advance_one_tick(); const auto canonical = canonical_state(server.room()), hash = sha256(canonical);
+    slow_need(server.room().executed_ticks() == tick+1 && clock.now_ms == (tick+1)*50 && server.replay().last_state_hash() == hash,
+      "actual owner advances one fixed tick without blocking transport");
+    for (std::size_t i = 0; i < peers.size(); ++i) {
+      const auto& p = server.room().players().at(peers[i].player); const auto& fixed = scenario.at("players").at(i);
+      const int x = fixed.at("spawn").at(0).get<int>(), y = fixed.at("spawn").at(1).get<int>();
+      slow_need(p.connected && !p.disconnect_deadline && p.x == (i == 1 ? std::min(100000,x+(tick+1)*400) : x) && p.y == y &&
+        p.score == 0 && p.last_accepted_seq() == (i == 1 ? 1U : 0U) && p.direction == (i == 1 ? Direction::east : Direction::stop) &&
+        p.pending.empty(),"fixed authoritative positions, bounds, sequences and connected state");
+    }
+    run.records.push_back(Json{{"tick",tick},{"state",slow_state(server.room())},{"canonical_record",canonical},{"state_hash",hash}}); hashes.push_back(hash);
+    if ((tick+1)%2 == 0) for (std::size_t i = 1; i < peers.size(); ++i) consume(peers[i]);
+    slow_need(!peers[0].udp->try_receive() && !peers[0].tcp->try_receive(),"held alpha has zero actual outbound deliveries");
+    auto actual = OutboundFixture::inspect(server,peers[0].connection); actual["tick"] = tick;
+    const auto stream = server.metrics().at("snapshot_streams").at(peers[0].player); actual["snapshot_stream"] = stream;
+    slow_need(actual.at("control") == 0 && actual.at("full") <= 1 && actual.at("delta") <= 1 && stream.at("acknowledged_seq") == 1 &&
+      actual.at("snapshots_sent") == 1 && actual.at("memory").at("transport_owned_pending_buffers") == 0,
+      "held alpha retains bounded snapshots at unchanged actual ACK1; no transport copy");
+    pending.push_back(std::move(actual));
+  }
+  const auto final = slow_state(server.room()), metrics = server.metrics(); const auto held = OutboundFixture::inspect(server,peers[0].connection);
+  slow_need(held.at("snapshots_generated") == 51 && held.at("snapshots_sent") == 1 && held.at("snapshots_coalesced") > 0 &&
+    held.at("snapshots_coalesced").get<std::uint64_t>()+held.at("full").get<std::uint64_t>()+held.at("delta").get<std::uint64_t>() == 50 &&
+    not_ready > 0 && !metrics.at("errors").contains("UDP_SEND_DROP"),"actual generated/coalesced retained buffers, no loss substitution");
+  Json healthy_counts, healthy_sequences, healthy_publications;
+  for (std::size_t i = 1; i < peers.size(); ++i) {
+    const auto& p = peers[i]; slow_need(p.latest == 51 && p.publications.size() == 51 &&
+      metrics.at("outbound_streams").at(p.player).at("snapshots_sent") == 51 &&
+      metrics.at("outbound_streams").at(p.player).at("snapshots_coalesced") == 0,"all51 healthy snapshots actually sent/applied/ACKed");
+    healthy_counts[p.role] = p.publications.size(); healthy_sequences[p.role] = p.sequences; healthy_publications[p.role] = p.publications;
+  }
+  // Export before the after-capture close; the immutable100-tick accepted log
+  // is the only input to the separately accounted offline reference process.
+  run.replay_bytes = server.replay().serialize(); const auto artifact = parse_replay_artifact(run.replay_bytes);
+  std::size_t admissions = 0; for (const auto& tick : artifact.at("ticks")) admissions += tick.at("events").size();
+  slow_need(artifact.at("ticks").size() == 100 && admissions == 1,"actual accepted journal contains one input and100 completed ticks");
+  auto fds = server.owned_descriptors(); for (const auto& p : peers) { fds.push_back(p.tcp->descriptor()); fds.push_back(p.udp->descriptor()); }
+  peers[0].tcp->close(); wait_for([&] { return server.cleanup().at("server_connections") == 3; },false);
+  slow_need(slow_state(server.room()) == final,"transport close releases buffers before owner state mutation");
+  const auto released = OutboundFixture::inspect(server,peers[0].connection);
+  slow_need(!released.at("connection_present").get<bool>() && released.at("memory").at("outbound_buffers") == 0 &&
+    released.at("memory").at("outbound_retained_bytes") == 0 && !descriptor_closed(peers[0].udp->descriptor()),
+    "alpha TCP close releases all transport queues/buffers while client UDP remains open");
+  server.drain_mailbox(); const auto after_close = slow_state(server.room()); const auto& alpha = server.room().players().at(peers[0].player);
+  slow_need(!alpha.connected && alpha.disconnect_deadline == 300 && alpha.direction == Direction::stop && alpha.pending.empty() &&
+    server.room().executed_ticks() == 100,"existing G11 disconnected grace preserved after final capture");
+  OutboundFixture::gate(server,{}); server.shutdown(); for (auto& p : peers) { p.tcp->close(); p.udp->close(); }
+  for (const int fd : fds) slow_need(descriptor_closed(fd),"every actual descriptor closed");
+  auto cleanup = server.cleanup(); for (const auto& [key,value] : cleanup.items()) { (void)key; slow_need(value == 0,"all active resources released"); }
+  const auto shutdown_metrics = server.metrics(); slow_need(Fd::live() == descriptors_before &&
+    shutdown_metrics.at("outbound_buffers_created") == shutdown_metrics.at("outbound_buffers_released"),"all actual owned buffer lifetimes and descriptors end");
+  cleanup.update(Json{{"descriptor_checks",fds.size()},{"all_descriptors_closed",true},{"tracked_descriptor_delta",0}});
+  run.evidence = Json{{"result","PASS"},{"mode","live"},{"thread","G12"},{"scenario_id",scenario.at("scenario_id")},{"process_id",::getpid()},
+    {"executed_ticks",100},{"virtual_ms",clock.now_ms},{"joins",joins},{"initial_state",initial},{"input",Json{{"request",input},{"response",accepted}}},
+    {"state_hashes",hashes},{"healthy_counts",healthy_counts},{"healthy_sequences",healthy_sequences},{"healthy_publications",healthy_publications},
+    {"initial_alpha_snapshot",peers[0].publications.at(0)},{"alpha_snapshots_during_hold",0},{"gate_installed_TCP_and_UDP",true},
+    {"readiness_trace",readiness},{"not_ready_callbacks",not_ready},{"pending_timeline",pending},{"held_final",held},
+    {"transport_owned_buffer_high_water",transport_high_water},{"transport_owned_bytes_high_water",transport_bytes_high_water},
+    {"final_state",final},{"alpha_transport_after_close",released},{"after_alpha_close",after_close},{"metrics_before_close",metrics},
+    {"accepted_journal_events",admissions},{"outbound_buffers_created",shutdown_metrics.at("outbound_buffers_created")},
+    {"outbound_buffers_released",shutdown_metrics.at("outbound_buffers_released")},{"network_fault_runs",0},{"load_runs",0},{"cleanup",cleanup}};
+  return run;
+}
+}
diff --git a/tests/g12.hpp b/tests/g12.hpp
new file mode 100644
index 0000000..7c6c260
--- /dev/null
+++ b/tests/g12.hpp
@@ -0,0 +1,18 @@
+#pragma once
+#include "g09.hpp"
+#ifndef ARENA_TEST_FIXTURES
+#error G12 readiness access is test-build only
+#endif
+namespace arena {
+struct OutboundFixture {
+  static std::uint64_t connection_id(Server& server, const std::string& session);
+  static void gate(Server& server, std::function<bool(std::uint64_t,bool)> ready);
+  static Json inspect(Server& server, std::uint64_t id);
+  static void control(Server& server, std::uint64_t id, const Json& value);
+  static void capture(Server& server, std::uint64_t id, bool full);
+  static void flush(Server& server, std::uint64_t id);
+};
+Json run_control_queue_probe();
+Json run_snapshot_queue_probe();
+ReplayRun run_backpressure_scenario(const Json& scenario);
+}
diff --git a/tests/g12_queue.cpp b/tests/g12_queue.cpp
new file mode 100644
index 0000000..a2ecce1
--- /dev/null
+++ b/tests/g12_queue.cpp
@@ -0,0 +1,161 @@
+#include "g12.hpp"
+#include <array>
+#include <chrono>
+#include <memory>
+
+namespace arena {
+namespace {
+void queue_need(bool value, const std::string& text) { if (!value) throw std::runtime_error("G12 queue: "+text); }
+Json memory(const Server& server) {
+  const auto all = server.metrics(); Json selected;
+  for (const auto* key : {"outbound_buffers","outbound_retained_bytes","outbound_buffer_high_water","outbound_bytes_high_water",
+      "outbound_buffers_created","outbound_buffers_released","transport_owned_pending_buffers","transport_owned_pending_bytes"}) selected[key] = all.at(key);
+  queue_need(all.at("outbound_buffers_created").get<std::uint64_t>() == all.at("outbound_buffers_released").get<std::uint64_t>()+
+    all.at("outbound_buffers").get<std::uint64_t>(),"actual move-only vector lifetime accounting");
+  return selected;
+}
+Json closed(Server& server, std::vector<TcpClient*> controls, std::vector<UdpClient*> realtime, const std::vector<int>& fds, int before) {
+  OutboundFixture::gate(server,{}); server.shutdown();
+  for (auto* peer : controls) peer->close(); for (auto* peer : realtime) peer->close();
+  for (const int fd : fds) queue_need(descriptor_closed(fd),"actual descriptor closed");
+  auto cleanup = server.cleanup(); for (const auto& [key,value] : cleanup.items()) { (void)key; queue_need(value == 0,"all live queue/transport/owner resources released"); }
+  queue_need(Fd::live() == before,"tracked descriptors released");
+  cleanup.update(Json{{"descriptor_checks",fds.size()},{"all_descriptors_closed",true},{"tracked_descriptor_delta",0}}); return cleanup;
+}
+}
+std::uint64_t OutboundFixture::connection_id(Server& server, const std::string& session) {
+  for (const auto& [fd,conn] : server.connections_) { (void)fd; if (conn.session_id == session) return conn.id; }
+  throw std::logic_error("test session is not connected");
+}
+void OutboundFixture::gate(Server& server, std::function<bool(std::uint64_t,bool)> ready) { server.fixture_outbound_ready_ = std::move(ready); }
+Json OutboundFixture::inspect(Server& server, std::uint64_t id) {
+  auto* conn = server.connection(id);
+  if (!conn) return Json{{"connection_present",false},{"control",0},{"full",0},{"delta",0},{"retained_bytes",0},{"memory",memory(server)}};
+  auto result = server.outbound_state(*conn); result["connection_present"] = true; result["controls"] = Json::array();
+  std::size_t bytes = 0, buffers = 0;
+  for (const auto& control : conn->outbound) {
+    result["controls"].push_back(Json{{"buffer_id",control.buffer_id},{"capacity_bytes",control.bytes.capacity()},
+      {"remaining_bytes",control.remaining().size()},{"wire",decode_complete_frame(control.bytes)}});
+    bytes += control.bytes.capacity(); ++buffers;
+  }
+  const auto snapshot = [&](const auto& pending, const char* key) {
+    result[key] = nullptr;
+    if (pending) {
+      result[key] = Json{{"buffer_id",pending->buffer_id},{"capacity_bytes",pending->bytes.capacity()},{"wire",decode_datagram(pending->bytes)}};
+      bytes += pending->bytes.capacity(); ++buffers;
+    }
+  };
+  snapshot(conn->pending_full,"pending_full"); snapshot(conn->pending_delta,"pending_delta");
+  queue_need(bytes == result.at("retained_bytes") && buffers == conn->outbound.size()+conn->pending_full.has_value()+conn->pending_delta.has_value(),
+    "inspect actual vector capacities and unique ownership"); result["memory"] = memory(server); return result;
+}
+void OutboundFixture::control(Server& server, std::uint64_t id, const Json& value) { server.queue(id,value); }
+void OutboundFixture::capture(Server& server, std::uint64_t id, bool full) {
+  auto* conn = server.connection(id); queue_need(conn != nullptr,"capture connection");
+  const auto hash = sha256(canonical_state(server.room()));
+  if (full) conn->snapshots.acknowledge(1,hash,true);
+  server.publish_snapshot(*conn,hash);
+}
+void OutboundFixture::flush(Server& server, std::uint64_t id) {
+  if (auto* conn = server.connection(id)) {
+    const int fd = conn->fd.get(); server.write_datagrams(*conn); server.write_ready(fd); server.update_udp_write_interest();
+  }
+}
+Json run_control_queue_probe() {
+  const int before = Fd::live(); ManualClock clock; Server server(clock); TcpClient peer(server.port());
+  peer.send(message("HELLO")); const auto welcome = peer.receive_type(server,"WELCOME");
+  const auto id = OutboundFixture::connection_id(server,welcome.at("session_id").get<std::string>());
+  auto fds = server.owned_descriptors(); fds.push_back(peer.descriptor());
+  const auto initial_memory = memory(server), initial_sent = server.metrics().at("sent_messages"); Json barriers = Json::array();
+  OutboundFixture::gate(server,[&](std::uint64_t connection, bool datagram) {
+    queue_need(connection == id && !datagram,"pure control barrier at actual TCP dequeue");
+    barriers.push_back(OutboundFixture::inspect(server,id)); return false;
+  });
+  for (int ordinal = 0; ordinal < 63; ++ordinal) {
+    auto control = message("ROOM_CREATED"); control.update(Json{{"room_id","control-"+std::to_string(ordinal)},{"status","LOBBY"}});
+    OutboundFixture::control(server,id,control);
+  }
+  const auto at63 = OutboundFixture::inspect(server,id);
+  queue_need(at63.at("connection_present") && at63.at("control") == 63 && !peer.peer_closed(),"63 pending controls remain open");
+  OutboundFixture::flush(server,id);
+  OutboundFixture::control(server,id,message("ROOM_CREATED"));
+  const auto after = OutboundFixture::inspect(server,id), final_memory = memory(server);
+  const auto deadline = std::chrono::steady_clock::now()+std::chrono::seconds(5);
+  while (!peer.peer_closed() && std::chrono::steady_clock::now() < deadline) server.poll_io(1);
+  queue_need(barriers.size() == 2 && barriers.back().at("control") == 64 && barriers.back().at("controls").size() == 64 &&
+    barriers.back().at("controls").back().at("wire").at("code") == "CONTROL_BACKPRESSURE", "64th attempt is the bounded terminal report");
+  queue_need(!after.at("connection_present").get<bool>() && peer.peer_closed() && server.metrics().at("errors").at("CONTROL_BACKPRESSURE") == 1,
+    "terminal connection closes despite not-ready transport");
+  queue_need(final_memory.at("outbound_buffers") == 0 && final_memory.at("outbound_retained_bytes") == 0 &&
+    final_memory.at("outbound_buffer_high_water") == 64 && server.metrics().at("outbound_control_high_water") == 64 &&
+    final_memory.at("outbound_buffers_created").get<std::uint64_t>()-initial_memory.at("outbound_buffers_created").get<std::uint64_t>() == 64 &&
+    final_memory.at("outbound_buffers_released").get<std::uint64_t>()-initial_memory.at("outbound_buffers_released").get<std::uint64_t>() == 64 &&
+    server.metrics().at("sent_messages") == initial_sent,"all64 cancelled vectors freed, none counted sent, no65th buffer");
+  const auto cleanup = closed(server,{&peer},{},fds,before);
+  return Json{{"case","control-threshold"},{"executed_ticks",0},{"at63",at63},{"pre_dequeue_barriers",barriers},
+    {"after_terminal",after},{"created_during_probe",64},{"released_during_probe",64},{"sent_during_probe",0},{"cleanup",cleanup}};
+}
+Json run_snapshot_queue_probe() {
+  const int before = Fd::live(); ManualClock clock; Server server(clock);
+  std::array<std::unique_ptr<TcpClient>,2> tcp; std::array<std::unique_ptr<UdpClient>,2> udp;
+  std::array<std::string,2> sessions, players;
+  for (std::size_t i = 0; i < 2; ++i) {
+    tcp[i] = std::make_unique<TcpClient>(server.port()); tcp[i]->send(message("HELLO"));
+    sessions[i] = tcp[i]->receive_type(server,"WELCOME").at("session_id").get<std::string>();
+    udp[i] = std::make_unique<UdpClient>(server.udp_port());
+  }
+  auto create = message("CREATE_ROOM"); create["session_id"] = sessions[0]; tcp[0]->send(create);
+  const auto room = tcp[0]->receive_type(server,"ROOM_CREATED").at("room_id").get<std::string>();
+  for (std::size_t i = 0; i < 2; ++i) {
+    auto join = message("JOIN_ROOM"); join.update(Json{{"session_id",sessions[i]},{"room_id",room}}); tcp[i]->send(join);
+    const auto result = tcp[i]->receive_type(server,"ROOM_JOINED"); players[i] = result.at("player_id").get<std::string>();
+    queue_need(result.at("status") == "LOBBY","normal unbound pure fixture joins");
+  }
+  for (std::size_t i = 0; i < 2; ++i) udp[i]->bind(*tcp[i],server,sessions[i]);
+  for (std::size_t i = 0; i < 2; ++i) {
+    const auto full = udp[i]->receive_type(server,"SNAPSHOT"); queue_need(full.at("snapshot_seq") == 1 && full.at("tick") == -1,"zero-tick initial full");
+    auto ack = message("SNAPSHOT_ACK"); ack.update(Json{{"session_id",sessions[i]},{"room_id",room},{"player_id",players[i]},
+      {"owner_epoch",0},{"snapshot_seq",1},{"state_hash",full.at("state_hash")}}); udp[i]->send(ack);
+    const auto deadline = std::chrono::steady_clock::now()+std::chrono::seconds(5);
+    while (server.metrics().at("snapshot_streams").at(players[i]).at("acknowledged_seq") != 1 && std::chrono::steady_clock::now() < deadline) server.pump(1);
+    queue_need(server.metrics().at("snapshot_streams").at(players[i]).at("acknowledged_seq") == 1,"valid actual acknowledged base1");
+  }
+  const auto id = OutboundFixture::connection_id(server,sessions[0]); auto fds = server.owned_descriptors();
+  for (std::size_t i = 0; i < 2; ++i) { fds.push_back(tcp[i]->descriptor()); fds.push_back(udp[i]->descriptor()); }
+  const auto initial_memory = memory(server); std::size_t not_ready = 0;
+  OutboundFixture::gate(server,[&](std::uint64_t connection, bool) { if (connection == id) { ++not_ready; return false; } return true; });
+  auto a = message("ROOM_CREATED"); a.update(Json{{"room_id",room},{"status","LOBBY"}});
+  const auto b = error_message("FRAME_SIZE_INVALID","fixture terminal control B");
+  OutboundFixture::control(server,id,a); OutboundFixture::control(server,id,b); OutboundFixture::flush(server,id);
+  Json transitions = Json::array();
+  for (const bool full : {true,false,false,true}) {
+    OutboundFixture::capture(server,id,full); transitions.push_back(OutboundFixture::inspect(server,id));
+  }
+  const auto& full2 = transitions.at(0); const auto& delta3 = transitions.at(1); const auto& delta4 = transitions.at(2); const auto& full5 = transitions.at(3);
+  queue_need(full2.at("pending_full").at("wire").at("snapshot_seq") == 2 && delta3.at("pending_delta").at("wire").at("snapshot_seq") == 3 &&
+    delta4.at("pending_delta").at("wire").at("snapshot_seq") == 4 && full5.at("pending_full").at("wire").at("snapshot_seq") == 5,
+    "exact FULL2 DELTA3 DELTA4 FULL5 sequence");
+  queue_need(delta3.at("pending_delta").at("wire").at("base_snapshot_seq") == 1 && delta4.at("pending_delta").at("wire").at("base_snapshot_seq") == 1,
+    "both deltas use actual valid acknowledged base");
+  for (const auto& row : transitions) queue_need(row.at("control") == 2 && row.at("controls").at(0).at("wire") == a &&
+    row.at("controls").at(1).at("wire") == b && row.at("full") <= 1 && row.at("delta") <= 1 &&
+    row.at("memory").at("transport_owned_pending_buffers") == 0,"controls unchanged/in-order; no hidden transport copies");
+  queue_need(full5.at("full") == 1 && full5.at("delta") == 0 && full5.at("snapshots_coalesced") == 3 &&
+    full5.at("memory").at("outbound_buffers_created").get<std::uint64_t>()-initial_memory.at("outbound_buffers_created").get<std::uint64_t>() == 6 &&
+    full5.at("memory").at("outbound_buffers_released").get<std::uint64_t>()-initial_memory.at("outbound_buffers_released").get<std::uint64_t>() == 3,
+    "three actual superseded buffers destroyed; remaining two controls and one full");
+  queue_need(full2.at("pending_full").at("buffer_id") != full5.at("pending_full").at("buffer_id") &&
+    delta3.at("pending_delta").at("buffer_id") != delta4.at("pending_delta").at("buffer_id"),"distinct real buffer lifetimes, not an operation counter");
+  OutboundFixture::gate(server,{}); OutboundFixture::flush(server,id);
+  queue_need(tcp[0]->receive(server) == a && tcp[0]->receive(server) == b,"actual TCP controls delivered in original order");
+  queue_need(udp[0]->receive_type(server,"SNAPSHOT") == full5.at("pending_full").at("wire"),"only surviving full reaches real UDP client");
+  const auto drained = OutboundFixture::inspect(server,id), final_memory = memory(server);
+  queue_need(drained.at("control") == 0 && drained.at("full") == 0 && drained.at("delta") == 0 && !udp[0]->try_receive() &&
+    final_memory.at("outbound_buffers") == 0 && final_memory.at("outbound_retained_bytes") == 0 &&
+    final_memory.at("outbound_buffers_released").get<std::uint64_t>()-initial_memory.at("outbound_buffers_released").get<std::uint64_t>() == 6 &&
+    server.room().executed_ticks() == 0,"drain releases all six real buffers without a simulation tick");
+  const auto cleanup = closed(server,{tcp[0].get(),tcp[1].get()},{udp[0].get(),udp[1].get()},fds,before);
+  return Json{{"case","mixed-snapshot-coalescing"},{"executed_ticks",0},{"transitions",transitions},{"not_ready_callbacks",not_ready},
+    {"discarded_snapshot_buffers",3},{"created_during_probe",6},{"released_during_probe",6},{"drained",drained},{"cleanup",cleanup}};
+}
+}
diff --git a/tests/scenario_main.cpp b/tests/scenario_main.cpp
index c588ccb..509463b 100644
--- a/tests/scenario_main.cpp
+++ b/tests/scenario_main.cpp
@@ -2,6 +2,7 @@
 #include "g09.hpp"
 #include "g10.hpp"
 #include "g11.hpp"
+#include "g12.hpp"
 #ifndef ARENA_TEST_FIXTURES
 #error Scenario fixture executable requires its separate test core
 #endif
@@ -33,7 +34,7 @@ int main(int argc, char** argv) {
       const bool variant = argc == 6 && std::string(argv[4]) == "--variant" && std::string(argv[5]) == "rejected-removed";
       if (argc != 4 && !variant) throw std::invalid_argument("unknown scenario variant");
       const auto scenario = arena::read_json_file(input);
-      if (scenario.at("thread") != "G07" && scenario.at("thread") != "G09") {
+      if (scenario.at("thread") != "G07" && scenario.at("thread") != "G09" && scenario.at("thread") != "G12") {
         if (variant) throw std::invalid_argument("variant is only active for G07");
         const auto evidence = scenario.at("thread") == "G08" ? arena::run_snapshot_scenario(scenario) :
           scenario.at("thread") == "G10" ? arena::run_ack_scenario(scenario) :
@@ -43,7 +44,10 @@ int main(int argc, char** argv) {
           {"scenario_id",evidence.at("scenario_id")},{"evidence",output.string()},{"cleanup",evidence.at("cleanup")}}.dump() << '\n';
         return evidence.at("result") == "PASS" ? 0 : 1;
       }
-      if (scenario.at("thread") == "G09") {
+      if (scenario.at("thread") == "G12") {
+        if (variant) throw std::invalid_argument("variant is only active for G07");
+        run = arena::run_backpressure_scenario(scenario);
+      } else if (scenario.at("thread") == "G09") {
         if (variant) throw std::invalid_argument("variant is only active for G07");
         run = arena::run_udp_scenario(scenario);
       } else run = arena::run_replay_scenario(scenario,variant);
diff --git a/tests/tests.cpp b/tests/tests.cpp
index f7b490b..f00f98e 100644
--- a/tests/tests.cpp
+++ b/tests/tests.cpp
@@ -1,5 +1,6 @@
 #include "scenario.hpp"
 #include "g09.hpp"
+#include "g12.hpp"
 #include <algorithm>
 #include <atomic>
 #include <cerrno>
@@ -809,7 +810,9 @@ int main(int argc, char** argv) {
       {"G06_pending_bound_after_rate", pending_bound_after_rate_activation},
       {"G07_zero_tick_canonical_SHA256", canonical_hash_bytes_without_ticks},
       {"G07_zero_tick_storage_packaging", replay_storage_and_packaging_without_ticks},
-      {"G08_zero_tick_33_snapshot_retention", snapshot_retention_without_ticks}};
+      {"G08_zero_tick_33_snapshot_retention", snapshot_retention_without_ticks},
+      {"G12_zero_tick_control_threshold", [] { std::cout << Json{{"G12_queue_probe",run_control_queue_probe()}}.dump() << '\n'; }},
+      {"G12_zero_tick_mixed_snapshot_coalescing", [] { std::cout << Json{{"G12_queue_probe",run_snapshot_queue_probe()}}.dump() << '\n'; }}};
   } else if (std::string(argv[1]) == "integration") {
     tests = {{"real_TCP_authority_and_cleanup", real_tcp_authority_and_cleanup}, {"standalone_process_shutdown", [&] {
       standalone_process_shutdown(std::filesystem::absolute(argv[0]).parent_path() / "arena"); }},
