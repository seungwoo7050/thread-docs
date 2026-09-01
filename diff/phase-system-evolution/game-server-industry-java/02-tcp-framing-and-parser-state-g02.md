# TCP Framing과 Parser State

## `feat: add bounded incremental TCP framing`

diff --git a/TRACK.md b/TRACK.md
index 70a535f..af98141 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -1,6 +1,6 @@
-# Java arena — G01 baseline
+# Java arena — through G02
 
-Thread: G01. Profile: realtime-core. Spec revision: `5a6e4a2f8fc71d4be18c3279583bfc2558d5c232`.
+Current thread: G02 (G01 baseline retained). Profile: realtime-core. Spec revision: `5a6e4a2f8fc71d4be18c3279583bfc2558d5c232`.
 
 ## Frozen versions
 
@@ -20,31 +20,38 @@ The wrapper uses the locally installed Temurin path when JAVA_HOME is unset. On
 ./track unit-test           # Gradle test, excludes ServerIntegrationTest
 ./track integration-test    # real loopback tests, timer/executor cleanup and bounded CLI SIGTERM
 ./track scenario-run /absolute/path/to/G01.json /absolute/path/to/result.json
+./track scenario-run /absolute/path/to/G02.json /absolute/path/to/framing-evidence.json
 ./track replay-verify /absolute/path/to/replay /absolute/path/to/evidence
 ./track server config/server.json
 ```
 
-`replay-verify` exits 2 with NOT ACTIVATED until G07. Build does not execute tests or scenarios. Unit/integration tasks re-execute tests every invocation; compilation is reused after the explicit build. Reports are in `build/test-results/{test,integrationTest}` and `build/reports/tests/{test,integrationTest}`. Netty leak detection is PARANOID for all test JVMs. Shell output and command exits are recorded in `evidence/G01-verification.md`; large/generated outputs are ignored.
+`replay-verify` exits 2 with NOT ACTIVATED until G07. Build does not execute tests or scenarios. Unit/integration tasks re-execute tests every invocation; compilation is reused after the explicit build. Reports are in `build/test-results/{test,integrationTest}` and `build/reports/tests/{test,integrationTest}`. Netty leak detection is PARANOID for all test JVMs. G01 verification remains in `evidence/G01-verification.md`. G02 commands, exits, durations and resource observations are recorded in `evidence/G02-verification.md` and `evidence/G02-observations.json`; raw stdout/XML copies are retained under ignored `evidence/runs/g02-initial/`.
 
 ## Ownership and bounds
 
-Connection lifetime belongs to its non-sharable Netty channel handler. Each accepted channel has one non-sharable `CompleteFrame` handler. It copies at most 16,384 JSON bytes and auto-releases each inbound `ByteBuf`. G01 requires exactly one complete frame per read, deliberately without an incremental parser. Receive allocation is 16,388 bytes and one message per read.
+Connection lifetime belongs to its non-sharable Netty channel handler. Each accepted channel has one non-sharable `CompleteFrame` handler. G02 replaces G01's one-complete-frame-per-read assumption with bounded incremental framing. A private, exclusively owned `ByteBuf` starts with four bytes of capacity and grows only after validating the unsigned payload length, to at most 16,388 bytes including the header. Reads copy only the current frame's missing bytes, so a coalesced read is parsed in order without accumulating the entire stream. A borrowed NIO view avoids another raw payload copy. Inbound read buffers are auto-released; cumulation is explicitly released on EOF, I/O error, terminal framing error and handler removal. Receive allocation remains 16,388 bytes and one buffer per read.
+
+Parser outcomes distinguish NEED_MORE_BYTES, COMPLETE_VALID_MESSAGE, MESSAGE_ERROR, TERMINAL_FRAME_ERROR and IO_END. Transport end reasons distinguish clean EOF, partial EOF, framing close and I/O error. Length 0 or >16,384 disables further reads and attempts FRAME_SIZE_INVALID before closing. Complete malformed messages remain recoverable. Strict UTF-8 decoding, duplicate-key detection, object-root enforcement, trailing-token rejection and active G02 schema checks occur before Room handoff. Missing/wrongly typed v/type are MESSAGE_INVALID, integer v other than 1 is PROTOCOL_VERSION_UNSUPPORTED, and unknown types are MESSAGE_TYPE_UNKNOWN. Unknown fields on known messages remain ignored. No sequence, target tick or later message schema is activated.
 
 Session registry and Room state belong to one dedicated room-owner thread. Network callbacks submit to its `ArrayBlockingQueue(1024)` and never mutate a Room. Each Room public operation checks the constructing owner thread; unit tests reject mutation from another thread. There is one room and at most eight accepted connections. UUID identifiers are server-generated, distinct from connection objects, and not input authority. Detailed lifecycle and identity matrices remain G03 work.
 
 Each player's pending input storage holds at most 64 intents and rejects overflow with `INPUT_QUEUE_FULL`. An owner tick drains that bounded storage, selects the latest pending direction/TAG, moves players in ASCII ID order, then evaluates one-shot TAG with 64-bit squared distance. Direction persists; TAG does not. No seq, target tick or rate-limit contract is activated. Player data is integer only; unknown position/score fields are ignored.
 
-Both Netty event loops use explicit bounded task and tail queues (1,024 each), not an unbounded executor queue. Room commands use a one-thread `ThreadPoolExecutor` with `AbortPolicy`; overflow causes a terminal `INPUT_QUEUE_FULL` reply attempt. Each connection bounds outstanding writes to 64. The last slot is reserved as a `CONTROL_BACKPRESSURE` terminal reply. No snapshot retention or delta queue exists at G01. Serialized outbound buffers transfer ownership to Netty on `writeAndFlush`; completion decrements an outstanding-buffer metric. Unit tests check actual inbound and outbound reference counts reach zero, including channel disposal. Snapshot cadence/coalescing remain later Threads.
+Both Netty event loops use explicit bounded task and tail queues (1,024 each), not an unbounded executor queue. Room commands use a one-thread `ThreadPoolExecutor` with `AbortPolicy`; overflow causes a terminal `INPUT_QUEUE_FULL` reply attempt. Each connection bounds outstanding writes to 64. The last slot is reserved as a `CONTROL_BACKPRESSURE` terminal reply. No snapshot retention or delta queue exists at G01. Parser error replies also pass through the same owner mailbox and bounded outbound path, preserving their order with preceding valid messages. Serialized outbound buffers transfer ownership to Netty on `writeAndFlush`; completion decrements an outstanding-buffer metric. Unit tests check actual inbound and outbound reference counts reach zero, including channel disposal. Snapshot cadence/coalescing remain later Threads.
 
 The manual clock advances an explicit 50ms per tick request with no sleeps or system-clock access. The TCP runner waits for INPUT_ACK sent after owner-side enqueue before advancing the clock. The standalone server uses a single 50ms timer thread with one wait and no delayed-task queue; G04 will replace its intentionally basic scheduling with an accumulator and bounded catch-up.
 
-The calling main/test thread coordinates shutdown: stop/join the timer, close listener and client channels, drain the I/O callback boundary, close/clear owner state, shut down/join the owner and both event loops. No event loop blocks on another thread. Clients observe LOBBY/RUNNING/FINISHED from server replies and CLOSED from TCP EOF, while the server records its actual terminal lifecycle. Assertions require zero live channels, pending writes, mailbox tasks and owned threads, terminated executors, stopped timer and locally closed client sockets.
+The calling main/test thread coordinates shutdown: stop/join the timer, close listener and client channels, drain the I/O callback boundary, close/clear owner state, shut down/join the owner and both event loops. No event loop blocks on another thread. Clients observe LOBBY/RUNNING/FINISHED from server replies and CLOSED from TCP EOF, while the server records its actual terminal lifecycle. Assertions require zero live channels, pending writes, parser buffers/allocated bytes, mailbox tasks and owned threads, terminated executors, stopped timer and locally closed client sockets. Unit tests inspect the actual cumulation reference count after release and reach exactly the maximum 16,388-byte capacity with a valid frame.
 
 The existing integration suite also starts the actual CLI as one child process on loopback port 0, waits for its SERVER_READY line, performs HELLO/WELCOME through TCP, sends normal SIGTERM, and requires exit 143 within five seconds, a zero-resource shutdown record and successful listener-port rebinding. The optional server config field `shutdown_evidence` names the JSON cleanup file written by the shutdown hook. The process check has no sleep, retry, changing load or canonical-scenario rerun.
 
 ## Fixed evidence and scope
 
-The canonical runner reads all clients, setup steps, input boundaries, directions, TAG roles and tick count from the supplied scenario. It resolves role names to actual server-issued identifiers and returns the final view received independently by both TCP clients. It never writes state or substitutes a separate simulation. Scenario SHA-256 is input provenance, not a state hash.
+The G01 canonical runner reads all clients, setup steps, input boundaries, directions, TAG roles and tick count from the supplied scenario. It resolves role names to actual server-issued identifiers and returns the final view received independently by both TCP clients. It never writes state or substitutes a separate simulation. Scenario SHA-256 is input provenance, not a state hash.
+
+G02 reads the supplied scenario's messages, fragmentation matrix, coalescing indices, malformed bytes and socket deadline. Exact read boundaries run through the same production handler in Netty EmbeddedChannel; the four malformed cases run over real loopback TCP while a separate connection continues HELLO/WELCOME. The test resource is an exact, SHA-256-checked copy of main's frozen G02 scenario, not an independently adjusted fixture. The pre-change unit run demonstrated seven failures and three all-at-once passes before production edits. Partial EOF, strict JSON forms, maximum frame capacity and transport I/O classification have fixed supplemental unit coverage.
+
+G02 initial ceilings: build/compile <=8, unit <=4 including pre-change reproduction, integration <=2, and one post-change canonical run because reproduction used a unit suite. Actual compile-bearing commands: 2 (one reproduction test compilation and one clean build); unit invocations: 2, integration: 1, post-change canonical: 1. All outputs/XML were preserved before clean builds could overwrite them. Network-fault and load runs are zero. G03 identity/lifecycle matrices and every later feature remain inactive.
 
 G01 initial budget: build/compile <=8, unit suites <=4, integration suites <=2, canonical scenario <=1; network-fault and load runs exactly zero. Main has its own separately frozen one-build/one-unit/one-integration/one-scenario verification budget. No test sleep, microbenchmark, fuzzing, replay, UDP, reconnect, many-room or distributed implementation is included.
 
diff --git a/evidence/G02-observations.json b/evidence/G02-observations.json
new file mode 100644
index 0000000..fd654f5
--- /dev/null
+++ b/evidence/G02-observations.json
@@ -0,0 +1,825 @@
+{
+  "thread": "G02",
+  "profile": "realtime-core",
+  "spec_revision": "5a6e4a2f8fc71d4be18c3279583bfc2558d5c232",
+  "attempt": "initial",
+  "start": "005fea800e2543dccaf4ef84c02dadde725fbafa",
+  "commands": [
+    {
+      "label": "reproduce-unit",
+      "category": "unit",
+      "command": [
+        "./track",
+        "unit-test",
+        "--tests",
+        "arena.CompleteFrameTest.frozenG02FramingMatrix"
+      ],
+      "exit": 1,
+      "duration_seconds": 5.446,
+      "evidence": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g02-initial/reproduce-unit"
+    },
+    {
+      "label": "build-1",
+      "category": "build",
+      "command": [
+        "./track",
+        "build"
+      ],
+      "exit": 0,
+      "duration_seconds": 5.721,
+      "evidence": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g02-initial/build-1"
+    },
+    {
+      "label": "verify-unit-1",
+      "category": "unit",
+      "command": [
+        "./track",
+        "unit-test"
+      ],
+      "exit": 0,
+      "duration_seconds": 4.229,
+      "evidence": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g02-initial/verify-unit-1"
+    },
+    {
+      "label": "verify-integration-1",
+      "category": "integration",
+      "command": [
+        "./track",
+        "integration-test"
+      ],
+      "exit": 0,
+      "duration_seconds": 4.867,
+      "evidence": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g02-initial/verify-integration-1"
+    },
+    {
+      "label": "verify-canonical",
+      "category": "canonical",
+      "command": [
+        "./track",
+        "scenario-run",
+        "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G02.json",
+        "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/G02-result.json"
+      ],
+      "exit": 0,
+      "duration_seconds": 1.365,
+      "evidence": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g02-initial/verify-canonical"
+    }
+  ],
+  "reproduction": {
+    "tests": 10,
+    "failures": 7,
+    "errors": 0,
+    "skipped": 0,
+    "observed_stdout": [
+      "HELLO 1/2/rest bytes=1 messages=0 open=false error=none inbound_refcnt=0",
+      "HELLO header/payload bytes=4 messages=0 open=false error=FRAME_SIZE_INVALID inbound_refcnt=0",
+      "HELLO all-at-once bytes=26 messages=1 open=true error=none inbound_refcnt=0",
+      "CREATE_ROOM 1/2/rest bytes=1 messages=0 open=false error=none inbound_refcnt=0",
+      "CREATE_ROOM header/payload bytes=4 messages=0 open=false error=FRAME_SIZE_INVALID inbound_refcnt=0",
+      "CREATE_ROOM all-at-once bytes=63 messages=1 open=true error=none inbound_refcnt=0",
+      "JOIN_ROOM 1/2/rest bytes=1 messages=0 open=false error=none inbound_refcnt=0",
+      "JOIN_ROOM header/payload bytes=4 messages=0 open=false error=FRAME_SIZE_INVALID inbound_refcnt=0",
+      "JOIN_ROOM all-at-once bytes=86 messages=1 open=true error=none inbound_refcnt=0",
+      "coalesced HELLO CREATE_ROOM bytes=89 messages=0 open=false error=FRAME_SIZE_INVALID inbound_refcnt=0"
+    ]
+  },
+  "verification": {
+    "unit_tests": 33,
+    "integration_tests": 4,
+    "failures": 0,
+    "errors": 0,
+    "skipped": 0,
+    "supplemental_stdout": [
+      "malformed zero-length code=FRAME_SIZE_INVALID connection=CLOSED state={\"outcome\":\"IO_END\",\"end_reason\":\"FRAMING_CLOSE\",\"messages\":0,\"message_errors\":0,\"framing_errors\":1,\"need_more_reads\":0,\"buffer_bytes\":0,\"buffer_high_water\":4,\"capacity_high_water\":4,\"buffer_ref_count\":0}",
+      "malformed oversize-length code=FRAME_SIZE_INVALID connection=CLOSED state={\"outcome\":\"IO_END\",\"end_reason\":\"FRAMING_CLOSE\",\"messages\":0,\"message_errors\":0,\"framing_errors\":1,\"need_more_reads\":0,\"buffer_bytes\":0,\"buffer_high_water\":4,\"capacity_high_water\":4,\"buffer_ref_count\":0}",
+      "protocol duplicate-key code=MESSAGE_INVALID connection=OPEN state={\"outcome\":\"MESSAGE_ERROR\",\"end_reason\":\"NONE\",\"messages\":0,\"message_errors\":1,\"framing_errors\":0,\"need_more_reads\":0,\"buffer_bytes\":0,\"buffer_high_water\":32,\"capacity_high_water\":64,\"buffer_ref_count\":1}",
+      "protocol invalid-utf8 code=MESSAGE_INVALID connection=OPEN state={\"outcome\":\"MESSAGE_ERROR\",\"end_reason\":\"NONE\",\"messages\":0,\"message_errors\":1,\"framing_errors\":0,\"need_more_reads\":0,\"buffer_bytes\":0,\"buffer_high_water\":38,\"capacity_high_water\":64,\"buffer_ref_count\":1}",
+      "max-size frame state={\"outcome\":\"COMPLETE_VALID_MESSAGE\",\"end_reason\":\"NONE\",\"messages\":1,\"message_errors\":0,\"framing_errors\":0,\"need_more_reads\":1,\"buffer_bytes\":0,\"buffer_high_water\":16388,\"capacity_high_water\":16388,\"buffer_ref_count\":1}",
+      "transport error state={\"outcome\":\"IO_END\",\"end_reason\":\"IO_ERROR\",\"messages\":0,\"message_errors\":0,\"framing_errors\":0,\"need_more_reads\":1,\"buffer_bytes\":0,\"buffer_high_water\":1,\"capacity_high_water\":4,\"buffer_ref_count\":0}",
+      "partial EOF prefix=1 state={\"outcome\":\"IO_END\",\"end_reason\":\"PARTIAL_EOF\",\"messages\":0,\"message_errors\":0,\"framing_errors\":0,\"need_more_reads\":1,\"buffer_bytes\":0,\"buffer_high_water\":1,\"capacity_high_water\":4,\"buffer_ref_count\":0}",
+      "partial EOF prefix=6 state={\"outcome\":\"IO_END\",\"end_reason\":\"PARTIAL_EOF\",\"messages\":0,\"message_errors\":0,\"framing_errors\":0,\"need_more_reads\":1,\"buffer_bytes\":0,\"buffer_high_water\":6,\"capacity_high_water\":64,\"buffer_ref_count\":0}",
+      "protocol root-array code=MESSAGE_INVALID connection=OPEN state={\"outcome\":\"MESSAGE_ERROR\",\"end_reason\":\"NONE\",\"messages\":0,\"message_errors\":1,\"framing_errors\":0,\"need_more_reads\":0,\"buffer_bytes\":0,\"buffer_high_water\":6,\"capacity_high_water\":64,\"buffer_ref_count\":1}",
+      "protocol missing-v code=MESSAGE_INVALID connection=OPEN state={\"outcome\":\"MESSAGE_ERROR\",\"end_reason\":\"NONE\",\"messages\":0,\"message_errors\":1,\"framing_errors\":0,\"need_more_reads\":0,\"buffer_bytes\":0,\"buffer_high_water\":20,\"capacity_high_water\":64,\"buffer_ref_count\":1}",
+      "protocol missing-type code=MESSAGE_INVALID connection=OPEN state={\"outcome\":\"MESSAGE_ERROR\",\"end_reason\":\"NONE\",\"messages\":0,\"message_errors\":1,\"framing_errors\":0,\"need_more_reads\":0,\"buffer_bytes\":0,\"buffer_high_water\":11,\"capacity_high_water\":64,\"buffer_ref_count\":1}",
+      "protocol string-v code=MESSAGE_INVALID connection=OPEN state={\"outcome\":\"MESSAGE_ERROR\",\"end_reason\":\"NONE\",\"messages\":0,\"message_errors\":1,\"framing_errors\":0,\"need_more_reads\":0,\"buffer_bytes\":0,\"buffer_high_water\":28,\"capacity_high_water\":64,\"buffer_ref_count\":1}",
+      "protocol floating-v code=MESSAGE_INVALID connection=OPEN state={\"outcome\":\"MESSAGE_ERROR\",\"end_reason\":\"NONE\",\"messages\":0,\"message_errors\":1,\"framing_errors\":0,\"need_more_reads\":0,\"buffer_bytes\":0,\"buffer_high_water\":28,\"capacity_high_water\":64,\"buffer_ref_count\":1}",
+      "protocol unsupported-v code=PROTOCOL_VERSION_UNSUPPORTED connection=OPEN state={\"outcome\":\"MESSAGE_ERROR\",\"end_reason\":\"NONE\",\"messages\":0,\"message_errors\":1,\"framing_errors\":0,\"need_more_reads\":0,\"buffer_bytes\":0,\"buffer_high_water\":26,\"capacity_high_water\":64,\"buffer_ref_count\":1}",
+      "protocol unknown-type code=MESSAGE_TYPE_UNKNOWN connection=OPEN state={\"outcome\":\"MESSAGE_ERROR\",\"end_reason\":\"NONE\",\"messages\":0,\"message_errors\":1,\"framing_errors\":0,\"need_more_reads\":0,\"buffer_bytes\":0,\"buffer_high_water\":33,\"capacity_high_water\":64,\"buffer_ref_count\":1}",
+      "protocol known-unknown-field code=ACCEPTED connection=OPEN state={\"outcome\":\"COMPLETE_VALID_MESSAGE\",\"end_reason\":\"NONE\",\"messages\":1,\"message_errors\":0,\"framing_errors\":0,\"need_more_reads\":0,\"buffer_bytes\":0,\"buffer_high_water\":39,\"capacity_high_water\":64,\"buffer_ref_count\":1}",
+      "protocol nan code=MESSAGE_INVALID connection=OPEN state={\"outcome\":\"MESSAGE_ERROR\",\"end_reason\":\"NONE\",\"messages\":0,\"message_errors\":1,\"framing_errors\":0,\"need_more_reads\":0,\"buffer_bytes\":0,\"buffer_high_water\":38,\"capacity_high_water\":64,\"buffer_ref_count\":1}",
+      "protocol infinity code=MESSAGE_INVALID connection=OPEN state={\"outcome\":\"MESSAGE_ERROR\",\"end_reason\":\"NONE\",\"messages\":0,\"message_errors\":1,\"framing_errors\":0,\"need_more_reads\":0,\"buffer_bytes\":0,\"buffer_high_water\":43,\"capacity_high_water\":64,\"buffer_ref_count\":1}",
+      "protocol trailing-object code=MESSAGE_INVALID connection=OPEN state={\"outcome\":\"MESSAGE_ERROR\",\"end_reason\":\"NONE\",\"messages\":0,\"message_errors\":1,\"framing_errors\":0,\"need_more_reads\":0,\"buffer_bytes\":0,\"buffer_high_water\":28,\"capacity_high_water\":64,\"buffer_ref_count\":1}"
+    ]
+  },
+  "canonical": {
+    "thread": "G02",
+    "scenario_id": "G02-framing",
+    "contract_version": 1,
+    "scenario_sha256": "a1d103416b07e5fdb30d349e1938123851727bcaa33ac99baf72404505464692",
+    "parser_bound_bytes": 16388,
+    "fragmentation_matrix": [
+      {
+        "reads": [
+          {
+            "outcome": "NEED_MORE_BYTES",
+            "end_reason": "NONE",
+            "messages": 0,
+            "message_errors": 0,
+            "framing_errors": 0,
+            "need_more_reads": 1,
+            "buffer_bytes": 1,
+            "buffer_high_water": 1,
+            "capacity_high_water": 4,
+            "buffer_ref_count": 1,
+            "read_bytes": 1,
+            "inbound_ref_count": 0
+          },
+          {
+            "outcome": "NEED_MORE_BYTES",
+            "end_reason": "NONE",
+            "messages": 0,
+            "message_errors": 0,
+            "framing_errors": 0,
+            "need_more_reads": 2,
+            "buffer_bytes": 3,
+            "buffer_high_water": 3,
+            "capacity_high_water": 4,
+            "buffer_ref_count": 1,
+            "read_bytes": 2,
+            "inbound_ref_count": 0
+          },
+          {
+            "outcome": "COMPLETE_VALID_MESSAGE",
+            "end_reason": "NONE",
+            "messages": 1,
+            "message_errors": 0,
+            "framing_errors": 0,
+            "need_more_reads": 2,
+            "buffer_bytes": 0,
+            "buffer_high_water": 26,
+            "capacity_high_water": 64,
+            "buffer_ref_count": 1,
+            "read_bytes": 23,
+            "inbound_ref_count": 0
+          }
+        ],
+        "parser": {
+          "outcome": "COMPLETE_VALID_MESSAGE",
+          "end_reason": "NONE",
+          "messages": 1,
+          "message_errors": 0,
+          "framing_errors": 0,
+          "need_more_reads": 2,
+          "buffer_bytes": 0,
+          "buffer_high_water": 26,
+          "capacity_high_water": 64,
+          "buffer_ref_count": 1
+        },
+        "parsed_types": [
+          "HELLO"
+        ],
+        "after_close": {
+          "outcome": "IO_END",
+          "end_reason": "CLEAN_EOF",
+          "messages": 1,
+          "message_errors": 0,
+          "framing_errors": 0,
+          "need_more_reads": 2,
+          "buffer_bytes": 0,
+          "buffer_high_water": 26,
+          "capacity_high_water": 64,
+          "buffer_ref_count": 0
+        },
+        "owned_buffer_ref_count_after_close": 0,
+        "message_type": "HELLO",
+        "fragmentation": "1/2/rest"
+      },
+      {
+        "reads": [
+          {
+            "outcome": "NEED_MORE_BYTES",
+            "end_reason": "NONE",
+            "messages": 0,
+            "message_errors": 0,
+            "framing_errors": 0,
+            "need_more_reads": 1,
+            "buffer_bytes": 4,
+            "buffer_high_water": 4,
+            "capacity_high_water": 4,
+            "buffer_ref_count": 1,
+            "read_bytes": 4,
+            "inbound_ref_count": 0
+          },
+          {
+            "outcome": "COMPLETE_VALID_MESSAGE",
+            "end_reason": "NONE",
+            "messages": 1,
+            "message_errors": 0,
+            "framing_errors": 0,
+            "need_more_reads": 1,
+            "buffer_bytes": 0,
+            "buffer_high_water": 26,
+            "capacity_high_water": 64,
+            "buffer_ref_count": 1,
+            "read_bytes": 22,
+            "inbound_ref_count": 0
+          }
+        ],
+        "parser": {
+          "outcome": "COMPLETE_VALID_MESSAGE",
+          "end_reason": "NONE",
+          "messages": 1,
+          "message_errors": 0,
+          "framing_errors": 0,
+          "need_more_reads": 1,
+          "buffer_bytes": 0,
+          "buffer_high_water": 26,
+          "capacity_high_water": 64,
+          "buffer_ref_count": 1
+        },
+        "parsed_types": [
+          "HELLO"
+        ],
+        "after_close": {
+          "outcome": "IO_END",
+          "end_reason": "CLEAN_EOF",
+          "messages": 1,
+          "message_errors": 0,
+          "framing_errors": 0,
+          "need_more_reads": 1,
+          "buffer_bytes": 0,
+          "buffer_high_water": 26,
+          "capacity_high_water": 64,
+          "buffer_ref_count": 0
+        },
+        "owned_buffer_ref_count_after_close": 0,
+        "message_type": "HELLO",
+        "fragmentation": "header/payload"
+      },
+      {
+        "reads": [
+          {
+            "outcome": "COMPLETE_VALID_MESSAGE",
+            "end_reason": "NONE",
+            "messages": 1,
+            "message_errors": 0,
+            "framing_errors": 0,
+            "need_more_reads": 0,
+            "buffer_bytes": 0,
+            "buffer_high_water": 26,
+            "capacity_high_water": 64,
+            "buffer_ref_count": 1,
+            "read_bytes": 26,
+            "inbound_ref_count": 0
+          }
+        ],
+        "parser": {
+          "outcome": "COMPLETE_VALID_MESSAGE",
+          "end_reason": "NONE",
+          "messages": 1,
+          "message_errors": 0,
+          "framing_errors": 0,
+          "need_more_reads": 0,
+          "buffer_bytes": 0,
+          "buffer_high_water": 26,
+          "capacity_high_water": 64,
+          "buffer_ref_count": 1
+        },
+        "parsed_types": [
+          "HELLO"
+        ],
+        "after_close": {
+          "outcome": "IO_END",
+          "end_reason": "CLEAN_EOF",
+          "messages": 1,
+          "message_errors": 0,
+          "framing_errors": 0,
+          "need_more_reads": 0,
+          "buffer_bytes": 0,
+          "buffer_high_water": 26,
+          "capacity_high_water": 64,
+          "buffer_ref_count": 0
+        },
+        "owned_buffer_ref_count_after_close": 0,
+        "message_type": "HELLO",
+        "fragmentation": "all-at-once"
+      },
+      {
+        "reads": [
+          {
+            "outcome": "NEED_MORE_BYTES",
+            "end_reason": "NONE",
+            "messages": 0,
+            "message_errors": 0,
+            "framing_errors": 0,
+            "need_more_reads": 1,
+            "buffer_bytes": 1,
+            "buffer_high_water": 1,
+            "capacity_high_water": 4,
+            "buffer_ref_count": 1,
+            "read_bytes": 1,
+            "inbound_ref_count": 0
+          },
+          {
+            "outcome": "NEED_MORE_BYTES",
+            "end_reason": "NONE",
+            "messages": 0,
+            "message_errors": 0,
+            "framing_errors": 0,
+            "need_more_reads": 2,
+            "buffer_bytes": 3,
+            "buffer_high_water": 3,
+            "capacity_high_water": 4,
+            "buffer_ref_count": 1,
+            "read_bytes": 2,
+            "inbound_ref_count": 0
+          },
+          {
+            "outcome": "COMPLETE_VALID_MESSAGE",
+            "end_reason": "NONE",
+            "messages": 1,
+            "message_errors": 0,
+            "framing_errors": 0,
+            "need_more_reads": 2,
+            "buffer_bytes": 0,
+            "buffer_high_water": 63,
+            "capacity_high_water": 64,
+            "buffer_ref_count": 1,
+            "read_bytes": 60,
+            "inbound_ref_count": 0
+          }
+        ],
+        "parser": {
+          "outcome": "COMPLETE_VALID_MESSAGE",
+          "end_reason": "NONE",
+          "messages": 1,
+          "message_errors": 0,
+          "framing_errors": 0,
+          "need_more_reads": 2,
+          "buffer_bytes": 0,
+          "buffer_high_water": 63,
+          "capacity_high_water": 64,
+          "buffer_ref_count": 1
+        },
+        "parsed_types": [
+          "CREATE_ROOM"
+        ],
+        "after_close": {
+          "outcome": "IO_END",
+          "end_reason": "CLEAN_EOF",
+          "messages": 1,
+          "message_errors": 0,
+          "framing_errors": 0,
+          "need_more_reads": 2,
+          "buffer_bytes": 0,
+          "buffer_high_water": 63,
+          "capacity_high_water": 64,
+          "buffer_ref_count": 0
+        },
+        "owned_buffer_ref_count_after_close": 0,
+        "message_type": "CREATE_ROOM",
+        "fragmentation": "1/2/rest"
+      },
+      {
+        "reads": [
+          {
+            "outcome": "NEED_MORE_BYTES",
+            "end_reason": "NONE",
+            "messages": 0,
+            "message_errors": 0,
+            "framing_errors": 0,
+            "need_more_reads": 1,
+            "buffer_bytes": 4,
+            "buffer_high_water": 4,
+            "capacity_high_water": 4,
+            "buffer_ref_count": 1,
+            "read_bytes": 4,
+            "inbound_ref_count": 0
+          },
+          {
+            "outcome": "COMPLETE_VALID_MESSAGE",
+            "end_reason": "NONE",
+            "messages": 1,
+            "message_errors": 0,
+            "framing_errors": 0,
+            "need_more_reads": 1,
+            "buffer_bytes": 0,
+            "buffer_high_water": 63,
+            "capacity_high_water": 64,
+            "buffer_ref_count": 1,
+            "read_bytes": 59,
+            "inbound_ref_count": 0
+          }
+        ],
+        "parser": {
+          "outcome": "COMPLETE_VALID_MESSAGE",
+          "end_reason": "NONE",
+          "messages": 1,
+          "message_errors": 0,
+          "framing_errors": 0,
+          "need_more_reads": 1,
+          "buffer_bytes": 0,
+          "buffer_high_water": 63,
+          "capacity_high_water": 64,
+          "buffer_ref_count": 1
+        },
+        "parsed_types": [
+          "CREATE_ROOM"
+        ],
+        "after_close": {
+          "outcome": "IO_END",
+          "end_reason": "CLEAN_EOF",
+          "messages": 1,
+          "message_errors": 0,
+          "framing_errors": 0,
+          "need_more_reads": 1,
+          "buffer_bytes": 0,
+          "buffer_high_water": 63,
+          "capacity_high_water": 64,
+          "buffer_ref_count": 0
+        },
+        "owned_buffer_ref_count_after_close": 0,
+        "message_type": "CREATE_ROOM",
+        "fragmentation": "header/payload"
+      },
+      {
+        "reads": [
+          {
+            "outcome": "COMPLETE_VALID_MESSAGE",
+            "end_reason": "NONE",
+            "messages": 1,
+            "message_errors": 0,
+            "framing_errors": 0,
+            "need_more_reads": 0,
+            "buffer_bytes": 0,
+            "buffer_high_water": 63,
+            "capacity_high_water": 64,
+            "buffer_ref_count": 1,
+            "read_bytes": 63,
+            "inbound_ref_count": 0
+          }
+        ],
+        "parser": {
+          "outcome": "COMPLETE_VALID_MESSAGE",
+          "end_reason": "NONE",
+          "messages": 1,
+          "message_errors": 0,
+          "framing_errors": 0,
+          "need_more_reads": 0,
+          "buffer_bytes": 0,
+          "buffer_high_water": 63,
+          "capacity_high_water": 64,
+          "buffer_ref_count": 1
+        },
+        "parsed_types": [
+          "CREATE_ROOM"
+        ],
+        "after_close": {
+          "outcome": "IO_END",
+          "end_reason": "CLEAN_EOF",
+          "messages": 1,
+          "message_errors": 0,
+          "framing_errors": 0,
+          "need_more_reads": 0,
+          "buffer_bytes": 0,
+          "buffer_high_water": 63,
+          "capacity_high_water": 64,
+          "buffer_ref_count": 0
+        },
+        "owned_buffer_ref_count_after_close": 0,
+        "message_type": "CREATE_ROOM",
+        "fragmentation": "all-at-once"
+      },
+      {
+        "reads": [
+          {
+            "outcome": "NEED_MORE_BYTES",
+            "end_reason": "NONE",
+            "messages": 0,
+            "message_errors": 0,
+            "framing_errors": 0,
+            "need_more_reads": 1,
+            "buffer_bytes": 1,
+            "buffer_high_water": 1,
+            "capacity_high_water": 4,
+            "buffer_ref_count": 1,
+            "read_bytes": 1,
+            "inbound_ref_count": 0
+          },
+          {
+            "outcome": "NEED_MORE_BYTES",
+            "end_reason": "NONE",
+            "messages": 0,
+            "message_errors": 0,
+            "framing_errors": 0,
+            "need_more_reads": 2,
+            "buffer_bytes": 3,
+            "buffer_high_water": 3,
+            "capacity_high_water": 4,
+            "buffer_ref_count": 1,
+            "read_bytes": 2,
+            "inbound_ref_count": 0
+          },
+          {
+            "outcome": "COMPLETE_VALID_MESSAGE",
+            "end_reason": "NONE",
+            "messages": 1,
+            "message_errors": 0,
+            "framing_errors": 0,
+            "need_more_reads": 2,
+            "buffer_bytes": 0,
+            "buffer_high_water": 86,
+            "capacity_high_water": 128,
+            "buffer_ref_count": 1,
+            "read_bytes": 83,
+            "inbound_ref_count": 0
+          }
+        ],
+        "parser": {
+          "outcome": "COMPLETE_VALID_MESSAGE",
+          "end_reason": "NONE",
+          "messages": 1,
+          "message_errors": 0,
+          "framing_errors": 0,
+          "need_more_reads": 2,
+          "buffer_bytes": 0,
+          "buffer_high_water": 86,
+          "capacity_high_water": 128,
+          "buffer_ref_count": 1
+        },
+        "parsed_types": [
+          "JOIN_ROOM"
+        ],
+        "after_close": {
+          "outcome": "IO_END",
+          "end_reason": "CLEAN_EOF",
+          "messages": 1,
+          "message_errors": 0,
+          "framing_errors": 0,
+          "need_more_reads": 2,
+          "buffer_bytes": 0,
+          "buffer_high_water": 86,
+          "capacity_high_water": 128,
+          "buffer_ref_count": 0
+        },
+        "owned_buffer_ref_count_after_close": 0,
+        "message_type": "JOIN_ROOM",
+        "fragmentation": "1/2/rest"
+      },
+      {
+        "reads": [
+          {
+            "outcome": "NEED_MORE_BYTES",
+            "end_reason": "NONE",
+            "messages": 0,
+            "message_errors": 0,
+            "framing_errors": 0,
+            "need_more_reads": 1,
+            "buffer_bytes": 4,
+            "buffer_high_water": 4,
+            "capacity_high_water": 4,
+            "buffer_ref_count": 1,
+            "read_bytes": 4,
+            "inbound_ref_count": 0
+          },
+          {
+            "outcome": "COMPLETE_VALID_MESSAGE",
+            "end_reason": "NONE",
+            "messages": 1,
+            "message_errors": 0,
+            "framing_errors": 0,
+            "need_more_reads": 1,
+            "buffer_bytes": 0,
+            "buffer_high_water": 86,
+            "capacity_high_water": 128,
+            "buffer_ref_count": 1,
+            "read_bytes": 82,
+            "inbound_ref_count": 0
+          }
+        ],
+        "parser": {
+          "outcome": "COMPLETE_VALID_MESSAGE",
+          "end_reason": "NONE",
+          "messages": 1,
+          "message_errors": 0,
+          "framing_errors": 0,
+          "need_more_reads": 1,
+          "buffer_bytes": 0,
+          "buffer_high_water": 86,
+          "capacity_high_water": 128,
+          "buffer_ref_count": 1
+        },
+        "parsed_types": [
+          "JOIN_ROOM"
+        ],
+        "after_close": {
+          "outcome": "IO_END",
+          "end_reason": "CLEAN_EOF",
+          "messages": 1,
+          "message_errors": 0,
+          "framing_errors": 0,
+          "need_more_reads": 1,
+          "buffer_bytes": 0,
+          "buffer_high_water": 86,
+          "capacity_high_water": 128,
+          "buffer_ref_count": 0
+        },
+        "owned_buffer_ref_count_after_close": 0,
+        "message_type": "JOIN_ROOM",
+        "fragmentation": "header/payload"
+      },
+      {
+        "reads": [
+          {
+            "outcome": "COMPLETE_VALID_MESSAGE",
+            "end_reason": "NONE",
+            "messages": 1,
+            "message_errors": 0,
+            "framing_errors": 0,
+            "need_more_reads": 0,
+            "buffer_bytes": 0,
+            "buffer_high_water": 86,
+            "capacity_high_water": 128,
+            "buffer_ref_count": 1,
+            "read_bytes": 86,
+            "inbound_ref_count": 0
+          }
+        ],
+        "parser": {
+          "outcome": "COMPLETE_VALID_MESSAGE",
+          "end_reason": "NONE",
+          "messages": 1,
+          "message_errors": 0,
+          "framing_errors": 0,
+          "need_more_reads": 0,
+          "buffer_bytes": 0,
+          "buffer_high_water": 86,
+          "capacity_high_water": 128,
+          "buffer_ref_count": 1
+        },
+        "parsed_types": [
+          "JOIN_ROOM"
+        ],
+        "after_close": {
+          "outcome": "IO_END",
+          "end_reason": "CLEAN_EOF",
+          "messages": 1,
+          "message_errors": 0,
+          "framing_errors": 0,
+          "need_more_reads": 0,
+          "buffer_bytes": 0,
+          "buffer_high_water": 86,
+          "capacity_high_water": 128,
+          "buffer_ref_count": 0
+        },
+        "owned_buffer_ref_count_after_close": 0,
+        "message_type": "JOIN_ROOM",
+        "fragmentation": "all-at-once"
+      }
+    ],
+    "coalescing": {
+      "reads": [
+        {
+          "outcome": "COMPLETE_VALID_MESSAGE",
+          "end_reason": "NONE",
+          "messages": 2,
+          "message_errors": 0,
+          "framing_errors": 0,
+          "need_more_reads": 0,
+          "buffer_bytes": 0,
+          "buffer_high_water": 63,
+          "capacity_high_water": 64,
+          "buffer_ref_count": 1,
+          "read_bytes": 89,
+          "inbound_ref_count": 0
+        }
+      ],
+      "parser": {
+        "outcome": "COMPLETE_VALID_MESSAGE",
+        "end_reason": "NONE",
+        "messages": 2,
+        "message_errors": 0,
+        "framing_errors": 0,
+        "need_more_reads": 0,
+        "buffer_bytes": 0,
+        "buffer_high_water": 63,
+        "capacity_high_water": 64,
+        "buffer_ref_count": 1
+      },
+      "parsed_types": [
+        "HELLO",
+        "CREATE_ROOM"
+      ],
+      "after_close": {
+        "outcome": "IO_END",
+        "end_reason": "CLEAN_EOF",
+        "messages": 2,
+        "message_errors": 0,
+        "framing_errors": 0,
+        "need_more_reads": 0,
+        "buffer_bytes": 0,
+        "buffer_high_water": 63,
+        "capacity_high_water": 64,
+        "buffer_ref_count": 0
+      },
+      "owned_buffer_ref_count_after_close": 0
+    },
+    "socket_isolation": {
+      "cases": [
+        {
+          "name": "zero-length",
+          "error_code": "FRAME_SIZE_INVALID",
+          "connection_effect": "CLOSED",
+          "terminal": true,
+          "healthy_connection_hello": "WELCOME",
+          "elapsed_ms": 1,
+          "client_socket_closed": true
+        },
+        {
+          "name": "oversize-length",
+          "error_code": "FRAME_SIZE_INVALID",
+          "connection_effect": "CLOSED",
+          "terminal": true,
+          "healthy_connection_hello": "WELCOME",
+          "elapsed_ms": 0,
+          "client_socket_closed": true
+        },
+        {
+          "name": "duplicate-key",
+          "error_code": "MESSAGE_INVALID",
+          "connection_effect": "OPEN_RECOVERABLE",
+          "terminal": false,
+          "same_connection_hello": "WELCOME",
+          "healthy_connection_hello": "WELCOME",
+          "elapsed_ms": 1,
+          "client_socket_closed": true
+        },
+        {
+          "name": "invalid-utf8",
+          "error_code": "MESSAGE_INVALID",
+          "connection_effect": "OPEN_RECOVERABLE",
+          "terminal": false,
+          "same_connection_hello": "WELCOME",
+          "healthy_connection_hello": "WELCOME",
+          "elapsed_ms": 1,
+          "client_socket_closed": true
+        }
+      ],
+      "runtime_metrics": {
+        "manual_time_ns": 0,
+        "pending_input_high_water": 0,
+        "mailbox_high_water": 1,
+        "outbound_high_water": 1,
+        "parser": {
+          "live_buffers": 1,
+          "allocated_bytes": 64,
+          "buffer_high_water": 38,
+          "capacity_high_water": 64,
+          "message_errors": 2,
+          "framing_errors": 2,
+          "partial_eofs": 0,
+          "io_errors": 0
+        }
+      },
+      "cleanup": {
+        "open_channels": 0,
+        "connections": 0,
+        "pending_writes": 0,
+        "mailbox_remaining": 0,
+        "live_threads": 0,
+        "timer_alive": false,
+        "owner_terminated": true,
+        "event_loops_terminated": true,
+        "pending_input_high_water": 0,
+        "mailbox_high_water": 1,
+        "outbound_high_water": 1,
+        "parser": {
+          "live_buffers": 0,
+          "allocated_bytes": 0,
+          "buffer_high_water": 38,
+          "capacity_high_water": 64,
+          "message_errors": 2,
+          "framing_errors": 2,
+          "partial_eofs": 0,
+          "io_errors": 0
+        },
+        "room_lifecycle": []
+      },
+      "all_client_sockets_closed": true
+    },
+    "state_hashes": "NOT_ACTIVATED_G07",
+    "network_fault_runs": 0,
+    "load_runs": 0
+  },
+  "budget_used": {
+    "compile_bearing_commands": 2,
+    "compile_tasks": 3,
+    "unit_including_reproduction": 2,
+    "integration": 1,
+    "prechange_canonical": 0,
+    "postchange_canonical": 1,
+    "network_fault": 0,
+    "load": 0
+  },
+  "unresolved": []
+}
diff --git a/evidence/G02-verification.md b/evidence/G02-verification.md
new file mode 100644
index 0000000..5d05ede
--- /dev/null
+++ b/evidence/G02-verification.md
@@ -0,0 +1,47 @@
+# G02 verification — initial attempt
+
+Profile: `realtime-core`. Spec revision: `5a6e4a2f8fc71d4be18c3279583bfc2558d5c232`.
+Start: `005fea800e2543dccaf4ef84c02dadde725fbafa` (`progress/industry-java/G01`).
+Frozen input: main `index/scenarios/G02.json`, SHA-256 `a1d103416b07e5fdb30d349e1938123851727bcaa33ac99baf72404505464692`.
+The checked-in test resource has identical bytes. No scenario, threshold, dependency version or G01 assertion changed.
+
+## Reproduce before production edits
+
+`./track unit-test --tests arena.CompleteFrameTest.frozenG02FramingMatrix` ran the existing G01 handler with exact fixture payloads and read splits. Exit **1**, 5.446 seconds; 10 tests, 7 failures, no errors/skips. Only the three all-at-once cases passed. For each message, a one-byte header prefix closed immediately with no message; a four-byte header closed with FRAME_SIZE_INVALID. The coalesced HELLO/CREATE_ROOM read also returned FRAME_SIZE_INVALID and closed, with zero messages. These were observed failures, not a deliberately damaged implementation or an unsupported scenario command.
+
+Full stdout and XML were copied to `evidence/runs/g02-initial/reproduce-unit/` before the clean build. Exact observation lines are preserved in `G02-observations.json`.
+
+## Fix
+
+`CompleteFrame` retains one bounded frame, preserves partial header/payload state, consumes coalesced frames in order and stops reading after a terminal length error. Raw cumulation starts at four bytes and cannot exceed 16,388. Strict UTF-8/JSON and the already-active message schemas are checked before Room handoff. Parser errors follow the existing bounded outbound path. EOF, partial EOF and I/O failure have distinct state evidence; all buffer ownership is released explicitly.
+
+## Actual verification commands
+
+All commands ran from the Java worktree using pinned Java 21.0.7, offline Gradle 8.10.2 and unchanged locks. Narrow escalated execution was required for Gradle native initialization and loopback sockets. Test JVMs used existing Netty PARANOID leak detection.
+
+| Command | Exit | Duration | Preserved output |
+|---|---:|---:|---|
+| `./track unit-test --tests arena.CompleteFrameTest.frozenG02FramingMatrix` | 1 (expected reproduction) | 5.446s | `evidence/runs/g02-initial/reproduce-unit/` |
+| `./track build` | 0 | 5.721s | `evidence/runs/g02-initial/build-1/` |
+| `./track unit-test` | 0 | 4.229s | `evidence/runs/g02-initial/verify-unit-1/` |
+| `./track integration-test` | 0 | 4.867s | `evidence/runs/g02-initial/verify-integration-1/` |
+| `./track scenario-run <absolute-main-G02.json> <absolute-Java-evidence/G02-result.json>` | 0 | 1.365s | `evidence/runs/g02-initial/verify-canonical/`, `evidence/G02-result.json` |
+
+The exact absolute canonical arguments and every invocation's exit/duration are recorded in `evidence/runs/g02-initial/commands.jsonl` and copied into `G02-observations.json`. There were no retry runs. Build/compile consumption was two compile-bearing commands (three compile tasks), below eight; unit consumption 2/4 includes reproduction, integration 1/2, post-change canonical 1/1. Pre-change canonical was not run. Network-fault and load runs: **0**.
+
+## Results and bounds
+
+All 33 unit tests and four integration tests passed without skips. All nine frozen fragmentation cases parsed their one expected message; coalescing preserved HELLO then CREATE_ROOM. The exact matrix uses the production handler through EmbeddedChannel, not a replacement parser. Malformed isolation uses real TCP and confirms a separate healthy connection replies after each case.
+
+| Frozen socket case | Error | Connection effect | Healthy peer |
+|---|---|---|---|
+| zero-length | FRAME_SIZE_INVALID | CLOSED | WELCOME |
+| oversize-length | FRAME_SIZE_INVALID | CLOSED | WELCOME |
+| duplicate-key | MESSAGE_INVALID | OPEN_RECOVERABLE; next HELLO succeeds | WELCOME |
+| invalid-utf8 | MESSAGE_INVALID | OPEN_RECOVERABLE; next HELLO succeeds | WELCOME |
+
+All four canonical socket cases completed in 0–1ms, below the unchanged 5,000ms ceiling. The unit maximum-size frame reached **16,388 buffered bytes and 16,388 allocated capacity**, then its actual ByteBuf reference count became zero. Invalid length allocated only the four-byte header. Canonical malformed peers reached 38 buffered bytes/64 allocated capacity. Final server cleanup showed zero channels, connections, pending writes, mailbox entries, live threads, parser buffers and parser allocated bytes; both event-loop groups and the owner terminated; no timer or client sockets remained. Existing real CLI SIGTERM/listener-rebind and owner-confinement tests still passed.
+
+The fixed supplemental JSON checks rejected root arrays, missing v/type, string/floating v, NaN, Infinity and trailing objects as MESSAGE_INVALID; v=2 produced PROTOCOL_VERSION_UNSUPPORTED; unknown type produced MESSAGE_TYPE_UNKNOWN; an unknown HELLO field was accepted. Every recoverable error allowed the next complete message. Partial-header and partial-payload EOF produced IO_END/PARTIAL_EOF and zero buffer references; the fixed transport exception produced IO_END/IO_ERROR, without framing/message errors.
+
+State hashes remain `NOT_ACTIVATED_G07`. No sequence/target-tick, replay, UDP, reconnect, multi-room or G03 identity matrix was added. Unresolved G02 items: **none**.
diff --git a/src/main/java/arena/ArenaServer.java b/src/main/java/arena/ArenaServer.java
index b354770..bf51b78 100644
--- a/src/main/java/arena/ArenaServer.java
+++ b/src/main/java/arena/ArenaServer.java
@@ -49,6 +49,7 @@ public final class ArenaServer implements AutoCloseable {
     private final Set<Thread> ownedThreads = ConcurrentHashMap.newKeySet();
     private final AtomicInteger connections = new AtomicInteger();
     private final AtomicInteger pendingWrites = new AtomicInteger();
+    private final CompleteFrame.Metrics parserMetrics = new CompleteFrame.Metrics();
     private final AtomicInteger outboundHighWater = new AtomicInteger();
     private final AtomicInteger mailboxHighWater = new AtomicInteger();
     private final AtomicBoolean closing = new AtomicBoolean();
@@ -147,7 +148,8 @@ public final class ArenaServer implements AutoCloseable {
                                     context.fireChannelInactive();
                                 }
                             });
-                            channel.pipeline().addLast(new CompleteFrame(message -> enqueue(peer, () -> handle(peer, message))));
+                            channel.pipeline().addLast(new CompleteFrame(message -> enqueue(peer, () -> handle(peer, message)),
+                                    (error, terminal) -> enqueue(peer, () -> peer.send(error, terminal)), parserMetrics));
                         }
                     }).bind(host, port).syncUninterruptibly().channel();
         } catch (RuntimeException failure) {
@@ -305,7 +307,8 @@ public final class ArenaServer implements AutoCloseable {
         if (closing.get()) return cleanup();
         return call(() -> Json.MAPPER.createObjectNode().put("manual_time_ns", manualNanos)
                 .put("pending_input_high_water", room == null ? 0 : room.inputHighWater())
-                .put("mailbox_high_water", mailboxHighWater.get()).put("outbound_high_water", outboundHighWater.get()));
+                .put("mailbox_high_water", mailboxHighWater.get()).put("outbound_high_water", outboundHighWater.get())
+                .set("parser", parserMetrics.view()));
     }
 
     public ObjectNode cleanup() {
@@ -317,6 +320,7 @@ public final class ArenaServer implements AutoCloseable {
                 .put("event_loops_terminated", acceptLoop.isTerminated() && ioLoop.isTerminated())
                 .put("pending_input_high_water", closedInputHighWater)
                 .put("mailbox_high_water", mailboxHighWater.get()).put("outbound_high_water", outboundHighWater.get());
+        result.set("parser", parserMetrics.view());
         var lifecycle = result.putArray("room_lifecycle");
         closedLifecycle.forEach(lifecycle::add);
         return result;
diff --git a/src/main/java/arena/CompleteFrame.java b/src/main/java/arena/CompleteFrame.java
index c6ce052..f7fec31 100644
--- a/src/main/java/arena/CompleteFrame.java
+++ b/src/main/java/arena/CompleteFrame.java
@@ -1,41 +1,217 @@
 package arena;
 
+import com.fasterxml.jackson.databind.JsonNode;
 import com.fasterxml.jackson.databind.node.ObjectNode;
 import io.netty.buffer.ByteBuf;
 import io.netty.buffer.Unpooled;
 import io.netty.channel.ChannelHandlerContext;
 import io.netty.channel.SimpleChannelInboundHandler;
 import java.io.IOException;
+import java.math.BigInteger;
+import java.util.concurrent.atomic.AtomicInteger;
+import java.util.function.BiConsumer;
 import java.util.function.Consumer;
 
-/** G01 deliberately assumes one complete frame per read; no cumulation or incremental parser. */
+/** Incrementally assembles complete TCP frames; every instance belongs to one channel. */
 final class CompleteFrame extends SimpleChannelInboundHandler<ByteBuf> {
     static final int MAX_BYTES = 16_384;
+    static final int BUFFER_BOUND = MAX_BYTES + 4;
+    enum Outcome { NEED_MORE_BYTES, COMPLETE_VALID_MESSAGE, MESSAGE_ERROR, TERMINAL_FRAME_ERROR, IO_END }
+    enum EndReason { NONE, CLEAN_EOF, PARTIAL_EOF, FRAMING_CLOSE, IO_ERROR }
+
+    /** Shared counters only; no mutable parser state crosses channel owners. */
+    static final class Metrics {
+        private final AtomicInteger liveBuffers = new AtomicInteger();
+        private final AtomicInteger allocatedBytes = new AtomicInteger();
+        private final AtomicInteger bufferHighWater = new AtomicInteger();
+        private final AtomicInteger capacityHighWater = new AtomicInteger();
+        private final AtomicInteger messageErrors = new AtomicInteger();
+        private final AtomicInteger framingErrors = new AtomicInteger();
+        private final AtomicInteger partialEofs = new AtomicInteger();
+        private final AtomicInteger ioErrors = new AtomicInteger();
+
+        ObjectNode view() {
+            return Json.MAPPER.createObjectNode().put("live_buffers", liveBuffers.get())
+                    .put("allocated_bytes", allocatedBytes.get()).put("buffer_high_water", bufferHighWater.get())
+                    .put("capacity_high_water", capacityHighWater.get()).put("message_errors", messageErrors.get())
+                    .put("framing_errors", framingErrors.get()).put("partial_eofs", partialEofs.get())
+                    .put("io_errors", ioErrors.get());
+        }
+    }
+
     private final Consumer<ObjectNode> receiver;
+    private final BiConsumer<ObjectNode, Boolean> responder;
+    private final Metrics metrics;
+    private ByteBuf accumulated;
+    private int payloadLength = -1;
+    private int bufferHighWater;
+    private int capacityHighWater;
+    private int messages;
+    private int messageErrors;
+    private int framingErrors;
+    private int needMoreReads;
+    private boolean terminal;
+    private Outcome outcome = Outcome.NEED_MORE_BYTES;
+    private EndReason endReason = EndReason.NONE;
 
-    // One instance per channel, not @Sharable. Inbound ByteBuf is always auto-released.
-    CompleteFrame(Consumer<ObjectNode> receiver) { this.receiver = receiver; }
+    // Not @Sharable. SimpleChannelInboundHandler releases every inbound read, including errors.
+    CompleteFrame(Consumer<ObjectNode> receiver) { this(receiver, null, new Metrics()); }
+    CompleteFrame(Consumer<ObjectNode> receiver, BiConsumer<ObjectNode, Boolean> responder, Metrics metrics) {
+        this.receiver = receiver;
+        this.responder = responder;
+        this.metrics = metrics;
+    }
 
-    @Override protected void channelRead0(ChannelHandlerContext context, ByteBuf buffer) {
-        if (buffer.readableBytes() < 4) {
-            context.close();
-            return;
+    @Override protected void channelRead0(ChannelHandlerContext context, ByteBuf source) {
+        while (source.isReadable() && !terminal) {
+            if (accumulated == null) {
+                accumulated = Unpooled.buffer(4, BUFFER_BOUND);
+                metrics.liveBuffers.incrementAndGet();
+                metrics.allocatedBytes.addAndGet(accumulated.capacity());
+                recordBuffer();
+            }
+            if (payloadLength < 0) {
+                copy(source, 4 - accumulated.writerIndex());
+                if (accumulated.writerIndex() < 4) { needMore(); return; }
+                long length = accumulated.getUnsignedInt(0);
+                if (length == 0 || length > MAX_BYTES) {
+                    terminal = true;
+                    outcome = Outcome.TERMINAL_FRAME_ERROR;
+                    endReason = EndReason.FRAMING_CLOSE;
+                    framingErrors++;
+                    metrics.framingErrors.incrementAndGet();
+                    context.channel().config().setAutoRead(false);
+                    releaseBuffer();
+                    reject(context, "FRAME_SIZE_INVALID", true);
+                    return;
+                }
+                payloadLength = (int) length;
+            }
+            copy(source, 4 + payloadLength - accumulated.writerIndex());
+            if (accumulated.writerIndex() < 4 + payloadLength) { needMore(); return; }
+            ObjectNode message = null;
+            String code;
+            try {
+                // A borrowed NIO view avoids a second retained raw-payload allocation.
+                message = Json.read(accumulated.nioBuffer(4, payloadLength));
+                code = protocolError(message);
+            } catch (IOException invalid) { code = "MESSAGE_INVALID"; }
+            accumulated.clear();
+            payloadLength = -1;
+            if (code == null) {
+                outcome = Outcome.COMPLETE_VALID_MESSAGE;
+                messages++;
+                receiver.accept(message);
+            } else {
+                outcome = Outcome.MESSAGE_ERROR;
+                messageErrors++;
+                metrics.messageErrors.incrementAndGet();
+                reject(context, code, false);
+            }
         }
-        long length = buffer.readUnsignedInt();
-        if (length == 0 || length > MAX_BYTES || length != buffer.readableBytes()) {
-            context.writeAndFlush(encode(error("FRAME_SIZE_INVALID", "G01 requires one complete frame")))
-                    .addListener(future -> context.close());
-            return;
+    }
+
+    private void copy(ByteBuf source, int needed) {
+        int count = Math.min(needed, source.readableBytes());
+        int oldCapacity = accumulated.capacity();
+        accumulated.writeBytes(source, count);
+        metrics.allocatedBytes.addAndGet(accumulated.capacity() - oldCapacity);
+        recordBuffer();
+    }
+
+    private void recordBuffer() {
+        bufferHighWater = Math.max(bufferHighWater, accumulated.writerIndex());
+        capacityHighWater = Math.max(capacityHighWater, accumulated.capacity());
+        metrics.bufferHighWater.accumulateAndGet(bufferHighWater, Math::max);
+        metrics.capacityHighWater.accumulateAndGet(capacityHighWater, Math::max);
+    }
+
+    private void needMore() { outcome = Outcome.NEED_MORE_BYTES; needMoreReads++; }
+
+    private void reject(ChannelHandlerContext context, String code, boolean close) {
+        ObjectNode reply = error(code, code);
+        if (responder != null) responder.accept(reply, close);
+        else context.writeAndFlush(encode(reply)).addListener(result -> {
+            if (close || !result.isSuccess()) context.close();
+        });
+    }
+
+    @Override public void channelInactive(ChannelHandlerContext context) {
+        if (endReason == EndReason.NONE) {
+            endReason = bufferedBytes() == 0 ? EndReason.CLEAN_EOF : EndReason.PARTIAL_EOF;
+            if (endReason == EndReason.PARTIAL_EOF) metrics.partialEofs.incrementAndGet();
         }
-        byte[] bytes = new byte[(int) length];
-        buffer.readBytes(bytes);
-        try { receiver.accept(Json.read(bytes)); }
-        catch (IOException | IllegalArgumentException invalid) {
-            context.writeAndFlush(encode(error("MESSAGE_INVALID", "JSON object required")));
+        terminal = true;
+        outcome = Outcome.IO_END;
+        releaseBuffer();
+        context.fireChannelInactive();
+    }
+
+    @Override public void exceptionCaught(ChannelHandlerContext context, Throwable cause) {
+        if (endReason == EndReason.NONE) {
+            endReason = EndReason.IO_ERROR;
+            metrics.ioErrors.incrementAndGet();
         }
+        terminal = true;
+        outcome = Outcome.IO_END;
+        releaseBuffer();
+        context.close();
     }
 
-    @Override public void exceptionCaught(ChannelHandlerContext context, Throwable cause) { context.close(); }
+    @Override public void handlerRemoved(ChannelHandlerContext context) { releaseBuffer(); }
+
+    private void releaseBuffer() {
+        if (accumulated == null) return;
+        int capacity = accumulated.capacity();
+        // The parser never retains or shares its cumulation, so this is the last reference.
+        boolean freed = accumulated.release();
+        accumulated = null;
+        payloadLength = -1;
+        metrics.liveBuffers.decrementAndGet();
+        metrics.allocatedBytes.addAndGet(-capacity);
+        if (!freed) throw new IllegalStateException("parser buffer has an unexpected owner");
+    }
+
+    int bufferedBytes() { return accumulated == null ? 0 : accumulated.writerIndex(); }
+    // Borrowed only, for reference-count verification; ownership remains with this handler.
+    ByteBuf ownedBuffer() { return accumulated; }
+    ObjectNode state() {
+        return Json.MAPPER.createObjectNode().put("outcome", outcome.name()).put("end_reason", endReason.name())
+                .put("messages", messages).put("message_errors", messageErrors).put("framing_errors", framingErrors)
+                .put("need_more_reads", needMoreReads).put("buffer_bytes", bufferedBytes())
+                .put("buffer_high_water", bufferHighWater).put("capacity_high_water", capacityHighWater)
+                .put("buffer_ref_count", accumulated == null ? 0 : accumulated.refCnt());
+    }
+
+    private static String protocolError(ObjectNode message) {
+        JsonNode version = message.get("v");
+        JsonNode typeNode = message.get("type");
+        if (version == null || !version.isIntegralNumber() || typeNode == null || !typeNode.isTextual())
+            return "MESSAGE_INVALID";
+        if (!version.bigIntegerValue().equals(BigInteger.ONE)) return "PROTOCOL_VERSION_UNSUPPORTED";
+        String type = typeNode.textValue();
+        try {
+            // Only schemas already active at G02. Sequence, target tick and future messages are not implemented.
+            switch (type) {
+                case "HELLO" -> { }
+                case "CREATE_ROOM" -> Json.text(message, "session_id");
+                case "JOIN_ROOM", "LEAVE_ROOM" -> {
+                    Json.text(message, "session_id");
+                    Json.text(message, "room_id");
+                }
+                case "INPUT" -> {
+                    Json.text(message, "session_id");
+                    Json.text(message, "room_id");
+                    Json.text(message, "player_id");
+                    Json.text(message, "direction");
+                    JsonNode target = message.get("tag_target_player_id");
+                    if (target == null || !(target.isNull() || target.isTextual())) return "MESSAGE_INVALID";
+                }
+                default -> { return "MESSAGE_TYPE_UNKNOWN"; }
+            }
+        } catch (IllegalArgumentException invalid) { return "MESSAGE_INVALID"; }
+        return null;
+    }
 
     static ObjectNode error(String code, String message) {
         return Json.message("ERROR").put("code", code).put("message", message);
diff --git a/src/main/java/arena/FramingScenario.java b/src/main/java/arena/FramingScenario.java
new file mode 100644
index 0000000..61edd23
--- /dev/null
+++ b/src/main/java/arena/FramingScenario.java
@@ -0,0 +1,211 @@
+package arena;
+
+import com.fasterxml.jackson.databind.JsonNode;
+import com.fasterxml.jackson.databind.node.ArrayNode;
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import io.netty.buffer.ByteBuf;
+import io.netty.buffer.Unpooled;
+import io.netty.channel.embedded.EmbeddedChannel;
+import java.io.EOFException;
+import java.io.IOException;
+import java.net.InetSocketAddress;
+import java.net.Socket;
+import java.nio.ByteBuffer;
+import java.nio.charset.StandardCharsets;
+import java.util.ArrayList;
+import java.util.Arrays;
+import java.util.HexFormat;
+import java.util.List;
+import java.util.concurrent.TimeUnit;
+
+/** G02 fixture runner: exact reads use the production handler; malformed peers use real TCP. */
+final class FramingScenario {
+    private FramingScenario() { }
+
+    static ObjectNode run(ObjectNode scenario, String scenarioHash) throws IOException {
+        require(scenario.path("contract_version").asInt() == 1, "unsupported G02 contract version");
+        require(scenario.path("parser_bound_bytes").asInt() == CompleteFrame.BUFFER_BOUND, "unfrozen parser bound");
+        require(scenario.path("execution_ceiling_ms_per_socket_case").asInt() == 5_000, "unfrozen socket ceiling");
+        ObjectNode result = Json.MAPPER.createObjectNode().put("thread", "G02")
+                .put("scenario_id", scenario.path("scenario_id").asText()).put("contract_version", 1)
+                .put("scenario_sha256", scenarioHash).put("parser_bound_bytes", CompleteFrame.BUFFER_BOUND);
+        ArrayNode matrix = result.putArray("fragmentation_matrix");
+        for (JsonNode value : scenario.withArray("messages")) {
+            require(value instanceof ObjectNode, "scenario message must be object");
+            ObjectNode message = (ObjectNode) value;
+            byte[] bytes = wire(Json.bytes(message));
+            for (JsonNode split : scenario.withArray("fragmentations")) {
+                int[] sizes = switch (split.asText()) {
+                    case "1/2/rest" -> new int[] {1, 2, bytes.length - 3};
+                    case "header/payload" -> new int[] {4, bytes.length - 4};
+                    case "all-at-once" -> new int[] {bytes.length};
+                    default -> throw new IOException("unsupported frozen split");
+                };
+                matrix.add(parserCase(bytes, List.of(message), sizes).put("message_type", message.path("type").asText())
+                        .put("fragmentation", split.asText()));
+            }
+        }
+        ArrayNode indices = scenario.withArray("coalesce_message_indices");
+        require(indices.size() == 2, "G02 has one coalesced pair");
+        ObjectNode first = (ObjectNode) scenario.withArray("messages").get(indices.get(0).asInt());
+        ObjectNode second = (ObjectNode) scenario.withArray("messages").get(indices.get(1).asInt());
+        byte[] a = wire(Json.bytes(first));
+        byte[] b = wire(Json.bytes(second));
+        byte[] coalesced = ByteBuffer.allocate(a.length + b.length).put(a).put(b).array();
+        result.set("coalescing", parserCase(coalesced, List.of(first, second), coalesced.length));
+        result.set("socket_isolation", socketCases(scenario));
+        result.put("state_hashes", "NOT_ACTIVATED_G07").put("network_fault_runs", 0).put("load_runs", 0);
+        return result;
+    }
+
+    private static ObjectNode parserCase(byte[] bytes, List<ObjectNode> expected, int... sizes) throws IOException {
+        List<ObjectNode> received = new ArrayList<>();
+        CompleteFrame parser = new CompleteFrame(received::add);
+        EmbeddedChannel channel = new EmbeddedChannel(parser);
+        ObjectNode result = Json.MAPPER.createObjectNode();
+        ArrayNode reads = result.putArray("reads");
+        ByteBuf owned = null;
+        int offset = 0;
+        try {
+            for (int size : sizes) {
+                require(size > 0 && offset + size <= bytes.length, "invalid fixed split");
+                ByteBuf chunk = Unpooled.wrappedBuffer(Arrays.copyOfRange(bytes, offset, offset + size));
+                channel.writeInbound(chunk);
+                offset += size;
+                ObjectNode state = parser.state();
+                reads.add(state.deepCopy().put("read_bytes", size).put("inbound_ref_count", chunk.refCnt()));
+                require(chunk.refCnt() == 0, "inbound buffer not released");
+                require(channel.isOpen(), "valid framing closed channel");
+                require(state.path("message_errors").asInt() == 0 && state.path("framing_errors").asInt() == 0,
+                        "valid framing emitted error");
+                require(state.path("buffer_high_water").asInt() <= CompleteFrame.BUFFER_BOUND
+                        && state.path("capacity_high_water").asInt() <= CompleteFrame.BUFFER_BOUND, "parser bound exceeded");
+                if (offset < bytes.length) {
+                    require(received.isEmpty(), "message dispatched before full payload");
+                    require(state.path("outcome").asText().equals("NEED_MORE_BYTES"), "partial read classification");
+                }
+            }
+            require(offset == bytes.length && received.equals(expected), "parsed content/order differs from fixture");
+            require(channel.outboundMessages().isEmpty(), "valid parser unexpectedly replied");
+            result.set("parser", parser.state());
+            ArrayNode types = result.putArray("parsed_types");
+            received.forEach(message -> types.add(message.path("type").asText()));
+            owned = parser.ownedBuffer();
+        } finally { channel.finishAndReleaseAll(); }
+        require(owned != null && owned.refCnt() == 0 && !channel.isOpen(), "parser disposal leaked buffer/channel");
+        result.set("after_close", parser.state());
+        result.put("owned_buffer_ref_count_after_close", owned.refCnt());
+        return result;
+    }
+
+    static ObjectNode socketCases(ObjectNode scenario) throws IOException {
+        int ceilingMs = scenario.path("execution_ceiling_ms_per_socket_case").asInt();
+        require(ceilingMs == 5_000, "unfrozen socket deadline");
+        ObjectNode evidence = Json.MAPPER.createObjectNode();
+        ArrayNode cases = evidence.putArray("cases");
+        ArenaServer server = new ArenaServer("127.0.0.1", 0, true);
+        List<WireSocket> clients = new ArrayList<>();
+        try {
+            long initialDeadline = deadline(ceilingMs);
+            WireSocket healthy = new WireSocket(server.port(), initialDeadline);
+            clients.add(healthy);
+            hello(healthy, initialDeadline);
+            for (JsonNode bad : scenario.withArray("malformed_cases")) {
+                long started = System.nanoTime();
+                long limit = started + TimeUnit.MILLISECONDS.toNanos(ceilingMs);
+                WireSocket malformed = new WireSocket(server.port(), limit);
+                clients.add(malformed);
+                ObjectNode observation = cases.addObject().put("name", bad.path("name").asText());
+                try {
+                    byte[] payload = bad.has("payload_text") ? bad.path("payload_text").asText().getBytes(StandardCharsets.UTF_8)
+                            : HexFormat.of().parseHex(bad.path("payload_hex").asText());
+                    int declared = bad.has("declared_length") ? bad.path("declared_length").asInt() : payload.length;
+                    byte[] request = ByteBuffer.allocate(4 + payload.length).putInt(declared).put(payload).array();
+                    malformed.send(request);
+                    ObjectNode error = malformed.read(limit);
+                    require(error.path("type").asText().equals("ERROR"), "malformed peer must receive error");
+                    boolean terminal = declared == 0 || declared > CompleteFrame.MAX_BYTES;
+                    String expected = terminal ? "FRAME_SIZE_INVALID" : "MESSAGE_INVALID";
+                    require(error.path("code").asText().equals(expected), "wrong malformed error code");
+                    require(!error.path("message").asText().isEmpty(), "error lacks human-readable message");
+                    observation.put("error_code", error.path("code").asText());
+                    if (terminal) {
+                        malformed.expectEof(limit);
+                        observation.put("connection_effect", "CLOSED").put("terminal", true);
+                    } else {
+                        hello(malformed, limit);
+                        observation.put("connection_effect", "OPEN_RECOVERABLE").put("terminal", false)
+                                .put("same_connection_hello", "WELCOME");
+                    }
+                    hello(healthy, limit);
+                    observation.put("healthy_connection_hello", "WELCOME");
+                    long elapsed = System.nanoTime() - started;
+                    require(elapsed < TimeUnit.MILLISECONDS.toNanos(ceilingMs), "socket case exceeded fixed deadline");
+                    observation.put("elapsed_ms", TimeUnit.NANOSECONDS.toMillis(elapsed));
+                } finally { malformed.close(); }
+                observation.put("client_socket_closed", malformed.isClosed());
+            }
+            evidence.set("runtime_metrics", server.metrics());
+        } finally {
+            for (WireSocket client : clients) client.close();
+            server.close();
+        }
+        require(clients.stream().allMatch(WireSocket::isClosed), "client socket leak");
+        ObjectNode cleanup = server.cleanup();
+        ScenarioRunner.assertCleanup(cleanup);
+        evidence.set("cleanup", cleanup);
+        evidence.put("all_client_sockets_closed", true);
+        return evidence;
+    }
+
+    private static void hello(WireSocket socket, long deadline) throws IOException {
+        socket.send(wire(Json.bytes(Json.message("HELLO"))));
+        require(socket.read(deadline).path("type").asText().equals("WELCOME"), "healthy/recovered peer did not respond");
+    }
+
+    private static byte[] wire(byte[] payload) {
+        return ByteBuffer.allocate(4 + payload.length).putInt(payload.length).put(payload).array();
+    }
+    private static long deadline(int ceilingMs) { return System.nanoTime() + TimeUnit.MILLISECONDS.toNanos(ceilingMs); }
+    private static void require(boolean condition, String message) throws IOException {
+        if (!condition) throw new IOException(message);
+    }
+
+    private static final class WireSocket implements AutoCloseable {
+        private final Socket socket = new Socket();
+        WireSocket(int port, long deadline) throws IOException {
+            try {
+                socket.connect(new InetSocketAddress("127.0.0.1", port), remainingMs(deadline));
+                socket.setTcpNoDelay(true);
+            } catch (IOException failure) { socket.close(); throw failure; }
+        }
+        void send(byte[] bytes) throws IOException { socket.getOutputStream().write(bytes); socket.getOutputStream().flush(); }
+        ObjectNode read(long deadline) throws IOException {
+            int size = ByteBuffer.wrap(readExact(4, deadline)).getInt();
+            require(size > 0 && size <= CompleteFrame.MAX_BYTES, "invalid server frame length");
+            return Json.read(readExact(size, deadline));
+        }
+        private byte[] readExact(int size, long deadline) throws IOException {
+            byte[] bytes = new byte[size];
+            int read = 0;
+            while (read < size) {
+                socket.setSoTimeout(remainingMs(deadline));
+                int count = socket.getInputStream().read(bytes, read, size - read);
+                if (count < 0) throw new EOFException("incomplete server frame");
+                read += count;
+            }
+            return bytes;
+        }
+        void expectEof(long deadline) throws IOException {
+            socket.setSoTimeout(remainingMs(deadline));
+            require(socket.getInputStream().read() == -1, "terminal frame error did not close connection");
+        }
+        boolean isClosed() { return socket.isClosed(); }
+        @Override public void close() throws IOException { socket.close(); }
+        private static int remainingMs(long deadline) throws IOException {
+            long nanos = deadline - System.nanoTime();
+            require(nanos > 0, "socket deadline exhausted");
+            return (int) Math.max(1, TimeUnit.NANOSECONDS.toMillis(nanos));
+        }
+    }
+}
diff --git a/src/main/java/arena/Json.java b/src/main/java/arena/Json.java
index 3870d71..e534af8 100644
--- a/src/main/java/arena/Json.java
+++ b/src/main/java/arena/Json.java
@@ -1,13 +1,23 @@
 package arena;
 
 import com.fasterxml.jackson.core.JsonProcessingException;
+import com.fasterxml.jackson.core.JsonFactory;
+import com.fasterxml.jackson.core.StreamReadFeature;
+import com.fasterxml.jackson.core.json.JsonReadFeature;
+import com.fasterxml.jackson.databind.DeserializationFeature;
 import com.fasterxml.jackson.databind.JsonNode;
 import com.fasterxml.jackson.databind.ObjectMapper;
 import com.fasterxml.jackson.databind.node.ObjectNode;
 import java.io.IOException;
+import java.nio.ByteBuffer;
+import java.nio.charset.CodingErrorAction;
+import java.nio.charset.StandardCharsets;
 
 final class Json {
-    static final ObjectMapper MAPPER = new ObjectMapper();
+    static final ObjectMapper MAPPER = new ObjectMapper(JsonFactory.builder()
+            .enable(StreamReadFeature.STRICT_DUPLICATE_DETECTION)
+            .disable(JsonReadFeature.ALLOW_NON_NUMERIC_NUMBERS).build())
+            .enable(DeserializationFeature.FAIL_ON_TRAILING_TOKENS);
     private Json() { }
 
     static ObjectNode message(String type) {
@@ -15,7 +25,13 @@ final class Json {
     }
 
     static ObjectNode read(byte[] bytes) throws IOException {
-        JsonNode node = MAPPER.readTree(bytes);
+        return read(ByteBuffer.wrap(bytes));
+    }
+
+    static ObjectNode read(ByteBuffer bytes) throws IOException {
+        String text = StandardCharsets.UTF_8.newDecoder().onMalformedInput(CodingErrorAction.REPORT)
+                .onUnmappableCharacter(CodingErrorAction.REPORT).decode(bytes).toString();
+        JsonNode node = MAPPER.readTree(text);
         if (!(node instanceof ObjectNode object)) throw new IOException("object required");
         return object;
     }
diff --git a/src/main/java/arena/ScenarioRunner.java b/src/main/java/arena/ScenarioRunner.java
index 163d63e..62a496c 100644
--- a/src/main/java/arena/ScenarioRunner.java
+++ b/src/main/java/arena/ScenarioRunner.java
@@ -21,6 +21,8 @@ final class ScenarioRunner {
         if (Files.size(inputPath) > 65_536) throw new IOException("scenario too large");
         byte[] scenarioBytes = Files.readAllBytes(inputPath);
         ObjectNode scenario = Json.read(scenarioBytes);
+        if (scenario.path("thread").asText().equals("G02"))
+            return FramingScenario.run(scenario, sha256(scenarioBytes));
         if (!scenario.path("thread").asText().equals("G01") || scenario.path("contract_version").asInt() != 1
                 || !scenario.path("clock").path("kind").asText().equals("manual")
                 || scenario.path("clock").path("tick_duration_ms").asInt() != 50
@@ -122,6 +124,12 @@ final class ScenarioRunner {
         if (cleanup.path("pending_input_high_water").asInt() > Room.INPUT_LIMIT) failures.add("input bound");
         if (cleanup.path("mailbox_high_water").asInt() > ArenaServer.MAILBOX_LIMIT) failures.add("mailbox bound");
         if (cleanup.path("outbound_high_water").asInt() > ArenaServer.OUTBOUND_LIMIT) failures.add("outbound bound");
+        JsonNode parser = cleanup.path("parser");
+        if (parser.path("live_buffers").asInt(-1) != 0 || parser.path("allocated_bytes").asInt(-1) != 0)
+            failures.add("parser buffer cleanup");
+        if (parser.path("buffer_high_water").asInt() > CompleteFrame.BUFFER_BOUND
+                || parser.path("capacity_high_water").asInt() > CompleteFrame.BUFFER_BOUND)
+            failures.add("parser bound");
         if (!failures.isEmpty()) throw new IOException("cleanup failure: " + failures);
     }
 
diff --git a/src/test/java/arena/CompleteFrameTest.java b/src/test/java/arena/CompleteFrameTest.java
index 4348f39..af9ef46 100644
--- a/src/test/java/arena/CompleteFrameTest.java
+++ b/src/test/java/arena/CompleteFrameTest.java
@@ -3,10 +3,19 @@ package arena;
 import static org.junit.jupiter.api.Assertions.*;
 import com.fasterxml.jackson.databind.node.ObjectNode;
 import io.netty.buffer.ByteBuf;
+import io.netty.buffer.Unpooled;
 import io.netty.channel.embedded.EmbeddedChannel;
+import java.nio.ByteBuffer;
+import java.nio.charset.StandardCharsets;
+import java.security.MessageDigest;
 import java.util.ArrayList;
+import java.util.Arrays;
+import java.util.Collection;
+import java.util.HexFormat;
 import java.util.List;
+import org.junit.jupiter.api.DynamicTest;
 import org.junit.jupiter.api.Test;
+import org.junit.jupiter.api.TestFactory;
 
 final class CompleteFrameTest {
     @Test void completeFrameIsConsumedAndInboundOwnershipReleased() {
@@ -35,4 +44,248 @@ final class CompleteFrameTest {
         assertThrows(IllegalArgumentException.class,
                 () -> CompleteFrame.encode(Json.message("ERROR").put("message", "x".repeat(CompleteFrame.MAX_BYTES))));
     }
+
+    @TestFactory Collection<DynamicTest> frozenG02FramingMatrix() throws Exception {
+        byte[] fixture;
+        try (var input = getClass().getResourceAsStream("/G02.json")) {
+            assertNotNull(input);
+            fixture = input.readAllBytes();
+        }
+        assertEquals("a1d103416b07e5fdb30d349e1938123851727bcaa33ac99baf72404505464692",
+                HexFormat.of().formatHex(MessageDigest.getInstance("SHA-256").digest(fixture)));
+        ObjectNode scenario = Json.read(fixture);
+        List<DynamicTest> cases = new ArrayList<>();
+        for (var message : scenario.withArray("messages")) {
+            ObjectNode value = (ObjectNode) message;
+            byte[] frame = wire(value);
+            for (var split : scenario.withArray("fragmentations")) {
+                String name = value.path("type").asText() + " " + split.asText();
+                int[] lengths = switch (split.asText()) {
+                    case "1/2/rest" -> new int[] {1, 2, frame.length - 3};
+                    case "header/payload" -> new int[] {4, frame.length - 4};
+                    case "all-at-once" -> new int[] {frame.length};
+                    default -> throw new IllegalArgumentException("unfrozen split");
+                };
+                cases.add(DynamicTest.dynamicTest(name, () -> verifyReads(name, frame, List.of(value), lengths)));
+            }
+        }
+        ObjectNode first = (ObjectNode) scenario.withArray("messages").get(scenario.withArray("coalesce_message_indices").get(0).asInt());
+        ObjectNode second = (ObjectNode) scenario.withArray("messages").get(scenario.withArray("coalesce_message_indices").get(1).asInt());
+        byte[] firstFrame = wire(first);
+        byte[] secondFrame = wire(second);
+        byte[] joined = ByteBuffer.allocate(firstFrame.length + secondFrame.length).put(firstFrame).put(secondFrame).array();
+        cases.add(DynamicTest.dynamicTest("coalesced HELLO CREATE_ROOM",
+                () -> verifyReads("coalesced HELLO CREATE_ROOM", joined, List.of(first, second), joined.length)));
+        return cases;
+    }
+
+    private static byte[] wire(ObjectNode value) {
+        byte[] payload = Json.bytes(value);
+        return ByteBuffer.allocate(4 + payload.length).putInt(payload.length).put(payload).array();
+    }
+
+    private static void verifyReads(String name, byte[] frame, List<ObjectNode> expected, int... lengths) {
+        List<ObjectNode> received = new ArrayList<>();
+        EmbeddedChannel channel = new EmbeddedChannel(new CompleteFrame(received::add));
+        int offset = 0;
+        try {
+            for (int length : lengths) {
+                ByteBuf chunk = Unpooled.wrappedBuffer(Arrays.copyOfRange(frame, offset, offset + length));
+                channel.writeInbound(chunk);
+                offset += length;
+                String error = "none";
+                ByteBuf reply = channel.readOutbound();
+                if (reply != null) {
+                    try {
+                        int size = reply.readInt();
+                        byte[] payload = new byte[size];
+                        reply.readBytes(payload);
+                        error = Json.read(payload).path("code").asText();
+                    } catch (java.io.IOException invalid) { throw new AssertionError(invalid); }
+                    finally { reply.release(); }
+                }
+                System.out.println(name + " bytes=" + offset + " messages=" + received.size()
+                        + " open=" + channel.isOpen() + " error=" + error + " inbound_refcnt=" + chunk.refCnt());
+                assertEquals(0, chunk.refCnt(), "inbound ownership released");
+                assertTrue(channel.isOpen(), "valid partial/coalesced frame must preserve connection");
+                assertEquals("none", error, "valid framing must not emit an error");
+                assertEquals(offset == frame.length ? expected.size() : 0, received.size());
+            }
+            assertEquals(frame.length, offset);
+            assertEquals(expected, received, "messages must preserve content and order");
+        } finally { channel.finishAndReleaseAll(); }
+        assertFalse(channel.isOpen());
+    }
+
+
+    private record ProtocolCase(String name, String payload, String code) { }
+
+    @TestFactory Collection<DynamicTest> fixedProtocolForms() {
+        List<ProtocolCase> forms = List.of(
+                new ProtocolCase("root-array", "[]", "MESSAGE_INVALID"),
+                new ProtocolCase("missing-v", "{\"type\":\"HELLO\"}", "MESSAGE_INVALID"),
+                new ProtocolCase("missing-type", "{\"v\":1}", "MESSAGE_INVALID"),
+                new ProtocolCase("string-v", "{\"v\":\"1\",\"type\":\"HELLO\"}", "MESSAGE_INVALID"),
+                new ProtocolCase("floating-v", "{\"v\":1.0,\"type\":\"HELLO\"}", "MESSAGE_INVALID"),
+                new ProtocolCase("unsupported-v", "{\"v\":2,\"type\":\"HELLO\"}", "PROTOCOL_VERSION_UNSUPPORTED"),
+                new ProtocolCase("unknown-type", "{\"v\":1,\"type\":\"UNRECOGNIZED\"}", "MESSAGE_TYPE_UNKNOWN"),
+                new ProtocolCase("known-unknown-field", "{\"v\":1,\"type\":\"HELLO\",\"extra\":true}", null),
+                new ProtocolCase("nan", "{\"v\":1,\"type\":\"HELLO\",\"extra\":NaN}", "MESSAGE_INVALID"),
+                new ProtocolCase("infinity", "{\"v\":1,\"type\":\"HELLO\",\"extra\":Infinity}", "MESSAGE_INVALID"),
+                new ProtocolCase("trailing-object", "{\"v\":1,\"type\":\"HELLO\"}{}", "MESSAGE_INVALID"));
+        return forms.stream().map(form -> DynamicTest.dynamicTest(form.name(),
+                () -> verifyProtocol(form.name(), form.payload().getBytes(StandardCharsets.UTF_8), form.code()))).toList();
+    }
+
+    private static void verifyProtocol(String name, byte[] payload, String code) throws Exception {
+        List<ObjectNode> received = new ArrayList<>();
+        CompleteFrame parser = new CompleteFrame(received::add);
+        EmbeddedChannel channel = new EmbeddedChannel(parser);
+        ByteBuf inbound = Unpooled.buffer(4 + payload.length).writeInt(payload.length).writeBytes(payload);
+        ByteBuf owned = null;
+        try {
+            channel.writeInbound(inbound);
+            assertEquals(0, inbound.refCnt());
+            assertTrue(channel.isOpen(), "message errors are recoverable");
+            if (code == null) {
+                assertEquals(1, received.size());
+                assertNull(channel.readOutbound());
+                assertEquals("COMPLETE_VALID_MESSAGE", parser.state().path("outcome").asText());
+            } else {
+                assertTrue(received.isEmpty());
+                assertEquals(code, readError(channel));
+                assertEquals("MESSAGE_ERROR", parser.state().path("outcome").asText());
+            }
+            System.out.println("protocol " + name + " code=" + (code == null ? "ACCEPTED" : code)
+                    + " connection=OPEN state=" + parser.state());
+            channel.writeInbound(CompleteFrame.encode(Json.message("HELLO")));
+            assertEquals(code == null ? 2 : 1, received.size(), "next frame must recover on the same channel");
+            assertNull(channel.readOutbound());
+            owned = parser.ownedBuffer();
+        } finally { channel.finishAndReleaseAll(); }
+        assertNotNull(owned);
+        assertEquals(0, owned.refCnt());
+        assertEquals(0, parser.state().path("buffer_bytes").asInt());
+    }
+
+    @Test void frozenMalformedFrameCorpusKeepsStableErrors() throws Exception {
+        ObjectNode fixture;
+        try (var input = getClass().getResourceAsStream("/G02.json")) {
+            assertNotNull(input);
+            fixture = Json.read(input.readAllBytes());
+        }
+        for (var malformed : fixture.withArray("malformed_cases")) {
+            byte[] payload = malformed.has("payload_text") ? malformed.path("payload_text").asText().getBytes(StandardCharsets.UTF_8)
+                    : HexFormat.of().parseHex(malformed.path("payload_hex").asText());
+            if (!malformed.has("declared_length")) {
+                verifyProtocol(malformed.path("name").asText(), payload, "MESSAGE_INVALID");
+                continue;
+            }
+            List<ObjectNode> received = new ArrayList<>();
+            CompleteFrame parser = new CompleteFrame(received::add);
+            EmbeddedChannel channel = new EmbeddedChannel(parser);
+            ByteBuf bytes = Unpooled.buffer(4).writeInt(malformed.path("declared_length").asInt());
+            try {
+                channel.writeInbound(bytes);
+                assertEquals(0, bytes.refCnt());
+                assertEquals("FRAME_SIZE_INVALID", readError(channel));
+                assertFalse(channel.isOpen());
+                assertTrue(received.isEmpty());
+                assertEquals(1, parser.state().path("framing_errors").asInt());
+                assertEquals("FRAMING_CLOSE", parser.state().path("end_reason").asText());
+                assertEquals(0, parser.state().path("buffer_ref_count").asInt());
+                assertEquals(4, parser.state().path("capacity_high_water").asInt(), "invalid length never allocates payload");
+                System.out.println("malformed " + malformed.path("name").asText()
+                        + " code=FRAME_SIZE_INVALID connection=CLOSED state=" + parser.state());
+            } finally { channel.finishAndReleaseAll(); }
+        }
+    }
+
+    private static String readError(EmbeddedChannel channel) throws Exception {
+        ByteBuf reply = channel.readOutbound();
+        assertNotNull(reply);
+        try {
+            int size = reply.readInt();
+            byte[] payload = new byte[size];
+            reply.readBytes(payload);
+            ObjectNode message = Json.read(payload);
+            assertEquals("ERROR", message.path("type").asText());
+            assertFalse(message.path("message").asText().isEmpty());
+            return message.path("code").asText();
+        } finally {
+            reply.release();
+            assertEquals(0, reply.refCnt(), "error reply ownership released");
+        }
+    }
+
+    @TestFactory Collection<DynamicTest> partialEofIsExplicitAndReleasesCumulation() {
+        return List.of(1, 6).stream().map(prefix -> DynamicTest.dynamicTest("partial EOF after " + prefix + " bytes", () -> {
+            CompleteFrame parser = new CompleteFrame(ignored -> fail("partial frame must not dispatch"));
+            EmbeddedChannel channel = new EmbeddedChannel(parser);
+            ByteBuf bytes = Unpooled.wrappedBuffer(Arrays.copyOf(wire(Json.message("HELLO")), prefix));
+            channel.writeInbound(bytes);
+            assertEquals(0, bytes.refCnt());
+            assertEquals("NEED_MORE_BYTES", parser.state().path("outcome").asText());
+            ByteBuf owned = parser.ownedBuffer();
+            assertNotNull(owned);
+            assertEquals(1, owned.refCnt());
+            channel.finishAndReleaseAll();
+            assertEquals(0, owned.refCnt());
+            assertEquals("IO_END", parser.state().path("outcome").asText());
+            assertEquals("PARTIAL_EOF", parser.state().path("end_reason").asText());
+            assertEquals(0, parser.state().path("buffer_bytes").asInt());
+            System.out.println("partial EOF prefix=" + prefix + " state=" + parser.state());
+        })).toList();
+    }
+
+    @Test void maximumValidFrameReachesButNeverExceedsBound() {
+        ObjectNode message = Json.message("HELLO").put("padding", "");
+        int emptySize = Json.bytes(message).length;
+        message.put("padding", "x".repeat(CompleteFrame.MAX_BYTES - emptySize));
+        byte[] frame = wire(message);
+        assertEquals(CompleteFrame.BUFFER_BOUND, frame.length);
+        List<ObjectNode> received = new ArrayList<>();
+        CompleteFrame parser = new CompleteFrame(received::add);
+        EmbeddedChannel channel = new EmbeddedChannel(parser);
+        ByteBuf header = Unpooled.wrappedBuffer(Arrays.copyOfRange(frame, 0, 4));
+        ByteBuf payload = Unpooled.wrappedBuffer(Arrays.copyOfRange(frame, 4, frame.length));
+        ByteBuf owned = null;
+        try {
+            channel.writeInbound(header);
+            assertEquals(4, parser.state().path("capacity_high_water").asInt());
+            assertEquals("NEED_MORE_BYTES", parser.state().path("outcome").asText());
+            channel.writeInbound(payload);
+            assertEquals(List.of(message), received);
+            assertEquals(0, header.refCnt());
+            assertEquals(0, payload.refCnt());
+            assertEquals(CompleteFrame.BUFFER_BOUND, parser.state().path("buffer_high_water").asInt());
+            assertEquals(CompleteFrame.BUFFER_BOUND, parser.state().path("capacity_high_water").asInt());
+            assertEquals(0, parser.bufferedBytes());
+            System.out.println("max-size frame state=" + parser.state());
+            owned = parser.ownedBuffer();
+        } finally { channel.finishAndReleaseAll(); }
+        assertNotNull(owned);
+        assertEquals(0, owned.refCnt());
+        assertEquals("CLEAN_EOF", parser.state().path("end_reason").asText());
+    }
+
+    @Test void transportFailureIsNotAFramingOrMessageError() {
+        CompleteFrame parser = new CompleteFrame(ignored -> fail("partial frame must not dispatch"));
+        EmbeddedChannel channel = new EmbeddedChannel(parser);
+        ByteBuf input = Unpooled.wrappedBuffer(new byte[] {0});
+        channel.writeInbound(input);
+        assertEquals(0, input.refCnt());
+        ByteBuf owned = parser.ownedBuffer();
+        assertNotNull(owned);
+        channel.pipeline().fireExceptionCaught(new java.io.IOException("fixed unit transport end"));
+        assertFalse(channel.isOpen());
+        assertEquals(0, owned.refCnt());
+        assertEquals("IO_END", parser.state().path("outcome").asText());
+        assertEquals("IO_ERROR", parser.state().path("end_reason").asText());
+        assertEquals(0, parser.state().path("message_errors").asInt());
+        assertEquals(0, parser.state().path("framing_errors").asInt());
+        channel.finishAndReleaseAll();
+        System.out.println("transport error state=" + parser.state());
+    }
+
 }
diff --git a/src/test/java/arena/ServerIntegrationTest.java b/src/test/java/arena/ServerIntegrationTest.java
index cd9334e..828d8cf 100644
--- a/src/test/java/arena/ServerIntegrationTest.java
+++ b/src/test/java/arena/ServerIntegrationTest.java
@@ -108,4 +108,27 @@ final class ServerIntegrationTest {
             }
         }
     }
+
+    @Test void frozenMalformedPeersLeaveHealthyConnectionRunningAndReleaseBuffers() throws Exception {
+        ObjectNode scenario;
+        try (var input = getClass().getResourceAsStream("/G02.json")) {
+            assertNotNull(input);
+            scenario = Json.read(input.readAllBytes());
+        }
+        ObjectNode evidence = FramingScenario.socketCases(scenario);
+        assertEquals(4, evidence.withArray("cases").size());
+        for (var observed : evidence.withArray("cases")) {
+            boolean terminal = observed.path("name").asText().endsWith("length");
+            assertEquals(terminal ? "FRAME_SIZE_INVALID" : "MESSAGE_INVALID", observed.path("error_code").asText());
+            assertEquals(terminal ? "CLOSED" : "OPEN_RECOVERABLE", observed.path("connection_effect").asText());
+            assertEquals("WELCOME", observed.path("healthy_connection_hello").asText());
+            assertTrue(observed.path("client_socket_closed").asBoolean());
+            assertTrue(observed.path("elapsed_ms").asLong() < 5_000);
+        }
+        assertTrue(evidence.path("all_client_sockets_closed").asBoolean());
+        assertEquals(0, evidence.path("cleanup").path("parser").path("live_buffers").asInt());
+        assertEquals(0, evidence.path("cleanup").path("parser").path("allocated_bytes").asInt());
+        System.out.println("G02 socket isolation " + evidence);
+    }
+
 }
diff --git a/src/test/resources/G02.json b/src/test/resources/G02.json
new file mode 100644
index 0000000..8bb331c
--- /dev/null
+++ b/src/test/resources/G02.json
@@ -0,0 +1,54 @@
+{
+  "scenario_id": "G02-framing",
+  "contract_version": 1,
+  "thread": "G02",
+  "messages": [
+    {
+      "v": 1,
+      "type": "HELLO"
+    },
+    {
+      "v": 1,
+      "type": "CREATE_ROOM",
+      "session_id": "session-fixture"
+    },
+    {
+      "v": 1,
+      "type": "JOIN_ROOM",
+      "session_id": "session-fixture",
+      "room_id": "room-fixture"
+    }
+  ],
+  "fragmentations": [
+    "1/2/rest",
+    "header/payload",
+    "all-at-once"
+  ],
+  "coalesce_message_indices": [
+    0,
+    1
+  ],
+  "malformed_cases": [
+    {
+      "name": "zero-length",
+      "declared_length": 0,
+      "payload_hex": ""
+    },
+    {
+      "name": "oversize-length",
+      "declared_length": 16385,
+      "payload_hex": ""
+    },
+    {
+      "name": "duplicate-key",
+      "payload_text": "{\"v\":1,\"v\":1,\"type\":\"HELLO\"}"
+    },
+    {
+      "name": "invalid-utf8",
+      "payload_hex": "7b2276223a312c2274797065223a2248454c4c4f222c226578747261223a22ff227d"
+    }
+  ],
+  "parser_bound_bytes": 16388,
+  "socket_case": "each malformed client is isolated while one other connection remains healthy",
+  "execution_ceiling_ms_per_socket_case": 5000
+}
