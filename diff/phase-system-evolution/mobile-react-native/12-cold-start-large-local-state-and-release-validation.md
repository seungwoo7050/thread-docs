# M15 — Cold Start, Large Local State와 Release Validation

## `feat(react-native): bound M15 release list loading`

diff --git a/TRACK.md b/TRACK.md
index e97d99e..35c2ff3 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -273,7 +273,33 @@ before cancellation, and convergence after one foreground reconnect request.
 Normal background work also resumed before that reconnect click and remained
 enabled. See [M14 verification](/private/tmp/mobile-systems-evolution-ed7baa2/react-native/verification/M14.md)
 for measured concurrency, preserved failures and the unchanged earlier evidence.
-No M15 or phase-2 implementation is included.
+No phase-2 implementation is included.
+
+## M15: bounded initial reads and virtualized pages (phase-1)
+
+The screen now loads a native SQLite page of at most 50 Items. COUNT and the
+rowid-ordered LIMIT/OFFSET query share one existing transaction; page indices
+clamp after edits. FlatList renders only that page, with explicit First, Previous,
+Next and Last controls and no automatic page prefetch. Startup, background
+notifications and display reloads use the bounded read. Selected Items and drafts
+remain in the existing ephemeral editor owner when navigating or remounting.
+
+A committed edit remains successful if its later page reload fails; the screen
+retains the prior page and offers a separate list reload. Durable scheduling and
+queue semantics are unchanged. Explicit mutations still perform their historical
+full-array reads; that result is never passed or sliced into the rendered list.
+M15 bounds initial loading, not every later mutation's database cost.
+
+Root verified a genuine nondebuggable RN 0.76.9 release with bundled Hermes JS:
+initial native Item materialization fell from 2000 to 50, followed by real last-page
+navigation and all four fixed edits in one ordinary process. The final database
+contained 2000 Items and four durable intents. The release helper seeds/audits
+outside the measured Activity window; its native SELECT observer counts rows
+before bridge/JavaScript processing. Debug Android test tasks remain the default;
+`-PmseTestBuildType=release` selects the release helper explicitly. No schema,
+sync, retry, background or dependency-version change is included. See
+[M15 verification](/private/tmp/mobile-systems-evolution-ed7baa2/react-native/verification/phase-1/M15.md)
+for exact artifacts, preserved failures and the retained earlier regression scope.
 
 ## Toolchain and commands
 
