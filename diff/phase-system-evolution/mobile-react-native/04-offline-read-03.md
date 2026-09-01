## `docs: record M04 runtime checks and trailer limitation`

diff --git a/verification/M04.md b/verification/M04.md
index 9d7c494..f68a5f6 100644
--- a/verification/M04.md
+++ b/verification/M04.md
@@ -93,9 +93,176 @@ the implementer protocol. Device commands require a fresh exclusive lease.
 | baseline-typecheck-01 | `npm run typecheck`; unchanged application source | 0 |
 | baseline-android-01 | Full unchanged-M03 Android baseline above; limitation reproduced, cleanup verified | 0 |
 | baseline-jest-01 | Desired fresh-state assertion fails on unchanged M03; one failed/six skipped | 1 |
+| typecheck-01 | `npm run typecheck`; implemented candidate | 0 |
+| jest-01 | `npm test -- --runInBand --watchman=false`; 3 suites, all 21 tests passed | 0 |
+| harness-syntax-02 | `python3 -m py_compile scripts/verify_m02.py scripts/verify_m03.py scripts/verify_m04.py` | 0 |
+| build-01 | Both debug APKs; 99 Gradle tasks, 12 executed/87 up-to-date, 22.452 seconds | 0 |
+| android-m04-01 | Fixed M04 plus frozen offline restart; 144.919 seconds, 236 adb commands | 0 |
+| android-clear-01 | `adb -s emulator-5554 shell pm clear com.mse.reactnative` before M01 regression | 0 |
+| android-ui-01 | Unchanged M01 instrumentation; 1 test, 0 failures/errors/skips, 54.951 seconds | 0 |
+| android-m03-01 | M03 HTTP/native two-database/restart regression; 124.188 seconds, 159 adb commands | 0 |
+| android-cleanup-01 | External `cleanup-app.py`; 6 adb commands stop app and verify network restoration | 0 |
+| artifact-audit-01 | External `audit-final.py`; frozen sources, APK, native DBs, visible status, XML and fixture exits verified | 0 |
 
-## Current verification boundary
+`baseline-typecheck-01` ran before the desired-status tests were added; the
+reproduction test commit is intentionally red. The only test-process failure in
+this attempt is the preserved expected baseline Jest failure. There were no
+failed fixed Android runs, fixture changes, or repeated fixed Android attempts.
+The separate metadata audit failure below remains unresolved.
 
