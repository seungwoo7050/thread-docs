# Replay와 State Hash

## `G07: record bounded replay and canonical state hashes`

diff --git a/CMakeLists.txt b/CMakeLists.txt
index f2b8be3..4bb3fd6 100644
--- a/CMakeLists.txt
+++ b/CMakeLists.txt
@@ -18,16 +18,25 @@ if(ARENA_TSAN)
   add_link_options(-fsanitize=thread)
 endif()
 find_package(nlohmann_json 3.12.0 EXACT CONFIG REQUIRED)
-add_library(arena_core src/game.cpp src/transport.cpp src/scenario.cpp src/lifecycle_scenario.cpp)
-target_include_directories(arena_core PUBLIC src)
-target_link_libraries(arena_core PUBLIC nlohmann_json::nlohmann_json)
-target_compile_options(arena_core PRIVATE -Wall -Wextra -Wpedantic -Werror)
+set(ARENA_SOURCES src/game.cpp src/transport.cpp src/replay.cpp src/scenario.cpp src/lifecycle_scenario.cpp)
+add_library(arena_core ${ARENA_SOURCES})
+# Fixture friends and their definitions are absent from the shipping target.
+add_library(arena_test_core ${ARENA_SOURCES})
+target_compile_definitions(arena_test_core PUBLIC ARENA_TEST_FIXTURES=1)
+foreach(core arena_core arena_test_core)
+  target_include_directories(${core} PUBLIC src)
+  target_link_libraries(${core} PUBLIC nlohmann_json::nlohmann_json)
+  target_compile_options(${core} PRIVATE -Wall -Wextra -Wpedantic -Werror)
+endforeach()
 add_executable(arena src/main.cpp)
 target_link_libraries(arena PRIVATE arena_core)
 target_compile_options(arena PRIVATE -Wall -Wextra -Wpedantic -Werror)
 add_executable(arena_tests tests/tests.cpp)
 target_link_libraries(arena_tests PRIVATE arena_core)
 target_compile_options(arena_tests PRIVATE -Wall -Wextra -Wpedantic -Werror)
+add_executable(arena_scenarios tests/scenario_main.cpp tests/g07.cpp)
+target_link_libraries(arena_scenarios PRIVATE arena_test_core)
+target_compile_options(arena_scenarios PRIVATE -Wall -Wextra -Wpedantic -Werror)
 enable_testing()
 add_test(NAME unit COMMAND arena_tests unit)
 add_test(NAME integration COMMAND arena_tests integration)
diff --git a/TRACK.md b/TRACK.md
index bd6ecf8..fe110c6 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -1,6 +1,6 @@
-# fundamentals-cpp — G04 fixed monotonic time
+# fundamentals-cpp — G07 deterministic replay and state hash
 
-SPEC_REVISION: `5a6e4a2f8fc71d4be18c3279583bfc2558d5c232`
+SPEC_REVISION: `c1d62196ab76b55652f5d75a67514f8c6d8210ce` (phase-1; earlier evidence retains its original revision)
 
 Completion profile: `realtime-core`
 
@@ -28,11 +28,14 @@ Run from this worktree; `track` resolves its own source directory. Build never r
 ./track scenario-run /absolute/path/to/G02.json /absolute/path/to/evidence.json
 ./track scenario-run /absolute/path/to/G03.json /absolute/path/to/evidence.json
 ./track scenario-run /absolute/path/to/G04.json /absolute/path/to/evidence.json
