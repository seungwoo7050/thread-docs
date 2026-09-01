## `fix(android): close fixture connections for M03 sync`

diff --git a/TRACK.md b/TRACK.md
index f071619..4ec773c 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -38,10 +38,10 @@ replacement, not just a new Activity. Draft/selection restoration is not claimed
 
 ## M03 boundary
 
-Attempt 1 is a candidate with **failed Android M03 acceptance**. Initial canonical pull
-works, but the create-sync step is unresolved and canonical restart is unrun. The complete
-failure ledger and reproducible commands are in `verification/M03.md`; do not treat M03
-as verified until main's bounded repair and independent verification succeed.
+Attempt 1 failed Android M03 acceptance. Repair 1 traced the create-sync failure to reuse
+of the HTTP/1.0 fixture's already closed connection. The client now requests
+`Connection: close` without enabling retries or changing the fixture. The original failures and repair
+verification remain in `verification/M03.md`; main independently verifies before tagging.
 
 The existing local CRUD still writes Room and reads committed rows before updating the UI.
 The new **Sync** button is the only network trigger. `ItemSync` compares those local rows
@@ -66,7 +66,8 @@ for verification only. It binds loopback port 18080, accessible to this emulator
 `10.0.2.2:18080`. No backend database, deployment, authentication or public service exists.
 
 OkHttp 4.12.0 is pinned; calls run on `Dispatchers.IO`, close response bodies, and disable
