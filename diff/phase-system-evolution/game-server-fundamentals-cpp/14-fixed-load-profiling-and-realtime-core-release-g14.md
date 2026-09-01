# Fixed Load, Profiling과 Realtime Core Release (G14)

## `test(release): verify fixed realtime-core load and resource bounds`

diff --git a/.gitignore b/.gitignore
index e878ad2..a5e52e3 100644
--- a/.gitignore
+++ b/.gitignore
@@ -2,3 +2,4 @@ build*/
 artifacts/
 *.dSYM/
 .DS_Store
+/evidence/G14-runs.jsonl
diff --git a/CMakeLists.txt b/CMakeLists.txt
index 4dd493d..ca7afbd 100644
--- a/CMakeLists.txt
+++ b/CMakeLists.txt
@@ -34,7 +34,7 @@ target_compile_options(arena PRIVATE -Wall -Wextra -Wpedantic -Werror)
 add_executable(arena_tests tests/tests.cpp tests/g09.cpp tests/g12_queue.cpp)
 target_link_libraries(arena_tests PRIVATE arena_test_core)
 target_compile_options(arena_tests PRIVATE -Wall -Wextra -Wpedantic -Werror)
-add_executable(arena_scenarios tests/scenario_main.cpp tests/g07.cpp tests/g09.cpp tests/g10.cpp tests/g11.cpp tests/g12.cpp tests/g12_queue.cpp tests/g13.cpp)
+add_executable(arena_scenarios tests/scenario_main.cpp tests/g07.cpp tests/g09.cpp tests/g10.cpp tests/g11.cpp tests/g12.cpp tests/g12_queue.cpp tests/g13.cpp tests/g14.cpp)
 target_link_libraries(arena_scenarios PRIVATE arena_test_core)
 target_compile_options(arena_scenarios PRIVATE -Wall -Wextra -Wpedantic -Werror)
 enable_testing()
diff --git a/TRACK.md b/TRACK.md
index f81dd74..9d34b35 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -1,9 +1,13 @@
-# fundamentals-cpp — G08 acknowledged full/delta snapshots
+# fundamentals-cpp — G14 realtime-core
 
 SPEC_REVISION: `c1d62196ab76b55652f5d75a67514f8c6d8210ce` (phase-1; earlier evidence retains its original revision)
 
 Completion profile: `realtime-core`
 
+Phase1 implements G01–G14 only: bounded single-owner Rooms, authoritative UDP
+inputs, replay/hash verification, acknowledged snapshots, reconnect grace,
+slow-consumer coalescing and isolated multi-Room scheduling. No phase2 services.
+
 Branch: `track/fundamentals-cpp` (orphan)
 
 ## Frozen toolchain and dependency
@@ -31,15 +35,30 @@ Run from this worktree; `track` resolves its own source directory. Build never r
 ./track scenario-run /absolute/path/to/G07.json /absolute/path/to/L1.json
 ./track scenario-run /absolute/path/to/G07.json /absolute/path/to/V.json --variant rejected-removed
 ./track scenario-run /absolute/path/to/G08.json /absolute/path/to/evidence.json
+./track scenario-run /absolute/path/to/G09.json /absolute/path/to/evidence.json
+./track scenario-run /absolute/path/to/G10.json /absolute/path/to/evidence.json
+./track scenario-run /absolute/path/to/G11.json /absolute/path/to/evidence.json
+./track scenario-run /absolute/path/to/G12.json /absolute/path/to/evidence.json
+./track scenario-run /absolute/path/to/G13.json /absolute/path/to/evidence.json
+./track scenario-run /absolute/path/to/G14.json /absolute/path/to/result.json
 ./track replay-verify /absolute/path/to/replay.json /absolute/path/to/evidence.json
 ./track server /absolute/path/to/config.json
 ```
 
-`scenario-run` and `replay-verify` use `arena_scenarios`, a separately compiled test executable. G07 writes sibling `.replay.json` and `.records.json` files beside its evidence. Replay verification runs once, returns0 on equality and1 on the first divergence, and reports the actual canonical record. Each invocation is a separate process; no command silently repeats a campaign.
+`scenario-run` and `replay-verify` use `arena_scenarios`, a separately compiled test executable. The G14 coordinator dispatches G01–G13 to that same executable without extra simulation. G07 writes sibling `.replay.json` and `.records.json` files beside its evidence. Replay verification runs once, returns0 on equality and1 on the first divergence, and reports the actual canonical record.
 The V option reads the original scenario and removes only its explicitly marked rejected inputs, preserving accepted TAG failures and lifecycle events. The ordinary `arena` server has no fixed-ID/roster initialization code or runtime fixture switch.
 `ARENA_BUILD_DIR` selects a clean build directory. `ARENA_EVIDENCE_DIR` selects command logs.
 The build wrapper invokes configure and compile separately; both are logged. Tests invoke the existing executable directly.
 
+G14 `scenario-run` owns exactly one32-Room×600-tick live process, followed by31
+separate600-tick accepted-journal replays after its measured interval and cleanup.
+Use a new output directory for each reserved run. Its default does not start a
+profiler; only the explicitly reserved profiled pass adds `--profile` for one
+native `sample` capture. All G14 runs require main's durable slot launcher and
+the fixed shared fixture; the exact approved argv are in `artifacts/g14/commands.json`.
+Unit/integration suites do not invoke this load. Baseline, profiled, worker-final
+and main-final are the four total slots across revisions/repairs, with no warmup.
+
 Exact Release build commands:
 
 ```sh
@@ -52,6 +71,7 @@ The fixed ownership test is also run with the available Apple ThreadSanitizer:
 ```sh
 ARENA_BUILD_DIR="$PWD/build-tsan" ARENA_TSAN=ON ./track build
 ARENA_BUILD_DIR="$PWD/build-tsan" TSAN_OPTIONS=halt_on_error=1 ./track unit-test
+ARENA_BUILD_DIR="$PWD/build-tsan" TSAN_OPTIONS=halt_on_error=1 ./track integration-test
 ```
 
 CMake's normal `CXXFLAGS`/`LDFLAGS` also support independent instrumentation. The `ARENA_TSAN` option only adds compiler/linker flags.
@@ -65,20 +85,20 @@ Operator commands are local process control, not extra wire protocol messages. I
 ## Current guarantee and ownership
 
 G01 establishes a baseline; no previous implementation was damaged or modified to manufacture a failure.
-One process owns one Room. Join commit order determines spawn slots. The second TCP-ready player starts tick 0.
+One process owns at most64 separate Room models and scheduling contexts on one owner thread. Join commit order determines spawn slots. A Room starts when at least two players have joined and every joined player has a valid UDP binding.
 Only integer arithmetic updates movement, clamp, TAG and score. TAG is one-shot; direction persists.
 The session executes ticks 0–1199, then sends authoritative `ROOM_FINISHED` and stops advancing.
 
 G02 replaces the complete-read assumption with one `FrameParser` per Connection. It consumes one bounded frame at a time and returns the consumed-byte count so a coalesced suffix is processed in order. Incomplete headers and payloads remain with their connection. Parser states are `need_more`, `message`, `message_error`, `terminal_frame_error` and `io_end`.
 
-Strict UTF-8 JSON decoding rejects duplicate keys at any object depth, invalid root/envelope/schema types, NaN, Infinity and trailing JSON. Unknown fields on the five active client message types remain ignored. Sequence, target tick and future message families stay inactive. Message errors enter the bounded owner mailbox in stream order and allow the next frame. Invalid lengths attempt one bounded nonblocking error flush and then close only that connection. Error text is fixed and bounded; parser exception text is never serialized.
+Strict UTF-8 JSON decoding rejects duplicate keys at any object depth, invalid root/envelope/schema types, NaN, Infinity and trailing JSON. Unknown forward-compatible fields remain ignored. Authenticated INPUT attempts are counted before full intent validation at the owner boundary; unsigned64 sequence, integer target window and duplicate/conflict rules remain active. Message errors allow the next valid frame. Invalid lengths attempt one bounded nonblocking error flush and close only that connection. Error text is fixed and bounded; parser exception text is never serialized.
 
 Clean EOF is `TRANSPORT_EOF`; partial header/payload EOF is `TRANSPORT_EOF_IN_FRAME`; failed I/O is `TRANSPORT_IO_ERROR`. All terminate the transport and clear retained bytes. These transport observations do not change Room lifecycle policy.
 
 - Connection lifetime: `Server::connections_`, each descriptor owned by move-only `Fd`.
 - Parser lifetime: the Connection owns a fixed 16,388-byte array and parser state; erasing the Connection releases that storage. No frame-sized allocation is made from an unchecked length, and no unbounded decoded-message list is accumulated.
-- Session identity: server-generated opaque per-connection identifier; no reconnect credential or persistence in G01.
-- Room state: the server's one execution context, enforced by Room owner-thread checks.
+- Session identity: server-generated opaque control session, one-time endpoint binding and rotated resume credentials; reconnect preserves the player within its200-tick grace window.
+- Room state: a distinct context per Room, enforced by owner-thread checks.
 - Mailbox: kqueue read callback produces bounded `Envelope` messages; `drain_mailbox` consumes them after I/O callbacks return. Only that owner phase or explicit tick mutates Room.
 - Serialization: each queued `PendingWrite` owns its byte vector and unsent offset until write completes. Nonblocking partial writes and EAGAIN retain the suffix.
 - Shutdown coordinator: `Server::shutdown`; it stops accepting, drains intent, closes Room, attempts bounded final control flushing, releases connections and kqueue, and clears queues.
@@ -94,15 +114,17 @@ G08 replaces the earlier debug state notices with contract-shaped sequenced snap
 | Parser buffer | exactly 16,388 bytes of owned storage per connection; partial data retained only within this bound |
 | Read buffer | separate 16,388-byte stack array per read; coalesced suffix consumed without growing parser storage |
 | Connections | 512; excess accepted descriptor closed and ADMISSION_REJECTED metric recorded |
-| Rooms | one; extra create gets ROOM_NOT_JOINABLE |
+| Rooms |64 per instance; fresh sessions may create, joined sessions remain ROOM_NOT_JOINABLE; capacity rejects ADMISSION_REJECTED |
 | Players | eight slots maximum; running Room rejects new joins |
 | Transport mailbox | 512 messages total, 64 per connection; INPUT_QUEUE_FULL |
-| Pending inputs | 64 per player; 65th is INPUT_QUEUE_FULL, accepted inputs retained until tick |
-| Outbound control | 64 messages per connection, each bounded frame; overflow disconnect with CONTROL_BACKPRESSURE metric |
+| Pending inputs |64 per player,32768 per instance; first-four-attempt rate gate precedes validation; accepted work retained until tick |
+| Outbound control |64 messages per connection; the64th pending attempt becomes terminal CONTROL_BACKPRESSURE, never a65th buffer |
+| Pending snapshots |one FULL and one DELTA per connection; superseded vectors released, controls preserved |
+| UDP payload |1200 bytes maximum; actual compact snapshot serializer and ingress enforced |
 | I/O iteration | 64 kqueue events, up to 64 accepts, up to 64 writes per ready connection |
-| Scheduler iteration | at most four 50ms ticks; unexecuted elapsed time remains in the integer accumulator and exposes OVERLOADED |
+| Scheduler iteration |one shared monotonic observation; at most four50ms ticks per Room; retain backlog, expose OVERLOADED and close only the Room after20 unrecovered deadlines |
 | Client evidence | 4,096 received messages per client; overflow is a test failure |
-| Runner input/output | 1 MiB JSON input, 512 input commands, 32 setup commands; 4 MiB evidence output |
+| Runner input/output |1MiB JSON input; legacy command/setup bounds unchanged; every JSON/raw stream output bounded to4MiB |
 | Replay capture/intake | 4,194,304 bytes including final LF; overflow latches incomplete capture and refuses export, while gameplay/hash computation continue |
 | Snapshot retention | newest32 full materializations per connected player stream; missing/unknown/evicted acknowledged base selects the next scheduled FULL |
 | Operator input | 4,096 bytes; overflow terminates with explicit failure |
@@ -121,7 +143,7 @@ Human-readable actual results are in `evidence/G01.md`; generated binaries and d
 The canonical scenario is supplied by main and read at runtime. The runner does not contain canonical positions/scores as output constants.
 It resolves role names to server-generated session/player IDs, sends complete frames over loopback, waits for owner-phase INPUT_ACK at each intended tick boundary, then advances the injected manual clock. It compares final TCP messages against the same production Room's view, checks client-observed EOF and authoritative CLOSED, and verifies actual descriptor invalidation using `fcntl(F_GETFD)`.
 
-## Deliberate next constraints
+## Historical stage evidence
 
 The G01 complete-frame limitation is preserved in the pre-change G02 reproduction evidence. G02's scenario runner reads main's frozen input, checks all nine parser/split combinations, forces those same chunks through real TCP by observing retained bytes before each next write, sends the concatenated pair in one actual write, and isolates each of the four malformed peers beside one persistent healthy connection. Fixture session identifiers intentionally reach existing SESSION_INVALID checks after valid parsing; the runner never substitutes server-specific input bytes.
 
@@ -133,7 +155,7 @@ G02 initial ceilings: 8 configure/compile invocations, 4 unit invocations includ
 
 The unchanged G02 implementation passed the fixed identity, six lifecycle cells and held-consumer owner reproduction. G03 adds focused regressions and a canonical runner, not a new identity system. The existing deque admission/take path is extracted into the private `Server::Mailbox` solely so a pure test can exercise the real 512 bound with exactly 513 attempts. Capacity, per-connection 64 bound, `INPUT_QUEUE_FULL`, FIFO order and owner dispatch remain unchanged.
 
-Connection generation identifies transport lifetime; HELLO creates its distinct opaque session identifier. A successful join commits a separate player identifier and stable slot. Room identifiers are separate opaque values. No resume credential is active. Network parsing/admission cannot mutate Room; `drain_mailbox` and explicit ticks belong to its recorded owner thread. The existing foreign-thread rejection test remains active.
+Connection generation identifies transport lifetime; HELLO creates its distinct opaque session identifier. A successful join commits a separate player identifier and stable slot. Room identifiers are separate opaque values. Network parsing/admission cannot mutate Room; `drain_mailbox` and explicit ticks belong to its recorded owner thread. The existing foreign-thread rejection test remains active. This table records G03's single-Room stage; G11 later adds disconnect grace and G13 permits fresh unjoined sessions to create other Rooms while preserving joined-session rejection.
 
 | Room state | CREATE_ROOM | JOIN_ROOM | Valid owner's LEAVE_ROOM / connection close |
 |---|---|---|---|
@@ -155,7 +177,7 @@ G04 adds a small integer `FixedTickAccumulator` and `Server::run_scheduler`. The
 
 The production clock is `std::chrono::steady_clock`; the canonical runner injects a manual monotonic reader into the same Server path. The fixed deltas yield `[1,1,2,0,4,2]` ticks and `[0,0,25,25,50,0]` remaining milliseconds. A wall-clock reversal is recorded only as external evidence. The complete unit suite preserves prior regressions, and a separate monotonic-mode execution extends the existing standalone shutdown integration test to verify actual adapter reads in the CLI scheduler.
 
-G05 sequence/target-tick and G06 four-attempt/authority guarantees remain active with their preserved regressions. UDP, reconnect, slow-consumer coalescing and many-room scheduling remain inactive.
+G05 sequence/target-tick and G06 four-attempt/authority guarantees remain active with their preserved regressions. G09–G13 add the transport, grace, coalescing and scheduling guarantees described above without changing the fixed gameplay rules.
 
 ## G07 replay boundary
 
