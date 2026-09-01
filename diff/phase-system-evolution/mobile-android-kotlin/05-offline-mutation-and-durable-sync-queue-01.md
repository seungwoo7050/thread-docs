# M05 — Offline Mutation과 Durable Sync Queue

## `test(android): freeze M05 offline intent-loss baseline`

diff --git a/TRACK.md b/TRACK.md
index 2a05222..2bb878c 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -129,6 +129,21 @@ A separate fixed latch case proves saved rows/time remain visible while explicit
 pending. Input snapshots, baseline hashes, all invocations and boundaries are in
 `verification/M04-inputs.json` and `verification/M04.md`.
 
+## M05 boundary — initial attempt in progress
+
+The frozen baseline performs four real offline UI changes, externally terminates the app,
+and relaunches the ordinary Activity while still offline. M04 preserves the local rows but
+loses three upload intents: its next Sync uploads only Gamma and restores Alpha and Beta.
+`verification/M05-inputs.json` and `verification/M05.md` record the fixed inputs, exact
+source/APK freeze, accepted HTTP operation, native SQLite/PID evidence and cleanup.
+The M05 implementation and independent final verification are pending.
+
+The test build now registers `M05ScenarioInstrumentation`, a small AndroidJUnitRunner
+subclass. Without `m05ExternalController=true` it delegates to the normal suite unchanged.
+That explicit argument runs only the fixed-ID initial UI launcher; the external process
+test intentionally kills it and never calls it again after the death boundary. It is not
+reported as a passing JUnit test. No test-only launcher is present in the app APK.
+
 ## Pinned build
 
 - JDK 17, Gradle 8.7 (wrapper distribution SHA-256 pinned)
@@ -173,7 +188,7 @@ before the process-death boundary:
 "$ANDROID_HOME/platform-tools/adb" -s emulator-5554 install -r app/build/outputs/apk/androidTest/debug/app-debug-androidTest.apk
 "$ANDROID_HOME/platform-tools/adb" -s emulator-5554 shell am instrument -w -r \
   -e class 'com.mobilesystemsevolution.kotlin.ItemUiTest#frozenM03SequenceUsesHttpRoomAndTwoIndependentInstances' \
