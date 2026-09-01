## `feat(android): persist ordered offline mutation intents`

diff --git a/TRACK.md b/TRACK.md
index 2bb878c..3f788f8 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -129,14 +129,37 @@ A separate fixed latch case proves saved rows/time remain visible while explicit
 pending. Input snapshots, baseline hashes, all invocations and boundaries are in
 `verification/M04-inputs.json` and `verification/M04.md`.
 
-## M05 boundary — initial attempt in progress
+## M05 boundary — independently verified
 
 The frozen baseline performs four real offline UI changes, externally terminates the app,
 and relaunches the ordinary Activity while still offline. M04 preserves the local rows but
 loses three upload intents: its next Sync uploads only Gamma and restores Alpha and Beta.
 `verification/M05-inputs.json` and `verification/M05.md` record the fixed inputs, exact
 source/APK freeze, accepted HTTP operation, native SQLite/PID evidence and cleanup.
-The M05 implementation and independent final verification are pending.
+Main accepted the baseline and independently passed the final fourteen Android methods plus
+the actual offline restart/drain scenario. Both host builds passed all seven methods; the
+two fixture contracts also passed. Initial attempt1 used no repairs.
+
+Schema3 adds a five-column `pending_mutations` table: local sequence, Item identity,
+operation, title payload and completion payload. Each local create/rename/completion/delete
+and its intent commit in the same Room transaction. The payload records that operation,
+not a later reconstruction from the current Item; deletion does not remove its intent.
+The screen reads committed local Items and displays the stored pending count, including
+after a failed drain. The existing failed-write draft/list behavior is retained.
+
+One explicit foreground Sync sends each stored payload in sequence, acknowledges that
+entry only after a successful response, then fetches and atomically stores canonical rows
+and the successful-refresh time. A new process reads the same queue without a comparison
+snapshot. There is no coalescing, automatic network trigger, retry loop or background job.
+Schema2→3 migration preserves rows/time and queues existing version-zero creations in
+rowid order, preserving M03 first-sync behavior. It cannot infer M04's lost older intents.
+The remote-success/local-ack crash window and idempotency remain explicitly deferred to M06.
+
+The final external run preserved all four ordered intents across PID11605 → absent → 12161
+while offline. One foreground drain accepted POST/PATCH/PATCH/DELETE in that order and left
+exactly Gamma/false/version1/1700000300000 and Queued edit/true/version3/1700000302000,
+with Beta absent and pending0. Main's raw SQLite/WAL, UI and HTTP evidence is linked in the
+M05 ledger. All tested execution bytes are unchanged; only reporting changes after the run.
 
 The test build now registers `M05ScenarioInstrumentation`, a small AndroidJUnitRunner
 subclass. Without `m05ExternalController=true` it delegates to the normal suite unchanged.
@@ -190,7 +213,7 @@ before the process-death boundary:
   -e class 'com.mobilesystemsevolution.kotlin.ItemUiTest#frozenM03SequenceUsesHttpRoomAndTwoIndependentInstances' \
   com.mobilesystemsevolution.kotlin.test/com.mobilesystemsevolution.kotlin.M05ScenarioInstrumentation
 python3 verification/canonical_restart.py --apk app/build/outputs/apk/debug/app-debug.apk \
-  --adb "$ANDROID_HOME/platform-tools/adb" --schema-version 2 --output /tmp/kotlin-M03-canonical-restart
+  --adb "$ANDROID_HOME/platform-tools/adb" --schema-version 3 --output /tmp/kotlin-M03-canonical-restart
 ```
 
 The filtered test resets only the fixture and its test databases before the sequence. The
@@ -204,7 +227,7 @@ The separate process-restart harness also requires the lease. Supply a new evide
 ```sh
 python3 verification/process_restart.py --expect durable \
   --apk app/build/outputs/apk/debug/app-debug.apk \
