# M04 — Offline Read

## `feat(android): add M04 refresh status and saved success time`

diff --git a/app/schemas/com.mobilesystemsevolution.kotlin.ItemDatabase/2.json b/app/schemas/com.mobilesystemsevolution.kotlin.ItemDatabase/2.json
new file mode 100644
index 0000000..1f62657
--- /dev/null
+++ b/app/schemas/com.mobilesystemsevolution.kotlin.ItemDatabase/2.json
@@ -0,0 +1,84 @@
+{
+  "formatVersion": 1,
+  "database": {
+    "version": 2,
+    "identityHash": "ab0a985408dc3a7a966bef0266c6ca21",
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
+      },
+      {
+        "tableName": "sync_metadata",
+        "createSql": "CREATE TABLE IF NOT EXISTS `${TABLE_NAME}` (`id` INTEGER NOT NULL, `lastSuccessfulRefreshAt` INTEGER NOT NULL, PRIMARY KEY(`id`))",
+        "fields": [
+          {
+            "fieldPath": "id",
+            "columnName": "id",
+            "affinity": "INTEGER",
+            "notNull": true
+          },
+          {
+            "fieldPath": "lastSuccessfulRefreshAt",
+            "columnName": "lastSuccessfulRefreshAt",
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
+      "INSERT OR REPLACE INTO room_master_table (id,identity_hash) VALUES(42, 'ab0a985408dc3a7a966bef0266c6ca21')"
+    ]
+  }
+}
\ No newline at end of file
diff --git a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemDatabaseTest.kt b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemDatabaseTest.kt
index 798daae..474d409 100644
--- a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemDatabaseTest.kt
+++ b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemDatabaseTest.kt
@@ -6,6 +6,7 @@ import androidx.test.platform.app.InstrumentationRegistry
 import kotlinx.coroutines.runBlocking
 import org.junit.Assert.assertEquals
 import org.junit.Assert.assertThrows
+import org.junit.Assert.assertNull
 import org.junit.Assert.assertTrue
 import org.junit.Before
 import org.junit.Test
@@ -52,7 +53,7 @@ class ItemDatabaseTest {
         }
         withDatabase { reopened ->
             assertEquals(expected, ItemStore(reopened.items()).items())
-            assertEquals(1, reopened.openHelper.readableDatabase.version)
+            assertEquals(2, reopened.openHelper.readableDatabase.version)
         }
     }
 
