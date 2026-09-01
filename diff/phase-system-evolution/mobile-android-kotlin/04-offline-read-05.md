## `test(android): record final M04 focus repair failure`

diff --git a/TRACK.md b/TRACK.md
index ede3cdb..3b61ebb 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -86,6 +86,12 @@ direct M03 seed and schema-2 external process restart pass. The full suite remai
 the new focus regression is unresolved, with no suite rerun or further source fix in this
 repair. `verification/M04.md` preserves both attempts and the complete cleanup record.
 
+Repair 2 added only M01 test completion/focus readiness predicates under the original
+five-second deadlines, preserving every CRUD action and assertion. Main's final suite still
+has ten passes and the same M01 Compose focus failure. **M04 is BLOCKED after both permitted
+repairs; no progress tag or M05 work follows.** Its failed final verification and diagnostic
+limits are recorded in `verification/M04.md`. The repair did not change product code or inputs.
+
 M04 reproduced M03's missing freshness and successful-refresh-time presentation without
 changing its working cached reads. The ordinary screen still reads Room before any network
 request. **Sync** remains the only refresh trigger; restoring transport does not itself send
diff --git a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt
index b6703f5..23d8c8d 100644
--- a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt
+++ b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt
@@ -72,37 +72,61 @@ class ItemUiTest {
         }
     }
 