@@ -163,7 +185,7 @@ The owner records every newly accepted canonical input at its original admission
 
 Replay files contain the initial player/slot/spawn mapping, contract/clock/session constants, accepted input and lifecycle events per tick, resulting hashes and the completed tick count. Offline verification reconstructs only the initial model, then uses the existing admission, leave and simulation functions; it never injects expected resulting state. Explicitly incomplete, malformed and oversized intake fails. Capture bytes and pending events are released at server shutdown.
 
-The G07 live fixture establishes four real TCP/session bindings before one unchanged owner start-condition evaluation in `arena_test_core`. Only that target defines `ARENA_TEST_FIXTURES`; `arena_core`/`arena` cannot link the bootstrap. Public joins still start at two and reject further RUNNING joins. Full suites retain earlier regressions and add only zero-tick G07 checks. The five200-tick live/replay/variant passes and separate38-tick negative probe are explicit commands, recorded in `evidence/G07-runs.jsonl`; results are in `evidence/G07.md`.
+The historical G07 live fixture establishes four real TCP/session bindings before one owner start-condition evaluation in `arena_test_core`. Only that target defines `ARENA_TEST_FIXTURES`; `arena_core`/`arena` cannot link the bootstrap. G09 later requires all joined UDP endpoints ready, and RUNNING still rejects new joins. Full suites retain earlier regressions and add only zero-tick G07 checks. The five200-tick live/replay/variant passes and separate38-tick negative probe are explicit commands, recorded in `evidence/G07-runs.jsonl`; results are in `evidence/G07.md`.
 
 ## G08 snapshot boundary
 
@@ -172,3 +194,24 @@ Each Connection owns its sequence, latest ACK and32 immutable full materializati
 Visible rows contain exactly `player_id`, `slot`, `x`, `y`, `direction`, `score`, `connectivity`, sorted by player ID and excluding LEFT records. Snapshot metadata carries the full canonical server hash, including internal sequence/cooldown and historical LEFT records. The client application check compares only the visible projection; it does not pretend those seven fields can reconstruct the complete canonical hash.
 
 The frozen G08 command executes196 real ticks once, captures99 snapshots for every remaining client, applies actual FULL/DELTA messages and sends the specified ACKs. Raw alpha captures include applied/authoritative visible state, canonical records/hashes and retained base IDs; all clients' cadence and feedback are retained. Earlier harnesses drain new periodic traffic at existing simulation boundaries and preserve gameplay, owner-order, lifecycle and cleanup assertions. Full unit adds one33-publication, zero-tick retention probe, not another canonical campaign. Evidence and exact command counts are in `evidence/G08.md` and `evidence/G08-runs.jsonl`.