+./track scenario-run /absolute/path/to/G07.json /absolute/path/to/L1.json
+./track scenario-run /absolute/path/to/G07.json /absolute/path/to/V.json --variant rejected-removed
 ./track replay-verify /absolute/path/to/replay.json /absolute/path/to/evidence.json
 ./track server /absolute/path/to/config.json
 ```
 
-`replay-verify` exits 2 with `NOT_ACTIVATED` / G07. It does not invent a replay implementation.
+`scenario-run` and `replay-verify` use `arena_scenarios`, a separately compiled test executable. G07 writes sibling `.replay.json` and `.records.json` files beside its evidence. Replay verification runs once, returns0 on equality and1 on the first divergence, and reports the actual canonical record. Each invocation is a separate process; no command silently repeats a campaign.
+The V option reads the original scenario and removes only its explicitly marked rejected inputs, preserving accepted TAG failures and lifecycle events. The ordinary `arena` server has no fixed-ID/roster initialization code or runtime fixture switch.
 `ARENA_BUILD_DIR` selects a clean build directory. `ARENA_EVIDENCE_DIR` selects command logs.
 The build wrapper invokes configure and compile separately; both are logged. Tests invoke the existing executable directly.
 
@@ -100,6 +103,7 @@ Both clients observe LOBBY, RUNNING and CLOSED via actual TCP. The canonical 120
 | Scheduler iteration | at most four 50ms ticks; unexecuted elapsed time remains in the integer accumulator and exposes OVERLOADED |
 | Client evidence | 4,096 received messages per client; overflow is a test failure |
 | Runner input/output | 1 MiB JSON input, 512 input commands, 32 setup commands; 4 MiB evidence output |
+| Replay capture/intake | 4,194,304 bytes including final LF; overflow latches incomplete capture and refuses export, while gameplay/hash computation continue |
 | Operator input | 4,096 bytes; overflow terminates with explicit failure |
 | Error text | 160 bytes on wire; CLI failure text 256 bytes |
 | Shutdown flush | 500ms wall ceiling; timeout is explicit metric, never a simulation clock input |
@@ -150,4 +154,12 @@ G04 adds a small integer `FixedTickAccumulator` and `Server::run_scheduler`. The
 
 The production clock is `std::chrono::steady_clock`; the canonical runner injects a manual monotonic reader into the same Server path. The fixed deltas yield `[1,1,2,0,4,2]` ticks and `[0,0,25,25,50,0]` remaining milliseconds. A wall-clock reversal is recorded only as external evidence. The complete unit suite preserves prior regressions, and a separate monotonic-mode execution extends the existing standalone shutdown integration test to verify actual adapter reads in the CLI scheduler.
 
-Input sequence/target tick (G05), abuse matrix (G06), replay/hash (G07), full/delta cadence (G08), UDP, reconnect, slow-consumer coalescing and many-room scheduling remain inactive.
+G05 sequence/target-tick and G06 four-attempt/authority guarantees remain active with their preserved regressions. G08+ replication, UDP, reconnect, slow-consumer coalescing and many-room scheduling remain inactive.
+
+## G07 replay boundary
+
+The owner records every newly accepted canonical input at its original admission boundary, including superseded inputs and future target ticks. Actual leave/disconnect changes are recorded at the same boundary. After each real tick it hashes the contract's exact ASCII-ordered canonical record, including final LF and unsigned last accepted sequence. SHA-256 uses the existing macOS CommonCrypto API from libSystem; no dependency was added. Session/connection credentials, timestamps and transport counters are excluded.
+
+Replay files contain the initial player/slot/spawn mapping, contract/clock/session constants, accepted input and lifecycle events per tick, resulting hashes and the completed tick count. Offline verification reconstructs only the initial model, then uses the existing admission, leave and simulation functions; it never injects expected resulting state. Explicitly incomplete, malformed and oversized intake fails. Capture bytes and pending events are released at server shutdown.
+
+The G07 live fixture establishes four real TCP/session bindings before one unchanged owner start-condition evaluation in `arena_test_core`. Only that target defines `ARENA_TEST_FIXTURES`; `arena_core`/`arena` cannot link the bootstrap. Public joins still start at two and reject further RUNNING joins. Full suites retain earlier regressions and add only zero-tick G07 checks. The five200-tick live/replay/variant passes and separate38-tick negative probe are explicit commands, recorded in `evidence/G07-runs.jsonl`; results are in `evidence/G07.md`.
diff --git a/evidence/G07-runs.jsonl b/evidence/G07-runs.jsonl
new file mode 100644
index 0000000..dc3ff49
--- /dev/null
+++ b/evidence/G07-runs.jsonl
@@ -0,0 +1,14 @@
+{"label":"baseline-compile","category":"compile","units":1,"ticks":0,"argv":["clang++","-std=c++20","-O2","-Wall","-Wextra","-Wpedantic","-Werror","-fsanitize=thread","-g","-I","src","-I","/opt/homebrew/include","artifacts/g07/reproduce.cpp","src/game.cpp","src/transport.cpp","-o","artifacts/g07/reproduce"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/baseline-compile.log","started_at":"2026-08-28T03:49:28.984236+00:00","duration_seconds":11.561376,"exit":1,"wrapper_pid":52661,"child_pid":52674,"ceiling_seconds":120,"timed_out":false}
+{"label":"baseline-compile-2","category":"compile","units":1,"ticks":0,"argv":["clang++","-std=c++20","-O2","-Wall","-Wextra","-Wpedantic","-Werror","-fsanitize=thread","-g","-I","src","-I","/opt/homebrew/include","artifacts/g07/reproduce.cpp","src/game.cpp","src/transport.cpp","-o","artifacts/g07/reproduce"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/baseline-compile-2.log","started_at":"2026-08-28T03:50:22.376641+00:00","duration_seconds":13.263956,"exit":0,"wrapper_pid":52904,"child_pid":52913,"ceiling_seconds":120,"timed_out":false}
+{"label":"baseline","category":"unit","units":1,"ticks":200,"expected_exit":1,"argv":["env","TSAN_OPTIONS=halt_on_error=1","./artifacts/g07/reproduce","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G07.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/baseline.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/baseline.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/baseline.json","started_at":"2026-08-28T03:51:03.849296+00:00","duration_seconds":0.915801,"exit":1,"wrapper_pid":53197,"child_pid":53206,"ceiling_seconds":120,"timed_out":false,"observed_ticks":200,"runtime_pid":53206}
+{"label":"build","category":"compile","units":2,"ticks":0,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g07-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/track","TSAN_OPTIONS=halt_on_error=1","ARENA_TSAN=ON","./track","build"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/build.log","started_at":"2026-08-28T04:10:22.351182+00:00","duration_seconds":88.546277,"exit":0,"wrapper_pid":59450,"child_pid":59460,"ceiling_seconds":120,"timed_out":false}
+{"label":"unit","category":"unit","units":1,"ticks":0,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g07-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/track","TSAN_OPTIONS=halt_on_error=1","./track","unit-test"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/unit.log","started_at":"2026-08-28T04:13:26.708031+00:00","duration_seconds":2.999033,"exit":0,"wrapper_pid":61655,"child_pid":61656,"ceiling_seconds":120,"timed_out":false}
+{"label":"integration","category":"integration","units":1,"ticks":0,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g07-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/track","TSAN_OPTIONS=halt_on_error=1","./track","integration-test"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/integration.log","started_at":"2026-08-28T04:13:29.752563+00:00","duration_seconds":1.46894,"exit":0,"wrapper_pid":61668,"child_pid":61669,"ceiling_seconds":120,"timed_out":false}
+{"label":"L1","category":"positive","units":1,"ticks":200,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g07-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/track","TSAN_OPTIONS=halt_on_error=1","./track","scenario-run","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G07.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/L1.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/L1.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/L1.json","started_at":"2026-08-28T04:13:31.260816+00:00","duration_seconds":0.814459,"exit":0,"wrapper_pid":61680,"child_pid":61681,"ceiling_seconds":120,"timed_out":false,"observed_ticks":200,"runtime_pid":61687}
+{"label":"L2","category":"positive","units":1,"ticks":200,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g07-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/track","TSAN_OPTIONS=halt_on_error=1","./track","scenario-run","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G07.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/L2.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/L2.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/L2.json","started_at":"2026-08-28T04:13:32.110402+00:00","duration_seconds":0.445921,"exit":0,"wrapper_pid":61691,"child_pid":61692,"ceiling_seconds":120,"timed_out":false,"observed_ticks":200,"runtime_pid":61698}
+{"label":"O1","category":"positive","units":1,"ticks":200,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g07-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/L1.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/O1.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/O1.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/O1.json","started_at":"2026-08-28T04:13:32.597120+00:00","duration_seconds":0.39721,"exit":0,"wrapper_pid":61701,"child_pid":61702,"ceiling_seconds":120,"timed_out":false,"observed_ticks":200,"runtime_pid":61708}
+{"label":"O2","category":"positive","units":1,"ticks":200,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g07-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/L1.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/O2.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/O2.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/O2.json","started_at":"2026-08-28T04:13:33.027637+00:00","duration_seconds":0.396546,"exit":0,"wrapper_pid":61711,"child_pid":61712,"ceiling_seconds":120,"timed_out":false,"observed_ticks":200,"runtime_pid":61718}
+{"label":"V","category":"positive","units":1,"ticks":200,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g07-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/track","TSAN_OPTIONS=halt_on_error=1","./track","scenario-run","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G07.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/V.json","--variant","rejected-removed"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/V.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/V.json","started_at":"2026-08-28T04:13:33.458351+00:00","duration_seconds":0.447921,"exit":0,"wrapper_pid":61722,"child_pid":61724,"ceiling_seconds":120,"timed_out":false,"observed_ticks":200,"runtime_pid":61730}
+{"label":"negative","category":"unit","units":1,"ticks":38,"expected_exit":1,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g07-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/L1.bad37.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/negative.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/negative.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/negative.json","started_at":"2026-08-28T04:13:33.950182+00:00","duration_seconds":0.396259,"exit":1,"wrapper_pid":61733,"child_pid":61734,"ceiling_seconds":120,"timed_out":false,"observed_ticks":38,"runtime_pid":61740}
+{"label": "arena-symbols", "category": "inspection", "units": 0, "ticks": 0, "argv": ["nm", "-C", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g07-tsan/arena"], "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp", "log": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/arena-symbols.log", "started_at": "2026-08-28T04:14:32.225892+00:00", "duration_seconds": 0.051043, "exit": 0}
+{"label": "arena_scenarios-symbols", "category": "inspection", "units": 0, "ticks": 0, "argv": ["nm", "-C", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g07-tsan/arena_scenarios"], "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp", "log": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/arena_scenarios-symbols.log", "started_at": "2026-08-28T04:14:32.280543+00:00", "duration_seconds": 0.020524, "exit": 0}
diff --git a/evidence/G07.md b/evidence/G07.md
new file mode 100644
index 0000000..abd6308
--- /dev/null
+++ b/evidence/G07.md
@@ -0,0 +1,37 @@
+# G07 — accepted-input replay and exact state hash
+
+THREAD G07; BRANCH `track/fundamentals-cpp`; PHASE phase-1; PROFILE `realtime-core`; ATTEMPT initial.
+SPEC_REVISION `c1d62196ab76b55652f5d75a67514f8c6d8210ce`; START `229b5102ab94eccc498376faecdfed28e8ffcb39`.
+Actual frozen input: `/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G07.json`, SHA-256 `d8c80202888d2e3dbeb1a7c03c14a73e84899d7cc65546e1c58a8b3102c70796`.
+
+## Preserved baseline
+
+All8 START production files matched before edits: `artifacts/g07/pre-change-production.json`, SHA-256 `629ea82fb771dae7294a0f9a86e6b87b7ebee2ab6ecb6082f0736986e3de7dde`. Baseline helper used existing friend declarations solely for test-state access, linked unchanged `game.cpp`/`transport.cpp`, and established four real HELLO/session/player bindings before one unchanged start evaluation. No public joins were claimed.
+
+Resolved before execution: `python3 artifacts/g07/run.py baseline` → `env TSAN_OPTIONS=halt_on_error=1 ./artifacts/g07/reproduce /Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G07.json /private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g07/baseline.json`.
+
+Expected exit1,200 actual ticks: G06 supplied no production replay artifact/hash sequence. All17 admission codes, LEAVE175, charlie's accepted-but-failed TAG199 and final physical/sequence state already passed (NOT_REPRODUCED). Bravo seq2 was accepted98/target100; alpha10 and11 both accepted100. All10 descriptors closed; active cleanup/FD delta0. Raw baseline SHA-256 `36bb9b29fb0674765687fbbb7ebf72c1a32e8872ae9cf646599804a5eecb38cb`. Main notified before production edits. One helper compile failed on a string/JSON comparison; its correction/recompile and original log remain in the ledger.
+
+## Implementation and bounded verification
+
+Owner capture records accepted inputs at admission (including superseded intents), real leave/disconnect events and exact per-tick canonical SHA-256. Initial mapping excludes credentials. Native CommonCrypto adds no dependency. Capture overflow latches incomplete recording, releases unfinished capture work, refuses export and leaves hashing/gameplay active; no new wire or Room-overload policy. The four-live-peer bootstrap is compiled only into `arena_scenarios`/`arena_test_core`; normal public start/join behavior remains unchanged.
+
+`evidence/G07-runs.jsonl` is the single exact command ledger, including failures, argv, raw paths, duration, exit, wrapper/child/runtime PID and G07-specific tick budget. `artifacts/g07/commands.json` resolved all commands/results before execution, including V's `--variant rejected-removed`. `artifacts/g07/verify.py` runs full unit/integration then L1/L2/O1/O2/V and the separate negative command sequentially, with120s ceilings and stop on unexpected failure. New unit checks execute zero Room ticks; earlier regressions remain intact.
+
+| Verification | Actual result | Exit |
+| --- | --- | ---: |
+| TSan configure/build | PASS,88.546277s | 0 |
+| Complete unit suite | 23/23 PASS,2.999033s | 0 |
+| Complete integration suite | 3/3 PASS,1.468940s | 0 |
+| L1 / L2 / O1 / O2 / V | Each200 ticks, identical200 hashes | 0 each |
+| Separate negative | First divergence37,38 actual ticks, actual record reported | 1 expected |
+
+Five fresh runtime PIDs: L1=61687,L2=61698,O1=61708,O2=61718,V=61730; negative=61740. Full raw results are `artifacts/g07/{L1,L2,O1,O2,V,negative}.json`, with sibling `.records.json` canonical bytes/states. L1's immutable replay `artifacts/g07/L1.replay.json` is23397 bytes, SHA-256 `34a5cfedc7092a7cd0a61c7f6b29773051bb4ba0d4fbd82e208407c1089ac11e`; L2 and V produced identical bytes, and O1/O2 read those same L1 bytes. Hash-list digest (each hash plus LF) `34fc152b26677e829a71add1aea790682735e65da544fbdd24b1b203e80261ea`; tick0 `dacaa773d40c5b7b68b6fc5b18d26da68d49622ad11f4c7e898ba55fd08231f6`, tick199 `e3c9b271e14da9708c26ea468b8c7a5293798a984f3880f77386f1f5e9799bf6`. Saved canonical bytes were additionally read and hashed with Python, without simulation.
+
+Actual admission results:13 ACCEPTED; INPUT_LATE10, INPUT_TOO_EARLY11, SEQUENCE_CONFLICT12, INPUT_STALE101. Replay retains all13 accepted inputs, bravo98/target100, both alpha10/11 at100, charlie's accepted TAG199, and delta's LEAVE175. Final alpha/bravo/charlie positions `[50000,50000]`; delta `[60400,50000]` LEFT/STOP. Directions NORTH/SOUTH/EAST/STOP; scores1/0/0/0; last sequences12/3/3/3; last TAG ticks199/-20/-20/-20; Room RUNNING,epoch0,session duration1200.
+
+`negative-mutation.json` records the disposable copy and sole one-byte change to `ticks[37].state_hash`; original L1 stayed immutable. `equality.json` records five-run equality. `build-proof.json` and two zero-tick symbol inspections show no fixture symbols/definition in `arena`/`arena_core`, with fixture symbols and `ARENA_TEST_FIXTURES=1` only in the test targets.
+
+Live high waters/bounds: connections4/512, attempts2/4, pending2/64, mailbox1/512, control1/64, parser224/16388, replay23397/4194304. Each live pass closed all10 descriptors; all active cleanup counters including replay bytes/events and FD delta were zero. The zero-tick recorder-only capacity probe reached exactly4194304 accounted bytes, rejected append29125, latched incomplete capture/refused export, preserved gameplay/hash operation, and released capture bytes. No TSan errors were reported.
+
+Actual budget: compile/configure4/8 (includes the failed helper compile), unit3/4 (baseline, full suite, separate negative), integration1/2, positives5/5=1000ticks, negative1/1=38ticks. Total G07 simulation1238 including baseline200; new unit checks0ticks; earlier-stage regressions unchanged. NETWORK_FAULT_RUNS0; LOAD_RUNS0. No G08+, dependency, spec/index/threads, tag or push work. Root independent C++/cross verification remains pending at submission.
diff --git a/src/game.cpp b/src/game.cpp
index 9b1d7ef..f41c15f 100644
--- a/src/game.cpp
+++ b/src/game.cpp
@@ -75,9 +75,12 @@ Player& Room::join(std::string player_id, std::string session_id, std::uint64_t
   const auto key = player.id;
   auto [found, inserted] = players_.emplace(key, std::move(player));
   if (!inserted) throw std::logic_error("server generated duplicate player id");
+  evaluate_start_condition();
+  return found->second;
+}
+void Room::evaluate_start_condition() {
   const auto ready = std::count_if(players_.begin(), players_.end(), [](const auto& pair) { return pair.second.connected; });
   if (ready >= 2) transition("RUNNING");
-  return found->second;
 }
 std::optional<std::string> Room::begin_input_attempt(const std::string& player_id) {
   assert_owner();
diff --git a/src/game.hpp b/src/game.hpp
index e13569a..21f261f 100644
--- a/src/game.hpp
+++ b/src/game.hpp
@@ -90,7 +90,11 @@ class Room {
   std::size_t input_attempt_high_water() const { return input_attempt_high_water_; }
  private:
   friend void initialize_shared_victim_fixture(Room& room);
+#ifdef ARENA_TEST_FIXTURES
+  friend struct ReplayFixture;
+#endif
   void assert_owner() const;
+  void evaluate_start_condition();
   void transition(std::string status);
   std::thread::id owner_;
   std::string id_;
diff --git a/src/main.cpp b/src/main.cpp
index 37f9bef..165cbbe 100644
--- a/src/main.cpp
+++ b/src/main.cpp
@@ -58,7 +58,7 @@ int serve(const arena::Json& config) {
 int main(int argc, char** argv) {
   try {
     if (argc >= 2 && std::string(argv[1]) == "replay-verify") {
-      std::cerr << "{\"error\":\"NOT_ACTIVATED\",\"thread\":\"G07\"}\n"; return 2;
+      std::cerr << "{\"error\":\"TEST_BUILD_ONLY\",\"command\":\"replay-verify\"}\n"; return 2;
     }
     if (argc == 4 && std::string(argv[1]) == "scenario-run") {
       const auto evidence = arena::run_scenario(arena::read_json_file(argv[2]));
diff --git a/src/replay.cpp b/src/replay.cpp
new file mode 100644
index 0000000..c0a28a7
--- /dev/null
+++ b/src/replay.cpp
@@ -0,0 +1,142 @@
+#include "replay.hpp"
+#include <CommonCrypto/CommonDigest.h>
+#include <algorithm>
+#include <array>
+#include <charconv>
+#include <limits>
+#include <stdexcept>
+
+namespace arena {
+namespace {
+template<class Integer> std::string decimal(Integer value) {
+  std::array<char,32> buffer{};
+  const auto result = std::to_chars(buffer.data(), buffer.data()+buffer.size(), value);
+  if (result.ec != std::errc{}) throw std::logic_error("canonical integer conversion failed");
+  return {buffer.data(), result.ptr};
+}
+}
+std::string sha256(std::string_view bytes) {
+  if (bytes.size() > std::numeric_limits<CC_LONG>::max()) throw std::length_error("SHA-256 input too large");
+  std::array<unsigned char,CC_SHA256_DIGEST_LENGTH> digest{};
+  CC_SHA256(bytes.data(), static_cast<CC_LONG>(bytes.size()), digest.data());
+  constexpr char hex[] = "0123456789abcdef";
+  std::string result;
+  result.reserve(digest.size()*2);
+  for (const auto byte : digest) { result += hex[byte >> 4U]; result += hex[byte & 15U]; }
+  return result;
+}
+std::string canonical_state(const Room& room) {
+  std::string record = "v=1|room=" + room.id() + "|tick=" + decimal(room.executed_ticks()-1) +
+    "|status=" + room.status() + "|owner_epoch=0\n";
+  // Room keys are its player identifiers; the map uses their ASCII byte order.
+  for (const auto& [id, player] : room.players()) {
+    record += "player=" + id + "|slot=" + decimal(player.slot) + "|x=" + decimal(player.x) +
+      "|y=" + decimal(player.y) + "|dir=" + direction_name(player.direction) +
+      "|score=" + decimal(player.score) + "|conn=" + (player.connected ? "CONNECTED" : "LEFT") +
+      "|last_seq=" + decimal(player.last_accepted_seq()) + "|last_tag_tick=" + decimal(player.last_tag_tick) + "\n";
+  }
+  return record;
+}
+Json parse_replay_artifact(std::string_view bytes) {
+  if (bytes.size() > max_replay_bytes) throw std::length_error("replay input exceeds 4194304 bytes");
+  auto artifact = Json::parse(bytes);
+  if (!artifact.at("contract_version").is_number_integer() || artifact.at("contract_version") != 1 ||
+      !artifact.at("owner_epoch").is_number_integer() || artifact.at("owner_epoch") != 0 ||
+      !artifact.at("tick_duration_ms").is_number_integer() || artifact.at("tick_duration_ms") != tick_duration_ms ||
+      !artifact.at("session_duration_ticks").is_number_integer() || artifact.at("session_duration_ticks") != session_ticks ||
+      !artifact.at("complete").is_boolean() || artifact.at("complete") != true ||
+      !artifact.at("executed_ticks").is_number_integer() || artifact.at("executed_ticks") < 0 ||
+      artifact.at("executed_ticks") > session_ticks || !artifact.at("ticks").is_array() ||
+      artifact.at("ticks").size() != artifact.at("executed_ticks").get<std::size_t>())
+    throw std::invalid_argument("incomplete or malformed replay header");
+  int tick = 0;
+  for (const auto& record : artifact.at("ticks")) {
+    const auto hash = record.at("state_hash").get<std::string>();
+    if (!record.at("tick").is_number_integer() || record.at("tick") != tick++ ||
+        !record.at("events").is_array() || record.at("events").size() > max_players*(max_input_attempts_per_tick+1) ||
+        hash.size() != 64 || !std::all_of(hash.begin(),hash.end(),[](unsigned char ch) {
+          return (ch >= '0' && ch <= '9') || (ch >= 'a' && ch <= 'f');
+        })) throw std::invalid_argument("malformed replay tick record");
+  }
+  return artifact;
+}
+void ReplayLog::start(const Room& room) {
+  if (!artifact_.is_null() || room.status() != "RUNNING" || room.executed_ticks() != 0)
+    throw std::logic_error("replay starts once at the initial RUNNING boundary");
+  Json initial = Json::array();
+  for (const auto& [id, player] : room.players()) initial.push_back(Json{{"player_id",id},{"slot",player.slot},
+    {"spawn",Json::array({player.x,player.y})},{"connectivity",player.connected ? "CONNECTED" : "LEFT"}});
+  Json artifact{{"contract_version",1},{"room_id",room.id()},{"owner_epoch",0},{"complete",true},{"executed_ticks",0},
+    {"tick_duration_ms",tick_duration_ms},{"session_duration_ticks",session_ticks},
+    {"initial_players",std::move(initial)},{"ticks",Json::array()}};
+  const auto bytes = artifact.dump().size()+1; // Includes the final file LF.
+  if (!reserve(bytes)) return;
+  artifact_ = std::move(artifact); bytes_ = bytes;
+  high_water_bytes_ = bytes_;
+}
+bool ReplayLog::reserve(std::size_t bytes) {
+  if (!complete()) return false;
+  if (bytes <= max_replay_bytes) return true;
+  // Capture failure is not a gameplay/transport policy. Retain a bounded
+  // incomplete prefix, release unfinished work, and refuse every export.
+  failure_ = "replay exceeds 4194304 bytes; recording incomplete";
+  pending_ = Json::array(); pending_bytes_ = 0;
+  return false;
+}
+void ReplayLog::append(Json event, int before_tick) {
+  if (!complete()) return;
+  if (artifact_.is_null() || before_tick < 0 || before_tick >= session_ticks ||
+      artifact_.at("ticks").size() != static_cast<std::size_t>(before_tick))
+    throw std::logic_error("replay event outside its admission boundary");
+  event["before_tick"] = before_tick;
+  const auto added = event.dump().size() + (pending_.empty() ? 0 : 1);
+  const auto overhead = Json{{"tick",before_tick},{"events",Json::array()},
+    {"state_hash",std::string(64,'0')}}.dump().size() + (before_tick == 0 ? 0 : 1) +
+    decimal(before_tick+1).size()-decimal(before_tick).size();
+  if (!reserve(bytes_ + pending_bytes_ + added + overhead)) return;
+  pending_.push_back(std::move(event));
+  pending_bytes_ += added;
+  high_water_bytes_ = std::max(high_water_bytes_,bytes_+pending_bytes_+overhead);
+}
+void ReplayLog::accepted_input(const Room& room, const std::string& player_id) {
+  const auto& input = *room.players().at(player_id).last_accepted_input;
+  append(Json{{"kind","INPUT"},{"player_id",player_id},{"seq",input.seq},
+    {"target_tick",std::get<std::uint64_t>(input.target_tick)},
+    {"direction",direction_name(input.intent.direction)},
+    {"tag_target_player_id",input.intent.tag_target ? Json(*input.intent.tag_target) : Json(nullptr)},
+    {"owner_epoch",0}},room.executed_ticks());
+}
+void ReplayLog::left(const Room& room, const std::string& player_id, const std::string& kind) {
+  append(Json{{"kind",kind},{"player_id",player_id}},room.executed_ticks());
+}
+void ReplayLog::finish_tick(const Room& room) {
+  last_state_hash_ = sha256(canonical_state(room));
+  if (!complete()) return; // Hashing and authoritative simulation continue.
+  const int tick = room.executed_ticks()-1;
+  if (artifact_.is_null() || artifact_.at("ticks").size() != static_cast<std::size_t>(tick) || tick >= session_ticks)
+    throw std::logic_error("replay tick order mismatch");
+  Json record{{"tick",tick},{"events",std::move(pending_)},{"state_hash",last_state_hash_}};
+  const auto added = record.dump().size() + (tick == 0 ? 0 : 1) +
+    decimal(tick+1).size()-decimal(tick).size();
+  if (!reserve(bytes_+added)) return;
+  artifact_["ticks"].push_back(std::move(record));
+  artifact_["executed_ticks"] = room.executed_ticks();
+  bytes_ += added;
+  pending_ = Json::array(); pending_bytes_ = 0;
+  high_water_bytes_ = std::max(high_water_bytes_,bytes_);
+}
+const Json& ReplayLog::artifact() const {
+  if (!complete()) throw std::runtime_error(failure_);
+  if (artifact_.is_null() || !pending_.empty()) throw std::logic_error("replay export requires a closed tick boundary");
+  return artifact_;
+}
+std::string ReplayLog::serialize() const {
+  auto bytes = artifact().dump() + '\n';
+  if (bytes.size() != bytes_) throw std::logic_error("replay byte accounting mismatch");
+  if (bytes.size() > max_replay_bytes) throw std::logic_error("replay byte bound mismatch");
+  return bytes;
+}
+void ReplayLog::clear() {
+  artifact_ = nullptr; pending_ = Json::array(); bytes_ = 0; pending_bytes_ = 0;
+}
+}
diff --git a/src/replay.hpp b/src/replay.hpp
new file mode 100644
index 0000000..a5f4c90
--- /dev/null
+++ b/src/replay.hpp
@@ -0,0 +1,39 @@
+#pragma once
+#include "game.hpp"
+#include <string_view>
+
+namespace arena {
+inline constexpr std::size_t max_replay_bytes = 4'194'304;
+std::string sha256(std::string_view bytes);
+std::string canonical_state(const Room& room);
+Json parse_replay_artifact(std::string_view bytes);
+
+// The owner records newly accepted inputs at admission, not only the intent
+// later selected for a target tick. No transport credentials enter this log.
+class ReplayLog {
+ public:
+  void start(const Room& room);
+  void accepted_input(const Room& room, const std::string& player_id);
+  void left(const Room& room, const std::string& player_id, const std::string& kind);
+  void finish_tick(const Room& room);
+  std::string serialize() const;
+  const Json& artifact() const;
+  std::size_t bytes() const { return bytes_ + pending_bytes_; }
+  std::size_t high_water_bytes() const { return high_water_bytes_; }
+  std::size_t pending_events() const { return pending_.size(); }
+  bool complete() const { return failure_.empty(); }
+  const std::string& failure() const { return failure_; }
+  const std::string& last_state_hash() const { return last_state_hash_; }
+  void clear();
+ private:
+  bool reserve(std::size_t bytes);
+  void append(Json event, int before_tick);
+  Json artifact_;
+  Json pending_ = Json::array();
+  std::size_t bytes_ = 0;
+  std::size_t pending_bytes_ = 0;
+  std::size_t high_water_bytes_ = 0;
+  std::string failure_;
+  std::string last_state_hash_;
+};
+}
diff --git a/src/scenario.cpp b/src/scenario.cpp
index b3b1f69..e82d307 100644
--- a/src/scenario.cpp
+++ b/src/scenario.cpp
@@ -45,7 +45,7 @@ Json read_json_file(const std::filesystem::path& path, std::size_t limit) {
 void write_json_file(const std::filesystem::path& path, const Json& value) {
   if (path.has_parent_path()) std::filesystem::create_directories(path.parent_path());
   const auto text = value.dump(2);
-  if (text.size() > 4'194'304) throw std::runtime_error("evidence output exceeds fixed bound");
+  if (text.size()+1 > max_replay_bytes) throw std::runtime_error("evidence output exceeds fixed bound");
   std::ofstream file(path, std::ios::binary);
   if (!file) throw std::runtime_error("cannot open evidence output");
   file << text << '\n';
diff --git a/src/transport.cpp b/src/transport.cpp
index 5b5d349..d907e44 100644
--- a/src/transport.cpp
+++ b/src/transport.cpp
@@ -423,6 +423,7 @@ void Server::handle(const Envelope& envelope) {
       queue(id, std::move(lobby));
       queue(id, std::move(reply));
       if (room_.status() == "RUNNING") {
+        replay_.start(room_);
         accumulator_.reset(read_monotonic());
         Json state = room_.view(); state.update(message("SNAPSHOT")); broadcast(state);
       }
@@ -432,13 +433,14 @@ void Server::handle(const Envelope& envelope) {
       reject("PLAYER_MISMATCH", "player must belong to this connection"); return;
     }
     if (type == "LEAVE_ROOM") {
-      room_.leave(id);
+      leave_room(id,"LEAVE_ROOM");
       Json state = room_.view(); state.update(message("SNAPSHOT")); queue(id, state); return;
     }
     const auto result = admit_input(room_, conn->player_id, value);
     if (result.error) {
       reject(*result.error, "input was not accepted"); return;
     }
+    if (!result.duplicate) replay_.accepted_input(room_,conn->player_id);
     Json reply = message("INPUT_ACK"); reply["player_id"] = conn->player_id; reply["accepted"] = true;
     reply["seq"] = value.at("seq").get<std::uint64_t>(); reply["code"] = result.duplicate ? "DUPLICATE" : "ACCEPTED";
     reply["last_accepted_seq"] = room_.players().at(conn->player_id).last_accepted_seq();
@@ -449,9 +451,18 @@ void Server::handle(const Envelope& envelope) {
     reject("MESSAGE_INVALID", "invalid input field");
   }
 }
+void Server::leave_room(std::uint64_t connection_id, const std::string& kind) {
+  std::string player_id;
+  if (room_.status() == "RUNNING") {
+    for (const auto& [id, player] : room_.players())
+      if (player.connection_id == connection_id && player.connected) player_id = id;
+  }
+  room_.leave(connection_id);
+  if (!player_id.empty()) replay_.left(room_,player_id,kind);
+}
 void Server::drain_mailbox() {
   // Room mutations happen after all ready I/O callbacks have completed.
-  for (const auto id : disconnected_) room_.leave(id);
+  for (const auto id : disconnected_) leave_room(id,"DISCONNECT");
   disconnected_.clear();
   const auto size = mailbox_.size();
   for (std::size_t i = 0; i < size; ++i) {
@@ -495,6 +506,7 @@ void Server::advance_one_tick() {
     error["player_id"] = failure.player_id; error["tick"] = room_.executed_ticks() - 1;
     ++errors_["ACTION_REJECTED"]; queue(failure.connection_id, std::move(error));
   }
+  replay_.finish_tick(room_);
   if (room_.status() == "FINISHED") {
     accumulator_.reset(0); last_batch_ = {};
     Json result = room_.view(); result.update(message("ROOM_FINISHED")); broadcast(result);
@@ -505,6 +517,9 @@ Json Server::metrics() const {
     {"mailbox_high_water", mailbox_high_water_}, {"outbound_control_high_water", outbound_high_water_},
     {"connection_high_water", connection_high_water_}, {"input_per_player_high_water", room_.input_high_water()},
     {"input_attempt_per_player_high_water", room_.input_attempt_high_water()},
+    {"replay_bytes_high_water",replay_.high_water_bytes()},
+    {"replay_capture_complete",replay_.complete()},{"replay_capture_error",replay_.failure()},
+    {"last_state_hash",replay_.last_state_hash()},
     {"max_read_bytes", max_read_bytes_}, {"parser_buffer_high_water", parser_high_water_},
     {"parser_storage_bytes_per_connection", FrameParser::storage_bytes}, {"need_more_events", need_more_events_},
     {"message_error_events", message_error_events_}, {"terminal_frame_events", terminal_frame_events_},
@@ -524,6 +539,7 @@ Json Server::cleanup() const {
   return Json{{"server_connections", connections_.size()}, {"server_descriptors", owned_descriptors().size()},
     {"mailbox_messages", mailbox_.size()}, {"pending_inputs", room_.pending_count()}, {"outbound_messages", queued},
     {"input_attempts", input_attempts},
+    {"replay_bytes",replay_.bytes()},{"replay_pending_events",replay_.pending_events()},
     {"parser_buffered_bytes", parser_buffered}, {"parser_storage_bytes", connections_.size() * FrameParser::storage_bytes},
     {"worker_threads", 0}, {"timers", 0}, {"disconnect_notifications", disconnected_.size()},
     {"scheduler_pending_ms", accumulator_.remaining_ms()}};
@@ -541,6 +557,7 @@ void Server::shutdown() {
   listener_.reset();
   drain_mailbox();
   room_.close();
+  replay_.clear();
   accumulator_.reset(0); last_batch_ = {};
   if (room_.status() == "CLOSED") {
     Json state = room_.view(); state.update(message("SNAPSHOT")); broadcast(state);
diff --git a/src/transport.hpp b/src/transport.hpp
index dc56b58..3e36529 100644
--- a/src/transport.hpp
+++ b/src/transport.hpp
@@ -1,5 +1,5 @@
 #pragma once
-#include "game.hpp"
+#include "replay.hpp"
 #include <array>
 #include <atomic>
 #include <cstddef>
@@ -78,6 +78,7 @@ class Server {
   TickBatch run_scheduler();
   void shutdown();
   const Room& room() const { return room_; }
+  const ReplayLog& replay() const { return replay_; }
   Json metrics() const;
   Json cleanup() const;
   std::vector<int> owned_descriptors() const;
@@ -104,6 +105,9 @@ class Server {
     std::deque<Envelope> entries_;
   };
   friend Json run_mailbox_probe(std::size_t capacity);
+#ifdef ARENA_TEST_FIXTURES
+  friend struct ReplayFixture;
+#endif
   Connection* connection(std::uint64_t id);
   void register_event(int fd, short filter, unsigned short flags, std::uint64_t connection_id = 0);
   void accept_ready();
@@ -114,6 +118,7 @@ class Server {
   void queue(std::uint64_t connection_id, Json value);
   void broadcast(const Json& value);
   void handle(const Envelope& envelope);
+  void leave_room(std::uint64_t connection_id, const std::string& kind);
   std::int64_t read_monotonic();
   std::string new_id(const std::string& prefix, std::uint64_t number) const;
   ManualClock& clock_;
@@ -130,6 +135,7 @@ class Server {
   Mailbox mailbox_;
   std::set<std::uint64_t> disconnected_;
   Room room_;
+  ReplayLog replay_;
   std::string nonce_;
   std::uint64_t next_connection_ = 1;
   std::uint64_t next_player_ = 1;
diff --git a/tests/g07.cpp b/tests/g07.cpp
new file mode 100644
index 0000000..fe51db4
--- /dev/null
+++ b/tests/g07.cpp
@@ -0,0 +1,251 @@
+#include "g07.hpp"
+#ifndef ARENA_TEST_FIXTURES
+#error G07 roster bootstrap must never be compiled into the shipping artifact
+#endif
+#include <algorithm>
+#include <memory>
+#include <set>
+#include <stdexcept>
+#include <unistd.h>
+
+namespace arena {
+namespace {
+void require(bool condition, const std::string& text) {
+  if (!condition) throw std::runtime_error("G07: " + text);
+}
+bool identifier(const std::string& value) {
+  return !value.empty() && value.size() <= 64 && std::all_of(value.begin(),value.end(),[](unsigned char ch) {
+    return (ch >= 'A' && ch <= 'Z') || (ch >= 'a' && ch <= 'z') ||
+      (ch >= '0' && ch <= '9') || ch == '_' || ch == '-';
+  });
+}
+struct Peer {
+  std::unique_ptr<TcpClient> tcp;
+  std::string session;
+  std::string player;
+};
+Json observed_state(const Room& room) {
+  auto state = room.view(); state["owner_epoch"] = 0;
+  for (auto& item : state["players"]) {
+    const auto& player = room.players().at(item.at("player_id").get<std::string>());
+    item["last_seq"] = player.last_accepted_seq(); item["pending"] = player.pending.size();
+    item["applied_seq"] = player.applied_seq ? Json(*player.applied_seq) : Json(nullptr);
+  }
+  return state;
+}
+Json tick_record(const Room& room, const std::string& hash) {
+  return Json{{"tick",room.executed_ticks()-1},{"canonical_record",canonical_state(room)},
+    {"state_hash",hash},{"state",observed_state(room)}};
+}
+Json request(const Json& event, const std::map<std::string,Peer>& peers, const std::string& room_id) {
+  const auto& peer = peers.at(event.at("client").get<std::string>());
+  const auto kind = event.at("kind").get<std::string>();
+  auto value = message(kind);
+  value["session_id"] = peer.session; value["room_id"] = room_id; value["player_id"] = peer.player;
+  if (kind == "INPUT") {
+    for (const auto* key : {"seq","target_tick","direction","owner_epoch"}) value[key] = event.at(key);
+    value["tag_target_player_id"] = event.at("tag_target_role").is_null() ? Json(nullptr) :
+      Json(peers.at(event.at("tag_target_role").get<std::string>()).player);
+  }
+  return value;
+}
+void final_fixture_state(const Room& room) {
+  const auto& a = room.players().at("player-00"); const auto& b = room.players().at("player-01");
+  const auto& c = room.players().at("player-02"); const auto& d = room.players().at("player-03");
+  require(a.x == 50000 && a.y == 50000 && a.direction == Direction::north && a.score == 1 && a.last_accepted_seq() == 12 && a.last_tag_tick == 199 && a.connected &&
+    b.x == 50000 && b.y == 50000 && b.direction == Direction::south && b.score == 0 && b.last_accepted_seq() == 3 && b.last_tag_tick == -20 && b.connected &&
+    c.x == 50000 && c.y == 50000 && c.direction == Direction::east && c.score == 0 && c.last_accepted_seq() == 3 && c.last_tag_tick == -20 && c.connected &&
+    d.x == 60400 && d.y == 50000 && d.direction == Direction::stop && d.score == 0 && d.last_accepted_seq() == 3 && d.last_tag_tick == -20 && !d.connected &&
+    room.status() == "RUNNING" && session_ticks == 1200, "fixed final gameplay state");
+}
+}
+
+// Defined only in this test target. Both core compilations retain the same
+// normal join logic; only this one has private fixture access.
+struct ReplayFixture {
+  static void initialize(Room& room, const std::string& room_id, std::vector<Player> players) {
+    room.assert_owner();
+    require(room.status() == "ABSENT" && players.size() >= 2 && players.size() <= max_players &&
+            identifier(room_id), "bounded initial replay roster");
+    const auto count = players.size();
+    room.create(room_id);
+    for (auto& player : players) {
+      const auto id = player.id;
+      require(room.players_.emplace(id,std::move(player)).second, "duplicate initial player");
+    }
+    room.next_slot_ = static_cast<int>(count);
+    room.evaluate_start_condition();
+    require(room.lifecycle() == std::vector<std::string>({"LOBBY","RUNNING"}) &&
+            room.executed_ticks() == 0, "one unchanged start evaluation");
+  }
+  static Json live(Server& server, const Json& scenario, const std::map<std::string,Peer>& peers) {
+    require(server.connections_.size() == 4 && peers.size() == 4, "four live TCP connections before bootstrap");
+    std::vector<Player> players;
+    Json bindings = Json::array();
+    for (const auto& item : scenario.at("players")) {
+      const auto role = item.at("client").get<std::string>(); const auto& peer = peers.at(role);
+      auto found = std::find_if(server.connections_.begin(),server.connections_.end(),
+        [&](const auto& entry) { return entry.second.session_id == peer.session; });
+      require(found != server.connections_.end() && found->second.player_id.empty(), "real server session binding");
+      auto& connection = found->second; connection.player_id = peer.player;
+      Player player; player.id = peer.player; player.session_id = connection.session_id;
+      player.connection_id = connection.id; player.slot = item.at("slot").get<int>();
+      require(identifier(player.id) && player.slot == static_cast<int>(players.size()), "frozen identifiers/ordered slots");
+      player.x = item.at("spawn").at(0).get<int>(); player.y = item.at("spawn").at(1).get<int>();
+      players.push_back(std::move(player));
+      bindings.push_back(Json{{"client",role},{"session_id",connection.session_id},{"player_id",connection.player_id},
+        {"connection_id",connection.id},{"server_descriptor",connection.fd.get()},{"client_descriptor",peer.tcp->descriptor()}});
+    }
+    // All four transport/session/player bindings precede the one start call.
+    initialize(server.room_,scenario.at("room_id").get<std::string>(),std::move(players));
+    server.replay_.start(server.room_);
+    server.accumulator_.reset(server.read_monotonic());
+    return bindings;
+  }
+  static void offline(Room& room, const Json& artifact) {
+    require(artifact.at("initial_players").is_array() && artifact.at("initial_players").size() >= 2 &&
+            artifact.at("initial_players").size() <= max_players, "bounded initial mapping");
+    const auto count = artifact.at("initial_players").size();
+    std::vector<Player> players; std::set<int> slots;
+    for (const auto& item : artifact.at("initial_players")) {
+      Player player; player.id = item.at("player_id").get<std::string>();
+      require(identifier(player.id) && item.at("slot").is_number_integer() && item.at("slot") >= 0 &&
+        item.at("slot") < count && (item.at("connectivity") == "CONNECTED" || item.at("connectivity") == "LEFT"),
+        "initial mapping");
+      player.connected = item.at("connectivity") == "CONNECTED";
+      player.slot = item.at("slot").get<int>();
+      require(slots.insert(player.slot).second && item.at("spawn").is_array() && item.at("spawn").size() == 2,
+        "unique initial slot/spawn");
+      for (const auto& coordinate : item.at("spawn"))
+        require(coordinate.is_number_integer() && coordinate >= 0 && coordinate <= 100000, "initial spawn bounds");
+      player.x = item.at("spawn").at(0).get<int>(); player.y = item.at("spawn").at(1).get<int>();
+      player.connection_id = static_cast<std::uint64_t>(player.slot)+1;
+      player.session_id = "offline-" + std::to_string(player.slot);
+      players.push_back(std::move(player));
+    }
+    initialize(room,artifact.at("room_id").get<std::string>(),std::move(players));
+  }
+};
+
+ReplayRun run_replay_scenario(const Json& scenario, bool rejected_removed) {
+  require(scenario.at("thread") == "G07" && scenario.at("contract_version") == 1 && scenario.at("seed") == 7050 &&
+    scenario.at("ticks") == 200 && scenario.at("players").size() == 4 && scenario.at("events").size() == 18 &&
+    scenario.at("clock").at("kind") == "manual" && scenario.at("clock").at("tick_duration_ms") == tick_duration_ms &&
+    scenario.at("artifact_max_bytes") == max_replay_bytes && scenario.at("socket_ceiling_ms") == 5000,
+    "frozen canonical dimensions");
+  const int descriptors_before = Fd::live();
+  ManualClock clock; Server server(clock); std::map<std::string,Peer> peers;
+  for (const auto& item : scenario.at("players")) {
+    const auto role = item.at("client").get<std::string>(); auto& peer = peers[role];
+    peer.tcp = std::make_unique<TcpClient>(server.port()); peer.tcp->send(message("HELLO"));
+    peer.session = peer.tcp->receive_type(server,"WELCOME").at("session_id").get<std::string>();
+    peer.player = item.at("player_id").get<std::string>();
+  }
+  const auto bindings = ReplayFixture::live(server,scenario,peers);
+  Json admissions = Json::array(), wire = Json::array(), lifecycle = Json::array(), actions = Json::array();
+  ReplayRun run; run.records = Json::array(); Json hashes = Json::array();
+  std::size_t removed = 0;
+  for (int tick = 0; tick < 200; ++tick) {
+    for (const auto& event : scenario.at("events")) {
+      if (event.at("before_tick") != tick) continue;
+      const auto kind = event.at("kind").get<std::string>();
+      if (rejected_removed && !event.at("include_in_rejected_removed_variant").get<bool>()) {
+        require(kind == "INPUT", "variant must preserve lifecycle"); ++removed; continue;
+      }
+      const auto role = event.at("client").get<std::string>(); auto& peer = peers.at(role);
+      const auto value = request(event,peers,server.room().id()); const auto before = observed_state(server.room());
+      peer.tcp->send(value); const auto response = peer.tcp->receive(server); const auto after = observed_state(server.room());
+      wire.push_back(Json{{"before_tick",tick},{"client",role},{"request",value},{"response",response},
+        {"before",before},{"after",after}});
+      if (kind == "INPUT") {
+        const auto code = response.at("code").get<std::string>();
+        require(code == event.at("expected_admission").get<std::string>(), "actual owner admission code");
+        if (code != "ACCEPTED") require(before == after, "rejected admission mutated gameplay");
+        admissions.push_back(Json{{"client",role},{"seq",event.at("seq")},{"before_tick",tick},
+          {"target_tick",event.at("target_tick")},{"code",code}});
+      } else {
+        require(kind == "LEAVE_ROOM" && response.at("type") == "SNAPSHOT" &&
+          !server.room().players().at(peer.player).connected, "real LEAVE_ROOM lifecycle");
+        lifecycle.push_back(Json{{"before_tick",tick},{"kind",kind},{"player_id",peer.player},{"client",role},{"connectivity","LEFT"}});
+      }
+    }
+    server.advance_one_tick();
+    require(server.room().executed_ticks() == tick+1 && clock.now_ms == (tick+1)*50, "one real owner tick");
+    const auto hash = server.replay().artifact().at("ticks").back().at("state_hash").get<std::string>();
+    hashes.push_back(hash); run.records.push_back(tick_record(server.room(),hash));
+    if (tick == 199) {
+      const auto failure = peers.at("charlie").tcp->receive(server);
+      require(failure.at("code") == "ACTION_REJECTED" && failure.at("tick") == 199 &&
+        failure.at("player_id") == peers.at("charlie").player, "accepted charlie TAG action failure");
+      actions.push_back(failure);
+    }
+  }
+  require(removed == (rejected_removed ? 4U : 0U) && admissions.size() == (rejected_removed ? 13U : 17U) &&
+    lifecycle.size() == 1, "exact variant/event count");
+  final_fixture_state(server.room()); run.replay_bytes = server.replay().serialize();
+  require(run.replay_bytes.size() <= max_replay_bytes, "artifact byte bound");
+  const auto final = observed_state(server.room()); const auto metrics = server.metrics();
+  auto descriptors = server.owned_descriptors();
+  for (const auto& [role,peer] : peers) { (void)role; descriptors.push_back(peer.tcp->descriptor()); }
+  server.shutdown();
+  for (const auto& [role,peer] : peers) { (void)role; peer.tcp->close(); }
+  for (int fd : descriptors) require(descriptor_closed(fd), "descriptor survived shutdown");
+  auto cleanup = server.cleanup();
+  for (const auto& [key,value] : cleanup.items()) { (void)key; require(value == 0,"active resource survived shutdown"); }
+  require(Fd::live() == descriptors_before, "tracked descriptor leak");
+  cleanup["descriptor_checks"] = descriptors.size(); cleanup["all_descriptors_closed"] = true;
+  cleanup["tracked_descriptor_delta"] = Fd::live()-descriptors_before;
+  run.evidence = Json{{"result","PASS"},{"scenario_id",scenario.at("scenario_id")},{"process_id",::getpid()},
+    {"mode","live"},{"variant",rejected_removed ? "rejected-removed" : "canonical"},{"executed_ticks",200},
+    {"fixture_bootstrap","test-build-only; four real bindings before one normal start evaluation"},
+    {"initial_bindings",bindings},{"admissions",admissions},{"wire_observations",wire},{"lifecycle_events",lifecycle},
+    {"action_errors",actions},{"state_hashes",hashes},{"final_state",final},{"metrics",metrics},{"cleanup",cleanup},
+    {"all_resources_released",true}};
+  return run;
+}
+
+ReplayRun verify_replay(const Json& artifact) {
+  // The CLI's bounded parser has validated completeness and record framing.
+  const int descriptors_before = Fd::live(); Room room; ReplayFixture::offline(room,artifact);
+  ReplayRun run; run.records = Json::array(); Json hashes = Json::array(), actions = Json::array();
+  Json divergence = nullptr;
+  for (const auto& tick : artifact.at("ticks")) {
+    const int before_tick = room.executed_ticks();
+    require(tick.at("tick").is_number_integer() && tick.at("tick") == before_tick && tick.at("events").is_array(),
+      "ordered tick record");
+    for (const auto& event : tick.at("events")) {
+      require(event.at("before_tick").is_number_integer() && event.at("before_tick") == before_tick, "original admission boundary");
+      const auto player_id = event.at("player_id").get<std::string>();
+      require(room.players().contains(player_id), "replay player is in initial mapping");
+      if (event.at("kind") == "INPUT") {
+        require(event.at("owner_epoch").is_number_integer() && event.at("owner_epoch") == 0, "recorded owner epoch");
+        const auto result = admit_input(room,player_id,event);
+        require(!result.error && !result.duplicate, "recorded accepted input failed production admission");
+      } else {
+        require(event.at("kind") == "LEAVE_ROOM" || event.at("kind") == "DISCONNECT", "known lifecycle record");
+        room.leave(room.players().at(player_id).connection_id);
+      }
+    }
+    for (const auto& failure : room.tick()) actions.push_back(Json{{"tick",before_tick},
+      {"player_id",failure.player_id},{"code","ACTION_REJECTED"}});
+    const auto record = canonical_state(room); const auto hash = sha256(record);
+    hashes.push_back(hash); run.records.push_back(tick_record(room,hash));
+    if (tick.at("state_hash") != hash) {
+      divergence = Json{{"first_divergent_tick",before_tick},{"expected_hash",tick.at("state_hash")},
+        {"actual_hash",hash},{"canonical_record",record}};
+      break;
+    }
+  }
+  const auto final = observed_state(room); room.close();
+  require(room.pending_count() == 0 && Fd::live() == descriptors_before, "offline cleanup");
+  for (const auto& [id,player] : room.players()) {
+    (void)id; require(!player.connected && player.pending.empty() && player.input_attempts == 0,"offline player cleanup");
+  }
+  run.evidence = Json{{"result",divergence.is_null() ? "PASS" : "DIVERGED"},{"mode","offline"},{"process_id",::getpid()},
+    {"executed_ticks",room.executed_ticks()},{"state_hashes",hashes},{"final_state",final},{"action_errors",actions},
+    {"divergence",divergence},{"cleanup",Json{{"pending_inputs",room.pending_count()},
+      {"tracked_descriptor_delta",Fd::live()-descriptors_before},{"worker_threads",0},{"timers",0}}},
+    {"all_resources_released",true}};
+  return run;
+}
+}
diff --git a/tests/g07.hpp b/tests/g07.hpp
new file mode 100644
index 0000000..3572c52
--- /dev/null
+++ b/tests/g07.hpp
@@ -0,0 +1,12 @@
+#pragma once
+#include "scenario.hpp"
+
+namespace arena {
+struct ReplayRun {
+  Json evidence;
+  Json records;
+  std::string replay_bytes;
+};
+ReplayRun run_replay_scenario(const Json& scenario, bool rejected_removed);
+ReplayRun verify_replay(const Json& artifact);
+}
diff --git a/tests/scenario_main.cpp b/tests/scenario_main.cpp
new file mode 100644
index 0000000..1198f7e
--- /dev/null
+++ b/tests/scenario_main.cpp
@@ -0,0 +1,70 @@
+#include "g07.hpp"
+#ifndef ARENA_TEST_FIXTURES
+#error Scenario fixture executable requires its separate test core
+#endif
+#include <fstream>
+#include <iostream>
+#include <stdexcept>
+
+namespace {
+std::string read_bytes(const std::filesystem::path& path) {
+  const auto size = std::filesystem::file_size(path);
+  if (size > arena::max_replay_bytes) throw std::length_error("replay input exceeds byte bound");
+  std::ifstream file(path,std::ios::binary);
+  if (!file) throw std::runtime_error("cannot open replay input");
+  std::string bytes(static_cast<std::size_t>(size),'\0');
+  file.read(bytes.data(),static_cast<std::streamsize>(bytes.size()));
+  if (!file || file.peek() != std::char_traits<char>::eof()) throw std::runtime_error("replay input size changed during read");
+  return bytes;
+}
+std::filesystem::path sibling(const std::filesystem::path& evidence, const std::string& suffix) {
+  return evidence.parent_path() / (evidence.stem().string()+suffix);
+}
+}
+int main(int argc, char** argv) {
+  try {
+    if (argc < 4) throw std::invalid_argument("scenario-run FILE OUT [--variant rejected-removed] | replay-verify FILE OUT");
+    const std::string command = argv[1]; const std::filesystem::path input = argv[2], output = argv[3];
+    arena::ReplayRun run;
+    if (command == "scenario-run") {
+      const bool variant = argc == 6 && std::string(argv[4]) == "--variant" && std::string(argv[5]) == "rejected-removed";
+      if (argc != 4 && !variant) throw std::invalid_argument("unknown scenario variant");
+      const auto scenario = arena::read_json_file(input);
+      if (scenario.at("thread") != "G07") {
+        if (variant) throw std::invalid_argument("variant is only active for G07");
+        const auto evidence = arena::run_scenario(scenario);
+        arena::write_json_file(output,evidence);
+        std::cout << arena::Json{{"result",evidence.at("result")},{"executed_ticks",evidence.at("executed_ticks")},
+          {"scenario_id",evidence.at("scenario_id")},{"evidence",output.string()},{"cleanup",evidence.at("cleanup")}}.dump() << '\n';
+        return 0;
+      }
+      run = arena::run_replay_scenario(scenario,variant);
+      const auto replay_path = sibling(output,".replay.json");
+      if (replay_path.has_parent_path()) std::filesystem::create_directories(replay_path.parent_path());
+      std::ofstream file(replay_path,std::ios::binary);
+      if (!file || run.replay_bytes.size() > arena::max_replay_bytes) throw std::runtime_error("bounded replay output unavailable");
+      file << run.replay_bytes; file.close();
+      if (!file) throw std::runtime_error("replay write failed");
+      run.evidence["replay_artifact"] = std::filesystem::absolute(replay_path).string();
+      run.evidence["artifact_bytes"] = std::filesystem::file_size(replay_path);
+      run.evidence["artifact_sha256"] = arena::sha256(run.replay_bytes);
+    } else if (command == "replay-verify" && argc == 4) {
+      const auto bytes = read_bytes(input);
+      run = arena::verify_replay(arena::parse_replay_artifact(bytes));
+      run.evidence["replay_artifact"] = std::filesystem::absolute(input).string();
+      run.evidence["artifact_bytes"] = bytes.size(); run.evidence["artifact_sha256"] = arena::sha256(bytes);
+    } else throw std::invalid_argument("unknown scenario command");
+    const auto records_path = sibling(output,".records.json");
+    arena::write_json_file(records_path,run.records);
+    run.evidence["canonical_records"] = std::filesystem::absolute(records_path).string();
+    arena::write_json_file(output,run.evidence);
+    std::cout << arena::Json{{"result",run.evidence.at("result")},{"process_id",run.evidence.at("process_id")},
+      {"executed_ticks",run.evidence.at("executed_ticks")},{"artifact_bytes",run.evidence.at("artifact_bytes")},
+      {"artifact_sha256",run.evidence.at("artifact_sha256")},{"evidence",output.string()},
+      {"divergence",run.evidence.value("divergence",arena::Json(nullptr))},{"cleanup",run.evidence.at("cleanup")}}.dump() << '\n';
+    return run.evidence.at("result") == "PASS" ? 0 : 1;
+  } catch (const std::exception& error) {
+    std::cerr << arena::Json{{"result","FAIL"},{"message",std::string(error.what()).substr(0,256)}}.dump() << '\n';
+    return 1;
+  }
+}
diff --git a/tests/tests.cpp b/tests/tests.cpp
index 7141dfb..6622039 100644
--- a/tests/tests.cpp
+++ b/tests/tests.cpp
@@ -554,6 +554,68 @@ void pending_bound_after_rate_activation() {
     {"alpha_position",Json::array({player.x,player.y})}}}}.dump() << '\n';
   room.close();
 }
+void canonical_hash_bytes_without_ticks() {
+  check(sha256("") == "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855", "SHA-256 empty vector");
+  check(sha256("abc") == "ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad", "SHA-256 abc vector");
+  Room room; room.create("hash-room");
+  auto& z = room.join("player-z","session-z",1);
+  auto& a = room.join("player-A","session-A",2);
+  const auto maximum = std::numeric_limits<std::uint64_t>::max();
+  check(!room.input(z.id,Input{maximum,std::uint64_t{0},{Direction::east,std::nullopt}}).error,
+        "exact maximum accepted sequence for serialization");
+  const std::string expected =
+    "v=1|room=hash-room|tick=-1|status=RUNNING|owner_epoch=0\n"
+    "player=player-A|slot=1|x=90000|y=90000|dir=STOP|score=0|conn=CONNECTED|last_seq=0|last_tag_tick=-20\n"
+    "player=player-z|slot=0|x=10000|y=10000|dir=STOP|score=0|conn=CONNECTED|last_seq=18446744073709551615|last_tag_tick=-20\n";
+  check(canonical_state(room) == expected, "exact ASCII order, unsigned sequence, signed cooldown and final LF");
+  const auto hash = sha256(expected);
+  check(hash.size() == 64 && sha256(expected.substr(0,expected.size()-1)) != hash, "final LF is hashed");
+  a.session_id = "different-session"; a.connection_id = 99;
+  z.input_attempts = 3;
+  check(canonical_state(room) == expected, "credentials, connections, attempts and pending effects are excluded");
+  check(room.executed_ticks() == 0 && z.applied_seq == std::nullopt && z.direction == Direction::stop,
+        "last_seq is acceptance state without an extra simulation tick");
+  std::cout << Json{{"G07_zero_tick_canonical",Json{{"record",expected},{"hash",hash},{"executed_ticks",0}}}}.dump() << '\n';
+  room.close();
+}
+void replay_storage_and_packaging_without_ticks() {
+  Room room; populate(room); ReplayLog log; log.start(room);
+  const auto initial = log.serialize();
+  check(initial.size() == log.bytes() && initial.back() == '\n' &&
+        parse_replay_artifact(initial).at("executed_ticks") == 0, "bounded complete initial packaging");
+  check(initial.find("session-00") == std::string::npos && initial.find("connection_id") == std::string::npos,
+        "no connection credentials in replay");
+  Json incomplete = log.artifact(); incomplete["complete"] = false;
+  const std::array<std::string,3> invalid{incomplete.dump(),"{",std::string(max_replay_bytes+1,' ')};
+  for (const auto& bytes : invalid) {
+    bool rejected = false;
+    try { (void)parse_replay_artifact(bytes); } catch (const std::exception&) { rejected = true; }
+    check(rejected,"incomplete, malformed and oversized intake fail before any tick");
+  }
+  check(!room.input("player-00",Input{1,std::uint64_t{0},{Direction::east,std::nullopt}}).error,"storage probe seed record");
+  const auto state = canonical_state(room); const auto pending = room.pending_count();
+  // Recorder-only capacity probe: repeatedly append a fixed already-decoded
+  // record. This is not a stream of accepted gameplay inputs or a simulation.
+  std::size_t appends = 0;
+  while (log.complete() && appends <= max_replay_bytes) {
+    log.accepted_input(room,"player-00"); ++appends;
+    check(log.bytes() <= max_replay_bytes && log.high_water_bytes() <= max_replay_bytes,"capture storage byte cap");
+  }
+  check(!log.complete() && !log.failure().empty() && log.pending_events() == 0,
+        "overflow latches incomplete recording and releases unfinished capture work");
+  bool refused = false;
+  try { (void)log.serialize(); } catch (const std::runtime_error&) { refused = true; }
+  check(refused,"incomplete capture can never export a partial success");
+  // With capture disabled this invokes only the hash path, not Room::tick.
+  log.finish_tick(room);
+  check(log.last_state_hash() == sha256(state) && canonical_state(room) == state &&
+    room.pending_count() == pending && room.executed_ticks() == 0,"hashing and gameplay state survive capture exhaustion");
+  const auto high_water = log.high_water_bytes(); log.clear(); room.close();
+  check(log.bytes() == 0 && log.pending_events() == 0 && room.pending_count() == 0,"capture/model cleanup");
+  std::cout << Json{{"G07_zero_tick_storage",Json{{"appends_including_rejected",appends},
+    {"high_water_bytes",high_water},{"bound",max_replay_bytes},{"failure",log.failure()},
+    {"export_refused",refused},{"executed_ticks",0},{"released_bytes",log.bytes()}}}}.dump() << '\n';
+}
 void real_tcp_authority_and_cleanup() {
   const auto scenario = Json::parse(R"({
     "scenario_id":"G01-three-tick-authority-smoke","contract_version":1,"thread":"G01","seed":7050,
@@ -695,7 +757,9 @@ int main(int argc, char** argv) {
       {"G06_four_movement_clamps", four_fixed_movement_clamps},
       {"G06_TAG_range_membership", tag_range_and_membership_edges},
       {"G06_shared_victim_ASCII_order", shared_victim_ascii_order},
-      {"G06_pending_bound_after_rate", pending_bound_after_rate_activation}};
+      {"G06_pending_bound_after_rate", pending_bound_after_rate_activation},
+      {"G07_zero_tick_canonical_SHA256", canonical_hash_bytes_without_ticks},
+      {"G07_zero_tick_storage_packaging", replay_storage_and_packaging_without_ticks}};
   } else if (std::string(argv[1]) == "integration") {
     tests = {{"real_TCP_authority_and_cleanup", real_tcp_authority_and_cleanup}, {"standalone_process_shutdown", [&] {
       standalone_process_shutdown(std::filesystem::absolute(argv[0]).parent_path() / "arena"); }},
diff --git a/track b/track
index 8389177..1015abc 100755
--- a/track
+++ b/track
@@ -30,8 +30,8 @@ case "$command" in
     ;;
   unit-test) run unit "$build_dir/arena_tests" unit ;;
   integration-test) run integration "$build_dir/arena_tests" integration ;;
-  scenario-run) run scenario "$build_dir/arena" scenario-run "$@" ;;
-  replay-verify) run replay "$build_dir/arena" replay-verify "$@" ;;
+  scenario-run) run scenario "$build_dir/arena_scenarios" scenario-run "$@" ;;
+  replay-verify) run replay "$build_dir/arena_scenarios" replay-verify "$@" ;;
   server) run server "$build_dir/arena" server "$@" ;;
   *) printf 'unknown command: %s\n' "$command" >&2; exit 2 ;;
 esac
