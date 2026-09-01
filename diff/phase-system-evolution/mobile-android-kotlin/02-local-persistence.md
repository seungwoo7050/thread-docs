# M02 — Local Persistence

## `feat(android): persist Items across app process restart`

diff --git a/TRACK.md b/TRACK.md
index 4714f61..67bfbbe 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -5,21 +5,44 @@ The fixed specification revision is recorded in `SPEC_REVISION` and every commit
 
 ## M01 boundary
 
-One Compose screen creates Items, edits their titles, toggles completion, deletes Items,
-and renders the remaining list. `MemoryItemStore` contains only immutable list snapshots
-held in process memory. There is no disk storage, network access, synchronization,
-background task, or global state framework. `version` is always zero and carries no
-synchronization semantics. Activity recreation and process restart may clear everything;
-this Thread makes no persistence or restoration guarantee.
+The M01 commit established one Compose screen for Item creation, title editing, completion,
+deletion, and list rendering. Its `MemoryItemStore` held only process memory and made no
+persistence guarantee. M02 reproduced the loss after an external Android force-stop before
+replacing that storage implementation. The original fixed CRUD sequence remains a regression.
 
 The list uses the same `toRows()` mapping checked by the domain test. Controls have text
 labels/content descriptions; row test tags preserve Item identity through edits.
 
