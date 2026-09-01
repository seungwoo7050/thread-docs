## `test(M07): preserve exhausted reset-parser failure`

diff --git a/scripts/verify_m07.py b/scripts/verify_m07.py
index ac1bcce..ee0fc00 100644
--- a/scripts/verify_m07.py
+++ b/scripts/verify_m07.py
@@ -22,6 +22,32 @@ URL = "http://127.0.0.1:18081"
 OFFLINE = {"airplane_mode_on": "1", "wifi_on": "0", "mobile_data": "0"}
 
 
+def package_in_live_activities(activities):
+    # The fixed API34 dump lists attached nodes with '*'. Fields such as
+    # mLastFocusedRootTask and mLastPausedActivity are historical references.
+    lines = activities.splitlines()
+    assert lines and lines[0] == "ACTIVITY MANAGER ACTIVITIES (dumpsys activity activities)", activities
+    assert any(re.fullmatch(r"Display #\d+ \(activities from top to bottom\):", line)
+               for line in lines), "Missing activity display hierarchy"
+    assert "ActivityTaskSupervisor state:" in lines, "Incomplete activity hierarchy dump"
+    assert "\ufffd" not in activities and "\x00" not in activities, "Corrupt activity hierarchy dump"
+    assert not re.search(r"(?im)^\s*(?:Error\b|[\w.]*Exception\b|Permission Denial\b)|"
+                         r"\bDUMP (?:TIMEOUT|TIMED OUT)\b",
+                         activities), activities
+    entries = []
+    for line in lines:
+        node = line.strip()
+        if re.match(r"\*\s*(?:Task\b|Hist\b|ActivityRecord\b)", node):
+            assert re.fullmatch(
+                r"\*\s+(?:Task\{[0-9a-f]+ #\d+ type=[^\s{}]+[^{}]*\}|"
+                r"(?:Hist\s+#\d+:\s+)?ActivityRecord\{[0-9a-f]+ u\d+ "
+                r"[^\s{}]+/[^\s{}]+ t-?\d+[^{}]*\})", node), f"Malformed activity hierarchy entry: {node}"
+            entries.append(node)
+    assert any(re.match(r"\*\s+Task\{", node) for node in entries), "Missing attached task entries"
+    package = rf"(?<![\w.]){re.escape(PACKAGE)}(?=[/\s}}])"
+    return any(re.search(package, node) for node in entries)
+
+
 def main():
     parser = argparse.ArgumentParser()
     parser.add_argument("--adb", default="adb")
@@ -103,9 +129,10 @@ def main():
             activities = adb("shell", "dumpsys", "activity", "activities", timeout=remaining())
             assert activities.startswith("ACTIVITY MANAGER ACTIVITIES"), activities
             observed = time.monotonic()
-            present = PACKAGE in activities
+            present = package_in_live_activities(activities)
             record["observations"].append({"elapsedSeconds": observed - started, "pid": pid,
-                "packageInActivities": present, "pidCommandIndex": pid_index,
+                "packageInActivities": PACKAGE in activities, "packageInLiveActivities": present,
+                "pidCommandIndex": pid_index,
                 "activitiesCommandIndex": activities_index})
             remaining()
             if not pid and not present:
diff --git a/verification/M07.md b/verification/M07.md
index 3589e02..bb0745d 100644
--- a/verification/M07.md
+++ b/verification/M07.md
@@ -40,11 +40,11 @@ Cumulative attempt2, repair1/2 used; START and both existing M07 commits remain
 
 `host-typecheck-01` PASS; `host-jest-01` **75/75 PASS**; `host-harness-syntax-01` PASS. Exact argv, source snapshots, outputs and exits are in the evidence root (`*.command.json`, `*.log`, `snapshots/`). Coverage retains M01–M06, including four M05 accepts and exact M06 replay, and adds A/B/C+explicit edit, original conflict persistence/no-send, own-successor loss/replay, external-version rejection, base-only collision, transaction rollback and both schema4 provenance cases. Older harnesses advance only current schema assertions and required version/tombstone fields; original scenario inputs and domain results are unchanged. The repaired M07 harness, fixture and M07 inputs remain frozen.
 
