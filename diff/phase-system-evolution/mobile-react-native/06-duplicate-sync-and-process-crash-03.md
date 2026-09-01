## `docs(M06): record main Android replay and collision verification`

diff --git a/verification/M06.md b/verification/M06.md
index f4f617c..980bef0 100644
--- a/verification/M06.md
+++ b/verification/M06.md
@@ -117,7 +117,42 @@ existing real SQLite file bridge; HTTP integration uses the loopback fixture.
 Full structured results: [jest-02-results.json](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M06/jest-02-results.json).
 
 The executed M06 fixture/harness/inputs remain byte-identical to the baseline
-freeze. Candidate hashes and copied APKs will be in `candidate-manifest.json`.
-Main owns final fixed Android verification on that frozen candidate; it has not
-yet run. This owner will not duplicate it and will change only this ledger after
-main returns the actual result.
+freeze. Candidate hashes and copied APKs are in `candidate-manifest.json`.
+At candidate freeze, main's final Android run was pending. This owner did not
+duplicate it; only this ledger changes after main's result.
+
+## Main final Android verification — 2026-08-28
+
+Main verified candidate `7c5bfbd9f0f3e6e2472e3e91f26e8a2cfffd403e` without rebuild
+or source changes. APK SHA-256:
+`f9ef2c2c384a21fda0da03243d1160097b0780d400d2084aee26844633dccb90`.
+From the React Native branch root:
+
+```sh
+python3 scripts/verify_m06.py --adb /opt/homebrew/share/android-commandlinetools/platform-tools/adb --serial emulator-5554 --node /Users/woopinbell/.local/share/fnm/node-versions/v22.22.0/installation/bin/node --apk /private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M06/m06-candidate.apk --evidence /private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M06/main-android-m06-01
+```
+
+**PASS**, exit0,111.275seconds,133 adb commands. Host PID49911 survived app
+PID8923 → absent → PID9317. The production-created intent retained
+`m06-create-001` and hash
+`09d1c3d2db9864ea761ce798ad13b1ee10faba85a8500b552467408d8245d2ba`
+across command boundary49–62, with no install/clear/state injection. One replay
+returned the original201 result: applied1, duplicate1; local/remote exactly
+crash-001/Crash safe/false/v1/1700000400000, acknowledgment persisted, pending0.
+
+The isolated production-created alternate payload returned409 and persisted
+terminal intent across PID9585→9881. An additional explicit drain sent nothing.
+The wrong declared hash then returned400 against the actual body; final evidence
+counts are applied1/duplicate1/identity-conflict1/hash-rejected1. Main independently
+read all10 native SQLite snapshots. Owned fixture49917 exited0, the app was absent,
+and recorded network0/1/1 with an active default network was preserved.
+
+The separate unchanged native M01 CRUD test also **passed1** and cleaned up its
+app process. M03–M05 regression coverage is in the56 host tests; their separate
+Android harnesses were not rerun on this candidate. All50 candidate files and both
+APKs stayed frozen through main verification; this final commit changes only this
+ledger. No M06 issue remains unresolved.
+
+Full records: [fixed raw result](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M06/main-android-m06-01/result.json),
+[main audit](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M06/main-android-audit.json),
+[native CRUD result](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M06/native-crud-01/result.json).
