## `test(android): repair M04 harness assertions`

diff --git a/TRACK.md b/TRACK.md
index 4eac7b2..ede3cdb 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -79,6 +79,13 @@ test class, and the fixed M04 UI sequence stopped at its offline error-text asse
 The failure ledger and unrun checks are preserved in `verification/M04.md`; a fresh limited
 repair and main's independent verification are required before M04 can be accepted.
 
+Repair 1 corrected only those two test-harness defects. All eleven Android methods now
+execute: the complete M04 path and all six database tests pass, but the unchanged M01 UI
+test fails with Compose's `ActiveParent with no focused child` exception. The same-APK
+direct M03 seed and schema-2 external process restart pass. The full suite remains failed;
+the new focus regression is unresolved, with no suite rerun or further source fix in this
+repair. `verification/M04.md` preserves both attempts and the complete cleanup record.
+
 M04 reproduced M03's missing freshness and successful-refresh-time presentation without
 changing its working cached reads. The ordinary screen still reads Room before any network
 request. **Sync** remains the only refresh trigger; restoring transport does not itself send
diff --git a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemDatabaseTest.kt b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemDatabaseTest.kt
index 474d409..e8ef369 100644
--- a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemDatabaseTest.kt
+++ b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemDatabaseTest.kt
@@ -110,7 +110,7 @@ class ItemDatabaseTest {
     }
 
     @Test