-Android product verification is **NOT_RUN** until main tests the frozen candidate; no owner final device run is authorized.
+At initial product freeze, Android product verification was **NOT_RUN**. The first main execution and repair2 status are recorded below.
 
 `android-build-01` PASS (22.282s): debug app and existing test runner. Preserved artifacts/hashes are in `artifacts.json`; the app is `m07-candidate.apk` (SHA256 `a42cc4c143e51aae73ff3645d829f2491e457c29607c659f2852c45d483391f7`), and the test-runner bytes remain unchanged from M06. `candidate-manifest.json` freezes all current files, host/build snapshots, artifacts and prior raw evidence paths.
 
-Main-only final invocation, from `/private/tmp/mobile-systems-evolution-ed7baa2/react-native`, after an exclusive device lease and a free fixture port18081:
+Original main-only final invocation (`main-android-m07-01`), from `/private/tmp/mobile-systems-evolution-ed7baa2/react-native`, after an exclusive device lease and a free fixture port18081:
 
 ```sh
 python3 /private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M07/run.py main-android-m07-01 python3 scripts/verify_m07.py \
@@ -53,3 +53,23 @@ python3 /private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-nat
   --apk /private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M07/m07-candidate.apk \
   --evidence /private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M07/main-android-m07-01
 ```
+
+## Main failure and bounded harness repair2
+
+Cumulative attempt3; repair **2/2**, remaining **0**. `main-android-m07-01` exited1 in 96.861s (291 adb commands): A completed conflict/reopen/no-retry/explicit-edit assertions; B failed before launch or seed; C did not run. Every B reset probe (55) had no PID and mentioned the package only in historical `mLastFocusedRootTask`. Cleanup restored network `0/1/1`, left no app PID, and reaped fixture87026 with exit0. Original results/logs remain unchanged; `repair2/before-inventory.json` preserves all 1,027 prior evidence-file hashes.
+
+Repair2 changes only this ledger and the reset predicate in `scripts/verify_m07.py`, against candidate `f43f5a69ac38611d1a9173cc8e686f5e16e1f43b`. Attached starred Task/Hist/ActivityRecord entries determine presence; historical pointers do not. Incomplete, corrupt or error dumps fail closed. Observations retain the raw `packageInActivities` value and add `packageInLiveActivities`. The 10s deadline, continuous 1s quiet period, all A/B/C assertions, baseline selector, actual death boundaries and launch behavior are unchanged.
+
+`repair2/host-checks-01.result.json`: **13/13 PASS**, covering all 62 captured A/B probes, 14 synthetic parser cases, 24 malformed/error cases, bounded timing and whole-harness AST equivalence except the predicate/diagnostics. No device, fixture, network or scenario-body execution occurred. All 51 other tracked files, both preserved/build-output APK copies and prior evidence remain exact; unchanged Jest/build results were not rerun.
+
+Frozen harness SHA256: `cec3490239dad72ed2ca0b604497e62508f8720e638bc55d244eb92c73f120d8`. `repair2/frozen-candidate.json` records the source/APK hashes, exact diff and host commands. Main's repaired final A/B/C and required Android regressions are **PENDING**. M07 is not complete; no tag or successor work is authorized.
+
+## Final repair2 Android result — BLOCKED
+
+Main's `main-repair2-android-m07-01` exited1 in 153.153364916s (300 adb commands). Raw cases A/B are PASS; C failed during its initial reset, before launch or seed. The frozen parser rejected the actual entry `* Hist  #0: ActivityRecord{7d672f3 u0 com.mse.reactnative/.MainActivity t198 f} isExiting}` as malformed. No further parser change or rerun was performed.
+
+Raw evidence remains under the evidence root: `main-repair2-android-m07-01/result.json`, `main-repair2-android-m07-01/commands.json`, `main-repair2-android-m07-01.command.json` and `main-repair2-android-m07-01.log`. The original frozen manifest and all earlier failures remain unchanged.
+
+The harness reports fixture98780 exit0, absent app PID and restored network `0/1/1`; main's independent cleanup verification is pending at this preservation checkpoint. M05/M06/native CRUD Android regressions were **NOT_RUN** after the failure.
+
+Attempt3, repair **2/2**, remaining **0**: the overall execution is **BLOCKED**. M07 is unverified; no M07 tag, M08 or additional repair is authorized. This commit only preserves the tested failing parser and truthful result metadata; product sources, tests, inputs, timings and APKs are unchanged.


