# M08 — Foreground, Background와 State Restoration

## `test(M08): freeze actual Activity state baseline`

diff --git a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/M08ActivityStateTest.kt b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/M08ActivityStateTest.kt
new file mode 100644
index 0000000..6495dc5
--- /dev/null
+++ b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/M08ActivityStateTest.kt
@@ -0,0 +1,262 @@
+package com.mobilesystemsevolution.kotlin
+
+import android.app.Activity
+import android.app.Application
+import android.database.Cursor
+import android.os.Bundle
+import android.os.Process
+import android.util.Log
+import androidx.compose.ui.semantics.SemanticsProperties
+import androidx.compose.ui.semantics.getOrNull
+import androidx.compose.ui.test.assertIsDisplayed
+import androidx.compose.ui.test.assertIsEnabled
+import androidx.compose.ui.test.junit4.createAndroidComposeRule
+import androidx.compose.ui.test.onAllNodesWithTag
+import androidx.compose.ui.test.onAllNodesWithText
+import androidx.compose.ui.test.onNodeWithContentDescription
+import androidx.compose.ui.test.onNodeWithTag
+import androidx.compose.ui.test.onNodeWithText
+import androidx.compose.ui.test.onRoot
+import androidx.compose.ui.test.performClick
+import androidx.compose.ui.test.performScrollTo
+import androidx.compose.ui.test.performTextReplacement
+import androidx.compose.ui.test.printToString
+import androidx.lifecycle.Lifecycle
+import androidx.test.ext.junit.runners.AndroidJUnit4
+import androidx.test.platform.app.InstrumentationRegistry
+import java.io.File
+import kotlinx.coroutines.runBlocking
+import org.json.JSONArray
+import org.json.JSONObject
+import org.junit.Assert.assertEquals
+import org.junit.Assert.assertFalse
+import org.junit.Assert.assertNotSame
+import org.junit.Assert.assertSame
+import org.junit.Assert.assertTrue
+import org.junit.Rule
+import org.junit.Test
+import org.junit.rules.ExternalResource
+import org.junit.rules.RuleChain
+import org.junit.runner.RunWith
+
+/** Own pre-launch seed and lifecycle observer; existing CRUD test setup stays unchanged. */
+@RunWith(AndroidJUnit4::class)
+class M08ActivityStateTest {
+    private val instrumentation = InstrumentationRegistry.getInstrumentation()
+    private val compose = createAndroidComposeRule<MainActivity>()
+    private val seed = Item("ui-001", "Saved title", false, 1, 1700000000000L)
+    private val draft = "Unsubmitted draft"
+    private val expectation = InstrumentationRegistry.getArguments().getString("m08Expectation", "restored")
+    private val events = mutableListOf<JSONObject>()
+    private val checkpoints = JSONArray()
+    private val report = JSONObject().put("scenario", "M08").put("expectation", expectation)
+        .put("status", "RUNNING").put("backgroundCycles", 0).put("recreations", 0)
+        .put("explicitSaves", 0).put("syncButtonPresses", 0)
+    private lateinit var application: Application
+    private lateinit var database: ItemDatabase
+    private lateinit var store: ItemStore
+    private lateinit var output: File
+
+    private fun event(name: String, activity: Activity) {
+        if (activity !is MainActivity) return
+        val value = JSONObject().put("event", name).put("pid", Process.myPid())
+            .put("application", System.identityHashCode(activity.application))
+            .put("activity", System.identityHashCode(activity))
+            .put("changingConfigurations", activity.isChangingConfigurations)
+        synchronized(events) { events.add(value) }
+        Log.i("M08Lifecycle", value.toString())
+    }
+
+    private val callbacks = object : Application.ActivityLifecycleCallbacks {
+        override fun onActivityCreated(activity: Activity, state: Bundle?) = event("created", activity)
+        override fun onActivityStarted(activity: Activity) = event("started", activity)
+        override fun onActivityResumed(activity: Activity) = event("resumed", activity)
+        override fun onActivityPaused(activity: Activity) = event("paused", activity)
+        override fun onActivityStopped(activity: Activity) = event("stopped", activity)
+        override fun onActivitySaveInstanceState(activity: Activity, state: Bundle) = event("saved", activity)
+        override fun onActivityDestroyed(activity: Activity) = event("destroyed", activity)
+    }
+
+    @get:Rule
+    val rules: RuleChain = RuleChain.outerRule(object : ExternalResource() {
+        override fun before() {
+            require(expectation in setOf("limitation", "restored"))
+            val context = instrumentation.targetContext
+            output = File(context.filesDir, "m08-$expectation-01")
+            check(!output.exists()) { "Refusing to overwrite M08 evidence" }
+            check(output.mkdirs())
+            context.deleteDatabase(ItemDatabase.NAME)
+            database = ItemDatabase.open(context)
+            store = ItemStore(database.items())
+            runBlocking { database.items().insert(seed.toEntity()) }
+            application = context.applicationContext as Application
+            instrumentation.runOnMainSync { application.registerActivityLifecycleCallbacks(callbacks) }
+            writeReport()
+        }
+
+        override fun after() {
+            if (::application.isInitialized) {
+                instrumentation.runOnMainSync { application.unregisterActivityLifecycleCallbacks(callbacks) }
+                report.put("testObserverRemoved", true)
+            }
+            if (::database.isInitialized) database.close()
+            if (::output.isInitialized) writeReport()
+        }
+    }).around(compose)
+
+    private fun writeReport() {
+        report.put("events", synchronized(events) { JSONArray(events.toList()) })
+            .put("checkpoints", checkpoints)
+        File(output, "result.json").writeText(report.toString(2) + "\n")
+    }
+
+    private fun awaitEditor(text: String, editing: Boolean) {
+        compose.waitUntil(5_000) {
+            val field = compose.onAllNodesWithTag("item-title-input").fetchSemanticsNodes().singleOrNull()
+            field != null && field.config.getOrNull(SemanticsProperties.EditableText)?.text == text &&
+                field.config.getOrNull(SemanticsProperties.Disabled) == null &&
+                compose.onAllNodesWithText(if (editing) "Save" else "Add").fetchSemanticsNodes().size == 1
+        }
+        compose.onNodeWithTag("item-title-input").performScrollTo().assertIsDisplayed().assertIsEnabled()
+        assertEquals(text, compose.onNodeWithTag("item-title-input").fetchSemanticsNode()
+            .config[SemanticsProperties.EditableText].text)
+        compose.onNodeWithTag("item-row-ui-001").performScrollTo().assertIsDisplayed()
+        if (editing) {
+            compose.onNodeWithText("Save").performScrollTo().assertIsDisplayed().assertIsEnabled()
+            compose.onNodeWithText("Cancel").performScrollTo().assertIsDisplayed()
+        } else {
+            compose.onNodeWithText("Cancel").assertDoesNotExist()
+        }
+    }
+
+    private fun assertUnsubmitted() = runBlocking {
+        assertEquals(listOf(seed), store.items())
+        assertTrue(store.pendingMutations().isEmpty())
+    }
+
+    private fun table(name: String, order: String): JSONArray {
+        val result = JSONArray()
+        database.openHelper.readableDatabase.query("SELECT * FROM $name ORDER BY $order").use { cursor ->
+            while (cursor.moveToNext()) {
+                val row = JSONObject()
+                cursor.columnNames.forEachIndexed { index, column ->
+                    row.put(column, when (cursor.getType(index)) {
+                        Cursor.FIELD_TYPE_NULL -> JSONObject.NULL
+                        Cursor.FIELD_TYPE_INTEGER -> cursor.getLong(index)
+                        Cursor.FIELD_TYPE_FLOAT -> cursor.getDouble(index)
+                        else -> cursor.getString(index)
+                    })
+                }
+                result.put(row)
+            }
+        }
+        return result
+    }
+
+    private fun capture(stage: String, activity: MainActivity, rendered: Boolean = true) {
+        val folder = File(output, stage).apply { check(mkdir()) }
+        val value = JSONObject().put("stage", stage).put("pid", Process.myPid())
+            .put("application", System.identityHashCode(activity.application))
+            .put("activity", System.identityHashCode(activity))
+            .put("items", table("items", "id"))
+            .put("pending", table("pending_mutations", "sequence"))
+            .put("acknowledged", table("acknowledged_mutations", "clientMutationId"))
+            .put("tombstones", table("tombstones", "id"))
+        instrumentation.runOnMainSync { value.put("lifecycle", activity.lifecycle.currentState.name) }
+        if (rendered) {
+            value.put("editorText", compose.onNodeWithTag("item-title-input").fetchSemanticsNode()
+                .config[SemanticsProperties.EditableText].text)
+            value.put("saveVisible", compose.onAllNodesWithText("Save").fetchSemanticsNodes().size == 1)
+            File(folder, "semantics.txt").writeText(compose.onRoot().printToString())
+        }
+        // These checkpoints occur after completed UI/storage operations, with no writer active.
+        for (suffix in listOf("", "-wal")) {
+            val source = instrumentation.targetContext.getDatabasePath(ItemDatabase.NAME + suffix)
+            if (source.exists()) source.copyTo(File(folder, ItemDatabase.NAME + suffix))
+        }
+        File(folder, "state.json").writeText(value.toString(2) + "\n")
+        checkpoints.put(value)
+        Log.i("M08Checkpoint", value.toString())
+        writeReport()
+    }
+
+    @Test
+    fun frozenM08ActivityLifecycleSequence() {
+        try {
+            val scenario = compose.activityRule.scenario
+            val original = compose.activity
+            val originalApplication = original.application
+            val originalPid = Process.myPid()
+            awaitEditor("", editing = false)
+            assertUnsubmitted()
+            capture("seeded", original)
+
+            compose.onNodeWithContentDescription("Edit Saved title").performScrollTo().performClick()
+            compose.onNodeWithTag("item-title-input").performTextReplacement(draft)
+            awaitEditor(draft, editing = true)
+            assertUnsubmitted()
+            capture("draft", original)
+
+            scenario.moveToState(Lifecycle.State.CREATED)
+            assertEquals(Lifecycle.State.CREATED, scenario.state)
+            scenario.onActivity { assertSame(original, it) }
+            assertUnsubmitted()
+            capture("background", original, rendered = false)
+            scenario.moveToState(Lifecycle.State.RESUMED)
+            report.put("backgroundCycles", 1)
+            assertSame(original, compose.activity)
+            assertSame(originalApplication, compose.activity.application)
+            assertEquals(originalPid, Process.myPid())
+            awaitEditor(draft, editing = true)
+            assertUnsubmitted()
+            capture("foreground", compose.activity)
+
+            scenario.recreate()
+            report.put("recreations", 1)
+            val recreated = compose.activity
+            assertNotSame(original, recreated)
+            assertSame(originalApplication, recreated.application)
+            assertEquals(originalPid, Process.myPid())
+            assertEquals(Lifecycle.State.RESUMED, scenario.state)
+            assertTrue(synchronized(events) { events.any {
+                it.getString("event") == "destroyed" && it.getInt("activity") == System.identityHashCode(original)
+            } })
+            assertTrue(synchronized(events) { events.any {
+                it.getString("event") == "created" && it.getInt("activity") == System.identityHashCode(recreated)
+            } })
+            awaitEditor(if (expectation == "restored") draft else "", editing = expectation == "restored")
+            assertUnsubmitted()
+            capture("recreated", recreated)
+
+            if (expectation == "limitation") {
+                report.put("status", "LIMITATION_REPRODUCED")
+                    .put("finalSave", "NOT_RUN_AFTER_REPRODUCED_SELECTION_AND_DRAFT_LOSS")
+            } else {
+                val beforeSave = System.currentTimeMillis()
+                compose.onNodeWithText("Save").performScrollTo().performClick()
+                report.put("explicitSaves", 1)
+                awaitEditor("", editing = false)
+                val afterSave = System.currentTimeMillis()
+                val saved = runBlocking { store.items() }.single()
+                assertEquals(seed.copy(title = draft, updatedAt = saved.updatedAt), saved)
+                assertTrue(saved.updatedAt in beforeSave..afterSave)
+                val intent = runBlocking { store.pendingMutations() }.single()
+                assertEquals("ui-001", intent.itemId)
+                assertEquals("RENAME", intent.operation)
+                assertEquals(draft, intent.title)
+                assertEquals(1L, intent.baseVersion)
+                assertFalse(intent.dispatched)
+                assertEquals(null, intent.terminalError)
+                compose.onNodeWithContentDescription("Edit Unsubmitted draft").performScrollTo().assertIsDisplayed()
+                capture("submitted", recreated)
+                report.put("saveClockBounds", JSONArray(listOf(beforeSave, afterSave)))
+                    .put("status", "PASS")
+            }
+            writeReport()
+        } catch (failure: Throwable) {
+            report.put("status", "FAIL").put("error", failure.stackTraceToString())
+            writeReport()
+            throw failure
+        }
+    }
+}
diff --git a/verification/M08-inputs.json b/verification/M08-inputs.json
new file mode 100644
index 0000000..ffccaf0
--- /dev/null
+++ b/verification/M08-inputs.json
@@ -0,0 +1,21 @@
+{
+  "thread": "M08",
+  "profile": "phase-1",
+  "specRevision": "61280dd86ce88b6e431f408241c0998a275960aa",
+  "start": "d01bcd480f9a6beb0fae759a7c63a093c47b3296",
+  "seed": {"id":"ui-001","title":"Saved title","completed":false,"version":1,"updatedAt":1700000000000},
+  "draft": "Unsubmitted draft",
+  "backgroundCycle": ["ActivityScenario.moveToState(CREATED)", "ActivityScenario.moveToState(RESUMED)"],
+  "backgroundCycleCount": 1,
+  "recreation": "ActivityScenario.recreate() on the instrumentation test thread",
+  "recreationCount": 1,
+  "identity": "Same PID and Application throughout; same Activity across background/foreground; old Activity destroyed and a distinct actual MainActivity resumed after recreation.",
+  "beforeSubmit": "Exact seeded Item, pending0, no HTTP requests. Selection is rendered Save/Cancel mode for the sole ui-001 row, and editable text is Unsubmitted draft.",
+  "baseline": "Unchanged M07 retains selection/draft across background/foreground but loses both after actual Activity recreation: empty editor, Add mode, no Cancel. Reproduce that limitation and leave the final Save unrun; do not re-enter or persist the lost draft.",
+  "fixed": "Selection/draft survives both boundaries. Press Save exactly once through the existing mutation path; the same Item becomes Unsubmitted draft/false/version1 and exactly one unsent RENAME intent has baseVersion1. No Sync action.",
+  "submittedTimestamp": "The existing production clock, bounded by the wall-clock readings immediately around the one explicit Save; no test-only product clock or mutation identity injection.",
+  "harness": {"uiWaitMilliseconds":5000,"adbCommandTimeoutSeconds":45,"instrumentationTimeoutSeconds":360,"preSeedTeardownWaitSeconds":30},
+  "transport": {"fixturePort":18080,"delayMs":0,"expectedApplicationHttpRequests":0},
+  "evidence": "Test-only lifecycle callbacks, rendered Compose semantics, raw native database/WAL snapshots at each checkpoint, and actual PID/Application/Activity identities. No product recreation hook or replacement UI.",
+  "scopeLimits": "No OS process kill recovery, durable draft database field, scheduler, push, new navigation framework, product test hooks, or M09 work."
+}
diff --git a/verification/M08.md b/verification/M08.md
new file mode 100644
index 0000000..970048c
--- /dev/null
+++ b/verification/M08.md
@@ -0,0 +1,34 @@
+# M08 — phase-1 Activity/editor state
+
+Status: unchanged-M07 baseline independently accepted; implementation/final verification pending.
+START `d01bcd480f9a6beb0fae759a7c63a093c47b3296`; SPEC_REVISION
+`61280dd86ce88b6e431f408241c0998a275960aa`; attempt1, repair0/2.
+
+## Frozen baseline
+
+Evidence root: `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/android-kotlin/M08`.
+[Manifest](../../evidence/phase-1/android-kotlin/M08/baseline-frozen-01/manifest.json)
+`b9e447955cff13c82aceb71a05df6eb2dd3c863db31c1834451f4f7e6d423f50` freezes 49 source
+hashes and three support copies. All 46 predecessor files stayed unchanged, including the
+product, existing tests, runner and fixture. Reused M07 app `2ffa4686…a13d`; test APK
+`d59f428e…f683`. The test-only build passed; compiled `frozenM08ActivityLifecycleSequence`
+is public void. No app rebuild, host-suite rerun or fixture-test rerun was performed.
+
+[Command ledger](../../evidence/phase-1/android-kotlin/M08/commands.jsonl): eight invocations.
+Build, static check, freeze, fixture, scenario, stop and audit exited 0. The pre-start port
+check returned expected exit1/empty. No failure or retry occurred.
+[Actual baseline](../../evidence/phase-1/android-kotlin/M08/baseline-android-01/result.json):
+one JUnit test passed at reproducing the limitation, zero failures/skips; 27 adb commands,
+9.959 s total. This is not a fixed-M08 acceptance pass. The one CREATED→RESUMED cycle kept
+Activity187410066 and its selected draft. One actual recreation replaced it with
+Activity44189861 while PID21558/Application88250467 stayed the same. The recreated UI
+showed an empty editor/Add mode, with no Cancel. All five native SQLite/WAL snapshots
+retained exact ui-001/Saved title/false/v1/1700000000000 and pending0. No HTTP request;
+fixture log offset262→262. Save was left unrun after the reproduced loss, not re-entered.
+
+[Byte/cleanup audit](../../evidence/phase-1/android-kotlin/M08/baseline-byte-cleanup-audit.json):
+49 sources, three copies and both APKs unchanged; 76 raw artifacts retained. Owned fixture
+46224 was checked by command/port ownership, stopped with SIGINT, and exited0. Direct ps
+and lsof checks prove PID absent/18080 free. App absent; network0/1/1, active123. Lease released.
+Main independently accepted [the raw baseline](</Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M08/main-baseline-audit.json>).
+Final runtime verification remains main-owned; no further owner device run is authorized.
diff --git a/verification/activity_state.py b/verification/activity_state.py
new file mode 100644
index 0000000..d7fe5d3
--- /dev/null
+++ b/verification/activity_state.py
@@ -0,0 +1,218 @@
+#!/usr/bin/env python3
+"""Run the one frozen M08 actual-Activity lifecycle case on the exclusively leased device.
+
+The test APK seeds before launch and exports its observed UI/lifecycle/native checkpoints.
+Initial clear and final force-stop are setup/cleanup, never part of the lifecycle sequence.
+"""
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
+import tarfile
+import tempfile
+import time
+
+from offline_queue_restart import OfflineQueueScenario, LAUNCHER
+from process_restart import DB_NAME, PACKAGE, SERIAL
+from version_conflict_restart import attached_own_records
+
+INPUTS_PATH = Path(__file__).with_name("M08-inputs.json")
+INPUTS = json.loads(INPUTS_PATH.read_text())
+
+
+class ActivityStateScenario(OfflineQueueScenario):
+    def __init__(self, args):
+        super().__init__(args)
+        self.result.update(scenario="M08", harnessSha256=hashlib.sha256(Path(__file__).read_bytes()).hexdigest(),
+                           inputsSha256=hashlib.sha256(INPUTS_PATH.read_bytes()).hexdigest(),
+                           fixtureLog=str(args.fixture_log.resolve()), fixtureLogStartOffset=self.fixture_offset)
+
+    def teardown_gate(self):
+        deadline = time.monotonic() + INPUTS["harness"]["preSeedTeardownWaitSeconds"]
+        while True:
+            pid = self.adb("shell", "pidof", PACKAGE, allow_failure=True)
+            attached = attached_own_records(self.adb("shell", "dumpsys", "activity", "activities"))
+            if not pid and not attached:
+                return
+            if time.monotonic() >= deadline:
+                raise AssertionError(f"Initial teardown incomplete: pid={pid!r}, attached={attached}")
+            time.sleep(0.2)
+
+    def instrument(self):
+        command = [str(self.args.adb), "-s", SERIAL, "shell", "am", "instrument", "-w", "-r",
+                   "-e", "class", f"{PACKAGE}.M08ActivityStateTest",
+                   "-e", "m08Expectation", self.args.expect, LAUNCHER]
+        self.command_count += 1
+        with (self.output / "commands.txt").open("a") as ledger:
+            ledger.write(f"{self.command_count:03d} {shlex.join(command)}\n")
+        started = time.monotonic()
+        with (self.output / "junit.stdout").open("wb") as stdout, (self.output / "junit.stderr").open("wb") as stderr:
+            process = subprocess.Popen(command, stdout=stdout, stderr=stderr)
+            try:
+                code = process.wait(timeout=INPUTS["harness"]["instrumentationTimeoutSeconds"])
+            except subprocess.TimeoutExpired:
+                process.kill()
+                process.wait()
+                self.result["junit"] = {"status": "TIMEOUT", "command": command, "hostPid": process.pid}
+                raise
+        self.result["junit"] = {"command": command, "hostPid": process.pid, "exit": code,
+                                "elapsedSeconds": round(time.monotonic() - started, 3)}
+        with (self.output / "commands.txt").open("a") as ledger:
+            ledger.write(f"    exit={code}\n")
+        return code, (self.output / "junit.stdout").read_text()
+
+    def export_native(self):
+        folder_name = f"m08-{self.args.expect}-01"
+        self.adb("exec-out", "run-as", PACKAGE, "tar", "-cf", "-", "-C", "files", folder_name, binary=True)
+        archive = self.output / f"command-{self.command_count:03d}.stdout"
+        native = self.output / "native"
+        native.mkdir()
+        with tarfile.open(archive) as source:
+            for member in source.getmembers():
+                path = Path(member.name)
+                assert not path.is_absolute() and ".." not in path.parts and path.parts[0] == folder_name
+                assert member.isdir() or member.isfile(), member.name
+            source.extractall(native, filter="data")
+        return native / folder_name
+
+    def audit_native(self, native):
+        observed = json.loads((native / "result.json").read_text())
+        fixed = self.args.expect == "restored"
+        expected_stages = ["seeded", "draft", "background", "foreground", "recreated"] + (["submitted"] if fixed else [])
+        assert observed["status"] == ("PASS" if fixed else "LIMITATION_REPRODUCED"), observed
+        assert observed["backgroundCycles"] == observed["recreations"] == 1
+        assert observed["explicitSaves"] == int(fixed) and observed["syncButtonPresses"] == 0
+        assert observed["testObserverRemoved"] is True
+        checkpoints = observed["checkpoints"]
+        assert [point["stage"] for point in checkpoints] == expected_stages
+        assert len({point["pid"] for point in checkpoints}) == 1
+        assert len({point["application"] for point in checkpoints}) == 1
+        original = checkpoints[0]["activity"]
+        recreated = checkpoints[4]["activity"]
+        assert original != recreated and all(point["activity"] == original for point in checkpoints[:4])
+        events = observed["events"]
+        assert {event["pid"] for event in events} == {checkpoints[0]["pid"]}
+        assert {event["application"] for event in events} == {checkpoints[0]["application"]}
+        assert [event["activity"] for event in events if event["event"] == "created"] == [original, recreated]
+        assert [event["activity"] for event in events if event["event"] == "destroyed"] == [original, recreated]
+        assert any(event["event"] == "stopped" and event["activity"] == original for event in events)
+        assert checkpoints[2]["lifecycle"] == "CREATED"
+        assert all(point["lifecycle"] == "RESUMED" for point in checkpoints if point["stage"] != "background")
+        seed = {**INPUTS["seed"], "completed": 0}
+        for point in checkpoints:
+            stage = point["stage"]
+            folder = native / stage
+            assert json.loads((folder / "state.json").read_text()) == point
+            with tempfile.TemporaryDirectory(prefix="mse-m08-native-") as temporary:
+                local = Path(temporary)
+                for suffix in ("", "-wal"):
+                    source = folder / (DB_NAME + suffix)
+                    if source.exists():
+                        shutil.copyfile(source, local / source.name)
+                with sqlite3.connect(local / DB_NAME) as database:
+                    assert database.execute("PRAGMA integrity_check").fetchone()[0] == "ok"
+                    assert database.execute("PRAGMA user_version").fetchone()[0] == 5
+                    def rows(table, order):
+                        fields = [row[1] for row in database.execute(f"PRAGMA table_info({table})")]
+                        return [dict(zip(fields, row)) for row in database.execute(f"SELECT * FROM {table} ORDER BY {order}")]
+                    assert [row[1] for row in database.execute("PRAGMA table_info(items)")] == ["id", "title", "completed", "version", "updatedAt"]
+                    items = rows("items", "id")
+                    pending = rows("pending_mutations", "sequence")
+                    assert items == point["items"] and pending == point["pending"]
+                    assert rows("acknowledged_mutations", "clientMutationId") == point["acknowledged"] == []
+                    assert rows("tombstones", "id") == point["tombstones"] == []
+                    assert rows("sync_metadata", "id") == []
+            if stage != "submitted":
+                assert items == [seed] and pending == [], (stage, items, pending)
+            else:
+                saved, = items
+                assert {**saved, "title": seed["title"], "updatedAt": seed["updatedAt"]} == seed
+                assert saved["title"] == INPUTS["draft"]
+                assert observed["saveClockBounds"][0] <= saved["updatedAt"] <= observed["saveClockBounds"][1]
+                intent, = pending
+                assert (intent["itemId"], intent["operation"], intent["title"], intent["baseVersion"], intent["dispatched"], intent["terminalError"]) == ("ui-001", "RENAME", INPUTS["draft"], 1, 0, None)
+            if stage != "background":
+                expected_draft = stage in ("draft", "foreground") or (fixed and stage == "recreated")
+                assert point["editorText"] == (INPUTS["draft"] if expected_draft else "")
+                assert point["saveVisible"] == expected_draft
+                assert (folder / "semantics.txt").read_text().strip()
+        self.result.update(nativeEvidence=str(native), nativeSnapshots=len(checkpoints),
+                           pid=checkpoints[0]["pid"], application=checkpoints[0]["application"],
+                           originalActivity=original, recreatedActivity=recreated,
+                           backgroundCycles=1, recreations=1, explicitSaves=int(fixed),
+                           observed=observed["status"])
+
+    def run(self):
+        assert "Item fixture listening" in self.args.fixture_log.read_text(), "Owned fixture not ready"
+        self.result["initialNetwork"] = self.wait_network(True)
+        self.adb("shell", "am", "force-stop", PACKAGE)
+        self.teardown_gate()
+        self.adb("install", "-r", str(self.args.apk.resolve()))
+        self.adb("install", "-r", str(self.args.test_apk.resolve()))
+        # All previous reports must already be exported; this is before the fixed seed/launch.
+        assert self.adb("shell", "pm", "clear", PACKAGE) == "Success"
+        self.teardown_gate()
+        self.adb("shell", "input", "keyevent", "KEYCODE_WAKEUP")
+        self.adb("shell", "wm", "dismiss-keyguard")
+        self.adb("logcat", "-c")
+        self.fixture_offset = self.args.fixture_log.stat().st_size
+        self.result["fixtureLogStartOffset"] = self.fixture_offset
+        code, text = self.instrument()
+        self.adb("logcat", "-d", "-v", "threadtime")
+        self.result["pidAfterJUnit"] = self.adb("shell", "pidof", PACKAGE, allow_failure=True)
+        native = self.export_native()
+        statuses = [int(value) for value in re.findall(r"INSTRUMENTATION_STATUS_CODE: (-?\d+)", text)]
+        assert code == 0 and statuses.count(0) == 1 and all(value in (0, 1) for value in statuses), text
+        assert re.search(r"OK \(1 tests?\)", text), text
+        assert "frozenM08ActivityLifecycleSequence" in text
+        self.result["junit"].update(status="PASS", tests=1, failures=0, skipped=0)
+        self.audit_native(native)
+        fixture_bytes = self.args.fixture_log.read_bytes()
+        window = fixture_bytes[self.fixture_offset:]
+        requests = [json.loads(line) for line in window.decode().splitlines() if line.startswith("{")]
+        assert requests == [], requests
+        self.result.update(applicationHttpRequests=0, fixtureLogEndOffset=len(fixture_bytes),
+                           status="PASS")
+
+
+def main():
+    parser = argparse.ArgumentParser(description=__doc__)
+    parser.add_argument("--expect", choices=("limitation", "restored"), required=True)
+    parser.add_argument("--apk", type=Path, required=True)
+    parser.add_argument("--test-apk", type=Path, required=True)
+    parser.add_argument("--adb", type=Path, default=Path(os.environ.get("ANDROID_HOME", "")) / "platform-tools/adb")
+    parser.add_argument("--fixture-log", type=Path, required=True)
+    parser.add_argument("--output", type=Path, required=True)
+    args = parser.parse_args()
+    scenario = ActivityStateScenario(args)
+    started = time.monotonic()
+    try:
+        scenario.run()
+    except Exception as error:
+        scenario.result.update(status="FAIL", error=repr(error))
+        raise
+    finally:
+        try:
+            scenario.adb("shell", "am", "force-stop", PACKAGE)
+            absent = not scenario.adb("shell", "pidof", PACKAGE, allow_failure=True)
+            network = scenario.restore_network()
+            scenario.result["cleanup"] = {"appAbsent": absent, "network": network}
+            assert absent
+        except Exception as cleanup_error:
+            scenario.result.update(status="FAIL", cleanupError=repr(cleanup_error))
+            raise
+        finally:
+            scenario.result.update(adbCommands=scenario.command_count,
+                                   elapsedSeconds=round(time.monotonic() - started, 3))
+            (scenario.output / "result.json").write_text(json.dumps(scenario.result, indent=2) + "\n")
+            print(json.dumps(scenario.result, indent=2), flush=True)
+
+
+if __name__ == "__main__":
+    main()


