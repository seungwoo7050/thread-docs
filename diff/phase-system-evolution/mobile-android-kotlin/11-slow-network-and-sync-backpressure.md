# M14 — 느린 Network와 Sync Backpressure

## `test: verify M14 pressure cancellation on unchanged product`

diff --git a/TRACK.md b/TRACK.md
index 97234bc..abb8c7d 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -280,6 +280,28 @@ OS commands. Attempt5/cumulative repair4, actual A6/B1 usage, follows the explic
 necessary-repair authorization; the product HTTP limit remains3. Main verified frozen bytes
 and actual cleanup. Only reporting changed after runtime; final history/tag audit remains main-owned.
 
+## M14 boundary — unchanged M10 regression passed
+
+The verified M10 product already satisfies the fixed pressure/cancellation case. M14 adds
+only test support: independent500ms fixture responses, actual UI/Room/WorkManager observations,
+five real production sync calls and an exact durable-ACK cancellation barrier. No product,
+schema, dependency, retry ceiling or scheduler configuration changed.
+
+Root's single Android execution passed: twelve offline UI creates, five calls during the
+same active drain, peak1 mutation, then exactly3 local ACKs. The observer closed its cursor,
+requested cancellation, and root turned the actual radios off before awaiting settlement.
+Settlement was observed after341ms with9 original pending envelopes. One ordinary Sync
+press on the same Activity/store after reconnect reached12 canonical version1 Items,
+applied12, duplicate0 and pending0. Native snapshots preserve the initial visible pressure
+and the paused durable queue without a forced UI reset. Greedy/SystemJobService deduplication
+occurred in PID14200; no new process-death guarantee is claimed by M14.
+
+Root reopened18 native databases and bound643 raw files/266 adb commands. New focused host1,
+fixture1 and helper2 checks passed; prior M10 host28, fixture14 and native24 evidence is
+retained, not rerun. Initial sandbox denials and root observation-parser diagnostics remain
+recorded in `verification/M14.md`. Attempt1, repair0, fixed-case usage1/3. Root confirmed the
+unchanged app/tested source bytes and actual cleanup; final commit/tag audit remains root-owned.
+
 ## Pinned build
 
 - JDK 17, Gradle 8.7 (wrapper distribution SHA-256 pinned)
