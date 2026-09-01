# Slow Consumer와 Bounded Outbound Work (G12)

## `feat: bound outbound snapshot ownership`

diff --git a/evidence/G12-command-ledger.jsonl b/evidence/G12-command-ledger.jsonl
new file mode 100644
index 0000000..7f0e418
--- /dev/null
+++ b/evidence/G12-command-ledger.jsonl
@@ -0,0 +1,12 @@
+{"kind": "activation", "thread": "G12", "profile": "realtime-core", "spec_revision": "c1d62196ab76b55652f5d75a67514f8c6d8210ce", "start": "10325022b2774d3284544f12680ba9ba41f77149", "scenario": "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G12.json", "scenario_sha256": "b5350ef5bfb9fa93cbde1fe0fd30079e6115fe26988514be59d63d9dee6bc6f3", "at": "2026-08-28T07:16:30.335845+00:00", "ceilings": {"compiler_tasks": 8, "unit_including_baseline_and_two_pure_queue_cases": 4, "integration": 2, "post_live": 1, "post_live_ticks": 100, "reference_replay": 1, "reference_ticks": 100, "fault": 0, "load": 0, "profiler": 0, "repairs": 2}, "baseline_signal": "actual alpha Netty writability=false; START lacks per-peer readiness/dequeue queue, so observe bypass instead of inventing a hold or buffering queue"}
+{"kind": "resolved_before_execution", "pass": "baseline", "category": "unchanged-production-reproduction", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test", "--tests", "arena.G12BaselineTest"], "environment": {"ARENA_G12_SCENARIO": "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G12.json", "ARENA_G12_EVIDENCE": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-initial/reproduce-unit/result.json"}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-initial/reproduce-unit", "reserved_g12_ticks": 100}
+{"kind": "resolved_before_execution", "pass": "build", "category": "clean-build", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "build"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-initial/verify-build", "reserved_g12_ticks": 0}
+{"kind": "resolved_before_execution", "pass": "unit", "category": "full-unit-exact-two-pure-queue-regressions", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-initial/verify-unit", "reserved_g12_ticks": 0}
+{"kind": "resolved_before_execution", "pass": "integration", "category": "integration", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "integration-test"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-initial/verify-integration", "reserved_g12_ticks": 0}
+{"kind": "resolved_before_execution", "pass": "canonical", "category": "fixed-live-slow-consumer", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "scenario-run", "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G12.json", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-initial/canonical/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-initial/canonical", "reserved_g12_ticks": 100, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-initial/canonical/result.replay.jsonl"}
+{"kind": "resolved_before_execution", "pass": "reference", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-initial/canonical/result.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-initial/reference/result.json"], "environment": {}, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-initial/reference", "reserved_g12_ticks": 100}
+{"pass": "baseline", "category": "unchanged-production-reproduction", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test", "--tests", "arena.G12BaselineTest"], "environment": {"ARENA_G12_SCENARIO": "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G12.json", "ARENA_G12_EVIDENCE": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-initial/reproduce-unit/result.json"}, "kind": "executed", "started_at": "2026-08-28T07:16:51.833330+00:00", "duration_seconds": 10.986, "command_process_id": 2811, "exit_code": 1, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-initial/reproduce-unit/stdout.log", "process_terminated": true, "compiler_tasks_executed": ["> Task :compileTestJava"], "xml": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-initial/reproduce-unit/xml", "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-initial/reproduce-unit/result.json", "simulation_process_id": 3051, "executed_ticks": 100}
+{"kind": "preverification_source_audit", "at": "2026-08-28T07:41:17.112746+00:00", "source_hashes": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-initial/post-source-hashes.json", "new_pure_cases": 2, "live_campaign_tests_in_unit": 0, "same_kind_inflight_suppression": "source-reviewed; do not claim runtime coverage unless actual fixed trace count is positive", "held_terminal_threshold": "63 open;64th terminal/cancel/close within64;0 delivered when not-ready", "next_passes": ["build", "unit", "integration", "canonical", "reference"], "stop_on_first_failure": true}
+{"pass": "build", "category": "clean-build", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "build"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T07:41:28.764632+00:00", "duration_seconds": 5.158, "command_process_id": 28648, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-initial/verify-build/stdout.log", "process_terminated": true, "compiler_tasks_executed": ["> Task :compileJava", "> Task :compileTestJava"]}
+{"pass": "unit", "category": "full-unit-exact-two-pure-queue-regressions", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test"], "environment": {}, "kind": "executed", "started_at": "2026-08-28T07:41:33.923530+00:00", "duration_seconds": 4.016, "command_process_id": 28706, "exit_code": 1, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-initial/verify-unit/stdout.log", "process_terminated": true, "compiler_tasks_executed": [], "xml": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-initial/verify-unit/xml"}
+{"kind": "runtime_failure_handoff", "at": "2026-08-28T07:42:38.939395+00:00", "head": "10325022b2774d3284544f12680ba9ba41f77149", "source_unchanged_since_post_build": true, "source_files": 45, "source_hashes": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-initial/verify-unit/failed-source-hashes.json", "failed_tree_archive": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-initial/verify-unit/failed-tree.tar", "failed_tree_sha256": "dbfd53499b0dd0dee2cea43b1afc15f28e2c3117a95095d60da3d44745bebd9e", "tracked_diff": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-initial/verify-unit/failed-tracked.diff", "unit": {"tests": 47, "failures": 1, "errors": 0, "skipped": 0}, "failure": "OutboundQueueTest mixed case line45: initial FULL1 in-memory ObjectNode equality with parsed JSON; identical textual JSON, numeric node representation suspected; fixed mixed sequence not reached", "unexecuted": ["integration", "canonical", "reference"], "consumed": {"compiler_tasks": 3, "compiler_bearing_commands": 2, "unit_including_baseline": 2, "integration": 0, "post_live": 0, "reference_replay": 0, "baseline_ticks": 100, "post_ticks": 0, "fault": 0, "load": 0, "profiler": 0, "repairs": 0}, "production_or_test_edits_after_failure": 0, "reruns_after_failure": 0, "commit_after_failure": false}
diff --git a/evidence/G12-repair1-command-ledger.jsonl b/evidence/G12-repair1-command-ledger.jsonl
new file mode 100644
index 0000000..1ffebbf
--- /dev/null
+++ b/evidence/G12-repair1-command-ledger.jsonl
@@ -0,0 +1,13 @@
+{"kind": "activation", "thread": "G12", "attempt": "repair1", "profile": "realtime-core", "spec_revision": "c1d62196ab76b55652f5d75a67514f8c6d8210ce", "start": "10325022b2774d3284544f12680ba9ba41f77149", "recorded_utc": "2026-08-28T07:51:13.590207+00:00", "scenario": "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G12.json", "scenario_sha256": "b5350ef5bfb9fa93cbde1fe0fd30079e6115fe26988514be59d63d9dee6bc6f3", "initial_failure_preserved": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-initial/verify-unit", "initial_consumed": {"compiler_tasks": 3, "compiler_bearing_commands": 2, "unit_including_baseline": 2, "integration": 0, "post_live": 0, "reference_replay": 0, "baseline_ticks": 100, "post_ticks": 0, "fault": 0, "load": 0, "profiler": 0, "repairs": 0}, "repair_ceilings": {"compiler_tasks": 8, "baseline_runs": 0, "full_unit_runs": 1, "integration_runs": 1, "live_runs": 1, "live_ticks": 100, "reference_runs": 1, "reference_ticks": 100, "fault_runs": 0, "load_runs": 0, "profiler_runs": 0, "runtime_retries": 0}, "initial_source_file_count": 45, "source_changes_from_failed_tree": ["src/test/java/arena/OutboundQueueTest.java"], "production_source_changes_from_failed_tree": 0, "pure_queue_cases": 2, "g12_live_campaigns_in_unit": 0, "runner": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair1/run_commands.py", "runner_sha256": "d874f18ed1945d380da3b8426c83f380e8cdf57471362c9de792d28f708b5c88", "pre_repair_source_hashes": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair1/pre-repair-source-hashes.json", "pre_verification_source_hashes": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair1/pre-verification-source-hashes.json", "repair_diff": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair1/repair-only.diff", "stop_on_first_runtime_failure": true}
+{"kind": "prelaunch_static_diagnostic", "at": "2026-08-28T07:51:13.590697+00:00", "command_kind": "Python evidence setup only; no compiler or application launched", "initial_exit_code": 1, "initial_diagnostic": "AssertionError from comparing inherited JAVA_HOME path string to pinned path string", "inherited_java_home": "/Users/woopinbell/.sdkman/candidates/java/current", "resolved_java_home": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem", "correction": "Verify resolved path and release file; actual child JAVA_HOME is the unchanged pinned21.0.7 runtime", "compile_or_runtime_budget_consumed": 0}
+{"kind": "resolved_before_execution", "attempt": "repair1", "pass": "build", "category": "clean-build", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "build"], "environment": {"HOME": "/Users/woopinbell", "PATH": "/opt/homebrew/opt/python@3.12/libexec/bin:/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/System/Cryptexes/App/usr/bin:/usr/bin:/bin:/usr/sbin:/sbin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/local/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/appleinternal/bin:/pkg/env/global/bin:/Library/Apple/usr/bin:/Users/woopinbell/.codex/tmp/arg0/codex-arg0Vyt1ls:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/override:/Users/woopinbell/.sdkman/candidates/java/current/bin:/Users/woopinbell/.sdkman/candidates/gradle/current/bin:/Users/woopinbell/.local/bin:/Users/woopinbell/development/flutter/bin:/Users/woopinbell/.pyenv/bin:/opt/homebrew/opt/python@3.12/libexec/bin:/Users/woopinbell/bin:/Users/woopinbell/.foundry/bin:/Users/woopinbell/.maestro/bin:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/fallback:/Applications/ChatGPT.app/Contents/Resources", "TMPDIR": "/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/", "LANG": "C.UTF-8", "LC_ALL": "C.UTF-8", "LC_CTYPE": "C.UTF-8", "USER": "woopinbell", "LOGNAME": "woopinbell", "SHELL": "/bin/zsh", "TERM": "dumb", "JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "environment_policy": "exact child environment; inherited nonsecret runtime keys plus resolved pinned JAVA_HOME; no external credentials or runtime option injection", "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair1/verify-build", "reserved_g12_ticks": 0}
+{"kind": "resolved_before_execution", "attempt": "repair1", "pass": "unit", "category": "full-unit-exact-two-frozen-pure-cases", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test"], "environment": {"HOME": "/Users/woopinbell", "PATH": "/opt/homebrew/opt/python@3.12/libexec/bin:/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/System/Cryptexes/App/usr/bin:/usr/bin:/bin:/usr/sbin:/sbin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/local/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/appleinternal/bin:/pkg/env/global/bin:/Library/Apple/usr/bin:/Users/woopinbell/.codex/tmp/arg0/codex-arg0Vyt1ls:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/override:/Users/woopinbell/.sdkman/candidates/java/current/bin:/Users/woopinbell/.sdkman/candidates/gradle/current/bin:/Users/woopinbell/.local/bin:/Users/woopinbell/development/flutter/bin:/Users/woopinbell/.pyenv/bin:/opt/homebrew/opt/python@3.12/libexec/bin:/Users/woopinbell/bin:/Users/woopinbell/.foundry/bin:/Users/woopinbell/.maestro/bin:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/fallback:/Applications/ChatGPT.app/Contents/Resources", "TMPDIR": "/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/", "LANG": "C.UTF-8", "LC_ALL": "C.UTF-8", "LC_CTYPE": "C.UTF-8", "USER": "woopinbell", "LOGNAME": "woopinbell", "SHELL": "/bin/zsh", "TERM": "dumb", "JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "environment_policy": "exact child environment; inherited nonsecret runtime keys plus resolved pinned JAVA_HOME; no external credentials or runtime option injection", "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair1/verify-unit", "reserved_g12_ticks": 0}
+{"kind": "resolved_before_execution", "attempt": "repair1", "pass": "integration", "category": "full-integration", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "integration-test"], "environment": {"HOME": "/Users/woopinbell", "PATH": "/opt/homebrew/opt/python@3.12/libexec/bin:/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/System/Cryptexes/App/usr/bin:/usr/bin:/bin:/usr/sbin:/sbin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/local/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/appleinternal/bin:/pkg/env/global/bin:/Library/Apple/usr/bin:/Users/woopinbell/.codex/tmp/arg0/codex-arg0Vyt1ls:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/override:/Users/woopinbell/.sdkman/candidates/java/current/bin:/Users/woopinbell/.sdkman/candidates/gradle/current/bin:/Users/woopinbell/.local/bin:/Users/woopinbell/development/flutter/bin:/Users/woopinbell/.pyenv/bin:/opt/homebrew/opt/python@3.12/libexec/bin:/Users/woopinbell/bin:/Users/woopinbell/.foundry/bin:/Users/woopinbell/.maestro/bin:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/fallback:/Applications/ChatGPT.app/Contents/Resources", "TMPDIR": "/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/", "LANG": "C.UTF-8", "LC_ALL": "C.UTF-8", "LC_CTYPE": "C.UTF-8", "USER": "woopinbell", "LOGNAME": "woopinbell", "SHELL": "/bin/zsh", "TERM": "dumb", "JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "environment_policy": "exact child environment; inherited nonsecret runtime keys plus resolved pinned JAVA_HOME; no external credentials or runtime option injection", "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair1/verify-integration", "reserved_g12_ticks": 0}
+{"kind": "resolved_before_execution", "attempt": "repair1", "pass": "canonical", "category": "fixed-live-slow-consumer", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "scenario-run", "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G12.json", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair1/canonical/result.json"], "environment": {"HOME": "/Users/woopinbell", "PATH": "/opt/homebrew/opt/python@3.12/libexec/bin:/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/System/Cryptexes/App/usr/bin:/usr/bin:/bin:/usr/sbin:/sbin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/local/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/appleinternal/bin:/pkg/env/global/bin:/Library/Apple/usr/bin:/Users/woopinbell/.codex/tmp/arg0/codex-arg0Vyt1ls:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/override:/Users/woopinbell/.sdkman/candidates/java/current/bin:/Users/woopinbell/.sdkman/candidates/gradle/current/bin:/Users/woopinbell/.local/bin:/Users/woopinbell/development/flutter/bin:/Users/woopinbell/.pyenv/bin:/opt/homebrew/opt/python@3.12/libexec/bin:/Users/woopinbell/bin:/Users/woopinbell/.foundry/bin:/Users/woopinbell/.maestro/bin:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/fallback:/Applications/ChatGPT.app/Contents/Resources", "TMPDIR": "/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/", "LANG": "C.UTF-8", "LC_ALL": "C.UTF-8", "LC_CTYPE": "C.UTF-8", "USER": "woopinbell", "LOGNAME": "woopinbell", "SHELL": "/bin/zsh", "TERM": "dumb", "JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "environment_policy": "exact child environment; inherited nonsecret runtime keys plus resolved pinned JAVA_HOME; no external credentials or runtime option injection", "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair1/canonical", "reserved_g12_ticks": 100}
+{"kind": "resolved_before_execution", "attempt": "repair1", "pass": "reference", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair1/canonical/result.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair1/reference/result.json"], "environment": {"HOME": "/Users/woopinbell", "PATH": "/opt/homebrew/opt/python@3.12/libexec/bin:/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/System/Cryptexes/App/usr/bin:/usr/bin:/bin:/usr/sbin:/sbin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/local/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/appleinternal/bin:/pkg/env/global/bin:/Library/Apple/usr/bin:/Users/woopinbell/.codex/tmp/arg0/codex-arg0Vyt1ls:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/override:/Users/woopinbell/.sdkman/candidates/java/current/bin:/Users/woopinbell/.sdkman/candidates/gradle/current/bin:/Users/woopinbell/.local/bin:/Users/woopinbell/development/flutter/bin:/Users/woopinbell/.pyenv/bin:/opt/homebrew/opt/python@3.12/libexec/bin:/Users/woopinbell/bin:/Users/woopinbell/.foundry/bin:/Users/woopinbell/.maestro/bin:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/fallback:/Applications/ChatGPT.app/Contents/Resources", "TMPDIR": "/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/", "LANG": "C.UTF-8", "LC_ALL": "C.UTF-8", "LC_CTYPE": "C.UTF-8", "USER": "woopinbell", "LOGNAME": "woopinbell", "SHELL": "/bin/zsh", "TERM": "dumb", "JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "environment_policy": "exact child environment; inherited nonsecret runtime keys plus resolved pinned JAVA_HOME; no external credentials or runtime option injection", "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair1/reference", "reserved_g12_ticks": 100}
+{"kind": "launch", "attempt": "repair1", "pass": "build", "at": "2026-08-28T07:51:42.986072+00:00", "argv": ["./track", "build"], "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "environment": {"HOME": "/Users/woopinbell", "PATH": "/opt/homebrew/opt/python@3.12/libexec/bin:/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/System/Cryptexes/App/usr/bin:/usr/bin:/bin:/usr/sbin:/sbin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/local/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/appleinternal/bin:/pkg/env/global/bin:/Library/Apple/usr/bin:/Users/woopinbell/.codex/tmp/arg0/codex-arg0Vyt1ls:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/override:/Users/woopinbell/.sdkman/candidates/java/current/bin:/Users/woopinbell/.sdkman/candidates/gradle/current/bin:/Users/woopinbell/.local/bin:/Users/woopinbell/development/flutter/bin:/Users/woopinbell/.pyenv/bin:/opt/homebrew/opt/python@3.12/libexec/bin:/Users/woopinbell/bin:/Users/woopinbell/.foundry/bin:/Users/woopinbell/.maestro/bin:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/fallback:/Applications/ChatGPT.app/Contents/Resources", "TMPDIR": "/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/", "LANG": "C.UTF-8", "LC_ALL": "C.UTF-8", "LC_CTYPE": "C.UTF-8", "USER": "woopinbell", "LOGNAME": "woopinbell", "SHELL": "/bin/zsh", "TERM": "dumb", "JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "runner_pid": 36619, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair1/verify-build"}
+{"pass": "build", "category": "clean-build", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "build"], "environment": {"HOME": "/Users/woopinbell", "PATH": "/opt/homebrew/opt/python@3.12/libexec/bin:/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/System/Cryptexes/App/usr/bin:/usr/bin:/bin:/usr/sbin:/sbin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/local/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/appleinternal/bin:/pkg/env/global/bin:/Library/Apple/usr/bin:/Users/woopinbell/.codex/tmp/arg0/codex-arg0Vyt1ls:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/override:/Users/woopinbell/.sdkman/candidates/java/current/bin:/Users/woopinbell/.sdkman/candidates/gradle/current/bin:/Users/woopinbell/.local/bin:/Users/woopinbell/development/flutter/bin:/Users/woopinbell/.pyenv/bin:/opt/homebrew/opt/python@3.12/libexec/bin:/Users/woopinbell/bin:/Users/woopinbell/.foundry/bin:/Users/woopinbell/.maestro/bin:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/fallback:/Applications/ChatGPT.app/Contents/Resources", "TMPDIR": "/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/", "LANG": "C.UTF-8", "LC_ALL": "C.UTF-8", "LC_CTYPE": "C.UTF-8", "USER": "woopinbell", "LOGNAME": "woopinbell", "SHELL": "/bin/zsh", "TERM": "dumb", "JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "attempt": "repair1", "started_at": "2026-08-28T07:51:42.986072+00:00", "finished_at": "2026-08-28T07:51:48.231025+00:00", "duration_seconds": 5.245027, "command_process_id": 36629, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair1/verify-build/stdout.log", "output_sha256": "684b1b9e91a57ec8bb0d5a91e6bc27b354cd5591e553e45432343bdb2ca5858a", "process_terminated": true, "compiler_tasks_executed": ["> Task :compileJava", "> Task :compileTestJava"], "compiler_tasks_consumed": 2, "source_hashes": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair1/verify-build/source-hashes.json", "source_hashes_sha256": "526d8aaf9a4c15eb691405187e3b4e65be09e48acf8de6c9499385db453f2422", "sources_unchanged": true, "raw_file_sha256": {"source-hashes.json": "526d8aaf9a4c15eb691405187e3b4e65be09e48acf8de6c9499385db453f2422", "stdout.log": "684b1b9e91a57ec8bb0d5a91e6bc27b354cd5591e553e45432343bdb2ca5858a"}}
+{"kind": "launch", "attempt": "repair1", "pass": "unit", "at": "2026-08-28T07:51:48.239738+00:00", "argv": ["./track", "unit-test"], "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "environment": {"HOME": "/Users/woopinbell", "PATH": "/opt/homebrew/opt/python@3.12/libexec/bin:/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/System/Cryptexes/App/usr/bin:/usr/bin:/bin:/usr/sbin:/sbin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/local/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/appleinternal/bin:/pkg/env/global/bin:/Library/Apple/usr/bin:/Users/woopinbell/.codex/tmp/arg0/codex-arg0Vyt1ls:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/override:/Users/woopinbell/.sdkman/candidates/java/current/bin:/Users/woopinbell/.sdkman/candidates/gradle/current/bin:/Users/woopinbell/.local/bin:/Users/woopinbell/development/flutter/bin:/Users/woopinbell/.pyenv/bin:/opt/homebrew/opt/python@3.12/libexec/bin:/Users/woopinbell/bin:/Users/woopinbell/.foundry/bin:/Users/woopinbell/.maestro/bin:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/fallback:/Applications/ChatGPT.app/Contents/Resources", "TMPDIR": "/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/", "LANG": "C.UTF-8", "LC_ALL": "C.UTF-8", "LC_CTYPE": "C.UTF-8", "USER": "woopinbell", "LOGNAME": "woopinbell", "SHELL": "/bin/zsh", "TERM": "dumb", "JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "runner_pid": 36619, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair1/verify-unit"}
+{"pass": "unit", "category": "full-unit-exact-two-frozen-pure-cases", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test"], "environment": {"HOME": "/Users/woopinbell", "PATH": "/opt/homebrew/opt/python@3.12/libexec/bin:/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/System/Cryptexes/App/usr/bin:/usr/bin:/bin:/usr/sbin:/sbin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/local/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/appleinternal/bin:/pkg/env/global/bin:/Library/Apple/usr/bin:/Users/woopinbell/.codex/tmp/arg0/codex-arg0Vyt1ls:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/override:/Users/woopinbell/.sdkman/candidates/java/current/bin:/Users/woopinbell/.sdkman/candidates/gradle/current/bin:/Users/woopinbell/.local/bin:/Users/woopinbell/development/flutter/bin:/Users/woopinbell/.pyenv/bin:/opt/homebrew/opt/python@3.12/libexec/bin:/Users/woopinbell/bin:/Users/woopinbell/.foundry/bin:/Users/woopinbell/.maestro/bin:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/fallback:/Applications/ChatGPT.app/Contents/Resources", "TMPDIR": "/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/", "LANG": "C.UTF-8", "LC_ALL": "C.UTF-8", "LC_CTYPE": "C.UTF-8", "USER": "woopinbell", "LOGNAME": "woopinbell", "SHELL": "/bin/zsh", "TERM": "dumb", "JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "attempt": "repair1", "started_at": "2026-08-28T07:51:48.239738+00:00", "finished_at": "2026-08-28T07:51:57.355558+00:00", "duration_seconds": 9.115919, "command_process_id": 36690, "exit_code": 1, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair1/verify-unit/stdout.log", "output_sha256": "0c598bc362da4969ec9ae875a966b99c6bf1b1b794887cb810dd6a383e34919b", "process_terminated": true, "compiler_tasks_executed": [], "compiler_tasks_consumed": 2, "source_hashes": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair1/verify-unit/source-hashes.json", "source_hashes_sha256": "526d8aaf9a4c15eb691405187e3b4e65be09e48acf8de6c9499385db453f2422", "sources_unchanged": true, "xml": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair1/verify-unit/xml", "test_counts": {"tests": 47, "failures": 1, "errors": 0, "skipped": 0}, "raw_file_sha256": {"source-hashes.json": "526d8aaf9a4c15eb691405187e3b4e65be09e48acf8de6c9499385db453f2422", "stdout.log": "0c598bc362da4969ec9ae875a966b99c6bf1b1b794887cb810dd6a383e34919b", "xml/TEST-arena.CompleteFrameTest.xml": "a46c5beb0ac49b68876f1eeabf3b43dd85fc93956f53167f64ca176bce152e97", "xml/TEST-arena.OutboundQueueTest.xml": "066ba425febc677201b14fb1d7d6229d34a5f896c6c589876f1fd856f9b0a22f", "xml/TEST-arena.ReplayFormatTest.xml": "741f956323430d664f73ba712a5b7dc599ddb51ffcfa93c733d32f4daac1825d", "xml/TEST-arena.RoomTest.xml": "ce315b15ae5f10421cb51012bdecf95647c7caca508d66bba205294a10e9ee03", "xml/TEST-arena.SnapshotStreamTest.xml": "d5321e22b570793f96674094bd590f45384cb0db5758ce44af640c8e0c0520bd", "xml/TEST-arena.UdpBoundaryTest.xml": "69ec4e52c7ef6704171fd46a8d7a112b4b56a58c6b565dd603daeb14f399fc01", "xml/binary/output.bin": "4598e374ab239163622a79a718c0e7a863970d927304cdcb41485712d541aa30", "xml/binary/output.bin.idx": "15827d356d0ff3f0738bee75b7f27d66c3ec85265c24a372770a8d6fc9c9d066", "xml/binary/results.bin": "79d47f281b17b83eceea3257bbf6ad07ce9d809f7cdd0a5f8aaacdc29d6ecc81"}}
+{"kind": "failure_handoff", "attempt": "repair1", "pass": "unit", "at": "2026-08-28T07:51:57.402266+00:00", "exit_code": 1, "source_hashes": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair1/verify-unit/failed-source-hashes.json", "tracked_diff": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair1/verify-unit/failed-tracked.diff", "archive": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair1/verify-unit/failed-tree.tar", "archive_sha256": "e0d9cb8d115ea5d81336713847344a3346d6a5a41cec3244b62551be00a851e5", "runtime_retry": false, "remaining_passes_not_run": true}
+{"kind": "runtime_failure_handoff", "at": "2026-08-28T07:54:08.755305+00:00", "thread": "G12", "attempt": "repair1", "head": "10325022b2774d3284544f12680ba9ba41f77149", "result": "FAILED_UNIT", "failure": "RoomTest.frozenG03IdentityLifecycle line79: FINISHED/LEAVE_ROOM client socket ceiling exceeded", "source_unchanged_since_build_and_failure": true, "production_changes_in_repair": 0, "observed_regression": {"snapshot_generations": 1018, "suppressed_inflight_snapshots": 1, "coalesced_snapshots": 0, "clock_last_monotonic_ns": 50800000000, "cleanup_passed": true}, "pure_queue_tests": {"tests": 2, "failures": 0, "errors": 0, "skipped": 0}, "observation": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair1/verify-unit/failure-observation.json", "observation_sha256": "fcabc696d21d4f93bb40af285556e88e69b0de38e4a97ecd747bd3401b1935a6", "consumed_this_repair": {"compiler_tasks": 2, "clean_builds": 1, "full_unit_runs": 1, "unit_tests": 47, "unit_failures": 1, "integration_runs": 0, "baseline_runs": 0, "post_live_runs": 0, "reference_runs": 0, "post_live_ticks": 0, "reference_ticks": 0, "fault_runs": 0, "load_runs": 0, "profiler_runs": 0, "runtime_retries": 0}, "consumed_initial_plus_repair1": {"compiler_tasks": 5, "unit_runs_including_initial_baseline": 3, "initial_baseline_ticks": 100, "integration_runs": 0, "post_live_runs": 0, "reference_runs": 0, "repairs": 1}, "unexecuted": ["integration", "canonical", "reference"], "post_failure_source_edits": 0, "commit_created": false, "next_action": "root review and, only if authorized, fresh bounded repair2"}
diff --git a/evidence/G12-repair2-command-ledger.jsonl b/evidence/G12-repair2-command-ledger.jsonl
new file mode 100644
index 0000000..dc33690
--- /dev/null
+++ b/evidence/G12-repair2-command-ledger.jsonl
@@ -0,0 +1,22 @@
+{"kind": "activation", "at": "2026-08-28T08:01:56.301790+00:00", "thread": "G12", "attempt": "repair2", "repair_ceiling": 2, "profile": "realtime-core", "spec_revision": "c1d62196ab76b55652f5d75a67514f8c6d8210ce", "start": "10325022b2774d3284544f12680ba9ba41f77149", "branch": "track/industry-java", "all45_equal_root_repair1": true, "production17_equal_initial_failure": true, "pre_repair_source_hashes": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/pre-repair-source-hashes.json", "pre_repair_source_hashes_sha256": "526d8aaf9a4c15eb691405187e3b4e65be09e48acf8de6c9499385db453f2422", "fixture": "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G12.json", "fixture_sha256": "b5350ef5bfb9fa93cbde1fe0fd30079e6115fe26988514be59d63d9dee6bc6f3", "previous_consumed": {"initial": {"compiler_tasks": 3, "unit_runs": 2, "baseline_ticks": 100, "integration_runs": 0, "post_live_runs": 0, "reference_runs": 0}, "repair1": {"compiler_tasks": 2, "clean_builds": 1, "full_unit_runs": 1, "tests": 47, "failures": 1, "integration_runs": 0, "post_live_runs": 0, "reference_runs": 0}}, "repair2_budget": {"clean_builds": 1, "compiler_tasks": 8, "full_unit_runs": 1, "integration_runs": 1, "live_runs": 1, "live_ticks": 100, "reference_runs": 1, "reference_ticks": 100, "baseline_runs": 0, "runtime_retries": 0, "fault_runs": 0, "load_runs": 0, "profiler_runs": 0}, "runtime_requires_root_source_review": true}
+{"kind": "resolved_before_execution", "attempt": "repair2", "pass": "build", "category": "clean-build", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "build"], "environment": {"HOME": "/Users/woopinbell", "PATH": "/opt/homebrew/opt/python@3.12/libexec/bin:/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/System/Cryptexes/App/usr/bin:/usr/bin:/bin:/usr/sbin:/sbin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/local/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/appleinternal/bin:/pkg/env/global/bin:/Library/Apple/usr/bin:/Users/woopinbell/.codex/tmp/arg0/codex-arg0Vyt1ls:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/override:/Users/woopinbell/.sdkman/candidates/java/current/bin:/Users/woopinbell/.sdkman/candidates/gradle/current/bin:/Users/woopinbell/.local/bin:/Users/woopinbell/development/flutter/bin:/Users/woopinbell/.pyenv/bin:/opt/homebrew/opt/python@3.12/libexec/bin:/Users/woopinbell/bin:/Users/woopinbell/.foundry/bin:/Users/woopinbell/.maestro/bin:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/fallback:/Applications/ChatGPT.app/Contents/Resources", "TMPDIR": "/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/", "LANG": "C.UTF-8", "LC_ALL": "C.UTF-8", "LC_CTYPE": "C.UTF-8", "USER": "woopinbell", "LOGNAME": "woopinbell", "SHELL": "/bin/zsh", "TERM": "dumb", "JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "environment_policy": "exact child environment; inherited nonsecret runtime keys plus resolved pinned JAVA_HOME; no external credentials or runtime option injection", "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/verify-build", "reserved_g12_ticks": 0}
+{"kind": "resolved_before_execution", "attempt": "repair2", "pass": "unit", "category": "full-unit-exact-two-frozen-pure-cases", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test"], "environment": {"HOME": "/Users/woopinbell", "PATH": "/opt/homebrew/opt/python@3.12/libexec/bin:/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/System/Cryptexes/App/usr/bin:/usr/bin:/bin:/usr/sbin:/sbin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/local/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/appleinternal/bin:/pkg/env/global/bin:/Library/Apple/usr/bin:/Users/woopinbell/.codex/tmp/arg0/codex-arg0Vyt1ls:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/override:/Users/woopinbell/.sdkman/candidates/java/current/bin:/Users/woopinbell/.sdkman/candidates/gradle/current/bin:/Users/woopinbell/.local/bin:/Users/woopinbell/development/flutter/bin:/Users/woopinbell/.pyenv/bin:/opt/homebrew/opt/python@3.12/libexec/bin:/Users/woopinbell/bin:/Users/woopinbell/.foundry/bin:/Users/woopinbell/.maestro/bin:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/fallback:/Applications/ChatGPT.app/Contents/Resources", "TMPDIR": "/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/", "LANG": "C.UTF-8", "LC_ALL": "C.UTF-8", "LC_CTYPE": "C.UTF-8", "USER": "woopinbell", "LOGNAME": "woopinbell", "SHELL": "/bin/zsh", "TERM": "dumb", "JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "environment_policy": "exact child environment; inherited nonsecret runtime keys plus resolved pinned JAVA_HOME; no external credentials or runtime option injection", "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/verify-unit", "reserved_g12_ticks": 0}
+{"kind": "resolved_before_execution", "attempt": "repair2", "pass": "integration", "category": "full-integration", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "integration-test"], "environment": {"HOME": "/Users/woopinbell", "PATH": "/opt/homebrew/opt/python@3.12/libexec/bin:/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/System/Cryptexes/App/usr/bin:/usr/bin:/bin:/usr/sbin:/sbin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/local/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/appleinternal/bin:/pkg/env/global/bin:/Library/Apple/usr/bin:/Users/woopinbell/.codex/tmp/arg0/codex-arg0Vyt1ls:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/override:/Users/woopinbell/.sdkman/candidates/java/current/bin:/Users/woopinbell/.sdkman/candidates/gradle/current/bin:/Users/woopinbell/.local/bin:/Users/woopinbell/development/flutter/bin:/Users/woopinbell/.pyenv/bin:/opt/homebrew/opt/python@3.12/libexec/bin:/Users/woopinbell/bin:/Users/woopinbell/.foundry/bin:/Users/woopinbell/.maestro/bin:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/fallback:/Applications/ChatGPT.app/Contents/Resources", "TMPDIR": "/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/", "LANG": "C.UTF-8", "LC_ALL": "C.UTF-8", "LC_CTYPE": "C.UTF-8", "USER": "woopinbell", "LOGNAME": "woopinbell", "SHELL": "/bin/zsh", "TERM": "dumb", "JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "environment_policy": "exact child environment; inherited nonsecret runtime keys plus resolved pinned JAVA_HOME; no external credentials or runtime option injection", "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/verify-integration", "reserved_g12_ticks": 0}
+{"kind": "resolved_before_execution", "attempt": "repair2", "pass": "canonical", "category": "fixed-live-slow-consumer", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "scenario-run", "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G12.json", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/canonical/result.json"], "environment": {"HOME": "/Users/woopinbell", "PATH": "/opt/homebrew/opt/python@3.12/libexec/bin:/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/System/Cryptexes/App/usr/bin:/usr/bin:/bin:/usr/sbin:/sbin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/local/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/appleinternal/bin:/pkg/env/global/bin:/Library/Apple/usr/bin:/Users/woopinbell/.codex/tmp/arg0/codex-arg0Vyt1ls:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/override:/Users/woopinbell/.sdkman/candidates/java/current/bin:/Users/woopinbell/.sdkman/candidates/gradle/current/bin:/Users/woopinbell/.local/bin:/Users/woopinbell/development/flutter/bin:/Users/woopinbell/.pyenv/bin:/opt/homebrew/opt/python@3.12/libexec/bin:/Users/woopinbell/bin:/Users/woopinbell/.foundry/bin:/Users/woopinbell/.maestro/bin:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/fallback:/Applications/ChatGPT.app/Contents/Resources", "TMPDIR": "/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/", "LANG": "C.UTF-8", "LC_ALL": "C.UTF-8", "LC_CTYPE": "C.UTF-8", "USER": "woopinbell", "LOGNAME": "woopinbell", "SHELL": "/bin/zsh", "TERM": "dumb", "JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "environment_policy": "exact child environment; inherited nonsecret runtime keys plus resolved pinned JAVA_HOME; no external credentials or runtime option injection", "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/canonical", "reserved_g12_ticks": 100}
+{"kind": "resolved_before_execution", "attempt": "repair2", "pass": "reference", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/canonical/result.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/reference/result.json"], "environment": {"HOME": "/Users/woopinbell", "PATH": "/opt/homebrew/opt/python@3.12/libexec/bin:/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/System/Cryptexes/App/usr/bin:/usr/bin:/bin:/usr/sbin:/sbin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/local/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/appleinternal/bin:/pkg/env/global/bin:/Library/Apple/usr/bin:/Users/woopinbell/.codex/tmp/arg0/codex-arg0Vyt1ls:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/override:/Users/woopinbell/.sdkman/candidates/java/current/bin:/Users/woopinbell/.sdkman/candidates/gradle/current/bin:/Users/woopinbell/.local/bin:/Users/woopinbell/development/flutter/bin:/Users/woopinbell/.pyenv/bin:/opt/homebrew/opt/python@3.12/libexec/bin:/Users/woopinbell/bin:/Users/woopinbell/.foundry/bin:/Users/woopinbell/.maestro/bin:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/fallback:/Applications/ChatGPT.app/Contents/Resources", "TMPDIR": "/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/", "LANG": "C.UTF-8", "LC_ALL": "C.UTF-8", "LC_CTYPE": "C.UTF-8", "USER": "woopinbell", "LOGNAME": "woopinbell", "SHELL": "/bin/zsh", "TERM": "dumb", "JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "environment_policy": "exact child environment; inherited nonsecret runtime keys plus resolved pinned JAVA_HOME; no external credentials or runtime option injection", "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/reference", "reserved_g12_ticks": 100}
+{"kind": "proposed_source_fix", "attempt": "repair2", "at": "2026-08-28T08:03:01.827233+00:00", "path": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/proposed-repair.diff", "sha256": "2862bd20f7fd34d5eb8e6cccf0b3842ef09863a53fda1769708ff3b354af9353", "production_modified": false, "runtime_commands_launched": 0}
+{"kind": "ownership_static_inspection", "attempt": "repair2", "at": "2026-08-28T08:04:31.855498+00:00", "argv": [["/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem/bin/javap", "-c", "-p", "-classpath", "/Users/woopinbell/.gradle/caches/modules-2/files-2.1/io.netty/netty-transport/4.1.114.Final/e0225a575f487904be8517092cbd74e01913533c/netty-transport-4.1.114.Final.jar", "io.netty.channel.ChannelOutboundBuffer"], ["/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem/bin/javap", "-c", "-p", "-classpath", "/Users/woopinbell/.gradle/caches/modules-2/files-2.1/io.netty/netty-transport/4.1.114.Final/e0225a575f487904be8517092cbd74e01913533c/netty-transport-4.1.114.Final.jar", "io.netty.channel.nio.AbstractNioMessageChannel"], ["/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem/bin/javap", "-c", "-p", "-classpath", "/Users/woopinbell/.gradle/caches/modules-2/files-2.1/io.netty/netty-transport/4.1.114.Final/e0225a575f487904be8517092cbd74e01913533c/netty-transport-4.1.114.Final.jar", "io.netty.channel.socket.nio.NioDatagramChannel"], ["/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem/bin/javap", "-c", "-p", "-classpath", "/Users/woopinbell/.gradle/caches/modules-2/files-2.1/io.netty/netty-transport/4.1.114.Final/e0225a575f487904be8517092cbd74e01913533c/netty-transport-4.1.114.Final.jar", "io.netty.channel.nio.AbstractNioChannel"], ["/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem/bin/javap", "-c", "-p", "-classpath", "/Users/woopinbell/.gradle/caches/modules-2/files-2.1/io.netty/netty-transport/4.1.114.Final/e0225a575f487904be8517092cbd74e01913533c/netty-transport-4.1.114.Final.jar", "io.netty.channel.DefaultChannelConfig"]], "compiler_tasks": 0, "runtime_campaigns": 0, "pinned_jar": "/Users/woopinbell/.gradle/caches/modules-2/files-2.1/io.netty/netty-transport/4.1.114.Final/e0225a575f487904be8517092cbd74e01913533c/netty-transport-4.1.114.Final.jar", "pinned_jar_sha256": "2a8609fe6a8b4c9d5965c6b901777b4bd0b26600647ee2aa7d4d93f4d5c780de", "evidence": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/netty-ownership-bytecode.txt", "evidence_sha256": "6b4501f2336f73bd62056876320b65adcf7babf470b1c7ad6485293ebb760d9a"}
+{"kind": "source_review_approved", "attempt": "repair2", "at": "2026-08-28T08:07:52.595136+00:00", "authority": "root", "approved_diff": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/proposed-repair.diff", "approved_diff_sha256": "2862bd20f7fd34d5eb8e6cccf0b3842ef09863a53fda1769708ff3b354af9353", "applied_exactly": true, "source_files": 45, "changed_from_root_repair1": ["src/main/java/arena/OutboundQueue.java"], "repair1_test_unchanged": true, "source_manifest": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/pre-verification-source-hashes.json", "source_manifest_sha256": "1f19d3883308c2b55c3de1392622686ff552ce3558a4cfd486e26b4f4db8c66f", "runner": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/run_commands.py", "runner_sha256": "2814c9a2ac8632a8a3ff11922ceaa821af6f05b2c50de25d72fc3a6472ba7e7c", "runtime_sequence": ["build", "unit", "integration", "canonical", "reference"], "no_runtime_retries": true, "compiler_tasks_before_verification": 0}
+{"kind": "launch", "attempt": "repair2", "pass": "build", "at": "2026-08-28T08:08:01.152180+00:00", "argv": ["./track", "build"], "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "environment": {"HOME": "/Users/woopinbell", "PATH": "/opt/homebrew/opt/python@3.12/libexec/bin:/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/System/Cryptexes/App/usr/bin:/usr/bin:/bin:/usr/sbin:/sbin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/local/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/appleinternal/bin:/pkg/env/global/bin:/Library/Apple/usr/bin:/Users/woopinbell/.codex/tmp/arg0/codex-arg0Vyt1ls:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/override:/Users/woopinbell/.sdkman/candidates/java/current/bin:/Users/woopinbell/.sdkman/candidates/gradle/current/bin:/Users/woopinbell/.local/bin:/Users/woopinbell/development/flutter/bin:/Users/woopinbell/.pyenv/bin:/opt/homebrew/opt/python@3.12/libexec/bin:/Users/woopinbell/bin:/Users/woopinbell/.foundry/bin:/Users/woopinbell/.maestro/bin:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/fallback:/Applications/ChatGPT.app/Contents/Resources", "TMPDIR": "/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/", "LANG": "C.UTF-8", "LC_ALL": "C.UTF-8", "LC_CTYPE": "C.UTF-8", "USER": "woopinbell", "LOGNAME": "woopinbell", "SHELL": "/bin/zsh", "TERM": "dumb", "JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "runner_pid": 49163, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/verify-build"}
+{"pass": "build", "category": "clean-build", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "build"], "environment": {"HOME": "/Users/woopinbell", "PATH": "/opt/homebrew/opt/python@3.12/libexec/bin:/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/System/Cryptexes/App/usr/bin:/usr/bin:/bin:/usr/sbin:/sbin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/local/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/appleinternal/bin:/pkg/env/global/bin:/Library/Apple/usr/bin:/Users/woopinbell/.codex/tmp/arg0/codex-arg0Vyt1ls:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/override:/Users/woopinbell/.sdkman/candidates/java/current/bin:/Users/woopinbell/.sdkman/candidates/gradle/current/bin:/Users/woopinbell/.local/bin:/Users/woopinbell/development/flutter/bin:/Users/woopinbell/.pyenv/bin:/opt/homebrew/opt/python@3.12/libexec/bin:/Users/woopinbell/bin:/Users/woopinbell/.foundry/bin:/Users/woopinbell/.maestro/bin:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/fallback:/Applications/ChatGPT.app/Contents/Resources", "TMPDIR": "/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/", "LANG": "C.UTF-8", "LC_ALL": "C.UTF-8", "LC_CTYPE": "C.UTF-8", "USER": "woopinbell", "LOGNAME": "woopinbell", "SHELL": "/bin/zsh", "TERM": "dumb", "JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "attempt": "repair2", "started_at": "2026-08-28T08:08:01.152180+00:00", "finished_at": "2026-08-28T08:08:06.392449+00:00", "duration_seconds": 5.24046, "command_process_id": 49172, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/verify-build/stdout.log", "output_sha256": "684b1b9e91a57ec8bb0d5a91e6bc27b354cd5591e553e45432343bdb2ca5858a", "process_terminated": true, "compiler_tasks_executed": ["> Task :compileJava", "> Task :compileTestJava"], "compiler_tasks_consumed": 2, "source_hashes": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/verify-build/source-hashes.json", "source_hashes_sha256": "1f19d3883308c2b55c3de1392622686ff552ce3558a4cfd486e26b4f4db8c66f", "sources_unchanged": true, "raw_file_sha256": {"source-hashes.json": "1f19d3883308c2b55c3de1392622686ff552ce3558a4cfd486e26b4f4db8c66f", "stdout.log": "684b1b9e91a57ec8bb0d5a91e6bc27b354cd5591e553e45432343bdb2ca5858a"}}
+{"kind": "launch", "attempt": "repair2", "pass": "unit", "at": "2026-08-28T08:08:06.400899+00:00", "argv": ["./track", "unit-test"], "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "environment": {"HOME": "/Users/woopinbell", "PATH": "/opt/homebrew/opt/python@3.12/libexec/bin:/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/System/Cryptexes/App/usr/bin:/usr/bin:/bin:/usr/sbin:/sbin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/local/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/appleinternal/bin:/pkg/env/global/bin:/Library/Apple/usr/bin:/Users/woopinbell/.codex/tmp/arg0/codex-arg0Vyt1ls:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/override:/Users/woopinbell/.sdkman/candidates/java/current/bin:/Users/woopinbell/.sdkman/candidates/gradle/current/bin:/Users/woopinbell/.local/bin:/Users/woopinbell/development/flutter/bin:/Users/woopinbell/.pyenv/bin:/opt/homebrew/opt/python@3.12/libexec/bin:/Users/woopinbell/bin:/Users/woopinbell/.foundry/bin:/Users/woopinbell/.maestro/bin:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/fallback:/Applications/ChatGPT.app/Contents/Resources", "TMPDIR": "/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/", "LANG": "C.UTF-8", "LC_ALL": "C.UTF-8", "LC_CTYPE": "C.UTF-8", "USER": "woopinbell", "LOGNAME": "woopinbell", "SHELL": "/bin/zsh", "TERM": "dumb", "JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "runner_pid": 49163, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/verify-unit"}
+{"pass": "unit", "category": "full-unit-exact-two-frozen-pure-cases", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "unit-test"], "environment": {"HOME": "/Users/woopinbell", "PATH": "/opt/homebrew/opt/python@3.12/libexec/bin:/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/System/Cryptexes/App/usr/bin:/usr/bin:/bin:/usr/sbin:/sbin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/local/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/appleinternal/bin:/pkg/env/global/bin:/Library/Apple/usr/bin:/Users/woopinbell/.codex/tmp/arg0/codex-arg0Vyt1ls:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/override:/Users/woopinbell/.sdkman/candidates/java/current/bin:/Users/woopinbell/.sdkman/candidates/gradle/current/bin:/Users/woopinbell/.local/bin:/Users/woopinbell/development/flutter/bin:/Users/woopinbell/.pyenv/bin:/opt/homebrew/opt/python@3.12/libexec/bin:/Users/woopinbell/bin:/Users/woopinbell/.foundry/bin:/Users/woopinbell/.maestro/bin:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/fallback:/Applications/ChatGPT.app/Contents/Resources", "TMPDIR": "/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/", "LANG": "C.UTF-8", "LC_ALL": "C.UTF-8", "LC_CTYPE": "C.UTF-8", "USER": "woopinbell", "LOGNAME": "woopinbell", "SHELL": "/bin/zsh", "TERM": "dumb", "JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "attempt": "repair2", "started_at": "2026-08-28T08:08:06.400899+00:00", "finished_at": "2026-08-28T08:08:10.374884+00:00", "duration_seconds": 3.974153, "command_process_id": 49231, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/verify-unit/stdout.log", "output_sha256": "033d04418caeef3569408eb644bf4c8eddcf518e4107a379a4930667d44e3933", "process_terminated": true, "compiler_tasks_executed": [], "compiler_tasks_consumed": 2, "source_hashes": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/verify-unit/source-hashes.json", "source_hashes_sha256": "1f19d3883308c2b55c3de1392622686ff552ce3558a4cfd486e26b4f4db8c66f", "sources_unchanged": true, "xml": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/verify-unit/xml", "test_counts": {"tests": 47, "failures": 0, "errors": 0, "skipped": 0}, "raw_file_sha256": {"source-hashes.json": "1f19d3883308c2b55c3de1392622686ff552ce3558a4cfd486e26b4f4db8c66f", "stdout.log": "033d04418caeef3569408eb644bf4c8eddcf518e4107a379a4930667d44e3933", "xml/TEST-arena.CompleteFrameTest.xml": "45da81889e0466b074a9030e73f90baaefe5b4f6ad8ff76c1eac3beef1d3ab5d", "xml/TEST-arena.OutboundQueueTest.xml": "9faecda106159472b944150d2a7c0b33fcdcb2aeab4a997d3d23e32a54013603", "xml/TEST-arena.ReplayFormatTest.xml": "735106e8f7a5c8fa51376a45d8d7a731c50bf15b05d891d84e79a1e99328b323", "xml/TEST-arena.RoomTest.xml": "7dafa1e40d8668ab28033b7318e4789955e783f5fc39e15aa81b78fff1723ea2", "xml/TEST-arena.SnapshotStreamTest.xml": "8ee6cfe9c377376a415bffb4e14519bb0c3e6d12d97da05a61f8507c166d1d68", "xml/TEST-arena.UdpBoundaryTest.xml": "71d97e432bd021f6b118ce033c7348d82db22c268c173811ef43125269a1814a", "xml/binary/output.bin": "07d4663f97e977f94e628a1865713f010a731401625dc019d7ee77bdd22fa1d5", "xml/binary/output.bin.idx": "2caf0dd73d1bb82ea401e83b02810b6933df9cf8261724eebb76b10cb93551dc", "xml/binary/results.bin": "4c4ca20a41df778e5ecfcf02b93085033c8f3fa5a5321d3f88b704f280a3e276"}}
+{"kind": "launch", "attempt": "repair2", "pass": "integration", "at": "2026-08-28T08:08:10.387636+00:00", "argv": ["./track", "integration-test"], "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "environment": {"HOME": "/Users/woopinbell", "PATH": "/opt/homebrew/opt/python@3.12/libexec/bin:/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/System/Cryptexes/App/usr/bin:/usr/bin:/bin:/usr/sbin:/sbin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/local/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/appleinternal/bin:/pkg/env/global/bin:/Library/Apple/usr/bin:/Users/woopinbell/.codex/tmp/arg0/codex-arg0Vyt1ls:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/override:/Users/woopinbell/.sdkman/candidates/java/current/bin:/Users/woopinbell/.sdkman/candidates/gradle/current/bin:/Users/woopinbell/.local/bin:/Users/woopinbell/development/flutter/bin:/Users/woopinbell/.pyenv/bin:/opt/homebrew/opt/python@3.12/libexec/bin:/Users/woopinbell/bin:/Users/woopinbell/.foundry/bin:/Users/woopinbell/.maestro/bin:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/fallback:/Applications/ChatGPT.app/Contents/Resources", "TMPDIR": "/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/", "LANG": "C.UTF-8", "LC_ALL": "C.UTF-8", "LC_CTYPE": "C.UTF-8", "USER": "woopinbell", "LOGNAME": "woopinbell", "SHELL": "/bin/zsh", "TERM": "dumb", "JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "runner_pid": 49163, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/verify-integration"}
+{"pass": "integration", "category": "full-integration", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "integration-test"], "environment": {"HOME": "/Users/woopinbell", "PATH": "/opt/homebrew/opt/python@3.12/libexec/bin:/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/System/Cryptexes/App/usr/bin:/usr/bin:/bin:/usr/sbin:/sbin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/local/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/appleinternal/bin:/pkg/env/global/bin:/Library/Apple/usr/bin:/Users/woopinbell/.codex/tmp/arg0/codex-arg0Vyt1ls:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/override:/Users/woopinbell/.sdkman/candidates/java/current/bin:/Users/woopinbell/.sdkman/candidates/gradle/current/bin:/Users/woopinbell/.local/bin:/Users/woopinbell/development/flutter/bin:/Users/woopinbell/.pyenv/bin:/opt/homebrew/opt/python@3.12/libexec/bin:/Users/woopinbell/bin:/Users/woopinbell/.foundry/bin:/Users/woopinbell/.maestro/bin:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/fallback:/Applications/ChatGPT.app/Contents/Resources", "TMPDIR": "/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/", "LANG": "C.UTF-8", "LC_ALL": "C.UTF-8", "LC_CTYPE": "C.UTF-8", "USER": "woopinbell", "LOGNAME": "woopinbell", "SHELL": "/bin/zsh", "TERM": "dumb", "JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "attempt": "repair2", "started_at": "2026-08-28T08:08:10.387636+00:00", "finished_at": "2026-08-28T08:08:15.179219+00:00", "duration_seconds": 4.791743, "command_process_id": 49276, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/verify-integration/stdout.log", "output_sha256": "796674d2963feb4c0f83aaaf547cfb0fe9e35f48cca2fbbcc394723d9aa28d9b", "process_terminated": true, "compiler_tasks_executed": [], "compiler_tasks_consumed": 2, "source_hashes": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/verify-integration/source-hashes.json", "source_hashes_sha256": "1f19d3883308c2b55c3de1392622686ff552ce3558a4cfd486e26b4f4db8c66f", "sources_unchanged": true, "xml": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/verify-integration/xml", "test_counts": {"tests": 4, "failures": 0, "errors": 0, "skipped": 0}, "raw_file_sha256": {"source-hashes.json": "1f19d3883308c2b55c3de1392622686ff552ce3558a4cfd486e26b4f4db8c66f", "stdout.log": "796674d2963feb4c0f83aaaf547cfb0fe9e35f48cca2fbbcc394723d9aa28d9b", "xml/TEST-arena.ServerIntegrationTest.xml": "8568858d4a9dba778b4b1fb1ccd9ec2b835fe9c048a9200821e632936bc14116", "xml/binary/output.bin": "acdd5aba2c4c9711f7a4287ebb95000bafa1113e6f7f268376c51d4e8bc589c9", "xml/binary/output.bin.idx": "a3b47318cda4d0aa44a67b9a4ac42b44eef78dc4fa23075e287150e318504d6e", "xml/binary/results.bin": "4b74a1963a097abad6deaeabd6e0cdeedbce4607654d2aa210875581eec07990"}}
+{"kind": "launch", "attempt": "repair2", "pass": "canonical", "at": "2026-08-28T08:08:15.189949+00:00", "argv": ["./track", "scenario-run", "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G12.json", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/canonical/result.json"], "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "environment": {"HOME": "/Users/woopinbell", "PATH": "/opt/homebrew/opt/python@3.12/libexec/bin:/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/System/Cryptexes/App/usr/bin:/usr/bin:/bin:/usr/sbin:/sbin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/local/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/appleinternal/bin:/pkg/env/global/bin:/Library/Apple/usr/bin:/Users/woopinbell/.codex/tmp/arg0/codex-arg0Vyt1ls:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/override:/Users/woopinbell/.sdkman/candidates/java/current/bin:/Users/woopinbell/.sdkman/candidates/gradle/current/bin:/Users/woopinbell/.local/bin:/Users/woopinbell/development/flutter/bin:/Users/woopinbell/.pyenv/bin:/opt/homebrew/opt/python@3.12/libexec/bin:/Users/woopinbell/bin:/Users/woopinbell/.foundry/bin:/Users/woopinbell/.maestro/bin:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/fallback:/Applications/ChatGPT.app/Contents/Resources", "TMPDIR": "/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/", "LANG": "C.UTF-8", "LC_ALL": "C.UTF-8", "LC_CTYPE": "C.UTF-8", "USER": "woopinbell", "LOGNAME": "woopinbell", "SHELL": "/bin/zsh", "TERM": "dumb", "JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "runner_pid": 49163, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/canonical"}
+{"pass": "canonical", "category": "fixed-live-slow-consumer", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "scenario-run", "/Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G12.json", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/canonical/result.json"], "environment": {"HOME": "/Users/woopinbell", "PATH": "/opt/homebrew/opt/python@3.12/libexec/bin:/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/System/Cryptexes/App/usr/bin:/usr/bin:/bin:/usr/sbin:/sbin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/local/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/appleinternal/bin:/pkg/env/global/bin:/Library/Apple/usr/bin:/Users/woopinbell/.codex/tmp/arg0/codex-arg0Vyt1ls:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/override:/Users/woopinbell/.sdkman/candidates/java/current/bin:/Users/woopinbell/.sdkman/candidates/gradle/current/bin:/Users/woopinbell/.local/bin:/Users/woopinbell/development/flutter/bin:/Users/woopinbell/.pyenv/bin:/opt/homebrew/opt/python@3.12/libexec/bin:/Users/woopinbell/bin:/Users/woopinbell/.foundry/bin:/Users/woopinbell/.maestro/bin:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/fallback:/Applications/ChatGPT.app/Contents/Resources", "TMPDIR": "/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/", "LANG": "C.UTF-8", "LC_ALL": "C.UTF-8", "LC_CTYPE": "C.UTF-8", "USER": "woopinbell", "LOGNAME": "woopinbell", "SHELL": "/bin/zsh", "TERM": "dumb", "JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "attempt": "repair2", "started_at": "2026-08-28T08:08:15.189949+00:00", "finished_at": "2026-08-28T08:08:16.392755+00:00", "duration_seconds": 1.203375, "command_process_id": 49417, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/canonical/stdout.log", "output_sha256": "db62faf612b77d5983c85f989e6849956348e58d2aba34d1b3065ab229cfbd81", "process_terminated": true, "compiler_tasks_executed": [], "compiler_tasks_consumed": 2, "source_hashes": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/canonical/source-hashes.json", "source_hashes_sha256": "1f19d3883308c2b55c3de1392622686ff552ce3558a4cfd486e26b4f4db8c66f", "sources_unchanged": true, "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/canonical/result.json", "result_sha256": "1d35225c222a2ffe1f54f8beedfa3721aab542ba8e9e29b9eb070024262d934f", "simulation_process_id": 49417, "executed_ticks": 100, "result_passed": true, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/canonical/result.replay.jsonl", "artifact_sha256": "1516780ff7f9caffc558f6a4ab5299a862251270fa278844aa9dd0f11a880326", "raw_file_sha256": {"result.json": "1d35225c222a2ffe1f54f8beedfa3721aab542ba8e9e29b9eb070024262d934f", "result.replay.jsonl": "1516780ff7f9caffc558f6a4ab5299a862251270fa278844aa9dd0f11a880326", "source-hashes.json": "1f19d3883308c2b55c3de1392622686ff552ce3558a4cfd486e26b4f4db8c66f", "stdout.log": "db62faf612b77d5983c85f989e6849956348e58d2aba34d1b3065ab229cfbd81"}}
+{"kind": "launch", "attempt": "repair2", "pass": "reference", "at": "2026-08-28T08:08:16.424209+00:00", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/canonical/result.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/reference/result.json"], "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "environment": {"HOME": "/Users/woopinbell", "PATH": "/opt/homebrew/opt/python@3.12/libexec/bin:/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/System/Cryptexes/App/usr/bin:/usr/bin:/bin:/usr/sbin:/sbin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/local/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/appleinternal/bin:/pkg/env/global/bin:/Library/Apple/usr/bin:/Users/woopinbell/.codex/tmp/arg0/codex-arg0Vyt1ls:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/override:/Users/woopinbell/.sdkman/candidates/java/current/bin:/Users/woopinbell/.sdkman/candidates/gradle/current/bin:/Users/woopinbell/.local/bin:/Users/woopinbell/development/flutter/bin:/Users/woopinbell/.pyenv/bin:/opt/homebrew/opt/python@3.12/libexec/bin:/Users/woopinbell/bin:/Users/woopinbell/.foundry/bin:/Users/woopinbell/.maestro/bin:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/fallback:/Applications/ChatGPT.app/Contents/Resources", "TMPDIR": "/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/", "LANG": "C.UTF-8", "LC_ALL": "C.UTF-8", "LC_CTYPE": "C.UTF-8", "USER": "woopinbell", "LOGNAME": "woopinbell", "SHELL": "/bin/zsh", "TERM": "dumb", "JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "runner_pid": 49163, "output_directory": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/reference"}
+{"pass": "reference", "category": "accepted-journal-offline", "cwd": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java", "argv": ["./track", "replay-verify", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/canonical/result.replay.jsonl", "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/reference/result.json"], "environment": {"HOME": "/Users/woopinbell", "PATH": "/opt/homebrew/opt/python@3.12/libexec/bin:/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/System/Cryptexes/App/usr/bin:/usr/bin:/bin:/usr/sbin:/sbin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/local/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/appleinternal/bin:/pkg/env/global/bin:/Library/Apple/usr/bin:/Users/woopinbell/.codex/tmp/arg0/codex-arg0Vyt1ls:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/override:/Users/woopinbell/.sdkman/candidates/java/current/bin:/Users/woopinbell/.sdkman/candidates/gradle/current/bin:/Users/woopinbell/.local/bin:/Users/woopinbell/development/flutter/bin:/Users/woopinbell/.pyenv/bin:/opt/homebrew/opt/python@3.12/libexec/bin:/Users/woopinbell/bin:/Users/woopinbell/.foundry/bin:/Users/woopinbell/.maestro/bin:/Users/woopinbell/.cache/codex-runtimes/codex-primary-runtime/dependencies/bin/fallback:/Applications/ChatGPT.app/Contents/Resources", "TMPDIR": "/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/", "LANG": "C.UTF-8", "LC_ALL": "C.UTF-8", "LC_CTYPE": "C.UTF-8", "USER": "woopinbell", "LOGNAME": "woopinbell", "SHELL": "/bin/zsh", "TERM": "dumb", "JAVA_HOME": "/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"}, "kind": "executed", "attempt": "repair2", "started_at": "2026-08-28T08:08:16.424209+00:00", "finished_at": "2026-08-28T08:08:16.656207+00:00", "duration_seconds": 0.232215, "command_process_id": 49439, "exit_code": 0, "output": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/reference/stdout.log", "output_sha256": "faf6203ab338f6155deb35c7c8a0bea3cb6eacc4207ac8b08629774ce915a6ec", "process_terminated": true, "compiler_tasks_executed": [], "compiler_tasks_consumed": 2, "source_hashes": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/reference/source-hashes.json", "source_hashes_sha256": "1f19d3883308c2b55c3de1392622686ff552ce3558a4cfd486e26b4f4db8c66f", "sources_unchanged": true, "result": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/reference/result.json", "result_sha256": "72ec210a8159223e2ae6a712876065f70675181b2e74f891d3685eefe00fa390", "simulation_process_id": 49439, "executed_ticks": 100, "result_passed": true, "artifact": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/canonical/result.replay.jsonl", "artifact_sha256": "1516780ff7f9caffc558f6a4ab5299a862251270fa278844aa9dd0f11a880326", "raw_file_sha256": {"result.json": "72ec210a8159223e2ae6a712876065f70675181b2e74f891d3685eefe00fa390", "source-hashes.json": "1f19d3883308c2b55c3de1392622686ff552ce3558a4cfd486e26b4f4db8c66f", "stdout.log": "faf6203ab338f6155deb35c7c8a0bea3cb6eacc4207ac8b08629774ce915a6ec"}}
+{"kind": "fixed_sequence_finished", "attempt": "repair2", "at": "2026-08-28T08:08:16.661691+00:00", "passes": ["build", "unit", "integration", "canonical", "reference"], "compiler_tasks": 2, "unit_runs": 1, "integration_runs": 1, "live_runs": 1, "live_ticks": 100, "reference_runs": 1, "reference_ticks": 100, "baseline_runs": 0, "fault_runs": 0, "load_runs": 0, "profiler_runs": 0, "runtime_retries": 0}
+{"kind": "raw_artifact_audit", "attempt": "repair2", "at": "2026-08-28T08:11:06.737840+00:00", "audit": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/verification-audit.json", "audit_sha256": "66cad27c5309ab25ff7a0ed6c9b7b1074d95e6f20b949cc3c8cbbaa36486bd73", "live_reference100_equal": true, "g03_finished1200_suppression0": true, "two_fixed_pure_cases_passed": true, "source_files_unchanged": 45, "additional_compiler_or_runtime_invocations": 0}
+{"kind": "ready_for_commit", "attempt": "repair2", "at": "2026-08-28T08:13:58.905716+00:00", "worker_verification_passed": true, "root_worker_artifact_review_passed": true, "root_immutable_end_verification": "pending", "compiler_tasks_this_repair": 2, "clean_builds_this_repair": 1, "unit_runs_this_repair": 1, "unit_tests": 47, "integration_runs_this_repair": 1, "integration_tests": 4, "live_runs_this_repair": 1, "live_ticks": 100, "reference_runs_this_repair": 1, "reference_ticks": 100, "baseline_repeats": 0, "runtime_retries": 0, "fault_runs": 0, "load_runs": 0, "profiler_runs": 0, "initial_plus_repairs_compiler_tasks": 7, "initial_plus_repairs_unit_invocations": 4, "repairs_consumed": 2, "source_scope": "initial G12 implementation plus unchanged repair1 test and approved OutboundQueue-only repair2", "source_manifest_sha256": "1f19d3883308c2b55c3de1392622686ff552ce3558a4cfd486e26b4f4db8c66f", "note": "evidence/G12-verification.md", "note_sha256": "0992e79bdb15d8cb92cdbde347f732e93decc51409b31cea4a8e905c7399dd6b", "commit_message": "/private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/runs/g12-repair2/commit-message.txt", "executed_git_hooks": [], "further_runtime_commands": 0}
diff --git a/evidence/G12-verification.md b/evidence/G12-verification.md
new file mode 100644
index 0000000..5d626b4
--- /dev/null
+++ b/evidence/G12-verification.md
@@ -0,0 +1,60 @@
+# G12 verification — Java, final bounded repair2
+
+START: `10325022b2774d3284544f12680ba9ba41f77149`. Branch: `track/industry-java`. Profile: `realtime-core`. Spec revision: `c1d62196ab76b55652f5d75a67514f8c6d8210ce`. Frozen fixture SHA-256: `b5350ef5bfb9fa93cbde1fe0fd30079e6115fe26988514be59d63d9dee6bc6f3`.
+
+Exact commands, timestamps, exits, compiler tasks, environments and artifact hashes are preserved in the three G12 command ledgers. Raw evidence remains under `evidence/runs/g12-initial/`, `g12-repair1/` and `g12-repair2/` in this worktree. No earlier failure, source archive or consumed budget was reset.
+
+## Reproduction and repair history
+
+The initial baseline ran `./track unit-test --tests arena.G12BaselineTest` against unchanged G11 production for the fixed100 ticks. Setting the real alpha channel's readiness false exposed that START had no per-peer pre-dequeue gate: alpha still sent and received50 snapshots. The evidence explicitly does not claim a successful hold or invented queue growth. Its intended assertion failed, with clean shutdown. Baseline source hashes, harness,100 records/hashes and raw XML remain under `g12-initial/reproduce-unit/`.
+
+The initial implementation adds the production per-peer outbound queue and the fixed scenario observer. The first full47-test run failed the mixed pure case's comparison between an in-memory numeric node and parsed JSON despite identical wire text. Repair1 changed only that test to exact wire-byte equality with finally-release; both fixed pure cases then passed. Repair1's full unit run failed unchanged G03 FINISHED/LEAVE_ROOM at1016 ticks, generated1018 snapshots and suppressed one still-marked-in-flight snapshot. Its adjacent FINISHED/CONNECTION_CLOSE cell completed1200 ticks with suppression0. Neither failed attempt ran integration, post-live or reference verification.
+
+Repair2 verified all45 source/resource files against root's preserved repair1 tree and all17 production files against the initial failure before changing only `OutboundQueue.java`. The successful test repair remains byte-identical. Root reviewed the exact repair diff before verification: SHA-256 `2862bd20f7fd34d5eb8e6cccf0b3842ef09863a53fda1769708ff3b354af9353`.
+
+## Ownership correction
+
+One short per-connection monitor now covers admission and one nonblocking transfer attempt. A promise is installed before transfer, and an already-completed promise is retired before releasing the transfer monitor or admitting another message. Retirement is idempotent when a deferred listener later runs. Completion schedules at most one flush task instead of recursively draining under that monitor.
+
+The pinned Netty4.1.114.Final bytecode confirms that NIO channels are nonblocking, write-spin defaults to16, and the successful outbound removal releases its buffer before completing its promise. TCP and UDP share the existing single I/O event loop, with at most eight connections. A held peer fails readiness before detach; no owner waits for peer readiness or future completion. The owner can contend for the bounded transfer critical section, which does not promise a hard wall-time deadline under OS preemption.
+
+No Netty-owned buffer is manually released. Queued and transport-owned FULL/DELTA buffers still count together, at most one each; ordered/control work retains its existing64 bound. New FULL coalesces queued FULL and DELTA; new DELTA coalesces queued DELTA; controls retain order. True incomplete Netty writes still occupy their slot and retain the existing same-kind suppression behavior. That branch was not exercised by the successful fixed runs (suppression0); it must not be described as the completed-write bookkeeping race that this repair fixes. No second pending snapshot, deferred snapshot payload, new game policy, dependency or fixture was introduced.
+
+## Final worker verification
+
+One pre-resolved sequence ran under pinned Java21.0.7/offline Gradle with Netty PARANOID checks; it stopped on failures by construction and needed no retry.
+
+| Command | Actual result |
+|---|---|
+| `./track build` | Exit0; one clean build, two compiler tasks. |
+| `./track unit-test` | Exit0;47 passed,0 failures/errors/skips. |
+| `./track integration-test` | Exit0;4 passed,0 failures/errors/skips. |
+| `./track scenario-run <original main G12.json> <fresh canonical/result.json>` | Exit0; one live100-tick run, PID49417. |
+| `./track replay-verify <canonical/result.replay.jsonl> <fresh reference/result.json>` | Exit0; separate accepted-journal offline100-tick run, PID49439; zero server instances. |
+
+Both unchanged G03 FINISHED cells now complete1200 ticks (last tick1199), generate1202 snapshots, coalesce0 and suppress0. Alpha and bravo both receive ROOM_FINISHED. All six lifecycle cells have suppression0 and zero-resource shutdown. The exact two G12 pure cases pass:63 controls remain open; attempt64 reports CONTROL_BACKPRESSURE within64, releases64 unsent buffers and claims0 deliveries. The mixed case preserves controls A/B and discards exactly three superseded snapshot buffers in two replacements; all superseded/cancelled refs reach0 and unsent FULL5 is not reported delivered. No third pure case or extra live campaign was added.
+
+All45 source/resource hashes stayed unchanged across every final command. Manifest SHA-256: `1f19d3883308c2b55c3de1392622686ff552ce3558a4cfd486e26b4f4db8c66f`. The raw-artifact audit is `g12-repair2/verification-audit.json`; its G03 extraction and original XML remain beside the unit output.
+
+## Fixed live and reference evidence
+
+During the hold, alpha generates50 snapshots, sends/receives0, coalesces49 and releases all49 superseded buffers. Its pending FULL/DELTA high-water marks are1/1, retained-byte high water1072, and final held queue is one713-byte FULL. Actual ownership-bound violations are0. The final held buffer is cancelled/released on alpha TCP close; transport/queue buffers and retained bytes then reach0. G11 legitimately preserves alpha as DISCONNECTED/STOP until Room/server shutdown.
+
+Bravo, charlie and delta each receive/apply/ACK sequences1..51 at ticks-1,1,3,..99. All153 snapshot hashes and visible projections match their actual capture. Bravo reaches `(100000,90000)` at tick24 and remains EAST/seq1/score0; the others remain at their specified spawn, STOP/seq0/score0. Everyone is CONNECTED through capture99.
+
+Live and offline have exactly equal100 canonical records,100 state hashes and final state. The accepted journal is60,439 bytes; the offline pass uses the existing replay-format verifier and creates no live server.
+
+- Tick0 hash: `dfad72b22bbfedcb847fe5551c4a67016ef4e7c91df91ade99f51dade7a4e325`.
+- Tick99 hash: `5d7f602c8e4eb781e28b0875e44cdda38df451fb27434a04b8963b6d6bc9f958`.
+- SHA-256 of all100 lowercase hashes joined with LF, including final LF: `39e6c21a5c4b671a19d8462d923c2d6769528cb622b3f07b0e9bd3704b1be52c`.
+- Accepted journal SHA-256: `1516780ff7f9caffc558f6a4ab5299a862251270fa278844aa9dd0f11a880326`.
+- Live result SHA-256: `1d35225c222a2ffe1f54f8beedfa3721aab542ba8e9e29b9eb070024262d934f`.
+- Reference result SHA-256: `72ec210a8159223e2ae6a712876065f70675181b2e74f891d3685eefe00fa390`.
+
+The151 observed actual transfers all complete with refcount0; all50 observed queued snapshot buffer lifetimes finish at0. All218 allocated outbound buffers are accounted for:168 sent,49 superseded and1 cancelled. Final queued/transport buffers and bytes, parser/UDP buffers, channels, sessions/recoverable sessions, credentials, snapshot/replay retention, mailbox work and live threads are0; timer stopped, owner/event loops terminated. Actual global retained-byte high water is2852; transport high water is one buffer per peer. No leak or ownership-mismatch diagnostic appeared in final command logs.
+
+## Budget and remaining gate
+
+Repair2 consumed2/8 compiler tasks, one clean build, one full unit run, one integration run, one live100 and one separate offline100. Baseline repeats, runtime retries, additional fault/load/profiler runs:0. Across initial+repair1+repair2:7 compiler tasks,4 unit invocations including the original baseline, one integration, one post-live and one reference; repairs2/2 are consumed. Earlier baseline and both failed full unit runs remain visible.
+
+Worker verification and root's review of these worker artifacts passed. Root's separate fresh verification against immutable END remains required. No G13 work, main/spec/index/threads write, push, history rewrite, progress tag or external infrastructure was performed.
diff --git a/src/main/java/arena/ArenaServer.java b/src/main/java/arena/ArenaServer.java
index 2aed916..593f22c 100644
--- a/src/main/java/arena/ArenaServer.java
+++ b/src/main/java/arena/ArenaServer.java
@@ -3,7 +3,6 @@ package arena;
 import com.fasterxml.jackson.databind.node.ObjectNode;
 import io.netty.bootstrap.Bootstrap;
 import io.netty.bootstrap.ServerBootstrap;
-import io.netty.buffer.Unpooled;
 import io.netty.channel.Channel;
 import io.netty.channel.ChannelHandlerContext;
 import io.netty.channel.ChannelInboundHandlerAdapter;
@@ -13,7 +12,6 @@ import io.netty.channel.DefaultSelectStrategyFactory;
 import io.netty.channel.FixedRecvByteBufAllocator;
 import io.netty.channel.nio.NioEventLoopGroup;
 import io.netty.channel.socket.SocketChannel;
-import io.netty.channel.socket.DatagramPacket;
 import io.netty.channel.socket.nio.NioDatagramChannel;
 import io.netty.channel.socket.nio.NioServerSocketChannel;
 import io.netty.util.concurrent.DefaultEventExecutorChooserFactory;
@@ -58,10 +56,11 @@ public final class ArenaServer implements AutoCloseable {
     private final Set<Peer> peers = ConcurrentHashMap.newKeySet();
     private final Set<Thread> ownedThreads = ConcurrentHashMap.newKeySet();
     private final AtomicInteger connections = new AtomicInteger();
-    private final AtomicInteger pendingWrites = new AtomicInteger();
+    private final OutboundQueue.Metrics outboundMetrics = new OutboundQueue.Metrics();
+    private final AtomicInteger pendingWrites = outboundMetrics.transportBuffers;
     private final CompleteFrame.Metrics parserMetrics = new CompleteFrame.Metrics();
     private final UdpData.Metrics udpMetrics = new UdpData.Metrics();
-    private final AtomicInteger outboundHighWater = new AtomicInteger();
+    private final AtomicInteger outboundHighWater = outboundMetrics.transportPerPeerHighWater;
     private final AtomicInteger mailboxHighWater = new AtomicInteger();
     private final AtomicBoolean closing = new AtomicBoolean();
     private final ThreadPoolExecutor owner;
@@ -108,55 +107,22 @@ public final class ArenaServer implements AutoCloseable {
 
     private final class Peer {
         final Channel channel;
-        final AtomicInteger outbound = new AtomicInteger();
+        final OutboundQueue outbound;
         final AtomicBoolean registered = new AtomicBoolean();
 
-        Peer(Channel channel) { this.channel = channel; }
+        Peer(Channel channel) { this.channel = channel; outbound = new OutboundQueue(channel, () -> udpListener, outboundMetrics, udpMetrics); }
 
         void send(ObjectNode message) {
             send(message, false);
         }
 
         void send(ObjectNode message, boolean terminal) {
-            if (!channel.isActive()) return;
-            int count = outbound.incrementAndGet();
-            if (count > OUTBOUND_LIMIT) {
-                outbound.decrementAndGet();
-                channel.close();
-                return;
-            }
-            outboundHighWater.accumulateAndGet(count, Math::max);
-            if (count == OUTBOUND_LIMIT) {
-                message = CompleteFrame.error("CONTROL_BACKPRESSURE", "control message bound reached");
-                terminal = true;
-            }
-            var buffer = CompleteFrame.encode(message);
-            pendingWrites.incrementAndGet();
-            boolean closeAfterWrite = terminal;
-            // Netty takes ownership even when the asynchronous write fails.
-            channel.writeAndFlush(buffer).addListener(result -> {
-                pendingWrites.decrementAndGet();
-                outbound.decrementAndGet();
-                if (!result.isSuccess() || closeAfterWrite) channel.close();
-            });
+            outbound.send(message, terminal);
         }
         void error(String code) { send(CompleteFrame.error(code, code)); }
 
         void realtime(ObjectNode message, InetSocketAddress endpoint) {
-            if (!channel.isActive()) return;
-            byte[] bytes;
-            try { bytes = UdpData.bytes(message); }
-            catch (IllegalArgumentException tooLarge) { udpMetrics.outboundOversize.incrementAndGet(); return; }
-            int count = outbound.incrementAndGet();
-            if (count >= OUTBOUND_LIMIT) {
-                outbound.decrementAndGet(); send(CompleteFrame.error("CONTROL_BACKPRESSURE", "control message bound reached"), true); return;
-            }
-            outboundHighWater.accumulateAndGet(count, Math::max); udpMetrics.outputHighWater.accumulateAndGet(bytes.length, Math::max);
-            pendingWrites.incrementAndGet();
-            udpListener.writeAndFlush(new DatagramPacket(Unpooled.wrappedBuffer(bytes), endpoint)).addListener(result -> {
-                pendingWrites.decrementAndGet(); outbound.decrementAndGet();
-                if (!result.isSuccess()) { udpMetrics.ioErrors.incrementAndGet(); channel.close(); }
-            });
+            outbound.realtime(message, endpoint);
         }
     }
 
@@ -227,6 +193,12 @@ public final class ArenaServer implements AutoCloseable {
                             execute(() -> handleUdp(packet)); udpMetrics.dispatched.incrementAndGet();
                         } catch (RejectedExecutionException full) { udpMetrics.mailboxDrops.incrementAndGet(); }
                     }, udpMetrics)).bind(host, 0).syncUninterruptibly().channel();
+            datagramBound.pipeline().addLast(new ChannelInboundHandlerAdapter() {
+                @Override public void channelWritabilityChanged(ChannelHandlerContext context) {
+                    if (context.channel().isWritable()) for (Peer peer : peers) peer.outbound.flushWhenReady();
+                    context.fireChannelWritabilityChanged();
+                }
+            });
         } catch (Exception failure) {
             if (datagramBound != null) datagramBound.close().syncUninterruptibly();
             if (bound != null) bound.close().syncUninterruptibly();
@@ -299,7 +271,7 @@ public final class ArenaServer implements AutoCloseable {
     }
 
     private void handleCommand(Peer peer, ObjectNode message, InetSocketAddress endpoint) {
-        if (closing.get() || !peer.channel.isActive()) return;
+        if (closing.get() || !peer.channel.isActive() || peer.outbound.terminal()) return;
         try {
             if (message.path("v").asInt(-1) != 1) { peer.error("PROTOCOL_VERSION_UNSUPPORTED"); return; }
             String type = Json.text(message, "type");
@@ -516,6 +488,7 @@ public final class ArenaServer implements AutoCloseable {
             result.set("clock", fixedClock == null ? closedClockMetrics.deepCopy() : fixedClock.view());
             result.set("parser", parserMetrics.view());
             result.set("udp", udpMetrics.view());
+            result.set("outbound", outboundMetrics.view());
             result.put("udp_bindings", sessions.values().stream().filter(s -> s.endpoint != null).count());
             result.put("active_sessions", sessions.size()).put("recoverable_sessions", recoverableSessions.size())
                     .put("resume_credentials", recoverableSessions.values().stream().filter(s -> s.resumeToken != null).count());
@@ -547,6 +520,7 @@ public final class ArenaServer implements AutoCloseable {
                 .put("mailbox_high_water", mailboxHighWater.get()).put("outbound_high_water", outboundHighWater.get());
         result.set("parser", parserMetrics.view());
         result.set("udp", udpMetrics.view());
+        result.set("outbound", outboundMetrics.view());
         result.set("clock", closedClockMetrics.deepCopy());
         var lifecycle = result.putArray("room_lifecycle");
         closedLifecycle.forEach(lifecycle::add);
diff --git a/src/main/java/arena/OutboundQueue.java b/src/main/java/arena/OutboundQueue.java
new file mode 100644
index 0000000..1b72602
--- /dev/null
+++ b/src/main/java/arena/OutboundQueue.java
@@ -0,0 +1,234 @@
+package arena;
+
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import io.netty.buffer.ByteBuf;
+import io.netty.buffer.Unpooled;
+import io.netty.channel.Channel;
+import io.netty.channel.ChannelHandlerContext;
+import io.netty.channel.ChannelInboundHandlerAdapter;
+import io.netty.channel.ChannelPromise;
+import io.netty.channel.socket.DatagramPacket;
+import java.net.InetSocketAddress;
+import java.util.ArrayDeque;
+import java.util.concurrent.RejectedExecutionException;
+import java.util.concurrent.atomic.AtomicBoolean;
+import java.util.concurrent.atomic.AtomicInteger;
+import java.util.concurrent.atomic.AtomicLong;
+import java.util.function.Supplier;
+
+/** One connection owns queued buffers; at most one detached write belongs to Netty. */
+final class OutboundQueue implements AutoCloseable {
+    private enum Kind { ORDERED, FULL, DELTA }
+    private record Message(ByteBuf buffer, Kind kind, InetSocketAddress endpoint, boolean terminal, int bytes) { }
+    static final class Metrics {
+        final AtomicInteger queuedBuffers = new AtomicInteger(), queuedBytes = new AtomicInteger();
+        final AtomicInteger transportBuffers = new AtomicInteger(), transportBytes = new AtomicInteger();
+        final AtomicInteger retainedBytes = new AtomicInteger(), retainedBytesHighWater = new AtomicInteger();
+        final AtomicInteger controlHighWater = new AtomicInteger(), fullHighWater = new AtomicInteger(), deltaHighWater = new AtomicInteger();
+        final AtomicInteger transportPerPeerHighWater = new AtomicInteger();
+        final AtomicLong generatedSnapshots = new AtomicLong(), coalescedSnapshots = new AtomicLong(), cancelledMessages = new AtomicLong();
+        final AtomicLong supersededReleases = new AtomicLong(), supersededReleaseErrors = new AtomicLong(), sentMessages = new AtomicLong();
+        final AtomicLong allocatedBuffers = new AtomicLong();
+        final AtomicLong suppressedSnapshots = new AtomicLong();
+        final AtomicLong cancelledReleases = new AtomicLong(), cancelledReleaseErrors = new AtomicLong();
+        ObjectNode view() {
+            return Json.MAPPER.createObjectNode().put("queued_buffers", queuedBuffers.get()).put("queued_bytes", queuedBytes.get())
+                    .put("transport_buffers", transportBuffers.get()).put("transport_bytes", transportBytes.get()).put("retained_bytes", retainedBytes.get())
+                    .put("retained_bytes_high_water", retainedBytesHighWater.get()).put("control_high_water", controlHighWater.get())
+                    .put("full_high_water", fullHighWater.get()).put("delta_high_water", deltaHighWater.get())
+                    .put("transport_per_peer_high_water", transportPerPeerHighWater.get()).put("generated_snapshots", generatedSnapshots.get())
+                    .put("coalesced_snapshots", coalescedSnapshots.get()).put("superseded_releases", supersededReleases.get())
+                    .put("superseded_release_errors", supersededReleaseErrors.get()).put("cancelled_messages", cancelledMessages.get()).put("sent_messages", sentMessages.get())
+                    .put("allocated_buffers", allocatedBuffers.get()).put("suppressed_inflight_snapshots", suppressedSnapshots.get())
+                    .put("cancelled_releases", cancelledReleases.get()).put("cancelled_release_errors", cancelledReleaseErrors.get());
+        }
+    }
+
+    private final Channel tcp;
+    private final Supplier<Channel> udp;
+    private final Metrics metrics;
+    private final UdpData.Metrics udpMetrics;
+    // Admission and one nonblocking transfer share the ownership monitor; no readiness/future wait holds it.
+    private final ArrayDeque<Message> ordered = new ArrayDeque<>(ArenaServer.OUTBOUND_LIMIT);
+    private Message full, delta, inFlight;
+    private ChannelPromise inFlightWrite;
+    private final AtomicBoolean flushScheduled = new AtomicBoolean();
+    private boolean closed, terminalQueued, backpressured, terminalSent;
+    private String terminalCode;
+    private int queuedBytes, pendingControlHighWater, pendingFullHighWater, pendingDeltaHighWater, retainedBytesHighWater;
+    private long generatedSnapshots, coalescedSnapshots, supersededReleases, cancelledMessages, sentMessages, sentSnapshots;
+    private long readinessChecks, readinessDeferrals, suppressedSnapshots, cancelledReleases;
+
+    OutboundQueue(Channel tcp, Supplier<Channel> udp, Metrics metrics, UdpData.Metrics udpMetrics) {
+        this.tcp = tcp; this.udp = udp; this.metrics = metrics; this.udpMetrics = udpMetrics;
+        tcp.pipeline().addLast(new ChannelInboundHandlerAdapter() {
+            @Override public void channelWritabilityChanged(ChannelHandlerContext context) {
+                if (context.channel().isWritable()) flushWhenReady(); context.fireChannelWritabilityChanged();
+            }
+            @Override public void channelInactive(ChannelHandlerContext context) { close(); context.fireChannelInactive(); }
+        });
+    }
+
+    void send(ObjectNode message, boolean terminal) { enqueue(message, null, terminal); }
+    void realtime(ObjectNode message, InetSocketAddress endpoint) { enqueue(message, endpoint, false); }
+
+    private void enqueue(ObjectNode message, InetSocketAddress endpoint, boolean terminal) {
+        if (!tcp.isActive()) return;
+        boolean accepted;
+        try { accepted = admit(message, endpoint, terminal); }
+        catch (IllegalArgumentException oversized) {
+            if (endpoint == null) throw oversized;
+            udpMetrics.outboundOversize.incrementAndGet(); return;
+        }
+        if (accepted) { if (terminal()) tcp.config().setAutoRead(false); flushWhenReady(); }
+        else if (!isClosed()) tcp.close();
+    }
+
+    private synchronized boolean admit(ObjectNode value, InetSocketAddress endpoint, boolean terminal) {
+        retireCompletedWrite();
+        if (closed) return false;
+        Kind kind = value.path("type").asText().equals("SNAPSHOT")
+                ? (value.path("kind").asText().equals("FULL") ? Kind.FULL : Kind.DELTA) : Kind.ORDERED;
+        if (kind == Kind.ORDERED) {
+            if (terminalQueued || orderedCount() >= ArenaServer.OUTBOUND_LIMIT) return false;
+            if (orderedCount() == ArenaServer.OUTBOUND_LIMIT - 1) {
+                value = CompleteFrame.error("CONTROL_BACKPRESSURE", "control message bound reached"); endpoint = null; terminal = true;
+                backpressured = true;
+            }
+        }
+        byte[] payload = endpoint == null ? Json.bytes(value) : UdpData.bytes(value);
+        if (endpoint == null && (payload.length < 1 || payload.length > CompleteFrame.MAX_BYTES)) throw new IllegalArgumentException("control payload byte bound");
+        if (kind != Kind.ORDERED) {
+            generatedSnapshots++; metrics.generatedSnapshots.incrementAndGet();
+            // Netty-owned data is immutable: reject a second same-kind buffer until real completion.
+            if (inFlight != null && inFlight.kind == kind) { suppressedSnapshots++; metrics.suppressedSnapshots.incrementAndGet(); return true; }
+        }
+        if (kind == Kind.FULL) { discard(full, true); discard(delta, true); full = null; delta = null; }
+        else if (kind == Kind.DELTA) { discard(delta, true); delta = null; }
+        int bytes = payload.length + (endpoint == null ? 4 : 0);
+        ByteBuf buffer = Unpooled.directBuffer(bytes, bytes);
+        if (endpoint == null) buffer.writeInt(payload.length); else udpMetrics.outputHighWater.accumulateAndGet(payload.length, Math::max);
+        buffer.writeBytes(payload); Message message = new Message(buffer, kind, endpoint, terminal, bytes); metrics.allocatedBuffers.incrementAndGet();
+        queuedBytes += bytes; metrics.queuedBuffers.incrementAndGet(); metrics.queuedBytes.addAndGet(bytes);
+        metrics.retainedBytesHighWater.accumulateAndGet(metrics.retainedBytes.addAndGet(bytes), Math::max);
+        if (kind == Kind.ORDERED) { ordered.addLast(message); if (terminal) { terminalQueued = true; terminalCode = value.path("code").asText(); } }
+        else if (kind == Kind.FULL) full = message; else delta = message;
+        int controls = controlCount(); pendingControlHighWater = Math.max(pendingControlHighWater, controls);
+        pendingFullHighWater = Math.max(pendingFullHighWater, kindCount(Kind.FULL)); pendingDeltaHighWater = Math.max(pendingDeltaHighWater, kindCount(Kind.DELTA));
+        retainedBytesHighWater = Math.max(retainedBytesHighWater, queuedBytes + (inFlight == null ? 0 : inFlight.bytes));
+        metrics.controlHighWater.accumulateAndGet(controls, Math::max); metrics.fullHighWater.accumulateAndGet(pendingFullHighWater, Math::max);
+        metrics.deltaHighWater.accumulateAndGet(pendingDeltaHighWater, Math::max); return true;
+    }
+
+    private int orderedCount() { return ordered.size() + (inFlight != null && inFlight.kind == Kind.ORDERED ? 1 : 0); }
+    private int kindCount(Kind kind) {
+        return ((kind == Kind.FULL ? full != null : delta != null) ? 1 : 0) + (inFlight != null && inFlight.kind == kind ? 1 : 0);
+    }
+    private int controlCount() {
+        int count = inFlight != null && inFlight.endpoint == null ? 1 : 0;
+        for (Message message : ordered) if (message.endpoint == null) count++;
+        return count;
+    }
+    private synchronized boolean isClosed() { return closed; }
+    synchronized boolean terminal() { return terminalQueued || closed; }
+    private synchronized boolean abortBackpressure(boolean tcpReady, boolean udpReady) {
+        Message next = ordered.peekFirst();
+        return backpressured && (inFlight != null || !tcpReady || next != null && next.endpoint != null && !udpReady);
+    }
+
+    void flushWhenReady() {
+        if (tcp.eventLoop().inEventLoop()) { flush(); return; }
+        scheduleFlush();
+    }
+
+    private void scheduleFlush() {
+        if (!flushScheduled.compareAndSet(false, true)) return;
+        try { tcp.eventLoop().execute(() -> { flushScheduled.set(false); flush(); }); }
+        catch (RejectedExecutionException rejected) { flushScheduled.set(false); close(); tcp.close(); }
+    }
+
+    private synchronized Message detach(boolean tcpReady, boolean udpReady) {
+        if (closed || inFlight != null) return null;
+        Message next = !ordered.isEmpty() ? ordered.peekFirst() : full != null ? full : delta;
+        if (next == null) return null;
+        readinessChecks++;
+        if (!tcpReady || next.endpoint != null && !udpReady) { readinessDeferrals++; return null; }
+        if (next.kind == Kind.ORDERED) ordered.removeFirst(); else if (next.kind == Kind.FULL) full = null; else delta = null;
+        inFlight = next; int bytes = next.bytes; queuedBytes -= bytes;
+        metrics.queuedBuffers.decrementAndGet(); metrics.queuedBytes.addAndGet(-bytes);
+        metrics.transportBuffers.incrementAndGet(); metrics.transportBytes.addAndGet(bytes); metrics.transportPerPeerHighWater.accumulateAndGet(1, Math::max);
+        return next;
+    }
+
+    private synchronized void flush() {
+        retireCompletedWrite();
+        if (!tcp.isActive()) { close(); return; }
+        Channel datagram = udp.get(); boolean udpReady = datagram != null && datagram.isActive() && datagram.isWritable();
+        if (abortBackpressure(tcp.isWritable(), udpReady)) { close(); tcp.close(); return; }
+        Message message = detach(tcp.isWritable(), udpReady);
+        if (message == null) return;
+        Channel target = message.endpoint == null ? tcp : datagram;
+        // The pinned NIO pipeline never waits for readiness: one bounded write attempt or Netty retention.
+        // Hold admission through the attempt so an already received packet cannot leave a stale slot.
+        ChannelPromise write = target.newPromise(); inFlightWrite = write;
+        write.addListener(completion -> completed(message, completion.isSuccess()));
+        target.writeAndFlush(message.endpoint == null ? message.buffer : new DatagramPacket(message.buffer, message.endpoint), write);
+        // Promise listeners may be deferred; a completed promise has already relinquished Netty ownership.
+        if (write.isDone()) completed(message, write.isSuccess());
+        if (!write.isDone() && abortBackpressure(tcp.isWritable(), udpReady)) { close(); tcp.close(); }
+    }
+
+    private void retireCompletedWrite() {
+        if (inFlightWrite != null && inFlightWrite.isDone()) completed(inFlight, inFlightWrite.isSuccess());
+    }
+
+    private synchronized void completed(Message message, boolean sent) {
+        // Transfer/admission may retire a completed promise before its deferred listener runs.
+        if (inFlight != message) return;
+        int bytes = message.bytes; inFlight = null; inFlightWrite = null;
+        metrics.transportBuffers.decrementAndGet(); metrics.transportBytes.addAndGet(-bytes); metrics.retainedBytes.addAndGet(-bytes);
+        if (sent) { sentMessages++; metrics.sentMessages.incrementAndGet(); if (message.kind != Kind.ORDERED) sentSnapshots++; if (message.terminal) terminalSent = true; }
+        if (!sent || message.terminal) {
+            if (!sent && message.endpoint != null) udpMetrics.ioErrors.incrementAndGet(); close(); tcp.close();
+        } else scheduleFlush(); // At most one task; never recursively drain under the ownership monitor.
+    }
+
+    private void discard(Message message, boolean superseded) {
+        if (message == null) return;
+        int bytes = message.bytes; queuedBytes -= bytes;
+        metrics.queuedBuffers.decrementAndGet(); metrics.queuedBytes.addAndGet(-bytes); metrics.retainedBytes.addAndGet(-bytes);
+        message.buffer.release();
+        if (superseded) {
+            coalescedSnapshots++; metrics.coalescedSnapshots.incrementAndGet();
+            if (message.buffer.refCnt() == 0) { supersededReleases++; metrics.supersededReleases.incrementAndGet(); }
+            else metrics.supersededReleaseErrors.incrementAndGet();
+        } else {
+            cancelledMessages++; metrics.cancelledMessages.incrementAndGet();
+            if (message.buffer.refCnt() == 0) { cancelledReleases++; metrics.cancelledReleases.incrementAndGet(); }
+            else metrics.cancelledReleaseErrors.incrementAndGet();
+        }
+    }
+
+    @Override public synchronized void close() {
+        if (closed) return; closed = true;
+        while (!ordered.isEmpty()) discard(ordered.removeFirst(), false);
+        discard(full, false); discard(delta, false); full = null; delta = null;
+        // An already detached write remains Netty-owned; only its real callback retires it.
+    }
+
+    synchronized ObjectNode view() {
+        int transportBytes = inFlight == null ? 0 : inFlight.bytes;
+        return Json.MAPPER.createObjectNode().put("closed", closed).put("pending_control", controlCount()).put("pending_ordered", orderedCount())
+                .put("pending_full", kindCount(Kind.FULL)).put("pending_delta", kindCount(Kind.DELTA))
+                .put("queued_full", full == null ? 0 : 1).put("queued_delta", delta == null ? 0 : 1)
+                .put("queued_buffers", ordered.size() + (full == null ? 0 : 1) + (delta == null ? 0 : 1)).put("queued_bytes", queuedBytes)
+                .put("transport_pending_buffers", inFlight == null ? 0 : 1).put("transport_pending_bytes", transportBytes)
+                .put("retained_bytes", queuedBytes + transportBytes).put("retained_bytes_high_water", retainedBytesHighWater)
+                .put("control_high_water", pendingControlHighWater).put("full_high_water", pendingFullHighWater).put("delta_high_water", pendingDeltaHighWater)
+                .put("generated_snapshots", generatedSnapshots).put("coalesced_snapshots", coalescedSnapshots).put("superseded_releases", supersededReleases)
+                .put("cancelled_messages", cancelledMessages).put("sent_messages", sentMessages).put("sent_snapshots", sentSnapshots)
+                .put("cancelled_releases", cancelledReleases).put("terminal", terminalQueued).put("terminal_code", terminalCode).put("terminal_report_sent", terminalSent)
+                .put("readiness_checks", readinessChecks).put("readiness_deferrals", readinessDeferrals)
+                .put("suppressed_inflight_snapshots", suppressedSnapshots).put("flush_task_pending", flushScheduled.get());
+    }
+}
diff --git a/src/main/java/arena/ScenarioRunner.java b/src/main/java/arena/ScenarioRunner.java
index e7c4f04..16a1e08 100644
--- a/src/main/java/arena/ScenarioRunner.java
+++ b/src/main/java/arena/ScenarioRunner.java
@@ -155,6 +155,11 @@ final class ScenarioRunner {
         if (cleanup.path("pending_input_high_water").asInt() > Room.INPUT_LIMIT) failures.add("input bound");
         if (cleanup.path("mailbox_high_water").asInt() > ArenaServer.MAILBOX_LIMIT) failures.add("mailbox bound");
         if (cleanup.path("outbound_high_water").asInt() > ArenaServer.OUTBOUND_LIMIT) failures.add("outbound bound");
+        JsonNode outbound = cleanup.path("outbound");
+        for (String field : List.of("queued_buffers", "queued_bytes", "transport_buffers", "transport_bytes", "retained_bytes", "superseded_release_errors", "cancelled_release_errors"))
+            if (outbound.path(field).asLong(-1) != 0) failures.add("outbound " + field);
+        if (outbound.path("control_high_water").asInt() > ArenaServer.OUTBOUND_LIMIT || outbound.path("full_high_water").asInt() > 1
+                || outbound.path("delta_high_water").asInt() > 1 || outbound.path("transport_per_peer_high_water").asInt() > 1) failures.add("outbound ownership bound");
         if (cleanup.path("replay_bytes").asInt(-1) != 0) failures.add("replay storage cleanup");
         if (cleanup.path("retained_snapshots").asInt(-1) != 0) failures.add("snapshot retention cleanup");
         if (cleanup.path("snapshot_retention_high_water").asInt() > SnapshotStream.RETENTION) failures.add("snapshot retention bound");
diff --git a/src/test/java/arena/BackpressureScenario.java b/src/test/java/arena/BackpressureScenario.java
new file mode 100644
index 0000000..6005fcf
--- /dev/null
+++ b/src/test/java/arena/BackpressureScenario.java
@@ -0,0 +1,281 @@
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
+import java.net.InetSocketAddress;
+import java.nio.ByteBuffer;
+import java.nio.channels.DatagramChannel;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.util.ArrayList;
+import java.util.LinkedHashMap;
+import java.util.IdentityHashMap;
+import java.util.List;
+import java.util.Map;
+import java.util.concurrent.TimeUnit;
+import java.util.concurrent.atomic.AtomicInteger;
+
+/** Test runtime only: one fixed pass and a nonblocking real-channel writability signal. */
+final class BackpressureScenario {
+    static final String SHA256 = "b5350ef5bfb9fa93cbde1fe0fd30079e6115fe26988514be59d63d9dee6bc6f3";
+    private BackpressureScenario() { }
+
+    static ReplayScenario.Observed run(Path path) throws Exception {
+        byte[] bytes = Files.readAllBytes(path);
+        if (!SHA256.equals(UdpScenario.hash(bytes))) throw new IOException("frozen G12 bytes required");
+        ObjectNode fixture = Json.read(bytes), result = Json.MAPPER.createObjectNode().put("thread", "G12")
+                .put("scenario_sha256", SHA256).put("process_id", ProcessHandle.current().pid()).put("mode", "LIVE_G12")
+                .put("readiness_signal", "real alpha TCP ChannelOutboundBuffer user-defined writability false; no buffering handler or blocked event loop")
+                .put("fault_campaigns", 0).put("load_runs", 0);
+        ArrayNode failures = result.putArray("failures"), ticks = result.putArray("ticks");
+        ArrayNode hashes = result.putArray("state_hashes"), records = result.putArray("canonical_records");
+        ArrayNode transitions = result.putArray("readiness_transitions"); result.putObject("captures"); result.putArray("unexpected_alpha_datagrams");
+        Map<String, TcpClient> clients = new LinkedHashMap<>(); Map<String, AckScenario.Replica> replicas = new LinkedHashMap<>();
+        ArenaServer server = new ArenaServer("127.0.0.1", 0, true); Probe probe = null; byte[] artifact = null;
+        try {
+            for (JsonNode player : fixture.withArray("players")) {
+                String role = player.path("client").asText(); TcpClient client = new TcpClient(server.port());
+                clients.put(role, client); replicas.put(role, new AckScenario.Replica()); client.hello();
+                ((ObjectNode) result.path("captures")).putArray(role);
+            }
+            result.set("bootstrap", ReplayFixture.bootstrap(server, fixture, clients));
+            for (var entry : clients.entrySet()) capture(server, entry.getKey(), entry.getValue(), replicas.get(entry.getKey()), result, true);
+            require(((java.util.Deque<?>) ReplayFixture.field(clients.get("alpha"), "received")).isEmpty(), "alpha bootstrap controls not drained");
+            probe = new Probe(server, clients); probe.hold(); transitions.add(probe.sample().put("before_tick", 0));
+            for (int tick = 0; tick < fixture.path("ticks").asInt(); tick++) {
+                for (JsonNode event : fixture.withArray("events")) if (event.path("before_tick").asInt() == tick) {
+                    TcpClient client = clients.get(event.path("client").asText()); ObjectNode input = client.auth("INPUT", client.roomId);
+                    for (String field : List.of("seq", "target_tick", "direction", "owner_epoch")) input.set(field, event.path(field));
+                    input.putNull("tag_target_player_id"); ObjectNode admission = result.putObject("input_admission").put("before_tick", tick);
+                    ObjectNode safe = input.deepCopy(); safe.remove("session_id"); admission.set("request", safe);
+                    admission.set("before", ReplayFixture.snapshot(server)); client.send(input); ObjectNode reply = client.until("INPUT_ACK");
+                    admission.set("reply", reply); admission.set("after", ReplayFixture.snapshot(server));
+                    require(reply.path("status").asText().equals("ACCEPTED") && reply.path("seq").asInt() == 1, "single bravo input admission");
+                }
+                server.advanceTicks(1); ObjectNode state = ReplayFixture.snapshot(server); String record = ReplayFixture.canonicalRecord(server);
+                ticks.add(state); hashes.add(state.path("state_hash")); records.add(record); authority(state, tick);
+                require(state.path("state_hash").asText().equals(ReplayLog.hash(record)), "canonical capture hash");
+                if ((tick + 1) % 2 == 0) for (var entry : clients.entrySet()) if (!entry.getKey().equals("alpha"))
+                    capture(server, entry.getKey(), entry.getValue(), replicas.get(entry.getKey()), result, true);
+                ObjectNode sample = probe.sample().put("after_tick", tick); transitions.add(sample); bounds(sample);
+                observeAlphaSocket(clients.get("alpha"), result.withArray("unexpected_alpha_datagrams"), tick);
+            }
+            result.set("final_state", ReplayFixture.snapshot(server)); result.set("alpha_stream", AckScenario.stream(server, clients.get("alpha")));
+            result.set("runtime_metrics", server.metrics()); artifact = server.replayArtifact(); result.put("artifact_bytes", artifact.length);
+            ObjectNode beforeClose = probe.sample(); result.set("held_transport", beforeClose);
+            result.put("held_snapshot_sends", beforeClose.path("by_client").path("alpha").path("snapshot_sends").asInt());
+            result.put("held_snapshot_receives", result.path("unexpected_alpha_datagrams").size());
+            ObjectNode queue = (ObjectNode) beforeClose.path("alpha_queue");
+            result.put("held_snapshot_generations", queue.path("generated_snapshots").asInt() - 1);
+            require(!beforeClose.path("alpha_ready").asBoolean() && result.path("held_snapshot_sends").asInt() == 0
+                    && result.path("held_snapshot_receives").asInt() == 0 && result.path("held_snapshot_generations").asInt() == 50
+                    && beforeClose.path("ownership_bound_violations").asInt() == 0
+                    && queue.path("coalesced_snapshots").asInt() > 0
+                    && queue.path("superseded_releases").equals(queue.path("coalesced_snapshots")), "actual readiness/coalescing ownership guarantee");
+            for (String role : List.of("bravo", "charlie", "delta")) require(result.path("captures").path(role).size() == 51, "healthy51 cadence");
+            clients.get("alpha").closeTcp(); result.set("post_alpha_close_state", awaitDisconnect(server));
+            ObjectNode afterClose = probe.sample(); result.set("post_alpha_close_transport", afterClose);
+            require(afterClose.path("alpha_queue").path("closed").asBoolean()
+                    && afterClose.path("alpha_queue").path("retained_bytes").asInt(-1) == 0
+                    && afterClose.path("alpha_observed_live_buffers").asInt(-1) == 0, "alpha close did not release queued/transport buffers");
+            server.close(); for (var entry : clients.entrySet()) if (!entry.getKey().equals("alpha")) entry.getValue().expectClosed();
+        } catch (Exception failure) {
+            failures.add(failure.getClass().getName() + ": " + failure.getMessage());
+            java.io.StringWriter trace = new java.io.StringWriter(); failure.printStackTrace(new java.io.PrintWriter(trace)); result.put("execution_error", trace.toString());
+        } finally {
+            server.close(); for (TcpClient client : clients.values()) client.close();
+            if (probe != null) { result.set("transport_trace", probe.trace()); result.set("queue_buffer_trace", probe.queueTrace()); result.set("transport_cleanup", probe.sample()); }
+        }
+        result.put("executed_ticks", ticks.size()); result.set("cleanup", server.cleanup()); ScenarioRunner.assertCleanup(server.cleanup());
+        boolean released = clients.values().stream().allMatch(TcpClient::isClosed)
+                && result.path("transport_cleanup").path("observed_live_buffers").asInt(-1) == 0;
+        result.put("all_resources_released", released).put("passed", failures.isEmpty() && released && ticks.size() == 100);
+        return new ReplayScenario.Observed(result, artifact);
+    }
+
+    private static void capture(ArenaServer server, String role, TcpClient client, AckScenario.Replica replica, ObjectNode result, boolean acknowledge) throws Exception {
+        ObjectNode wire = client.until("SNAPSHOT"), state = ReplayFixture.snapshot(server); String record = ReplayFixture.canonicalRecord(server);
+        long previous = replica.sequence; String application = replica.apply(wire);
+        require(wire.path("snapshot_seq").asLong() == previous + 1 && application.equals("APPLIED")
+                && wire.path("tick").equals(state.path("tick")) && wire.path("state_hash").asText().equals(ReplayLog.hash(record))
+                && replica.state.equals(SnapshotScenario.projection(state)) && Json.bytes(wire).length <= 1_200, "actual snapshot cadence/hash/projection " + role);
+        ObjectNode cell = ((ArrayNode) result.path("captures").path(role)).addObject().put("seq", replica.sequence)
+                .put("capture_tick", state.path("tick").asInt()).put("applied", true).put("ack_sent", acknowledge);
+        cell.set("wire", wire); cell.set("visible_state", replica.state.deepCopy());
+        if (acknowledge) {
+            ObjectNode ack = client.auth("SNAPSHOT_ACK", client.roomId).put("snapshot_seq", replica.sequence).put("state_hash", wire.path("state_hash").asText());
+            int received = ReplayFixture.udpReceived(server); client.send(ack); ReplayFixture.udpBarrier(server, received + 1);
+            long watermark = AckScenario.stream(server, client).path("acknowledged_seq").asLong(); cell.put("server_ack_watermark", watermark);
+            require(watermark == replica.sequence && state.equals(ReplayFixture.snapshot(server)), "ACK admission changed state or watermark");
+        }
+    }
+
+    private static void authority(ObjectNode state, int tick) throws IOException {
+        require(state.path("tick").asInt() == tick && state.path("status").asText().equals("RUNNING"), "owner tick/status progress");
+        for (int slot = 0; slot < 4; slot++) {
+            JsonNode player = ReplayScenario.player(state, "player-0" + slot);
+            require(player.path("x").asInt() == (slot == 1 ? Math.min(100_000, 90_000 + 400 * (tick + 1)) : Room.SPAWNS[slot][0])
+                    && player.path("y").asInt() == Room.SPAWNS[slot][1] && player.path("score").asInt() == 0
+                    && player.path("last_accepted_seq").asInt() == (slot == 1 ? 1 : 0) && player.path("pending_inputs").asInt() == 0
+                    && player.path("direction").asText().equals(slot == 1 ? "EAST" : "STOP")
+                    && player.path("connectivity").asText().equals("CONNECTED"), "fixed authority " + slot + "/" + tick);
+        }
+    }
+    private static ObjectNode awaitDisconnect(ArenaServer server) throws Exception {
+        long deadline = System.nanoTime() + TimeUnit.SECONDS.toNanos(5);
+        while (true) {
+            ObjectNode state = ReplayFixture.snapshot(server);
+            if (ReplayScenario.player(state, "player-00").path("connectivity").asText().equals("DISCONNECTED")) return state;
+            if (System.nanoTime() >= deadline) throw new IOException("alpha owner disconnect barrier ceiling");
+        }
+    }
+    private static void require(boolean condition, String message) throws IOException { if (!condition) throw new IOException(message); }
+    private static void observeAlphaSocket(TcpClient alpha, ArrayNode unexpected, int tick) throws Exception {
+        ByteBuffer bytes = ByteBuffer.allocate(1_201); DatagramChannel udp = (DatagramChannel) ReplayFixture.field(alpha, "udp");
+        var sender = udp.receive(bytes);
+        if (sender != null) { bytes.flip(); unexpected.addObject().put("after_tick", tick).set("wire", Json.read(bytes)); }
+    }
+    private static void bounds(ObjectNode sample) throws IOException {
+        JsonNode queue = sample.path("alpha_queue");
+        require(!sample.path("alpha_ready").asBoolean() && queue.path("pending_full").asInt() <= 1 && queue.path("pending_delta").asInt() <= 1
+                && queue.path("pending_control").asInt() <= 64 && queue.path("transport_pending_buffers").asInt() <= 1,
+                "queued plus transport-owned snapshot/control bound");
+    }
+
+    /** Observes existing writes only. It neither retains a buffer nor queues/suppresses/rewrites a message. */
+    static final class Probe {
+        private final ArenaServer server; private final Object alphaPeer; private final Channel alpha, udp;
+        private final NioEventLoopGroup io; private final Map<InetSocketAddress, String> roles = new LinkedHashMap<>();
+        private final Map<String, OutboundQueue> queues = new LinkedHashMap<>();
+        private final List<QueuedBuffer> queuedBuffers = new ArrayList<>(); private final Map<ByteBuf, Boolean> seen = new IdentityHashMap<>();
+        private final List<ObservedWrite> writes = new ArrayList<>(); private int liveWrites, liveBytes, pendingHighWater, bytesHighWater;
+        private int tcpFlushes, udpFlushes, ownershipBoundViolations;
+        private static final class QueuedBuffer {
+            final ByteBuf buffer; final String kind; final long sequence; final int bytes, initialRefs; boolean releasedWhileActive;
+            QueuedBuffer(ByteBuf buffer, ObjectNode message) {
+                this.buffer = buffer; kind = message.path("kind").asText(); sequence = message.path("snapshot_seq").asLong();
+                bytes = buffer.readableBytes(); initialRefs = buffer.refCnt();
+            }
+        }
+        private static final class ObservedWrite {
+            final ByteBuf buffer; final String role, type; final long sequence; final int bytes, initialRefs; final boolean ready;
+            boolean completed, sent; int completionRefs;
+            ObjectNode boundsBeforeTransfer, boundsBeforeCompletion;
+            ObservedWrite(ByteBuf buffer, String role, ObjectNode message, boolean ready) {
+                this.buffer = buffer; this.role = role; type = message.path("type").asText(); sequence = message.path("snapshot_seq").asLong();
+                bytes = buffer.readableBytes(); initialRefs = buffer.refCnt(); this.ready = ready;
+            }
+        }
+        Probe(ArenaServer server, Map<String, TcpClient> clients) throws Exception {
+            this.server = server; io = (NioEventLoopGroup) ReplayFixture.field(server, "ioLoop"); udp = (Channel) ReplayFixture.field(server, "udpListener");
+            alphaPeer = ReplayFixture.owned(server, () -> {
+                Object selected = null;
+                for (var entry : ((Map<?, ?>) ReplayFixture.field(server, "sessions")).entrySet())
+                    for (var client : clients.entrySet()) if (ReplayFixture.field(entry.getValue(), "id").equals(client.getValue().sessionId)) {
+                        queues.put(client.getKey(), (OutboundQueue) ReplayFixture.field(entry.getKey(), "outbound"));
+                        if (client.getKey().equals("alpha")) selected = entry.getKey();
+                    }
+                if (selected == null || queues.size() != clients.size()) throw new IOException("actual peer queues unavailable"); return selected;
+            });
+            alpha = (Channel) ReplayFixture.field(alphaPeer, "channel");
+            for (var entry : clients.entrySet()) roles.put(entry.getValue().localUdpEndpoint(), entry.getKey());
+            io.submit(() -> { alpha.pipeline().addFirst("g12-write-observer", observer(false)); udp.pipeline().addFirst("g12-write-observer", observer(true)); }).get(5, TimeUnit.SECONDS);
+        }
+        private ChannelOutboundHandlerAdapter observer(boolean datagram) {
+            return new ChannelOutboundHandlerAdapter() {
+                @Override public void write(ChannelHandlerContext context, Object message, ChannelPromise promise) throws Exception {
+                    ByteBuf buffer = datagram ? ((DatagramPacket) message).content() : (ByteBuf) message;
+                    String role = datagram ? roles.get(((DatagramPacket) message).recipient()) : "alpha";
+                    if (role == null) throw new IOException("unmapped observed endpoint");
+                    int offset = datagram ? 0 : 4; byte[] bytes = new byte[buffer.readableBytes() - offset]; buffer.getBytes(buffer.readerIndex() + offset, bytes);
+                    ObjectNode decoded = Json.read(bytes); ObservedWrite observed = new ObservedWrite(buffer, role, decoded, alpha.isWritable());
+                    observed.boundsBeforeTransfer = queues.get(role).view();
+                    if (!withinBounds(observed.boundsBeforeTransfer)) ownershipBoundViolations++;
+                    if (writes.size() >= 512) throw new IOException("fixed observation bound"); writes.add(observed);
+                    liveWrites++; liveBytes += observed.bytes; pendingHighWater = Math.max(pendingHighWater, liveWrites); bytesHighWater = Math.max(bytesHighWater, liveBytes);
+                    promise.addListener(completion -> {
+                        observed.completed = true; observed.sent = completion.isSuccess(); observed.completionRefs = buffer.refCnt();
+                        observed.boundsBeforeCompletion = queues.get(role).view();
+                        if (!withinBounds(observed.boundsBeforeCompletion)) ownershipBoundViolations++;
+                        liveWrites--; liveBytes -= observed.bytes;
+                    });
+                    context.write(message, promise);
+                }
+                @Override public void flush(ChannelHandlerContext context) { if (datagram) udpFlushes++; else tcpFlushes++; context.flush(); }
+            };
+        }
+        void hold() throws Exception { io.submit(() -> alpha.unsafe().outboundBuffer().setUserDefinedWritability(1, false)).get(5, TimeUnit.SECONDS); }
+        ObjectNode sample() throws Exception {
+            if (io.isTerminated()) return view();
+            return io.submit(this::view).get(5, TimeUnit.SECONDS);
+        }
+        private ObjectNode view() throws Exception {
+            OutboundQueue queue = queues.get("alpha");
+            synchronized (queue) {
+                for (String field : List.of("full", "delta")) {
+                    Object message = ReplayFixture.field(queue, field); if (message == null) continue;
+                    ByteBuf buffer = (ByteBuf) ReplayFixture.field(message, "buffer");
+                    if (seen.put(buffer, true) == null) {
+                        byte[] bytes = new byte[buffer.readableBytes()]; buffer.getBytes(buffer.readerIndex(), bytes); queuedBuffers.add(new QueuedBuffer(buffer, Json.read(bytes)));
+                    }
+                }
+            }
+            int refs = 0, bytes = 0; ObjectNode counts = Json.MAPPER.createObjectNode();
+            for (String role : roles.values()) counts.putObject(role).put("snapshot_writes", 0).put("snapshot_sends", 0).put("pending_writes", 0).put("sent_while_alpha_not_ready", 0);
+            for (ObservedWrite write : writes) {
+                if (write.buffer.refCnt() > 0) { refs++; bytes += write.bytes; }
+                ObjectNode cell = (ObjectNode) counts.path(write.role);
+                if (!write.completed) increment(cell, "pending_writes");
+                if (write.type.equals("SNAPSHOT")) { increment(cell, "snapshot_writes"); if (write.sent) increment(cell, "snapshot_sends"); }
+                if (write.sent && !write.ready) increment(cell, "sent_while_alpha_not_ready");
+            }
+            int alphaRefs = 0;
+            for (QueuedBuffer buffer : queuedBuffers) {
+                if (buffer.buffer.refCnt() > 0) { refs++; alphaRefs++; bytes += buffer.bytes; }
+                else if (alpha.isActive()) buffer.releasedWhileActive = true;
+            }
+            ObjectNode result = Json.MAPPER.createObjectNode().put("alpha_channel_active", alpha.isActive()).put("alpha_ready", alpha.isWritable())
+                    .put("per_peer_predequeue_queue_available", true).put("coalescing_available", true)
+                    .put("server_pending_writes", ((AtomicInteger) ReplayFixture.field(server, "pendingWrites")).get())
+                    .put("actual_tcp_pending_bytes", alpha.unsafe().outboundBuffer() == null ? 0 : alpha.unsafe().outboundBuffer().totalPendingWriteBytes())
+                    .put("actual_udp_pending_bytes", udp.unsafe().outboundBuffer() == null ? 0 : udp.unsafe().outboundBuffer().totalPendingWriteBytes())
+                    .put("transport_pending_writes", liveWrites).put("transport_pending_bytes", liveBytes)
+                    .put("transport_pending_high_water", pendingHighWater).put("transport_bytes_high_water", bytesHighWater)
+                    .put("tcp_flush_calls", tcpFlushes).put("udp_flush_calls", udpFlushes)
+                    .put("ownership_bound_violations", ownershipBoundViolations)
+                    .put("observed_live_buffers", refs).put("observed_retained_bytes", bytes).put("alpha_observed_live_buffers", alphaRefs);
+            result.set("alpha_queue", queue.view()); result.set("by_client", counts); return result;
+        }
+        ArrayNode trace() {
+            ArrayNode result = Json.MAPPER.createArrayNode();
+            for (ObservedWrite write : writes) {
+                ObjectNode cell = result.addObject().put("client", write.role).put("type", write.type).put("snapshot_seq", write.sequence)
+                    .put("bytes", write.bytes).put("alpha_ready_before_transfer", write.ready).put("initial_ref_count", write.initialRefs)
+                    .put("completed", write.completed).put("sent", write.sent).put("ref_count_at_completion", write.completionRefs).put("final_ref_count", write.buffer.refCnt());
+                cell.set("queue_at_transfer", write.boundsBeforeTransfer); cell.set("queue_before_completion_callback", write.boundsBeforeCompletion);
+            }
+            return result;
+        }
+        ArrayNode queueTrace() {
+            ArrayNode result = Json.MAPPER.createArrayNode();
+            for (QueuedBuffer buffer : queuedBuffers) result.addObject().put("snapshot_seq", buffer.sequence).put("kind", buffer.kind).put("bytes", buffer.bytes)
+                    .put("initial_ref_count", buffer.initialRefs).put("released_while_alpha_active", buffer.releasedWhileActive).put("final_ref_count", buffer.buffer.refCnt());
+            return result;
+        }
+        private static void increment(ObjectNode value, String name) { value.put(name, value.path(name).asInt() + 1); }
+        private static boolean withinBounds(ObjectNode queue) {
+            return queue.path("pending_full").asInt() <= 1 && queue.path("pending_delta").asInt() <= 1
+                    && queue.path("pending_control").asInt() <= 64 && queue.path("pending_ordered").asInt() <= 64
+                    && queue.path("transport_pending_buffers").asInt() <= 1;
+        }
+    }
+}
diff --git a/src/test/java/arena/OutboundQueueTest.java b/src/test/java/arena/OutboundQueueTest.java
new file mode 100644
index 0000000..fb58128
--- /dev/null
+++ b/src/test/java/arena/OutboundQueueTest.java
@@ -0,0 +1,113 @@
+package arena;
+
+import static org.junit.jupiter.api.Assertions.*;
+import com.fasterxml.jackson.databind.node.ArrayNode;
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import io.netty.buffer.ByteBuf;
+import io.netty.channel.embedded.EmbeddedChannel;
+import io.netty.channel.socket.DatagramPacket;
+import java.net.InetSocketAddress;
+import java.util.ArrayList;
+import java.util.List;
+import org.junit.jupiter.api.Test;
+
+/** Exactly the two frozen pure queue cases; no live Room campaign or simulation tick. */
+final class OutboundQueueTest {
+    @Test void control63RemainsOpenAndAttempt64RetainsTerminalWithinBound() throws Exception {
+        try (World world = new World()) {
+            world.ready(false); List<ObjectNode> requests = new ArrayList<>();
+            for (int i = 0; i < 63; i++) { ObjectNode message = CompleteFrame.error("MESSAGE_INVALID", "control-" + i); requests.add(message); world.queue.send(message, false); }
+            ObjectNode at63 = world.queue.view(); assertTrue(world.tcp.isActive()); assertEquals(63, at63.path("pending_control").asInt());
+            assertEquals(63, world.metrics.allocatedBuffers.get()); assertNull(world.tcp.readOutbound());
+            List<ByteBuf> buffers = pending(world.queue);
+            for (int i = 0; i < 63; i++) assertEquals(requests.get(i), decodeTcp(buffers.get(i)));
+            world.queue.send(CompleteFrame.error("MESSAGE_INVALID", "attempt64"), false);
+            ObjectNode at64 = world.queue.view(); assertEquals(64, at64.path("control_high_water").asInt()); assertEquals(64, world.metrics.allocatedBuffers.get());
+            assertEquals("CONTROL_BACKPRESSURE", at64.path("terminal_code").asText()); assertTrue(world.queue.terminal());
+            assertFalse(at64.path("terminal_report_sent").asBoolean()); assertEquals(64, at64.path("cancelled_messages").asInt());
+            assertEquals(64, at64.path("cancelled_releases").asInt()); assertEquals(0, world.metrics.cancelledReleaseErrors.get());
+            assertEquals(0, at64.path("transport_pending_buffers").asInt()); ArrayNode delivered = world.drainTcp(); assertEquals(0, delivered.size());
+            assertFalse(world.tcp.isOpen()); assertTrue(buffers.stream().allMatch(buffer -> buffer.refCnt() == 0)); world.assertEmpty();
+            ObjectNode result = Json.MAPPER.createObjectNode().put("probe", "G12 control-threshold").put("actual_tick_calls", 0)
+                    .put("allocated_buffers", world.metrics.allocatedBuffers.get()).put("delivered_controls", delivered.size()).put("all_buffer_refs_zero", true);
+            result.set("at63", at63); result.set("at64", at64); result.set("delivered", delivered); result.set("cleanup", world.queue.view());
+            System.out.println(result);
+        }
+    }
+
+    @Test void mixedControlsSurviveExactlyThreeSupersededSnapshotBuffers() throws Exception {
+        try (World world = new World()) {
+            Room room = new Room("queue-room"); room.join("alpha"); room.join("bravo"); room.ready("alpha"); room.ready("bravo");
+            SnapshotStream stream = new SnapshotStream();
+            try {
+                ObjectNode initial = stream.next(room, true); world.queue.realtime(initial, world.endpoint);
+                DatagramPacket sent = world.udp.readOutbound(); assertNotNull(sent);
+                // Compare the wire bytes: decoded small integers need not retain the LongNode type.
+                try { assertArrayEquals(Json.bytes(initial), udpBytes(sent.content())); }
+                finally { sent.release(); }
+                assertEquals(0, sent.refCnt());
+                stream.acknowledge(1, initial.path("state_hash").asText(), false); world.ready(false);
+                ObjectNode controlA = Json.message("ROOM_CREATED").put("room_id", room.id).put("status", "LOBBY");
+                ObjectNode controlB = CompleteFrame.error("FRAME_SIZE_INVALID", "terminal-B");
+                world.queue.send(controlA, false); world.queue.send(controlB, true); List<ByteBuf> controls = pending(world.queue);
+                ObjectNode full2 = stream.next(room, true); world.queue.realtime(full2, world.endpoint); ByteBuf oldFull = buffer(world.queue, "full");
+                ObjectNode delta3 = stream.next(room, false); world.queue.realtime(delta3, world.endpoint); ByteBuf oldDelta = buffer(world.queue, "delta");
+                ObjectNode delta4 = stream.next(room, false); world.queue.realtime(delta4, world.endpoint); ByteBuf newerDelta = buffer(world.queue, "delta");
+                assertEquals(0, oldDelta.refCnt()); assertEquals(1, oldFull.refCnt());
+                ObjectNode full5 = stream.next(room, true); world.queue.realtime(full5, world.endpoint); ByteBuf newestFull = buffer(world.queue, "full");
+                ObjectNode beforeDrain = world.queue.view();
+                assertEquals(2, beforeDrain.path("pending_control").asInt()); assertEquals(1, beforeDrain.path("pending_full").asInt());
+                assertEquals(0, beforeDrain.path("pending_delta").asInt()); assertEquals(3, beforeDrain.path("coalesced_snapshots").asInt());
+                assertEquals(3, beforeDrain.path("superseded_releases").asInt()); assertEquals(0, world.metrics.supersededReleaseErrors.get());
+                assertEquals(0, oldFull.refCnt()); assertEquals(0, newerDelta.refCnt()); assertEquals(1, newestFull.refCnt());
+                assertEquals(controlA, decodeTcp(controls.get(0))); assertEquals(controlB, decodeTcp(controls.get(1)));
+                assertEquals(1, delta3.path("base_snapshot_seq").asInt()); assertEquals(1, delta4.path("base_snapshot_seq").asInt());
+                assertEquals("FULL", full2.path("kind").asText()); assertEquals(5, full5.path("snapshot_seq").asInt());
+                world.ready(true); ArrayNode delivered = world.drainTcp(); assertEquals(2, delivered.size());
+                assertEquals(controlA, delivered.get(0)); assertEquals(controlB, delivered.get(1));
+                assertFalse(world.tcp.isOpen()); assertNull(world.udp.readOutbound()); assertEquals(0, newestFull.refCnt());
+                assertTrue(controls.stream().allMatch(buffer -> buffer.refCnt() == 0)); assertEquals(0, room.executedTicks()); world.assertEmpty();
+                ObjectNode result = Json.MAPPER.createObjectNode().put("probe", "G12 mixed-snapshot-coalescing").put("actual_tick_calls", room.executedTicks())
+                        .put("acknowledged_base", 1).put("discarded_snapshot_count", 3).put("replacement_operations", 2)
+                        .put("all_superseded_and_cancelled_refs_zero", true).put("unsent_full5_reported_delivered", false);
+                ArrayNode snapshots = result.putArray("snapshot_sequence"); for (ObjectNode snapshot : List.of(full2, delta3, delta4, full5)) snapshots.add(snapshot);
+                result.set("before_drain", beforeDrain); result.set("delivered_controls", delivered); result.set("cleanup", world.queue.view()); System.out.println(result);
+            } finally { stream.close(); room.close(); }
+        }
+    }
+
+    private static final class World implements AutoCloseable {
+        final EmbeddedChannel tcp = new EmbeddedChannel(), udp = new EmbeddedChannel();
+        final OutboundQueue.Metrics metrics = new OutboundQueue.Metrics();
+        final OutboundQueue queue = new OutboundQueue(tcp, () -> udp, metrics, new UdpData.Metrics());
+        final InetSocketAddress endpoint = new InetSocketAddress("127.0.0.1", 9);
+        void ready(boolean ready) { tcp.unsafe().outboundBuffer().setUserDefinedWritability(1, ready); tcp.runPendingTasks(); udp.runPendingTasks(); }
+        ArrayNode drainTcp() throws Exception {
+            tcp.runPendingTasks(); ArrayNode result = Json.MAPPER.createArrayNode(); ByteBuf buffer;
+            while ((buffer = tcp.readOutbound()) != null) { try { result.add(decodeTcp(buffer)); } finally { buffer.release(); } }
+            return result;
+        }
+        void assertEmpty() {
+            assertEquals(0, metrics.queuedBuffers.get()); assertEquals(0, metrics.queuedBytes.get());
+            assertEquals(0, metrics.transportBuffers.get()); assertEquals(0, metrics.transportBytes.get()); assertEquals(0, metrics.retainedBytes.get());
+            assertEquals(0, queue.view().path("pending_control").asInt()); assertEquals(0, queue.view().path("pending_full").asInt());
+            assertEquals(0, queue.view().path("pending_delta").asInt()); assertFalse(queue.view().path("flush_task_pending").asBoolean());
+        }
+        @Override public void close() { queue.close(); tcp.finishAndReleaseAll(); udp.finishAndReleaseAll(); assertEmpty(); }
+    }
+    private static ByteBuf buffer(OutboundQueue queue, String field) throws Exception { return (ByteBuf) ReplayFixture.field(ReplayFixture.field(queue, field), "buffer"); }
+    private static List<ByteBuf> pending(OutboundQueue queue) throws Exception {
+        List<ByteBuf> result = new ArrayList<>();
+        synchronized (queue) {
+            for (Object message : (Iterable<?>) ReplayFixture.field(queue, "ordered")) result.add((ByteBuf) ReplayFixture.field(message, "buffer"));
+            for (String field : List.of("full", "delta")) if (ReplayFixture.field(queue, field) != null) result.add(buffer(queue, field));
+        }
+        return result;
+    }
+    private static ObjectNode decodeTcp(ByteBuf buffer) throws Exception {
+        int offset = buffer.readerIndex(), length = buffer.getInt(offset); byte[] bytes = new byte[length]; buffer.getBytes(offset + 4, bytes); return Json.read(bytes);
+    }
+    private static byte[] udpBytes(ByteBuf buffer) {
+        byte[] bytes = new byte[buffer.readableBytes()]; buffer.getBytes(buffer.readerIndex(), bytes); return bytes;
+    }
+}
diff --git a/src/test/java/arena/ReplayFormatTest.java b/src/test/java/arena/ReplayFormatTest.java
index 186224b..d820c5d 100644
--- a/src/test/java/arena/ReplayFormatTest.java
+++ b/src/test/java/arena/ReplayFormatTest.java
@@ -51,7 +51,8 @@ final class ReplayFormatTest {
             assertNull(jar.getJarEntry("G09.json"));
             assertNull(jar.getJarEntry("G10.json"));
             assertNull(jar.getJarEntry("G11.json"));
-            for (String name : List.of("ReplayFixture", "ReplayScenario", "ReplayVerifier", "ReplayTool", "G07BaselineTest", "SnapshotScenario", "G08BaselineTest", "UdpScenario", "UdpFaultProxy", "UdpBoundaryTest", "G09BaselineTest", "AckScenario", "G10BaselineTest", "ReconnectScenario", "G11BaselineTest")) {
+            assertNull(jar.getJarEntry("G12.json"));
+            for (String name : List.of("ReplayFixture", "ReplayScenario", "ReplayVerifier", "ReplayTool", "G07BaselineTest", "SnapshotScenario", "G08BaselineTest", "UdpScenario", "UdpFaultProxy", "UdpBoundaryTest", "G09BaselineTest", "AckScenario", "G10BaselineTest", "ReconnectScenario", "G11BaselineTest", "BackpressureScenario", "G12BaselineTest", "OutboundQueueTest")) {
                 assertNull(jar.getJarEntry("arena/" + name + ".class"));
                 assertThrows(ClassNotFoundException.class, () -> Class.forName("arena." + name, false, production));
             }
diff --git a/src/test/java/arena/ReplayTool.java b/src/test/java/arena/ReplayTool.java
index 9e94f67..9a9f4d0 100644
--- a/src/test/java/arena/ReplayTool.java
+++ b/src/test/java/arena/ReplayTool.java
@@ -20,11 +20,12 @@ public final class ReplayTool {
             if (Files.size(input) > 65_536) throw new IllegalArgumentException("scenario byte bound");
             ObjectNode scenario = Json.read(Files.readAllBytes(input));
             String thread = scenario.path("thread").asText();
-            if (thread.equals("G07") || thread.equals("G08") || thread.equals("G09") || thread.equals("G10")) {
+            if (thread.equals("G07") || thread.equals("G08") || thread.equals("G09") || thread.equals("G10") || thread.equals("G12")) {
                 boolean variant = args.length == 5 && args[3].equals("--variant") && args[4].equals("rejected-removed");
                 if (args.length != 3 && !(thread.equals("G07") && variant)) throw new IllegalArgumentException("unknown scenario variant");
                 ReplayScenario.Observed observed = thread.equals("G07") ? ReplayScenario.run(input, variant)
-                        : thread.equals("G08") ? SnapshotScenario.run(input) : thread.equals("G09") ? UdpScenario.run(input) : AckScenario.run(input);
+                        : thread.equals("G08") ? SnapshotScenario.run(input) : thread.equals("G09") ? UdpScenario.run(input)
+                        : thread.equals("G10") ? AckScenario.run(input) : BackpressureScenario.run(input);
                 result = observed.result();
                 Path artifact = output.resolveSibling(output.getFileName().toString().replaceFirst("\\.json$", "") + ".replay.jsonl");
                 if (observed.replay() != null) {
diff --git a/src/test/resources/G12.json b/src/test/resources/G12.json
new file mode 100644
index 0000000..05fad33
--- /dev/null
+++ b/src/test/resources/G12.json
@@ -0,0 +1,126 @@
+{
+  "scenario_id": "G12-bounded-slow-consumer",
+  "contract_version": 1,
+  "thread": "G12",
+  "seed": 7050,
+  "clock": {
+    "kind": "manual",
+    "tick_duration_ms": 50
+  },
+  "ticks": 100,
+  "room_id": "room-fixture",
+  "players": [
+    {
+      "client": "alpha",
+      "player_id": "player-00",
+      "slot": 0,
+      "spawn": [
+        10000,
+        10000
+      ]
+    },
+    {
+      "client": "bravo",
+      "player_id": "player-01",
+      "slot": 1,
+      "spawn": [
+        90000,
+        90000
+      ]
+    },
+    {
+      "client": "charlie",
+      "player_id": "player-02",
+      "slot": 2,
+      "spawn": [
+        10000,
+        90000
+      ]
+    },
+    {
+      "client": "delta",
+      "player_id": "player-03",
+      "slot": 3,
+      "spawn": [
+        90000,
+        10000
+      ]
+    }
+  ],
+  "initialization": "normal TCP HELLO/JOIN while all four UDP-unbound; bind all four; drain bootstrap controls; receive/apply/ACK initial FULL1 before holding alpha",
+  "events": [
+    {
+      "before_tick": 0,
+      "kind": "INPUT",
+      "client": "bravo",
+      "seq": 1,
+      "target_tick": 0,
+      "direction": "EAST",
+      "tag_target_role": null,
+      "owner_epoch": 0
+    }
+  ],
+  "slow_consumer": {
+    "client": "alpha",
+    "begin_before_tick": 0,
+    "end_after_tick": 99,
+    "hold": "test-only readiness gate before actual production outbound dequeue for both TCP and UDP; return not-ready, never block owner/event loop",
+    "acknowledged_snapshot": 1,
+    "end_action": "close alpha TCP after final authoritative capture; then shut down Room/server and release all remaining resources"
+  },
+  "healthy_clients": [
+    "bravo",
+    "charlie",
+    "delta"
+  ],
+  "snapshots": {
+    "initial_seq": 1,
+    "initial_tick": -1,
+    "every_executed_ticks": 2,
+    "expected_healthy_count_each": 51,
+    "healthy_ack": "apply and ACK each received snapshot",
+    "slow_during_hold_received": 0,
+    "retention": 32,
+    "pending_full_limit": 1,
+    "pending_delta_limit": 1
+  },
+  "control_bound": {
+    "max_pending": 64,
+    "open_at_pending": 63,
+    "terminal_enqueue_attempt": 64,
+    "terminal_code": "CONTROL_BACKPRESSURE",
+    "no_65th_buffer": true
+  },
+  "supplemental_cases": [
+    {
+      "name": "control-threshold",
+      "steps": [
+        "hold actual outbound dequeue",
+        "enqueue 63 ordinary controls; connection remains open",
+        "attempt 64th; terminal CONTROL_BACKPRESSURE within capacity64",
+        "drain/close and release every buffer"
+      ]
+    },
+    {
+      "name": "mixed-snapshot-coalescing",
+      "steps": [
+        "retain ordinary control A and terminal control B in original order",
+        "enqueue FULL2 then DELTA3 based on a valid acknowledged snapshot",
+        "enqueue newer DELTA4; replaces DELTA3",
+        "enqueue newer FULL5; replaces pending FULL2 and DELTA4",
+        "assert controls A/B unchanged, pending full1/delta0, superseded snapshot buffers released",
+        "drain/close and release remaining buffers"
+      ],
+      "discarded_snapshot_count": 3,
+      "counter_definition": "count superseded snapshot messages/buffers, not replacement operations"
+    }
+  ],
+  "socket_ceiling_ms": 5000,
+  "fixed_budget": {
+    "post_live_scenarios": 1,
+    "post_live_ticks": 100,
+    "reference_replay_ticks": 100,
+    "additional_fault_campaigns": 0,
+    "load_runs": 0
+  }
+}