-automatic connection retries/redirects. These options follow the pinned
+automatic connection retries/redirects. Each request explicitly closes its connection to
+match this fixture's HTTP/1.0 transport. These options follow the pinned
 [OkHttp client API](https://github.com/square/okhttp/blob/parent-4.12.0/okhttp/src/main/kotlin/okhttp3/OkHttpClient.kt).
 Cleartext traffic is permitted only for the local emulator host, following Android's
 [per-domain network security configuration](https://developer.android.com/privacy-and-security/security-config#CleartextTrafficPermitted).
@@ -106,11 +107,16 @@ M03's HTTP/Room/UI convergence and canonical transaction rollback tests. The app
 `com.mobilesystemsevolution.kotlin`; the test package adds `.test`.
 
 To leave exactly the M03 canonical table in the app database and check it across an actual
-external process replacement, run on the same lease:
+external process replacement, run on the same lease. Gradle UTP uninstalls both APKs after
+its suite; this direct invocation retains the app and database. All installation happens
+before the process-death boundary:
 
 ```sh
-ANDROID_SERIAL=emulator-5554 ./gradlew --no-daemon :app:connectedDebugAndroidTest \
-  '-Pandroid.testInstrumentationRunnerArguments.class=com.mobilesystemsevolution.kotlin.ItemUiTest#frozenM03SequenceUsesHttpRoomAndTwoIndependentInstances'
+"$ANDROID_HOME/platform-tools/adb" -s emulator-5554 install -r app/build/outputs/apk/debug/app-debug.apk
+"$ANDROID_HOME/platform-tools/adb" -s emulator-5554 install -r app/build/outputs/apk/androidTest/debug/app-debug-androidTest.apk
+"$ANDROID_HOME/platform-tools/adb" -s emulator-5554 shell am instrument -w -r \
+  -e class 'com.mobilesystemsevolution.kotlin.ItemUiTest#frozenM03SequenceUsesHttpRoomAndTwoIndependentInstances' \
+  com.mobilesystemsevolution.kotlin.test/androidx.test.runner.AndroidJUnitRunner
 python3 verification/canonical_restart.py --apk app/build/outputs/apk/debug/app-debug.apk \
   --adb "$ANDROID_HOME/platform-tools/adb" --output /tmp/kotlin-M03-canonical-restart
 ```
diff --git a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt
index 1eb6b97..943b47e 100644
--- a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt
+++ b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt
@@ -193,7 +193,7 @@ class ItemUiTest {
             fun awaitRoom(expected: List<Item>, timeout: Long) {
                 try {
                     compose.waitUntil(timeout) { runBlocking { first.items() } == expected }
-                } catch (error: Exception) {
+                } catch (error: Throwable) {
                     android.util.Log.e("M03Click", "Room=${runBlocking { first.items() }}; expected=$expected\n${compose.onRoot().printToString()}")
                     throw error
                 }
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/HttpItemRemote.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/HttpItemRemote.kt
index 53d245b..1460c6d 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/HttpItemRemote.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/HttpItemRemote.kt
@@ -45,6 +45,9 @@ class HttpItemRemote(
             val url = base.newBuilder().addPathSegment("items")
                 .apply { if (id != null) addPathSegment(id) }.build()
             val request = Request.Builder().url(url)
+                // This HTTP/1.0 fixture closes each connection; do not pool a closed socket.
+                // Keep retries disabled rather than resending a mutation after an EOF.
+                .header("Connection", "close")
                 .method(method, body?.toString()?.toRequestBody("application/json".toMediaType()))
                 .build()
             client.newCall(request).execute().use { response ->
diff --git a/verification/M03.md b/verification/M03.md
index 2189d2c..5435b01 100644
--- a/verification/M03.md
+++ b/verification/M03.md
@@ -1,6 +1,7 @@
-# M03 verification — attempt 1
+# M03 verification — initial attempt and repair 1
 
-**Handoff status: Android acceptance failed; candidate is not verified complete.**
+**Initial attempt 1 handoff: Android acceptance failed.** The original ledger below is
+retained; bounded repair 1 (attempt 2 overall) is recorded at the end of this document.
 
 - Thread: M03; branch: `track/android-kotlin`
 - START: `8a607ee5a98033c000eaf22ab3e70e8c54d992ab` (verified M02 END)
@@ -169,3 +170,112 @@ read successfully. A no-match `rg` for absent AGENTS.md returned 1 during inspec
 was not a test. There were three failed Android M03 executions and one failed test build;
 none were omitted or relabeled as passes. Main owns bounded repair dispatch, independent
 verification and progress tagging. No M03 progress tag is created by this candidate.
+
+## Repair 1 — attempt 2 overall
+
+**Repair verification: PASS; main's independent verification is still required.**
+
+Repair START is `7e7dd5bc3dcdea70b1708eeaa4bcc3d8d7b905b8`, the clean rejected candidate.
+The original verified M02 START, scenario hash, runtime, inputs, HTTP statuses/payloads,
+timeouts, final five-field tables and all existing Android assertions remain unchanged.
+All new raw evidence is under the existing evidence root's `repair1/` directory.
+
+### Diagnosis before fix
+
+One explicitly leased filtered run added only temporary production trace statements and
+changed the test's diagnostic catch to `Throwable`, rethrowing the original failure.
+It failed at the same create wait. The trace proved that the second enabled Sync click
+entered `change` with `busy=false`, launched its coroutine, read the correct Room diff,
+selected `device-001` for creation, and dispatched POST. OkHttp then raised
+`IOException: unexpected end of stream` in `Http1ExchangeCodec.readResponseHeaders`.
+The sync-error node was visible. The fixture received only reset and the initial GET.
+This rules out the earlier click/IME hypotheses for that execution.
+
+The fixture uses Python's default HTTP/1.0 single-request connections. OkHttp 4.12.0
+adds `Connection: Keep-Alive` by default and marks a connection unavailable for reuse
+when the request or response explicitly says `close`. This is supported by the
+[Python handler documentation](https://docs.python.org/3/library/http.server.html#http.server.BaseHTTPRequestHandler.protocol_version),
+[pinned BridgeInterceptor](https://github.com/square/okhttp/blob/parent-4.12.0/okhttp/src/main/kotlin/okhttp3/internal/http/BridgeInterceptor.kt)
+and [pinned CallServerInterceptor](https://github.com/square/okhttp/blob/parent-4.12.0/okhttp/src/main/kotlin/okhttp3/internal/http/CallServerInterceptor.kt).
+
+A host transport probe then used the exact pinned OkHttp/Okio/Kotlin JARs and two fresh
+instances of the unchanged fixture. Each case issued GET then the fixed Gamma POST,
+with zero delay and retries disabled. The original case reused connection identity
+511473681 and failed with the same EOF (subprocess exit 1, canonical fixture unchanged).
+The explicit-close case opened separate GET/POST connections and returned 200 then the
+exact 201 canonical Gamma response (exit 0). This was a transport prefix diagnostic,
+not a substitute for Android acceptance. Its source, frozen inputs, exact commands,
+JAR hashes and both outputs are in `transport-probe-inputs.json`, `ConnectionProbe.java`,
+`connection_probe.py`, and `connection-probe-results.json`.
+
+### Minimal fix
+
+`HttpItemRemote` now adds the request header `Connection: close`. The fixture, response
+headers/statuses/payloads, synchronization algorithm, Room schema and timeout stay unchanged.
+Retries remain disabled. All temporary production traces were removed; only the test's
+failure-reporting catch remains so assertion failures also preserve Room/semantics evidence.
+The existing M03 Android sequence is the regression for this exact transport failure.
+
+### Repair invocation ledger
+
+Commands use the fixed environment already listed above. `repair1/run_recorded.py` records
+the full argument list, timestamps, process ID, exit and complete output in `commands.jsonl`
+and the named JSON/log files. Each Android action requires main's explicit device lease.
+
+| Invocation | Command | Exit | Actual result / evidence |
+| --- | --- | --- | --- |
+| Diagnostic build 1 | `./gradlew --no-daemon --console=plain :app:assembleDebug :app:assembleDebugAndroidTest` | 0 | 17s; APKs rebuilt, no tests; `diagnostic-build-01.log` |
+| Diagnostic Android 1 | `./gradlew --no-daemon --console=plain :app:connectedDebugAndroidTest '-Pandroid.testInstrumentationRunnerArguments.class=com.mobilesystemsevolution.kotlin.ItemUiTest#frozenM03SequenceUsesHttpRoomAndTwoIndependentInstances'` | 1 | 32s; one executed/failed at create; `android-diagnostic-01-results/`; immutable source/APK manifests |
+| Host connection probe 1 | `python3 repair1/connection_probe.py` (absolute evidence path) | 0 | JDK17 javac succeeded; original Java case exit 1 with EOF; explicit-close case exit 0 with exact 201; no Android action |
+| Fixed host/build 1 | `./gradlew --no-daemon --console=plain :app:testDebugUnitTest :app:assembleDebug :app:assembleDebugAndroidTest` | 0 | 22s; five tests actually executed/passed, app APK rebuilt; test APK packaging UP-TO-DATE; `fixed-host-build-01.log`, `fixed-host-01-TEST-*.xml` |
+| Fixture contract 1 | `python3 -m unittest discover -s fixture -v` | 0 | One real-loopback contract test executed/passed; `fixture-contract-01.log` |
+| Fixed Android suite 1 | `./gradlew --no-daemon --console=plain :app:connectedDebugAndroidTest` | 0 | 45s; eight tests executed/passed, including exact UI/HTTP/Room/two-instance convergence; all APK build tasks UP-TO-DATE; `android-fixed-suite-01-results/` |
+| Post-suite database precheck 1 | `python3 repair1/capture_current_db.py` (absolute evidence path) | 1 | Read-only precheck failed before any database read or restart: UTP had uninstalled the app, so `run-as` reported unknown package; `full-suite-post-db/` |
+| Direct seed app install 1 | `adb -s emulator-5554 install -r app/build/outputs/apk/debug/app-debug.apk` | 0 | Same fixed APK; setup before the process-death boundary; `direct-install-app-01.log` |
+| Direct seed test install 1 | `adb -s emulator-5554 install -r app/build/outputs/apk/androidTest/debug/app-debug-androidTest.apk` | 0 | Same fixed test APK; `direct-install-test-01.log` |
+| Direct Android seed 1 | `adb -s emulator-5554 shell am instrument -w -r -e class 'com.mobilesystemsevolution.kotlin.ItemUiTest#frozenM03SequenceUsesHttpRoomAndTwoIndependentInstances' com.mobilesystemsevolution.kotlin.test/androidx.test.runner.AndroidJUnitRunner` | 0 | Same frozen sequence executed once more: `OK (1 test)`, instrumentation code -1; 8s; `android-direct-seed-01.log` |
+| Canonical process restart 1 | `python3 verification/canonical_restart.py --apk app/build/outputs/apk/debug/app-debug.apk --adb /opt/homebrew/share/android-commandlinetools/platform-tools/adb --output /private/tmp/mobile-systems-evolution-ed7baa2/evidence/android-kotlin/M03/repair1/canonical-restart-01` | 0 | PASS, 9s; exact canonical UI/SQLite table before and after actual process replacement; `canonical-restart-01/result.json` and raw command/UI/DB evidence |
+
+Diagnostic fixture command: `python3 -u fixture/server.py --port 18080`, owned PID 93342 /
+tool session 93297. It was stopped with Ctrl-C, exit 0; a subsequent listen check found
+no port 18080 listener. The diagnostic device lease was released without any emulator
+profile or connectivity changes. `fixture-01-process.json` records the lifecycle.
+
+Fixed app APK SHA-256: `7b28b424926d97df4c01c8ab197cc09159bbd4728880aa7e526486b406bce4bd`.
+Fixed test APK SHA-256: `fa9b2bb4035882f077bc97801fdbd646b6aa6bb866d5c5ca3ade1cda716ee385`.
+The source/APK manifests also preserve all diagnostic hashes and unchanged fixture/harness
+hashes. The fixed suite passed, but UTP's `uninstall_after_test: true` removes both APKs
+after it finishes; the preserved `utp.0.log` confirms both uninstall commands. Consequently
+the earlier Gradle-filtered seed instructions cannot retain a database for canonical restart.
+The read-only precheck failure is retained, not reported as a restart or persistence failure.
+
+Main then explicitly approved installing the same APKs and directly invoking the unchanged
+filtered test to retain its database. The full `adb` path for all commands above is
+`/opt/homebrew/share/android-commandlinetools/platform-tools/adb`. This setup and test
+complete before the unchanged restart harness starts. `TRACK.md` now documents this
+working seeding command; no Gradle option, fixture input or test assertion was altered.
+
+The full suite passed all eight tests, including original local CRUD, write-error handling,
+schema preservation/reopen, atomic canonical replacement, isolated-database reproduction,
+and the exact M03 sequence. The direct seed independently passed that same M03 sequence.
+Both runs checked first/second Room databases and remote against the frozen final table.
+Canonical restart then read that exact table in the normal UI, captured raw SQLite/WAL,
+force-stopped PID 21414, confirmed its absence (expected `pidof` exit 1), and relaunched
+PID 21490. All five fields were equal before and after; Beta stayed absent. Its 18-command
+ledger contains no install, clear, instrumentation or database mutation. The final screenshot
+also shows precisely Gamma incomplete and Alpha synced completed.
+
+The second owned fixture process was PID 96741 / tool session 10830. It was stopped with
+Ctrl-C, exit 0; no listener remained on port 18080. The fixture log did not change during
+normal launch/process restart, proving that this persistence check used local saved state.
+`fixture-02-process.json` records that check. Both device leases were released and no
+emulator profile, network, permissions or connectivity settings were changed.
+
+Repair failure ledger remains explicit: the logging-only Android diagnostic failed once;
+the original Java transport case failed once as the recorded reproduction; and the
+read-only post-suite precheck failed once because UTP removed the package. The fixed
+host tests, fixture test, full Android suite, direct filtered seed and real canonical
+restart all passed. No other repair build/test invocation occurred. The original attempt's
+three failed M03 runs and one test-compilation failure remain recorded above.
+There are no unresolved M03 defects in this repair handoff. Future durable upload/retry,
+idempotency, conflict and lifecycle/background guarantees remain outside the M03 boundary.
