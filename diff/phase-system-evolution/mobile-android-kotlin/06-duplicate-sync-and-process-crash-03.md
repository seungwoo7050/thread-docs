## `M06 preserve unverified matcher repair at global stop`

diff --git a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt
index 1e0c763..b635dc5 100644
--- a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt
+++ b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt
@@ -540,7 +540,7 @@ class ItemUiTest {
             compose.onNodeWithText("Sync").performClick()
             awaitText("Stale local data · sync error")
             awaitCrudText("Different payload")
-            compose.onNodeWithTag("sync-error").assertIsDisplayed().assertTextContains("identity_conflict")
+            compose.onNodeWithTag("sync-error").assertIsDisplayed().assertTextContains("identity_conflict", substring = true)
             compose.onNodeWithTag("pending-count").assertTextEquals("Pending changes: 1")
             val pending = runBlocking { store.pendingMutations().single() }
             assertEquals("identity_conflict", pending.terminalError)
@@ -556,7 +556,7 @@ class ItemUiTest {
             compose.runOnUiThread { compose.activity.setContent { MaterialTheme { ItemScreen(reopened, resumed) } } }
             awaitText("Stale local data · sync error")
             awaitCrudText("Different payload")
-            compose.onNodeWithTag("sync-error").assertIsDisplayed().assertTextContains("identity_conflict")
+            compose.onNodeWithTag("sync-error").assertIsDisplayed().assertTextContains("identity_conflict", substring = true)
             compose.onNodeWithText("Sync").performClick()
             awaitCrudText("Different payload")
             assertEquals(listOf(pending), runBlocking { reopened.pendingMutations() })
diff --git a/verification/M06.md b/verification/M06.md
index dd23f9e..0073747 100644
--- a/verification/M06.md
+++ b/verification/M06.md
@@ -2,8 +2,8 @@
 
 START `9b66fe83f24808ade59d9313a460a3c76652e52c`; branch `track/android-kotlin`.
 SPEC_REVISION `61280dd86ce88b6e431f408241c0998a275960aa`.
-Initial attempt1, repair0/2 at failure. Baseline accepted; fixed candidate failed final
-JUnit (16/17 pass); external replay NOT_RUN. Repair1/2 authorized, not yet applied.
+Initial attempt1 failed final JUnit (16/17 pass); external replay NOT_RUN.
+Repair1/2 built, uncommitted and awaiting main review/full17/replay. Baseline accepted.
 
 Evidence (`E`): `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/android-kotlin/M06`.
 Exact arguments/environment/exits are in `E/commands.jsonl` and each named JSON/log.
@@ -67,3 +67,42 @@ Ledger: initial attempt1 failed; repair1/2 is authorized for only the two new
 `identity_conflict` matchers and a test APK rebuild. No second repair is started.
 This checkpoint contains the original failing test bytes, not a repaired candidate.
 Main owns the subsequent full17 and external replay acceptance runs.
+
+## Repair1/2 candidate — build only, runtime pending
+
+Preservation commit `fb95f5994868a0cb00b4a825081c00d9d312c29b` contains exactly the
+original ten frozen source paths and the truthful failed-run verification above.
+Pinned Compose BOM2024.06.00 resolves ui-test1.6.8. Local cached AAR/runtime
+`AssertionsKt.class` bytes match; `E/repair1-compose-api-01.log:303–349` shows
+the `substring` parameter in slot2 and its default `false` (`iconst_0`, `istore_2`).
+Only the two new assertions at `ItemUiTest.kt:543,559` now pass `substring = true`.
+All other test methods/helpers, product, fixture, inputs, timeouts, acceptance,
+protocol and Gradle configuration remain unchanged.
+
+Command `./gradlew --no-daemon --console=plain :app:assembleDebugAndroidTest`
+exited0; raw log and exact environment/arguments/exits are
+`E/repair1-test-build-01.log` and `.json`. Main compile tasks were UP-TO-DATE;
+the app APK was not rebuilt and still has the original frozen hash above.
+New test APK SHA256 `7a0e070d4bf139f1e51d54df4c260c1d4c164eaeab1508241aacc36696303582`.
+`E/repair1-built-01/manifest.json` freezes all42 current source hashes and the new
+test APK, referencing the existing app and 11PASS host XML without copying/rerunning.
+
+Ledger: initial attempt1 failed16/17; repair1/2 built only; repair2/2 not started.
+No owner device, fixture, native suite or replay execution occurred for repair1.
+The original failed evidence is retained; repair1 full17/reopen/no-retry/replay
+are NOT_RUN/UNVERIFIED pending main. No final acceptance is claimed.
+
+## Administrative checkpoint — UNVERIFIED / PAUSED_BY_GLOBAL_BLOCK
+
+Root stopped execution after React Native M07 exhausted repair2. M06 is paused,
+not complete: current attempt2, repair1/2 used, remaining1. The original Android
+suite remains17 executed/16PASS/1FAIL. The repair1 test APK was only built;
+its full17, reopen/no-retry and external replay are NOT_RUN/UNVERIFIED.
+No additional implementation, build, device, fixture or runtime execution followed.
+
+This administrative checkpoint preserves only the two matcher edits and this ledger.
+The existing-byte freeze `E/repair1-built-01/manifest.json` has SHA256
+`c475d4a58ee931971abe8a32283be2e7db01a34307c5da28ece8e46bf676402f` and preserves
+the pre-administrative report snapshot; the final checkpoint metadata references
+this freeze and records the final report hash. Original failed source/APKs/raw logs
+and host11PASS evidence remain intact. No acceptance or completion tag is authorized.


## `M06 correct current paused verification header`

diff --git a/verification/M06.md b/verification/M06.md
index 0073747..841d76e 100644
--- a/verification/M06.md
+++ b/verification/M06.md
@@ -3,7 +3,7 @@
 START `9b66fe83f24808ade59d9313a460a3c76652e52c`; branch `track/android-kotlin`.
 SPEC_REVISION `61280dd86ce88b6e431f408241c0998a275960aa`.
 Initial attempt1 failed final JUnit (16/17 pass); external replay NOT_RUN.
-Repair1/2 built, uncommitted and awaiting main review/full17/replay. Baseline accepted.
+Repair1/2 preserved in an administrative checkpoint; runtime UNVERIFIED and paused by global block. Baseline accepted.
 
 Evidence (`E`): `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/android-kotlin/M06`.
 Exact arguments/environment/exits are in `E/commands.jsonl` and each named JSON/log.


## `docs(android): record M06 repaired runtime evidence`

diff --git a/verification/M06.md b/verification/M06.md
index 841d76e..72c4cb7 100644
--- a/verification/M06.md
+++ b/verification/M06.md
@@ -3,7 +3,8 @@
 START `9b66fe83f24808ade59d9313a460a3c76652e52c`; branch `track/android-kotlin`.
 SPEC_REVISION `61280dd86ce88b6e431f408241c0998a275960aa`.
 Initial attempt1 failed final JUnit (16/17 pass); external replay NOT_RUN.
-Repair1/2 preserved in an administrative checkpoint; runtime UNVERIFIED and paused by global block. Baseline accepted.
+Current attempt2, repair1/2: main's repaired Android17/17 and external replay PASS.
+Baseline accepted; main's final byte/history audit and tag remain pending.
 
 Evidence (`E`): `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/android-kotlin/M06`.
 Exact arguments/environment/exits are in `E/commands.jsonl` and each named JSON/log.
@@ -68,7 +69,7 @@ Ledger: initial attempt1 failed; repair1/2 is authorized for only the two new
 This checkpoint contains the original failing test bytes, not a repaired candidate.
 Main owns the subsequent full17 and external replay acceptance runs.
 
-## Repair1/2 candidate — build only, runtime pending
+## Historical repair1/2 checkpoint — build only, runtime pending
 
 Preservation commit `fb95f5994868a0cb00b4a825081c00d9d312c29b` contains exactly the
 original ten frozen source paths and the truthful failed-run verification above.
@@ -92,7 +93,7 @@ No owner device, fixture, native suite or replay execution occurred for repair1.
 The original failed evidence is retained; repair1 full17/reopen/no-retry/replay
 are NOT_RUN/UNVERIFIED pending main. No final acceptance is claimed.
 
-## Administrative checkpoint — UNVERIFIED / PAUSED_BY_GLOBAL_BLOCK
+## Historical administrative checkpoint — UNVERIFIED / PAUSED_BY_GLOBAL_BLOCK
 
 Root stopped execution after React Native M07 exhausted repair2. M06 is paused,
 not complete: current attempt2, repair1/2 used, remaining1. The original Android
@@ -106,3 +107,32 @@ The existing-byte freeze `E/repair1-built-01/manifest.json` has SHA256
 the pre-administrative report snapshot; the final checkpoint metadata references
 this freeze and records the final report hash. Original failed source/APKs/raw logs
 and host11PASS evidence remain intact. No acceptance or completion tag is authorized.
+
+## Resumed main runtime — attempt2, repair1/2 PASS
+
+Following the user's phase-1 continuation authorization, main tested committed candidate
+`153ec58675d30ab7b6f11debbfe8acc0408bcd69` without another implementation/build change.
+The [main runtime result](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M06/main-repair1-android-01/result.json)
+records all17 Android methods passing:11 Room and6 UI, including the repaired collision
+test's visible terminal error, Room reopen and no-resend assertions. The original16/17
+failure and its raw evidence above remain retained; repair2/2 was not started.
+
+The unchanged [external replay scenario result](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M06/main-repair1-android-01/replay/result.json)
+passes with73 recorded adb commands and PID **4572 → absent → 4866**. Its native schema4
+captures retain the exact Item and pending identity/hash across death. The first request
+returns a visible lost-response error before force-stop; plain Activity relaunch performs
+no automatic resend. One explicit replay then yields pending0 and a stored acknowledgment
+with the original201 response body. Both wire requests use `m06-create-001` and hash
+`09d1c3d2db9864ea761ce798ad13b1ee10faba85a8500b552467408d8245d2ba`.
+Remote counters are applied1/duplicates1/requests2/droppedResponses1; its sole Item is
+crash-001/Crash safe/false/version1/1700000400000. No install, clear, seed or instrumentation
+crosses the death boundary. Expected initial-controller termination is not a JUnit pass.
+
+Main used unchanged app APK
+`0857d61276825965652bb1d1c12a348eff84a0f33688b0201e434ab7a66fa79a` and repaired test APK
+`7a0e070d4bf139f1e51d54df4c260c1d4c164eaeab1508241aacc36696303582`.
+Cleanup records owned fixture55736 exit0, app absent and network0/1/1 with active network101.
+Raw JUnit, fixture wire log, PID commands and four native DB/WAL captures remain in
+`main-repair1-android-01`; no duplicate raw copies or owner-side runtime rerun were made.
+This finalization changes only this report. Main owns the remaining independent audit/tag;
+M07 is not assigned by this metadata commit.
