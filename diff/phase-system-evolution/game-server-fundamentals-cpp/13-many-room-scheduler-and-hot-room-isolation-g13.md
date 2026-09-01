# Many-room Scheduler와 Hot-room Isolation (G13)

## `feat: isolate per-Room scheduling and overload cleanup`

diff --git a/CMakeLists.txt b/CMakeLists.txt
index 5a056ed..4dd493d 100644
--- a/CMakeLists.txt
+++ b/CMakeLists.txt
@@ -34,7 +34,7 @@ target_compile_options(arena PRIVATE -Wall -Wextra -Wpedantic -Werror)
 add_executable(arena_tests tests/tests.cpp tests/g09.cpp tests/g12_queue.cpp)
 target_link_libraries(arena_tests PRIVATE arena_test_core)
 target_compile_options(arena_tests PRIVATE -Wall -Wextra -Wpedantic -Werror)
-add_executable(arena_scenarios tests/scenario_main.cpp tests/g07.cpp tests/g09.cpp tests/g10.cpp tests/g11.cpp tests/g12.cpp tests/g12_queue.cpp)
+add_executable(arena_scenarios tests/scenario_main.cpp tests/g07.cpp tests/g09.cpp tests/g10.cpp tests/g11.cpp tests/g12.cpp tests/g12_queue.cpp tests/g13.cpp)
 target_link_libraries(arena_scenarios PRIVATE arena_test_core)
 target_compile_options(arena_scenarios PRIVATE -Wall -Wextra -Wpedantic -Werror)
 enable_testing()
