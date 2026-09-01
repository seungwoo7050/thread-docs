## `docs(android): record failed M04 initial verification`

diff --git a/TRACK.md b/TRACK.md
index 4ec773c..4eac7b2 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -72,6 +72,40 @@ match this fixture's HTTP/1.0 transport. These options follow the pinned
 Cleartext traffic is permitted only for the local emulator host, following Android's
 [per-domain network security configuration](https://developer.android.com/privacy-and-security/security-config#CleartextTrafficPermitted).
 
+## M04 boundary
+
+**Initial M04 candidate: Android acceptance failed.** The test runner rejected the database
+test class, and the fixed M04 UI sequence stopped at its offline error-text assertion.
+The failure ledger and unrun checks are preserved in `verification/M04.md`; a fresh limited
+repair and main's independent verification are required before M04 can be accepted.
+
+M04 reproduced M03's missing freshness and successful-refresh-time presentation without
+changing its working cached reads. The ordinary screen still reads Room before any network
+request. **Sync** remains the only refresh trigger; restoring transport does not itself send
+a request. The screen now shows stale, refreshing, fresh, or sync-error state and the last
+successful refresh in epoch milliseconds. Fresh means the latest explicit refresh committed
+in this session, not that another client cannot have changed remote data since then.
+A new sync instance starts stale and loads its saved timestamp locally. Local edits also
+mark the view stale. Failed refreshes retain both the saved Items and the previous success
+time; the error is transient session state, not durable upload intent.
+
+Room schema 2 adds only one `sync_metadata` row for `lastSuccessfulRefreshAt`. The v1→v2
+migration leaves the five Item columns and values unchanged. Canonical replacement,
+timestamp update, and readback share one transaction; failures roll back the whole change.
+Both schema exports remain in `app/schemas/`. Unknown schema versions still fail without
+destructive fallback. There is no queue, retry framework, background work, or conflict logic.
+
+The small fixture adds `GET /__control` for request/failure counters and `POST /__control`
+with `getFailures` and/or `nextTimestamp` for fixed test inputs. A GET failure consumes one
+counter entry and returns HTTP 503, with delay always zero. Another test client uses the
+existing PATCH endpoint to change canonical data. Existing M03 payloads remain unchanged.
+The M04 Android test forces only its client transport offline in an instrumentation-only
+OkHttp interceptor before `chain.proceed`; this is not OS connectivity testing. Successful
+and HTTP-error paths use real HTTP, and every displayed Item is read from real Room.
+A separate fixed latch case proves saved rows/time remain visible while explicit HTTP is
+pending. Input snapshots, baseline hashes, all invocations and boundaries are in
+`verification/M04-inputs.json` and `verification/M04.md`.
+
 ## Pinned build
 
 - JDK 17, Gradle 8.7 (wrapper distribution SHA-256 pinned)
@@ -118,7 +152,7 @@ before the process-death boundary:
   -e class 'com.mobilesystemsevolution.kotlin.ItemUiTest#frozenM03SequenceUsesHttpRoomAndTwoIndependentInstances' \
   com.mobilesystemsevolution.kotlin.test/androidx.test.runner.AndroidJUnitRunner
 python3 verification/canonical_restart.py --apk app/build/outputs/apk/debug/app-debug.apk \
-  --adb "$ANDROID_HOME/platform-tools/adb" --output /tmp/kotlin-M03-canonical-restart
+  --adb "$ANDROID_HOME/platform-tools/adb" --schema-version 2 --output /tmp/kotlin-M03-canonical-restart
 ```
 
 The filtered test resets only the fixture and its test databases before the sequence. The
@@ -132,6 +166,7 @@ The separate process-restart harness also requires the lease. Supply a new evide
 ```sh
 python3 verification/process_restart.py --expect durable \
   --apk app/build/outputs/apk/debug/app-debug.apk \
+  --schema-version 2 \
   --adb "$ANDROID_HOME/platform-tools/adb" --output /tmp/kotlin-M02-restart-evidence
 ```
 
diff --git a/verification/M04.md b/verification/M04.md
new file mode 100644
index 0000000..c281875
--- /dev/null
+++ b/verification/M04.md
@@ -0,0 +1,136 @@
+# M04 verification — initial attempt 1
+
+**FAIL: Android acceptance is incomplete. No M04 progress tag is claimed.**
+
+- Branch: `track/android-kotlin`; START: `f3591d527b40bd00e2a885c4a53d2508bae209fd`
+- Spec revision: `ed7baa246ee947c6778e0f84751c9b91cec7abfe`
+- Frozen scenario SHA-256: `905cf29e09f4e1f085e3804f86fbe9284a519d825547dea1906702b8bacffd43`
+- Device: unchanged API 34 Pixel 6 ARM64 Google APIs, `emulator-5554`
+- Raw evidence: `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/android-kotlin/M04/`
+
+## Frozen inputs and baseline
+
+`verification/M04-inputs.json` was frozen before the first baseline execution (SHA-256
+`0e8161210879f5a6e07980c37bf744a62721dca9d80d5d34f20be7be63bd05b0`). It retains the exact
+two M03 seeds, success clocks 1700000200000/1700000202000, one forced transport-offline
+failure, separate GET HTTP 503 with delay zero, and the other client's Remote revised
+mutation at 1700000201000. The supporting native local-read gate, v1 migration, rollback,
+fixed-clock transitions and prior regressions were specified before execution. No local
+mutation occurs in the fixed M04 sequence. No input, failure point or acceptance was tuned.
+
+Before production edits, the clean predecessor commit, all tracked source hashes, source
+archive and app/test APKs were preserved. `baseline-M03.apk` SHA-256 is
+`87df373dfa970fa9402a27acc7c0c7d8b0ff4126fc24c634e07b16e2d84d68f7`, matching main's verified
+M03 build. `baseline-production.diff` is empty. The test-only baseline source/APK and hash
+manifests remain in evidence; its absence assertions are not part of the fixed suite.
+
+The baseline actually loaded both seeds over HTTP into native Room, then forced its test
+client transport offline before `chain.proceed`. Both visible rows and all five persisted
+fields survived; a generic sync error appeared, but freshness and successful-refresh-time
+nodes were absent before and after failure. Logcat records two attempted requests and only
+one delivered HTTP request, matching the fixture log. The filtered test passed by asserting
+this M03 limitation, not by asserting M04 correctness. No OS connectivity change was used.
+
+## Candidate and immutable hashes
+
+The candidate adds STALE/REFRESHING/FRESH/ERROR session state and a Room v2 singleton
+`lastSuccessfulRefreshAt`. Canonical rows, timestamp and readback share one transaction.
+Failed network refreshes do not enter that write. New sync instances read saved rows/time
+locally and start stale; only explicit Sync triggers network. The existing M03 comparison
+algorithm, HTTP `Connection: close`, disabled retries and Item fields are preserved.
+The fixture adds only GET failure/request counters and a deterministic clock control.
+
+Frozen built app SHA-256: `453b57aaba796d7f8e7b62b5eb6d8bbce77f4cf0d2c3b73255a3f71bf476ccb1`.
+Frozen test APK SHA-256: `d21a74b86753440655ac4915f9c0dec38e3fe4aa5e727832d5034a49aaa36a8e`.
+Both APKs are copied outside Git. `fixed-built-manifest.json` preserves every product,
+test, schema, fixture and harness hash (manifest SHA-256
+`561f42d361a13eeb11d06f09ab8b5a6a7bcf884799cd92b14dbfa0bfff3ba239`). All those hashes were
+checked again after the failed suite and still matched. The complete pre-device 31-file
+tree is archived in `candidate-tree.tar` with `candidate-tree-manifest.json` (SHA-256
+`30aaf2f29dbcd1daffe0b0d61378ef8a6a7331f32ff8d04f8e639b33ff6f5cef`). Only reporting changed
+after that device run.
+
+## Commands and all invocations
+
+Run from this worktree. Every command was wrapped by the evidence directory's
+`run_recorded.py LABEL COMMAND...`, preserving arguments, environment, timestamps, PID,
+exit and complete stdout/stderr in `commands.jsonl` and each LABEL's JSON/log files.
+`G` below means `./gradlew --no-daemon --console=plain` with this exact environment:
+
+```sh
+export JAVA_HOME=/opt/homebrew/opt/openjdk@17/libexec/openjdk.jdk/Contents/Home
+export ANDROID_HOME=/opt/homebrew/share/android-commandlinetools
+export ANDROID_SDK_ROOT=/opt/homebrew/share/android-commandlinetools
+export GRADLE_USER_HOME=/private/tmp/mobile-systems-evolution-ed7baa2/gradle-kotlin
+export ANDROID_SERIAL=emulator-5554
+```
+
+| Label | Command | Exit | Actual result |
+| --- | --- | --- | --- |
+| baseline-build-01 | `G :app:assembleDebug :app:assembleDebugAndroidTest` | 0 | 13.8s; unchanged app, test-only baseline APK built; no tests |
+| baseline-android-01 | `G :app:connectedDebugAndroidTest '-Pandroid.testInstrumentationRunnerArguments.class=com.mobilesystemsevolution.kotlin.ItemUiTest#m03BaselineRetainsRowsWithoutRefreshStatusOrTime'` | 0 | 25.4s; one expected-limitation test passed (7.103s) |
+| fixture-contract-01 | `python3 -m unittest discover -s fixture -v` | 0 | Both real-loopback M03/M04 HTTP contract tests passed |
+| fixed-host-build-01 | `G :app:testDebugUnitTest :app:assembleDebug :app:assembleDebugAndroidTest` | 1 | 37.0s; six host tests passed; Android test compilation then failed on a cross-module nullable-property smart cast |
+| fixed-host-build-02 | Same full build command | 0 | 27.5s; test helper uses a local nullable value; host task UP-TO-DATE, zero newly executed host tests |
+| fixed-android-suite-01 | `G :app:connectedDebugAndroidTest` | 1 | 69.5s; six reported outcomes: four passes, two failures; detailed limits below |
+| cleanup-device-01 | `python3 /private/tmp/mobile-systems-evolution-ed7baa2/evidence/android-kotlin/M04/verify_cleanup.py` | 0 | Read-only cleanup check: airplane/wifi/mobile 0/1/1, active default network 117 |
+
+The compile correction changed only a local binding in the assertion helper; its before/
+after source and reason are preserved in `fixed-input-sources.tar`,
+`fixed-ItemUiTest-compile-correction.kt` and `test-compile-correction.json`. Inputs, assertions
+and production code stayed unchanged. Existing kapt unused-option warnings remain recorded.
+An initial no-match AGENTS.md search and both no-listener `lsof` checks exited 1 as expected;
+they were not acceptance executions. No other build/test invocation, diagnostic rerun,
+performance run, device matrix or retry loop occurred.
+
+## Actual Android result and unresolved verification
+
+`fixed-android-suite-01-results/` and `-report/` preserve complete XML, UTP and per-test logcat.
+The suite intended eleven methods but reported only six outcomes because JUnit rejected
+the entire database test class: `v1MigrationPreservesItemsAndRefreshTimeSurvivesReopen()`
+has an expression body ending in `Log.i`, inferring a non-void return. Thus **none of the
+six database methods executed**, including migration, rollback and earlier schema tests.
+
+The fixed M04 UI test passed initial HTTP seed/Room/list/fresh/time assertions, then reached
+the expected stale/error state and previous success time after forced transport failure.
+It failed in `assertM04Status`: the default `assertTextContains("Forced offline")` did not
+match the full visible text `Refresh failed: Forced offline. Saved items retained; remote
+outcome unconfirmed.` The post-failure Room assertions, HTTP 503, remote mutation and final
+reconnection were **not reached on Android**. Host coverage is not a substitute for them.
+
+Four actual Android methods passed: original M01 UI CRUD, retained draft/list on a rejected
+write, the unchanged M03 HTTP/Room/two-instance sequence, and the separate local-first case.
+The latter displayed native saved rows, stale status and saved time with zero HTTP calls;
+while explicit HTTP waited at its test latch, rows/time stayed visible, then one real GET
+completed and the screen became fresh. The latch is only a deterministic test input;
+online transport and persistence are real. No OS offline or process-death claim is made.
+
+Main required a fresh limited repair after this failed suite. No candidate source or input
+changed and no same-agent repair was run. Direct M03 APK installs/reseed and the following
+canonical restart command were prepared but **UNRUN**, because the suite prerequisite failed:
+
+```sh
+python3 verification/canonical_restart.py --apk app/build/outputs/apk/debug/app-debug.apk \
+  --adb /opt/homebrew/share/android-commandlinetools/platform-tools/adb --schema-version 2 \
+  --output /private/tmp/mobile-systems-evolution-ed7baa2/evidence/android-kotlin/M04/canonical-restart-01
+```
+
+The restart helpers only add an explicit expected-schema option, defaulting to legacy v1.
+The existing exact Item equality, external force-stop/PID checks, and prohibition on
+install/clear/database writes across the death boundary are unchanged. TRACK documents the
+same-APK direct instrumentation seed needed after UTP uninstalls packages.
+
+## Cleanup and scope
+
+Both fixture launches were `python3 -u fixture/server.py --port 18080`: baseline PID 15242 /
+tool session 1722 and candidate PID 26072 / session 57163. Both were stopped by their owner
+with Ctrl-C, exit 0; each subsequent `lsof -nP -iTCP:18080 -sTCP:LISTEN` was empty, exit 1.
+Both explicit device leases were released. M04 test cleanup restored its transport flag
+and GET failure counter; the separate gate was released. No device profile or OS network
+setting was changed. Cleanup commands/results are in `device-leases-and-sessions.json` and
+`cleanup-device-commands.json`.
+
+Freshness is session status, the successful-refresh time is durable, and no age timeout or
+automatic reconnect request is introduced. No offline queue, background execution, retry
+framework, mutation identity, conflict machinery, push, native module or iOS work was added.
+M04 acceptance remains unresolved until a fresh repair and main's independent verification.