## `test(M07): retain attached exiting activities in reset barrier`

diff --git a/scripts/verify_m07.py b/scripts/verify_m07.py
index ee0fc00..77e38d8 100644
--- a/scripts/verify_m07.py
+++ b/scripts/verify_m07.py
@@ -34,6 +34,8 @@ def package_in_live_activities(activities):
     assert not re.search(r"(?im)^\s*(?:Error\b|[\w.]*Exception\b|Permission Denial\b)|"
                          r"\bDUMP (?:TIMEOUT|TIMED OUT)\b",
                          activities), activities
+    # API34 appends ' isExiting}' after ActivityRecord's first closing brace.
+    # It remains attached and must keep the reset barrier busy.
     entries = []
     for line in lines:
         node = line.strip()
@@ -41,7 +43,7 @@ def package_in_live_activities(activities):
             assert re.fullmatch(
                 r"\*\s+(?:Task\{[0-9a-f]+ #\d+ type=[^\s{}]+[^{}]*\}|"
                 r"(?:Hist\s+#\d+:\s+)?ActivityRecord\{[0-9a-f]+ u\d+ "
-                r"[^\s{}]+/[^\s{}]+ t-?\d+[^{}]*\})", node), f"Malformed activity hierarchy entry: {node}"
+                r"[^\s{}]+/[^\s{}]+ t-?\d+[^{}]*\}(?: isExiting\})?)", node), f"Malformed activity hierarchy entry: {node}"
             entries.append(node)
     assert any(re.match(r"\*\s+Task\{", node) for node in entries), "Missing attached task entries"
     package = rf"(?<![\w.]){re.escape(PACKAGE)}(?=[/\s}}])"