+## M02 boundary
+
+`ItemStore` now reads and changes local Room rows through `ItemDao`. Each mutation is one
+SQLite statement; no optimistic Item list is published. The screen waits for the write and
+subsequent committed read before clearing a submitted draft or showing the updated list.
+Controls are disabled while storage is busy. Storage errors remain visible with the draft
+and last confirmed rows retained; Reload attempts another read. An initial read failure
+does not present an empty database as a successful load.
+
+`ItemDatabase` creates `items.db` schema version 1 on the first query and validates/reopens
+that same file thereafter. The Activity owns the database handle and closes it on destroy.
+The exported schema is checked into `app/schemas/`. Only id, title, completed, version, and
+updatedAt are stored. Explicit mappings preserve all five fields; `version` still has no
+sync semantics. Insertion order uses SQLite rowid without another metadata field.
+
+There is no destructive-migration fallback: an unsupported schema fails without resetting
+the stored Items. This follows [Room's schema and migration guidance](https://developer.android.com/training/data-storage/room/migrating-db-versions).
+Room 2.6.1 is pinned to retain the existing Kotlin 1.9 toolchain; [Room 2.7 requires Kotlin 2.0](https://developer.android.com/jetpack/androidx/releases/room#2.7.0).
+Suspend DAO methods run storage work off the main thread using [Room's coroutine support](https://developer.android.com/training/data-storage/room/async-queries).
+
+No network, remote fixture, sync intent/queue, conflict policy, background work, push,
+global state framework, or future schema columns are included. Draft/selection restoration
+is not claimed. Persisted Items survive a full app-process replacement, not just a new Activity.
+
 ## Pinned build
 
 - JDK 17, Gradle 8.7 (wrapper distribution SHA-256 pinned)
 - Android Gradle Plugin 8.6.1, Kotlin 1.9.24, Compose compiler 1.5.14
 - Compose BOM 2024.06.00, Activity Compose 1.9.0
+- Room runtime/KT extensions/compiler 2.6.1, Kotlin kapt 1.9.24
 - Compile SDK 35, target SDK 34, minimum SDK 26
 - Required verification device: API 34, Pixel 6, ARM64 Google APIs image, serial `emulator-5554`
 
@@ -36,9 +59,24 @@ ANDROID_SERIAL=emulator-5554 ./gradlew --no-daemon :app:connectedDebugAndroidTes
 ```
 
 The second command installs the app and instrumentation APK and runs the actual Compose
-UI sequence. It requires the shared emulator lease. The application package is
+UI sequence plus persistence/error tests. It requires the shared emulator lease. The application package is
 `com.mobilesystemsevolution.kotlin`; the test package adds `.test`.
 
+The separate process-restart harness also requires the lease. Supply a new evidence directory:
+
+```sh
+python3 verification/process_restart.py --expect durable \
+  --apk app/build/outputs/apk/debug/app-debug.apk \
+  --adb "$ANDROID_HOME/platform-tools/adb" --output /tmp/kotlin-M02-restart-evidence
+```
+
+It starts with one explicit app-data clear, performs the exact UI sequence, captures the
+committed SQLite state, externally force-stops and relaunches, checks different PIDs and
+the restored UI, and compares all five fields and deleted-Item absence. Nothing clears or
+reinstalls app data across the process boundary. Raw database/WAL copies are read on the
+host without changing app storage. `--expect memory-loss` is solely for the preserved M01
+baseline APK; `verification/M02.md` records both phases and every test invocation.
+
 ## Frozen M01 sequence
 
 Start empty; create Alpha; create Beta; rename Alpha to Alpha edited; mark it completed,
diff --git a/app/build.gradle.kts b/app/build.gradle.kts
index e8aaf67..c06bb7c 100644
--- a/app/build.gradle.kts
+++ b/app/build.gradle.kts
@@ -1,6 +1,7 @@
 plugins {
     id("com.android.application")
     id("org.jetbrains.kotlin.android")
+    id("org.jetbrains.kotlin.kapt")
 }
 
 android {
@@ -26,11 +27,18 @@ android {
     packaging.resources.excludes += "/META-INF/{AL2.0,LGPL2.1}"
 }
 
+kapt {
+    arguments { arg("room.schemaLocation", "$projectDir/schemas") }
+}
+
 dependencies {
     implementation(platform("androidx.compose:compose-bom:2024.06.00"))
     implementation("androidx.activity:activity-compose:1.9.0")
     implementation("androidx.compose.material3:material3")
     implementation("androidx.compose.ui:ui")
+    implementation("androidx.room:room-runtime:2.6.1")
+    implementation("androidx.room:room-ktx:2.6.1")
+    kapt("androidx.room:room-compiler:2.6.1")
 
     testImplementation("junit:junit:4.13.2")
     androidTestImplementation(platform("androidx.compose:compose-bom:2024.06.00"))
diff --git a/app/schemas/com.mobilesystemsevolution.kotlin.ItemDatabase/1.json b/app/schemas/com.mobilesystemsevolution.kotlin.ItemDatabase/1.json
new file mode 100644
index 0000000..9123de4
--- /dev/null
+++ b/app/schemas/com.mobilesystemsevolution.kotlin.ItemDatabase/1.json
@@ -0,0 +1,58 @@
+{
+  "formatVersion": 1,
+  "database": {
+    "version": 1,
+    "identityHash": "8d9e5e27ab3b33aa75b77b7bc50e06a3",
+    "entities": [
+      {
+        "tableName": "items",
+        "createSql": "CREATE TABLE IF NOT EXISTS `${TABLE_NAME}` (`id` TEXT NOT NULL, `title` TEXT NOT NULL, `completed` INTEGER NOT NULL, `version` INTEGER NOT NULL, `updatedAt` INTEGER NOT NULL, PRIMARY KEY(`id`))",
+        "fields": [
+          {
+            "fieldPath": "id",
+            "columnName": "id",
+            "affinity": "TEXT",
+            "notNull": true
+          },
+          {
+            "fieldPath": "title",
+            "columnName": "title",
+            "affinity": "TEXT",
+            "notNull": true
+          },
+          {
+            "fieldPath": "completed",
+            "columnName": "completed",
+            "affinity": "INTEGER",
+            "notNull": true
+          },
+          {
+            "fieldPath": "version",
+            "columnName": "version",
+            "affinity": "INTEGER",
+            "notNull": true
+          },
+          {
+            "fieldPath": "updatedAt",
+            "columnName": "updatedAt",
+            "affinity": "INTEGER",
+            "notNull": true
+          }
+        ],
+        "primaryKey": {
+          "autoGenerate": false,
+          "columnNames": [
+            "id"
+          ]
+        },
+        "indices": [],
+        "foreignKeys": []
+      }
+    ],
+    "views": [],
+    "setupQueries": [
+      "CREATE TABLE IF NOT EXISTS room_master_table (id INTEGER PRIMARY KEY,identity_hash TEXT)",
+      "INSERT OR REPLACE INTO room_master_table (id,identity_hash) VALUES(42, '8d9e5e27ab3b33aa75b77b7bc50e06a3')"
+    ]
+  }
+}
\ No newline at end of file
diff --git a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemDatabaseTest.kt b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemDatabaseTest.kt
new file mode 100644
index 0000000..6ddf8e2
--- /dev/null
+++ b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemDatabaseTest.kt
@@ -0,0 +1,90 @@
+package com.mobilesystemsevolution.kotlin
+
+import android.database.sqlite.SQLiteDatabase
+import androidx.test.ext.junit.runners.AndroidJUnit4
+import androidx.test.platform.app.InstrumentationRegistry
+import kotlinx.coroutines.runBlocking
+import org.junit.Assert.assertEquals
+import org.junit.Assert.assertThrows
+import org.junit.Assert.assertTrue
+import org.junit.Before
+import org.junit.Test
+import org.junit.runner.RunWith
+
+@RunWith(AndroidJUnit4::class)
+class ItemDatabaseTest {
+    private val context = InstrumentationRegistry.getInstrumentation().targetContext
+    private val name = "m02-room-test.db"
+    private val fixture = Item("item-001", "Alpha edited", true, 0, 1700000006000L)
+
+    @Before
+    fun emptyTestDatabase() {
+        context.deleteDatabase(name)
+    }
+
+    private suspend fun <T> withDatabase(block: suspend (ItemDatabase) -> T): T {
+        val database = ItemDatabase.open(context, name)
+        try {
+            return block(database)
+        } finally {
+            database.close()
+        }
+    }
+
+    @Test
+    fun frozenCrudSurvivesDatabaseReopen() = runBlocking {
+        val expected = listOf(Item("item-001", "Alpha edited", true, 0, 1700000005000L))
+        withDatabase { database ->
+            val ids = listOf("item-001", "item-002").iterator()
+            var time = 1700000000000L
+            val store = ItemStore(database.items(), nextId = { ids.next() }, now = { time.also { time += 1000 } })
+            assertTrue(store.items().isEmpty())
+            store.create("Alpha")
+            store.create("Beta")
+            store.rename("item-001", "Alpha edited")
+            store.setCompleted("item-001", true)
+            assertTrue(store.items().first().completed)
+            store.setCompleted("item-001", false)
+            assertEquals(false, store.items().first().completed)
+            store.setCompleted("item-001", true)
+            store.delete("item-002")
+            assertEquals(expected, store.items())
+        }
+        withDatabase { reopened ->
+            assertEquals(expected, ItemStore(reopened.items()).items())
+            assertEquals(1, reopened.openHelper.readableDatabase.version)
+        }
+    }
+
+    @Test
+    fun allFiveFixtureFieldsSurviveSqliteRoundTripAndReopen() = runBlocking {
+        withDatabase { it.items().insert(fixture.toEntity()) }
+        withDatabase { reopened ->
+            assertEquals(listOf(fixture), reopened.items().readAll().map(ItemEntity::toDomain))
+        }
+    }
+
+    @Test
+    fun unknownSchemaVersionRejectsOpenWithoutDeletingExistingData() {
+        runBlocking { withDatabase { it.items().insert(fixture.toEntity()) } }
+        val path = context.getDatabasePath(name).path
+        SQLiteDatabase.openDatabase(path, null, SQLiteDatabase.OPEN_READWRITE).use { it.version = 99 }
+        val reopened = ItemDatabase.open(context, name)
+        try {
+            assertThrows(IllegalStateException::class.java) {
+                runBlocking { reopened.items().readAll() }
+            }
+        } finally {
+            reopened.close()
+        }
+        SQLiteDatabase.openDatabase(path, null, SQLiteDatabase.OPEN_READONLY).use { database ->
+            assertEquals(99, database.version)
+            database.rawQuery("SELECT id, title, completed, version, updatedAt FROM items", null).use { cursor ->
+                assertEquals(1, cursor.count)
+                assertTrue(cursor.moveToFirst())
+                assertEquals(fixture, Item(cursor.getString(0), cursor.getString(1), cursor.getInt(2) != 0,
+                    cursor.getLong(3), cursor.getLong(4)))
+            }
+        }
+    }
+}
diff --git a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt
index 8a6b52f..9f02269 100644
--- a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt
+++ b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt
@@ -1,5 +1,7 @@
 package com.mobilesystemsevolution.kotlin
 
+import androidx.activity.compose.setContent
+import androidx.compose.material3.MaterialTheme
 import androidx.compose.ui.semantics.SemanticsProperties
 import androidx.compose.ui.semantics.getOrNull
 import androidx.compose.ui.test.SemanticsMatcher
@@ -8,54 +10,96 @@ import androidx.compose.ui.test.assertIsDisplayed
 import androidx.compose.ui.test.assertIsOff
 import androidx.compose.ui.test.assertIsOn
 import androidx.compose.ui.test.assertTextEquals
+import androidx.compose.ui.test.assertTextContains
 import androidx.compose.ui.test.junit4.createAndroidComposeRule
 import androidx.compose.ui.test.onNodeWithContentDescription
 import androidx.compose.ui.test.onNodeWithTag
 import androidx.compose.ui.test.onNodeWithText
+import androidx.compose.ui.test.onAllNodesWithText
+import androidx.compose.ui.test.onAllNodesWithTag
 import androidx.compose.ui.test.performClick
 import androidx.compose.ui.test.performTextInput
 import androidx.compose.ui.test.performTextReplacement
 import androidx.test.ext.junit.runners.AndroidJUnit4
+import androidx.test.platform.app.InstrumentationRegistry
+import androidx.compose.ui.state.ToggleableState
+import kotlinx.coroutines.runBlocking
+import org.junit.Assert.assertEquals
 import org.junit.Rule
 import org.junit.Test
 import org.junit.runner.RunWith
+import org.junit.rules.ExternalResource
+import org.junit.rules.RuleChain
 
 @RunWith(AndroidJUnit4::class)
 class ItemUiTest {
+    private val compose = createAndroidComposeRule<MainActivity>()
+
+    // Test-only empty state before Activity launch, never during a restart boundary.
     @get:Rule
-    val compose = createAndroidComposeRule<MainActivity>()
+    val rules: RuleChain = RuleChain.outerRule(object : ExternalResource() {
+        override fun before() {
+            val context = InstrumentationRegistry.getInstrumentation().targetContext
+            context.deleteDatabase(ItemDatabase.NAME)
+            context.deleteDatabase("m02-ui-error.db")
+        }
+    }).around(compose)
+
+    private val rowMatcher = SemanticsMatcher("Item row") {
+        it.config.getOrNull(SemanticsProperties.TestTag)?.startsWith("item-row-") == true
+    }
+
+    private fun awaitText(text: String) {
+        compose.waitUntil(5_000) {
+            compose.onAllNodesWithText(text).fetchSemanticsNodes().isNotEmpty()
+        }
+    }
+
+    private fun awaitCompleted(completed: Boolean) {
+        val expected = if (completed) ToggleableState.On else ToggleableState.Off
+        compose.waitUntil(5_000) {
+            compose.onNodeWithContentDescription("Completed: Alpha edited").fetchSemanticsNode()
+                .config[SemanticsProperties.ToggleableState] == expected
+        }
+    }
 
     @Test
     fun frozenM01SequenceThroughActualAndroidUi() {
-        val rowMatcher = SemanticsMatcher("Item row") {
-            it.config.getOrNull(SemanticsProperties.TestTag)?.startsWith("item-row-") == true
-        }
+        awaitText("Items (0)")
         compose.onNodeWithTag("item-count").assertTextEquals("Items (0)")
         compose.onNodeWithText("No items").assertIsDisplayed()
 
         compose.onNodeWithTag("item-title-input").performTextInput("Alpha")
         compose.onNodeWithText("Add").performClick()
+        awaitText("Alpha")
         compose.onNodeWithText("Alpha").assertIsDisplayed()
         val firstRowTag = compose.onAllNodes(rowMatcher).fetchSemanticsNodes().single()
             .config[SemanticsProperties.TestTag]
 
         compose.onNodeWithTag("item-title-input").performTextInput("Beta")
         compose.onNodeWithText("Add").performClick()
+        awaitText("Beta")
         compose.onNodeWithText("Beta").assertIsDisplayed()
         compose.onAllNodes(rowMatcher).assertCountEquals(2)
 
         compose.onNodeWithContentDescription("Edit Alpha").performClick()
         compose.onNodeWithTag("item-title-input").performTextReplacement("Alpha edited")
         compose.onNodeWithText("Save").performClick()
+        awaitText("Alpha edited")
         compose.onNodeWithText("Alpha edited").assertIsDisplayed()
         compose.onNodeWithTag(firstRowTag).assertIsDisplayed()
 
         val completed = compose.onNodeWithContentDescription("Completed: Alpha edited")
-        completed.assertIsOff().performClick().assertIsOn()
-        completed.performClick().assertIsOff()
-        completed.performClick().assertIsOn()
+        completed.assertIsOff().performClick()
+        awaitCompleted(true)
+        completed.assertIsOn().performClick()
+        awaitCompleted(false)
+        completed.assertIsOff().performClick()
+        awaitCompleted(true)
+        completed.assertIsOn()
 
         compose.onNodeWithContentDescription("Delete Beta").performClick()
+        awaitText("Items (1)")
         compose.onNodeWithTag("item-count").assertTextEquals("Items (1)")
         compose.onAllNodes(rowMatcher).assertCountEquals(1)
         compose.onNodeWithTag(firstRowTag).assertIsDisplayed()
@@ -65,4 +109,34 @@ class ItemUiTest {
         compose.onNodeWithText("Beta").assertDoesNotExist()
         compose.onNodeWithText("Alpha").assertDoesNotExist()
     }
+
+    @Test
+    fun failedWriteKeepsDraftAndCommittedList() {
+        val context = InstrumentationRegistry.getInstrumentation().targetContext
+        val database = ItemDatabase.open(context, "m02-ui-error.db")
+        val fixture = Item("item-001", "Alpha edited", true, 0, 1700000006000L)
+        try {
+            runBlocking { database.items().insert(fixture.toEntity()) }
+            val store = ItemStore(database.items(), nextId = { "item-001" }, now = { 1700000006000L })
+            compose.runOnUiThread {
+                compose.activity.setContent { MaterialTheme { ItemScreen(store) } }
+            }
+            awaitText("Alpha edited")
+            compose.onNodeWithTag("item-title-input").performTextInput("Beta")
+            compose.onNodeWithText("Add").performClick()
+            compose.waitUntil(5_000) {
+                compose.onAllNodesWithTag("storage-error").fetchSemanticsNodes().isNotEmpty()
+            }
+            compose.onNodeWithTag("storage-error").assertIsDisplayed()
+            compose.onNodeWithTag("item-title-input").assertTextContains("Beta")
+            compose.onNodeWithTag("item-count").assertTextEquals("Items (1)")
+            compose.onAllNodes(rowMatcher).assertCountEquals(1)
+            compose.onNodeWithTag("item-row-item-001").assertIsDisplayed()
+            compose.onNodeWithText("Alpha edited").assertIsDisplayed()
+            compose.onNodeWithContentDescription("Completed: Alpha edited").assertIsOn()
+            assertEquals(listOf(fixture), runBlocking { store.items() })
+        } finally {
+            database.close()
+        }
+    }
 }
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemDatabase.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemDatabase.kt
new file mode 100644
index 0000000..00c62fa
--- /dev/null
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemDatabase.kt
@@ -0,0 +1,56 @@
+package com.mobilesystemsevolution.kotlin
+
+import android.content.Context
+import androidx.room.Dao
+import androidx.room.Database
+import androidx.room.Entity
+import androidx.room.Insert
+import androidx.room.PrimaryKey
+import androidx.room.Query
+import androidx.room.Room
+import androidx.room.RoomDatabase
+
+@Entity(tableName = "items")
+data class ItemEntity(
+    @PrimaryKey val id: String,
+    val title: String,
+    val completed: Boolean,
+    val version: Long,
+    val updatedAt: Long,
+)
+
+fun Item.toEntity() = ItemEntity(id, title, completed, version, updatedAt)
+fun ItemEntity.toDomain() = Item(id, title, completed, version, updatedAt)
+
+@Dao
+interface ItemDao {
+    // Preserve the M01 insertion order without adding a future metadata column.
+    @Query("SELECT * FROM items ORDER BY rowid")
+    suspend fun readAll(): List<ItemEntity>
+
+    @Insert
+    suspend fun insert(item: ItemEntity)
+
+    @Query("UPDATE items SET title = :title, updatedAt = :updatedAt WHERE id = :id")
+    suspend fun rename(id: String, title: String, updatedAt: Long): Int
+
+    @Query("UPDATE items SET completed = :completed, updatedAt = :updatedAt WHERE id = :id")
+    suspend fun setCompleted(id: String, completed: Boolean, updatedAt: Long): Int
+
+    @Query("DELETE FROM items WHERE id = :id")
+    suspend fun delete(id: String): Int
+}
+
+@Database(entities = [ItemEntity::class], version = 1, exportSchema = true)
+abstract class ItemDatabase : RoomDatabase() {
+    abstract fun items(): ItemDao
+
+    companion object {
+        const val NAME = "items.db"
+
+        fun open(context: Context, name: String = NAME): ItemDatabase =
+            Room.databaseBuilder(context.applicationContext, ItemDatabase::class.java, name)
+                // Create v1 only on first open; reject unknown schemas instead of erasing data.
+                .build()
+    }
+}
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemStore.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemStore.kt
new file mode 100644
index 0000000..25153c6
--- /dev/null
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemStore.kt
@@ -0,0 +1,45 @@
+package com.mobilesystemsevolution.kotlin
+
+import java.util.UUID
+
+data class Item(
+    val id: String,
+    val title: String,
+    val completed: Boolean,
+    val version: Long,
+    val updatedAt: Long,
+)
+
+/** M02 reads committed local rows. Version remains zero without sync semantics. */
+class ItemStore(
+    private val dao: ItemDao,
+    private val nextId: () -> String = { UUID.randomUUID().toString() },
+    private val now: () -> Long = System::currentTimeMillis,
+) {
+    suspend fun items(): List<Item> = dao.readAll().map(ItemEntity::toDomain)
+
+    suspend fun create(title: String) {
+        val validTitle = title.trim().also { require(it.isNotEmpty()) }
+        dao.insert(Item(nextId(), validTitle, false, 0, now()).toEntity())
+    }
+
+    suspend fun rename(id: String, title: String) {
+        val validTitle = title.trim().also { require(it.isNotEmpty()) }
+        check(dao.rename(id, validTitle, now()) == 1) { "Item no longer exists" }
+    }
+
+    suspend fun setCompleted(id: String, completed: Boolean) {
+        check(dao.setCompleted(id, completed, now()) == 1) { "Item no longer exists" }
+    }
+
+    suspend fun delete(id: String) {
+        check(dao.delete(id) == 1) { "Item no longer exists" }
+    }
+}
+
+/** The same plain mapping is consumed by the screen and checked in host tests. */
+data class ItemRow(val id: String, val title: String, val completed: Boolean) {
+    val tag: String get() = "item-row-$id"
+}
+
+fun List<Item>.toRows(): List<ItemRow> = map { ItemRow(it.id, it.title, it.completed) }
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
index 8c32f83..fc3492c 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
@@ -18,9 +18,11 @@ import androidx.compose.material3.OutlinedTextField
 import androidx.compose.material3.Text
 import androidx.compose.material3.TextButton
 import androidx.compose.runtime.Composable
+import androidx.compose.runtime.LaunchedEffect
 import androidx.compose.runtime.getValue
 import androidx.compose.runtime.mutableStateOf
 import androidx.compose.runtime.remember
+import androidx.compose.runtime.rememberCoroutineScope
 import androidx.compose.runtime.setValue
 import androidx.compose.ui.Alignment
 import androidx.compose.ui.Modifier
@@ -29,23 +31,58 @@ import androidx.compose.ui.platform.testTag
 import androidx.compose.ui.semantics.contentDescription
 import androidx.compose.ui.semantics.semantics
 import androidx.compose.ui.unit.dp
+import kotlinx.coroutines.CancellationException
+import kotlinx.coroutines.launch
 
 class MainActivity : ComponentActivity() {
-    private val store = MemoryItemStore()
+    private lateinit var database: ItemDatabase
 
     override fun onCreate(savedInstanceState: Bundle?) {
         super.onCreate(savedInstanceState)
+        database = ItemDatabase.open(applicationContext)
+        val store = ItemStore(database.items())
         setContent { MaterialTheme { ItemScreen(store) } }
     }
+
+    override fun onDestroy() {
+        super.onDestroy()
+        database.close()
+    }
 }
 
 @Composable
-private fun ItemScreen(store: MemoryItemStore) {
-    var items by remember { mutableStateOf(store.items) }
+internal fun ItemScreen(store: ItemStore) {
+    var items by remember(store) { mutableStateOf<List<Item>?>(null) }
+    var busy by remember(store) { mutableStateOf(true) }
+    var storageError by remember(store) { mutableStateOf<String?>(null) }
     var title by remember { mutableStateOf("") }
     var editingId by remember { mutableStateOf<String?>(null) }
+    val scope = rememberCoroutineScope()
     val focus = LocalFocusManager.current
-    val rows = items.toRows()
+    val rows = items.orEmpty().toRows()
+
+    suspend fun accessStorage(action: suspend () -> Unit = {}, after: () -> Unit = {}) {
+        try {
+            action()
+            items = store.items()
+            storageError = null
+            after()
+        } catch (cancelled: CancellationException) {
+            throw cancelled
+        } catch (error: Exception) {
+            storageError = "Local storage error. Change not confirmed; please retry."
+        } finally {
+            busy = false
+        }
+    }
+
+    fun change(action: suspend () -> Unit, after: () -> Unit = {}) {
+        if (busy) return
+        busy = true
+        scope.launch { accessStorage(action, after) }
+    }
+
+    LaunchedEffect(store) { accessStorage() }
 
     Column(
         modifier = Modifier.fillMaxSize().verticalScroll(rememberScrollState()).padding(16.dp),
@@ -57,39 +94,50 @@ private fun ItemScreen(store: MemoryItemStore) {
             onValueChange = { title = it },
             label = { Text("Item title") },
             singleLine = true,
+            enabled = !busy && items != null,
             modifier = Modifier.fillMaxWidth().testTag("item-title-input"),
         )
         Row {
             Button(
-                enabled = title.isNotBlank(),
+                enabled = !busy && items != null && title.isNotBlank(),
                 onClick = {
                     val id = editingId
-                    if (id == null) store.create(title) else store.rename(id, title)
-                    items = store.items
-                    title = ""
-                    editingId = null
-                    focus.clearFocus()
+                    change(
+                        action = { if (id == null) store.create(title) else store.rename(id, title) },
+                        after = {
+                            title = ""
+                            editingId = null
+                            focus.clearFocus()
+                        },
+                    )
                 },
             ) { Text(if (editingId == null) "Add" else "Save") }
             if (editingId != null) {
-                TextButton(onClick = {
+                TextButton(enabled = !busy, onClick = {
                     title = ""
                     editingId = null
                     focus.clearFocus()
                 }) { Text("Cancel") }
             }
         }
-        Text("Items (${rows.size})", modifier = Modifier.testTag("item-count"))
-        if (rows.isEmpty()) Text("No items")
+        if (busy) Text(if (items == null) "Loading local items…" else "Saving locally…")
+        storageError?.let { message ->
+            Text(message, modifier = Modifier.testTag("storage-error"))
+            TextButton(enabled = !busy, onClick = { change(action = {}) }) { Text("Reload") }
+        }
+        if (items != null) {
+            Text("Items (${rows.size})", modifier = Modifier.testTag("item-count"))
+            if (rows.isEmpty()) Text("No items")
+        }
         rows.forEach { row ->
             Column(Modifier.fillMaxWidth().testTag(row.tag)) {
                 Text(row.title, style = MaterialTheme.typography.titleMedium)
                 Row(verticalAlignment = Alignment.CenterVertically) {
                     Checkbox(
                         checked = row.completed,
+                        enabled = !busy,
                         onCheckedChange = {
-                            store.setCompleted(row.id, it)
-                            items = store.items
+                            change(action = { store.setCompleted(row.id, it) })
                         },
                         modifier = Modifier.semantics {
                             contentDescription = "Completed: ${row.title}"
@@ -97,6 +145,7 @@ private fun ItemScreen(store: MemoryItemStore) {
                     )
                     Text(if (row.completed) "Completed" else "Incomplete")
                     TextButton(
+                        enabled = !busy,
                         onClick = {
                             editingId = row.id
                             title = row.title
@@ -106,13 +155,17 @@ private fun ItemScreen(store: MemoryItemStore) {
                         },
                     ) { Text("Edit") }
                     TextButton(
+                        enabled = !busy,
                         onClick = {
-                            store.delete(row.id)
-                            items = store.items
-                            if (editingId == row.id) {
-                                editingId = null
-                                title = ""
-                            }
+                            change(
+                                action = { store.delete(row.id) },
+                                after = {
+                                    if (editingId == row.id) {
+                                        editingId = null
+                                        title = ""
+                                    }
+                                },
+                            )
                         },
                         modifier = Modifier.semantics {
                             contentDescription = "Delete ${row.title}"
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/MemoryItemStore.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/MemoryItemStore.kt
deleted file mode 100644
index d686423..0000000
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/MemoryItemStore.kt
+++ /dev/null
@@ -1,49 +0,0 @@
-package com.mobilesystemsevolution.kotlin
-
-import java.util.UUID
-
-data class Item(
-    val id: String,
-    val title: String,
-    val completed: Boolean,
-    val version: Long,
-    val updatedAt: Long,
-)
-
-/** M01 owns only process memory. Version is a reserved field, always zero here. */
-class MemoryItemStore(
-    private val nextId: () -> String = { UUID.randomUUID().toString() },
-    private val now: () -> Long = System::currentTimeMillis,
-) {
-    var items: List<Item> = emptyList()
-        private set
-
-    fun create(title: String) {
-        val validTitle = title.trim().also { require(it.isNotEmpty()) }
-        items = items + Item(nextId(), validTitle, false, 0, now())
-    }
-
-    fun rename(id: String, title: String) {
-        val validTitle = title.trim().also { require(it.isNotEmpty()) }
-        items = items.map { item ->
-            if (item.id == id) item.copy(title = validTitle, updatedAt = now()) else item
-        }
-    }
-
-    fun setCompleted(id: String, completed: Boolean) {
-        items = items.map { item ->
-            if (item.id == id) item.copy(completed = completed, updatedAt = now()) else item
-        }
-    }
-
-    fun delete(id: String) {
-        items = items.filterNot { it.id == id }
-    }
-}
-
-/** The same plain mapping is consumed by the screen and checked in host tests. */
-data class ItemRow(val id: String, val title: String, val completed: Boolean) {
-    val tag: String get() = "item-row-$id"
-}
-
-fun List<Item>.toRows(): List<ItemRow> = map { ItemRow(it.id, it.title, it.completed) }
diff --git a/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt b/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
new file mode 100644
index 0000000..79610ca
--- /dev/null
+++ b/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
@@ -0,0 +1,93 @@
+package com.mobilesystemsevolution.kotlin
+
+import org.junit.Assert.assertEquals
+import org.junit.Assert.assertFalse
+import org.junit.Assert.assertTrue
+import org.junit.Assert.assertThrows
+import org.junit.Test
+import kotlinx.coroutines.runBlocking
+
+class ItemStoreTest {
+    @Test
+    fun frozenM01SequencePreservesIdentityAndRenderedRows() = runBlocking {
+        val ids = listOf("item-001", "item-002").iterator()
+        var time = 1700000000000L
+        val store = ItemStore(FakeItemDao(), nextId = { ids.next() }, now = { time.also { time += 1000 } })
+        assertTrue(store.items().isEmpty())
+        assertTrue(store.items().toRows().isEmpty())
+
+        store.create("Alpha")
+        assertEquals(Item("item-001", "Alpha", false, 0, 1700000000000L), store.items().single())
+        store.create("Beta")
+        assertEquals(
+            listOf(ItemRow("item-001", "Alpha", false), ItemRow("item-002", "Beta", false)),
+            store.items().toRows(),
+        )
+
+        store.rename("item-001", "Alpha edited")
+        assertEquals("item-001", store.items().first().id)
+        assertEquals("Alpha edited", store.items().toRows().first().title)
+        assertEquals(1700000002000L, store.items().first().updatedAt)
+
+        store.setCompleted("item-001", true)
+        assertTrue(store.items().toRows().first().completed)
+        store.setCompleted("item-001", false)
+        assertFalse(store.items().toRows().first().completed)
+        store.setCompleted("item-001", true)
+        assertTrue(store.items().toRows().first().completed)
+        store.delete("item-002")
+
+        assertEquals(
+            listOf(Item("item-001", "Alpha edited", true, 0, 1700000005000L)),
+            store.items(),
+        )
+        assertEquals(listOf(ItemRow("item-001", "Alpha edited", true)), store.items().toRows())
+        assertEquals("item-row-item-001", store.items().toRows().single().tag)
+        assertFalse(ids.hasNext())
+    }
+
+    @Test
+    fun allFiveFieldsMapInBothDirections() {
+        val fixture = Item("item-001", "Alpha edited", true, 0, 1700000006000L)
+        val entity = ItemEntity("item-001", "Alpha edited", true, 0, 1700000006000L)
+        assertEquals(entity, fixture.toEntity())
+        assertEquals(fixture, entity.toDomain())
+        // A mapper must preserve the values, not hard-code M02's current version/completion.
+        val other = fixture.copy(completed = false, version = 7, updatedAt = 1700000007000L)
+        assertEquals(other, other.toEntity().toDomain())
+    }
+
+    @Test
+    fun rejectedWriteDoesNotReplaceCommittedItem() {
+        val store = ItemStore(FakeItemDao(), nextId = { "item-001" }, now = { 1700000006000L })
+        runBlocking { store.create("Alpha edited") }
+        val before = runBlocking { store.items() }
+        assertThrows(IllegalStateException::class.java) { runBlocking { store.create("Beta") } }
+        assertEquals(before, runBlocking { store.items() })
+    }
+
+    private class FakeItemDao : ItemDao {
+        private val rows = linkedMapOf<String, ItemEntity>()
+
+        override suspend fun readAll(): List<ItemEntity> = rows.values.toList()
+
+        override suspend fun insert(item: ItemEntity) {
+            check(item.id !in rows) { "Duplicate primary key" }
+            rows[item.id] = item
+        }
+
+        override suspend fun rename(id: String, title: String, updatedAt: Long): Int {
+            val item = rows[id] ?: return 0
+            rows[id] = item.copy(title = title, updatedAt = updatedAt)
+            return 1
+        }
+
+        override suspend fun setCompleted(id: String, completed: Boolean, updatedAt: Long): Int {
+            val item = rows[id] ?: return 0
+            rows[id] = item.copy(completed = completed, updatedAt = updatedAt)
+            return 1
+        }
+
+        override suspend fun delete(id: String): Int = if (rows.remove(id) == null) 0 else 1
+    }
+}
diff --git a/app/src/test/java/com/mobilesystemsevolution/kotlin/MemoryItemStoreTest.kt b/app/src/test/java/com/mobilesystemsevolution/kotlin/MemoryItemStoreTest.kt
deleted file mode 100644
index 926b803..0000000
--- a/app/src/test/java/com/mobilesystemsevolution/kotlin/MemoryItemStoreTest.kt
+++ /dev/null
@@ -1,54 +0,0 @@
-package com.mobilesystemsevolution.kotlin
-
-import org.junit.Assert.assertEquals
-import org.junit.Assert.assertFalse
-import org.junit.Assert.assertTrue
-import org.junit.Test
-
-class MemoryItemStoreTest {
-    @Test
-    fun frozenM01SequencePreservesIdentityAndRenderedRows() {
-        val ids = listOf("item-001", "item-002").iterator()
-        var time = 1700000000000L
-        val store = MemoryItemStore(nextId = { ids.next() }, now = { time.also { time += 1000 } })
-        assertTrue(store.items.isEmpty())
-        assertTrue(store.items.toRows().isEmpty())
-
-        store.create("Alpha")
-        assertEquals(Item("item-001", "Alpha", false, 0, 1700000000000L), store.items.single())
-        store.create("Beta")
-        assertEquals(
-            listOf(ItemRow("item-001", "Alpha", false), ItemRow("item-002", "Beta", false)),
-            store.items.toRows(),
-        )
-
-        store.rename("item-001", "Alpha edited")
-        assertEquals("item-001", store.items.first().id)
-        assertEquals("Alpha edited", store.items.toRows().first().title)
-        assertEquals(1700000002000L, store.items.first().updatedAt)
-
-        store.setCompleted("item-001", true)
-        assertTrue(store.items.toRows().first().completed)
-        store.setCompleted("item-001", false)
-        assertFalse(store.items.toRows().first().completed)
-        store.setCompleted("item-001", true)
-        assertTrue(store.items.toRows().first().completed)
-        store.delete("item-002")
-
-        assertEquals(
-            listOf(Item("item-001", "Alpha edited", true, 0, 1700000005000L)),
-            store.items,
-        )
-        assertEquals(listOf(ItemRow("item-001", "Alpha edited", true)), store.items.toRows())
-        assertEquals("item-row-item-001", store.items.toRows().single().tag)
-        assertFalse(ids.hasNext())
-    }
-
-    @Test
-    fun newStoreHasNoPersistence() {
-        val store = MemoryItemStore()
-        store.create("Memory only")
-        assertEquals(1, store.items.size)
-        assertTrue(MemoryItemStore().items.isEmpty())
-    }
-}
diff --git a/build.gradle.kts b/build.gradle.kts
index bbd3daa..e15018f 100644
--- a/build.gradle.kts
+++ b/build.gradle.kts
@@ -1,4 +1,5 @@
 plugins {
     id("com.android.application") version "8.6.1" apply false
     id("org.jetbrains.kotlin.android") version "1.9.24" apply false
+    id("org.jetbrains.kotlin.kapt") version "1.9.24" apply false
 }
diff --git a/verification/M02.md b/verification/M02.md
new file mode 100644
index 0000000..80f39db
--- /dev/null
+++ b/verification/M02.md
@@ -0,0 +1,129 @@
+# M02 verification — attempt 1
+
+- Thread: M02; branch: `track/android-kotlin`
+- START: `067d78ea3d38c9787971f867f3590f2ec41687cf` (verified M01 END)
+- Spec revision: `ed7baa246ee947c6778e0f84751c9b91cec7abfe`
+- Frozen M02 scenario SHA-256: `48a5002b9d340cd4b42157e7fa3604498f3348e5d359436d76f69ca087f0ac6a`
+- Flow: new process-termination constraint → Reproduce → Fix → Verify → Commit → Report
+- Device: shared API 34 Pixel 6 ARM64 Google APIs, `emulator-5554`, unchanged profile
+- Raw evidence root: `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/android-kotlin/M02/`
+
+## Frozen inputs
+
+The shared M01 sequence is unchanged: start empty; create Alpha; create Beta; rename the
+first Item to Alpha edited; complete, uncomplete, and complete it; delete Beta. Terminate
+only after the final committed UI state is visible. The harness invokes `am force-stop`
+externally, confirms `pidof` is empty, launches again, and requires a different process ID.
+There is no clear, reinstall, instrumentation, or database mutation across that boundary.
+The single scenario-start `pm clear` is explicit in the command ledger.
+
+The mapping fixture is exactly `Item("item-001", "Alpha edited", true, 0, 1700000006000)`.
+The original M01 deterministic CRUD clock still starts at 1700000000000 and advances by
+1000 per create/rename/completion change; its last changed timestamp is 1700000005000.
+Deleting Beta does not update Alpha's timestamp. The mapping fixture is a separate round trip.
+
+Storage failure test inputs were fixed before the first fixed-build verification:
+duplicate primary key `item-001` when adding `Beta` over an existing fixture, and a dedicated
+test database with the same five-column schema/fixture but `PRAGMA user_version=99`.
+The duplicate must abort without replacing the existing row, clear success, or lost draft.
+The unsupported schema must reject opening without destroying the stored row. These failure
+fixtures do not touch the app's process-restart acceptance database.
+
+## Reproduce — unchanged verified M01
+
+Before any production modification, `baseline-M01.apk` was copied from the verified build.
+SHA-256: `60cb9e3c8402e4e75ffedcc4ae4d7f4f41845e9fa37d2520faf92811c7d08d95`.
+The reproduction harness SHA-256 is
+`8d0abf7be76a4b408c3ccbc599eca5f54adcbb11e214ce30c9b2879a806c8daf`.
+The immutable APK, harness copy, frozen-input JSON and every command/output are retained.
+
+The actual Android UI showed Items (1), Alpha edited, completed true, and no Beta before
+termination. PID 7640 was force-stopped; the subsequent `pidof` exited 1 with empty output,
+as expected for a stopped process. Launch created PID 8212 and displayed Items (0)/No items.
+The new constraint therefore exposes M01's data loss. The harness exits 0 because it asserts
+the expected baseline limitation; this is not a claim that the baseline is durable.
+Both screenshots were visually inspected; the harness parsed the UI XML snapshots. No baseline production change
+was made to cause the failure. The emulator lease was released immediately after this run.
+
+## Invocation ledger
+
+Every acceptance/test invocation and failure is recorded here. Commands run from the track
+root. Full stdout/stderr, XML, raw database/WAL copies and per-adb command exits stay outside
+Git under the evidence root. No performance/battery runs, soak, matrix, or network tests.
+
+| Invocation | Exact command | Exit | Result |
+| --- | --- | --- | --- |
+| Baseline process restart 1 | `python3 verification/process_restart.py --expect memory-loss --apk /private/tmp/mobile-systems-evolution-ed7baa2/evidence/android-kotlin/M02/baseline-M01.apk --output /private/tmp/mobile-systems-evolution-ed7baa2/evidence/android-kotlin/M02/baseline-01 --adb /opt/homebrew/share/android-commandlinetools/platform-tools/adb` | 0 | Expected M01 loss reproduced; 70 adb commands; PIDs 7640 → 8212; `baseline-01.log`, `baseline-01/result.json`, `baseline-01/commands.txt` |
+| Host/build 1 | `env JAVA_HOME=/opt/homebrew/opt/openjdk@17/libexec/openjdk.jdk/Contents/Home ANDROID_HOME=/opt/homebrew/share/android-commandlinetools GRADLE_USER_HOME=/private/tmp/mobile-systems-evolution-ed7baa2/gradle-kotlin ./gradlew --no-daemon --console=plain :app:testDebugUnitTest :app:assembleDebug :app:assembleDebugAndroidTest` | 0 | 3 host tests passed, 0 failures/errors/skips; app/test APKs built in 44s; `host-build-01.log`, `host-results-01.xml` |
+| Android suite 1 | `env JAVA_HOME=/opt/homebrew/opt/openjdk@17/libexec/openjdk.jdk/Contents/Home ANDROID_HOME=/opt/homebrew/share/android-commandlinetools GRADLE_USER_HOME=/private/tmp/mobile-systems-evolution-ed7baa2/gradle-kotlin ANDROID_SERIAL=emulator-5554 ./gradlew --no-daemon --console=plain :app:connectedDebugAndroidTest` | 0 | 5 actual Android tests passed, 0 failures/errors/skips; build/test command 30s, tests 14.516s; `android-suite-01.log`, `android-suite-results-01.xml` |
+| Fixed process restart 1 | `python3 verification/process_restart.py --expect durable --apk app/build/outputs/apk/debug/app-debug.apk --output /private/tmp/mobile-systems-evolution-ed7baa2/evidence/android-kotlin/M02/durable-01 --adb /opt/homebrew/share/android-commandlinetools/platform-tools/adb` | 0 | Exact survivor restored; Beta absent; 79 adb commands; PIDs 9446 → 10017; `durable-01.log`, `durable-01/result.json`, `durable-01/commands.txt` |
+
+Gradle verification uses the following fixed environment, with explicit sandbox escalation
+for dependencies/native tools and a separate emulator lease before connected tests:
+
+```sh
+JAVA_HOME=/opt/homebrew/opt/openjdk@17/libexec/openjdk.jdk/Contents/Home
+ANDROID_HOME=/opt/homebrew/share/android-commandlinetools
+GRADLE_USER_HOME=/private/tmp/mobile-systems-evolution-ed7baa2/gradle-kotlin
+```
+
+Host tests cover the original fixed CRUD/row mapping sequence, all five fields in both
+mapping directions, and a rejected write retaining previously stored state. The old
+`newStoreHasNoPersistence` test was replaced because M02 intentionally changes that guarantee.
+
+Android tests executed the unchanged M01 UI mutations on real Room-backed storage, a real
+SQLite primary-key failure with visible error and retained draft/list, the fixed CRUD
+sequence across database close/open, the exact five-field mapping fixture through SQLite,
+and an unsupported schema open that leaves the stored row and version 99 intact. Database
+reopening is recorded as supporting coverage, not as process-termination verification.
+
+The Android XML timestamp is 2026-08-27T23:43:35 UTC. It contains all five successful test
+cases, including both `ItemUiTest` methods. Raw per-test instrumentation logs are copied
+under `android-instrumentation-01-*`. Kapt emitted unused-option warnings in test source
+sets, but production schema generation and all compilation/testing succeeded.
+
+## Fixed actual process-termination result
+
+The same harness SHA-256 was used without changes. The fixed APK SHA-256 is
+`cdc94c258321d8af0707f454cb8b980fb350397f8544fef68dc4b84e8b3b2968`.
+Before termination, the external UI controls completed the frozen sequence. The harness
+captured the committed database and WAL, force-stopped PID 9446, observed no app PID, and
+launched PID 10017. The restored UI screenshot was visually inspected: one completed
+Alpha edited Item, no Beta. Its independently copied SQLite rows exactly matched:
+
+| Field | Before stop and after relaunch |
+| --- | --- |
+| id | `8793cfb9-fdcb-4562-b6f9-2df96768cfd4` |
+| title | `Alpha edited` |
+| completed | `1` (SQLite Boolean true) |
+| version | `0` |
+| updatedAt | `1787874287456` |
+
+The first ID was captured after the two creates and stayed unchanged through edits.
+Deleted Beta's captured ID, `db8d4a52-4aed-4e7c-ba85-cc9f793fbf4c`, was absent before
+termination and after relaunch. All three raw database/WAL snapshots and their parsed
+JSON are under `durable-01/after-create`, `before-stop`, and `after-relaunch`.
+Host SQLite reads use disposable copies so the raw captured files stay unchanged.
+There was no instrumentation/repository recreation standing in for process death,
+and no clear, reinstall, data deletion or rewriting between final save and restored assertions.
+The emulator lease was returned immediately after this successful run.
+
+## Counters and scope audit
+
+- Implementation attempts: 1; repairs: 0.
+- Baseline process-loss reproductions: 1 (expected limitation confirmed).
+- Fixed host invocations: 1; tests passed: 3; failed/errors/skipped: 0.
+- Fixed Android suite invocations: 1; tests passed: 5; failed/errors/skipped: 0.
+- Fixed external process-restart invocations: 1 passed; 0 failed.
+- No failed test/build/setup invocation. The two empty `pidof` checks intentionally exit 1
+  and confirm process termination; they are not ignored test failures.
+- Static whitespace checks: `git diff --check` and staged `git diff --cached --check`.
+
+Room is the only new runtime storage dependency. The schema export contains only the five
+required Item fields and Room's own schema identity table. Writes/readback occur before UI
+confirmation, and storage errors preserve the last confirmed state. No network permission,
+sync fixture/queue, conflict metadata, background scheduler, push, native module, or
+state-management framework was added. M01 code was not sabotaged to produce the baseline.
+All changes extend the verified M01 END linearly; the M02 commit has exactly the required
+Thread and frozen Spec-Revision trailers. Tagging and independent re-verification remain
+the main orchestrator's responsibility.
diff --git a/verification/process_restart.py b/verification/process_restart.py
new file mode 100644
index 0000000..8ffa2da
--- /dev/null
+++ b/verification/process_restart.py
@@ -0,0 +1,227 @@
+#!/usr/bin/env python3
+"""Frozen M02 UI sequence and external process death on the leased API 34 device.
+
+Run only with the shared emulator lease. The only database clear is at the start.
+All adb commands and captured outputs are retained in a new evidence directory.
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
+import xml.etree.ElementTree as ET
+
+
+PACKAGE = "com.mobilesystemsevolution.kotlin"
+SERIAL = "emulator-5554"
+DB_NAME = "items.db"
+UI_XML = "/sdcard/mse-kotlin-m02-ui.xml"
+
+
+class Scenario:
+    def __init__(self, args):
+        self.args = args
+        self.output = args.output.resolve()
+        self.output.mkdir(parents=True, exist_ok=False)
+        self.command_count = 0
+        self.dump_count = 0
+        self.result = {
+            "scenario": "M02", "expectation": args.expect,
+            "serial": SERIAL, "package": PACKAGE,
+            "apkSha256": hashlib.sha256(args.apk.read_bytes()).hexdigest(),
+            "harnessSha256": hashlib.sha256(Path(__file__).read_bytes()).hexdigest(),
+        }
+
+    def adb(self, *args, binary=False, allow_failure=False):
+        command = [str(self.args.adb), "-s", SERIAL, *args]
+        self.command_count += 1
+        prefix = self.output / f"command-{self.command_count:03d}"
+        with (self.output / "commands.txt").open("a") as ledger:
+            ledger.write(f"{self.command_count:03d} {shlex.join(command)}\n")
+        result = subprocess.run(command, capture_output=True, timeout=45)
+        prefix.with_suffix(".stdout").write_bytes(result.stdout)
+        prefix.with_suffix(".stderr").write_bytes(result.stderr)
+        with (self.output / "commands.txt").open("a") as ledger:
+            ledger.write(f"    exit={result.returncode}\n")
+        if not allow_failure and result.returncode:
+            raise AssertionError(f"Command failed: {shlex.join(command)}; {result.stderr!r}")
+        return result.stdout if binary else result.stdout.decode("utf-8", errors="replace").strip()
+
+    def hierarchy(self):
+        self.adb("shell", "uiautomator", "dump", UI_XML)
+        xml = self.adb("exec-out", "cat", UI_XML, binary=True)
+        self.dump_count += 1
+        (self.output / f"ui-{self.dump_count:02d}.xml").write_bytes(xml)
+        return ET.fromstring(xml)
+
+    def wait_for(self, predicate, description):
+        deadline = time.monotonic() + 30
+        while True:
+            tree = self.hierarchy()
+            if predicate(tree):
+                return tree
+            if time.monotonic() >= deadline:
+                raise AssertionError(f"UI did not reach: {description}")
+            time.sleep(0.2)
+
+    @staticmethod
+    def matching(tree, **attributes):
+        return [node for node in tree.iter("node")
+                if all(node.get(key) == value for key, value in attributes.items())]
+
+    def tap(self, **attributes):
+        tree = self.wait_for(lambda root: bool(self.matching(root, **attributes)), str(attributes))
+        nodes = self.matching(tree, **attributes)
+        if len(nodes) != 1:
+            raise AssertionError(f"Expected one control {attributes}, found {len(nodes)}")
+        bounds = [int(value) for value in re.findall(r"\d+", nodes[0].get("bounds", ""))]
+        if len(bounds) != 4:
+            raise AssertionError(f"Missing control bounds: {attributes}")
+        self.adb("shell", "input", "tap", str((bounds[0] + bounds[2]) // 2),
+                 str((bounds[1] + bounds[3]) // 2))
+
+    def text(self, value):
+        self.adb("shell", "input", "text", value.replace(" ", "%s"))
+
+    def wait_text(self, text):
+        return self.wait_for(lambda tree: bool(self.matching(tree, text=text)), text)
+
+    def wait_completed(self, completed):
+        return self.wait_for(
+            lambda tree: bool(self.matching(tree, **{
+                "content-desc": "Completed: Alpha edited", "checked": str(completed).lower(),
+            })), f"completed={completed}",
+        )
+
+    def snapshot(self, name):
+        """Read a quiescent committed database and its WAL; never modify app storage."""
+        folder = self.output / name
+        folder.mkdir()
+        names = self.adb("shell", "run-as", PACKAGE, "ls", "databases").splitlines()
+        if DB_NAME not in names:
+            raise AssertionError("Persistent database missing")
+        for suffix in ("", "-wal"):
+            filename = DB_NAME + suffix
+            if filename in names:
+                data = self.adb("exec-out", "run-as", PACKAGE, "cat", f"databases/{filename}", binary=True)
+                (folder / filename).write_bytes(data)
+        # SQLite may checkpoint a copied WAL while reading. Keep the raw evidence untouched.
+        with tempfile.TemporaryDirectory(prefix="mse-m02-snapshot-") as temporary:
+            local = Path(temporary)
+            for captured in folder.iterdir():
+                shutil.copyfile(captured, local / captured.name)
+            with sqlite3.connect(local / DB_NAME) as database:
+                columns = [row[1] for row in database.execute("PRAGMA table_info(items)")]
+                if columns != ["id", "title", "completed", "version", "updatedAt"]:
+                    raise AssertionError(f"Unexpected schema fields: {columns}")
+                version = database.execute("PRAGMA user_version").fetchone()[0]
+                if version != 1:
+                    raise AssertionError(f"Expected initial schema 1, got {version}")
+                rows = [dict(zip(columns, row)) for row in database.execute(
+                    "SELECT id, title, completed, version, updatedAt FROM items ORDER BY rowid"
+                )]
+        (folder / "items.json").write_text(json.dumps(rows, indent=2) + "\n")
+        return rows
+
+    def final_ui(self):
+        tree = self.wait_text("Items (1)")
+        assert self.matching(tree, text="Alpha edited"), "Final title absent"
+        assert self.matching(tree, **{"content-desc": "Completed: Alpha edited", "checked": "true"})
+        assert not self.matching(tree, text="Beta"), "Deleted Beta is visible"
+        assert not self.matching(tree, text="Alpha"), "Old Alpha title is visible"
+
+    def run(self):
+        self.adb("install", "-r", str(self.args.apk.resolve()))
+        assert self.adb("shell", "pm", "clear", PACKAGE) == "Success"
+        self.adb("shell", "am", "start", "-W", "-n", f"{PACKAGE}/.MainActivity")
+        self.wait_text("Items (0)")
+        self.wait_text("No items")
+
+        self.tap(**{"class": "android.widget.EditText"})
+        self.text("Alpha")
+        self.tap(text="Add")
+        self.wait_text("Alpha")
+        self.tap(**{"class": "android.widget.EditText"})
+        self.text("Beta")
+        self.tap(text="Add")
+        self.wait_text("Items (2)")
+        if self.args.expect == "durable":
+            initial = self.snapshot("after-create")
+            assert len(initial) == 2 and [row["title"] for row in initial] == ["Alpha", "Beta"]
+
+        self.tap(**{"content-desc": "Edit Alpha"})
+        self.tap(**{"class": "android.widget.EditText"})
+        self.adb("shell", "input", "keyevent", "KEYCODE_MOVE_END")
+        self.adb("shell", "input", "keyevent", *(["KEYCODE_DEL"] * len("Alpha")))
+        self.text("Alpha edited")
+        self.tap(text="Save")
+        self.wait_text("Alpha edited")
+        self.wait_completed(False)
+        for completed in (True, False, True):
+            self.tap(**{"content-desc": "Completed: Alpha edited"})
+            self.wait_completed(completed)
+        self.tap(**{"content-desc": "Delete Beta"})
+        self.final_ui()
+        (self.output / "before-stop.png").write_bytes(self.adb("exec-out", "screencap", "-p", binary=True))
+        if self.args.expect == "durable":
+            before = self.snapshot("before-stop")
+            assert len(before) == 1
+            assert before[0]["id"] == initial[0]["id"]
+            assert before[0]["title"] == "Alpha edited" and before[0]["completed"] == 1
+            assert before[0]["version"] == 0 and before[0]["updatedAt"] >= initial[0]["updatedAt"]
+            assert initial[1]["id"] not in [row["id"] for row in before]
+            self.result["beforeStop"] = before
+
+        old_pid = self.adb("shell", "pidof", PACKAGE)
+        assert old_pid.isdigit(), f"Missing or multiple app PIDs: {old_pid!r}"
+        self.adb("shell", "am", "force-stop", PACKAGE)
+        assert not self.adb("shell", "pidof", PACKAGE, allow_failure=True), "Old app process still exists"
+        # No clear, reinstall, repository reset, or instrumentation occurs across this boundary.
+        self.adb("shell", "am", "start", "-W", "-n", f"{PACKAGE}/.MainActivity")
+        new_pid = self.adb("shell", "pidof", PACKAGE)
+        assert new_pid.isdigit() and new_pid != old_pid, f"Process not replaced: {old_pid} -> {new_pid}"
+        self.result.update({"beforePid": int(old_pid), "afterPid": int(new_pid)})
+        if self.args.expect == "memory-loss":
+            tree = self.wait_text("Items (0)")
+            assert self.matching(tree, text="No items")
+            assert not self.matching(tree, text="Alpha edited")
+            self.result["observed"] = "M01 survivor lost after external process termination"
+        else:
+            self.final_ui()
+            after = self.snapshot("after-relaunch")
+            assert after == before, f"All-five-field persistence mismatch: {before!r} -> {after!r}"
+            self.result["afterRelaunch"] = after
+            self.result["observed"] = "Exact survivor restored; deleted Beta absent"
+        (self.output / "after-relaunch.png").write_bytes(self.adb("exec-out", "screencap", "-p", binary=True))
+        self.result["status"] = "PASS"
+
+
+def main():
+    parser = argparse.ArgumentParser(description=__doc__)
+    parser.add_argument("--expect", choices=("memory-loss", "durable"), required=True)
+    parser.add_argument("--apk", type=Path, required=True)
+    parser.add_argument("--output", type=Path, required=True)
+    parser.add_argument("--adb", type=Path, default=Path(os.environ.get("ANDROID_HOME", "")) / "platform-tools/adb")
+    args = parser.parse_args()
+    scenario = Scenario(args)
+    try:
+        scenario.run()
+    except Exception as error:
+        scenario.result.update({"status": "FAIL", "error": repr(error)})
+        raise
+    finally:
+        scenario.result["adbCommands"] = scenario.command_count
+        (scenario.output / "result.json").write_text(json.dumps(scenario.result, indent=2) + "\n")
+        print(json.dumps(scenario.result, indent=2), flush=True)
+
+
+if __name__ == "__main__":
+    main()
