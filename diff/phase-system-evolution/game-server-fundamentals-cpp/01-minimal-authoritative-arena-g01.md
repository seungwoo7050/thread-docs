# 최소 Authoritative Arena

## `feat: establish kqueue authoritative arena baseline`

diff --git a/.gitignore b/.gitignore
new file mode 100644
index 0000000..e878ad2
--- /dev/null
+++ b/.gitignore
@@ -0,0 +1,4 @@
+build*/
+artifacts/
+*.dSYM/
+.DS_Store
diff --git a/CMakeLists.txt b/CMakeLists.txt
new file mode 100644
index 0000000..2b35fc8
--- /dev/null
+++ b/CMakeLists.txt
@@ -0,0 +1,33 @@
+cmake_minimum_required(VERSION 3.31.6)
+project(arena_session_server VERSION 0.1.0 LANGUAGES CXX)
+if(NOT APPLE)
+  message(FATAL_ERROR "This track freezes the macOS kqueue backend; other backends are out of G01 scope")
+endif()
+if(NOT CMAKE_CXX_COMPILER_ID STREQUAL "AppleClang" OR NOT CMAKE_CXX_COMPILER_VERSION VERSION_EQUAL "21.0.0.21000101")
+  message(FATAL_ERROR "dependencies.lock freezes AppleClang 21.0.0")
+endif()
+if(NOT CMAKE_VERSION VERSION_EQUAL "3.31.6")
+  message(FATAL_ERROR "dependencies.lock freezes CMake 3.31.6")
+endif()
+set(CMAKE_CXX_STANDARD 20)
+set(CMAKE_CXX_STANDARD_REQUIRED ON)
+set(CMAKE_CXX_EXTENSIONS OFF)
+option(ARENA_TSAN "Instrument the fixed ownership/unit suite with ThreadSanitizer" OFF)
+if(ARENA_TSAN)
+  add_compile_options(-fsanitize=thread -g)
+  add_link_options(-fsanitize=thread)
+endif()
+find_package(nlohmann_json 3.12.0 EXACT CONFIG REQUIRED)
+add_library(arena_core src/game.cpp src/transport.cpp src/scenario.cpp)
+target_include_directories(arena_core PUBLIC src)
+target_link_libraries(arena_core PUBLIC nlohmann_json::nlohmann_json)
+target_compile_options(arena_core PRIVATE -Wall -Wextra -Wpedantic -Werror)
+add_executable(arena src/main.cpp)
+target_link_libraries(arena PRIVATE arena_core)
+target_compile_options(arena PRIVATE -Wall -Wextra -Wpedantic -Werror)
+add_executable(arena_tests tests/tests.cpp)
+target_link_libraries(arena_tests PRIVATE arena_core)
+target_compile_options(arena_tests PRIVATE -Wall -Wextra -Wpedantic -Werror)
+enable_testing()
+add_test(NAME unit COMMAND arena_tests unit)
+add_test(NAME integration COMMAND arena_tests integration)
diff --git a/TRACK.md b/TRACK.md
new file mode 100644
index 0000000..69f9422
--- /dev/null
+++ b/TRACK.md
@@ -0,0 +1,111 @@
+# fundamentals-cpp — G01 baseline
+
+SPEC_REVISION: `5a6e4a2f8fc71d4be18c3279583bfc2558d5c232`
+
+Completion profile: `realtime-core`
+
+Branch: `track/fundamentals-cpp` (orphan)
+
+## Frozen toolchain and dependency
+
+- C++20, Apple clang 21.0.0 (`clang-2100.1.1.101`), `/usr/bin/clang++`.
+- CMake identifies the same compiler build as `21.0.0.21000101`; its full identifier is enforced, not truncated banner text.
+- CMake 3.31.6, Release build with `-Wall -Wextra -Wpedantic -Werror`.
+- nlohmann_json 3.12.0, exact CMake package under `/opt/homebrew`.
+- Darwin 25.6.0 arm64, direct POSIX sockets and kqueue; no network framework.
+- ArenaCheck 1: in-tree exception-based C++ unit/integration harness, no downloaded test dependency.
+- Version and installed header hashes are frozen in `dependencies.lock`. CMake rejects a different compiler, CMake or JSON version. No external services.
+
+## Command contract
+
+Run from this worktree; `track` resolves its own source directory. Build never runs tests implicitly.
+
+```sh
+./track build
+./track unit-test
+./track integration-test
+./track scenario-run /absolute/path/to/G01.json /absolute/path/to/evidence.json
+./track replay-verify /absolute/path/to/replay.json /absolute/path/to/evidence.json
+./track server /absolute/path/to/config.json
+```
+
+`replay-verify` exits 2 with `NOT_ACTIVATED` / G07. It does not invent a replay implementation.
+`ARENA_BUILD_DIR` selects a clean build directory. `ARENA_EVIDENCE_DIR` selects command logs.
+The build wrapper invokes configure and compile separately; both are logged. Tests invoke the existing executable directly.
+
+Exact Release build commands:
+
+```sh
+cmake -S "$PWD" -B "$PWD/build" -DCMAKE_BUILD_TYPE=Release -DCMAKE_PREFIX_PATH=/opt/homebrew -DCMAKE_CXX_COMPILER=/usr/bin/clang++ -DARENA_TSAN=OFF
+cmake --build "$PWD/build" --parallel 2
+```
+
+The fixed ownership test is also run with the available Apple ThreadSanitizer:
+
+```sh
+ARENA_BUILD_DIR="$PWD/build-tsan" ARENA_TSAN=ON ./track build
+ARENA_BUILD_DIR="$PWD/build-tsan" TSAN_OPTIONS=halt_on_error=1 ./track unit-test
+```
+
+CMake's normal `CXXFLAGS`/`LDFLAGS` also support independent instrumentation. The `ARENA_TSAN` option only adds compiler/linker flags.
+Network tests require permission to bind loopback sockets in restricted execution environments. No Internet is used.
+
+The server config accepts `listen_port` (0 chooses an OS-assigned port) and `clock: "manual"`.
+It prints a JSON `READY` line, services real TCP, and accepts local operator lines on stdin:
+`tick` advances one authoritative 50ms step if running; `state` prints current state; `stop`, EOF, SIGTERM or SIGINT closes the server.
+Operator commands are local process control, not extra wire protocol messages. A 10ms I/O wait does not advance simulation.
+
+## Current guarantee and ownership
+
+G01 establishes a baseline; no previous implementation was damaged or modified to manufacture a failure.
+One process owns one Room. Join commit order determines spawn slots. The second TCP-ready player starts tick 0.
+Only integer arithmetic updates movement, clamp, TAG and score. TAG is one-shot; direction persists.
+The session executes ticks 0–1199, then sends authoritative `ROOM_FINISHED` and stops advancing.
+
+- Connection lifetime: `Server::connections_`, each descriptor owned by move-only `Fd`.
+- Session identity: server-generated opaque per-connection identifier; no reconnect credential or persistence in G01.
+- Room state: the server's one execution context, enforced by Room owner-thread checks.
+- Mailbox: kqueue read callback produces bounded `Envelope` messages; `drain_mailbox` consumes them after I/O callbacks return. Only that owner phase or explicit tick mutates Room.
+- Serialization: each queued `PendingWrite` owns its byte vector and unsent offset until write completes. Nonblocking partial writes and EAGAIN retain the suffix.
+- Shutdown coordinator: `Server::shutdown`; it stops accepting, drains intent, closes Room, attempts bounded final control flushing, releases connections and kqueue, and clears queues.
+- No worker thread, OS tick timer, external process, distributed lease renewer or snapshot retention is allocated by the server.
+
+`SNAPSHOT` is a minimal baseline state notice at join/start/close. It has no snapshot sequence, delta base, hash or periodic cadence. `ROOM_FINISHED` includes the same authoritative final view. Full/delta replication first activates in G08.
+Both clients observe LOBBY, RUNNING and CLOSED via actual TCP. The canonical 1200-tick run additionally observes FINISHED.
+
+## Explicit resource bounds
+
+| Resource | G01 bound and excess behavior |
+|---|---|
+| TCP JSON payload | 16,384 bytes; rejected/disconnected on oversized or incomplete G01 frame |
+| Read buffer | one stack array of 16,388 bytes per read; no retained incremental parser |
+| Connections | 512; excess accepted descriptor closed and ADMISSION_REJECTED metric recorded |
+| Rooms | one; extra create gets ROOM_NOT_JOINABLE |
+| Players | eight slots maximum; running Room rejects new joins |
+| Transport mailbox | 512 messages total, 64 per connection; INPUT_QUEUE_FULL |
+| Pending inputs | 64 per player; 65th is INPUT_QUEUE_FULL, accepted inputs retained until tick |
+| Outbound control | 64 messages per connection, each bounded frame; overflow disconnect with CONTROL_BACKPRESSURE metric |
+| I/O iteration | 64 kqueue events, up to 64 accepts, up to 64 writes per ready connection |
+| Manual tick work | one explicit tick call; no accumulator or catch-up implemented |
+| Client evidence | 4,096 received messages per client; overflow is a test failure |
+| Runner input/output | 1 MiB JSON input, 512 input commands, 32 setup commands; 4 MiB evidence output |
+| Operator input | 4,096 bytes; overflow terminates with explicit failure |
+| Error text | 160 bytes on wire; CLI failure text 256 bytes |
+| Shutdown flush | 500ms wall ceiling; timeout is explicit metric, never a simulation clock input |
+
+Frame-bound, input-bound and RAII assertions run in unit tests; real socket descriptor cleanup and queue high-water checks run in integration/scenario tests. A foreign-thread Room mutation is rejected before touching state.
+The standalone integration case starts the CLI on port 0, waits for READY, sends one real HELLO and receives WELCOME, sends SIGTERM once, requires exit 0 and reaping within a fixed 5s process deadline, and rebinds the released listener port. The signal handler only sets a stop flag; the ordinary owner loop performs cleanup.
+
+## Fixed verification and budget
+
+Initial attempt budget: at most 8 configure/compile invocations (conservatively counting both), 4 unit suites, 2 integration suites, 1 canonical scenario, 0 load and 0 fault runs.
+This file fixes commands before the first build. Every wrapper command records epoch start, duration in seconds, exit code, exact executable/arguments and log path in `artifacts/evidence/runs.tsv`, including failed commands.
+Human-readable actual results are in `evidence/G01.md`; generated binaries and detailed dumps stay ignored.
+
+The canonical scenario is supplied by main and read at runtime. The runner does not contain canonical positions/scores as output constants.
+It resolves role names to server-generated session/player IDs, sends complete frames over loopback, waits for owner-phase INPUT_ACK at each intended tick boundary, then advances the injected manual clock. It compares final TCP messages against the same production Room's view, checks client-observed CLOSED, and verifies actual descriptor invalidation using `fcntl(F_GETFD)`.
+
+## Deliberate next constraints
+
+The G01 server assumes exactly one complete frame per nonblocking read and keeps no incomplete frame state. Fragmentation/coalescing and strict malformed JSON validation are G02 work. This limitation is reported, not hidden behind the test client's transport.
+Detailed identity/lifecycle matrices (G03), clock accumulator/catch-up (G04), input sequence/target tick (G05), abuse matrix (G06), replay/hash (G07), full/delta cadence (G08), UDP, reconnect, slow-consumer coalescing and many-room scheduling remain inactive.
diff --git a/dependencies.lock b/dependencies.lock
new file mode 100644
index 0000000..7b59e50
--- /dev/null
+++ b/dependencies.lock
@@ -0,0 +1,20 @@
+{
+  "thread": "G01",
+  "spec_revision": "5a6e4a2f8fc71d4be18c3279583bfc2558d5c232",
+  "platform": "Darwin 25.6.0 arm64",
+  "reactor": "POSIX kqueue",
+  "language": "C++20",
+  "compiler": "Apple clang 21.0.0 (clang-2100.1.1.101)",
+  "cmake_compiler_version": "21.0.0.21000101",
+  "compiler_path": "/usr/bin/clang++",
+  "cmake": "3.31.6",
+  "json": {
+    "name": "nlohmann_json",
+    "version": "3.12.0",
+    "cmake_prefix": "/opt/homebrew",
+    "json_hpp_sha256": "88d7eb42b19599cab48856576dd4ea52538df72ad7c57200c50de98bda246458",
+    "abi_macros_hpp_sha256": "cebb5441f1ce851444f47a7d8ea0446339ee499df83215ebd873c8347a191bd7"
+  },
+  "test_framework": "ArenaCheck 1 (in-tree C++20 exception-based harness)",
+  "external_services": []
+}
diff --git a/evidence/G01-canonical-summary.json b/evidence/G01-canonical-summary.json
new file mode 100644
index 0000000..c554e31
--- /dev/null
+++ b/evidence/G01-canonical-summary.json
@@ -0,0 +1,84 @@
+{
+  "scenario_id": "G01-single-room",
+  "thread": "G01",
+  "contract_version": 1,
+  "seed": 7050,
+  "transport": "real-loopback-TCP/kqueue",
+  "clock": {
+    "kind": "manual",
+    "tick_duration_ms": 50
+  },
+  "executed_ticks": 1200,
+  "last_tick": 1199,
+  "manual_clock_ms": 60000,
+  "players": {
+    "alpha": {
+      "connectivity": "CONNECTED",
+      "direction": "STOP",
+      "last_tag_tick": 200,
+      "player_id": "player-59ff17b179156b44-0000000001",
+      "score": 1,
+      "slot": 0,
+      "x": 50000,
+      "y": 50000
+    },
+    "bravo": {
+      "connectivity": "CONNECTED",
+      "direction": "STOP",
+      "last_tag_tick": -20,
+      "player_id": "player-59ff17b179156b44-0000000002",
+      "score": 0,
+      "slot": 1,
+      "x": 50000,
+      "y": 50000
+    }
+  },
+  "lifecycle": [
+    "LOBBY",
+    "RUNNING",
+    "FINISHED",
+    "CLOSED"
+  ],
+  "client_lifecycle": {
+    "alpha": [
+      "LOBBY",
+      "RUNNING",
+      "FINISHED",
+      "CLOSED"
+    ],
+    "bravo": [
+      "LOBBY",
+      "RUNNING",
+      "FINISHED",
+      "CLOSED"
+    ]
+  },
+  "metrics": {
+    "connection_high_water": 2,
+    "errors": {},
+    "input_per_player_high_water": 1,
+    "mailbox_high_water": 1,
+    "max_read_bytes": 250,
+    "outbound_control_high_water": 3,
+    "partial_writes": 0,
+    "received_messages": 11,
+    "sent_messages": 19
+  },
+  "cleanup": {
+    "all_descriptors_closed": true,
+    "client_descriptors": 0,
+    "descriptor_checks": 6,
+    "disconnect_notifications": 0,
+    "mailbox_messages": 0,
+    "outbound_messages": 0,
+    "pending_inputs": 0,
+    "server_connections": 0,
+    "server_descriptors": 0,
+    "timers": 0,
+    "tracked_descriptor_delta": 0,
+    "worker_threads": 0
+  },
+  "state_hashes": "INACTIVE_UNTIL_G07",
+  "result": "PASS",
+  "scenario_sha256": "dae00089308ed65d27c1a196308216bb8b1abac0586a86dbfa83a797eb1dc51a"
+}
diff --git a/evidence/G01-runs.tsv b/evidence/G01-runs.tsv
new file mode 100644
index 0000000..fd93cd6
--- /dev/null
+++ b/evidence/G01-runs.tsv
@@ -0,0 +1,12 @@
+1787879547	1	1	cmake -S /private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp -B /private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build -DCMAKE_BUILD_TYPE=Release -DCMAKE_PREFIX_PATH=/opt/homebrew -DCMAKE_CXX_COMPILER=/usr/bin/clang++ -DARENA_TSAN=OFF	/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/evidence/configure-1787879547-21028.log
+1787879565	0	0	cmake -S /private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp -B /private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build -DCMAKE_BUILD_TYPE=Release -DCMAKE_PREFIX_PATH=/opt/homebrew -DCMAKE_CXX_COMPILER=/usr/bin/clang++ -DARENA_TSAN=OFF	/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/evidence/configure-1787879565-21386.log
+1787879565	9	0	cmake --build /private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build --parallel 2	/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/evidence/build-1787879565-21386.log
+1787879585	0	0	/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build/arena_tests unit	/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/evidence/unit-1787879585-21789.log
+1787879596	0	0	/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build/arena_tests integration	/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/evidence/integration-1787879596-21882.log
+1787879643	0	0	cmake -S /private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp -B /private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-tsan -DCMAKE_BUILD_TYPE=Release -DCMAKE_PREFIX_PATH=/opt/homebrew -DCMAKE_CXX_COMPILER=/usr/bin/clang++ -DARENA_TSAN=ON	/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/evidence/configure-1787879643-22563.log
+1787879643	10	0	cmake --build /private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-tsan --parallel 2	/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/evidence/build-1787879643-22563.log
+1787879661	1	0	/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-tsan/arena_tests unit	/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/evidence/unit-1787879661-22998.log
+1787879675	1	0	/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-tsan/arena_tests integration	/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/evidence/integration-1787879675-23135.log
+1787879706	1	0	/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-tsan/arena scenario-run /Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G01.json /private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/G01-canonical.json	/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/evidence/scenario-1787879706-23497.log
+1787879723	0	0	cmake -S /private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp -B /private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build -DCMAKE_BUILD_TYPE=Release -DCMAKE_PREFIX_PATH=/opt/homebrew -DCMAKE_CXX_COMPILER=/usr/bin/clang++ -DARENA_TSAN=OFF	/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/evidence/configure-1787879723-23766.log
+1787879723	4	0	cmake --build /private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build --parallel 2	/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/evidence/build-1787879723-23766.log
diff --git a/evidence/G01.md b/evidence/G01.md
new file mode 100644
index 0000000..5e40a78
--- /dev/null
+++ b/evidence/G01.md
@@ -0,0 +1,84 @@
+# G01 verification — initial attempt
+
+- THREAD: G01
+- BRANCH: track/fundamentals-cpp
+- PROFILE: realtime-core
+- SPEC_REVISION: 5a6e4a2f8fc71d4be18c3279583bfc2558d5c232
+- ATTEMPT: initial, repairs 0
+- START: UNBORN orphan
+- END: the commit containing this file; resolve with `git log -1 --format=%H -- evidence/G01.md`.
+- COMMITS: one coherent baseline feature, its necessary tests, version freeze and evidence; no history rewrite or progress tag by implementer.
+
+## Establishment
+
+There was no previous implementation. This Thread established the real kqueue/TCP single-room baseline rather than manufacturing a failure.
+The canonical input was main's `index/scenarios/G01.json`, SHA-256 `dae00089308ed65d27c1a196308216bb8b1abac0586a86dbfa83a797eb1dc51a`, checked immediately before its only execution.
+No scenario parameter, clock, player count, acceptance threshold, seed or expected outcome was changed.
+
+## Verification commands and actual results
+
+All commands ran in `/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp`.
+`G01-runs.tsv` contains every configure/compile/suite/scenario command, including the failure, with exact arguments, exit code, duration in whole seconds and detailed log path. Durations have one-second resolution, so subsecond successes may read zero.
+
+| Order | Command | Exit | Actual observation |
+|---|---|---:|---|
+| 1 | `./track build` configure | 1 | CMake reports Apple compiler as 21.0.0.21000101, unlike banner 21.0.0; exact detection corrected without changing compiler |
+| 2 | `./track build` configure + compile | 0 / 0 | Clean Release build, warnings as errors; 0s configure, 9s compile |
+| 3 | `./track unit-test` | 0 | 7/7 tests, zero tracked descriptors |
+| 4 | `./track integration-test` | 0 | 2/2 fixed cases; real TCP authority/cleanup and standalone process SIGTERM/rebind |
+| 5 | `ARENA_BUILD_DIR="$PWD/build-tsan" ARENA_TSAN=ON ./track build` | 0 / 0 | Clean instrumented Release build; 0s configure, 10s compile |
+| 6 | `ARENA_BUILD_DIR="$PWD/build-tsan" TSAN_OPTIONS=halt_on_error=1 ./track unit-test` | 0 | 7/7, no TSan diagnostic, zero tracked descriptors |
+| 7 | `ARENA_BUILD_DIR="$PWD/build-tsan" TSAN_OPTIONS=halt_on_error=1 ./track integration-test` | 0 | 2/2, no TSan diagnostic, child exit 0/reaped, released port rebindable |
+| 8 | `ARENA_BUILD_DIR="$PWD/build-tsan" TSAN_OPTIONS=halt_on_error=1 ./track scenario-run /Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G01.json /private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/G01-canonical.json` | 0 | Canonical 1200 ticks, final score 1/0, all resources released; no TSan diagnostic |
+| 9 | `./track build` configure + compile | 0 / 0 | Final normal Release rebuild after SIGTERM/EINTR handling; 0s configure, 4s compile |
+
+Between the initial and final instrumented verification the operator loop was tightened so SIGTERM interrupts do not become an I/O failure. The canonical run and final TSan suites exercised the final source. The final uninstrumented executable was rebuilt from that same source. No source change occurred after canonical verification.
+
+The first staged `git diff --cached --check` exited 2 for two Markdown trailing-space line breaks in TRACK.md. Those documentation spaces were removed before commit; no implementation or scenario changed. This static check is not a build/test run.
+
+### Budget consumption
+
+- Configure/compile: **7/8**, conservatively counting every configure separately, including the failed first configure (four configures and three compiles).
+- Unit suites: **2/4**.
+- Integration suites: **2/2**; the two fixed tests are within each suite, not separate retry campaigns.
+- Canonical scenario: **1/1**.
+- NETWORK_FAULT_RUNS: **0**.
+- LOAD_RUNS: **0**.
+- No unrecorded build/test retries, microbenchmarks or parameter sweeps.
+
+## Actual canonical result
+
+The complete TCP input/ACK/output trace is `artifacts/G01-canonical.json` (ignored generated evidence).
+The committed `G01-canonical-summary.json` is selected directly from that output, not a hand-built expected result.
+
+| Role | Slot | Position | Direction | Score | Last successful TAG tick |
+|---|---:|---|---|---:|---:|
+| alpha | 0 | (50000, 50000) | STOP | 1 | 200 |
+| bravo | 1 | (50000, 50000) | STOP | 0 | -20 |
+
+Executed ticks: 1200; final execution tick: 1199; manual time: 60000ms.
+Room and **both actual TCP clients** observed `LOBBY → RUNNING → FINISHED → CLOSED`.
+Server/client final views matched. Generated IDs were resolved from WELCOME/ROOM_JOINED, never hard-coded by the scenario.
+Every intended INPUT_ACK was observed at its specified tick boundary before manual clock advancement.
+
+Canonical metrics: two live connections at high water; mailbox high water 1/512; pending inputs per player high water 1/64; outbound control high water 3/64; maximum read 250/16388 bytes; 11 received and 19 sent messages; no error events.
+The small fixture did not force a kernel partial write (`partial_writes=0`); an owned-buffer unit test verifies suffix retention across moves, and the production write loop preserves EAGAIN/partial offsets. No slow-consumer test was claimed early.
+
+## Cleanup and ownership evidence
+
+- Six real descriptors (listener, kqueue, two accepted sockets, two clients) were checked using `fcntl(F_GETFD)` after close: all EBADF.
+- Post-stop server connections, descriptors, mailbox entries, pending input, outbound messages, disconnect notifications, client descriptors and tracked descriptor delta: all 0.
+- Server worker threads and timers: 0 allocated / 0 remaining. The unit-only foreign thread was joined and its mutation was rejected before any Room state access.
+- Standalone process case: real HELLO/WELCOME, one SIGTERM, normal exit 0 and reaped within 5s, no retained socket, listener port successfully rebound.
+- All real network tests used requested loopback permission. No external service or Internet network test was used.
+
+## STATE_HASHES
+
+Inactive until G07. The scenario-input SHA above is an input-file identity, not a prematurely implemented state hash or replay feature.
+
+## RESOURCE_BOUNDS and remaining scope
+
+Bounds and overflow actions are documented in TRACK.md and exercised where relevant by the named unit/integration tests.
+The G01 complete-frame read assumption remains deliberate: no incremental parser, strict malformed matrix, sequence/tick window, replay, periodic full/delta replication, UDP, reconnect or many-room work was added.
+
+UNRESOLVED: no known G01 acceptance failure. Independent main verification and progress-tag creation remain the orchestrator's responsibility. Later Thread constraints remain intentionally unimplemented.
diff --git a/server.example.json b/server.example.json
new file mode 100644
index 0000000..0a75404
--- /dev/null
+++ b/server.example.json
@@ -0,0 +1 @@
+{"listen_port":0,"clock":"manual"}
diff --git a/src/game.cpp b/src/game.cpp
new file mode 100644
index 0000000..7bf6772
--- /dev/null
+++ b/src/game.cpp
@@ -0,0 +1,159 @@
+#include "game.hpp"
+#include <algorithm>
+#include <array>
+#include <set>
+#include <stdexcept>
+#include <utility>
+
+namespace arena {
+std::string direction_name(Direction direction) {
+  switch (direction) {
+    case Direction::stop: return "STOP";
+    case Direction::north: return "NORTH";
+    case Direction::east: return "EAST";
+    case Direction::south: return "SOUTH";
+    case Direction::west: return "WEST";
+  }
+  throw std::logic_error("invalid internal direction");
+}
+std::optional<Direction> parse_direction(const std::string& value) {
+  if (value == "STOP") return Direction::stop;
+  if (value == "NORTH") return Direction::north;
+  if (value == "EAST") return Direction::east;
+  if (value == "SOUTH") return Direction::south;
+  if (value == "WEST") return Direction::west;
+  return std::nullopt;
+}
+Json message(const std::string& type) { return Json{{"v", 1}, {"type", type}}; }
+Json error_message(const std::string& code, const std::string& description) {
+  Json result = message("ERROR");
+  result["code"] = code;
+  result["message"] = description.substr(0, 160);
+  return result;
+}
+Room::Room() : owner_(std::this_thread::get_id()) {}
+void Room::assert_owner() const {
+  if (std::this_thread::get_id() != owner_) throw std::logic_error("Room mutation outside its owner");
+}
+void Room::transition(std::string status) {
+  status_ = std::move(status);
+  lifecycle_.push_back(status_);
+}
+void Room::create(std::string id) {
+  assert_owner();
+  if (status_ != "ABSENT") throw std::logic_error("single room already exists");
+  id_ = std::move(id);
+  transition("LOBBY");
+}
+Player& Room::join(std::string player_id, std::string session_id, std::uint64_t connection_id) {
+  assert_owner();
+  if (status_ != "LOBBY" || next_slot_ >= static_cast<int>(max_players)) throw std::logic_error("room not joinable");
+  constexpr std::array<std::array<int, 2>, 8> spawns{{
+      {10000, 10000}, {90000, 90000}, {10000, 90000}, {90000, 10000},
+      {50000, 10000}, {50000, 90000}, {10000, 50000}, {90000, 50000}}};
+  const int slot = next_slot_++;
+  Player player{std::move(player_id), std::move(session_id), connection_id, slot,
+                spawns[static_cast<std::size_t>(slot)][0], spawns[static_cast<std::size_t>(slot)][1],
+                Direction::stop, 0, -20, true, {}};
+  const auto key = player.id;
+  auto [found, inserted] = players_.emplace(key, std::move(player));
+  if (!inserted) throw std::logic_error("server generated duplicate player id");
+  const auto ready = std::count_if(players_.begin(), players_.end(), [](const auto& pair) { return pair.second.connected; });
+  if (ready >= 2) transition("RUNNING");
+  return found->second;
+}
+std::optional<std::string> Room::input(const std::string& player_id, Intent intent) {
+  assert_owner();
+  if (status_ != "RUNNING") return "ROOM_NOT_RUNNING";
+  auto found = players_.find(player_id);
+  if (found == players_.end() || !found->second.connected) return "PLAYER_MISMATCH";
+  auto& queue = found->second.pending;
+  if (queue.size() == max_pending_inputs) return "INPUT_QUEUE_FULL";
+  queue.push_back(std::move(intent));
+  input_high_water_ = std::max(input_high_water_, queue.size());
+  return std::nullopt;
+}
+std::vector<ActionFailure> Room::tick() {
+  assert_owner();
+  std::vector<ActionFailure> failures;
+  if (status_ != "RUNNING") return failures;
+  std::map<std::string, std::string> tags;
+  for (auto& [id, player] : players_) {
+    if (!player.connected) continue;
+    if (!player.pending.empty()) {
+      const Intent& intent = player.pending.back();
+      player.direction = intent.direction;
+      if (intent.tag_target) tags.emplace(id, *intent.tag_target);
+      player.pending.clear();
+    }
+    switch (player.direction) {
+      case Direction::stop: break;
+      case Direction::north: player.y += 400; break;
+      case Direction::east: player.x += 400; break;
+      case Direction::south: player.y -= 400; break;
+      case Direction::west: player.x -= 400; break;
+    }
+    player.x = std::clamp(player.x, 0, 100000);
+    player.y = std::clamp(player.y, 0, 100000);
+  }
+  std::set<std::string> tagged;
+  for (const auto& [actor_id, target_id] : tags) {
+    Player& actor = players_.at(actor_id);
+    auto target = players_.find(target_id);
+    bool valid = target != players_.end() && target_id != actor_id && target->second.connected &&
+                 executed_ticks_ - actor.last_tag_tick >= 20 && !tagged.contains(target_id);
+    if (valid) {
+      const std::int64_t dx = actor.x - target->second.x;
+      const std::int64_t dy = actor.y - target->second.y;
+      valid = dx * dx + dy * dy <= 2500LL * 2500LL;
+    }
+    if (valid) {
+      ++actor.score;
+      actor.last_tag_tick = executed_ticks_;
+      tagged.insert(target_id);
+    } else {
+      failures.push_back({actor.connection_id, actor_id});
+    }
+  }
+  ++executed_ticks_;
+  if (executed_ticks_ == session_ticks) transition("FINISHED");
+  return failures;
+}
+void Room::leave(std::uint64_t connection_id) {
+  assert_owner();
+  for (auto& [id, player] : players_) {
+    (void)id;
+    if (player.connection_id == connection_id) {
+      player.connected = false;
+      player.direction = Direction::stop;
+      player.pending.clear();
+    }
+  }
+}
+void Room::close() {
+  assert_owner();
+  if (status_ == "ABSENT" || status_ == "CLOSED") return;
+  for (auto& [id, player] : players_) {
+    (void)id;
+    player.connected = false;
+    player.direction = Direction::stop;
+    player.pending.clear();
+  }
+  transition("CLOSED");
+}
+std::size_t Room::pending_count() const {
+  std::size_t count = 0;
+  for (const auto& [id, player] : players_) { (void)id; count += player.pending.size(); }
+  return count;
+}
+Json Room::view() const {
+  Json result{{"room_id", id_}, {"status", status_}, {"tick", executed_ticks_ - 1},
+              {"executed_ticks", executed_ticks_}, {"players", Json::array()}};
+  for (const auto& [id, player] : players_) {
+    result["players"].push_back(Json{{"player_id", id}, {"slot", player.slot}, {"x", player.x}, {"y", player.y},
+      {"direction", direction_name(player.direction)}, {"score", player.score},
+      {"connectivity", player.connected ? "CONNECTED" : "LEFT"}, {"last_tag_tick", player.last_tag_tick}});
+  }
+  return result;
+}
+}
diff --git a/src/game.hpp b/src/game.hpp
new file mode 100644
index 0000000..2d2e3c5
--- /dev/null
+++ b/src/game.hpp
@@ -0,0 +1,86 @@
+#pragma once
+#include <cstdint>
+#include <deque>
+#include <map>
+#include <optional>
+#include <string>
+#include <thread>
+#include <vector>
+#include <nlohmann/json.hpp>
+
+namespace arena {
+using Json = nlohmann::json;
+inline constexpr std::size_t max_frame_bytes = 16'384;
+inline constexpr std::size_t max_connections = 512;
+inline constexpr std::size_t max_players = 8;
+inline constexpr std::size_t max_pending_inputs = 64;
+inline constexpr std::size_t max_control_messages = 64;
+inline constexpr std::size_t max_mailbox_messages = max_players * max_pending_inputs;
+inline constexpr int session_ticks = 1'200;
+inline constexpr int tick_duration_ms = 50;
+
+enum class Direction { stop, north, east, south, west };
+std::string direction_name(Direction direction);
+std::optional<Direction> parse_direction(const std::string& value);
+Json message(const std::string& type);
+Json error_message(const std::string& code, const std::string& description);
+
+struct Intent {
+  Direction direction = Direction::stop;
+  std::optional<std::string> tag_target;
+};
+struct Player {
+  std::string id;
+  std::string session_id;
+  std::uint64_t connection_id = 0;
+  int slot = 0;
+  int x = 0;
+  int y = 0;
+  Direction direction = Direction::stop;
+  int score = 0;
+  int last_tag_tick = -20;
+  bool connected = true;
+  std::deque<Intent> pending;
+};
+struct ActionFailure {
+  std::uint64_t connection_id;
+  std::string player_id;
+};
+
+// This object is mutated only in the owner phase, never from a kqueue callback.
+class Room {
+ public:
+  Room();
+  void create(std::string id);
+  Player& join(std::string player_id, std::string session_id, std::uint64_t connection_id);
+  std::optional<std::string> input(const std::string& player_id, Intent intent);
+  std::vector<ActionFailure> tick();
+  void leave(std::uint64_t connection_id);
+  void close();
+  Json view() const;
+  const std::map<std::string, Player>& players() const { return players_; }
+  const std::string& id() const { return id_; }
+  const std::string& status() const { return status_; }
+  const std::vector<std::string>& lifecycle() const { return lifecycle_; }
+  int executed_ticks() const { return executed_ticks_; }
+  std::size_t pending_count() const;
+  std::size_t input_high_water() const { return input_high_water_; }
+ private:
+  void assert_owner() const;
+  void transition(std::string status);
+  std::thread::id owner_;
+  std::string id_;
+  std::string status_ = "ABSENT";
+  std::vector<std::string> lifecycle_;
+  std::map<std::string, Player> players_;
+  int next_slot_ = 0;
+  int executed_ticks_ = 0;
+  std::size_t input_high_water_ = 0;
+};
+
+// G01 advances one explicit 50ms step. Accumulator/catch-up first activate in G04.
+struct ManualClock {
+  std::int64_t now_ms = 0;
+  void advance_one() { now_ms += tick_duration_ms; }
+};
+}
diff --git a/src/main.cpp b/src/main.cpp
new file mode 100644
index 0000000..fbc2c8b
--- /dev/null
+++ b/src/main.cpp
@@ -0,0 +1,75 @@
+#include "scenario.hpp"
+#include <array>
+#include <cerrno>
+#include <charconv>
+#include <csignal>
+#include <iostream>
+#include <poll.h>
+#include <stdexcept>
+#include <unistd.h>
+
+namespace {
+volatile std::sig_atomic_t stop_requested = 0;
+void request_stop(int) { stop_requested = 1; }
+int serve(const arena::Json& config) {
+  if (config.value("clock", std::string("manual")) != "manual") throw std::runtime_error("G01 server supports manual clock only");
+  const auto port = config.value("listen_port", 0);
+  if (port < 0 || port > 65535) throw std::runtime_error("listen_port outside range");
+  arena::ManualClock clock;
+  arena::Server server(clock, static_cast<std::uint16_t>(port));
+  std::signal(SIGTERM, request_stop);
+  std::signal(SIGINT, request_stop);
+  std::cout << arena::Json{{"status", "READY"}, {"port", server.port()}, {"clock", "manual"}}.dump() << std::endl;
+  std::string operator_input;
+  bool stopped = false;
+  while (!stopped && !stop_requested) {
+    server.pump(10);
+    if (stop_requested) break;
+    pollfd input{STDIN_FILENO, POLLIN, 0};
+    if (::poll(&input, 1, 0) < 0) {
+      if (errno == EINTR) continue;
+      throw std::runtime_error("stdin readiness failed");
+    }
+    if (!(input.revents & (POLLIN | POLLHUP))) continue;
+    std::array<char, 1024> bytes{};
+    const auto count = ::read(STDIN_FILENO, bytes.data(), bytes.size());
+    if (count < 0) throw std::runtime_error("operator read failed");
+    if (count == 0) { stopped = true; continue; }
+    operator_input.append(bytes.data(), static_cast<std::size_t>(count));
+    if (operator_input.size() > 4096) throw std::runtime_error("operator command buffer exceeded 4096 bytes");
+    for (;;) {
+      const auto end = operator_input.find('\n');
+      if (end == std::string::npos) break;
+      const auto command = operator_input.substr(0, end);
+      operator_input.erase(0, end + 1);
+      if (command == "stop") { stopped = true; break; }
+      if (command == "state") { std::cout << server.room().view().dump() << std::endl; continue; }
+      if (command == "tick") { server.advance_one_tick(); continue; }
+      throw std::runtime_error("manual operator commands are tick, state, stop");
+    }
+  }
+  server.shutdown();
+  std::cout << arena::Json{{"status", "STOPPED"}, {"cleanup", server.cleanup()}}.dump() << std::endl;
+  return 0;
+}
+}
+int main(int argc, char** argv) {
+  try {
+    if (argc >= 2 && std::string(argv[1]) == "replay-verify") {
+      std::cerr << "{\"error\":\"NOT_ACTIVATED\",\"thread\":\"G07\"}\n"; return 2;
+    }
+    if (argc == 4 && std::string(argv[1]) == "scenario-run") {
+      const auto evidence = arena::run_scenario(arena::read_json_file(argv[2]));
+      arena::write_json_file(argv[3], evidence);
+      std::cout << arena::Json{{"result", evidence.at("result")}, {"scenario_id", evidence.at("scenario_id")},
+        {"executed_ticks", evidence.at("executed_ticks")}, {"evidence", argv[3]}, {"cleanup", evidence.at("cleanup")}}.dump() << '\n';
+      return 0;
+    }
+    if (argc == 3 && std::string(argv[1]) == "server") return serve(arena::read_json_file(argv[2], arena::max_frame_bytes));
+    std::cerr << "usage: arena scenario-run FILE OUT | server CONFIG | replay-verify FILE OUT\n";
+    return 2;
+  } catch (const std::exception& error) {
+    std::cerr << arena::Json{{"result", "FAIL"}, {"message", std::string(error.what()).substr(0, 256)}}.dump() << '\n';
+    return 1;
+  }
+}
diff --git a/src/scenario.cpp b/src/scenario.cpp
new file mode 100644
index 0000000..a074a8e
--- /dev/null
+++ b/src/scenario.cpp
@@ -0,0 +1,178 @@
+#include "scenario.hpp"
+#include <fstream>
+#include <memory>
+#include <stdexcept>
+
+namespace arena {
+namespace {
+void require(bool value, const std::string& text) { if (!value) throw std::runtime_error(text); }
+struct Participant {
+  std::unique_ptr<TcpClient> tcp;
+  std::string session_id;
+  std::string player_id;
+  int slot = -1;
+};
+Json for_player(const std::string& type, const Participant& participant, const std::string& room_id) {
+  Json value = message(type);
+  if (type != "HELLO") value["session_id"] = participant.session_id;
+  if (type == "JOIN_ROOM" || type == "INPUT") value["room_id"] = room_id;
+  if (type == "INPUT") value["player_id"] = participant.player_id;
+  return value;
+}
+std::vector<std::string> observed_lifecycle(const TcpClient& client) {
+  std::vector<std::string> result;
+  for (const auto& observation : client.observations()) {
+    if (!observation.contains("status")) continue;
+    auto status = observation.at("status").get<std::string>();
+    if (result.empty() || result.back() != status) result.push_back(std::move(status));
+  }
+  return result;
+}
+}
+Json read_json_file(const std::filesystem::path& path, std::size_t limit) {
+  const auto size = std::filesystem::file_size(path);
+  if (size > limit) throw std::runtime_error("JSON file exceeds bounded input size");
+  std::ifstream file(path, std::ios::binary);
+  if (!file) throw std::runtime_error("cannot open JSON input");
+  Json value; file >> value;
+  return value;
+}
+void write_json_file(const std::filesystem::path& path, const Json& value) {
+  if (path.has_parent_path()) std::filesystem::create_directories(path.parent_path());
+  const auto text = value.dump(2);
+  if (text.size() > 4'194'304) throw std::runtime_error("evidence output exceeds fixed bound");
+  std::ofstream file(path, std::ios::binary);
+  if (!file) throw std::runtime_error("cannot open evidence output");
+  file << text << '\n';
+  if (!file) throw std::runtime_error("evidence write failed");
+}
+Json run_scenario(const Json& scenario) {
+  require(scenario.at("contract_version") == 1 && scenario.at("thread") == "G01", "only G01 contract v1 is active");
+  require(scenario.at("clock").at("kind") == "manual" && scenario.at("clock").at("tick_duration_ms") == tick_duration_ms,
+          "G01 runner requires the fixed 50ms manual clock");
+  const int ticks = scenario.at("ticks").get<int>();
+  require(ticks > 0 && ticks <= session_ticks, "scenario tick count outside baseline bound");
+  require(scenario.at("clients").size() >= 2 && scenario.at("clients").size() <= max_players, "client count outside bound");
+  require(scenario.at("setup").size() <= 32 && scenario.at("inputs").size() <= 512, "scenario command bound exceeded");
+  require(scenario.at("shutdown") == "graceful-after-finished" || scenario.at("shutdown") == "graceful", "unknown shutdown policy");
+  const int descriptors_before = Fd::live();
+  ManualClock clock;
+  Server server(clock);
+  std::map<std::string, Participant> clients;
+  std::string room_id;
+  Json evidence{{"scenario_id", scenario.at("scenario_id")}, {"thread", "G01"}, {"contract_version", 1},
+                {"seed", scenario.at("seed")}, {"transport", "real-loopback-TCP/kqueue"}, {"clock", scenario.at("clock")},
+                {"accepted_inputs", Json::array()}, {"setup_responses", Json::array()}};
+  for (const auto& role_value : scenario.at("clients")) {
+    const auto role = role_value.get<std::string>();
+    require(role.size() <= 64 && !clients.contains(role), "client role must be unique and bounded");
+    clients.emplace(role, Participant{std::make_unique<TcpClient>(server.port()), {}, {}, -1});
+  }
+  for (const auto& step : scenario.at("setup")) {
+    const auto role = step.at("client").get<std::string>();
+    auto& participant = clients.at(role);
+    const auto type = step.at("type").get<std::string>();
+    require(type == "HELLO" || type == "CREATE_ROOM" || type == "JOIN_ROOM", "unsupported G01 setup operation");
+    participant.tcp->send(for_player(type, participant, room_id));
+    const std::string response_type = type == "HELLO" ? "WELCOME" : type == "CREATE_ROOM" ? "ROOM_CREATED" : "ROOM_JOINED";
+    const auto response = participant.tcp->receive_type(server, response_type);
+    evidence["setup_responses"].push_back(Json{{"client", role}, {"response", response}});
+    if (type == "HELLO") participant.session_id = response.at("session_id").get<std::string>();
+    if (type == "CREATE_ROOM") room_id = response.at("room_id").get<std::string>();
+    if (type == "JOIN_ROOM") {
+      participant.player_id = response.at("player_id").get<std::string>();
+      participant.slot = response.at("slot").get<int>();
+    }
+  }
+  require(server.room().status() == "RUNNING", "setup did not start the real server room");
+  std::size_t consumed_inputs = 0;
+  int previous_tick = -1;
+  for (const auto& input : scenario.at("inputs")) {
+    const int before = input.at("before_tick").get<int>();
+    require(before >= previous_tick && before >= 0 && before < ticks, "inputs must be ordered within the fixed timeline");
+    previous_tick = before;
+  }
+  for (int tick = 0; tick < ticks; ++tick) {
+    while (consumed_inputs < scenario.at("inputs").size() && scenario.at("inputs")[consumed_inputs].at("before_tick") == tick) {
+      const Json& input = scenario.at("inputs")[consumed_inputs++];
+      const auto role = input.at("client").get<std::string>();
+      auto& participant = clients.at(role);
+      Json request = for_player("INPUT", participant, room_id);
+      request["direction"] = input.at("direction");
+      request["tag_target_player_id"] = nullptr;
+      if (!input.at("tag_target").is_null()) {
+        const auto target = input.at("tag_target").get<std::string>();
+        request["tag_target_player_id"] = clients.at(target).player_id;
+      }
+      // Optional untrusted fields enable an authority regression; the real
+      // server ignores them. The canonical file has neither of these fields.
+      for (const auto* field : {"position", "score"}) if (input.contains(field)) request[field] = input.at(field);
+      participant.tcp->send(request);
+      const auto acknowledgement = participant.tcp->receive_type(server, "INPUT_ACK");
+      require(acknowledgement.at("accepted") == true && acknowledgement.at("tick") == tick,
+              "input acknowledgement did not establish the intended tick boundary");
+      evidence["accepted_inputs"].push_back(Json{{"client", role}, {"before_tick", tick},
+        {"direction", input.at("direction")}, {"tag_target", input.at("tag_target")}, {"ack", acknowledgement}});
+    }
+    // Every intended input has crossed TCP and received an owner-phase ACK.
+    server.advance_one_tick();
+    require(server.room().executed_ticks() == tick + 1, "manual tick did not execute exactly once");
+  }
+  require(consumed_inputs == scenario.at("inputs").size(), "scenario inputs were not all consumed");
+  const Json final = server.room().view();
+  evidence["server_final"] = final;
+  evidence["executed_ticks"] = server.room().executed_ticks();
+  evidence["last_tick"] = final.at("tick");
+  evidence["manual_clock_ms"] = clock.now_ms;
+  evidence["players"] = Json::object();
+  evidence["client_finished"] = Json::object();
+  for (auto& [role, participant] : clients) {
+    for (const auto& player : final.at("players")) {
+      if (player.at("player_id") == participant.player_id) evidence["players"][role] = player;
+    }
+    require(evidence["players"].contains(role), "server final is missing a joined role");
+    if (server.room().status() == "FINISHED") {
+      auto client_final = participant.tcp->receive_type(server, "ROOM_FINISHED");
+      evidence["client_finished"][role] = client_final;
+      client_final.erase("v"); client_final.erase("type");
+      require(client_final == final, "TCP client final differs from authoritative room view");
+    }
+  }
+  if (scenario.at("shutdown") == "graceful-after-finished") require(server.room().status() == "FINISHED", "shutdown preceded FINISHED");
+  auto closed_fds = server.owned_descriptors();
+  server.shutdown();
+  evidence["client_lifecycle"] = Json::object();
+  evidence["client_observations"] = Json::object();
+  for (auto& [role, participant] : clients) {
+    bool closed_seen = false;
+    for (int pending = 0; pending < 64; ++pending) {
+      const auto state = participant.tcp->receive_type(server, "SNAPSHOT");
+      if (state.at("status") == "CLOSED") { closed_seen = true; break; }
+    }
+    require(closed_seen, "client did not observe CLOSED before EOF");
+    closed_fds.push_back(participant.tcp->descriptor());
+    participant.tcp->close();
+    evidence["client_lifecycle"][role] = observed_lifecycle(*participant.tcp);
+    evidence["client_observations"][role] = participant.tcp->observations();
+  }
+  bool all_closed = true;
+  for (const int fd : closed_fds) all_closed = all_closed && descriptor_closed(fd);
+  evidence["cleanup"] = server.cleanup();
+  evidence["cleanup"]["descriptor_checks"] = closed_fds.size();
+  evidence["cleanup"]["all_descriptors_closed"] = all_closed;
+  evidence["cleanup"]["tracked_descriptor_delta"] = Fd::live() - descriptors_before;
+  evidence["cleanup"]["client_descriptors"] = 0;
+  require(all_closed && Fd::live() == descriptors_before, "file descriptor leak after graceful shutdown");
+  const auto cleanup = server.cleanup();
+  for (const auto& [key, count] : cleanup.items()) { (void)key; require(count == 0, "server retained an owned resource"); }
+  evidence["lifecycle"] = server.room().lifecycle();
+  evidence["metrics"] = server.metrics();
+  require(server.metrics().at("input_per_player_high_water").get<std::size_t>() <= max_pending_inputs, "input bound violated");
+  require(server.metrics().at("mailbox_high_water").get<std::size_t>() <= max_mailbox_messages, "mailbox bound violated");
+  require(server.metrics().at("outbound_control_high_water").get<std::size_t>() <= max_control_messages, "outbound bound violated");
+  require(server.metrics().at("max_read_bytes").get<std::size_t>() <= max_frame_bytes + 4, "read buffer bound violated");
+  evidence["state_hashes"] = "INACTIVE_UNTIL_G07";
+  evidence["result"] = "PASS";
+  return evidence;
+}
+}
diff --git a/src/scenario.hpp b/src/scenario.hpp
new file mode 100644
index 0000000..21314f8
--- /dev/null
+++ b/src/scenario.hpp
@@ -0,0 +1,8 @@
+#pragma once
+#include "transport.hpp"
+#include <filesystem>
+namespace arena {
+Json read_json_file(const std::filesystem::path& path, std::size_t limit = 1'048'576);
+void write_json_file(const std::filesystem::path& path, const Json& value);
+Json run_scenario(const Json& scenario);
+}
diff --git a/src/transport.cpp b/src/transport.cpp
new file mode 100644
index 0000000..a71f18c
--- /dev/null
+++ b/src/transport.cpp
@@ -0,0 +1,424 @@
+#include "transport.hpp"
+#include <algorithm>
+#include <array>
+#include <cerrno>
+#include <chrono>
+#include <cstring>
+#include <fcntl.h>
+#include <iomanip>
+#include <netinet/in.h>
+#include <netinet/tcp.h>
+#include <poll.h>
+#include <random>
+#include <sstream>
+#include <stdexcept>
+#include <sys/event.h>
+#include <sys/socket.h>
+#include <unistd.h>
+#include <utility>
+
+namespace arena {
+namespace {
+[[noreturn]] void system_failure(const char* operation) {
+  throw std::runtime_error(std::string(operation) + ": " + std::strerror(errno));
+}
+void nonblocking(int fd) {
+  const int flags = ::fcntl(fd, F_GETFL, 0);
+  if (flags == -1 || ::fcntl(fd, F_SETFL, flags | O_NONBLOCK) == -1) system_failure("fcntl nonblocking");
+  if (::fcntl(fd, F_SETFD, FD_CLOEXEC) == -1) system_failure("fcntl close-on-exec");
+}
+void socket_options(int fd) {
+  const int yes = 1;
+  if (::setsockopt(fd, SOL_SOCKET, SO_NOSIGPIPE, &yes, sizeof(yes)) == -1) system_failure("SO_NOSIGPIPE");
+  if (::setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &yes, sizeof(yes)) == -1) system_failure("TCP_NODELAY");
+}
+std::uint32_t payload_size(const std::uint8_t* data) {
+  return (static_cast<std::uint32_t>(data[0]) << 24U) |
+         (static_cast<std::uint32_t>(data[1]) << 16U) |
+         (static_cast<std::uint32_t>(data[2]) << 8U) | static_cast<std::uint32_t>(data[3]);
+}
+bool transient_io() { return errno == EAGAIN || errno == EWOULDBLOCK || errno == EINTR; }
+}
+std::atomic<int> Fd::live_{0};
+Fd::Fd(int value) : value_(value) { if (value_ >= 0) ++live_; }
+Fd::~Fd() { reset(); }
+Fd::Fd(Fd&& other) noexcept : value_(std::exchange(other.value_, -1)) {}
+Fd& Fd::operator=(Fd&& other) noexcept {
+  if (this != &other) { reset(); value_ = std::exchange(other.value_, -1); }
+  return *this;
+}
+void Fd::reset() {
+  if (value_ >= 0) { ::close(value_); value_ = -1; --live_; }
+}
+bool descriptor_closed(int fd) { return ::fcntl(fd, F_GETFD) == -1 && errno == EBADF; }
+std::vector<std::uint8_t> encode_frame(const Json& value) {
+  const std::string payload = value.dump();
+  if (payload.empty() || payload.size() > max_frame_bytes) throw std::length_error("frame size exceeds contract");
+  const auto size = static_cast<std::uint32_t>(payload.size());
+  std::vector<std::uint8_t> bytes{static_cast<std::uint8_t>(size >> 24U), static_cast<std::uint8_t>(size >> 16U),
+                                 static_cast<std::uint8_t>(size >> 8U), static_cast<std::uint8_t>(size)};
+  bytes.insert(bytes.end(), payload.begin(), payload.end());
+  return bytes;
+}
+Json decode_complete_frame(std::span<const std::uint8_t> bytes) {
+  if (bytes.size() < 4) throw std::length_error("G01 requires a complete header in one read");
+  const std::uint32_t size = payload_size(bytes.data());
+  if (size == 0 || size > max_frame_bytes || bytes.size() != size + 4U)
+    throw std::length_error("G01 requires exactly one bounded complete frame in one read");
+  auto value = Json::parse(bytes.begin() + 4, bytes.end());
+  if (!value.is_object()) throw std::invalid_argument("JSON root must be an object");
+  return value;
+}
+void PendingWrite::consume(std::size_t count) {
+  if (count > remaining().size()) throw std::logic_error("write offset exceeds owned buffer");
+  offset += count;
+}
+Server::Server(ManualClock& clock, std::uint16_t port) : clock_(clock), reactor_(::kqueue()), listener_(::socket(AF_INET, SOCK_STREAM, 0)) {
+  if (reactor_.get() == -1 || listener_.get() == -1) system_failure("create kqueue/listener");
+  std::random_device random;
+  std::ostringstream nonce;
+  nonce << std::hex << std::setfill('0') << std::setw(8) << random() << std::setw(8) << random();
+  nonce_ = nonce.str();
+  const int yes = 1;
+  if (::setsockopt(listener_.get(), SOL_SOCKET, SO_REUSEADDR, &yes, sizeof(yes)) == -1) system_failure("SO_REUSEADDR");
+  nonblocking(listener_.get());
+  if (::fcntl(reactor_.get(), F_SETFD, FD_CLOEXEC) == -1) system_failure("reactor close-on-exec");
+  sockaddr_in address{};
+  address.sin_family = AF_INET;
+  address.sin_addr.s_addr = htonl(INADDR_LOOPBACK);
+  address.sin_port = htons(port);
+  if (::bind(listener_.get(), reinterpret_cast<sockaddr*>(&address), sizeof(address)) == -1) system_failure("bind loopback");
+  if (::listen(listener_.get(), 64) == -1) system_failure("listen");
+  socklen_t size = sizeof(address);
+  if (::getsockname(listener_.get(), reinterpret_cast<sockaddr*>(&address), &size) == -1) system_failure("getsockname");
+  port_ = ntohs(address.sin_port);
+  register_event(listener_.get(), EVFILT_READ, EV_ADD);
+}
+Server::~Server() {
+  // Destructors cannot report errors; explicit shutdown is required by callers
+  // and tested. RAII still closes every descriptor on an exceptional exit.
+  connections_.clear();
+}
+void Server::register_event(int fd, short filter, unsigned short flags, std::uint64_t connection_id) {
+  struct kevent event{};
+  EV_SET(&event, static_cast<uintptr_t>(fd), filter, flags, 0, 0,
+         reinterpret_cast<void*>(static_cast<uintptr_t>(connection_id)));
+  if (::kevent(reactor_.get(), &event, 1, nullptr, 0, nullptr) == -1) system_failure("kevent registration");
+}
+Server::Connection* Server::connection(std::uint64_t id) {
+  for (auto& [fd, conn] : connections_) { (void)fd; if (conn.id == id) return &conn; }
+  return nullptr;
+}
+std::string Server::new_id(const std::string& prefix, std::uint64_t number) const {
+  std::ostringstream out;
+  out << prefix << '-' << nonce_ << '-' << std::setw(10) << std::setfill('0') << number;
+  return out.str();
+}
+void Server::accept_ready() {
+  // A single ready listener cannot monopolize one I/O iteration.
+  for (int accepted = 0; accepted < 64; ++accepted) {
+    Fd fd(::accept(listener_.get(), nullptr, nullptr));
+    if (fd.get() == -1) { if (transient_io()) return; system_failure("accept"); }
+    if (connections_.size() == max_connections) { ++errors_["ADMISSION_REJECTED"]; continue; }
+    nonblocking(fd.get());
+    socket_options(fd.get());
+    const int raw = fd.get();
+    const auto id = next_connection_++;
+    connections_.emplace(raw, Connection{std::move(fd), id, {}, {}, {}, 0});
+    register_event(raw, EVFILT_READ, EV_ADD, id);
+    register_event(raw, EVFILT_WRITE, EV_ADD | EV_DISABLE, id);
+    connection_high_water_ = std::max(connection_high_water_, connections_.size());
+  }
+}
+void Server::disconnect(int fd, const std::string& reason) {
+  const auto found = connections_.find(fd);
+  if (found == connections_.end()) return;
+  disconnected_.insert(found->second.id);
+  if (!reason.empty()) ++errors_[reason];
+  // Closing a descriptor removes its kqueue registrations. The generation id
+  // in udata prevents old events from touching a subsequently reused fd.
+  connections_.erase(found);
+}
+void Server::read_ready(int fd) {
+  auto found = connections_.find(fd);
+  if (found == connections_.end()) return;
+  std::array<std::uint8_t, max_frame_bytes + 4> bytes{};
+  const auto received = ::recv(fd, bytes.data(), bytes.size(), 0);
+  if (received == 0) { disconnect(fd, ""); return; }
+  if (received < 0) { if (!transient_io()) disconnect(fd, "TRANSPORT_IO_ERROR"); return; }
+  const auto count = static_cast<std::size_t>(received);
+  max_read_bytes_ = std::max(max_read_bytes_, count);
+  try {
+    Json value = decode_complete_frame(std::span(bytes).first(count));
+    if (mailbox_.size() == max_mailbox_messages || found->second.pending_requests == max_pending_inputs) {
+      queue(found->second.id, error_message("INPUT_QUEUE_FULL", "bounded transport mailbox is full"));
+      return;
+    }
+    ++found->second.pending_requests;
+    mailbox_.push_back({found->second.id, std::move(value)});
+    mailbox_high_water_ = std::max(mailbox_high_water_, mailbox_.size());
+    ++received_messages_;
+  } catch (const std::length_error&) {
+    // G01 has no partial-frame storage. G02 will replace this assumption.
+    disconnect(fd, "FRAME_SIZE_INVALID");
+  } catch (const Json::exception&) {
+    queue(found->second.id, error_message("MESSAGE_INVALID", "invalid JSON message"));
+  } catch (const std::invalid_argument&) {
+    queue(found->second.id, error_message("MESSAGE_INVALID", "JSON root must be an object"));
+  }
+}
+void Server::write_ready(int fd) {
+  auto found = connections_.find(fd);
+  if (found == connections_.end()) return;
+  auto& writes = found->second.outbound;
+  // Bounded per-ready work, preserving every unsent suffix in owned storage.
+  for (int work = 0; work < 64 && !writes.empty(); ++work) {
+    auto& write = writes.front();
+    const auto bytes = write.remaining();
+    const auto sent = ::send(fd, bytes.data(), bytes.size(), 0);
+    if (sent < 0) { if (!transient_io()) disconnect(fd, "TRANSPORT_IO_ERROR"); return; }
+    if (sent == 0) return;
+    if (static_cast<std::size_t>(sent) < bytes.size()) ++partial_writes_;
+    write.consume(static_cast<std::size_t>(sent));
+    if (write.remaining().empty()) { writes.pop_front(); ++sent_messages_; }
+  }
+  if (writes.empty()) register_event(fd, EVFILT_WRITE, EV_DISABLE, found->second.id);
+}
+void Server::poll_io(int timeout_ms) {
+  if (reactor_.get() < 0) return;
+  std::array<struct kevent, 64> events{};
+  timespec timeout{timeout_ms / 1000, (timeout_ms % 1000) * 1000000L};
+  const int count = ::kevent(reactor_.get(), nullptr, 0, events.data(), static_cast<int>(events.size()), &timeout);
+  if (count < 0) { if (errno == EINTR) return; system_failure("kevent wait"); }
+  for (int i = 0; i < count; ++i) {
+    const auto& event = events[static_cast<std::size_t>(i)];
+    const int fd = static_cast<int>(event.ident);
+    if (fd == listener_.get()) { if (!stopping_) accept_ready(); continue; }
+    const auto found = connections_.find(fd);
+    if (found == connections_.end() || found->second.id != reinterpret_cast<uintptr_t>(event.udata)) continue;
+    if (event.flags & EV_ERROR) { disconnect(fd, "TRANSPORT_IO_ERROR"); continue; }
+    if (event.filter == EVFILT_READ) {
+      if (!stopping_) read_ready(fd);
+    } else if (event.filter == EVFILT_WRITE) {
+      write_ready(fd);
+    }
+  }
+}
+void Server::queue(std::uint64_t connection_id, Json value) {
+  auto* conn = connection(connection_id);
+  if (conn == nullptr) return;
+  if (conn->outbound.size() == max_control_messages) {
+    disconnect(conn->fd.get(), "CONTROL_BACKPRESSURE");
+    return;
+  }
+  conn->outbound.push_back(PendingWrite{encode_frame(value), 0});
+  outbound_high_water_ = std::max(outbound_high_water_, conn->outbound.size());
+  register_event(conn->fd.get(), EVFILT_WRITE, EV_ENABLE, conn->id);
+}
+void Server::broadcast(const Json& value) {
+  std::vector<std::uint64_t> ids;
+  ids.reserve(connections_.size());
+  for (const auto& [fd, conn] : connections_) { (void)fd; if (!conn.player_id.empty()) ids.push_back(conn.id); }
+  for (auto id : ids) queue(id, value);
+}
+void Server::handle(const Envelope& envelope) {
+  auto* conn = connection(envelope.connection_id);
+  if (conn == nullptr) return;
+  const Json& value = envelope.value;
+  const auto id = conn->id;
+  auto reject = [&](const std::string& code, const std::string& text) {
+    ++errors_[code]; queue(id, error_message(code, text));
+  };
+  try {
+    if (!value.contains("v") || !value.at("v").is_number_integer() || !value.contains("type") || !value.at("type").is_string()) {
+      reject("MESSAGE_INVALID", "v integer and type string required"); return;
+    }
+    if (value.at("v") != 1) { reject("PROTOCOL_VERSION_UNSUPPORTED", "only protocol v1 is supported"); return; }
+    const auto type = value.at("type").get<std::string>();
+    if (type == "HELLO") {
+      if (conn->session_id.empty()) conn->session_id = new_id("session", id);
+      Json reply = message("WELCOME"); reply["session_id"] = conn->session_id;
+      queue(id, std::move(reply)); return;
+    }
+    if (type != "CREATE_ROOM" && type != "JOIN_ROOM" && type != "LEAVE_ROOM" && type != "INPUT") {
+      reject("MESSAGE_TYPE_UNKNOWN", "unknown message type"); return;
+    }
+    if (conn->session_id.empty() || value.at("session_id").get<std::string>() != conn->session_id) {
+      reject("SESSION_INVALID", "HELLO session must match this connection"); return;
+    }
+    if (type == "CREATE_ROOM") {
+      if (room_.status() != "ABSENT") { reject("ROOM_NOT_JOINABLE", "G01 supports one room"); return; }
+      room_.create(new_id("room", 1));
+      Json reply = message("ROOM_CREATED"); reply["room_id"] = room_.id(); reply["status"] = room_.status();
+      queue(id, std::move(reply)); return;
+    }
+    if (value.at("room_id").get<std::string>() != room_.id() || room_.status() == "ABSENT") {
+      reject("ROOM_NOT_FOUND", "unknown room"); return;
+    }
+    if (type == "JOIN_ROOM") {
+      if (room_.status() != "LOBBY" || !conn->player_id.empty() || room_.players().size() == max_players) {
+        reject("ROOM_NOT_JOINABLE", "room is not joinable"); return;
+      }
+      Json lobby = room_.view(); lobby.update(message("SNAPSHOT"));
+      conn->player_id = new_id("player", next_player_++);
+      const auto& player = room_.join(conn->player_id, conn->session_id, id);
+      Json reply = message("ROOM_JOINED"); reply["room_id"] = room_.id(); reply["player_id"] = player.id;
+      reply["slot"] = player.slot; reply["status"] = room_.status();
+      queue(id, std::move(lobby));
+      queue(id, std::move(reply));
+      if (room_.status() == "RUNNING") {
+        Json state = room_.view(); state.update(message("SNAPSHOT")); broadcast(state);
+      }
+      return;
+    }
+    if (conn->player_id.empty() || value.at("player_id").get<std::string>() != conn->player_id) {
+      reject("PLAYER_MISMATCH", "player must belong to this connection"); return;
+    }
+    if (type == "LEAVE_ROOM") {
+      room_.leave(id);
+      Json state = room_.view(); state.update(message("SNAPSHOT")); queue(id, state); return;
+    }
+    const auto direction = parse_direction(value.at("direction").get<std::string>());
+    if (!direction) { reject("MESSAGE_INVALID", "direction is not a cardinal enum"); return; }
+    std::optional<std::string> target;
+    const auto& tag = value.at("tag_target_player_id");
+    if (!tag.is_null()) target = tag.get<std::string>();
+    if (target && target->size() > 64) { reject("MESSAGE_INVALID", "identifier too long"); return; }
+    if (const auto error = room_.input(conn->player_id, Intent{*direction, target})) {
+      reject(*error, "input was not accepted"); return;
+    }
+    Json reply = message("INPUT_ACK"); reply["player_id"] = conn->player_id; reply["accepted"] = true;
+    reply["tick"] = room_.executed_ticks(); queue(id, std::move(reply));
+  } catch (const Json::exception&) {
+    reject("MESSAGE_INVALID", "required field missing or wrong type");
+  }
+}
+void Server::drain_mailbox() {
+  // Room mutations happen after all ready I/O callbacks have completed.
+  for (const auto id : disconnected_) room_.leave(id);
+  disconnected_.clear();
+  const auto size = mailbox_.size();
+  for (std::size_t i = 0; i < size; ++i) {
+    Envelope envelope = std::move(mailbox_.front()); mailbox_.pop_front();
+    if (auto* conn = connection(envelope.connection_id); conn != nullptr) --conn->pending_requests;
+    handle(envelope);
+  }
+}
+void Server::pump(int timeout_ms) { poll_io(timeout_ms); drain_mailbox(); }
+void Server::advance_one_tick() {
+  if (stopping_) return;
+  drain_mailbox();
+  if (room_.status() != "RUNNING") return;
+  clock_.advance_one();
+  for (const auto& failure : room_.tick()) {
+    auto error = error_message("ACTION_REJECTED", "TAG conditions not satisfied");
+    error["player_id"] = failure.player_id; error["tick"] = room_.executed_ticks() - 1;
+    ++errors_["ACTION_REJECTED"]; queue(failure.connection_id, std::move(error));
+  }
+  if (room_.status() == "FINISHED") {
+    Json result = room_.view(); result.update(message("ROOM_FINISHED")); broadcast(result);
+  }
+}
+Json Server::metrics() const {
+  return Json{{"received_messages", received_messages_}, {"sent_messages", sent_messages_},
+    {"mailbox_high_water", mailbox_high_water_}, {"outbound_control_high_water", outbound_high_water_},
+    {"connection_high_water", connection_high_water_}, {"input_per_player_high_water", room_.input_high_water()},
+    {"max_read_bytes", max_read_bytes_}, {"partial_writes", partial_writes_}, {"errors", errors_}};
+}
+Json Server::cleanup() const {
+  std::size_t queued = 0;
+  for (const auto& [fd, conn] : connections_) { (void)fd; queued += conn.outbound.size(); }
+  return Json{{"server_connections", connections_.size()}, {"server_descriptors", owned_descriptors().size()},
+    {"mailbox_messages", mailbox_.size()}, {"pending_inputs", room_.pending_count()}, {"outbound_messages", queued},
+    {"worker_threads", 0}, {"timers", 0}, {"disconnect_notifications", disconnected_.size()}};
+}
+std::vector<int> Server::owned_descriptors() const {
+  std::vector<int> descriptors;
+  if (reactor_.get() >= 0) descriptors.push_back(reactor_.get());
+  if (listener_.get() >= 0) descriptors.push_back(listener_.get());
+  for (const auto& [fd, conn] : connections_) { (void)conn; descriptors.push_back(fd); }
+  return descriptors;
+}
+void Server::shutdown() {
+  if (stopping_) return;
+  stopping_ = true;
+  listener_.reset();
+  drain_mailbox();
+  room_.close();
+  if (room_.status() == "CLOSED") {
+    Json state = room_.view(); state.update(message("SNAPSHOT")); broadcast(state);
+  }
+  // Only transport flushing uses a wall deadline; no simulation runs here.
+  const auto deadline = std::chrono::steady_clock::now() + std::chrono::milliseconds(500);
+  for (;;) {
+    bool pending = false;
+    for (const auto& [fd, conn] : connections_) { (void)fd; pending = pending || !conn.outbound.empty(); }
+    if (!pending) break;
+    if (std::chrono::steady_clock::now() >= deadline) { ++errors_["SHUTDOWN_FLUSH_TIMEOUT"]; break; }
+    poll_io(1);
+  }
+  connections_.clear(); mailbox_.clear(); disconnected_.clear(); reactor_.reset();
+}
+TcpClient::TcpClient(std::uint16_t port) : fd_(::socket(AF_INET, SOCK_STREAM, 0)) {
+  if (fd_.get() == -1) system_failure("client socket");
+  socket_options(fd_.get());
+  // Connection deadline is a test ceiling, never an input to simulation time.
+  nonblocking(fd_.get());
+  sockaddr_in address{}; address.sin_family = AF_INET;
+  address.sin_addr.s_addr = htonl(INADDR_LOOPBACK); address.sin_port = htons(port);
+  if (::connect(fd_.get(), reinterpret_cast<sockaddr*>(&address), sizeof(address)) == -1) {
+    if (errno != EINPROGRESS) system_failure("client connect");
+    pollfd waiting{fd_.get(), POLLOUT, 0};
+    if (::poll(&waiting, 1, 2000) <= 0) throw std::runtime_error("client connect deadline exceeded");
+    int error = 0; socklen_t size = sizeof(error);
+    if (::getsockopt(fd_.get(), SOL_SOCKET, SO_ERROR, &error, &size) == -1) system_failure("client connect status");
+    if (error != 0) { errno = error; system_failure("client connect completion"); }
+  }
+}
+void TcpClient::send(const Json& value) {
+  PendingWrite write{encode_frame(value), 0};
+  const auto deadline = std::chrono::steady_clock::now() + std::chrono::seconds(2);
+  while (!write.remaining().empty()) {
+    const auto bytes = write.remaining();
+    const auto count = ::send(fd_.get(), bytes.data(), bytes.size(), 0);
+    if (count > 0) write.consume(static_cast<std::size_t>(count));
+    else if (count < 0 && !transient_io()) system_failure("client send");
+    if (std::chrono::steady_clock::now() >= deadline) throw std::runtime_error("client send deadline exceeded");
+  }
+}
+std::optional<Json> TcpClient::try_receive() {
+  std::array<std::uint8_t, max_frame_bytes + 4> bytes{};
+  auto count = ::recv(fd_.get(), bytes.data(), 4, MSG_PEEK);
+  if (count == 0) return std::nullopt;
+  if (count < 0) { if (transient_io()) return std::nullopt; system_failure("client peek header"); }
+  if (count < 4) return std::nullopt;
+  const auto size = payload_size(bytes.data());
+  if (size == 0 || size > max_frame_bytes) throw std::runtime_error("server sent invalid frame length");
+  const auto total = static_cast<std::size_t>(size) + 4;
+  count = ::recv(fd_.get(), bytes.data(), total, MSG_PEEK);
+  if (count < 0) { if (transient_io()) return std::nullopt; system_failure("client peek body"); }
+  if (static_cast<std::size_t>(count) < total) return std::nullopt;
+  count = ::recv(fd_.get(), bytes.data(), total, 0);
+  if (count != static_cast<ssize_t>(total)) throw std::runtime_error("client complete-frame read failed");
+  Json value = decode_complete_frame(std::span(bytes).first(total));
+  if (observations_.size() == 4096) throw std::runtime_error("client observation bound exceeded");
+  observations_.push_back(value);
+  return value;
+}
+Json TcpClient::receive(Server& server) {
+  const auto deadline = std::chrono::steady_clock::now() + std::chrono::seconds(2);
+  do {
+    server.pump();
+    if (auto reply = try_receive()) return *reply;
+  } while (std::chrono::steady_clock::now() < deadline);
+  throw std::runtime_error("TCP response deadline exceeded");
+}
+Json TcpClient::receive_type(Server& server, const std::string& type) {
+  for (int count = 0; count < 64; ++count) {
+    Json value = receive(server);
+    if (value.at("type") == type) return value;
+    if (value.at("type") == "ERROR") throw std::runtime_error("server error: " + value.dump());
+  }
+  throw std::runtime_error("expected response absent within control bound");
+}
+}
diff --git a/src/transport.hpp b/src/transport.hpp
new file mode 100644
index 0000000..90824ef
--- /dev/null
+++ b/src/transport.hpp
@@ -0,0 +1,109 @@
+#pragma once
+#include "game.hpp"
+#include <atomic>
+#include <cstddef>
+#include <span>
+#include <set>
+
+namespace arena {
+class Fd {
+ public:
+  explicit Fd(int value = -1);
+  ~Fd();
+  Fd(Fd&& other) noexcept;
+  Fd& operator=(Fd&& other) noexcept;
+  Fd(const Fd&) = delete;
+  Fd& operator=(const Fd&) = delete;
+  int get() const { return value_; }
+  void reset();
+  static int live() { return live_.load(); }
+ private:
+  int value_;
+  static std::atomic<int> live_;
+};
+
+std::vector<std::uint8_t> encode_frame(const Json& value);
+Json decode_complete_frame(std::span<const std::uint8_t> bytes);
+struct PendingWrite {
+  std::vector<std::uint8_t> bytes;
+  std::size_t offset = 0;
+  std::span<const std::uint8_t> remaining() const { return std::span(bytes).subspan(offset); }
+  void consume(std::size_t count);
+};
+
+class Server {
+ public:
+  explicit Server(ManualClock& clock, std::uint16_t port = 0);
+  ~Server();
+  Server(const Server&) = delete;
+  Server& operator=(const Server&) = delete;
+  std::uint16_t port() const { return port_; }
+  void poll_io(int timeout_ms = 0);
+  void drain_mailbox();
+  void pump(int timeout_ms = 0);
+  void advance_one_tick();
+  void shutdown();
+  const Room& room() const { return room_; }
+  Json metrics() const;
+  Json cleanup() const;
+  std::vector<int> owned_descriptors() const;
+ private:
+  struct Connection {
+    Fd fd;
+    std::uint64_t id;
+    std::string session_id;
+    std::string player_id;
+    std::deque<PendingWrite> outbound;
+    std::size_t pending_requests = 0;
+  };
+  struct Envelope { std::uint64_t connection_id; Json value; };
+  Connection* connection(std::uint64_t id);
+  void register_event(int fd, short filter, unsigned short flags, std::uint64_t connection_id = 0);
+  void accept_ready();
+  void read_ready(int fd);
+  void write_ready(int fd);
+  void disconnect(int fd, const std::string& reason);
+  void queue(std::uint64_t connection_id, Json value);
+  void broadcast(const Json& value);
+  void handle(const Envelope& envelope);
+  std::string new_id(const std::string& prefix, std::uint64_t number) const;
+  ManualClock& clock_;
+  Fd reactor_;
+  Fd listener_;
+  std::uint16_t port_ = 0;
+  std::map<int, Connection> connections_;
+  std::deque<Envelope> mailbox_;
+  std::set<std::uint64_t> disconnected_;
+  Room room_;
+  std::string nonce_;
+  std::uint64_t next_connection_ = 1;
+  std::uint64_t next_player_ = 1;
+  std::size_t mailbox_high_water_ = 0;
+  std::size_t outbound_high_water_ = 0;
+  std::size_t connection_high_water_ = 0;
+  std::size_t max_read_bytes_ = 0;
+  std::uint64_t received_messages_ = 0;
+  std::uint64_t sent_messages_ = 0;
+  std::uint64_t partial_writes_ = 0;
+  std::map<std::string, std::uint64_t> errors_;
+  bool stopping_ = false;
+};
+
+// Test/CLI client owns real TCP. Kernel-peeking waits for one bounded complete
+// response; the server's G01 read intentionally retains no partial-frame state.
+class TcpClient {
+ public:
+  explicit TcpClient(std::uint16_t port);
+  void send(const Json& value);
+  std::optional<Json> try_receive();
+  Json receive(Server& server);
+  Json receive_type(Server& server, const std::string& type);
+  void close() { fd_.reset(); }
+  int descriptor() const { return fd_.get(); }
+  const std::vector<Json>& observations() const { return observations_; }
+ private:
+  Fd fd_;
+  std::vector<Json> observations_;
+};
+bool descriptor_closed(int fd);
+}
diff --git a/tests/tests.cpp b/tests/tests.cpp
new file mode 100644
index 0000000..0b886d0
--- /dev/null
+++ b/tests/tests.cpp
@@ -0,0 +1,262 @@
+#include "scenario.hpp"
+#include <atomic>
+#include <cerrno>
+#include <chrono>
+#include <csignal>
+#include <filesystem>
+#include <functional>
+#include <iostream>
+#include <netinet/in.h>
+#include <poll.h>
+#include <spawn.h>
+#include <sstream>
+#include <stdexcept>
+#include <sys/socket.h>
+#include <sys/wait.h>
+#include <thread>
+#include <unistd.h>
+
+extern char** environ;
+namespace {
+using namespace arena;
+void check(bool condition, const std::string& text) { if (!condition) throw std::runtime_error(text); }
+void populate(Room& room) {
+  room.create("unit-room");
+  room.join("player-00", "session-00", 1);
+  room.join("player-01", "session-01", 2);
+}
+void lifecycle_and_duration() {
+  Room room;
+  room.create("unit-room");
+  check(room.status() == "LOBBY", "create enters lobby");
+  const auto& first = room.join("player-00", "session-00", 1);
+  check(first.slot == 0 && first.x == 10000 && first.y == 10000 && first.last_tag_tick == -20, "first spawn constants");
+  check(room.status() == "LOBBY", "one player does not start");
+  const auto& second = room.join("player-01", "session-01", 2);
+  check(second.slot == 1 && second.x == 90000 && second.y == 90000, "second spawn constants");
+  check(room.status() == "RUNNING", "two TCP-ready players start");
+  for (int tick = 0; tick < session_ticks; ++tick) {
+    room.tick();
+    check(room.executed_ticks() == tick + 1, "exactly one authoritative tick");
+    check(room.status() == (tick == session_ticks - 1 ? "FINISHED" : "RUNNING"), "finish only after 1200 executed ticks");
+  }
+  room.tick();
+  check(room.executed_ticks() == 1200 && room.view().at("tick") == 1199, "finished tick does not advance");
+  check(room.input("player-00", {}) == "ROOM_NOT_RUNNING", "finished input rejected");
+  room.close();
+  check(room.lifecycle() == std::vector<std::string>({"LOBBY", "RUNNING", "FINISHED", "CLOSED"}), "complete lifecycle");
+  check(room.pending_count() == 0, "close clears input");
+}
+void movement_is_integer_and_bounded() {
+  Room room; populate(room);
+  room.input("player-00", {Direction::east, std::nullopt});
+  room.input("player-01", {Direction::south, std::nullopt});
+  room.tick(); room.tick();
+  check(room.players().at("player-00").x == 10800, "direction persists at 400 integer units per tick");
+  check(room.players().at("player-01").y == 89200, "south moves negative y");
+  for (int tick = 0; tick < 300; ++tick) room.tick();
+  check(room.players().at("player-00").x == 100000 && room.players().at("player-01").y == 0, "arena clamps both bounds");
+  room.input("player-00", {Direction::stop, std::nullopt}); room.tick();
+  check(room.players().at("player-00").x == 100000, "STOP stops movement");
+}
+void tag_uses_wide_distance_and_one_shot_intent() {
+  Room room;
+  room.create("unit-room");
+  auto& actor = room.join("player-00", "session-00", 1);
+  auto& target = room.join("player-01", "session-01", 2);
+  // Unit-only setup of exact arithmetic boundaries. The production network
+  // never exposes position/score mutation to clients.
+  actor.x = 0; actor.y = 0; target.x = 100000; target.y = 100000;
+  room.input(actor.id, {Direction::stop, target.id});
+  const auto failures = room.tick();
+  check(failures.size() == 1 && actor.score == 0 && actor.last_tag_tick == -20, "20,000,000,000 squared distance rejected without overflow");
+  actor.x = 50000; actor.y = 50000; target.x = 52500; target.y = 50000;
+  room.input(actor.id, {Direction::stop, target.id});
+  check(room.tick().empty() && actor.score == 1 && target.score == 0, "inclusive TAG range awards actor only");
+  const int success_tick = actor.last_tag_tick;
+  for (int tick = 0; tick < 40; ++tick) room.tick();
+  check(actor.score == 1 && actor.last_tag_tick == success_tick, "TAG does not repeat after cooldown without another intent");
+}
+void input_capacity_is_explicit() {
+  Room room; populate(room);
+  for (std::size_t input = 0; input < max_pending_inputs; ++input)
+    check(!room.input("player-00", {Direction::east, std::nullopt}), "first 64 pending accepted");
+  check(room.input("player-00", {Direction::west, std::nullopt}) == "INPUT_QUEUE_FULL", "65th pending rejected explicitly");
+  check(room.input_high_water() == 64 && room.pending_count() == 64, "bounded pending metric");
+  room.tick();
+  check(room.pending_count() == 0 && room.players().at("player-00").x == 10400, "one movement per tick despite many intents");
+}
+void complete_frame_and_owned_write_buffer() {
+  const auto input = Json{{"v", 1}, {"type", "HELLO"}};
+  const auto original = encode_frame(input);
+  check(decode_complete_frame(original) == input, "complete frame roundtrip");
+  const std::size_t size = original.size() - 4;
+  check(original[0] == 0 && original[1] == 0 && original[2] == 0 && original[3] == size, "big endian length");
+  std::deque<PendingWrite> queue;
+  queue.push_back(PendingWrite{encode_frame(input), 0});
+  queue.front().consume(7);
+  auto moved = std::move(queue.front()); queue.pop_front();
+  check(std::vector<std::uint8_t>(moved.remaining().begin(), moved.remaining().end()) ==
+        std::vector<std::uint8_t>(original.begin() + 7, original.end()), "partial-write suffix remains owned after move");
+  moved.consume(moved.remaining().size());
+  check(moved.remaining().empty(), "write completes exactly at buffer end");
+  bool rejected = false;
+  try { encode_frame(Json{{"payload", std::string(max_frame_bytes, 'x')}}); } catch (const std::length_error&) { rejected = true; }
+  check(rejected, "serialization frame allocation bound");
+}
+void descriptor_ownership() {
+  int pipe_fds[2];
+  check(::pipe(pipe_fds) == 0, "pipe fixture creation");
+  const int before = Fd::live();
+  {
+    Fd read_end(pipe_fds[0]), write_end(pipe_fds[1]);
+    Fd moved(std::move(read_end));
+    check(read_end.get() == -1 && moved.get() == pipe_fds[0] && Fd::live() == before + 2, "fd move transfers sole ownership");
+  }
+  check(Fd::live() == before && descriptor_closed(pipe_fds[0]) && descriptor_closed(pipe_fds[1]), "RAII closes both actual descriptors");
+}
+void foreign_thread_mutation_is_rejected() {
+  Room room; populate(room);
+  std::atomic<bool> rejected{false};
+  std::thread foreign([&] {
+    try { room.input("player-00", {Direction::east, std::nullopt}); }
+    catch (const std::logic_error&) { rejected.store(true); }
+  });
+  foreign.join();
+  check(rejected.load() && room.pending_count() == 0, "foreign thread cannot mutate the single owner room");
+}
+void real_tcp_authority_and_cleanup() {
+  const auto scenario = Json::parse(R"({
+    "scenario_id":"G01-three-tick-authority-smoke","contract_version":1,"thread":"G01","seed":7050,
+    "clock":{"kind":"manual","tick_duration_ms":50},"ticks":3,"clients":["alpha","bravo"],
+    "setup":[{"client":"alpha","type":"HELLO"},{"client":"bravo","type":"HELLO"},
+      {"client":"alpha","type":"CREATE_ROOM"},{"client":"alpha","type":"JOIN_ROOM"},{"client":"bravo","type":"JOIN_ROOM"}],
+    "inputs":[{"before_tick":0,"client":"alpha","direction":"EAST","tag_target":null,"position":{"x":0,"y":0},"score":999},
+      {"before_tick":0,"client":"bravo","direction":"WEST","tag_target":null},
+      {"before_tick":1,"client":"alpha","direction":"STOP","tag_target":null},
+      {"before_tick":1,"client":"bravo","direction":"STOP","tag_target":null}],"shutdown":"graceful"
+  })");
+  const auto evidence = run_scenario(scenario);
+  check(evidence.at("players").at("alpha").at("x") == 10400 && evidence.at("players").at("alpha").at("y") == 10000,
+        "TCP client forged position ignored");
+  check(evidence.at("players").at("alpha").at("score") == 0, "TCP client forged score ignored");
+  check(evidence.at("players").at("bravo").at("x") == 89600, "second real TCP client moved independently");
+  check(evidence.at("executed_ticks") == 3 && evidence.at("manual_clock_ms") == 150, "injected clock drove actual server");
+  check(evidence.at("cleanup").at("descriptor_checks") == 6 && evidence.at("cleanup").at("all_descriptors_closed") == true,
+        "listener, kqueue, two accepted and two client descriptors closed");
+  for (const auto* role : {"alpha", "bravo"}) {
+    check(evidence.at("client_lifecycle").at(role) == Json::array({"LOBBY", "RUNNING", "CLOSED"}), "lifecycle observed over TCP");
+  }
+  std::cout << Json{{"integration_evidence", evidence}}.dump() << '\n';
+}
+void standalone_process_shutdown(const std::filesystem::path& executable) {
+  const auto config = std::filesystem::temp_directory_path() / ("arena-g01-server-" + std::to_string(::getpid()) + ".json");
+  write_json_file(config, Json{{"listen_port", 0}, {"clock", "manual"}});
+  int control_pipe[2], output_pipe[2];
+  check(::pipe(control_pipe) == 0 && ::pipe(output_pipe) == 0, "process pipe creation");
+  Fd control_read(control_pipe[0]), control_write(control_pipe[1]), output_read(output_pipe[0]), output_write(output_pipe[1]);
+  posix_spawn_file_actions_t actions;
+  check(::posix_spawn_file_actions_init(&actions) == 0, "spawn actions init");
+  ::posix_spawn_file_actions_adddup2(&actions, control_read.get(), STDIN_FILENO);
+  ::posix_spawn_file_actions_adddup2(&actions, output_write.get(), STDOUT_FILENO);
+  for (const int fd : {control_read.get(), control_write.get(), output_read.get(), output_write.get()})
+    ::posix_spawn_file_actions_addclose(&actions, fd);
+  std::string binary = executable.string(), mode = "server", config_name = config.string();
+  char* argv[] = {binary.data(), mode.data(), config_name.data(), nullptr};
+  pid_t child = -1;
+  const int spawned = ::posix_spawn(&child, binary.c_str(), &actions, nullptr, argv, environ);
+  ::posix_spawn_file_actions_destroy(&actions);
+  check(spawned == 0, "server process spawned");
+  struct ChildGuard {
+    pid_t pid;
+    bool reaped = false;
+    ~ChildGuard() { if (!reaped) { ::kill(pid, SIGKILL); ::waitpid(pid, nullptr, 0); } }
+  } child_guard{child};
+  control_read.reset(); output_write.reset();
+  std::string output;
+  int status = 0;
+  const auto deadline = std::chrono::steady_clock::now() + std::chrono::seconds(5);
+  while (output.find('\n') == std::string::npos && std::chrono::steady_clock::now() < deadline) {
+    pollfd readable{output_read.get(), POLLIN, 0};
+    if (::poll(&readable, 1, 10) > 0 && (readable.revents & (POLLIN | POLLHUP))) {
+      char bytes[4096]; const auto count = ::read(output_read.get(), bytes, sizeof(bytes));
+      if (count > 0) output.append(bytes, static_cast<std::size_t>(count));
+      if (count == 0) break;
+    }
+  }
+  check(output.find('\n') != std::string::npos, "child READY line within fixed process deadline");
+  const auto ready_message = Json::parse(output.substr(0, output.find('\n')));
+  check(ready_message.at("status") == "READY", "child process reported readiness");
+  const auto port = ready_message.at("port").get<std::uint16_t>();
+  TcpClient client(port);
+  client.send(message("HELLO"));
+  std::optional<Json> welcome;
+  while (!welcome && std::chrono::steady_clock::now() < deadline) {
+    welcome = client.try_receive();
+    if (!welcome) { pollfd readable{client.descriptor(), POLLIN, 0}; ::poll(&readable, 1, 10); }
+  }
+  check(welcome && welcome->at("type") == "WELCOME" && welcome->contains("session_id"), "standalone real HELLO/WELCOME");
+  check(::kill(child, SIGTERM) == 0, "normal SIGTERM sent once");
+  while (std::chrono::steady_clock::now() < deadline) {
+    pollfd readable{output_read.get(), POLLIN, 0};
+    if (::poll(&readable, 1, 10) > 0 && (readable.revents & (POLLIN | POLLHUP))) {
+      char bytes[4096]; const auto count = ::read(output_read.get(), bytes, sizeof(bytes));
+      if (count > 0) output.append(bytes, static_cast<std::size_t>(count));
+      if (output.size() > max_frame_bytes) break;
+    }
+    if (::waitpid(child, &status, WNOHANG) == child) { child_guard.reaped = true; break; }
+  }
+  if (child_guard.reaped) {
+    char bytes[4096]; ssize_t count;
+    while ((count = ::read(output_read.get(), bytes, sizeof(bytes))) > 0) output.append(bytes, static_cast<std::size_t>(count));
+  }
+  client.close(); control_write.reset();
+  output_read.reset();
+  std::filesystem::remove(config);
+  check(child_guard.reaped, "graceful child completed within process deadline");
+  check(WIFEXITED(status) && WEXITSTATUS(status) == 0, "server process exit code zero");
+  bool ready = false, stopped = false;
+  std::istringstream lines(output); std::string line;
+  while (std::getline(lines, line)) {
+    const auto entry = Json::parse(line);
+    if (entry.at("status") == "READY") ready = true;
+    if (entry.at("status") == "STOPPED") {
+      stopped = true;
+      for (const auto& [key, count] : entry.at("cleanup").items()) { (void)key; check(count == 0, "standalone cleanup all zero"); }
+    }
+  }
+  check(ready && stopped, "standalone readiness and explicit stop observed");
+  Fd rebind(::socket(AF_INET, SOCK_STREAM, 0));
+  const int yes = 1;
+  check(rebind.get() >= 0 && ::setsockopt(rebind.get(), SOL_SOCKET, SO_REUSEADDR, &yes, sizeof(yes)) == 0, "rebind probe socket");
+  sockaddr_in address{}; address.sin_family = AF_INET;
+  address.sin_addr.s_addr = htonl(INADDR_LOOPBACK); address.sin_port = htons(port);
+  check(::bind(rebind.get(), reinterpret_cast<sockaddr*>(&address), sizeof(address)) == 0, "stopped listener port is rebindable");
+  rebind.reset();
+  std::cout << Json{{"process_exit", 0}, {"reaped", child_guard.reaped}, {"hello_welcome", true},
+    {"sigterm", true}, {"port_rebindable", true}, {"output", output}}.dump() << '\n';
+}
+}
+int main(int argc, char** argv) {
+  using Test = std::pair<std::string, std::function<void()>>;
+  if (argc != 2) { std::cerr << "expected unit or integration\n"; return 2; }
+  std::vector<Test> tests;
+  if (std::string(argv[1]) == "unit") {
+    tests = {{"lifecycle_and_1200_ticks", lifecycle_and_duration}, {"integer_movement_clamp", movement_is_integer_and_bounded},
+      {"TAG_wide_distance_one_shot", tag_uses_wide_distance_and_one_shot_intent}, {"pending_input_bound", input_capacity_is_explicit},
+      {"complete_frame_owned_partial_write", complete_frame_and_owned_write_buffer}, {"RAII_descriptor_ownership", descriptor_ownership},
+      {"foreign_thread_mutation_rejected", foreign_thread_mutation_is_rejected}};
+  } else if (std::string(argv[1]) == "integration") {
+    tests = {{"real_TCP_authority_and_cleanup", real_tcp_authority_and_cleanup}, {"standalone_process_shutdown", [&] {
+      standalone_process_shutdown(std::filesystem::absolute(argv[0]).parent_path() / "arena"); }}};
+  } else { std::cerr << "unknown suite\n"; return 2; }
+  std::size_t passed = 0;
+  for (const auto& [name, test] : tests) {
+    try { test(); ++passed; std::cout << arena::Json{{"test", name}, {"result", "PASS"}}.dump() << '\n'; }
+    catch (const std::exception& error) {
+      std::cerr << arena::Json{{"test", name}, {"result", "FAIL"}, {"error", error.what()}}.dump() << '\n'; return 1;
+    }
+  }
+  std::cout << arena::Json{{"suite", argv[1]}, {"passed", passed}, {"failed", 0}, {"live_tracked_descriptors", arena::Fd::live()}}.dump() << '\n';
+  return 0;
+}
diff --git a/track b/track
new file mode 100755
index 0000000..8389177
--- /dev/null
+++ b/track
@@ -0,0 +1,37 @@
+#!/bin/bash
+set -u
+set -o pipefail
+root="$(CDPATH= cd -- "$(dirname -- "$0")" && pwd)"
+build_dir="${ARENA_BUILD_DIR:-$root/build}"
+evidence_dir="${ARENA_EVIDENCE_DIR:-$root/artifacts/evidence}"
+mkdir -p "$evidence_dir"
+run() {
+  local label="$1"
+  shift
+  local start finish result log
+  start="$(date +%s)"
+  log="$evidence_dir/${label}-${start}-$$.log"
+  "$@" 2>&1 | tee "$log"
+  result=${PIPESTATUS[0]}
+  finish="$(date +%s)"
+  printf '%s\t%s\t%s\t%s\t%s\n' "$start" "$((finish-start))" "$result" "$*" "$log" >> "$evidence_dir/runs.tsv"
+  return "$result"
+}
+if [ "$#" -eq 0 ]; then
+  printf 'usage: ./track build|unit-test|integration-test|scenario-run FILE OUT|replay-verify FILE OUT|server CONFIG\n' >&2
+  exit 2
+fi
+command="$1"
+shift
+case "$command" in
+  build)
+    run configure cmake -S "$root" -B "$build_dir" -DCMAKE_BUILD_TYPE=Release -DCMAKE_PREFIX_PATH=/opt/homebrew -DCMAKE_CXX_COMPILER=/usr/bin/clang++ "-DARENA_TSAN=${ARENA_TSAN:-OFF}" || exit "$?"
+    run build cmake --build "$build_dir" --parallel 2
+    ;;
+  unit-test) run unit "$build_dir/arena_tests" unit ;;
+  integration-test) run integration "$build_dir/arena_tests" integration ;;
+  scenario-run) run scenario "$build_dir/arena" scenario-run "$@" ;;
+  replay-verify) run replay "$build_dir/arena" replay-verify "$@" ;;
+  server) run server "$build_dir/arena" server "$@" ;;
+  *) printf 'unknown command: %s\n' "$command" >&2; exit 2 ;;
+esac
