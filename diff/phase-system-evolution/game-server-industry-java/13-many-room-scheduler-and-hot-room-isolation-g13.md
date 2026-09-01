# Many-room Scheduler와 Hot-room Isolation (G13)

## `feat: isolate bounded many-room scheduling`

diff --git a/evidence/G13-command-ledger.jsonl b/evidence/G13-command-ledger.jsonl
new file mode 100644
index 0000000..a6b72ae
--- /dev/null
+++ b/evidence/G13-command-ledger.jsonl
@@ -0,0 +1,44 @@
+{"kind": "activation", "thread": "G13", "profile": "realtime-core", "spec_revision": "c1d62196ab76b55652f5d75a67514f8c6d8210ce", "start": "adfd4549bc9204de094be12d9028a541ea44d899", "scenario": "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G13.json", "scenario_sha256": "ad6304b939483b4498a884ebe62681d1d3bea8810bb2518642c5f5efc9b1e70b", "at": "2026-08-28T08:34:57.871933+00:00", "ceilings": {"compiler_tasks": 8, "unit_including_baseline": 4, "integration": 2, "post_live": 1, "normal_reference_processes": 31, "normal_reference_ticks": 775, "fault": 0, "load": 0, "profiler": 0, "repairs": 2}, "baseline_scope": "one real G12 instance, first ordinary Room/four joins/binds and fresh session second CREATE; no ticks or replacement servers"}
+{"kind": "resolved_before_execution", "category": "unchanged-production-prerequisite", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test", "--tests", "arena.G13BaselineTest"], "environment": {"ARENA_G13_SCENARIO": "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G13.json", "ARENA_G13_EVIDENCE": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reproduce-unit/result.json"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reproduce-unit", "reserved_g13_ticks": 0, "pass": "baseline"}
+{"kind": "resolved_before_execution", "category": "clean-build", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "build"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/verify-build", "reserved_g13_ticks": 0, "pass": "build"}
+{"kind": "resolved_before_execution", "category": "full-unit-existing-regressions-no-G13-live-campaign", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/verify-unit", "reserved_g13_ticks": 0, "pass": "unit"}
+{"kind": "resolved_before_execution", "category": "integration", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "integration-test"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/verify-integration", "reserved_g13_ticks": 0, "pass": "integration"}
+{"kind": "resolved_before_execution", "category": "fixed-32-room-live", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "scenario-run", "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G13.json", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical", "reserved_g13_ticks": 795, "normal_ticks": 775, "hot_ticks": 20, "pass": "canonical"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-01.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-01/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-01", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-01.replay.jsonl", "pass": "reference-room-01"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-02.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-02/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-02", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-02.replay.jsonl", "pass": "reference-room-02"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-03.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-03/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-03", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-03.replay.jsonl", "pass": "reference-room-03"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-04.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-04/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-04", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-04.replay.jsonl", "pass": "reference-room-04"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-05.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-05/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-05", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-05.replay.jsonl", "pass": "reference-room-05"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-06.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-06/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-06", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-06.replay.jsonl", "pass": "reference-room-06"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-07.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-07/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-07", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-07.replay.jsonl", "pass": "reference-room-07"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-08.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-08/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-08", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-08.replay.jsonl", "pass": "reference-room-08"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-09.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-09/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-09", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-09.replay.jsonl", "pass": "reference-room-09"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-10.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-10/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-10", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-10.replay.jsonl", "pass": "reference-room-10"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-11.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-11/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-11", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-11.replay.jsonl", "pass": "reference-room-11"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-12.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-12/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-12", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-12.replay.jsonl", "pass": "reference-room-12"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-13.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-13/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-13", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-13.replay.jsonl", "pass": "reference-room-13"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-14.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-14/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-14", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-14.replay.jsonl", "pass": "reference-room-14"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-15.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-15/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-15", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-15.replay.jsonl", "pass": "reference-room-15"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-16.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-16/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-16", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-16.replay.jsonl", "pass": "reference-room-16"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-17.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-17/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-17", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-17.replay.jsonl", "pass": "reference-room-17"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-18.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-18/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-18", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-18.replay.jsonl", "pass": "reference-room-18"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-19.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-19/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-19", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-19.replay.jsonl", "pass": "reference-room-19"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-20.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-20/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-20", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-20.replay.jsonl", "pass": "reference-room-20"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-21.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-21/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-21", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-21.replay.jsonl", "pass": "reference-room-21"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-22.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-22/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-22", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-22.replay.jsonl", "pass": "reference-room-22"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-23.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-23/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-23", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-23.replay.jsonl", "pass": "reference-room-23"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-24.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-24/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-24", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-24.replay.jsonl", "pass": "reference-room-24"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-25.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-25/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-25", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-25.replay.jsonl", "pass": "reference-room-25"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-26.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-26/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-26", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-26.replay.jsonl", "pass": "reference-room-26"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-27.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-27/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-27", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-27.replay.jsonl", "pass": "reference-room-27"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-28.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-28/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-28", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-28.replay.jsonl", "pass": "reference-room-28"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-29.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-29/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-29", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-29.replay.jsonl", "pass": "reference-room-29"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-30.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-30/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-30", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-30.replay.jsonl", "pass": "reference-room-30"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-31.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-31/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reference/room-31", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.room-31.replay.jsonl", "pass": "reference-room-31"}
+{"pass": "baseline", "category": "unchanged-production-prerequisite", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test", "--tests", "arena.G13BaselineTest"], "environment": {"ARENA_G13_SCENARIO": "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G13.json", "ARENA_G13_EVIDENCE": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reproduce-unit/result.json"}, "kind": "executed", "started_at": "2026-08-28T08:35:07.415046+00:00", "duration_seconds": 5.027, "command_process_id": 81314, "exit_code": 1, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reproduce-unit/stdout.log", "process_terminated": true, "compiler_tasks_executed": ["> Task :compileTestJava"], "xml": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reproduce-unit/xml", "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/reproduce-unit/result.json", "simulation_process_id": 81369, "executed_ticks": 0}
+{"kind": "preverification_source_audit", "at": "2026-08-28T08:53:11.878135+00:00", "source_hashes": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/post-source-hashes.json", "baseline_disposable_test_removed": true, "live_campaign_tests_in_unit": 0, "post_live_normal_ticks": 775, "post_live_hot_ticks": 20, "offline_processes": 31, "offline_ticks": 775, "old_G12_production_queue_unchanged": true, "next_passes": ["build", "unit", "integration", "canonical", "reference-room-01", "reference-room-02", "reference-room-03", "reference-room-04", "reference-room-05", "reference-room-06", "reference-room-07", "reference-room-08", "reference-room-09", "reference-room-10", "reference-room-11", "reference-room-12", "reference-room-13", "reference-room-14", "reference-room-15", "reference-room-16", "reference-room-17", "reference-room-18", "reference-room-19", "reference-room-20", "reference-room-21", "reference-room-22", "reference-room-23", "reference-room-24", "reference-room-25", "reference-room-26", "reference-room-27", "reference-room-28", "reference-room-29", "reference-room-30", "reference-room-31"], "stop_on_first_runtime_failure": true, "room_mailbox_observation": "actual bounded admission/drain after each transport-owner handoff; no256 simultaneous backlog claim"}
+{"pass": "build", "category": "clean-build", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "build"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T08:53:29.715470+00:00", "duration_seconds": 6.811, "command_process_id": 99653, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/verify-build/stdout.log", "process_terminated": true, "compiler_tasks_executed": ["> Task :compileJava", "> Task :compileTestJava"]}
+{"pass": "unit", "category": "full-unit-existing-regressions-no-G13-live-campaign", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T08:54:18.010343+00:00", "duration_seconds": 4.768, "command_process_id": 602, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/verify-unit/stdout.log", "process_terminated": true, "compiler_tasks_executed": [], "xml": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/verify-unit/xml"}
+{"pass": "integration", "category": "integration", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "integration-test"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T08:54:22.782857+00:00", "duration_seconds": 4.656, "command_process_id": 712, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/verify-integration/stdout.log", "process_terminated": true, "compiler_tasks_executed": [], "xml": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/verify-integration/xml"}
+{"pass": "canonical", "category": "fixed-32-room-live", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "scenario-run", "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G13.json", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/result.json"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T08:54:27.440315+00:00", "duration_seconds": 2.329, "command_process_id": 914, "exit_code": 1, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/stdout.log", "process_terminated": true, "compiler_tasks_executed": []}
+{"kind": "runtime_failure_handoff", "at": "2026-08-28T08:55:44.011236+00:00", "head": "adfd4549bc9204de094be12d9028a541ea44d899", "source_unchanged_since_post_build": true, "source_files": 48, "source_hashes": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/failed-source-hashes.json", "failed_tree_archive": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/failed-tree.tar", "failed_tree_sha256": "bf28ff23640c6ac0a86a61270e33539eaf4204349cf22af21f97f6e877f26173", "tracked_diff": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-initial/canonical/failed-tracked.diff", "unit": {"tests": 47, "failures": 0, "errors": 0, "skipped": 0}, "integration": {"tests": 4, "failures": 0, "errors": 0, "skipped": 0}, "failure": "actual canonical JVM exits1: ManyRoomScenario.java:173 calls owner-guarded Room.replayStoredBytes after server.close from caller thread; Room.assertOwner throws. No result/artifact was exported, so prior live progress/any caught earlier exception is not recoverable from stdout and is not claimed.", "canonical_result_exported": false, "canonical_actual_ticks": null, "unexecuted": ["reference-room-01", "reference-room-02", "reference-room-03", "reference-room-04", "reference-room-05", "reference-room-06", "reference-room-07", "reference-room-08", "reference-room-09", "reference-room-10", "reference-room-11", "reference-room-12", "reference-room-13", "reference-room-14", "reference-room-15", "reference-room-16", "reference-room-17", "reference-room-18", "reference-room-19", "reference-room-20", "reference-room-21", "reference-room-22", "reference-room-23", "reference-room-24", "reference-room-25", "reference-room-26", "reference-room-27", "reference-room-28", "reference-room-29", "reference-room-30", "reference-room-31"], "consumed": {"compiler_tasks": 3, "compiler_bearing_commands": 2, "unit_including_baseline": 2, "integration": 1, "post_live_attempts": 1, "post_live_planned_ticks": 795, "post_live_observed_ticks": null, "reference_processes": 0, "reference_ticks": 0, "baseline_ticks": 0, "fault": 0, "load": 0, "profiler": 0, "repairs": 0}, "production_or_test_edits_after_failure": 0, "reruns_after_failure": 0, "commit_after_failure": false}
diff --git a/evidence/G13-repair1-command-ledger.jsonl b/evidence/G13-repair1-command-ledger.jsonl
new file mode 100644
index 0000000..9a6c11d
--- /dev/null
+++ b/evidence/G13-repair1-command-ledger.jsonl
@@ -0,0 +1,107 @@
+{"kind": "activation", "thread": "G13", "profile": "realtime-core", "spec_revision": "c1d62196ab76b55652f5d75a67514f8c6d8210ce", "start": "adfd4549bc9204de094be12d9028a541ea44d899", "attempt": "repair1", "at": "2026-08-28T09:02:30.358588+00:00", "reproduction": "reuse preserved initial failed fixed live; no additional baseline", "initial_archive_sha256": "bf28ff23640c6ac0a86a61270e33539eaf4204349cf22af21f97f6e877f26173", "ceilings": {"compiler_tasks": 8, "unit": 4, "integration": 2, "post_live": 1, "normal_reference_processes": 31, "normal_reference_ticks": 775, "fault": 0, "load": 0, "profiler": 0}, "scope": "only ManyRoomScenario evidence lifecycle; no production/old-test/fixture edits"}
+{"kind": "resolved_before_execution", "category": "clean-build", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "build"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/verify-build", "reserved_g13_ticks": 0, "pass": "build", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "full-unit-existing-regressions-no-G13-live-campaign", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/verify-unit", "reserved_g13_ticks": 0, "pass": "unit", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "integration", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "integration-test"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/verify-integration", "reserved_g13_ticks": 0, "pass": "integration", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "fixed-32-room-live", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "scenario-run", "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G13.json", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical", "reserved_g13_ticks": 795, "normal_ticks": 775, "hot_ticks": 20, "pass": "canonical", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-01.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-01/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-01", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-01.replay.jsonl", "pass": "reference-room-01", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-02.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-02/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-02", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-02.replay.jsonl", "pass": "reference-room-02", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-03.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-03/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-03", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-03.replay.jsonl", "pass": "reference-room-03", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-04.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-04/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-04", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-04.replay.jsonl", "pass": "reference-room-04", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-05.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-05/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-05", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-05.replay.jsonl", "pass": "reference-room-05", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-06.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-06/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-06", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-06.replay.jsonl", "pass": "reference-room-06", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-07.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-07/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-07", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-07.replay.jsonl", "pass": "reference-room-07", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-08.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-08/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-08", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-08.replay.jsonl", "pass": "reference-room-08", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-09.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-09/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-09", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-09.replay.jsonl", "pass": "reference-room-09", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-10.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-10/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-10", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-10.replay.jsonl", "pass": "reference-room-10", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-11.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-11/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-11", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-11.replay.jsonl", "pass": "reference-room-11", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-12.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-12/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-12", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-12.replay.jsonl", "pass": "reference-room-12", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-13.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-13/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-13", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-13.replay.jsonl", "pass": "reference-room-13", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-14.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-14/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-14", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-14.replay.jsonl", "pass": "reference-room-14", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-15.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-15/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-15", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-15.replay.jsonl", "pass": "reference-room-15", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-16.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-16/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-16", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-16.replay.jsonl", "pass": "reference-room-16", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-17.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-17/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-17", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-17.replay.jsonl", "pass": "reference-room-17", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-18.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-18/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-18", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-18.replay.jsonl", "pass": "reference-room-18", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-19.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-19/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-19", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-19.replay.jsonl", "pass": "reference-room-19", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-20.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-20/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-20", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-20.replay.jsonl", "pass": "reference-room-20", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-21.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-21/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-21", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-21.replay.jsonl", "pass": "reference-room-21", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-22.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-22/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-22", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-22.replay.jsonl", "pass": "reference-room-22", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-23.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-23/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-23", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-23.replay.jsonl", "pass": "reference-room-23", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-24.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-24/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-24", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-24.replay.jsonl", "pass": "reference-room-24", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-25.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-25/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-25", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-25.replay.jsonl", "pass": "reference-room-25", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-26.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-26/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-26", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-26.replay.jsonl", "pass": "reference-room-26", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-27.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-27/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-27", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-27.replay.jsonl", "pass": "reference-room-27", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-28.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-28/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-28", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-28.replay.jsonl", "pass": "reference-room-28", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-29.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-29/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-29", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-29.replay.jsonl", "pass": "reference-room-29", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-30.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-30/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-30", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-30.replay.jsonl", "pass": "reference-room-30", "attempt": "repair1"}
+{"kind": "resolved_before_execution", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-31.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-31/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-31", "reserved_g13_ticks": 25, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-31.replay.jsonl", "pass": "reference-room-31", "attempt": "repair1"}
+{"kind": "started", "pass": "build", "at": "2026-08-28T09:04:52.522273+00:00", "argv": ["./track", "build"], "ceiling_seconds": 300}
+{"pass": "build", "category": "clean-build", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "build"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:04:52.522273+00:00", "duration_seconds": 6.615, "command_process_id": 12262, "process_group_id": 12262, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/verify-build/stdout.log", "output_sha256": "b98717e0c3db39e162147bfaafacab5376fb644cfceab55f605bff5f89f72b6e", "process_terminated": true, "compiler_tasks_executed": ["> Task :compileJava", "> Task :compileTestJava"]}
+{"kind": "started", "pass": "unit", "at": "2026-08-28T09:04:59.153861+00:00", "argv": ["./track", "unit-test"], "ceiling_seconds": 300}
+{"pass": "unit", "category": "full-unit-existing-regressions-no-G13-live-campaign", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:04:59.153861+00:00", "duration_seconds": 5.437, "command_process_id": 12385, "process_group_id": 12385, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/verify-unit/stdout.log", "output_sha256": "a02349cf5186eb8550ba8fa9b47ac545d29341340857fe62cf90e23ffacbafef", "process_terminated": true, "compiler_tasks_executed": [], "xml": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/verify-unit/xml"}
+{"kind": "started", "pass": "integration", "at": "2026-08-28T09:05:04.597200+00:00", "argv": ["./track", "integration-test"], "ceiling_seconds": 300}
+{"pass": "integration", "category": "integration", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "integration-test"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:05:04.597200+00:00", "duration_seconds": 5.437, "command_process_id": 12500, "process_group_id": 12500, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/verify-integration/stdout.log", "output_sha256": "b81bb3390f4865ef54ab200b23643c0291e78a1a1f17b82b8977f410b7f1c74a", "process_terminated": true, "compiler_tasks_executed": [], "xml": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/verify-integration/xml"}
+{"kind": "started", "pass": "canonical", "at": "2026-08-28T09:05:27.431447+00:00", "argv": ["./track", "scenario-run", "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G13.json", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.json"], "ceiling_seconds": 300}
+{"pass": "canonical", "category": "fixed-32-room-live", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "scenario-run", "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G13.json", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:05:27.431447+00:00", "duration_seconds": 2.459, "command_process_id": 12975, "process_group_id": 12975, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/stdout.log", "output_sha256": "c902e5e7508b78d2fd26bfec2a9d0223f439874995cb9c1934960774c303994d", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.json", "result_sha256": "7fdb0a28cf1d991fe668b53e59eae6d5d177c50ed775e2ec788dca736bdb5ffe", "simulation_process_id": 12975, "executed_ticks": 795}
+{"kind": "started", "pass": "reference-room-01", "at": "2026-08-28T09:06:14.005428+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-01.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-01/result.json"], "ceiling_seconds": 300}
+{"pass": "reference-room-01", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-01.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-01/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:06:14.005428+00:00", "duration_seconds": 0.353, "command_process_id": 13704, "process_group_id": 13704, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-01/stdout.log", "output_sha256": "f70c34438e1cd386c678bcb7a2951a824a098bb68586607f9d8a19a5c4871dee", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-01/result.json", "result_sha256": "545ab7d1209cf87ebb84b2bb66b631fd2571c826cc8a1faea4cb7ac48761e775", "simulation_process_id": 13704, "executed_ticks": 25}
+{"kind": "started", "pass": "reference-room-02", "at": "2026-08-28T09:06:14.361445+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-02.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-02/result.json"], "ceiling_seconds": 300}
+{"pass": "reference-room-02", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-02.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-02/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:06:14.361445+00:00", "duration_seconds": 0.297, "command_process_id": 13709, "process_group_id": 13709, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-02/stdout.log", "output_sha256": "1fed767d04cc89cf6be276dbf3dc41bd4ee7e04c798fa96a4a497eeb6ef91a13", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-02/result.json", "result_sha256": "52aaea9f42cab87430e4fe7922c159baa5a3adfcfbb6bdbeff5ca200f3153d18", "simulation_process_id": 13709, "executed_ticks": 25}
+{"kind": "started", "pass": "reference-room-03", "at": "2026-08-28T09:06:14.660882+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-03.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-03/result.json"], "ceiling_seconds": 300}
+{"pass": "reference-room-03", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-03.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-03/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:06:14.660882+00:00", "duration_seconds": 0.291, "command_process_id": 13716, "process_group_id": 13716, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-03/stdout.log", "output_sha256": "751a2aafe4b3b6c1e1d18e5afd4bc8a44a9fc88959962f83cf4aaac03368e2cc", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-03/result.json", "result_sha256": "327160f38a95e6f1ae4c9c3bb1d6f8fdb3e3a2d9f5bb4d0ba95b845595f43c29", "simulation_process_id": 13716, "executed_ticks": 25}
+{"kind": "started", "pass": "reference-room-04", "at": "2026-08-28T09:06:14.955505+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-04.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-04/result.json"], "ceiling_seconds": 300}
+{"pass": "reference-room-04", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-04.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-04/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:06:14.955505+00:00", "duration_seconds": 0.284, "command_process_id": 13722, "process_group_id": 13722, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-04/stdout.log", "output_sha256": "873631276ee652fb7964aa17ea81157834f80158cc1f748e315195d69c7e83c7", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-04/result.json", "result_sha256": "2013cc22617ce09fa2a75371d39c61a793543aaf80f57c56b5389f38ab053189", "simulation_process_id": 13722, "executed_ticks": 25}
+{"kind": "started", "pass": "reference-room-05", "at": "2026-08-28T09:06:15.242283+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-05.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-05/result.json"], "ceiling_seconds": 300}
+{"pass": "reference-room-05", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-05.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-05/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:06:15.242283+00:00", "duration_seconds": 0.24, "command_process_id": 13727, "process_group_id": 13727, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-05/stdout.log", "output_sha256": "d24fa20004f016797988f01885e765d8e7d9739c8e6fef57eab7ffeb12b697b2", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-05/result.json", "result_sha256": "9fd1782622c3c3643127ded3990647503a7133e458c71d9af8cec4706d232c44", "simulation_process_id": 13727, "executed_ticks": 25}
+{"kind": "started", "pass": "reference-room-06", "at": "2026-08-28T09:06:15.485231+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-06.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-06/result.json"], "ceiling_seconds": 300}
+{"pass": "reference-room-06", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-06.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-06/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:06:15.485231+00:00", "duration_seconds": 0.293, "command_process_id": 13732, "process_group_id": 13732, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-06/stdout.log", "output_sha256": "3f41175b38e4d9d6b026fccab86f0df37907ae01499d3a18b85cb16091530dc1", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-06/result.json", "result_sha256": "9552191d2714fb62a1b44e13c7143cb723d901e2b849a243a7b67d0f7a2bba5d", "simulation_process_id": 13732, "executed_ticks": 25}
+{"kind": "started", "pass": "reference-room-07", "at": "2026-08-28T09:06:15.781317+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-07.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-07/result.json"], "ceiling_seconds": 300}
+{"pass": "reference-room-07", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-07.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-07/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:06:15.781317+00:00", "duration_seconds": 0.242, "command_process_id": 13765, "process_group_id": 13765, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-07/stdout.log", "output_sha256": "e5350e99b4b8ccd1bbfbda2975b4fef97f4ae8bda519e77167515b8b293e5f0e", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-07/result.json", "result_sha256": "396305da830b49f2781b15c85670b286b1b48b23b1c97bcac7534ab4cc711b2f", "simulation_process_id": 13765, "executed_ticks": 25}
+{"kind": "started", "pass": "reference-room-08", "at": "2026-08-28T09:06:16.025976+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-08.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-08/result.json"], "ceiling_seconds": 300}
+{"pass": "reference-room-08", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-08.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-08/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:06:16.025976+00:00", "duration_seconds": 0.237, "command_process_id": 13770, "process_group_id": 13770, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-08/stdout.log", "output_sha256": "f6edc3fd74cb5b5409a96c835133e5a48f69e26aa69dbee197d06db26041a81a", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-08/result.json", "result_sha256": "18b9ed8c4b81b32eb1f8f81243a8a15f867302826649666a5ba679033e39aef0", "simulation_process_id": 13770, "executed_ticks": 25}
+{"kind": "started", "pass": "reference-room-09", "at": "2026-08-28T09:06:16.266762+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-09.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-09/result.json"], "ceiling_seconds": 300}
+{"pass": "reference-room-09", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-09.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-09/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:06:16.266762+00:00", "duration_seconds": 0.296, "command_process_id": 13775, "process_group_id": 13775, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-09/stdout.log", "output_sha256": "794d4a9497b2e2a9dd151f52cd4ef21bec666a4bd637bf57b249b680b8d0ecd8", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-09/result.json", "result_sha256": "21be34962dd03959ba961bdcf6c9248a9fa001e972d8c8aed046db1afcb32683", "simulation_process_id": 13775, "executed_ticks": 25}
+{"kind": "started", "pass": "reference-room-10", "at": "2026-08-28T09:06:16.565950+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-10.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-10/result.json"], "ceiling_seconds": 300}
+{"pass": "reference-room-10", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-10.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-10/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:06:16.565950+00:00", "duration_seconds": 0.232, "command_process_id": 13780, "process_group_id": 13780, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-10/stdout.log", "output_sha256": "d2a42bfb06e86d4f8afa762c1f1495e0fe04d6fd1656b550cff843d16a3dd961", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-10/result.json", "result_sha256": "906cb2766c2ae5e1d4f6d20f126696765ab8bf695366f54f145578e3284fc16a", "simulation_process_id": 13780, "executed_ticks": 25}
+{"kind": "started", "pass": "reference-room-11", "at": "2026-08-28T09:06:16.801250+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-11.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-11/result.json"], "ceiling_seconds": 300}
+{"pass": "reference-room-11", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-11.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-11/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:06:16.801250+00:00", "duration_seconds": 0.295, "command_process_id": 13785, "process_group_id": 13785, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-11/stdout.log", "output_sha256": "506a9f8d9e3c3a485cac1e5bea6ea94625534543b5a48dad75bed95e63ceb0e7", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-11/result.json", "result_sha256": "7cc12bdb186dc14451d2c118f674049c49fdcdc623618dee5380cd1b2d1b3fc9", "simulation_process_id": 13785, "executed_ticks": 25}
+{"kind": "started", "pass": "reference-room-12", "at": "2026-08-28T09:06:17.099248+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-12.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-12/result.json"], "ceiling_seconds": 300}
+{"pass": "reference-room-12", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-12.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-12/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:06:17.099248+00:00", "duration_seconds": 0.286, "command_process_id": 13790, "process_group_id": 13790, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-12/stdout.log", "output_sha256": "0715e43ae9efd4c555796e3d6ffe4abf2cdbb93223c34b9208ddafa905e9327b", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-12/result.json", "result_sha256": "38a1f4689fe1afa5ba29d881bd27a6beabcf16ba930e03bda768eded323807da", "simulation_process_id": 13790, "executed_ticks": 25}
+{"kind": "started", "pass": "reference-room-13", "at": "2026-08-28T09:06:17.388908+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-13.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-13/result.json"], "ceiling_seconds": 300}
+{"pass": "reference-room-13", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-13.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-13/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:06:17.388908+00:00", "duration_seconds": 0.295, "command_process_id": 13797, "process_group_id": 13797, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-13/stdout.log", "output_sha256": "aa43c5b2807ec324ef55d1c9c51d9301fb89d735cf2ff26f1f7872176d4adfa0", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-13/result.json", "result_sha256": "30e5ee5c550178545e962b5a2fcc9dec6356721aa66b63eeb9bbe2ea0611c2b1", "simulation_process_id": 13797, "executed_ticks": 25}
+{"kind": "started", "pass": "reference-room-14", "at": "2026-08-28T09:06:17.686978+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-14.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-14/result.json"], "ceiling_seconds": 300}
+{"pass": "reference-room-14", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-14.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-14/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:06:17.686978+00:00", "duration_seconds": 0.243, "command_process_id": 13830, "process_group_id": 13830, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-14/stdout.log", "output_sha256": "ea850bd323d73f8bb43082914642fbd6e4601d6010b7b851e715c8dc8b00b5d7", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-14/result.json", "result_sha256": "8330036c4ca3e099e4159a1112726c08ee78b448b1bde9bcc1ab9a413e6f832e", "simulation_process_id": 13830, "executed_ticks": 25}
+{"kind": "started", "pass": "reference-room-15", "at": "2026-08-28T09:06:17.933145+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-15.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-15/result.json"], "ceiling_seconds": 300}
+{"pass": "reference-room-15", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-15.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-15/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:06:17.933145+00:00", "duration_seconds": 0.242, "command_process_id": 13835, "process_group_id": 13835, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-15/stdout.log", "output_sha256": "4d01fe4fb690e6040947be69df36d5070fff57fdcd51aa34779c788a67fad7b6", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-15/result.json", "result_sha256": "8679f5aac4d0ac2d87a12863646c0290b92283d33c093cc804b10b07f33ec924", "simulation_process_id": 13835, "executed_ticks": 25}
+{"kind": "started", "pass": "reference-room-16", "at": "2026-08-28T09:06:18.177724+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-16.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-16/result.json"], "ceiling_seconds": 300}
+{"pass": "reference-room-16", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-16.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-16/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:06:18.177724+00:00", "duration_seconds": 0.285, "command_process_id": 13842, "process_group_id": 13842, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-16/stdout.log", "output_sha256": "92517b70e58bba3e8855634a6c2b670daa5c97f8571a2d7e639e7c7e90dab8f4", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-16/result.json", "result_sha256": "de72eef14edf02fbe27103f9a110028beb015dacb3cac2c3d5c502489626519a", "simulation_process_id": 13842, "executed_ticks": 25}
+{"kind": "started", "pass": "reference-room-17", "at": "2026-08-28T09:06:18.466235+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-17.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-17/result.json"], "ceiling_seconds": 300}
+{"pass": "reference-room-17", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-17.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-17/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:06:18.466235+00:00", "duration_seconds": 0.291, "command_process_id": 13847, "process_group_id": 13847, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-17/stdout.log", "output_sha256": "e69fe555822a0e2f2c157232507e3884be17e8745c08f0486bedb43bda9ccac1", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-17/result.json", "result_sha256": "97fa5147d1331184fe51a1bf9e012c03fd62fce6f4c73fcd6661452fc5b58654", "simulation_process_id": 13847, "executed_ticks": 25}
+{"kind": "started", "pass": "reference-room-18", "at": "2026-08-28T09:06:18.760286+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-18.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-18/result.json"], "ceiling_seconds": 300}
+{"pass": "reference-room-18", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-18.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-18/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:06:18.760286+00:00", "duration_seconds": 0.297, "command_process_id": 13852, "process_group_id": 13852, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-18/stdout.log", "output_sha256": "f39c64186f88864a34a25a5e7a510f3bbe1d9d352200d113479db116fa732c7c", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-18/result.json", "result_sha256": "226e8d76fe24581bfde434cf9ba5452782daab24d6459a14f51d48ad6ea712c4", "simulation_process_id": 13852, "executed_ticks": 25}
+{"kind": "started", "pass": "reference-room-19", "at": "2026-08-28T09:06:19.060372+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-19.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-19/result.json"], "ceiling_seconds": 300}
+{"pass": "reference-room-19", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-19.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-19/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:06:19.060372+00:00", "duration_seconds": 0.242, "command_process_id": 13857, "process_group_id": 13857, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-19/stdout.log", "output_sha256": "e7ff02ddeacc4d56d921c1467cf4be8d7e557b472bfd2db071ae50616cadef88", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-19/result.json", "result_sha256": "3ec24504fe6e36f3525b449fb1280fc01478a1c2e9815e7c8b7171ff3a6bceac", "simulation_process_id": 13857, "executed_ticks": 25}
+{"kind": "started", "pass": "reference-room-20", "at": "2026-08-28T09:06:19.307277+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-20.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-20/result.json"], "ceiling_seconds": 300}
+{"pass": "reference-room-20", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-20.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-20/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:06:19.307277+00:00", "duration_seconds": 0.286, "command_process_id": 13862, "process_group_id": 13862, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-20/stdout.log", "output_sha256": "cabf7ef173380a5f99e7057845aea25f7fa11d96061d0da0d29b6724f70e51d1", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-20/result.json", "result_sha256": "b56480e342f848968a4585c0f8e7a6874ebb623eac8f35b272d5382612143883", "simulation_process_id": 13862, "executed_ticks": 25}
+{"kind": "started", "pass": "reference-room-21", "at": "2026-08-28T09:06:19.596231+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-21.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-21/result.json"], "ceiling_seconds": 300}
+{"pass": "reference-room-21", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-21.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-21/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:06:19.596231+00:00", "duration_seconds": 0.289, "command_process_id": 13867, "process_group_id": 13867, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-21/stdout.log", "output_sha256": "9a1a64df446cfb10dd0d91b6ddeca52134069459457ccd3af03ed45d73cf3807", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-21/result.json", "result_sha256": "63bbb292ca33e0ac3cc0601dcf7f686429cae060a537918217040e9e57a05e94", "simulation_process_id": 13867, "executed_ticks": 25}
+{"kind": "started", "pass": "reference-room-22", "at": "2026-08-28T09:06:19.889160+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-22.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-22/result.json"], "ceiling_seconds": 300}
+{"pass": "reference-room-22", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-22.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-22/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:06:19.889160+00:00", "duration_seconds": 0.281, "command_process_id": 13902, "process_group_id": 13902, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-22/stdout.log", "output_sha256": "0744884a15b0e12ab6c13fde5e5f19105ab4b1dba618ead65b5da044ae7ec069", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-22/result.json", "result_sha256": "b56e66ec0fd36238fca1de99ff6fcc47bd95d7e48c61c516e57af8f280709f27", "simulation_process_id": 13902, "executed_ticks": 25}
+{"kind": "started", "pass": "reference-room-23", "at": "2026-08-28T09:06:20.173328+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-23.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-23/result.json"], "ceiling_seconds": 300}
+{"pass": "reference-room-23", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-23.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-23/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:06:20.173328+00:00", "duration_seconds": 0.294, "command_process_id": 13907, "process_group_id": 13907, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-23/stdout.log", "output_sha256": "9fff526875552b472fd8ffc8968cb0f1a0413630e31b9669a0ee87b8391fa6e6", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-23/result.json", "result_sha256": "bebc570d4ddd07712fcf654d29c2c61c062d0c11ed59c1d6f4c335136bc8d6ca", "simulation_process_id": 13907, "executed_ticks": 25}
+{"kind": "started", "pass": "reference-room-24", "at": "2026-08-28T09:06:20.471303+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-24.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-24/result.json"], "ceiling_seconds": 300}
+{"pass": "reference-room-24", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-24.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-24/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:06:20.471303+00:00", "duration_seconds": 0.29, "command_process_id": 13913, "process_group_id": 13913, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-24/stdout.log", "output_sha256": "7bce368dc7820fee508c9c8ae903e3ed2997edfacd3aa19790053e451286328d", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-24/result.json", "result_sha256": "4da3cc6d764dbcd8c742129d7d4f198ed6578eeb9699a4fed2566e9b592a798b", "simulation_process_id": 13913, "executed_ticks": 25}
+{"kind": "started", "pass": "reference-room-25", "at": "2026-08-28T09:06:20.764558+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-25.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-25/result.json"], "ceiling_seconds": 300}
+{"pass": "reference-room-25", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-25.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-25/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:06:20.764558+00:00", "duration_seconds": 0.343, "command_process_id": 13918, "process_group_id": 13918, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-25/stdout.log", "output_sha256": "db621d4441321f60c071c74afc5e7a6cb9ff669438c402db0e260847d15278af", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-25/result.json", "result_sha256": "e177e93f649c5e47b75b838ed614ac728fd6538bf18ebf4178e55d75382902c0", "simulation_process_id": 13918, "executed_ticks": 25}
+{"kind": "started", "pass": "reference-room-26", "at": "2026-08-28T09:06:21.114987+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-26.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-26/result.json"], "ceiling_seconds": 300}
+{"pass": "reference-room-26", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-26.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-26/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:06:21.114987+00:00", "duration_seconds": 0.62, "command_process_id": 13938, "process_group_id": 13938, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-26/stdout.log", "output_sha256": "f3d2c976eb851dfd3cc2f638808810f2406c39b14d20afe3e24870100c377836", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-26/result.json", "result_sha256": "5846cd0b60363b807b96bd05d74093ed304be2867b5decd7c9da2569deb2a8a5", "simulation_process_id": 13938, "executed_ticks": 25}
+{"kind": "started", "pass": "reference-room-27", "at": "2026-08-28T09:06:21.739970+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-27.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-27/result.json"], "ceiling_seconds": 300}
+{"pass": "reference-room-27", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-27.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-27/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:06:21.739970+00:00", "duration_seconds": 0.291, "command_process_id": 13979, "process_group_id": 13979, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-27/stdout.log", "output_sha256": "7ac2143327fd05e8405695ce70bb286f4a2bdb0f30feab357c6355a5d8702cbb", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-27/result.json", "result_sha256": "8e2b8d6c89dbd455a0d25a4fb7c37ced7858705d4f13b1eecba4ff9f31804a5d", "simulation_process_id": 13979, "executed_ticks": 25}
+{"kind": "started", "pass": "reference-room-28", "at": "2026-08-28T09:06:22.034634+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-28.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-28/result.json"], "ceiling_seconds": 300}
+{"pass": "reference-room-28", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-28.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-28/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:06:22.034634+00:00", "duration_seconds": 0.293, "command_process_id": 13984, "process_group_id": 13984, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-28/stdout.log", "output_sha256": "6bd220da8cf34c0ea3bf5bc3d1e3d8d6030df48287b8a0ad6fef6f1e8a2d1985", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-28/result.json", "result_sha256": "07dd20a6d5ab93078df54a80a7e345a3870612302e378110aeac664c0858a186", "simulation_process_id": 13984, "executed_ticks": 25}
+{"kind": "started", "pass": "reference-room-29", "at": "2026-08-28T09:06:22.330380+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-29.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-29/result.json"], "ceiling_seconds": 300}
+{"pass": "reference-room-29", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-29.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-29/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:06:22.330380+00:00", "duration_seconds": 0.239, "command_process_id": 13992, "process_group_id": 13992, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-29/stdout.log", "output_sha256": "345378efbbb7b2f0c21e89b0793cf9008ed5181f44563ac8832ae17a42444f03", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-29/result.json", "result_sha256": "8ad0948b39e720c216b9f1ba850f0366bd9bcffaee3f3f54a6c1587552c8f001", "simulation_process_id": 13992, "executed_ticks": 25}
+{"kind": "started", "pass": "reference-room-30", "at": "2026-08-28T09:06:22.573204+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-30.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-30/result.json"], "ceiling_seconds": 300}
+{"pass": "reference-room-30", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-30.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-30/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:06:22.573204+00:00", "duration_seconds": 0.286, "command_process_id": 13997, "process_group_id": 13997, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-30/stdout.log", "output_sha256": "df3a0edb0d7d11f2cc120c27cf4e96c138df6d18c9f21e9679775ec479527b43", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-30/result.json", "result_sha256": "55c957f3daec8b1e9faad102c8aba53e11a123b3e8aa9e9b29ae03c3b4013ba1", "simulation_process_id": 13997, "executed_ticks": 25}
+{"kind": "started", "pass": "reference-room-31", "at": "2026-08-28T09:06:22.862727+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-31.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-31/result.json"], "ceiling_seconds": 300}
+{"pass": "reference-room-31", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/canonical/result.room-31.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-31/result.json"], "environment": {"JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "started_at": "2026-08-28T09:06:22.862727+00:00", "duration_seconds": 0.237, "command_process_id": 14002, "process_group_id": 14002, "exit_code": 0, "child_exit_code": 0, "timed_out": false, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-31/stdout.log", "output_sha256": "c5ec6c42689d284b675cff83b0db91ac3a48fa849af22fe2f290336cca6db942", "process_terminated": true, "compiler_tasks_executed": [], "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/reference/room-31/result.json", "result_sha256": "ab8c566ef88ba42f3366ca17f6ed6d9c98f4aedace9cde582909a23926855fb5", "simulation_process_id": 14002, "executed_ticks": 25}
+{"kind": "verified", "thread": "G13", "attempt": "repair1", "budget": {"compiler_tasks": 2, "unit": 1, "integration": 1, "post_live": 1, "post_live_ticks": 795, "reference_processes": 31, "reference_ticks": 775, "baseline": 0, "fault": 0, "load": 0, "profiler": 0}, "summary": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g13-repair1/worker-verification-summary.json", "summary_sha256": "e9824f821055e114dde7fd35f060abeea662e0d29c0f69dbae5b4d695bb3a0e1"}
diff --git a/evidence/G13-verification.md b/evidence/G13-verification.md
new file mode 100644
index 0000000..c9ccd03
--- /dev/null
+++ b/evidence/G13-verification.md
@@ -0,0 +1,41 @@
+# G13 — bounded many-Room scheduling
+
+- Profile: `realtime-core`; spec revision: `c1d62196ab76b55652f5d75a67514f8c6d8210ce`.
+- START: `adfd4549bc9204de094be12d9028a541ea44d899`; submission attempt: repair1.
+- Fixed input: main `index/scenarios/G13.json`, SHA-256 `ad6304b939483b4498a884ebe62681d1d3bea8810bb2518642c5f5efc9b1e70b`; committed resource is byte-identical.
+
+## Ownership and implementation
+
+Each Room has its own `RoomRuntime`, bounded mailbox, clock accumulator and deadline accounting. The existing bounded serial owner executes Room mutations; Netty only hands off transport intent. Each mailbox drain admits at most64 commands, and each scheduler iteration reads one shared monotonic time and executes at most4 ticks per Room. Session membership selects the actual Room for input, replication, reconnect and disconnect. Historical first-Room fields remain observation aliases, not routing authority.
+
+The32-Room fixture holds only Room0 simulation for225ms between service opportunities. A capped quantum leaving backlog exposes operational overload without replacing RUNNING. Twenty consecutive unrecovered deadlines close only Room0, clear accepted pending input and remove its scheduler/clock registration. Instance bounds remain64 Rooms,512 connections and at most32768 accepted pending entries from the64×8×64 storage bounds. No distributed admission policy, external service or future-stage feature was added.
+
+## Reproduction and preserved failure
+
+The unchanged G12 server created and started its first Room with four clients. A fresh fifth session's second CREATE returned `ROOM_NOT_JOINABLE`; the baseline executed zero ticks. Its source hashes, raw response, failed assertion and cleanup remain under `evidence/runs/g13-initial/reproduce-unit/`.
+
+The initial implementation passed47 unit and4 integration tests, then its one live command exited1 while post-close evidence called owner-guarded `Room.replayStoredBytes` from the caller thread. No result was exported, so its actual tick progress and any earlier caught exception are unknown. The preserved failed archive SHA is `bf28ff23640c6ac0a86a61270e33539eaf4204349cf22af21f97f6e877f26173`; no initial evidence was overwritten.
+
+Repair1 changes only `ManyRoomScenario` evidence lifecycle. Raw post-owner inspection requires observed owner termination; production owner guards remain unchanged. Cleanup exceptions append to failures without replacing an earlier execution error. Every original scenario and cleanup assertion remains in effect. The exact repair diff SHA is `20d1268d649c5dfb63f74ad4e369b62f0c9f5ecff53c8567a4ddfa831807aa87`; all47 other tested source/resource files are unchanged from the failed attempt.
+
+## Verification
+
+Exact argv, timestamps, exits, raw-log hashes and process IDs are in `G13-command-ledger.jsonl` and `G13-repair1-command-ledger.jsonl`. Repair1 raw files are under `evidence/runs/g13-repair1/`.
+
+| Command | Actual result |
+|---|---|
+| `./track build` | clean build exit0; two compiler tasks |
+| `./track unit-test` | exit0;47 tests, no failure/error/skip |
+| `./track integration-test` | exit0;4 tests, no failure/error/skip |
+| `./track scenario-run <fixed-main-G13.json> <fresh-result.json>` | one live pass, exit0;795 ticks |
+| `./track replay-verify <room-XX-journal> <fresh-room-XX-result.json>` |31 separate processes for01–31,25 ticks each, all exit0 |
+
+Live JVM PID12975 completed20 hot-Room ticks and775 normal-Room ticks. All775 normal canonical records and hashes equal their own separately replayed accepted journals. Room0 accepted96 inputs and rejected1440 with `INPUT_RATE_EXCEEDED`;16 accepted pending entries were cleared at `ROOM_OVERLOAD` at1200ms. The other31 Rooms reached25 ticks at1250ms and remained RUNNING until explicit shutdown.
+
+Observed maxima: global mailbox15, per-Room mailbox1, catch-up4, accepted pending140 instance-wide and4 per player, control queue4, pending FULL1/DELTA1 and one transport buffer per peer. These are observed high-water values, not saturation claims. All6584 passively observed writes completed and released their original buffers with zero lifetime or ownership-bound violations. Server metrics include128 earlier WELCOME writes outside that passive probe. All32 Rooms, sockets, buffers, mailboxes, timers, credentials and owned threads were released.
+
+Live result SHA-256: `7fdb0a28cf1d991fe668b53e59eae6d5d177c50ed775e2ec788dca736bdb5ffe`. The48-file tested source manifest SHA-256 is `9b1c680d7f4d9fda7d1eb4e7b667f8f5871d2636d3cfb3663fddc4a7e378f728`. Full per-Room reference paths and hashes are in the raw `worker-verification-summary.json`.
+
+Room0 tick19 hash: `aae1d77ffb90207df11cb2c13b140c94bcb66423d7a01860e7f813bf42be97ad`. Room31 tick24 hash: `7e7af196c536addb058f6e31b27cf81ae483293d02df399772791fe02a3dd445`.
+
+Initial consumption remains3 compiler tasks,2 unit commands including baseline,1 integration command and1 failed live attempt with unknown actual tick progress. Repair1 consumed2 compiler tasks,1 unit command,1 integration command,1 live pass and31×25 reference ticks; no additional baseline, fault, load or profiler runs. Main's independent verification and cross-track comparison remain pending; no G14 work, tag or push was performed.
diff --git a/src/main/java/arena/ArenaServer.java b/src/main/java/arena/ArenaServer.java
index 593f22c..57e1f10 100644
--- a/src/main/java/arena/ArenaServer.java
+++ b/src/main/java/arena/ArenaServer.java
@@ -24,6 +24,7 @@ import java.util.ArrayList;
 import java.util.Base64;
 import java.util.HashMap;
 import java.util.HashSet;
+import java.util.LinkedHashMap;
 import java.util.List;
 import java.util.Map;
 import java.util.Set;
@@ -43,11 +44,14 @@ import java.util.concurrent.atomic.AtomicInteger;
 import java.util.concurrent.locks.LockSupport;
 import java.util.function.LongSupplier;
 
-/** A single simulation owner, with Netty used only for nonblocking transport. */
+/** Independent Room runtimes share one bounded serial owner; Netty only performs nonblocking transport. */
 public final class ArenaServer implements AutoCloseable {
     static final int MAILBOX_LIMIT = 1_024;
     static final int EVENT_LOOP_QUEUE_LIMIT = 1_024;
-    static final int CONNECTION_LIMIT = 8;
+    static final int ROOM_LIMIT = 64;
+    static final int CONNECTION_LIMIT = 512;
+    // Every admitted Room has at most eight stable slots, each bounded by Room.INPUT_LIMIT.
+    static final int TOTAL_INPUT_LIMIT = ROOM_LIMIT * Room.SPAWNS.length * Room.INPUT_LIMIT;
     static final int OUTBOUND_LIMIT = 64;
     static final long UDP_BIND_TTL_NANOS = TimeUnit.MILLISECONDS.toNanos(5_000);
     private static final SecureRandom RANDOM = new SecureRandom();
@@ -63,6 +67,7 @@ public final class ArenaServer implements AutoCloseable {
     private final AtomicInteger outboundHighWater = outboundMetrics.transportPerPeerHighWater;
     private final AtomicInteger mailboxHighWater = new AtomicInteger();
     private final AtomicBoolean closing = new AtomicBoolean();
+    private final AtomicBoolean clockQueued = new AtomicBoolean();
     private final ThreadPoolExecutor owner;
     private final NioEventLoopGroup acceptLoop;
     private final NioEventLoopGroup ioLoop;
@@ -74,8 +79,10 @@ public final class ArenaServer implements AutoCloseable {
     private final Thread ticker;
     // The following fields, including the session registry, are exclusively room-owner state.
     private final Map<Peer, Session> sessions = new HashMap<>();
-    // Only successful joins enter this registry: bounded by the Room's eight stable slots.
+    // Only successful joins enter this registry: at most 64 Rooms x eight stable slots.
     private final Map<String, Session> recoverableSessions = new HashMap<>();
+    private final Map<String, RoomRuntime> rooms = new LinkedHashMap<>();
+    // Historical first-Room observation aliases; routing always uses the Room registry/membership.
     private Room room;
     private FixedTickClock fixedClock;
     private long manualNanos;
@@ -87,6 +94,11 @@ public final class ArenaServer implements AutoCloseable {
     private volatile int closedRecoverableSessions;
     private volatile int closedResumeCredentials;
     private volatile int closedBindCredentials;
+    private volatile int closedRoomCount;
+    private volatile int closedRoomMailboxCount;
+    private volatile int closedPendingInputs;
+    private int roomsHighWater;
+    private int totalInputHighWater;
     private int snapshotRetentionHighWater;
     private volatile ObjectNode closedClockMetrics = Json.MAPPER.createObjectNode().put("active", false)
             .put("accumulator_ns", 0L).put("max_ticks_per_iteration", 0);
@@ -99,6 +111,7 @@ public final class ArenaServer implements AutoCloseable {
         String resumeToken;
         InetSocketAddress endpoint;
         String playerId;
+        RoomRuntime membership;
         boolean reconnectFullPending;
         Session(long now) { bindIssuedAt = now; }
         void disconnected() { snapshots.invalidateBase(); bindToken = null; endpoint = null; reconnectFullPending = false; }
@@ -216,8 +229,9 @@ public final class ArenaServer implements AutoCloseable {
                 while (!closing.get()) {
                     LockSupport.parkNanos(FixedTickClock.TICK_NANOS);
                     if (closing.get()) break;
-                    try { execute(this::clockIteration); }
-                    catch (RejectedExecutionException overload) { break; }
+                    if (!clockQueued.compareAndSet(false, true)) continue;
+                    try { execute(() -> { try { clockIteration(); } finally { clockQueued.set(false); } }); }
+                    catch (RejectedExecutionException overload) { clockQueued.set(false); }
                 }
             });
             ticker.start();
@@ -244,7 +258,12 @@ public final class ArenaServer implements AutoCloseable {
     public int udpPort() { return ((InetSocketAddress) udpListener.localAddress()).getPort(); }
 
     private void execute(Runnable runnable) {
-        owner.execute(runnable);
+        owner.execute(() -> {
+            runnable.run();
+            // Each transport handoff admits to its actual Room mailbox, then bounded owner work runs.
+            // No simulation tick or clock advance is performed by a mailbox drain.
+            for (RoomRuntime runtime : rooms.values()) runtime.drain();
+        });
         mailboxHighWater.accumulateAndGet(owner.getQueue().size(), Math::max);
     }
 
@@ -271,6 +290,14 @@ public final class ArenaServer implements AutoCloseable {
     }
 
     private void handleCommand(Peer peer, ObjectNode message, InetSocketAddress endpoint) {
+        Session session = sessions.get(peer);
+        RoomRuntime target = message.path("type").asText().equals("JOIN_ROOM")
+                ? rooms.get(message.path("room_id").asText()) : session == null ? null : session.membership;
+        if (target == null) applyCommand(peer, message, endpoint, null);
+        else if (!target.offer(() -> applyCommand(peer, message, endpoint, target))) peer.error("INPUT_QUEUE_FULL");
+    }
+
+    private void applyCommand(Peer peer, ObjectNode message, InetSocketAddress endpoint, RoomRuntime runtime) {
         if (closing.get() || !peer.channel.isActive() || peer.outbound.terminal()) return;
         try {
             if (message.path("v").asInt(-1) != 1) { peer.error("PROTOCOL_VERSION_UNSUPPORTED"); return; }
@@ -292,14 +319,18 @@ public final class ArenaServer implements AutoCloseable {
                     && (!endpoint.equals(session.endpoint) || !Json.epochZero(message))) {
                 peer.error("UDP_BIND_INVALID"); return;
             }
+            Room room = runtime == null ? null : runtime.room;
             switch (type) {
                 case "CREATE_ROOM" -> {
-                    if (room != null) { peer.error("ROOM_NOT_JOINABLE"); break; }
-                    room = new Room(opaque(16));
-                    peer.send(Json.message("ROOM_CREATED").put("room_id", room.id).put("status", "LOBBY"));
+                    if (session.playerId != null || rooms.size() == ROOM_LIMIT) { peer.error("ROOM_NOT_JOINABLE"); break; }
+                    String id; do { id = opaque(16); } while (rooms.containsKey(id));
+                    Room created = new Room(id); rooms.put(id, new RoomRuntime(created, monotonicNanos));
+                    if (this.room == null) this.room = created;
+                    roomsHighWater = Math.max(roomsHighWater, rooms.size());
+                    peer.send(Json.message("ROOM_CREATED").put("room_id", created.id).put("status", "LOBBY"));
                 }
                 case "JOIN_ROOM" -> {
-                    if (!roomMatches(peer, message)) break;
+                    if (!roomMatches(peer, message, runtime)) break;
                     if (session.playerId != null || room.status() != Room.Status.LOBBY) {
                         peer.error("ROOM_NOT_JOINABLE"); break;
                     }
@@ -308,13 +339,13 @@ public final class ArenaServer implements AutoCloseable {
                     Room.Player player;
                     try { player = room.join(playerId); }
                     catch (IllegalStateException notJoinable) { peer.error("ROOM_NOT_JOINABLE"); break; }
-                    session.playerId = player.id;
+                    session.playerId = player.id; session.membership = runtime;
                     session.resumeToken = opaque(24); recoverableSessions.put(session.id, session);
                     if (session.endpoint != null) room.ready(player.id);
                     peer.send(Json.message("ROOM_JOINED").put("room_id", room.id).put("player_id", player.id)
                             .put("slot", player.slot).put("status", room.status().name()).put("resume_token", session.resumeToken));
                     if (room.status() == Room.Status.RUNNING) {
-                        startReplication();
+                        startReplication(runtime);
                     }
                 }
                 case "UDP_BIND" -> {
@@ -327,12 +358,12 @@ public final class ArenaServer implements AutoCloseable {
                     peer.realtime(Json.message("UDP_BOUND").put("owner_epoch", 0), endpoint);
                     if (session.reconnectFullPending) {
                         session.reconnectFullPending = false;
-                        if (fixedClock == null && room.status() == Room.Status.RUNNING) startReplication();
+                        if (runtime.clock() == null && room.status() == Room.Status.RUNNING) startReplication(runtime);
                         else publishSnapshot(peer, session, true);
-                    } else if (room != null) startReplication();
+                    } else if (room != null) startReplication(runtime);
                 }
                 case "PING" -> {
-                    if (!roomMatches(peer, message)) break;
+                    if (!roomMatches(peer, message, runtime)) break;
                     if (session.playerId == null || !session.playerId.equals(Json.text(message, "player_id"))) { peer.error("PLAYER_MISMATCH"); break; }
                     var nonce = message.get("ping_id");
                     if (nonce != null && (!nonce.isIntegralNumber() || !nonce.canConvertToLong())) { peer.error("MESSAGE_INVALID"); break; }
@@ -341,7 +372,7 @@ public final class ArenaServer implements AutoCloseable {
                     peer.realtime(pong, endpoint);
                 }
                 case "SNAPSHOT_ACK" -> {
-                    if (!roomMatches(peer, message)) break;
+                    if (!roomMatches(peer, message, runtime)) break;
                     if (session.playerId == null || !session.playerId.equals(Json.text(message, "player_id"))) {
                         peer.error("PLAYER_MISMATCH"); break;
                     }
@@ -350,7 +381,7 @@ public final class ArenaServer implements AutoCloseable {
                             message.has("resync_required") && message.get("resync_required").booleanValue());
                 }
                 case "INPUT" -> {
-                    if (!roomMatches(peer, message)) break;
+                    if (!roomMatches(peer, message, runtime)) break;
                     if (session.playerId == null || !session.playerId.equals(Json.text(message, "player_id"))) {
                         peer.error("PLAYER_MISMATCH"); break;
                     }
@@ -359,12 +390,13 @@ public final class ArenaServer implements AutoCloseable {
                     String target = message.path("tag_target_player_id").isNull() ? null : Json.text(message, "tag_target_player_id");
                     Room.Intent intent = new Room.Intent(Json.sequence(message), Json.targetTick(message), direction, target);
                     String code = room.accept(session.playerId, intent);
+                    totalInputHighWater = Math.max(totalInputHighWater, pendingInputs());
                     if (code == null || code.equals("DUPLICATE"))
                         peer.realtime(Json.message("INPUT_ACK").put("status", code == null ? "ACCEPTED" : code).put("seq", intent.seq()), endpoint);
                     else peer.error(code);
                 }
                 case "LEAVE_ROOM" -> {
-                    if (!roomMatches(peer, message)) break;
+                    if (!roomMatches(peer, message, runtime)) break;
                     if (session.playerId != null) room.left(session.playerId);
                     peer.channel.close();
                 }
@@ -373,8 +405,8 @@ public final class ArenaServer implements AutoCloseable {
         } catch (IllegalArgumentException invalid) { peer.error("MESSAGE_INVALID"); }
     }
 
-    private boolean roomMatches(Peer peer, ObjectNode message) {
-        if (room == null || !room.id.equals(Json.text(message, "room_id"))) {
+    private boolean roomMatches(Peer peer, ObjectNode message, RoomRuntime runtime) {
+        if (runtime == null || rooms.get(runtime.room.id) != runtime || !runtime.room.id.equals(Json.text(message, "room_id"))) {
             peer.error("ROOM_NOT_FOUND"); return false;
         }
         return true;
@@ -383,7 +415,10 @@ public final class ArenaServer implements AutoCloseable {
     private void reconnect(Peer peer, ObjectNode message) {
         Session provisional = sessions.get(peer);
         Session existing = recoverableSessions.get(Json.text(message, "session_id"));
+        RoomRuntime runtime = existing == null ? null : existing.membership;
+        Room room = runtime == null ? null : runtime.room;
         if (provisional == null || provisional.playerId != null || existing == null || room == null
+                || rooms.get(room.id) != runtime
                 || !room.id.equals(Json.text(message, "room_id")) || existing.resumeToken == null
                 || !existing.resumeToken.equals(Json.text(message, "resume_token"))) {
             peer.error("RECONNECT_INVALID"); return;
@@ -401,57 +436,88 @@ public final class ArenaServer implements AutoCloseable {
     private void disconnect(Peer peer) {
         Session session = sessions.remove(peer);
         if (session != null) {
+            RoomRuntime runtime = session.membership;
+            Room room = runtime == null ? null : runtime.room;
             if (session.playerId != null && room != null) {
                 room.disconnect(session.playerId);
                 if (room.player(session.playerId).connectivity == Room.Connectivity.DISCONNECTED) session.disconnected();
                 else { session.release(); recoverableSessions.remove(session.id); }
             } else session.release();
-            if (room != null) startReplication();
+            if (room != null) startReplication(runtime);
         }
     }
 
-    private void startReplication() {
-        if (fixedClock != null || room.status() != Room.Status.RUNNING || closing.get()) return;
-        fixedClock = new FixedTickClock(monotonicNanos);
-        publishSnapshots(true);
+    private void startReplication(RoomRuntime runtime) {
+        if (closing.get() || !runtime.start()) return;
+        if (runtime.room == room) fixedClock = runtime.clock();
+        publishSnapshots(runtime, true);
     }
 
-    private void publishSnapshots(boolean forceFull) {
+    private void publishSnapshots(RoomRuntime runtime, boolean forceFull) {
         for (var entry : sessions.entrySet()) {
             Session session = entry.getValue();
-            if (session.playerId == null || !room.player(session.playerId).connected()) continue;
+            if (session.membership != runtime || session.playerId == null || !runtime.room.player(session.playerId).connected()) continue;
             if (session.endpoint == null) continue;
             publishSnapshot(entry.getKey(), session, forceFull);
         }
     }
 
     private void publishSnapshot(Peer peer, Session session, boolean forceFull) {
-        peer.realtime(session.snapshots.next(room, forceFull), session.endpoint);
+        peer.realtime(session.snapshots.next(session.membership.room, forceFull), session.endpoint);
         snapshotRetentionHighWater = Math.max(snapshotRetentionHighWater, session.snapshots.highWater());
     }
 
-    private void broadcast(ObjectNode message) {
-        for (var entry : sessions.entrySet()) if (entry.getValue().playerId != null) entry.getKey().send(message);
+    private void broadcast(RoomRuntime runtime, ObjectNode message) {
+        for (var entry : sessions.entrySet()) if (entry.getValue().membership == runtime) entry.getKey().send(message);
     }
 
-    private void tick() {
+    private void tick(RoomRuntime runtime) {
+        Room room = runtime.room;
         if (room == null || room.status() != Room.Status.RUNNING || closing.get()) return;
         for (Room.Rejection rejection : room.tick()) {
             sessions.forEach((peer, session) -> {
-                if (rejection.playerId().equals(session.playerId)) peer.error(rejection.code());
+                if (session.membership == runtime && rejection.playerId().equals(session.playerId)) peer.error(rejection.code());
             });
         }
         if (room.executedTicks() % 2 == 0 || room.status() == Room.Status.FINISHED)
-            publishSnapshots(room.status() == Room.Status.FINISHED);
+            publishSnapshots(runtime, room.status() == Room.Status.FINISHED);
         if (room.status() == Room.Status.FINISHED) {
-            broadcast(room.view("ROOM_FINISHED"));
+            broadcast(runtime, room.view("ROOM_FINISHED"));
         }
     }
 
     private void clockIteration() {
-        if (fixedClock == null || room.status() != Room.Status.RUNNING || closing.get()) return;
-        int due = fixedClock.poll();
-        for (int i = 0; i < due; i++) tick();
+        if (closing.get()) return;
+        long now = monotonicNanos.getAsLong();
+        var active = rooms.values().iterator();
+        while (active.hasNext()) {
+            RoomRuntime runtime = active.next(); runtime.drain();
+            int due = runtime.service(now);
+            for (int i = 0; i < due; i++) tick(runtime);
+            if (runtime.deadlineMissed()) {
+                closeOverloaded(runtime); active.remove();
+            }
+        }
+    }
+
+    private int pendingInputs() {
+        return rooms.values().stream().mapToInt(r -> r.room.pendingInputCount()).sum();
+    }
+
+    private void closeOverloaded(RoomRuntime runtime) {
+        var active = sessions.entrySet().iterator();
+        while (active.hasNext()) {
+            var entry = active.next(); Session session = entry.getValue();
+            if (session.membership != runtime) continue;
+            session.release(); recoverableSessions.remove(session.id); active.remove();
+            entry.getKey().send(CompleteFrame.error("ROOM_OVERLOAD", "ROOM_OVERLOAD"), true);
+        }
+        var recovery = recoverableSessions.values().iterator();
+        while (recovery.hasNext()) {
+            Session session = recovery.next();
+            if (session.membership == runtime) { session.release(); recovery.remove(); }
+        }
+        runtime.close();
     }
 
     /** Manual scheduler wake: read the injected monotonic source on the same owner path as the timer. */
@@ -480,8 +546,9 @@ public final class ArenaServer implements AutoCloseable {
         if (closing.get()) return cleanup();
         return call(() -> {
             ObjectNode result = Json.MAPPER.createObjectNode().put("manual_time_ns", manualNanos)
-                    .put("pending_input_high_water", room == null ? 0 : room.inputHighWater())
-                    .put("replay_bytes", room == null ? 0 : room.replayStoredBytes())
+                    .put("pending_input_high_water", rooms.values().stream().mapToInt(r -> r.room.inputHighWater()).max().orElse(0))
+                    .put("pending_inputs", pendingInputs()).put("total_input_high_water", totalInputHighWater)
+                    .put("replay_bytes", rooms.values().stream().mapToInt(r -> r.room.replayStoredBytes()).sum())
                     .put("retained_snapshots", sessions.values().stream().mapToInt(s -> s.snapshots.retainedCount()).sum())
                     .put("snapshot_retention_high_water", snapshotRetentionHighWater)
                     .put("mailbox_high_water", mailboxHighWater.get()).put("outbound_high_water", outboundHighWater.get());
@@ -492,6 +559,11 @@ public final class ArenaServer implements AutoCloseable {
             result.put("udp_bindings", sessions.values().stream().filter(s -> s.endpoint != null).count());
             result.put("active_sessions", sessions.size()).put("recoverable_sessions", recoverableSessions.size())
                     .put("resume_credentials", recoverableSessions.values().stream().filter(s -> s.resumeToken != null).count());
+            result.put("active_rooms", rooms.size()).put("rooms_high_water", roomsHighWater)
+                    .put("active_room_clocks", rooms.values().stream().filter(r -> r.clock() != null).count())
+                    .put("room_limit", ROOM_LIMIT).put("connection_limit", CONNECTION_LIMIT).put("total_input_limit", TOTAL_INPUT_LIMIT)
+                    .put("clock_task_queued", clockQueued.get());
+            var roomViews = result.putArray("rooms"); rooms.values().forEach(r -> roomViews.add(r.view()));
             return result;
         });
     }
@@ -515,6 +587,9 @@ public final class ArenaServer implements AutoCloseable {
                 .put("pending_input_high_water", closedInputHighWater)
                 .put("replay_bytes", closedReplayBytes)
                 .put("active_sessions", closedActiveSessions).put("recoverable_sessions", closedRecoverableSessions)
+                .put("active_rooms", closedRoomCount).put("room_mailbox_remaining", closedRoomMailboxCount)
+                .put("pending_inputs", closedPendingInputs).put("rooms_high_water", roomsHighWater)
+                .put("total_input_high_water", totalInputHighWater).put("clock_task_queued", clockQueued.get())
                 .put("resume_credentials", closedResumeCredentials).put("bind_credentials", closedBindCredentials)
                 .put("retained_snapshots", closedSnapshotCount).put("snapshot_retention_high_water", snapshotRetentionHighWater)
                 .put("mailbox_high_water", mailboxHighWater.get()).put("outbound_high_water", outboundHighWater.get());
@@ -548,12 +623,18 @@ public final class ArenaServer implements AutoCloseable {
             sessions.clear(); recoverableSessions.clear(); closedActiveSessions = sessions.size(); closedRecoverableSessions = recoverableSessions.size();
             closedResumeCredentials = (int) allSessions.stream().filter(s -> s.resumeToken != null).count();
             closedBindCredentials = (int) allSessions.stream().filter(s -> s.bindToken != null).count();
+            for (RoomRuntime runtime : rooms.values()) {
+                runtime.close();
+                closedReplayBytes += runtime.room.replayStoredBytes();
+                closedPendingInputs += runtime.room.pendingInputCount();
+                closedRoomMailboxCount += runtime.view().path("mailbox_pending").asInt();
+                closedInputHighWater = Math.max(closedInputHighWater, runtime.room.inputHighWater());
+            }
             if (room != null) {
-                room.close();
-                closedReplayBytes = room.replayStoredBytes();
                 closedLifecycle = room.lifecycle();
-                closedInputHighWater = room.inputHighWater();
+                closedInputHighWater = Math.max(closedInputHighWater, room.inputHighWater());
             }
+            rooms.clear(); closedRoomCount = rooms.size();
             if (fixedClock != null) { fixedClock.stop(); closedClockMetrics = fixedClock.view(); }
             return null;
         });
diff --git a/src/main/java/arena/FixedTickClock.java b/src/main/java/arena/FixedTickClock.java
index 7ba1b83..5213bf1 100644
--- a/src/main/java/arena/FixedTickClock.java
+++ b/src/main/java/arena/FixedTickClock.java
@@ -22,19 +22,25 @@ final class FixedTickClock {
 
     static LongSupplier systemMonotonicSource() { return System::nanoTime; }
 
-    int poll() {
+    int poll() { return poll(monotonicNanos.getAsLong(), true); }
+
+    /** A held Room still accrues the shared clock; only its simulation service is deferred. */
+    int poll(long now, boolean serviceReady) {
         if (!active) throw new IllegalStateException("clock stopped");
-        long now = monotonicNanos.getAsLong();
         long elapsed = now - previousNanos;
         if (elapsed < 0) throw new IllegalStateException("monotonic source moved backwards");
         previousNanos = now;
         accumulatorNanos = Math.addExact(accumulatorNanos, elapsed);
-        lastTicks = (int) Math.min(accumulatorNanos / TICK_NANOS, MAX_CATCH_UP_TICKS);
+        lastTicks = serviceReady ? (int) Math.min(accumulatorNanos / TICK_NANOS, MAX_CATCH_UP_TICKS) : 0;
         accumulatorNanos -= lastTicks * TICK_NANOS;
         maxTicks = Math.max(maxTicks, lastTicks);
         return lastTicks;
     }
 
+    long dueTicks() { return accumulatorNanos / TICK_NANOS; }
+    long observedNanos() { return previousNanos; }
+    int lastTicks() { return lastTicks; }
+
     void stop() { active = false; accumulatorNanos = 0; lastTicks = 0; }
 
     ObjectNode view() {
diff --git a/src/main/java/arena/Room.java b/src/main/java/arena/Room.java
index 62e12c6..7c8efdd 100644
--- a/src/main/java/arena/Room.java
+++ b/src/main/java/arena/Room.java
@@ -77,6 +77,7 @@ final class Room {
     Status status() { assertOwner(); return status; }
     int executedTicks() { assertOwner(); return executedTicks; }
     int inputHighWater() { assertOwner(); return inputHighWater; }
+    int pendingInputCount() { assertOwner(); return players.values().stream().mapToInt(p -> p.pending.size()).sum(); }
     List<String> lifecycle() { assertOwner(); return List.copyOf(lifecycle); }
     Player player(String id) { assertOwner(); return players.get(id); }
 
diff --git a/src/main/java/arena/RoomRuntime.java b/src/main/java/arena/RoomRuntime.java
new file mode 100644
index 0000000..573b67f
--- /dev/null
+++ b/src/main/java/arena/RoomRuntime.java
@@ -0,0 +1,98 @@
+package arena;
+
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import java.util.ArrayDeque;
+import java.util.function.LongSupplier;
+
+/** Independent Room state/mailbox/clock, serialized on the instance's bounded owner executor. */
+final class RoomRuntime {
+    static final int COMMAND_QUANTUM = 64;
+    static final int MISSED_DEADLINE_LIMIT = 20;
+    final Room room;
+    private final ArrayDeque<Runnable> mailbox = new ArrayDeque<>();
+    private final LongSupplier monotonicNanos;
+    private FixedTickClock clock;
+    private long startedNanos;
+    private long countedDeadlines;
+    private long serviceEligibleNanos;
+    private int missedDeadlines;
+    private boolean overloaded;
+    private int mailboxHighWater;
+    private int lastDrain;
+    private int maxDrain;
+    private long admittedCommands;
+    private long drainedCommands;
+    private int terminalPendingCleared;
+    private boolean closed;
+
+    RoomRuntime(Room room, LongSupplier monotonicNanos) { this.room = room; this.monotonicNanos = monotonicNanos; }
+
+    boolean offer(Runnable command) {
+        room.assertOwner();
+        if (closed || mailbox.size() == ArenaServer.MAILBOX_LIMIT) return false;
+        mailbox.addLast(command); admittedCommands++;
+        mailboxHighWater = Math.max(mailboxHighWater, mailbox.size());
+        return true;
+    }
+
+    void drain() {
+        room.assertOwner(); lastDrain = 0;
+        while (!closed && !mailbox.isEmpty() && lastDrain < COMMAND_QUANTUM) {
+            Runnable command = mailbox.removeFirst(); lastDrain++; drainedCommands++; command.run();
+        }
+        maxDrain = Math.max(maxDrain, lastDrain);
+    }
+
+    boolean start() {
+        room.assertOwner();
+        if (closed || clock != null || room.status() != Room.Status.RUNNING) return false;
+        clock = new FixedTickClock(monotonicNanos);
+        startedNanos = clock.observedNanos(); serviceEligibleNanos = startedNanos; return true;
+    }
+
+    FixedTickClock clock() { room.assertOwner(); return clock; }
+
+    int service(long now) {
+        room.assertOwner();
+        if (closed || clock == null || room.status() != Room.Status.RUNNING) return 0;
+        return clock.poll(now, now >= serviceEligibleNanos);
+    }
+
+    /** Called after this iteration's eligible service, once per real monotonic deadline. */
+    boolean deadlineMissed() {
+        room.assertOwner();
+        if (closed || clock == null || room.status() != Room.Status.RUNNING) return false;
+        long deadlines = (clock.observedNanos() - startedNanos) / FixedTickClock.TICK_NANOS;
+        long newlyDue = deadlines - countedDeadlines;
+        countedDeadlines = deadlines;
+        if (clock.dueTicks() == 0) { missedDeadlines = 0; overloaded = false; }
+        else {
+            missedDeadlines = (int) Math.min(MISSED_DEADLINE_LIMIT, missedDeadlines + newlyDue);
+            if (clock.lastTicks() == FixedTickClock.MAX_CATCH_UP_TICKS) overloaded = true;
+        }
+        return missedDeadlines == MISSED_DEADLINE_LIMIT;
+    }
+
+    void close() {
+        room.assertOwner();
+        if (closed) return;
+        closed = true; terminalPendingCleared = room.pendingInputCount(); mailbox.clear(); room.close();
+        if (clock != null) clock.stop();
+    }
+
+    ObjectNode view() {
+        room.assertOwner();
+        ObjectNode result = Json.MAPPER.createObjectNode().put("room_id", room.id).put("owner_thread", Thread.currentThread().getName())
+                .put("status", room.status().name()).put("executed_ticks", room.executedTicks()).put("scheduled", clock != null && !closed)
+                .put("service_eligible_ns", serviceEligibleNanos).put("counted_deadlines", countedDeadlines)
+                .put("consecutive_missed_deadlines", missedDeadlines).put("overloaded", overloaded)
+                .put("operational_state", closed ? "STOPPED" : overloaded ? "OVERLOADED" : "READY")
+                .put("mailbox_pending", mailbox.size()).put("mailbox_high_water", mailboxHighWater)
+                .put("last_mailbox_drain", lastDrain).put("max_mailbox_drain", maxDrain)
+                .put("admitted_commands", admittedCommands).put("drained_commands", drainedCommands)
+                .put("pending_inputs", room.pendingInputCount()).put("pending_input_high_water", room.inputHighWater())
+                .put("terminal_pending_cleared", terminalPendingCleared).put("replay_bytes", room.replayStoredBytes()).put("closed", closed);
+        if (clock != null) result.set("clock", clock.view());
+        return result;
+    }
+}
diff --git a/src/test/java/arena/ManyRoomScenario.java b/src/test/java/arena/ManyRoomScenario.java
new file mode 100644
index 0000000..f6fe70a
--- /dev/null
+++ b/src/test/java/arena/ManyRoomScenario.java
@@ -0,0 +1,434 @@
+package arena;
+
+import com.fasterxml.jackson.databind.JsonNode;
+import com.fasterxml.jackson.databind.node.ArrayNode;
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import io.netty.buffer.ByteBuf;
+import io.netty.channel.Channel;
+import io.netty.channel.ChannelHandlerContext;
+import io.netty.channel.ChannelOutboundHandlerAdapter;
+import io.netty.channel.ChannelPromise;
+import io.netty.channel.nio.NioEventLoopGroup;
+import io.netty.channel.socket.DatagramPacket;
+import java.io.IOException;
+import java.lang.reflect.Field;
+import java.net.InetSocketAddress;
+import java.nio.charset.StandardCharsets;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.util.ArrayList;
+import java.util.Collections;
+import java.util.LinkedHashMap;
+import java.util.List;
+import java.util.Map;
+import java.util.concurrent.TimeUnit;
+import java.util.concurrent.atomic.AtomicLong;
+
+/** Test runtime only: one frozen instance, actual transports and one shared manual clock. */
+final class ManyRoomScenario {
+    static final String SHA256 = "ad6304b939483b4498a884ebe62681d1d3bea8810bb2518642c5f5efc9b1e70b";
+    record Observed(ObjectNode result, List<byte[]> artifacts) { }
+    private ManyRoomScenario() { }
+
+    private static final class LiveRoom {
+        final ObjectNode initial, result;
+        final Map<String, TcpClient> clients = new LinkedHashMap<>();
+        final Map<String, AckScenario.Replica> replicas = new LinkedHashMap<>();
+        final Map<String, OutboundQueue> queues = new LinkedHashMap<>();
+        final Map<String, Integer> receivedSnapshots = new LinkedHashMap<>();
+        RoomRuntime runtime;
+        String initialRecord;
+        LiveRoom(ObjectNode initial, ObjectNode result) {
+            this.initial = initial; this.result = result;
+            result.put("room_id", initial.path("room_id").asText());
+            result.putArray("states"); result.putArray("canonical_records"); result.putArray("state_hashes");
+            result.putArray("admissions"); result.putArray("snapshots");
+        }
+        String id() { return initial.path("room_id").asText(); }
+    }
+
+    static Observed run(Path path) throws Exception {
+        byte[] bytes = Files.readAllBytes(path);
+        if (!SHA256.equals(UdpScenario.hash(bytes))) throw new IOException("frozen G13 bytes required");
+        ObjectNode fixture = Json.read(bytes), result = Json.MAPPER.createObjectNode().put("thread", "G13")
+                .put("scenario_sha256", SHA256).put("process_id", ProcessHandle.current().pid()).put("mode", "LIVE_G13")
+                .put("server_instances", 1).put("clock", "one shared manual monotonic source")
+                .put("overload_observation", "RoomRuntime operational_state is authoritative; nested clock.overloaded reports raw accumulator backlog, including deliberate holds")
+                .put("fault_campaigns", 0).put("load_runs", 0).put("profiler_runs", 0);
+        ArrayNode failures = result.putArray("failures"), clockTrace = result.putArray("clock_trace"), burstTrace = result.putArray("hot_bursts");
+        AtomicLong now = new AtomicLong(); ArenaServer server = new ArenaServer("127.0.0.1", 0, now::get);
+        List<LiveRoom> rooms = new ArrayList<>(); List<byte[]> artifacts = new ArrayList<>(Collections.nCopies(32, null));
+        Map<String, TcpClient> clients = new LinkedHashMap<>(); WriteProbe probe = null;
+        try {
+            for (JsonNode source : fixture.withArray("rooms")) {
+                ObjectNode initial = source.deepCopy(); LiveRoom room = new LiveRoom(initial, result.withArray("rooms").addObject()); rooms.add(room);
+                for (JsonNode player : initial.withArray("players")) {
+                    String role = player.path("player_id").asText(); ((ObjectNode) player).put("client", role);
+                    TcpClient client = new TcpClient(server.port()); clients.put(role, client); room.clients.put(role, client);
+                    room.replicas.put(role, new AckScenario.Replica()); room.receivedSnapshots.put(role, 0); client.hello();
+                }
+            }
+            probe = new WriteProbe(server, clients);
+            for (LiveRoom room : rooms) {
+                room.result.set("bootstrap", ReplayFixture.bootstrap(server, room.initial, room.clients));
+                room.runtime = ReplayFixture.owned(server, () -> ReplayFixture.runtime(server, room.id()));
+                room.initialRecord = ReplayFixture.owned(server, room.runtime.room::canonicalRecord);
+                room.result.put("initial_canonical_record", room.initialRecord);
+                room.result.set("owner_mapping", ReplayFixture.owned(server, room.runtime::view));
+                for (String role : room.clients.keySet()) room.queues.put(role, probe.queues.get(role));
+            }
+            require(rooms.size() == 32 && rooms.stream().map(r -> r.runtime).distinct().count() == 32, "distinct32 runtime ownership");
+            captureSnapshots(server, rooms);
+            result.set("initialized_metrics", server.metrics());
+            LiveRoom hot = rooms.getFirst(); eligibleAt(server, hot, 225);
+            burst(server, hot, 0, now, burstTrace);
+            int serviceOrdinal = 0;
+            for (JsonNode event : fixture.path("clock").withArray("event_times_ms")) {
+                int time = event.asInt(); boolean deadline = contains(fixture.path("clock").withArray("deadlines_ms"), time);
+                boolean service = contains(fixture.path("clock").withArray("room0_service_ms"), time);
+                if (deadline) for (int index = 1; index < rooms.size(); index++) normalInputs(server, rooms.get(index), time / 50 - 1, time, now.get());
+                ObjectNode beforeHot = ReplayFixture.owned(server, () -> ReplayFixture.view(hot.runtime.room));
+                String beforeHotRecord = time == 1200 ? ReplayFixture.owned(server, hot.runtime.room::canonicalRecord) : null;
+                now.set(TimeUnit.MILLISECONDS.toNanos(time)); server.runClockIteration();
+                ObjectNode actual = server.metrics(), cell = clockTrace.addObject().put("time_ms", time).put("actual_deadline", deadline).put("hot_service_opportunity", service);
+                cell.set("rooms", actual.path("rooms")); cell.set("hot_runtime", ReplayFixture.owned(server, hot.runtime::view));
+                cell.put("active_rooms", actual.path("active_rooms").asInt()).put("active_room_clocks", actual.path("active_room_clocks").asInt())
+                        .put("global_mailbox_high_water", actual.path("mailbox_high_water").asInt()).put("total_pending_inputs", actual.path("pending_inputs").asInt());
+                for (int index = 1; index < rooms.size(); index++) {
+                    LiveRoom room = rooms.get(index);
+                    ObjectNode state = ReplayFixture.owned(server, () -> ReplayFixture.view(room.runtime.room));
+                    require(state.path("executed_ticks").asInt() == time / 50, "normal independent clock " + room.id() + "/" + time);
+                    if (deadline) recordTick(room, state, ReplayFixture.owned(server, room.runtime.room::canonicalRecord));
+                }
+                if (service) {
+                    serviceOrdinal++; hotJournal(server, hot);
+                    ObjectNode after = ReplayFixture.owned(server, () -> ReplayFixture.view(hot.runtime.room));
+                    cell.set("hot_state_after_service", after); hot.result.withArray("states").add(after);
+                    authority(after, serviceOrdinal * 4, 64 * (serviceOrdinal - 1) + 4, 0);
+                    require(after.path("executed_ticks").asInt() - beforeHot.path("executed_ticks").asInt() == 4, "one capped hot quantum");
+                }
+                require(cell.path("hot_runtime").path("executed_ticks").asInt() == 4 * serviceOrdinal, "no hidden hot service");
+                int expectedMisses = time <= 200 ? time / 50 : time == 225 ? 0 : Math.min(20, time / 50 - 4);
+                require(cell.path("hot_runtime").path("consecutive_missed_deadlines").asInt() == expectedMisses, "actual consecutive deadline accounting " + time);
+                require(cell.path("hot_runtime").path("overloaded").asBoolean() == (time >= 450), "overload operational flag " + time);
+                if (time < 1200) require(cell.path("hot_runtime").path("status").asText().equals("RUNNING"), "operational state replaced lifecycle");
+                captureSnapshots(server, rooms);
+                if (service) {
+                    burst(server, hot, serviceOrdinal, now, burstTrace); eligibleAt(server, hot, time + 225);
+                    if (serviceOrdinal == 5) artifacts.set(0, ReplayFixture.owned(server, hot.runtime.room::replayArtifact));
+                }
+                if (time == 1200) {
+                    ObjectNode terminal = result.putObject("hot_terminal").put("at_ms", time);
+                    terminal.set("state_before", beforeHot); terminal.set("state_after", ReplayFixture.owned(server, () -> ReplayFixture.view(hot.runtime.room)));
+                    terminal.put("current_canonical_record_before", beforeHotRecord).put("current_hash_before", ReplayLog.hash(beforeHotRecord))
+                            .put("state_hash_field_semantics", "last closed tick19 hash; final burst updates accepted sequence between ticks and never rewrites tick19");
+                    terminal.set("runtime_after", ReplayFixture.owned(server, hot.runtime::view));
+                    terminal.put("registered_after", ReplayFixture.owned(server, () -> ((Map<?, ?>) ReplayFixture.field(server, "rooms")).containsKey(hot.id())));
+                    for (var entry : hot.clients.entrySet()) {
+                        ObjectNode reply = entry.getValue().control(); terminal.withArray("terminal_controls").addObject().put("player", entry.getKey()).set("wire", reply);
+                        require(reply.path("code").asText().equals("ROOM_OVERLOAD"), "only ROOM_OVERLOAD terminates hot Room"); entry.getValue().expectClosed();
+                    }
+                    authority(beforeHot, 20, 324, 4);
+                    require(!terminal.path("registered_after").asBoolean() && actual.path("active_rooms").asInt() == 31
+                            && terminal.path("runtime_after").path("terminal_pending_cleared").asInt() == 16
+                            && terminal.path("runtime_after").path("pending_inputs").asInt() == 0
+                            && !terminal.path("runtime_after").path("clock").path("active").asBoolean(), "hot terminal ownership release");
+                }
+            }
+            for (int index = 0; index < rooms.size(); index++) {
+                LiveRoom room = rooms.get(index); room.result.set("runtime_before_shutdown", ReplayFixture.owned(server, room.runtime::view));
+                room.result.set("final_state", ReplayFixture.owned(server, () -> ReplayFixture.view(room.runtime.room)));
+                room.result.put("executed_ticks", room.result.path("runtime_before_shutdown").path("executed_ticks").asInt());
+                if (index > 0) {
+                    require(room.result.withArray("states").size() == 25, "normal25 raw states"); authority((ObjectNode) room.result.path("final_state"), 25, 25, 0);
+                    artifacts.set(index, ReplayFixture.owned(server, room.runtime.room::replayArtifact));
+                    for (String role : room.clients.keySet()) require(room.receivedSnapshots.get(role) == 13, "normal start plus12 snapshot cadence");
+                }
+                ObjectNode transport = room.result.putObject("transport_before_shutdown"); room.queues.forEach((role, queue) -> transport.set(role, queue.view()));
+            }
+            result.put("hot_accepted", 0).put("hot_rate_rejections", 0);
+            for (JsonNode burst : burstTrace) {
+                result.put("hot_accepted", result.path("hot_accepted").asInt() + burst.path("accepted").asInt());
+                result.put("hot_rate_rejections", result.path("hot_rate_rejections").asInt() + burst.path("rate_rejections").asInt());
+            }
+            require(result.path("hot_accepted").asInt() == 96 && result.path("hot_rate_rejections").asInt() == 1440, "fixed hot admission totals");
+            result.set("runtime_metrics", server.metrics());
+            server.close();
+            for (int index = 1; index < rooms.size(); index++) for (TcpClient client : rooms.get(index).clients.values()) client.expectClosed();
+        } catch (Exception failure) {
+            failures.add(failure.getClass().getName() + ": " + failure.getMessage());
+            java.io.StringWriter trace = new java.io.StringWriter(); failure.printStackTrace(new java.io.PrintWriter(trace)); result.put("execution_error", trace.toString());
+        } finally {
+            try { server.close(); }
+            catch (Exception failure) { cleanupFailure(result, failures, "server close", failure); }
+            for (var entry : clients.entrySet()) {
+                try { entry.getValue().close(); }
+                catch (Exception failure) { cleanupFailure(result, failures, "client close " + entry.getKey(), failure); }
+            }
+            if (probe != null) {
+                try {
+                    require(server.cleanup().path("event_loops_terminated").asBoolean(), "transport observation requires terminated event loops");
+                    result.set("observed_transport_cleanup", probe.finish());
+                } catch (Exception failure) { cleanupFailure(result, failures, "transport observation", failure); }
+            }
+        }
+        ObjectNode cleanup = server.cleanup(); result.set("cleanup", cleanup);
+        try { ScenarioRunner.assertCleanup(cleanup); }
+        catch (Exception failure) { cleanupFailure(result, failures, "server cleanup assertions", failure); }
+        int ticks = 0, normalTicks = 0;
+        for (int i = 0; i < rooms.size(); i++) {
+            LiveRoom room = rooms.get(i); int count = room.result.withArray("state_hashes").size(); ticks += count; if (i > 0) normalTicks += count;
+            if (room.runtime != null) {
+                try {
+                    require(cleanup.path("owner_terminated").asBoolean(), "Room cleanup observation requires terminated owner");
+                    // The joined owner can no longer mutate these objects; keep every production owner guard intact.
+                    int pending = 0;
+                    for (Object player : ((Map<?, ?>) ReplayFixture.field(room.runtime.room, "players")).values()) pending += ((java.util.Deque<?>) ReplayFixture.field(player, "pending")).size();
+                    ReplayLog replay = (ReplayLog) ReplayFixture.field(room.runtime.room, "replay");
+                    int replayBytes = replay == null ? 0 : replay.storedBytes();
+                    Room.Status status = (Room.Status) ReplayFixture.field(room.runtime.room, "status");
+                    room.result.putObject("cleanup").put("status", status.toString()).put("pending_inputs", pending)
+                            .put("replay_bytes", replayBytes).put("mailbox_pending", ((java.util.Deque<?>) ReplayFixture.field(room.runtime, "mailbox")).size());
+                    if (pending != 0 || replayBytes != 0 || status != Room.Status.CLOSED) failures.add("Room cleanup " + room.id());
+                } catch (Exception failure) { cleanupFailure(result, failures, "Room cleanup " + room.id(), failure); }
+            }
+        }
+        boolean released = clients.values().stream().allMatch(TcpClient::isClosed)
+                && result.path("observed_transport_cleanup").path("live_buffers").asInt(-1) == 0
+                && result.path("observed_transport_cleanup").path("pending_writes").asInt(-1) == 0
+                && result.path("observed_transport_cleanup").path("lifetime_violations").asInt(-1) == 0
+                && result.path("observed_transport_cleanup").path("ownership_bound_violations").asInt(-1) == 0
+                && server.cleanup().path("active_rooms").asInt(-1) == 0 && server.cleanup().path("pending_inputs").asInt(-1) == 0
+                && server.cleanup().path("room_mailbox_remaining").asInt(-1) == 0 && !server.cleanup().path("clock_task_queued").asBoolean();
+        result.put("executed_ticks", ticks).put("normal_ticks", normalTicks).put("all_resources_released", released)
+                .put("passed", failures.isEmpty() && released && ticks == 795 && normalTicks == 775);
+        return new Observed(result, artifacts);
+    }
+
+    private static void cleanupFailure(ObjectNode result, ArrayNode failures, String phase, Exception failure) {
+        failures.add(phase + ": " + failure.getClass().getName() + ": " + failure.getMessage());
+        java.io.StringWriter trace = new java.io.StringWriter(); failure.printStackTrace(new java.io.PrintWriter(trace));
+        result.withArray("cleanup_errors").addObject().put("phase", phase).put("error", trace.toString());
+    }
+
+    private static boolean contains(ArrayNode values, int time) { for (JsonNode value : values) if (value.asInt() == time) return true; return false; }
+
+    private static ObjectNode input(TcpClient client, int seq, int tick, String direction) {
+        return client.auth("INPUT", client.roomId).put("seq", seq).put("target_tick", tick).put("direction", direction).putNull("tag_target_player_id");
+    }
+    private static ObjectNode aliased(ObjectNode message, String player) {
+        ObjectNode safe = message.deepCopy(); safe.remove("session_id"); safe.put("session_alias", player); return safe;
+    }
+    private static void normalInputs(ArenaServer server, LiveRoom room, int tick, int deadline, long clock) throws Exception {
+        require(ReplayFixture.owned(server, room.runtime.room::executedTicks) == tick, "normal input boundary");
+        int received = ReplayFixture.udpReceived(server); Map<String, ObjectNode> cells = new LinkedHashMap<>();
+        for (JsonNode player : room.initial.withArray("players")) {
+            String id = player.path("player_id").asText(); TcpClient client = room.clients.get(id);
+            ObjectNode request = input(client, tick + 1, tick, player.path("direction").asText());
+            ObjectNode cell = room.result.withArray("admissions").addObject().put("before_deadline_ms", deadline)
+                    .put("at_monotonic_ns", clock).put("player", id); cell.set("request", aliased(request, id)); cells.put(id, cell); client.send(request);
+        }
+        for (var entry : room.clients.entrySet()) {
+            ObjectNode reply = entry.getValue().control(); cells.get(entry.getKey()).set("reply", reply);
+            require(reply.path("type").asText().equals("INPUT_ACK") && reply.path("status").asText().equals("ACCEPTED")
+                    && reply.path("seq").asInt() == tick + 1, "normal actual accepted INPUT " + room.id());
+        }
+        ReplayFixture.udpBarrier(server, received + 4);
+    }
+
+    private static void burst(ArenaServer server, LiveRoom hot, int ordinal, AtomicLong now, ArrayNode trace) throws Exception {
+        int tick = ordinal * 4; long fixedTime = now.get();
+        require(ReplayFixture.owned(server, hot.runtime.room::executedTicks) == tick, "hot burst real next tick");
+        ObjectNode burst = trace.addObject().put("ordinal", ordinal).put("at_ms", TimeUnit.NANOSECONDS.toMillis(fixedTime)).put("target_tick", tick);
+        int accepted = 0, rejected = 0;
+        for (int group = 0; group < 16; group++) {
+            ObjectNode cell = burst.withArray("transport_groups").addObject().put("ordinal", group);
+            int received = ReplayFixture.udpReceived(server); cell.put("udp_received_before", received);
+            for (JsonNode player : hot.initial.withArray("players")) {
+                String id = player.path("player_id").asText(); TcpClient client = hot.clients.get(id);
+                for (int offset = 1; offset <= 4; offset++) {
+                    ObjectNode request = input(client, ordinal * 64 + group * 4 + offset, tick, player.path("direction").asText());
+                    cell.withArray("sent_in_order").add(aliased(request, id)); client.send(request);
+                }
+            }
+            for (var entry : hot.clients.entrySet()) for (int offset = 1; offset <= 4; offset++) {
+                ObjectNode reply = entry.getValue().control(); int seq = ordinal * 64 + group * 4 + offset;
+                cell.withArray("replies_in_observed_peer_order").addObject().put("player", entry.getKey()).put("request_seq", seq).set("wire", reply);
+                if (group == 0) {
+                    require(reply.path("type").asText().equals("INPUT_ACK") && reply.path("status").asText().equals("ACCEPTED")
+                            && reply.path("seq").asInt() == seq, "first four hot attempts accepted"); accepted++;
+                } else { require(reply.path("code").asText().equals("INPUT_RATE_EXCEEDED"), "remaining hot attempts rate rejected"); rejected++; }
+            }
+            ReplayFixture.udpBarrier(server, received + 16); cell.put("udp_received_after", ReplayFixture.udpReceived(server));
+            cell.set("actual_room_queue_after_drain", ReplayFixture.owned(server, hot.runtime::view));
+            require(now.get() == fixedTime && ReplayFixture.owned(server, hot.runtime.room::executedTicks) == tick, "group advanced simulation/clock");
+        }
+        burst.put("accepted", accepted).put("rate_rejections", rejected);
+        ObjectNode state = ReplayFixture.owned(server, () -> ReplayFixture.view(hot.runtime.room)); burst.set("state_after", state);
+        for (JsonNode player : state.withArray("players")) require(player.path("last_accepted_seq").asInt() == ordinal * 64 + 4
+                && player.path("pending_inputs").asInt() == 4, "hot accepted pending storage");
+        require(accepted == 16 && rejected == 240, "one fixed burst totals");
+    }
+
+    private static void recordTick(LiveRoom room, ObjectNode state, String record) throws IOException {
+        int tick = room.result.withArray("states").size();
+        require(state.path("tick").asInt() == tick && state.path("state_hash").asText().equals(ReplayLog.hash(record)), "normal raw canonical tick/hash");
+        authority(state, tick + 1, tick + 1, 0); room.result.withArray("states").add(state);
+        room.result.withArray("canonical_records").add(record); room.result.withArray("state_hashes").add(state.path("state_hash"));
+    }
+
+    private static void hotJournal(ArenaServer server, LiveRoom hot) throws Exception {
+        byte[] artifact = ReplayFixture.owned(server, hot.runtime.room::replayArtifact);
+        for (String line : new String(artifact, StandardCharsets.UTF_8).split("\n")) {
+            ObjectNode row = Json.read(line.getBytes(StandardCharsets.UTF_8)); if (!row.path("kind").asText().equals("TICK")) continue;
+            int tick = row.path("tick").asInt(); if (tick < hot.result.withArray("state_hashes").size()) continue;
+            require(tick == hot.result.withArray("state_hashes").size() && row.path("expected_hash").asText().equals(ReplayLog.hash(row.path("canonical_record").asText())), "hot actual journal tick continuity");
+            hot.result.withArray("canonical_records").add(row.path("canonical_record")); hot.result.withArray("state_hashes").add(row.path("expected_hash"));
+        }
+        hot.result.put("tick_record_source", "unchanged actual owner-written journal; no replay execution");
+        hot.result.put("state_capture_boundaries", "five actual service endpoints; all20 intervening tick canonical records are retained");
+    }
+
+    private static void captureSnapshots(ArenaServer server, List<LiveRoom> rooms) throws Exception {
+        awaitTransport(server, rooms); int received = ReplayFixture.udpReceived(server), acknowledgements = 0;
+        Map<TcpClient, ObjectNode> lastAcknowledgements = new LinkedHashMap<>();
+        for (LiveRoom room : rooms) for (var entry : room.clients.entrySet()) {
+            String role = entry.getKey(); TcpClient client = entry.getValue(); int sent = room.queues.get(role).view().path("sent_snapshots").asInt();
+            while (room.receivedSnapshots.get(role) < sent) {
+                ObjectNode wire = client.until("SNAPSHOT"); int tick = wire.path("tick").asInt();
+                String record = tick == -1 ? room.initialRecord : room.result.withArray("canonical_records").path(tick).asText();
+                AckScenario.Replica replica = room.replicas.get(role); String application = replica.apply(wire);
+                ObjectNode cell = room.result.withArray("snapshots").addObject().put("player", role).put("actual_received", true)
+                        .put("application", application).put("bytes", Json.bytes(wire).length); cell.set("wire", wire);
+                require(application.equals("APPLIED") && wire.path("room_id").asText().equals(room.id()) && !record.isEmpty()
+                        && wire.path("state_hash").asText().equals(ReplayLog.hash(record)) && replica.state.equals(visibleRecord(record))
+                        && Json.bytes(wire).length <= 1200, "actual snapshot/hash/replica " + role + "/" + tick);
+                ObjectNode ack = client.auth("SNAPSHOT_ACK", client.roomId).put("snapshot_seq", replica.sequence).put("state_hash", wire.path("state_hash").asText());
+                cell.set("actual_ack", aliased(ack, role)); client.send(ack); acknowledgements++;
+                lastAcknowledgements.put(client, cell); room.receivedSnapshots.put(role, room.receivedSnapshots.get(role) + 1);
+            }
+        }
+        if (acknowledgements > 0) ReplayFixture.udpBarrier(server, received + acknowledgements);
+        for (var entry : lastAcknowledgements.entrySet()) {
+            long watermark = AckScenario.stream(server, entry.getKey()).path("acknowledged_seq").asLong();
+            entry.getValue().put("server_ack_watermark_after_batch", watermark);
+            require(watermark == entry.getValue().path("actual_ack").path("snapshot_seq").asLong(), "only received/applied snapshot ACK watermark");
+        }
+    }
+
+    private static void awaitTransport(ArenaServer server, List<LiveRoom> rooms) throws Exception {
+        NioEventLoopGroup io = (NioEventLoopGroup) ReplayFixture.field(server, "ioLoop"); long deadline = System.nanoTime() + TimeUnit.SECONDS.toNanos(5);
+        while (true) {
+            io.submit(() -> { }).get(5, TimeUnit.SECONDS); boolean idle = true;
+            for (LiveRoom room : rooms) for (OutboundQueue queue : room.queues.values()) {
+                ObjectNode view = queue.view();
+                require(view.path("pending_ordered").asInt() <= 64 && view.path("pending_full").asInt() <= 1
+                        && view.path("pending_delta").asInt() <= 1 && view.path("transport_pending_buffers").asInt() <= 1, "retained+transport bound");
+                if (view.path("retained_bytes").asInt() != 0 || view.path("flush_task_pending").asBoolean()) idle = false;
+            }
+            if (idle) return;
+            if (System.nanoTime() >= deadline) throw new IOException("real nonblocking transport drain barrier ceiling");
+        }
+    }
+
+    /** Projection of captured canonical bytes, not a second simulation or reference tick. */
+    private static ArrayNode visibleRecord(String record) {
+        ArrayNode players = Json.MAPPER.createArrayNode(); String[] lines = record.split("\n");
+        for (int index = 1; index < lines.length; index++) {
+            Map<String, String> values = new LinkedHashMap<>();
+            for (String field : lines[index].split("\\|")) { int separator = field.indexOf('='); values.put(field.substring(0, separator), field.substring(separator + 1)); }
+            players.addObject().put("player_id", values.get("player")).put("slot", Integer.parseInt(values.get("slot")))
+                    .put("x", Integer.parseInt(values.get("x"))).put("y", Integer.parseInt(values.get("y")))
+                    .put("direction", values.get("dir")).put("score", Integer.parseInt(values.get("score"))).put("connectivity", values.get("conn"));
+        }
+        return players;
+    }
+
+    private static void authority(ObjectNode state, int ticks, int seq, int pendingPerPlayer) throws IOException {
+        require(state.path("executed_ticks").asInt() == ticks && state.path("status").asText().equals("RUNNING"), "actual Room tick/lifecycle");
+        String[] directions = {"EAST", "WEST", "SOUTH", "NORTH"};
+        for (JsonNode player : state.withArray("players")) {
+            int slot = player.path("slot").asInt(), x = Room.SPAWNS[slot][0], y = Room.SPAWNS[slot][1];
+            if (slot == 0) x += ticks * 400; else if (slot == 1) x -= ticks * 400; else if (slot == 2) y -= ticks * 400; else y += ticks * 400;
+            require(player.path("x").asInt() == x && player.path("y").asInt() == y && player.path("score").asInt() == 0
+                    && player.path("direction").asText().equals(directions[slot]) && player.path("connectivity").asText().equals("CONNECTED")
+                    && player.path("last_accepted_seq").asInt() == seq && player.path("pending_inputs").asInt() == pendingPerPlayer,
+                    "fixed integer authority " + state.path("room_id").asText() + "/" + ticks + "/" + slot);
+        }
+    }
+    private static void eligibleAt(ArenaServer server, LiveRoom room, int time) throws Exception {
+        ReplayFixture.owned(server, () -> {
+            Field eligible = RoomRuntime.class.getDeclaredField("serviceEligibleNanos"); eligible.setAccessible(true);
+            eligible.setLong(room.runtime, TimeUnit.MILLISECONDS.toNanos(time)); return null;
+        });
+    }
+
+    /** Passive lifetime observation of original ByteBufs; no retain, buffering, readiness change or rewriting. */
+    private static final class WriteProbe {
+        final Map<String, OutboundQueue> queues = new LinkedHashMap<>();
+        private final List<Write> writes = new ArrayList<>();
+        private final Map<InetSocketAddress, String> endpoints = new LinkedHashMap<>();
+        private int pending, pendingHighWater, pendingBytes, bytesHighWater, boundsViolations;
+        private static final class Write {
+            final ByteBuf buffer; final String player, type; final int bytes, initialRefs;
+            boolean completed, sent; int completionRefs;
+            Write(ByteBuf buffer, String player, String type) {
+                this.buffer = buffer; this.player = player; this.type = type; bytes = buffer.readableBytes(); initialRefs = buffer.refCnt();
+            }
+        }
+        WriteProbe(ArenaServer server, Map<String, TcpClient> clients) throws Exception {
+            Map<String, Channel> controls = ReplayFixture.owned(server, () -> {
+                Map<String, Channel> channels = new LinkedHashMap<>();
+                for (var entry : ((Map<?, ?>) ReplayFixture.field(server, "sessions")).entrySet()) for (var client : clients.entrySet())
+                    if (ReplayFixture.field(entry.getValue(), "id").equals(client.getValue().sessionId)) {
+                        queues.put(client.getKey(), (OutboundQueue) ReplayFixture.field(entry.getKey(), "outbound"));
+                        channels.put(client.getKey(), (Channel) ReplayFixture.field(entry.getKey(), "channel"));
+                    }
+                return channels;
+            });
+            require(controls.size() == clients.size(), "actual session/peer mapping");
+            for (var entry : clients.entrySet()) endpoints.put(entry.getValue().localUdpEndpoint(), entry.getKey());
+            NioEventLoopGroup io = (NioEventLoopGroup) ReplayFixture.field(server, "ioLoop"); Channel udp = (Channel) ReplayFixture.field(server, "udpListener");
+            io.submit(() -> {
+                udp.pipeline().addFirst("g13-original-buffer-observer", observer(null));
+                controls.forEach((role, channel) -> channel.pipeline().addFirst("g13-original-buffer-observer", observer(role)));
+            }).get(5, TimeUnit.SECONDS);
+        }
+        private ChannelOutboundHandlerAdapter observer(String tcpRole) {
+            return new ChannelOutboundHandlerAdapter() {
+                @Override public void write(ChannelHandlerContext context, Object message, ChannelPromise promise) throws Exception {
+                    boolean datagram = tcpRole == null; ByteBuf buffer = datagram ? ((DatagramPacket) message).content() : (ByteBuf) message;
+                    String role = datagram ? endpoints.get(((DatagramPacket) message).recipient()) : tcpRole;
+                    if (role == null || writes.size() == 8192) throw new IOException("fixed passive observation bound/mapping");
+                    int offset = datagram ? 0 : 4; byte[] bytes = new byte[buffer.readableBytes() - offset]; buffer.getBytes(buffer.readerIndex() + offset, bytes);
+                    String type = Json.read(bytes).path("type").asText(); // Only type is retained; credentials are never logged.
+                    ObjectNode queue = queues.get(role).view();
+                    if (queue.path("pending_ordered").asInt() > 64 || queue.path("pending_full").asInt() > 1
+                            || queue.path("pending_delta").asInt() > 1 || queue.path("transport_pending_buffers").asInt() > 1) boundsViolations++;
+                    Write write = new Write(buffer, role, type); writes.add(write); pending++; pendingBytes += write.bytes;
+                    pendingHighWater = Math.max(pendingHighWater, pending); bytesHighWater = Math.max(bytesHighWater, pendingBytes);
+                    promise.addListener(completion -> {
+                        write.completed = true; write.sent = completion.isSuccess(); write.completionRefs = buffer.refCnt(); pending--; pendingBytes -= write.bytes;
+                    });
+                    context.write(message, promise);
+                }
+            };
+        }
+        ObjectNode finish() {
+            ObjectNode result = Json.MAPPER.createObjectNode().put("observed_writes", writes.size()).put("pending_writes", pending)
+                    .put("pending_bytes", pendingBytes).put("pending_high_water", pendingHighWater).put("pending_bytes_high_water", bytesHighWater)
+                    .put("ownership_bound_violations", boundsViolations);
+            ObjectNode types = result.putObject("by_type"), players = result.putObject("by_player"); int live = 0, ownershipErrors = 0;
+            for (Write write : writes) {
+                types.put(write.type, types.path(write.type).asInt() + 1); players.put(write.player, players.path(write.player).asInt() + 1);
+                if (write.buffer.refCnt() != 0) live++;
+                if (write.initialRefs != 1 || !write.completed || !write.sent || write.completionRefs != 0) ownershipErrors++;
+            }
+            result.put("live_buffers", live).put("lifetime_violations", ownershipErrors).put("all_observed_refcounts_zero", live == 0);
+            ObjectNode queueCleanup = result.putObject("peer_queues"); queues.forEach((role, queue) -> queueCleanup.set(role, queue.view()));
+            return result;
+        }
+    }
+
+    private static void require(boolean condition, String message) throws IOException { if (!condition) throw new IOException(message); }
+}
diff --git a/src/test/java/arena/ReplayFixture.java b/src/test/java/arena/ReplayFixture.java
index f495560..0c818d5 100644
--- a/src/test/java/arena/ReplayFixture.java
+++ b/src/test/java/arena/ReplayFixture.java
@@ -18,21 +18,26 @@ final class ReplayFixture {
         ObjectNode observed = joinFixed(server, initial, clients);
         for (TcpClient client : clients.values()) client.bind();
         observed.put("bound_live_sessions", clients.size()).put("normal_start_evaluations", 1);
-        observed.set("state", snapshot(server)); return observed;
+        observed.set("state", snapshot(server, Json.identifier(initial, "room_id"))); return observed;
     }
 
     static ObjectNode joinFixed(ArenaServer server, ObjectNode initial, Map<String, TcpClient> clients) throws Exception {
         ObjectNode observed = Json.MAPPER.createObjectNode().put("kind", "normal TCP joins; test-build-only identifier remapping");
         var joins = observed.putArray("joins"); String fixedRoom = Json.identifier(initial, "room_id");
-        clients.values().iterator().next().createRoom();
-        owned(server, () -> { set(field(server, "room"), "id", fixedRoom); return null; });
+        String generatedRoom = clients.values().iterator().next().createRoom();
+        owned(server, () -> {
+            @SuppressWarnings("unchecked") Map<String, RoomRuntime> registry = (Map<String, RoomRuntime>) field(server, "rooms");
+            RoomRuntime runtime = registry.remove(generatedRoom);
+            if (runtime == null || registry.containsKey(fixedRoom)) throw new IOException("fixed Room identity collision");
+            set(runtime.room, "id", fixedRoom); registry.put(fixedRoom, runtime); return null;
+        });
         for (JsonNode fixture : initial.withArray("players")) {
             String role = fixture.path("client").asText(); TcpClient client = clients.get(role);
             ObjectNode joined = client.join(fixedRoom); joins.addObject().put("client", role).set("response", joined);
             if (!joined.path("status").asText().equals("LOBBY")) throw new IOException("unbound ordinary join started Room");
             String generated = client.playerId, fixed = fixture.path("player_id").asText();
             owned(server, () -> {
-                Room room = (Room) field(server, "room"); Room.Player player = room.player(generated);
+                Room room = runtime(server, fixedRoom).room; Room.Player player = room.player(generated);
                 if (player.slot != fixture.path("slot").asInt() || player.x != fixture.path("spawn").get(0).asInt()
                         || player.y != fixture.path("spawn").get(1).asInt()) throw new IOException("normal spawn differs from fixed roster");
                 @SuppressWarnings("unchecked") Map<String, Room.Player> roster = (Map<String, Room.Player>) field(room, "players");
@@ -47,7 +52,7 @@ final class ReplayFixture {
             });
             client.playerId = fixed;
         }
-        observed.set("unbound_state", snapshot(server));
+        observed.set("unbound_state", snapshot(server, fixedRoom));
         return observed;
     }
 
@@ -74,6 +79,13 @@ final class ReplayFixture {
     }
 
     static ObjectNode snapshot(ArenaServer server) throws Exception { return owned(server, () -> view((Room) field(server, "room"))); }
+    static ObjectNode snapshot(ArenaServer server, String id) throws Exception { return owned(server, () -> view(runtime(server, id).room)); }
+    static RoomRuntime runtime(ArenaServer server, String id) throws Exception {
+        @SuppressWarnings("unchecked") Map<String, RoomRuntime> registry = (Map<String, RoomRuntime>) field(server, "rooms");
+        RoomRuntime runtime = registry.get(id);
+        if (runtime == null) throw new IOException("Room absent from active registry: " + id);
+        return runtime;
+    }
     static ObjectNode view(Room room) {
         ObjectNode state = room.view("SNAPSHOT");
         state.put("state_hash", room.stateHash());
diff --git a/src/test/java/arena/ReplayTool.java b/src/test/java/arena/ReplayTool.java
index 9a9f4d0..c2b5328 100644
--- a/src/test/java/arena/ReplayTool.java
+++ b/src/test/java/arena/ReplayTool.java
@@ -33,6 +33,17 @@ public final class ReplayTool {
                     result.put("replay_artifact_path", artifact.toAbsolutePath().toString()).put("artifact_sha256",
                             HexFormat.of().formatHex(MessageDigest.getInstance("SHA-256").digest(observed.replay())));
                 }
+            } else if (thread.equals("G13")) {
+                if (args.length != 3) throw new IllegalArgumentException("G13 has one fixed 32-Room pass");
+                ManyRoomScenario.Observed observed = ManyRoomScenario.run(input); result = observed.result();
+                for (int i = 0; i < observed.artifacts().size(); i++) {
+                    byte[] bytes = observed.artifacts().get(i); if (bytes == null) continue;
+                    ObjectNode cell = (ObjectNode) result.withArray("rooms").get(i);
+                    Path artifact = output.resolveSibling(output.getFileName().toString().replaceFirst("\\.json$", "") + "." + cell.path("room_id").asText() + ".replay.jsonl");
+                    Files.write(artifact, bytes, StandardOpenOption.CREATE_NEW);
+                    cell.put("replay_artifact_path", artifact.toAbsolutePath().toString()).put("artifact_bytes", bytes.length)
+                            .put("artifact_sha256", HexFormat.of().formatHex(MessageDigest.getInstance("SHA-256").digest(bytes)));
+                }
             } else if (thread.equals("G11")) {
                 if (args.length != 3) throw new IllegalArgumentException("G11 has one fixed three-case pass");
                 ReconnectScenario.Observed observed = ReconnectScenario.run(input); result = observed.result();
diff --git a/src/test/resources/G13.json b/src/test/resources/G13.json
new file mode 100644
index 0000000..e544e77
--- /dev/null
+++ b/src/test/resources/G13.json
@@ -0,0 +1,1474 @@
+{
+  "scenario_id": "G13-hot-room-isolation",
+  "contract_version": 1,
+  "thread": "G13",
+  "seed": 7050,
+  "clock": {
+    "kind": "shared-manual-monotonic",
+    "tick_duration_ms": 50,
+    "end_ms": 1250,
+    "deadlines_ms": [
+      50,
+      100,
+      150,
+      200,
+      250,
+      300,
+      350,
+      400,
+      450,
+      500,
+      550,
+      600,
+      650,
+      700,
+      750,
+      800,
+      850,
+      900,
+      950,
+      1000,
+      1050,
+      1100,
+      1150,
+      1200,
+      1250
+    ],
+    "room0_service_ms": [
+      225,
+      450,
+      675,
+      900,
+      1125
+    ],
+    "event_times_ms": [
+      50,
+      100,
+      150,
+      200,
+      225,
+      250,
+      300,
+      350,
+      400,
+      450,
+      500,
+      550,
+      600,
+      650,
+      675,
+      700,
+      750,
+      800,
+      850,
+      900,
+      950,
+      1000,
+      1050,
+      1100,
+      1125,
+      1150,
+      1200,
+      1250
+    ]
+  },
+  "rooms": [
+    {
+      "room_id": "room-00",
+      "players": [
+        {
+          "player_id": "player-00-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-00-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-00-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-00-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    },
+    {
+      "room_id": "room-01",
+      "players": [
+        {
+          "player_id": "player-01-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-01-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-01-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-01-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    },
+    {
+      "room_id": "room-02",
+      "players": [
+        {
+          "player_id": "player-02-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-02-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-02-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-02-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    },
+    {
+      "room_id": "room-03",
+      "players": [
+        {
+          "player_id": "player-03-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-03-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-03-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-03-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    },
+    {
+      "room_id": "room-04",
+      "players": [
+        {
+          "player_id": "player-04-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-04-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-04-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-04-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    },
+    {
+      "room_id": "room-05",
+      "players": [
+        {
+          "player_id": "player-05-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-05-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-05-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-05-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    },
+    {
+      "room_id": "room-06",
+      "players": [
+        {
+          "player_id": "player-06-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-06-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-06-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-06-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    },
+    {
+      "room_id": "room-07",
+      "players": [
+        {
+          "player_id": "player-07-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-07-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-07-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-07-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    },
+    {
+      "room_id": "room-08",
+      "players": [
+        {
+          "player_id": "player-08-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-08-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-08-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-08-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    },
+    {
+      "room_id": "room-09",
+      "players": [
+        {
+          "player_id": "player-09-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-09-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-09-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-09-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    },
+    {
+      "room_id": "room-10",
+      "players": [
+        {
+          "player_id": "player-10-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-10-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-10-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-10-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    },
+    {
+      "room_id": "room-11",
+      "players": [
+        {
+          "player_id": "player-11-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-11-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-11-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-11-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    },
+    {
+      "room_id": "room-12",
+      "players": [
+        {
+          "player_id": "player-12-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-12-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-12-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-12-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    },
+    {
+      "room_id": "room-13",
+      "players": [
+        {
+          "player_id": "player-13-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-13-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-13-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-13-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    },
+    {
+      "room_id": "room-14",
+      "players": [
+        {
+          "player_id": "player-14-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-14-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-14-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-14-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    },
+    {
+      "room_id": "room-15",
+      "players": [
+        {
+          "player_id": "player-15-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-15-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-15-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-15-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    },
+    {
+      "room_id": "room-16",
+      "players": [
+        {
+          "player_id": "player-16-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-16-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-16-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-16-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    },
+    {
+      "room_id": "room-17",
+      "players": [
+        {
+          "player_id": "player-17-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-17-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-17-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-17-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    },
+    {
+      "room_id": "room-18",
+      "players": [
+        {
+          "player_id": "player-18-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-18-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-18-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-18-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    },
+    {
+      "room_id": "room-19",
+      "players": [
+        {
+          "player_id": "player-19-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-19-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-19-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-19-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    },
+    {
+      "room_id": "room-20",
+      "players": [
+        {
+          "player_id": "player-20-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-20-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-20-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-20-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    },
+    {
+      "room_id": "room-21",
+      "players": [
+        {
+          "player_id": "player-21-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-21-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-21-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-21-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    },
+    {
+      "room_id": "room-22",
+      "players": [
+        {
+          "player_id": "player-22-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-22-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-22-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-22-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    },
+    {
+      "room_id": "room-23",
+      "players": [
+        {
+          "player_id": "player-23-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-23-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-23-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-23-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    },
+    {
+      "room_id": "room-24",
+      "players": [
+        {
+          "player_id": "player-24-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-24-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-24-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-24-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    },
+    {
+      "room_id": "room-25",
+      "players": [
+        {
+          "player_id": "player-25-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-25-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-25-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-25-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    },
+    {
+      "room_id": "room-26",
+      "players": [
+        {
+          "player_id": "player-26-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-26-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-26-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-26-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    },
+    {
+      "room_id": "room-27",
+      "players": [
+        {
+          "player_id": "player-27-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-27-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-27-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-27-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    },
+    {
+      "room_id": "room-28",
+      "players": [
+        {
+          "player_id": "player-28-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-28-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-28-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-28-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    },
+    {
+      "room_id": "room-29",
+      "players": [
+        {
+          "player_id": "player-29-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-29-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-29-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-29-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    },
+    {
+      "room_id": "room-30",
+      "players": [
+        {
+          "player_id": "player-30-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-30-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-30-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-30-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    },
+    {
+      "room_id": "room-31",
+      "players": [
+        {
+          "player_id": "player-31-0",
+          "slot": 0,
+          "spawn": [
+            10000,
+            10000
+          ],
+          "direction": "EAST"
+        },
+        {
+          "player_id": "player-31-1",
+          "slot": 1,
+          "spawn": [
+            90000,
+            90000
+          ],
+          "direction": "WEST"
+        },
+        {
+          "player_id": "player-31-2",
+          "slot": 2,
+          "spawn": [
+            10000,
+            90000
+          ],
+          "direction": "SOUTH"
+        },
+        {
+          "player_id": "player-31-3",
+          "slot": 3,
+          "spawn": [
+            90000,
+            10000
+          ],
+          "direction": "NORTH"
+        }
+      ]
+    }
+  ],
+  "initialization": "one server instance; normal create/join and four UDP binds per Room; only fixed server-generated identifiers are test-build injected",
+  "normal_inputs": {
+    "room_indices": [
+      1,
+      31
+    ],
+    "before_each_deadline": "all four players send one INPUT for upcoming local tick0..24",
+    "seq": "tick+1",
+    "target_tick": "tick",
+    "direction": "slot direction in rooms mapping",
+    "tag_target_player_id": null,
+    "owner_epoch": 0
+  },
+  "hot_room": {
+    "index": 0,
+    "simulation_service_hold_ms": 225,
+    "hold_effect": "only simulation service; ingress, I/O, deadline accounting and other Rooms continue; never advance an independent Room clock",
+    "max_ticks_per_service": 4,
+    "burst_at_ms": [
+      0,
+      225,
+      450,
+      675,
+      900,
+      1125
+    ],
+    "target_ticks": [
+      0,
+      4,
+      8,
+      12,
+      16,
+      20
+    ],
+    "attempts_per_player_per_burst": 64,
+    "sequence": "64*burst_index+ordinal1..64",
+    "direction": "slot direction in rooms mapping",
+    "transport_groups": "16 fixed groups per burst; each group sends next4 attempts per player through real UDP, drains actual owner ingress and consumes all ACK/TCPERROR before next group, without advancing simulation time/tick",
+    "expected_per_player_burst": {
+      "accepted": 4,
+      "INPUT_RATE_EXCEEDED": 60
+    },
+    "overload_deadline_rule": "after eligible service at each actual50ms deadline, increment streak if still behind; reset whenever fully caught up even between deadlines",
+    "last_recovery_ms": 225,
+    "uninterrupted_missed_deadlines_ms": [
+      250,
+      300,
+      350,
+      400,
+      450,
+      500,
+      550,
+      600,
+      650,
+      700,
+      750,
+      800,
+      850,
+      900,
+      950,
+      1000,
+      1050,
+      1100,
+      1150,
+      1200
+    ],
+    "expected_terminal_ms": 1200,
+    "expected_terminal_code": "ROOM_OVERLOAD",
+    "expected_executed_ticks": 20,
+    "expected_total_accepted": 96,
+    "expected_total_rate_rejections": 1440,
+    "expected_pending_inputs_cleared_at_terminal": 16
+  },
+  "expected_normal_ticks": 25,
+  "reference": "one offline accepted-journal replay for each normalRoom,25ticks each; compare all775 per-tick hashes",
+  "resource_limits": {
+    "rooms": 64,
+    "connections": 512,
+    "accepted_pending_per_player": 64,
+    "total_accepted_pending": 32768,
+    "catchup_per_room_per_iteration": 4
+  },
+  "socket_ceiling_ms": 5000,
+  "additional_load_runs": 0,
+  "extra_fault_campaigns": 0
+}
