## `docs: record verified M05 Android durability`

diff --git a/verification/M05.md b/verification/M05.md
index 4013797..75022de 100644
--- a/verification/M05.md
+++ b/verification/M05.md
@@ -129,3 +129,35 @@ is explicitly mocked and is not Android or remote-integration evidence.
 Per coordinated verification, main will directly execute the final fixed Android
 scenario and relevant regression once on the frozen candidate. No implemented
 Android success is claimed yet and this agent will not duplicate that run.
+
+## Main final verification — resumed 2026-08-28
+
+The user-authorized resume retained attempt1, the earlier stop/failures and frozen
+candidate `ad8ad4da8f04fd959a50b011873f53824de201e7`. No product change, rebuild or
+duplicate host/device run was needed. Main executed this exact command from the
+React Native branch root, using the unchanged pre-baseline harness and inputs:
+
+```sh
+python3 scripts/verify_m05.py --adb /opt/homebrew/share/android-commandlinetools/platform-tools/adb --serial emulator-5554 --node /Users/woopinbell/.local/share/fnm/node-versions/v22.22.0/installation/bin/node --apk /private/tmp/mobile-systems-evolution-ed7baa2/evidence/react-native/M05/m05-candidate.apk --evidence /private/tmp/mobile-systems-evolution-ed7baa2/evidence/react-native/M05/main-android-m05-01
+```
+
+**PASS, exit0:** 241 adb commands. Host PID8042 survived app PID4169 → absent →
+PID5541. Both local rows and all four ordered intents survived the offline death
+boundary136–177 without install/clear/database modification. One foreground drain
+accepted create, rename, completion and delete separately. Local/remote ended at
+Gamma/false/v1/1700000300000 and Queued edit/true/v3/1700000302000; pending0.
+Owned fixture PID8053 exited0, network0/1/1 was restored and the app was absent.
+
+Main independently read all nine native SQLite snapshots and verified all46
+candidate files remained frozen. The separate unchanged native M01 CRUD test
+passed1 (`OK (1 test)`, about23seconds including setup), with the app stopped
+afterward. M03/M04 host regressions are included in the39 passing tests; their
+separate Android harnesses were not rerun for this candidate.
+
+Full records: [M05 result](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/react-native/M05/main-android-m05-01/result.json),
+[main audit](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/react-native/M05/main-android-audit.json),
+[native CRUD result](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/react-native/M05/main-native-crud-01/result.json)
+and its adjacent `instrumentation.log`/`commands.json`. Tested APK SHA-256:
+`6698265d6cd2e7d4517b404a8287b41f3d557d5dc346b100cf24e1dde96012a2`.
+Only this verification ledger changes in the final commit. No M05 issue remains
+unresolved; main performs the final history/trailer audit before tagging.
