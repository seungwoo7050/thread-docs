# M08 — Foreground, Background와 State Restoration

## `test(M08): preserve actual Activity recreation baseline`

diff --git a/android/app/src/androidTest/assets/m08-inputs.json b/android/app/src/androidTest/assets/m08-inputs.json
new file mode 100644
index 0000000..0b8e11f
--- /dev/null
+++ b/android/app/src/androidTest/assets/m08-inputs.json
@@ -0,0 +1,23 @@
+{
+  "seed": {"id": "ui-001", "title": "Saved title", "completed": false, "version": 1, "updatedAt": 1700000000000},
+  "draft": "Unsubmitted draft",
+  "backgroundCycles": 1,
+  "recreations": 1,
+  "submissions": 1,
+  "networkDelayMs": 0,
+  "uiTimeoutMs": 15000,
+  "schemaVersion": 5,
+  "seedSql": [
+    "CREATE TABLE items (id TEXT PRIMARY KEY NOT NULL, title TEXT NOT NULL CHECK(length(trim(title)) > 0), completed INTEGER NOT NULL CHECK(completed IN (0, 1)), version INTEGER NOT NULL CHECK(version >= 0), updated_at INTEGER NOT NULL)",
+    "CREATE TABLE local_identity (singleton INTEGER PRIMARY KEY CHECK(singleton = 1), next_id INTEGER NOT NULL CHECK(next_id > 0))",
+    "INSERT INTO local_identity VALUES (1, 1)",
+    "CREATE TABLE sync_metadata (singleton INTEGER PRIMARY KEY CHECK(singleton = 1), last_successful_refresh_at INTEGER CHECK(last_successful_refresh_at IS NULL OR last_successful_refresh_at >= 0), last_acknowledgement TEXT)",
+    "INSERT INTO sync_metadata VALUES (1, NULL, NULL)",
+    "CREATE TABLE pending_mutations (sequence INTEGER PRIMARY KEY AUTOINCREMENT, kind TEXT NOT NULL CHECK(kind IN ('create', 'rename', 'toggle', 'delete')), item_id TEXT NOT NULL, payload TEXT CHECK(payload IS NOT NULL OR kind = 'delete'), client_mutation_id TEXT NOT NULL, payload_hash TEXT NOT NULL, terminal_error TEXT CHECK(terminal_error IS NULL OR terminal_error = 'identity_conflict'), dispatched INTEGER NOT NULL CHECK(dispatched IN (0, 1)))",
+    "CREATE UNIQUE INDEX pending_mutation_identity ON pending_mutations (client_mutation_id)",
+    "INSERT INTO sqlite_sequence (name, seq) VALUES ('pending_mutations', 0)",
+    "CREATE TABLE remote_versions (id TEXT PRIMARY KEY NOT NULL, version INTEGER NOT NULL CHECK(version > 0), updated_at INTEGER, deleted INTEGER NOT NULL CHECK(deleted IN (0, 1)), canonical_item TEXT CHECK((deleted = 0 AND canonical_item IS NOT NULL) OR (deleted = 1 AND canonical_item IS NULL)))",
+    "CREATE TABLE mutation_conflicts (client_mutation_id TEXT PRIMARY KEY NOT NULL, intent TEXT NOT NULL, reason TEXT NOT NULL CHECK(reason IN ('version_conflict', 'unversioned_legacy')), item TEXT, tombstone TEXT)",
+    "PRAGMA user_version = 5"
+  ]
+}
diff --git a/android/app/src/androidTest/java/com/mse/reactnative/M08LifecycleTest.java b/android/app/src/androidTest/java/com/mse/reactnative/M08LifecycleTest.java
new file mode 100644
index 0000000..4ba9e85
--- /dev/null
+++ b/android/app/src/androidTest/java/com/mse/reactnative/M08LifecycleTest.java
@@ -0,0 +1,252 @@
+package com.mse.reactnative;
+
+import android.app.Activity;
+import android.app.Application;
+import android.content.Context;
+import android.database.Cursor;
+import android.database.sqlite.SQLiteDatabase;
+import android.os.Bundle;
+import android.os.Process;
+import android.os.SystemClock;
+import android.util.Log;
+import androidx.lifecycle.Lifecycle;
+import androidx.test.core.app.ActivityScenario;
+import androidx.test.ext.junit.runners.AndroidJUnit4;
+import androidx.test.platform.app.InstrumentationRegistry;
+import androidx.test.uiautomator.By;
+import androidx.test.uiautomator.UiDevice;
+import androidx.test.uiautomator.UiObject2;
+import androidx.test.uiautomator.Until;
+import java.io.File;
+import java.io.FileInputStream;
+import java.io.FileOutputStream;
+import java.io.InputStream;
+import java.nio.charset.StandardCharsets;
+import org.json.JSONArray;
+import org.json.JSONObject;
+import org.junit.Test;
+import org.junit.runner.RunWith;
+import static org.junit.Assert.*;
+
+/** One real stop/resume and one Activity recreation; never a process restart. */
+@RunWith(AndroidJUnit4.class)
+public class M08LifecycleTest {
+    private final Context context = InstrumentationRegistry.getInstrumentation().getTargetContext();
+    private final Application application = (Application) context.getApplicationContext();
+    private final UiDevice device = UiDevice.getInstance(InstrumentationRegistry.getInstrumentation());
+    private final File output = new File(context.getFilesDir(), "m08-evidence");
+    private final File database = new File(context.getFilesDir(), "items.db");
+    private final JSONObject result = object("status", "RUNNING");
+    private final JSONArray events = new JSONArray();
+    private final JSONObject databases = new JSONObject();
+    private final JSONArray checkpoints = new JSONArray();
+    private final int pid = Process.myPid();
+    private JSONObject input;
+    private long timeout;
+
+    private static JSONObject object(Object... pairs) {
+        JSONObject value = new JSONObject();
+        try { for (int i = 0; i < pairs.length; i += 2) value.put((String) pairs[i], pairs[i + 1]); }
+        catch (Exception error) { throw new IllegalStateException(error); }
+        return value;
+    }
+
+    private void event(String name, Activity activity, Bundle state) {
+        if (!(activity instanceof MainActivity)) return;
+        JSONObject value = object("event", name, "pid", Process.myPid(),
+                "application", System.identityHashCode(activity.getApplication()),
+                "activity", System.identityHashCode(activity), "time", SystemClock.elapsedRealtime(),
+                "hasSavedState", state != null);
+        synchronized (events) { events.put(value); }
+        Log.i("M08Lifecycle", value.toString());
+    }
+
+    private final Application.ActivityLifecycleCallbacks callbacks = new Application.ActivityLifecycleCallbacks() {
+        public void onActivityCreated(Activity a, Bundle b) { event("created", a, b); }
+        public void onActivityStarted(Activity a) { event("started", a, null); }
+        public void onActivityResumed(Activity a) { event("resumed", a, null); }
+        public void onActivityPaused(Activity a) { event("paused", a, null); }
+        public void onActivityStopped(Activity a) { event("stopped", a, null); }
+        public void onActivitySaveInstanceState(Activity a, Bundle b) { event("saved", a, b); }
+        public void onActivityDestroyed(Activity a) { event("destroyed", a, null); }
+    };
+
+    private UiObject2 control(String label) {
+        UiObject2 value = device.wait(Until.findObject(By.desc(label)), timeout);
+        assertNotNull("Missing accessible control: " + label, value);
+        return value;
+    }
+
+    private void write(String name, String value) throws Exception {
+        try (FileOutputStream stream = new FileOutputStream(new File(output, name))) {
+            stream.write(value.getBytes(StandardCharsets.UTF_8));
+        }
+    }
+
+    private void save() throws Exception {
+        synchronized (events) { result.put("events", new JSONArray(events.toString())); }
+        result.put("databases", databases);
+        result.put("checkpoints", checkpoints);
+        write("result.json", result.toString(2));
+    }
+
+    private static void copy(File source, File target) throws Exception {
+        try (InputStream in = new FileInputStream(source); FileOutputStream out = new FileOutputStream(target)) {
+            byte[] buffer = new byte[8192];
+            int length;
+            while ((length = in.read(buffer)) != -1) out.write(buffer, 0, length);
+        }
+    }
+
+    private JSONObject snapshot(String name) throws Exception {
+        JSONObject value = new JSONObject();
+        try (SQLiteDatabase db = SQLiteDatabase.openDatabase(database.getPath(), null, SQLiteDatabase.OPEN_READONLY)) {
+            value.put("schemaVersion", db.getVersion());
+            for (String table : new String[]{"items", "pending_mutations", "mutation_conflicts", "remote_versions", "sync_metadata", "local_identity", "sqlite_sequence"}) {
+                JSONArray rows = new JSONArray();
+                try (Cursor cursor = db.rawQuery("SELECT * FROM " + table + " ORDER BY rowid", null)) {
+                    while (cursor.moveToNext()) {
+                        JSONObject row = new JSONObject();
+                        for (int i = 0; i < cursor.getColumnCount(); i++) {
+                            Object cell = cursor.isNull(i) ? JSONObject.NULL
+                                    : cursor.getType(i) == Cursor.FIELD_TYPE_INTEGER ? cursor.getLong(i) : cursor.getString(i);
+                            row.put(cursor.getColumnName(i), cell);
+                        }
+                        rows.put(row);
+                    }
+                }
+                value.put(table, rows);
+            }
+            // The UI is idle at these checkpoints. Preserve actual native files,
+            // including any WAL, separately from the queried JSON evidence.
+            for (String suffix : new String[]{"", "-wal", "-shm"}) {
+                File source = new File(database.getPath() + suffix);
+                if (source.exists()) copy(source, new File(output, name + ".db" + suffix));
+            }
+        }
+        databases.put(name, value);
+        write(name + ".json", value.toString(2));
+        save();
+        return value;
+    }
+
+    private void checkpoint(String name, ActivityScenario<MainActivity> scenario, MainActivity activity) throws Exception {
+        assertEquals("Application process changed", pid, Process.myPid());
+        assertSame("Application identity changed", application, activity.getApplication());
+        Object reactContext = ((MainApplication) application).getReactNativeHost().getReactInstanceManager().getCurrentReactContext();
+        checkpoints.put(object("name", name, "pid", Process.myPid(), "application", System.identityHashCode(application),
+                "activity", System.identityHashCode(activity), "reactContext", System.identityHashCode(reactContext),
+                "state", scenario.getState().name(), "time", SystemClock.elapsedRealtime()));
+        save();
+    }
+
+    private boolean editor(String stage) throws Exception {
+        UiObject2 field = device.findObject(By.desc("Edit item title"));
+        boolean restored = field != null && input.getString("draft").equals(field.getText())
+                && device.findObjects(By.desc("Edit item title")).size() == 1
+                && device.findObjects(By.desc("Save title")).size() == 1;
+        result.put(stage + "Editor", object("restored", restored, "editControls", device.findObjects(By.desc("Edit item title")).size(),
+                "draft", field == null ? JSONObject.NULL : field.getText(),
+                "newTitleControls", device.findObjects(By.desc("New item title")).size()));
+        device.dumpWindowHierarchy(new File(output, stage + ".xml"));
+        assertTrue(device.takeScreenshot(new File(output, stage + ".png")));
+        save();
+        return restored;
+    }
+
+    @Test
+    public void draftSurvivesOneBackgroundCycleAndOneActivityRecreation() throws Throwable {
+        assertTrue("Fresh evidence directory required", output.mkdir());
+        try (InputStream stream = InstrumentationRegistry.getInstrumentation().getContext().getAssets().open("m08-inputs.json")) {
+            java.io.ByteArrayOutputStream bytes = new java.io.ByteArrayOutputStream();
+            byte[] buffer = new byte[4096];
+            int size;
+            while ((size = stream.read(buffer)) != -1) bytes.write(buffer, 0, size);
+            input = new JSONObject(bytes.toString("UTF-8"));
+        }
+        timeout = input.getLong("uiTimeoutMs");
+        JSONObject seed = input.getJSONObject("seed");
+        assertFalse("Seeding is allowed only before the first launch", database.exists());
+        try (SQLiteDatabase db = SQLiteDatabase.openOrCreateDatabase(database, null)) {
+            db.beginTransaction();
+            try {
+                JSONArray statements = input.getJSONArray("seedSql");
+                for (int i = 0; i < statements.length(); i++) db.execSQL(statements.getString(i));
+                db.execSQL("INSERT INTO items VALUES (?, ?, 0, 1, ?)", new Object[]{seed.getString("id"), seed.getString("title"), seed.getLong("updatedAt")});
+                db.execSQL("INSERT INTO remote_versions VALUES (?, 1, ?, 0, ?)", new Object[]{seed.getString("id"), seed.getLong("updatedAt"), seed.toString()});
+                db.setTransactionSuccessful();
+            } finally { db.endTransaction(); }
+        }
+        JSONObject original = snapshot("seed");
+        application.registerActivityLifecycleCallbacks(callbacks);
+        try (ActivityScenario<MainActivity> scenario = ActivityScenario.launch(MainActivity.class)) {
+            MainActivity[] first = new MainActivity[1];
+            scenario.onActivity(activity -> first[0] = activity);
+            control("Local storage ready");
+            control("Item count: 1");
+            control("Pending uploads: 0");
+            checkpoint("initial", scenario, first[0]);
+            control("Edit " + seed.getString("title")).click();
+            UiObject2 field = control("Edit item title");
+            field.click();
+            field.setText(input.getString("draft"));
+            device.pressBack();
+            assertTrue("Initial draft was not rendered", editor("draft"));
+            assertEquals(original.toString(), snapshot("draft").toString());
+
+            scenario.moveToState(Lifecycle.State.CREATED);
+            assertEquals(Lifecycle.State.CREATED, scenario.getState());
+            checkpoint("background", scenario, first[0]);
+            assertEquals(original.toString(), snapshot("background").toString());
+            scenario.moveToState(Lifecycle.State.RESUMED);
+            scenario.onActivity(activity -> assertSame("Background cycle replaced Activity", first[0], activity));
+            control("Local storage ready");
+            checkpoint("foreground", scenario, first[0]);
+            assertTrue("Background cycle lost editor", editor("foreground"));
+            assertEquals(original.toString(), snapshot("foreground").toString());
+
+            scenario.recreate(); // On the test thread; the native Activity is actually destroyed.
+            MainActivity[] second = new MainActivity[1];
+            scenario.onActivity(activity -> second[0] = activity);
+            assertNotSame("Activity recreation did not replace Activity", first[0], second[0]);
+            assertTrue("Prior Activity was not destroyed", first[0].isDestroyed());
+            control("Local storage ready");
+            checkpoint("recreated", scenario, second[0]);
+            boolean restored = editor("recreated");
+            assertEquals(original.toString(), snapshot("recreated").toString());
+            assertTrue("M08_RECREATION_DRAFT_OR_SELECTION_LOST", restored);
+
+            result.put("submitStartedAt", System.currentTimeMillis());
+            control("Save title").click(); // The scenario's only submit.
+            control("Pending uploads: 1");
+            control("Local storage ready");
+            control("New item title");
+            assertTrue(device.hasObject(By.text(input.getString("draft"))));
+            JSONObject submitted = snapshot("submitted");
+            JSONArray items = submitted.getJSONArray("items");
+            assertEquals(1, items.length());
+            assertEquals(seed.getString("id"), items.getJSONObject(0).getString("id"));
+            assertEquals(input.getString("draft"), items.getJSONObject(0).getString("title"));
+            assertEquals(2, items.getJSONObject(0).getInt("version"));
+            JSONArray pending = submitted.getJSONArray("pending_mutations");
+            assertEquals("Only one callback may commit an edit", 1, pending.length());
+            assertEquals("rename", pending.getJSONObject(0).getString("kind"));
+            assertEquals(seed.getString("id"), pending.getJSONObject(0).getString("item_id"));
+            assertEquals(0, pending.getJSONObject(0).getInt("dispatched"));
+            assertEquals(1, new JSONObject(pending.getJSONObject(0).getString("payload")).getInt("baseVersion"));
+            checkpoint("submitted", scenario, second[0]);
+            device.dumpWindowHierarchy(new File(output, "submitted.xml"));
+            assertTrue(device.takeScreenshot(new File(output, "submitted.png")));
+            result.put("status", "PASS");
+        } catch (Throwable failure) {
+            result.put("status", "FAIL");
+            result.put("error", failure.toString());
+            result.put("stack", Log.getStackTraceString(failure));
+            throw failure;
+        } finally {
+            application.unregisterActivityLifecycleCallbacks(callbacks);
+            result.put("callbacksUnregistered", true);
+            save();
+        }
+    }
+}
diff --git a/scripts/verify_m08.py b/scripts/verify_m08.py
new file mode 100644
index 0000000..80ead36
--- /dev/null
+++ b/scripts/verify_m08.py
@@ -0,0 +1,256 @@
+#!/usr/bin/env python3
+"""One frozen M08 native lifecycle invocation. Requires the exclusive device lease."""
+import argparse
+import hashlib
+import io
+import json
+import os
+from pathlib import Path
+import re
+import socket
+import sqlite3
+import subprocess
+import tarfile
+import time
+from urllib.request import urlopen
+from verify_m07 import package_in_live_activities
+
+PACKAGE = "com.mse.reactnative"
+TEST = "com.mse.reactnative.M08LifecycleTest"
+NETWORK_KEYS = ("airplane_mode_on", "wifi_on", "mobile_data")
+TABLES = ("items", "pending_mutations", "mutation_conflicts", "remote_versions", "sync_metadata", "local_identity", "sqlite_sequence")
+
+
+def main():
+    parser = argparse.ArgumentParser()
+    parser.add_argument("--adb", default="adb")
+    parser.add_argument("--serial", default="emulator-5554")
+    parser.add_argument("--node", default="node")
+    parser.add_argument("--apk", required=True)
+    parser.add_argument("--test-apk", required=True)
+    parser.add_argument("--evidence", required=True)
+    parser.add_argument("--baseline", action="store_true")
+    args = parser.parse_args()
+    root = Path(__file__).resolve().parent.parent
+    evidence = Path(args.evidence).resolve()
+    evidence.mkdir(parents=True, exist_ok=False)
+    commands, controls = [], []
+    inputs_file = root / "android/app/src/androidTest/assets/m08-inputs.json"
+    inputs = json.loads(inputs_file.read_text())
+    sha = lambda path: hashlib.sha256(Path(path).read_bytes()).hexdigest()
+    result = {"status": "RUNNING", "baseline": args.baseline, "hostPid": os.getpid(),
+              "apkSha256": sha(args.apk), "testApkSha256": sha(args.test_apk),
+              "harnessSha256": sha(__file__), "resetPredicateSourceSha256": sha(root / "scripts/verify_m07.py"),
+              "inputsSha256": sha(inputs_file), "fixtureSha256": sha(root / "fixture/server.cjs")}
+
+    def save():
+        (evidence / "result.json").write_text(json.dumps(result, indent=2) + "\n")
+
+    def adb(label, *parts, check=True, binary=False, timeout=60):
+        command = [args.adb, "-s", args.serial, *parts]
+        entry = {"label": label, "command": command, "timeoutSeconds": timeout}
+        started = time.monotonic()
+        try:
+            completed = subprocess.run(command, capture_output=True, timeout=timeout)
+            raw, err = completed.stdout, completed.stderr
+            entry["exit"] = completed.returncode
+        except subprocess.TimeoutExpired as error:
+            raw, err = error.stdout, error.stderr
+            entry.update(exit=None, error=repr(error))
+        for stream, data in (("stdout", raw), ("stderr", err)):
+            entry[stream] = None
+            if data is not None:
+                filename = f"adb-{len(commands):04d}-{label}.{stream}"
+                (evidence / filename).write_bytes(data)
+                entry[stream] = filename
+        entry["elapsedSeconds"] = time.monotonic() - started
+        commands.append(entry)
+        (evidence / "commands.json").write_text(json.dumps(commands, indent=2) + "\n")
+        assert entry["exit"] is not None, entry
+        if check:
+            assert entry["exit"] == 0, entry
+        return raw if binary else raw.decode(errors="replace").strip()
+
+    def network(label):
+        settings = {key: adb(label + "-" + key, "shell", "settings", "get", "global", key) for key in NETWORK_KEYS}
+        connectivity = adb(label + "-connectivity", "shell", "dumpsys", "connectivity")
+        assert "Active default network: none" not in connectivity
+        return settings
+
+    def reset_barrier():
+        # Same verified API34 predicate/10s deadline/1s quiet period as M07.
+        started, quiet = time.monotonic(), None
+        observations = result.setdefault("resetObservations", [])
+        while True:
+            remaining = 10 - (time.monotonic() - started)
+            assert remaining > 0, "Initial reset teardown did not settle within 10s"
+            pid = adb("reset-pid", "shell", "pidof", PACKAGE, check=False, timeout=remaining)
+            assert commands[-1]["exit"] in (0, 1)
+            remaining = 10 - (time.monotonic() - started)
+            assert remaining > 0
+            activities = adb("reset-activities", "shell", "dumpsys", "activity", "activities", timeout=remaining)
+            present = package_in_live_activities(activities)
+            now = time.monotonic()
+            observations.append({"elapsedSeconds": now - started, "pid": pid, "liveActivity": present,
+                                 "pidCommandIndex": len(commands) - 2, "activityCommandIndex": len(commands) - 1})
+            save()
+            assert now - started < 10
+            if not pid and not present:
+                quiet = now if quiet is None else quiet
+                if now - quiet >= 1:
+                    return
+            else:
+                quiet = None
+            time.sleep(min(0.1, 10 - (time.monotonic() - started)))
+
+    def remote():
+        event = {"method": "GET", "path": "/__state"}
+        try:
+            with urlopen("http://127.0.0.1:18081/__state", timeout=3) as reply:
+                value = json.load(reply)
+                event.update(status=reply.status, response=value)
+            assert event["status"] == 200
+            return value
+        except Exception as error:
+            event["error"] = repr(error)
+            raise
+        finally:
+            controls.append(event)
+            (evidence / "http-controls.json").write_text(json.dumps(controls, indent=2) + "\n")
+
+    fixture = None
+    original_network = None
+    started = time.monotonic()
+    try:
+        with socket.socket() as probe:
+            assert probe.connect_ex(("127.0.0.1", 18081)) != 0, "Fixture port18081 already owned"
+        with (evidence / "fixture.log").open("wb") as log:
+            fixture = subprocess.Popen([args.node, str(root / "fixture/server.cjs")], stdout=log, stderr=subprocess.STDOUT)
+        result["fixturePid"] = fixture.pid
+        deadline = time.monotonic() + 5
+        while True:
+            assert fixture.poll() is None, "Owned fixture exited during startup"
+            try:
+                before_remote = remote()
+                break
+            except Exception:
+                if time.monotonic() >= deadline:
+                    raise
+                time.sleep(0.1)
+        assert before_remote["requests"] == []
+        original_network = network("initial")
+        result["networkBefore"] = original_network
+        assert original_network == dict(zip(NETWORK_KEYS, ("0", "1", "1")))
+        for label, apk in (("app", args.apk), ("test", args.test_apk)):
+            assert "Success" in adb("install-" + label, "install", "-r", str(Path(apk).resolve()))
+        adb("stop-before-seed", "shell", "am", "force-stop", PACKAGE)
+        assert adb("clear-before-seed", "shell", "pm", "clear", PACKAGE) == "Success"
+        reset_barrier()
+        adb("wake", "shell", "input", "keyevent", "KEYCODE_WAKEUP")
+        adb("keyguard", "shell", "wm", "dismiss-keyguard")
+        adb("clear-logcat", "logcat", "-c")
+        suite = adb("instrumentation", "shell", "am", "instrument", "-w", "-r", "-e", "class", TEST,
+                    PACKAGE + ".test/androidx.test.runner.AndroidJUnitRunner", timeout=180)
+        result["instrumentationCodes"] = re.findall(r"^INSTRUMENTATION_STATUS_CODE: (-?\d+)$", suite, re.M)
+        archive = adb("native-evidence", "exec-out", "run-as", PACKAGE, "tar", "-cf", "-", "files/m08-evidence", binary=True)
+        (evidence / "native-evidence.tar").write_bytes(archive)
+        native_dir = evidence / "native"
+        native_dir.mkdir()
+        with tarfile.open(fileobj=io.BytesIO(archive)) as saved:
+            for member in saved:
+                if member.isdir():
+                    continue
+                assert member.isfile() and member.name.startswith("files/m08-evidence/")
+                relative = Path(member.name).relative_to("files/m08-evidence")
+                assert len(relative.parts) == 1 and relative.name not in (".", "..")
+                (native_dir / relative).write_bytes(saved.extractfile(member).read())
+        native = json.loads((native_dir / "result.json").read_text())
+        result["nativeStatus"] = native["status"]
+        points = {p["name"]: p for p in native["checkpoints"]}
+        assert list(points)[:4] == ["initial", "background", "foreground", "recreated"]
+        initial = points["initial"]
+        for point in points.values():
+            assert point["pid"] == initial["pid"] and point["application"] == initial["application"]
+        assert points["background"]["state"] == "CREATED"
+        assert points["foreground"]["state"] == points["recreated"]["state"] == "RESUMED"
+        assert initial["activity"] == points["background"]["activity"] == points["foreground"]["activity"]
+        assert points["recreated"]["activity"] != initial["activity"]
+        for point, counts in (("foreground", {"created": 1, "stopped": 1, "resumed": 2, "destroyed": 0}),
+                              ("recreated", {"created": 2, "stopped": 2, "resumed": 3, "destroyed": 1})):
+            events = [event for event in native["events"] if event["time"] <= points[point]["time"]]
+            for name, count in counts.items():
+                assert sum(event["event"] == name for event in events) == count, (point, name, events)
+        assert all(event["pid"] == initial["pid"] and event["application"] == initial["application"] for event in native["events"])
+        assert native["callbacksUnregistered"] is True
+        assert native["draftEditor"]["restored"] and native["foregroundEditor"]["restored"]
+        snapshots = native["databases"]
+        for name, expected in snapshots.items():
+            path = native_dir / (name + ".db")
+            before_hash = sha(path)
+            with sqlite3.connect(f"file:{path}?mode=ro", uri=True) as db:
+                db.row_factory = sqlite3.Row
+                actual = {"schemaVersion": db.execute("PRAGMA user_version").fetchone()[0]}
+                actual.update({table: [dict(row) for row in db.execute("SELECT * FROM " + table + " ORDER BY rowid")] for table in TABLES})
+            assert actual == expected, name
+            assert sha(path) == before_hash
+        assert snapshots["seed"]["items"] == [{"id": "ui-001", "title": "Saved title", "completed": 0, "version": 1, "updated_at": 1700000000000}]
+        assert snapshots["seed"]["pending_mutations"] == []
+        for name in ("draft", "background", "foreground", "recreated"):
+            assert snapshots[name] == snapshots["seed"], name
+        if native["status"] == "PASS":
+            assert "OK (1 test)" in suite and "FAILURES" not in suite
+            assert result["instrumentationCodes"] == ["1", "0"]
+            assert native["recreatedEditor"]["restored"]
+            submitted = snapshots["submitted"]
+            assert len(submitted["items"]) == len(submitted["pending_mutations"]) == 1
+            item = submitted["items"][0]
+            assert item == {"id": "ui-001", "title": inputs["draft"], "completed": 0, "version": 2, "updated_at": item["updated_at"]}
+            assert item["updated_at"] >= native["submitStartedAt"]
+            operation = submitted["pending_mutations"][0]
+            payload = {"title": inputs["draft"], "baseVersion": 1}
+            assert json.loads(operation["payload"]) == payload
+            canonical = json.dumps({"method": "PATCH", "path": "/items/ui-001", "payload": payload}, sort_keys=True, separators=(",", ":"))
+            assert operation["payload_hash"] == hashlib.sha256(canonical.encode()).hexdigest()
+            assert operation["client_mutation_id"] and operation["kind"] == "rename" and operation["item_id"] == "ui-001"
+            assert operation["dispatched"] == 0 and operation["terminal_error"] is None
+            for table in ("remote_versions", "sync_metadata", "local_identity", "mutation_conflicts"):
+                assert submitted[table] == snapshots["seed"][table]
+            result["status"] = "PASS_BASELINE_ALREADY_CORRECT" if args.baseline else "PASS"
+        else:
+            assert args.baseline and "M08_RECREATION_DRAFT_OR_SELECTION_LOST" in native.get("error", ""), native
+            assert native["recreatedEditor"]["restored"] is False and "submitted" not in snapshots
+            assert "FAILURES!!!" in suite and result["instrumentationCodes"] in (["1", "-2"], ["1", "-1"])
+            result["status"] = "BASELINE_LIMIT_REPRODUCED"
+        after_remote = remote()
+        assert after_remote == before_remote, "Lifecycle must not implicitly synchronize"
+        result.update(checkpoints=native["checkpoints"], nativeDatabaseCount=len(snapshots), fixtureRequests=after_remote["requests"])
+    except Exception as error:
+        result.update(status="FAIL", error=repr(error))
+    finally:
+        try:
+            adb("logcat", "logcat", "-d", "-v", "threadtime")
+            adb("cleanup-stop", "shell", "am", "force-stop", PACKAGE)
+            result["pidAfterCleanup"] = adb("cleanup-pid", "shell", "pidof", PACKAGE, check=False)
+            assert not result["pidAfterCleanup"]
+            result["networkAfter"] = network("cleanup")
+            if original_network is not None:
+                assert result["networkAfter"] == original_network
+        except Exception as error:
+            result.update(status="FAIL", cleanupError=repr(error))
+        if fixture is not None:
+            fixture.terminate()
+            try:
+                result["fixtureExit"] = fixture.wait(timeout=5)
+                assert result["fixtureExit"] == 0
+            except Exception as error:
+                fixture.kill()
+                result["fixtureExit"] = fixture.wait(timeout=5)
+                result.update(status="FAIL", fixtureCleanupError=repr(error))
+        result.update(elapsedSeconds=time.monotonic() - started, adbCommands=len(commands))
+        save()
+        print(json.dumps(result), flush=True)
+    return 1 if result["status"] == "FAIL" else 0
+
+
+if __name__ == "__main__":
+    raise SystemExit(main())
diff --git a/verification/M08.md b/verification/M08.md
new file mode 100644
index 0000000..a86afaa
--- /dev/null
+++ b/verification/M08.md
@@ -0,0 +1,14 @@
+# M08 verification — phase-1
+
+- Spec revision: `61280dd86ce88b6e431f408241c0998a275960aa`.
+- START: verified M07 `34e03d3123e513e0dcd0ea2e55be981f6577249b`.
+- Attempt1; repairs0/2. Current checkpoint: baseline reproduced; product/final verification pending.
+- Evidence root: `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M08/`.
+
+## Frozen unchanged-app baseline
+
+[Frozen manifest](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M08/frozen-baseline.json) records the exact command, all 53 unchanged START files, the new native test/asset/runner and preserved APKs. App SHA256 remains `a42cc4c143e51aae73ff3645d829f2491e457c29607c659f2852c45d483391f7`; instrumentation is `fcf6931c379bdbb2bcf6ec02c280e3f04b09b3de8068a77915850401856f3043`. Test-only build PASS (12.882s), frozen support/SQL/sequence checks PASS. Existing ActivityScenario/UIAutomator dependencies and legacy Hermes configuration suffice; no production hook or dependency change.
+
+The single [baseline invocation](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M08/baseline-android-01.command.json) exited0 in 17.057s, with 34 adb commands: **BASELINE_LIMIT_REPRODUCED**, not Android acceptance PASS. Native JUnit failed only `M08_RECREATION_DRAFT_OR_SELECTION_LOST`. The draft survived one CREATED→RESUMED cycle; real recreation destroyed Activity46567430 and created232223380 with saved state, losing the editor/draft. PID15260, Application88250467 and ReactContext136876754 stayed identical. All five native databases retained exact `ui-001` / `Saved title` / version1 and pending0; no submit or HTTP request occurred.
+
+[Raw result](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M08/baseline-android-01/result.json), native archive/DB/UI/lifecycle logs and [main's independent baseline audit](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M08/main-baseline-audit.json) retain the complete proof. Cleanup left the app absent, network0/1/1 active and fixture90263 exited0/absent. The denied initial host `ps` probe and subsequent approved absence check are preserved beside the result; neither reran the Android scenario. Production implementation and main's final M08/native CRUD remain pending.