@@ -305,9 +331,9 @@ node fixture/server.cjs
 ```
 
 Debug builds bundle their JavaScript and use Hermes with developer support disabled,
-so instrumentation never depends on a Metro server. The Android application and
-instrumentation APK use Android's generated debug signing key. This is not a
-production signing or release validation guarantee.
+so instrumentation never depends on a Metro server. Verification app and helper
+APKs use Android's generated debug signing key, not production signing credentials.
+M15 separately verifies the nondebuggable release variant and its bundled JS.
 
 The pinned [React Native 0.76 template](https://github.com/react-native-community/template/tree/0.76-stable/template)
 provides the platform bootstrap conventions. Legacy architecture remains in use;
diff --git a/__tests__/App.test.tsx b/__tests__/App.test.tsx
index ae90004..8f0751d 100644
--- a/__tests__/App.test.tsx
+++ b/__tests__/App.test.tsx
@@ -1,5 +1,5 @@
 import React from 'react';
-import {Button, DeviceEventEmitter, Keyboard, NativeModules} from 'react-native';
+import {Button, DeviceEventEmitter, FlatList, Keyboard, NativeModules} from 'react-native';
 import {act, fireEvent, render, screen, waitFor} from '@testing-library/react-native';
 import RootApp, {createEditorMemory} from '../src/App';
 import {openItemStore} from '../src/itemStore';
@@ -98,7 +98,7 @@ test('M02 database opening error disables writes and offers a non-destructive re
 test('M03 explicit sync publishes a committed database read, and never a service response', async () => {
   const fixture = [{id: 'remote-001', title: 'Alpha', completed: false, version: 1, updatedAt: 1700000000000}];
   const store = await openItemStore();
-  const read = jest.spyOn(store, 'read');
+  const read = jest.spyOn(store, 'readPage');
   const synchronize = jest.fn(async () => {await store.replaceSnapshot(fixture);});
   render(<App openStore={async () => store} createSync={() => ({initialized: true, identityPrefix: 'device', synchronize})} />);
   await saved();
@@ -744,7 +744,7 @@ test('M10 a committed late Save still registers work while suppressing old edito
 test('M10 background listener cleanup and stale callback guards leave a remounted draft untouched', async () => {
   const oldStore = await openItemStore('unit-old-root.db');
   await oldStore.replaceSnapshot([m08Input.seed]);
-  const oldRead = jest.spyOn(oldStore, 'read');
+  const oldRead = jest.spyOn(oldStore, 'readPage');
   const listen = jest.spyOn(DeviceEventEmitter, 'addListener');
   const first = render(<App openStore={async () => oldStore} />);
   await saved();
@@ -900,3 +900,119 @@ test.each(['resolve', 'reject'] as const)('M14 early observer disposal removes i
     expect(bridge.stopObservingForegroundNetwork.mock.calls).toEqual(bridge.observeForegroundNetwork.mock.calls);
   } finally {listen.mockRestore();}
 });
+
+const pageUnits = () => Array.from({length: 51}, (_, n) => ({id: `unit-page-${n + 1}`,
+  title: `Unit ${n + 1}`, completed: false, version: 1, updatedAt: 1700000000000}));
+
+test('M15 one initial page is virtualized; normal navigation retains an off-page draft and four durable edits', async () => {
+  const fixture = pageUnits();
+  const store = await openItemStore();
+  await store.replaceSnapshot(fixture);
+  const fullRead = jest.spyOn(store, 'read');
+  const pageRead = jest.spyOn(store, 'readPage');
+  render(<App openStore={async () => store} testIdentityPrefix="unit-local" />);
+  await saved();
+  expect(pageRead.mock.calls).toEqual([[]]);
+  expect(fullRead).not.toHaveBeenCalled();
+  expect(screen.getByLabelText('Item count: 51')).toBeTruthy();
+  const list = screen.UNSAFE_getByType(FlatList);
+  expect(list.props.data).toEqual(fixture.slice(0, 50));
+  expect(list.props.onEndReached).toBeUndefined();
+  expect(screen.getByText('Unit 1')).toBeTruthy();
+  expect(screen.queryByText('Unit 51')).toBeNull();
+  fireEvent.press(screen.getByLabelText('Last page'));
+  await saved();
+  expect(screen.getByLabelText('Page 2 of 2')).toBeTruthy();
+  fireEvent.press(screen.getByLabelText('Edit Unit 51'));
+  fireEvent.changeText(screen.getByLabelText('Edit item title'), 'Edited last unit');
+  fireEvent.press(screen.getByLabelText('First page'));
+  await saved();
+  await act(async () => {DeviceEventEmitter.emit('BackgroundSyncChanged'); await serializeSync(async () => {});});
+  expect(screen.getByLabelText('Edit item title').props.value).toBe('Edited last unit');
+  expect(editorMemory.current.editingId).toBe('unit-page-51');
+  fireEvent.press(screen.getByLabelText('Next page'));
+  await saved();
+  expect(screen.getByText('Unit 51')).toBeTruthy();
+  fireEvent.press(screen.getByLabelText('Previous page'));
+  await saved();
+  fireEvent.press(screen.getByLabelText('Save title'));
+  await saved();
+  fireEvent.press(screen.getByLabelText('Last page'));
+  await saved();
+  fireEvent.press(screen.getByLabelText('Mark Edited last unit complete'));
+  await saved();
+  fireEvent.changeText(screen.getByLabelText('New item title'), 'New page unit');
+  fireEvent.press(screen.getByLabelText('Add item'));
+  await saved();
+  expect(screen.getByText('New page unit')).toBeTruthy();
+  expect(screen.getByLabelText('Item count: 52')).toBeTruthy();
+  fireEvent.press(screen.getByLabelText('Delete New page unit'));
+  await saved();
+  expect(screen.getByLabelText('Item count: 51')).toBeTruthy();
+  expect(screen.getByRole('checkbox', {name: 'Mark Edited last unit incomplete'}).props.accessibilityState.checked).toBe(true);
+  expect(fullRead).not.toHaveBeenCalled();
+  expect(await store.read()).toEqual([...fixture.slice(0, 50), {...fixture[50], title: 'Edited last unit',
+    completed: true, version: 3, updatedAt: expect.any(Number)}]);
+  const pending = await store.readPending();
+  expect(pending.map(({kind, itemId, payload, dispatched}) => ({kind, itemId, payload, dispatched}))).toEqual([
+    {kind: 'rename', itemId: 'unit-page-51', payload: {title: 'Edited last unit', baseVersion: 1}, dispatched: false},
+    {kind: 'toggle', itemId: 'unit-page-51', payload: {completed: true, baseVersion: 1}, dispatched: false},
+    {kind: 'create', itemId: 'unit-local-001', payload: {id: 'unit-local-001', title: 'New page unit', completed: false}, dispatched: false},
+    {kind: 'delete', itemId: 'unit-local-001', payload: {baseVersion: 0}, dispatched: false},
+  ]);
+  expect(new Set(pending.map(item => item.clientMutationId)).size).toBe(4);
+  expect(await store.readLastSuccessfulRefresh()).toBeNull();
+});
+
+test('M15 a page refresh failure after COMMIT keeps the edit successful and offers a read-only list retry', async () => {
+  const store = await openItemStore();
+  await store.replaceSnapshot([m08Input.seed]);
+  render(<App openStore={async () => store} />);
+  await saved();
+  fireEvent.press(screen.getByLabelText('Edit Saved title'));
+  fireEvent.changeText(screen.getByLabelText('Edit item title'), 'Committed title');
+  failNextSql(/^SELECT .* FROM items ORDER BY rowid LIMIT/);
+  fireEvent.press(screen.getByLabelText('Save title'));
+  await saved();
+  expect(screen.getByText('Could not reload the list. Saved changes are retained.')).toBeTruthy();
+  expect(screen.queryByText('Change not saved')).toBeNull();
+  expect(screen.getByText('Saved title')).toBeTruthy();
+  expect(screen.getByLabelText('New item title').props.value).toBe('');
+  expect(NativeModules.BackgroundSync.schedule).toHaveBeenCalledTimes(1);
+  expect(await store.readPending()).toHaveLength(1);
+  expect((await store.read())[0].title).toBe('Committed title');
+  fireEvent.press(screen.getByRole('button', {name: /reload list/i}));
+  await saved();
+  expect(screen.getByText('Committed title')).toBeTruthy();
+  expect(screen.queryByText('Could not reload the list. Saved changes are retained.')).toBeNull();
+  expect(NativeModules.BackgroundSync.schedule).toHaveBeenCalledTimes(1);
+  expect(await store.readPending()).toHaveLength(1);
+});
+
+test('M15 a queued user page wins after an older background read without clearing the draft', async () => {
+  const store = await openItemStore();
+  await store.replaceSnapshot(pageUnits());
+  render(<App openStore={async () => store} />);
+  await saved();
+  fireEvent.press(screen.getByLabelText('Edit Unit 1'));
+  fireEvent.changeText(screen.getByLabelText('Edit item title'), 'Keep first draft');
+  let release!: () => void;
+  let started!: () => void;
+  const held = new Promise<void>(resolve => {release = resolve;});
+  const didStart = new Promise<void>(resolve => {started = resolve;});
+  const read = store.readPage.bind(store);
+  jest.spyOn(store, 'readPage').mockImplementationOnce(async index => {
+    const savedPage = await read(index); started(); await held; return savedPage;
+  });
+  try {
+    await act(async () => {DeviceEventEmitter.emit('BackgroundSyncChanged'); await didStart;});
+    fireEvent.press(screen.getByLabelText('Last page'));
+    await act(async () => {release(); await held; await serializeSync(async () => {});});
+    await saved();
+    expect(screen.getByLabelText('Page 2 of 2')).toBeTruthy();
+    expect(screen.getByText('Unit 51')).toBeTruthy();
+    expect(screen.getByLabelText('Edit item title').props.value).toBe('Keep first draft');
+    expect(editorMemory.current.editingId).toBe('unit-page-1');
+    expect(await store.readPending()).toEqual([]);
+  } finally {release();}
+});
diff --git a/__tests__/items.test.ts b/__tests__/items.test.ts
index a6725a9..7185644 100644
--- a/__tests__/items.test.ts
+++ b/__tests__/items.test.ts
@@ -1,7 +1,7 @@
 import {Item, itemsReducer} from '../src/items';
 import SQLite from 'react-native-sqlite-2';
 import {ItemMutation, itemToRow, openItemStore, PendingMutation, rowToItem, SCHEMA_VERSION} from '../src/itemStore';
-import {closeDatabases, connection, failNextSql} from './sqliteNative';
+import {closeDatabases, connection, failNextSql, nativeSqlite} from './sqliteNative';
 import {canonicalJson, mutationHash, mutationTarget} from '../src/mutationProtocol';
 
 const m06 = require('../verification/M06-inputs.json') as {
@@ -97,6 +97,49 @@ test('M02 unsupported schema is rejected without recreating or deleting existing
   expect(connection().prepare('PRAGMA user_version').get()?.user_version).toBe(SCHEMA_VERSION + 1);
 });
 
+test('M15 native pages are explicitly bounded, keep insertion order and clamp after last-page deletion', async () => {
+  // 51 rows cross one page boundary; this is not the fixed 2000-row device run.
+  const fixture: Item[] = Array.from({length: 51}, (_, n) => ({id: `page-${51 - n}`,
+    title: `Page unit ${n}`, completed: false, version: 1, updatedAt: 1700000000000 + n}));
+  const store = await openItemStore();
+  await store.replaceSnapshot(fixture);
+  const native = jest.spyOn(nativeSqlite, 'exec');
+  try {
+    expect(await store.readPage()).toEqual({items: fixture.slice(0, 50), index: 0, total: 51});
+    expect(await store.readPage(999)).toEqual({items: [fixture[50]], index: 1, total: 51});
+    expect(await store.readPage(-1)).toEqual({items: fixture.slice(0, 50), index: 0, total: 51});
+    const statements = native.mock.calls.flatMap(([, queries]) => queries);
+    expect(statements.filter(([sql]) => /\bFROM items\b/.test(sql))).toEqual([
+      ['SELECT COUNT(*) AS count FROM items', []],
+      ['SELECT id, title, completed, version, updated_at FROM items ORDER BY rowid LIMIT ? OFFSET ?', [50, 0]],
+      ['SELECT COUNT(*) AS count FROM items', []],
+      ['SELECT id, title, completed, version, updated_at FROM items ORDER BY rowid LIMIT ? OFFSET ?', [50, 50]],
+      ['SELECT COUNT(*) AS count FROM items', []],
+      ['SELECT id, title, completed, version, updated_at FROM items ORDER BY rowid LIMIT ? OFFSET ?', [50, 0]],
+    ]);
+    expect(statements.filter(([sql]) => /^BEGIN/.test(sql))).toHaveLength(3);
+    expect(statements.filter(([sql]) => /^END/.test(sql))).toHaveLength(3);
+  } finally {native.mockRestore();}
+  await store.mutate({type: 'delete', id: fixture[50].id});
+  expect(await store.readPage(1)).toEqual({items: fixture.slice(0, 50), index: 0, total: 50});
+  expect(await store.readPending()).toEqual([expect.objectContaining({kind: 'delete', itemId: fixture[50].id,
+    payload: {baseVersion: 1}, dispatched: false})]);
+});
+
+test('M15 empty and invalid pages cannot fall back to an unbounded query or publish a failed transaction', async () => {
+  const store = await openItemStore();
+  expect(await store.readPage(1)).toEqual({items: [], index: 0, total: 0});
+  const native = jest.spyOn(nativeSqlite, 'exec');
+  try {
+    await expect(store.readPage(NaN)).rejects.toThrow('Invalid Item page');
+    await expect(store.readPage(0.5)).rejects.toThrow('Invalid Item page');
+    expect(native).not.toHaveBeenCalled();
+  } finally {native.mockRestore();}
+  failNextSql(/^END/);
+  await expect(store.readPage()).rejects.toThrow('injected persistence failure');
+  expect(await store.readPending()).toEqual([]);
+});
+
 test('M03 baseline: separate local databases cannot observe another instance without synchronization', async () => {
   const first = await openItemStore('m03-first.db');
   const second = await openItemStore('m03-second.db');
diff --git a/android/app/build.gradle b/android/app/build.gradle
index bc8f7d2..62ebbf7 100644
--- a/android/app/build.gradle
+++ b/android/app/build.gradle
@@ -12,6 +12,14 @@ android {
     namespace "com.mse.reactnative"
     compileSdk rootProject.ext.compileSdkVersion
     buildToolsVersion rootProject.ext.buildToolsVersion
+    // M15 selects release explicitly; earlier debug instrumentation stays available.
+    testBuildType providers.gradleProperty("mseTestBuildType").getOrElse("debug")
+    buildTypes {
+        release {
+            debuggable false
+            signingConfig signingConfigs.debug
+        }
+    }
     defaultConfig {
         applicationId "com.mse.reactnative"
         minSdkVersion rootProject.ext.minSdkVersion
diff --git a/android/app/src/androidTest/assets/m15-inputs.json b/android/app/src/androidTest/assets/m15-inputs.json
new file mode 100644
index 0000000..262cc44
--- /dev/null
+++ b/android/app/src/androidTest/assets/m15-inputs.json
@@ -0,0 +1,43 @@
+{
+  "profile": "phase-1",
+  "thread": "M15",
+  "count": 2000,
+  "probeCount": 2,
+  "schemaVersion": 5,
+  "databaseLocation": "files/items.db",
+  "itemIdPrefix": "large-",
+  "titlePrefix": "Item ",
+  "decimalWidth": 4,
+  "completed": false,
+  "version": 1,
+  "updatedAt": 1700000000000,
+  "initialNextLocalId": 1,
+  "initialPending": 0,
+  "initialConflicts": 0,
+  "initialRowBound": 50,
+  "coldStartUiTimeoutSeconds": 60,
+  "networkTimeoutSeconds": 15,
+  "auditOutsideMeasuredWindow": true,
+  "finalOperations": {
+    "renameId": "large-2000",
+    "renameTitle": "Item 2000 edited",
+    "complete": true,
+    "createTitle": "Large local",
+    "deleteCreated": true,
+    "expectedItems": 2000,
+    "expectedPending": 4
+  },
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
diff --git a/android/app/src/androidTest/java/com/mse/reactnative/M15ReleaseDataTest.java b/android/app/src/androidTest/java/com/mse/reactnative/M15ReleaseDataTest.java
new file mode 100644
index 0000000..d291cfb
--- /dev/null
+++ b/android/app/src/androidTest/java/com/mse/reactnative/M15ReleaseDataTest.java
@@ -0,0 +1,237 @@
+package com.mse.reactnative;
+
+import static org.junit.Assert.*;
+
+import android.app.Instrumentation;
+import android.content.Context;
+import android.content.pm.ApplicationInfo;
+import android.database.Cursor;
+import android.net.ConnectivityManager;
+import android.os.Bundle;
+import android.os.Process;
+import android.os.SystemClock;
+import androidx.test.ext.junit.runners.AndroidJUnit4;
+import androidx.test.platform.app.InstrumentationRegistry;
+import androidx.test.runner.lifecycle.ActivityLifecycleMonitorRegistry;
+import androidx.test.runner.lifecycle.Stage;
+import io.requery.android.database.sqlite.SQLiteDatabase;
+import java.io.ByteArrayOutputStream;
+import java.io.File;
+import java.io.FileInputStream;
+import java.io.FileOutputStream;
+import java.io.InputStream;
+import java.nio.charset.StandardCharsets;
+import java.security.MessageDigest;
+import java.util.Locale;
+import org.json.JSONArray;
+import org.json.JSONObject;
+import org.junit.Test;
+import org.junit.runner.RunWith;
+
+/** Release-targeted setup/audit only. No Activity rule, launch, React state or UI. */
+@RunWith(AndroidJUnit4.class)
+public class M15ReleaseDataTest {
+    private final Instrumentation instrumentation = InstrumentationRegistry.getInstrumentation();
+    private final Context context = instrumentation.getTargetContext();
+    // RNSqlite2Module.getDatabase uses filesDir, not Android's databases folder.
+    private final File database = new File(context.getFilesDir(), "items.db");
+    private File output;
+    private JSONObject input;
+    private JSONObject result;
+
+    private static byte[] bytes(InputStream stream) throws Exception {
+        ByteArrayOutputStream output = new ByteArrayOutputStream();
+        byte[] block = new byte[8192];
+        int size;
+        while ((size = stream.read(block)) != -1) output.write(block, 0, size);
+        return output.toByteArray();
+    }
+
+    private static String sha(byte[] value) throws Exception {
+        StringBuilder text = new StringBuilder();
+        for (byte cell : MessageDigest.getInstance("SHA-256").digest(value))
+            text.append(String.format(Locale.ROOT, "%02x", cell & 255));
+        return text.toString();
+    }
+
+    private void write(String name, byte[] value) throws Exception {
+        try (FileOutputStream stream = new FileOutputStream(new File(output, name))) {
+            stream.write(value);
+            stream.getFD().sync();
+        }
+    }
+
+    private void writeJson(String name, JSONObject value) throws Exception {
+        write(name, value.toString(2).getBytes(StandardCharsets.UTF_8));
+    }
+
+    private void assertNoActivity() {
+        instrumentation.runOnMainSync(() -> {
+            for (Stage stage : Stage.values()) {
+                if (stage != Stage.DESTROYED)
+                    assertTrue("Seed/audit must not launch an Activity: " + stage,
+                            ActivityLifecycleMonitorRegistry.getInstance().getActivitiesInStage(stage).isEmpty());
+            }
+        });
+    }
+
+    private void initialize(String phase) throws Exception {
+        File external = context.getExternalFilesDir(null);
+        assertNotNull("External evidence directory must be available to the shell", external);
+        output = new File(external, "m15-" + phase);
+        assertTrue("Never overwrite an earlier helper invocation", output.mkdir());
+        byte[] asset;
+        try (InputStream stream = instrumentation.getContext().getAssets().open("m15-inputs.json")) {
+            asset = bytes(stream);
+        }
+        input = new JSONObject(new String(asset, StandardCharsets.UTF_8));
+        write("inputs.json", asset);
+        ApplicationInfo app = context.getApplicationInfo();
+        ConnectivityManager network = context.getSystemService(ConnectivityManager.class);
+        result = new JSONObject().put("status", "RUNNING").put("phase", phase)
+                .put("pid", Process.myPid()).put("package", context.getPackageName())
+                .put("buildType", BuildConfig.BUILD_TYPE).put("debug", BuildConfig.DEBUG)
+                .put("applicationFlags", app.flags).put("databasePath", database.getPath())
+                .put("outputPath", output.getPath()).put("inputSha256", sha(asset))
+                .put("activityLaunchRule", false).put("startedAt", System.currentTimeMillis())
+                .put("startedElapsedNs", SystemClock.elapsedRealtimeNanos())
+                .put("activeNetwork", network.getActiveNetwork() == null ? JSONObject.NULL : network.getActiveNetwork().toString());
+        writeJson("result.json", result);
+        assertEquals("release", BuildConfig.BUILD_TYPE);
+        assertFalse(BuildConfig.DEBUG);
+        assertEquals("Installed target must be nondebuggable", 0, app.flags & ApplicationInfo.FLAG_DEBUGGABLE);
+        assertEquals("com.mse.reactnative", context.getPackageName());
+        assertNull("Actual transport must be offline before seed/audit", network.getActiveNetwork());
+        assertNoActivity();
+    }
+
+    private JSONObject item(int ordinal) throws Exception {
+        String suffix = String.format(Locale.ROOT, "%04d", ordinal);
+        return new JSONObject().put("id", input.getString("itemIdPrefix") + suffix)
+                .put("title", input.getString("titlePrefix") + suffix)
+                .put("completed", false).put("version", 1)
+                .put("updatedAt", input.getLong("updatedAt"));
+    }
+
+    private void seed(int count) throws Exception {
+        assertTrue(count == input.getInt("probeCount") || count == input.getInt("count"));
+        assertFalse("Seed only after explicit pre-measurement app-data setup", database.exists());
+        try (SQLiteDatabase db = SQLiteDatabase.openOrCreateDatabase(database, null)) {
+            db.beginTransaction();
+            try {
+                JSONArray schema = input.getJSONArray("seedSql");
+                for (int i = 0; i < schema.length(); i++) db.execSQL(schema.getString(i));
+                for (int i = 1; i <= count; i++) {
+                    JSONObject value = item(i);
+                    db.execSQL("INSERT INTO items (id, title, completed, version, updated_at) VALUES (?, ?, 0, 1, ?)",
+                            new Object[]{value.getString("id"), value.getString("title"), value.getLong("updatedAt")});
+                    db.execSQL("INSERT INTO remote_versions (id, version, updated_at, deleted, canonical_item) VALUES (?, 1, ?, 0, ?)",
+                            new Object[]{value.getString("id"), value.getLong("updatedAt"), value.toString()});
+                }
+                db.setTransactionSuccessful();
+            } finally {
+                db.endTransaction();
+            }
+        }
+        result.put("seededItems", count).put("writerClosedBeforeAudit", true);
+    }
+
+    private static JSONObject row(Cursor cursor) throws Exception {
+        JSONObject value = new JSONObject();
+        for (int i = 0; i < cursor.getColumnCount(); i++) {
+            Object cell;
+            switch (cursor.getType(i)) {
+                case Cursor.FIELD_TYPE_NULL: cell = JSONObject.NULL; break;
+                case Cursor.FIELD_TYPE_INTEGER: cell = cursor.getLong(i); break;
+                case Cursor.FIELD_TYPE_FLOAT: cell = cursor.getDouble(i); break;
+                case Cursor.FIELD_TYPE_STRING: cell = cursor.getString(i); break;
+                default: throw new AssertionError("Unexpected blob in Item schema");
+            }
+            value.put(cursor.getColumnName(i), cell);
+        }
+        return value;
+    }
+
+    private JSONObject snapshot() throws Exception {
+        assertTrue("Audit never creates a missing database", database.isFile());
+        JSONObject raw = new JSONObject();
+        // Seed's writer is closed; external audit follows a recorded force-stop.
+        // MainApplication has no Item opener or Activity rule. Copy before our
+        // independent read-only connection, and bind every DB/WAL/SHM byte.
+        for (String suffix : new String[]{"", "-wal", "-shm"}) {
+            File source = new File(database.getPath() + suffix);
+            if (!source.exists()) continue;
+            byte[] data;
+            try (InputStream stream = new FileInputStream(source)) { data = bytes(stream); }
+            write("items.db" + suffix, data);
+            raw.put("items.db" + suffix, new JSONObject().put("sha256", sha(data)).put("bytes", data.length));
+        }
+        JSONObject tables = new JSONObject();
+        JSONArray schema = new JSONArray();
+        JSONObject observation;
+        try (SQLiteDatabase db = SQLiteDatabase.openDatabase(database.getPath(), null, SQLiteDatabase.OPEN_READONLY,
+                damaged -> { throw new AssertionError("Release audit refuses corruption recovery or deletion"); })) {
+            assertTrue(db.isReadOnly());
+            assertEquals(input.getInt("schemaVersion"), db.getVersion());
+            try (Cursor cursor = db.rawQuery("PRAGMA integrity_check", null)) {
+                assertTrue(cursor.moveToFirst());
+                assertEquals("ok", cursor.getString(0));
+                assertFalse(cursor.moveToNext());
+            }
+            String engineVersion;
+            try (Cursor cursor = db.rawQuery("SELECT sqlite_version()", null)) {
+                assertTrue(cursor.moveToFirst()); engineVersion = cursor.getString(0);
+            }
+            try (Cursor cursor = db.rawQuery("SELECT name,sql FROM sqlite_master WHERE type='table' ORDER BY name", null)) {
+                while (cursor.moveToNext()) schema.put(row(cursor));
+            }
+            for (int i = 0; i < schema.length(); i++) {
+                String name = schema.getJSONObject(i).getString("name");
+                assertTrue(name.matches("[A-Za-z_][A-Za-z_0-9]*"));
+                JSONArray values = new JSONArray();
+                try (Cursor cursor = db.rawQuery("SELECT * FROM \"" + name + "\" ORDER BY rowid", null)) {
+                    while (cursor.moveToNext()) values.put(row(cursor));
+                }
+                tables.put(name, values);
+            }
+            observation = new JSONObject().put("schemaVersion", db.getVersion())
+                    .put("engineClass", db.getClass().getName()).put("engineVersion", engineVersion)
+                    .put("readOnly", db.isReadOnly()).put("writerHandleBorrowed", false);
+        }
+        observation.put("schema", schema).put("tables", tables).put("rawFiles", raw)
+                .put("pid", Process.myPid()).put("measuredStartupWindow", false);
+        writeJson("database.json", observation);
+        return observation;
+    }
+
+    private void run(String phase, int count) throws Throwable {
+        try {
+            initialize(phase);
+            if (count > 0) seed(count);
+            JSONObject data = snapshot();
+            if (count > 0) {
+                assertEquals(count, data.getJSONObject("tables").getJSONArray("items").length());
+                assertEquals(count, data.getJSONObject("tables").getJSONArray("remote_versions").length());
+                assertEquals(0, data.getJSONObject("tables").getJSONArray("pending_mutations").length());
+                assertEquals(0, data.getJSONObject("tables").getJSONArray("mutation_conflicts").length());
+            }
+            assertNoActivity();
+            result.put("status", "PASS").put("noActivityAtExit", true).put("allHandlesClosed", true);
+        } catch (Throwable error) {
+            if (result != null) result.put("status", "FAIL").put("error", error.toString());
+            throw error;
+        } finally {
+            if (result != null) {
+                result.put("endedAt", System.currentTimeMillis()).put("endedElapsedNs", SystemClock.elapsedRealtimeNanos());
+                writeJson("result.json", result);
+                Bundle status = new Bundle();
+                status.putString("m15ResultPath", new File(output, "result.json").getPath());
+                instrumentation.sendStatus(0, status);
+            }
+        }
+    }
+
+    @Test public void seedSmall() throws Throwable { run("seed", 2); }
+    @Test public void seedLarge() throws Throwable { run("seed", 2000); }
+    @Test public void audit() throws Throwable { run("audit", 0); }
+}
diff --git a/android/app/src/main/res/xml/network_security_config.xml b/android/app/src/main/res/xml/network_security_config.xml
index 70d48a3..7702390 100644
--- a/android/app/src/main/res/xml/network_security_config.xml
+++ b/android/app/src/main/res/xml/network_security_config.xml
@@ -2,6 +2,6 @@
 <network-security-config>
     <base-config cleartextTrafficPermitted="false" />
     <domain-config cleartextTrafficPermitted="true">
-        <domain>10.0.2.2</domain>
+        <domain includeSubdomains="false">10.0.2.2</domain>
     </domain-config>
 </network-security-config>
diff --git a/patches/react-native-sqlite-2+3.6.3.patch b/patches/react-native-sqlite-2+3.6.3.patch
index 592fdc6..9baa728 100644
--- a/patches/react-native-sqlite-2+3.6.3.patch
+++ b/patches/react-native-sqlite-2+3.6.3.patch
@@ -16,3 +16,93 @@ index 095932c..0a0938a 100644
 -<manifest xmlns:android="http://schemas.android.com/apk/res/android"
 -          package="dog.craftz.sqlite_2">
 +<manifest xmlns:android="http://schemas.android.com/apk/res/android">
+diff --git a/node_modules/react-native-sqlite-2/android/src/main/java/dog/craftz/sqlite_2/RNSqlite2Module.java b/node_modules/react-native-sqlite-2/android/src/main/java/dog/craftz/sqlite_2/RNSqlite2Module.java
+--- a/node_modules/react-native-sqlite-2/android/src/main/java/dog/craftz/sqlite_2/RNSqlite2Module.java
++++ b/node_modules/react-native-sqlite-2/android/src/main/java/dog/craftz/sqlite_2/RNSqlite2Module.java
+@@ -19,13 +19,17 @@
+ import io.requery.android.database.sqlite.SQLiteStatement;
+ import android.os.Handler;
+ import android.os.HandlerThread;
++import android.os.Process;
++import android.os.SystemClock;
+
+ import org.json.JSONArray;
+ import org.json.JSONException;
++import org.json.JSONObject;
+
+ import java.io.File;
+ import java.util.HashMap;
+ import java.util.Map;
++import java.util.concurrent.atomic.AtomicLong;
+
+ @ReactModule(name = RNSqlite2Module.NAME)
+ public class RNSqlite2Module extends ReactContextBaseJavaModule {
+@@ -36,6 +40,7 @@
+   private static final boolean DEBUG_MODE = false;
+
+   private static final String TAG = RNSqlite2Module.class.getSimpleName();
++  private static final AtomicLong SELECT_SEQUENCE = new AtomicLong();
+
+   private static final Object[][] EMPTY_ROWS = new Object[][]{};
+   private static final String[] EMPTY_COLUMNS = new String[]{};
+@@ -205,29 +210,57 @@
+   // do a select operation
+   private SQLitePLuginResult doSelectInBackgroundAndPossiblyThrow(String sql, ReadableArray queryArgs, SQLiteDatabase db) {
+     debug("\"all\" query: %s", sql);
++    final long sequence = SELECT_SEQUENCE.incrementAndGet();
++    final long started = SystemClock.elapsedRealtimeNanos();
++    String[] bindArgs = null;
++    String[] observedColumns = EMPTY_COLUMNS;
++    int cursorRows = -1;
++    int materializedRows = 0;
++    boolean complete = false;
+     Cursor cursor = null;
+     try {
+-      String[] bindArgs = convertParamsToStringArray(queryArgs);
++      bindArgs = convertParamsToStringArray(queryArgs);
+       cursor = db.rawQuery(sql, bindArgs);
+       int numRows = cursor.getCount();
++      cursorRows = numRows;
+       if (numRows == 0) {
++        complete = true;
+         return EMPTY_RESULT;
+       }
+       int numColumns = cursor.getColumnCount();
+       Object[][] rows = new Object[numRows][];
+       String[] columnNames = cursor.getColumnNames();
++      observedColumns = columnNames;
+       for (int i = 0; cursor.moveToNext(); i++) {
+         Object[] row = new Object[numColumns];
+         for (int j = 0; j < numColumns; j++) {
+           row[j] = getValueFromCursor(cursor, j, cursor.getType(j));
+         }
+         rows[i] = row;
++        materializedRows++;
+       }
+       debug("returning %d rows", numRows);
++      complete = true;
+       return new SQLitePLuginResult(rows, columnNames, 0, 0, null);
+     } finally {
+-      if (cursor != null) {
+-        cursor.close();
++      boolean closed = false;
++      try {
++        if (cursor != null) cursor.close();
++        closed = true;
++      } finally {
++        // Count the existing row-building loop before bridge encoding/JS slicing.
++        // No extra query, cursor walk, row payload or per-row log is introduced.
++        try {
++          Log.i("MSESqlRows", new JSONObject().put("sequence", sequence)
++              .put("pid", Process.myPid()).put("database", db.getPath())
++              .put("sql", sql).put("bindArgs", bindArgs == null ? JSONObject.NULL : new JSONArray(bindArgs))
++              .put("columns", new JSONArray(observedColumns)).put("cursorRows", cursorRows)
++              .put("materializedRows", materializedRows).put("complete", complete && closed)
++              .put("startedElapsedNs", started).put("endedElapsedNs", SystemClock.elapsedRealtimeNanos())
++              .toString());
++        } catch (Throwable ignored) {
++          // Observation must never change the database result or its exception.
++        }
+       }
+     }
+   }
diff --git a/scripts/verify_m15.py b/scripts/verify_m15.py
new file mode 100644
index 0000000..4520240
--- /dev/null
+++ b/scripts/verify_m15.py
@@ -0,0 +1,552 @@
+#!/usr/bin/env python3
+"""Root-owned release helper probe or one frozen M15 2000-Item scenario.
+
+The probe seeds/audits two rows without any Activity. The full baseline ends
+instrumentation before one ordinary offline cold launch; it never injects JS
+state or substitutes an instrumentation runner for that measured launch.
+"""
+import argparse
+from decimal import Decimal, InvalidOperation
+import hashlib
+import json
+import os
+from pathlib import Path
+import re
+import subprocess
+import time
+from urllib.parse import quote
+import xml.etree.ElementTree as ET
+from verify_m07 import package_in_live_activities
+from verify_m14 import native_database
+
+PACKAGE = "com.mse.reactnative"
+HELPER = PACKAGE + ".M15ReleaseDataTest"
+RUNNER = PACKAGE + ".test/androidx.test.runner.AndroidJUnitRunner"
+KEYS = ("airplane_mode_on", "wifi_on", "mobile_data")
+ONLINE = dict(zip(KEYS, ("0", "1", "1")))
+OFFLINE = dict(zip(KEYS, ("1", "0", "0")))
+
+
+def sha(path):
+    return hashlib.sha256(Path(path).read_bytes()).hexdigest()
+
+
+def audit_database(directory, analysis, inputs):
+    helper = json.loads((directory / "result.json").read_text())
+    assert helper["status"] == "PASS" and helper["buildType"] == "release"
+    assert not helper["debug"] and not helper["applicationFlags"] & 2
+    assert helper["noActivityAtExit"] and helper["allHandlesClosed"]
+    assert helper["activeNetwork"] is None and not helper["activityLaunchRule"]
+    assert helper["inputSha256"] == sha(directory / "inputs.json")
+    assert json.loads((directory / "inputs.json").read_text()) == inputs
+    reported = json.loads((directory / "database.json").read_text())
+    assert reported["readOnly"] and not reported["writerHandleBorrowed"]
+    assert reported["engineClass"] == "io.requery.android.database.sqlite.SQLiteDatabase"
+    assert reported["engineVersion"] == "3.49.0" and not reported["measuredStartupWindow"]
+    actual, hashes = native_database(directory / "items.db", analysis)
+    assert actual == {key: reported[key] for key in ("schemaVersion", "tables")}
+    assert actual["schemaVersion"] == inputs["schemaVersion"] == 5
+    assert hashes == {name: value["sha256"] for name, value in reported["rawFiles"].items()}
+    return {"helperPid": helper["pid"], "rawSha256": hashes, "tables": actual["tables"]}
+
+
+def audit_seed(directory, analysis, inputs, count):
+    snapshot = audit_database(directory, analysis, inputs)
+    tables = snapshot["tables"]
+    expected = [{"id": inputs["itemIdPrefix"] + f"{n:04d}",
+                 "title": inputs["titlePrefix"] + f"{n:04d}", "completed": 0,
+                 "version": 1, "updated_at": inputs["updatedAt"]} for n in range(1, count + 1)]
+    assert tables["items"] == expected
+    assert tables["pending_mutations"] == tables["mutation_conflicts"] == []
+    assert tables["local_identity"] == [{"singleton": 1, "next_id": 1}]
+    assert tables["sync_metadata"] == [{"singleton": 1, "last_successful_refresh_at": None, "last_acknowledgement": None}]
+    assert tables["sqlite_sequence"] == [{"name": "pending_mutations", "seq": 0}]
+    assert len(tables["remote_versions"]) == count
+    for item, known in zip(expected, tables["remote_versions"]):
+        canonical = {"id": item["id"], "title": item["title"], "completed": False,
+                     "version": 1, "updatedAt": inputs["updatedAt"]}
+        assert known == {"id": item["id"], "version": 1, "updated_at": inputs["updatedAt"],
+                         "deleted": 0, "canonical_item": known["canonical_item"]}
+        assert json.loads(known["canonical_item"]) == canonical
+    return {**snapshot, "rows": count}
+
+
+def audit_edits(directory, analysis, inputs, seeded, created_id, device_time):
+    snapshot = audit_database(directory, analysis, inputs)
+    tables = snapshot["tables"]
+    operations = inputs["finalOperations"]
+    expected = [dict(item) for item in seeded["tables"]["items"]]
+    assert created_id not in {item["id"] for item in expected} and created_id.endswith("-001")
+    last = next(item for item in tables["items"] if item["id"] == operations["renameId"])
+    assert isinstance(last["updated_at"], int) and device_time[0] <= last["updated_at"] <= device_time[1]
+    expected[-1].update(title=operations["renameTitle"], completed=1, version=3, updated_at=last["updated_at"])
+    assert tables["items"] == expected and len(expected) == operations["expectedItems"]
+    assert tables["remote_versions"] == seeded["tables"]["remote_versions"]
+    assert tables["sync_metadata"] == seeded["tables"]["sync_metadata"]
+    assert tables["mutation_conflicts"] == []
+    assert tables["local_identity"] == [{"singleton": 1, "next_id": 2}]
+    assert tables["sqlite_sequence"] == [{"name": "pending_mutations", "seq": 4}]
+    envelopes = [
+        ("rename", operations["renameId"], {"title": operations["renameTitle"], "baseVersion": 1}),
+        ("toggle", operations["renameId"], {"completed": True, "baseVersion": 1}),
+        ("create", created_id, {"id": created_id, "title": operations["createTitle"], "completed": False}),
+        ("delete", created_id, {"baseVersion": 0}),
+    ]
+    pending = tables["pending_mutations"]
+    assert len(pending) == operations["expectedPending"] == 4
+    identities = []
+    for index, (row, (kind, item_id, payload)) in enumerate(zip(pending, envelopes), 1):
+        method = "POST" if kind == "create" else "DELETE" if kind == "delete" else "PATCH"
+        path = "/items" if kind == "create" else "/items/" + quote(item_id, safe="~()*!.'-")
+        canonical = json.dumps({"method": method, "path": path, "payload": payload},
+                               sort_keys=True, separators=(",", ":"), ensure_ascii=False)
+        digest = hashlib.sha256(canonical.encode()).hexdigest()
+        assert json.loads(row["payload"]) == payload
+        identity = row["client_mutation_id"]
+        assert isinstance(identity, str) and identity
+        assert row == {"sequence": index, "kind": kind, "item_id": item_id, "payload": row["payload"],
+                       "client_mutation_id": identity, "payload_hash": digest, "terminal_error": None, "dispatched": 0}
+        identities.append({"clientMutationId": identity, "payloadHash": digest, "canonical": canonical})
+    assert len({value["clientMutationId"] for value in identities}) == 4
+    return {**snapshot, "rows": len(expected), "createdId": created_id, "finalUpdatedAt": last["updated_at"],
+            "createdTimestampRetainedAfterDelete": False, "identities": identities}
+
+
+def materializations(raw, pid):
+    events = []
+    for line in raw.splitlines():
+        if "MSESqlRows" not in line:
+            continue
+        match = re.search(r"\bMSESqlRows\s*:\s*(\{.*)$", line)
+        assert match, ("Missing or truncated native trace", line)
+        event = json.loads(match.group(1))
+        assert event["pid"] == pid and event["database"].endswith("/files/items.db")
+        assert event["complete"] and event["materializedRows"] == event["cursorRows"]
+        assert 0 <= event["startedElapsedNs"] <= event["endedElapsedNs"]
+        events.append(event)
+    assert events, "No native materialization observation; cannot infer zero rows"
+    assert sorted(event["sequence"] for event in events) == list(range(1, len(events) + 1))
+    item_events = []
+    for event in events:
+        sql = event["sql"]
+        item_source = bool(re.search(r"\bFROM\s+[\"`]?items\b", sql, re.I))
+        canonical_source = bool(re.search(r"\bFROM\s+[\"`]?remote_versions\b", sql, re.I))
+        count_aggregate = bool(re.match(r"\s*SELECT\s+COUNT\s*\(\s*\*\s*\)", sql, re.I))
+        if (item_source or canonical_source) and not count_aggregate:
+            item_events.append(event)
+    return {"events": events, "itemQueries": item_events,
+            "materializedItemRows": sum(event["materializedRows"] for event in item_events),
+            "allNativeSelectRows": sum(event["materializedRows"] for event in events),
+            "boundary": "Completed native cursor row objects before bridge encoding or JS slicing",
+            "window": "Every complete native SELECT in the supplied measured-PID log"}
+
+
+def initial_page_window(observed, row_bound, expected_offset):
+    # A scalar COUNT for the deliberate navigation may precede this boundary;
+    # it materializes no Item. All prior entity reads count, including callbacks.
+    def page_offset(event):
+        if re.search(r"\bFROM items ORDER BY rowid LIMIT \? OFFSET \?$", event["sql"]):
+            # The pinned bridge uses Double.toString for JS numeric bindings.
+            # Preserve exact integer requirements while accepting "50.0".
+            values = []
+            for value in event["bindArgs"]:
+                try:
+                    number = Decimal(value)
+                except (InvalidOperation, TypeError, ValueError) as error:
+                    raise AssertionError("Invalid native numeric binding") from error
+                assert number.is_finite() and number == number.to_integral_value()
+                values.append(int(number))
+            assert len(values) == 2
+            limit, offset = values
+            assert limit == row_bound
+            return offset
+        return None
+    boundary = next(event for event in observed["events"] if (page_offset(event) or 0) > 0)
+    assert page_offset(boundary) == expected_offset
+    events = [event for event in observed["events"] if event["sequence"] < boundary["sequence"]]
+    item_events = [event for event in observed["itemQueries"] if event["sequence"] < boundary["sequence"]]
+    assert item_events and all(page_offset(event) == 0 for event in item_events)
+    total = sum(event["materializedRows"] for event in item_events)
+    assert 0 < total <= row_bound
+    return {"events": events, "itemQueries": item_events, "materializedItemRows": total,
+            "allNativeSelectRows": sum(event["materializedRows"] for event in events),
+            "firstUserPageQuery": boundary,
+            "window": "All Item/canonical entity reads preceding the first deliberate positive-offset page query"}
+
+
+def usable(xml):
+    nodes = list(ET.fromstring(xml).iter("node"))
+    labels = {node.get("content-desc"): node for node in nodes}
+    required = ("Local storage ready", "Pending uploads: 0", "Edit Item 0001", "Mark Item 0001 complete")
+    if not all(label in labels for label in required):
+        return False
+    if any(labels[label].get("enabled") != "true" for label in required[2:]):
+        return False
+    titles = [node for node in nodes if node.get("text") == "Item 0001"]
+    return bool(titles) and any(node.get("bounds") not in (None, "[0,0][0,0]") for node in titles)
+
+
+def main():
+    parser = argparse.ArgumentParser()
+    parser.add_argument("--mode", choices=("probe", "baseline", "implementation"), required=True)
+    parser.add_argument("--adb", required=True)
+    parser.add_argument("--serial", default="emulator-5554")
+    parser.add_argument("--apk", required=True)
+    parser.add_argument("--test-apk", required=True)
+    parser.add_argument("--evidence", required=True)
+    args = parser.parse_args()
+    root = Path(__file__).resolve().parent.parent
+    evidence = Path(args.evidence).resolve()
+    evidence.mkdir(parents=True, exist_ok=False)
+    inputs_path = root / "verification/phase-1/M15-inputs.json"
+    assert inputs_path.read_bytes() == (root / "android/app/src/androidTest/assets/m15-inputs.json").read_bytes()
+    inputs = json.loads(inputs_path.read_text())
+    commands = []
+    result = {"status": "RUNNING", "mode": args.mode, "hostPid": os.getpid(),
+              "apkSha256": sha(args.apk), "testApkSha256": sha(args.test_apk),
+              "harnessSha256": sha(__file__), "inputsSha256": sha(inputs_path),
+              "full2000Invocation": args.mode != "probe", "fixtureUsed": False,
+              "ordinaryLaunches": 0, "initialQueryAndRenderChanged": args.mode == "implementation"}
+    began = time.monotonic()
+    original_network = None
+
+    def save():
+        (evidence / "result.json").write_text(json.dumps(result, indent=2) + "\n")
+
+    def adb(label, *parts, check=True, binary=False, timeout=60):
+        command = [args.adb, "-s", args.serial, *parts]
+        record = {"label": label, "argv": command, "timeoutSeconds": timeout,
+                  "startedAt": int(time.time() * 1000), "startedMonotonicNs": time.monotonic_ns()}
+        try:
+            done = subprocess.run(command, capture_output=True, timeout=timeout)
+            raw, err = done.stdout, done.stderr
+            record["exit"] = done.returncode
+        except subprocess.TimeoutExpired as error:
+            raw, err = error.stdout or b"", error.stderr or b""
+            record.update(exit=None, error=repr(error))
+        for key, data in (("stdout", raw), ("stderr", err)):
+            filename = f"adb-{len(commands):04d}-{label}.{key}"
+            (evidence / filename).write_bytes(data)
+            record[key] = filename
+        record.update(endedAt=int(time.time() * 1000), endedMonotonicNs=time.monotonic_ns())
+        commands.append(record)
+        (evidence / "commands.json").write_text(json.dumps(commands, indent=2) + "\n")
+        assert record["exit"] is not None, record
+        if check:
+            assert record["exit"] == 0, record
+        return raw if binary else raw.decode(errors="replace").strip()
+
+    def network(label, expected=None):
+        if expected is not None:
+            adb(label + "-airplane", "shell", "cmd", "connectivity", "airplane-mode", "enable" if expected == OFFLINE else "disable")
+            adb(label + "-wifi", "shell", "svc", "wifi", "disable" if expected == OFFLINE else "enable")
+            adb(label + "-data", "shell", "svc", "data", "disable" if expected == OFFLINE else "enable")
+        deadline = time.monotonic() + inputs["networkTimeoutSeconds"]
+        while True:
+            settings = {key: adb(label + "-" + key, "shell", "settings", "get", "global", key) for key in KEYS}
+            state = adb(label + "-connectivity", "shell", "dumpsys", "connectivity")
+            offline = "Active default network: none" in state
+            if expected is None or (settings == expected and offline == (expected == OFFLINE)):
+                assert offline == (settings == OFFLINE)
+                return settings
+            assert time.monotonic() < deadline, (expected, settings)
+            time.sleep(0.1)
+
+    def stopped(label):
+        adb(label + "-stop", "shell", "am", "force-stop", PACKAGE)
+        deadline = time.monotonic() + 10
+        quiet_since = None
+        while True:
+            left = deadline - time.monotonic()
+            assert left > 0, "Target teardown did not settle"
+            pid = adb(label + "-pid", "shell", "pidof", PACKAGE, check=False, timeout=left)
+            assert commands[-1]["exit"] in (0, 1)
+            left = deadline - time.monotonic()
+            assert left > 0, "Target teardown did not settle"
+            activities = adb(label + "-activities", "shell", "dumpsys", "activity", "activities", timeout=left)
+            if not pid and not package_in_live_activities(activities):
+                quiet_since = time.monotonic() if quiet_since is None else quiet_since
+                if time.monotonic() - quiet_since >= 1:
+                    return
+            else:
+                quiet_since = None
+            assert time.monotonic() < deadline, "Target did not stop"
+            time.sleep(0.1)
+
+    def helper(method, phase):
+        raw = adb(phase + "-instrumentation", "shell", "am", "instrument", "-w", "-r",
+                  "-e", "class", HELPER + "#" + method, RUNNER, timeout=60)
+        assert "OK (1 test)" in raw and "FAILURES" not in raw, raw
+        destination = evidence / phase
+        adb(phase + "-pull", "pull", "/sdcard/Android/data/" + PACKAGE + "/files/m15-" + phase, str(destination))
+        return destination
+
+    def element(xml, label, attribute="content-desc"):
+        return next((node for node in ET.fromstring(xml).iter("node") if node.get(attribute) == label), None)
+
+    def bounds(node):
+        x1, y1, x2, y2 = map(int, re.findall(r"-?\d+", node.get("bounds", "")))
+        assert x2 > x1 and y2 > y1, node.attrib
+        return x1, y1, x2, y2
+
+    def later_ui(label, predicate, deadline=None):
+        if deadline is None:
+            deadline = time.monotonic() + inputs["coldStartUiTimeoutSeconds"]
+        observations = result.setdefault("laterUiObservations", [])
+        while time.monotonic() < deadline:
+            path = f"/sdcard/mse-rn-m15-{result['measuredPid']}-{result['startMonotonicNs']}-action-{len(observations)}.xml"
+            dump = adb(label + "-dump", "shell", "uiautomator", "dump", path, check=False,
+                       timeout=deadline - time.monotonic())
+            observation = {"label": label, "path": path, "dumpCommand": len(commands) - 1,
+                           "freshDump": commands[-1]["exit"] == 0 and path in dump, "matched": False}
+            xml = None
+            if observation["freshDump"]:
+                remaining = deadline - time.monotonic()
+                assert remaining > 0, "UI deadline elapsed before reading fresh dump"
+                xml = adb(label + "-read", "shell", "cat", path, timeout=remaining)
+                observation["readCommand"] = len(commands) - 1
+                try:
+                    observation["matched"] = bool(predicate(xml))
+                except ET.ParseError as parse_error:
+                    observation["parseError"] = repr(parse_error)
+            observations.append(observation)
+            save()
+            if observation["matched"]:
+                return xml
+        raise AssertionError("UI condition not observed: " + label)
+
+    def tap(label, xml, target):
+        node = element(xml, target)
+        assert node is not None and node.get("enabled") == "true", target
+        x1, y1, x2, y2 = bounds(node)
+        adb(label, "shell", "input", "tap", str((x1 + x2) // 2), str((y1 + y2) // 2))
+        result.setdefault("uiActions", []).append({"label": label, "target": target, "command": len(commands) - 1})
+        save()
+
+    def ready(xml, pending, total):
+        return all(element(xml, label) is not None for label in
+                   ("Local storage ready", f"Pending uploads: {pending}", f"Item count: {total}"))
+
+    def visible_control(xml, label):
+        node = element(xml, label)
+        if node is None or node.get("enabled") != "true":
+            return False
+        try:
+            bounds(node)
+            return True
+        except (AssertionError, ValueError):
+            return False
+
+    def reach_last_item():
+        target = "Edit " + inputs["titlePrefix"] + f"{inputs['count']:04d}"
+        deadline = time.monotonic() + inputs["coldStartUiTimeoutSeconds"]
+        scroll = 0
+        while time.monotonic() < deadline:
+            xml = later_ui("last-item-scroll-" + str(scroll), lambda _: True, deadline)
+            if visible_control(xml, target):
+                return xml
+            containers = [node for node in ET.fromstring(xml).iter("node")
+                          if node.get("scrollable") == "true" and node.get("package") == PACKAGE]
+            assert len(containers) == 1, "Expected one real virtualized list scroll container"
+            x1, y1, x2, y2 = bounds(containers[0])
+            margin = max(1, (y2 - y1) // 8)
+            adb("scroll-last-item-" + str(scroll), "shell", "input", "swipe",
+                str((x1 + x2) // 2), str(y2 - margin), str((x1 + x2) // 2), str(y1 + margin), "150")
+            scroll += 1
+        raise AssertionError("Item2000 not reachable by normal last-page scrolling")
+
+    def enter_title(label, xml, old, new, editing):
+        target = "Edit item title" if editing else "New item title"
+        field = element(xml, target)
+        assert field is not None and field.get("class") == "android.widget.EditText"
+        if editing:
+            assert field.get("text") == old
+        else:
+            # UiAutomator may report the hint as text. The disabled Add control
+            # and normal post-Save reset are the empty-draft evidence, not that hint.
+            add = element(xml, "Add item")
+            assert add is not None and add.get("enabled") == "false"
+        tap(label + "-focus", xml, target)
+        if old:
+            adb(label + "-end", "shell", "input", "keyevent", "KEYCODE_MOVE_END")
+            adb(label + "-clear", "shell", "input", "keyevent", *(["KEYCODE_DEL"] * len(old)))
+        adb(label + "-text", "shell", "input", "text", new.replace(" ", "%s"))
+        submit = "Save title" if editing else "Add item"
+        return later_ui(label + "-entered", lambda value: element(value, target) is not None
+                        and element(value, target).get("text") == new and visible_control(value, submit))
+
+    def exercise_ui(pid_text):
+        operations = inputs["finalOperations"]
+        last_page = (inputs["count"] + inputs["initialRowBound"] - 1) // inputs["initialRowBound"]
+        xml = later_ui("before-first-navigation", lambda value: usable(value) and visible_control(value, "Last page"))
+        result["firstNavigationCommand"] = len(commands)
+        tap("navigate-last-page", xml, "Last page")
+        later_ui("last-page-ready", lambda value: ready(value, 0, inputs["count"])
+                 and element(value, f"Page {last_page} of {last_page}") is not None)
+        xml = reach_last_item()
+        (evidence / "last-item-before-edit.xml").write_text(xml)
+        assert adb("before-edits-pid", "shell", "pidof", PACKAGE) == pid_text
+        device_start = int(adb("before-edits-device-seconds", "shell", "date", "+%s")) * 1000
+        tap("edit-last-item", xml, "Edit " + inputs["titlePrefix"] + f"{inputs['count']:04d}")
+        xml = later_ui("last-item-editor", lambda value: element(value, "Edit item title") is not None)
+        xml = enter_title("rename", xml, inputs["titlePrefix"] + f"{inputs['count']:04d}", operations["renameTitle"], True)
+        tap("rename-submit", xml, "Save title")
+        xml = later_ui("rename-committed", lambda value: ready(value, 1, inputs["count"])
+                       and visible_control(value, "Mark " + operations["renameTitle"] + " complete"))
+        tap("complete-submit", xml, "Mark " + operations["renameTitle"] + " complete")
+        xml = later_ui("complete-committed", lambda value: ready(value, 2, inputs["count"])
+                       and visible_control(value, "Mark " + operations["renameTitle"] + " incomplete"))
+        assert element(xml, "Mark " + operations["renameTitle"] + " incomplete").get("checked") == "true"
+        xml = enter_title("create", xml, "", operations["createTitle"], False)
+        tap("create-submit", xml, "Add item")
+        xml = later_ui("create-committed", lambda value: ready(value, 3, inputs["count"] + 1)
+                       and visible_control(value, "Last page"))
+        # The new Item is on the newly created final page. Reach it through the
+        # same production navigation; no automatic jump or database injection.
+        tap("navigate-created-page", xml, "Last page")
+        xml = later_ui("created-page-ready", lambda value: ready(value, 3, inputs["count"] + 1)
+                       and visible_control(value, "Delete " + operations["createTitle"]))
+        title = element(xml, operations["createTitle"], "text")
+        assert title is not None and title.get("resource-id", "").startswith("item-title-")
+        created_id = title.get("resource-id")[len("item-title-"):]
+        result["createdIdFromUi"] = created_id
+        (evidence / "created-item.xml").write_text(xml)
+        tap("delete-created-submit", xml, "Delete " + operations["createTitle"])
+        xml = later_ui("delete-committed", lambda value: ready(value, 4, inputs["count"])
+                       and element(value, f"Page {last_page} of {last_page}") is not None)
+        (evidence / "final-ui.xml").write_text(xml)
+        (evidence / "final-ui.png").write_bytes(adb("final-ui-screenshot", "exec-out", "screencap", "-p", binary=True))
+        result["deviceEditTimeBounds"] = [device_start, int(adb("after-edits-device-seconds", "shell", "date", "+%s")) * 1000 + 999]
+        assert adb("after-edits-pid", "shell", "pidof", PACKAGE) == pid_text
+        result["offlineAfterEdits"] = network("after-edits-network")
+        assert result["offlineAfterEdits"] == OFFLINE
+        return created_id
+
+    save()
+    try:
+        original_network = network("original")
+        assert original_network == ONLINE, "Root lease must begin in its recorded online state"
+        assert adb("install-release", "install", "-r", args.apk).endswith("Success")
+        assert adb("install-helper", "install", "-r", "-t", args.test_apk).endswith("Success")
+        adb("installed-package", "shell", "dumpsys", "package", PACKAGE)
+        adb("installed-path", "shell", "pm", "path", PACKAGE)
+        stopped("before-seed")
+        assert adb("fresh-input-state", "shell", "pm", "clear", PACKAGE) == "Success"
+        result["offlineBeforeSeed"] = network("offline", OFFLINE)
+        count = inputs["probeCount"] if args.mode == "probe" else inputs["count"]
+        seed = helper("seedSmall" if args.mode == "probe" else "seedLarge", "seed")
+        seeded = audit_seed(seed, evidence / "seed-analysis", inputs, count)
+        result["seed"] = {key: value for key, value in seeded.items() if key != "tables"}
+        stopped("after-seed")
+        # No install, clear, seed, instrumentation, provider or JS injection below
+        # this boundary until the measured normal launch has been stopped.
+        result["postSeedBoundaryCommand"] = len(commands)
+        created_id = None
+        if args.mode != "probe":
+            adb("clear-logcat-before-measurement", "logcat", "-c")
+            start_ns = time.monotonic_ns()
+            result["ordinaryLaunches"] = 1
+            result["startMonotonicNs"] = start_ns
+            save()
+            adb("ordinary-cold-launch", "shell", "am", "start", "-W", "-n", PACKAGE + "/.MainActivity")
+            pid_text = adb("measured-pid", "shell", "pidof", PACKAGE)
+            assert re.fullmatch(r"[0-9]+", pid_text), pid_text
+            pid = int(pid_text)
+            assert pid != seeded["helperPid"]
+            result["measuredPid"] = pid
+            deadline = time.monotonic() + inputs["coldStartUiTimeoutSeconds"]
+            observations = result["initialUiObservations"] = []
+            while True:
+                # App-data clear cannot remove old /sdcard dumps. A unique path
+                # plus a successful dump prevents accepting stale UI evidence.
+                path = f"/sdcard/mse-rn-m15-{pid}-{start_ns}-{len(observations)}.xml"
+                dump = adb("initial-ui-dump", "shell", "uiautomator", "dump", path, check=False)
+                observed = {"path": path, "dumpCommand": len(commands) - 1,
+                            "freshDump": commands[-1]["exit"] == 0 and path in dump, "usable": False}
+                if observed["freshDump"]:
+                    xml = adb("initial-ui-read", "shell", "cat", path)
+                    try:
+                        observed["usable"] = usable(xml)
+                    except ET.ParseError as parse_error:
+                        observed["parseError"] = repr(parse_error)
+                observations.append(observed)
+                save()
+                if observed["usable"]:
+                    result["firstUsableMonotonicNs"] = time.monotonic_ns()
+                    result["firstUsableSeconds"] = (result["firstUsableMonotonicNs"] - start_ns) / 1e9
+                    (evidence / "first-usable.xml").write_text(xml)
+                    assert "Item count: 2000" in xml
+                    break
+                assert time.monotonic() < deadline, "First usable Item0001 was not observed"
+            save()
+            (evidence / "first-usable.png").write_bytes(adb("first-usable-screenshot", "exec-out", "screencap", "-p", binary=True))
+            gfx = adb("initial-gfxinfo", "shell", "dumpsys", "gfxinfo", PACKAGE)
+            views = re.search(r"Total Views:\s*([0-9]+)", gfx)
+            result["supplementalNativeTotalViews"] = int(views.group(1)) if views else None
+            result["nativeViewsAreNotItemRowCount"] = True
+            result["offlineAfterInitialLoad"] = network("initial-network")
+            assert result["offlineAfterInitialLoad"] == OFFLINE
+            assert adb("initial-pid-stable", "shell", "pidof", PACKAGE) == pid_text
+            adb("initial-activities", "shell", "dumpsys", "activity", "activities")
+            before_navigation = None
+            if args.mode == "implementation":
+                before_navigation = materializations(adb("before-navigation-logcat", "logcat", "-d", "-v", "threadtime"), pid)
+                (evidence / "before-navigation-materializations.json").write_text(json.dumps(before_navigation, indent=2) + "\n")
+                assert 0 < before_navigation["materializedItemRows"] <= inputs["initialRowBound"]
+                result["initialSelectCallsBeforeNavigation"] = len(before_navigation["events"])
+                created_id = exercise_ui(pid_text)
+            result["measuredWindowEndCommand"] = len(commands)
+            stopped("end-measured-window")
+            # Keep the complete log through stop. In implementation mode, later
+            # explicit mutation reads are retained separately, never hidden or
+            # miscounted as the initial window.
+            raw = adb("initial-logcat", "logcat", "-d", "-v", "threadtime")
+            observed = materializations(raw, pid)
+            if args.mode == "implementation":
+                assert observed["events"][:len(before_navigation["events"])] == before_navigation["events"]
+                initial = initial_page_window(observed, inputs["initialRowBound"],
+                                              ((inputs["count"] - 1) // inputs["initialRowBound"]) * inputs["initialRowBound"])
+                (evidence / "all-materializations.json").write_text(json.dumps(observed, indent=2) + "\n")
+                result["laterUserWorkMaterializedItemRows"] = observed["materializedItemRows"] - initial["materializedItemRows"]
+                result["laterUserWorkIncludesHistoricalFullMutationReads"] = True
+            else:
+                initial = observed
+                assert initial["materializedItemRows"] >= inputs["count"] > inputs["initialRowBound"]
+            (evidence / "initial-materializations.json").write_text(json.dumps(initial, indent=2) + "\n")
+            result["initialMaterializedItemRows"] = initial["materializedItemRows"]
+            result["initialSelectCalls"] = len(initial["events"])
+        audited = helper("audit", "audit")
+        if args.mode == "implementation":
+            after = audit_edits(audited, evidence / "audit-analysis", inputs, seeded, created_id, result["deviceEditTimeBounds"])
+        else:
+            after = audit_seed(audited, evidence / "audit-analysis", inputs, count)
+            assert after["tables"] == seeded["tables"]
+        result["audit"] = {key: value for key, value in after.items() if key != "tables"}
+        result["status"] = "PASS_HELPER_PROBE" if args.mode == "probe" else "PASS" if args.mode == "implementation" else "LIMITATION_REPRODUCED"
+    except BaseException as error:
+        result.update(status="FAIL", error=repr(error))
+        try:
+            adb("failure-logcat", "logcat", "-d", "-v", "threadtime", check=False)
+            adb("failure-activities", "shell", "dumpsys", "activity", "activities", check=False)
+            for phase in ("seed", "audit"):
+                if not (evidence / phase).exists():
+                    adb("failure-" + phase + "-pull", "pull", "/sdcard/Android/data/" + PACKAGE + "/files/m15-" + phase,
+                        str(evidence / ("failure-" + phase)), check=False)
+        except BaseException as capture_error:
+            result["failureCaptureError"] = repr(capture_error)
+    finally:
+        try:
+            stopped("cleanup")
+            result["cleanupNetwork"] = network("restore", original_network) if original_network is not None else None
+            result["pidAfterCleanup"] = adb("cleanup-final-pid", "shell", "pidof", PACKAGE, check=False)
+            assert not result["pidAfterCleanup"]
+        except BaseException as cleanup_error:
+            result.update(status="FAIL", cleanupError=repr(cleanup_error))
+        result.update(elapsedSeconds=time.monotonic() - began, adbCommands=len(commands))
+        save()
+    print(json.dumps({key: result.get(key) for key in ("status", "mode", "elapsedSeconds", "firstUsableSeconds", "initialMaterializedItemRows", "error")}))
+    return 0 if result["status"] in ("PASS_HELPER_PROBE", "LIMITATION_REPRODUCED", "PASS") else 1
+
+
+if __name__ == "__main__":
+    raise SystemExit(main())
diff --git a/src/App.tsx b/src/App.tsx
index 99e74cd..656b986 100644
--- a/src/App.tsx
+++ b/src/App.tsx
@@ -1,7 +1,6 @@
 import React, {useEffect, useRef, useState} from 'react';
-import {Button, DeviceEventEmitter, Keyboard, Pressable, SafeAreaView, ScrollView, StyleSheet, Text, TextInput, View} from 'react-native';
-import {Item} from './items';
-import {ItemMutation, ItemStore, openItemStore, openRuntimeItemStore} from './itemStore';
+import {Button, DeviceEventEmitter, FlatList, Keyboard, Pressable, SafeAreaView, StyleSheet, Text, TextInput, View} from 'react-native';
+import {ITEM_PAGE_SIZE, ItemMutation, ItemPage, ItemStore, openItemStore, openRuntimeItemStore} from './itemStore';
 import {assertSyncActive, ForegroundSync, SyncSession} from './sync';
 import {backgroundBridge, BackgroundState, observeForegroundSync, ownsAutomaticSync, schedulePending, serializeSync} from './backgroundSync';
 
@@ -30,7 +29,8 @@ export default function App({openStore = openItemStore, createSync = defaultSync
   testRefreshClock?: boolean;
   editorMemory?: ReturnType<typeof createEditorMemory>;
 }) {
-  const [items, setItems] = useState<Item[]>([]);
+  const [page, setPage] = useState<ItemPage>({items: [], index: 0, total: 0});
+  const [listError, setListError] = useState<string | null>(null);
   const [ready, setReady] = useState(false);
   const [busy, setBusy] = useState(true);
   const [error, setError] = useState<string | null>(null);
@@ -47,6 +47,8 @@ export default function App({openStore = openItemStore, createSync = defaultSync
   const foreground = useRef<AbortController | null>(null);
   const busyRef = useRef(true);
   const mounted = useRef(false);
+  const pageIndex = useRef(0);
+  const pageRequest = useRef(0);
   const [{draft, editingId}, setEditor] = useState(() => editorMemory.current);
 
   useEffect(() => {
@@ -55,12 +57,11 @@ export default function App({openStore = openItemStore, createSync = defaultSync
       void serializeSync(async () => {
         const opened = store.current;
         if (!mounted.current || !opened) {return;}
-        const saved = await opened.read();
+        await reloadPage(opened);
         const pending = await opened.readPending();
         const conflicts = await opened.readConflicts();
         const state = await backgroundBridge.getState();
         if (!mounted.current || store.current !== opened) {return;}
-        setItems(saved);
         setPendingCount(pending.length);
         setIdentityBlocked(pending.some(operation => operation.terminalError === 'identity_conflict'));
         setConflictCount(conflicts.length);
@@ -79,6 +80,39 @@ export default function App({openStore = openItemStore, createSync = defaultSync
     setEditor(next);
   }
 
+  function publishPage(saved: ItemPage) {
+    pageIndex.current = saved.index;
+    setPage(saved);
+    setListError(null);
+  }
+
+  async function reloadPage(opened: ItemStore, index = pageIndex.current) {
+    const request = ++pageRequest.current;
+    const current = () => mounted.current && store.current === opened && request === pageRequest.current;
+    try {
+      const saved = await opened.readPage(index);
+      if (current()) {publishPage(saved);}
+    } catch {
+      // A display read cannot undo a completed edit or clear its draft as a
+      // failed save. Keep the last confirmed page and offer a separate retry.
+      if (current()) {setListError('Could not reload the list. Saved changes are retained.');}
+    }
+  }
+
+  async function changePage(index: number) {
+    const opened = store.current;
+    if (!mounted.current || !opened || busyRef.current) {return;}
+    busyRef.current = true;
+    setBusy(true);
+    try {
+      await serializeSync(async () => {
+        if (mounted.current && store.current === opened) {await reloadPage(opened, index);}
+      });
+    } finally {
+      if (mounted.current && store.current === opened) {busyRef.current = false; setBusy(false);}
+    }
+  }
+
   useEffect(() => {
     let active = true;
     let recoverPending = false;
@@ -90,7 +124,7 @@ export default function App({openStore = openItemStore, createSync = defaultSync
       ? openRuntimeItemStore(testMutationIdentity ? () => testMutationIdentity : undefined) : openStore();
     opening.then(opened => serializeSync(async () => {
       if (!active) {return;}
-      const saved = await opened.read();
+      const saved = await opened.readPage();
       const lastRefresh = await opened.readLastSuccessfulRefresh();
       const pending = await opened.readPending();
       const conflicts = await opened.readConflicts();
@@ -103,7 +137,7 @@ export default function App({openStore = openItemStore, createSync = defaultSync
       if (active) {
         store.current = opened;
         sync.current = createSync(opened, testIdentityPrefix, testRefreshClock);
-        setItems(saved);
+        publishPage(saved);
         setLastSuccessfulRefreshAt(lastRefresh);
         setPendingCount(pending.length);
         setIdentityBlocked(pending.some(operation => operation.terminalError === 'identity_conflict'));
@@ -156,12 +190,14 @@ export default function App({openStore = openItemStore, createSync = defaultSync
       // SQLite already commits each edit/intent atomically against ACKs. Do not
       // hold the runtime upload lock across a late local callback: a remounted
       // editor must still be able to read that committed state.
-      const committed = await origin.mutate(action, identityPrefix);
+      // The historical mutation API still returns a full snapshot for callers
+      // outside this screen. Never render or slice it into the paged list.
+      await origin.mutate(action, identityPrefix);
       return await serializeSync(async () => {
-        let saved = committed;
-        try {if (mounted.current && store.current === origin) {saved = await origin.read();}}
-        catch { /* A failed status read cannot undo the confirmed local COMMIT. */ }
-        if (mounted.current && store.current === origin) {setItems(saved); setRefresh({status: 'stale'});}
+        if (mounted.current && store.current === origin) {
+          await reloadPage(origin);
+          if (mounted.current && store.current === origin) {setRefresh({status: 'stale'});}
+        }
         try {
           // The committed edit must register work even if its editor unmounted.
           // Native scheduling retains an existing cycle and its durable count.
@@ -213,10 +249,9 @@ export default function App({openStore = openItemStore, createSync = defaultSync
       });
       if (!performed) {if (current()) {setRefresh({status: 'stale'});} return;}
       if (!current()) {return;}
-      const saved = await origin.read();
+      await reloadPage(origin);
       const lastRefresh = await origin.readLastSuccessfulRefresh();
       if (current()) {
-        setItems(saved);
         setLastSuccessfulRefreshAt(lastRefresh);
         setRefresh({status: 'fresh'});
       }
@@ -224,10 +259,7 @@ export default function App({openStore = openItemStore, createSync = defaultSync
       // A conflict can commit its canonical winner before a later GET fails.
       // Show that committed state, while retaining the refresh error/time.
       if (current()) {
-        try {
-          const saved = await origin.read();
-          if (current()) {setItems(saved);}
-        } catch { /* Keep the last confirmed list. */ }
+        await reloadPage(origin);
         if (current()) {setRefresh({status: 'error', message: `Could not refresh: ${reason instanceof Error ? reason.message : String(reason)}`});}
       }
     } finally {
@@ -308,11 +340,34 @@ export default function App({openStore = openItemStore, createSync = defaultSync
       />
       <Button title={editingId === null ? 'Add item' : 'Save title'} accessibilityLabel={editingId === null ? 'Add item' : 'Save title'} onPress={saveTitle} disabled={!ready || busy || !draft.trim()} />
       {editingId !== null && <Button title="Cancel edit" disabled={busy} onPress={() => updateEditor({editingId: null, draft: ''})} />}
-      {ready && <Text accessibilityLabel={`Item count: ${items.length}`} style={styles.count}>
-        {items.length} {items.length === 1 ? 'item' : 'items'}
+      {ready && <Text accessibilityLabel={`Item count: ${page.total}`} style={styles.count}>
+        {page.total} {page.total === 1 ? 'item' : 'items'}
       </Text>}
-      <ScrollView keyboardShouldPersistTaps="handled">
-        {items.map(item => (
+      {listError !== null && <>
+        <Text accessibilityRole="alert">{listError}</Text>
+        <Button title="Reload list" disabled={busy} onPress={() => void changePage(page.index)} />
+      </>}
+      {ready && page.total > ITEM_PAGE_SIZE && <View>
+        <Text accessibilityLabel={`Page ${page.index + 1} of ${Math.ceil(page.total / ITEM_PAGE_SIZE)}`}>
+          Page {page.index + 1} of {Math.ceil(page.total / ITEM_PAGE_SIZE)}
+        </Text>
+        <View style={styles.actions}>
+          <Button title="First" accessibilityLabel="First page" disabled={busy || page.index === 0} onPress={() => void changePage(0)} />
+          <Button title="Previous" accessibilityLabel="Previous page" disabled={busy || page.index === 0} onPress={() => void changePage(page.index - 1)} />
+          <Button title="Next" accessibilityLabel="Next page" disabled={busy || (page.index + 1) * ITEM_PAGE_SIZE >= page.total} onPress={() => void changePage(page.index + 1)} />
+          <Button title="Last" accessibilityLabel="Last page" disabled={busy || (page.index + 1) * ITEM_PAGE_SIZE >= page.total} onPress={() => void changePage(Math.ceil(page.total / ITEM_PAGE_SIZE) - 1)} />
+        </View>
+      </View>}
+      <FlatList
+        key={page.index}
+        style={styles.list}
+        data={page.items}
+        keyExtractor={item => item.id}
+        initialNumToRender={8}
+        maxToRenderPerBatch={8}
+        windowSize={5}
+        keyboardShouldPersistTaps="handled"
+        renderItem={({item}) => (
           <View key={item.id} testID={`item-row-${item.id}`} style={styles.row}>
             <Text testID={`item-title-${item.id}`} style={styles.title}>{item.title}</Text>
             <Pressable
@@ -332,8 +387,8 @@ export default function App({openStore = openItemStore, createSync = defaultSync
               }} />
             </View>
           </View>
-        ))}
-      </ScrollView>
+        )}
+      />
     </SafeAreaView>
   );
 }
@@ -343,6 +398,7 @@ const styles = StyleSheet.create({
   heading: {fontSize: 22, fontWeight: 'bold', color: '#111111'},
   input: {borderWidth: 1, borderColor: '#666666', padding: 10, marginVertical: 12, color: '#111111'},
   count: {marginVertical: 12},
+  list: {flex: 1},
   row: {borderBottomWidth: 1, borderBottomColor: '#dddddd', paddingVertical: 12},
   title: {fontSize: 18, color: '#111111'},
   toggle: {paddingVertical: 12},
diff --git a/src/itemStore.ts b/src/itemStore.ts
index 7d60116..2266c22 100644
--- a/src/itemStore.ts
+++ b/src/itemStore.ts
@@ -4,6 +4,9 @@ import {mutationHash, mutationTarget, newMutationIdentity} from './mutationProto
 
 export const DATABASE_NAME = 'items.db';
 export const SCHEMA_VERSION = 5;
+export const ITEM_PAGE_SIZE = 50;
+
+export type ItemPage = {items: Item[]; index: number; total: number};
 
 export type ItemRow = {
   id: string;
@@ -61,6 +64,7 @@ type RemoteVersion = {id: string; version: number; updatedAt: number | null; del
 
 export interface ItemStore {
   read(): Promise<Item[]>;
+  readPage(index?: number): Promise<ItemPage>;
   readLastSuccessfulRefresh(): Promise<number | null>;
   readPending(): Promise<PendingMutation[]>;
   readConflicts(): Promise<MutationConflict[]>;
@@ -179,6 +183,27 @@ class SqliteItemStore implements ItemStore {
     });
   }
 
+  readPage(index = 0): Promise<ItemPage> {
+    if (!Number.isSafeInteger(index)) {return Promise.reject(new Error('Invalid Item page'));}
+    return new Promise((resolve, reject) => {
+      let page: ItemPage = {items: [], index: 0, total: 0};
+      // SELECT-only transaction(): unlike this library's readTransaction(), it
+      // starts a native transaction, so COUNT and the clamped page agree.
+      this.database.transaction(tx => {
+        tx.executeSql('SELECT COUNT(*) AS count FROM items', [], (_, count) => {
+          const total = count.rows.item(0).count;
+          if (!Number.isSafeInteger(total) || total < 0) {throw new Error('Invalid Item count');}
+          const last = Math.max(0, Math.ceil(total / ITEM_PAGE_SIZE) - 1);
+          page = {items: [], index: Math.max(0, Math.min(index, last)), total};
+          tx.executeSql('SELECT id, title, completed, version, updated_at FROM items ORDER BY rowid LIMIT ? OFFSET ?',
+            [ITEM_PAGE_SIZE, page.index * ITEM_PAGE_SIZE], (_, result) => {
+              for (let i = 0; i < result.rows.length; i++) {page.items.push(rowToItem(result.rows.item(i)));}
+            });
+        });
+      }, reject, () => resolve(page));
+    });
+  }
+
   readLastSuccessfulRefresh(): Promise<number | null> {
     return new Promise((resolve, reject) => {
       let last: number | null = null;
diff --git a/verification/phase-1/M15-inputs.json b/verification/phase-1/M15-inputs.json
new file mode 100644
index 0000000..262cc44
--- /dev/null
+++ b/verification/phase-1/M15-inputs.json
@@ -0,0 +1,43 @@
+{
+  "profile": "phase-1",
+  "thread": "M15",
+  "count": 2000,
+  "probeCount": 2,
+  "schemaVersion": 5,
+  "databaseLocation": "files/items.db",
+  "itemIdPrefix": "large-",
+  "titlePrefix": "Item ",
+  "decimalWidth": 4,
+  "completed": false,
+  "version": 1,
+  "updatedAt": 1700000000000,
+  "initialNextLocalId": 1,
+  "initialPending": 0,
+  "initialConflicts": 0,
+  "initialRowBound": 50,
+  "coldStartUiTimeoutSeconds": 60,
+  "networkTimeoutSeconds": 15,
+  "auditOutsideMeasuredWindow": true,
+  "finalOperations": {
+    "renameId": "large-2000",
+    "renameTitle": "Item 2000 edited",
+    "complete": true,
+    "createTitle": "Large local",
+    "deleteCreated": true,
+    "expectedItems": 2000,
+    "expectedPending": 4
+  },
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
diff --git a/verification/phase-1/M15.md b/verification/phase-1/M15.md
new file mode 100644
index 0000000..65cdab6
--- /dev/null
+++ b/verification/phase-1/M15.md
@@ -0,0 +1,84 @@
+# M15 — bounded release startup and normal large-list edits
+
+Status: **PASS, independently accepted by root**; final Git audit/tag remain root-owned.
+Profile: `phase-1`; spec: `61280dd86ce88b6e431f408241c0998a275960aa`.
+START: `f3464eb7b8859380c613c866bd0fa58eca0a69dd` (`progress/phase-1/react-native/M14`).
+Attempt 2 / repair 1 (patch formatting only); fixed2000 budget **2/3 used**.
+No owner device invocation, warmup, duplicate full case, phase-2 work or push.
+
+## Frozen lineage and reproduction
+
+The [final manifest](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M15/implementation-candidate-manifest.json)
+(`b7dc26d56d2d7a78eba857368d2324d96c667ce8f06a86ce8b24d3736d0af2b3`)
+pins 77 source/build copies,128 external records, original inputs, wrapper/environment
+and the exact `implementationArgv`. Root alone executed that command, once.
+[Artifact provenance](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M15/implementation-artifact-provenance.json)
+binds the generated Hermes bundle/maps, exact product source content, resolved
+RN/Hermes 0.76.9 release AARs and native compiled observation/helper files.
+
+Final app SHA256: `bbaa5ce16f73f1c2a1568e061989cb132909963e4b401bf61547746dd4d4ec1e`.
+Helper SHA256: `650c2632b5ba2034ae0aa30d427aec69d98aa788e787d30464256b490c167ee0`.
+The helper is byte-identical to the accepted baseline helper. Only the app ZIP's
+Hermes bundle changed from the baseline app; root independently checked the
+installed signatures, nondebuggable flags, native artifacts and production JS.
+
+## Actual Android evidence
+
+| Root-owned run | Ordinary PID | Initial native Item rows | Observed first usable UI |
+| --- | --- | --- | --- |
+| Unchanged query/render release baseline | 24028 | 2000 | 16.640510791s |
+| Final bounded release | 25449 | 50 | 3.674518833s |
+
+These are single launch-to-fresh-UI observations, including controller latency,
+not repeated timing benchmarks or an added latency SLA. The native counter runs
+after materializing each row and before bridge/JS processing; every read preceding
+the first deliberate positive-offset page query is included in the initial window.
+
+The [baseline acceptance](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M15/main-baseline-acceptance.json)
+reopens two unchanged native databases and confirms the original unbounded query.
+The earlier [two-row helper probe](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M15/main-helper-probe-acceptance.json)
+passed real seed/audit instrumentation without an Activity launch; it was not rerun.
+
+The [final root acceptance](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M15/main-implementation-acceptance.json)
+SHA256 `ee59a75c0022ec99cb53ec454154e35d548156f3f5c1eefa29d42820a6f26c7f`
+binds 385 raw files,182 adb commands,19 fresh UI polls and all 53 native SELECTs.
+Wrapper exit 0 /65.213s. One ordinary PID remained through navigation, rename
+`large-2000` to `Item 2000 edited`, completion, creation of `Large local` with its
+real identity, and deletion of that created Item. Root reopened native DB/WAL:
+2000 final Items,1999 untouched, edited Item completed/version 3, all 2000 prior
+canonical rows unchanged, and four original durable nonce/hash intents, all still
+undispatched and unacknowledged. Actual timestamps and all four envelopes are in
+the acceptance record; the deleted create's timestamp is not fabricated.
+
+Root verified [installed artifacts](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M15/main-implementation-installed/result.json),
+[visual controls](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M15/main-visual-review.json)
+and [cleanup](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M15/main-final-cleanup-01/result.json).
+The app was absent and network 0/1/1 restored; no fixture server was used.
+
+## Host/build checks and preserved limits
+
+`implementation-host-02`: **126/126, three suites PASS**; `implementation-typecheck-02`
+and small controller checks02 PASS. New small fixtures cover native page order,
+clamping/transaction failure, draft/navigation ownership, four queued mutations,
+postcommit reload failure and callback ordering. The corrected Decimal controller
+check preserves exact integral native bindings, limit 50 and the navigation offset.
+`implementation-release-build-01` passed in 21.2957s with explicit
+`-PmseTestBuildType=release`, `assembleRelease` and `assembleReleaseAndroidTest`,
+offline, with release lint enabled. Exact argv, outputs and source inventories are
+linked from the frozen manifest; no tests/builds were repeated after root acceptance.
+
+Preserved failures include the original support-checker patch argument, release
+build01 lint error (explicit `includeSubdomains="false"` retained the existing
+Android default), manifest-parser preparation assumption, repair1 patch-context
+whitespace, and host01's 125/126 result (only a new Android button-label selector
+needed case-insensitive matching). Their raw records and diagnoses remain in the
+manifest's `preservedFailures` and [preparation notes](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M15/implementation-preparation-notes.json).
+Patch repair proved identical applied native bytes and reused the baseline APKs.
+
+Known retained cost: deliberate navigation/edits materialized 16455 Item rows after
+the initial window, including historical full mutation reads. Those arrays are
+not rendered or sliced into pages; M15 does not claim all-operation optimization.
+Prior Android M01–M10/M14 evidence is retained, **not rerun on this final release**.
+The current host suite and this real release navigation/four-CRUD run are the new
+regressions; schema/sync/identity/conflict/cancellation/background contracts remain
+unchanged. No unrelated business feature or phase-2 scope was added.