-  com.mobilesystemsevolution.kotlin.test/androidx.test.runner.AndroidJUnitRunner
+  com.mobilesystemsevolution.kotlin.test/com.mobilesystemsevolution.kotlin.M05ScenarioInstrumentation
 python3 verification/canonical_restart.py --apk app/build/outputs/apk/debug/app-debug.apk \
   --adb "$ANDROID_HOME/platform-tools/adb" --schema-version 2 --output /tmp/kotlin-M03-canonical-restart
 ```
diff --git a/app/build.gradle.kts b/app/build.gradle.kts
index 51140b5..e172ae5 100644
--- a/app/build.gradle.kts
+++ b/app/build.gradle.kts
@@ -14,7 +14,7 @@ android {
         targetSdk = 34
         versionCode = 1
         versionName = "0.1"
-        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
+        testInstrumentationRunner = "com.mobilesystemsevolution.kotlin.M05ScenarioInstrumentation"
     }
 
     buildFeatures { compose = true }
diff --git a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/M05ScenarioInstrumentation.kt b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/M05ScenarioInstrumentation.kt
new file mode 100644
index 0000000..74a9b38
--- /dev/null
+++ b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/M05ScenarioInstrumentation.kt
@@ -0,0 +1,49 @@
+package com.mobilesystemsevolution.kotlin
+
+import android.app.Activity
+import android.content.Intent
+import android.os.Bundle
+import androidx.activity.compose.setContent
+import androidx.compose.material3.MaterialTheme
+import androidx.test.runner.AndroidJUnitRunner
+import java.util.concurrent.CountDownLatch
+import java.util.concurrent.TimeUnit
+
+/** Initial fixed-ID launcher only; the external controller performs every real UI action. */
+class M05ScenarioInstrumentation : AndroidJUnitRunner() {
+    private var externalController = false
+
+    override fun onCreate(arguments: Bundle?) {
+        externalController = arguments?.getString("m05ExternalController") == "true"
+        super.onCreate(arguments)
+    }
+
+    override fun onStart() {
+        if (!externalController) {
+            super.onStart()
+            return
+        }
+        var database: ItemDatabase? = null
+        try {
+            val activity = startActivitySync(
+                Intent(targetContext, MainActivity::class.java).addFlags(Intent.FLAG_ACTIVITY_NEW_TASK),
+            ) as MainActivity
+            val opened = ItemDatabase.open(targetContext)
+            database = opened
+            var localTime = 1_700_000_300_000L
+            val store = ItemStore(opened.items(), nextId = { "device-001" }, now = { localTime.also { localTime += 1_000 } })
+            val sync = ItemSync(store, HttpItemRemote(), now = { 1_700_000_300_000L })
+            runOnMainSync { activity.setContent { MaterialTheme { ItemScreen(store, sync) } } }
+            waitForIdleSync()
+            sendStatus(1, Bundle().apply { putString("m05", "ready-for-external-ui-controller") })
+            // This is not a JUnit test. The required force-stop intentionally terminates it.
+            CountDownLatch(1).await(300, TimeUnit.SECONDS)
+            finish(Activity.RESULT_CANCELED, Bundle().apply { putString("error", "External controller did not terminate the app within 300 seconds") })
+        } catch (error: Exception) {
+            error.printStackTrace()
+            finish(Activity.RESULT_CANCELED, Bundle().apply { putString("error", error.toString()) })
+        } finally {
+            database?.close()
+        }
+    }
+}
diff --git a/verification/M05-inputs.json b/verification/M05-inputs.json
new file mode 100644
index 0000000..c9b5808
--- /dev/null
+++ b/verification/M05-inputs.json
@@ -0,0 +1,38 @@
+{
+  "thread": "M05",
+  "profile": "phase-1",
+  "specRevision": "61280dd86ce88b6e431f408241c0998a275960aa",
+  "start": "a53adb9381044303d72340dddd34c1b3db820c56",
+  "seed": [
+    {"id": "remote-001", "title": "Alpha", "completed": false, "version": 1, "updatedAt": 1700000000000},
+    {"id": "remote-002", "title": "Beta", "completed": false, "version": 1, "updatedAt": 1700000000000}
+  ],
+  "nextRemoteMutationTimestamp": 1700000300000,
+  "remoteTimestampStep": 1000,
+  "initialLocalTimestamp": 1700000300000,
+  "localTimestampStep": 1000,
+  "initialSuccessfulRefreshTimestamp": 1700000300000,
+  "operations": [
+    {"sequence": 1, "itemId": "device-001", "operation": "CREATE", "title": "Gamma", "completed": false},
+    {"sequence": 2, "itemId": "remote-001", "operation": "RENAME", "title": "Queued edit", "completed": null},
+    {"sequence": 3, "itemId": "remote-001", "operation": "COMPLETE", "title": null, "completed": true},
+    {"sequence": 4, "itemId": "remote-002", "operation": "DELETE", "title": null, "completed": null}
+  ],
+  "offline": "Disable Wi-Fi and mobile data before the first mutation; require no active default network; preserve offline state through external process death and plain Activity relaunch.",
+  "afterRelaunch": "Restore the original network and press Sync exactly once.",
+  "transport": {"fixturePort": 18080, "delayMs": 0, "getFailures": 0, "httpCallTimeoutSeconds": 10},
+  "harness": {"uiWaitSeconds": 30, "adbCommandTimeoutSeconds": 45, "networkWaitSeconds": 30, "initialLauncherTimeoutSeconds": 300, "launcherTerminationWaitSeconds": 15},
+  "expectedFinal": [
+    {"id": "device-001", "title": "Gamma", "completed": false, "version": 1, "updatedAt": 1700000300000},
+    {"id": "remote-001", "title": "Queued edit", "completed": true, "version": 3, "updatedAt": 1700000302000}
+  ],
+  "baseline": "Unchanged M04 retains local rows across process death but has no pending-intent table. One explicit post-restart Sync uploads Gamma only; unchanged remote Alpha and Beta replace the lost edit/completion/deletion intent.",
+  "supportingCoverage": [
+    "Native Room reopen retains every ordered payload and all five Item fields.",
+    "A rejecting pending-insert trigger rolls back each of the four local mutation types; an Item insert failure adds no pending intent.",
+    "Schema-2 migration preserves canonical/local rows and refresh time and queues only existing version-zero creations in rowid order.",
+    "Host new-session drain uses four ordered persisted payloads and yields the exact canonical rows; remote failure retains unacknowledged work.",
+    "Existing M01-M04 host and Android regression cases remain required."
+  ],
+  "scopeLimits": "No mutation identity, baseVersion, crash-safe remote acknowledgment, background work, automatic retry, coalescing, conflict or push support."
+}
diff --git a/verification/M05.md b/verification/M05.md
new file mode 100644
index 0000000..de14c3e
--- /dev/null
+++ b/verification/M05.md
@@ -0,0 +1,74 @@
+# M05 verification — phase-1, initial attempt 1
+
+START: `a53adb9381044303d72340dddd34c1b3db820c56` (verified phase-1 M04).
+Branch: `track/android-kotlin`. SPEC_REVISION: `61280dd86ce88b6e431f408241c0998a275960aa`.
+Baseline reproduction passed; M05 implementation/final verification is still pending.
+No repair has been requested or used for this new Thread; the maximum remains two.
+
+Raw evidence (`E`): `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/android-kotlin/M05/`.
+`preflight.json` and `start-source.tar` preserve the clean predecessor and all 32 tracked files.
+The shared scenario and `verification/M05-inputs.json` were frozen before device execution.
+Input SHA-256: `c49681a03970e40518614606932367b300a8651ed9adac7b906fc93dba2c2775`.
+
+## Frozen baseline
+
+The four actual UI operations are create device-001/Gamma, rename remote-001 to Queued edit,
+complete remote-001, and delete remote-002. The next remote timestamp is 1700000300000,
+advancing by 1000 per accepted mutation. Wi-Fi and mobile data are disabled before the
+first mutation, with no active default network required through process death/relaunch.
+The initial launcher injects only the fixed ID and clocks into real production ItemStore,
+ItemSync and ItemScreen. All mutations are external UI taps/text input; none is a seed write.
+The normal Activity alone relaunches after force-stop, using its saved database.
+
+`baseline-frozen-01/manifest.json` has SHA-256
+`e01f7e9f98831522226845997e609f1f04073036be02956d50041e78f4964f62` and preserves 35 source files,
+their archive/copies, both APKs, generated test manifest and exact command arrays.
+The baseline app is byte-identical to verified M04:
+`f86e8e0b0875450055139e03d892dbf21c3471b00ca89b97c176c45b6c9d3ae6`.
+The test APK is `d16430aa0cc1479db47a36e916dea8954c0af725a31b6709c0b3a05b2cc28d56`.
+All prior tracked files remain unchanged except one test-runner registration setting.
+The runner delegates ordinary suites to the existing AndroidJUnitRunner; only the explicit
+`m05ExternalController=true` argument starts the parked initial launcher. Its termination
+is not a passing JUnit test. A read-only review found no concrete controller blocker.
+
+## Invocations and setup findings
+
+The build environment remains JDK17, the existing Android SDK and Gradle cache. Exact
+environment, arguments, timestamps, PIDs, exits and stdout/stderr are in `commands.jsonl`
+and each named JSON/log. `G` means `./gradlew --no-daemon --console=plain`.
+
+| Invocation | Exit | Actual result |
+| --- | --- | --- |
+| baseline-build-01, `G :app:assembleDebugAndroidTest` | 0 | Test-only launcher compiled; no tests/device calls |
+| Static generated-manifest validation | 1 | AGP had replaced the standalone launcher entry with the configured AndroidJUnitRunner; no runtime attempt |
+| baseline-build-02, same command | 0 | Registered a minimal AndroidJUnitRunner subclass in the test build setting; generated manifest validated |
+| freeze-baseline-01 | 0 | All preexisting product bytes and verified app APK unchanged; exact inputs/harness/APKs frozen |
+| baseline-controller-01 | 0 | One actual baseline via frozen `offline_queue_restart.py --expect limitation --schema-version 2`; 110 recorded adb calls |
+
+The first build's source, manifest and APK remain in `baseline-build-01-artifacts/` with the
+static assertion failure and reason. Two mistyped read-only evidence/manifest paths were
+corrected using file discovery; they were not build/test invocations. No runtime failure,
+retry, timeout relaxation, action retry, assertion removal or product change occurred.
+`ready-for-baseline-lease.json` records the exact baseline and fixture wrapper arguments.
+
+## Observed M04 limitation and cleanup
+
+The single leased baseline actually completed all four UI mutations. Before and after
+offline PID **10234 → absent → 10790**, raw native SQLite/WAL captures show exactly
+Queued edit/true/version1/1700000302000 and Gamma/false/version0/1700000300000,
+Beta absent, schema2 and unchanged saved refresh time. No pending-intent table exists.
+The initial controller reports `Process crashed`/instrumentation code0 after the required
+force-stop; this expected termination is recorded separately, with **zero JUnit suites**.
+There is no install, clear, seed, instrumentation or native write after that boundary.
+
+After restoring transport, one explicit Sync accepts exactly one POST: Gamma/version1 at
+1700000300000. It then restores unchanged Alpha/false/version1 and Beta/false/version1.
+Thus the local edits survived storage/process loss, but rename/completion/deletion upload
+intent did not. The raw fixture log preserves the accepted operation and next timestamp
+1700000301000. Before-stop and after-drain screenshots were visually checked.
+
+`baseline-lease-release.json` verifies the actual app is absent, network settings restored
+to 0/1/1 with active default network105, all frozen source/APK hashes unchanged, and the
+owned fixture PID52377 stopped with SIGINT (tool session84904, exit0, PID absent, port18080
+free). The lease was released. Main owns the eventual final Android suite/restart/drain;
+the implementer will not duplicate it.
diff --git a/verification/offline_queue_restart.py b/verification/offline_queue_restart.py
new file mode 100644
index 0000000..3fd4e23
--- /dev/null
+++ b/verification/offline_queue_restart.py
@@ -0,0 +1,318 @@
+#!/usr/bin/env python3
+"""Frozen M05 actual UI edits, external process loss, and one foreground drain.
+
+Requires the exclusive shared-emulator lease and an owned fixture on port 18080.
+The initial instrumentation supplies only fixed ID/clock constructors. After the
+force-stop, plain MainActivity reads production storage with no test injection.
+"""
+
+import argparse
+import hashlib
+import json
+import os
+from pathlib import Path
+import re
+import shlex
+import shutil
+import sqlite3
+import subprocess
+import tempfile
+import time
+import urllib.request
+
+from process_restart import DB_NAME, PACKAGE, SERIAL, Scenario
+
+
+INPUTS_PATH = Path(__file__).with_name("M05-inputs.json")
+INPUTS = json.loads(INPUTS_PATH.read_text())
+LAUNCHER = f"{PACKAGE}.test/{PACKAGE}.M05ScenarioInstrumentation"
+
+
+class OfflineQueueScenario(Scenario):
+    def __init__(self, args):
+        super().__init__(args)
+        self.result.update(
+            scenario="M05", profile="phase-1",
+            captureHelperSha256=self.result["harnessSha256"],
+            harnessSha256=hashlib.sha256(Path(__file__).read_bytes()).hexdigest(),
+            inputsSha256=hashlib.sha256(INPUTS_PATH.read_bytes()).hexdigest(),
+            testApkSha256=hashlib.sha256(args.test_apk.read_bytes()).hexdigest(),
+        )
+        self.launcher = None
+        self.launcher_files = []
+        self.network_changed = False
+        self.fixture_offset = args.fixture_log.stat().st_size
+
+    def http(self, path, body=None):
+        data = None if body is None else json.dumps(body).encode()
+        request = urllib.request.Request(
+            f"http://127.0.0.1:18080{path}", data=data,
+            headers={"Content-Type": "application/json"},
+        )
+        with urllib.request.urlopen(request, timeout=10) as response:
+            value = json.load(response)
+            record = {"method": request.get_method(), "path": path, "body": body,
+                      "status": response.status, "response": value}
+        with (self.output / "fixture-control.jsonl").open("a") as log:
+            log.write(json.dumps(record) + "\n")
+        return value
+
+    def network_state(self):
+        settings = {key: self.adb("shell", "settings", "get", "global", key)
+                    for key in ("airplane_mode_on", "wifi_on", "mobile_data")}
+        connectivity = self.adb("shell", "dumpsys", "connectivity")
+        match = re.search(r"Active default network:\s*(\S+)", connectivity)
+        assert match, "Could not observe the active default network"
+        return {**settings, "activeDefaultNetwork": match.group(1)}
+
+    def wait_network(self, online):
+        deadline = time.monotonic() + INPUTS["harness"]["networkWaitSeconds"]
+        while True:
+            state = self.network_state()
+            expected = "1" if online else "0"
+            active = state["activeDefaultNetwork"].isdigit()
+            if (state["airplane_mode_on"] == "0" and state["wifi_on"] == expected
+                    and state["mobile_data"] == expected and active == online):
+                return state
+            if time.monotonic() >= deadline:
+                raise AssertionError(f"Network did not become online={online}: {state}")
+            time.sleep(0.2)
+
+    def go_offline(self):
+        self.network_changed = True
+        self.adb("shell", "svc", "wifi", "disable")
+        self.adb("shell", "svc", "data", "disable")
+        self.result["offlineNetwork"] = self.wait_network(False)
+
+    def restore_network(self):
+        self.adb("shell", "svc", "wifi", "enable")
+        self.adb("shell", "svc", "data", "enable")
+        state = self.wait_network(True)
+        self.network_changed = False
+        return state
+
+    def start_launcher(self):
+        command = [str(self.args.adb), "-s", SERIAL, "shell", "am", "instrument", "-w", "-r",
+                   "-e", "m05ExternalController", "true", LAUNCHER]
+        self.command_count += 1
+        with (self.output / "commands.txt").open("a") as log:
+            log.write(f"{self.command_count:03d} {shlex.join(command)} [parked until external force-stop]\n")
+        self.launcher_files = [(self.output / "launcher.stdout").open("wb"),
+                               (self.output / "launcher.stderr").open("wb")]
+        self.launcher = subprocess.Popen(command, stdout=self.launcher_files[0], stderr=self.launcher_files[1])
+        self.result["initialLauncher"] = {"command": command, "hostPid": self.launcher.pid,
+                                          "purpose": "fixed-ID initial UI only; not a JUnit suite"}
+        deadline = time.monotonic() + INPUTS["harness"]["uiWaitSeconds"]
+        while "ready-for-external-ui-controller" not in (self.output / "launcher.stdout").read_text():
+            if self.launcher.poll() is not None or time.monotonic() >= deadline:
+                raise AssertionError("Initial fixed-ID launcher did not become ready")
+            time.sleep(0.2)
+
+    def record_launcher_termination(self, expected):
+        if self.launcher is None:
+            return
+        code = self.launcher.wait(timeout=INPUTS["harness"]["launcherTerminationWaitSeconds"])
+        for stream in self.launcher_files:
+            stream.close()
+        self.launcher_files = []
+        self.result["initialLauncher"].update(
+            exit=code, expectedControllerTermination=expected,
+            outcome="terminated with the actual app process; no JUnit pass claimed" if expected else "cleanup after scenario failure",
+        )
+        self.launcher = None
+
+    def completed_ui(self, titles, completed_title=None):
+        def ready(tree):
+            if not self.matching(tree, **{"class": "android.widget.EditText", "text": "",
+                                        "enabled": "true", "focused": "false"}):
+                return False
+            if not self.matching(tree, text=f"Items ({len(titles)})"):
+                return False
+            for title in titles:
+                if not self.matching(tree, **{"class": "android.widget.TextView", "text": title}):
+                    return False
+            if completed_title and not self.matching(tree, **{
+                "content-desc": f"Completed: {completed_title}", "checked": "true", "enabled": "true",
+            }):
+                return False
+            return True
+        return self.wait_for(ready, f"committed rows {titles}, completed={completed_title}, ready editor")
+
+    def local_ui(self):
+        tree = self.completed_ui(["Gamma", "Queued edit"], "Queued edit")
+        assert not self.matching(tree, text="Beta")
+        assert not self.matching(tree, text="Alpha")
+        assert self.matching(tree, **{"content-desc": "Completed: Gamma", "checked": "false"})
+        if self.args.expect == "durable":
+            assert self.matching(tree, text="Pending changes: 4"), "Four pending changes not visible"
+        else:
+            assert not any(node.get("text", "").startswith("Pending changes:") for node in tree.iter("node"))
+
+    def capture(self, name):
+        rows = self.snapshot(name)
+        folder = self.output / name
+        with tempfile.TemporaryDirectory(prefix="mse-m05-snapshot-") as temporary:
+            local = Path(temporary)
+            for suffix in ("", "-wal"):
+                if (folder / (DB_NAME + suffix)).exists():
+                    shutil.copyfile(folder / (DB_NAME + suffix), local / (DB_NAME + suffix))
+            with sqlite3.connect(local / DB_NAME) as database:
+                tables = [row[0] for row in database.execute("SELECT name FROM sqlite_master WHERE type='table'")]
+                pending = None
+                if "pending_mutations" in tables:
+                    columns = [row[1] for row in database.execute("PRAGMA table_info(pending_mutations)")]
+                    assert columns == ["sequence", "itemId", "operation", "title", "completed"], columns
+                    pending = [dict(zip(columns, row)) for row in database.execute(
+                        "SELECT sequence,itemId,operation,title,completed FROM pending_mutations ORDER BY sequence"
+                    )]
+                metadata = [list(row) for row in database.execute("SELECT id,lastSuccessfulRefreshAt FROM sync_metadata ORDER BY id")]
+        captured = {"items": rows, "pending": pending, "refreshMetadata": metadata}
+        (folder / "storage.json").write_text(json.dumps(captured, indent=2) + "\n")
+        (self.output / f"{name}.png").write_bytes(self.adb("exec-out", "screencap", "-p", binary=True))
+        return captured
+
+    def assert_local_storage(self, captured):
+        assert sorted(captured["items"], key=lambda item: item["id"]) == [
+            {"id": "device-001", "title": "Gamma", "completed": 0, "version": 0, "updatedAt": 1700000300000},
+            {"id": "remote-001", "title": "Queued edit", "completed": 1, "version": 1, "updatedAt": 1700000302000},
+        ], captured
+        assert captured["refreshMetadata"] == [[1, INPUTS["initialSuccessfulRefreshTimestamp"]]]
+        expected_pending = INPUTS["operations"] if self.args.expect == "durable" else None
+        assert captured["pending"] == expected_pending, captured["pending"]
+
+    def run(self):
+        self.result["initialNetwork"] = self.wait_network(True)
+        self.adb("install", "-r", str(self.args.apk.resolve()))
+        self.adb("install", "-r", str(self.args.test_apk.resolve()))
+        assert self.adb("shell", "pm", "clear", PACKAGE) == "Success"
+        self.http("/__reset", {})
+        self.http("/__control", {"nextTimestamp": INPUTS["nextRemoteMutationTimestamp"]})
+        control = self.http("/__control")
+        assert control["getFailures"] == 0 and control["delayMs"] == 0
+        self.start_launcher()
+        self.completed_ui([])
+        self.tap(text="Sync")
+        self.completed_ui(["Alpha", "Beta"])
+        self.wait_text("Fresh local data")
+        seeded = self.capture("seeded")
+        assert seeded["items"] == INPUTS["seed"]
+        assert seeded["pending"] == ([] if self.args.expect == "durable" else None)
+
+        self.go_offline()
+        self.tap(**{"class": "android.widget.EditText"})
+        self.text("Gamma")
+        self.tap(text="Add")
+        self.completed_ui(["Alpha", "Beta", "Gamma"])
+        self.tap(**{"content-desc": "Edit Alpha"})
+        self.tap(**{"class": "android.widget.EditText"})
+        self.adb("shell", "input", "keyevent", "KEYCODE_MOVE_END")
+        self.adb("shell", "input", "keyevent", *(["KEYCODE_DEL"] * len("Alpha")))
+        self.text("Queued edit")
+        self.tap(text="Save")
+        self.completed_ui(["Queued edit", "Beta", "Gamma"])
+        self.tap(**{"content-desc": "Completed: Queued edit"})
+        self.completed_ui(["Queued edit", "Beta", "Gamma"], "Queued edit")
+        self.tap(**{"content-desc": "Delete Beta"})
+        self.local_ui()
+        before = self.capture("before-stop")
+        self.assert_local_storage(before)
+        assert self.http("/__state")["items"] == INPUTS["seed"], "Offline edits affected remote rows"
+        assert self.http("/__control")["nextTimestamp"] == INPUTS["nextRemoteMutationTimestamp"]
+        self.result["beforeStop"] = before
+
+        old_pid = self.adb("shell", "pidof", PACKAGE)
+        assert old_pid.isdigit(), f"Expected one app PID: {old_pid!r}"
+        assert self.launcher.poll() is None, "Initial launcher ended before the process boundary"
+        self.adb("shell", "am", "force-stop", PACKAGE)
+        assert not self.adb("shell", "pidof", PACKAGE, allow_failure=True), "Old app process remains"
+        self.result["processAbsentAfterForceStop"] = True
+        self.record_launcher_termination(expected=True)
+        self.result["offlineBeforeRelaunch"] = self.wait_network(False)
+        # No installation, clear, seed, native write, or instrumentation after this boundary.
+        self.adb("shell", "am", "start", "-W", "-n", f"{PACKAGE}/.MainActivity")
+        new_pid = self.adb("shell", "pidof", PACKAGE)
+        assert new_pid.isdigit() and old_pid != new_pid, f"PID not replaced: {old_pid} -> {new_pid}"
+        self.result.update(beforePid=int(old_pid), afterPid=int(new_pid))
+        self.local_ui()
+        after = self.capture("after-relaunch")
+        assert after == before, f"Persistent state changed at process boundary: {before} -> {after}"
+        self.result["afterRelaunch"] = after
+        self.result["offlineAfterRelaunch"] = self.wait_network(False)
+        assert self.http("/__state")["items"] == INPUTS["seed"]
+
+        self.result["restoredNetwork"] = self.restore_network()
+        self.tap(text="Sync")
+        self.wait_text("Fresh local data")
+        expected_final = INPUTS["expectedFinal"]
+        if self.args.expect == "limitation":
+            expected_final = [INPUTS["expectedFinal"][0], *INPUTS["seed"]]
+        self.completed_ui([item["title"] for item in expected_final], "Queued edit" if self.args.expect == "durable" else None)
+        final = self.capture("after-drain")
+        assert sorted(final["items"], key=lambda item: item["id"]) == expected_final, final
+        assert final["pending"] == ([] if self.args.expect == "durable" else None)
+        if self.args.expect == "durable":
+            self.wait_text("Pending changes: 0")
+        remote = self.http("/__state")
+        assert remote["items"] == expected_final, remote
+        raw = self.args.fixture_log.read_bytes()[self.fixture_offset:]
+        (self.output / "fixture-window.log").write_bytes(raw)
+        accepted = [json.loads(line) for line in raw.splitlines() if line.startswith(b"{")]
+        accepted = [event for event in accepted if event["method"] in ("POST", "PATCH", "DELETE")
+                    and event["path"].startswith("/items") and event["status"] in (200, 201)]
+        expected_operations = [("POST", "/items", 201)]
+        if self.args.expect == "durable":
+            expected_operations += [("PATCH", "/items/remote-001", 200),
+                                    ("PATCH", "/items/remote-001", 200),
+                                    ("DELETE", "/items/remote-002", 200)]
+        assert [(event["method"], event["path"], event["status"]) for event in accepted] == expected_operations, accepted
+        assert accepted[0]["response"] == {"item": expected_final[0]}
+        if self.args.expect == "durable":
+            renamed = {**INPUTS["expectedFinal"][1], "completed": False, "version": 2, "updatedAt": 1700000301000}
+            assert accepted[1]["response"] == {"item": renamed}
+            assert accepted[2]["response"] == {"item": INPUTS["expectedFinal"][1]}
+            assert accepted[3]["response"] == {"deletedId": "remote-002"}
+        assert self.http("/__control")["nextTimestamp"] == 1700000300000 + 1000 * len(expected_operations)
+        self.result.update(afterDrain=final, remote=remote, acceptedOperations=accepted,
+                           observed="four durable intents drained" if self.args.expect == "durable" else "M04 uploaded Gamma only; rename/completion/deletion intent lost")
+        self.result["status"] = "PASS"
+
+    def cleanup(self):
+        self.adb("shell", "am", "force-stop", PACKAGE)
+        self.record_launcher_termination(expected=False)
+        if self.network_changed:
+            self.restore_network()
+        absent = not self.adb("shell", "pidof", PACKAGE, allow_failure=True)
+        state = self.wait_network(True)
+        self.result["cleanup"] = {"appAbsent": absent, "network": state}
+        assert absent, "App remains after cleanup"
+
+
+def main():
+    parser = argparse.ArgumentParser(description=__doc__)
+    parser.add_argument("--expect", choices=("limitation", "durable"), required=True)
+    parser.add_argument("--apk", type=Path, required=True)
+    parser.add_argument("--test-apk", type=Path, required=True)
+    parser.add_argument("--output", type=Path, required=True)
+    parser.add_argument("--fixture-log", type=Path, required=True)
+    parser.add_argument("--schema-version", type=int, required=True)
+    parser.add_argument("--adb", type=Path, default=Path(os.environ.get("ANDROID_HOME", "")) / "platform-tools/adb")
+    scenario = OfflineQueueScenario(parser.parse_args())
+    try:
+        scenario.run()
+    except Exception as error:
+        scenario.result.update(status="FAIL", error=repr(error))
+        raise
+    finally:
+        try:
+            scenario.cleanup()
+        except Exception as error:
+            scenario.result.update(status="FAIL", cleanupError=repr(error))
+            raise
+        finally:
+            scenario.result["adbCommands"] = scenario.command_count
+            (scenario.output / "result.json").write_text(json.dumps(scenario.result, indent=2) + "\n")
+            print(json.dumps(scenario.result, indent=2), flush=True)
+
+
+if __name__ == "__main__":
+    main()