diff --git a/app/src/androidTest/assets/M14-inputs.json b/app/src/androidTest/assets/M14-inputs.json
new file mode 100644
index 0000000..2927636
--- /dev/null
+++ b/app/src/androidTest/assets/M14-inputs.json
@@ -0,0 +1,133 @@
+{
+  "thread": "M14",
+  "profile": "phase-1",
+  "specRevision": "61280dd86ce88b6e431f408241c0998a275960aa",
+  "start": "bde06a693e4b96707e9c800152687c2b025df54a",
+  "items": [
+    {
+      "id": "pressure-001",
+      "title": "Pressure 001",
+      "completed": false,
+      "clientMutationId": "m14-create-001",
+      "payloadHash": "ae07b5d93d038ef1068755451d9cf6ed6fd415234fc834bb01e4b1698e2a713f",
+      "localUpdatedAt": 1700001000000
+    },
+    {
+      "id": "pressure-002",
+      "title": "Pressure 002",
+      "completed": false,
+      "clientMutationId": "m14-create-002",
+      "payloadHash": "29e613dfa77b8e6910680f063fa0c904cb972a580c83d6fce4ab353bcf19e9b5",
+      "localUpdatedAt": 1700001001000
+    },
+    {
+      "id": "pressure-003",
+      "title": "Pressure 003",
+      "completed": false,
+      "clientMutationId": "m14-create-003",
+      "payloadHash": "26b52518fe6f7145c3702c0eeb1c254888cc33cb0640bfd899ccd658d22cbdac",
+      "localUpdatedAt": 1700001002000
+    },
+    {
+      "id": "pressure-004",
+      "title": "Pressure 004",
+      "completed": false,
+      "clientMutationId": "m14-create-004",
+      "payloadHash": "6ae7172640b9c23da3c0ff578611b1243a1f6b9ad688bf45104be7db5340def5",
+      "localUpdatedAt": 1700001003000
+    },
+    {
+      "id": "pressure-005",
+      "title": "Pressure 005",
+      "completed": false,
+      "clientMutationId": "m14-create-005",
+      "payloadHash": "98c1b632b8da7811e6df23fc13e2bd685760d00bb84cbe74e067339952825285",
+      "localUpdatedAt": 1700001004000
+    },
+    {
+      "id": "pressure-006",
+      "title": "Pressure 006",
+      "completed": false,
+      "clientMutationId": "m14-create-006",
+      "payloadHash": "7b9ec807cd9363b8935959287284a4f18ae08dbeac85dc775351ec9d39793a25",
+      "localUpdatedAt": 1700001005000
+    },
+    {
+      "id": "pressure-007",
+      "title": "Pressure 007",
+      "completed": false,
+      "clientMutationId": "m14-create-007",
+      "payloadHash": "3013081e7f0959ad4c2b60f1aeab7df2110a594176f91d9f5eb0869e22b439d4",
+      "localUpdatedAt": 1700001006000
+    },
+    {
+      "id": "pressure-008",
+      "title": "Pressure 008",
+      "completed": false,
+      "clientMutationId": "m14-create-008",
+      "payloadHash": "675bf54598ee25e8f61be10926a97d607fa94752b8762bfab7a9759608bf7f9f",
+      "localUpdatedAt": 1700001007000
+    },
+    {
+      "id": "pressure-009",
+      "title": "Pressure 009",
+      "completed": false,
+      "clientMutationId": "m14-create-009",
+      "payloadHash": "d452b2dd73c5040e79d46ca57c1c869c5dd60204619931dba0d33bf4a40c13d0",
+      "localUpdatedAt": 1700001008000
+    },
+    {
+      "id": "pressure-010",
+      "title": "Pressure 010",
+      "completed": false,
+      "clientMutationId": "m14-create-010",
+      "payloadHash": "6be83218520a3faccb5e28ecb7503aec56bdfbc4dacc631686dc22665dba1861",
+      "localUpdatedAt": 1700001009000
+    },
+    {
+      "id": "pressure-011",
+      "title": "Pressure 011",
+      "completed": false,
+      "clientMutationId": "m14-create-011",
+      "payloadHash": "861951569c5960b3d77c6986561778b6398deb36d6afa5e8e138a153465bbd6c",
+      "localUpdatedAt": 1700001010000
+    },
+    {
+      "id": "pressure-012",
+      "title": "Pressure 012",
+      "completed": false,
+      "clientMutationId": "m14-create-012",
+      "payloadHash": "f85243c5292cc488ec16889c3fe25a4c58fcc51564e47ec21e0fedf120245948",
+      "localUpdatedAt": 1700001011000
+    }
+  ],
+  "responseDelayMs": 500,
+  "firstTimestamp": 1700001000000,
+  "nextTimestamp": 1700001012000,
+  "additionalProductionRequests": 5,
+  "ackBarrier": 3,
+  "pendingAtSettlement": 9,
+  "maximumConcurrentMutations": 2,
+  "reconnectForegroundRequests": 1,
+  "schemaVersion": 6,
+  "workDatabaseVersion": 20,
+  "workManagerVersion": "2.9.1",
+  "uniqueWorkName": "item-automatic-sync",
+  "workClass": "com.mobilesystemsevolution.kotlin.ItemSyncWorker",
+  "processDeath": false,
+  "schedulerDisabled": false,
+  "debugClockFile": false,
+  "harness": {
+    "uiWaitMs": 5000,
+    "networkWaitMs": 30000,
+    "initialDrainWaitMs": 30000,
+    "barrierWaitMs": 30000,
+    "cancellationSettlementMs": 12000,
+    "controlGateWaitMs": 30000,
+    "finalDrainWaitMs": 30000,
+    "instrumentationTimeoutSeconds": 180,
+    "adbTimeoutSeconds": 45,
+    "preSeedTeardownWaitSeconds": 30
+  },
+  "triggerAccounting": "Initial automatic dispatch is observed; exactly five additional direct production ItemSync.synchronize calls occur during its active request. No initial Sync button. One ordinary Sync button after reconnect."
+}
diff --git a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/M14PressureTest.kt b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/M14PressureTest.kt
new file mode 100644
index 0000000..4267825
--- /dev/null
+++ b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/M14PressureTest.kt
@@ -0,0 +1,521 @@
+package com.mobilesystemsevolution.kotlin
+
+import android.app.Activity
+import android.app.Application
+import android.database.Cursor
+import android.database.sqlite.SQLiteDatabase
+import android.graphics.Bitmap
+import android.graphics.Rect
+import android.net.ConnectivityManager
+import android.net.NetworkCapabilities
+import android.os.Bundle
+import android.os.Process
+import android.os.SystemClock
+import android.util.Log
+import android.util.Xml
+import android.view.accessibility.AccessibilityNodeInfo
+import androidx.activity.compose.setContent
+import androidx.compose.material3.MaterialTheme
+import androidx.compose.ui.semantics.SemanticsProperties
+import androidx.compose.ui.semantics.getOrNull
+import androidx.compose.ui.test.assertIsDisplayed
+import androidx.compose.ui.test.assertIsEnabled
+import androidx.compose.ui.test.assertIsOff
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
+import androidx.room.InvalidationTracker
+import androidx.test.ext.junit.runners.AndroidJUnit4
+import androidx.test.platform.app.InstrumentationRegistry
+import androidx.work.Logger
+import androidx.work.Operation
+import androidx.work.WorkManager
+import java.io.File
+import java.net.HttpURLConnection
+import java.net.URL
+import java.util.concurrent.CopyOnWriteArrayList
+import java.util.concurrent.CountDownLatch
+import java.util.concurrent.TimeUnit
+import java.util.concurrent.atomic.AtomicBoolean
+import java.util.concurrent.atomic.AtomicReference
+import kotlinx.coroutines.CancellationException
+import kotlinx.coroutines.CoroutineScope
+import kotlinx.coroutines.CoroutineStart
+import kotlinx.coroutines.Dispatchers
+import kotlinx.coroutines.Job
+import kotlinx.coroutines.SupervisorJob
+import kotlinx.coroutines.cancel
+import kotlinx.coroutines.joinAll
+import kotlinx.coroutines.launch
+import kotlinx.coroutines.runBlocking
+import kotlinx.coroutines.withTimeout
+import org.json.JSONArray
+import org.json.JSONObject
+import org.junit.Assert.assertEquals
+import org.junit.Assert.assertSame
+import org.junit.Assert.assertTrue
+import org.junit.Rule
+import org.junit.Test
+import org.junit.rules.ExternalResource
+import org.junit.rules.RuleChain
+import org.junit.runner.RunWith
+import org.xmlpull.v1.XmlSerializer
+
+/** Fixed IDs at initial construction only; every edit and drain uses production code. */
+@RunWith(AndroidJUnit4::class)
+class M14PressureTest {
+    private val instrumentation = InstrumentationRegistry.getInstrumentation()
+    private val context = instrumentation.targetContext
+    private val inputs = JSONObject(instrumentation.context.assets.open("M14-inputs.json").bufferedReader().use { it.readText() })
+    private val limits = inputs.getJSONObject("harness")
+    private val fixed = inputs.getJSONArray("items")
+    private val compose = createAndroidComposeRule<MainActivity>()
+    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.Default)
+    private val jobs = CopyOnWriteArrayList<Job>()
+    private val events = mutableListOf<JSONObject>()
+    private val checkpoints = JSONArray()
+    private val guard = Any()
+    private val barrierArmed = AtomicBoolean(false)
+    private val barrierHandled = AtomicBoolean(false)
+    private val barrierReady = CountDownLatch(1)
+    private val callbackFailure = AtomicReference<Throwable>()
+    private val cancelOperation = AtomicReference<Operation>()
+    private var cancelAtNs = 0L
+    private lateinit var output: File
+    private lateinit var database: ItemDatabase
+    private lateinit var store: ItemStore
+    private lateinit var sync: ItemSync
+    private lateinit var application: ItemApplication
+    private lateinit var manager: WorkManager
+    private lateinit var previousLogger: Logger
+    private val report = JSONObject().put("scenario", "M14").put("status", "RUNNING")
+        .put("actualUiCreates", 0).put("additionalProductionRequests", 0).put("foregroundSyncPresses", 0)
+        .put("processDeath", false).put("schedulerDisabled", false).put("debugClockFile", false)
+
+    private fun event(name: String, details: JSONObject = JSONObject()): JSONObject {
+        val value = details.put("event", name).put("pid", Process.myPid())
+            .put("thread", Thread.currentThread().name).put("elapsedRealtimeNs", SystemClock.elapsedRealtimeNanos())
+            .put("wallTimeMs", System.currentTimeMillis())
+        synchronized(guard) { events.add(value) }
+        Log.i("M14Pressure", value.toString())
+        return value
+    }
+
+    private fun lifecycle(name: String, activity: Activity) {
+        if (activity is MainActivity) event(name, JSONObject().put("activity", System.identityHashCode(activity))
+            .put("application", System.identityHashCode(activity.application)))
+    }
+
+    private val callbacks = object : Application.ActivityLifecycleCallbacks {
+        override fun onActivityCreated(activity: Activity, state: Bundle?) = lifecycle("created", activity)
+        override fun onActivityStarted(activity: Activity) = lifecycle("started", activity)
+        override fun onActivityResumed(activity: Activity) = lifecycle("resumed", activity)
+        override fun onActivityPaused(activity: Activity) = lifecycle("paused", activity)
+        override fun onActivityStopped(activity: Activity) = lifecycle("stopped", activity)
+        override fun onActivitySaveInstanceState(activity: Activity, state: Bundle) = lifecycle("saved", activity)
+        override fun onActivityDestroyed(activity: Activity) = lifecycle("destroyed", activity)
+    }
+
+    @get:Rule
+    val rules: RuleChain = RuleChain.outerRule(object : ExternalResource() {
+        override fun before() {
+            output = File(context.filesDir, "m14-pressure-01")
+            check(!output.exists() && output.mkdirs()) { "Refusing to overwrite M14 evidence" }
+            check(!File(context.filesDir, "m10-clock").exists()) { "M10 clock/scheduler override must be absent" }
+            database = ItemDatabase.open(context)
+            application = context.applicationContext as ItemApplication
+            manager = WorkManager.getInstance(context)
+            assertTrue(manager.getWorkInfosForUniqueWork(ItemWorkScheduler.UNIQUE_WORK).get().isEmpty())
+            previousLogger = Logger.get()
+            Logger.setLogger(Logger.LogcatLogger(Log.DEBUG)) // Observation only; no scheduler/clock override.
+            var idIndex = 0
+            var mutationIndex = 0
+            var timeIndex = 0
+            store = ItemStore(database.items(), nextId = { fixed.getJSONObject(idIndex++).getString("id") },
+                now = { fixed.getJSONObject(timeIndex++).getLong("localUpdatedAt") },
+                nextMutationId = { fixed.getJSONObject(mutationIndex++).getString("clientMutationId") },
+                afterCommit = application::scheduleCommitted)
+            runBlocking {
+                assertTrue(store.items().isEmpty())
+                assertTrue(store.pendingMutations().isEmpty())
+                assertTrue(store.acknowledgedMutations().isEmpty())
+            }
+            val remote = HttpItemRemote()
+            val observed = object : ItemRemote {
+                override suspend fun items() = snapshot().items
+                override suspend fun snapshot(): CanonicalSnapshot {
+                    event("foreground-get-enter")
+                    return try { remote.snapshot() } finally { event("foreground-get-settled") }
+                }
+                override suspend fun send(mutation: PendingMutation): MutationResult {
+                    event("foreground-send-enter", JSONObject().put("identity", mutation.clientMutationId)
+                        .put("payloadHash", mutation.payloadHash).put("sequence", mutation.sequence))
+                    var outcome = "error"
+                    try {
+                        val result = remote.send(mutation)
+                        outcome = "returned-${result.statusCode}"
+                        return result
+                    } catch (cancelled: CancellationException) {
+                        outcome = "cancelled"
+                        throw cancelled
+                    } finally {
+                        event("foreground-send-settled", JSONObject().put("identity", mutation.clientMutationId)
+                            .put("outcome", outcome))
+                    }
+                }
+            }
+            sync = ItemSync(store, observed, schedulePending = { application.scheduleCommitted(store) })
+            instrumentation.runOnMainSync { application.registerActivityLifecycleCallbacks(callbacks) }
+            writeReport()
+        }
+
+        override fun after() {
+            barrierArmed.set(false)
+            if (::database.isInitialized) database.invalidationTracker.removeObserver(ackObserver)
+            scope.cancel()
+            if (::application.isInitialized) instrumentation.runOnMainSync {
+                application.unregisterActivityLifecycleCallbacks(callbacks)
+            }
+            if (::previousLogger.isInitialized) Logger.setLogger(previousLogger)
+            if (::database.isInitialized) database.close()
+            if (::output.isInitialized) {
+                synchronized(guard) { report.put("testObserverRemoved", true) }
+                writeReport()
+            }
+        }
+    }).around(compose)
+
+    private fun atomicJson(file: File, value: JSONObject) {
+        val temporary = File(file.parentFile, file.name + ".tmp")
+        temporary.writeText(value.toString(2) + "\n")
+        check(temporary.renameTo(file)) { "Could not publish ${file.name}" }
+    }
+
+    private fun writeReport() = synchronized(guard) {
+        report.put("events", JSONArray(events.toList())).put("checkpoints", checkpoints)
+        atomicJson(File(output, "result.json"), report)
+    }
+
+    private fun phase(name: String, details: JSONObject = JSONObject()) {
+        val value = event("phase-$name", details)
+        writeReport()
+        synchronized(guard) { atomicJson(File(output, "phase.json"), JSONObject(value.toString()).put("phase", name)) }
+    }
+
+    private fun gate(name: String) {
+        val path = File(output, "$name.gate")
+        val deadline = SystemClock.elapsedRealtime() + limits.getLong("controlGateWaitMs")
+        while (!path.exists()) {
+            check(SystemClock.elapsedRealtime() < deadline) { "Missing root gate: $name" }
+            Thread.sleep(10)
+        }
+        check(path.readText() == "$name\n") { "Malformed root gate: $name" }
+        event("root-gate-$name")
+    }
+
+    private fun awaitNetwork(online: Boolean) {
+        val connectivity = context.getSystemService(ConnectivityManager::class.java)
+        val deadline = SystemClock.elapsedRealtime() + limits.getLong("networkWaitMs")
+        while (true) {
+            val network = connectivity.activeNetwork
+            val capabilities = network?.let(connectivity::getNetworkCapabilities)
+            val ready = capabilities?.hasCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET) == true &&
+                capabilities.hasCapability(NetworkCapabilities.NET_CAPABILITY_VALIDATED)
+            if (if (online) ready else network == null) {
+                event("native-network", JSONObject().put("online", online).put("network", network?.toString() ?: JSONObject.NULL)
+                    .put("capabilities", capabilities?.toString() ?: JSONObject.NULL))
+                return
+            }
+            check(SystemClock.elapsedRealtime() < deadline) { "Native network not online=$online: $network $capabilities" }
+            Thread.sleep(10)
+        }
+    }
+
+    private fun fixture(): JSONObject {
+        val connection = URL("http://10.0.2.2:18080/__m14").openConnection() as HttpURLConnection
+        connection.connectTimeout = limits.getInt("networkWaitMs")
+        connection.readTimeout = limits.getInt("networkWaitMs")
+        return try {
+            check(connection.responseCode == 200)
+            connection.inputStream.bufferedReader().use { JSONObject(it.readText()) }
+        } finally { connection.disconnect() }
+    }
+
+    private fun rows(cursor: Cursor): JSONArray = cursor.use {
+        val result = JSONArray()
+        while (it.moveToNext()) {
+            val row = JSONObject()
+            it.columnNames.forEachIndexed { index, column ->
+                row.put(column, when (it.getType(index)) {
+                    Cursor.FIELD_TYPE_NULL -> JSONObject.NULL
+                    Cursor.FIELD_TYPE_INTEGER -> it.getLong(index)
+                    Cursor.FIELD_TYPE_FLOAT -> it.getDouble(index)
+                    Cursor.FIELD_TYPE_BLOB -> it.getBlob(index).joinToString("") { byte -> "%02x".format(byte) }
+                    else -> it.getString(index)
+                })
+            }
+            result.put(row)
+        }
+        result
+    }
+
+    private fun table(name: String, order: String) = rows(database.openHelper.readableDatabase.query(
+        "SELECT * FROM $name ORDER BY $order"))
+
+    private fun copyDatabase(source: File, folder: File) {
+        for (suffix in listOf("", "-wal")) {
+            val file = File(source.path + suffix)
+            if (file.exists()) file.copyTo(File(folder, file.name))
+        }
+    }
+
+    private fun xmlNode(xml: XmlSerializer, node: AccessibilityNodeInfo) {
+        val bounds = Rect().also(node::getBoundsInScreen)
+        xml.startTag(null, "node")
+        for ((key, value) in mapOf("text" to node.text, "content-desc" to node.contentDescription,
+            "class" to node.className, "package" to node.packageName, "resource-id" to node.viewIdResourceName,
+            "enabled" to node.isEnabled, "focused" to node.isFocused, "checked" to node.isChecked,
+            "bounds" to bounds.toShortString())) xml.attribute(null, key, value?.toString() ?: "")
+        for (index in 0 until node.childCount) node.getChild(index)?.let { child ->
+            try { xmlNode(xml, child) } finally { child.recycle() }
+        }
+        xml.endTag(null, "node")
+    }
+
+    private fun capture(stage: String, withWork: Boolean = false) {
+        val folder = File(output, stage).apply { check(mkdir()) }
+        val activity = compose.activity
+        val value = event("checkpoint", JSONObject().put("stage", stage)
+            .put("application", System.identityHashCode(activity.application))
+            .put("activity", System.identityHashCode(activity)).put("store", System.identityHashCode(store))
+            .put("items", table("items", "id")).put("pending", table("pending_mutations", "sequence"))
+            .put("acknowledged", table("acknowledged_mutations", "clientMutationId"))
+            .put("automatic", table("automatic_sync", "id")).put("tombstones", table("tombstones", "id"))
+            .put("metadata", table("sync_metadata", "id")).put("allocator", table("sqlite_sequence", "name")))
+        copyDatabase(context.getDatabasePath(ItemDatabase.NAME), folder)
+        if (withWork) {
+            val file = File(context.noBackupFilesDir, "androidx.work.workdb")
+            check(file.isFile)
+            SQLiteDatabase.openDatabase(file.path, null, SQLiteDatabase.OPEN_READONLY).use { work ->
+                val observed = JSONObject().put("version", work.version)
+                for ((name, order) in mapOf("WorkSpec" to "id", "WorkName" to "name,work_spec_id",
+                    "SystemIdInfo" to "work_spec_id,generation", "Dependency" to "work_spec_id,prerequisite_id")) {
+                    observed.put(name, rows(work.rawQuery("SELECT * FROM $name ORDER BY $order", null)))
+                }
+                value.put("workDatabase", observed)
+                copyDatabase(file, folder)
+            }
+        }
+        File(folder, "semantics.txt").writeText(compose.onRoot().printToString())
+        val pending = compose.onNodeWithTag("pending-count").fetchSemanticsNode().config[SemanticsProperties.Text]
+        value.put("renderedPendingLabel", pending.joinToString { it.text })
+        instrumentation.uiAutomation.rootInActiveWindow.let { root ->
+            checkNotNull(root) { "No actual accessibility root" }
+            File(folder, "ui.xml").outputStream().use { stream ->
+                val xml = Xml.newSerializer().apply { setOutput(stream, "UTF-8"); startDocument("UTF-8", true) }
+                xml.startTag(null, "hierarchy")
+                try { xmlNode(xml, root) } finally { root.recycle() }
+                xml.endTag(null, "hierarchy").endDocument()
+            }
+        }
+        checkNotNull(instrumentation.uiAutomation.takeScreenshot()).let { bitmap ->
+            File(folder, "ui.png").outputStream().use { check(bitmap.compress(Bitmap.CompressFormat.PNG, 100, it)) }
+            bitmap.recycle()
+        }
+        atomicJson(File(folder, "state.json"), value)
+        synchronized(guard) { checkpoints.put(value) }
+        writeReport()
+    }
+
+    private fun awaitEditor(count: Int) {
+        compose.waitUntil(limits.getLong("uiWaitMs")) {
+            val field = compose.onAllNodesWithTag("item-title-input").fetchSemanticsNodes().singleOrNull()
+            field != null && field.config.getOrNull(SemanticsProperties.EditableText)?.text == "" &&
+                field.config.getOrNull(SemanticsProperties.Disabled) == null &&
+                compose.onAllNodesWithText("Items ($count)").fetchSemanticsNodes().size == 1
+        }
+        compose.onNodeWithTag("item-title-input").performScrollTo().assertIsDisplayed().assertIsEnabled()
+    }
+
+    // One statement/snapshot; cursor is closed before any cancellation or phase I/O.
+    private val ackObserver = object : InvalidationTracker.Observer("acknowledged_mutations") {
+        override fun onInvalidated(tables: Set<String>) {
+            if (!barrierArmed.get() || barrierHandled.get()) return
+            try {
+                val sql = "SELECT (SELECT count(*) FROM acknowledged_mutations) AS ackCount, " +
+                    "(SELECT count(*) FROM pending_mutations) AS pendingCount, " +
+                    "(SELECT group_concat(clientMutationId, ',') FROM " +
+                    "(SELECT clientMutationId FROM acknowledged_mutations ORDER BY clientMutationId)) AS ackIds, " +
+                    "(SELECT group_concat(clientMutationId, ',') FROM " +
+                    "(SELECT clientMutationId FROM pending_mutations ORDER BY sequence)) AS pendingIds"
+                val cursor = database.openHelper.readableDatabase.query(sql)
+                val values = rows(cursor).getJSONObject(0)
+                val closedAt = SystemClock.elapsedRealtimeNanos()
+                check(cursor.isClosed)
+                values.put("query", sql).put("cursorClosed", true).put("cursorClosedNs", closedAt)
+                val count = values.getInt("ackCount")
+                check(count <= inputs.getInt("ackBarrier")) { "Exact ACK barrier missed: $values" }
+                if (count == inputs.getInt("ackBarrier") && barrierHandled.compareAndSet(false, true)) {
+                    check(values.getInt("pendingCount") == inputs.getInt("pendingAtSettlement"))
+                    cancelAtNs = SystemClock.elapsedRealtimeNanos()
+                    jobs.forEach { it.cancel(CancellationException("M14 exact durable ACK barrier")) }
+                    cancelOperation.set(manager.cancelUniqueWork(ItemWorkScheduler.UNIQUE_WORK))
+                    values.put("cancelRequestedNs", cancelAtNs).put("cancelledProductionJobs", jobs.size)
+                    synchronized(guard) { report.put("ackBarrier", values) }
+                    // Publish immediately; root turns actual radios off BEFORE settlement waits.
+                    phase("cancel-requested", JSONObject(values.toString()))
+                    barrierReady.countDown()
+                }
+            } catch (failure: Throwable) {
+                callbackFailure.set(failure)
+                barrierReady.countDown()
+            }
+        }
+    }
+
+    @Test
+    fun frozenM14PressureCancellationAndReconnect() {
+        try {
+            awaitNetwork(false)
+            val originalActivity = compose.activity
+            val originalPid = Process.myPid()
+            compose.waitForIdle()
+            compose.runOnUiThread { originalActivity.setContent { MaterialTheme { ItemScreen(store, sync) } } }
+            awaitEditor(0)
+            for (index in 0 until fixed.length()) {
+                val item = fixed.getJSONObject(index)
+                compose.onNodeWithTag("item-title-input").performTextReplacement(item.getString("title"))
+                compose.onNodeWithText("Add").assertIsEnabled().performClick()
+                awaitEditor(index + 1)
+                compose.onNodeWithTag("item-row-${item.getString("id")}").performScrollTo().assertIsDisplayed()
+                compose.onNodeWithContentDescription("Completed: ${item.getString("title")}").assertIsOff()
+                runBlocking {
+                    assertEquals(index + 1, store.items().size)
+                    assertEquals(index + 1, store.pendingMutations().size)
+                    assertTrue(store.acknowledgedMutations().isEmpty())
+                }
+                synchronized(guard) { report.put("actualUiCreates", index + 1) }
+                capture("created-${index + 1}")
+                compose.onNodeWithTag("item-title-input").performScrollTo()
+            }
+            val original = runBlocking { store.pendingMutations() }
+            for (index in original.indices) {
+                val input = fixed.getJSONObject(index)
+                assertEquals(pendingMutation(input.getString("id"), "CREATE", input.getString("title"), false,
+                    input.getString("clientMutationId"), (index + 1).toLong()), original[index])
+                assertEquals(input.getString("payloadHash"), original[index].payloadHash)
+            }
+            compose.onNodeWithText("Pending changes: 12").performScrollTo().assertIsDisplayed()
+            capture("queued", withWork = true)
+            database.invalidationTracker.addObserver(ackObserver)
+            phase("queued")
+            gate("online")
+            awaitNetwork(true)
+            val initialDeadline = SystemClock.elapsedRealtime() + limits.getLong("initialDrainWaitMs")
+            var initial: JSONObject
+            while (true) {
+                initial = fixture()
+                if (initial.getInt("inFlight") > 0) break
+                check(SystemClock.elapsedRealtime() < initialDeadline) { "No automatic drain became active" }
+                Thread.sleep(10)
+            }
+            check(initial.getInt("mutationRequests") == 1 && initial.getInt("applied") == 1)
+            synchronized(guard) { report.put("beforeFiveRequests", initial) }
+            for (number in 1..inputs.getInt("additionalProductionRequests")) {
+                jobs += scope.launch(start = CoroutineStart.UNDISPATCHED) {
+                    event("production-request", JSONObject().put("number", number).put("entrypoint", "ItemSync.synchronize"))
+                    try {
+                        sync.synchronize()
+                    } catch (cancelled: CancellationException) {
+                        throw cancelled
+                    } catch (failure: Exception) {
+                        callbackFailure.set(failure)
+                        barrierReady.countDown()
+                    } finally {
+                        event("production-request-settled", JSONObject().put("number", number))
+                    }
+                }
+            }
+            barrierArmed.set(true)
+            val afterFive = fixture()
+            check(afterFive.getInt("mutationRequests") == 1 && afterFive.getInt("inFlight") == 1) {
+                "Five real calls did not land within the same initial active request: $afterFive"
+            }
+            synchronized(guard) {
+                report.put("additionalProductionRequests", jobs.size).put("afterFiveRequests", afterFive)
+            }
+            assertTrue("Exact ACK barrier never arrived", barrierReady.await(limits.getLong("barrierWaitMs"), TimeUnit.MILLISECONDS))
+            callbackFailure.get()?.let { throw it }
+            gate("offline")
+            awaitNetwork(false)
+            val remaining = limits.getLong("cancellationSettlementMs") - (SystemClock.elapsedRealtimeNanos() - cancelAtNs) / 1_000_000
+            check(remaining > 0) { "Offline transition exceeded cancellation deadline" }
+            runBlocking { withTimeout(remaining) { jobs.joinAll() } }
+            val workRemaining = limits.getLong("cancellationSettlementMs") - (SystemClock.elapsedRealtimeNanos() - cancelAtNs) / 1_000_000
+            check(workRemaining > 0)
+            checkNotNull(cancelOperation.get()).result.get(workRemaining, TimeUnit.MILLISECONDS)
+            event("cancellation-settled", JSONObject().put("elapsedMs", (SystemClock.elapsedRealtimeNanos() - cancelAtNs) / 1_000_000))
+            barrierArmed.set(false)
+            database.invalidationTracker.removeObserver(ackObserver)
+            runBlocking {
+                assertEquals(3, store.acknowledgedMutations().size)
+                val retained = store.pendingMutations()
+                assertEquals(original.drop(3), retained.map { it.copy(dispatched = false) })
+            }
+            assertTrue(jobs.all { it.isCompleted && it.isCancelled })
+            assertEquals(SyncState.STALE, sync.status.value.state)
+            assertSame(originalActivity, compose.activity)
+            assertEquals(originalPid, Process.myPid())
+            capture("settled", withWork = true)
+            phase("settled")
+            gate("reconnect")
+            awaitNetwork(true)
+            compose.onNodeWithText("Sync").performScrollTo().assertIsDisplayed().assertIsEnabled().performClick()
+            synchronized(guard) { report.put("foregroundSyncPresses", 1) }
+            event("foreground-sync-click")
+            compose.waitUntil(limits.getLong("finalDrainWaitMs")) {
+                compose.onAllNodesWithText("Pending changes: 0").fetchSemanticsNodes().size == 1 &&
+                    compose.onAllNodesWithText("Fresh local data").fetchSemanticsNodes().size == 1
+            }
+            awaitEditor(12)
+            runBlocking {
+                assertTrue(store.pendingMutations().isEmpty())
+                val actual = store.items().sortedBy(Item::id)
+                assertEquals(12, actual.size)
+                for (index in actual.indices) {
+                    val input = fixed.getJSONObject(index)
+                    assertEquals(Item(input.getString("id"), input.getString("title"), false, 1,
+                        inputs.getLong("firstTimestamp") + index * 1_000L), actual[index])
+                    compose.onNodeWithTag("item-row-${input.getString("id")}").assertExists()
+                }
+                assertEquals(original.map { it.clientMutationId to it.payloadHash },
+                    store.acknowledgedMutations().map { it.clientMutationId to it.payloadHash })
+            }
+            assertSame(originalActivity, compose.activity)
+            assertSame(application, compose.activity.application)
+            assertEquals(originalPid, Process.myPid())
+            val final = fixture()
+            assertEquals(12, final.getInt("applied"))
+            assertEquals(12, final.getJSONArray("items").length())
+            assertEquals(inputs.getLong("nextTimestamp"), final.getLong("nextTimestamp"))
+            assertEquals(0, final.getInt("inFlight"))
+            assertTrue(final.getInt("peakInFlight") <= inputs.getInt("maximumConcurrentMutations"))
+            synchronized(guard) { report.put("finalFixture", final).put("status", "PASS") }
+            compose.onNodeWithText("Pending changes: 0").performScrollTo().assertIsDisplayed()
+            capture("final", withWork = true)
+            phase("complete")
+        } catch (failure: Throwable) {
+            synchronized(guard) { report.put("status", "FAIL").put("error", failure.stackTraceToString()) }
+            phase("failed")
+            throw failure
+        }
+    }
+}
diff --git a/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt b/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
index 9c372c5..cbf8acc 100644
--- a/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
+++ b/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
@@ -6,7 +6,12 @@ import org.junit.Assert.assertFalse
 import org.junit.Assert.assertTrue
 import org.junit.Assert.assertThrows
 import org.junit.Test