-This reproduction commit does not claim a fixed M04 implementation or completed
-verification. Final host/build/Android results and handoff scope are appended
-after implementation. No progress tag or main index record is created here.
+The build's up-to-date tasks are not runtime verification. `android-ui-01` had
+100 Gradle tasks (5 executed, 95 up-to-date), but `connectedDebugAndroidTest`
+actually executed and produced a fresh one-test JUnit XML. It was not a cached
+test result. XML/logcat are retained in `android-ui-results-01/`, with an explicit
+runtime-task/XML audit in `android-ui-result-audit-01.json`. The M01 test source
+and instrumentation APK are unchanged. Existing Node SQLite experimental and
+Hermes global-name warnings remain in raw logs.
+
+## Implementation and repeatable commands
+
+Production startup still reads SQLite without fetching. The UI owns a small
+refresh presentation state, separate from local-save errors: stale on startup
+or local edit, refreshing during an explicit request, fresh after successful
+snapshot commit/read, and error on failed refresh while retaining the old list.
+Fresh describes the last successful check, not continuous remote knowledge.
+
+Schema v2 adds only one last-success timestamp in `sync_metadata`. It commits
+with the Item snapshot in one SQLite transaction. A literal v1 migration test
+preserves all five fields and allocator3, including reopening. A failed metadata
+UPDATE rolls back both rows and time. Unsupported v3 is still rejected without
+deletion; the previous unsupported-v2 test advances because v2 is now supported.
+The Android test clock is a `BuildConfig.DEBUG`-gated initial prop. Only time is
+controlled: Android executes real RN fetch and native SQLite under actual OS
+offline settings. No custom transport or native module substitutes for that path.
+
+Run from the branch root with the protocol runtime:
+
+```sh
+export PATH=/Users/woopinbell/.local/share/fnm/node-versions/v22.22.0/installation/bin:$PATH
+export JAVA_HOME=/opt/homebrew/opt/openjdk@17/libexec/openjdk.jdk/Contents/Home
+export ANDROID_HOME=/opt/homebrew/share/android-commandlinetools
+export ANDROID_SDK_ROOT=/opt/homebrew/share/android-commandlinetools
+export GRADLE_USER_HOME=/private/tmp/mobile-systems-evolution-ed7baa2/gradle-react-native
+export ANDROID_SERIAL=emulator-5554
+/opt/homebrew/bin/npm run typecheck
+/opt/homebrew/bin/npm test -- --runInBand --watchman=false
+cd android
+./gradlew --no-daemon :app:assembleDebug :app:assembleDebugAndroidTest
+```
+
+With an exclusive lease, from the branch root, the exact executed M04 command was
+as follows. On repetition use a new evidence directory; the harness refuses to
+overwrite existing evidence. Its owned fixture starts with the fixed seeds.
+
+```sh
+python3 scripts/verify_m04.py --adb /opt/homebrew/share/android-commandlinetools/platform-tools/adb --serial emulator-5554 --node /Users/woopinbell/.local/share/fnm/node-versions/v22.22.0/installation/bin/node --apk android/app/build/outputs/apk/debug/app-debug.apk --evidence /private/tmp/mobile-systems-evolution-ed7baa2/evidence/react-native/M04/android-m04-01
+```
+
+For the unchanged M01 regression, clear test data with the `android-clear-01`
+command above, then run from `android/`:
+
+```sh
+./gradlew --no-daemon :app:connectedDebugAndroidTest -Pandroid.testInstrumentationRunnerArguments.class=com.mse.reactnative.M01ItemUiTest
+```
+
+Check that this runtime task executes rather than reporting up-to-date; a repeat
+may use Gradle's `--rerun-tasks` to force execution. That option was not needed in
+this attempt. The M03 regression command, from the branch root, was:
+
+```sh
+python3 scripts/verify_m03.py --adb /opt/homebrew/share/android-commandlinetools/platform-tools/adb --serial emulator-5554 --node /Users/woopinbell/.local/share/fnm/node-versions/v22.22.0/installation/bin/node --apk android/app/build/outputs/apk/debug/app-debug.apk --evidence /private/tmp/mobile-systems-evolution-ed7baa2/evidence/react-native/M04/android-m03-01
+```
+
+Each M02/M03 harness differs from START in exactly two numeric values: the strict
+`PRAGMA user_version` assertion and its reported `schema_version` advance from 1
+to 2. All Item, identity, HTTP, sequence, timeout and process-boundary assertions
+are unchanged. The standalone M02 device harness was not rerun here: its existing
+host coverage passed, and real offline/native restart was exercised by M04 while
+the full M03 native restart and convergence regression also passed.
+
+## Actual Android results and cleanup
+
+The fixed M04 native database and visible labels agree at every checkpoint:
+
+| Checkpoint | Rows | Sync status | Last successful refresh |
+|---|---|---|---:|
+| First successful refresh | Exact Alpha/Beta seeds | fresh | 1700000200000 |
+| Offline refresh failure | Same seeds | error; local storage ready | 1700000200000 |
+| HTTP503 refresh failure | Same seeds | error; local storage ready | 1700000200000 |
+| Explicit reconnect | Remote revised v2/1700000201000; Beta v1/1700000000000 | fresh | 1700000202000 |
+| Supplemental offline process restart | Same two reconciled rows | stale | 1700000202000 |
+
+There were exactly three Item HTTP responses: GET200, GET503, GET200. The failed
+offline request did not reach the fixture, and neither initial nor restarted
+startup performed an HTTP request. Item count stayed exactly two, with no local
+mutations or duplicates. UI XML and PNGs show both rows and the separate status
+and time. Visual inspection confirmed the failure and reconciled screenshots have
+no clipped status or missing row.
+
+M04 host PID20067 survived app PID25629 being force-stopped, confirmed absent,
+then relaunched as PID27369. No reset, install or database mutation occurred
+across the supplemental restart boundary. The whole SQLite file after first
+refresh is byte-identical to both failed-refresh copies; the reconciled database
+is byte-identical to its offline-restarted copy. The fixed sequence finished at
+adb command178; the separate restart followed it.
+
+M04 owned fixture PID20068 exited 0 after `Popen.terminate()`/wait. M03 owned
+fixture PID24933 also exited 0; host PID24922 survived its app PID28467 → absent
+→ PID29379 boundary. Its first preserved and independent second native databases
+converged to the exact earlier Gamma/Alpha-synced result with ten HTTP exchanges.
+No M03 connectivity settings changed.
+
+The final cleanup command was:
+
+```sh
+python3 /private/tmp/mobile-systems-evolution-ed7baa2/evidence/react-native/M04/cleanup-app.py
+```
+
+It stopped the RN app, confirmed no PID, checked airplane/Wi-Fi/mobile data were
+the original 0/1/1, and confirmed an active default network. All six adb calls are
+in `final-cleanup-commands.json`; expected exit1 from `pidof` means the process is
+absent, not a failed verification. Both fixture processes and the app were stopped
+before the exclusive lease was released. No device action followed that release.
+
+| Preserved artifact | SHA-256 |
+|---|---|
+| `m04-tested-app.apk` | `befae993950e4d0ee049c80707da4f158acdb8f9601abd721c2da78782c60964` |
+| Unchanged `m04-test-runner.apk` | `7259ba83ec046d34a3e489ebec46ddcf9ce89dd136a934784c0c89669795b7c1` |
+| First-refresh/offline-error/HTTP-error native DB | `67108bdf09c4d86fde0f51ccd51ec9e2a33f01d381564a60287eb439157963b1` |
+| Reconciled/offline-startup native DB | `8a7bd9aab6ec14e39ef57f6f8fc64033e15dc3915fc117cd896043ccf4949ed1` |
+
+`artifact-audit-01.json` passed for all 19 source/test/harness files from the
+successful build, unchanged desired-status tests from the baseline, frozen
+fixture/inputs, the APK, native DBs and actual status XML. APK copies above were
+preserved before main's independent rebuild could replace build outputs.
+
+## Unresolved commit metadata; no history rewrite
+
+Runtime checks passed on the implementation tree
+`9457f0c6258bce865fdef0d97daa9ca9c52a8ef0` (original candidate
+`6ffdc17b50705c3426d5076f0833a80cad779be7`). The final metadata audit found a
+separate defect: the first two M04 commit commands supplied `Thread` and
+`Spec-Revision` as separate `-m` paragraphs. Both exact lines occur once, but the
+blank line means Git's `%(trailers:key=Thread,valueonly)` returns empty. This does
+not satisfy the required parsed trailer block.
+
+The affected commits are:
+
+- `12421f604801f428acd02bd4b339adc0782b0bf0`, tree `3da0763192784d58d5dede884f1a8f5704802ea9`.
+- `6ffdc17b50705c3426d5076f0833a80cad779be7`, tree `9457f0c6258bce865fdef0d97daa9ca9c52a8ef0`.
+
+`metadata-audit-01.json` records **FAIL**, original messages, parsed values and
+tree IDs; both complete original commit objects are preserved alongside it. The
+read-only collection command itself exited 0. No commit was amended, rebased or
+otherwise rewritten. Main instructed this agent to retain the defect and use a
+fresh limited repair for message-only normalization if its independent audit
+confirms the need. The evidence-only handoff commit uses a contiguous trailer
+block, but the earlier two defects remain **UNRESOLVED** at this handoff.
+
+No M04 runtime defect remains known. No queue, durable upload intent, automatic
+retry, background work, conflict/push framework, new dependency or native module
+was introduced. M03's session-only upload and first-pull replacement limitations
+remain. There was no performance benchmark, device matrix expansion or acceptance
+tuning. No progress tag/main index entry was created and no later Thread started.
+M04 must not be tagged verified until main resolves the metadata defect and
+finishes its independent verification.


