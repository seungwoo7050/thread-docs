# M03 — Remote Synchronization Baseline

## `feat(android): add M03 foreground sync candidate`

diff --git a/.gitignore b/.gitignore
index c48dbf7..c16eca6 100644
--- a/.gitignore
+++ b/.gitignore
@@ -4,3 +4,4 @@ local.properties
 .idea/
 *.iml
 .DS_Store
+__pycache__/
diff --git a/TRACK.md b/TRACK.md
index 67bfbbe..f071619 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -33,9 +33,43 @@ the stored Items. This follows [Room's schema and migration guidance](https://de
 Room 2.6.1 is pinned to retain the existing Kotlin 1.9 toolchain; [Room 2.7 requires Kotlin 2.0](https://developer.android.com/jetpack/androidx/releases/room#2.7.0).
 Suspend DAO methods run storage work off the main thread using [Room's coroutine support](https://developer.android.com/training/data-storage/room/async-queries).
 
-No network, remote fixture, sync intent/queue, conflict policy, background work, push,
-global state framework, or future schema columns are included. Draft/selection restoration
-is not claimed. Persisted Items survive a full app-process replacement, not just a new Activity.
+M02 included no network or remote fixture. Persisted Items survive a full app-process
+replacement, not just a new Activity. Draft/selection restoration is not claimed.
+
+## M03 boundary
+
+Attempt 1 is a candidate with **failed Android M03 acceptance**. Initial canonical pull
+works, but the create-sync step is unresolved and canonical restart is unrun. The complete
+failure ledger and reproducible commands are in `verification/M03.md`; do not treat M03
+as verified until main's bounded repair and independent verification succeed.
+
+The existing local CRUD still writes Room and reads committed rows before updating the UI.
+The new **Sync** button is the only network trigger. `ItemSync` compares those local rows
+against its last successful canonical snapshot, sends changed creates/title/completion/deletes
+serially, fetches the canonical list, replaces Room rows in one transaction, and reads Room
+back. `ItemScreen` then reads through `ItemStore`; no HTTP response is a second UI store.
+The schema remains version 1 with the same five fields. Remote versions and timestamps
+replace local values after successful synchronization.
+
+The comparison snapshot is memory only. On first sync, existing version-zero local creates
+are uploaded before downloading the canonical list. Synced rows are initially downloaded
+as canonical; edits/deletes interrupted by a process restart are **not** durable sync intent.
+There is no queue, automatic/offline upload, retry loop, mutation identity, hash, baseVersion,
+conflict policy, scheduler, or push. Partial remote success and restart-before-upload recovery
+are not guaranteed by M03. HTTP failure is reported without claiming remote confirmation.
+Ordinary app launch reads saved Room rows without starting a network request.
+
+`fixture/server.py` is one local in-memory Python HTTP server. Its fixed seed, zero delay,
+clock, sorted GET and POST/PATCH/DELETE responses implement the shared M03 scenario.
+`POST /__reset` restores that fixed state; `GET /__state` reports Items and nextTimestamp
+for verification only. It binds loopback port 18080, accessible to this emulator as
+`10.0.2.2:18080`. No backend database, deployment, authentication or public service exists.
+
+OkHttp 4.12.0 is pinned; calls run on `Dispatchers.IO`, close response bodies, and disable
+automatic connection retries/redirects. These options follow the pinned
+[OkHttp client API](https://github.com/square/okhttp/blob/parent-4.12.0/okhttp/src/main/kotlin/okhttp3/OkHttpClient.kt).
+Cleartext traffic is permitted only for the local emulator host, following Android's
+[per-domain network security configuration](https://developer.android.com/privacy-and-security/security-config#CleartextTrafficPermitted).
 
 ## Pinned build
 
@@ -43,6 +77,7 @@ is not claimed. Persisted Items survive a full app-process replacement, not just
 - Android Gradle Plugin 8.6.1, Kotlin 1.9.24, Compose compiler 1.5.14
 - Compose BOM 2024.06.00, Activity Compose 1.9.0
 - Room runtime/KT extensions/compiler 2.6.1, Kotlin kapt 1.9.24
+- OkHttp 4.12.0; Python 3 standard library for the local fixture
 - Compile SDK 35, target SDK 34, minimum SDK 26
 - Required verification device: API 34, Pixel 6, ARM64 Google APIs image, serial `emulator-5554`
 
@@ -55,13 +90,37 @@ Keep machine paths in the environment or ignored `local.properties`, never in Gi
 
 ```sh
 ./gradlew --no-daemon :app:testDebugUnitTest :app:assembleDebug :app:assembleDebugAndroidTest
+python3 -m unittest discover -s fixture -v
+python3 -u fixture/server.py --port 18080
+```
+
+Keep the fixture running in that terminal. In another terminal, after acquiring the shared
+emulator lease:
+
+```sh
 ANDROID_SERIAL=emulator-5554 ./gradlew --no-daemon :app:connectedDebugAndroidTest
 ```
 
-The second command installs the app and instrumentation APK and runs the actual Compose
-UI sequence plus persistence/error tests. It requires the shared emulator lease. The application package is
+This installs both APKs and runs the original Compose UI/persistence/error regressions plus
+M03's HTTP/Room/UI convergence and canonical transaction rollback tests. The application package is
 `com.mobilesystemsevolution.kotlin`; the test package adds `.test`.
 
+To leave exactly the M03 canonical table in the app database and check it across an actual
+external process replacement, run on the same lease:
+
+```sh
+ANDROID_SERIAL=emulator-5554 ./gradlew --no-daemon :app:connectedDebugAndroidTest \
+  '-Pandroid.testInstrumentationRunnerArguments.class=com.mobilesystemsevolution.kotlin.ItemUiTest#frozenM03SequenceUsesHttpRoomAndTwoIndependentInstances'
+python3 verification/canonical_restart.py --apk app/build/outputs/apk/debug/app-debug.apk \
+  --adb "$ANDROID_HOME/platform-tools/adb" --output /tmp/kotlin-M03-canonical-restart
+```
+
+The filtered test resets only the fixture and its test databases before the sequence. The
+restart harness then performs no install, clear, instrumentation or data writes; it launches
+the normal Activity, captures exact UI/SQLite state, externally force-stops and relaunches,
+requires different PIDs, and compares both canonical Items in all five fields. See
+`verification/M03.md` for the frozen inputs, actual invocations and evidence.
+
 The separate process-restart harness also requires the lease. Supply a new evidence directory:
 
 ```sh
diff --git a/app/build.gradle.kts b/app/build.gradle.kts
index c06bb7c..51140b5 100644
--- a/app/build.gradle.kts
+++ b/app/build.gradle.kts
@@ -38,6 +38,7 @@ dependencies {
     implementation("androidx.compose.ui:ui")
     implementation("androidx.room:room-runtime:2.6.1")
     implementation("androidx.room:room-ktx:2.6.1")
+    implementation("com.squareup.okhttp3:okhttp:4.12.0")
     kapt("androidx.room:room-compiler:2.6.1")
 
     testImplementation("junit:junit:4.13.2")
diff --git a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemDatabaseTest.kt b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemDatabaseTest.kt
index 6ddf8e2..798daae 100644
--- a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemDatabaseTest.kt
+++ b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemDatabaseTest.kt
@@ -64,6 +64,49 @@ class ItemDatabaseTest {
         }
     }
 
+    @Test
+    fun isolatedM02DatabasesCannotShareChangesWithoutSync() = runBlocking {
+        val secondName = "m03-isolated-second.db"
+        context.deleteDatabase(secondName)
+        val second = ItemDatabase.open(context, secondName)
+        try {
+            withDatabase { first ->
+                val firstStore = ItemStore(first.items(), nextId = { "device-001" },
+                    now = { 1700000100000L })
+                val secondStore = ItemStore(second.items())
+                assertTrue(firstStore.items().isEmpty())
+                assertTrue(secondStore.items().isEmpty())
+                firstStore.create("Gamma")
+                val expected = listOf(Item("device-001", "Gamma", false, 0, 1700000100000L))
+                assertEquals(expected, firstStore.items())
+                assertTrue("Independent Room database cannot see the first instance's write",
+                    secondStore.items().isEmpty())
+                android.util.Log.i("M03Reproduction", "first=${firstStore.items()}; second=${secondStore.items()}")
+            }
+        } finally {
+            second.close()
+        }
+        val reopened = ItemDatabase.open(context, secondName)
+        try {
+            assertTrue(ItemStore(reopened.items()).items().isEmpty())
+        } finally {
+            reopened.close()
+        }
+    }
+
+    @Test
+    fun canonicalReplacementRollsBackIfAnyRowCannotBeInserted() = runBlocking {
+        withDatabase { database ->
+            val store = ItemStore(database.items())
+            database.items().insert(fixture.toEntity())
+            val duplicate = Item("remote-001", "Alpha", false, 1, 1700000000000L)
+            assertThrows(android.database.sqlite.SQLiteConstraintException::class.java) {
+                runBlocking { store.replaceWithCanonical(listOf(duplicate, duplicate)) }
+            }
+            assertEquals(listOf(fixture), store.items())
+        }
+    }
+
     @Test
     fun unknownSchemaVersionRejectsOpenWithoutDeletingExistingData() {
         runBlocking { withDatabase { it.items().insert(fixture.toEntity()) } }
diff --git a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt
index 9f02269..1eb6b97 100644
--- a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt
+++ b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt
@@ -1,5 +1,7 @@
 package com.mobilesystemsevolution.kotlin
 
+import android.view.WindowInsets
+import android.view.inputmethod.InputMethodManager
 import androidx.activity.compose.setContent
 import androidx.compose.material3.MaterialTheme
 import androidx.compose.ui.semantics.SemanticsProperties
@@ -7,6 +9,7 @@ import androidx.compose.ui.semantics.getOrNull
 import androidx.compose.ui.test.SemanticsMatcher
 import androidx.compose.ui.test.assertCountEquals
 import androidx.compose.ui.test.assertIsDisplayed
+import androidx.compose.ui.test.assertIsEnabled
 import androidx.compose.ui.test.assertIsOff
 import androidx.compose.ui.test.assertIsOn
 import androidx.compose.ui.test.assertTextEquals
@@ -15,15 +18,22 @@ import androidx.compose.ui.test.junit4.createAndroidComposeRule
 import androidx.compose.ui.test.onNodeWithContentDescription
 import androidx.compose.ui.test.onNodeWithTag
 import androidx.compose.ui.test.onNodeWithText
+import androidx.compose.ui.test.onRoot
 import androidx.compose.ui.test.onAllNodesWithText
 import androidx.compose.ui.test.onAllNodesWithTag
 import androidx.compose.ui.test.performClick
+import androidx.compose.ui.test.performScrollTo
 import androidx.compose.ui.test.performTextInput
 import androidx.compose.ui.test.performTextReplacement
+import androidx.compose.ui.test.printToString
 import androidx.test.ext.junit.runners.AndroidJUnit4
 import androidx.test.platform.app.InstrumentationRegistry
 import androidx.compose.ui.state.ToggleableState
 import kotlinx.coroutines.runBlocking
+import okhttp3.OkHttpClient
+import okhttp3.Request
+import okhttp3.RequestBody.Companion.toRequestBody
+import org.json.JSONObject
 import org.junit.Assert.assertEquals
 import org.junit.Rule
 import org.junit.Test
@@ -139,4 +149,146 @@ class ItemUiTest {
             database.close()
         }
     }
+
+    @Test
+    fun frozenM03SequenceUsesHttpRoomAndTwoIndependentInstances() {
+        val context = InstrumentationRegistry.getInstrumentation().targetContext
+        fixtureControl("POST", "__reset")
+        context.deleteDatabase("m03-second.db")
+        val firstDatabase = ItemDatabase.open(context)
+        val secondDatabase = ItemDatabase.open(context, "m03-second.db")
+        try {
+            val first = ItemStore(firstDatabase.items(), nextId = { "device-001" }, now = { 1800000000000L })
+            val remote = HttpItemRemote()
+            val sync = ItemSync(first, remote)
+            val seeds = listOf(
+                Item("remote-001", "Alpha", false, 1, 1700000000000L),
+                Item("remote-002", "Beta", false, 1, 1700000000000L),
+            )
+            val gamma = Item("device-001", "Gamma", false, 1, 1700000100000L)
+            val renamed = Item("remote-001", "Alpha synced", false, 2, 1700000101000L)
+            val completed = Item("remote-001", "Alpha synced", true, 3, 1700000102000L)
+            assertEquals(emptyList<Item>(), runBlocking { first.items() })
+            compose.runOnUiThread {
+                compose.activity.setContent { MaterialTheme { ItemScreen(first, sync) } }
+            }
+            awaitText("Items (0)")
+
+            fun closeKeyboard() {
+                compose.runOnUiThread {
+                    val view = compose.activity.window.decorView
+                    compose.activity.getSystemService(InputMethodManager::class.java)
+                        .hideSoftInputFromWindow(view.windowToken, 0)
+                }
+                compose.waitUntil(5_000) {
+                    var hidden = false
+                    compose.runOnUiThread {
+                        hidden = compose.activity.window.decorView.rootWindowInsets
+                            ?.isVisible(WindowInsets.Type.ime()) == false
+                    }
+                    hidden
+                }
+            }
+
+            fun awaitRoom(expected: List<Item>, timeout: Long) {
+                try {
+                    compose.waitUntil(timeout) { runBlocking { first.items() } == expected }
+                } catch (error: Exception) {
+                    android.util.Log.e("M03Click", "Room=${runBlocking { first.items() }}; expected=$expected\n${compose.onRoot().printToString()}")
+                    throw error
+                }
+            }
+
+            fun awaitSyncReady() {
+                compose.waitUntil(5_000) {
+                    compose.onNodeWithText("Sync").fetchSemanticsNode()
+                        .config.getOrNull(SemanticsProperties.Disabled) == null
+                }
+            }
+
+            fun synchronizeAndReadRoom(expected: List<Item>) {
+                // A draft title can already match awaitText while its local write is in flight.
+                awaitSyncReady()
+                closeKeyboard()
+                val button = compose.onNodeWithText("Sync").performScrollTo()
+                    .assertIsDisplayed().assertIsEnabled()
+                android.util.Log.i("M03Click", "before sync: ${runBlocking { first.items() }}\n${button.printToString()}")
+                button.performClick()
+                awaitRoom(expected, 10_000)
+                awaitSyncReady()
+                compose.onNodeWithTag("sync-error").assertDoesNotExist()
+                assertEquals(expected, runBlocking { firstDatabase.items().readAll().map(ItemEntity::toDomain) })
+                android.util.Log.i("M03Canonical", "first=$expected")
+            }
+
+            synchronizeAndReadRoom(seeds)
+            compose.onNodeWithText("Alpha").assertIsDisplayed()
+            compose.onNodeWithText("Beta").assertIsDisplayed()
+            compose.onNodeWithTag("item-count").assertTextEquals("Items (2)")
+
+            compose.onNodeWithTag("item-title-input").performTextInput("Gamma")
+            closeKeyboard()
+            compose.onNodeWithText("Add").performScrollTo().assertIsDisplayed().assertIsEnabled().performClick()
+            awaitRoom(seeds + gamma.copy(version = 0, updatedAt = 1800000000000L), 5_000)
+            android.util.Log.i("M03Click", "committed after Add=${runBlocking { first.items() }}")
+            awaitText("Gamma")
+            synchronizeAndReadRoom(listOf(gamma, *seeds.toTypedArray()))
+
+            compose.onNodeWithContentDescription("Edit Alpha").performScrollTo().performClick()
+            compose.onNodeWithTag("item-title-input").performTextReplacement("Alpha synced")
+            closeKeyboard()
+            compose.onNodeWithText("Save").performScrollTo().assertIsDisplayed().assertIsEnabled().performClick()
+            awaitText("Alpha synced")
+            synchronizeAndReadRoom(listOf(gamma, renamed, seeds[1]))
+
+            compose.onNodeWithContentDescription("Completed: Alpha synced").performScrollTo().assertIsOff().performClick()
+            compose.waitUntil(5_000) {
+                compose.onNodeWithContentDescription("Completed: Alpha synced").fetchSemanticsNode()
+                    .config[SemanticsProperties.ToggleableState] == ToggleableState.On
+            }
+            synchronizeAndReadRoom(listOf(gamma, completed, seeds[1]))
+
+            compose.onNodeWithContentDescription("Delete Beta").performScrollTo().performClick()
+            awaitText("Items (2)")
+            val expected = listOf(gamma, completed)
+            synchronizeAndReadRoom(expected)
+            compose.onNodeWithText("Beta").assertDoesNotExist()
+
+            val second = ItemStore(secondDatabase.items())
+            assertEquals(emptyList<Item>(), runBlocking { second.items() })
+            val secondSync = ItemSync(second, HttpItemRemote())
+            runBlocking { secondSync.synchronize() }
+            assertEquals(expected, runBlocking { first.items() })
+            assertEquals(expected, runBlocking { second.items() })
+            assertEquals(expected, runBlocking { remote.items() })
+            assertEquals(1700000104000L, fixtureControl("GET", "__state").getLong("nextTimestamp"))
+            android.util.Log.i("M03Canonical", "final first=${runBlocking { first.items() }}; second=${runBlocking { second.items() }}")
+            compose.runOnUiThread {
+                compose.activity.setContent { MaterialTheme { ItemScreen(second, secondSync) } }
+            }
+            awaitText("Items (2)")
+            compose.onAllNodes(rowMatcher).assertCountEquals(2)
+            compose.onNodeWithTag("item-row-device-001").assertIsDisplayed()
+            compose.onNodeWithTag("item-row-remote-001").assertIsDisplayed()
+            compose.onNodeWithText("Gamma").assertIsDisplayed()
+            compose.onNodeWithText("Alpha synced").assertIsDisplayed()
+            compose.onNodeWithContentDescription("Completed: Gamma").assertIsOff()
+            compose.onNodeWithContentDescription("Completed: Alpha synced").assertIsOn()
+            compose.onNodeWithText("Beta").assertDoesNotExist()
+        } finally {
+            firstDatabase.close()
+            secondDatabase.close()
+        }
+    }
+
+    private fun fixtureControl(method: String, path: String): JSONObject {
+        val request = Request.Builder().url(HttpItemRemote.FIXTURE_URL + path)
+            .method(method, if (method == "POST") byteArrayOf().toRequestBody() else null).build()
+        return OkHttpClient.Builder().retryOnConnectionFailure(false).build().newCall(request).execute().use {
+            assertEquals(200, it.code)
+            val body = requireNotNull(it.body).string()
+            android.util.Log.i("M03Fixture", "$method /$path ${it.code} $body")
+            JSONObject(body)
+        }
+    }
 }
diff --git a/app/src/main/AndroidManifest.xml b/app/src/main/AndroidManifest.xml
index 3fa0c3c..3874211 100644
--- a/app/src/main/AndroidManifest.xml
+++ b/app/src/main/AndroidManifest.xml
@@ -1,8 +1,10 @@
 <?xml version="1.0" encoding="utf-8"?>
 <manifest xmlns:android="http://schemas.android.com/apk/res/android">
+    <uses-permission android:name="android.permission.INTERNET" />
     <application
         android:allowBackup="false"
         android:label="Offline Item Tracker"
+        android:networkSecurityConfig="@xml/network_security_config"
         android:supportsRtl="true"
         android:theme="@android:style/Theme.Material.Light.NoActionBar">
         <activity android:name=".MainActivity" android:exported="true">
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/HttpItemRemote.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/HttpItemRemote.kt
new file mode 100644
index 0000000..53d245b
--- /dev/null
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/HttpItemRemote.kt
@@ -0,0 +1,69 @@
+package com.mobilesystemsevolution.kotlin
+
+import java.io.IOException
+import java.util.concurrent.TimeUnit
+import kotlinx.coroutines.Dispatchers
+import kotlinx.coroutines.withContext
+import okhttp3.HttpUrl.Companion.toHttpUrl
+import okhttp3.MediaType.Companion.toMediaType
+import okhttp3.OkHttpClient
+import okhttp3.Request
+import okhttp3.RequestBody.Companion.toRequestBody
+import org.json.JSONObject
+
+class HttpItemRemote(
+    baseUrl: String = FIXTURE_URL,
+    private val client: OkHttpClient = CLIENT,
+) : ItemRemote {
+    private val base = baseUrl.toHttpUrl()
+
+    override suspend fun items(): List<Item> {
+        val rows = request("GET", null, null, 200).getJSONArray("items")
+        return List(rows.length()) { rows.getJSONObject(it).toItem() }
+    }
+
+    override suspend fun create(item: Item): Item {
+        val body = JSONObject().put("id", item.id).put("title", item.title)
+            .put("completed", item.completed)
+        return request("POST", null, body, 201).getJSONObject("item").toItem()
+    }
+
+    override suspend fun patch(id: String, title: String?, completed: Boolean?): Item {
+        val body = JSONObject()
+        title?.let { body.put("title", it) }
+        completed?.let { body.put("completed", it) }
+        return request("PATCH", id, body, 200).getJSONObject("item").toItem()
+    }
+
+    override suspend fun delete(id: String): String =
+        request("DELETE", id, null, 200).getString("deletedId").also {
+            if (it != id) throw IOException("Unexpected deleted Item identity")
+        }
+
+    private suspend fun request(method: String, id: String?, body: JSONObject?, status: Int): JSONObject =
+        withContext(Dispatchers.IO) {
+            val url = base.newBuilder().addPathSegment("items")
+                .apply { if (id != null) addPathSegment(id) }.build()
+            val request = Request.Builder().url(url)
+                .method(method, body?.toString()?.toRequestBody("application/json".toMediaType()))
+                .build()
+            client.newCall(request).execute().use { response ->
+                if (response.code != status) throw IOException("Sync HTTP ${response.code}")
+                JSONObject(response.body?.string() ?: throw IOException("Empty sync response"))
+            }
+        }
+
+    private fun JSONObject.toItem() = Item(
+        getString("id"), getString("title"), getBoolean("completed"),
+        getLong("version"), getLong("updatedAt"),
+    )
+
+    companion object {
+        const val FIXTURE_URL = "http://10.0.2.2:18080/"
+        private val CLIENT = OkHttpClient.Builder()
+            .retryOnConnectionFailure(false)
+            .followRedirects(false)
+            .callTimeout(10, TimeUnit.SECONDS)
+            .build()
+    }
+}
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemDatabase.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemDatabase.kt
index 00c62fa..0a8f5ff 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemDatabase.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemDatabase.kt
@@ -9,6 +9,7 @@ import androidx.room.PrimaryKey
 import androidx.room.Query
 import androidx.room.Room
 import androidx.room.RoomDatabase
+import androidx.room.Transaction
 
 @Entity(tableName = "items")
 data class ItemEntity(
@@ -39,6 +40,15 @@ interface ItemDao {
 
     @Query("DELETE FROM items WHERE id = :id")
     suspend fun delete(id: String): Int
+
+    @Query("DELETE FROM items")
+    suspend fun deleteAll()
+
+    @Transaction
+    suspend fun replaceAll(items: List<ItemEntity>) {
+        deleteAll()
+        items.forEach { insert(it) }
+    }
 }
 
 @Database(entities = [ItemEntity::class], version = 1, exportSchema = true)
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemStore.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemStore.kt
index 25153c6..0c4ba57 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemStore.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemStore.kt
@@ -10,7 +10,7 @@ data class Item(
     val updatedAt: Long,
 )
 
-/** M02 reads committed local rows. Version remains zero without sync semantics. */
+/** UI reads committed local rows; local creates stay at version zero until synchronized. */
 class ItemStore(
     private val dao: ItemDao,
     private val nextId: () -> String = { UUID.randomUUID().toString() },
@@ -18,6 +18,10 @@ class ItemStore(
 ) {
     suspend fun items(): List<Item> = dao.readAll().map(ItemEntity::toDomain)
 
+    suspend fun replaceWithCanonical(items: List<Item>) {
+        dao.replaceAll(items.map(Item::toEntity))
+    }
+
     suspend fun create(title: String) {
         val validTitle = title.trim().also { require(it.isNotEmpty()) }
         dao.insert(Item(nextId(), validTitle, false, 0, now()).toEntity())
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemSync.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemSync.kt
new file mode 100644
index 0000000..ed7547f
--- /dev/null
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemSync.kt
@@ -0,0 +1,35 @@
+package com.mobilesystemsevolution.kotlin
+
+interface ItemRemote {
+    suspend fun items(): List<Item>
+    suspend fun create(item: Item): Item
+    suspend fun patch(id: String, title: String?, completed: Boolean?): Item
+    suspend fun delete(id: String): String
+}
+
+/** One explicit foreground call at a time; this comparison snapshot is not durable intent. */
+class ItemSync(private val store: ItemStore, private val remote: ItemRemote) {
+    private var lastSynced: Map<String, Item>? = null
+
+    suspend fun synchronize() {
+        val local = store.items().associateBy(Item::id)
+        val previous = lastSynced
+        for (item in local.values.sortedBy(Item::id)) {
+            val before = previous?.get(item.id)
+            if (before == null) {
+                // Preserve local M02 creations on an initial sync without adding a queue.
+                if (previous != null || item.version == 0L) remote.create(item)
+            } else {
+                val title = item.title.takeIf { it != before.title }
+                val completed = item.completed.takeIf { it != before.completed }
+                if (title != null || completed != null) remote.patch(item.id, title, completed)
+            }
+        }
+        previous?.keys?.filter { it !in local }?.sorted()?.forEach { remote.delete(it) }
+
+        val canonical = remote.items()
+        store.replaceWithCanonical(canonical)
+        // Read back committed Room state; neither the screen nor sync owns a second UI list.
+        lastSynced = store.items().associateBy(Item::id)
+    }
+}
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
index fc3492c..4e32736 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
@@ -41,7 +41,8 @@ class MainActivity : ComponentActivity() {
         super.onCreate(savedInstanceState)
         database = ItemDatabase.open(applicationContext)
         val store = ItemStore(database.items())
-        setContent { MaterialTheme { ItemScreen(store) } }
+        val sync = ItemSync(store, HttpItemRemote())
+        setContent { MaterialTheme { ItemScreen(store, sync) } }
     }
 
     override fun onDestroy() {
@@ -51,10 +52,12 @@ class MainActivity : ComponentActivity() {
 }
 
 @Composable
-internal fun ItemScreen(store: ItemStore) {
+internal fun ItemScreen(store: ItemStore, sync: ItemSync? = null) {
     var items by remember(store) { mutableStateOf<List<Item>?>(null) }
     var busy by remember(store) { mutableStateOf(true) }
     var storageError by remember(store) { mutableStateOf<String?>(null) }
+    var syncError by remember(store) { mutableStateOf<String?>(null) }
+    var syncing by remember(store) { mutableStateOf(false) }
     var title by remember { mutableStateOf("") }
     var editingId by remember { mutableStateOf<String?>(null) }
     val scope = rememberCoroutineScope()
@@ -66,19 +69,26 @@ internal fun ItemScreen(store: ItemStore) {
             action()
             items = store.items()
             storageError = null
+            syncError = null
             after()
         } catch (cancelled: CancellationException) {
             throw cancelled
         } catch (error: Exception) {
-            storageError = "Local storage error. Change not confirmed; please retry."
+            if (syncing) {
+                syncError = "Sync failed. Remote outcome unconfirmed."
+            } else {
+                storageError = "Local storage error. Change not confirmed; please retry."
+            }
         } finally {
             busy = false
+            syncing = false
         }
     }
 
-    fun change(action: suspend () -> Unit, after: () -> Unit = {}) {
+    fun change(action: suspend () -> Unit, after: () -> Unit = {}, isSync: Boolean = false) {
         if (busy) return
         busy = true
+        syncing = isSync
         scope.launch { accessStorage(action, after) }
     }
 
@@ -119,8 +129,19 @@ internal fun ItemScreen(store: ItemStore) {
                     focus.clearFocus()
                 }) { Text("Cancel") }
             }
+            if (sync != null) {
+                TextButton(
+                    enabled = !busy && items != null,
+                    onClick = { change(action = { sync.synchronize() }, isSync = true) },
+                ) { Text("Sync") }
+            }
         }
-        if (busy) Text(if (items == null) "Loading local items…" else "Saving locally…")
+        if (busy) Text(when {
+            syncing -> "Synchronizing…"
+            items == null -> "Loading local items…"
+            else -> "Saving locally…"
+        })
+        syncError?.let { Text(it, modifier = Modifier.testTag("sync-error")) }
         storageError?.let { message ->
             Text(message, modifier = Modifier.testTag("storage-error"))
             TextButton(enabled = !busy, onClick = { change(action = {}) }) { Text("Reload") }
diff --git a/app/src/main/res/xml/network_security_config.xml b/app/src/main/res/xml/network_security_config.xml
new file mode 100644
index 0000000..70d48a3
--- /dev/null
+++ b/app/src/main/res/xml/network_security_config.xml
@@ -0,0 +1,7 @@
+<?xml version="1.0" encoding="utf-8"?>
+<network-security-config>
+    <base-config cleartextTrafficPermitted="false" />
+    <domain-config cleartextTrafficPermitted="true">
+        <domain>10.0.2.2</domain>
+    </domain-config>
+</network-security-config>
diff --git a/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt b/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
index 79610ca..aa196dd 100644
--- a/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
+++ b/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
@@ -66,6 +66,81 @@ class ItemStoreTest {
         assertEquals(before, runBlocking { store.items() })
     }
 
+    @Test
+    fun frozenM03SyncPersistsCanonicalFieldsInTwoIndependentStores() = runBlocking {
+        val remote = FakeRemote()
+        val first = ItemStore(FakeItemDao(), nextId = { "device-001" }, now = { 1800000000000L })
+        val sync = ItemSync(first, remote)
+        assertTrue(first.items().isEmpty())
+        sync.synchronize()
+        assertEquals(remote.rows.values.toList(), first.items())
+        first.create("Gamma")
+        sync.synchronize()
+        first.rename("remote-001", "Alpha synced")
+        sync.synchronize()
+        first.setCompleted("remote-001", true)
+        sync.synchronize()
+        first.delete("remote-002")
+        sync.synchronize()
+
+        val expected = listOf(
+            Item("device-001", "Gamma", false, 1, 1700000100000L),
+            Item("remote-001", "Alpha synced", true, 3, 1700000102000L),
+        )
+        val second = ItemStore(FakeItemDao())
+        assertTrue(second.items().isEmpty())
+        ItemSync(second, remote).synchronize()
+        assertEquals(expected, first.items())
+        assertEquals(expected, second.items())
+        assertEquals(expected, remote.rows.values.sortedBy(Item::id))
+        assertEquals(1700000104000L, remote.time)
+        assertEquals(listOf("GET", "POST device-001", "GET", "PATCH remote-001 title=Alpha synced completed=null",
+            "GET", "PATCH remote-001 title=null completed=true", "GET", "DELETE remote-002", "GET", "GET"),
+            remote.calls)
+    }
+
+    @Test
+    fun firstExplicitSyncPreservesAnExistingVersionZeroCreation() = runBlocking {
+        val remote = FakeRemote()
+        val store = ItemStore(FakeItemDao(), nextId = { "device-001" }, now = { 1600000000000L })
+        store.create("Gamma")
+        ItemSync(store, remote).synchronize()
+        assertEquals(Item("device-001", "Gamma", false, 1, 1700000100000L), store.items().first())
+        assertEquals(3, store.items().size)
+        assertEquals(listOf("POST device-001", "GET"), remote.calls)
+    }
+
+    private class FakeRemote : ItemRemote {
+        val rows = linkedMapOf(
+            "remote-001" to Item("remote-001", "Alpha", false, 1, 1700000000000L),
+            "remote-002" to Item("remote-002", "Beta", false, 1, 1700000000000L),
+        )
+        val calls = mutableListOf<String>()
+        var time = 1700000100000L
+        private fun tick() = time.also { time += 1000 }
+        override suspend fun items(): List<Item> {
+            calls += "GET"
+            return rows.values.sortedBy(Item::id)
+        }
+        override suspend fun create(item: Item): Item {
+            calls += "POST ${item.id}"
+            check(item.id !in rows)
+            return item.copy(version = 1, updatedAt = tick()).also { rows[item.id] = it }
+        }
+        override suspend fun patch(id: String, title: String?, completed: Boolean?): Item {
+            calls += "PATCH $id title=$title completed=$completed"
+            val item = rows.getValue(id)
+            return item.copy(title = title ?: item.title, completed = completed ?: item.completed,
+                version = item.version + 1, updatedAt = tick()).also { rows[id] = it }
+        }
+        override suspend fun delete(id: String): String {
+            calls += "DELETE $id"
+            check(rows.remove(id) != null)
+            tick()
+            return id
+        }
+    }
+
     private class FakeItemDao : ItemDao {
         private val rows = linkedMapOf<String, ItemEntity>()
 
@@ -89,5 +164,7 @@ class ItemStoreTest {
         }
 
         override suspend fun delete(id: String): Int = if (rows.remove(id) == null) 0 else 1
+
+        override suspend fun deleteAll() { rows.clear() }
     }
 }
diff --git a/fixture/server.py b/fixture/server.py
new file mode 100644
index 0000000..75d05ce
--- /dev/null
+++ b/fixture/server.py
@@ -0,0 +1,134 @@
+#!/usr/bin/env python3
+"""M03's local, in-memory HTTP fixture. No delay, injected failures, or external services."""
+
+import argparse
+from http.server import BaseHTTPRequestHandler, HTTPServer
+import json
+from urllib.parse import unquote
+
+
+class Fixture(HTTPServer):
+    def __init__(self, address):
+        super().__init__(address, Handler)
+        self.reset()
+
+    def reset(self):
+        self.items = {
+            "remote-001": dict(id="remote-001", title="Alpha", completed=False,
+                               version=1, updatedAt=1700000000000),
+            "remote-002": dict(id="remote-002", title="Beta", completed=False,
+                               version=1, updatedAt=1700000000000),
+        }
+        self.next_timestamp = 1700000100000
+
+    def tick(self):
+        timestamp = self.next_timestamp
+        self.next_timestamp += 1000
+        return timestamp
+
+    def sorted_items(self):
+        return [self.items[key] for key in sorted(self.items)]
+
+
+class Handler(BaseHTTPRequestHandler):
+    def respond(self, status, payload):
+        body = json.dumps(payload, separators=(",", ":")).encode()
+        self.send_response(status)
+        self.send_header("Content-Type", "application/json")
+        self.send_header("Content-Length", str(len(body)))
+        self.end_headers()
+        self.wfile.write(body)
+        print(json.dumps({"method": self.command, "path": self.path, "status": status,
+                          "response": payload}), flush=True)
+
+    def log_message(self, *_args):
+        pass  # Each response has one structured evidence line instead.
+
+    def payload(self):
+        result = json.loads(self.rfile.read(int(self.headers.get("Content-Length", "0"))))
+        if not isinstance(result, dict):
+            raise ValueError("Expected an object")
+        return result
+
+    @staticmethod
+    def valid_fields(payload):
+        if "title" in payload and (not isinstance(payload["title"], str) or not payload["title"].strip()):
+            raise ValueError("Expected a nonblank title")
+        if "completed" in payload and type(payload["completed"]) is not bool:
+            raise ValueError("Expected a Boolean")
+
+    def do_GET(self):
+        if self.path == "/items":
+            self.respond(200, {"items": self.server.sorted_items()})
+        elif self.path == "/__state":
+            self.respond(200, {"items": self.server.sorted_items(),
+                               "nextTimestamp": self.server.next_timestamp})
+        else:
+            self.respond(404, {"error": "not found"})
+
+    def do_POST(self):
+        if self.path == "/__reset":
+            self.server.reset()
+            self.respond(200, {"items": self.server.sorted_items(),
+                               "nextTimestamp": self.server.next_timestamp})
+            return
+        if self.path != "/items":
+            self.respond(404, {"error": "not found"})
+            return
+        try:
+            data = self.payload()
+            if set(data) != {"id", "title", "completed"}:
+                raise ValueError("Expected id, title, completed")
+            if not isinstance(data["id"], str) or not data["id"] or data["id"] in self.server.items:
+                raise ValueError("Expected a new nonempty id")
+            self.valid_fields(data)
+        except (ValueError, TypeError) as error:
+            self.respond(400, {"error": str(error)})
+            return
+        item = dict(data, version=1, updatedAt=self.server.tick())
+        self.server.items[item["id"]] = item
+        self.respond(201, {"item": item})
+
+    def item_id(self):
+        if not self.path.startswith("/items/"):
+            return None
+        item_id = unquote(self.path[len("/items/"):])
+        return item_id if item_id in self.server.items else None
+
+    def do_PATCH(self):
+        item_id = self.item_id()
+        if item_id is None:
+            self.respond(404, {"error": "not found"})
+            return
+        try:
+            data = self.payload()
+            if not data or not set(data) <= {"title", "completed"}:
+                raise ValueError("Expected changed title/completed")
+            self.valid_fields(data)
+        except (ValueError, TypeError) as error:
+            self.respond(400, {"error": str(error)})
+            return
+        item = self.server.items[item_id]
+        item.update(data, version=item["version"] + 1, updatedAt=self.server.tick())
+        self.respond(200, {"item": item})
+
+    def do_DELETE(self):
+        item_id = self.item_id()
+        if item_id is None:
+            self.respond(404, {"error": "not found"})
+            return
+        del self.server.items[item_id]
+        self.server.tick()
+        self.respond(200, {"deletedId": item_id})
+
+
+if __name__ == "__main__":
+    parser = argparse.ArgumentParser(description=__doc__)
+    parser.add_argument("--port", type=int, default=18080)
+    args = parser.parse_args()
+    with Fixture(("127.0.0.1", args.port)) as fixture:
+        print(f"M03 fixture listening on http://127.0.0.1:{args.port}; delay=0; failures=none", flush=True)
+        try:
+            fixture.serve_forever()
+        except KeyboardInterrupt:
+            pass
diff --git a/fixture/test_server.py b/fixture/test_server.py
new file mode 100644
index 0000000..40e7df3
--- /dev/null
+++ b/fixture/test_server.py
@@ -0,0 +1,55 @@
+import json
+from threading import Thread
+import unittest
+from urllib.request import Request, urlopen
+
+from server import Fixture
+
+
+class FixtureContractTest(unittest.TestCase):
+    def test_frozen_m03_exact_http_contract(self):
+        with Fixture(("127.0.0.1", 0)) as server:
+            thread = Thread(target=server.serve_forever, daemon=True)
+            thread.start()
+            try:
+                base = f"http://127.0.0.1:{server.server_port}"
+
+                def request(method, path, payload=None):
+                    data = None if payload is None else json.dumps(payload).encode()
+                    with urlopen(Request(base + path, data=data, method=method,
+                                         headers={"Content-Type": "application/json"}), timeout=5) as response:
+                        return response.status, json.load(response)
+
+                seeds = [
+                    dict(id="remote-001", title="Alpha", completed=False, version=1, updatedAt=1700000000000),
+                    dict(id="remote-002", title="Beta", completed=False, version=1, updatedAt=1700000000000),
+                ]
+                gamma = dict(id="device-001", title="Gamma", completed=False, version=1, updatedAt=1700000100000)
+                renamed = dict(id="remote-001", title="Alpha synced", completed=False,
+                               version=2, updatedAt=1700000101000)
+                completed = dict(renamed, completed=True, version=3, updatedAt=1700000102000)
+                self.assertEqual((200, {"items": seeds}), request("GET", "/items"))
+                self.assertEqual((201, {"item": gamma}), request("POST", "/items", {
+                    "id": "device-001", "title": "Gamma", "completed": False,
+                }))
+                self.assertEqual((200, {"items": [gamma, *seeds]}), request("GET", "/items"))
+                self.assertEqual((200, {"item": renamed}), request("PATCH", "/items/remote-001", {
+                    "title": "Alpha synced",
+                }))
+                self.assertEqual((200, {"items": [gamma, renamed, seeds[1]]}), request("GET", "/items"))
+                self.assertEqual((200, {"item": completed}), request("PATCH", "/items/remote-001", {
+                    "completed": True,
+                }))
+                self.assertEqual((200, {"items": [gamma, completed, seeds[1]]}), request("GET", "/items"))
+                self.assertEqual((200, {"deletedId": "remote-002"}), request("DELETE", "/items/remote-002"))
+                final = {"items": [gamma, completed]}
+                self.assertEqual((200, final), request("GET", "/items"))
+                self.assertEqual((200, final), request("GET", "/items"))  # Fresh second reader.
+                self.assertEqual((200, dict(final, nextTimestamp=1700000104000)), request("GET", "/__state"))
+            finally:
+                server.shutdown()
+                thread.join(timeout=5)
+
+
+if __name__ == "__main__":
+    unittest.main()
diff --git a/verification/M03.md b/verification/M03.md
new file mode 100644
index 0000000..2189d2c
--- /dev/null
+++ b/verification/M03.md
@@ -0,0 +1,171 @@
+# M03 verification — attempt 1
+
+**Handoff status: Android acceptance failed; candidate is not verified complete.**
+
+- Thread: M03; branch: `track/android-kotlin`
+- START: `8a607ee5a98033c000eaf22ab3e70e8c54d992ab` (verified M02 END)
+- Spec revision: `ed7baa246ee947c6778e0f84751c9b91cec7abfe`
+- Frozen shared M03 scenario SHA-256: `46589027d2fd585c5c7cc1ca86f409f0ec90c5eaba459c16f4843363cf4cd1e9`
+- Flow: new shared-canonical-state constraint → Reproduce → Fix → Verify → Commit → Report
+- Device: unchanged API 34 / Pixel 6 / ARM64 Google APIs, `emulator-5554`
+- Raw evidence root: `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/android-kotlin/M03/`
+
+## Frozen inputs and scope
+
+Remote starts with remote-001/Alpha/false/1/1700000000000 and
+remote-002/Beta/false/1/1700000000000. First local database starts empty. Next mutation
+timestamp is 1700000100000 and advances by 1000 ms, including successful deletion.
+Fixture port is 18080; emulator host is 10.0.2.2. Delay is zero; no failure is injected.
+The first instance syncs, creates device-001/Gamma/false and syncs, renames remote-001 to
+Alpha synced and syncs, completes it and syncs, deletes remote-002 and syncs. A fresh second
+Room database then syncs. Both databases and remote must contain exactly:
+
+| id | title | completed | version | updatedAt |
+| --- | --- | --- | --- | --- |
+| device-001 | Gamma | false | 1 | 1700000100000 |
+| remote-001 | Alpha synced | true | 3 | 1700000102000 |
+
+remote-002 is absent and fixture nextTimestamp is 1700000104000. The exact HTTP status
+codes and response object shapes are checked by `fixture/test_server.py`; Android uses
+the real OkHttp/HTTP/Room path, not that host fake. The deterministic client-side clock
+in the sync tests is 1800000000000, intentionally distinct from canonical timestamps.
+
+Supporting cases were recorded in `supporting-cases.json` before the first fixed test:
+an existing version-zero device-001/Gamma creation at 1600000000000 survives first explicit
+sync and receives canonical version 1/time 1700000100000; replacing Room with two identical
+remote-001 primary keys aborts the whole transaction and retains the M02 five-field fixture;
+the exact M03 canonical table survives external process replacement. No acceptance inputs
+were adjusted after results. This is not a performance scenario; no matrix, soak or timing search.
+
+## Reproduce — unchanged M02 production
+
+Only `ItemDatabaseTest#isolatedM02DatabasesCannotShareChangesWithoutSync` was added before
+reproduction. It opens two different Room files, writes device-001/Gamma/false/version0/
+1700000100000 to the first, and confirms the second stays empty even after closing/reopening.
+Actual Android logcat records `first=[Item(id=device-001, title=Gamma, completed=false,
+version=0, updatedAt=1700000100000)]; second=[]`. The one filtered test passed by observing
+the expected M02 limitation, not by claiming synchronization existed.
+
+`baseline-production.diff` is empty; `reproduce-only.diff`, source hashes, unchanged APK,
+XML and per-test logcat were captured before production edits. Baseline APK SHA-256 is
+`ca438938d6d8dd3761fe2d35fbe21e0a63a5b87c607c2d3a020d32cdae58855e`.
+The baseline XML timestamp is 2026-08-28T00:05:17 UTC, one passed test, no failures/errors/skips.
+The first explicit device lease was returned immediately after capture.
+
+## Reproducible commands and invocation ledger
+
+Run from this track root. The fixed environment for every Gradle invocation was:
+
+```sh
+export JAVA_HOME=/opt/homebrew/opt/openjdk@17/libexec/openjdk.jdk/Contents/Home
+export ANDROID_HOME=/opt/homebrew/share/android-commandlinetools
+export ANDROID_SDK_ROOT=/opt/homebrew/share/android-commandlinetools
+export GRADLE_USER_HOME=/private/tmp/mobile-systems-evolution-ed7baa2/gradle-kotlin
+```
+
+Gradle/dependency/native execution and local HTTP binding used explicit sandbox escalation.
+Every device invocation requires an exclusive lease from main; no emulator settings change.
+
+| Invocation | Command (with environment above) | Exit | Actual result / raw file |
+| --- | --- | --- | --- |
+| Reproduction build 1 | `./gradlew --no-daemon --console=plain :app:assembleDebug :app:assembleDebugAndroidTest` | 0 | APKs built; no tests; 12s; `reproduce-build-01.log` |
+| Reproduction Android 1 | `ANDROID_SERIAL=emulator-5554 ./gradlew --no-daemon --console=plain :app:connectedDebugAndroidTest '-Pandroid.testInstrumentationRunnerArguments.class=com.mobilesystemsevolution.kotlin.ItemDatabaseTest#isolatedM02DatabasesCannotShareChangesWithoutSync'` | 0 | 1 test passed; 17s; `reproduce-android-01.log`, `reproduce-android-01-results/` |
+| Fixture contract 1 | `python3 -m unittest discover -s fixture -v` | 0 | 1 real-loopback HTTP contract test passed; `fixture-contract-01.log` |
+| Fixed host/build 1 | `./gradlew --no-daemon --console=plain :app:testDebugUnitTest :app:assembleDebug :app:assembleDebugAndroidTest` | 0 | 5 host tests passed, both APKs built; 42s; `host-build-01.log`, `host-01-TEST-*.xml` |
+| Android suite 1 | `ANDROID_SERIAL=emulator-5554 ./gradlew --no-daemon --console=plain :app:connectedDebugAndroidTest` | 1 | 8 executed: 7 passed, M03 create-step wait failed; 49s; `android-suite-01.log`, `android-suite-01-results/` |
+| Fixed host/build 2 | `./gradlew --no-daemon --console=plain :app:testDebugUnitTest :app:assembleDebug :app:assembleDebugAndroidTest` | 0 | Test-only fix compiled; host task UP-TO-DATE, 0 newly executed tests; 22s; `host-build-02.log` |
+| Android suite 2 | `ANDROID_SERIAL=emulator-5554 ./gradlew --no-daemon --console=plain :app:connectedDebugAndroidTest` | 1 | 8 executed: 7 passed, same M03 create-step wait failed; 52s; `android-suite-02.log`, `android-suite-02-results/` |
+| Test build 3 | `./gradlew --no-daemon --console=plain :app:assembleDebugAndroidTest` | 1 | Test diagnostic helper did not compile: Espresso unavailable on compile classpath; no tests; 14s; `test-build-03.log` |
+| Test build 4 | `./gradlew --no-daemon --console=plain :app:assembleDebugAndroidTest` | 0 | Platform keyboard/Room diagnostic helper compiled; no tests; 34s; `test-build-04.log` |
+| Filtered diagnostic 1 | `ANDROID_SERIAL=emulator-5554 ./gradlew --no-daemon --console=plain :app:connectedDebugAndroidTest '-Pandroid.testInstrumentationRunnerArguments.class=com.mobilesystemsevolution.kotlin.ItemUiTest#frozenM03SequenceUsesHttpRoomAndTwoIndependentInstances'` | 1 | 1 executed, failed; 1m20s; `android-filtered-diagnostic-01.log`, `android-filtered-diagnostic-01-results/` |
+
+For actual Android acceptance, keep this server running in a separate terminal:
+
+```sh
+python3 -u fixture/server.py --port 18080
+```
+
+The exact Android commands are below. The final restart command is prepared but was **not
+run**: the filtered scenario failed, so the required canonical seed was not established.
+Only run the restart harness after a successful filtered test, with the device lease.
+
+```sh
+ANDROID_SERIAL=emulator-5554 ./gradlew --no-daemon --console=plain :app:connectedDebugAndroidTest
+ANDROID_SERIAL=emulator-5554 ./gradlew --no-daemon --console=plain :app:connectedDebugAndroidTest \
+  '-Pandroid.testInstrumentationRunnerArguments.class=com.mobilesystemsevolution.kotlin.ItemUiTest#frozenM03SequenceUsesHttpRoomAndTwoIndependentInstances'
+python3 verification/canonical_restart.py --apk app/build/outputs/apk/debug/app-debug.apk \
+  --adb /opt/homebrew/share/android-commandlinetools/platform-tools/adb \
+  --output /private/tmp/mobile-systems-evolution-ed7baa2/evidence/android-kotlin/M03/canonical-restart-01
+```
+
+On success, the filtered test seeds app `items.db` with the exact table before the separate
+restart harness; other UI regressions clear app data only at their own scenario start. The
+harness performs no install, clear, instrumentation or storage mutation. It launches normal
+MainActivity, captures UI/SQLite and PID, force-stops externally, confirms no PID, relaunches
+with a different PID, and checks exact fields and deleted-item absence. The M02 capture helper
+is unchanged and keeps raw database/WAL files separate from disposable host SQLite reads.
+
+## Verification results
+
+Actual passing coverage: one baseline reproduction, five host tests, one real HTTP fixture
+contract test, and all seven non-M03 Android regression tests in each of the two suites.
+The seven include the unchanged original UI CRUD, visible storage-error/draft retention,
+database reopen/mapping, unknown-schema preservation, canonical transaction rollback, and
+the isolated-database limitation. Neither full Android suite passed as a whole.
+
+All three Android M03 executions passed the initial HTTP pull and stored/displayed the two
+seed Items. The last probe also proved Add committed the exact local row
+`device-001/Gamma/false/0/1800000000000`; before the next click Sync was asserted visible and
+enabled, at the same bounds as its first successful click. Nevertheless, the fixture saw
+no POST and the create's canonical Room assertion timed out. Root cause remains unresolved;
+the earlier draft/IME explanations were not established. The attempted post-timeout tree
+diagnostic did not emit from `catch (Exception)`, while pre-click/committed-row logs did.
+Second-database convergence and the exact final Android table were **not reached**.
+
+Main limited the next probe to one filtered run and required a clean failing handoff if it
+failed. That limit was honored: no additional suite, reseed, runtime iteration or canonical
+process restart was run. The final device lease was released and the owned fixture was
+stopped with Ctrl-C, exit 0. Restart it with the command above; `/__reset` restores the fixed
+seed and clock. `handoff-verification.json` and `commands.txt` summarize the full ledger.
+
+Preserved fixed app APK SHA-256 (unchanged across test-only diagnostics):
+`11c5d0f7603927cd7b258a2d4558f7960f73164da168b45e292b0d4bd8d0bdb0`.
+Final diagnostic test APK SHA-256:
+`6d57cf78d2a7004c3086f3236f494d30640b70fe1ff4010964b1f61ab66feb70`.
+Final test-source snapshot SHA-256:
+`8670e76906014ddc5bcd3f20f43ad988862709db1b91c7fe380d669f7397db4c`.
+The unrun canonical harness SHA-256 is
+`264d434675d82fc35cd2c938050ff91766f1e1601c6e54ee35ce730db9c51fa4`;
+its unchanged M02 capture helper is
+`8d0abf7be76a4b408c3ccbc599eca5f54adcbb11e214ce30c9b2879a806c8daf`.
+
+## Boundary and failures
+
+Room schema v1 and its five columns are unchanged. Local CRUD remains available without
+starting HTTP. Only manual Sync uploads the transient comparison diff and atomically saves
+the full canonical snapshot before UI readback. Initial version-zero creates are handled
+without a new metadata column. Ordinary startup has a local read path; canonical restart
+has not been verified in M03.
+Restart-before-upload (especially rename/delete), partial remote success, idempotency,
+conflicts and background execution are deliberately not guaranteed here. No durable intent,
+queue, offline-upload worker, retry framework, conflict framework, scheduler or push was added.
+
+Android suites 1 and 2 both passed the initial seed sync but timed out waiting for the
+canonical Gamma create; the fixture received no POST. The first diagnosis was that
+`awaitText("Gamma")` could match the draft before save and the Sync click could be disabled.
+Waiting for enabled Sync alone did not resolve suite 2, so this was not accepted as a full
+explanation. The device lease was returned after each failure and all XML/logcat/fixture
+logs were preserved. The next test-only change adds committed-row/semantics diagnostics,
+explicit keyboard close, and scroll-to-visible enabled-button interaction. No timeout,
+fixture, response, mutation sequence, expected state or production file changed.
+It also checks exact local Room rows immediately after Add to distinguish an uncommitted
+draft from a failed synchronization. A test helper import failed to compile in build 3;
+the helper was changed to existing Android InputMethodManager/WindowInsets APIs without
+adding a dependency or arbitrary sleeps. The IME explanation remains provisional.
+
+Kapt emitted its existing unused-option warnings for test source sets. Documentation lookup returned one
+404 and one unavailable old OkHttp documentation URL; the pinned official source was then
+read successfully. A no-match `rg` for absent AGENTS.md returned 1 during inspection; it
+was not a test. There were three failed Android M03 executions and one failed test build;
+none were omitted or relabeled as passes. Main owns bounded repair dispatch, independent
+verification and progress tagging. No M03 progress tag is created by this candidate.
diff --git a/verification/canonical_restart.py b/verification/canonical_restart.py
new file mode 100644
index 0000000..5246774
--- /dev/null
+++ b/verification/canonical_restart.py
@@ -0,0 +1,73 @@
+#!/usr/bin/env python3
+"""M03: verify an already synchronized items.db across external Android process death.
+
+First run ItemUiTest#frozenM03SequenceUsesHttpRoomAndTwoIndependentInstances on the leased
+emulator. This harness does not install, clear, seed, or mutate any app data. It reuses M02's
+read-only database/WAL capture and UI helpers, then requires the exact M03 canonical table.
+"""
+
+import argparse
+import hashlib
+import json
+import os
+from pathlib import Path
+
+from process_restart import PACKAGE, Scenario
+
+
+EXPECTED = [
+    dict(id="device-001", title="Gamma", completed=0, version=1, updatedAt=1700000100000),
+    dict(id="remote-001", title="Alpha synced", completed=1, version=3, updatedAt=1700000102000),
+]
+
+
+class CanonicalScenario(Scenario):
+    def final_ui(self):
+        tree = self.wait_text("Items (2)")
+        assert self.matching(tree, text="Gamma")
+        assert self.matching(tree, text="Alpha synced")
+        assert self.matching(tree, **{"content-desc": "Completed: Gamma", "checked": "false"})
+        assert self.matching(tree, **{"content-desc": "Completed: Alpha synced", "checked": "true"})
+        assert not self.matching(tree, text="Beta")
+
+    def run(self):
+        self.result.update(scenario="M03", captureHelperSha256=self.result["harnessSha256"],
+                           harnessSha256=hashlib.sha256(Path(__file__).read_bytes()).hexdigest())
+        self.adb("shell", "am", "start", "-W", "-n", f"{PACKAGE}/.MainActivity")
+        self.final_ui()
+        before = self.snapshot("before-stop")
+        assert sorted(before, key=lambda row: row["id"]) == EXPECTED, before
+        (self.output / "before-stop.png").write_bytes(self.adb("exec-out", "screencap", "-p", binary=True))
+        old_pid = self.adb("shell", "pidof", PACKAGE)
+        assert old_pid.isdigit(), old_pid
+        self.adb("shell", "am", "force-stop", PACKAGE)
+        assert not self.adb("shell", "pidof", PACKAGE, allow_failure=True)
+        self.adb("shell", "am", "start", "-W", "-n", f"{PACKAGE}/.MainActivity")
+        new_pid = self.adb("shell", "pidof", PACKAGE)
+        assert new_pid.isdigit() and new_pid != old_pid, (old_pid, new_pid)
+        self.final_ui()
+        after = self.snapshot("after-relaunch")
+        assert sorted(after, key=lambda row: row["id"]) == EXPECTED, after
+        assert before == after
+        (self.output / "after-relaunch.png").write_bytes(self.adb("exec-out", "screencap", "-p", binary=True))
+        self.result.update(status="PASS", beforePid=int(old_pid), afterPid=int(new_pid),
+                           beforeStop=before, afterRelaunch=after)
+
+
+if __name__ == "__main__":
+    parser = argparse.ArgumentParser(description=__doc__)
+    parser.add_argument("--apk", type=Path, required=True, help="Built APK identity only; never installed here")
+    parser.add_argument("--output", type=Path, required=True)
+    parser.add_argument("--adb", type=Path, default=Path(os.environ.get("ANDROID_HOME", "")) / "platform-tools/adb")
+    args = parser.parse_args()
+    args.expect = "canonical"
+    scenario = CanonicalScenario(args)
+    try:
+        scenario.run()
+    except Exception as error:
+        scenario.result.update(status="FAIL", error=repr(error))
+        raise
+    finally:
+        scenario.result["adbCommands"] = scenario.command_count
+        (scenario.output / "result.json").write_text(json.dumps(scenario.result, indent=2) + "\n")
+        print(json.dumps(scenario.result, indent=2), flush=True)


