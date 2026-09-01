## `docs(M08): record verified native lifecycle and CRUD results`

diff --git a/verification/M08.md b/verification/M08.md
index 011c8dd..61de8c1 100644
--- a/verification/M08.md
+++ b/verification/M08.md
@@ -2,7 +2,7 @@
 
 - Spec revision: `61280dd86ce88b6e431f408241c0998a275960aa`.
 - START: verified M07 `34e03d3123e513e0dcd0ea2e55be981f6577249b`.
-- Attempt1; repairs0/2. Current checkpoint: baseline accepted, host checks PASS; final Android verification pending.
+- Attempt1; repairs0/2. Current status: **main host and Android verification PASS**; final metadata/history audit and tag remain main-owned.
 - Evidence root: `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M08/`.
 
 ## Frozen unchanged-app baseline
@@ -11,7 +11,7 @@
 
 The single [baseline invocation](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M08/baseline-android-01.command.json) exited0 in 17.057s, with 34 adb commands: **BASELINE_LIMIT_REPRODUCED**, not Android acceptance PASS. Native JUnit failed only `M08_RECREATION_DRAFT_OR_SELECTION_LOST`. The draft survived one CREATED→RESUMED cycle; real recreation destroyed Activity46567430 and created232223380 with saved state, losing the editor/draft. PID15260, Application88250467 and ReactContext136876754 stayed identical. All five native databases retained exact `ui-001` / `Saved title` / version1 and pending0; no submit or HTTP request occurred.
 
-[Raw result](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M08/baseline-android-01/result.json), native archive/DB/UI/lifecycle logs and [main's independent baseline audit](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M08/main-baseline-audit.json) retain the complete proof. Cleanup left the app absent, network0/1/1 active and fixture90263 exited0/absent. The denied initial host `ps` probe and subsequent approved absence check are preserved beside the result; neither reran the Android scenario. Production implementation and main's final M08/native CRUD remain pending.
+[Raw result](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M08/baseline-android-01/result.json), native archive/DB/UI/lifecycle logs and [main's independent baseline audit](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M08/main-baseline-audit.json) retain the complete proof. Cleanup left the app absent, network0/1/1 active and fixture90263 exited0/absent. The denied initial host `ps` probe and subsequent approved absence check are preserved beside the result; neither reran the Android scenario. Production implementation and main's final checks were pending at this baseline checkpoint.
 
 ## Implementation and host checks
 
@@ -20,3 +20,13 @@ Following main's baseline acceptance, only `src/App.tsx` changes production beha
 `host-typecheck-01` **PASS** (1.368s); `host-jest-01` **80/80 PASS** (5.929s), preserving all 75 prior tests. Five focused cases cover draft versus committed state, remount/one explicit mutation, stale handlers, late Save/keyboard cleanup, failed Save/cancel and disposed opening effects. Exact argv, source snapshots and outputs are under the evidence root. No owner fixed Android invocation or repeated baseline occurred; main's final M08/native CRUD is **NOT_RUN** at this checkpoint.
 
 `candidate-app-build-01` **PASS** (10.518s), app-only. Preserved candidate SHA256: `4dc8e86b32b1ca9f72fdf578a57fc197f7c1e81416725111c03e30278b98bbc6`. The only changed APK ZIP entry from M07 is `assets/index.android.bundle`; the instrumentation APK remains exactly frozen. `candidate-preservation-01` confirms host/build source identity and unchanged storage, sync, native implementation and execution support. [Candidate manifest](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M08/candidate-manifest.json) freezes the final files/APKs and exact main command (same harness without `--baseline`, output `main-android-m08-01`). Main alone runs the required final M08 and native CRUD checks; no other long Android scenarios are repeated for unchanged persistence/sync code.
+
+## Main final verification — PASS
+
+Main verified frozen candidate `bdf80286a1028c3a1972d76687fceaa812fd6934`; all 57 files and both APKs remained exact. [Completed runtime audit](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M08/main-all-runtime-verification.json).
+
+- **New host execution:** typecheck PASS (3.169s), Jest80/80 PASS (6.073s). `main-host-jest-01` failed before tests because Watchman `fchmod` was denied by the sandbox; its raw failure remains preserved. The same frozen command/code passed as approved `main-host-jest-02`, with no product repair. [Host audit](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M08/main-host-audit.json).
+- **New M08 Android execution PASS:** one native test, 18.746s overall, 34 adb commands. PID15596/Application88250467/ReactContext151148893 stayed stable; Activity46567430 survived background/foreground, then recreation replaced it with180411011. Selected `ui-001` and its draft survived. Six native databases prove unchanged `Saved title`/v1/pending0 until one explicit Save produced `Unsubmitted draft`/v2 and one base1 intent, dispatched0. HTTP requests0. [Raw result](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M08/main-android-m08-01/result.json), [lifecycle/database audit](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M08/main-lifecycle-audit.json).
+- **New native CRUD execution PASS:** one test, 27.691s. [Main audit](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M08/main-native-crud-audit.json).
+
+Main independently confirmed network0/1/1, absent app, fixture5558 exited0/absent and port18081 free. Prior M05/M06/M07 Android evidence is reused only for unchanged storage/sync/native/fixture behavior; those long scenarios were not rerun. Current host tests cover the changed App guards. This final update changes only this ledger; no owner rebuild/device repeat occurred. Attempt1/repairs0 remains unchanged; main's final commit audit/tag and any M09 assignment are pending.