+    private fun awaitCrudText(text: String) {
+        compose.waitUntil(5_000) {
+            compose.onAllNodesWithText(text).fetchSemanticsNodes().any {
+                it.config.getOrNull(SemanticsProperties.EditableText) == null
+            } && crudReady()
+        }
+        android.util.Log.i("M01Ready", "$text: committed text; editor enabled, empty, unfocused; IME hidden")
+    }
+
+    private fun crudReady(): Boolean {
+        // Room completion and Android IME hiding are not driven by Compose's test clock.
+        // Rows can render before readSavedStatus, draft clearing and busy=false finish.
+        val input = compose.onNodeWithTag("item-title-input").fetchSemanticsNode().config
+        var imeHidden = false
+        compose.runOnUiThread {
+            imeHidden = compose.activity.window.decorView.rootWindowInsets
+                ?.isVisible(WindowInsets.Type.ime()) == false
+        }
+        return input.getOrNull(SemanticsProperties.Disabled) == null &&
+            input.getOrNull(SemanticsProperties.EditableText)?.text == "" &&
+            input.getOrNull(SemanticsProperties.Focused) == false && imeHidden
+    }
+
     private fun awaitCompleted(completed: Boolean) {
         val expected = if (completed) ToggleableState.On else ToggleableState.Off
         compose.waitUntil(5_000) {
             compose.onNodeWithContentDescription("Completed: Alpha edited").fetchSemanticsNode()
-                .config[SemanticsProperties.ToggleableState] == expected
+                .config[SemanticsProperties.ToggleableState] == expected && crudReady()
         }
+        android.util.Log.i("M01Ready", "completed=$completed; editor enabled, empty, unfocused; IME hidden")
     }
 
     @Test
     fun frozenM01SequenceThroughActualAndroidUi() {
-        awaitText("Items (0)")
+        awaitCrudText("Items (0)")
         compose.onNodeWithTag("item-count").assertTextEquals("Items (0)")
         compose.onNodeWithText("No items").assertIsDisplayed()
 
         compose.onNodeWithTag("item-title-input").performTextInput("Alpha")
         compose.onNodeWithText("Add").performClick()
-        awaitText("Alpha")
+        awaitCrudText("Alpha")
         compose.onNodeWithText("Alpha").assertIsDisplayed()
         val firstRowTag = compose.onAllNodes(rowMatcher).fetchSemanticsNodes().single()
             .config[SemanticsProperties.TestTag]
 
         compose.onNodeWithTag("item-title-input").performTextInput("Beta")
         compose.onNodeWithText("Add").performClick()
-        awaitText("Beta")
+        awaitCrudText("Beta")
         compose.onNodeWithText("Beta").assertIsDisplayed()
         compose.onAllNodes(rowMatcher).assertCountEquals(2)
 
         compose.onNodeWithContentDescription("Edit Alpha").performClick()
         compose.onNodeWithTag("item-title-input").performTextReplacement("Alpha edited")
         compose.onNodeWithText("Save").performClick()
-        awaitText("Alpha edited")
+        awaitCrudText("Alpha edited")
         compose.onNodeWithText("Alpha edited").assertIsDisplayed()
         compose.onNodeWithTag(firstRowTag).assertIsDisplayed()
 
@@ -116,7 +140,7 @@ class ItemUiTest {
         completed.assertIsOn()
 
         compose.onNodeWithContentDescription("Delete Beta").performClick()
-        awaitText("Items (1)")
+        awaitCrudText("Items (1)")
         compose.onNodeWithTag("item-count").assertTextEquals("Items (1)")
         compose.onAllNodes(rowMatcher).assertCountEquals(1)
         compose.onNodeWithTag(firstRowTag).assertIsDisplayed()
diff --git a/verification/M04.md b/verification/M04.md
index 8f8a5d6..b9558df 100644
--- a/verification/M04.md
+++ b/verification/M04.md
@@ -262,3 +262,118 @@ the main worktree; it was not a test.
 No other repair build/test invocation or production/input change occurred. The single
 unresolved regression is the M01 Compose focus failure. The full suite remains failed;
 main owns the remaining bounded repair decision and any eventual acceptance/tag.
+
+## Repair 2 — attempt 3 overall, final allowed repair
+
+**FAIL / BLOCKED: main's final eleven-method Android suite still fails the M01 focus test.
+Both permitted repairs are exhausted. No M04 tag, further repair or M05 work is claimed.**
+
+Repair START is `da088bd9e3b36f71b9b260887e36961b34d75b07`; the overall M04 base remains
+verified M03 END `f3591d527b40bd00e2a885c4a53d2508bae209fd`. Preflight confirmed that exact
+clean HEAD, branch and peeled M03 tag before edits. The earlier failure ledger is unchanged.
+Raw evidence is `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/android-kotlin/M04/repair2/`.
+`repair1-raw-failure/` preserves the rejected eleven-method result, including XML SHA-256
+`d7959c81efd8bc2f575d45eed1b5db874bb7acf933cc5e858a8aee9b949bee91`, with all copied-file hashes.
+
+### Diagnosis, limits and narrow correction
+
+The single authorized M01-only diagnostic added test-side action labels, the existing
+`awaitText` match configurations, and failure-only tree logging. The app, original actions,
+assertions and wait limits stayed unchanged. It passed; **the original exception was not
+reproduced and its exact failing action/internal focus cause remains unproven**. Every
+observed title match in this run was a committed Text node, not a draft. No second diagnostic
+was run, and the passing diagnostic does not invalidate repair 1's failure.
+
+There is concrete missing synchronization: `accessStorage` publishes rows before awaiting
+`readSavedStatus`, clearing the submitted draft/focus, and setting `busy=false`. The diagnostic
+also logged the rename action starting at 10:55:23.989 before the preceding native IME hide
+completed at 10:55:24.469. Compose idleness alone does not await external work, as described in
+[Android's test synchronization guidance](https://developer.android.com/develop/ui/compose/testing/synchronization).
+The pinned Compose 1.6.8 source archives and inspection commands are retained outside Git;
+no library defect or dependency fix is claimed.
+
+Only M01's existing waits change. Each still has its original 5,000 ms deadline. Title/count
+waits require a non-editable Text match and an enabled, empty, unfocused editor with the
+native IME reported hidden. Completion waits require the original expected checkbox state
+and the same readiness. These predicates observe the app's own completion effects; they do
+not clear focus, hide the keyboard, sleep, retry actions or bypass the UI. All original CRUD
+operations, order, inputs and assertions remain. `M01Ready` logs identify successful barriers.
+The generic `awaitText`, all later UI methods (M03, M04, failed-write and pending-HTTP), the
+migration `Unit` correction and exact full-error matcher remain byte-identical to START.
+
+`before-main-scope.json` verifies that scope, all production/fixture/build/schema/host-test/
+restart/input hashes, and all 140 unchanged app ZIP entry contents. Frozen M04 input SHA-256
+is still `0e8161210879f5a6e07980c37bf744a62721dca9d80d5d34f20be7be63bd05b0`.
+
+### Commands, failures and candidate identity
+
+`G`, `ADB` and the environment retain the exact definitions above. Every recorded invocation
+has its full command, output, timestamps and exit in this repair's `commands.jsonl` and
+matching JSON/log files. This table includes every build/test invocation and failed retry.
+
+| Label | Command | Exit | Actual result |
+| --- | --- | --- | --- |
+| upstream-ui-source-01 / -02 | `curl -fL --connect-timeout 15` for the pinned official Google Maven source archive | 6 / 0 | Sandbox DNS failure, then approved download; no test |
+| diagnostic-build-01 / -02 | `G :app:assembleDebug :app:assembleDebugAndroidTest` | 1 / 0 | Sandbox denied daemon socket before tasks; approved retry built the diagnostic, 20.965s |
+| diagnostic-m01-01 | `ADB shell am instrument -w -r -e class 'com.mobilesystemsevolution.kotlin.ItemUiTest#frozenM01SequenceThroughActualAndroidUi' com.mobilesystemsevolution.kotlin.test/androidx.test.runner.AndroidJUnitRunner` | 0 | One diagnostic only, `OK (1 test)`, 10.512s instrumentation / 11.300s command |
+| fixed-host-build-01 | `G :app:testDebugUnitTest :app:assembleDebug :app:assembleDebugAndroidTest --rerun-tasks` | 0 | 44.371s; all six host tests actually executed and passed; both APKs rebuilt |
+| fixture-contract-01 | `python3 -m unittest discover -s fixture -v` | 0 | Both unchanged HTTP contract tests passed |
+| verify-fixed-scope-01 / -02 | `python3 E/verify_scope.py` | 1 / 0 | External audit-script newline quoting typo, corrected; scope/APK comparison then passed; no app/test change or execution |
+| main-host-android-01 (main-owned) | `G :app:testDebugUnitTest --rerun :app:connectedDebugAndroidTest` | 1 | Host 6/6 pass; Android 10/11 pass, zero skipped, same M01 focus failure; 71.171s suite / 1m27s Gradle |
+
+Other preflight read errors and the no-file AGENTS lookup are recorded in
+`preflight-notes.json`; none was a test execution. Later report-only patch context and
+not-yet-copied evidence lookup errors are recorded in `final-failure-received.json`.
+An overly broad tag-reference listing was
+then narrowed to the Kotlin tag; no other-track source, tests, implementation artifacts or
+reports were opened or used. Fixture-contract servers were local to that completed command.
+No persistent fixture was launched by this repair agent.
+
+Diagnostic app SHA-256 is `96b483645287f90b1c668cb9eee3e4879c4839b0739a3425f7ae4555aeafb70c`;
+its logging-only test APK is `a6010acea789df99425c25aeca2d357d15b6b92f6b73ac26fb7c4964ea25e371`.
+Their full source snapshot is `diagnostic-built-01/`. The final candidate was frozen after
+host checks in `fixed-built-01/`, with all 32 tracked-file hashes and a source archive:
+
+| Artifact | SHA-256 |
+| --- | --- |
+| App APK | `96b483645287f90b1c668cb9eee3e4879c4839b0739a3425f7ae4555aeafb70c` |
+| Test APK | `622a4047195a1e5d5e8e567a2a25d6e0d6d0822f1a4f6d5022575bfed8f8a231` |
+| ItemUiTest.kt | `bee4ddc4c3c8475574a094a8ad2074725e74f6d835b2737893fff88ebc64320e` |
+| Frozen manifest | `81552d87c1f39ef55f630dba23cadbaab5309a6f96b98889612ed687881dfe1e` |
+
+### Coordinated final verification and cleanup
+
+Main explicitly took ownership of the single final full eleven-method Android suite,
+same-APK direct M03 seed and unchanged schema-2 canonical restart, before the final evidence
+commit. `ready-for-main.json` contains exact commands and input/APK/harness hashes.
+The authorized protocol update is saved as `coordinated-final-protocol.md`; it changes
+verification ownership, not frozen specifications, inputs or acceptance. The repair agent
+does not duplicate that final suite. Main reported all six host tests passing and the actual
+eleven-method Android suite failing with ten passes and the same M01 `ActiveParent with no
+focused child` exception. The tested APK hashes exactly match the frozen candidate above.
+This attempted readiness correction did not resolve the regression. The final direct M03
+seed and canonical restart were not run after the failed suite; repair 1's earlier passing
+restart is historical evidence, not a substitute for this attempt's incomplete verification.
+Main's raw result and cleanup evidence is under
+`/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/android-kotlin/M04/main-repair2-01/`.
+The completed raw XML, UTP, per-method logcat, host XML, fixture log, candidate diff and
+cleanup commands were copied unchanged to `main-final-01/`, with per-file hashes.
+`main-final-inspection.json` checks the actual eleven methods, six host tests, exact APK
+identity and failure. Final Android XML SHA-256 is
+`3a147331e9cb6902497315279f614bb1b53cfb6bc682694c8c888895b6769ff8`.
+Its log confirms the new Items (0), Alpha and Beta readiness barriers completed; the exception
+then recurred before the Alpha edited barrier. Thus the failure persists despite those
+observed readiness conditions; its internal cause remains unresolved. No tests are rerun.
+
+Main stopped its fixture PID 75165 / tool session 88663 with Ctrl-C, exit 0. Actual cleanup
+confirmed that PID absent, port 18080 free, the Kotlin app PID absent and network settings
+0/1/1 with active network 125. `cleanup-result.json` reports PASS with its ten raw commands.
+The later user-authorized main workflow-document change does not alter this attempt's
+original `ed7baa…` execution anchor, frozen inputs or exhausted repair budget.
+
+The one diagnostic lease was released after own-app force-stop and an empty PID query
+(expected exit 1). Before/final checks confirmed API34/ARM64, airplane/wifi/mobile 0/1/1
+and active network 125. No OS settings were changed. The direct diagnostic installs occurred
+before that diagnostic; no restart-boundary test was performed under this lease.
+`diagnostic-lease-release.json` and raw device commands preserve cleanup. No M05 work,
+production change, retry framework or other new scope was added.