-    fun v1MigrationPreservesItemsAndRefreshTimeSurvivesReopen() = runBlocking {
+    fun v1MigrationPreservesItemsAndRefreshTimeSurvivesReopen(): Unit = runBlocking {
         val seeds = listOf(
             Item("remote-001", "Alpha", false, 1, 1700000000000L),
             Item("remote-002", "Beta", false, 1, 1700000000000L),
diff --git a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt
index fc09a83..b6703f5 100644
--- a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt
+++ b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt
@@ -439,7 +439,8 @@ class ItemUiTest {
             .assertTextEquals("Last successful refresh: ${expected.lastSuccessfulRefreshAt ?: "never"}")
         val error = expected.error
         if (error == null) compose.onNodeWithTag("sync-error").assertDoesNotExist()
-        else compose.onNodeWithTag("sync-error").assertIsDisplayed().assertTextContains(error)
+        else compose.onNodeWithTag("sync-error").assertIsDisplayed()
+            .assertTextEquals("Refresh failed: $error. Saved items retained; remote outcome unconfirmed.")
         compose.onNodeWithTag("storage-error").assertDoesNotExist()
     }
 
diff --git a/verification/M04.md b/verification/M04.md
index c281875..8f8a5d6 100644
--- a/verification/M04.md
+++ b/verification/M04.md
@@ -134,3 +134,131 @@ Freshness is session status, the successful-refresh time is durable, and no age
 automatic reconnect request is introduced. No offline queue, background execution, retry
 framework, mutation identity, conflict machinery, push, native module or iOS work was added.
 M04 acceptance remains unresolved until a fresh repair and main's independent verification.
+
+## Repair 1 — attempt 2 overall
+
+**Overall repair verification: FAIL.** The two authorized harness defects are corrected,
+and the fixed M04 path passes, but the unchanged M01 UI regression failed. No M04 progress
+tag or all-regressions-pass result is claimed.
+
+Repair START is `4e75f6c73676f65720bd6e751d1cef5fd2ea2bb9`; the original M04 START at
+verified M03 END remains `f3591d527b40bd00e2a885c4a53d2508bae209fd`. Raw repair evidence is under
+`/private/tmp/mobile-systems-evolution-ed7baa2/evidence/android-kotlin/M04/repair1/`.
+The initial failure ledger above is unchanged. The existing failed Android XML (SHA-256
+`55076c4c9d6757b3c9be2312bf9ee669cebc28344ba31c2e35b0c6bf610837ee`) was copied to
+`initial-raw-failure/`; the known failures were not rerun before the repair.
+
+### Authorized changes and frozen identity
+
+Only two test statements changed: the migration method explicitly returns `Unit`, and
+`assertM04Status` requires the entire visible error with `assertTextEquals`:
+`Refresh failed: <error>. Saved items retained; remote outcome unconfirmed.`
+The full offline and HTTP 503 messages remain required, alongside the existing displayed
+status, retained rows and previous successful-refresh time. No assertion was removed.
+
+`before-device-scope.json` verifies exactly those two replacements against the captured
+START tree. All production, fixture, build, schema, restart-helper and host-test sources
+are unchanged. Frozen `M04-inputs.json` still has SHA-256
+`0e8161210879f5a6e07980c37bf744a62721dca9d80d5d34f20be7be63bd05b0`.
+No clock, delay, failure counter, mutation sequence, timeout, acceptance or earlier test
+method changed. `javap` confirms all six database test methods compile to `void`.
+
+The rebuilt APKs, captured with all source hashes in `fixed-built-01/`, are:
+
+| APK | SHA-256 |
+| --- | --- |
+| App | `96b483645287f90b1c668cb9eee3e4879c4839b0739a3425f7ae4555aeafb70c` |
+| Tests | `7d130a6f18031927503728ab4630c831f8a003057208ad993926553f505d2334` |
+
+`--rerun-tasks` forced actual host-test execution. The app container hash changed on the
+rebuild, but `app-archive-comparison.json` proves every ZIP entry name and uncompressed
+content hash equals the initial app. Both APK hashes were checked again after the suite;
+the direct seed and canonical restart used those same built files.
+
+### Repair command and failure ledger
+
+`G` retains the exact JDK17/SDK/Gradle-cache environment defined above and means
+`./gradlew --no-daemon --console=plain`. `ADB` below is
+`/opt/homebrew/share/android-commandlinetools/platform-tools/adb -s emulator-5554`.
+`E` is the absolute repair evidence directory given above. Commands are invoked from
+this Kotlin worktree through `E/run_recorded.py LABEL COMMAND...`; complete arguments,
+environment, PID, timestamps, exits and output are in `commands.jsonl` and named files.
+Read-only preflight commands, including early search errors, are separately preserved in
+`preflight-command-ledger.json`.
+
+| Label | Command | Exit | Actual result |
+| --- | --- | --- | --- |
+| fixed-host-build-01 | `G :app:testDebugUnitTest :app:assembleDebug :app:assembleDebugAndroidTest --rerun-tasks` | 0 | 29.802s; all six host tests executed and passed; both APKs built |
+| fixture-contract-01 | `python3 -m unittest discover -s fixture -v` | 0 | Two unchanged real HTTP contract tests passed |
+| verify-junit-signatures-01 | JDK17 `javap -classpath app/build/tmp/kotlin-classes/debugAndroidTest com.mobilesystemsevolution.kotlin.ItemDatabaseTest` | 0 | All six test methods return void |
+| android-suite-01 | `G :app:connectedDebugAndroidTest` | 1 | 53.814s; all eleven methods executed: ten passed, one failed, zero skipped |
+| collect-android-results-01 | `python3 E/collect_results.py android` | 1 | Preserved all XML/UTP/logcat/report files, then rejected the failed suite; no new tests |
+| direct-install-app-01 | `ADB install -r app/build/outputs/apk/debug/app-debug.apk` | 0 | Same built app installed before the seed/death boundary |
+| direct-install-test-01 | `ADB install -r app/build/outputs/apk/androidTest/debug/app-debug-androidTest.apk` | 0 | Same built test APK installed before that boundary |
+| android-direct-seed-01 | `ADB shell am instrument -w -r -e class 'com.mobilesystemsevolution.kotlin.ItemUiTest#frozenM03SequenceUsesHttpRoomAndTwoIndependentInstances' com.mobilesystemsevolution.kotlin.test/androidx.test.runner.AndroidJUnitRunner` | 0 | 16.037s; unchanged M03 sequence passed, `OK (1 test)`, instrumentation code -1 |
+| canonical-restart-01 | `python3 verification/canonical_restart.py --apk app/build/outputs/apk/debug/app-debug.apk --adb /opt/homebrew/share/android-commandlinetools/platform-tools/adb --schema-version 2 --output E/canonical-restart-01` | 0 | 13.371s; exact canonical rows survived real external process replacement |
+
+The six native database tests now actually pass, including v1 migration, retained success
+time after reopen, atomic row/time rollback, unknown-schema preservation, and prior
+round-trip/CRUD/isolated-database cases. The complete M04 UI sequence passes all four logged
+phases: initial success, forced offline before response, separate HTTP 503, and explicit
+reconnection. Failure phases keep both saved rows and success time 1700000200000; the final
+Room/UI/remote state is exactly Remote revised/version 2 and Beta/version 1 with no duplicate.
+The final success time is 1700000202000, with four transport attempts, three delivered
+requests and two success-clock reads. Both complete visible error messages are asserted.
+
+The independent pending-HTTP local-read case, unchanged M03 UI/HTTP/Room convergence, and
+rejected-write draft/list regression also pass. Complete phase semantics, per-method logcat,
+the eleven-method XML and UTP records are in `android-results-01/`.
+
+The sole failed method is `frozenM01SequenceThroughActualAndroidUi`. Its actual exception is
+`java.lang.IllegalArgumentException: ActiveParent with no focused child`, starting in
+`FocusTransactionsKt.requireActiveChild` and proceeding through Compose semantics
+`requestFocus`. That M01 method and production code are unchanged by this repair.
+The failure is unresolved; it is not relabeled as a harmless flake. Its complete logcat
+and XML stack are retained. The existing test emitted no failure-time screenshot or UI
+hierarchy; no replay was performed to obtain one.
+
+Main was notified immediately after this distinct failure. No suite rerun or further source
+repair occurred. Main explicitly required continuing only the already-approved direct seed
+and canonical restart, followed by this failed-suite handoff. The complete eleven-test
+suite ran once; the direct M03-only seed ran once.
+
+### Canonical process restart and cleanup
+
+UTP uninstalls both packages after the suite. The same-APK installs and unchanged direct
+seed therefore completed before the canonical harness began. The direct seed's logcat was
+captured by app UID 10236, starting at the recorded device timestamp; its exact commands
+are in `direct-seed-before-capture-commands.json` and
+`direct-seed-after-capture-commands.json`. Read-only fixture snapshots after the suite and
+after the direct seed both returned HTTP 200 and `getFailures=0`.
+
+The unchanged canonical harness made eighteen recorded adb calls. It captured real schema-2
+SQLite/WAL and visible UI, force-stopped PID 2653, confirmed its absence (expected empty
+`pidof`, exit 1), and relaunched PID 2739. Before and after contain precisely
+device-001/Gamma/false/version 1/1700000100000 and
+remote-001/Alpha synced/true/version 3/1700000102000; remote-002 stays absent.
+There was no install, clear, instrumentation, or database write across that boundary.
+Both screenshots were visually inspected. The owned fixture log remained exactly 11809
+bytes with SHA-256 `e9f602883efa3dc42ea2c3dd56fb46795bf43323b4014417a0ad8d54a1e48a38`
+through the normal launches/restart, confirming no HTTP request in that interval.
+
+One exclusive device lease covered the suite, approved continuation and cleanup; it was
+returned explicitly. The fixture command was `python3 -u fixture/server.py --port 18080`,
+owned PID 46806/tool session 78420. Owner Ctrl-C stopped it with exit 0. Port 18080 had no
+listener afterward. The Kotlin app was force-stopped after verification and its PID query
+was empty (exit 1). No OS setting was changed; both the initial and final device checks
+reported API34/ARM64, airplane/wifi/mobile 0/1/1 and active default network 123.
+
+A preliminary post-suite settings check and the final post-restart check both passed
+(`device-cleanup-state-01` and `-02`); the first preceded main's continuation instruction.
+The sandbox denied one read-only `ps -p 46806 -o pid=,command=` spawn, exit 1. That denial is
+preserved separately and in the ledger; the properly escalated retry returned empty/exit 1,
+confirming the fixture PID was absent. Both no-listener `lsof` queries, the no-match
+AGENTS search, and the intentionally absent PIDs retain their expected exit 1. The initial
+search for shared spec directories from the track worktree exited 2 and was corrected to
+the main worktree; it was not a test.
+
+No other repair build/test invocation or production/input change occurred. The single
+unresolved regression is the M01 Compose focus failure. The full suite remains failed;
+main owns the remaining bounded repair decision and any eventual acceptance/tag.


