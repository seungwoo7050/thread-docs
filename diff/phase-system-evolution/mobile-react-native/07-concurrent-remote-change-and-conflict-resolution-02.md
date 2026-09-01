## `test(M07): bound reset teardown and resume missing baseline case`

diff --git a/scripts/verify_m07.py b/scripts/verify_m07.py
index f7f002a..ac1bcce 100644
--- a/scripts/verify_m07.py
+++ b/scripts/verify_m07.py
@@ -30,14 +30,21 @@ def main():
     parser.add_argument("--apk", required=True)
     parser.add_argument("--evidence", required=True)
     parser.add_argument("--baseline", action="store_true")
+    parser.add_argument("--case", dest="baseline_case", choices=("A", "B", "C"),
+                        help="Resume one baseline case; requires --baseline. Final verification always runs A/B/C.")
     args = parser.parse_args()
+    if args.baseline_case and not args.baseline:
+        parser.error("--case requires --baseline; final verification must run A/B/C")
     project = Path(__file__).resolve().parent.parent
     evidence = Path(args.evidence).resolve()
     evidence.mkdir(parents=True, exist_ok=False)
     inputs_path = project / "verification/M07-inputs.json"
     inputs = json.loads(inputs_path.read_text())
+    selected_cases = [case for case in inputs["cases"]
+                      if args.baseline_case is None or case["case"] == args.baseline_case]
     commands, controls = [], []
     result = {"status": "RUNNING", "baseline": args.baseline, "host_pid": os.getpid(), "cases": [],
+              "selected_cases": [case["case"] for case in selected_cases],
               "apk_sha256": hashlib.sha256(Path(args.apk).read_bytes()).hexdigest(),
               "harness_sha256": hashlib.sha256(Path(__file__).read_bytes()).hexdigest(),
               "fixture_sha256": hashlib.sha256((project / "fixture/server.cjs").read_bytes()).hexdigest(),
@@ -46,9 +53,26 @@ def main():
     def save():
         (evidence / "result.json").write_text(json.dumps(result, indent=2) + "\n")
 
-    def adb(*parts, check=True, binary=False):
+    def adb(*parts, check=True, binary=False, timeout=60):
         command = [args.adb, "-s", args.serial, *parts]
-        completed = subprocess.run(command, capture_output=True, timeout=60)
+        started = time.monotonic()
+        try:
+            completed = subprocess.run(command, capture_output=True, timeout=timeout)
+        except subprocess.TimeoutExpired as error:
+            entry = {"command": command, "exit": None, "timeoutSeconds": timeout,
+                     "elapsedSeconds": time.monotonic() - started, "error": repr(error)}
+            for stream in ("stdout", "stderr"):
+                output = getattr(error, stream)
+                entry[stream] = None
+                if output is not None:
+                    raw = output if isinstance(output, bytes) else output.encode()
+                    path = evidence / f"adb-{len(commands):04d}-timeout.{stream}"
+                    path.write_bytes(raw)
+                    entry[stream + "File"] = path.name
+                    entry[stream] = "<binary>" if binary and stream == "stdout" else raw.decode(errors="replace")
+            commands.append(entry)
+            (evidence / "commands.json").write_text(json.dumps(commands, indent=2) + "\n")
+            raise
         commands.append({"command": command, "exit": completed.returncode,
                          "stdout": "<binary>" if binary else completed.stdout.decode(errors="replace"),
                          "stderr": completed.stderr.decode(errors="replace")})
@@ -57,6 +81,43 @@ def main():
             raise AssertionError(commands[-1])
         return completed.stdout if binary else completed.stdout.decode(errors="replace").strip()
 
+    def wait_for_reset_teardown(name):
+        # Only before initial seed: pm clear can leave an old task's delayed
+        # removal callback. Do not launch while that task/process is settling.
+        started = time.monotonic()
+        deadline, quiet_since = started + 10, None
+        record = {"case": name, "status": "RUNNING", "timeoutSeconds": 10,
+                  "quietSeconds": 1, "observations": []}
+        result.setdefault("reset_barriers", []).append(record)
+
+        def remaining():
+            value = deadline - time.monotonic()
+            assert value > 0, "Initial reset teardown did not settle within 10s"
+            return value
+
+        while True:
+            pid_index = len(commands)
+            pid = adb("shell", "pidof", PACKAGE, check=False, timeout=remaining())
+            assert commands[-1]["exit"] in (0, 1), commands[-1]
+            activities_index = len(commands)
+            activities = adb("shell", "dumpsys", "activity", "activities", timeout=remaining())
+            assert activities.startswith("ACTIVITY MANAGER ACTIVITIES"), activities
+            observed = time.monotonic()
+            present = PACKAGE in activities
+            record["observations"].append({"elapsedSeconds": observed - started, "pid": pid,
+                "packageInActivities": present, "pidCommandIndex": pid_index,
+                "activitiesCommandIndex": activities_index})
+            remaining()
+            if not pid and not present:
+                if quiet_since is None:
+                    quiet_since = observed
+                if observed - quiet_since >= 1:
+                    record.update(status="PASS", elapsedSeconds=observed - started)
+                    return
+            else:
+                quiet_since = None
+            time.sleep(min(0.1, remaining()))
+
     def remote(path="/__state", body=None):
         event = {"method": "GET" if body is None else "POST", "path": path, "body": body}
         try:
@@ -246,7 +307,7 @@ def main():
         adb("install", "-r", args.apk)
         adb("shell", "input", "keyevent", "KEYCODE_WAKEUP")
         adb("shell", "wm", "dismiss-keyguard")
-        for case in inputs["cases"]:
+        for case in selected_cases:
             name = case["case"]
             current = {"case": name, "status": "RUNNING"}
             result["cases"].append(current)
@@ -254,6 +315,7 @@ def main():
             assert reset == {"items": [inputs["seed"]], "nextTimestamp": 1700000501000, "requests": [], "delayMs": 0}
             adb("shell", "am", "force-stop", PACKAGE)
             assert adb("shell", "pm", "clear", PACKAGE) == "Success"
+            wait_for_reset_teardown(name)
             launch(case["clientMutationId"])
             tap("Synchronize")
             find("Sync status: fresh")
diff --git a/verification/M07.md b/verification/M07.md
index b1f6577..d76e42e 100644
--- a/verification/M07.md
+++ b/verification/M07.md
@@ -15,3 +15,21 @@ Frozen support is preserved in `frozen-support/` and `frozen-inputs-before-basel
 Cleanup in the original result: network restored to `0/1/1`, app PID absent, owned fixture exit0. Root's preserved `threads/evidence/phase-1/react-native/M07/main-launch-diagnostic-01/` identifies the old task172 removal callback killing new process14551 at 13:17:39.542. A fresh bounded harness repair uses repair1 of the hard maximum2; this count remains part of M07.
 
 Host-only audit `repair1-original-audit-01` passed (exit0): ten raw A/B database snapshots, ID-only substitution, wire hashes, side effects and UI; original evidence inventory retained at `repair1/original-evidence-sha256.json`.
+
+## Bounded harness repair1 (setup only)
+
+Initial support checkpoint: `9a7ebe37e544f99216e8c249a39ff84d5cb4915f`. Only the harness and this ledger change afterward. Before each initial seed, after `pm clear`, the harness requires absent app PID and no package in the activity dump for 1s, within a strict 10s setup deadline; raw probe output and observation indices are retained. No barrier is added across a tested death boundary.
+
+The 1s quiet period uses the 1000ms removal callback in [official AOSP ActivityTaskSupervisor source](https://android.googlesource.com/platform/frameworks/base/+/178bc0484818a0fd42b710135cb2e1d6c6f87347/services/core/java/com/android/server/wm/ActivityTaskSupervisor.java) as its basis, alongside the actual preserved task172 kill log. This source is not claimed to identify the emulator's exact framework build. The existing 60s adb timeout, network controls, scenario timing and all acceptance assertions remain unchanged. Timeout failures now preserve the command plus available raw stdout/stderr and still propagate; unavailable output remains null.
+
+`--case C` is allowed only with `--baseline`; omitted selection and every final verification run retain A/B/C. Host-only `repair1-host-checks-01` passed (exit0): syntax, unchanged scenario AST/assertions, selector rejection, stable/bounded teardown behavior, malformed probe rejection, and timeout evidence preservation. It executed no device commands.
+
+Repaired harness SHA256: `23b2c1f127f385ee759ff0a6242801ec62794c51fc97561dd652b5f1280bd694`. Frozen diff, source and exact C-only command are recorded in `repair1/frozen-before-C.json`. Root review and an exclusive baseline-C lease were required before the single execution; C was unverified at this pre-run checkpoint.
+
+## Repair1 baseline C: reproduced and audited
+
+Frozen `repair1-baseline-C-01` (`--baseline --case C`) exited0 in 49.932s. The exact `m07-deleted-001` unversioned PATCH returned404; native state/pending1 remained unchanged with no conflict metadata, while the remote tombstone stayed version2 and appliedCount0. Five native database snapshots, final UI, HTTP controls, 129 adb command records and the exact invocation are retained under the evidence root. A/B were not repeated.
+
+The reset gate passed in 1.137s with seven recorded absent-PID/package observations. It ran successfully, but this C-only execution did not deliberately recreate the old teardown race. Cleanup restored network `0/1/1`, left app PID absent and reaped owned fixture70992 with exit0. No additional device execution or APK rebuild occurred.
+
+Main independently accepted the actual C database/UI/wire evidence and frozen byte identity; audit: `threads/evidence/phase-1/react-native/M07/main-repair1-C-audit.json` in the main worktree. Original A/B evidence plus this C run complete baseline reproduction only. Repair usage stays **1/2**; M07 product implementation and official final A/B/C verification remain outstanding. No M07 completion tag is created by this repair.