+import kotlinx.coroutines.CompletableDeferred
+import kotlinx.coroutines.CoroutineStart
+import kotlinx.coroutines.joinAll
+import kotlinx.coroutines.launch
 import kotlinx.coroutines.runBlocking
+import kotlinx.coroutines.withTimeout
 import java.io.IOException
 
 class ItemStoreTest {
@@ -863,6 +868,66 @@ class ItemStoreTest {
         assertEquals(requests, remote.calls)
     }
 
+    @Test
+    fun m14SmallConcurrentTriggersCancelAndReplayOriginalIntent(): Unit = runBlocking {
+        var index = 0
+        val store = ItemStore(FakeItemDao(), nextId = { "small-${index + 1}" },
+            nextMutationId = { "small-create-${++index}" })
+        store.create("First")
+        store.create("Second")
+        val original = store.pendingMutations()
+        val remote = FakeRemote().apply { rows.clear() }
+        val secondApplied = CompletableDeferred<Unit>()
+        val neverReleased = CompletableDeferred<Unit>()
+        var pauseSecond = true
+        var inFlight = 0
+        var peak = 0
+        val observed = object : ItemRemote {
+            override suspend fun items() = remote.items()
+            override suspend fun send(mutation: PendingMutation): MutationResult {
+                inFlight++
+                peak = maxOf(peak, inFlight)
+                try {
+                    val response = remote.send(mutation)
+                    if (pauseSecond && mutation.clientMutationId == original[1].clientMutationId) {
+                        secondApplied.complete(Unit)
+                        neverReleased.await()
+                    }
+                    return response
+                } finally {
+                    inFlight--
+                }
+            }
+        }
+        val sync = ItemSync(store, observed)
+        var accepted = 0
+        val jobs = List(3) { launch(start = CoroutineStart.UNDISPATCHED) {
+            accepted++
+            sync.synchronize()
+        } }
+        withTimeout(5_000) { secondApplied.await() }
+        assertEquals(3, accepted)
+        assertEquals(1, store.acknowledgedMutations().size)
+        val retained = original[1].copy(dispatched = true)
+        assertEquals(listOf(retained), store.pendingMutations())
+        jobs.forEach { it.cancel() }
+        withTimeout(5_000) { jobs.joinAll() }
+        assertEquals(0, inFlight)
+        assertEquals(1, peak)
+        assertEquals(listOf(retained), store.pendingMutations())
+        assertEquals(2, remote.applied)
+        pauseSecond = false
+        sync.synchronize()
+        assertEquals(listOf(original[0].copy(dispatched = true), retained, retained), remote.mutations)
+        assertEquals(2, remote.applied)
+        assertEquals(1, remote.duplicates)
+        assertEquals(1, peak)
+        assertTrue(store.pendingMutations().isEmpty())
+        assertEquals(original.map { it.clientMutationId to it.payloadHash },
+            store.acknowledgedMutations().map { it.clientMutationId to it.payloadHash })
+        assertEquals(remote.rows.values.sortedBy(Item::id), store.items())
+    }
+
     private fun conflictRemote() = FakeRemote().apply {
         rows.clear()
         rows["conflict-001"] = Item("conflict-001", "Initial", false, 1, 1700000000000L)
diff --git a/fixture/server.py b/fixture/server.py
index 1abab59..f2ea2d0 100644
--- a/fixture/server.py
+++ b/fixture/server.py
@@ -24,15 +24,19 @@ def payload_hash(method, path, payload):
 
 
 def serialized_response(action):
-    """Keep canonical decisions serial; only response I/O and the M09 hold run unlocked."""
+    """Keep canonical decisions serial; response delays and I/O run unlocked."""
     @wraps(action)
     def handle(self):
         self.held_response = None
         self.response_headers_sent = False
         with self.server.state_lock:
+            pressure = self.server.begin_pressure(self)
             action(self)
             status, payload, delivery = self.pending_response
             held = self.held_response
+            if pressure is not None:
+                pressure.update(status=status, **getattr(self, "mutation_evidence", {}))
+                print(json.dumps(dict(event="m14-arrival", **pressure)), flush=True)
         if held is not None:
             released = held["release"].wait(max(0, held["deadline"] - time.monotonic()))
             with self.server.state_lock:
@@ -40,7 +44,25 @@ def serialized_response(action):
                 held["finished"] = time.monotonic()
                 self.server.dropped_responses += 1
             delivery = "dropped"
-        self.write_response(status, payload, delivery)
+        outcome = "error"
+        try:
+            if pressure is not None:
+                # Each handler waits independently; the state lock never covers500ms.
+                time.sleep(0.5)
+            self.write_response(status, payload, delivery)
+            outcome = "dropped" if delivery == "dropped" else "written"
+        except (BrokenPipeError, ConnectionResetError) as error:
+            if pressure is None:
+                raise
+            outcome = type(error).__name__
+        finally:
+            if pressure is not None:
+                with self.server.state_lock:
+                    pressure.update(finishedMonotonicNs=time.monotonic_ns(),
+                                    responseOutcome=outcome, headersSent=self.response_headers_sent)
+                    if pressure["mutation"]:
+                        self.server.pressure_inflight -= 1
+                    print(json.dumps(dict(event="m14-finished", **pressure)), flush=True)
     return handle
 
 
@@ -77,6 +99,9 @@ class Fixture(ThreadingHTTPServer):
         self.hold_identity = None
         self.held = None
         self.m10_case = None
+        self.m14_enabled = False
+        self.pressure_events = []
+        self.pressure_inflight = self.pressure_peak = 0
 
     def tick(self):
         timestamp = self.next_timestamp
@@ -146,6 +171,26 @@ class Fixture(ThreadingHTTPServer):
     def work_state(self):
         return dict(self.conflict_state(), case=self.m10_case, getRequests=self.get_requests)
 
+    def begin_pressure(self, handler):
+        if not self.m14_enabled or not (handler.path == "/items" or handler.path.startswith("/items/")):
+            return None
+        mutation = handler.command != "GET"
+        if mutation:
+            self.pressure_inflight += 1
+            self.pressure_peak = max(self.pressure_peak, self.pressure_inflight)
+        event = dict(number=len(self.pressure_events) + 1, method=handler.command, path=handler.path,
+                     mutation=mutation, arrivalMonotonicNs=time.monotonic_ns(),
+                     arrivalWallTimeMs=time.time_ns() // 1_000_000, delayMs=500,
+                     finishedMonotonicNs=None, responseOutcome=None)
+        self.pressure_events.append(event)
+        return event
+
+    def pressure_state(self):
+        return dict(self.conflict_state(), enabled=self.m14_enabled, getRequests=self.get_requests,
+                    delayMs=500 if self.m14_enabled else 0, inFlight=self.pressure_inflight,
+                    peakInFlight=self.pressure_peak, events=self.pressure_events,
+                    hostMonotonicNs=time.monotonic_ns(), hostWallTimeMs=time.time_ns() // 1_000_000)
+
 
 class Handler(BaseHTTPRequestHandler):
     def respond(self, status, payload, delivery="delivered"):
@@ -290,15 +335,33 @@ class Handler(BaseHTTPRequestHandler):
             self.respond(200, self.server.death_state())
         elif self.path == "/__m10":
             self.respond(200, self.server.work_state())
+        elif self.path == "/__m14":
+            self.respond(200, self.server.pressure_state())
         else:
             self.respond(404, {"error": "not found"})
 
     @serialized_response
     def do_POST(self):
-        if self.path in ("/__reset", "/__m06/reset", "/__m07/reset", "/__m09/reset", "/__m10/reset"):
+        if self.path in ("/__reset", "/__m06/reset", "/__m07/reset", "/__m09/reset", "/__m10/reset", "/__m14/reset"):
+            if self.server.pressure_inflight:
+                self.respond(409, {"error": "inflight_requests_must_finish_before_reset"})
+                return
             if self.server.held is not None and self.server.held["outcome"] == "held":
                 self.respond(409, {"error": "held_response_must_finish_before_reset"})
                 return
+        if self.path == "/__m14/reset":
+            try:
+                if self.payload():
+                    raise ValueError("Expected empty reset object")
+            except (ValueError, TypeError) as error:
+                self.respond(400, {"error": str(error)})
+                return
+            self.server.reset()
+            self.server.items = {}
+            self.server.next_timestamp = 1700001000000
+            self.server.m14_enabled = True
+            self.respond(200, self.server.pressure_state())
+            return
         if self.path == "/__m10/reset":
             try:
                 data = self.payload()
diff --git a/fixture/test_server.py b/fixture/test_server.py
index daa250e..0a22fe5 100644
--- a/fixture/test_server.py
+++ b/fixture/test_server.py
@@ -1,7 +1,7 @@
 import json
 from contextlib import contextmanager
 from http.client import RemoteDisconnected
-from threading import Thread
+from threading import Barrier, Thread
 import time
 import unittest
 from urllib.request import Request, urlopen
@@ -39,6 +39,61 @@ def mutation_wire(method, path, payload, identity):
 
 
 class FixtureContractTest(unittest.TestCase):
+    def test_m14_two_independent_delays_keep_control_responsive(self):
+        with m06_fixture() as request:
+            reset = request("POST", "/__m14/reset", {})[1]
+            self.assertEqual(([], 500, 0, 0, 1700001000000),
+                             (reset["items"], reset["delayMs"], reset["inFlight"],
+                              reset["peakInFlight"], reset["nextTimestamp"]))
+            barrier = Barrier(3)
+            responses, failures = {}, []
+
+            def send(number):
+                try:
+                    body = dict(id=f"small-{number}", title=f"Small {number}", completed=False)
+                    barrier.wait(timeout=5)
+                    responses[number] = request("POST", "/items", mutation_wire(
+                        "POST", "/items", body, f"small-create-{number}"))
+                except Exception as error:
+                    failures.append(error)
+
+            senders = [Thread(target=send, args=(number,)) for number in (1, 2)]
+            for sender in senders:
+                sender.start()
+            try:
+                barrier.wait(timeout=5)
+                deadline = time.monotonic() + 5
+                while True:
+                    active = request("GET", "/__m14")[1]
+                    if active["inFlight"] == 2:
+                        break
+                    self.assertLess(time.monotonic(), deadline, "Two requests never overlapped")
+                    time.sleep(0.01)
+                self.assertEqual(2, active["peakInFlight"])
+                self.assertTrue(all(event["finishedMonotonicNs"] is None for event in active["events"]))
+                self.assertEqual({}, responses, "Control must return before either delayed response")
+            finally:
+                for sender in senders:
+                    sender.join(timeout=5)
+            self.assertFalse(any(sender.is_alive() for sender in senders))
+            self.assertEqual([], failures)
+            self.assertEqual([201, 201], [responses[n][0] for n in (1, 2)])
+            self.assertEqual(200, request("GET", "/items")[0])
+            state = request("GET", "/__m14")[1]
+            self.assertEqual((2, 0, 2, 1, 0, 2, 1700001002000),
+                             (state["applied"], state["duplicates"], state["mutationRequests"],
+                              state["getRequests"], state["inFlight"], state["peakInFlight"],
+                              state["nextTimestamp"]))
+            events = state["events"]
+            self.assertEqual([True, True, False], [event["mutation"] for event in events])
+            self.assertLess(events[1]["arrivalMonotonicNs"], events[0]["finishedMonotonicNs"])
+            for event in events:
+                self.assertGreaterEqual(event["finishedMonotonicNs"] - event["arrivalMonotonicNs"], 500_000_000)
+                self.assertEqual((500, "written", True),
+                                 (event["delayMs"], event["responseOutcome"], event["headersSent"]))
+            self.assertEqual([1, 1], [item["version"] for item in state["items"]])
+            self.assertEqual([1700001000000, 1700001001000], sorted(item["updatedAt"] for item in state["items"]))
+
     def test_m10_fixed_a503503201_keeps_identity_and_applies_once(self):
         payload = dict(id="work-001", title="Background item", completed=False)
         wire = mutation_wire("POST", "/items", payload, "m10-create-001")
diff --git a/verification/M14-inputs.json b/verification/M14-inputs.json
new file mode 100644
index 0000000..2927636
--- /dev/null
+++ b/verification/M14-inputs.json
@@ -0,0 +1,133 @@
+{
+  "thread": "M14",
+  "profile": "phase-1",
+  "specRevision": "61280dd86ce88b6e431f408241c0998a275960aa",
+  "start": "bde06a693e4b96707e9c800152687c2b025df54a",
+  "items": [
+    {
+      "id": "pressure-001",
+      "title": "Pressure 001",
+      "completed": false,
+      "clientMutationId": "m14-create-001",
+      "payloadHash": "ae07b5d93d038ef1068755451d9cf6ed6fd415234fc834bb01e4b1698e2a713f",
+      "localUpdatedAt": 1700001000000
+    },
+    {
+      "id": "pressure-002",
+      "title": "Pressure 002",
+      "completed": false,
+      "clientMutationId": "m14-create-002",
+      "payloadHash": "29e613dfa77b8e6910680f063fa0c904cb972a580c83d6fce4ab353bcf19e9b5",
+      "localUpdatedAt": 1700001001000
+    },
+    {
+      "id": "pressure-003",
+      "title": "Pressure 003",
+      "completed": false,
+      "clientMutationId": "m14-create-003",
+      "payloadHash": "26b52518fe6f7145c3702c0eeb1c254888cc33cb0640bfd899ccd658d22cbdac",
+      "localUpdatedAt": 1700001002000
+    },
+    {
+      "id": "pressure-004",
+      "title": "Pressure 004",
+      "completed": false,
+      "clientMutationId": "m14-create-004",
+      "payloadHash": "6ae7172640b9c23da3c0ff578611b1243a1f6b9ad688bf45104be7db5340def5",
+      "localUpdatedAt": 1700001003000
+    },
+    {
+      "id": "pressure-005",
+      "title": "Pressure 005",
+      "completed": false,
+      "clientMutationId": "m14-create-005",
+      "payloadHash": "98c1b632b8da7811e6df23fc13e2bd685760d00bb84cbe74e067339952825285",
+      "localUpdatedAt": 1700001004000
+    },
+    {
+      "id": "pressure-006",
+      "title": "Pressure 006",
+      "completed": false,
+      "clientMutationId": "m14-create-006",
+      "payloadHash": "7b9ec807cd9363b8935959287284a4f18ae08dbeac85dc775351ec9d39793a25",
+      "localUpdatedAt": 1700001005000
+    },
+    {
+      "id": "pressure-007",
+      "title": "Pressure 007",
+      "completed": false,
+      "clientMutationId": "m14-create-007",
+      "payloadHash": "3013081e7f0959ad4c2b60f1aeab7df2110a594176f91d9f5eb0869e22b439d4",
+      "localUpdatedAt": 1700001006000
+    },
+    {
+      "id": "pressure-008",
+      "title": "Pressure 008",
+      "completed": false,
+      "clientMutationId": "m14-create-008",
+      "payloadHash": "675bf54598ee25e8f61be10926a97d607fa94752b8762bfab7a9759608bf7f9f",
+      "localUpdatedAt": 1700001007000
+    },
+    {
+      "id": "pressure-009",
+      "title": "Pressure 009",
+      "completed": false,
+      "clientMutationId": "m14-create-009",
+      "payloadHash": "d452b2dd73c5040e79d46ca57c1c869c5dd60204619931dba0d33bf4a40c13d0",
+      "localUpdatedAt": 1700001008000
+    },
+    {
+      "id": "pressure-010",
+      "title": "Pressure 010",
+      "completed": false,
+      "clientMutationId": "m14-create-010",
+      "payloadHash": "6be83218520a3faccb5e28ecb7503aec56bdfbc4dacc631686dc22665dba1861",
+      "localUpdatedAt": 1700001009000
+    },
+    {
+      "id": "pressure-011",
+      "title": "Pressure 011",
+      "completed": false,
+      "clientMutationId": "m14-create-011",
+      "payloadHash": "861951569c5960b3d77c6986561778b6398deb36d6afa5e8e138a153465bbd6c",
+      "localUpdatedAt": 1700001010000
+    },
+    {
+      "id": "pressure-012",
+      "title": "Pressure 012",
+      "completed": false,
+      "clientMutationId": "m14-create-012",
+      "payloadHash": "f85243c5292cc488ec16889c3fe25a4c58fcc51564e47ec21e0fedf120245948",
+      "localUpdatedAt": 1700001011000
+    }
+  ],
+  "responseDelayMs": 500,
+  "firstTimestamp": 1700001000000,
+  "nextTimestamp": 1700001012000,
+  "additionalProductionRequests": 5,
+  "ackBarrier": 3,
+  "pendingAtSettlement": 9,
+  "maximumConcurrentMutations": 2,
+  "reconnectForegroundRequests": 1,
+  "schemaVersion": 6,
+  "workDatabaseVersion": 20,
+  "workManagerVersion": "2.9.1",
+  "uniqueWorkName": "item-automatic-sync",
+  "workClass": "com.mobilesystemsevolution.kotlin.ItemSyncWorker",
+  "processDeath": false,
+  "schedulerDisabled": false,
+  "debugClockFile": false,
+  "harness": {
+    "uiWaitMs": 5000,
+    "networkWaitMs": 30000,
+    "initialDrainWaitMs": 30000,
+    "barrierWaitMs": 30000,
+    "cancellationSettlementMs": 12000,
+    "controlGateWaitMs": 30000,
+    "finalDrainWaitMs": 30000,
+    "instrumentationTimeoutSeconds": 180,
+    "adbTimeoutSeconds": 45,
+    "preSeedTeardownWaitSeconds": 30
+  },
+  "triggerAccounting": "Initial automatic dispatch is observed; exactly five additional direct production ItemSync.synchronize calls occur during its active request. No initial Sync button. One ordinary Sync button after reconnect."
+}
diff --git a/verification/M14.md b/verification/M14.md
new file mode 100644
index 0000000..16d789f
--- /dev/null
+++ b/verification/M14.md
@@ -0,0 +1,60 @@
+# M14 verification ledger — unchanged M10 regression passed
+
+- Profile: `phase-1`; Thread: `M14`; branch: `track/android-kotlin`.
+- SPEC_REVISION: `61280dd86ce88b6e431f408241c0998a275960aa`.
+- START: `bde06a693e4b96707e9c800152687c2b025df54a` (verified M10).
+- Attempt1 / repair0; actual fixed12 usage **1/3**, with no owner device run or warmup.
+- Status: root accepted the unchanged product; only seven tested support files and reporting are committed. Final history/tag audit remains root-owned.
+
+Evidence root `E`:
+`/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/android-kotlin/M14/`.
+Exact commands, timings, failures and hashes remain in the original records.
+
+| Check | Actual result | Evidence under E |
+| --- | --- | --- |
+| Initial launches | Fixture loopback bind and Gradle socket access denied by sandbox; preserved, no device execution | `fixture-host-01.*`, `support-build-01.*` |
+| Focused fixture | One test PASS,1.532s; two independent mutation delays plus GET, responsive control, measured overlap | `fixture-host-02.*`, `fixture-pretest-01/` |
+| Host/test build | One two-Item/three-trigger cancellation/replay test PASS; test-only APK build PASS,16.096s recorded command; M10 app reused | `support-build-02.*`, `support-prebuild-01/`, `support-built-01/` |
+| Harness helpers | Two tests PASS: disposable two-row WAL read and exact shell-v2 gate transport, including negative cases; no Android | `harness-host-01.*`, `support_host.py` |
+| Freeze preflight |69 sources,7 copies; all28 old host methods and14 old fixture methods unchanged; new public-void JUnit method/DEX/asset/runner checked | `source-audit.json`, `preflight-static-01.*`, `verify-freeze-01.*` |
+| Root fixed12 | Single actual JUnit PASS,0 skipped,32.22s JUnit;36.195s scenario;266 adb,18 native DB/WAL snapshots,643 raw files | `baseline-android-01/`, `baseline-android-01.*` |
+
+## Root acceptance
+
+Twelve real offline UI Adds commit pressure-001…012 with their original identities/hashes.
+Five actual `ItemSync.synchronize` calls enter during the same initial automatic request.
+The fixture independently delays every response500ms; root measured mutation peak1, within2.
+The single-statement observer sees ACKs001…003 and nine pending intents, closes the cursor,
+then requests cancellation. Actual offline transition precedes the settlement wait; settlement
+is observed341ms after the request. Nine original envelopes remain. The initial UI pressure
+label is captured; paused pending9 is a native SQLite observation, without resetting the UI.
+
+One ordinary foreground Sync after reconnect reaches exactly12 canonical Items, all version1,
+timestamps1700001000000…1700001011000, applied12/duplicate0/pending0. PID14200,
+Activity81861308, Application57006661 and store168667315 remain the same. Greedy starts
+processing and SystemJobService's concurrent request is deduplicated for WorkSpec
+`379e3aac-54fb-46e1-9219-4d3dd5fb68a5`; this is not an OS-created replacement-process claim.
+The scheduler stays enabled and M10's debug clock override is absent. No product repair is needed.
+
+Root's [independent acceptance](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M14/main-baseline-acceptance.json)
+has SHA256 `e38a0cbfdd18d5c8b6fe6faba11fe8cd56e08c65eac3af83fe1c40aa6e7ddf4a`.
+It binds all69 frozen sources, actual native SQLite/WAL, UI, HTTP, logs and clock evidence.
+[Cleanup](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M14/main-baseline-cleanup-01/result.json)
+confirms owned fixture54348 actually exited0 and is absent, port18080 is free, app processes
+are absent and network0/1/1 is restored.
+
+Root retains `main-preflight-attempt1.json` and `main-acceptance-parser-attempt1/2/3.json`
+beside that audit. These correct observation assumptions about the existing custom runner,
+the reset log window, a stripped final newline and truncated checkpoint Logcat lines.
+They do not change source, acceptance, timing, repair counts or device-run counts.
+Prior M10 host28/fixture14/native24 and OS-process evidence remain valid and are **not rerun**.
+
+## Frozen bytes
+
+`baseline-frozen-01/manifest.json`, SHA256
+`23fe9f4392c6be0dbfe7fbcd3d0085857931df7199988bfaf64ec395f90addb0`, contains exact root commands.
+App SHA256 `20a3793542f23e63037badeb3aa4d81e678dcd623b14207b3fc3b08333d85c01` is the verified M10 APK.
+Test SHA256 `e5b7093d476dcc5d19faceaf1624d6853cead6c9cb0e0248cad2e2dbaf7e707a` adds only M14 support.
+All69 sources and both APKs were compared unchanged before this reporting update.
+After root PASS, only TRACK and this ledger changed; no execution source, build, test,
+fixture, input or device action followed. No M15 work, tag or push is included.
diff --git a/verification/pressure_cancel.py b/verification/pressure_cancel.py
new file mode 100644
index 0000000..c19279e
--- /dev/null
+++ b/verification/pressure_cancel.py
@@ -0,0 +1,461 @@
+#!/usr/bin/env python3
+"""Root-only fixed M14: unchanged app, actual offline UI, cancellation and reconnect.
+
+Test-only gate files order real radio transitions around native observations. No
+Item/queue data is injected; no Activity restart, job forcing, or clock override.
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
+import xml.etree.ElementTree as ET
+
+from activity_state import ActivityStateScenario
+from background_work import BackgroundWork, job_block
+from offline_queue_restart import OfflineQueueScenario, LAUNCHER
+from process_restart import DB_NAME, PACKAGE, SERIAL
+
+INPUTS_PATH = Path(__file__).with_name('M14-inputs.json')
+INPUTS = json.loads(INPUTS_PATH.read_text())
+LIMITS = INPUTS['harness']
+REPORT_DIR = 'm14-pressure-01'
+WORK_DB = 'androidx.work.workdb'
+
+
+def sha(data):
+    return hashlib.sha256(data).hexdigest()
+
+
+def pending(item, index):
+    return dict(sequence=index + 1, itemId=item['id'], operation='CREATE', title=item['title'],
+                completed=0, clientMutationId=item['clientMutationId'], payloadHash=item['payloadHash'],
+                terminalError=None, baseVersion=None, dispatched=0)
+
+
+def read_database(folder, name, tables, version):
+    """Open a disposable copy so raw native DB/WAL evidence cannot be checkpointed."""
+    with tempfile.TemporaryDirectory(prefix='mse-m14-native-') as temporary:
+        local = Path(temporary)
+        for suffix in ('', '-wal'):
+            source = folder / (name + suffix)
+            if source.exists():
+                shutil.copyfile(source, local / source.name)
+        with sqlite3.connect(local / name) as db:
+            assert db.execute('PRAGMA integrity_check').fetchone()[0] == 'ok'
+            assert db.execute('PRAGMA user_version').fetchone()[0] == version
+            output = {}
+            for table, order in tables.items():
+                columns = [row[1] for row in db.execute(f'PRAGMA table_info({table})')]
+                assert columns, table
+                output[table] = [dict(zip(columns, [value.hex() if isinstance(value, bytes) else value for value in row]))
+                                 for row in db.execute(f'SELECT * FROM {table} ORDER BY {order}')]
+            return output
+
+
+def audit_fixture(state):
+    expected = INPUTS['items']
+    assert state['enabled'] and state['delayMs'] == 500
+    assert state['applied'] == 12 and state['nextTimestamp'] == INPUTS['nextTimestamp']
+    assert state['items'] == [dict(id=item['id'], title=item['title'], completed=False, version=1,
+                                  updatedAt=INPUTS['firstTimestamp'] + index * 1000)
+                              for index, item in enumerate(expected)]
+    assert state['identityConflicts'] == state['hashRejections'] == state['versionConflicts'] == 0
+    assert state['tombstones'] == [] and state['inFlight'] == 0
+    assert state['mutationRequests'] == 12 + state['duplicates']
+    assert state['getRequests'] == 1
+    assert len(state['results']) == 12
+    by_identity = {item['clientMutationId']: item for item in expected}
+    for result in state['results']:
+        assert result['payloadHash'] == by_identity[result['clientMutationId']]['payloadHash']
+        assert result['status'] == 201
+    intervals = []
+    mutations = []
+    for event in state['events']:
+        assert event['delayMs'] == 500
+        assert event['finishedMonotonicNs'] - event['arrivalMonotonicNs'] >= 500_000_000
+        assert event['responseOutcome'] in ('written', 'BrokenPipeError', 'ConnectionResetError')
+        if not event['mutation']:
+            assert (event['method'], event['path'], event['status']) == ('GET', '/items', 200)
+            continue
+        assert (event['method'], event['path'], event['status']) == ('POST', '/items', 201)
+        wire = event['request']
+        item = by_identity[wire['clientMutationId']]
+        body = dict(id=item['id'], title=item['title'], completed=False)
+        assert wire == dict(body, clientMutationId=item['clientMutationId'], payloadHash=item['payloadHash'])
+        canonical = json.dumps(dict(method='POST', path='/items', payload=body), sort_keys=True,
+                               separators=(',', ':'), ensure_ascii=False)
+        assert event['canonical'] == canonical
+        assert event['actualPayloadHash'] == sha(canonical.encode()) == item['payloadHash']
+        intervals += [(event['arrivalMonotonicNs'], 1), (event['finishedMonotonicNs'], -1)]
+        mutations.append(wire['clientMutationId'])
+    active = peak = 0
+    for _, change in sorted(intervals):
+        active += change
+        assert active >= 0
+        peak = max(peak, active)
+    assert active == 0 and peak == state['peakInFlight'] <= INPUTS['maximumConcurrentMutations']
+    assert len(mutations) == state['mutationRequests']
+    assert list(dict.fromkeys(mutations)) == [item['clientMutationId'] for item in expected]
+    return dict(measuredPeak=peak, actualMutations=len(mutations), applied=state['applied'],
+                duplicates=state['duplicates'], originalIdentities=list(dict.fromkeys(mutations)))
+
+
+class PressureScenario(OfflineQueueScenario):
+    # Exact previously verified raw-command/teardown helpers, including shell-v2 input.
+    adb = BackgroundWork.adb
+    teardown_gate = ActivityStateScenario.teardown_gate
+
+    def __init__(self, args):
+        super().__init__(args)
+        self.invocations = []
+        self.instrumentation = None
+        self.streams = []
+        self.native = None
+        self.result.update(scenario='M14', profile='phase-1', expectation='unchanged-M10-regression',
+                           start=INPUTS['start'], specRevision=INPUTS['specRevision'],
+                           harnessSha256=sha(Path(__file__).read_bytes()), inputsSha256=sha(INPUTS_PATH.read_bytes()),
+                           queueSeeded=False, processDeath=False, activityRecreated=False, schedulerDisabled=False,
+                           debugClockFile=False, forcedJobRuns=0, fixtureLog=str(args.fixture_log.resolve()))
+
+    def app_json(self, name, optional=False):
+        value = self.adb('shell', '-T', '-n', 'run-as', PACKAGE, 'cat', f'files/{REPORT_DIR}/{name}',
+                         allow_failure=optional)
+        if self.last_command['exit']:
+            error = (self.output / f'command-{self.command_count:03d}.stderr').read_text()
+            assert optional and self.last_command['exit'] == 1 and 'No such file or directory' in error, error
+            return None
+        return json.loads(value)
+
+    def wait_phase(self, expected):
+        deadline = min(self.instrument_started + LIMITS['instrumentationTimeoutSeconds'],
+                       time.monotonic() + LIMITS['controlGateWaitMs'] / 1000)
+        while True:
+            phase = self.app_json('phase.json', optional=True)
+            if phase and phase['phase'] == expected:
+                record = dict(expected=expected, observed=phase, command=self.command_count,
+                              hostMonotonic=time.monotonic())
+                self.result.setdefault('phaseObservations', []).append(record)
+                return phase
+            assert not phase or phase['phase'] != 'failed', phase
+            assert self.instrumentation.poll() is None, 'JUnit ended before expected phase ' + expected
+            assert time.monotonic() < deadline, ('Native phase deadline', expected, phase)
+            time.sleep(0.1)
+
+    def gate(self, name):
+        data = (name + '\n').encode('ascii')
+        path = f'files/{REPORT_DIR}/{name}.gate'
+        echoed = self.adb('shell', '-T', 'run-as', PACKAGE, 'tee', path + '.next', stdin=data, binary=True)
+        assert echoed == data and self.last_command['stderrBytes'] == 0
+        self.adb('shell', 'run-as', PACKAGE, 'mv', path + '.next', path)
+        assert self.adb('shell', '-T', '-n', 'run-as', PACKAGE, 'cat', path, binary=True) == data
+        self.result.setdefault('gates', []).append(dict(name=name, lastCommand=self.command_count,
+                                                       hostMonotonic=time.monotonic(), bytes=data.decode()))
+
+    def start_test(self):
+        command = [str(self.args.adb), '-s', SERIAL, 'shell', 'am', 'instrument', '-w', '-r',
+                   '-e', 'class', PACKAGE + '.M14PressureTest#frozenM14PressureCancellationAndReconnect', LAUNCHER]
+        self.command_count += 1
+        self.instrument_number = self.command_count
+        self.invocations.append(tuple(command[3:]))
+        self.streams = [(self.output / 'junit.stdout').open('wb'), (self.output / 'junit.stderr').open('wb')]
+        self.instrument_started = time.monotonic()
+        self.instrumentation = subprocess.Popen(command, stdout=self.streams[0], stderr=self.streams[1])
+        self.result['junit'] = dict(command=command, number=self.command_count, hostPid=self.instrumentation.pid,
+                                    startedMonotonic=self.instrument_started, status='RUNNING')
+        (self.output / 'junit-start.json').write_text(json.dumps(self.result['junit'], indent=2) + '\n')
+        with (self.output / 'commands.txt').open('a') as log:
+            log.write(f'{self.command_count:03d} {shlex.join(command)} [single asynchronous JUnit]\n')
+
+    def finish_test(self):
+        if not self.instrumentation or self.result['junit'].get('exit') is not None:
+            return
+        remaining = self.instrument_started + LIMITS['instrumentationTimeoutSeconds'] - time.monotonic()
+        timed_out = False
+        code = self.instrumentation.poll()
+        if code is None:
+            try:
+                code = self.instrumentation.wait(timeout=max(0, remaining))
+            except subprocess.TimeoutExpired:
+                timed_out = True
+                self.instrumentation.kill()
+                code = self.instrumentation.wait(timeout=LIMITS['adbTimeoutSeconds'])
+        for stream in self.streams:
+            stream.close()
+        self.streams = []
+        self.result['junit'].update(exit=code, endedMonotonic=time.monotonic(),
+                                    elapsedSeconds=round(time.monotonic() - self.instrument_started, 3),
+                                    status='TIMEOUT' if timed_out else 'FINISHED')
+        with (self.output / 'command-records.jsonl').open('a') as log:
+            log.write(json.dumps(self.result['junit']) + '\n')
+        with (self.output / 'commands.txt').open('a') as log:
+            log.write(f'    junit command {self.instrument_number:03d} exit={code}\n')
+        assert not timed_out, 'The single instrumentation invocation exceeded its frozen deadline'
+
+    def export_native(self):
+        if self.native is not None:
+            return self.native
+        self.adb('exec-out', 'run-as', PACKAGE, 'tar', '-cf', '-', '-C', 'files', REPORT_DIR, binary=True)
+        archive = self.output / f'command-{self.command_count:03d}.stdout'
+        native = self.output / 'native'
+        native.mkdir()
+        with tarfile.open(archive) as source:
+            for member in source.getmembers():
+                path = Path(member.name)
+                assert not path.is_absolute() and '..' not in path.parts and path.parts[0] == REPORT_DIR
+                assert member.isdir() or member.isfile(), member.name
+            source.extractall(native, filter='data')
+        self.native = native / REPORT_DIR
+        self.result['nativeEvidence'] = str(self.native)
+        return self.native
+
+    def audit_native(self):
+        observed = json.loads((self.native / 'result.json').read_text())
+        assert observed['status'] == 'PASS' and observed['testObserverRemoved']
+        assert (observed['actualUiCreates'], observed['additionalProductionRequests'], observed['foregroundSyncPresses']) == (12, 5, 1)
+        assert not observed['processDeath'] and not observed['schedulerDisabled'] and not observed['debugClockFile']
+        points = observed['checkpoints']
+        assert [point['stage'] for point in points] == [f'created-{number}' for number in range(1, 13)] + ['queued', 'settled', 'final']
+        for key in ('pid', 'activity', 'application', 'store'):
+            assert len({point[key] for point in points}) == 1, key
+        original = [pending(item, index) for index, item in enumerate(INPUTS['items'])]
+        item_tables = dict(items='id', pending_mutations='sequence', acknowledged_mutations='clientMutationId',
+                           automatic_sync='id', tombstones='id', sync_metadata='id', sqlite_sequence='name')
+        names = dict(items='items', pending_mutations='pending', acknowledged_mutations='acknowledged',
+                     automatic_sync='automatic', tombstones='tombstones', sync_metadata='metadata', sqlite_sequence='allocator')
+        database_count = 0
+        for index, point in enumerate(points):
+            stage, folder = point['stage'], self.native / point['stage']
+            assert json.loads((folder / 'state.json').read_text()) == point
+            tables = read_database(folder, DB_NAME, item_tables, INPUTS['schemaVersion'])
+            database_count += 1
+            for table, field in names.items():
+                assert tables[table] == point[field], (stage, table)
+            assert tables['tombstones'] == []
+            if stage.startswith('created-') or stage == 'queued':
+                count = index + 1 if stage.startswith('created-') else 12
+                assert point['pending'] == original[:count] and point['acknowledged'] == []
+                assert point['items'] == [dict(id=item['id'], title=item['title'], completed=0, version=0,
+                                             updatedAt=item['localUpdatedAt']) for item in INPUTS['items'][:count]]
+                assert point['renderedPendingLabel'] == f'Pending changes: {count}'
+                assert point['metadata'] == []
+            elif stage == 'settled':
+                assert [{**row, 'dispatched': 0} for row in point['pending']] == original[3:]
+                assert len(point['acknowledged']) == 3
+                assert point['metadata'] == []
+            else:
+                assert point['pending'] == [] and len(point['acknowledged']) == 12
+                assert point['renderedPendingLabel'] == 'Pending changes: 0'
+                assert point['items'] == [{**item, 'completed': 0} for item in observed['finalFixture']['items']]
+                metadata, = point['metadata']
+                assert metadata['id'] == 1 and metadata['lastSuccessfulRefreshAt'] > 0
+            assert point['allocator'] == [dict(name='pending_mutations', seq=min(index + 1, 12))]
+            for ack in point['acknowledged']:
+                item = next(item for item in INPUTS['items'] if item['clientMutationId'] == ack['clientMutationId'])
+                assert ack['payloadHash'] == item['payloadHash'] and ack['statusCode'] == 201
+                assert json.loads(ack['responseBody'])['item']['id'] == item['id']
+            assert (folder / 'semantics.txt').read_text().strip()
+            assert ET.parse(folder / 'ui.xml').getroot().tag == 'hierarchy'
+            assert (folder / 'ui.png').read_bytes().startswith(b'\x89PNG\r\n\x1a\n')
+            if 'workDatabase' in point:
+                work = read_database(folder, WORK_DB, dict(WorkSpec='id', WorkName='name,work_spec_id',
+                    SystemIdInfo='work_spec_id,generation', Dependency='work_spec_id,prerequisite_id'), INPUTS['workDatabaseVersion'])
+                database_count += 1
+                assert point['workDatabase'] == dict(version=INPUTS['workDatabaseVersion'], **work)
+                cycle, = point['automatic']
+                spec, = work['WorkSpec']
+                assert spec['id'] == cycle['workId'] == self.work_id
+                assert spec['generation'] == 0 and spec['worker_class_name'] == INPUTS['workClass']
+                assert spec['required_network_type'] == 1
+                assert work['WorkName'] == [dict(name=INPUTS['uniqueWorkName'], work_spec_id=self.work_id)]
+                assert cycle['httpAttempts'] <= 3
+                if stage in ('settled', 'final'):
+                    assert spec['state'] == 5, ('Current unique work not cancelled', stage, spec)
+        events = observed['events']
+        requests = [event for event in events if event['event'] == 'production-request']
+        completions = [event for event in events if event['event'] == 'production-request-settled']
+        assert [event['number'] for event in requests] == list(range(1, 6))
+        assert sorted(event['number'] for event in completions) == list(range(1, 6))
+        assert len([event for event in events if event['event'] == 'created']) == 1
+        assert len([event for event in events if event['event'] == 'destroyed']) == 1
+        assert not [event for event in events if event['event'] == 'saved']
+        before, after = observed['beforeFiveRequests'], observed['afterFiveRequests']
+        for state in (before, after):
+            assert state['mutationRequests'] == state['inFlight'] == 1 and state['getRequests'] == 0
+            event, = state['events']
+            assert event['finishedMonotonicNs'] is None
+        assert before['events'][0]['arrivalMonotonicNs'] == after['events'][0]['arrivalMonotonicNs']
+        barrier = observed['ackBarrier']
+        assert (barrier['ackCount'], barrier['pendingCount'], barrier['cancelledProductionJobs']) == (3, 9, 5)
+        assert barrier['cursorClosed'] and barrier['cursorClosedNs'] <= barrier['cancelRequestedNs']
+        assert barrier['ackIds'].split(',') == [item['clientMutationId'] for item in INPUTS['items'][:3]]
+        assert barrier['pendingIds'].split(',') == [item['clientMutationId'] for item in INPUTS['items'][3:]]
+        assert all(event['elapsedRealtimeNs'] < barrier['cancelRequestedNs'] for event in requests)
+        offline, = [event for event in events if event['event'] == 'root-gate-offline']
+        settled, = [event for event in events if event['event'] == 'cancellation-settled']
+        reconnect, = [event for event in events if event['event'] == 'root-gate-reconnect']
+        assert barrier['cancelRequestedNs'] <= offline['elapsedRealtimeNs'] <= settled['elapsedRealtimeNs'] < reconnect['elapsedRealtimeNs']
+        assert settled['elapsedMs'] <= LIMITS['cancellationSettlementMs']
+        assert not [event for event in events if event['event'] == 'foreground-send-enter'
+                    and barrier['cancelRequestedNs'] < event['elapsedRealtimeNs'] < reconnect['elapsedRealtimeNs']]
+        assert not [event for event in completions if event['elapsedRealtimeNs'] > settled['elapsedRealtimeNs']]
+        sends = [event for event in events if event['event'] == 'foreground-send-enter']
+        finishes = [event for event in events if event['event'] == 'foreground-send-settled']
+        assert len(sends) == len(finishes)
+        self.result.update(nativeDatabases=database_count, nativeCheckpoints=len(points),
+                           activity=points[0]['activity'], application=points[0]['application'], pid=points[0]['pid'],
+                           productionRequests=5, foregroundReconnectRequests=1, localAckBarrier=3,
+                           retainedPending=9, cancellationElapsedMs=settled['elapsedMs'])
+
+    def run(self):
+        assert 'Item fixture listening' in self.args.fixture_log.read_text(), 'Owned fixture is not ready'
+        self.result['initialNetwork'] = self.wait_network(True)
+        self.adb('shell', 'am', 'force-stop', PACKAGE)
+        self.teardown_gate()
+        self.adb('install', '-r', str(self.args.apk.resolve()))
+        self.adb('install', '-r', str(self.args.test_apk.resolve()))
+        assert self.adb('shell', 'pm', 'clear', PACKAGE) == 'Success'
+        self.teardown_gate()
+        self.adb('shell', 'input', 'keyevent', 'KEYCODE_WAKEUP')
+        self.adb('shell', 'wm', 'dismiss-keyguard')
+        self.adb('logcat', '-c')
+        reset = self.http('/__m14/reset', {})
+        assert reset['items'] == reset['events'] == [] and reset['applied'] == 0
+        self.fixture_offset = self.args.fixture_log.stat().st_size
+        self.result['fixtureLogStartOffset'] = self.fixture_offset
+        self.go_offline()
+        self.start_test()
+        queued = self.wait_phase('queued')
+        self.result['queuedNetwork'] = self.wait_network(False)
+        point = self.app_json('queued/state.json')
+        self.work_id, = [cycle['workId'] for cycle in point['automatic']]
+        mapping, = point['workDatabase']['SystemIdInfo']
+        assert mapping['work_spec_id'] == self.work_id and mapping['generation'] == 0
+        native_pid = self.adb('shell', 'pidof', PACKAGE)
+        assert native_pid == str(queued['pid'])
+        self.result['queuedJobBinding'] = job_block(self.adb('shell', 'dumpsys', 'jobscheduler', PACKAGE),
+                                                    self.work_id, mapping['system_id'])
+        self.result['workId'] = self.work_id
+        no_http = self.http('/__m14')
+        assert no_http['mutationRequests'] == no_http['getRequests'] == 0
+        assert 'm10-clock' not in self.adb('shell', 'run-as', PACKAGE, 'ls', 'files').splitlines()
+        self.gate('online')  # Permission first; native is already waiting as the real network comes up.
+        self.result['initialOnlineNetwork'] = self.restore_network()
+        barrier = self.wait_phase('cancel-requested')
+        assert barrier['cursorClosed'] and barrier['ackCount'] == 3 and barrier['pendingCount'] == 9
+        self.result['offlineRequestedHostMonotonic'] = time.monotonic()
+        self.go_offline()  # Do not wait for cancellation/HTTP settlement before this actual transition.
+        self.result['offlineConfirmedHostMonotonic'] = time.monotonic()
+        self.gate('offline')
+        self.wait_phase('settled')
+        assert self.adb('shell', 'pidof', PACKAGE) == native_pid
+        paused = self.app_json('settled/state.json')
+        assert len(paused['acknowledged']) == 3 and len(paused['pending']) == 9
+        self.result['pausedFixture'] = self.http('/__m14')
+        assert self.result['pausedFixture']['inFlight'] == 0
+        assert self.result['pausedFixture']['peakInFlight'] <= 2
+        self.result['cancelledJobsDump'] = self.adb('shell', 'dumpsys', 'jobscheduler', PACKAGE)
+        before_reconnect = self.http('/__m14')
+        assert before_reconnect['events'] == self.result['pausedFixture']['events'], 'New send after settlement while offline'
+        self.result['offlineBeforeReconnect'] = self.wait_network(False)
+        self.gate('reconnect')
+        self.result['reconnectNetwork'] = self.restore_network()
+        self.wait_phase('complete')
+        self.finish_test()
+        self.export_native()
+        text = (self.output / 'junit.stdout').read_text()
+        statuses = [int(value) for value in re.findall(r'INSTRUMENTATION_STATUS_CODE: (-?\d+)', text)]
+        assert self.result['junit']['exit'] == 0 and statuses == [1, 0], text
+        assert re.search(r'OK \(1 tests?\)', text) and 'frozenM14PressureCancellationAndReconnect' in text, text
+        self.result['junit'].update(status='PASS', tests=1, failures=0, skipped=0)
+        self.audit_native()
+        self.result['finalFixture'] = self.http('/__m14')
+        self.result['fixtureAudit'] = audit_fixture(self.result['finalFixture'])
+        self.result['status'] = 'PASS'
+
+    def collect(self):
+        if self.instrumentation is not None and self.instrumentation.poll() is None:
+            # Failure cleanup only, never a task reset or second case.
+            self.result['junit']['cleanupTerminated'] = True
+            self.adb('shell', 'am', 'force-stop', PACKAGE)
+            self.finish_test()
+        for stream in self.streams:
+            stream.close()
+        self.streams = []
+        if self.instrumentation is not None:
+            if self.result['junit'].get('exit') is None:
+                self.finish_test()
+            if self.native is None:
+                try:
+                    self.export_native()
+                except Exception as error:
+                    self.result['partialNativeExportError'] = repr(error)
+        logcat = self.adb('logcat', '-d', '-v', 'threadtime')
+        (self.output / 'logcat.txt').write_text(logcat)
+        if self.result.get('status') == 'PASS':
+            lines = [line for line in logcat.splitlines() if self.work_id in line]
+            assert any('ItemSyncWorker' in line for line in lines), 'No actual Worker execution log'
+            self.result['schedulerObservation'] = dict(workId=self.work_id, sameProcess=True,
+                greedy=[line for line in lines if 'WM-GreedyScheduler' in line],
+                systemJobService=[line for line in lines if 'WM-SystemJobService' in line],
+                worker=[line for line in lines if 'ItemSyncWorker' in line],
+                processor=[line for line in lines if 'WM-Processor' in line],
+                claim='Raw winner/dedup trace; no OS-created replacement process is claimed for M14.')
+        fixture_bytes = self.args.fixture_log.read_bytes()
+        window = fixture_bytes[self.fixture_offset:]
+        (self.output / 'fixture-window.log').write_bytes(window)
+        self.result.update(fixtureLogEndOffset=len(fixture_bytes), fixtureWindowSha256=sha(window))
+
+
+def main():
+    parser = argparse.ArgumentParser(description=__doc__)
+    parser.add_argument('--apk', type=Path, required=True)
+    parser.add_argument('--test-apk', type=Path, required=True)
+    parser.add_argument('--fixture-log', type=Path, required=True)
+    parser.add_argument('--output', type=Path, required=True)
+    parser.add_argument('--adb', type=Path, default=Path(os.environ.get('ANDROID_HOME', '')) / 'platform-tools/adb')
+    parser.set_defaults(expect='unchanged-M10-regression')
+    args = parser.parse_args()
+    scenario = PressureScenario(args)
+    started = time.monotonic()
+    error = None
+    try:
+        scenario.run()
+    except Exception as failure:
+        error = failure
+        scenario.result.update(status='FAIL', error=repr(failure))
+    finally:
+        try:
+            scenario.collect()
+        except Exception as collection_error:
+            scenario.result.update(status='FAIL', collectionError=repr(collection_error))
+            error = error or collection_error
+        try:
+            scenario.adb('shell', 'am', 'force-stop', PACKAGE)
+            absent = not scenario.adb('shell', 'pidof', PACKAGE, allow_failure=True)
+            network = scenario.restore_network()
+            scenario.result['cleanup'] = dict(appAbsent=absent, network=network)
+            assert absent
+        except Exception as cleanup_error:
+            scenario.result.update(status='FAIL', cleanupError=repr(cleanup_error))
+            error = error or cleanup_error
+        scenario.result.update(adbCommands=scenario.command_count, elapsedSeconds=round(time.monotonic() - started, 3))
+        (scenario.output / 'result.json').write_text(json.dumps(scenario.result, indent=2) + '\n')
+        raw = {str(path.relative_to(scenario.output)): sha(path.read_bytes())
+               for path in sorted(scenario.output.rglob('*')) if path.is_file()}
+        (scenario.output / 'raw-files.json').write_text(json.dumps(raw, indent=2) + '\n')
+        print(json.dumps(scenario.result, indent=2), flush=True)
+    if error is not None:
+        raise error
+
+
+if __name__ == '__main__':
+    main()