@@ -98,12 +99,49 @@ class ItemDatabaseTest {
     fun canonicalReplacementRollsBackIfAnyRowCannotBeInserted() = runBlocking {
         withDatabase { database ->
             val store = ItemStore(database.items())
-            database.items().insert(fixture.toEntity())
+            store.replaceWithCanonical(listOf(fixture), 1700000200000L)
             val duplicate = Item("remote-001", "Alpha", false, 1, 1700000000000L)
             assertThrows(android.database.sqlite.SQLiteConstraintException::class.java) {
-                runBlocking { store.replaceWithCanonical(listOf(duplicate, duplicate)) }
+                runBlocking { store.replaceWithCanonical(listOf(duplicate, duplicate), 1700000202000L) }
             }
             assertEquals(listOf(fixture), store.items())
+            assertEquals(1700000200000L, store.lastSuccessfulRefreshAt())
+        }
+    }
+
+    @Test
+    fun v1MigrationPreservesItemsAndRefreshTimeSurvivesReopen() = runBlocking {
+        val seeds = listOf(
+            Item("remote-001", "Alpha", false, 1, 1700000000000L),
+            Item("remote-002", "Beta", false, 1, 1700000000000L),
+        )
+        val path = context.getDatabasePath(name)
+        requireNotNull(path.parentFile).mkdirs()
+        SQLiteDatabase.openOrCreateDatabase(path, null).use { legacy ->
+            legacy.execSQL("CREATE TABLE IF NOT EXISTS `items` (`id` TEXT NOT NULL, " +
+                "`title` TEXT NOT NULL, `completed` INTEGER NOT NULL, `version` INTEGER NOT NULL, " +
+                "`updatedAt` INTEGER NOT NULL, PRIMARY KEY(`id`))")
+            legacy.execSQL("CREATE TABLE IF NOT EXISTS room_master_table (id INTEGER PRIMARY KEY,identity_hash TEXT)")
+            legacy.execSQL("INSERT OR REPLACE INTO room_master_table (id,identity_hash) " +
+                "VALUES(42, '8d9e5e27ab3b33aa75b77b7bc50e06a3')")
+            seeds.forEach { item ->
+                legacy.execSQL("INSERT INTO items (id,title,completed,version,updatedAt) VALUES(?,?,?,?,?)",
+                    arrayOf(item.id, item.title, if (item.completed) 1 else 0, item.version, item.updatedAt))
+            }
+            legacy.version = 1
+        }
+        withDatabase { migrated ->
+            val store = ItemStore(migrated.items())
+            assertEquals(seeds, store.items())
+            assertEquals(2, migrated.openHelper.readableDatabase.version)
+            assertNull(store.lastSuccessfulRefreshAt())
+            store.replaceWithCanonical(seeds, 1700000200000L)
+        }
+        withDatabase { reopened ->
+            val store = ItemStore(reopened.items())
+            assertEquals(seeds, store.items())
+            assertEquals(1700000200000L, store.lastSuccessfulRefreshAt())
+            android.util.Log.i("M04Migration", "schema=2; rows=${store.items()}; lastSuccess=${store.lastSuccessfulRefreshAt()}")
         }
     }
 
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemDatabase.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemDatabase.kt
index 0a8f5ff..6e3fdc6 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemDatabase.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemDatabase.kt
@@ -5,11 +5,14 @@ import androidx.room.Dao
 import androidx.room.Database
 import androidx.room.Entity
 import androidx.room.Insert
+import androidx.room.OnConflictStrategy
 import androidx.room.PrimaryKey
 import androidx.room.Query
 import androidx.room.Room
 import androidx.room.RoomDatabase
 import androidx.room.Transaction
+import androidx.room.migration.Migration
+import androidx.sqlite.db.SupportSQLiteDatabase
 
 @Entity(tableName = "items")
 data class ItemEntity(
@@ -23,6 +26,12 @@ data class ItemEntity(
 fun Item.toEntity() = ItemEntity(id, title, completed, version, updatedAt)
 fun ItemEntity.toDomain() = Item(id, title, completed, version, updatedAt)
 
+@Entity(tableName = "sync_metadata")
+data class SyncMetadata(
+    @PrimaryKey val id: Int = 1,
+    val lastSuccessfulRefreshAt: Long,
+)
+
 @Dao
 interface ItemDao {
     // Preserve the M01 insertion order without adding a future metadata column.
@@ -44,23 +53,40 @@ interface ItemDao {
     @Query("DELETE FROM items")
     suspend fun deleteAll()
 
+    @Query("SELECT lastSuccessfulRefreshAt FROM sync_metadata WHERE id = 1")
+    suspend fun lastSuccessfulRefreshAt(): Long?
+
+    @Insert(onConflict = OnConflictStrategy.REPLACE)
+    suspend fun saveSyncMetadata(metadata: SyncMetadata)
+
     @Transaction
-    suspend fun replaceAll(items: List<ItemEntity>) {
+    suspend fun replaceAll(items: List<ItemEntity>, refreshedAt: Long? = null): List<ItemEntity> {
         deleteAll()
         items.forEach { insert(it) }
+        if (refreshedAt != null) saveSyncMetadata(SyncMetadata(lastSuccessfulRefreshAt = refreshedAt))
+        // A failed write/read rolls back both canonical rows and their successful-refresh time.
+        return readAll()
     }
 }
 
-@Database(entities = [ItemEntity::class], version = 1, exportSchema = true)
+@Database(entities = [ItemEntity::class, SyncMetadata::class], version = 2, exportSchema = true)
 abstract class ItemDatabase : RoomDatabase() {
     abstract fun items(): ItemDao
 
     companion object {
         const val NAME = "items.db"
 
+        private val MIGRATION_1_2 = object : Migration(1, 2) {
+            override fun migrate(db: SupportSQLiteDatabase) {
+                db.execSQL("CREATE TABLE IF NOT EXISTS `sync_metadata` " +
+                    "(`id` INTEGER NOT NULL, `lastSuccessfulRefreshAt` INTEGER NOT NULL, PRIMARY KEY(`id`))")
+            }
+        }
+
         fun open(context: Context, name: String = NAME): ItemDatabase =
             Room.databaseBuilder(context.applicationContext, ItemDatabase::class.java, name)
-                // Create v1 only on first open; reject unknown schemas instead of erasing data.
+                .addMigrations(MIGRATION_1_2)
+                // Preserve existing Items; reject unknown schemas instead of erasing data.
                 .build()
     }
 }
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemStore.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemStore.kt
index 0c4ba57..fb531a1 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemStore.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemStore.kt
@@ -18,9 +18,10 @@ class ItemStore(
 ) {
     suspend fun items(): List<Item> = dao.readAll().map(ItemEntity::toDomain)
 
-    suspend fun replaceWithCanonical(items: List<Item>) {
-        dao.replaceAll(items.map(Item::toEntity))
-    }
+    suspend fun lastSuccessfulRefreshAt(): Long? = dao.lastSuccessfulRefreshAt()
+
+    suspend fun replaceWithCanonical(items: List<Item>, refreshedAt: Long? = null): List<Item> =
+        dao.replaceAll(items.map(Item::toEntity), refreshedAt).map(ItemEntity::toDomain)
 
     suspend fun create(title: String) {
         val validTitle = title.trim().also { require(it.isNotEmpty()) }
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemSync.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemSync.kt
index ed7547f..024b238 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemSync.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemSync.kt
@@ -1,5 +1,9 @@
 package com.mobilesystemsevolution.kotlin
 
+import kotlinx.coroutines.CancellationException
+import kotlinx.coroutines.flow.MutableStateFlow
+import kotlinx.coroutines.flow.asStateFlow
+
 interface ItemRemote {
     suspend fun items(): List<Item>
     suspend fun create(item: Item): Item
@@ -7,29 +11,63 @@ interface ItemRemote {
     suspend fun delete(id: String): String
 }
 
+enum class SyncState { STALE, REFRESHING, FRESH, ERROR }
+
+data class SyncStatus(
+    val state: SyncState = SyncState.STALE,
+    val lastSuccessfulRefreshAt: Long? = null,
+    val error: String? = null,
+)
+
 /** One explicit foreground call at a time; this comparison snapshot is not durable intent. */
-class ItemSync(private val store: ItemStore, private val remote: ItemRemote) {
+class ItemSync(
+    private val store: ItemStore,
+    private val remote: ItemRemote,
+    private val now: () -> Long = System::currentTimeMillis,
+) {
     private var lastSynced: Map<String, Item>? = null
+    private val mutableStatus = MutableStateFlow(SyncStatus())
+    val status = mutableStatus.asStateFlow()
+
+    // A new session is stale until explicitly refreshed, even when a saved timestamp exists.
+    suspend fun readSavedStatus() {
+        mutableStatus.value = mutableStatus.value.copy(lastSuccessfulRefreshAt = store.lastSuccessfulRefreshAt())
+    }
+
+    fun markLocalChange() {
+        mutableStatus.value = mutableStatus.value.copy(state = SyncState.STALE, error = null)
+    }
 
     suspend fun synchronize() {
-        val local = store.items().associateBy(Item::id)
-        val previous = lastSynced
-        for (item in local.values.sortedBy(Item::id)) {
-            val before = previous?.get(item.id)
-            if (before == null) {
-                // Preserve local M02 creations on an initial sync without adding a queue.
-                if (previous != null || item.version == 0L) remote.create(item)
-            } else {
-                val title = item.title.takeIf { it != before.title }
-                val completed = item.completed.takeIf { it != before.completed }
-                if (title != null || completed != null) remote.patch(item.id, title, completed)
+        try {
+            mutableStatus.value = SyncStatus(SyncState.REFRESHING, store.lastSuccessfulRefreshAt())
+            val local = store.items().associateBy(Item::id)
+            val previous = lastSynced
+            for (item in local.values.sortedBy(Item::id)) {
+                val before = previous?.get(item.id)
+                if (before == null) {
+                    // Preserve local M02 creations on an initial sync without adding a queue.
+                    if (previous != null || item.version == 0L) remote.create(item)
+                } else {
+                    val title = item.title.takeIf { it != before.title }
+                    val completed = item.completed.takeIf { it != before.completed }
+                    if (title != null || completed != null) remote.patch(item.id, title, completed)
+                }
             }
-        }
-        previous?.keys?.filter { it !in local }?.sorted()?.forEach { remote.delete(it) }
+            previous?.keys?.filter { it !in local }?.sorted()?.forEach { remote.delete(it) }
 
-        val canonical = remote.items()
-        store.replaceWithCanonical(canonical)
-        // Read back committed Room state; neither the screen nor sync owns a second UI list.
-        lastSynced = store.items().associateBy(Item::id)
+            val canonical = remote.items()
+            val refreshedAt = now()
+            // Commit the canonical list and its timestamp together, then use the Room readback.
+            lastSynced = store.replaceWithCanonical(canonical, refreshedAt).associateBy(Item::id)
+            mutableStatus.value = SyncStatus(SyncState.FRESH, refreshedAt)
+        } catch (cancelled: CancellationException) {
+            mutableStatus.value = mutableStatus.value.copy(state = SyncState.STALE, error = null)
+            throw cancelled
+        } catch (error: Exception) {
+            mutableStatus.value = mutableStatus.value.copy(state = SyncState.ERROR,
+                error = error.message ?: "Refresh failed")
+            throw error
+        }
     }
 }
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
index 4e32736..6fd9e49 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
@@ -18,6 +18,7 @@ import androidx.compose.material3.OutlinedTextField
 import androidx.compose.material3.Text
 import androidx.compose.material3.TextButton
 import androidx.compose.runtime.Composable
+import androidx.compose.runtime.collectAsState
 import androidx.compose.runtime.LaunchedEffect
 import androidx.compose.runtime.getValue
 import androidx.compose.runtime.mutableStateOf
@@ -56,8 +57,8 @@ internal fun ItemScreen(store: ItemStore, sync: ItemSync? = null) {
     var items by remember(store) { mutableStateOf<List<Item>?>(null) }
     var busy by remember(store) { mutableStateOf(true) }
     var storageError by remember(store) { mutableStateOf<String?>(null) }
-    var syncError by remember(store) { mutableStateOf<String?>(null) }
     var syncing by remember(store) { mutableStateOf(false) }
+    val syncStatus = sync?.status?.collectAsState()?.value
     var title by remember { mutableStateOf("") }
     var editingId by remember { mutableStateOf<String?>(null) }
     val scope = rememberCoroutineScope()
@@ -68,15 +69,13 @@ internal fun ItemScreen(store: ItemStore, sync: ItemSync? = null) {
         try {
             action()
             items = store.items()
+            sync?.readSavedStatus()
             storageError = null
-            syncError = null
             after()
         } catch (cancelled: CancellationException) {
             throw cancelled
         } catch (error: Exception) {
-            if (syncing) {
-                syncError = "Sync failed. Remote outcome unconfirmed."
-            } else {
+            if (!syncing || sync?.status?.value?.state != SyncState.ERROR) {
                 storageError = "Local storage error. Change not confirmed; please retry."
             }
         } finally {
@@ -113,7 +112,10 @@ internal fun ItemScreen(store: ItemStore, sync: ItemSync? = null) {
                 onClick = {
                     val id = editingId
                     change(
-                        action = { if (id == null) store.create(title) else store.rename(id, title) },
+                        action = {
+                            if (id == null) store.create(title) else store.rename(id, title)
+                            sync?.markLocalChange()
+                        },
                         after = {
                             title = ""
                             editingId = null
@@ -136,12 +138,24 @@ internal fun ItemScreen(store: ItemStore, sync: ItemSync? = null) {
                 ) { Text("Sync") }
             }
         }
-        if (busy) Text(when {
-            syncing -> "Synchronizing…"
+        if (busy && !syncing) Text(when {
             items == null -> "Loading local items…"
             else -> "Saving locally…"
         })
-        syncError?.let { Text(it, modifier = Modifier.testTag("sync-error")) }
+        syncStatus?.let { status ->
+            Text(when (status.state) {
+                SyncState.STALE -> "Stale local data"
+                SyncState.REFRESHING -> "Stale local data · refreshing…"
+                SyncState.FRESH -> "Fresh local data"
+                SyncState.ERROR -> "Stale local data · sync error"
+            }, modifier = Modifier.testTag("sync-status"))
+            Text("Last successful refresh: ${status.lastSuccessfulRefreshAt ?: "never"}",
+                modifier = Modifier.testTag("last-successful-refresh"))
+            status.error?.let {
+                Text("Refresh failed: $it. Saved items retained; remote outcome unconfirmed.",
+                    modifier = Modifier.testTag("sync-error"))
+            }
+        }
         storageError?.let { message ->
             Text(message, modifier = Modifier.testTag("storage-error"))
             TextButton(enabled = !busy, onClick = { change(action = {}) }) { Text("Reload") }
@@ -158,7 +172,10 @@ internal fun ItemScreen(store: ItemStore, sync: ItemSync? = null) {
                         checked = row.completed,
                         enabled = !busy,
                         onCheckedChange = {
-                            change(action = { store.setCompleted(row.id, it) })
+                            change(action = {
+                                store.setCompleted(row.id, it)
+                                sync?.markLocalChange()
+                            })
                         },
                         modifier = Modifier.semantics {
                             contentDescription = "Completed: ${row.title}"
@@ -179,7 +196,10 @@ internal fun ItemScreen(store: ItemStore, sync: ItemSync? = null) {
                         enabled = !busy,
                         onClick = {
                             change(
-                                action = { store.delete(row.id) },
+                                action = {
+                                    store.delete(row.id)
+                                    sync?.markLocalChange()
+                                },
                                 after = {
                                     if (editingId == row.id) {
                                         editingId = null
diff --git a/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt b/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
index aa196dd..ff39907 100644
--- a/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
+++ b/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
@@ -6,6 +6,7 @@ import org.junit.Assert.assertTrue
 import org.junit.Assert.assertThrows
 import org.junit.Test
 import kotlinx.coroutines.runBlocking
+import java.io.IOException
 
 class ItemStoreTest {
     @Test
@@ -70,7 +71,7 @@ class ItemStoreTest {
     fun frozenM03SyncPersistsCanonicalFieldsInTwoIndependentStores() = runBlocking {
         val remote = FakeRemote()
         val first = ItemStore(FakeItemDao(), nextId = { "device-001" }, now = { 1800000000000L })
-        val sync = ItemSync(first, remote)
+        val sync = ItemSync(first, remote, now = { 1800000000000L })
         assertTrue(first.items().isEmpty())
         sync.synchronize()
         assertEquals(remote.rows.values.toList(), first.items())
@@ -89,7 +90,7 @@ class ItemStoreTest {
         )
         val second = ItemStore(FakeItemDao())
         assertTrue(second.items().isEmpty())
-        ItemSync(second, remote).synchronize()
+        ItemSync(second, remote, now = { 1800000000000L }).synchronize()
         assertEquals(expected, first.items())
         assertEquals(expected, second.items())
         assertEquals(expected, remote.rows.values.sortedBy(Item::id))
@@ -104,22 +105,91 @@ class ItemStoreTest {
         val remote = FakeRemote()
         val store = ItemStore(FakeItemDao(), nextId = { "device-001" }, now = { 1600000000000L })
         store.create("Gamma")
-        ItemSync(store, remote).synchronize()
+        ItemSync(store, remote, now = { 1800000000000L }).synchronize()
         assertEquals(Item("device-001", "Gamma", false, 1, 1700000100000L), store.items().first())
         assertEquals(3, store.items().size)
         assertEquals(listOf("POST device-001", "GET"), remote.calls)
     }
 
+    @Test
+    fun frozenM04RefreshFailuresKeepRowsAndSuccessTimeUntilExplicitReconnection() = runBlocking {
+        val remote = FakeRemote()
+        val store = ItemStore(FakeItemDao(), now = { 1700000200000L })
+        val seeds = remote.rows.values.toList()
+        var time = 1700000200000L
+        var clockReads = 0
+        val sync = ItemSync(store, remote, now = { clockReads++; time })
+        val states = mutableListOf(sync.status.value.state)
+        remote.beforeGet = { states += sync.status.value.state }
+        sync.readSavedStatus()
+        assertEquals(SyncStatus(), sync.status.value)
+        sync.synchronize()
+        states += sync.status.value.state
+        assertEquals(SyncStatus(SyncState.FRESH, 1700000200000L), sync.status.value)
+        assertEquals(seeds, store.items())
+
+        remote.offline = true
+        assertThrows(IOException::class.java) { runBlocking { sync.synchronize() } }
+        states += sync.status.value.state
+        assertEquals(SyncStatus(SyncState.ERROR, 1700000200000L, "Forced offline"), sync.status.value)
+        assertEquals(seeds, store.items())
+        assertEquals(1700000200000L, store.lastSuccessfulRefreshAt())
+        assertEquals(listOf("GET"), remote.calls)
+
+        remote.offline = false
+        remote.getFailures = 1
+        assertThrows(IOException::class.java) { runBlocking { sync.synchronize() } }
+        states += sync.status.value.state
+        assertEquals(SyncStatus(SyncState.ERROR, 1700000200000L, "Sync HTTP 503"), sync.status.value)
+        assertEquals(seeds, store.items())
+        assertEquals(1700000200000L, store.lastSuccessfulRefreshAt())
+        assertEquals(1, clockReads) // Neither failed refresh even reads the success clock.
+
+        remote.offline = true
+        remote.time = 1700000201000L
+        remote.patch("remote-001", "Remote revised", null) // The other client; no local edit.
+        assertEquals(seeds, store.items())
+        remote.offline = false
+        remote.getFailures = 0
+        time = 1700000202000L
+        sync.synchronize()
+        states += sync.status.value.state
+        val expected = listOf(seeds[0].copy(title = "Remote revised", version = 2,
+            updatedAt = 1700000201000L), seeds[1])
+        assertEquals(expected, store.items())
+        assertEquals(expected, remote.rows.values.toList())
+        assertEquals(SyncStatus(SyncState.FRESH, 1700000202000L), sync.status.value)
+        assertEquals(1700000202000L, store.lastSuccessfulRefreshAt())
+        assertEquals(2, clockReads)
+        assertEquals(listOf(SyncState.STALE, SyncState.REFRESHING, SyncState.FRESH,
+            SyncState.REFRESHING, SyncState.ERROR, SyncState.REFRESHING, SyncState.ERROR,
+            SyncState.REFRESHING, SyncState.FRESH), states)
+        val freshInstance = ItemSync(store, remote, now = { 1700000202000L })
+        freshInstance.readSavedStatus()
+        assertEquals(SyncStatus(SyncState.STALE, 1700000202000L), freshInstance.status.value)
+        assertEquals(listOf("GET", "GET", "PATCH remote-001 title=Remote revised completed=null", "GET"),
+            remote.calls)
+    }
+
     private class FakeRemote : ItemRemote {
         val rows = linkedMapOf(
             "remote-001" to Item("remote-001", "Alpha", false, 1, 1700000000000L),
             "remote-002" to Item("remote-002", "Beta", false, 1, 1700000000000L),
         )
         val calls = mutableListOf<String>()
+        var offline = false
+        var getFailures = 0
+        var beforeGet: () -> Unit = {}
         var time = 1700000100000L
         private fun tick() = time.also { time += 1000 }
         override suspend fun items(): List<Item> {
+            beforeGet()
+            if (offline) throw IOException("Forced offline")
             calls += "GET"
+            if (getFailures > 0) {
+                getFailures--
+                throw IOException("Sync HTTP 503")
+            }
             return rows.values.sortedBy(Item::id)
         }
         override suspend fun create(item: Item): Item {
@@ -143,6 +213,13 @@ class ItemStoreTest {
 
     private class FakeItemDao : ItemDao {
         private val rows = linkedMapOf<String, ItemEntity>()
+        private var lastRefresh: Long? = null
+
+        override suspend fun lastSuccessfulRefreshAt(): Long? = lastRefresh
+
+        override suspend fun saveSyncMetadata(metadata: SyncMetadata) {
+            lastRefresh = metadata.lastSuccessfulRefreshAt
+        }
 
         override suspend fun readAll(): List<ItemEntity> = rows.values.toList()
 


