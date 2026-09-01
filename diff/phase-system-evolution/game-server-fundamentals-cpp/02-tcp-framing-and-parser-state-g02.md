# TCP Framing과 Parser State

## `feat: decode bounded incremental TCP frames`

diff --git a/TRACK.md b/TRACK.md
index 69f9422..1d922a9 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -1,4 +1,4 @@
-# fundamentals-cpp — G01 baseline
+# fundamentals-cpp — G02 bounded TCP framing
 
 SPEC_REVISION: `5a6e4a2f8fc71d4be18c3279583bfc2558d5c232`
 
@@ -25,6 +25,7 @@ Run from this worktree; `track` resolves its own source directory. Build never r
 ./track unit-test
 ./track integration-test
 ./track scenario-run /absolute/path/to/G01.json /absolute/path/to/evidence.json
+./track scenario-run /absolute/path/to/G02.json /absolute/path/to/evidence.json
 ./track replay-verify /absolute/path/to/replay.json /absolute/path/to/evidence.json
 ./track server /absolute/path/to/config.json
 ```
@@ -62,7 +63,14 @@ One process owns one Room. Join commit order determines spawn slots. The second
 Only integer arithmetic updates movement, clamp, TAG and score. TAG is one-shot; direction persists.
 The session executes ticks 0–1199, then sends authoritative `ROOM_FINISHED` and stops advancing.
 
+G02 replaces the complete-read assumption with one `FrameParser` per Connection. It consumes one bounded frame at a time and returns the consumed-byte count so a coalesced suffix is processed in order. Incomplete headers and payloads remain with their connection. Parser states are `need_more`, `message`, `message_error`, `terminal_frame_error` and `io_end`.
+
+Strict UTF-8 JSON decoding rejects duplicate keys at any object depth, invalid root/envelope/schema types, NaN, Infinity and trailing JSON. Unknown fields on the five active client message types remain ignored. Sequence, target tick and future message families stay inactive. Message errors enter the bounded owner mailbox in stream order and allow the next frame. Invalid lengths attempt one bounded nonblocking error flush and then close only that connection. Error text is fixed and bounded; parser exception text is never serialized.
+
+Clean EOF is `TRANSPORT_EOF`; partial header/payload EOF is `TRANSPORT_EOF_IN_FRAME`; failed I/O is `TRANSPORT_IO_ERROR`. All terminate the transport and clear retained bytes. These transport observations do not change Room lifecycle policy.
+
 - Connection lifetime: `Server::connections_`, each descriptor owned by move-only `Fd`.
+- Parser lifetime: the Connection owns a fixed 16,388-byte array and parser state; erasing the Connection releases that storage. No frame-sized allocation is made from an unchecked length, and no unbounded decoded-message list is accumulated.
 - Session identity: server-generated opaque per-connection identifier; no reconnect credential or persistence in G01.
 - Room state: the server's one execution context, enforced by Room owner-thread checks.
 - Mailbox: kqueue read callback produces bounded `Envelope` messages; `drain_mailbox` consumes them after I/O callbacks return. Only that owner phase or explicit tick mutates Room.
@@ -75,10 +83,11 @@ Both clients observe LOBBY, RUNNING and CLOSED via actual TCP. The canonical 120
 
 ## Explicit resource bounds
 
-| Resource | G01 bound and excess behavior |
+| Resource | Current bound and excess behavior |
 |---|---|
-| TCP JSON payload | 16,384 bytes; rejected/disconnected on oversized or incomplete G01 frame |
-| Read buffer | one stack array of 16,388 bytes per read; no retained incremental parser |
+| TCP JSON payload | 1–16,384 bytes; size 0/16,385+ attempts FRAME_SIZE_INVALID and disconnects |
+| Parser buffer | exactly 16,388 bytes of owned storage per connection; partial data retained only within this bound |
+| Read buffer | separate 16,388-byte stack array per read; coalesced suffix consumed without growing parser storage |
 | Connections | 512; excess accepted descriptor closed and ADMISSION_REJECTED metric recorded |
 | Rooms | one; extra create gets ROOM_NOT_JOINABLE |
 | Players | eight slots maximum; running Room rejects new joins |
@@ -96,7 +105,7 @@ Both clients observe LOBBY, RUNNING and CLOSED via actual TCP. The canonical 120
 Frame-bound, input-bound and RAII assertions run in unit tests; real socket descriptor cleanup and queue high-water checks run in integration/scenario tests. A foreign-thread Room mutation is rejected before touching state.
 The standalone integration case starts the CLI on port 0, waits for READY, sends one real HELLO and receives WELCOME, sends SIGTERM once, requires exit 0 and reaping within a fixed 5s process deadline, and rebinds the released listener port. The signal handler only sets a stop flag; the ordinary owner loop performs cleanup.
 
-## Fixed verification and budget
+## G01 baseline verification and budget
 
 Initial attempt budget: at most 8 configure/compile invocations (conservatively counting both), 4 unit suites, 2 integration suites, 1 canonical scenario, 0 load and 0 fault runs.
 This file fixes commands before the first build. Every wrapper command records epoch start, duration in seconds, exit code, exact executable/arguments and log path in `artifacts/evidence/runs.tsv`, including failed commands.
@@ -107,5 +116,10 @@ It resolves role names to server-generated session/player IDs, sends complete fr
 
 ## Deliberate next constraints
 
-The G01 server assumes exactly one complete frame per nonblocking read and keeps no incomplete frame state. Fragmentation/coalescing and strict malformed JSON validation are G02 work. This limitation is reported, not hidden behind the test client's transport.
+The G01 complete-frame limitation is preserved in the pre-change G02 reproduction evidence. G02's scenario runner reads main's frozen input, checks all nine parser/split combinations, forces those same chunks through real TCP by observing retained bytes before each next write, sends the concatenated pair in one actual write, and isolates each of the four malformed peers beside one persistent healthy connection. Fixture session identifiers intentionally reach existing SESSION_INVALID checks after valid parsing; the runner never substitutes server-specific input bytes.
+
+Pure unit cases also validate the frozen supplemental protocol list, an exact 16,384-byte valid payload, partial header/payload EOF, and clean versus failed I/O. Unchanged G01 tests remain in the unit and integration suites. The parser bound is tested at its exact maximum, not inferred from small-frame observations.
+
+G02 initial ceilings: 8 configure/compile invocations, 4 unit invocations including reproduction, 2 integration suites, at most 2 canonical runs (only one post-change run when reproduction is pure unit), zero network-fault/load runs. The actual G02 command ledger, retained failures and results are linked from `evidence/G02.md`. Builds/tests use the unchanged frozen toolchain and ordinary `ARENA_TSAN=ON` configuration. Each `./track build` consumes two conservative configure/compile units.
+
 Detailed identity/lifecycle matrices (G03), clock accumulator/catch-up (G04), input sequence/target tick (G05), abuse matrix (G06), replay/hash (G07), full/delta cadence (G08), UDP, reconnect, slow-consumer coalescing and many-room scheduling remain inactive.
diff --git a/evidence/G02-canonical-summary.json b/evidence/G02-canonical-summary.json
new file mode 100644
index 0000000..eca57ff
--- /dev/null
+++ b/evidence/G02-canonical-summary.json
@@ -0,0 +1,516 @@
+{
+  "thread": "G02",
+  "profile": "realtime-core",
+  "spec_revision": "5a6e4a2f8fc71d4be18c3279583bfc2558d5c232",
+  "start": "79cbe0e54ffa8bf231c81a04b1136864dc9c1e78",
+  "scenario_sha256": "a1d103416b07e5fdb30d349e1938123851727bcaa33ac99baf72404505464692",
+  "result": "PASS",
+  "fragmentation_matrix": [
+    {
+      "type": "HELLO",
+      "fragmentation": "1/2/rest",
+      "chunk_sizes": [
+        1,
+        2,
+        23
+      ],
+      "observed_states": [
+        "need_more",
+        "need_more",
+        "message"
+      ],
+      "messages": [
+        {
+          "type": "HELLO",
+          "v": 1
+        }
+      ],
+      "parser_high_water_bytes": 26,
+      "result": "PASS"
+    },
+    {
+      "type": "HELLO",
+      "fragmentation": "header/payload",
+      "chunk_sizes": [
+        4,
+        22
+      ],
+      "observed_states": [
+        "need_more",
+        "message"
+      ],
+      "messages": [
+        {
+          "type": "HELLO",
+          "v": 1
+        }
+      ],
+      "parser_high_water_bytes": 26,
+      "result": "PASS"
+    },
+    {
+      "type": "HELLO",
+      "fragmentation": "all-at-once",
+      "chunk_sizes": [
+        26
+      ],
+      "observed_states": [
+        "message"
+      ],
+      "messages": [
+        {
+          "type": "HELLO",
+          "v": 1
+        }
+      ],
+      "parser_high_water_bytes": 26,
+      "result": "PASS"
+    },
+    {
+      "type": "CREATE_ROOM",
+      "fragmentation": "1/2/rest",
+      "chunk_sizes": [
+        1,
+        2,
+        60
+      ],
+      "observed_states": [
+        "need_more",
+        "need_more",
+        "message"
+      ],
+      "messages": [
+        {
+          "session_id": "session-fixture",
+          "type": "CREATE_ROOM",
+          "v": 1
+        }
+      ],
+      "parser_high_water_bytes": 63,
+      "result": "PASS"
+    },
+    {
+      "type": "CREATE_ROOM",
+      "fragmentation": "header/payload",
+      "chunk_sizes": [
+        4,
+        59
+      ],
+      "observed_states": [
+        "need_more",
+        "message"
+      ],
+      "messages": [
+        {
+          "session_id": "session-fixture",
+          "type": "CREATE_ROOM",
+          "v": 1
+        }
+      ],
+      "parser_high_water_bytes": 63,
+      "result": "PASS"
+    },
+    {
+      "type": "CREATE_ROOM",
+      "fragmentation": "all-at-once",
+      "chunk_sizes": [
+        63
+      ],
+      "observed_states": [
+        "message"
+      ],
+      "messages": [
+        {
+          "session_id": "session-fixture",
+          "type": "CREATE_ROOM",
+          "v": 1
+        }
+      ],
+      "parser_high_water_bytes": 63,
+      "result": "PASS"
+    },
+    {
+      "type": "JOIN_ROOM",
+      "fragmentation": "1/2/rest",
+      "chunk_sizes": [
+        1,
+        2,
+        83
+      ],
+      "observed_states": [
+        "need_more",
+        "need_more",
+        "message"
+      ],
+      "messages": [
+        {
+          "room_id": "room-fixture",
+          "session_id": "session-fixture",
+          "type": "JOIN_ROOM",
+          "v": 1
+        }
+      ],
+      "parser_high_water_bytes": 86,
+      "result": "PASS"
+    },
+    {
+      "type": "JOIN_ROOM",
+      "fragmentation": "header/payload",
+      "chunk_sizes": [
+        4,
+        82
+      ],
+      "observed_states": [
+        "need_more",
+        "message"
+      ],
+      "messages": [
+        {
+          "room_id": "room-fixture",
+          "session_id": "session-fixture",
+          "type": "JOIN_ROOM",
+          "v": 1
+        }
+      ],
+      "parser_high_water_bytes": 86,
+      "result": "PASS"
+    },
+    {
+      "type": "JOIN_ROOM",
+      "fragmentation": "all-at-once",
+      "chunk_sizes": [
+        86
+      ],
+      "observed_states": [
+        "message"
+      ],
+      "messages": [
+        {
+          "room_id": "room-fixture",
+          "session_id": "session-fixture",
+          "type": "JOIN_ROOM",
+          "v": 1
+        }
+      ],
+      "parser_high_water_bytes": 86,
+      "result": "PASS"
+    }
+  ],
+  "socket_fragmentation": [
+    {
+      "type": "HELLO",
+      "fragmentation": "1/2/rest",
+      "observed_partial_bytes": [
+        1,
+        3
+      ],
+      "response_type": "WELCOME",
+      "response_code": null,
+      "parser_buffered_after": 0
+    },
+    {
+      "type": "HELLO",
+      "fragmentation": "header/payload",
+      "observed_partial_bytes": [
+        4
+      ],
+      "response_type": "WELCOME",
+      "response_code": null,
+      "parser_buffered_after": 0
+    },
+    {
+      "type": "HELLO",
+      "fragmentation": "all-at-once",
+      "observed_partial_bytes": [],
+      "response_type": "WELCOME",
+      "response_code": null,
+      "parser_buffered_after": 0
+    },
+    {
+      "type": "CREATE_ROOM",
+      "fragmentation": "1/2/rest",
+      "observed_partial_bytes": [
+        1,
+        3
+      ],
+      "response_type": "ERROR",
+      "response_code": "SESSION_INVALID",
+      "parser_buffered_after": 0
+    },
+    {
+      "type": "CREATE_ROOM",
+      "fragmentation": "header/payload",
+      "observed_partial_bytes": [
+        4
+      ],
+      "response_type": "ERROR",
+      "response_code": "SESSION_INVALID",
+      "parser_buffered_after": 0
+    },
+    {
+      "type": "CREATE_ROOM",
+      "fragmentation": "all-at-once",
+      "observed_partial_bytes": [],
+      "response_type": "ERROR",
+      "response_code": "SESSION_INVALID",
+      "parser_buffered_after": 0
+    },
+    {
+      "type": "JOIN_ROOM",
+      "fragmentation": "1/2/rest",
+      "observed_partial_bytes": [
+        1,
+        3
+      ],
+      "response_type": "ERROR",
+      "response_code": "SESSION_INVALID",
+      "parser_buffered_after": 0
+    },
+    {
+      "type": "JOIN_ROOM",
+      "fragmentation": "header/payload",
+      "observed_partial_bytes": [
+        4
+      ],
+      "response_type": "ERROR",
+      "response_code": "SESSION_INVALID",
+      "parser_buffered_after": 0
+    },
+    {
+      "type": "JOIN_ROOM",
+      "fragmentation": "all-at-once",
+      "observed_partial_bytes": [],
+      "response_type": "ERROR",
+      "response_code": "SESSION_INVALID",
+      "parser_buffered_after": 0
+    }
+  ],
+  "coalescing": {
+    "messages": [
+      {
+        "type": "HELLO",
+        "v": 1
+      },
+      {
+        "session_id": "session-fixture",
+        "type": "CREATE_ROOM",
+        "v": 1
+      }
+    ],
+    "socket_write_calls": 1,
+    "socket_response_types": [
+      "WELCOME",
+      "ERROR"
+    ],
+    "socket_response_codes": [
+      null,
+      "SESSION_INVALID"
+    ]
+  },
+  "malformed": [
+    {
+      "name": "zero-length",
+      "parser_state": "terminal_frame_error",
+      "parser_code": "FRAME_SIZE_INVALID",
+      "wire_code": "FRAME_SIZE_INVALID",
+      "connection_effect": "closed",
+      "healthy_connection_survived": true,
+      "cleanup": {
+        "accepted_descriptor_closed": true,
+        "client_descriptor_closed": true,
+        "remaining_server_connections": 1,
+        "retained_peer_parser_bytes": 0
+      }
+    },
+    {
+      "name": "oversize-length",
+      "parser_state": "terminal_frame_error",
+      "parser_code": "FRAME_SIZE_INVALID",
+      "wire_code": "FRAME_SIZE_INVALID",
+      "connection_effect": "closed",
+      "healthy_connection_survived": true,
+      "cleanup": {
+        "accepted_descriptor_closed": true,
+        "client_descriptor_closed": true,
+        "remaining_server_connections": 1,
+        "retained_peer_parser_bytes": 0
+      }
+    },
+    {
+      "name": "duplicate-key",
+      "parser_state": "message_error",
+      "parser_code": "MESSAGE_INVALID",
+      "wire_code": "MESSAGE_INVALID",
+      "connection_effect": "open",
+      "healthy_connection_survived": true,
+      "cleanup": {
+        "accepted_descriptor_closed": true,
+        "client_descriptor_closed": true,
+        "remaining_server_connections": 1,
+        "retained_peer_parser_bytes": 0
+      }
+    },
+    {
+      "name": "invalid-utf8",
+      "parser_state": "message_error",
+      "parser_code": "MESSAGE_INVALID",
+      "wire_code": "MESSAGE_INVALID",
+      "connection_effect": "open",
+      "healthy_connection_survived": true,
+      "cleanup": {
+        "accepted_descriptor_closed": true,
+        "client_descriptor_closed": true,
+        "remaining_server_connections": 1,
+        "retained_peer_parser_bytes": 0
+      }
+    }
+  ],
+  "unit_protocol": [
+    {
+      "buffered_bytes": 0,
+      "code": "MESSAGE_INVALID",
+      "name": "root-array",
+      "next_state": "message",
+      "state": "message_error"
+    },
+    {
+      "buffered_bytes": 0,
+      "code": "MESSAGE_INVALID",
+      "name": "missing-v",
+      "next_state": "message",
+      "state": "message_error"
+    },
+    {
+      "buffered_bytes": 0,
+      "code": "MESSAGE_INVALID",
+      "name": "missing-type",
+      "next_state": "message",
+      "state": "message_error"
+    },
+    {
+      "buffered_bytes": 0,
+      "code": "MESSAGE_INVALID",
+      "name": "v-string",
+      "next_state": "message",
+      "state": "message_error"
+    },
+    {
+      "buffered_bytes": 0,
+      "code": "MESSAGE_INVALID",
+      "name": "v-floating",
+      "next_state": "message",
+      "state": "message_error"
+    },
+    {
+      "buffered_bytes": 0,
+      "code": "PROTOCOL_VERSION_UNSUPPORTED",
+      "name": "v-2",
+      "next_state": "message",
+      "state": "message_error"
+    },
+    {
+      "buffered_bytes": 0,
+      "code": "MESSAGE_TYPE_UNKNOWN",
+      "name": "unknown-type",
+      "next_state": "message",
+      "state": "message_error"
+    },
+    {
+      "buffered_bytes": 0,
+      "code": "",
+      "name": "known-unknown-field",
+      "next_state": "message",
+      "state": "message"
+    },
+    {
+      "buffered_bytes": 0,
+      "code": "MESSAGE_INVALID",
+      "name": "NaN",
+      "next_state": "message",
+      "state": "message_error"
+    },
+    {
+      "buffered_bytes": 0,
+      "code": "MESSAGE_INVALID",
+      "name": "Infinity",
+      "next_state": "message",
+      "state": "message_error"
+    },
+    {
+      "buffered_bytes": 0,
+      "code": "MESSAGE_INVALID",
+      "name": "trailing-object",
+      "next_state": "message",
+      "state": "message_error"
+    }
+  ],
+  "unit_bounds_and_transport_end": {
+    "G02_max_payload_bytes": 16384,
+    "clean_eof_code": "TRANSPORT_EOF",
+    "io_error_code": "TRANSPORT_IO_ERROR",
+    "parser_high_water_bytes": 16388,
+    "parser_storage_bytes": 16388,
+    "partial_eof": [
+      {
+        "buffered_after": 0,
+        "code": "TRANSPORT_EOF_IN_FRAME",
+        "partial_bytes": 3,
+        "state": "io_end"
+      },
+      {
+        "buffered_after": 0,
+        "code": "TRANSPORT_EOF_IN_FRAME",
+        "partial_bytes": 25,
+        "state": "io_end"
+      }
+    ],
+    "retained_bytes": 0
+  },
+  "metrics": {
+    "connection_high_water": 2,
+    "errors": {
+      "FRAME_SIZE_INVALID": 2,
+      "MESSAGE_INVALID": 2,
+      "SESSION_INVALID": 7
+    },
+    "input_per_player_high_water": 0,
+    "io_end_events": 2,
+    "mailbox_high_water": 2,
+    "max_read_bytes": 89,
+    "message_error_events": 2,
+    "need_more_events": 9,
+    "outbound_control_high_water": 2,
+    "parser_buffer_high_water": 86,
+    "parser_storage_bytes_per_connection": 16388,
+    "partial_eof_events": 0,
+    "partial_writes": 0,
+    "received_messages": 20,
+    "sent_messages": 22,
+    "terminal_frame_events": 2
+  },
+  "cleanup": {
+    "all_descriptors_closed": true,
+    "descriptor_checks": 12,
+    "disconnect_notifications": 0,
+    "mailbox_messages": 0,
+    "outbound_messages": 0,
+    "parser_buffered_bytes": 0,
+    "parser_storage_bytes": 0,
+    "pending_inputs": 0,
+    "server_connections": 0,
+    "server_descriptors": 0,
+    "timers": 0,
+    "tracked_descriptor_delta": 0,
+    "worker_threads": 0
+  },
+  "run_counts": {
+    "build": 6,
+    "unit": 3,
+    "integration": 1,
+    "canonical": 1
+  },
+  "network_fault_runs": 0,
+  "load_runs": 0,
+  "state_hashes": "INACTIVE_UNTIL_G07",
+  "source_evidence": "/private/tmp/game-server-systems-evolution-5a6e4a2f/evidence/G02/fundamentals-cpp/initial/G02-canonical.json"
+}
diff --git a/evidence/G02-runs.tsv b/evidence/G02-runs.tsv
new file mode 100644
index 0000000..8ce3229
--- /dev/null
+++ b/evidence/G02-runs.tsv
@@ -0,0 +1,11 @@
+category	units	label	exit	duration_seconds	argv_json	output
+build	1	reproduce-compile	0	6.998015	["/usr/bin/clang++", "-std=c++20", "-O2", "-Wall", "-Wextra", "-Wpedantic", "-Werror", "-I", "src", "-I", "/opt/homebrew/include", "/private/tmp/game-server-systems-evolution-5a6e4a2f/evidence/G02/fundamentals-cpp/initial/reproduce-g01-parser.cpp", "src/transport.cpp", "src/game.cpp", "-o", "/private/tmp/game-server-systems-evolution-5a6e4a2f/evidence/G02/fundamentals-cpp/initial/reproduce-g01-parser"]	/private/tmp/game-server-systems-evolution-5a6e4a2f/evidence/G02/fundamentals-cpp/initial/reproduce-compile/output.txt
+unit	1	reproduce-g01-unit	-6	0.494443	["/private/tmp/game-server-systems-evolution-5a6e4a2f/evidence/G02/fundamentals-cpp/initial/reproduce-g01-parser", "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G02.json", "/private/tmp/game-server-systems-evolution-5a6e4a2f/evidence/G02/fundamentals-cpp/initial/reproduction.json"]	/private/tmp/game-server-systems-evolution-5a6e4a2f/evidence/G02/fundamentals-cpp/initial/reproduce-g01-unit/output.txt
+build	1	reproduce-safe-compile	0	8.294723	["/usr/bin/clang++", "-std=c++20", "-O2", "-Wall", "-Wextra", "-Wpedantic", "-Werror", "-I", "src", "-I", "/opt/homebrew/include", "/private/tmp/game-server-systems-evolution-5a6e4a2f/evidence/G02/fundamentals-cpp/initial/reproduce-g01-parser-safe.cpp", "src/transport.cpp", "src/game.cpp", "-o", "/private/tmp/game-server-systems-evolution-5a6e4a2f/evidence/G02/fundamentals-cpp/initial/reproduce-g01-parser-safe"]	/private/tmp/game-server-systems-evolution-5a6e4a2f/evidence/G02/fundamentals-cpp/initial/reproduce-safe-compile/output.txt
+unit	1	reproduce-g01-unit-safe	1	0.571996	["/private/tmp/game-server-systems-evolution-5a6e4a2f/evidence/G02/fundamentals-cpp/initial/reproduce-g01-parser-safe", "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G02.json", "/private/tmp/game-server-systems-evolution-5a6e4a2f/evidence/G02/fundamentals-cpp/initial/reproduction-safe.json"]	/private/tmp/game-server-systems-evolution-5a6e4a2f/evidence/G02/fundamentals-cpp/initial/reproduce-g01-unit-safe/output.txt
+build	2	g02-tsan-build	0	17.247396	["env", "ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g02-tsan", "ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/evidence/G02/fundamentals-cpp/initial/track-logs", "ARENA_TSAN=ON", "./track", "build"]	/private/tmp/game-server-systems-evolution-5a6e4a2f/evidence/G02/fundamentals-cpp/initial/g02-tsan-build/output.txt
+build	1	g02-tsan-review-compile	0	18.025283	["cmake", "--build", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g02-tsan", "--parallel", "2"]	/private/tmp/game-server-systems-evolution-5a6e4a2f/evidence/G02/fundamentals-cpp/initial/g02-tsan-review-compile/output.txt
+build	1	g02-tsan-frozen-cases-compile	0	5.036045	["cmake", "--build", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g02-tsan", "--parallel", "2"]	/private/tmp/game-server-systems-evolution-5a6e4a2f/evidence/G02/fundamentals-cpp/initial/g02-tsan-frozen-cases-compile/output.txt
+unit	1	g02-tsan-unit	0	1.176592	["env", "ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g02-tsan", "ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/evidence/G02/fundamentals-cpp/initial/track-logs", "TSAN_OPTIONS=halt_on_error=1", "./track", "unit-test"]	/private/tmp/game-server-systems-evolution-5a6e4a2f/evidence/G02/fundamentals-cpp/initial/g02-tsan-unit/output.txt
+integration	1	g02-tsan-integration	0	1.112441	["env", "ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g02-tsan", "ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/evidence/G02/fundamentals-cpp/initial/track-logs", "TSAN_OPTIONS=halt_on_error=1", "./track", "integration-test"]	/private/tmp/game-server-systems-evolution-5a6e4a2f/evidence/G02/fundamentals-cpp/initial/g02-tsan-integration/output.txt
+canonical	1	g02-tsan-canonical	0	0.368727	["env", "ARENA_BUILD_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/fundamentals-cpp/build-g02-tsan", "ARENA_EVIDENCE_DIR=/private/tmp/game-server-systems-evolution-5a6e4a2f/evidence/G02/fundamentals-cpp/initial/track-logs", "TSAN_OPTIONS=halt_on_error=1", "./track", "scenario-run", "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G02.json", "/private/tmp/game-server-systems-evolution-5a6e4a2f/evidence/G02/fundamentals-cpp/initial/G02-canonical.json"]	/private/tmp/game-server-systems-evolution-5a6e4a2f/evidence/G02/fundamentals-cpp/initial/g02-tsan-canonical/output.txt
diff --git a/evidence/G02.md b/evidence/G02.md
new file mode 100644
index 0000000..653d6f3
--- /dev/null
+++ b/evidence/G02.md
@@ -0,0 +1,76 @@
+# G02 — TCP framing and parser state
+
+- THREAD: G02; BRANCH: `track/fundamentals-cpp`; PROFILE: `realtime-core`; ATTEMPT: initial.
+- SPEC_REVISION: `5a6e4a2f8fc71d4be18c3279583bfc2558d5c232`.
+- START: `79cbe0e54ffa8bf231c81a04b1136864dc9c1e78` (verified G01).
+- Frozen input: main `index/scenarios/G02.json`, SHA-256 `a1d103416b07e5fdb30d349e1938123851727bcaa33ac99baf72404505464692`.
+- Commit role: one G02 implementation commit with focused regressions and actual evidence; no dependency or game-rule change.
+
+## Reproduction before production changes
+
+An external helper compiled the unchanged G01 `src/transport.cpp` and `src/game.cpp`, then called the existing `decode_complete_frame` with bytes generated from the frozen scenario. It did not call an unsupported scenario mode or modify production to provoke a failure.
+
+The preserved successful reproduction execution exited **1**, reporting eight missing guarantees: six fragmented message cases terminated with `FRAME_SIZE_INVALID`; the concatenated pair terminated with the same code; duplicate JSON keys were accepted. All three complete single-frame cases passed. Invalid length and invalid UTF-8 rejection already worked; those individual properties were not reproduced as defects.
+
+The first helper execution instead aborted with **SIGABRT (-6; wrapper exit 250)** while serializing a JSON library diagnostic containing raw invalid UTF-8. This was a helper evidence-output failure, not a server crash. Its source, empty partial evidence, command and output remain preserved. Only diagnostic serialization was changed to replace invalid text; the same input bytes and unchanged G01 production sources were used again. Both compiles and both unit executions are charged below.
+
+Immutable evidence root:
+
+```text
+/private/tmp/game-server-systems-evolution-5a6e4a2f/evidence/G02/fundamentals-cpp/initial/
+```
+
+Relevant files there are `reproduce-g01-unit/output.txt` (helper failure), `reproduce-g01-unit-safe/output.txt`, `reproduction-safe.json`, and both `reproduce-g01-parser*.cpp` sources/binaries. `verify-pre-change/output.txt` records clean G01 status and the unchanged scenario hash before compilation.
+
+## Fix
+
+Each Connection owns one fixed 16,388-byte parser array. `consume` returns one result and its consumed-byte count, preserving a coalesced suffix without retaining an unbounded list. Partial headers and bodies remain buffered. Invalid lengths are rejected as soon as four header bytes arrive, before any peer-sized allocation.
+
+Strict object/envelope/request decoding rejects duplicates, invalid UTF-8, missing or incorrectly typed fields, unsupported versions, unknown active message types, NaN, Infinity and trailing objects. Unknown fields remain compatible. Recoverable errors use the bounded owner mailbox in stream order; a later frame on that connection can succeed. Framing errors attempt one bounded nonblocking error flush, then close only that peer.
+
+Source review also identified the frozen JSON lexer's raw-NUL end sentinel and a possible double classification when error flushing fails. The production payload guard rejects raw NUL before JSON parsing; escaped `\u0000` remains legal. I/O termination reports `TRANSPORT_IO_ERROR` independently of a prior framing failure. These guards were reviewed statically, without changing the fixed corpus.
+
+An intermediate compilation included proposed extra pure checks for those guards. No test was executed from that version. Main required the original supplemental list; the unexecuted source was saved as `unexecuted-review-extra-tests.cpp` with an explanatory note, and the extra checks were removed before the final test compilation. The original G01 assertions and fixed G02 supplemental cases remain unchanged.
+
+## Verification and exact budget
+
+`G02-runs.tsv` contains every build/test argv, exit, duration and immutable output path. The external `commands.jsonl` also records inspection, artifact, git and failed shell commands, including one harmless quoted-glob correction before reproduction. No failed command was hidden or overwritten.
+
+| Execution | Exit | Duration |
+|---|---:|---:|
+| Original reproduction compile | 0 | 6.998015s |
+| Original reproduction helper output failure | -6 | 0.494443s |
+| Safe-diagnostic reproduction compile | 0 | 8.294723s |
+| Actual G01 reproduction | 1 (expected failure) | 0.571996s |
+| `ARENA_TSAN=ON ./track build` (configure + compile) | 0 | 17.247396s |
+| Reviewed production guards / unexecuted checks compile | 0 | 18.025283s |
+| Fixed-list final test compile | 0 | 5.036045s |
+| `TSAN_OPTIONS=halt_on_error=1 ./track unit-test` | 0 | 1.176592s |
+| `TSAN_OPTIONS=halt_on_error=1 ./track integration-test` | 0 | 1.112441s |
+| `TSAN_OPTIONS=halt_on_error=1 ./track scenario-run <frozen G02.json> <new output>` | 0 | 0.368727s |
+
+The final executions select `build-g02-tsan` with `ARENA_BUILD_DIR`; exact absolute environment/argv values are in the TSV. Integration and canonical execution used explicitly permitted loopback binding. The usual frozen compiler, CMake, JSON dependency and ordinary sanitizer configuration were retained.
+
+Conservative counts: **6/8 configure/compile**, **3/4 unit** including both reproduction helpers, **1/2 integration**, and **1 post-change canonical**. No redundant pre-change canonical command was run. Network-fault runs: **0**. Load runs: **0**.
+
+The unit suite passed **9/9**, including all seven unchanged G01 tests, the fixed protocol supplemental list and maximum/EOF checks. Integration passed **2/2**, preserving real G01 authority and standalone HELLO/WELCOME, SIGTERM, process reaping and listener rebind assertions. ThreadSanitizer reported no error in the executed suites/scenario.
+
+## Observed canonical evidence
+
+`G02-canonical-summary.json` is selected from actual `G02-canonical.json` and unit output, not a hand-built expected output. The complete canonical file, exact source/binary hashes and the frozen input hash are recorded in `verification-manifest.json` under the external evidence root.
+
+- All **9** parser fragmentation rows passed with unchanged logical messages. The same **9** real TCP rows observed partial-buffer sizes before sending each next chunk. HELLO returned WELCOME; fixture CREATE/JOIN session IDs reached the existing `SESSION_INVALID` check after valid parsing.
+- The actual concatenated HELLO/CREATE pair used **one socket write**. Parser and wire response order were preserved.
+- Length 0 and 16,385: `FRAME_SIZE_INVALID`, complete error observed, peer closed.
+- Duplicate key and invalid UTF-8: `MESSAGE_INVALID`, peer stayed open and a subsequent HELLO succeeded.
+- A separate persistent healthy connection returned WELCOME with the same session after each malformed peer. No process termination occurred.
+- Canonical parser high water: **86/16,388 bytes**; read high water: **89/16,388 bytes**. The pure maximum-size test independently reached **16,388**, accepting exactly **16,384 payload bytes**.
+- Canonical metrics: **9** need-more events, **2** recoverable message errors, **2** terminal framing errors, connection high water **2/512**, mailbox high water **2/512**, outbound high water **2/64**.
+- Partial header (3 bytes) and partial body (25 bytes) EOF were pure-unit `io_end / TRANSPORT_EOF_IN_FRAME`; clean EOF was `TRANSPORT_EOF`; failed I/O was `TRANSPORT_IO_ERROR`. Retained bytes were zero. No additional malformed socket corpus was used for EOF.
+- All **12** checked actual client/server/listener/reactor descriptors were closed. Final server connections, descriptors, parser buffers/storage, mailbox, outbound, pending input, disconnect notifications and tracked descriptor delta were **0**. No server worker thread or timer was allocated.
+
+## Remaining scope
+
+STATE_HASHES: inactive until G07. NETWORK_FAULT_RUNS: 0. LOAD_RUNS: 0.
+
+UNRESOLVED: no known G02 completion failure. Independent main verification, cross-track comparison and progress tagging remain main's responsibility. G03 identity/lifecycle expansion, sequence/target tick, replay, UDP, reconnect and multi-room features remain inactive.
diff --git a/src/scenario.cpp b/src/scenario.cpp
index a074a8e..ea8e94e 100644
--- a/src/scenario.cpp
+++ b/src/scenario.cpp
@@ -1,6 +1,10 @@
 #include "scenario.hpp"
+#include <algorithm>
+#include <chrono>
 #include <fstream>
+#include <iomanip>
 #include <memory>
+#include <sstream>
 #include <stdexcept>
 
 namespace arena {
@@ -46,7 +50,238 @@ void write_json_file(const std::filesystem::path& path, const Json& value) {
   file << text << '\n';
   if (!file) throw std::runtime_error("evidence write failed");
 }
+namespace {
+std::string bytes_hex(std::span<const std::uint8_t> bytes) {
+  std::ostringstream out;
+  for (const auto byte : bytes) out << std::hex << std::setfill('0') << std::setw(2) << static_cast<unsigned>(byte);
+  return out.str();
+}
+std::vector<std::size_t> fragment_sizes(const std::string& split, std::size_t total) {
+  if (split == "1/2/rest") return {1, 2, total - 3};
+  if (split == "header/payload") return {4, total - 4};
+  if (split == "all-at-once") return {total};
+  throw std::runtime_error("unknown fixed fragmentation");
+}
+Json parse_observation(const ParseResult& result, const FrameParser& parser) {
+  Json value{{"state", parse_state_name(result.state)}, {"consumed", result.consumed},
+    {"buffered_bytes", parser.buffered_bytes()}, {"partial_frame", result.partial_frame}};
+  if (!result.code.empty()) value["code"] = result.code;
+  if (result.state == ParseState::message) value["message"] = result.value;
+  return value;
+}
+std::vector<std::uint8_t> malformed_frame(const Json& item) {
+  std::vector<std::uint8_t> payload;
+  if (item.contains("payload_text")) {
+    const auto text = item.at("payload_text").get<std::string>();
+    payload.assign(text.begin(), text.end());
+  } else {
+    const auto hex = item.at("payload_hex").get<std::string>();
+    require(hex.size() % 2 == 0 && hex.size() <= max_frame_bytes * 2, "invalid bounded fixture hex");
+    for (std::size_t index = 0; index < hex.size(); index += 2) {
+      std::size_t parsed = 0;
+      const auto byte = std::stoul(hex.substr(index, 2), &parsed, 16);
+      require(parsed == 2 && byte <= 255, "invalid fixture hex byte");
+      payload.push_back(static_cast<std::uint8_t>(byte));
+    }
+  }
+  const auto length = item.value("declared_length", static_cast<std::uint32_t>(payload.size()));
+  require(payload.size() <= max_frame_bytes, "fixture payload exceeds fixed bound");
+  std::vector<std::uint8_t> frame{static_cast<std::uint8_t>(length >> 24U), static_cast<std::uint8_t>(length >> 16U),
+    static_cast<std::uint8_t>(length >> 8U), static_cast<std::uint8_t>(length)};
+  frame.insert(frame.end(), payload.begin(), payload.end());
+  return frame;
+}
+Json run_framing_scenario(const Json& scenario) {
+  require(scenario.at("contract_version") == 1, "G02 requires protocol v1");
+  require(scenario.at("messages").size() == 3 && scenario.at("fragmentations").size() == 3 &&
+          scenario.at("malformed_cases").size() == 4 && scenario.at("coalesce_message_indices").size() == 2,
+          "G02 requires the frozen matrix dimensions");
+  const auto bound = scenario.at("parser_bound_bytes").get<std::size_t>();
+  const int ceiling = scenario.at("execution_ceiling_ms_per_socket_case").get<int>();
+  require(bound == FrameParser::storage_bytes && ceiling == 5000, "G02 fixed bound/deadline changed");
+  Json evidence{{"scenario_id", scenario.at("scenario_id")}, {"thread", "G02"}, {"contract_version", 1},
+    {"transport", "production-FrameParser/real-loopback-TCP/kqueue"}, {"parser_bound_bytes", bound},
+    {"fragmentation_matrix", Json::array()}, {"socket_fragmentation", Json::array()},
+    {"malformed", Json::array()}, {"executed_ticks", 0}, {"state_hashes", "INACTIVE_UNTIL_G07"}};
+  std::size_t parser_high_water = 0;
+  for (const auto& value : scenario.at("messages")) {
+    const auto frame = encode_frame(value);
+    for (const auto& split_value : scenario.at("fragmentations")) {
+      const auto split = split_value.get<std::string>();
+      const auto chunks = fragment_sizes(split, frame.size());
+      FrameParser parser;
+      Json row{{"type", value.at("type")}, {"fragmentation", split}, {"frame_hex", bytes_hex(frame)},
+        {"chunk_sizes", chunks}, {"reads", Json::array()}, {"messages", Json::array()}};
+      std::size_t offset = 0;
+      for (const auto size : chunks) {
+        const auto result = parser.consume(std::span(frame).subspan(offset, size));
+        offset += size;
+        require(result.consumed == size, "parser did not consume the fixed chunk");
+        require(result.state == (offset == frame.size() ? ParseState::message : ParseState::need_more),
+                "fragmentation changed complete/need-more state");
+        row["reads"].push_back(parse_observation(result, parser));
+        if (result.state == ParseState::message) row["messages"].push_back(result.value);
+      }
+      require(row.at("messages") == Json::array({value}) && parser.buffered_bytes() == 0, "fragmentation changed message");
+      require(parser.high_water_bytes() <= bound, "parser bound exceeded by fragmentation");
+      row["parser_high_water_bytes"] = parser.high_water_bytes();
+      row["result"] = "PASS";
+      parser_high_water = std::max(parser_high_water, parser.high_water_bytes());
+      evidence["fragmentation_matrix"].push_back(row);
+    }
+  }
+  std::vector<std::uint8_t> pair;
+  Json pair_messages = Json::array();
+  for (const auto& index : scenario.at("coalesce_message_indices")) {
+    const auto& value = scenario.at("messages").at(index.get<std::size_t>());
+    const auto frame = encode_frame(value);
+    pair.insert(pair.end(), frame.begin(), frame.end()); pair_messages.push_back(value);
+  }
+  FrameParser combined_parser;
+  auto remaining = std::span<const std::uint8_t>(pair);
+  Json combined{{"bytes_hex", bytes_hex(pair)}, {"messages", Json::array()}, {"reads", Json::array()}};
+  while (!remaining.empty()) {
+    const auto result = combined_parser.consume(remaining);
+    require(result.state == ParseState::message && result.consumed > 0, "coalesced message was not decoded");
+    remaining = remaining.subspan(result.consumed);
+    combined["messages"].push_back(result.value);
+    combined["reads"].push_back(parse_observation(result, combined_parser));
+  }
+  require(combined.at("messages") == pair_messages && combined_parser.buffered_bytes() == 0, "coalescing lost order or suffix");
+  combined["parser_high_water_bytes"] = combined_parser.high_water_bytes();
+  require(combined_parser.high_water_bytes() <= bound, "coalescing exceeded parser bound");
+  parser_high_water = std::max(parser_high_water, combined_parser.high_water_bytes());
+  evidence["coalescing"] = combined;
+
+  const int descriptors_before = Fd::live();
+  ManualClock clock;
+  Server server(clock);
+  TcpClient healthy(server.port());
+  require(scenario.at("messages").at(0).at("type") == "HELLO", "first frozen message must be HELLO");
+  const auto hello = scenario.at("messages").at(0);
+  healthy.send(hello);
+  const auto welcome = healthy.receive_type(server, "WELCOME");
+  const auto healthy_session = welcome.at("session_id");
+  // Force every partial read to reach the production server before the next
+  // chunk is written; this does not rely on the kernel preserving write size.
+  for (const auto& value : scenario.at("messages")) {
+    const auto frame = encode_frame(value);
+    for (const auto& split_value : scenario.at("fragmentations")) {
+      const auto split = split_value.get<std::string>();
+      const auto deadline = std::chrono::steady_clock::now() + std::chrono::milliseconds(ceiling);
+      Json row{{"type", value.at("type")}, {"fragmentation", split}, {"observed_partial_bytes", Json::array()}};
+      std::size_t offset = 0;
+      for (const auto size : fragment_sizes(split, frame.size())) {
+        healthy.send_bytes(std::span(frame).subspan(offset, size));
+        offset += size;
+        if (offset == frame.size()) break;
+        while (server.cleanup().at("parser_buffered_bytes") != offset && std::chrono::steady_clock::now() < deadline)
+          server.pump(1);
+        require(server.cleanup().at("parser_buffered_bytes") == offset, "server did not retain the fixed partial chunk");
+        require(!healthy.try_receive(), "server replied before the complete frame");
+        row["observed_partial_bytes"].push_back(server.cleanup().at("parser_buffered_bytes"));
+      }
+      const auto response = healthy.receive(server);
+      if (value.at("type") == "HELLO")
+        require(response.at("type") == "WELCOME" && response.at("session_id") == healthy_session, "fragmented HELLO failed");
+      else
+        require(response.at("type") == "ERROR" && response.at("code") == "SESSION_INVALID",
+                "fixture identity should reach existing session validation after parsing");
+      require(std::chrono::steady_clock::now() < deadline, "fragmentation socket deadline exceeded");
+      row["response"] = response;
+      row["parser_buffered_after"] = server.cleanup().at("parser_buffered_bytes");
+      evidence["socket_fragmentation"].push_back(row);
+    }
+  }
+  require(pair_messages.at(0).at("type") == "HELLO" && pair_messages.at(1).at("type") == "CREATE_ROOM",
+          "frozen coalescing pair changed");
+  const auto writes = healthy.send_bytes(pair);
+  require(writes == 1, "frozen concatenated pair must be sent in one write");
+  const auto first_response = healthy.receive(server), second_response = healthy.receive(server);
+  require(first_response.at("type") == "WELCOME" && second_response.at("type") == "ERROR" &&
+          second_response.at("code") == "SESSION_INVALID", "wire coalescing response order changed");
+  evidence["coalescing"]["socket_write_calls"] = writes;
+  evidence["coalescing"]["socket_responses"] = Json::array({first_response, second_response});
+  std::size_t descriptor_checks = 0;
+  for (const auto& item : scenario.at("malformed_cases")) {
+    const auto frame = malformed_frame(item);
+    const bool terminal = item.contains("declared_length");
+    const auto expected_state = terminal ? ParseState::terminal_frame_error : ParseState::message_error;
+    const std::string expected_code = terminal ? "FRAME_SIZE_INVALID" : "MESSAGE_INVALID";
+    FrameParser parser;
+    const auto parsed = parser.consume(frame);
+    require(parsed.state == expected_state && parsed.code == expected_code && parser.buffered_bytes() == 0,
+            "malformed fixture has incorrect parser classification");
+    require(parser.high_water_bytes() <= bound, "malformed input exceeded parser bound");
+    Json row{{"name", item.at("name")}, {"frame_hex", bytes_hex(frame)}, {"parser", parse_observation(parsed, parser)},
+      {"parser_high_water_bytes", parser.high_water_bytes()}};
+    const auto before = server.owned_descriptors();
+    const auto deadline = std::chrono::steady_clock::now() + std::chrono::milliseconds(ceiling);
+    TcpClient malformed(server.port());
+    const int client_fd = malformed.descriptor();
+    while (server.cleanup().at("server_connections") != 2 && std::chrono::steady_clock::now() < deadline) server.pump(1);
+    require(server.cleanup().at("server_connections") == 2, "malformed probe was not accepted before deadline");
+    int accepted_fd = -1;
+    for (const auto fd : server.owned_descriptors())
+      if (std::find(before.begin(), before.end(), fd) == before.end()) accepted_fd = fd;
+    require(accepted_fd >= 0, "malformed connection descriptor not observed");
+    malformed.send_bytes(frame);
+    const auto error = malformed.receive_type(server, "ERROR");
+    require(error.at("code") == expected_code && error.at("message").get<std::string>().size() <= 160,
+            "malformed wire error is missing or unbounded");
+    row["wire_error"] = error;
+    if (terminal) {
+      while (!malformed.peer_closed() && std::chrono::steady_clock::now() < deadline) server.pump(1);
+      require(malformed.peer_closed(), "terminal framing error did not close the peer");
+      row["connection_effect"] = "closed";
+    } else {
+      malformed.send(hello);
+      const auto recovered = malformed.receive_type(server, "WELCOME");
+      require(recovered.contains("session_id") && !malformed.peer_closed(), "recoverable error lost the connection");
+      row["connection_effect"] = "open";
+      row["recovery_response"] = recovered;
+    }
+    malformed.close();
+    while (server.cleanup().at("server_connections") != 1 && std::chrono::steady_clock::now() < deadline) server.pump(1);
+    require(server.cleanup().at("server_connections") == 1 && descriptor_closed(client_fd) && descriptor_closed(accepted_fd),
+            "malformed client/accepted socket not cleaned up");
+    descriptor_checks += 2;
+    require(server.cleanup().at("parser_storage_bytes") == bound && server.cleanup().at("parser_buffered_bytes") == 0,
+            "malformed peer retained parser storage");
+    healthy.send(hello);
+    const auto survivor = healthy.receive_type(server, "WELCOME");
+    require(survivor.at("session_id") == healthy_session, "healthy connection lost its identity or server progress");
+    require(std::chrono::steady_clock::now() < deadline, "malformed socket case exceeded fixed ceiling");
+    row["healthy_connection_survived"] = true;
+    row["healthy_response"] = survivor;
+    row["cleanup"] = Json{{"client_descriptor_closed", descriptor_closed(client_fd)},
+      {"accepted_descriptor_closed", descriptor_closed(accepted_fd)},
+      {"retained_peer_parser_bytes", server.cleanup().at("parser_storage_bytes").get<std::size_t>() - bound},
+      {"remaining_server_connections", server.cleanup().at("server_connections")}};
+    evidence["malformed"].push_back(row);
+  }
+  std::vector<int> closed_fds = server.owned_descriptors();
+  closed_fds.push_back(healthy.descriptor());
+  server.shutdown(); healthy.close();
+  bool all_closed = true;
+  for (const auto fd : closed_fds) all_closed = all_closed && descriptor_closed(fd);
+  require(all_closed && Fd::live() == descriptors_before, "G02 descriptor ownership leaked");
+  const auto cleanup = server.cleanup();
+  for (const auto& [key, count] : cleanup.items()) { (void)key; require(count == 0, "G02 server resource retained"); }
+  evidence["cleanup"] = cleanup;
+  evidence["cleanup"]["descriptor_checks"] = descriptor_checks + closed_fds.size();
+  evidence["cleanup"]["all_descriptors_closed"] = all_closed;
+  evidence["cleanup"]["tracked_descriptor_delta"] = Fd::live() - descriptors_before;
+  evidence["metrics"] = server.metrics();
+  require(server.metrics().at("parser_buffer_high_water").get<std::size_t>() <= bound, "production socket parser exceeded bound");
+  require(server.metrics().at("connection_high_water") == 2, "healthy/malformed isolation connection bound changed");
+  evidence["parser_matrix_high_water_bytes"] = parser_high_water;
+  evidence["result"] = "PASS";
+  return evidence;
+}
+}
 Json run_scenario(const Json& scenario) {
+  if (scenario.at("thread") == "G02") return run_framing_scenario(scenario);
   require(scenario.at("contract_version") == 1 && scenario.at("thread") == "G01", "only G01 contract v1 is active");
   require(scenario.at("clock").at("kind") == "manual" && scenario.at("clock").at("tick_duration_ms") == tick_duration_ms,
           "G01 runner requires the fixed 50ms manual clock");
diff --git a/src/transport.cpp b/src/transport.cpp
index a71f18c..3eb23b8 100644
--- a/src/transport.cpp
+++ b/src/transport.cpp
@@ -38,6 +38,45 @@ std::uint32_t payload_size(const std::uint8_t* data) {
          (static_cast<std::uint32_t>(data[2]) << 8U) | static_cast<std::uint32_t>(data[3]);
 }
 bool transient_io() { return errno == EAGAIN || errno == EWOULDBLOCK || errno == EINTR; }
+Json parse_object(std::span<const std::uint8_t> payload) {
+  // nlohmann 3.12's lexer treats raw NUL as end-of-input. A framed payload
+  // cannot have an early terminator that hides unchecked trailing bytes.
+  if (std::find(payload.begin(), payload.end(), std::uint8_t{0}) != payload.end())
+    throw std::invalid_argument("raw NUL in JSON payload");
+  std::vector<std::set<std::string>> object_keys;
+  const auto callback = [&](int, Json::parse_event_t event, Json& value) {
+    if (event == Json::parse_event_t::object_start) object_keys.emplace_back();
+    if (event == Json::parse_event_t::key && !object_keys.back().insert(value.get<std::string>()).second)
+      throw std::invalid_argument("duplicate JSON key");
+    if (event == Json::parse_event_t::object_end) object_keys.pop_back();
+    return true;
+  };
+  // The frozen JSON parser rejects invalid UTF-8, non-JSON numbers and trailing
+  // input. The callback rejects duplicates before a DOM can overwrite a key.
+  auto value = Json::parse(payload.begin(), payload.end(), callback);
+  if (!value.is_object()) throw std::invalid_argument("JSON root must be an object");
+  return value;
+}
+std::string request_error(const Json& value) {
+  if (!value.contains("v") || !value.at("v").is_number_integer() ||
+      !value.contains("type") || !value.at("type").is_string()) return "MESSAGE_INVALID";
+  if (value.at("v") != 1) return "PROTOCOL_VERSION_UNSUPPORTED";
+  const auto type = value.at("type").get<std::string>();
+  if (type == "HELLO") return {};
+  if (type != "CREATE_ROOM" && type != "JOIN_ROOM" && type != "LEAVE_ROOM" && type != "INPUT")
+    return "MESSAGE_TYPE_UNKNOWN";
+  const auto string_field = [&](const char* name) { return value.contains(name) && value.at(name).is_string(); };
+  if (!string_field("session_id")) return "MESSAGE_INVALID";
+  if (type == "CREATE_ROOM") return {};
+  if (!string_field("room_id")) return "MESSAGE_INVALID";
+  if (type == "JOIN_ROOM") return {};
+  if (!string_field("player_id")) return "MESSAGE_INVALID";
+  if (type == "LEAVE_ROOM") return {};
+  if (!string_field("direction") || !value.contains("tag_target_player_id")) return "MESSAGE_INVALID";
+  const auto& target = value.at("tag_target_player_id");
+  if (!target.is_null() && !target.is_string()) return "MESSAGE_INVALID";
+  return {};
+}
 }
 std::atomic<int> Fd::live_{0};
 Fd::Fd(int value) : value_(value) { if (value_ >= 0) ++live_; }
@@ -61,13 +100,64 @@ std::vector<std::uint8_t> encode_frame(const Json& value) {
   return bytes;
 }
 Json decode_complete_frame(std::span<const std::uint8_t> bytes) {
-  if (bytes.size() < 4) throw std::length_error("G01 requires a complete header in one read");
+  if (bytes.size() < 4) throw std::length_error("complete header required");
   const std::uint32_t size = payload_size(bytes.data());
   if (size == 0 || size > max_frame_bytes || bytes.size() != size + 4U)
-    throw std::length_error("G01 requires exactly one bounded complete frame in one read");
-  auto value = Json::parse(bytes.begin() + 4, bytes.end());
-  if (!value.is_object()) throw std::invalid_argument("JSON root must be an object");
-  return value;
+    throw std::length_error("exactly one bounded complete frame required");
+  return parse_object(bytes.subspan(4));
+}
+std::string parse_state_name(ParseState state) {
+  switch (state) {
+    case ParseState::need_more: return "need_more";
+    case ParseState::message: return "message";
+    case ParseState::message_error: return "message_error";
+    case ParseState::terminal_frame_error: return "terminal_frame_error";
+    case ParseState::io_end: return "io_end";
+  }
+  throw std::logic_error("invalid parser state");
+}
+ParseResult FrameParser::consume(std::span<const std::uint8_t> input) {
+  if (terminal_) { auto result = *terminal_; result.consumed = 0; return result; }
+  std::size_t consumed = 0;
+  while (!input.empty()) {
+    const auto count = std::min(expected_ - used_, input.size());
+    std::copy_n(input.begin(), count, bytes_.begin() + static_cast<std::ptrdiff_t>(used_));
+    used_ += count; consumed += count; input = input.subspan(count);
+    high_water_ = std::max(high_water_, used_);
+    if (used_ < expected_) break;
+    if (expected_ == 4) {
+      const auto size = payload_size(bytes_.data());
+      if (size == 0 || size > max_frame_bytes) {
+        used_ = 0;
+        terminal_ = ParseResult{ParseState::terminal_frame_error, consumed, {}, "FRAME_SIZE_INVALID", false};
+        return *terminal_;
+      }
+      expected_ = static_cast<std::size_t>(size) + 4;
+      continue;
+    }
+    const auto complete_size = expected_;
+    used_ = 0; expected_ = 4;
+    try {
+      auto value = parse_object(std::span(bytes_).subspan(4, complete_size - 4));
+      const auto code = request_error(value);
+      if (!code.empty()) return {ParseState::message_error, consumed, {}, code, false};
+      return {ParseState::message, consumed, std::move(value), {}, false};
+    } catch (const Json::exception&) {
+      // Never serialize raw exception text: invalid UTF-8 may be present in it.
+      return {ParseState::message_error, consumed, {}, "MESSAGE_INVALID", false};
+    } catch (const std::invalid_argument&) {
+      return {ParseState::message_error, consumed, {}, "MESSAGE_INVALID", false};
+    }
+  }
+  return {ParseState::need_more, consumed, {}, {}, false};
+}
+ParseResult FrameParser::finish(bool io_error) {
+  if (terminal_ && !io_error) { auto result = *terminal_; result.consumed = 0; return result; }
+  const bool partial = used_ != 0 || (terminal_ && terminal_->partial_frame);
+  used_ = 0; expected_ = 4;
+  terminal_ = ParseResult{ParseState::io_end, 0, {},
+    io_error ? "TRANSPORT_IO_ERROR" : partial ? "TRANSPORT_EOF_IN_FRAME" : "TRANSPORT_EOF", partial};
+  return *terminal_;
 }
 void PendingWrite::consume(std::size_t count) {
   if (count > remaining().size()) throw std::logic_error("write offset exceeds owned buffer");
@@ -124,7 +214,7 @@ void Server::accept_ready() {
     socket_options(fd.get());
     const int raw = fd.get();
     const auto id = next_connection_++;
-    connections_.emplace(raw, Connection{std::move(fd), id, {}, {}, {}, 0});
+    connections_.emplace(raw, Connection{std::move(fd), id, {}, {}, {}, 0, {}});
     register_event(raw, EVFILT_READ, EV_ADD, id);
     register_event(raw, EVFILT_WRITE, EV_ADD | EV_DISABLE, id);
     connection_high_water_ = std::max(connection_high_water_, connections_.size());
@@ -139,32 +229,51 @@ void Server::disconnect(int fd, const std::string& reason) {
   // in udata prevents old events from touching a subsequently reused fd.
   connections_.erase(found);
 }
+void Server::end_transport(int fd, bool io_error) {
+  const auto found = connections_.find(fd);
+  if (found == connections_.end()) return;
+  const auto end = found->second.parser.finish(io_error);
+  ++io_end_events_;
+  if (!io_error && end.partial_frame) ++partial_eof_events_;
+  disconnect(fd, io_error ? "TRANSPORT_IO_ERROR" : end.partial_frame ? end.code : "");
+}
 void Server::read_ready(int fd) {
   auto found = connections_.find(fd);
   if (found == connections_.end()) return;
   std::array<std::uint8_t, max_frame_bytes + 4> bytes{};
   const auto received = ::recv(fd, bytes.data(), bytes.size(), 0);
-  if (received == 0) { disconnect(fd, ""); return; }
-  if (received < 0) { if (!transient_io()) disconnect(fd, "TRANSPORT_IO_ERROR"); return; }
+  if (received == 0) { end_transport(fd, false); return; }
+  if (received < 0) { if (!transient_io()) end_transport(fd, true); return; }
   const auto count = static_cast<std::size_t>(received);
   max_read_bytes_ = std::max(max_read_bytes_, count);
-  try {
-    Json value = decode_complete_frame(std::span(bytes).first(count));
+  auto remaining = std::span<const std::uint8_t>(bytes).first(count);
+  while (!remaining.empty()) {
+    found = connections_.find(fd);
+    if (found == connections_.end()) return;
+    auto parsed = found->second.parser.consume(remaining);
+    parser_high_water_ = std::max(parser_high_water_, found->second.parser.high_water_bytes());
+    remaining = remaining.subspan(parsed.consumed);
+    if (parsed.state == ParseState::need_more) { ++need_more_events_; return; }
+    if (parsed.state == ParseState::terminal_frame_error) {
+      ++terminal_frame_events_; ++errors_[parsed.code];
+      const auto id = found->second.id;
+      queue(id, error_message(parsed.code, "payload length must be from 1 to 16384 bytes"));
+      // One bounded nonblocking flush attempt, then close even if the peer
+      // cannot receive a complete error frame. No closing peer is retained.
+      write_ready(fd);
+      disconnect(fd, "");
+      return;
+    }
+    if (parsed.state == ParseState::io_end) return;
+    if (parsed.state == ParseState::message_error) ++message_error_events_;
     if (mailbox_.size() == max_mailbox_messages || found->second.pending_requests == max_pending_inputs) {
       queue(found->second.id, error_message("INPUT_QUEUE_FULL", "bounded transport mailbox is full"));
-      return;
+      continue;
     }
     ++found->second.pending_requests;
-    mailbox_.push_back({found->second.id, std::move(value)});
+    mailbox_.push_back({found->second.id, std::move(parsed.value), std::move(parsed.code)});
     mailbox_high_water_ = std::max(mailbox_high_water_, mailbox_.size());
     ++received_messages_;
-  } catch (const std::length_error&) {
-    // G01 has no partial-frame storage. G02 will replace this assumption.
-    disconnect(fd, "FRAME_SIZE_INVALID");
-  } catch (const Json::exception&) {
-    queue(found->second.id, error_message("MESSAGE_INVALID", "invalid JSON message"));
-  } catch (const std::invalid_argument&) {
-    queue(found->second.id, error_message("MESSAGE_INVALID", "JSON root must be an object"));
   }
 }
 void Server::write_ready(int fd) {
@@ -176,7 +285,7 @@ void Server::write_ready(int fd) {
     auto& write = writes.front();
     const auto bytes = write.remaining();
     const auto sent = ::send(fd, bytes.data(), bytes.size(), 0);
-    if (sent < 0) { if (!transient_io()) disconnect(fd, "TRANSPORT_IO_ERROR"); return; }
+    if (sent < 0) { if (!transient_io()) end_transport(fd, true); return; }
     if (sent == 0) return;
     if (static_cast<std::size_t>(sent) < bytes.size()) ++partial_writes_;
     write.consume(static_cast<std::size_t>(sent));
@@ -196,7 +305,7 @@ void Server::poll_io(int timeout_ms) {
     if (fd == listener_.get()) { if (!stopping_) accept_ready(); continue; }
     const auto found = connections_.find(fd);
     if (found == connections_.end() || found->second.id != reinterpret_cast<uintptr_t>(event.udata)) continue;
-    if (event.flags & EV_ERROR) { disconnect(fd, "TRANSPORT_IO_ERROR"); continue; }
+    if (event.flags & EV_ERROR) { end_transport(fd, true); continue; }
     if (event.filter == EVFILT_READ) {
       if (!stopping_) read_ready(fd);
     } else if (event.filter == EVFILT_WRITE) {
@@ -229,6 +338,9 @@ void Server::handle(const Envelope& envelope) {
   auto reject = [&](const std::string& code, const std::string& text) {
     ++errors_[code]; queue(id, error_message(code, text));
   };
+  if (!envelope.parser_error.empty()) {
+    reject(envelope.parser_error, "invalid framed request"); return;
+  }
   try {
     if (!value.contains("v") || !value.at("v").is_number_integer() || !value.contains("type") || !value.at("type").is_string()) {
       reject("MESSAGE_INVALID", "v integer and type string required"); return;
@@ -323,13 +435,20 @@ Json Server::metrics() const {
   return Json{{"received_messages", received_messages_}, {"sent_messages", sent_messages_},
     {"mailbox_high_water", mailbox_high_water_}, {"outbound_control_high_water", outbound_high_water_},
     {"connection_high_water", connection_high_water_}, {"input_per_player_high_water", room_.input_high_water()},
-    {"max_read_bytes", max_read_bytes_}, {"partial_writes", partial_writes_}, {"errors", errors_}};
+    {"max_read_bytes", max_read_bytes_}, {"parser_buffer_high_water", parser_high_water_},
+    {"parser_storage_bytes_per_connection", FrameParser::storage_bytes}, {"need_more_events", need_more_events_},
+    {"message_error_events", message_error_events_}, {"terminal_frame_events", terminal_frame_events_},
+    {"io_end_events", io_end_events_}, {"partial_eof_events", partial_eof_events_},
+    {"partial_writes", partial_writes_}, {"errors", errors_}};
 }
 Json Server::cleanup() const {
-  std::size_t queued = 0;
-  for (const auto& [fd, conn] : connections_) { (void)fd; queued += conn.outbound.size(); }
+  std::size_t queued = 0, parser_buffered = 0;
+  for (const auto& [fd, conn] : connections_) {
+    (void)fd; queued += conn.outbound.size(); parser_buffered += conn.parser.buffered_bytes();
+  }
   return Json{{"server_connections", connections_.size()}, {"server_descriptors", owned_descriptors().size()},
     {"mailbox_messages", mailbox_.size()}, {"pending_inputs", room_.pending_count()}, {"outbound_messages", queued},
+    {"parser_buffered_bytes", parser_buffered}, {"parser_storage_bytes", connections_.size() * FrameParser::storage_bytes},
     {"worker_threads", 0}, {"timers", 0}, {"disconnect_notifications", disconnected_.size()}};
 }
 std::vector<int> Server::owned_descriptors() const {
@@ -376,15 +495,24 @@ TcpClient::TcpClient(std::uint16_t port) : fd_(::socket(AF_INET, SOCK_STREAM, 0)
   }
 }
 void TcpClient::send(const Json& value) {
-  PendingWrite write{encode_frame(value), 0};
+  send_bytes(encode_frame(value));
+}
+std::size_t TcpClient::send_bytes(std::span<const std::uint8_t> bytes) {
+  std::size_t writes = 0;
   const auto deadline = std::chrono::steady_clock::now() + std::chrono::seconds(2);
-  while (!write.remaining().empty()) {
-    const auto bytes = write.remaining();
+  while (!bytes.empty()) {
     const auto count = ::send(fd_.get(), bytes.data(), bytes.size(), 0);
-    if (count > 0) write.consume(static_cast<std::size_t>(count));
+    if (count > 0) { bytes = bytes.subspan(static_cast<std::size_t>(count)); ++writes; }
     else if (count < 0 && !transient_io()) system_failure("client send");
     if (std::chrono::steady_clock::now() >= deadline) throw std::runtime_error("client send deadline exceeded");
   }
+  return writes;
+}
+bool TcpClient::peer_closed() const {
+  std::uint8_t byte = 0;
+  const auto count = ::recv(fd_.get(), &byte, 1, MSG_PEEK);
+  if (count < 0 && !transient_io()) system_failure("client EOF probe");
+  return count == 0;
 }
 std::optional<Json> TcpClient::try_receive() {
   std::array<std::uint8_t, max_frame_bytes + 4> bytes{};
diff --git a/src/transport.hpp b/src/transport.hpp
index 90824ef..ed30026 100644
--- a/src/transport.hpp
+++ b/src/transport.hpp
@@ -1,5 +1,6 @@
 #pragma once
 #include "game.hpp"
+#include <array>
 #include <atomic>
 #include <cstddef>
 #include <span>
@@ -24,6 +25,33 @@ class Fd {
 
 std::vector<std::uint8_t> encode_frame(const Json& value);
 Json decode_complete_frame(std::span<const std::uint8_t> bytes);
+
+enum class ParseState { need_more, message, message_error, terminal_frame_error, io_end };
+std::string parse_state_name(ParseState state);
+struct ParseResult {
+  ParseState state = ParseState::need_more;
+  std::size_t consumed = 0;
+  Json value;
+  std::string code;
+  bool partial_frame = false;
+};
+
+// One result at a time: the caller consumes any suffix before the next read.
+// No vector of coalesced messages or peer-declared allocation is retained.
+class FrameParser {
+ public:
+  static constexpr std::size_t storage_bytes = max_frame_bytes + 4;
+  ParseResult consume(std::span<const std::uint8_t> bytes);
+  ParseResult finish(bool io_error = false);
+  std::size_t buffered_bytes() const { return used_; }
+  std::size_t high_water_bytes() const { return high_water_; }
+ private:
+  std::array<std::uint8_t, storage_bytes> bytes_{};
+  std::size_t used_ = 0;
+  std::size_t high_water_ = 0;
+  std::size_t expected_ = 4;
+  std::optional<ParseResult> terminal_;
+};
 struct PendingWrite {
   std::vector<std::uint8_t> bytes;
   std::size_t offset = 0;
@@ -55,14 +83,16 @@ class Server {
     std::string player_id;
     std::deque<PendingWrite> outbound;
     std::size_t pending_requests = 0;
+    FrameParser parser;
   };
-  struct Envelope { std::uint64_t connection_id; Json value; };
+  struct Envelope { std::uint64_t connection_id; Json value; std::string parser_error; };
   Connection* connection(std::uint64_t id);
   void register_event(int fd, short filter, unsigned short flags, std::uint64_t connection_id = 0);
   void accept_ready();
   void read_ready(int fd);
   void write_ready(int fd);
   void disconnect(int fd, const std::string& reason);
+  void end_transport(int fd, bool io_error);
   void queue(std::uint64_t connection_id, Json value);
   void broadcast(const Json& value);
   void handle(const Envelope& envelope);
@@ -82,6 +112,12 @@ class Server {
   std::size_t outbound_high_water_ = 0;
   std::size_t connection_high_water_ = 0;
   std::size_t max_read_bytes_ = 0;
+  std::size_t parser_high_water_ = 0;
+  std::uint64_t need_more_events_ = 0;
+  std::uint64_t message_error_events_ = 0;
+  std::uint64_t terminal_frame_events_ = 0;
+  std::uint64_t io_end_events_ = 0;
+  std::uint64_t partial_eof_events_ = 0;
   std::uint64_t received_messages_ = 0;
   std::uint64_t sent_messages_ = 0;
   std::uint64_t partial_writes_ = 0;
@@ -90,12 +126,14 @@ class Server {
 };
 
 // Test/CLI client owns real TCP. Kernel-peeking waits for one bounded complete
-// response; the server's G01 read intentionally retains no partial-frame state.
+// response. send_bytes exposes exact frozen fragmentation/coalescing fixtures.
 class TcpClient {
  public:
   explicit TcpClient(std::uint16_t port);
   void send(const Json& value);
+  std::size_t send_bytes(std::span<const std::uint8_t> bytes);
   std::optional<Json> try_receive();
+  bool peer_closed() const;
   Json receive(Server& server);
   Json receive_type(Server& server, const std::string& type);
   void close() { fd_.reset(); }
diff --git a/tests/tests.cpp b/tests/tests.cpp
index 0b886d0..12ac70d 100644
--- a/tests/tests.cpp
+++ b/tests/tests.cpp
@@ -125,6 +125,90 @@ void foreign_thread_mutation_is_rejected() {
   foreign.join();
   check(rejected.load() && room.pending_count() == 0, "foreign thread cannot mutate the single owner room");
 }
+std::vector<std::uint8_t> framed_text(const std::string& text) {
+  const auto size = static_cast<std::uint32_t>(text.size());
+  std::vector<std::uint8_t> frame{static_cast<std::uint8_t>(size >> 24U), static_cast<std::uint8_t>(size >> 16U),
+    static_cast<std::uint8_t>(size >> 8U), static_cast<std::uint8_t>(size)};
+  frame.insert(frame.end(), text.begin(), text.end());
+  return frame;
+}
+void strict_protocol_and_message_recovery() {
+  struct Case { std::string name; std::string text; std::string code; };
+  // The supplemental protocol list was fixed in main's G02 plan before the
+  // G01 reproduction. These are pure parser cases, not extra socket fuzzing.
+  const std::vector<Case> cases{
+    {"root-array", "[]", "MESSAGE_INVALID"},
+    {"missing-v", R"({"type":"HELLO"})", "MESSAGE_INVALID"},
+    {"missing-type", R"({"v":1})", "MESSAGE_INVALID"},
+    {"v-string", R"({"v":"1","type":"HELLO"})", "MESSAGE_INVALID"},
+    {"v-floating", R"({"v":1.0,"type":"HELLO"})", "MESSAGE_INVALID"},
+    {"v-2", R"({"v":2,"type":"HELLO"})", "PROTOCOL_VERSION_UNSUPPORTED"},
+    {"unknown-type", R"({"v":1,"type":"UNKNOWN"})", "MESSAGE_TYPE_UNKNOWN"},
+    {"known-unknown-field", R"({"v":1,"type":"HELLO","extra":{"nested":[1,2]}})", ""},
+    {"NaN", R"({"v":1,"type":"HELLO","extra":NaN})", "MESSAGE_INVALID"},
+    {"Infinity", R"({"v":1,"type":"HELLO","extra":Infinity})", "MESSAGE_INVALID"},
+    {"trailing-object", R"({"v":1,"type":"HELLO"}{})", "MESSAGE_INVALID"}
+  };
+  const auto next = encode_frame(message("HELLO"));
+  Json evidence = Json::array();
+  for (const auto& item : cases) {
+    auto bytes = framed_text(item.text);
+    const auto frame_size = bytes.size();
+    bytes.insert(bytes.end(), next.begin(), next.end());
+    FrameParser parser;
+    const auto first = parser.consume(bytes);
+    check(first.state == (item.code.empty() ? ParseState::message : ParseState::message_error), item.name + " classification");
+    check(first.code == item.code && first.consumed == frame_size, item.name + " stable code and exact framing boundary");
+    const auto second = parser.consume(std::span(bytes).subspan(first.consumed));
+    check(second.state == ParseState::message && second.value == message("HELLO") && second.consumed == next.size(),
+          item.name + " retains next coalesced message after recoverable error");
+    check(parser.buffered_bytes() == 0 && parser.high_water_bytes() <= FrameParser::storage_bytes,
+          item.name + " no retained payload after complete frame");
+    evidence.push_back(Json{{"name", item.name}, {"state", parse_state_name(first.state)}, {"code", first.code},
+      {"next_state", parse_state_name(second.state)}, {"buffered_bytes", parser.buffered_bytes()}});
+  }
+  std::cout << Json{{"G02_protocol_evidence", evidence}}.dump() << '\n';
+}
+void parser_maximum_and_transport_end() {
+  Json maximum{{"v", 1}, {"type", "HELLO"}, {"extra", ""}};
+  maximum["extra"] = std::string(max_frame_bytes - maximum.dump().size(), 'x');
+  const auto frame = encode_frame(maximum);
+  check(frame.size() == 16388, "valid max-size fixture is exactly 16384 payload bytes");
+  FrameParser parser;
+  const auto header = parser.consume(std::span(frame).first(4));
+  check(header.state == ParseState::need_more && header.consumed == 4 && parser.buffered_bytes() == 4,
+        "full header is not a complete payload");
+  const auto complete = parser.consume(std::span(frame).subspan(4));
+  check(complete.state == ParseState::message && complete.value == maximum && parser.buffered_bytes() == 0,
+        "valid maximum frame is accepted without losing bytes");
+  check(FrameParser::storage_bytes == 16388 && parser.high_water_bytes() == 16388, "fixed parser storage bound is reached, not exceeded");
+  const auto clean_end = parser.finish();
+  check(clean_end.state == ParseState::io_end && clean_end.code == "TRANSPORT_EOF" && !clean_end.partial_frame,
+        "clean transport EOF has no partial frame");
+  const auto hello = encode_frame(message("HELLO"));
+  Json ends = Json::array();
+  for (const auto count : {std::size_t{3}, hello.size() - 1}) {
+    FrameParser partial;
+    const auto waiting = partial.consume(std::span(hello).first(count));
+    check(waiting.state == ParseState::need_more && partial.buffered_bytes() == count, "partial header/payload retained before EOF");
+    const auto end = partial.finish();
+    check(end.state == ParseState::io_end && end.code == "TRANSPORT_EOF_IN_FRAME" && end.partial_frame,
+          "partial EOF is transport-terminal, not a message or framing size error");
+    const auto after = partial.consume(hello);
+    check(after.state == ParseState::io_end && after.consumed == 0 && partial.buffered_bytes() == 0,
+          "transport end releases buffered bytes and cannot reopen the stream");
+    ends.push_back(Json{{"partial_bytes", count}, {"state", parse_state_name(end.state)},
+      {"code", end.code}, {"buffered_after", partial.buffered_bytes()}});
+  }
+  FrameParser broken;
+  broken.consume(std::span(hello).first(1));
+  const auto io_error = broken.finish(true);
+  check(io_error.state == ParseState::io_end && io_error.code == "TRANSPORT_IO_ERROR" && io_error.partial_frame &&
+        broken.buffered_bytes() == 0, "I/O failure is distinct from orderly partial EOF");
+  std::cout << Json{{"G02_max_payload_bytes", frame.size() - 4}, {"parser_high_water_bytes", parser.high_water_bytes()},
+    {"parser_storage_bytes", FrameParser::storage_bytes}, {"partial_eof", ends}, {"clean_eof_code", clean_end.code},
+    {"io_error_code", io_error.code}, {"retained_bytes", broken.buffered_bytes()}}.dump() << '\n';
+}
 void real_tcp_authority_and_cleanup() {
   const auto scenario = Json::parse(R"({
     "scenario_id":"G01-three-tick-authority-smoke","contract_version":1,"thread":"G01","seed":7050,
@@ -245,7 +329,9 @@ int main(int argc, char** argv) {
     tests = {{"lifecycle_and_1200_ticks", lifecycle_and_duration}, {"integer_movement_clamp", movement_is_integer_and_bounded},
       {"TAG_wide_distance_one_shot", tag_uses_wide_distance_and_one_shot_intent}, {"pending_input_bound", input_capacity_is_explicit},
       {"complete_frame_owned_partial_write", complete_frame_and_owned_write_buffer}, {"RAII_descriptor_ownership", descriptor_ownership},
-      {"foreign_thread_mutation_rejected", foreign_thread_mutation_is_rejected}};
+      {"foreign_thread_mutation_rejected", foreign_thread_mutation_is_rejected},
+      {"G02_strict_protocol_message_recovery", strict_protocol_and_message_recovery},
+      {"G02_maximum_frame_transport_end", parser_maximum_and_transport_end}};
   } else if (std::string(argv[1]) == "integration") {
     tests = {{"real_TCP_authority_and_cleanup", real_tcp_authority_and_cleanup}, {"standalone_process_shutdown", [&] {
       standalone_process_shutdown(std::filesystem::absolute(argv[0]).parent_path() / "arena"); }}};