diff --git a/verification/M07.md b/verification/M07.md
index bb0745d..71b0844 100644
--- a/verification/M07.md
+++ b/verification/M07.md
@@ -73,3 +73,15 @@ Raw evidence remains under the evidence root: `main-repair2-android-m07-01/resul
 The harness reports fixture98780 exit0, absent app PID and restored network `0/1/1`; main's independent cleanup verification is pending at this preservation checkpoint. M05/M06/native CRUD Android regressions were **NOT_RUN** after the failure.
 
 Attempt3, repair **2/2**, remaining **0**: the overall execution is **BLOCKED**. M07 is unverified; no M07 tag, M08 or additional repair is authorized. This commit only preserves the tested failing parser and truthful result metadata; product sources, tests, inputs, timings and APKs are unchanged.
+
+## Authorized supplemental parser repair3 — candidate pending
+
+The subsequent user request, "phase 1 종료까지 작업을 지속하세요.", permits one narrow supplement for the reported parser blocker, recorded by main in `threads/RESUME-PHASE-1-2026-08-28.md`. Cumulative **attempt4 / total repair3**; the original default **2/2** exhaustion and BLOCKED evidence above remain intact. This uses the sole authorized supplement; no further repair or general limit increase is inferred.
+
+From `301b132721a4f26d701011a56218c945cdcdc00b`, only the ActivityRecord regex accepts the exact optional ` isExiting}` suffix. Attached exiting Hist/token records remain **live/busy**, including with absent PID. Task syntax, malformed-dump rejection, the overall 10s deadline, continuous 1s quiet period, launch/death boundaries, inputs and final A/B/C selection/assertions are unchanged.
+
+`repair3/before-inventory.json` freezes all 53 source files and 1,149 prior evidence files. The exact raw command290 dump is copied to `repair3/fixtures/activities-0290.txt`. `repair3/before-reproduction-01` reproduced the frozen repair2 parser's exact rejection (exit0, 0.174190541s). `repair3/host-checks-01` passed **23/23** (exit0, 1.483543959s; test body 1.342570916s), reusing the untouched 13-test repair2 suite. Added checks cover the raw failure, isolated exiting Hist/token records, package boundaries, 16 malformed suffix/component cases, busy→absent/reappearance quiet transitions, deadline enforcement and exact source/evidence preservation. These are host simulations, not Android acceptance.
+
+Harness SHA256: `c462e65b3cb99bfa222be903c2994e21f456ee902f63818b11b7c0c983a3844c`. `repair3/frozen-candidate.json` freezes the candidate source, diff, commands, fixture/input hashes and unchanged preserved/build-output APK copies. All 51 other tracked files and prior evidence remain exact. No adb, fixture/network call, product test, Jest or APK rebuild occurred in this supplement.
+
+Main's final **A/B/C plus required M05/M06/native CRUD Android regressions are PENDING** on the same APKs. M07 remains unverified; this candidate creates no completion tag or successor authorization. If acceptance fails, the supplement is exhausted and execution must stop BLOCKED.


## `docs(M07): record main verified Android outcomes`

diff --git a/verification/M07.md b/verification/M07.md
index 71b0844..06a6547 100644
--- a/verification/M07.md
+++ b/verification/M07.md
@@ -3,6 +3,7 @@
 - Spec revision: `61280dd86ce88b6e431f408241c0998a275960aa`.
 - START: verified M06 `f1107d4f8f667dc4d440358181b4ff92f0b9e030`.
 - Evidence root: `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M07/`.
+- Current status: **main runtime PASS**, cumulative **attempt4 / total repair3**. Final metadata/history audit and tag remain main-owned; the earlier checkpoints below are retained unchanged.
 
 ## Initial baseline (attempt1): incomplete
 
@@ -85,3 +86,14 @@ From `301b132721a4f26d701011a56218c945cdcdc00b`, only the ActivityRecord regex a
 Harness SHA256: `c462e65b3cb99bfa222be903c2994e21f456ee902f63818b11b7c0c983a3844c`. `repair3/frozen-candidate.json` freezes the candidate source, diff, commands, fixture/input hashes and unchanged preserved/build-output APK copies. All 51 other tracked files and prior evidence remain exact. No adb, fixture/network call, product test, Jest or APK rebuild occurred in this supplement.
 
 Main's final **A/B/C plus required M05/M06/native CRUD Android regressions are PENDING** on the same APKs. M07 remains unverified; this candidate creates no completion tag or successor authorization. If acceptance fails, the supplement is exhausted and execution must stop BLOCKED.
+
+## Main verification after authorized supplement — PASS
+
+Main tested `203cc58756a4798b2d914ee56b459c29493b797d` using the unchanged app/test APKs. Its [completed runtime audit](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M07/main-all-runtime-verification.json) directly verifies **42 native databases** (23 M07, nine M05, ten M06), preserving the existing 75/75 host tests and 23/23 parser checks.
+
+- **M07 A/B/C PASS:** 228.150s, 415 adb commands. PIDs A5857→absent→6352, B7264→absent→7727, C8398→absent→8855. Original conflicts survive restart without ordinary resend; A's fresh identity/base2 produces the exact version3 explicit edit. [Raw result](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M07/main-repair3-android-m07-01/result.json), [exact invocation](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M07/main-repair3-android-m07-01.command.json), [main audit](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M07/main-repair3-ABC-audit.json).
+- **M05 regression PASS:** 166.808s, 241 adb commands; PID9158→absent→10456, four ordered accepts with bases none/1/2/1, exact original canonical results, pending0. [Raw result](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M07/main-repair3-android-m05-01/result.json), [main audit](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M07/main-m05-regression-audit.json).
+- **M06 regression PASS:** 93.935s, 133 adb commands; PID11486→absent→11923, identical replay applied1/duplicate1. Collision PID12166→absent→12449 preserves terminal evidence without resend; an incorrect actual body/hash returns400. [Raw result](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M07/main-repair3-android-m06-01/result.json), [main audit](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M07/main-m06-regression-audit.json).
+- **Native CRUD PASS (one test):** [result](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M07/main-repair3-native-crud-01/result.json), [main audit](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M07/main-native-crud-audit.json).
+
+Each run restored network `0/1/1` and left the app absent; all owned fixtures exited0. No owner runtime repeat or rebuild occurred. Attempt4/repair3 retains default2/2 exhaustion plus the sole [authorized supplement](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/RESUME-PHASE-1-2026-08-28.md), with zero repairs remaining. This final update changes only this ledger; main's final byte/history audit and tag remain outstanding, and M08 is not authorized.