-  --schema-version 2 \
+  --schema-version 3 \
   --adb "$ANDROID_HOME/platform-tools/adb" --output /tmp/kotlin-M02-restart-evidence
 ```
 
@@ -215,6 +238,24 @@ reinstalls app data across the process boundary. Raw database/WAL copies are rea
 host without changing app storage. `--expect memory-loss` is solely for the preserved M01
 baseline APK; `verification/M02.md` records both phases and every test invocation.
 
+The M05 external scenario requires the same exclusive lease and a fresh fixture log.
+Start `python3 -u fixture/server.py --port 18080 > /tmp/kotlin-M05-fixture.log` in its own
+terminal, then run:
+
+```sh
+python3 verification/offline_queue_restart.py --expect durable --schema-version 3 \
+  --apk app/build/outputs/apk/debug/app-debug.apk \
+  --test-apk app/build/outputs/apk/androidTest/debug/app-debug-androidTest.apk \
+  --adb "$ANDROID_HOME/platform-tools/adb" --fixture-log /tmp/kotlin-M05-fixture.log \
+  --output /tmp/kotlin-M05-restart-evidence
+```
+
+It installs/clears only before initial seeding, drives all four UI mutations, captures native
+rows/queue and actual PID loss, relaunches plain MainActivity offline, then restores the
+original network and presses Sync once. No injection/installation occurs after process
+death. Its cleanup restores 0/1/1 with an active network and force-stops only its app.
+The fixture must be stopped by its owner after the scenario. Main owns final execution.
+
 ## Frozen M01 sequence
 
 Start empty; create Alpha; create Beta; rename Alpha to Alpha edited; mark it completed,
diff --git a/app/schemas/com.mobilesystemsevolution.kotlin.ItemDatabase/3.json b/app/schemas/com.mobilesystemsevolution.kotlin.ItemDatabase/3.json
new file mode 100644
index 0000000..191108c
--- /dev/null
+++ b/app/schemas/com.mobilesystemsevolution.kotlin.ItemDatabase/3.json
@@ -0,0 +1,128 @@
+{
+  "formatVersion": 1,
+  "database": {
+    "version": 3,
+    "identityHash": "e0f9e9ee2066e917948856f649abd584",
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
+      },
+      {
+        "tableName": "pending_mutations",
+        "createSql": "CREATE TABLE IF NOT EXISTS `${TABLE_NAME}` (`sequence` INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL, `itemId` TEXT NOT NULL, `operation` TEXT NOT NULL, `title` TEXT, `completed` INTEGER)",
+        "fields": [
+          {
+            "fieldPath": "sequence",
+            "columnName": "sequence",
+            "affinity": "INTEGER",
+            "notNull": true
+          },
+          {
+            "fieldPath": "itemId",
+            "columnName": "itemId",
+            "affinity": "TEXT",
+            "notNull": true
+          },
+          {
+            "fieldPath": "operation",
+            "columnName": "operation",
+            "affinity": "TEXT",
+            "notNull": true
+          },
+          {
+            "fieldPath": "title",
+            "columnName": "title",
+            "affinity": "TEXT",
+            "notNull": false
+          },
+          {
+            "fieldPath": "completed",
+            "columnName": "completed",
+            "affinity": "INTEGER",
+            "notNull": false
+          }
+        ],
+        "primaryKey": {
+          "autoGenerate": true,
+          "columnNames": [
+            "sequence"
+          ]
+        },
+        "indices": [],
+        "foreignKeys": []
+      }
+    ],
+    "views": [],
+    "setupQueries": [
+      "CREATE TABLE IF NOT EXISTS room_master_table (id INTEGER PRIMARY KEY,identity_hash TEXT)",
+      "INSERT OR REPLACE INTO room_master_table (id,identity_hash) VALUES(42, 'e0f9e9ee2066e917948856f649abd584')"
+    ]
+  }
+}
\ No newline at end of file
diff --git a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemDatabaseTest.kt b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemDatabaseTest.kt
index e8ef369..3242b08 100644
--- a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemDatabaseTest.kt
+++ b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemDatabaseTest.kt
@@ -53,7 +53,7 @@ class ItemDatabaseTest {
         }
         withDatabase { reopened ->
             assertEquals(expected, ItemStore(reopened.items()).items())
-            assertEquals(2, reopened.openHelper.readableDatabase.version)
+            assertEquals(3, reopened.openHelper.readableDatabase.version)
         }
     }
 
@@ -133,7 +133,7 @@ class ItemDatabaseTest {
         withDatabase { migrated ->
             val store = ItemStore(migrated.items())
             assertEquals(seeds, store.items())
-            assertEquals(2, migrated.openHelper.readableDatabase.version)
+            assertEquals(3, migrated.openHelper.readableDatabase.version)
             assertNull(store.lastSuccessfulRefreshAt())
             store.replaceWithCanonical(seeds, 1700000200000L)
         }
@@ -141,7 +141,120 @@ class ItemDatabaseTest {
             val store = ItemStore(reopened.items())
             assertEquals(seeds, store.items())
             assertEquals(1700000200000L, store.lastSuccessfulRefreshAt())
-            android.util.Log.i("M04Migration", "schema=2; rows=${store.items()}; lastSuccess=${store.lastSuccessfulRefreshAt()}")
+            android.util.Log.i("M04Migration", "schema=3; rows=${store.items()}; lastSuccess=${store.lastSuccessfulRefreshAt()}")
+        }
+    }
+
+    @Test
+    fun frozenM05FourIntentsAndLocalRowsSurviveDatabaseReopen(): Unit = runBlocking {
+        val expectedRows = listOf(
+            Item("remote-001", "Queued edit", true, 1, 1700000302000L),
+            Item("device-001", "Gamma", false, 0, 1700000300000L),
+        )
+        val expectedPending = listOf(
+            PendingMutation(1, "device-001", "CREATE", "Gamma", false),
+            PendingMutation(2, "remote-001", "RENAME", "Queued edit", null),
+            PendingMutation(3, "remote-001", "COMPLETE", null, true),
+            PendingMutation(4, "remote-002", "DELETE", null, null),
+        )
+        withDatabase { database ->
+            var time = 1700000300000L
+            val store = ItemStore(database.items(), nextId = { "device-001" }, now = { time.also { time += 1000 } })
+            store.replaceWithCanonical(listOf(
+                Item("remote-001", "Alpha", false, 1, 1700000000000L),
+                Item("remote-002", "Beta", false, 1, 1700000000000L),
+            ), 1700000300000L)
+            store.create("Gamma")
+            store.rename("remote-001", "Queued edit")
+            store.setCompleted("remote-001", true)
+            store.delete("remote-002")
+            assertEquals(expectedRows, store.items())
+            assertEquals(expectedPending, store.pendingMutations())
+        }
+        withDatabase { reopened ->
+            val store = ItemStore(reopened.items())
+            assertEquals(3, reopened.openHelper.readableDatabase.version)
+            assertEquals(expectedRows, store.items())
+            assertEquals(expectedPending, store.pendingMutations())
+            assertEquals(1700000300000L, store.lastSuccessfulRefreshAt())
+            android.util.Log.i("M05Queue", "reopenedRows=${store.items()}; pending=${store.pendingMutations()}")
+        }
+    }
+
+    @Test
+    fun pendingInsertFailureRollsBackEachLocalMutationAndItemFailureAddsNoIntent(): Unit = runBlocking {
+        withDatabase { database ->
+            val seeds = listOf(
+                Item("remote-001", "Alpha", false, 1, 1700000000000L),
+                Item("remote-002", "Beta", false, 1, 1700000000000L),
+            )
+            val store = ItemStore(database.items(), nextId = { "device-001" }, now = { 1700000300000L })
+            store.replaceWithCanonical(seeds, 1700000300000L)
+            database.openHelper.writableDatabase.execSQL(
+                "CREATE TRIGGER reject_pending BEFORE INSERT ON pending_mutations " +
+                    "BEGIN SELECT RAISE(ABORT, 'Rejected pending intent'); END",
+            )
+            suspend fun rejected(operation: String, change: suspend () -> Unit) {
+                assertThrows(android.database.sqlite.SQLiteConstraintException::class.java) {
+                    runBlocking { change() }
+                }
+                assertEquals(operation, seeds, store.items())
+                assertTrue(operation, store.pendingMutations().isEmpty())
+                assertEquals(1700000300000L, store.lastSuccessfulRefreshAt())
+                android.util.Log.i("M05Atomicity", "$operation rejected; rows and pending unchanged")
+            }
+            rejected("CREATE") { store.create("Gamma") }
+            rejected("RENAME") { store.rename("remote-001", "Queued edit") }
+            rejected("COMPLETE") { store.setCompleted("remote-001", true) }
+            rejected("DELETE") { store.delete("remote-002") }
+            database.openHelper.writableDatabase.execSQL("DROP TRIGGER reject_pending")
+            val duplicate = ItemStore(database.items(), nextId = { "remote-001" }, now = { 1700000300000L })
+            assertThrows(android.database.sqlite.SQLiteConstraintException::class.java) {
+                runBlocking { duplicate.create("Duplicate") }
+            }
+            assertEquals(seeds, store.items())
+            assertTrue(store.pendingMutations().isEmpty())
+        }
+    }
+
+    @Test
+    fun v2MigrationPreservesRowsAndTimeAndQueuesOnlyUnsentCreations(): Unit = runBlocking {
+        val rows = listOf(
+            Item("remote-001", "Alpha", false, 1, 1700000000000L),
+            Item("device-001", "Gamma", false, 0, 1700000300000L),
+        )
+        val expectedPending = listOf(PendingMutation(1, "device-001", "CREATE", "Gamma", false))
+        val path = context.getDatabasePath(name)
+        requireNotNull(path.parentFile).mkdirs()
+        SQLiteDatabase.openOrCreateDatabase(path, null).use { legacy ->
+            legacy.execSQL("CREATE TABLE IF NOT EXISTS `items` (`id` TEXT NOT NULL, " +
+                "`title` TEXT NOT NULL, `completed` INTEGER NOT NULL, `version` INTEGER NOT NULL, " +
+                "`updatedAt` INTEGER NOT NULL, PRIMARY KEY(`id`))")
+            legacy.execSQL("CREATE TABLE IF NOT EXISTS `sync_metadata` " +
+                "(`id` INTEGER NOT NULL, `lastSuccessfulRefreshAt` INTEGER NOT NULL, PRIMARY KEY(`id`))")
+            legacy.execSQL("INSERT INTO sync_metadata VALUES(1,1700000200000)")
+            legacy.execSQL("CREATE TABLE IF NOT EXISTS room_master_table (id INTEGER PRIMARY KEY,identity_hash TEXT)")
+            legacy.execSQL("INSERT OR REPLACE INTO room_master_table (id,identity_hash) " +
+                "VALUES(42, 'ab0a985408dc3a7a966bef0266c6ca21')")
+            rows.forEach { item ->
+                legacy.execSQL("INSERT INTO items (id,title,completed,version,updatedAt) VALUES(?,?,?,?,?)",
+                    arrayOf(item.id, item.title, if (item.completed) 1 else 0, item.version, item.updatedAt))
+            }
+            legacy.version = 2
+        }
+        withDatabase { migrated ->
+            val store = ItemStore(migrated.items())
+            assertEquals(3, migrated.openHelper.readableDatabase.version)
+            assertEquals(rows, store.items())
+            assertEquals(expectedPending, store.pendingMutations())
+            assertEquals(1700000200000L, store.lastSuccessfulRefreshAt())
+        }
+        withDatabase { reopened ->
+            val store = ItemStore(reopened.items())
+            assertEquals(rows, store.items())
+            assertEquals(expectedPending, store.pendingMutations())
+            assertEquals(1700000200000L, store.lastSuccessfulRefreshAt())
+            android.util.Log.i("M05Migration", "schema=3; rows=${store.items()}; pending=${store.pendingMutations()}")
         }
     }
 
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/HttpItemRemote.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/HttpItemRemote.kt
index 1460c6d..1bdc311 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/HttpItemRemote.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/HttpItemRemote.kt
@@ -22,9 +22,8 @@ class HttpItemRemote(
         return List(rows.length()) { rows.getJSONObject(it).toItem() }
     }
 
-    override suspend fun create(item: Item): Item {
-        val body = JSONObject().put("id", item.id).put("title", item.title)
-            .put("completed", item.completed)
+    override suspend fun create(id: String, title: String, completed: Boolean): Item {
+        val body = JSONObject().put("id", id).put("title", title).put("completed", completed)
         return request("POST", null, body, 201).getJSONObject("item").toItem()
     }
 
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemDatabase.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemDatabase.kt
index 6e3fdc6..1c074f8 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemDatabase.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemDatabase.kt
@@ -32,6 +32,16 @@ data class SyncMetadata(
     val lastSuccessfulRefreshAt: Long,
 )
 
+/** Payloads outlive their Item, including deletion; sequence is local FIFO order only. */
+@Entity(tableName = "pending_mutations")
+data class PendingMutation(
+    @PrimaryKey(autoGenerate = true) val sequence: Long = 0,
+    val itemId: String,
+    val operation: String,
+    val title: String? = null,
+    val completed: Boolean? = null,
+)
+
 @Dao
 interface ItemDao {
     // Preserve the M01 insertion order without adding a future metadata column.
@@ -59,6 +69,40 @@ interface ItemDao {
     @Insert(onConflict = OnConflictStrategy.REPLACE)
     suspend fun saveSyncMetadata(metadata: SyncMetadata)
 
+    @Query("SELECT * FROM pending_mutations ORDER BY sequence")
+    suspend fun readPending(): List<PendingMutation>
+
+    @Insert
+    suspend fun insertPending(mutation: PendingMutation)
+
+    @Query("DELETE FROM pending_mutations WHERE sequence = :sequence")
+    suspend fun deletePending(sequence: Long): Int
+
+    @Transaction
+    suspend fun createLocal(item: ItemEntity) {
+        insert(item)
+        insertPending(PendingMutation(itemId = item.id, operation = "CREATE",
+            title = item.title, completed = item.completed))
+    }
+
+    @Transaction
+    suspend fun renameLocal(id: String, title: String, updatedAt: Long) {
+        check(rename(id, title, updatedAt) == 1) { "Item no longer exists" }
+        insertPending(PendingMutation(itemId = id, operation = "RENAME", title = title))
+    }
+
+    @Transaction
+    suspend fun setCompletedLocal(id: String, completed: Boolean, updatedAt: Long) {
+        check(setCompleted(id, completed, updatedAt) == 1) { "Item no longer exists" }
+        insertPending(PendingMutation(itemId = id, operation = "COMPLETE", completed = completed))
+    }
+
+    @Transaction
+    suspend fun deleteLocal(id: String) {
+        check(delete(id) == 1) { "Item no longer exists" }
+        insertPending(PendingMutation(itemId = id, operation = "DELETE"))
+    }
+
     @Transaction
     suspend fun replaceAll(items: List<ItemEntity>, refreshedAt: Long? = null): List<ItemEntity> {
         deleteAll()
@@ -69,7 +113,7 @@ interface ItemDao {
     }
 }
 
-@Database(entities = [ItemEntity::class, SyncMetadata::class], version = 2, exportSchema = true)
+@Database(entities = [ItemEntity::class, SyncMetadata::class, PendingMutation::class], version = 3, exportSchema = true)
 abstract class ItemDatabase : RoomDatabase() {
     abstract fun items(): ItemDao
 
@@ -83,9 +127,21 @@ abstract class ItemDatabase : RoomDatabase() {
             }
         }
 
+        private val MIGRATION_2_3 = object : Migration(2, 3) {
+            override fun migrate(db: SupportSQLiteDatabase) {
+                db.execSQL("CREATE TABLE IF NOT EXISTS `pending_mutations` " +
+                    "(`sequence` INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL, `itemId` TEXT NOT NULL, " +
+                    "`operation` TEXT NOT NULL, `title` TEXT, `completed` INTEGER)")
+                // Preserve M03's first-sync behavior for existing unsent local creations.
+                // M04 never stored enough information to infer other interrupted edits.
+                db.execSQL("INSERT INTO pending_mutations (itemId, operation, title, completed) " +
+                    "SELECT id, 'CREATE', title, completed FROM items WHERE version = 0 ORDER BY rowid")
+            }
+        }
+
         fun open(context: Context, name: String = NAME): ItemDatabase =
             Room.databaseBuilder(context.applicationContext, ItemDatabase::class.java, name)
-                .addMigrations(MIGRATION_1_2)
+                .addMigrations(MIGRATION_1_2, MIGRATION_2_3)
                 // Preserve existing Items; reject unknown schemas instead of erasing data.
                 .build()
     }
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemStore.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemStore.kt
index fb531a1..adb48ac 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemStore.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemStore.kt
@@ -20,25 +20,31 @@ class ItemStore(
 
     suspend fun lastSuccessfulRefreshAt(): Long? = dao.lastSuccessfulRefreshAt()
 
+    suspend fun pendingMutations(): List<PendingMutation> = dao.readPending()
+
+    suspend fun acknowledge(sequence: Long) {
+        check(dao.deletePending(sequence) == 1) { "Pending change no longer exists" }
+    }
+
     suspend fun replaceWithCanonical(items: List<Item>, refreshedAt: Long? = null): List<Item> =
         dao.replaceAll(items.map(Item::toEntity), refreshedAt).map(ItemEntity::toDomain)
 
     suspend fun create(title: String) {
         val validTitle = title.trim().also { require(it.isNotEmpty()) }
-        dao.insert(Item(nextId(), validTitle, false, 0, now()).toEntity())
+        dao.createLocal(Item(nextId(), validTitle, false, 0, now()).toEntity())
     }
 
     suspend fun rename(id: String, title: String) {
         val validTitle = title.trim().also { require(it.isNotEmpty()) }
-        check(dao.rename(id, validTitle, now()) == 1) { "Item no longer exists" }
+        dao.renameLocal(id, validTitle, now())
     }
 
     suspend fun setCompleted(id: String, completed: Boolean) {
-        check(dao.setCompleted(id, completed, now()) == 1) { "Item no longer exists" }
+        dao.setCompletedLocal(id, completed, now())
     }
 
     suspend fun delete(id: String) {
-        check(dao.delete(id) == 1) { "Item no longer exists" }
+        dao.deleteLocal(id)
     }
 }
 
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemSync.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemSync.kt
index 024b238..37f7a21 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemSync.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemSync.kt
@@ -6,7 +6,7 @@ import kotlinx.coroutines.flow.asStateFlow
 
 interface ItemRemote {
     suspend fun items(): List<Item>
-    suspend fun create(item: Item): Item
+    suspend fun create(id: String, title: String, completed: Boolean): Item
     suspend fun patch(id: String, title: String?, completed: Boolean?): Item
     suspend fun delete(id: String): String
 }
@@ -19,13 +19,12 @@ data class SyncStatus(
     val error: String? = null,
 )
 
-/** One explicit foreground call at a time; this comparison snapshot is not durable intent. */
+/** One explicit foreground call drains the durable payloads in their original order. */
 class ItemSync(
     private val store: ItemStore,
     private val remote: ItemRemote,
     private val now: () -> Long = System::currentTimeMillis,
 ) {
-    private var lastSynced: Map<String, Item>? = null
     private val mutableStatus = MutableStateFlow(SyncStatus())
     val status = mutableStatus.asStateFlow()
 
@@ -41,25 +40,23 @@ class ItemSync(
     suspend fun synchronize() {
         try {
             mutableStatus.value = SyncStatus(SyncState.REFRESHING, store.lastSuccessfulRefreshAt())
-            val local = store.items().associateBy(Item::id)
-            val previous = lastSynced
-            for (item in local.values.sortedBy(Item::id)) {
-                val before = previous?.get(item.id)
-                if (before == null) {
-                    // Preserve local M02 creations on an initial sync without adding a queue.
-                    if (previous != null || item.version == 0L) remote.create(item)
-                } else {
-                    val title = item.title.takeIf { it != before.title }
-                    val completed = item.completed.takeIf { it != before.completed }
-                    if (title != null || completed != null) remote.patch(item.id, title, completed)
+            for (mutation in store.pendingMutations()) {
+                when (mutation.operation) {
+                    "CREATE" -> remote.create(mutation.itemId, requireNotNull(mutation.title),
+                        requireNotNull(mutation.completed))
+                    "RENAME" -> remote.patch(mutation.itemId, requireNotNull(mutation.title), null)
+                    "COMPLETE" -> remote.patch(mutation.itemId, null, requireNotNull(mutation.completed))
+                    "DELETE" -> remote.delete(mutation.itemId)
+                    else -> error("Unknown pending operation: ${mutation.operation}")
                 }
+                // M06 will handle the remote-success/local-ack crash window and idempotency.
+                store.acknowledge(mutation.sequence)
             }
-            previous?.keys?.filter { it !in local }?.sorted()?.forEach { remote.delete(it) }
 
             val canonical = remote.items()
             val refreshedAt = now()
             // Commit the canonical list and its timestamp together, then use the Room readback.
-            lastSynced = store.replaceWithCanonical(canonical, refreshedAt).associateBy(Item::id)
+            store.replaceWithCanonical(canonical, refreshedAt)
             mutableStatus.value = SyncStatus(SyncState.FRESH, refreshedAt)
         } catch (cancelled: CancellationException) {
             mutableStatus.value = mutableStatus.value.copy(state = SyncState.STALE, error = null)
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
index 09061c6..4dacc39 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
@@ -55,6 +55,7 @@ class MainActivity : ComponentActivity() {
 @Composable
 internal fun ItemScreen(store: ItemStore, sync: ItemSync? = null) {
     var items by remember(store) { mutableStateOf<List<Item>?>(null) }
+    var pendingCount by remember(store) { mutableStateOf(0) }
     var busy by remember(store) { mutableStateOf(true) }
     var storageError by remember(store) { mutableStateOf<String?>(null) }
     var syncing by remember(store) { mutableStateOf(false) }
@@ -67,7 +68,12 @@ internal fun ItemScreen(store: ItemStore, sync: ItemSync? = null) {
 
     suspend fun accessStorage(action: suspend () -> Unit = {}, after: () -> Unit = {}) {
         try {
-            action()
+            try {
+                action()
+            } finally {
+                // A failed refresh may follow accepted uploads; show the remaining stored work.
+                pendingCount = store.pendingMutations().size
+            }
             items = store.items()
             sync?.readSavedStatus()
             storageError = null
@@ -162,6 +168,7 @@ internal fun ItemScreen(store: ItemStore, sync: ItemSync? = null) {
             TextButton(enabled = !busy, onClick = { change(action = {}) }) { Text("Reload") }
         }
         if (items != null) {
+            Text("Pending changes: $pendingCount", modifier = Modifier.testTag("pending-count"))
             Text("Items (${rows.size})", modifier = Modifier.testTag("item-count"))
             if (rows.isEmpty()) Text("No items")
         }
diff --git a/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt b/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
index ff39907..68f9264 100644
--- a/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
+++ b/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
@@ -63,8 +63,10 @@ class ItemStoreTest {
         val store = ItemStore(FakeItemDao(), nextId = { "item-001" }, now = { 1700000006000L })
         runBlocking { store.create("Alpha edited") }
         val before = runBlocking { store.items() }
+        val pendingBefore = runBlocking { store.pendingMutations() }
         assertThrows(IllegalStateException::class.java) { runBlocking { store.create("Beta") } }
         assertEquals(before, runBlocking { store.items() })
+        assertEquals(pendingBefore, runBlocking { store.pendingMutations() })
     }
 
     @Test
@@ -171,6 +173,58 @@ class ItemStoreTest {
             remote.calls)
     }
 
+    @Test
+    fun frozenM05NewSessionDrainsFourStoredPayloadsInOrder(): Unit = runBlocking {
+        val dao = FakeItemDao()
+        var localTime = 1700000300000L
+        val store = ItemStore(dao, nextId = { "device-001" }, now = { localTime.also { localTime += 1000 } })
+        val remote = FakeRemote().apply { time = 1700000300000L }
+        val seeds = remote.rows.values.toList()
+        ItemSync(store, remote, now = { 1700000300000L }).synchronize()
+        remote.beforeMutation = { if (remote.offline) throw IOException("Forced offline") }
+        remote.offline = true
+        store.create("Gamma")
+        store.rename("remote-001", "Queued edit")
+        store.setCompleted("remote-001", true)
+        store.delete("remote-002")
+
+        val pending = listOf(
+            PendingMutation(1, "device-001", "CREATE", "Gamma", false),
+            PendingMutation(2, "remote-001", "RENAME", "Queued edit", null),
+            PendingMutation(3, "remote-001", "COMPLETE", null, true),
+            PendingMutation(4, "remote-002", "DELETE", null, null),
+        )
+        val local = listOf(
+            Item("remote-001", "Queued edit", true, 1, 1700000302000L),
+            Item("device-001", "Gamma", false, 0, 1700000300000L),
+        )
+        assertEquals(local, store.items())
+        assertEquals(pending, store.pendingMutations())
+        assertEquals(seeds, remote.rows.values.toList())
+        // A new sync/store has no comparison snapshot. Native tests prove actual durability.
+        val reopened = ItemStore(dao)
+        val sync = ItemSync(reopened, remote, now = { 1700000300000L })
+        assertEquals(local, reopened.items())
+        assertEquals(pending, reopened.pendingMutations())
+        assertThrows(IOException::class.java) { runBlocking { sync.synchronize() } }
+        assertEquals(local, reopened.items())
+        assertEquals(pending, reopened.pendingMutations())
+        assertEquals(listOf("GET"), remote.calls)
+
+        remote.offline = false
+        sync.synchronize()
+        val canonical = listOf(
+            Item("device-001", "Gamma", false, 1, 1700000300000L),
+            Item("remote-001", "Queued edit", true, 3, 1700000302000L),
+        )
+        assertEquals(canonical, reopened.items())
+        assertEquals(canonical, remote.rows.values.sortedBy(Item::id))
+        assertTrue(reopened.pendingMutations().isEmpty())
+        assertEquals(1700000304000L, remote.time)
+        assertEquals(listOf("GET", "POST device-001", "PATCH remote-001 title=Queued edit completed=null",
+            "PATCH remote-001 title=null completed=true", "DELETE remote-002", "GET"), remote.calls)
+    }
+
     private class FakeRemote : ItemRemote {
         val rows = linkedMapOf(
             "remote-001" to Item("remote-001", "Alpha", false, 1, 1700000000000L),
@@ -180,6 +234,7 @@ class ItemStoreTest {
         var offline = false
         var getFailures = 0
         var beforeGet: () -> Unit = {}
+        var beforeMutation: () -> Unit = {}
         var time = 1700000100000L
         private fun tick() = time.also { time += 1000 }
         override suspend fun items(): List<Item> {
@@ -192,18 +247,21 @@ class ItemStoreTest {
             }
             return rows.values.sortedBy(Item::id)
         }
-        override suspend fun create(item: Item): Item {
-            calls += "POST ${item.id}"
-            check(item.id !in rows)
-            return item.copy(version = 1, updatedAt = tick()).also { rows[item.id] = it }
+        override suspend fun create(id: String, title: String, completed: Boolean): Item {
+            beforeMutation()
+            calls += "POST $id"
+            check(id !in rows)
+            return Item(id, title, completed, 1, tick()).also { rows[id] = it }
         }
         override suspend fun patch(id: String, title: String?, completed: Boolean?): Item {
+            beforeMutation()
             calls += "PATCH $id title=$title completed=$completed"
             val item = rows.getValue(id)
             return item.copy(title = title ?: item.title, completed = completed ?: item.completed,
                 version = item.version + 1, updatedAt = tick()).also { rows[id] = it }
         }
         override suspend fun delete(id: String): String {
+            beforeMutation()
             calls += "DELETE $id"
             check(rows.remove(id) != null)
             tick()
@@ -214,6 +272,18 @@ class ItemStoreTest {
     private class FakeItemDao : ItemDao {
         private val rows = linkedMapOf<String, ItemEntity>()
         private var lastRefresh: Long? = null
+        private val pending = linkedMapOf<Long, PendingMutation>()
+        private var nextSequence = 1L
+
+        override suspend fun readPending(): List<PendingMutation> = pending.values.sortedBy { it.sequence }
+
+        override suspend fun insertPending(mutation: PendingMutation) {
+            val sequence = if (mutation.sequence == 0L) nextSequence++ else mutation.sequence
+            check(sequence !in pending)
+            pending[sequence] = mutation.copy(sequence = sequence)
+        }
+
+        override suspend fun deletePending(sequence: Long): Int = if (pending.remove(sequence) == null) 0 else 1
 
         override suspend fun lastSuccessfulRefreshAt(): Long? = lastRefresh
 
diff --git a/verification/M05.md b/verification/M05.md
index de14c3e..4b8f655 100644
--- a/verification/M05.md
+++ b/verification/M05.md
@@ -2,7 +2,8 @@
 
 START: `a53adb9381044303d72340dddd34c1b3db820c56` (verified phase-1 M04).
 Branch: `track/android-kotlin`. SPEC_REVISION: `61280dd86ce88b6e431f408241c0998a275960aa`.
-Baseline reproduction passed; M05 implementation/final verification is still pending.
+**PASS:** baseline reproduction, host7, fixture2, main's Android14, and the actual offline
+process restart/drain. No skipped Android method and no M05 repair.
 No repair has been requested or used for this new Thread; the maximum remains two.
 
 Raw evidence (`E`): `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/android-kotlin/M05/`.
@@ -72,3 +73,116 @@ to 0/1/1 with active default network105, all frozen source/APK hashes unchanged,
 owned fixture PID52377 stopped with SIGINT (tool session84904, exit0, PID absent, port18080
 free). The lease was released. Main owns the eventual final Android suite/restart/drain;
 the implementer will not duplicate it.
+
+## Candidate and checks before main's final run
+
+M05 replaces the volatile comparison snapshot with the durable FIFO table `pending_mutations`.
+It stores only sequence, itemId, operation, title and completed; no future mutation identity,
+baseVersion, retry metadata or background mechanism is added. Four Room transaction wrappers
+commit the local Item statement and matching payload together. Explicit Sync sends the saved
+payloads in order, removes each entry only after its response succeeds, then uses the existing
+atomic canonical-list/refresh-time replacement. Local rendering still reads Room.
+The create HTTP interface now accepts its three actual payload fields, avoiding dependence
+on a current Item that may have been edited or deleted after queuing.
+
+Schema2→3 preserves both existing tables and backfills only version-zero local creations in
+rowid order. It deliberately does not invent prior M04 edit/delete intent. The chained v1
+migration and unsupported-schema rejection remain covered; old schema exports are unchanged.
+The pending count is read from storage after an action, even when that action fails after
+acknowledging some uploads. The existing M04 focus-before-disable correction and all five
+prior UI test methods remain unchanged. No test input, timeout or fixture changes after
+the frozen baseline are used.
+
+The first candidate host/build command passed seven tests and built both APKs. Before final
+freeze, a read-only review found that an ordinary later request error could skip the UI's
+pending-count refresh after earlier accepted uploads. Only that local count read moved into
+the action's `finally` block. `fixed-build-01-before-review/` preserves the first source/APKs
+and host result, while `pending-count-review.json` and `.diff` preserve the finding and exact
+ordering correction. No runtime failure was replayed and no Android test was added or run.
+This remains initial attempt1, repair0; no final candidate was rejected.
+
+The seven host methods include all six prior regressions and the fixed four-payload M05
+new-session drain. A forced offline attempt preserves every pending payload and local row;
+one online drain then produces exact canonical timestamps/versions and an empty queue.
+The fake DAO is not a process-durability claim. Three new native methods cover Room reopen,
+a rejecting pending-insert trigger rolling back each of the four Item operations, duplicate
+Item insertion adding no intent, and schema2 migration/backfill/time preservation. Existing
+native schema expectations advance from2 to3 without removing any prior assertion. Static
+`javap` checks all nine native and five existing UI methods return void. Main then executed
+those exact fourteen methods in the final run below.
+
+| Invocation | Exit | Actual result |
+| --- | --- | --- |
+| fixed-host-build-01, `G :app:testDebugUnitTest --rerun :app:assembleDebug :app:assembleDebugAndroidTest` | 0 | Seven host methods passed, zero failures/skips; both APKs built |
+| fixture-contract-01, `python3 -m unittest discover -s fixture -v` | 0 | Both unchanged real-loopback fixture contract tests passed |
+| preserve-fixed-build-01 | 0 | First candidate source/APKs/host XML retained before the count-read correction |
+| fixed-host-build-02, same host/build command | 0 | 16.796s; seven host methods actually rerun and passed, zero failures/skips; corrected app built, test APK unchanged |
+| freeze-candidate-02 | 0 | 37 source files, source archive/copies, both APKs, host XML and all fourteen compiled method signatures frozen |
+| Main direct `am instrument` JUnit run | 0 | 14/14 pass, zero skips; 68.197s instrumentation / 71.273s command |
+| Main frozen external M05 scenario | 0 | 88.636s; 112 recorded adb calls, actual offline process replacement and exact four-operation drain |
+| receive-main-01 | 0 | All 37 frozen files and both APKs match; 292 raw main evidence files copied/hashed; native SQLite/WAL snapshots directly checked |
+
+The remaining remote-success/local-ack crash ambiguity is an M06 constraint, not an M05
+guarantee. No identity, hash, baseVersion, conflict handling, scheduler, automatic reconnect,
+push or phase-2 behavior is implemented.
+
+## Frozen final candidate and main runtime result
+
+`fixed-built-02/manifest.json` SHA-256:
+`6b38f2ceda6c770257ce14def330cb9d59d3fec762a96af14d3e0767092d754a`.
+All execution inputs remained unchanged after this freeze. The earlier runner-registration
+setup failure and both successful prefinal host/build invocations are retained above.
+
+| APK | SHA-256 |
+| --- | --- |
+| App | `0a0cf8888bef40a382171c34bc4d5ce85b2b17b7b1af49edc894743cee31f006` |
+| Tests | `298c557e63ca7531b314fda107a39e1b031a8b8f01b908ee4784e0a945303339` |
+
+Main's raw evidence is
+`/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M05/main-android-01/`;
+its independent review is `../main-android-audit.json`. Copies and per-file hashes are in
+`E/main-final-01/`, `E/main-final-audit.json` and `E/main-final-inspection.json`.
+Main installed the frozen APKs and ran the real suite directly, without another Gradle build:
+
+```sh
+ADB shell am instrument -w -r \
+  -e class 'com.mobilesystemsevolution.kotlin.ItemDatabaseTest,com.mobilesystemsevolution.kotlin.ItemUiTest' \
+  com.mobilesystemsevolution.kotlin.test/com.mobilesystemsevolution.kotlin.M05ScenarioInstrumentation
+```
+
+Here `ADB` is the fixed SDK adb executable with `-s emulator-5554`. Raw `junit.stdout`
+records fourteen starts and fourteen successful completions, `OK (14 tests)` and final
+instrumentation code-1; no method is skipped. The complete logcat records all four rejected
+pending inserts with unchanged rows/queue, native reopen/migration results, and the unchanged
+five M01–M04 UI regressions. No duplicate owner-side final Android suite was run.
+
+Main then executed the unchanged frozen external harness with `--expect durable
+--schema-version 3`. Initial installs/clear/fixed-ID launcher precede the death boundary.
+All four UI mutations complete before PID **11605 → absent → 12161**. Wi-Fi/mobile remain
+0/0 with no active default network across force-stop and normal Activity relaunch; no
+instrumentation, native seed, install or clear occurs after that boundary. The expected
+launcher process termination is recorded separately and is not an additional JUnit pass.
+
+Direct reads of disposable native SQLite/WAL copies confirm schema3, the unchanged five
+Item fields, all four ordered payloads, and exact row/refresh metadata equality before and
+after restart. The relaunch screenshot visibly shows pending4. One foreground Sync then
+accepts exactly POST Gamma, PATCH title, PATCH completion, DELETE Beta; their canonical
+responses and the final next timestamp1700000304000 remain in the raw fixture trace.
+
+| Final Item | Title | Completed | Version | updatedAt |
+| --- | --- | --- | --- | --- |
+| device-001 | Gamma | false | 1 | 1700000300000 |
+| remote-001 | Queued edit | true | 3 | 1700000302000 |
+
+Remote and native Room match these exact rows, Beta is absent and the pending queue is empty.
+The after-drain screenshot visibly shows pending0; both final screenshots were inspected.
+Main stopped owned fixture PID59447 with exit0; its PID is absent and port18080 is free.
+The app is absent and network0/1/1 is restored with active network107. Main's read-only
+fixture-PID audit first hit sandbox EPERM, then confirmed absence with an approved `ps`;
+this was not a test failure or repeated runtime execution.
+
+After receiving PASS, the implementer verified all 37 source files and both APKs against
+the frozen manifest before updating only TRACK and this report. The final product/tests/
+schema commit contains the tested bytes, with contiguous M05/phase-1/original frozen revision
+trailers. The baseline commit and all earlier evidence remain intact. No build, device run,
+new repair, successor implementation, tag, history rewrite or remote push occurs in finalization.