diff --git a/evidence/G13-runs.jsonl b/evidence/G13-runs.jsonl
new file mode 100644
index 0000000..2196153
--- /dev/null
+++ b/evidence/G13-runs.jsonl
@@ -0,0 +1,38 @@
+{"label":"baseline-compile","category":"compile","units":1,"ticks":0,"ceiling_seconds":180,"argv":["clang++","-std=c++20","-O2","-Wall","-Wextra","-Wpedantic","-Werror","-fsanitize=thread","-g","-DARENA_TEST_FIXTURES=1","-I","src","-I","tests","-I","/opt/homebrew/include","artifacts/g13/reproduce.cpp","src/game.cpp","src/transport.cpp","src/replay.cpp","src/snapshot.cpp","-o","artifacts/g13/reproduce"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/baseline-compile.log","started_at":"2026-08-28T08:30:30.552464+00:00","duration_seconds":20.794857,"exit":0,"wrapper_pid":76255,"child_pid":76269,"timed_out":false}
+{"label":"baseline","category":"unit","units":1,"ticks":0,"ceiling_seconds":120,"argv":["env","TSAN_OPTIONS=halt_on_error=1","./artifacts/g13/reproduce","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G13.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/baseline.json"],"expected_exit":1,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/baseline.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/baseline.json","started_at":"2026-08-28T08:31:46.709102+00:00","duration_seconds":0.894833,"exit":1,"wrapper_pid":78770,"child_pid":78793,"timed_out":false,"observed_ticks":0,"runtime_pid":78793}
+{"label":"build","category":"compile","units":2,"ticks":0,"ceiling_seconds":180,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","ARENA_TSAN=ON","./track","build"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/build.log","started_at":"2026-08-28T09:03:24.142119+00:00","duration_seconds":55.470444,"exit":2,"wrapper_pid":10569,"child_pid":10580,"timed_out":false}
+{"label":"build-retry-1","category":"compile","units":1,"ticks":0,"ceiling_seconds":180,"argv":["cmake","--build","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","--parallel","2"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/build-retry-1.log","started_at":"2026-08-28T09:05:15.077757+00:00","duration_seconds":6.07377,"exit":0,"wrapper_pid":12721,"child_pid":12730,"timed_out":false}
+{"label":"unit","category":"unit","units":1,"ticks":0,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","unit-test"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/unit.log","started_at":"2026-08-28T09:06:24.599185+00:00","duration_seconds":3.382804,"exit":0,"wrapper_pid":14044,"child_pid":14045,"timed_out":false}
+{"label":"integration","category":"integration","units":1,"ticks":0,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","integration-test"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/integration.log","started_at":"2026-08-28T09:06:28.015229+00:00","duration_seconds":1.619388,"exit":0,"wrapper_pid":14117,"child_pid":14118,"timed_out":false}
+{"label":"canonical","category":"canonical","units":1,"ticks":795,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","scenario-run","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G13.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.json","started_at":"2026-08-28T09:06:29.674551+00:00","duration_seconds":6.78268,"exit":0,"wrapper_pid":14149,"child_pid":14150,"timed_out":false,"observed_ticks":795,"runtime_pid":14156}
+{"label":"reference-room-01","category":"reference","units":1,"ticks":25,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.room-01.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-01.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference-room-01.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-01.json","started_at":"2026-08-28T09:06:36.504873+00:00","duration_seconds":0.404208,"exit":0,"wrapper_pid":14305,"child_pid":14306,"timed_out":false,"observed_ticks":25,"runtime_pid":14312}
+{"label":"reference-room-02","category":"reference","units":1,"ticks":25,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.room-02.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-02.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference-room-02.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-02.json","started_at":"2026-08-28T09:06:36.948131+00:00","duration_seconds":0.342862,"exit":0,"wrapper_pid":14315,"child_pid":14316,"timed_out":false,"observed_ticks":25,"runtime_pid":14322}
+{"label":"reference-room-03","category":"reference","units":1,"ticks":25,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.room-03.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-03.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference-room-03.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-03.json","started_at":"2026-08-28T09:06:37.325448+00:00","duration_seconds":0.344131,"exit":0,"wrapper_pid":14329,"child_pid":14330,"timed_out":false,"observed_ticks":25,"runtime_pid":14336}
+{"label":"reference-room-04","category":"reference","units":1,"ticks":25,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.room-04.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-04.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference-room-04.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-04.json","started_at":"2026-08-28T09:06:37.704623+00:00","duration_seconds":0.347042,"exit":0,"wrapper_pid":14339,"child_pid":14340,"timed_out":false,"observed_ticks":25,"runtime_pid":14346}
+{"label":"reference-room-05","category":"reference","units":1,"ticks":25,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.room-05.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-05.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference-room-05.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-05.json","started_at":"2026-08-28T09:06:38.086833+00:00","duration_seconds":0.406906,"exit":0,"wrapper_pid":14349,"child_pid":14350,"timed_out":false,"observed_ticks":25,"runtime_pid":14384}
+{"label":"reference-room-06","category":"reference","units":1,"ticks":25,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.room-06.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-06.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference-room-06.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-06.json","started_at":"2026-08-28T09:06:38.617850+00:00","duration_seconds":0.40985,"exit":0,"wrapper_pid":14387,"child_pid":14388,"timed_out":false,"observed_ticks":25,"runtime_pid":14394}
+{"label":"reference-room-07","category":"reference","units":1,"ticks":25,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.room-07.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-07.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference-room-07.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-07.json","started_at":"2026-08-28T09:06:39.066761+00:00","duration_seconds":0.350107,"exit":0,"wrapper_pid":14397,"child_pid":14399,"timed_out":false,"observed_ticks":25,"runtime_pid":14405}
+{"label":"reference-room-08","category":"reference","units":1,"ticks":25,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.room-08.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-08.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference-room-08.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-08.json","started_at":"2026-08-28T09:06:39.453645+00:00","duration_seconds":0.348058,"exit":0,"wrapper_pid":14414,"child_pid":14415,"timed_out":false,"observed_ticks":25,"runtime_pid":14421}
+{"label":"reference-room-09","category":"reference","units":1,"ticks":25,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.room-09.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-09.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference-room-09.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-09.json","started_at":"2026-08-28T09:06:39.838867+00:00","duration_seconds":0.394001,"exit":0,"wrapper_pid":14424,"child_pid":14425,"timed_out":false,"observed_ticks":25,"runtime_pid":14431}
+{"label":"reference-room-10","category":"reference","units":1,"ticks":25,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.room-10.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-10.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference-room-10.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-10.json","started_at":"2026-08-28T09:06:40.270241+00:00","duration_seconds":0.344959,"exit":0,"wrapper_pid":14462,"child_pid":14463,"timed_out":false,"observed_ticks":25,"runtime_pid":14469}
+{"label":"reference-room-11","category":"reference","units":1,"ticks":25,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.room-11.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-11.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference-room-11.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-11.json","started_at":"2026-08-28T09:06:40.650125+00:00","duration_seconds":0.350089,"exit":0,"wrapper_pid":14472,"child_pid":14473,"timed_out":false,"observed_ticks":25,"runtime_pid":14479}
+{"label":"reference-room-12","category":"reference","units":1,"ticks":25,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.room-12.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-12.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference-room-12.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-12.json","started_at":"2026-08-28T09:06:41.035129+00:00","duration_seconds":0.346609,"exit":0,"wrapper_pid":14482,"child_pid":14483,"timed_out":false,"observed_ticks":25,"runtime_pid":14489}
+{"label":"reference-room-13","category":"reference","units":1,"ticks":25,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.room-13.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-13.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference-room-13.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-13.json","started_at":"2026-08-28T09:06:41.418958+00:00","duration_seconds":0.393797,"exit":0,"wrapper_pid":14494,"child_pid":14499,"timed_out":false,"observed_ticks":25,"runtime_pid":14507}
+{"label":"reference-room-14","category":"reference","units":1,"ticks":25,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.room-14.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-14.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference-room-14.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-14.json","started_at":"2026-08-28T09:06:41.850970+00:00","duration_seconds":0.397108,"exit":0,"wrapper_pid":14517,"child_pid":14518,"timed_out":false,"observed_ticks":25,"runtime_pid":14524}
+{"label":"reference-room-15","category":"reference","units":1,"ticks":25,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.room-15.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-15.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference-room-15.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-15.json","started_at":"2026-08-28T09:06:42.288675+00:00","duration_seconds":0.404305,"exit":0,"wrapper_pid":14555,"child_pid":14556,"timed_out":false,"observed_ticks":25,"runtime_pid":14562}
+{"label":"reference-room-16","category":"reference","units":1,"ticks":25,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.room-16.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-16.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference-room-16.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-16.json","started_at":"2026-08-28T09:06:42.729251+00:00","duration_seconds":0.394503,"exit":0,"wrapper_pid":14572,"child_pid":14573,"timed_out":false,"observed_ticks":25,"runtime_pid":14579}
+{"label":"reference-room-17","category":"reference","units":1,"ticks":25,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.room-17.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-17.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference-room-17.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-17.json","started_at":"2026-08-28T09:06:43.158595+00:00","duration_seconds":0.415043,"exit":0,"wrapper_pid":14582,"child_pid":14583,"timed_out":false,"observed_ticks":25,"runtime_pid":14589}
+{"label":"reference-room-18","category":"reference","units":1,"ticks":25,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.room-18.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-18.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference-room-18.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-18.json","started_at":"2026-08-28T09:06:43.611921+00:00","duration_seconds":0.394877,"exit":0,"wrapper_pid":14592,"child_pid":14593,"timed_out":false,"observed_ticks":25,"runtime_pid":14599}
+{"label":"reference-room-19","category":"reference","units":1,"ticks":25,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.room-19.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-19.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference-room-19.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-19.json","started_at":"2026-08-28T09:06:44.046984+00:00","duration_seconds":0.406202,"exit":0,"wrapper_pid":14629,"child_pid":14630,"timed_out":false,"observed_ticks":25,"runtime_pid":14636}
+{"label":"reference-room-20","category":"reference","units":1,"ticks":25,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.room-20.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-20.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference-room-20.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-20.json","started_at":"2026-08-28T09:06:44.490092+00:00","duration_seconds":0.394949,"exit":0,"wrapper_pid":14672,"child_pid":14673,"timed_out":false,"observed_ticks":25,"runtime_pid":14679}
+{"label":"reference-room-21","category":"reference","units":1,"ticks":25,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.room-21.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-21.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference-room-21.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-21.json","started_at":"2026-08-28T09:06:44.923067+00:00","duration_seconds":0.392044,"exit":0,"wrapper_pid":14683,"child_pid":14684,"timed_out":false,"observed_ticks":25,"runtime_pid":14690}
+{"label":"reference-room-22","category":"reference","units":1,"ticks":25,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.room-22.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-22.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference-room-22.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-22.json","started_at":"2026-08-28T09:06:45.356353+00:00","duration_seconds":0.406133,"exit":0,"wrapper_pid":14696,"child_pid":14697,"timed_out":false,"observed_ticks":25,"runtime_pid":14703}
+{"label":"reference-room-23","category":"reference","units":1,"ticks":25,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.room-23.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-23.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference-room-23.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-23.json","started_at":"2026-08-28T09:06:45.800386+00:00","duration_seconds":0.394697,"exit":0,"wrapper_pid":14706,"child_pid":14707,"timed_out":false,"observed_ticks":25,"runtime_pid":14713}
+{"label":"reference-room-24","category":"reference","units":1,"ticks":25,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.room-24.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-24.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference-room-24.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-24.json","started_at":"2026-08-28T09:06:46.238944+00:00","duration_seconds":0.405375,"exit":0,"wrapper_pid":14745,"child_pid":14774,"timed_out":false,"observed_ticks":25,"runtime_pid":14780}
+{"label":"reference-room-25","category":"reference","units":1,"ticks":25,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.room-25.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-25.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference-room-25.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-25.json","started_at":"2026-08-28T09:06:46.682944+00:00","duration_seconds":0.34511,"exit":0,"wrapper_pid":14786,"child_pid":14787,"timed_out":false,"observed_ticks":25,"runtime_pid":14793}
+{"label":"reference-room-26","category":"reference","units":1,"ticks":25,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.room-26.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-26.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference-room-26.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-26.json","started_at":"2026-08-28T09:06:47.061463+00:00","duration_seconds":0.396502,"exit":0,"wrapper_pid":14796,"child_pid":14797,"timed_out":false,"observed_ticks":25,"runtime_pid":14803}
+{"label":"reference-room-27","category":"reference","units":1,"ticks":25,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.room-27.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-27.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference-room-27.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-27.json","started_at":"2026-08-28T09:06:47.497174+00:00","duration_seconds":0.3474,"exit":0,"wrapper_pid":14808,"child_pid":14809,"timed_out":false,"observed_ticks":25,"runtime_pid":14815}
+{"label":"reference-room-28","category":"reference","units":1,"ticks":25,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.room-28.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-28.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference-room-28.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-28.json","started_at":"2026-08-28T09:06:47.879455+00:00","duration_seconds":0.345075,"exit":0,"wrapper_pid":14818,"child_pid":14819,"timed_out":false,"observed_ticks":25,"runtime_pid":14825}
+{"label":"reference-room-29","category":"reference","units":1,"ticks":25,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.room-29.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-29.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference-room-29.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-29.json","started_at":"2026-08-28T09:06:48.260161+00:00","duration_seconds":0.396593,"exit":0,"wrapper_pid":14828,"child_pid":14857,"timed_out":false,"observed_ticks":25,"runtime_pid":14863}
+{"label":"reference-room-30","category":"reference","units":1,"ticks":25,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.room-30.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-30.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference-room-30.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-30.json","started_at":"2026-08-28T09:06:48.696204+00:00","duration_seconds":0.399934,"exit":0,"wrapper_pid":14866,"child_pid":14867,"timed_out":false,"observed_ticks":25,"runtime_pid":14873}
+{"label":"reference-room-31","category":"reference","units":1,"ticks":25,"ceiling_seconds":120,"argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g13-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/track","TSAN_OPTIONS=halt_on_error=1","./track","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/canonical.room-31.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-31.json"],"expected_exit":0,"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference-room-31.log","result":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g13/reference.room-31.json","started_at":"2026-08-28T09:06:49.133996+00:00","duration_seconds":0.344461,"exit":0,"wrapper_pid":14876,"child_pid":14877,"timed_out":false,"observed_ticks":25,"runtime_pid":14883}
diff --git a/evidence/G13.md b/evidence/G13.md
new file mode 100644
index 0000000..0422579
--- /dev/null
+++ b/evidence/G13.md
@@ -0,0 +1,48 @@
+# G13 — Room scheduling and isolation
+
+- Branch: `track/fundamentals-cpp`; initial attempt, repairs0/2.
+- START: `8bb15c64eb22cf174dc63ffd9248d2ab386841c2`.
+- Profile: `realtime-core`; Spec-Revision: `c1d62196ab76b55652f5d75a67514f8c6d8210ce`.
+- Frozen G13 fixture SHA-256: `ad6304b939483b4498a884ebe62681d1d3bea8810bb2518642c5f5efc9b1e70b`.
+
+## Reproduction
+
+The unchanged G12 server hosted one normally joined/UDP-bound four-player Room, then rejected a fifth TCP/HELLO client's actual second CREATE_ROOM with ROOM_NOT_JOINABLE. The first Room's canonical state stayed identical; downstream32-Room phases were unavailable and zero ticks ran. All17 descriptors closed. The fifth client had only its TCP session, not a UDP binding.17 source/fixture files were byte-identical to START after the baseline. No production change preceded it.
+
+Frozen helper, source manifest, command plans and baseline raw output: `artifacts/g13/reproduce.cpp`, `source-manifest.json`, `baseline-preservation.json`, `commands.before-compile-retry.json`, `baseline.json`. Baseline SHA-256: `329ce35b49e56c63faf29b0653b21c3ca892eb941601e91ea1681da50d819df8`.
+
+## Change
+
+The one Server owns bounded Room contexts: model, accepted journal, resume records, fixed accumulator and deadline state. Connection identity routes input, snapshots, reconnect and terminal output to its Room. One shared monotonic observation services each scheduled Room once per iteration, at most4 ticks; only executed ticks advance each Room's simulation counter. The default counter and original G04 assertions remain unchanged. Staged transport disconnects reach their owner before the next tick.
+
+After20 unrecovered deadlines, only the affected Room closes ROOM_OVERLOAD, clears pending/grace/journal/transport resources, and leaves the scheduler/timer registry. Its bounded closed model remains inspectable until shutdown, which empties the Room registry. Generated R14/P8 identifiers remain within the existing1200-byte maximum-player serializer probe; credentials and accepted1..64-byte IDs are unchanged. New fixture holds/identity injection/observers exist only in the test build.
+
+## Actual verification
+
+`evidence/G13-runs.jsonl` contains every resolved argv, raw log path, process, exit and duration. `artifacts/g13/commands.json` preserves the command plan and the exact compilation retry; `verify.py` ran the reserved post-build commands sequentially, stopping on failure.
+
+| Budget | Used | Result |
+| --- | ---: | --- |
+| Compile | 4/8 | baseline1; configure+build2 failed on two new test string/JSON comparisons; explicit string extraction, build-only retry1 passed |
+| Unit | 2/4 | unchanged baseline exit1; full27 tests exit0, including preserved64/65 and new zero-tick aggregate guard |
+| Integration | 1/2 | full4 tests exit0, including the existing11-case UDP matrix |
+| Post live | 1 |795 actual ticks:775 normal +20 hot; exit0,6.783s |
+| Offline reference |31 |31 distinct processes ×25 ticks=775; all exit0; every canonical byte record/hash matches live |
+| Fault / load / profiler |0 /0 /0 | none |
+
+All three built executables link TSan. No post-runtime failure, extra campaign, source edit after verification, or hidden G13 live run inside a suite occurred.
+
+Room0 served4/8/12/16/20 cumulative ticks at225/450/675/900/1125ms, retaining25/50/75/100/125ms. Miss streaks were0/5/9/14/18. OVERLOADED first appeared450ms. At1200ms its20th unrecovered deadline closed only Room0, cleared16 accepted pending entries and four server sockets, and removed its scheduler/deadline entry. Last closed tick19 has last_seq260; the distinct pre-terminal between-tick state has last_seq324.96 admissions and1440 INPUT_RATE_EXCEEDED responses were actually consumed across the frozen96 transport groups. Each normal Room executed25 ticks and remained RUNNING through1250ms.
+
+Observed high waters: Rooms32/64, connections128/512, accepted pending/player4/64, total pending140/32768, mailbox16/512, control/player4/64, owned buffers16/1440 bytes. UDP ingress/outbound maxima240/714 bytes. All6420 received datagrams,4980 sent datagrams and1732 TCP replies are counted. All6712 owned buffers were released; all387 descriptor checks passed; all final cleanup/registry/deadline fields are zero. The original pure64/65 queue test remains unchanged and passes separately.
+
+## Raw artifacts
+
+Under `artifacts/g13/`: `canonical.json` (setup, actual owner mapping, clock/service traces, per-Room hashes, terminal/resources); `canonical.hot-inputs.json` (every actual hot request/response); `canonical.room-NN.records.json` (all795 scalar/canonical tick records, normal admissions and received snapshots); normal `canonical.room-NN.replay.json`; `reference.room-NN.json` and `.records.json` forNN01..31. `checks.json` records all775 comparisons, file hashes and compact resource/budget checks. No hot partial journal is exported as complete.
+
+- Canonical SHA-256: `57525f60eae3891feea37b5a4a8a6558a058a7ac9cc9eb8dd18d75feace3363b`.
+- Hot-input raw SHA-256: `c0d4618b39412f60b5b96c462b3b9ebd13510a91d7297c2da64eda99d0ff8920`.
+- Normal hash-array map SHA-256 (sorted compact JSON): `af43735164dc41dcd24c0eb794054f30aa369220ad0d4f45432c5c99329090e3`.
+- Hot tick19 hash: `aae1d77ffb90207df11cb2c13b140c94bcb66423d7a01860e7f813bf42be97ad`.
+
+Unresolved worker findings: none. Parent independent/cross-track acceptance remains a separate gate. No tags, push, main/spec/index changes or G14 work.
diff --git a/src/game.cpp b/src/game.cpp
index 5a8f969..554fd74 100644
--- a/src/game.cpp
+++ b/src/game.cpp
@@ -15,11 +15,11 @@ void FixedTickAccumulator::reset(std::int64_t monotonic_ms) {
   previous_ms_ = monotonic_ms;
   remaining_ms_ = 0;
 }
-TickBatch FixedTickAccumulator::advance(std::int64_t monotonic_ms) {
+TickBatch FixedTickAccumulator::advance(std::int64_t monotonic_ms, bool service_ready) {
   if (monotonic_ms < previous_ms_) throw std::logic_error("monotonic clock moved backwards");
   remaining_ms_ += monotonic_ms - previous_ms_;
   previous_ms_ = monotonic_ms;
-  const auto ticks = static_cast<int>(std::min<std::int64_t>(remaining_ms_ / tick_duration_ms, max_catch_up_ticks));
+  const auto ticks = service_ready ? static_cast<int>(std::min<std::int64_t>(remaining_ms_ / tick_duration_ms, max_catch_up_ticks)) : 0;
   remaining_ms_ -= static_cast<std::int64_t>(ticks) * tick_duration_ms;
   return {ticks, remaining_ms_, remaining_ms_ >= tick_duration_ms};
 }
@@ -106,7 +106,7 @@ std::optional<std::string> Room::begin_input_attempt(const std::string& player_i
   input_attempt_high_water_ = std::max(input_attempt_high_water_, attempts);
   return std::nullopt;
 }
-InputResult Room::input(const std::string& player_id, Input input) {
+InputResult Room::input(const std::string& player_id, Input input, bool instance_has_capacity) {
   assert_owner();
   if (status_ != "RUNNING") return {"ROOM_NOT_RUNNING", false};
   auto found = players_.find(player_id);
@@ -126,6 +126,8 @@ InputResult Room::input(const std::string& player_id, Input input) {
   if (tick == nullptr || *tick < next_tick) return {"INPUT_LATE", false};
   if (*tick > next_tick + max_future_input_ticks) return {"INPUT_TOO_EARLY", false};
   auto& queue = player.pending;
+  // A retry has already returned above and never consumes new storage.
+  if (!instance_has_capacity) return {"ADMISSION_REJECTED", false};
   if (queue.size() == max_pending_inputs) return {"INPUT_QUEUE_FULL", false};
   queue.push_back(input);
   player.last_accepted_input = std::move(input);
diff --git a/src/game.hpp b/src/game.hpp
index e99aef5..4919fb9 100644
--- a/src/game.hpp
+++ b/src/game.hpp
@@ -15,13 +15,16 @@ inline constexpr std::size_t max_frame_bytes = 16'384;
 inline constexpr std::size_t max_datagram_bytes = 1'200;
 inline constexpr std::int64_t udp_bind_token_ttl_ms = 5'000;
 inline constexpr std::size_t max_connections = 512;
+inline constexpr std::size_t max_rooms = 64;
 inline constexpr std::size_t max_players = 8;
 inline constexpr std::size_t max_pending_inputs = 64;
+inline constexpr std::size_t max_total_pending_inputs = max_rooms * max_players * max_pending_inputs;
 inline constexpr std::size_t max_control_messages = 64;
 inline constexpr std::size_t max_mailbox_messages = max_players * max_pending_inputs;
 inline constexpr int session_ticks = 1'200;
 inline constexpr int tick_duration_ms = 50;
 inline constexpr int max_catch_up_ticks = 4;
+inline constexpr int max_unrecovered_deadlines = 20;
 inline constexpr std::uint64_t max_future_input_ticks = 4;
 inline constexpr std::size_t max_input_attempts_per_tick = 4;
 inline constexpr int reconnect_grace_ticks = 200;
@@ -82,7 +85,7 @@ class Room {
   Player& join(std::string player_id, std::string session_id, std::uint64_t connection_id);
   void bind_realtime(std::uint64_t connection_id);
   std::optional<std::string> begin_input_attempt(const std::string& player_id);
-  InputResult input(const std::string& player_id, Input input);
+  InputResult input(const std::string& player_id, Input input, bool instance_has_capacity = true);
   std::vector<ActionFailure> tick();
   void leave(std::uint64_t connection_id);
   void disconnect(std::uint64_t connection_id);
@@ -102,6 +105,7 @@ class Room {
 #ifdef ARENA_TEST_FIXTURES
   friend struct ReplayFixture;
   friend struct UdpFixture;
+  friend struct IsolationFixture;
 #endif
   void assert_owner() const;
   void evaluate_start_condition();
@@ -132,7 +136,7 @@ struct TickBatch {
 class FixedTickAccumulator {
  public:
   void reset(std::int64_t monotonic_ms);
-  TickBatch advance(std::int64_t monotonic_ms);
+  TickBatch advance(std::int64_t monotonic_ms, bool service_ready = true);
   std::int64_t remaining_ms() const { return remaining_ms_; }
  private:
   std::int64_t previous_ms_ = 0;
diff --git a/src/transport.cpp b/src/transport.cpp
index 23a2f70..00f5ac9 100644
--- a/src/transport.cpp
+++ b/src/transport.cpp
@@ -114,10 +114,10 @@ Input decode_input(const Json& value) {
   // fields are ignored. Only these typed logical fields define a retry.
   return input;
 }
-InputResult admit_input(Room& room, const std::string& player_id, const Json& value) {
+InputResult admit_input(Room& room, const std::string& player_id, const Json& value, bool instance_has_capacity) {
   if (const auto error = room.begin_input_attempt(player_id)) return {error, false};
   try {
-    return room.input(player_id, decode_input(value));
+    return room.input(player_id, decode_input(value), instance_has_capacity);
   } catch (const Json::exception&) {
     return {"MESSAGE_INVALID", false};
   } catch (const std::invalid_argument&) {
@@ -286,17 +286,48 @@ Server::Connection* Server::connection(std::uint64_t id) {
   for (auto& [fd, conn] : connections_) { (void)fd; if (conn.id == id) return &conn; }
   return nullptr;
 }
+Server::RoomContext* Server::context(const std::string& id) {
+  const auto found = rooms_.find(id);
+  return found == rooms_.end() ? nullptr : found->second.get();
+}
+Server::RoomContext* Server::allocate_room(const std::string& id) {
+  if (rooms_.size() >= max_rooms || rooms_.contains(id)) return nullptr;
+  auto owned = std::make_unique<RoomContext>(rooms_.empty() ? &clock_ : nullptr);
+  auto* result = owned.get();
+  rooms_.emplace(id,std::move(owned));
+  if (first_room_id_.empty()) first_room_id_ = id;
+  room_high_water_ = std::max(room_high_water_,rooms_.size());
+  return result;
+}
+const Room& Server::room() const {
+  const auto found = rooms_.find(first_room_id_);
+  return found == rooms_.end() ? retired_room_ : found->second->room;
+}
+const Room& Server::room(const std::string& id) const { return rooms_.at(id)->room; }
+const ReplayLog& Server::replay() const {
+  const auto found = rooms_.find(first_room_id_);
+  return found == rooms_.end() ? retired_replay_ : found->second->replay;
+}
+const ReplayLog& Server::replay(const std::string& id) const { return rooms_.at(id)->replay; }
+std::size_t Server::pending_inputs() const {
+  std::size_t count = 0;
+  for (const auto& [id, owned] : rooms_) { (void)id; count += owned->room.pending_count(); }
+  return count;
+}
 std::string Server::new_id(const std::string& prefix, std::uint64_t number) const {
 #ifdef ARENA_TEST_FIXTURES
+  if (prefix == "room" && !fixture_room_ids_.empty()) return fixture_room_ids_.at(static_cast<std::size_t>(number-1));
   if (prefix == "room" && fixture_room_id_) return *fixture_room_id_;
   if (prefix == "player" && !fixture_player_ids_.empty()) return fixture_player_ids_.at(static_cast<std::size_t>(number-1));
 #endif
-  // Public Room identity needs three fewer bytes for the reachable G11
-  // seven-DISCONNECTED full. Session IDs and 128-bit credentials are unchanged.
-  // Player identity is room-scoped; at most8 joins need one suffix digit.
-  // R14/P8 keeps the seven-DISCONNECTED max8 FULL within1200 bytes.
-  if (prefix == "room") return "r"+nonce_.substr(0,13);
-  if (prefix == "player") return "p"+nonce_.substr(0,6)+std::to_string(number);
+  // Fixed-width counters distinguish64 Rooms and their512 players without
+  // enlarging the R14/P8 seven-DISCONNECTED FULL. Credentials stay128-bit.
+  if (prefix == "room" || prefix == "player") {
+    std::ostringstream out;
+    out << (prefix == "room" ? "r"+nonce_.substr(0,11) : "p"+nonce_.substr(0,4))
+        << std::hex << std::setfill('0') << std::setw(prefix == "room" ? 2 : 3) << number;
+    return out.str();
+  }
   std::ostringstream out;
   out << prefix << '-' << nonce_ << '-' << std::setw(10) << std::setfill('0') << number;
   return out.str();
@@ -311,7 +342,7 @@ void Server::accept_ready() {
     socket_options(fd.get());
     const int raw = fd.get();
     const auto id = next_connection_++;
-    connections_.emplace(raw, Connection{std::move(fd), id, {}, {}, {}, 0, {}, {}});
+    connections_.emplace(raw, Connection{std::move(fd), id, {}, {}, {}, {}, 0, {}, {}});
     register_event(raw, EVFILT_READ, EV_ADD, id);
     register_event(raw, EVFILT_WRITE, EV_ADD | EV_DISABLE, id);
     connection_high_water_ = std::max(connection_high_water_, connections_.size());
@@ -447,10 +478,12 @@ void Server::queue(std::uint64_t connection_id, Json value) {
   register_event(conn->fd.get(), EVFILT_WRITE, EV_ENABLE, conn->id);
   if (terminal) { const int fd = conn->fd.get(); write_ready(fd); disconnect(fd,"CONTROL_BACKPRESSURE"); }
 }
-void Server::broadcast(const Json& value) {
+void Server::broadcast(const std::string& room_id, const Json& value) {
   std::vector<std::uint64_t> ids;
   ids.reserve(connections_.size());
-  for (const auto& [fd, conn] : connections_) { (void)fd; if (!conn.player_id.empty()) ids.push_back(conn.id); }
+  for (const auto& [fd, conn] : connections_) {
+    (void)fd; if (conn.room_id == room_id && !conn.player_id.empty()) ids.push_back(conn.id);
+  }
   for (auto id : ids) queue(id, value);
 }
 void Server::read_datagrams() {
@@ -574,26 +607,31 @@ void Server::bind_datagram(Connection& conn, const Envelope& envelope) {
   conn.udp_endpoint = envelope.udp_endpoint; conn.bind_token.clear();
   auto reply = message("UDP_BOUND"); reply["session_id"] = conn.session_id; reply["owner_epoch"] = 0;
   send_realtime(conn.id,reply);
-  const auto before = room_.status(); room_.bind_realtime(conn.id);
-  if (before == "LOBBY" && room_.status() == "RUNNING") start_room();
+  auto* owned = context(conn.room_id);
+  if (!owned) return; // A valid UDP binding may precede the normal Room join.
+  const auto before = owned->room.status(); owned->room.bind_realtime(conn.id);
+  if (before == "LOBBY" && owned->room.status() == "RUNNING") start_room(*owned);
   else if (conn.full_after_bind) {
     // Connectivity changed between ticks; the cached completed-tick hash is
     // historical. Capture current STOP/CONNECTED state without rewriting it.
-    publish_snapshot(conn,sha256(canonical_state(room_)));
+    publish_snapshot(conn,sha256(canonical_state(owned->room)));
   }
   conn.full_after_bind = false;
 }
-void Server::start_room() {
-  replay_.start(room_);
-  accumulator_.reset(read_monotonic());
-  publish_snapshots(sha256(canonical_state(room_)));
+void Server::start_room(RoomContext& owned) {
+  owned.replay.start(owned.room);
+  const auto now = read_monotonic();
+  owned.accumulator.reset(now); owned.next_deadline_ms = now+tick_duration_ms;
+  scheduled_rooms_.insert(owned.room.id());
+  publish_snapshots(owned,sha256(canonical_state(owned.room)));
 }
-void Server::publish_snapshots(const std::string& state_hash) {
+void Server::publish_snapshots(RoomContext& owned, const std::string& state_hash) {
   std::vector<std::uint64_t> ids;
   for (const auto& [fd, conn] : connections_) {
     (void)fd;
-    const auto player = room_.players().find(conn.player_id);
-    if (player != room_.players().end() && player->second.connected && player->second.connection_id == conn.id &&
+    if (conn.room_id != owned.room.id()) continue;
+    const auto player = owned.room.players().find(conn.player_id);
+    if (player != owned.room.players().end() && player->second.connected && player->second.connection_id == conn.id &&
         player->second.realtime_ready && conn.udp_endpoint) ids.push_back(conn.id);
   }
   for (const auto id : ids) {
@@ -603,9 +641,11 @@ void Server::publish_snapshots(const std::string& state_hash) {
   }
 }
 void Server::publish_snapshot(Connection& conn, const std::string& state_hash) {
-  auto snapshot = conn.snapshots.publish(room_,state_hash);
+  auto* owned = context(conn.room_id);
+  if (!owned) return;
+  auto snapshot = conn.snapshots.publish(owned->room,state_hash);
   snapshot_retention_high_water_ = std::max(snapshot_retention_high_water_,conn.snapshots.high_water());
-  if (const auto record = resume_.find(conn.player_id); record != resume_.end())
+  if (const auto record = owned->resume.find(conn.player_id); record != owned->resume.end())
     record->second.last_snapshot_seq = conn.snapshots.last_sequence();
   send_realtime(conn.id,snapshot);
 }
@@ -613,32 +653,33 @@ void Server::reconnect(Connection& conn, const Json& value) {
   const auto reject = [&](const std::string& code) {
     ++errors_[code]; queue(conn.id,error_message(code,"resume identity, credential or grace invalid"));
   };
-  if (conn.session_id.empty() || !conn.player_id.empty() || value.at("room_id").get<std::string>() != room_.id()) {
+  auto* owned = context(value.at("room_id").get<std::string>());
+  if (!owned || conn.session_id.empty() || !conn.player_id.empty()) {
     reject("RECONNECT_INVALID"); return;
   }
   const auto session = value.at("session_id").get<std::string>();
-  const auto player = std::find_if(room_.players().begin(),room_.players().end(),[&](const auto& item) {
+  const auto player = std::find_if(owned->room.players().begin(),owned->room.players().end(),[&](const auto& item) {
     return item.second.session_id == session;
   });
-  if (player == room_.players().end()) { reject("RECONNECT_INVALID"); return; }
-  const auto saved = resume_.find(player->first);
-  if (saved == resume_.end() || saved->second.token.empty() || value.at("resume_token").get<std::string>() != saved->second.token) {
+  if (player == owned->room.players().end()) { reject("RECONNECT_INVALID"); return; }
+  const auto saved = owned->resume.find(player->first);
+  if (saved == owned->resume.end() || saved->second.token.empty() || value.at("resume_token").get<std::string>() != saved->second.token) {
     reject("RECONNECT_INVALID"); return;
   }
   if (player->second.connected) { reject("RECONNECT_INVALID"); return; }
-  if (!player->second.disconnect_deadline || room_.executed_ticks() >= *player->second.disconnect_deadline) {
+  if (!player->second.disconnect_deadline || owned->room.executed_ticks() >= *player->second.disconnect_deadline) {
     reject("RECONNECT_EXPIRED"); return;
   }
   const auto resume_token = new_bind_token(), bind_token = new_bind_token();
-  if (!room_.reconnect(player->first,conn.id)) { reject("RECONNECT_INVALID"); return; }
+  if (!owned->room.reconnect(player->first,conn.id)) { reject("RECONNECT_INVALID"); return; }
   // Retire the HELLO provisional identity and credential. The stable session
   // belongs only to this new connection; the old TCP/UDP binding is gone.
-  conn.session_id = session; conn.player_id = player->first; conn.udp_endpoint.reset();
+  conn.session_id = session; conn.player_id = player->first; conn.room_id = owned->room.id(); conn.udp_endpoint.reset();
   conn.bind_token = bind_token; conn.token_issued_ms = monotonic_now_(); conn.full_after_bind = true;
   conn.snapshots.reconnect_after(saved->second.last_snapshot_seq); saved->second.token = resume_token;
-  if (room_.status() == "RUNNING") replay_.reconnected(room_,conn.player_id);
+  if (owned->room.status() == "RUNNING") owned->replay.reconnected(owned->room,conn.player_id);
   auto reply = message("RECONNECTED");
-  reply.update(Json{{"session_id",session},{"room_id",room_.id()},{"player_id",conn.player_id},
+  reply.update(Json{{"session_id",session},{"room_id",owned->room.id()},{"player_id",conn.player_id},
     {"last_accepted_seq",player->second.last_accepted_seq()},{"resume_token",resume_token},
     {"udp_bind_token",bind_token},{"udp_port",udp_port_},{"owner_epoch",0}});
   queue(conn.id,std::move(reply));
@@ -684,37 +725,42 @@ void Server::handle(const Envelope& envelope) {
       reject("SESSION_INVALID", "HELLO session must match this connection"); return;
     }
     if (type == "CREATE_ROOM") {
-      if (room_.status() != "ABSENT") { reject("ROOM_NOT_JOINABLE", "G01 supports one room"); return; }
-      room_.create(new_id("room", 1));
-      Json reply = message("ROOM_CREATED"); reply["room_id"] = room_.id(); reply["status"] = room_.status();
+      if (!conn->player_id.empty()) { reject("ROOM_NOT_JOINABLE","joined session already belongs to a Room"); return; }
+      if (rooms_.size() >= max_rooms) { reject("ADMISSION_REJECTED","instance Room capacity reached"); return; }
+      const auto room_id = new_id("room",next_room_++);
+      auto* created = allocate_room(room_id);
+      if (!created) { reject("ADMISSION_REJECTED","Room allocation unavailable"); return; }
+      created->room.create(room_id);
+      Json reply = message("ROOM_CREATED"); reply["room_id"] = room_id; reply["status"] = created->room.status();
       queue(id, std::move(reply)); return;
     }
-    if (value.at("room_id").get<std::string>() != room_.id() || room_.status() == "ABSENT") {
+    auto* owned = context(value.at("room_id").get<std::string>());
+    if (!owned || owned->room.status() == "ABSENT") {
       reject("ROOM_NOT_FOUND", "unknown room"); return;
     }
     if (type == "JOIN_ROOM") {
-      if (room_.status() != "LOBBY" || !conn->player_id.empty() || room_.players().size() == max_players) {
+      if (owned->room.status() != "LOBBY" || !conn->player_id.empty() || owned->room.players().size() == max_players) {
         reject("ROOM_NOT_JOINABLE", "room is not joinable"); return;
       }
-      conn->player_id = new_id("player", next_player_++);
-      const auto& player = room_.join(conn->player_id, conn->session_id, id);
-      const auto resume_token = new_bind_token(); resume_.emplace(player.id,ResumeRecord{resume_token});
-      if (conn->udp_endpoint) room_.bind_realtime(id);
-      Json reply = message("ROOM_JOINED"); reply["room_id"] = room_.id(); reply["player_id"] = player.id;
-      reply["slot"] = player.slot; reply["status"] = room_.status();
+      conn->room_id = owned->room.id(); conn->player_id = new_id("player", next_player_++);
+      const auto& player = owned->room.join(conn->player_id, conn->session_id, id);
+      const auto resume_token = new_bind_token(); owned->resume.emplace(player.id,ResumeRecord{resume_token});
+      if (conn->udp_endpoint) owned->room.bind_realtime(id);
+      Json reply = message("ROOM_JOINED"); reply["room_id"] = owned->room.id(); reply["player_id"] = player.id;
+      reply["slot"] = player.slot; reply["status"] = owned->room.status();
       reply["resume_token"] = resume_token;
       queue(id, std::move(reply));
-      if (room_.status() == "RUNNING") start_room();
+      if (owned->room.status() == "RUNNING") start_room(*owned);
       return;
     }
-    if (conn->player_id.empty() || value.at("player_id").get<std::string>() != conn->player_id) {
+    if (conn->room_id != owned->room.id() || conn->player_id.empty() || value.at("player_id").get<std::string>() != conn->player_id) {
       reject("PLAYER_MISMATCH", "player must belong to this connection"); return;
     }
     if (type == "LEAVE_ROOM") {
       leave_room(id,"LEAVE_ROOM"); return;
     }
     if (type == "PING") {
-      auto reply = message("PONG"); reply["room_id"] = room_.id(); reply["player_id"] = conn->player_id;
+      auto reply = message("PONG"); reply["room_id"] = owned->room.id(); reply["player_id"] = conn->player_id;
       reply["owner_epoch"] = 0; send_realtime(id,reply); return;
     }
     if (type == "SNAPSHOT_ACK") {
@@ -727,15 +773,18 @@ void Server::handle(const Envelope& envelope) {
         throw std::invalid_argument("resync_required must be boolean");
       conn->snapshots.acknowledge(sequence.get<std::uint64_t>(),state_hash,value.value("resync_required",false)); return;
     }
-    const auto result = admit_input(room_, conn->player_id, value);
+    const auto result = admit_input(owned->room, conn->player_id, value, pending_inputs() < max_total_pending_inputs);
     if (result.error) {
       reject(*result.error, "input was not accepted"); return;
     }
-    if (!result.duplicate) replay_.accepted_input(room_,conn->player_id);
+    if (!result.duplicate) {
+      owned->replay.accepted_input(owned->room,conn->player_id);
+      total_pending_high_water_ = std::max(total_pending_high_water_,pending_inputs());
+    }
     Json reply = message("INPUT_ACK"); reply["player_id"] = conn->player_id; reply["accepted"] = true;
     reply["seq"] = value.at("seq").get<std::uint64_t>(); reply["code"] = result.duplicate ? "DUPLICATE" : "ACCEPTED";
-    reply["last_accepted_seq"] = room_.players().at(conn->player_id).last_accepted_seq();
-    reply["tick"] = room_.executed_ticks(); send_realtime(id,reply);
+    reply["last_accepted_seq"] = owned->room.players().at(conn->player_id).last_accepted_seq();
+    reply["tick"] = owned->room.executed_ticks(); send_realtime(id,reply);
   } catch (const Json::exception&) {
     reject("MESSAGE_INVALID", "required field missing or wrong type");
   } catch (const std::invalid_argument&) {
@@ -743,22 +792,28 @@ void Server::handle(const Envelope& envelope) {
   }
 }
 void Server::leave_room(std::uint64_t connection_id, const std::string& kind) {
-  const auto previous_status = room_.status();
-  std::string player_id;
-  for (const auto& [id, player] : room_.players())
-    if (player.connection_id == connection_id && player.connected) player_id = id;
-  if (kind == "DISCONNECT") room_.disconnect(connection_id);
-  else { room_.leave(connection_id); resume_.erase(player_id); }
-  if (!player_id.empty() && previous_status == "RUNNING") replay_.left(room_,player_id,kind);
-  if (auto* conn = connection(connection_id)) {
-    conn->snapshots.clear(); conn->pending_full.reset(); conn->pending_delta.reset(); update_udp_write_interest();
+  // Transport close has already erased its connection; resolve the old owner
+  // by the bounded64*8 model records, never by a client-supplied player ID.
+  for (auto& [room_id, owned] : rooms_) {
+    (void)room_id;
+    const auto found = std::find_if(owned->room.players().begin(),owned->room.players().end(),[&](const auto& row) {
+      return row.second.connection_id == connection_id && row.second.connected;
+    });
+    if (found == owned->room.players().end()) continue;
+    const auto player_id = found->first, previous_status = owned->room.status();
+    if (kind == "DISCONNECT") owned->room.disconnect(connection_id);
+    else { owned->room.leave(connection_id); owned->resume.erase(player_id); }
+    if (previous_status == "RUNNING") owned->replay.left(owned->room,player_id,kind);
+    if (auto* conn = connection(connection_id)) {
+      conn->snapshots.clear(); conn->pending_full.reset(); conn->pending_delta.reset(); update_udp_write_interest();
+    }
+    if (previous_status == "LOBBY" && owned->room.status() == "RUNNING") start_room(*owned);
+    return;
   }
-  if (previous_status == "LOBBY" && room_.status() == "RUNNING") start_room();
 }
 void Server::drain_mailbox() {
   // Room mutations happen after all ready I/O callbacks have completed.
-  for (const auto id : disconnected_) leave_room(id,"DISCONNECT");
-  disconnected_.clear();
+  drain_disconnects();
   const auto size = mailbox_.size();
   for (std::size_t i = 0; i < size; ++i) {
     Envelope envelope = mailbox_.take();
@@ -766,6 +821,12 @@ void Server::drain_mailbox() {
     handle(envelope);
   }
 }
+void Server::drain_disconnects() {
+  // Output can close a transport during owner work. Hand off those bounded
+  // notifications before the next tick, including within a catch-up quantum.
+  const auto disconnected = std::exchange(disconnected_,{});
+  for (const auto id : disconnected) leave_room(id,"DISCONNECT");
+}
 void Server::pump(int timeout_ms) { poll_io(timeout_ms); drain_mailbox(); }
 std::int64_t Server::read_monotonic() {
   const auto now = monotonic_now_();
@@ -776,40 +837,113 @@ TickBatch Server::run_scheduler() {
   if (stopping_) return {};
   drain_mailbox();
   ++scheduler_iterations_;
-  const auto now = read_monotonic();
-  if (room_.status() != "RUNNING") {
-    accumulator_.reset(now); last_batch_ = {}; return last_batch_;
-  }
-  auto batch = accumulator_.advance(now);
-  const auto before = room_.executed_ticks();
-  for (int tick = 0; tick < batch.ticks; ++tick) advance_one_tick();
-  batch.ticks = room_.executed_ticks() - before;
-  if (room_.status() != "RUNNING") {
-    accumulator_.reset(now); batch.remaining_ms = 0; batch.overloaded = false;
+  const auto now = read_monotonic(); // One source observation for the entire instance iteration.
+  last_batch_ = {};
+  // At most64 entries and one capped quantum per entry. Completion/removal
+  // cannot invalidate iteration or grant another quantum at unchanged time.
+  const std::vector<std::string> ready(scheduled_rooms_.begin(),scheduled_rooms_.end());
+  for (const auto& id : ready) {
+    auto& owned = *rooms_.at(id);
+    bool service_ready = true;
+#ifdef ARENA_TEST_FIXTURES
+    if (fixture_service_ready_) service_ready = fixture_service_ready_(id,now);
+#endif
+    auto batch = owned.accumulator.advance(now,service_ready);
+    const auto before = owned.room.executed_ticks();
+    for (int tick = 0; tick < batch.ticks; ++tick) execute_tick(owned);
+    batch.ticks = owned.room.executed_ticks()-before;
+    owned.catch_up_high_water = std::max(owned.catch_up_high_water,batch.ticks);
+    catch_up_high_water_ = std::max(catch_up_high_water_,batch.ticks);
+    if (owned.room.status() != "RUNNING") {
+      owned.accumulator.reset(now); batch.remaining_ms = 0; owned.overloaded = false;
+    } else {
+      // Count elapsed50ms deadlines, not scheduler calls or leftover ticks.
+      // Arithmetic bounds catch-up accounting even after a large clock jump.
+      const auto deadlines = now >= *owned.next_deadline_ms ? (now-*owned.next_deadline_ms)/tick_duration_ms+1 : 0;
+      *owned.next_deadline_ms += deadlines*tick_duration_ms;
+      if (batch.remaining_ms < tick_duration_ms) {
+        owned.missed_deadlines = 0; owned.overloaded = false;
+      } else {
+        if (service_ready) owned.overloaded = batch.overloaded;
+        owned.missed_deadlines = static_cast<int>(std::min<std::int64_t>(max_unrecovered_deadlines,owned.missed_deadlines+deadlines));
+      }
+      if (owned.missed_deadlines == max_unrecovered_deadlines) {
+        close_overloaded(owned,now); batch.remaining_ms = 0;
+      }
+    }
+    batch.overloaded = owned.room.status() == "RUNNING" && owned.overloaded;
+    owned.last_batch = batch;
+    last_batch_.ticks += batch.ticks;
+    last_batch_.remaining_ms = std::max(last_batch_.remaining_ms,batch.remaining_ms);
+    last_batch_.overloaded = last_batch_.overloaded || batch.overloaded;
   }
-  catch_up_high_water_ = std::max(catch_up_high_water_, batch.ticks);
-  last_batch_ = batch;
-  return batch;
+  return last_batch_;
 }
 void Server::advance_one_tick() {
   if (stopping_) return;
   drain_mailbox();
-  if (room_.status() != "RUNNING") return;
-  clock_.advance_one();
-  for (const auto& failure : room_.tick()) {
+  if (auto* owned = context(first_room_id_)) {
+    execute_tick(*owned);
+    if (owned->room.status() != "RUNNING") last_batch_ = {};
+  }
+}
+void Server::execute_tick(RoomContext& owned) {
+  drain_disconnects();
+  if (owned.room.status() != "RUNNING") return;
+  owned.simulation_clock->advance_one();
+  for (const auto& failure : owned.room.tick()) {
     auto error = error_message("ACTION_REJECTED", "TAG conditions not satisfied");
-    error["player_id"] = failure.player_id; error["tick"] = room_.executed_ticks() - 1;
-    ++errors_["ACTION_REJECTED"]; queue(failure.connection_id, std::move(error));
+    error["player_id"] = failure.player_id; error["tick"] = owned.room.executed_ticks()-1;
+    ++errors_["ACTION_REJECTED"]; queue(failure.connection_id,std::move(error));
   }
-  replay_.finish_tick(room_);
-  if (room_.executed_ticks() % 2 == 0) publish_snapshots(replay_.last_state_hash());
-  if (room_.status() == "FINISHED") {
-    accumulator_.reset(0); last_batch_ = {};
-    Json result = room_.view(); result.update(message("ROOM_FINISHED")); broadcast(result);
+  owned.replay.finish_tick(owned.room);
+#ifdef ARENA_TEST_FIXTURES
+  if (fixture_tick_observer_) fixture_tick_observer_(owned.room,owned.replay);
+#endif
+  if (owned.room.executed_ticks() % 2 == 0) publish_snapshots(owned,owned.replay.last_state_hash());
+  if (owned.room.status() == "FINISHED") {
+    scheduled_rooms_.erase(owned.room.id()); owned.next_deadline_ms.reset();
+    owned.accumulator.reset(0); owned.last_batch = {};
+    Json result = owned.room.view(); result.update(message("ROOM_FINISHED")); broadcast(owned.room.id(),result);
   }
 }
+void Server::close_overloaded(RoomContext& owned, std::int64_t now) {
+  owned.terminal_code = "ROOM_OVERLOAD"; owned.terminal_ms = now;
+  owned.terminal_pending_cleared = owned.room.pending_count();
+  ++errors_["ROOM_OVERLOAD"];
+  scheduled_rooms_.erase(owned.room.id()); owned.next_deadline_ms.reset();
+  owned.room.close(); owned.resume.clear(); owned.replay.clear(); owned.accumulator.reset(now);
+  std::vector<std::pair<int,std::uint64_t>> peers;
+  for (const auto& [fd,conn] : connections_) if (conn.room_id == owned.room.id()) peers.emplace_back(fd,conn.id);
+  for (const auto& [fd,id] : peers) {
+    queue(id,error_message("ROOM_OVERLOAD","Room missed20 consecutive unrecovered deadlines"));
+    write_ready(fd); disconnect(fd,"");
+    disconnected_.erase(id); // Its owner is already terminal; no grace record is created.
+  }
+}
+Json Server::room_metrics(const RoomContext& owned) const {
+  return Json{{"status",owned.room.status()},{"executed_ticks",owned.room.executed_ticks()},
+    {"simulation_ms",owned.simulation_clock->now_ms},{"scheduled",scheduled_rooms_.contains(owned.room.id())},
+    {"next_deadline_ms",owned.next_deadline_ms ? Json(*owned.next_deadline_ms) : Json(nullptr)},
+    {"ticks_last_iteration",owned.last_batch.ticks},{"remaining_ms",owned.accumulator.remaining_ms()},
+    {"missed_deadlines",owned.missed_deadlines},{"catch_up_high_water",owned.catch_up_high_water},
+    {"operational_state",owned.overloaded ? "OVERLOADED" : "NORMAL"},
+    {"pending_inputs",owned.room.pending_count()},{"resume_records",owned.resume.size()},{"resume_record_capacity",max_players},
+    {"input_per_player_high_water",owned.room.input_high_water()},{"replay_bytes",owned.replay.bytes()},
+    {"replay_bytes_high_water",owned.replay.high_water_bytes()},{"replay_pending_events",owned.replay.pending_events()},
+    {"replay_capture_complete",owned.replay.complete()},{"last_state_hash",owned.replay.last_state_hash()},
+    {"terminal_code",owned.terminal_code.empty() ? Json(nullptr) : Json(owned.terminal_code)},
+    {"terminal_ms",owned.terminal_ms ? Json(*owned.terminal_ms) : Json(nullptr)},
+    {"terminal_pending_cleared",owned.terminal_pending_cleared}};
+}
 Json Server::metrics() const {
-  auto streams = Json::object(), outbound = Json::object();
+  auto streams = Json::object(), outbound = Json::object(), room_states = Json::object();
+  std::size_t resumes = 0, input_high_water = room().input_high_water(), attempts_high_water = room().input_attempt_high_water();
+  for (const auto& [id,owned] : rooms_) {
+    room_states[id] = room_metrics(*owned); resumes += owned->resume.size();
+    input_high_water = std::max(input_high_water,owned->room.input_high_water());
+    attempts_high_water = std::max(attempts_high_water,owned->room.input_attempt_high_water());
+  }
   std::size_t bound = 0, tokens = 0, sessions = 0, queued_buffers = 0, queued_bytes = 0;
   for (const auto& [fd, conn] : connections_) {
     (void)fd;
@@ -822,7 +956,10 @@ Json Server::metrics() const {
   }
   return Json{{"received_messages", received_messages_}, {"sent_messages", sent_messages_},
     {"mailbox_high_water", mailbox_high_water_}, {"outbound_control_high_water", outbound_high_water_},
-    {"connection_high_water", connection_high_water_}, {"input_per_player_high_water", room_.input_high_water()},
+    {"connection_high_water", connection_high_water_}, {"input_per_player_high_water", input_high_water},
+    {"rooms",room_states},{"room_capacity",max_rooms},{"room_high_water",room_high_water_},
+    {"scheduled_rooms",scheduled_rooms_.size()},{"total_pending_inputs",pending_inputs()},
+    {"total_pending_capacity",max_total_pending_inputs},{"total_pending_high_water",total_pending_high_water_},
     {"snapshot_retention_high_water",snapshot_retention_high_water_},{"snapshot_streams",streams},
     {"outbound_streams",outbound},{"outbound_buffers",outbound_memory_.buffers},{"outbound_retained_bytes",outbound_memory_.bytes},
     {"outbound_buffer_high_water",outbound_memory_.high_water_buffers},{"outbound_bytes_high_water",outbound_memory_.high_water_bytes},
@@ -831,11 +968,11 @@ Json Server::metrics() const {
     {"udp_received_datagrams",received_datagrams_},{"udp_sent_datagrams",sent_datagrams_},
     {"udp_payload_high_water",datagram_high_water_},{"udp_outbound_high_water",outbound_datagram_high_water_},
     {"udp_receive_buffer_bytes",max_datagram_bytes+1},{"udp_bound_endpoints",bound},{"udp_bind_tokens",tokens},
-    {"active_sessions",sessions},{"resume_records",resume_.size()},{"resume_record_capacity",max_players},
-    {"input_attempt_per_player_high_water", room_.input_attempt_high_water()},
-    {"replay_bytes_high_water",replay_.high_water_bytes()},
-    {"replay_capture_complete",replay_.complete()},{"replay_capture_error",replay_.failure()},
-    {"last_state_hash",replay_.last_state_hash()},
+    {"active_sessions",sessions},{"resume_records",resumes},{"resume_record_capacity",max_rooms*max_players},
+    {"input_attempt_per_player_high_water", attempts_high_water},
+    {"replay_bytes_high_water",replay().high_water_bytes()},
+    {"replay_capture_complete",replay().complete()},{"replay_capture_error",replay().failure()},
+    {"last_state_hash",replay().last_state_hash()},
     {"max_read_bytes", max_read_bytes_}, {"parser_buffer_high_water", parser_high_water_},
     {"parser_storage_bytes_per_connection", FrameParser::storage_bytes}, {"need_more_events", need_more_events_},
     {"message_error_events", message_error_events_}, {"terminal_frame_events", terminal_frame_events_},
@@ -843,7 +980,7 @@ Json Server::metrics() const {
     {"partial_writes", partial_writes_}, {"errors", errors_},
     {"scheduler", Json{{"iterations", scheduler_iterations_}, {"monotonic_reads", monotonic_reads_},
       {"ticks_last_iteration", last_batch_.ticks}, {"catch_up_high_water", catch_up_high_water_},
-      {"remaining_ms", accumulator_.remaining_ms()}, {"overloaded", last_batch_.overloaded},
+      {"remaining_ms", last_batch_.remaining_ms}, {"overloaded", last_batch_.overloaded},
       {"operational_state", last_batch_.overloaded ? "OVERLOADED" : "NORMAL"}}}};
 }
 Json Server::cleanup() const {
@@ -854,18 +991,27 @@ Json Server::cleanup() const {
     endpoints += conn.udp_endpoint.has_value(); tokens += !conn.bind_token.empty();
     sessions += !conn.session_id.empty();
   }
-  for (const auto& [id, player] : room_.players()) { (void)id; input_attempts += player.input_attempts; grace += player.disconnect_deadline.has_value(); }
+  std::size_t resumes = 0, replay_bytes = 0, replay_events = 0, timers = 0;
+  std::int64_t scheduler_pending = 0;
+  for (const auto& [room_id,owned] : rooms_) {
+    (void)room_id;
+    for (const auto& [id,player] : owned->room.players()) {
+      (void)id; input_attempts += player.input_attempts; grace += player.disconnect_deadline.has_value();
+    }
+    resumes += owned->resume.size(); replay_bytes += owned->replay.bytes(); replay_events += owned->replay.pending_events();
+    timers += owned->next_deadline_ms.has_value(); scheduler_pending += owned->accumulator.remaining_ms();
+  }
   return Json{{"server_connections", connections_.size()}, {"server_descriptors", owned_descriptors().size()},
-    {"mailbox_messages", mailbox_.size()}, {"pending_inputs", room_.pending_count()}, {"outbound_messages", queued},
+    {"mailbox_messages", mailbox_.size()}, {"pending_inputs", pending_inputs()}, {"outbound_messages", queued},
     {"outbound_buffers",outbound_memory_.buffers},{"outbound_retained_bytes",outbound_memory_.bytes},
     {"input_attempts", input_attempts},
     {"retained_snapshots",retained_snapshots},
-    {"active_sessions",sessions},{"resume_records",resume_.size()},{"grace_deadlines",grace},
+    {"active_sessions",sessions},{"resume_records",resumes},{"grace_deadlines",grace},
     {"udp_bound_endpoints",endpoints},{"udp_bind_tokens",tokens},{"udp_descriptors",datagram_.get() >= 0 ? 1 : 0},
-    {"replay_bytes",replay_.bytes()},{"replay_pending_events",replay_.pending_events()},
+    {"replay_bytes",replay_bytes},{"replay_pending_events",replay_events},
     {"parser_buffered_bytes", parser_buffered}, {"parser_storage_bytes", connections_.size() * FrameParser::storage_bytes},
-    {"worker_threads", 0}, {"timers", 0}, {"disconnect_notifications", disconnected_.size()},
-    {"scheduler_pending_ms", accumulator_.remaining_ms()}};
+    {"worker_threads", 0}, {"timers", timers},{"room_registry",rooms_.size()},{"scheduler_rooms",scheduled_rooms_.size()}, {"disconnect_notifications", disconnected_.size()},
+    {"scheduler_pending_ms", scheduler_pending}};
 }
 std::vector<int> Server::owned_descriptors() const {
   std::vector<int> descriptors;
@@ -881,10 +1027,12 @@ void Server::shutdown() {
   listener_.reset();
   datagram_.reset();
   drain_mailbox();
-  room_.close();
-  resume_.clear();
-  replay_.clear();
-  accumulator_.reset(0); last_batch_ = {};
+  for (auto& [id,owned] : rooms_) {
+    owned->room.close(); owned->resume.clear(); owned->replay.clear();
+    if (id == first_room_id_) { retired_room_ = std::move(owned->room); retired_replay_ = std::move(owned->replay); }
+  }
+  if (rooms_.empty()) retired_room_.close();
+  rooms_.clear(); scheduled_rooms_.clear(); last_batch_ = {};
   // Only transport flushing uses a wall deadline; no simulation runs here.
   const auto deadline = std::chrono::steady_clock::now() + std::chrono::milliseconds(500);
   for (;;) {
diff --git a/src/transport.hpp b/src/transport.hpp
index 2426203..4e668a3 100644
--- a/src/transport.hpp
+++ b/src/transport.hpp
@@ -5,6 +5,7 @@
 #include <atomic>
 #include <cstddef>
 #include <functional>
+#include <memory>
 #include <span>
 #include <set>
 #include <netinet/in.h>
@@ -32,7 +33,7 @@ std::vector<std::uint8_t> encode_datagram(const Json& value);
 Json decode_datagram(std::span<const std::uint8_t> bytes);
 Input decode_input(const Json& value);
 // The caller has attributed the request to its authenticated Room/player.
-InputResult admit_input(Room& room, const std::string& player_id, const Json& value);
+InputResult admit_input(Room& room, const std::string& player_id, const Json& value, bool instance_has_capacity = true);
 
 enum class ParseState { need_more, message, message_error, terminal_frame_error, io_end };
 std::string parse_state_name(ParseState state);
@@ -96,8 +97,10 @@ class Server {
   void advance_one_tick();
   TickBatch run_scheduler();
   void shutdown();
-  const Room& room() const { return room_; }
-  const ReplayLog& replay() const { return replay_; }
+  const Room& room() const;
+  const Room& room(const std::string& id) const;
+  const ReplayLog& replay() const;
+  const ReplayLog& replay(const std::string& id) const;
   Json metrics() const;
   Json cleanup() const;
   std::vector<int> owned_descriptors() const;
@@ -107,6 +110,7 @@ class Server {
     std::uint64_t id;
     std::string session_id;
     std::string player_id;
+    std::string room_id;
     std::deque<PendingWrite> outbound;
     std::size_t pending_requests = 0;
     FrameParser parser;
@@ -122,6 +126,25 @@ class Server {
   // At most one record per bounded Room player, including expired players
   // until Room teardown so their current credential gets EXPIRED, not reset.
   struct ResumeRecord { std::string token; std::uint64_t last_snapshot_seq = 0; };
+  // Every Room owns its model, journal, grace records and scheduling state.
+  // Simulation counters are advanced only by executed ticks; the scheduler's
+  // single monotonic observation is shared by all these accumulators.
+  struct RoomContext {
+    explicit RoomContext(ManualClock* first_clock) : simulation_clock(first_clock ? first_clock : &local_clock) {}
+    Room room;
+    ReplayLog replay;
+    std::map<std::string,ResumeRecord> resume;
+    ManualClock local_clock;
+    ManualClock* simulation_clock;
+    FixedTickAccumulator accumulator;
+    TickBatch last_batch;
+    std::optional<std::int64_t> next_deadline_ms;
+    int missed_deadlines = 0, catch_up_high_water = 0;
+    bool overloaded = false;
+    std::string terminal_code;
+    std::optional<std::int64_t> terminal_ms;
+    std::size_t terminal_pending_cleared = 0;
+  };
   struct Envelope {
     std::uint64_t connection_id; Json value; std::string parser_error;
     std::optional<sockaddr_in> udp_endpoint = {};
@@ -142,10 +165,20 @@ class Server {
   friend struct ReplayFixture;
   friend struct UdpFixture;
   friend struct OutboundFixture;
+  friend struct IsolationFixture;
   std::optional<std::string> fixture_room_id_;
+  std::vector<std::string> fixture_room_ids_;
   std::vector<std::string> fixture_player_ids_;
   std::function<bool(std::uint64_t,bool)> fixture_outbound_ready_;
+  std::function<bool(const std::string&,std::int64_t)> fixture_service_ready_;
+  std::function<void(const Room&,const ReplayLog&)> fixture_tick_observer_;
 #endif
+  RoomContext* context(const std::string& id);
+  RoomContext* allocate_room(const std::string& id);
+  std::size_t pending_inputs() const;
+  void execute_tick(RoomContext& context);
+  void close_overloaded(RoomContext& context, std::int64_t now);
+  Json room_metrics(const RoomContext& context) const;
   Connection* connection(std::uint64_t id);
   void register_event(int fd, short filter, unsigned short flags, std::uint64_t connection_id = 0);
   void accept_ready();
@@ -162,18 +195,18 @@ class Server {
   void disconnect(int fd, const std::string& reason);
   void end_transport(int fd, bool io_error);
   void queue(std::uint64_t connection_id, Json value);
-  void broadcast(const Json& value);
-  void start_room();
-  void publish_snapshots(const std::string& state_hash);
+  void broadcast(const std::string& room_id, const Json& value);
+  void start_room(RoomContext& context);
+  void publish_snapshots(RoomContext& context, const std::string& state_hash);
   void publish_snapshot(Connection& conn, const std::string& state_hash);
   void reconnect(Connection& conn, const Json& value);
   void handle(const Envelope& envelope);
   void leave_room(std::uint64_t connection_id, const std::string& kind);
+  void drain_disconnects();
   std::int64_t read_monotonic();
   std::string new_id(const std::string& prefix, std::uint64_t number) const;
   ManualClock& clock_;
   MonotonicNow monotonic_now_;
-  FixedTickAccumulator accumulator_;
   TickBatch last_batch_;
   std::uint64_t scheduler_iterations_ = 0;
   std::uint64_t monotonic_reads_ = 0;
@@ -187,12 +220,18 @@ class Server {
   std::map<int, Connection> connections_;
   Mailbox mailbox_;
   std::set<std::uint64_t> disconnected_;
-  Room room_;
-  ReplayLog replay_;
-  std::map<std::string,ResumeRecord> resume_;
+  std::map<std::string,std::unique_ptr<RoomContext>> rooms_;
+  std::set<std::string> scheduled_rooms_;
+  std::string first_room_id_;
+  // One closed default model may remain for legacy post-shutdown inspection;
+  // it retains no journal/work/transport/timer ownership or mutable exposure.
+  Room retired_room_;
+  ReplayLog retired_replay_;
   std::string nonce_;
   std::uint64_t next_connection_ = 1;
   std::uint64_t next_player_ = 1;
+  std::uint64_t next_room_ = 1;
+  std::size_t room_high_water_ = 0, total_pending_high_water_ = 0;
   std::size_t mailbox_high_water_ = 0;
   std::size_t outbound_high_water_ = 0;
   std::size_t connection_high_water_ = 0;
diff --git a/tests/g07.cpp b/tests/g07.cpp
index faeccd4..250836f 100644
--- a/tests/g07.cpp
+++ b/tests/g07.cpp
@@ -110,7 +110,7 @@ struct ReplayFixture {
       require(found != server.connections_.end() && found->second.player_id.empty(), "real server session binding");
       auto& connection = found->second;
       require(connection.udp_endpoint.has_value(), "real UDP binding precedes the historical fixture start");
-      connection.player_id = peer.player;
+      connection.player_id = peer.player; connection.room_id = scenario.at("room_id").get<std::string>();
       Player player; player.id = peer.player; player.session_id = connection.session_id;
       player.connection_id = connection.id; player.slot = item.at("slot").get<int>();
       require(identifier(player.id) && player.slot == static_cast<int>(players.size()), "frozen identifiers/ordered slots");
@@ -120,8 +120,10 @@ struct ReplayFixture {
         {"connection_id",connection.id},{"server_descriptor",connection.fd.get()},{"client_descriptor",peer.tcp->descriptor()}});
     }
     // All four transport/session/player bindings precede the one start call.
-    initialize(server.room_,scenario.at("room_id").get<std::string>(),std::move(players));
-    server.start_room();
+    auto* owned = server.allocate_room(scenario.at("room_id").get<std::string>());
+    require(owned != nullptr,"initial historical Room uses the bounded production registry");
+    initialize(owned->room,scenario.at("room_id").get<std::string>(),std::move(players));
+    server.start_room(*owned);
     return bindings;
   }
   static void offline(Room& room, const Json& artifact) {
diff --git a/tests/g09.cpp b/tests/g09.cpp
index c17804b..e4286ae 100644
--- a/tests/g09.cpp
+++ b/tests/g09.cpp
@@ -46,7 +46,7 @@ struct UdpFixture {
     full["snapshot_seq"] = 601; return full;
   }
   static void identifiers(Server& server, const Json& scenario) {
-    need(server.room_.status() == "ABSENT" && server.connections_.empty(),"ID injection precedes all normal joins");
+    need(server.room().status() == "ABSENT" && server.connections_.empty(),"ID injection precedes all normal joins");
     const auto room = scenario.at("room_id").get<std::string>(); std::vector<std::string> ids; std::set<std::string> unique;
     need(identifier(room) && !scenario.at("players").empty() && scenario.at("players").size() <= max_players,"bounded fixture identifiers");
     for (const auto& row : scenario.at("players")) {
@@ -77,11 +77,11 @@ struct UdpFixture {
     {
       ManualClock clock; Server candidate(clock);
       try { identifiers(candidate,invalid); } catch (const std::length_error&) { rejected = true; }
-      need(rejected && !candidate.fixture_room_id_ && candidate.room_.status() == "ABSENT","oversized fixture is explicitly rejected before injection");
+      need(rejected && !candidate.fixture_room_id_ && candidate.room().status() == "ABSENT","oversized fixture is explicitly rejected before injection");
       candidate.shutdown();
     }
     const auto before = server.errors_["UDP_OUTBOUND_SIZE_INVALID"], sent = server.sent_datagrams_;
-    const auto& player = server.room_.players().at(player_id);
+    const auto& player = server.room().players().at(player_id);
     server.send_realtime(player.connection_id,maximum_full(std::string(64,'R'),long_ids));
     need(server.errors_["UDP_OUTBOUND_SIZE_INVALID"] == before+1 && server.sent_datagrams_ == sent,"real outbound size rejection emits no datagram");
     Json lengths = Json::array(); for (const auto& id : ids) lengths.push_back(id.size());
diff --git a/tests/g13.cpp b/tests/g13.cpp
new file mode 100644
index 0000000..e9ea221
--- /dev/null
+++ b/tests/g13.cpp
@@ -0,0 +1,416 @@
+#include "g13.hpp"
+#ifndef ARENA_TEST_FIXTURES
+#error G13 Room service holds and identity injection are test-build only
+#endif
+#include <algorithm>
+#include <array>
+#include <chrono>
+#include <fstream>
+#include <iomanip>
+#include <memory>
+#include <set>
+#include <sstream>
+#include <unistd.h>
+
+namespace arena {
+namespace {
+void isolation_need(bool value, const std::string& description) {
+  if (!value) throw std::runtime_error("G13: "+description);
+}
+std::string room_name(std::size_t index) {
+  std::ostringstream out; out << "room-" << std::setw(2) << std::setfill('0') << index; return out.str();
+}
+Json isolation_state(const Room& room) {
+  auto state = room.view(); state["owner_epoch"] = 0;
+  for (auto& row : state["players"]) {
+    const auto& p = room.players().at(row.at("player_id").get<std::string>());
+    row["last_seq"] = p.last_accepted_seq(); row["pending"] = p.pending.size();
+    row["applied_seq"] = p.applied_seq ? Json(*p.applied_seq) : Json(nullptr);
+    row["disconnect_deadline"] = p.disconnect_deadline ? Json(*p.disconnect_deadline) : Json(nullptr);
+  }
+  return state;
+}
+Json visible(const Json& state) {
+  Json rows = Json::array();
+  for (const auto& player : state.at("players")) if (player.at("connectivity") != "LEFT") {
+    Json row; for (const auto* field : {"player_id","slot","x","y","direction","score","connectivity"}) row[field] = player.at(field);
+    rows.push_back(std::move(row));
+  }
+  return Json{{"room_id",state.at("room_id")},{"tick",state.at("tick")},{"owner_epoch",0},{"status",state.at("status")},{"players",rows}};
+}
+Json redact(Json request, const std::string& alias) {
+  request.erase("session_id"); request["session_alias"] = alias; return request;
+}
+struct IsolationPeer {
+  std::unique_ptr<TcpClient> tcp;
+  std::unique_ptr<UdpClient> udp;
+  std::string session, player;
+  std::uint64_t connection = 0, latest = 0;
+  std::map<std::uint64_t,Json> retained;
+  Json snapshots = Json::array();
+};
+struct IsolationRoom {
+  std::string id;
+  std::array<IsolationPeer,4> peers;
+  Json initial, initial_hash, records = Json::array(), admissions = Json::array();
+};
+}
+
+struct IsolationFixture {
+  static void configure(Server& server, const Json& scenario,
+      std::function<bool(const std::string&,std::int64_t)> service,
+      std::function<void(const Room&,const ReplayLog&)> observer) {
+    isolation_need(server.rooms_.empty() && server.connections_.empty(),"fixture injection before any real connection or Room");
+    for (std::size_t i = 0; i < 32; ++i) {
+      const auto& fixed = scenario.at("rooms").at(i); const auto id = room_name(i);
+      isolation_need(fixed.at("room_id") == id && fixed.at("players").size() == 4,"exact fixed Room mapping");
+      server.fixture_room_ids_.push_back(id);
+      for (std::size_t slot = 0; slot < 4; ++slot) {
+        const auto player = "player-"+id.substr(5)+"-"+std::to_string(slot);
+        isolation_need(fixed.at("players").at(slot).at("player_id") == player &&
+          fixed.at("players").at(slot).at("slot") == slot,"exact fixed player allocator mapping");
+        server.fixture_player_ids_.push_back(player);
+      }
+    }
+    server.fixture_service_ready_ = std::move(service); server.fixture_tick_observer_ = std::move(observer);
+  }
+  static std::uint64_t connection_id(const Server& server, const std::string& session) {
+    for (const auto& [fd,conn] : server.connections_) { (void)fd; if (conn.session_id == session) return conn.id; }
+    throw std::logic_error("G13 real session missing");
+  }
+  static std::uint64_t received(const Server& server) { return server.received_datagrams_; }
+  static bool drained(const Server& server) { return server.mailbox_.size() == 0; }
+  static std::uint64_t acknowledged(Server& server, std::uint64_t id) {
+    const auto* conn = server.connection(id);
+    isolation_need(conn != nullptr,"ACK belongs to a live bound connection");
+    return conn->snapshots.metrics().at("acknowledged_seq").get<std::uint64_t>();
+  }
+  static Json schedule(const Server& server) {
+    Json rooms = Json::object(); for (const auto& [id,owned] : server.rooms_) rooms[id] = server.room_metrics(*owned);
+    return Json{{"rooms",rooms},{"scheduled_room_ids",server.scheduled_rooms_},{"scheduler_iterations",server.scheduler_iterations_},
+      {"monotonic_reads",server.monotonic_reads_},{"mailbox_high_water",server.mailbox_high_water_},
+      {"total_pending_inputs",server.pending_inputs()},{"total_pending_high_water",server.total_pending_high_water_}};
+  }
+  static Json mapping(const Server& server) {
+    Json rows = Json::array(); std::set<const Room*> unique;
+    for (const auto& [id,owned] : server.rooms_) {
+      isolation_need(unique.insert(&owned->room).second && owned->room.owner_ == std::this_thread::get_id(),"distinct Room models on their actual single owner");
+      std::ostringstream owner; owner << owned->room.owner_; Json players = Json::array();
+      for (const auto& [player_id,p] : owned->room.players()) {
+        const auto found = std::find_if(server.connections_.begin(),server.connections_.end(),[&](const auto& row) { return row.second.id == p.connection_id; });
+        isolation_need(found != server.connections_.end() && found->second.room_id == id && found->second.player_id == player_id &&
+          found->second.session_id == p.session_id && found->second.udp_endpoint && p.realtime_ready,"ordinary Room/session/UDP owner mapping");
+        players.push_back(Json{{"player_id",player_id},{"connection_id",p.connection_id},{"server_descriptor",found->first},
+          {"session_matches_owner",true},{"udp_endpoint_bound",true}});
+      }
+      rows.push_back(Json{{"room_id",id},{"room_object",reinterpret_cast<std::uintptr_t>(&owned->room)},
+        {"owner_thread",owner.str()},{"simulation_counter",reinterpret_cast<std::uintptr_t>(owned->simulation_clock)},
+        {"scheduled",server.scheduled_rooms_.contains(id)},{"players",players}});
+    }
+    return rows;
+  }
+};
+
+Json run_isolation_scenario(const Json& scenario, const std::filesystem::path& output) {
+  isolation_need(scenario.at("thread") == "G13" && scenario.at("contract_version") == 1 && scenario.at("seed") == 7050 &&
+    scenario.at("rooms").size() == 32 && scenario.at("clock").at("kind") == "shared-manual-monotonic" &&
+    scenario.at("clock").at("tick_duration_ms") == 50 && scenario.at("clock").at("end_ms") == 1250 &&
+    scenario.at("expected_normal_ticks") == 25 && scenario.at("socket_ceiling_ms") == 5000,"frozen32-Room single-instance fixture");
+  isolation_need(scenario.at("resource_limits") == Json{{"rooms",max_rooms},{"connections",max_connections},
+    {"accepted_pending_per_player",max_pending_inputs},{"total_accepted_pending",max_total_pending_inputs},
+    {"catchup_per_room_per_iteration",max_catch_up_ticks}},"unchanged fixed instance bounds");
+  const auto services = scenario.at("clock").at("room0_service_ms").get<std::vector<std::int64_t>>();
+  const auto events = scenario.at("clock").at("event_times_ms").get<std::vector<std::int64_t>>();
+  const auto deadlines = scenario.at("clock").at("deadlines_ms").get<std::vector<std::int64_t>>();
+  isolation_need(services == std::vector<std::int64_t>({225,450,675,900,1125}) && events.size() == 28 && deadlines.size() == 25 &&
+    scenario.at("hot_room").at("burst_at_ms") == Json::array({0,225,450,675,900,1125}) &&
+    scenario.at("hot_room").at("target_ticks") == Json::array({0,4,8,12,16,20}) &&
+    scenario.at("hot_room").at("attempts_per_player_per_burst") == 64,"fixed wake and burst schedule");
+  const int fd_before = Fd::live(); ManualClock executed_clock; std::int64_t shared_now = 0;
+  Server server(executed_clock,0,[&] { return shared_now; }); std::array<IsolationRoom,32> rooms;
+  std::map<std::string,std::size_t> indices;
+  for (std::size_t i = 0; i < rooms.size(); ++i) { rooms[i].id = room_name(i); indices.emplace(rooms[i].id,i); }
+  Json service_trace = Json::array();
+  IsolationFixture::configure(server,scenario,[&](const std::string& id, std::int64_t now) {
+    const bool eligible = id != rooms[0].id || std::find(services.begin(),services.end(),now) != services.end();
+    isolation_need(now == shared_now,"all Room service decisions see the same clock observation");
+    service_trace.push_back(Json{{"room_id",id},{"monotonic_ms",now},{"eligible",eligible}}); return eligible;
+  },[&](const Room& room, const ReplayLog& replay) {
+    const auto index = indices.at(room.id()); auto& rows = rooms.at(index).records;
+    const auto record = canonical_state(room), hash = sha256(record);
+    isolation_need(rows.size() < (room.id() == rooms[0].id ? 20U : 25U) && room.executed_ticks() == static_cast<int>(rows.size())+1 &&
+      replay.last_state_hash() == hash,"exact per-executed-tick owner capture, including all four catch-up ticks");
+    const int tick = room.executed_ticks()-1, distance = room.executed_ticks()*400;
+    for (const auto& [id,p] : room.players()) {
+      (void)id;
+      const auto& fixed = scenario.at("rooms").at(index).at("players").at(static_cast<std::size_t>(p.slot));
+      const auto accepted = static_cast<std::uint64_t>(index == 0 ? tick/4*64+4 : tick+1);
+      const auto applied = index == 0 && tick%4 != 0 ? std::optional<std::uint64_t>{} : std::optional<std::uint64_t>{accepted};
+      const int x = p.slot == 0 ? 10000+distance : p.slot == 1 ? 90000-distance : p.slot == 2 ? 10000 : 90000;
+      const int y = p.slot == 0 ? 10000 : p.slot == 1 ? 90000 : p.slot == 2 ? 90000-distance : 10000+distance;
+      isolation_need(p.x == x && p.y == y && p.score == 0 && p.last_tag_tick == -20 && p.connected && !p.disconnect_deadline &&
+        p.pending.empty() && p.last_accepted_seq() == accepted && p.applied_seq == applied && direction_name(p.direction) == fixed.at("direction").get<std::string>(),
+        "each actual tick preserves the fixed movement, highest sequence, pending and authority rules");
+    }
+    rows.push_back(Json{{"tick",room.executed_ticks()-1},{"monotonic_ms",shared_now},{"state",isolation_state(room)},
+      {"canonical_record",record},{"state_hash",hash}});
+  });
+  const auto wait_for = [&](const auto& ready) {
+    const auto ceiling = std::chrono::steady_clock::now()+std::chrono::seconds(5);
+    while (!ready() && std::chrono::steady_clock::now() < ceiling) server.pump();
+    isolation_need(ready(),"actual socket/owner ingress barrier within5s");
+  };
+  const auto ingress = [&](std::uint64_t target) {
+    wait_for([&] { return IsolationFixture::received(server) >= target && IsolationFixture::drained(server); });
+    isolation_need(IsolationFixture::received(server) == target,"exact actual datagram ingress count");
+  };
+  const auto consume = [&](IsolationRoom& fixture, IsolationPeer& peer) {
+    const auto wire = peer.udp->receive_type(server,"SNAPSHOT"); const auto sequence = wire.at("snapshot_seq").get<std::uint64_t>();
+    const auto tick = wire.at("tick").get<int>();
+    const auto& capture = tick < 0 ? fixture.initial : fixture.records.at(static_cast<std::size_t>(tick)).at("state");
+    const auto hash = tick < 0 ? fixture.initial_hash : fixture.records.at(static_cast<std::size_t>(tick)).at("state_hash");
+    isolation_need(sequence == peer.latest+1 && tick == static_cast<int>(sequence)*2-3 && wire.at("state_hash") == hash &&
+      wire.at("room_id") == fixture.id && wire.at("owner_epoch") == 0 && encode_datagram(wire).size() <= max_datagram_bytes,
+      "actual sequential snapshot for this Room and its captured canonical tick");
+    std::map<std::string,Json> players; std::string status;
+    if (wire.at("kind") == "FULL") {
+      isolation_need(wire.at("base_snapshot_seq").is_null() && wire.size() == 12,"FULL shape"); status = wire.at("status").get<std::string>();
+    } else {
+      isolation_need(wire.at("kind") == "DELTA" && wire.size() == 11 && peer.retained.contains(wire.at("base_snapshot_seq").get<std::uint64_t>()),
+        "DELTA uses a real retained acknowledged base, including two publications within one quantum");
+      const auto& base = peer.retained.at(wire.at("base_snapshot_seq").get<std::uint64_t>()); status = base.at("status").get<std::string>();
+      for (const auto& row : base.at("players")) players.emplace(row.at("player_id").get<std::string>(),row);
+    }
+    for (const auto& removed : wire.at("removed_player_ids")) players.erase(removed.get<std::string>());
+    std::string previous;
+    for (const auto& row : wire.at("players")) {
+      const auto id = row.at("player_id").get<std::string>();
+      isolation_need(row.size() == 7 && id > previous,"sorted minimal player projection"); previous = id; players[id] = row;
+    }
+    Json rows = Json::array(); for (const auto& [id,row] : players) { (void)id; rows.push_back(row); }
+    Json applied{{"room_id",fixture.id},{"tick",tick},{"owner_epoch",0},{"status",status},{"players",rows}};
+    isolation_need(applied == visible(capture),"no cross-Room replication or state contamination");
+    peer.latest = sequence; peer.retained[sequence] = applied;
+    if (peer.retained.size() > snapshot_retention) peer.retained.erase(peer.retained.begin());
+    auto ack = message("SNAPSHOT_ACK"); ack.update(Json{{"session_id",peer.session},{"room_id",fixture.id},{"player_id",peer.player},
+      {"snapshot_seq",sequence},{"state_hash",hash},{"owner_epoch",0}}); peer.udp->send(ack);
+    peer.snapshots.push_back(Json{{"wire",wire},{"ack",redact(ack,peer.player)},{"replica_equal",true}});
+  };
+  Json setup = Json::array();
+  for (std::size_t i = 0; i < rooms.size(); ++i) {
+    auto& fixture = rooms[i]; const auto& fixed = scenario.at("rooms").at(i); Json joins = Json::array(), bindings = Json::array();
+    for (auto& peer : fixture.peers) {
+      peer.tcp = std::make_unique<TcpClient>(server.port()); peer.tcp->send(message("HELLO"));
+      const auto welcome = peer.tcp->receive_type(server,"WELCOME"); peer.session = welcome.at("session_id").get<std::string>();
+      isolation_need(peer.tcp->has_bind_token() && !welcome.contains("udp_bind_token"),"private HELLO binding credential");
+      peer.connection = IsolationFixture::connection_id(server,peer.session); peer.udp = std::make_unique<UdpClient>(server.udp_port());
+    }
+    auto create = message("CREATE_ROOM"); create["session_id"] = fixture.peers[0].session; fixture.peers[0].tcp->send(create);
+    const auto created = fixture.peers[0].tcp->receive_type(server,"ROOM_CREATED");
+    isolation_need(created.at("room_id") == fixture.id && created.at("status") == "LOBBY","ordinary independent CREATE_ROOM");
+    for (std::size_t slot = 0; slot < 4; ++slot) {
+      auto& peer = fixture.peers[slot]; auto join = message("JOIN_ROOM"); join.update(Json{{"session_id",peer.session},{"room_id",fixture.id}});
+      peer.tcp->send(join); const auto joined = peer.tcp->receive_type(server,"ROOM_JOINED"); peer.player = joined.at("player_id").get<std::string>();
+      const auto& row = fixed.at("players").at(slot); const auto& p = server.room(fixture.id).players().at(peer.player);
+      isolation_need(joined.at("status") == "LOBBY" && joined.at("slot") == slot && peer.player == row.at("player_id").get<std::string>() &&
+        p.x == row.at("spawn").at(0) && p.y == row.at("spawn").at(1) && peer.tcp->has_resume_token(),"four ordinary unbound joins preserve slot/spawn/grace contract");
+      joins.push_back(Json{{"reply",joined},{"resume_token_present",true},{"client_TCP_descriptor",peer.tcp->descriptor()},
+        {"client_UDP_descriptor",peer.udp->descriptor()}});
+    }
+    for (std::size_t slot = 0; slot < 4; ++slot) {
+      auto& peer = fixture.peers[slot]; peer.udp->bind(*peer.tcp,server,peer.session);
+      const auto status = server.room(fixture.id).status();
+      isolation_need(status == (slot == 3 ? "RUNNING" : "LOBBY"),"unchanged all-joined UDP readiness before start");
+      bindings.push_back(Json{{"player_id",peer.player},{"status",status},{"bound_reply_received",true}});
+    }
+    fixture.initial = isolation_state(server.room(fixture.id)); fixture.initial_hash = sha256(canonical_state(server.room(fixture.id)));
+    const auto received = IsolationFixture::received(server);
+    for (auto& peer : fixture.peers) consume(fixture,peer);
+    ingress(received+4);
+    for (auto& peer : fixture.peers) isolation_need(IsolationFixture::acknowledged(server,peer.connection) == 1,"initial FULL1 actually applied and ACKed");
+    setup.push_back(Json{{"created",created},{"joins",joins},{"bindings",bindings}});
+  }
+  const auto mapping = IsolationFixture::mapping(server), initialized = IsolationFixture::schedule(server);
+  isolation_need(mapping.size() == 32 && initialized.at("scheduled_room_ids").size() == 32 && executed_clock.now_ms == 0 && shared_now == 0,
+    "one server owns32 distinct live Rooms without simulation during initialization");
+  auto descriptors = server.owned_descriptors();
+  for (const auto& fixture : rooms) for (const auto& peer : fixture.peers) { descriptors.push_back(peer.tcp->descriptor()); descriptors.push_back(peer.udp->descriptor()); }
+  const auto input = [&](std::size_t index, std::size_t slot, std::uint64_t seq, int tick) {
+    const auto& peer = rooms[index].peers[slot];
+    return Json{{"v",1},{"type","INPUT"},{"session_id",peer.session},{"room_id",rooms[index].id},{"player_id",peer.player},
+      {"seq",seq},{"target_tick",tick},{"direction",scenario.at("rooms").at(index).at("players").at(slot).at("direction")},
+      {"tag_target_player_id",nullptr},{"owner_epoch",0}};
+  };
+  Json groups = Json::array(); std::size_t hot_accepted = 0, hot_rejected = 0;
+  const auto burst = [&](int index) {
+    const int target = scenario.at("hot_room").at("target_ticks").at(static_cast<std::size_t>(index)).get<int>();
+    isolation_need(shared_now == scenario.at("hot_room").at("burst_at_ms").at(static_cast<std::size_t>(index)) &&
+      server.room(rooms[0].id).executed_ticks() == target && server.room(rooms[0].id).pending_count() == 0,"burst at fixed service boundary and next tick");
+    for (int group = 0; group < 16; ++group) {
+      const auto received = IsolationFixture::received(server); const auto before_time = shared_now;
+      std::array<std::array<Json,4>,4> requests; Json admissions = Json::array();
+      for (std::size_t slot = 0; slot < 4; ++slot) for (int ordinal = 0; ordinal < 4; ++ordinal) {
+        auto& request = requests[slot][static_cast<std::size_t>(ordinal)];
+        request = input(0,slot,static_cast<std::uint64_t>(index*64+group*4+ordinal+1),target); rooms[0].peers[slot].udp->send(request);
+      }
+      ingress(received+16);
+      for (std::size_t slot = 0; slot < 4; ++slot) for (std::size_t ordinal = 0; ordinal < 4; ++ordinal) {
+        auto& peer = rooms[0].peers[slot]; const auto& request = requests[slot][ordinal];
+        const auto reply = group == 0 ? peer.udp->receive_type(server,"INPUT_ACK") : peer.tcp->receive_type(server,"ERROR");
+        if (group == 0) {
+          isolation_need(reply.at("accepted") && reply.at("code") == "ACCEPTED" && reply.at("seq") == request.at("seq") &&
+            reply.at("tick") == target && reply.at("player_id") == peer.player,"real first-four owner admission"); ++hot_accepted;
+        } else { isolation_need(reply.at("code") == "INPUT_RATE_EXCEEDED","all remaining60 attempts retain G06 rate policy"); ++hot_rejected; }
+        admissions.push_back(Json{{"request",redact(request,peer.player)},{"response",reply}});
+      }
+      const auto& hot = server.room(rooms[0].id);
+      isolation_need(shared_now == before_time && hot.executed_ticks() == target && hot.pending_count() == 16,"no tick/time change or pending growth within transport groups");
+      for (const auto& peer : rooms[0].peers) isolation_need(hot.players().at(peer.player).last_accepted_seq() == static_cast<std::uint64_t>(index*64+4) &&
+        !peer.tcp->try_receive() && !peer.udp->try_receive(),"all group ACK/errors consumed before next group");
+      groups.push_back(Json{{"burst",index},{"group",group},{"monotonic_ms",shared_now},{"target_tick",target},
+        {"pending_inputs",hot.pending_count()},{"admissions",admissions}});
+    }
+  };
+  const auto normal_inputs = [&](int tick) {
+    // Ordinary consumers continue pumping the real shared transport. All124
+    // inputs are accepted before the one scheduler call; time never advances
+    // between these Room groups, and none of their pending work is executed.
+    for (std::size_t i = 1; i < rooms.size(); ++i) {
+      const auto received = IsolationFixture::received(server);
+      for (std::size_t slot = 0; slot < 4; ++slot)
+        rooms[i].peers[slot].udp->send(input(i,slot,static_cast<std::uint64_t>(tick+1),tick));
+      ingress(received+4);
+      for (std::size_t slot = 0; slot < 4; ++slot) {
+        auto& peer = rooms[i].peers[slot]; const auto reply = peer.udp->receive_type(server,"INPUT_ACK");
+        isolation_need(reply.at("accepted") && reply.at("code") == "ACCEPTED" && reply.at("seq") == tick+1 &&
+          reply.at("tick") == tick && reply.at("player_id") == peer.player,"normal Room input admitted despite hot ingress");
+        rooms[i].admissions.push_back(Json{{"request",redact(input(i,slot,static_cast<std::uint64_t>(tick+1),tick),peer.player)},
+          {"response",reply},{"admitted_at_ms",shared_now}});
+      }
+    }
+  };
+  const auto consume_scheduled = [&] {
+    const auto received = IsolationFixture::received(server); std::size_t sent = 0;
+    for (auto& fixture : rooms) for (auto& peer : fixture.peers) {
+      const auto expected = static_cast<std::uint64_t>(server.room(fixture.id).executed_ticks()/2+1);
+      while (peer.latest < expected) { consume(fixture,peer); ++sent; }
+    }
+    ingress(received+sent);
+    for (auto& fixture : rooms) if (server.room(fixture.id).status() == "RUNNING") for (auto& peer : fixture.peers)
+      isolation_need(IsolationFixture::acknowledged(server,peer.connection) == peer.latest,"actual applied snapshot ACK watermark before next event");
+  };
+  burst(0); Json timeline = Json::array(), terminal;
+  int normal_tick = 0, service_index = 0;
+  for (const auto time : events) {
+    const bool deadline = std::find(deadlines.begin(),deadlines.end(),time) != deadlines.end();
+    if (deadline) normal_inputs(normal_tick);
+    shared_now = time; const auto before = IsolationFixture::schedule(server);
+    Json before_terminal; if (time == 1200) before_terminal = isolation_state(server.room(rooms[0].id));
+    const auto batch = server.run_scheduler(); const auto after = IsolationFixture::schedule(server);
+    isolation_need(after.at("scheduler_iterations").get<std::uint64_t>() == before.at("scheduler_iterations").get<std::uint64_t>()+1 &&
+      after.at("monotonic_reads").get<std::uint64_t>() == before.at("monotonic_reads").get<std::uint64_t>()+1,"one scheduler iteration and one monotonic read per fixed event");
+    if (deadline) ++normal_tick;
+    for (std::size_t i = 1; i < rooms.size(); ++i) isolation_need(server.room(rooms[i].id).executed_ticks() == normal_tick &&
+      server.room(rooms[i].id).status() == "RUNNING" && after.at("rooms").at(rooms[i].id).at("missed_deadlines") == 0,
+      "every normal Room meets every deadline on the shared scheduler");
+    const bool hot_service = std::find(services.begin(),services.end(),time) != services.end();
+    if (hot_service) ++service_index;
+    const auto& hot = after.at("rooms").at(rooms[0].id);
+    isolation_need(hot.at("executed_ticks") == service_index*4 && executed_clock.now_ms == service_index*200 &&
+      hot.at("catch_up_high_water") <= 4 && batch.ticks == (deadline ? 31 : 0)+(hot_service ? 4 : 0),"one bounded per-Room quantum with actual default executed-time counter");
+    if (time < 1200) {
+      const auto misses = time < 225 ? time/50 : time/50-4;
+      isolation_need(hot.at("status") == "RUNNING" && hot.at("remaining_ms") == time-service_index*200 && hot.at("missed_deadlines") == misses &&
+        hot.at("operational_state") == (time < 450 ? "NORMAL" : "OVERLOADED"),"retained backlog, real deadline streak and operational flag");
+    }
+    consume_scheduled();
+    if (time == 1200) {
+      const auto closed = isolation_state(server.room(rooms[0].id)); Json reports = Json::array(), closed_server_descriptors = Json::array();
+      for (auto& peer : rooms[0].peers) {
+        const auto report = peer.tcp->receive_type(server,"ERROR");
+        isolation_need(report.at("code") == "ROOM_OVERLOAD","only hot Room receives explicit terminal error");
+        wait_for([&] { return peer.tcp->peer_closed(); }); reports.push_back(report);
+      }
+      for (const auto& p : mapping.at(0).at("players")) {
+        const auto fd = p.at("server_descriptor").get<int>();
+        isolation_need(descriptor_closed(fd),"all four hot server sockets close at the actual terminal boundary");
+        closed_server_descriptors.push_back(fd);
+      }
+      isolation_need(before_terminal.at("status") == "RUNNING" && closed.at("status") == "CLOSED" &&
+        hot.at("terminal_code") == "ROOM_OVERLOAD" && hot.at("terminal_ms") == 1200 && hot.at("missed_deadlines") == 20 &&
+        hot.at("terminal_pending_cleared") == 16 && hot.at("pending_inputs") == 0 && hot.at("resume_records") == 0 &&
+        hot.at("replay_bytes") == 0 && hot.at("replay_pending_events") == 0 && !hot.at("scheduled").get<bool>() &&
+        hot.at("next_deadline_ms").is_null() && after.at("scheduled_room_ids").size() == 31 &&
+        server.cleanup().at("server_connections") == 124,"hot-only terminal clears all16 accepted pending entries and its registry/work/transport resources");
+      for (const auto& row : before_terminal.at("players")) isolation_need(row.at("pending") == 4 && row.at("last_seq") == 324,"last between-tick admission preserved separately from tick19");
+      terminal = Json{{"before",before_terminal},{"after",closed},{"reports",reports},{"schedule",hot},
+        {"closed_server_descriptors",closed_server_descriptors},{"remaining_connections",124}};
+    }
+    timeline.push_back(Json{{"monotonic_ms",time},{"deadline",deadline},{"hot_service",hot_service},{"executed_in_iteration",batch.ticks},{"after",after}});
+    if (hot_service) burst(service_index);
+  }
+  isolation_need(normal_tick == 25 && service_index == 5 && hot_accepted == 96 && hot_rejected == 1440 && groups.size() == 96 &&
+    service_trace.size() == 32*27+31,"exact live campaign, service decisions and hot attempt counts");
+  const auto metrics = server.metrics(); Json room_results = Json::object(); std::size_t total_ticks = 0;
+  for (std::size_t i = 0; i < rooms.size(); ++i) {
+    auto& fixture = rooms[i]; const auto& model = server.room(fixture.id); const auto& last = fixture.records.back().at("state");
+    const int ticks = i == 0 ? 20 : 25; total_ticks += fixture.records.size();
+    isolation_need(fixture.records.size() == static_cast<std::size_t>(ticks) && model.executed_ticks() == ticks,"exact per-Room actual tick count");
+    for (std::size_t slot = 0; slot < 4; ++slot) {
+      const auto& p = last.at("players").at(slot); const int distance = ticks*400;
+      const int x = slot == 0 ? 10000+distance : slot == 1 ? 90000-distance : slot == 2 ? 10000 : 90000;
+      const int y = slot == 0 ? 10000 : slot == 1 ? 90000 : slot == 2 ? 90000-distance : 10000+distance;
+      isolation_need(p.at("x") == x && p.at("y") == y && p.at("score") == 0 && p.at("pending") == 0 &&
+        p.at("last_seq") == (i == 0 ? 260 : 25),"actual final historical positions, pending and accepted sequence");
+    }
+    Json hashes = Json::array(), publications = Json::object();
+    for (const auto& row : fixture.records) hashes.push_back(row.at("state_hash"));
+    for (auto& peer : fixture.peers) {
+      isolation_need(peer.latest == static_cast<std::uint64_t>(ticks/2+1) && peer.snapshots.size() == peer.latest &&
+        !peer.udp->try_receive() && !peer.tcp->try_receive(),"every expected healthy snapshot and control was consumed once");
+      publications[peer.player] = peer.snapshots;
+    }
+    const auto stem = output.stem().string()+"."+fixture.id;
+    const auto records_path = output.parent_path()/(stem+".records.json");
+    write_json_file(records_path,Json{{"room_id",fixture.id},{"initial_state",fixture.initial},{"initial_hash",fixture.initial_hash},
+      {"ticks",fixture.records},{"input_admissions",fixture.admissions},{"snapshots",publications}});
+    Json result{{"executed_ticks",ticks},{"canonical_records",std::filesystem::absolute(records_path).string()},
+      {"state_hashes",hashes},{"last_tick_state",last},{"final_state",isolation_state(model)},
+      {"snapshots_per_player",ticks/2+1},{"schedule",metrics.at("rooms").at(fixture.id)}};
+    if (i != 0) {
+      const auto bytes = server.replay(fixture.id).serialize(); const auto artifact = parse_replay_artifact(bytes);
+      std::size_t accepted = 0; for (const auto& tick : artifact.at("ticks")) accepted += tick.at("events").size();
+      isolation_need(artifact.at("ticks").size() == 25 && accepted == 100 && fixture.admissions.size() == 100,"one actual25-tick accepted journal for each normal Room");
+      const auto replay_path = output.parent_path()/(stem+".replay.json"); std::ofstream file(replay_path,std::ios::binary);
+      isolation_need(file.good() && bytes.size() <= max_replay_bytes,"bounded immutable accepted-journal output");
+      file << bytes; file.close(); isolation_need(file.good(),"accepted journal persisted");
+      result.update(Json{{"replay_artifact",std::filesystem::absolute(replay_path).string()},{"artifact_bytes",bytes.size()},
+        {"artifact_sha256",sha256(bytes)},{"accepted_journal_events",accepted}});
+    }
+    room_results[fixture.id] = std::move(result);
+  }
+  isolation_need(total_ticks == 795 && metrics.at("room_high_water") == 32 && metrics.at("connection_high_water") == 128 &&
+    metrics.at("total_pending_high_water") <= max_total_pending_inputs && metrics.at("input_per_player_high_water") <= max_pending_inputs &&
+    metrics.at("mailbox_high_water") <= max_mailbox_messages && metrics.at("outbound_control_high_water") < max_control_messages &&
+    !metrics.at("errors").contains("CONTROL_BACKPRESSURE") && !metrics.at("errors").contains("UDP_SEND_DROP") &&
+    metrics.at("errors").at("INPUT_RATE_EXCEEDED") == 1440 && metrics.at("errors").at("ROOM_OVERLOAD") == 1,"actual global resource bounds without extra capacity campaigns");
+  const auto before_shutdown = server.cleanup(); server.shutdown();
+  for (auto& fixture : rooms) for (auto& peer : fixture.peers) { peer.tcp->close(); peer.udp->close(); }
+  for (const int fd : descriptors) isolation_need(descriptor_closed(fd),"every actual client/server descriptor closed");
+  auto cleanup = server.cleanup(); for (const auto& [key,value] : cleanup.items()) { (void)key; isolation_need(value == 0,"all actual Room registries, deadlines, queues and transports released"); }
+  const auto shutdown_metrics = server.metrics();
+  isolation_need(Fd::live() == fd_before && shutdown_metrics.at("outbound_buffers_created") == shutdown_metrics.at("outbound_buffers_released"),"all owned descriptors and buffer lifetimes end");
+  cleanup.update(Json{{"descriptor_checks",descriptors.size()},{"all_descriptors_closed",true},{"tracked_descriptor_delta",0}});
+  const auto hot_path = output.parent_path()/(output.stem().string()+".hot-inputs.json");
+  write_json_file(hot_path,groups);
+  return Json{{"result","PASS"},{"thread","G13"},{"scenario_id",scenario.at("scenario_id")},{"process_id",::getpid()},
+    {"server_instances",1},{"executed_ticks",total_ticks},{"normal_executed_ticks",775},{"hot_executed_ticks",20},
+    {"shared_monotonic_ms",shared_now},{"default_executed_simulation_ms",executed_clock.now_ms},{"setup",setup},
+    {"owner_mapping",mapping},{"initial_schedule",initialized},{"clock_trace",timeline},{"service_trace",service_trace},
+    {"hot_burst_groups",std::filesystem::absolute(hot_path).string()},{"hot_burst_group_count",groups.size()},
+    {"hot_accepted",hot_accepted},{"hot_rate_rejections",hot_rejected},{"hot_terminal",terminal},
+    {"rooms",room_results},{"metrics",metrics},{"before_shutdown",before_shutdown},
+    {"outbound_buffers_created",shutdown_metrics.at("outbound_buffers_created")},{"outbound_buffers_released",shutdown_metrics.at("outbound_buffers_released")},
+    {"reference_processes_reserved",31},{"reference_ticks_reserved",775},{"network_fault_runs",0},{"load_runs",0},{"cleanup",cleanup}};
+}
+}
diff --git a/tests/g13.hpp b/tests/g13.hpp
new file mode 100644
index 0000000..35aa730
--- /dev/null
+++ b/tests/g13.hpp
@@ -0,0 +1,5 @@
+#pragma once
+#include "g07.hpp"
+namespace arena {
+Json run_isolation_scenario(const Json& scenario, const std::filesystem::path& output);
+}
diff --git a/tests/scenario_main.cpp b/tests/scenario_main.cpp
index 509463b..f1b1fa8 100644
--- a/tests/scenario_main.cpp
+++ b/tests/scenario_main.cpp
@@ -3,6 +3,7 @@
 #include "g10.hpp"
 #include "g11.hpp"
 #include "g12.hpp"
+#include "g13.hpp"
 #ifndef ARENA_TEST_FIXTURES
 #error Scenario fixture executable requires its separate test core
 #endif
@@ -38,7 +39,8 @@ int main(int argc, char** argv) {
         if (variant) throw std::invalid_argument("variant is only active for G07");
         const auto evidence = scenario.at("thread") == "G08" ? arena::run_snapshot_scenario(scenario) :
           scenario.at("thread") == "G10" ? arena::run_ack_scenario(scenario) :
-          scenario.at("thread") == "G11" ? arena::run_resume_scenario(scenario) : arena::run_scenario(scenario);
+          scenario.at("thread") == "G11" ? arena::run_resume_scenario(scenario) :
+          scenario.at("thread") == "G13" ? arena::run_isolation_scenario(scenario,output) : arena::run_scenario(scenario);
         arena::write_json_file(output,evidence);
         std::cout << arena::Json{{"result",evidence.at("result")},{"executed_ticks",evidence.at("executed_ticks")},
           {"scenario_id",evidence.at("scenario_id")},{"evidence",output.string()},{"cleanup",evidence.at("cleanup")}}.dump() << '\n';
diff --git a/tests/tests.cpp b/tests/tests.cpp
index f00f98e..bcaecb7 100644
--- a/tests/tests.cpp
+++ b/tests/tests.cpp
@@ -316,6 +316,24 @@ Input parsed_input(const std::string& seq) {
   check(parsed.value.at("seq").is_number_unsigned(), "positive sequence remained an exact unsigned JSON integer");
   return decode_input(parsed.value);
 }
+void instance_pending_admission_without_ticks() {
+  Room room; populate(room); const auto initial = room.view();
+  const auto request = Json::parse(input_wire("1"));
+  const auto denied = admit_input(room,"player-00",request,false);
+  check(denied.error == "ADMISSION_REJECTED" && room.pending_count() == 0 && room.view() == initial &&
+    room.players().at("player-00").last_accepted_seq() == 0,"instance guard denies new storage without authoritative sequence/state change");
+  const auto accepted = admit_input(room,"player-00",request,true);
+  const auto retry = admit_input(room,"player-00",request,false);
+  const auto overflow = admit_input(room,"player-00",Json::parse(input_wire("2")),false);
+  check(!accepted.error && !accepted.duplicate && !retry.error && retry.duplicate && overflow.error == "ADMISSION_REJECTED" &&
+    room.pending_count() == 1 && room.players().at("player-00").last_accepted_seq() == 1 && room.executed_ticks() == 0,
+    "exact retry consumes no aggregate capacity; a later sequence cannot replace accepted work");
+  room.close();
+  check(room.pending_count() == 0,"pure admission guard releases accepted storage");
+  std::cout << Json{{"G13_zero_tick_instance_guard",Json{{"room_limit",max_rooms},{"connection_limit",max_connections},
+    {"pending_per_player",max_pending_inputs},{"total_pending_limit",max_total_pending_inputs},
+    {"first_rejection",*denied.error},{"retry_duplicate",retry.duplicate},{"next_rejection",*overflow.error},{"executed_ticks",0}}}}.dump() << '\n';
+}
 void fixed_pending_sequence_bound() {
   Room room; populate(room);
   const auto& player = room.players().at("player-00");
@@ -811,6 +829,7 @@ int main(int argc, char** argv) {
       {"G07_zero_tick_canonical_SHA256", canonical_hash_bytes_without_ticks},
       {"G07_zero_tick_storage_packaging", replay_storage_and_packaging_without_ticks},
       {"G08_zero_tick_33_snapshot_retention", snapshot_retention_without_ticks},
+      {"G13_zero_tick_instance_admission", instance_pending_admission_without_ticks},
       {"G12_zero_tick_control_threshold", [] { std::cout << Json{{"G12_queue_probe",run_control_queue_probe()}}.dump() << '\n'; }},
       {"G12_zero_tick_mixed_snapshot_coalescing", [] { std::cout << Json{{"G12_queue_probe",run_snapshot_queue_probe()}}.dump() << '\n'; }}};
   } else if (std::string(argv[1]) == "integration") {