## `docs: record M04 trailer-only repair`

diff --git a/verification/M04.md b/verification/M04.md
index f68a5f6..5bfeaac 100644
--- a/verification/M04.md
+++ b/verification/M04.md
@@ -266,3 +266,45 @@ remain. There was no performance benchmark, device matrix expansion or acceptanc
 tuning. No progress tag/main index entry was created and no later Thread started.
 M04 must not be tagged verified until main resolves the metadata defect and
 finishes its independent verification.
+
+## Metadata repair1 — attempt 2
+
+The preceding unresolved-metadata section is the preserved attempt-1 handoff.
+Main confirmed that failure and explicitly authorized normalization of only the
+three unverified M04 commits. Repair START was
+`e69dd845b3d9fe8be905f66a807dbce68de82ff9`; the verified M03 base remains
+`8ce636c1aecdb19a098568aa122fd32f239d16a5`.
+
+| Original commit | Normalized commit | Identical tree |
+|---|---|---|
+| `12421f604801f428acd02bd4b339adc0782b0bf0` | `08197c5d018b73e4ab406d894c60b74cfb2f8b21` | `3da0763192784d58d5dede884f1a8f5704802ea9` |
+| `6ffdc17b50705c3426d5076f0833a80cad779be7` | `9f4be8f3652c5989bf129c592c5abc6ef487c140` | `9457f0c6258bce865fdef0d97daa9ca9c52a8ef0` |
+| `e69dd845b3d9fe8be905f66a807dbce68de82ff9` | `d3e313baa74e2f4a5ed8f666a295c51765f8feff` | `1f94d1ca73bfa958033ae46b4555f2cccaf99dca` |
+
+One blank line between the trailers was removed from each of the first two
+messages. The third message is byte-identical; only the necessary parent links
+changed. Every original tree, author and committer header (including timestamps)
+was preserved. No logical commit was squashed, and no verified history was rewritten.
+
+Raw repair evidence is
+`/private/tmp/mobile-systems-evolution-ed7baa2/evidence/react-native/M04/repair1/`.
+Before changing the branch, `*.original.commit`, `*.original.message` and
+`*.original.metadata.json` saved all original bytes and metadata; `old-to-new.json`
+saved the mapping. `pre-ref-validation.json` precedes the exact-old-HEAD
+compare-and-swap update of only `refs/heads/track/react-native`.
+
+`python3 <raw repair evidence>/metadata_repair.py prepare` and `install` both
+exited 0. `normalized-history-audit.json` records PASS for Git-parsed M04/revision
+trailers, exact tree equality, linearity, all 43 tracked-file hashes, unchanged
+main/spec/scenario, and all existing progress tags. `commands.jsonl` records the
+exact Git commands and exits. `discovery-notes.json` retains one failed read of
+a misplaced log path and its successful correction; it was not a runtime test.
+
+The runtime tests above were already run on identical product/test/fixture/harness
+and dependency inputs. All 42 files other than this ledger also match main's
+`main-pretest-source.json`. This repair ran no build, test, fixture server, or
+device command and claims no new runtime result. Its only file change is this
+M04 ledger append. The post-commit command is
+`python3 <raw repair evidence>/metadata_repair.py final-audit`; its full result
+and final SHA are recorded in `final-audit.json`. Main independently checks the
+final history and acceptance before creating any progress tag.