+
+## G09–G14 release boundary
+
+TCP owns control/lifecycle traffic; authenticated endpoint-bound UDP owns INPUT,
+INPUT_ACK and SNAPSHOT traffic. G10 monotonically acknowledges only retained
+matching publications and schedules full resync on invalid/missing bases. G11
+keeps disconnected STOP players visible for200 ticks, rotates resume credentials
+and sends the current-state FULL after a fresh endpoint bind. G12 retains/coalesces
+only actual pending snapshot vectors. G13 services bounded Room contexts fairly
+from one shared monotonic observation, with Room-local backlog/terminal cleanup.
+
+G14 adds a fixed Release measurement harness. Its snapshot-only test barrier
+holds Room0/slot0 after FULL1/ACK1 without stopping600 INPUT_ACKs or TCP controls.
+The same process includes128 test clients and their bounded observation histories,
+so CPU/RSS/malloc measurements explicitly include that overhead. Actual server
+queue/retention/journal/registry/timer counts accompany samples at initialization,
+200/400/600 ticks and shutdown. Offline replays and final export are outside the
+measured live interval. Raw per-tick records, admissions, received snapshots and
+slow pre-dequeue observations stay under `artifacts/g14`; compact results and the
+command summary are `evidence/G14.md` and `evidence/G14-commands.jsonl`. The latter
+pins the unchanged raw `evidence/G14-runs.jsonl` path, SHA and source line numbers.
diff --git a/evidence/G14-commands.jsonl b/evidence/G14-commands.jsonl
new file mode 100644
index 0000000..8dae8c1
--- /dev/null
+++ b/evidence/G14-commands.jsonl
@@ -0,0 +1,105 @@
+{"kind":"raw_ledger","path":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/evidence/G14-runs.jsonl","sha256":"cac0c7a35df9fd9c1922ab2778b23dc68f651cc256787ca2df54231f9d10df1a","records":298,"policy":"Original path and bytes preserved; start/spawn/finish events include every actual child; this file only selects each completed command."}
+{"raw_line":1,"category":"compile","argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/track","ARENA_TSAN=OFF","./track","build"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/release-build.log","exit_code":0,"label":"release-build","units":2,"started_utc":"2026-08-28T09:45:04.611874+00:00","duration_seconds":46.343789,"wrapper_pid":45420,"child_pid":45447}
+{"raw_line":4,"category":"load-live","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","scenario-run","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G14.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.json","--load-control"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:22.574752+00:00","monotonic_seconds":132003.768070875},"finished":{"utc":"2026-08-28T09:47:33.204767+00:00","monotonic_seconds":132014.398003166},"duration_seconds":10.629932,"pid":49049,"wrapper_pid":49047,"ticks":19200,"reaped":true}
+{"raw_line":7,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.room-01.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-01.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-01.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:33.334158+00:00","monotonic_seconds":132014.527395416},"finished":{"utc":"2026-08-28T09:47:33.372038+00:00","monotonic_seconds":132014.565278875},"duration_seconds":0.037883,"pid":49232,"wrapper_pid":49047,"ticks":600,"reaped":true}
+{"raw_line":10,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.room-02.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-02.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-02.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:33.387337+00:00","monotonic_seconds":132014.580571625},"finished":{"utc":"2026-08-28T09:47:33.425347+00:00","monotonic_seconds":132014.618589083},"duration_seconds":0.038017,"pid":49233,"wrapper_pid":49047,"ticks":600,"reaped":true}
+{"raw_line":13,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.room-03.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-03.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-03.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:33.440169+00:00","monotonic_seconds":132014.6334055},"finished":{"utc":"2026-08-28T09:47:33.481756+00:00","monotonic_seconds":132014.674995625},"duration_seconds":0.04159,"pid":49234,"wrapper_pid":49047,"ticks":600,"reaped":true}
+{"raw_line":16,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.room-04.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-04.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-04.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:33.496465+00:00","monotonic_seconds":132014.689715791},"finished":{"utc":"2026-08-28T09:47:33.536191+00:00","monotonic_seconds":132014.729430875},"duration_seconds":0.039715,"pid":49235,"wrapper_pid":49047,"ticks":600,"reaped":true}
+{"raw_line":19,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.room-05.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-05.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-05.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:33.550760+00:00","monotonic_seconds":132014.743996291},"finished":{"utc":"2026-08-28T09:47:33.591387+00:00","monotonic_seconds":132014.784628208},"duration_seconds":0.040632,"pid":49236,"wrapper_pid":49047,"ticks":600,"reaped":true}
+{"raw_line":22,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.room-06.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-06.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-06.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:33.607361+00:00","monotonic_seconds":132014.80059425},"finished":{"utc":"2026-08-28T09:47:33.647982+00:00","monotonic_seconds":132014.84122425},"duration_seconds":0.04063,"pid":49237,"wrapper_pid":49047,"ticks":600,"reaped":true}
+{"raw_line":25,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.room-07.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-07.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-07.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:33.663327+00:00","monotonic_seconds":132014.856561416},"finished":{"utc":"2026-08-28T09:47:33.704270+00:00","monotonic_seconds":132014.897508},"duration_seconds":0.040947,"pid":49238,"wrapper_pid":49047,"ticks":600,"reaped":true}
+{"raw_line":28,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.room-08.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-08.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-08.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:33.720984+00:00","monotonic_seconds":132014.914216625},"finished":{"utc":"2026-08-28T09:47:33.761730+00:00","monotonic_seconds":132014.95497125},"duration_seconds":0.040755,"pid":49239,"wrapper_pid":49047,"ticks":600,"reaped":true}
+{"raw_line":31,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.room-09.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-09.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-09.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:33.778117+00:00","monotonic_seconds":132014.971351291},"finished":{"utc":"2026-08-28T09:47:33.816341+00:00","monotonic_seconds":132015.009578541},"duration_seconds":0.038227,"pid":49240,"wrapper_pid":49047,"ticks":600,"reaped":true}
+{"raw_line":34,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.room-10.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-10.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-10.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:33.831958+00:00","monotonic_seconds":132015.025191166},"finished":{"utc":"2026-08-28T09:47:33.869142+00:00","monotonic_seconds":132015.062381541},"duration_seconds":0.03719,"pid":49241,"wrapper_pid":49047,"ticks":600,"reaped":true}
+{"raw_line":37,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.room-11.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-11.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-11.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:33.884574+00:00","monotonic_seconds":132015.077807708},"finished":{"utc":"2026-08-28T09:47:33.924243+00:00","monotonic_seconds":132015.11748025},"duration_seconds":0.039673,"pid":49242,"wrapper_pid":49047,"ticks":600,"reaped":true}
+{"raw_line":40,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.room-12.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-12.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-12.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:33.940236+00:00","monotonic_seconds":132015.133465958},"finished":{"utc":"2026-08-28T09:47:33.980547+00:00","monotonic_seconds":132015.173790958},"duration_seconds":0.040325,"pid":49243,"wrapper_pid":49047,"ticks":600,"reaped":true}
+{"raw_line":43,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.room-13.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-13.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-13.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:33.995758+00:00","monotonic_seconds":132015.188987291},"finished":{"utc":"2026-08-28T09:47:34.036290+00:00","monotonic_seconds":132015.229528166},"duration_seconds":0.040541,"pid":49244,"wrapper_pid":49047,"ticks":600,"reaped":true}
+{"raw_line":46,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.room-14.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-14.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-14.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:34.051999+00:00","monotonic_seconds":132015.245228458},"finished":{"utc":"2026-08-28T09:47:34.092608+00:00","monotonic_seconds":132015.285844875},"duration_seconds":0.040616,"pid":49245,"wrapper_pid":49047,"ticks":600,"reaped":true}
+{"raw_line":49,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.room-15.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-15.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-15.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:34.107616+00:00","monotonic_seconds":132015.30084475},"finished":{"utc":"2026-08-28T09:47:34.148184+00:00","monotonic_seconds":132015.341420708},"duration_seconds":0.040576,"pid":49246,"wrapper_pid":49047,"ticks":600,"reaped":true}
+{"raw_line":52,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.room-16.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-16.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-16.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:34.162715+00:00","monotonic_seconds":132015.355939875},"finished":{"utc":"2026-08-28T09:47:34.203169+00:00","monotonic_seconds":132015.396402208},"duration_seconds":0.040462,"pid":49248,"wrapper_pid":49047,"ticks":600,"reaped":true}
+{"raw_line":55,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.room-17.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-17.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-17.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:34.217366+00:00","monotonic_seconds":132015.410591708},"finished":{"utc":"2026-08-28T09:47:34.258021+00:00","monotonic_seconds":132015.451255041},"duration_seconds":0.040663,"pid":49249,"wrapper_pid":49047,"ticks":600,"reaped":true}
+{"raw_line":58,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.room-18.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-18.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-18.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:34.273699+00:00","monotonic_seconds":132015.466925458},"finished":{"utc":"2026-08-28T09:47:34.314377+00:00","monotonic_seconds":132015.50761125},"duration_seconds":0.040686,"pid":49250,"wrapper_pid":49047,"ticks":600,"reaped":true}
+{"raw_line":61,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.room-19.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-19.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-19.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:34.330043+00:00","monotonic_seconds":132015.523269583},"finished":{"utc":"2026-08-28T09:47:34.371086+00:00","monotonic_seconds":132015.564323125},"duration_seconds":0.041054,"pid":49251,"wrapper_pid":49047,"ticks":600,"reaped":true}
+{"raw_line":64,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.room-20.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-20.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-20.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:34.389809+00:00","monotonic_seconds":132015.583036208},"finished":{"utc":"2026-08-28T09:47:34.430803+00:00","monotonic_seconds":132015.624035},"duration_seconds":0.040999,"pid":49253,"wrapper_pid":49047,"ticks":600,"reaped":true}
+{"raw_line":67,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.room-21.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-21.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-21.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:34.446247+00:00","monotonic_seconds":132015.639474958},"finished":{"utc":"2026-08-28T09:47:34.486875+00:00","monotonic_seconds":132015.680108625},"duration_seconds":0.040634,"pid":49254,"wrapper_pid":49047,"ticks":600,"reaped":true}
+{"raw_line":70,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.room-22.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-22.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-22.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:34.502664+00:00","monotonic_seconds":132015.695890958},"finished":{"utc":"2026-08-28T09:47:34.543343+00:00","monotonic_seconds":132015.736576958},"duration_seconds":0.040686,"pid":49255,"wrapper_pid":49047,"ticks":600,"reaped":true}
+{"raw_line":73,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.room-23.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-23.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-23.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:34.558752+00:00","monotonic_seconds":132015.751977125},"finished":{"utc":"2026-08-28T09:47:34.598497+00:00","monotonic_seconds":132015.791730958},"duration_seconds":0.039754,"pid":49256,"wrapper_pid":49047,"ticks":600,"reaped":true}
+{"raw_line":76,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.room-24.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-24.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-24.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:34.615943+00:00","monotonic_seconds":132015.809181041},"finished":{"utc":"2026-08-28T09:47:34.657028+00:00","monotonic_seconds":132015.85026125},"duration_seconds":0.04108,"pid":49258,"wrapper_pid":49047,"ticks":600,"reaped":true}
+{"raw_line":79,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.room-25.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-25.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-25.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:34.673127+00:00","monotonic_seconds":132015.866353625},"finished":{"utc":"2026-08-28T09:47:34.713949+00:00","monotonic_seconds":132015.907183708},"duration_seconds":0.04083,"pid":49259,"wrapper_pid":49047,"ticks":600,"reaped":true}
+{"raw_line":82,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.room-26.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-26.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-26.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:34.730495+00:00","monotonic_seconds":132015.923722625},"finished":{"utc":"2026-08-28T09:47:34.771794+00:00","monotonic_seconds":132015.965025125},"duration_seconds":0.041303,"pid":49260,"wrapper_pid":49047,"ticks":600,"reaped":true}
+{"raw_line":85,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.room-27.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-27.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-27.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:34.788348+00:00","monotonic_seconds":132015.981573458},"finished":{"utc":"2026-08-28T09:47:34.826909+00:00","monotonic_seconds":132016.02014675},"duration_seconds":0.038573,"pid":49261,"wrapper_pid":49047,"ticks":600,"reaped":true}
+{"raw_line":88,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.room-28.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-28.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-28.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:34.843997+00:00","monotonic_seconds":132016.037221041},"finished":{"utc":"2026-08-28T09:47:34.880826+00:00","monotonic_seconds":132016.074056166},"duration_seconds":0.036835,"pid":49262,"wrapper_pid":49047,"ticks":600,"reaped":true}
+{"raw_line":91,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.room-29.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-29.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-29.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:34.895892+00:00","monotonic_seconds":132016.0891145},"finished":{"utc":"2026-08-28T09:47:34.936568+00:00","monotonic_seconds":132016.129803166},"duration_seconds":0.040689,"pid":49263,"wrapper_pid":49047,"ticks":600,"reaped":true}
+{"raw_line":94,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.room-30.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-30.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-30.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:34.953411+00:00","monotonic_seconds":132016.146634958},"finished":{"utc":"2026-08-28T09:47:34.994087+00:00","monotonic_seconds":132016.187316291},"duration_seconds":0.040681,"pid":49264,"wrapper_pid":49047,"ticks":600,"reaped":true}
+{"raw_line":97,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/live.room-31.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-31.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/reference.room-31.log","exit_code":0,"started":{"utc":"2026-08-28T09:47:35.009817+00:00","monotonic_seconds":132016.203037916},"finished":{"utc":"2026-08-28T09:47:35.048491+00:00","monotonic_seconds":132016.241720375},"duration_seconds":0.038682,"pid":49265,"wrapper_pid":49047,"ticks":600,"reaped":true}
+{"raw_line":98,"category":"load-slot","argv":["python3","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/tools/g14_load_slot.py","--track","fundamentals-cpp","--slot","baseline","--revision","95d3cea363f103a2ae4f6f2b160d194304d33b19","--receipt","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/evidence/G14/fundamentals-cpp/load-slots/baseline.json","--log","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline.slot.log","--","env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/track","./track","scenario-run","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G14.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline/result.json","--ledger","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/evidence/G14-runs.jsonl"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/baseline.log","exit_code":0,"label":"baseline","units":1,"started_utc":"2026-08-28T09:47:21.046719+00:00","duration_seconds":14.814816,"wrapper_pid":48936,"child_pid":48945}
+{"raw_line":103,"category":"profiler","argv":["/usr/bin/sample","49808","5","1","-mayDie","-file","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/native-sample.txt"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/native-sample.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:12.453409+00:00","monotonic_seconds":132053.646310333},"finished":{"utc":"2026-08-28T09:48:18.460760+00:00","monotonic_seconds":132059.653622083},"duration_seconds":6.007312,"pid":49809,"wrapper_pid":49806,"ticks":0,"reaped":true}
+{"raw_line":104,"category":"load-live","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","scenario-run","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G14.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.json","--load-control"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:12.381842+00:00","monotonic_seconds":132053.574744541},"finished":{"utc":"2026-08-28T09:48:22.298113+00:00","monotonic_seconds":132063.490939333},"duration_seconds":9.916195,"pid":49808,"wrapper_pid":49806,"ticks":19200,"reaped":true}
+{"raw_line":107,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.room-01.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-01.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-01.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:22.444876+00:00","monotonic_seconds":132063.637701083},"finished":{"utc":"2026-08-28T09:48:22.485360+00:00","monotonic_seconds":132063.678190083},"duration_seconds":0.040489,"pid":49900,"wrapper_pid":49806,"ticks":600,"reaped":true}
+{"raw_line":110,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.room-02.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-02.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-02.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:22.499875+00:00","monotonic_seconds":132063.69269925},"finished":{"utc":"2026-08-28T09:48:22.536874+00:00","monotonic_seconds":132063.729704333},"duration_seconds":0.037005,"pid":49901,"wrapper_pid":49806,"ticks":600,"reaped":true}
+{"raw_line":113,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.room-03.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-03.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-03.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:22.552175+00:00","monotonic_seconds":132063.744998625},"finished":{"utc":"2026-08-28T09:48:22.591894+00:00","monotonic_seconds":132063.784721958},"duration_seconds":0.039723,"pid":49902,"wrapper_pid":49806,"ticks":600,"reaped":true}
+{"raw_line":116,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.room-04.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-04.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-04.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:22.607578+00:00","monotonic_seconds":132063.800401708},"finished":{"utc":"2026-08-28T09:48:22.646927+00:00","monotonic_seconds":132063.839756375},"duration_seconds":0.039355,"pid":49903,"wrapper_pid":49806,"ticks":600,"reaped":true}
+{"raw_line":119,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.room-05.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-05.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-05.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:22.661174+00:00","monotonic_seconds":132063.853997125},"finished":{"utc":"2026-08-28T09:48:22.701956+00:00","monotonic_seconds":132063.894785125},"duration_seconds":0.040788,"pid":49904,"wrapper_pid":49806,"ticks":600,"reaped":true}
+{"raw_line":122,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.room-06.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-06.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-06.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:22.716285+00:00","monotonic_seconds":132063.909107916},"finished":{"utc":"2026-08-28T09:48:22.756860+00:00","monotonic_seconds":132063.949686666},"duration_seconds":0.040579,"pid":49905,"wrapper_pid":49806,"ticks":600,"reaped":true}
+{"raw_line":125,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.room-07.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-07.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-07.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:22.771092+00:00","monotonic_seconds":132063.963913791},"finished":{"utc":"2026-08-28T09:48:22.811858+00:00","monotonic_seconds":132064.004690041},"duration_seconds":0.040776,"pid":49906,"wrapper_pid":49806,"ticks":600,"reaped":true}
+{"raw_line":128,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.room-08.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-08.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-08.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:22.827222+00:00","monotonic_seconds":132064.020043166},"finished":{"utc":"2026-08-28T09:48:22.865993+00:00","monotonic_seconds":132064.058825416},"duration_seconds":0.038782,"pid":49907,"wrapper_pid":49806,"ticks":600,"reaped":true}
+{"raw_line":131,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.room-09.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-09.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-09.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:22.881197+00:00","monotonic_seconds":132064.074020958},"finished":{"utc":"2026-08-28T09:48:22.921169+00:00","monotonic_seconds":132064.1139965},"duration_seconds":0.039976,"pid":49908,"wrapper_pid":49806,"ticks":600,"reaped":true}
+{"raw_line":134,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.room-10.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-10.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-10.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:22.937241+00:00","monotonic_seconds":132064.130064875},"finished":{"utc":"2026-08-28T09:48:22.977402+00:00","monotonic_seconds":132064.170234},"duration_seconds":0.040169,"pid":49910,"wrapper_pid":49806,"ticks":600,"reaped":true}
+{"raw_line":137,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.room-11.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-11.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-11.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:22.993715+00:00","monotonic_seconds":132064.186536833},"finished":{"utc":"2026-08-28T09:48:23.034841+00:00","monotonic_seconds":132064.227673416},"duration_seconds":0.041137,"pid":49911,"wrapper_pid":49806,"ticks":600,"reaped":true}
+{"raw_line":140,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.room-12.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-12.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-12.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:23.051946+00:00","monotonic_seconds":132064.244767541},"finished":{"utc":"2026-08-28T09:48:23.092932+00:00","monotonic_seconds":132064.285760541},"duration_seconds":0.040993,"pid":49912,"wrapper_pid":49806,"ticks":600,"reaped":true}
+{"raw_line":143,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.room-13.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-13.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-13.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:23.108161+00:00","monotonic_seconds":132064.300982458},"finished":{"utc":"2026-08-28T09:48:23.147481+00:00","monotonic_seconds":132064.340308791},"duration_seconds":0.039326,"pid":49913,"wrapper_pid":49806,"ticks":600,"reaped":true}
+{"raw_line":146,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.room-14.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-14.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-14.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:23.163883+00:00","monotonic_seconds":132064.356703375},"finished":{"utc":"2026-08-28T09:48:23.204710+00:00","monotonic_seconds":132064.397534208},"duration_seconds":0.040831,"pid":49914,"wrapper_pid":49806,"ticks":600,"reaped":true}
+{"raw_line":149,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.room-15.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-15.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-15.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:23.220310+00:00","monotonic_seconds":132064.4131275},"finished":{"utc":"2026-08-28T09:48:23.260579+00:00","monotonic_seconds":132064.453405541},"duration_seconds":0.040278,"pid":49915,"wrapper_pid":49806,"ticks":600,"reaped":true}
+{"raw_line":152,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.room-16.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-16.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-16.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:23.277434+00:00","monotonic_seconds":132064.470251625},"finished":{"utc":"2026-08-28T09:48:23.315851+00:00","monotonic_seconds":132064.508679375},"duration_seconds":0.038428,"pid":49916,"wrapper_pid":49806,"ticks":600,"reaped":true}
+{"raw_line":155,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.room-17.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-17.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-17.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:23.331567+00:00","monotonic_seconds":132064.524383583},"finished":{"utc":"2026-08-28T09:48:23.371393+00:00","monotonic_seconds":132064.564214125},"duration_seconds":0.039831,"pid":49917,"wrapper_pid":49806,"ticks":600,"reaped":true}
+{"raw_line":158,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.room-18.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-18.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-18.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:23.387999+00:00","monotonic_seconds":132064.580818583},"finished":{"utc":"2026-08-28T09:48:23.428008+00:00","monotonic_seconds":132064.620834166},"duration_seconds":0.040016,"pid":49918,"wrapper_pid":49806,"ticks":600,"reaped":true}
+{"raw_line":161,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.room-19.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-19.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-19.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:23.442549+00:00","monotonic_seconds":132064.635365875},"finished":{"utc":"2026-08-28T09:48:23.480153+00:00","monotonic_seconds":132064.672976708},"duration_seconds":0.037611,"pid":49919,"wrapper_pid":49806,"ticks":600,"reaped":true}
+{"raw_line":164,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.room-20.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-20.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-20.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:23.496693+00:00","monotonic_seconds":132064.68950825},"finished":{"utc":"2026-08-28T09:48:23.537702+00:00","monotonic_seconds":132064.730526625},"duration_seconds":0.041018,"pid":49920,"wrapper_pid":49806,"ticks":600,"reaped":true}
+{"raw_line":167,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.room-21.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-21.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-21.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:23.553588+00:00","monotonic_seconds":132064.746406458},"finished":{"utc":"2026-08-28T09:48:23.590847+00:00","monotonic_seconds":132064.783671166},"duration_seconds":0.037265,"pid":49921,"wrapper_pid":49806,"ticks":600,"reaped":true}
+{"raw_line":170,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.room-22.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-22.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-22.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:23.606728+00:00","monotonic_seconds":132064.799541875},"finished":{"utc":"2026-08-28T09:48:23.644080+00:00","monotonic_seconds":132064.836904333},"duration_seconds":0.037362,"pid":49922,"wrapper_pid":49806,"ticks":600,"reaped":true}
+{"raw_line":173,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.room-23.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-23.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-23.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:23.659886+00:00","monotonic_seconds":132064.852700125},"finished":{"utc":"2026-08-28T09:48:23.700970+00:00","monotonic_seconds":132064.893793666},"duration_seconds":0.041094,"pid":49923,"wrapper_pid":49806,"ticks":600,"reaped":true}
+{"raw_line":176,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.room-24.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-24.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-24.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:23.717007+00:00","monotonic_seconds":132064.909820125},"finished":{"utc":"2026-08-28T09:48:23.757776+00:00","monotonic_seconds":132064.950593875},"duration_seconds":0.040774,"pid":49924,"wrapper_pid":49806,"ticks":600,"reaped":true}
+{"raw_line":179,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.room-25.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-25.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-25.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:23.773300+00:00","monotonic_seconds":132064.966112375},"finished":{"utc":"2026-08-28T09:48:23.811048+00:00","monotonic_seconds":132065.0038665},"duration_seconds":0.037754,"pid":49925,"wrapper_pid":49806,"ticks":600,"reaped":true}
+{"raw_line":182,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.room-26.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-26.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-26.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:23.826641+00:00","monotonic_seconds":132065.01945225},"finished":{"utc":"2026-08-28T09:48:23.864173+00:00","monotonic_seconds":132065.056999},"duration_seconds":0.037547,"pid":49926,"wrapper_pid":49806,"ticks":600,"reaped":true}
+{"raw_line":185,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.room-27.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-27.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-27.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:23.912247+00:00","monotonic_seconds":132065.105066291},"finished":{"utc":"2026-08-28T09:48:23.955803+00:00","monotonic_seconds":132065.148621583},"duration_seconds":0.043555,"pid":49941,"wrapper_pid":49806,"ticks":600,"reaped":true}
+{"raw_line":188,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.room-28.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-28.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-28.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:23.973001+00:00","monotonic_seconds":132065.165813125},"finished":{"utc":"2026-08-28T09:48:24.014719+00:00","monotonic_seconds":132065.207537375},"duration_seconds":0.041724,"pid":49943,"wrapper_pid":49806,"ticks":600,"reaped":true}
+{"raw_line":191,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.room-29.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-29.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-29.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:24.030286+00:00","monotonic_seconds":132065.223096333},"finished":{"utc":"2026-08-28T09:48:24.071149+00:00","monotonic_seconds":132065.263967208},"duration_seconds":0.040871,"pid":49944,"wrapper_pid":49806,"ticks":600,"reaped":true}
+{"raw_line":194,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.room-30.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-30.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-30.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:24.087206+00:00","monotonic_seconds":132065.280016166},"finished":{"utc":"2026-08-28T09:48:24.128107+00:00","monotonic_seconds":132065.3209295},"duration_seconds":0.040913,"pid":49945,"wrapper_pid":49806,"ticks":600,"reaped":true}
+{"raw_line":197,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/live.room-31.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-31.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/reference.room-31.log","exit_code":0,"started":{"utc":"2026-08-28T09:48:24.142987+00:00","monotonic_seconds":132065.335797166},"finished":{"utc":"2026-08-28T09:48:24.181895+00:00","monotonic_seconds":132065.374711958},"duration_seconds":0.038915,"pid":49946,"wrapper_pid":49806,"ticks":600,"reaped":true}
+{"raw_line":198,"category":"load-slot","argv":["python3","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/tools/g14_load_slot.py","--track","fundamentals-cpp","--slot","profiled","--revision","95d3cea363f103a2ae4f6f2b160d194304d33b19","--receipt","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/evidence/G14/fundamentals-cpp/load-slots/profiled.json","--log","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled.slot.log","--","env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/track","./track","scenario-run","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G14.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled/result.json","--ledger","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/evidence/G14-runs.jsonl","--profile"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/profiled.log","exit_code":0,"label":"profiled","units":1,"started_utc":"2026-08-28T09:48:10.723839+00:00","duration_seconds":14.324503,"wrapper_pid":49681,"child_pid":49690}
+{"raw_line":199,"category":"compile","argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/track","ARENA_TSAN=ON","./track","build"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/tsan-build.log","exit_code":0,"label":"tsan-build","units":2,"started_utc":"2026-08-28T09:49:32.755991+00:00","duration_seconds":59.369176,"wrapper_pid":50596,"child_pid":50605}
+{"raw_line":200,"category":"unit","argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/track","TSAN_OPTIONS=halt_on_error=1","./track","unit-test"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/unit.log","exit_code":0,"label":"unit","units":1,"started_utc":"2026-08-28T09:50:45.722080+00:00","duration_seconds":2.503679,"wrapper_pid":51515,"child_pid":51516}
+{"raw_line":201,"category":"integration","argv":["env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-tsan","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/track","TSAN_OPTIONS=halt_on_error=1","./track","integration-test"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/integration.log","exit_code":0,"label":"integration","units":1,"started_utc":"2026-08-28T09:50:48.266963+00:00","duration_seconds":1.445395,"wrapper_pid":51540,"child_pid":51542}
+{"raw_line":204,"category":"load-live","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","scenario-run","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G14.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.json","--load-control"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.log","exit_code":0,"started":{"utc":"2026-08-28T09:50:51.369945+00:00","monotonic_seconds":132212.561519083},"finished":{"utc":"2026-08-28T09:51:01.293553+00:00","monotonic_seconds":132222.485049625},"duration_seconds":9.923531,"pid":51701,"wrapper_pid":51699,"ticks":19200,"reaped":true}
+{"raw_line":207,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.room-01.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-01.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-01.log","exit_code":0,"started":{"utc":"2026-08-28T09:51:01.352804+00:00","monotonic_seconds":132222.544299625},"finished":{"utc":"2026-08-28T09:51:01.393249+00:00","monotonic_seconds":132222.58474875},"duration_seconds":0.040449,"pid":52007,"wrapper_pid":51699,"ticks":600,"reaped":true}
+{"raw_line":210,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.room-02.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-02.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-02.log","exit_code":0,"started":{"utc":"2026-08-28T09:51:01.407780+00:00","monotonic_seconds":132222.59927425},"finished":{"utc":"2026-08-28T09:51:01.447916+00:00","monotonic_seconds":132222.639418583},"duration_seconds":0.040144,"pid":52008,"wrapper_pid":51699,"ticks":600,"reaped":true}
+{"raw_line":213,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.room-03.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-03.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-03.log","exit_code":0,"started":{"utc":"2026-08-28T09:51:01.463781+00:00","monotonic_seconds":132222.655277875},"finished":{"utc":"2026-08-28T09:51:01.502371+00:00","monotonic_seconds":132222.693876666},"duration_seconds":0.038599,"pid":52009,"wrapper_pid":51699,"ticks":600,"reaped":true}
+{"raw_line":216,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.room-04.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-04.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-04.log","exit_code":0,"started":{"utc":"2026-08-28T09:51:01.518061+00:00","monotonic_seconds":132222.709562583},"finished":{"utc":"2026-08-28T09:51:01.559076+00:00","monotonic_seconds":132222.750576666},"duration_seconds":0.041014,"pid":52010,"wrapper_pid":51699,"ticks":600,"reaped":true}
+{"raw_line":219,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.room-05.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-05.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-05.log","exit_code":0,"started":{"utc":"2026-08-28T09:51:01.574363+00:00","monotonic_seconds":132222.765859208},"finished":{"utc":"2026-08-28T09:51:01.614658+00:00","monotonic_seconds":132222.806159},"duration_seconds":0.0403,"pid":52011,"wrapper_pid":51699,"ticks":600,"reaped":true}
+{"raw_line":222,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.room-06.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-06.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-06.log","exit_code":0,"started":{"utc":"2026-08-28T09:51:01.631113+00:00","monotonic_seconds":132222.822611291},"finished":{"utc":"2026-08-28T09:51:01.671257+00:00","monotonic_seconds":132222.862757875},"duration_seconds":0.040147,"pid":52012,"wrapper_pid":51699,"ticks":600,"reaped":true}
+{"raw_line":225,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.room-07.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-07.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-07.log","exit_code":0,"started":{"utc":"2026-08-28T09:51:01.686856+00:00","monotonic_seconds":132222.878349083},"finished":{"utc":"2026-08-28T09:51:01.725718+00:00","monotonic_seconds":132222.917219375},"duration_seconds":0.03887,"pid":52013,"wrapper_pid":51699,"ticks":600,"reaped":true}
+{"raw_line":228,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.room-08.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-08.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-08.log","exit_code":0,"started":{"utc":"2026-08-28T09:51:01.742093+00:00","monotonic_seconds":132222.933585416},"finished":{"utc":"2026-08-28T09:51:01.782415+00:00","monotonic_seconds":132222.9739195},"duration_seconds":0.040334,"pid":52014,"wrapper_pid":51699,"ticks":600,"reaped":true}
+{"raw_line":231,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.room-09.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-09.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-09.log","exit_code":0,"started":{"utc":"2026-08-28T09:51:01.797983+00:00","monotonic_seconds":132222.989486083},"finished":{"utc":"2026-08-28T09:51:01.839085+00:00","monotonic_seconds":132223.030583833},"duration_seconds":0.041098,"pid":52015,"wrapper_pid":51699,"ticks":600,"reaped":true}
+{"raw_line":234,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.room-10.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-10.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-10.log","exit_code":0,"started":{"utc":"2026-08-28T09:51:01.854851+00:00","monotonic_seconds":132223.04634325},"finished":{"utc":"2026-08-28T09:51:01.894880+00:00","monotonic_seconds":132223.086372625},"duration_seconds":0.040029,"pid":52016,"wrapper_pid":51699,"ticks":600,"reaped":true}
+{"raw_line":237,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.room-11.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-11.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-11.log","exit_code":0,"started":{"utc":"2026-08-28T09:51:01.909743+00:00","monotonic_seconds":132223.10123575},"finished":{"utc":"2026-08-28T09:51:01.947408+00:00","monotonic_seconds":132223.138909833},"duration_seconds":0.037674,"pid":52018,"wrapper_pid":51699,"ticks":600,"reaped":true}
+{"raw_line":240,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.room-12.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-12.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-12.log","exit_code":0,"started":{"utc":"2026-08-28T09:51:01.963387+00:00","monotonic_seconds":132223.154881666},"finished":{"utc":"2026-08-28T09:51:02.002374+00:00","monotonic_seconds":132223.193875083},"duration_seconds":0.038993,"pid":52019,"wrapper_pid":51699,"ticks":600,"reaped":true}
+{"raw_line":243,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.room-13.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-13.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-13.log","exit_code":0,"started":{"utc":"2026-08-28T09:51:02.017206+00:00","monotonic_seconds":132223.208697625},"finished":{"utc":"2026-08-28T09:51:02.058219+00:00","monotonic_seconds":132223.249717125},"duration_seconds":0.04102,"pid":52020,"wrapper_pid":51699,"ticks":600,"reaped":true}
+{"raw_line":246,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.room-14.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-14.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-14.log","exit_code":0,"started":{"utc":"2026-08-28T09:51:02.073770+00:00","monotonic_seconds":132223.26526125},"finished":{"utc":"2026-08-28T09:51:02.114349+00:00","monotonic_seconds":132223.305868208},"duration_seconds":0.040607,"pid":52021,"wrapper_pid":51699,"ticks":600,"reaped":true}
+{"raw_line":249,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.room-15.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-15.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-15.log","exit_code":0,"started":{"utc":"2026-08-28T09:51:02.130059+00:00","monotonic_seconds":132223.321552041},"finished":{"utc":"2026-08-28T09:51:02.169039+00:00","monotonic_seconds":132223.360539375},"duration_seconds":0.038987,"pid":52022,"wrapper_pid":51699,"ticks":600,"reaped":true}
+{"raw_line":252,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.room-16.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-16.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-16.log","exit_code":0,"started":{"utc":"2026-08-28T09:51:02.184824+00:00","monotonic_seconds":132223.376316041},"finished":{"utc":"2026-08-28T09:51:02.225736+00:00","monotonic_seconds":132223.41723375},"duration_seconds":0.040918,"pid":52023,"wrapper_pid":51699,"ticks":600,"reaped":true}
+{"raw_line":255,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.room-17.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-17.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-17.log","exit_code":0,"started":{"utc":"2026-08-28T09:51:02.241158+00:00","monotonic_seconds":132223.432647208},"finished":{"utc":"2026-08-28T09:51:02.281811+00:00","monotonic_seconds":132223.473330125},"duration_seconds":0.040683,"pid":52024,"wrapper_pid":51699,"ticks":600,"reaped":true}
+{"raw_line":258,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.room-18.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-18.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-18.log","exit_code":0,"started":{"utc":"2026-08-28T09:51:02.298283+00:00","monotonic_seconds":132223.489785208},"finished":{"utc":"2026-08-28T09:51:02.334574+00:00","monotonic_seconds":132223.526070458},"duration_seconds":0.036285,"pid":52025,"wrapper_pid":51699,"ticks":600,"reaped":true}
+{"raw_line":261,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.room-19.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-19.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-19.log","exit_code":0,"started":{"utc":"2026-08-28T09:51:02.350322+00:00","monotonic_seconds":132223.541810333},"finished":{"utc":"2026-08-28T09:51:02.391262+00:00","monotonic_seconds":132223.582757833},"duration_seconds":0.040948,"pid":52026,"wrapper_pid":51699,"ticks":600,"reaped":true}
+{"raw_line":264,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.room-20.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-20.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-20.log","exit_code":0,"started":{"utc":"2026-08-28T09:51:02.409619+00:00","monotonic_seconds":132223.601107166},"finished":{"utc":"2026-08-28T09:51:02.450279+00:00","monotonic_seconds":132223.6417985},"duration_seconds":0.040691,"pid":52027,"wrapper_pid":51699,"ticks":600,"reaped":true}
+{"raw_line":267,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.room-21.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-21.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-21.log","exit_code":0,"started":{"utc":"2026-08-28T09:51:02.466259+00:00","monotonic_seconds":132223.657751458},"finished":{"utc":"2026-08-28T09:51:02.507328+00:00","monotonic_seconds":132223.698824041},"duration_seconds":0.041073,"pid":52028,"wrapper_pid":51699,"ticks":600,"reaped":true}
+{"raw_line":270,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.room-22.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-22.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-22.log","exit_code":0,"started":{"utc":"2026-08-28T09:51:02.523678+00:00","monotonic_seconds":132223.71516625},"finished":{"utc":"2026-08-28T09:51:02.564299+00:00","monotonic_seconds":132223.755796875},"duration_seconds":0.040631,"pid":52029,"wrapper_pid":51699,"ticks":600,"reaped":true}
+{"raw_line":273,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.room-23.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-23.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-23.log","exit_code":0,"started":{"utc":"2026-08-28T09:51:02.581391+00:00","monotonic_seconds":132223.772892458},"finished":{"utc":"2026-08-28T09:51:02.619001+00:00","monotonic_seconds":132223.810497125},"duration_seconds":0.037605,"pid":52044,"wrapper_pid":51699,"ticks":600,"reaped":true}
+{"raw_line":276,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.room-24.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-24.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-24.log","exit_code":0,"started":{"utc":"2026-08-28T09:51:02.634494+00:00","monotonic_seconds":132223.825980416},"finished":{"utc":"2026-08-28T09:51:02.675823+00:00","monotonic_seconds":132223.867317041},"duration_seconds":0.041337,"pid":52045,"wrapper_pid":51699,"ticks":600,"reaped":true}
+{"raw_line":279,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.room-25.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-25.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-25.log","exit_code":0,"started":{"utc":"2026-08-28T09:51:02.691432+00:00","monotonic_seconds":132223.882916958},"finished":{"utc":"2026-08-28T09:51:02.731307+00:00","monotonic_seconds":132223.922816083},"duration_seconds":0.039899,"pid":52046,"wrapper_pid":51699,"ticks":600,"reaped":true}
+{"raw_line":282,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.room-26.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-26.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-26.log","exit_code":0,"started":{"utc":"2026-08-28T09:51:02.747646+00:00","monotonic_seconds":132223.939131333},"finished":{"utc":"2026-08-28T09:51:02.784684+00:00","monotonic_seconds":132223.976212458},"duration_seconds":0.037081,"pid":52047,"wrapper_pid":51699,"ticks":600,"reaped":true}
+{"raw_line":285,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.room-27.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-27.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-27.log","exit_code":0,"started":{"utc":"2026-08-28T09:51:02.800939+00:00","monotonic_seconds":132223.992422875},"finished":{"utc":"2026-08-28T09:51:02.841093+00:00","monotonic_seconds":132224.032587291},"duration_seconds":0.040164,"pid":52048,"wrapper_pid":51699,"ticks":600,"reaped":true}
+{"raw_line":288,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.room-28.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-28.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-28.log","exit_code":0,"started":{"utc":"2026-08-28T09:51:02.857676+00:00","monotonic_seconds":132224.04915975},"finished":{"utc":"2026-08-28T09:51:02.896770+00:00","monotonic_seconds":132224.08826575},"duration_seconds":0.039106,"pid":52049,"wrapper_pid":51699,"ticks":600,"reaped":true}
+{"raw_line":291,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.room-29.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-29.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-29.log","exit_code":0,"started":{"utc":"2026-08-28T09:51:02.912535+00:00","monotonic_seconds":132224.104019458},"finished":{"utc":"2026-08-28T09:51:02.951288+00:00","monotonic_seconds":132224.142780583},"duration_seconds":0.038761,"pid":52051,"wrapper_pid":51699,"ticks":600,"reaped":true}
+{"raw_line":294,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.room-30.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-30.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-30.log","exit_code":0,"started":{"utc":"2026-08-28T09:51:02.967887+00:00","monotonic_seconds":132224.159374833},"finished":{"utc":"2026-08-28T09:51:03.005303+00:00","monotonic_seconds":132224.196795125},"duration_seconds":0.03742,"pid":52052,"wrapper_pid":51699,"ticks":600,"reaped":true}
+{"raw_line":297,"category":"reference","argv":["/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release/arena_scenarios","replay-verify","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/live.room-31.replay.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-31.json"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/reference.room-31.log","exit_code":0,"started":{"utc":"2026-08-28T09:51:03.020867+00:00","monotonic_seconds":132224.212348458},"finished":{"utc":"2026-08-28T09:51:03.061126+00:00","monotonic_seconds":132224.25261525},"duration_seconds":0.040267,"pid":52053,"wrapper_pid":51699,"ticks":600,"reaped":true}
+{"raw_line":298,"category":"load-slot","argv":["python3","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/tools/g14_load_slot.py","--track","fundamentals-cpp","--slot","worker-final","--revision","95d3cea363f103a2ae4f6f2b160d194304d33b19","--receipt","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/evidence/G14/fundamentals-cpp/load-slots/worker-final.json","--log","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final.slot.log","--","env","ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g14-release","ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/track","./track","scenario-run","/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G14.json","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final/result.json","--ledger","/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/evidence/G14-runs.jsonl"],"cwd":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp","log":"/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/artifacts/g14/worker-final.log","exit_code":0,"label":"worker-final","units":1,"started_utc":"2026-08-28T09:50:49.746641+00:00","duration_seconds":14.250924,"wrapper_pid":51569,"child_pid":51570}
diff --git a/evidence/G14.md b/evidence/G14.md
new file mode 100644
index 0000000..7d302ca
--- /dev/null
+++ b/evidence/G14.md
@@ -0,0 +1,59 @@
+# G14 — realtime-core Release evidence
+
+- START: `95d3cea363f103a2ae4f6f2b160d194304d33b19`; branch `track/fundamentals-cpp`; initial attempt, repairs0/2.
+- SPEC_REVISION: `c1d62196ab76b55652f5d75a67514f8c6d8210ce`; phase1/profile `realtime-core`.
+- Fixture: main `index/scenarios/G14.json`, SHA256 `9e3045a578683cab48abe838a71336b457863811098228462153df7c512b5e6d`.
+- Reproduction: **NOT_REPRODUCED** for every fixed structural guarantee. All13 production/lock files remained byte-identical to START before, during and after all three measured runs. **Zero production changes.**
+
+## Execution and integrity
+
+The fixed Release harness was frozen before baseline; it uses ordinary four-player joins/binds in32 real Rooms, one shared unsigned32 LCG, all128 actual INPUT_ACKs before each50ms step, and the actual G12 snapshot dequeue/storage. The test-only hold distinguishes queued ownership from the one temporary direct UDP reply buffer; it never blocks INPUT_ACK, pins ACK1, changes capacity, or replaces the queue. G13 fixture access is reused. G01–G13 scenario dispatch and full suites retain their previous execution paths/assertions.
+
+`evidence/G14-commands.jsonl` contains104 actual completed command records with argv, time, exit, PID, log and original raw line. The original298-event ledger remains at `evidence/G14-runs.jsonl` (ignored, original bytes/path), SHA256 `cac0c7a35df9fd9c1922ab2778b23dc68f651cc256787ca2df54231f9d10df1a`. Every child/replay/profile pass and failure outcome is retained; there were no failed application/build commands.
+
+| Budget | Actual |
+|---|---|
+| Configure/compile |4/8: Release configure+build and separate TSan configure+build |
+| Unit |2/4 including unchanged-production load reproduction; full TSan suite27 passed |
+| Integration |1/2; full TSan suite4 passed, including the existing11-cell UDP matrix |
+| Worker live slots |3/3: baseline, profiled, worker-final; root main-final remains reserved |
+| Actual live/reference ticks |57600 /55800 total; each slot19200 live +31 separate600-tick replays |
+| Profiler/warm/extra fault |1/1 native sample; warm0; extra fault0 |
+
+Every load used the mandated root `g14_load_slot.py` launcher. Each receipt is `main/index/evidence/G14/fundamentals-cpp/load-slots/<slot>.json`, state COMPLETED/exit0 with its process group empty. Unit/integration contain no G14 campaign. Standard `./track scenario-run G14.json OUT` owns the live child and all31 replay processes; the default never profiles. The explicit profiled command alone adds `--profile`. Socket ceilings are5s and the mandatory launcher's whole-load ceiling is600s; neither drives simulation.
+
+## Actual measured interval
+
+| Run | Wall s | User CPU s | System CPU s | RSS at600, bytes | Allocator in-use at600, bytes |
+|---|---:|---:|---:|---:|---:|
+| baseline | 9.665924 | 6.860734 | 2.348234 | 329105408 | 320027072 |
+| profiled | 9.376139 | 6.608888 | 2.305130 | 329121792 | 320027072 |
+| worker-final | 9.250105 | 6.811208 | 2.246641 | 329302016 | 320027088 |
+
+These are whole-process measurements: production owner/transport plus128 real test clients, their existing bounded message histories (902 observations for each healthy client,602 for the slow client), replicas, assertions, streaming observers and memory sampling. Initialization, export, shutdown and offline verification are outside the interval. No throughput ranking or improvement is inferred from run-to-run timing noise.
+
+RSS uses Mach task info; CPU uses getrusage in seconds; Darwin peak RSS is bytes; allocator counters use malloc_zone_statistics(NULL) across all zones (blocks=count, sizes=bytes). Its max_size_in_use returned0 and is not treated as a usable peak counter. Per-snapshot heap bytes are unavailable; actual retained counts and32-entry bounds are recorded. Every run samples at initialization,200/400/600 ticks and after shutdown.
+
+The single sample attached to live PID49808 (coordinator49806; profiler49809), requested5s/1ms, exit0/reaped. Native start was18:48:12.522+0900; first observed reap09:48:18.460Z preceded DONE09:48:21.833Z and all export/replay work. The preserved native report accounts for4242 samples. Top collapsed observations include write535, sendto476, malloc-tiny300 and free282. These include harness work and do not isolate a necessary safe production change, so no optimization was applied. C++ steady-clock and Python monotonic epochs differ; paired READY/DONE and native wall timestamps establish order instead of subtracting unrelated epochs.
+
+## Structural and resource results
+
+All three runs accepted76800 real UDP inputs (600/player), captured19200 actual Room ticks, and stayed RUNNING after599 with the1200-tick session rule unchanged. All127 healthy clients received/applied/ACKed301 snapshots each (38227 total). Slow player remained CONNECTED with600 INPUT_ACKs, only initial FULL1 received,300 actual held generations,299 superseded buffers destroyed and final FULL301/tick599 retained. Ordinary retention expiry/full fallback remained active. All31 immutable actual accepted journals replayed600 ticks each; every raw canonical record and hash matched.
+
+Observed high-water values were Rooms32/64, connections128/512, mailbox4/512, total pending128/32768, pending/player1/64, controls1/64, outbound buffers3/2121bytes, snapshot retention32/32 and UDP ingress241/output721bytes (limit1200). Each Room's catch-up high-water was1/4, with no missed deadline. Actual cleanup closed387 network/reactor descriptors plus97 raw-file descriptors and released115744/115744 owned buffers. All active queue/session/parser/Room/registry/timer/journal counters ended at0.
+
+Worker-final allocator in-use fell from320027088bytes at600 to8447088bytes after shutdown while RSS stayed335151104bytes and reserved allocator space352321536bytes. The remaining process memory includes bounded report objects and allocator-retained pages; zero live ownership counters, not RSS alone, establish cleanup.
+
+## Raw paths and hashes
+
+All paths below are relative to this worktree unless prefixed main. Large raw/profile artifacts are deliberately uncommitted.
+
+- `artifacts/g14/commands.json`: resolved mandatory argv/receipt/output paths, profiler attachment and budgets.
+- `artifacts/g14/production-START.json`, `harness-frozen.json`, `release-preflight.json`: unchanged START hashes, fixed harness hashes, actual Release flags/tool/link/binary observations.
+- `artifacts/g14/{baseline,profiled,worker-final}/result.json`: all command timelines, child PIDs,31 reference results and97 raw-file SHA256 values.
+- Each run's `live.json` contains full actual per-Room600 hash arrays/final states, setup, owner mapping and bounds; `live.memory.json` contains all five samples. Sibling `live.room-NN.{records,inputs,snapshots}.jsonl` and `live.slow.jsonl` retain all observed states/records/wires/admissions/dequeue transitions. `live.room-NN.replay.json` and `reference.room-NN.{json,records.json}` retain actual journals and all31 offline outcomes.
+- `artifacts/g14/profiled/native-sample.txt`: SHA256 `070dac45ca1267d64d9fbb2dc5aa95ec15aae27a15c1087d6332133e87d92a42`; original capture/log remain unchanged.
+- `artifacts/g14/checks.json`: read-only checksum/count/hash-equality/bounds audit; all32×600 live hash maps equal across the three worker runs, map SHA256 `07a832f62ebce8b1cc88e6319d464e5ef1ae82b4b276a21819a942570311bdc4`.
+- Worker-final result SHA256 `ebf2c8494373b5758ea2027a2ed4a2fa888265fb0341b85e7ce72c79a6232aa3`; live SHA256 `8c01b9f470325183df41f042f91954c50edf8cf621375832f4974f6d09ada116`.
+
+No unresolved worker failure. No additional load, profile, business feature, phase2 work, tag or push. Immutable END awaits root's one reserved independent final slot and release gate.
diff --git a/tests/g12.hpp b/tests/g12.hpp
index 7c6c260..547618b 100644
--- a/tests/g12.hpp
+++ b/tests/g12.hpp
@@ -4,6 +4,9 @@
 #error G12 readiness access is test-build only
 #endif
 namespace arena {
+struct SnapshotHold {
+  std::uint64_t not_ready = 0, direct_udp_ready = 0, control_ready = 0, last_generation = 0;
+};
 struct OutboundFixture {
   static std::uint64_t connection_id(Server& server, const std::string& session);
   static void gate(Server& server, std::function<bool(std::uint64_t,bool)> ready);
@@ -11,6 +14,7 @@ struct OutboundFixture {
   static void control(Server& server, std::uint64_t id, const Json& value);
   static void capture(Server& server, std::uint64_t id, bool full);
   static void flush(Server& server, std::uint64_t id);
+  static void hold_snapshots(Server&, std::uint64_t id, SnapshotHold&, std::function<void(const Json&)> observer);
 };
 Json run_control_queue_probe();
 Json run_snapshot_queue_probe();
diff --git a/tests/g12_queue.cpp b/tests/g12_queue.cpp
index a2ecce1..07b5e34 100644
--- a/tests/g12_queue.cpp
+++ b/tests/g12_queue.cpp
@@ -28,6 +28,35 @@ std::uint64_t OutboundFixture::connection_id(Server& server, const std::string&
   throw std::logic_error("test session is not connected");
 }
 void OutboundFixture::gate(Server& server, std::function<bool(std::uint64_t,bool)> ready) { server.fixture_outbound_ready_ = std::move(ready); }
+void OutboundFixture::hold_snapshots(Server& server, std::uint64_t id, SnapshotHold& hold, std::function<void(const Json&)> observer) {
+  gate(server,[&server,id,&hold,observer=std::move(observer)](std::uint64_t connection, bool datagram) {
+    if (connection != id) return true;
+    if (!datagram) { ++hold.control_ready; return true; }
+    std::size_t queued = 0;
+    for (const auto& [fd,conn] : server.connections_) {
+      (void)fd; queued += conn.outbound.size()+conn.pending_full.has_value()+conn.pending_delta.has_value();
+    }
+    // At the existing snapshot dequeue every PendingWrite belongs to a real
+    // queue slot. send_realtime's other UDP replies instead own one additional
+    // local buffer. Its move constructor transfers, never duplicates, ownership.
+    queue_need(server.outbound_memory_.buffers >= queued && server.outbound_memory_.buffers-queued <= 1,
+      "snapshot-only readiness distinguishes actual queued and direct buffer ownership");
+    if (server.outbound_memory_.buffers != queued) { ++hold.direct_udp_ready; return true; }
+    auto* conn = server.connection(id); queue_need(conn && (conn->pending_full || conn->pending_delta),"actual snapshot pre-dequeue boundary");
+    ++hold.not_ready;
+    if (conn->snapshots_generated != hold.last_generation) {
+      hold.last_generation = conn->snapshots_generated;
+      auto state = server.outbound_state(*conn); state["stream"] = conn->snapshots.metrics();
+      state["pending_full"] = conn->pending_full ? decode_datagram(conn->pending_full->bytes) : Json(nullptr);
+      state["pending_delta"] = conn->pending_delta ? decode_datagram(conn->pending_delta->bytes) : Json(nullptr);
+      state["actual_buffer_ids"] = Json::array();
+      if (conn->pending_full) state["actual_buffer_ids"].push_back(conn->pending_full->buffer_id);
+      if (conn->pending_delta) state["actual_buffer_ids"].push_back(conn->pending_delta->buffer_id);
+      state["queued_buffers"] = queued; state["live_buffers"] = server.outbound_memory_.buffers; observer(state);
+    }
+    return false;
+  });
+}
 Json OutboundFixture::inspect(Server& server, std::uint64_t id) {
   auto* conn = server.connection(id);
   if (!conn) return Json{{"connection_present",false},{"control",0},{"full",0},{"delta",0},{"retained_bytes",0},{"memory",memory(server)}};
diff --git a/tests/g13.cpp b/tests/g13.cpp
index e9ea221..09f304f 100644
--- a/tests/g13.cpp
+++ b/tests/g13.cpp
@@ -111,6 +111,18 @@ struct IsolationFixture {
   }
 };
 
+namespace multi_room_fixture {
+void configure(Server& server, const Json& scenario, std::function<void(const Room&,const ReplayLog&)> observer) {
+  IsolationFixture::configure(server,scenario,{},std::move(observer));
+}
+std::uint64_t connection_id(const Server& server, const std::string& session) { return IsolationFixture::connection_id(server,session); }
+std::uint64_t received(const Server& server) { return IsolationFixture::received(server); }
+bool drained(const Server& server) { return IsolationFixture::drained(server); }
+std::uint64_t acknowledged(Server& server, std::uint64_t id) { return IsolationFixture::acknowledged(server,id); }
+Json schedule(const Server& server) { return IsolationFixture::schedule(server); }
+Json mapping(const Server& server) { return IsolationFixture::mapping(server); }
+}
+
 Json run_isolation_scenario(const Json& scenario, const std::filesystem::path& output) {
   isolation_need(scenario.at("thread") == "G13" && scenario.at("contract_version") == 1 && scenario.at("seed") == 7050 &&
     scenario.at("rooms").size() == 32 && scenario.at("clock").at("kind") == "shared-manual-monotonic" &&
diff --git a/tests/g13.hpp b/tests/g13.hpp
index 35aa730..e699783 100644
--- a/tests/g13.hpp
+++ b/tests/g13.hpp
@@ -2,4 +2,15 @@
 #include "g07.hpp"
 namespace arena {
 Json run_isolation_scenario(const Json& scenario, const std::filesystem::path& output);
+// Shared access to the existing test-build-only32-Room owner fixture. These
+// functions observe real connections/queues; they do not create Room models.
+namespace multi_room_fixture {
+void configure(Server&, const Json&, std::function<void(const Room&,const ReplayLog&)>);
+std::uint64_t connection_id(const Server&, const std::string&);
+std::uint64_t received(const Server&);
+bool drained(const Server&);
+std::uint64_t acknowledged(Server&, std::uint64_t);
+Json schedule(const Server&);
+Json mapping(const Server&);
+}
 }
diff --git a/tests/g14.cpp b/tests/g14.cpp
new file mode 100644
index 0000000..df433af
--- /dev/null
+++ b/tests/g14.cpp
@@ -0,0 +1,392 @@
+#include "g14.hpp"
+#include "g12.hpp"
+#ifndef ARENA_TEST_FIXTURES
+#error G14 measurement and readiness harness is test-build only
+#endif
+#include <array>
+#include <cerrno>
+#include <chrono>
+#include <fcntl.h>
+#include <fstream>
+#include <iostream>
+#include <libproc.h>
+#include <mach/mach.h>
+#include <malloc/malloc.h>
+#include <memory>
+#include <sys/resource.h>
+#include <unistd.h>
+
+namespace arena {
+namespace {
+void load_need(bool value, const std::string& message) {
+  if (!value) throw std::runtime_error("G14: "+message);
+}
+// Stream observations instead of accumulating a second server history. Every
+// file retains the existing4MiB evidence ceiling and owns its actual FD.
+struct Lines {
+  std::filesystem::path path;
+  Fd fd;
+  std::size_t bytes = 0, count = 0;
+  explicit Lines(std::filesystem::path value) : path(std::move(value)),
+      fd(::open(path.c_str(),O_WRONLY|O_CREAT|O_EXCL|O_CLOEXEC,0600)) {
+    load_need(fd.get() >= 0,"exclusive bounded raw output");
+  }
+  void append(const Json& value) {
+    const auto line = value.dump()+"\n";
+    load_need(bytes+line.size() <= max_replay_bytes,"raw output bound");
+    std::size_t offset = 0;
+    while (offset < line.size()) {
+      const auto written = ::write(fd.get(),line.data()+offset,line.size()-offset);
+      if (written < 0 && errno == EINTR) continue;
+      load_need(written > 0,"raw observation write"); offset += static_cast<std::size_t>(written);
+    }
+    bytes += line.size(); ++count;
+  }
+  Json summary() const { return Json{{"path",std::filesystem::absolute(path).string()},{"bytes",bytes},{"records",count}}; }
+  void close() { fd.reset(); }
+};
+Json state_of(const Room& room) {
+  auto state = room.view(); state["owner_epoch"] = 0;
+  for (auto& row : state["players"]) {
+    const auto& player = room.players().at(row.at("player_id").get<std::string>());
+    row["last_seq"] = player.last_accepted_seq(); row["pending"] = player.pending.size();
+    row["applied_seq"] = player.applied_seq ? Json(*player.applied_seq) : Json(nullptr);
+    row["disconnect_deadline"] = player.disconnect_deadline ? Json(*player.disconnect_deadline) : Json(nullptr);
+  }
+  return state;
+}
+Json visible(const Json& state) {
+  Json players = Json::array();
+  for (const auto& player : state.at("players")) if (player.at("connectivity") != "LEFT") {
+    Json row;
+    for (const auto* field : {"player_id","slot","x","y","direction","score","connectivity"}) row[field] = player.at(field);
+    players.push_back(std::move(row));
+  }
+  return Json{{"room_id",state.at("room_id")},{"tick",state.at("tick")},{"owner_epoch",0},{"status",state.at("status")},{"players",players}};
+}
+Json redact(Json value, const std::string& alias) { value.erase("session_id"); value["session_alias"] = alias; return value; }
+double seconds(const timeval& value) { return static_cast<double>(value.tv_sec)+static_cast<double>(value.tv_usec)/1'000'000.0; }
+Json process_sample() {
+  rusage usage{}; load_need(::getrusage(RUSAGE_SELF,&usage) == 0,"process CPU usage available");
+  mach_task_basic_info_data_t basic{}; mach_msg_type_number_t size = MACH_TASK_BASIC_INFO_COUNT;
+  const auto vm = ::task_info(mach_task_self(),MACH_TASK_BASIC_INFO,reinterpret_cast<task_info_t>(&basic),&size);
+  malloc_statistics_t allocator{}; ::malloc_zone_statistics(nullptr,&allocator);
+  proc_taskinfo task{}; const auto task_bytes = ::proc_pidinfo(::getpid(),PROC_PIDTASKINFO,0,&task,sizeof(task));
+  return Json{{"cpu_user_seconds",seconds(usage.ru_utime)},{"cpu_system_seconds",seconds(usage.ru_stime)},
+    {"rss_bytes",vm == KERN_SUCCESS ? Json(basic.resident_size) : Json(nullptr)},
+    {"rss_source","task_info(MACH_TASK_BASIC_INFO), bytes"},{"rss_observation_code",vm},
+    {"peak_rss_bytes",usage.ru_maxrss},{"peak_rss_source","getrusage(RUSAGE_SELF).ru_maxrss, Darwin bytes"},
+    {"malloc_blocks_in_use",allocator.blocks_in_use},{"malloc_size_in_use_bytes",allocator.size_in_use},
+    {"malloc_reserved_bytes",allocator.size_allocated},{"malloc_max_touched_bytes",allocator.max_size_in_use},
+    {"allocator_source","malloc_zone_statistics(NULL), all process zones; blocks=count, sizes=bytes"},
+    {"threads",task_bytes == static_cast<int>(sizeof(task)) ? Json(task.pti_threadnum) : Json(nullptr)},
+    {"threads_source","proc_pidinfo(PROC_PIDTASKINFO).pti_threadnum"},{"tracked_owned_fds",Fd::live()},
+    {"gc",nullptr},{"gc_availability","not applicable: C++ has no tracing GC"}};
+}
+struct LoadPeer {
+  std::unique_ptr<TcpClient> tcp;
+  std::unique_ptr<UdpClient> udp;
+  std::string session, player;
+  std::uint64_t connection = 0, latest = 0, accepted = 0, snapshot_acks = 0;
+  std::map<std::uint64_t,Json> retained;
+};
+struct LoadRoom {
+  std::string id;
+  std::array<LoadPeer,4> peers;
+  Json initial, current, hashes = Json::array();
+  std::string initial_hash, current_hash;
+  std::unique_ptr<Lines> records, admissions, snapshots;
+};
+void control_boundary(const char* event, const char* expected, bool controlled, const Json& fields = Json::object()) {
+  if (!controlled) return;
+  auto value = fields; value.update(Json{{"event",event},{"process_id",::getpid()},
+    {"steady_seconds",std::chrono::duration<double>(std::chrono::steady_clock::now().time_since_epoch()).count()}});
+  std::cout << value.dump() << std::endl;
+  std::string line; load_need(static_cast<bool>(std::getline(std::cin,line)) && line == expected,"single bounded runner phase handoff");
+}
+}
+
+Json run_release_load(const Json& scenario, const std::filesystem::path& output, bool controlled) {
+  load_need(scenario.at("thread") == "G14" && scenario.at("seed") == 7050 && scenario.at("rooms").size() == 32 &&
+    scenario.at("clock") == Json{{"kind","shared-manual-monotonic"},{"tick_duration_ms",50},{"ticks",600},{"virtual_duration_ms",30000}} &&
+    scenario.at("expected_input_attempts") == 76800 && scenario.at("expected_accepted_inputs_per_player") == 600 &&
+    scenario.at("socket_ceiling_ms") == 5000 && session_ticks == 1200,"fixed G14 workload without shortened session");
+  const auto& generator = scenario.at("input_generator");
+  load_need(generator.at("algorithm") == "LCG32" && generator.at("initial_state") == 7050 && generator.at("multiplier") == 1664525 &&
+    generator.at("increment") == 1013904223 && generator.at("modulus") == 4294967296ULL &&
+    generator.at("directions") == Json::array({"STOP","NORTH","EAST","SOUTH","WEST"}),"one shared unsigned32 generator");
+  load_need(scenario.at("resource_limits") == Json{{"rooms",max_rooms},{"connections",max_connections},{"udp_payload_bytes",max_datagram_bytes},
+    {"tcp_payload_bytes",max_frame_bytes},{"accepted_pending_per_player",max_pending_inputs},{"total_accepted_pending",max_total_pending_inputs},
+    {"pending_full_per_client",1},{"pending_delta_per_client",1},{"controls_per_client",max_control_messages},
+    {"snapshot_retention_per_client",snapshot_retention},{"catchup_per_room_per_iteration",max_catch_up_ticks}},"unchanged complete resource limits");
+  if (output.has_parent_path()) std::filesystem::create_directories(output.parent_path());
+  const auto before_process = process_sample(); const int fd_before = Fd::live();
+  ManualClock executed_clock; std::int64_t shared_now = 0; SnapshotHold hold;
+  std::array<LoadRoom,32> rooms; std::map<std::string,std::size_t> indices;
+  Server server(executed_clock,0,[&] { return shared_now; });
+  const auto prefix = output.stem().string();
+  auto slow = std::make_unique<Lines>(output.parent_path()/(prefix+".slow.jsonl"));
+  Json samples = Json::array(), setup = Json::array(), result, mapping;
+  std::vector<int> descriptors; std::size_t accepted_total = 0, executed_total = 0;
+  std::string phase = "initialization"; Json interval;
+  const auto cleanup = [&] {
+    // Keep the hold active while shutdown cancels snapshots; releasing it
+    // first would transmit a late301st-generation snapshot.
+    server.shutdown(); OutboundFixture::gate(server,{});
+    for (auto& room : rooms) {
+      for (auto& peer : room.peers) {
+        if (peer.tcp) peer.tcp->close(); if (peer.udp) peer.udp->close();
+        peer.tcp.reset(); peer.udp.reset(); peer.retained.clear();
+      }
+      if (room.records) room.records->close(); if (room.admissions) room.admissions->close(); if (room.snapshots) room.snapshots->close();
+    }
+    slow->close();
+  };
+  const auto sample = [&](int ticks, const char* at) {
+    auto process = process_sample(); const auto metrics = server.metrics(), owned = server.cleanup();
+    std::size_t journals = 0, journal_high_water = 0, controls = 0, full = 0, delta = 0;
+    for (const auto& [id,row] : metrics.at("rooms").items()) {
+      (void)id; journals += row.at("replay_bytes").get<std::size_t>(); journal_high_water += row.at("replay_bytes_high_water").get<std::size_t>();
+      load_need(row.at("replay_capture_complete") && row.at("replay_bytes") <= max_replay_bytes && row.at("replay_pending_events") == 0,
+        "complete bounded owner journals at observation boundary");
+    }
+    for (const auto& [id,row] : metrics.at("outbound_streams").items()) {
+      (void)id; controls += row.at("control").get<std::size_t>(); full += row.at("full").get<std::size_t>(); delta += row.at("delta").get<std::size_t>();
+      load_need(row.at("full") <= 1 && row.at("delta") <= 1 && row.at("control") < max_control_messages,"per-connection real pending bounds");
+    }
+    process.update(Json{{"after_executed_ticks_per_room",ticks},{"boundary",at},{"monotonic_ms",shared_now},
+      {"owner_resources",owned},{"production_metrics",metrics},{"pending_controls",controls},{"pending_full",full},{"pending_delta",delta},
+      {"journal_serialized_bytes",journals},{"journal_serialized_high_water_bytes",journal_high_water},
+      {"snapshot_heap_bytes",nullptr},{"snapshot_heap_bytes_availability","no per-container allocator counter; actual retained counts and32-entry bound recorded"}});
+    samples.push_back(std::move(process));
+  };
+  try {
+    for (std::size_t index = 0; index < rooms.size(); ++index) {
+      auto& room = rooms[index]; room.id = scenario.at("rooms").at(index).at("room_id").get<std::string>(); indices.emplace(room.id,index);
+      room.records = std::make_unique<Lines>(output.parent_path()/(prefix+"."+room.id+".records.jsonl"));
+      room.admissions = std::make_unique<Lines>(output.parent_path()/(prefix+"."+room.id+".inputs.jsonl"));
+      room.snapshots = std::make_unique<Lines>(output.parent_path()/(prefix+"."+room.id+".snapshots.jsonl"));
+    }
+    multi_room_fixture::configure(server,scenario,[&](const Room& model, const ReplayLog& replay) {
+      auto& room = rooms.at(indices.at(model.id()));
+      load_need(room.hashes.size() < 600 && model.executed_ticks() == static_cast<int>(room.hashes.size())+1 && model.status() == "RUNNING",
+        "one actual owner tick per shared50ms step, session remains RUNNING");
+      const auto canonical = canonical_state(model), hash = sha256(canonical);
+      load_need(replay.last_state_hash() == hash,"actual replay and canonical tick hash");
+      for (const auto& [id,p] : model.players()) {
+        (void)id; load_need(p.connected && !p.disconnect_deadline && p.pending.empty() && p.score == 0 && p.last_tag_tick == -20 &&
+          p.last_accepted_seq() == static_cast<std::uint64_t>(model.executed_ticks()) && p.applied_seq == p.last_accepted_seq() &&
+          p.x >= 0 && p.x <= 100000 && p.y >= 0 && p.y <= 100000,"all four admitted inputs applied without identity, grace or authority drift");
+      }
+      room.current = state_of(model); room.current_hash = hash; room.hashes.push_back(hash); ++executed_total;
+      room.records->append(Json{{"tick",model.executed_ticks()-1},{"monotonic_ms",shared_now},{"state",room.current},
+        {"canonical_record",canonical},{"state_hash",hash}});
+    });
+    const auto wait_for = [&](const auto& ready) {
+      const auto deadline = std::chrono::steady_clock::now()+std::chrono::seconds(5);
+      while (!ready() && std::chrono::steady_clock::now() < deadline) server.pump();
+      load_need(ready(),"actual owner/socket readiness within5s ceiling");
+    };
+    const auto ingress = [&](std::uint64_t target) {
+      wait_for([&] { return multi_room_fixture::received(server) >= target && multi_room_fixture::drained(server); });
+      load_need(multi_room_fixture::received(server) == target,"exact datagram ingress without retry");
+    };
+    const auto receive = [&](LoadPeer& peer, const char* type) {
+      std::optional<Json> value;
+      wait_for([&] { if (!value) value = peer.udp->try_receive(); return value.has_value(); });
+      load_need(value->at("type") == type,"next actual UDP message has expected type, no skipped evidence"); return *value;
+    };
+    const auto consume_snapshot = [&](LoadRoom& room, LoadPeer& peer) {
+      const auto wire = receive(peer,"SNAPSHOT"); const auto sequence = wire.at("snapshot_seq").get<std::uint64_t>();
+      const int tick = wire.at("tick").get<int>(); const auto& state = tick < 0 ? room.initial : room.current;
+      const auto& hash = tick < 0 ? room.initial_hash : room.current_hash;
+      load_need(sequence == peer.latest+1 && tick == static_cast<int>(sequence)*2-3 && wire.at("state_hash") == hash &&
+        wire.at("room_id") == room.id && wire.at("owner_epoch") == 0 && encode_datagram(wire).size() <= max_datagram_bytes,
+        "continuous actual snapshot sequence, byte bound and captured tick hash");
+      std::map<std::string,Json> players; std::string status;
+      if (wire.at("kind") == "FULL") {
+        load_need(wire.at("base_snapshot_seq").is_null() && wire.size() == 12,"FULL wire contract"); status = wire.at("status").get<std::string>();
+      } else {
+        load_need(wire.at("kind") == "DELTA" && wire.size() == 11 && peer.retained.contains(wire.at("base_snapshot_seq").get<std::uint64_t>()),
+          "delta base was actually received and applied");
+        const auto& base = peer.retained.at(wire.at("base_snapshot_seq").get<std::uint64_t>()); status = base.at("status").get<std::string>();
+        for (const auto& row : base.at("players")) players.emplace(row.at("player_id").get<std::string>(),row);
+      }
+      for (const auto& id : wire.at("removed_player_ids")) players.erase(id.get<std::string>());
+      std::string previous;
+      for (const auto& row : wire.at("players")) {
+        const auto id = row.at("player_id").get<std::string>(); load_need(row.size() == 7 && id > previous,"minimal sorted projection");
+        previous = id; players[id] = row;
+      }
+      Json rows = Json::array(); for (const auto& [id,row] : players) { (void)id; rows.push_back(row); }
+      Json applied{{"room_id",room.id},{"tick",tick},{"owner_epoch",0},{"status",status},{"players",rows}};
+      load_need(applied == visible(state),"actual healthy replica equals the captured Room projection");
+      peer.latest = sequence; peer.retained[sequence] = applied;
+      if (peer.retained.size() > snapshot_retention) peer.retained.erase(peer.retained.begin());
+      auto ack = message("SNAPSHOT_ACK"); ack.update(Json{{"session_id",peer.session},{"room_id",room.id},{"player_id",peer.player},
+        {"snapshot_seq",sequence},{"state_hash",hash},{"owner_epoch",0}}); peer.udp->send(ack); ++peer.snapshot_acks;
+      room.snapshots->append(Json{{"player_id",peer.player},{"wire",wire},{"ack",redact(ack,peer.player)},{"replica_equal",true}});
+    };
+    for (auto& room : rooms) {
+      Json joins = Json::array();
+      for (auto& peer : room.peers) {
+        peer.tcp = std::make_unique<TcpClient>(server.port()); peer.tcp->send(message("HELLO"));
+        const auto welcome = peer.tcp->receive_type(server,"WELCOME"); peer.session = welcome.at("session_id").get<std::string>();
+        load_need(peer.tcp->has_bind_token() && !welcome.contains("udp_bind_token"),"real private HELLO credential");
+        peer.connection = multi_room_fixture::connection_id(server,peer.session); peer.udp = std::make_unique<UdpClient>(server.udp_port());
+      }
+      auto create = message("CREATE_ROOM"); create["session_id"] = room.peers[0].session; room.peers[0].tcp->send(create);
+      const auto created = room.peers[0].tcp->receive_type(server,"ROOM_CREATED");
+      load_need(created.at("room_id") == room.id && created.at("status") == "LOBBY","ordinary real CREATE_ROOM");
+      for (std::size_t slot = 0; slot < 4; ++slot) {
+        auto& peer = room.peers[slot]; auto join = message("JOIN_ROOM"); join.update(Json{{"session_id",peer.session},{"room_id",room.id}});
+        peer.tcp->send(join); auto reply = peer.tcp->receive_type(server,"ROOM_JOINED"); peer.player = reply.at("player_id").get<std::string>();
+        const auto& fixed = scenario.at("rooms").at(indices.at(room.id)).at("players").at(slot); const auto& player = server.room(room.id).players().at(peer.player);
+        load_need(reply.at("status") == "LOBBY" && reply.at("slot") == slot && peer.player == fixed.at("player_id").get<std::string>() &&
+          player.x == fixed.at("spawn").at(0) && player.y == fixed.at("spawn").at(1) && peer.tcp->has_resume_token(),"normal four unbound joins and spawn mapping");
+        joins.push_back(Json{{"reply",reply},{"bind_credential_present",true},{"resume_credential_present",true}});
+      }
+      for (std::size_t slot = 0; slot < 4; ++slot) {
+        auto& peer = room.peers[slot]; peer.udp->bind(*peer.tcp,server,peer.session);
+        load_need(server.room(room.id).status() == (slot == 3 ? "RUNNING" : "LOBBY"),"normal min2 AND all joined UDP-ready start");
+      }
+      room.initial = room.current = state_of(server.room(room.id)); room.initial_hash = room.current_hash = sha256(canonical_state(server.room(room.id)));
+      const auto received = multi_room_fixture::received(server); for (auto& peer : room.peers) consume_snapshot(room,peer); ingress(received+4);
+      for (auto& peer : room.peers) load_need(multi_room_fixture::acknowledged(server,peer.connection) == 1,"actual initial FULL1 ACK");
+      setup.push_back(Json{{"room_id",room.id},{"created",created},{"joins",joins},{"initial_state",room.initial},{"initial_hash",room.initial_hash}});
+    }
+    mapping = multi_room_fixture::mapping(server); const auto initialized = multi_room_fixture::schedule(server);
+    load_need(mapping.size() == 32 && initialized.at("scheduled_room_ids").size() == 32 && executed_clock.now_ms == 0 && shared_now == 0,
+      "one instance and32 actual owner models, no initialization ticks");
+    descriptors = server.owned_descriptors(); descriptors.push_back(slow->fd.get());
+    for (auto& room : rooms) {
+      descriptors.push_back(room.records->fd.get()); descriptors.push_back(room.admissions->fd.get()); descriptors.push_back(room.snapshots->fd.get());
+      for (auto& peer : room.peers) { descriptors.push_back(peer.tcp->descriptor()); descriptors.push_back(peer.udp->descriptor()); }
+    }
+    OutboundFixture::hold_snapshots(server,rooms[0].peers[0].connection,hold,[&](const Json& queue) {
+      load_need(queue.at("full") <= 1 && queue.at("delta") <= 1 && queue.at("snapshots_sent") == 1 && queue.at("stream").at("acknowledged_seq") == 1 &&
+        queue.at("queued_buffers") == queue.at("live_buffers"),"snapshot-only pre-dequeue ownership, no pinned base or dropped admission ACK");
+      auto row = queue; row["monotonic_ms"] = shared_now; slow->append(row);
+    });
+    sample(0,"initialized");
+    control_boundary("G14_READY","RUN",controlled,Json{{"executed_ticks",0}});
+    phase = "measured-live"; const auto cpu_start = process_sample(); const auto wall_start = std::chrono::steady_clock::now();
+    std::uint32_t random = 7050; const auto directions = generator.at("directions").get<std::vector<std::string>>();
+    for (int tick = 0; tick < 600; ++tick) {
+      for (auto& room : rooms) {
+        const auto received = multi_room_fixture::received(server); std::array<Json,4> requests;
+        for (std::size_t slot = 0; slot < 4; ++slot) {
+          random = random*std::uint32_t{1664525}+std::uint32_t{1013904223}; auto& peer = room.peers[slot];
+          requests[slot] = Json{{"v",1},{"type","INPUT"},{"session_id",peer.session},{"room_id",room.id},{"player_id",peer.player},
+            {"seq",tick+1},{"target_tick",tick},{"direction",directions.at((random >> 16U)%5U)},{"tag_target_player_id",nullptr},{"owner_epoch",0}};
+          peer.udp->send(requests[slot]);
+        }
+        ingress(received+4);
+        for (std::size_t slot = 0; slot < 4; ++slot) {
+          auto& peer = room.peers[slot]; const auto reply = receive(peer,"INPUT_ACK");
+          load_need(reply.at("accepted") && reply.at("code") == "ACCEPTED" && reply.at("seq") == tick+1 && reply.at("tick") == tick &&
+            reply.at("player_id") == peer.player && server.room(room.id).executed_ticks() == tick,"actual current tick UDP admission and ACK before shared clock step");
+          const auto& player = server.room(room.id).players().at(peer.player);
+          load_need(player.pending.size() == 1 && player.last_accepted_seq() == static_cast<std::uint64_t>(tick+1) && player.connected,"one real pending intent for each player, slow included");
+          ++peer.accepted; ++accepted_total;
+          room.admissions->append(Json{{"request",redact(requests[slot],peer.player)},{"response",reply},{"admitted_at_ms",shared_now}});
+        }
+      }
+      load_need(accepted_total == static_cast<std::size_t>(tick+1)*128 && shared_now == tick*50,"all128 ACK barrier without pacing/resend or extra PRNG draws");
+      shared_now += 50; const auto batch = server.run_scheduler();
+      load_need(batch.ticks == 32 && batch.remaining_ms == 0 && !batch.overloaded && executed_clock.now_ms == (tick+1)*50 &&
+        executed_total == static_cast<std::size_t>(tick+1)*32,"every actual Room serviced once, separate executed-time counter preserved");
+      if (tick%2 == 1) {
+        // Bound real ACK ingress groups to four; do not overfill the shared
+        // UDP receive queue merely to implement the fixture's consumers.
+        for (auto& room : rooms) {
+          const auto received = multi_room_fixture::received(server); std::size_t sent = 0;
+          for (auto& peer : room.peers) if (&peer != &rooms[0].peers[0]) { consume_snapshot(room,peer); ++sent; }
+          ingress(received+sent);
+          for (auto& peer : room.peers) load_need(multi_room_fixture::acknowledged(server,peer.connection) == peer.latest,"server ACK watermark follows only actually applied snapshot");
+        }
+      }
+      if ((tick+1)%200 == 0) sample(tick+1,"live");
+    }
+    const auto wall_end = std::chrono::steady_clock::now(); const auto cpu_end = process_sample();
+    interval = Json{{"wall_seconds",std::chrono::duration<double>(wall_end-wall_start).count()},
+      {"steady_start_seconds",std::chrono::duration<double>(wall_start.time_since_epoch()).count()},
+      {"steady_end_seconds",std::chrono::duration<double>(wall_end.time_since_epoch()).count()},
+      {"cpu_user_seconds",cpu_end.at("cpu_user_seconds").get<double>()-cpu_start.at("cpu_user_seconds").get<double>()},
+      {"cpu_system_seconds",cpu_end.at("cpu_system_seconds").get<double>()-cpu_start.at("cpu_system_seconds").get<double>()},
+      {"scope","one Release process: production owner/transport plus128 real test clients, their existing4096-message observation caps (~902 messages each), assertions, streamed observers and samples; initialization/export/shutdown/offline excluded"},
+      {"start_boundary","before tick0 INPUT"},{"end_boundary","after tick599 healthy snapshots and actual ACK ingress"},
+      {"cpu_start",cpu_start},{"cpu_end",cpu_end}};
+    control_boundary("G14_MEASURED_DONE","EXPORT",controlled,Json{{"executed_ticks",executed_total},{"wall_seconds",interval.at("wall_seconds")}});
+    phase = "export"; const auto metrics = server.metrics(); Json room_results = Json::object();
+    load_need(accepted_total == 76800 && executed_total == 19200 && shared_now == 30000 && hold.direct_udp_ready == 600 && slow->count == 300,
+      "fixed live count, slow600 direct replies and300 actual held snapshot generations");
+    for (std::size_t index = 0; index < rooms.size(); ++index) {
+      auto& room = rooms[index]; Json peers = Json::object();
+      load_need(room.hashes.size() == 600 && room.records->count == 600 && room.admissions->count == 2400 && server.room(room.id).status() == "RUNNING",
+        "all600 live owner records and2400 actual admissions retained");
+      for (std::size_t slot = 0; slot < 4; ++slot) {
+        auto& peer = room.peers[slot]; const auto expected = index == 0 && slot == 0 ? 1U : 301U;
+        load_need(peer.accepted == 600 && peer.latest == expected && peer.snapshot_acks == expected && !peer.udp->try_receive() &&
+          !peer.tcp->try_receive() && !peer.tcp->peer_closed(),"every required actual INPUT_ACK/snapshot consumed once, all control transports remain open");
+        std::size_t observed_inputs = 0, observed_snapshots = 0;
+        for (const auto& event : peer.udp->observations()) { observed_inputs += event.at("type") == "INPUT_ACK"; observed_snapshots += event.at("type") == "SNAPSHOT"; }
+        load_need(observed_inputs == 600 && observed_snapshots == expected,"actual client observations independently count ACK and publication totals");
+        peers[peer.player] = Json{{"accepted_inputs",peer.accepted},{"observed_input_acks",observed_inputs},{"received_snapshots",observed_snapshots},
+          {"sent_snapshot_acks",peer.snapshot_acks},{"server_acknowledged_seq",multi_room_fixture::acknowledged(server,peer.connection)},
+          {"last_applied_seq",peer.latest},{"connectivity",server.room(room.id).players().at(peer.player).connectivity()},
+          {"tcp_open",true},{"udp_observations",peer.udp->observations().size()},{"client_retained_count",peer.retained.size()}};
+      }
+      auto value = Json{{"executed_ticks",600},{"state_hashes",room.hashes},{"final_state",room.current},{"players",peers},
+        {"records",room.records->summary()},{"admissions",room.admissions->summary()},{"snapshots",room.snapshots->summary()},
+        {"schedule",metrics.at("rooms").at(room.id)}};
+      if (index != 0) {
+        const auto bytes = server.replay(room.id).serialize(); const auto& artifact = server.replay(room.id).artifact(); std::size_t accepted = 0;
+        for (const auto& tick : artifact.at("ticks")) accepted += tick.at("events").size();
+        load_need(artifact.at("ticks").size() == 600 && accepted == 2400 && bytes.size() <= max_replay_bytes,"actual immutable600-tick accepted journal");
+        const auto path = output.parent_path()/(prefix+"."+room.id+".replay.json");
+        load_need(!std::filesystem::exists(path),"replay evidence never overwritten"); std::ofstream file(path,std::ios::binary);
+        file << bytes; file.close(); load_need(file.good(),"complete journal export");
+        value.update(Json{{"replay_artifact",std::filesystem::absolute(path).string()},{"artifact_bytes",bytes.size()},
+          {"artifact_sha256",sha256(bytes)},{"accepted_journal_events",accepted}});
+      }
+      room_results[room.id] = std::move(value);
+    }
+    const auto held = OutboundFixture::inspect(server,rooms[0].peers[0].connection);
+    load_need(held.at("snapshots_generated") == 301 && held.at("snapshots_sent") == 1 && held.at("snapshots_coalesced") == 299 &&
+      held.at("full") == 1 && held.at("delta") == 0 && held.at("pending_full").at("wire").at("snapshot_seq") == 301 &&
+      held.at("pending_full").at("wire").at("tick") == 599 && held.at("pending_full").at("wire").at("kind") == "FULL",
+      "actual final FULL301,299 destroyed superseded buffers and one retained snapshot");
+    load_need(metrics.at("errors").empty() && metrics.at("connection_high_water") == 128 && metrics.at("room_high_water") == 32 &&
+      metrics.at("input_per_player_high_water") <= max_pending_inputs && metrics.at("total_pending_high_water") <= max_total_pending_inputs &&
+      metrics.at("mailbox_high_water") <= max_mailbox_messages && metrics.at("outbound_control_high_water") < max_control_messages &&
+      metrics.at("snapshot_retention_high_water") <= snapshot_retention && metrics.at("udp_payload_high_water") <= max_datagram_bytes &&
+      metrics.at("udp_outbound_high_water") <= max_datagram_bytes,"all fixed bounded production queues/bytes without capacity campaigns");
+    const auto before_shutdown = server.cleanup(); cleanup(); sample(600,"after-shutdown");
+    auto final = server.cleanup(); for (const auto& [key,value] : final.items()) { (void)key; load_need(value == 0,"all active owner/transport/work resources released"); }
+    for (const auto fd : descriptors) load_need(descriptor_closed(fd),"actual socket/reactor/raw-file descriptor closure");
+    const auto after = server.metrics();
+    load_need(Fd::live() == fd_before && after.at("outbound_buffers_created") == after.at("outbound_buffers_released"),"all tracked FD and actual buffer lifetimes end");
+    final.update(Json{{"descriptor_checks",descriptors.size()},{"all_descriptors_closed",true},{"tracked_descriptor_delta",0},{"raw_files_closed",97}});
+    const auto memory_path = output.parent_path()/(prefix+".memory.json"); write_json_file(memory_path,samples);
+    result = Json{{"result","PASS"},{"thread","G14"},{"scenario_id",scenario.at("scenario_id")},{"process_id",::getpid()},
+      {"build_mode","Release"},{"server_instances",1},{"executed_ticks",executed_total},{"accepted_inputs",accepted_total},
+      {"reference_ticks_reserved",18600},{"reference_processes_reserved",31},{"shared_monotonic_ms",shared_now},{"final_lcg_state",random},
+      {"default_executed_simulation_ms",executed_clock.now_ms},{"setup",setup},{"owner_mapping",mapping},{"rooms",room_results},
+      {"measurement",interval},{"before_initialization",before_process},
+      {"memory_samples",std::filesystem::absolute(memory_path).string()},{"memory_sample_count",samples.size()},{"metrics",metrics},
+      {"slow",Json{{"trace",slow->summary()},{"not_ready_callbacks",hold.not_ready},{"direct_udp_ready",hold.direct_udp_ready},
+        {"control_ready",hold.control_ready},{"final_pending",held},{"snapshot_hold_only",true}}},
+      {"before_shutdown",before_shutdown},{"outbound_buffers_created",after.at("outbound_buffers_created")},
+      {"outbound_buffers_released",after.at("outbound_buffers_released")},{"cleanup",final}};
+  } catch (const std::exception& error) {
+    const auto failure = std::string(error.what()).substr(0,256); std::string cleanup_failure;
+    try { cleanup(); } catch (const std::exception& cleanup_error) { cleanup_failure = std::string(cleanup_error.what()).substr(0,256); }
+    result = Json{{"result","FAIL"},{"thread","G14"},{"scenario_id",scenario.at("scenario_id")},{"process_id",::getpid()},
+      {"failure",failure},{"failure_phase",phase},{"executed_ticks",executed_total},{"accepted_inputs",accepted_total},
+      {"shared_monotonic_ms",shared_now},{"memory_samples",samples},{"measurement",interval},{"cleanup",server.cleanup()},
+      {"cleanup_failure",cleanup_failure},{"slow_generations",slow->count},{"source_records_are_partial",true}};
+  }
+  return result;
+}
+}
diff --git a/tests/g14.hpp b/tests/g14.hpp
new file mode 100644
index 0000000..0f4aa00
--- /dev/null
+++ b/tests/g14.hpp
@@ -0,0 +1,5 @@
+#pragma once
+#include "g13.hpp"
+namespace arena {
+Json run_release_load(const Json& scenario, const std::filesystem::path& output, bool controlled);
+}
diff --git a/tests/g14_run.py b/tests/g14_run.py
new file mode 100644
index 0000000..85deaca
--- /dev/null
+++ b/tests/g14_run.py
@@ -0,0 +1,253 @@
+#!/usr/bin/env python3
+"""One reserved G14 live process, optional single sample, then31 explicit replays.
+
+This command neither reserves nor retries a load. Its caller must use the frozen
+root slot launcher. No unit/integration command calls this runner. Profiling is
+off by default, including root's main-final. All children stay in the launcher's
+process group; cleanup never targets a process name or unrelated process.
+"""
+import argparse
+import datetime
+import hashlib
+import json
+import os
+from pathlib import Path
+import selectors
+import signal
+import subprocess
+import sys
+import time
+
+FIXTURE_SHA = "9e3045a578683cab48abe838a71336b457863811098228462153df7c512b5e6d"
+
+
+def stamp():
+    return {"utc": datetime.datetime.now(datetime.timezone.utc).isoformat(),
+            "monotonic_seconds": time.monotonic()}
+
+
+def sha(path):
+    return hashlib.sha256(Path(path).read_bytes()).hexdigest()
+
+
+def require(value, message):
+    if not value:
+        raise RuntimeError(message)
+
+
+def write_json(path, value):
+    data = json.dumps(value, indent=2) + "\n"
+    require(len(data.encode()) <= 4194304, "bounded result export")
+    with path.open("x") as stream:
+        stream.write(data)
+
+
+def main():
+    parser = argparse.ArgumentParser(description=__doc__)
+    parser.add_argument("--binary", type=Path, required=True)
+    parser.add_argument("scenario", type=Path)
+    parser.add_argument("output", type=Path)
+    parser.add_argument("--ledger", type=Path)
+    parser.add_argument("--profile", action="store_true")
+    args, extra = parser.parse_known_args()
+    binary, scenario, output = (p.resolve() for p in (args.binary, args.scenario, args.output))
+    require(scenario.stat().st_size <= 1048576, "bounded scenario input")
+    if json.loads(scenario.read_text()).get("thread") != "G14":
+        require(not args.profile and args.ledger is None, "G14 options require its fixture")
+        os.execv(str(binary), [str(binary), "scenario-run", str(scenario), str(output), *extra])
+    require(not extra, "unknown G14 option")
+    require(sha(scenario) == FIXTURE_SHA, "fixed G14 fixture bytes")
+    require(not output.exists(), "never overwrite an earlier load result")
+    output.parent.mkdir(parents=True, exist_ok=True)
+    ledger = args.ledger.resolve() if args.ledger else output.parent / "commands.jsonl"
+    ledger.parent.mkdir(parents=True, exist_ok=True)
+    live_path, live_log = output.parent / "live.json", output.parent / "live.log"
+    require(not live_path.exists() and not live_log.exists(), "one live invocation per output directory")
+    started = stamp()
+    deadline = time.monotonic() + 600
+    children = []
+    records = []
+    timeline = []
+    profiler = {"requested": args.profile, "capture_count": 0, "status": "NOT_REQUESTED"}
+    profile_process = None
+    profile_stream = None
+    live = None
+    live_record = None
+    references = []
+    result = {"result": "FAIL", "thread": "G14", "fixture_sha256": FIXTURE_SHA,
+              "wrapper_pid": os.getpid(), "started": started,
+              "live_result": str(live_path), "ledger": str(ledger)}
+
+    def record(value):
+        with ledger.open("a") as stream:
+            stream.write(json.dumps(value, separators=(",", ":")) + "\n")
+            stream.flush()
+
+    def launch(argv, category, log, ticks, **kwargs):
+        entry = {"event": "started", "category": category, "argv": argv,
+                 "cwd": str(Path.cwd()), "log": str(log), "ticks": ticks,
+                 "wrapper_pid": os.getpid(), "started": stamp()}
+        record(entry)
+        process = subprocess.Popen(argv, **kwargs)
+        children.append(process)
+        entry["pid"] = process.pid
+        records.append(entry)
+        record(dict(entry, event="spawned"))
+        return process, entry
+
+    def finish(process, entry):
+        if entry.get("event") == "finished":
+            return
+        entry.update(event="finished", exit_code=process.returncode, finished=stamp(), reaped=process.returncode is not None)
+        record(entry)
+
+    def profile_finish():
+        if profile_process is None or profiler.get("reaped") is True:
+            return
+        profile_process.wait(timeout=min(20, max(0.001, deadline-time.monotonic())))
+        finish(profile_process, profiler["command"])
+        path = Path(profiler["raw_file"])
+        profiler.update(exit_code=profile_process.returncode, reaped=True, observed_finish=stamp(),
+                        status="CAPTURED" if profile_process.returncode == 0 and path.exists() else "UNAVAILABLE",
+                        raw_sha256=sha(path) if path.exists() else None)
+
+    def interrupt(signum, frame):
+        del frame
+        raise InterruptedError("runner interrupted by signal " + str(signum))
+
+    for sig in (signal.SIGTERM, signal.SIGINT, signal.SIGHUP):
+        signal.signal(sig, interrupt)
+    try:
+        argv = [str(binary), "scenario-run", str(scenario), str(live_path), "--load-control"]
+        with live_log.open("xb") as raw:
+            live, live_record = launch(argv, "load-live", live_log, 19200,
+                                       stdin=subprocess.PIPE, stdout=subprocess.PIPE, stderr=subprocess.STDOUT, bufsize=0)
+            selector = selectors.DefaultSelector()
+            selector.register(live.stdout, selectors.EVENT_READ)
+            pending = b""
+            phase = "initialization"
+            while selector.get_map():
+                require(time.monotonic() < deadline, "600s whole-load process ceiling")
+                if profile_process is not None and profile_process.poll() is not None and profiler.get("reaped") is not True:
+                    profile_finish()
+                for key, _ in selector.select(timeout=min(1, max(0, deadline-time.monotonic()))):
+                    chunk = os.read(key.fd, 65536)
+                    if not chunk:
+                        selector.unregister(key.fileobj)
+                        break
+                    raw.write(chunk)
+                    raw.flush()
+                    pending += chunk
+                    require(len(pending) <= 1048576, "bounded child stdout line")
+                    while b"\n" in pending:
+                        line, pending = pending.split(b"\n", 1)
+                        try:
+                            event = json.loads(line)
+                        except (ValueError, UnicodeDecodeError):
+                            continue
+                        if event.get("event") == "G14_READY":
+                            require(phase == "initialization" and event["process_id"] == live.pid and event["executed_ticks"] == 0,
+                                    "one initialized live PID before first tick")
+                            timeline.append({"kind": "READY", "observed": stamp(), "child": event})
+                            if args.profile:
+                                profile_file = output.parent / "native-sample.txt"
+                                profile_log = output.parent / "native-sample.log"
+                                profile_stream = profile_log.open("xb")
+                                sample_argv = ["/usr/bin/sample", str(live.pid), "5", "1", "-mayDie", "-file", str(profile_file)]
+                                profiler.update(capture_count=1, target_pid=live.pid, raw_file=str(profile_file), log=str(profile_log), status="STARTING")
+                                profile_process, profile_record = launch(sample_argv, "profiler", profile_log, 0,
+                                                                           stdout=profile_stream, stderr=subprocess.STDOUT)
+                                profiler.update(pid=profile_process.pid, command=profile_record, started=stamp())
+                            timeline.append({"kind": "RUN", **stamp()})
+                            live.stdin.write(b"RUN\n")
+                            phase = "live"
+                        elif event.get("event") == "G14_MEASURED_DONE":
+                            require(phase == "live" and event["process_id"] == live.pid and event["executed_ticks"] == 19200,
+                                    "one complete fixed live interval")
+                            timeline.append({"kind": "DONE", "observed": stamp(), "child": event})
+                            profile_finish()
+                            timeline.append({"kind": "EXPORT", **stamp()})
+                            live.stdin.write(b"EXPORT\n")
+                            live.stdin.close()
+                            phase = "export"
+            selector.close()
+            live.wait(timeout=max(0.001, deadline-time.monotonic()))
+            finish(live, live_record)
+        require(live.returncode == 0, "live process failed; no replay or later load is authorized")
+        require(phase == "export", "live phase handoffs missing")
+        observed = json.loads(live_path.read_text())
+        require(observed["result"] == "PASS" and observed["executed_ticks"] == 19200 and observed["accepted_inputs"] == 76800,
+                "complete fixed live evidence required")
+        result.update(executed_ticks=observed["executed_ticks"], accepted_inputs=observed["accepted_inputs"],
+                      live_sha256=sha(live_path), measurement=observed["measurement"], cleanup=observed["cleanup"])
+        raw_files = []
+        for room in observed["rooms"].values():
+            for kind in ("records", "admissions", "snapshots"):
+                raw_files.append(dict(room[kind], kind=kind, sha256=sha(room[kind]["path"])))
+        raw_files.append(dict(observed["slow"]["trace"], kind="slow-pre-dequeue", sha256=sha(observed["slow"]["trace"]["path"])))
+        result.update(raw_files=raw_files, memory_samples_sha256=sha(observed["memory_samples"]), binary_sha256=sha(binary))
+        # No replay starts until measurement, sample, export, shutdown and the
+        # live process have all finished. Each Room has its own real process.
+        for index in range(1, 32):
+            room_id = f"room-{index:02d}"
+            room = observed["rooms"][room_id]
+            replay = Path(room["replay_artifact"])
+            require(sha(replay) == room["artifact_sha256"], "immutable actual accepted journal")
+            reference = output.parent / ("reference." + room_id + ".json")
+            log = output.parent / ("reference." + room_id + ".log")
+            command = [str(binary), "replay-verify", str(replay), str(reference)]
+            with log.open("xb") as stream:
+                process, entry = launch(command, "reference", log, 600, stdout=stream, stderr=subprocess.STDOUT)
+                process.wait(timeout=max(0.001, deadline-time.monotonic()))
+                finish(process, entry)
+            require(process.returncode == 0, "reference failed; no replay retry is authorized")
+            check = json.loads(reference.read_text())
+            require(check["result"] == "PASS" and check["executed_ticks"] == 600 and check["state_hashes"] == room["state_hashes"],
+                    "all600 actual replay/live hashes")
+            live_records = [json.loads(line) for line in Path(room["records"]["path"]).read_text().splitlines()]
+            reference_records = json.loads(Path(check["canonical_records"]).read_text())
+            require(len(live_records) == len(reference_records) == 600 and all(
+                a["tick"] == b["tick"] and a["canonical_record"] == b["canonical_record"] and a["state_hash"] == b["state_hash"]
+                for a, b in zip(live_records, reference_records)), "all600 actual canonical record pairs")
+            references.append({"room_id": room_id, "result": "PASS", "executed_ticks": 600, "process_id": process.pid,
+                               "output": str(reference), "sha256": sha(reference), "artifact_sha256": sha(replay),
+                               "records_sha256": sha(check["canonical_records"]), "matched_record_pairs": 600})
+        result.update(result="PASS", reference_ticks=sum(row["executed_ticks"] for row in references),
+                      references=references, network_fault_runs=0, live_load_runs=1)
+    except BaseException as error:
+        result.update(result="FAIL", error=f"{type(error).__name__}: {error}", references=references,
+                      reference_ticks=sum(row["executed_ticks"] for row in references), partial_evidence_preserved=True)
+    finally:
+        # The root launcher is an additional process-group fail-safe. Here only
+        # our directly owned children are signalled/reaped, never relaunched.
+        for sig in (signal.SIGTERM, signal.SIGINT, signal.SIGHUP):
+            signal.signal(sig, signal.SIG_IGN)
+        for process in reversed(children):
+            if process.poll() is None:
+                process.terminate()
+                try:
+                    process.wait(timeout=3)
+                except subprocess.TimeoutExpired:
+                    process.kill()
+                    process.wait(timeout=2)
+                result["result"] = "FAIL"
+                result["forced_child_cleanup"] = True
+            for entry in records:
+                if entry.get("pid") == process.pid:
+                    finish(process, entry)
+        if profile_stream is not None:
+            profile_stream.close()
+        if live is not None:
+            if live.stdout is not None:
+                live.stdout.close()
+            if live.stdin is not None:
+                live.stdin.close()
+        result.update(finished=stamp(), timeline=timeline, profiler=profiler,
+                      child_pids=[process.pid for process in children], all_children_reaped=all(process.returncode is not None for process in children))
+        write_json(output, result)
+    print(json.dumps({key: result.get(key) for key in ("result", "executed_ticks", "reference_ticks", "accepted_inputs", "error")}), flush=True)
+    return 0 if result["result"] == "PASS" else 1
+
+
+if __name__ == "__main__":
+    raise SystemExit(main())
diff --git a/tests/scenario_main.cpp b/tests/scenario_main.cpp
index f1b1fa8..1ebb23b 100644
--- a/tests/scenario_main.cpp
+++ b/tests/scenario_main.cpp
@@ -4,6 +4,7 @@
 #include "g11.hpp"
 #include "g12.hpp"
 #include "g13.hpp"
+#include "g14.hpp"
 #ifndef ARENA_TEST_FIXTURES
 #error Scenario fixture executable requires its separate test core
 #endif
@@ -33,14 +34,17 @@ int main(int argc, char** argv) {
     arena::ReplayRun run;
     if (command == "scenario-run") {
       const bool variant = argc == 6 && std::string(argv[4]) == "--variant" && std::string(argv[5]) == "rejected-removed";
-      if (argc != 4 && !variant) throw std::invalid_argument("unknown scenario variant");
+      const bool load_control = argc == 5 && std::string(argv[4]) == "--load-control";
+      if (argc != 4 && !variant && !load_control) throw std::invalid_argument("unknown scenario variant");
       const auto scenario = arena::read_json_file(input);
+      if (load_control && scenario.at("thread") != "G14") throw std::invalid_argument("load control is only active for G14");
       if (scenario.at("thread") != "G07" && scenario.at("thread") != "G09" && scenario.at("thread") != "G12") {
         if (variant) throw std::invalid_argument("variant is only active for G07");
         const auto evidence = scenario.at("thread") == "G08" ? arena::run_snapshot_scenario(scenario) :
           scenario.at("thread") == "G10" ? arena::run_ack_scenario(scenario) :
           scenario.at("thread") == "G11" ? arena::run_resume_scenario(scenario) :
-          scenario.at("thread") == "G13" ? arena::run_isolation_scenario(scenario,output) : arena::run_scenario(scenario);
+          scenario.at("thread") == "G13" ? arena::run_isolation_scenario(scenario,output) :
+          scenario.at("thread") == "G14" ? arena::run_release_load(scenario,output,load_control) : arena::run_scenario(scenario);
         arena::write_json_file(output,evidence);
         std::cout << arena::Json{{"result",evidence.at("result")},{"executed_ticks",evidence.at("executed_ticks")},
           {"scenario_id",evidence.at("scenario_id")},{"evidence",output.string()},{"cleanup",evidence.at("cleanup")}}.dump() << '\n';
diff --git a/track b/track
index 1015abc..58214ca 100755
--- a/track
+++ b/track
@@ -30,7 +30,7 @@ case "$command" in
     ;;
   unit-test) run unit "$build_dir/arena_tests" unit ;;
   integration-test) run integration "$build_dir/arena_tests" integration ;;
-  scenario-run) run scenario "$build_dir/arena_scenarios" scenario-run "$@" ;;
+  scenario-run) run scenario python3 "$root/tests/g14_run.py" --binary "$build_dir/arena_scenarios" "$@" ;;
   replay-verify) run replay "$build_dir/arena_scenarios" replay-verify "$@" ;;
   server) run server "$build_dir/arena" server "$@" ;;
   *) printf 'unknown command: %s\n' "$command" >&2; exit 2 ;;
