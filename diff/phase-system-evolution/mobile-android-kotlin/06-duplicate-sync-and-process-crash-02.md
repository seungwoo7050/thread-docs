## `M06 preserve failed Android candidate before matcher repair`

diff --git a/app/schemas/com.mobilesystemsevolution.kotlin.ItemDatabase/4.json b/app/schemas/com.mobilesystemsevolution.kotlin.ItemDatabase/4.json
new file mode 100644
index 0000000..7243b6c
--- /dev/null
+++ b/app/schemas/com.mobilesystemsevolution.kotlin.ItemDatabase/4.json
@@ -0,0 +1,184 @@
+{
+  "formatVersion": 1,
+  "database": {
+    "version": 4,
+    "identityHash": "ec430a1d472230ac967db0ce7b482bcb",
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
+        "createSql": "CREATE TABLE IF NOT EXISTS `${TABLE_NAME}` (`sequence` INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL, `itemId` TEXT NOT NULL, `operation` TEXT NOT NULL, `title` TEXT, `completed` INTEGER, `clientMutationId` TEXT NOT NULL, `payloadHash` TEXT NOT NULL, `terminalError` TEXT)",
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
+          },
+          {
+            "fieldPath": "clientMutationId",
+            "columnName": "clientMutationId",
+            "affinity": "TEXT",
+            "notNull": true
+          },
+          {
+            "fieldPath": "payloadHash",
+            "columnName": "payloadHash",
+            "affinity": "TEXT",
+            "notNull": true
+          },
+          {
+            "fieldPath": "terminalError",
+            "columnName": "terminalError",
+            "affinity": "TEXT",
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
+      },
+      {
+        "tableName": "acknowledged_mutations",
+        "createSql": "CREATE TABLE IF NOT EXISTS `${TABLE_NAME}` (`clientMutationId` TEXT NOT NULL, `payloadHash` TEXT NOT NULL, `statusCode` INTEGER NOT NULL, `responseBody` TEXT NOT NULL, PRIMARY KEY(`clientMutationId`))",
+        "fields": [
+          {
+            "fieldPath": "clientMutationId",
+            "columnName": "clientMutationId",
+            "affinity": "TEXT",
+            "notNull": true
+          },
+          {
+            "fieldPath": "payloadHash",
+            "columnName": "payloadHash",
+            "affinity": "TEXT",
+            "notNull": true
+          },
+          {
+            "fieldPath": "statusCode",
+            "columnName": "statusCode",
+            "affinity": "INTEGER",
+            "notNull": true
+          },
+          {
+            "fieldPath": "responseBody",
+            "columnName": "responseBody",
+            "affinity": "TEXT",
+            "notNull": true
+          }
+        ],
+        "primaryKey": {
+          "autoGenerate": false,
+          "columnNames": [
+            "clientMutationId"
+          ]
+        },
+        "indices": [],
+        "foreignKeys": []
+      }
+    ],
+    "views": [],
+    "setupQueries": [
+      "CREATE TABLE IF NOT EXISTS room_master_table (id INTEGER PRIMARY KEY,identity_hash TEXT)",
+      "INSERT OR REPLACE INTO room_master_table (id,identity_hash) VALUES(42, 'ec430a1d472230ac967db0ce7b482bcb')"
+    ]
+  }
+}
\ No newline at end of file
diff --git a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemDatabaseTest.kt b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemDatabaseTest.kt
index 3242b08..43d9792 100644
--- a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemDatabaseTest.kt
+++ b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemDatabaseTest.kt
@@ -53,7 +53,7 @@ class ItemDatabaseTest {
         }
         withDatabase { reopened ->
             assertEquals(expected, ItemStore(reopened.items()).items())
-            assertEquals(3, reopened.openHelper.readableDatabase.version)
+            assertEquals(4, reopened.openHelper.readableDatabase.version)
         }
     }
 
@@ -133,7 +133,7 @@ class ItemDatabaseTest {
         withDatabase { migrated ->
             val store = ItemStore(migrated.items())
             assertEquals(seeds, store.items())
-            assertEquals(3, migrated.openHelper.readableDatabase.version)
+            assertEquals(4, migrated.openHelper.readableDatabase.version)
             assertNull(store.lastSuccessfulRefreshAt())
             store.replaceWithCanonical(seeds, 1700000200000L)
         }
@@ -141,7 +141,7 @@ class ItemDatabaseTest {
             val store = ItemStore(reopened.items())
             assertEquals(seeds, store.items())
             assertEquals(1700000200000L, store.lastSuccessfulRefreshAt())
-            android.util.Log.i("M04Migration", "schema=3; rows=${store.items()}; lastSuccess=${store.lastSuccessfulRefreshAt()}")
+            android.util.Log.i("M04Migration", "schema=4; rows=${store.items()}; lastSuccess=${store.lastSuccessfulRefreshAt()}")
         }
     }
 
@@ -152,14 +152,16 @@ class ItemDatabaseTest {
             Item("device-001", "Gamma", false, 0, 1700000300000L),
         )
         val expectedPending = listOf(
-            PendingMutation(1, "device-001", "CREATE", "Gamma", false),
-            PendingMutation(2, "remote-001", "RENAME", "Queued edit", null),
-            PendingMutation(3, "remote-001", "COMPLETE", null, true),
-            PendingMutation(4, "remote-002", "DELETE", null, null),
+            pendingMutation("device-001", "CREATE", "Gamma", false, "m05-1", 1),
+            pendingMutation("remote-001", "RENAME", "Queued edit", null, "m05-2", 2),
+            pendingMutation("remote-001", "COMPLETE", null, true, "m05-3", 3),
+            pendingMutation("remote-002", "DELETE", null, null, "m05-4", 4),
         )
         withDatabase { database ->
             var time = 1700000300000L
-            val store = ItemStore(database.items(), nextId = { "device-001" }, now = { time.also { time += 1000 } })
+            var mutationId = 0
+            val store = ItemStore(database.items(), nextId = { "device-001" }, now = { time.also { time += 1000 } },
+                nextMutationId = { "m05-${++mutationId}" })
             store.replaceWithCanonical(listOf(
                 Item("remote-001", "Alpha", false, 1, 1700000000000L),
                 Item("remote-002", "Beta", false, 1, 1700000000000L),
@@ -173,7 +175,7 @@ class ItemDatabaseTest {
         }
         withDatabase { reopened ->
             val store = ItemStore(reopened.items())
-            assertEquals(3, reopened.openHelper.readableDatabase.version)
+            assertEquals(4, reopened.openHelper.readableDatabase.version)
             assertEquals(expectedRows, store.items())
             assertEquals(expectedPending, store.pendingMutations())
             assertEquals(1700000300000L, store.lastSuccessfulRefreshAt())
@@ -223,7 +225,8 @@ class ItemDatabaseTest {
             Item("remote-001", "Alpha", false, 1, 1700000000000L),
             Item("device-001", "Gamma", false, 0, 1700000300000L),
         )
-        val expectedPending = listOf(PendingMutation(1, "device-001", "CREATE", "Gamma", false))
+        val expectedPayload = listOf(listOf(1L, "device-001", "CREATE", "Gamma", false))
+        var identifiedPending = emptyList<PendingMutation>()
         val path = context.getDatabasePath(name)
         requireNotNull(path.parentFile).mkdirs()
         SQLiteDatabase.openOrCreateDatabase(path, null).use { legacy ->
@@ -244,17 +247,125 @@ class ItemDatabaseTest {
         }
         withDatabase { migrated ->
             val store = ItemStore(migrated.items())
-            assertEquals(3, migrated.openHelper.readableDatabase.version)
+            assertEquals(4, migrated.openHelper.readableDatabase.version)
             assertEquals(rows, store.items())
-            assertEquals(expectedPending, store.pendingMutations())
+            identifiedPending = store.pendingMutations()
+            assertEquals(expectedPayload, identifiedPending.map { listOf(it.sequence, it.itemId, it.operation, it.title, it.completed) })
+            assertTrue(identifiedPending.single().clientMutationId.isNotBlank())
+            assertEquals(identifiedPending.single().request().hash(), identifiedPending.single().payloadHash)
             assertEquals(1700000200000L, store.lastSuccessfulRefreshAt())
         }
         withDatabase { reopened ->
             val store = ItemStore(reopened.items())
             assertEquals(rows, store.items())
-            assertEquals(expectedPending, store.pendingMutations())
+            assertEquals(identifiedPending, store.pendingMutations())
             assertEquals(1700000200000L, store.lastSuccessfulRefreshAt())
-            android.util.Log.i("M05Migration", "schema=3; rows=${store.items()}; pending=${store.pendingMutations()}")
+            android.util.Log.i("M05Migration", "schema=4; rows=${store.items()}; pending=${store.pendingMutations()}")
+        }
+    }
+
+    @Test
+    fun m06AcknowledgmentAndExactDequeueRollbackTogetherAndSurviveReopen(): Unit = runBlocking {
+        val result = MutationResult(201,
+            "{\"item\":{\"id\":\"crash-001\",\"title\":\"Crash safe\",\"completed\":false,\"version\":1,\"updatedAt\":1700000400000}}")
+        var pending = emptyList<PendingMutation>()
+        var local = emptyList<Item>()
+        withDatabase { database ->
+            val identities = listOf("m06-create-001", "m06-rename-001").iterator()
+            val store = ItemStore(database.items(), nextId = { "crash-001" }, now = { 1700000400000L },
+                nextMutationId = { identities.next() })
+            store.create("Crash safe")
+            store.rename("crash-001", "Later local edit")
+            pending = store.pendingMutations()
+            local = store.items()
+            val sql = database.openHelper.writableDatabase
+            for ((table, event) in listOf("acknowledged_mutations" to "INSERT", "pending_mutations" to "DELETE")) {
+                sql.execSQL("CREATE TRIGGER reject_ack BEFORE $event ON $table " +
+                    "BEGIN SELECT RAISE(ABORT, 'Rejected acknowledgment transaction'); END")
+                assertThrows(android.database.sqlite.SQLiteConstraintException::class.java) {
+                    runBlocking { store.acknowledge(pending.first(), result) }
+                }
+                assertEquals(pending, store.pendingMutations())
+                assertTrue(store.acknowledgedMutations().isEmpty())
+                assertEquals(local, store.items())
+                sql.execSQL("DROP TRIGGER reject_ack")
+                android.util.Log.i("M06Atomicity", "$event $table rejected; original pending and rows retained; no receipt")
+            }
+            assertThrows(IllegalStateException::class.java) {
+                runBlocking { store.acknowledge(pending.first().copy(payloadHash = "0".repeat(64)), result) }
+            }
+            assertEquals(pending, store.pendingMutations())
+            assertTrue(store.acknowledgedMutations().isEmpty())
+            store.acknowledge(pending.first(), result)
+            assertEquals(pending.drop(1), store.pendingMutations())
+            assertEquals(local, store.items()) // Recording the old CREATE reply cannot erase the later edit.
+        }
+        withDatabase { reopened ->
+            val store = ItemStore(reopened.items())
+            assertEquals(pending.drop(1), store.pendingMutations())
+            assertEquals(local, store.items())
+            assertEquals(listOf(AcknowledgedMutation("m06-create-001", pending.first().payloadHash,
+                result.statusCode, result.responseBody)), store.acknowledgedMutations())
+            android.util.Log.i("M06Atomicity", "reopened pending=${store.pendingMutations()}; acknowledged=${store.acknowledgedMutations()}")
+        }
+    }
+
+    @Test
+    fun m06V3MigrationPreservesPayloadsIdentitiesAndNonemptyOrEmptyAllocator(): Unit = runBlocking {
+        val legacyPayloads = listOf(
+            listOf(2L, "device-001", "CREATE", "Gamma", false),
+            listOf(4L, "remote-001", "RENAME", "Queued edit", null),
+            listOf(6L, "remote-001", "COMPLETE", null, true),
+            listOf(8L, "remote-002", "DELETE", null, null),
+        )
+        for (hasPending in listOf(true, false)) {
+            context.deleteDatabase(name)
+            val path = context.getDatabasePath(name)
+            requireNotNull(path.parentFile).mkdirs()
+            SQLiteDatabase.openOrCreateDatabase(path, null).use { legacy ->
+                legacy.execSQL("CREATE TABLE items (id TEXT NOT NULL PRIMARY KEY, title TEXT NOT NULL, completed INTEGER NOT NULL, " +
+                    "version INTEGER NOT NULL, updatedAt INTEGER NOT NULL)")
+                legacy.execSQL("INSERT INTO items VALUES('device-001','Newer local title',1,0,1700000400000)")
+                legacy.execSQL("CREATE TABLE sync_metadata (id INTEGER NOT NULL PRIMARY KEY, lastSuccessfulRefreshAt INTEGER NOT NULL)")
+                legacy.execSQL("INSERT INTO sync_metadata VALUES(1,1700000300000)")
+                legacy.execSQL("CREATE TABLE pending_mutations (sequence INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL, " +
+                    "itemId TEXT NOT NULL, operation TEXT NOT NULL, title TEXT, completed INTEGER)")
+                if (hasPending) legacyPayloads.forEach { row ->
+                    legacy.execSQL("INSERT INTO pending_mutations VALUES(?,?,?,?,?)",
+                        arrayOf(row[0], row[1], row[2], row[3], (row[4] as Boolean?)?.let { if (it) 1 else 0 }))
+                }
+                legacy.execSQL("INSERT INTO pending_mutations VALUES(12,'removed','DELETE',NULL,NULL)")
+                legacy.execSQL("DELETE FROM pending_mutations WHERE sequence=12")
+                legacy.version = 3
+            }
+            var preserved = emptyList<PendingMutation>()
+            withDatabase { database ->
+                val store = ItemStore(database.items(), nextId = { "new-001" }, now = { 1700000401000L },
+                    nextMutationId = { "m06-after-migration" })
+                assertEquals(4, database.openHelper.readableDatabase.version)
+                val queued = store.pendingMutations()
+                assertEquals(if (hasPending) legacyPayloads else emptyList<List<Any?>>(),
+                    queued.map { listOf(it.sequence, it.itemId, it.operation, it.title, it.completed) })
+                assertEquals(queued.size, queued.map { it.clientMutationId }.toSet().size)
+                queued.forEach {
+                    assertEquals(it.clientMutationId, java.util.UUID.fromString(it.clientMutationId).toString())
+                    assertEquals(it.request().hash(), it.payloadHash)
+                    assertNull(it.terminalError)
+                }
+                assertEquals(listOf(Item("device-001", "Newer local title", true, 0, 1700000400000L)), store.items())
+                assertEquals(1700000300000L, store.lastSuccessfulRefreshAt())
+                assertTrue(store.acknowledgedMutations().isEmpty())
+                store.create("After migration")
+                preserved = store.pendingMutations()
+                assertEquals(13L, preserved.last().sequence)
+                assertEquals(queued, preserved.dropLast(1))
+            }
+            withDatabase { reopened ->
+                val store = ItemStore(reopened.items())
+                assertEquals(preserved, store.pendingMutations())
+                assertEquals(1700000300000L, store.lastSuccessfulRefreshAt())
+                android.util.Log.i("M06Migration", "legacyPending=$hasPending; reopened=${store.pendingMutations()}; allocator=13")
+            }
         }
     }
 
diff --git a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt
index 23d8c8d..1e0c763 100644
--- a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt
+++ b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt
@@ -517,4 +517,67 @@ class ItemUiTest {
         }
     }
 
+    @Test
+    fun m06ActualHttpCollisionStaysVisibleAndTerminalAfterRoomReopen() {
+        fixtureControl("POST", "__m06/reset", JSONObject().put("dropFirstCreate", false))
+        val original = runBlocking {
+            HttpItemRemote().send(pendingMutation("crash-001", "CREATE", "Crash safe", false, "m06-create-001"))
+        }
+        assertEquals(201, original.statusCode)
+        val context = InstrumentationRegistry.getInstrumentation().targetContext
+        val name = "m06-identity-collision.db"
+        context.deleteDatabase(name)
+        var database = ItemDatabase.open(context, name)
+        try {
+            val store = ItemStore(database.items(), nextId = { "crash-001" }, now = { 1700000400000L },
+                nextMutationId = { "m06-create-001" })
+            val sync = ItemSync(store, HttpItemRemote())
+            compose.runOnUiThread { compose.activity.setContent { MaterialTheme { ItemScreen(store, sync) } } }
+            awaitCrudText("Items (0)")
+            compose.onNodeWithTag("item-title-input").performTextInput("Different payload")
+            compose.onNodeWithText("Add").performClick()
+            awaitCrudText("Different payload")
+            compose.onNodeWithText("Sync").performClick()
+            awaitText("Stale local data · sync error")
+            awaitCrudText("Different payload")
+            compose.onNodeWithTag("sync-error").assertIsDisplayed().assertTextContains("identity_conflict")
+            compose.onNodeWithTag("pending-count").assertTextEquals("Pending changes: 1")
+            val pending = runBlocking { store.pendingMutations().single() }
+            assertEquals("identity_conflict", pending.terminalError)
+            assertEquals("3578ce895fdfea0354ac912266ee46823b075efb9a4f54f07333fe6f101bda73", pending.payloadHash)
+            assertEquals(emptyList<AcknowledgedMutation>(), runBlocking { store.acknowledgedMutations() })
+
+            compose.runOnUiThread { compose.activity.setContent {} }
+            compose.waitForIdle()
+            database.close()
+            database = ItemDatabase.open(context, name)
+            val reopened = ItemStore(database.items())
+            val resumed = ItemSync(reopened, HttpItemRemote())
+            compose.runOnUiThread { compose.activity.setContent { MaterialTheme { ItemScreen(reopened, resumed) } } }
+            awaitText("Stale local data · sync error")
+            awaitCrudText("Different payload")
+            compose.onNodeWithTag("sync-error").assertIsDisplayed().assertTextContains("identity_conflict")
+            compose.onNodeWithText("Sync").performClick()
+            awaitCrudText("Different payload")
+            assertEquals(listOf(pending), runBlocking { reopened.pendingMutations() })
+            assertEquals(SyncState.ERROR, resumed.status.value.state)
+            val remote = fixtureControl("GET", "__m06")
+            assertEquals(2, remote.getInt("mutationRequests")) // Initial create and one rejected collision only.
+            assertEquals(1, remote.getInt("applied"))
+            assertEquals(1, remote.getInt("identityConflicts"))
+            assertEquals(0, remote.getInt("duplicates"))
+            assertEquals(1700000401000L, remote.getLong("nextTimestamp"))
+            assertEquals(JSONObject(original.responseBody).getJSONObject("item").toString(),
+                remote.getJSONArray("items").getJSONObject(0).toString())
+            assertEquals(1, remote.getJSONArray("items").length())
+            assertEquals(0, fixtureControl("GET", "__control").getInt("getRequests"))
+            android.util.Log.i("M06Collision", "visible terminal=$pending; actual fixture=$remote; no resend after Room reopen")
+        } finally {
+            compose.runOnUiThread { compose.activity.setContent {} }
+            compose.waitForIdle()
+            database.close()
+            context.deleteDatabase(name)
+        }
+    }
+
 }
diff --git a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/M05ScenarioInstrumentation.kt b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/M05ScenarioInstrumentation.kt
index 8e3edf5..fb01161 100644
--- a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/M05ScenarioInstrumentation.kt
+++ b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/M05ScenarioInstrumentation.kt
@@ -6,6 +6,7 @@ import android.os.Bundle
 import androidx.activity.compose.setContent
 import androidx.compose.material3.MaterialTheme
 import androidx.test.runner.AndroidJUnitRunner
+import java.util.UUID
 import java.util.concurrent.CountDownLatch
 import java.util.concurrent.TimeUnit
 
@@ -35,7 +36,8 @@ class M05ScenarioInstrumentation : AndroidJUnitRunner() {
             val initialTime = if (m06) 1_700_000_400_000L else 1_700_000_300_000L
             var localTime = initialTime
             val store = ItemStore(opened.items(), nextId = { if (m06) "crash-001" else "device-001" },
-                now = { localTime.also { localTime += 1_000 } })
+                now = { localTime.also { localTime += 1_000 } },
+                nextMutationId = { if (m06) "m06-create-001" else UUID.randomUUID().toString() })
             val sync = ItemSync(store, HttpItemRemote(), now = { initialTime })
             runOnMainSync { activity.setContent { MaterialTheme { ItemScreen(store, sync) } } }
             waitForIdleSync()
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/HttpItemRemote.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/HttpItemRemote.kt
index 1bdc311..9e04375 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/HttpItemRemote.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/HttpItemRemote.kt
@@ -18,31 +18,26 @@ class HttpItemRemote(
     private val base = baseUrl.toHttpUrl()
 
     override suspend fun items(): List<Item> {
-        val rows = request("GET", null, null, 200).getJSONArray("items")
+        val rows = JSONObject(request("GET", "/items", null, 200).responseBody).getJSONArray("items")
         return List(rows.length()) { rows.getJSONObject(it).toItem() }
     }
 
-    override suspend fun create(id: String, title: String, completed: Boolean): Item {
-        val body = JSONObject().put("id", id).put("title", title).put("completed", completed)
-        return request("POST", null, body, 201).getJSONObject("item").toItem()
-    }
-
-    override suspend fun patch(id: String, title: String?, completed: Boolean?): Item {
+    override suspend fun send(mutation: PendingMutation): MutationResult {
+        val stored = mutation.request()
         val body = JSONObject()
-        title?.let { body.put("title", it) }
-        completed?.let { body.put("completed", it) }
-        return request("PATCH", id, body, 200).getJSONObject("item").toItem()
+        stored.payload?.forEach { (key, value) -> body.put(key, value) }
+        body.put("clientMutationId", mutation.clientMutationId).put("payloadHash", mutation.payloadHash)
+        val result = request(stored.method, stored.path, body, if (stored.method == "POST") 201 else 200)
+        val response = JSONObject(result.responseBody)
+        val responseId = if (stored.method == "DELETE") response.getString("deletedId")
+                         else response.getJSONObject("item").toItem().id
+        if (responseId != mutation.itemId) throw IOException("Unexpected acknowledged Item identity")
+        return result
     }
 
-    override suspend fun delete(id: String): String =
-        request("DELETE", id, null, 200).getString("deletedId").also {
-            if (it != id) throw IOException("Unexpected deleted Item identity")
-        }
-
-    private suspend fun request(method: String, id: String?, body: JSONObject?, status: Int): JSONObject =
+    private suspend fun request(method: String, path: String, body: JSONObject?, status: Int): MutationResult =
         withContext(Dispatchers.IO) {
-            val url = base.newBuilder().addPathSegment("items")
-                .apply { if (id != null) addPathSegment(id) }.build()
+            val url = base.newBuilder().encodedPath(path).build()
             val request = Request.Builder().url(url)
                 // This HTTP/1.0 fixture closes each connection; do not pool a closed socket.
                 // Keep retries disabled rather than resending a mutation after an EOF.
@@ -50,8 +45,12 @@ class HttpItemRemote(
                 .method(method, body?.toString()?.toRequestBody("application/json".toMediaType()))
                 .build()
             client.newCall(request).execute().use { response ->
+                val raw = response.body?.string() ?: throw IOException("Empty sync response")
+                if (response.code == 409 && JSONObject(raw).optString("error") == "identity_conflict") {
+                    throw MutationIdentityConflict()
+                }
                 if (response.code != status) throw IOException("Sync HTTP ${response.code}")
-                JSONObject(response.body?.string() ?: throw IOException("Empty sync response"))
+                MutationResult(response.code, raw)
             }
         }
 
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemDatabase.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemDatabase.kt
index 1c074f8..582497c 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemDatabase.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemDatabase.kt
@@ -13,6 +13,7 @@ import androidx.room.RoomDatabase
 import androidx.room.Transaction
 import androidx.room.migration.Migration
 import androidx.sqlite.db.SupportSQLiteDatabase
+import java.util.UUID
 
 @Entity(tableName = "items")
 data class ItemEntity(
@@ -40,6 +41,17 @@ data class PendingMutation(
     val operation: String,
     val title: String? = null,
     val completed: Boolean? = null,
+    val clientMutationId: String,
+    val payloadHash: String,
+    val terminalError: String? = null,
+)
+
+@Entity(tableName = "acknowledged_mutations")
+data class AcknowledgedMutation(
+    @PrimaryKey val clientMutationId: String,
+    val payloadHash: String,
+    val statusCode: Int,
+    val responseBody: String,
 )
 
 @Dao
@@ -75,32 +87,51 @@ interface ItemDao {
     @Insert
     suspend fun insertPending(mutation: PendingMutation)
 
-    @Query("DELETE FROM pending_mutations WHERE sequence = :sequence")
-    suspend fun deletePending(sequence: Long): Int
+    @Query("DELETE FROM pending_mutations WHERE sequence = :sequence AND clientMutationId = :clientMutationId " +
+        "AND payloadHash = :payloadHash AND terminalError IS NULL")
+    suspend fun deletePending(sequence: Long, clientMutationId: String, payloadHash: String): Int
+
+    @Insert
+    suspend fun insertAcknowledged(result: AcknowledgedMutation)
+
+    @Query("SELECT * FROM acknowledged_mutations ORDER BY clientMutationId")
+    suspend fun readAcknowledged(): List<AcknowledgedMutation>
 
     @Transaction
-    suspend fun createLocal(item: ItemEntity) {
+    suspend fun acknowledge(sequence: Long, result: AcknowledgedMutation) {
+        // Record the original success without overwriting later queued local Item edits.
+        insertAcknowledged(result)
+        check(deletePending(sequence, result.clientMutationId, result.payloadHash) == 1) {
+            "Pending change no longer matches acknowledgment"
+        }
+    }
+
+    @Query("UPDATE pending_mutations SET terminalError = 'identity_conflict' " +
+        "WHERE sequence = :sequence AND clientMutationId = :clientMutationId AND payloadHash = :payloadHash")
+    suspend fun markIdentityConflict(sequence: Long, clientMutationId: String, payloadHash: String): Int
+
+    @Transaction
+    suspend fun createLocal(item: ItemEntity, mutation: PendingMutation) {
         insert(item)
-        insertPending(PendingMutation(itemId = item.id, operation = "CREATE",
-            title = item.title, completed = item.completed))
+        insertPending(mutation)
     }
 
     @Transaction
-    suspend fun renameLocal(id: String, title: String, updatedAt: Long) {
+    suspend fun renameLocal(id: String, title: String, updatedAt: Long, mutation: PendingMutation) {
         check(rename(id, title, updatedAt) == 1) { "Item no longer exists" }
-        insertPending(PendingMutation(itemId = id, operation = "RENAME", title = title))
+        insertPending(mutation)
     }
 
     @Transaction
-    suspend fun setCompletedLocal(id: String, completed: Boolean, updatedAt: Long) {
+    suspend fun setCompletedLocal(id: String, completed: Boolean, updatedAt: Long, mutation: PendingMutation) {
         check(setCompleted(id, completed, updatedAt) == 1) { "Item no longer exists" }
-        insertPending(PendingMutation(itemId = id, operation = "COMPLETE", completed = completed))
+        insertPending(mutation)
     }
 
     @Transaction
-    suspend fun deleteLocal(id: String) {
+    suspend fun deleteLocal(id: String, mutation: PendingMutation) {
         check(delete(id) == 1) { "Item no longer exists" }
-        insertPending(PendingMutation(itemId = id, operation = "DELETE"))
+        insertPending(mutation)
     }
 
     @Transaction
@@ -113,7 +144,8 @@ interface ItemDao {
     }
 }
 
-@Database(entities = [ItemEntity::class, SyncMetadata::class, PendingMutation::class], version = 3, exportSchema = true)
+@Database(entities = [ItemEntity::class, SyncMetadata::class, PendingMutation::class, AcknowledgedMutation::class],
+    version = 4, exportSchema = true)
 abstract class ItemDatabase : RoomDatabase() {
     abstract fun items(): ItemDao
 
@@ -139,9 +171,43 @@ abstract class ItemDatabase : RoomDatabase() {
             }
         }
 
+        private val MIGRATION_3_4 = object : Migration(3, 4) {
+            override fun migrate(db: SupportSQLiteDatabase) {
+                val highwater = db.query("SELECT seq FROM sqlite_sequence WHERE name = 'pending_mutations'").use {
+                    if (it.moveToFirst()) it.getLong(0) else 0L
+                }
+                db.execSQL("CREATE TABLE `pending_mutations_new` " +
+                    "(`sequence` INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL, `itemId` TEXT NOT NULL, " +
+                    "`operation` TEXT NOT NULL, `title` TEXT, `completed` INTEGER, " +
+                    "`clientMutationId` TEXT NOT NULL, `payloadHash` TEXT NOT NULL, `terminalError` TEXT)")
+                db.query("SELECT sequence,itemId,operation,title,completed FROM pending_mutations ORDER BY sequence").use { rows ->
+                    while (rows.moveToNext()) {
+                        val mutation = pendingMutation(rows.getString(1), rows.getString(2),
+                            if (rows.isNull(3)) null else rows.getString(3),
+                            if (rows.isNull(4)) null else rows.getInt(4) != 0,
+                            UUID.randomUUID().toString(), rows.getLong(0))
+                        db.execSQL("INSERT INTO pending_mutations_new " +
+                            "(sequence,itemId,operation,title,completed,clientMutationId,payloadHash,terminalError) " +
+                            "VALUES(?,?,?,?,?,?,?,NULL)", arrayOf(mutation.sequence, mutation.itemId, mutation.operation,
+                            mutation.title, mutation.completed?.let { if (it) 1 else 0 },
+                            mutation.clientMutationId, mutation.payloadHash))
+                    }
+                }
+                db.execSQL("DROP TABLE pending_mutations")
+                db.execSQL("ALTER TABLE pending_mutations_new RENAME TO pending_mutations")
+                // Preserve FIFO allocation even when all previous high sequences were deleted.
+                db.execSQL("INSERT INTO sqlite_sequence(name,seq) SELECT 'pending_mutations', ? " +
+                    "WHERE NOT EXISTS (SELECT 1 FROM sqlite_sequence WHERE name = 'pending_mutations')", arrayOf(highwater))
+                db.execSQL("UPDATE sqlite_sequence SET seq = max(seq, ?) WHERE name = 'pending_mutations'", arrayOf(highwater))
+                db.execSQL("CREATE TABLE `acknowledged_mutations` " +
+                    "(`clientMutationId` TEXT NOT NULL, `payloadHash` TEXT NOT NULL, `statusCode` INTEGER NOT NULL, " +
+                    "`responseBody` TEXT NOT NULL, PRIMARY KEY(`clientMutationId`))")
+            }
+        }
+
         fun open(context: Context, name: String = NAME): ItemDatabase =
             Room.databaseBuilder(context.applicationContext, ItemDatabase::class.java, name)
-                .addMigrations(MIGRATION_1_2, MIGRATION_2_3)
+                .addMigrations(MIGRATION_1_2, MIGRATION_2_3, MIGRATION_3_4)
                 // Preserve existing Items; reject unknown schemas instead of erasing data.
                 .build()
     }
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemStore.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemStore.kt
index adb48ac..fbde32e 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemStore.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemStore.kt
@@ -15,6 +15,7 @@ class ItemStore(
     private val dao: ItemDao,
     private val nextId: () -> String = { UUID.randomUUID().toString() },
     private val now: () -> Long = System::currentTimeMillis,
+    private val nextMutationId: () -> String = { UUID.randomUUID().toString() },
 ) {
     suspend fun items(): List<Item> = dao.readAll().map(ItemEntity::toDomain)
 
@@ -22,8 +23,17 @@ class ItemStore(
 
     suspend fun pendingMutations(): List<PendingMutation> = dao.readPending()
 
-    suspend fun acknowledge(sequence: Long) {
-        check(dao.deletePending(sequence) == 1) { "Pending change no longer exists" }
+    suspend fun acknowledgedMutations(): List<AcknowledgedMutation> = dao.readAcknowledged()
+
+    suspend fun acknowledge(mutation: PendingMutation, result: MutationResult) {
+        dao.acknowledge(mutation.sequence, AcknowledgedMutation(mutation.clientMutationId,
+            mutation.payloadHash, result.statusCode, result.responseBody))
+    }
+
+    suspend fun markIdentityConflict(mutation: PendingMutation) {
+        check(dao.markIdentityConflict(mutation.sequence, mutation.clientMutationId, mutation.payloadHash) == 1) {
+            "Pending identity conflict no longer exists"
+        }
     }
 
     suspend fun replaceWithCanonical(items: List<Item>, refreshedAt: Long? = null): List<Item> =
@@ -31,20 +41,23 @@ class ItemStore(
 
     suspend fun create(title: String) {
         val validTitle = title.trim().also { require(it.isNotEmpty()) }
-        dao.createLocal(Item(nextId(), validTitle, false, 0, now()).toEntity())
+        val item = Item(nextId(), validTitle, false, 0, now()).toEntity()
+        dao.createLocal(item, pendingMutation(item.id, "CREATE", item.title, item.completed, nextMutationId()))
     }
 
     suspend fun rename(id: String, title: String) {
         val validTitle = title.trim().also { require(it.isNotEmpty()) }
-        dao.renameLocal(id, validTitle, now())
+        dao.renameLocal(id, validTitle, now(), pendingMutation(id, "RENAME", title = validTitle,
+            clientMutationId = nextMutationId()))
     }
 
     suspend fun setCompleted(id: String, completed: Boolean) {
-        dao.setCompletedLocal(id, completed, now())
+        dao.setCompletedLocal(id, completed, now(), pendingMutation(id, "COMPLETE", completed = completed,
+            clientMutationId = nextMutationId()))
     }
 
     suspend fun delete(id: String) {
-        dao.deleteLocal(id)
+        dao.deleteLocal(id, pendingMutation(id, "DELETE", clientMutationId = nextMutationId()))
     }
 }
 
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemSync.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemSync.kt
index 37f7a21..0a7bdcb 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemSync.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemSync.kt
@@ -6,9 +6,7 @@ import kotlinx.coroutines.flow.asStateFlow
 
 interface ItemRemote {
     suspend fun items(): List<Item>
-    suspend fun create(id: String, title: String, completed: Boolean): Item
-    suspend fun patch(id: String, title: String?, completed: Boolean?): Item
-    suspend fun delete(id: String): String
+    suspend fun send(mutation: PendingMutation): MutationResult
 }
 
 enum class SyncState { STALE, REFRESHING, FRESH, ERROR }
@@ -31,6 +29,9 @@ class ItemSync(
     // A new session is stale until explicitly refreshed, even when a saved timestamp exists.
     suspend fun readSavedStatus() {
         mutableStatus.value = mutableStatus.value.copy(lastSuccessfulRefreshAt = store.lastSuccessfulRefreshAt())
+        store.pendingMutations().firstOrNull { it.terminalError != null }?.let {
+            mutableStatus.value = mutableStatus.value.copy(state = SyncState.ERROR, error = it.terminalError)
+        }
     }
 
     fun markLocalChange() {
@@ -41,16 +42,14 @@ class ItemSync(
         try {
             mutableStatus.value = SyncStatus(SyncState.REFRESHING, store.lastSuccessfulRefreshAt())
             for (mutation in store.pendingMutations()) {
-                when (mutation.operation) {
-                    "CREATE" -> remote.create(mutation.itemId, requireNotNull(mutation.title),
-                        requireNotNull(mutation.completed))
-                    "RENAME" -> remote.patch(mutation.itemId, requireNotNull(mutation.title), null)
-                    "COMPLETE" -> remote.patch(mutation.itemId, null, requireNotNull(mutation.completed))
-                    "DELETE" -> remote.delete(mutation.itemId)
-                    else -> error("Unknown pending operation: ${mutation.operation}")
+                if (mutation.terminalError != null) throw MutationIdentityConflict()
+                val result = try {
+                    remote.send(mutation)
+                } catch (collision: MutationIdentityConflict) {
+                    store.markIdentityConflict(mutation)
+                    throw collision
                 }
-                // M06 will handle the remote-success/local-ack crash window and idempotency.
-                store.acknowledge(mutation.sequence)
+                store.acknowledge(mutation, result)
             }
 
             val canonical = remote.items()
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/MutationIdentity.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/MutationIdentity.kt
new file mode 100644
index 0000000..1ca46dd
--- /dev/null
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/MutationIdentity.kt
@@ -0,0 +1,84 @@
+package com.mobilesystemsevolution.kotlin
+
+import java.io.IOException
+import java.security.MessageDigest
+import okhttp3.HttpUrl
+
+data class MutationResult(val statusCode: Int, val responseBody: String)
+
+class MutationIdentityConflict : IOException("identity_conflict")
+
+internal data class MutationRequest(val method: String, val path: String, val payload: Map<String, Any?>?) {
+    fun canonical(): String = canonicalJson(mapOf("method" to method, "path" to path, "payload" to payload))
+    fun hash(): String = MessageDigest.getInstance("SHA-256").digest(canonical().toByteArray(Charsets.UTF_8))
+        .joinToString("") { (it.toInt() and 0xff).toString(16).padStart(2, '0') }
+}
+
+internal fun mutationRequest(itemId: String, operation: String, title: String?, completed: Boolean?): MutationRequest {
+    val method = when (operation) {
+        "CREATE" -> "POST"
+        "RENAME", "COMPLETE" -> "PATCH"
+        "DELETE" -> "DELETE"
+        else -> error("Unknown pending operation: $operation")
+    }
+    // Use the same encoded path segment rules as the actual HTTP request.
+    val path = HttpUrl.Builder().scheme("http").host("localhost").addPathSegment("items")
+        .apply { if (operation != "CREATE") addPathSegment(itemId) }.build().encodedPath
+    val payload = when (operation) {
+        "CREATE" -> mapOf("id" to itemId, "title" to requireNotNull(title), "completed" to requireNotNull(completed))
+        "RENAME" -> mapOf("title" to requireNotNull(title))
+        "COMPLETE" -> mapOf("completed" to requireNotNull(completed))
+        else -> null // DELETE hashes JSON null; its wire body contains only identity metadata.
+    }
+    return MutationRequest(method, path, payload)
+}
+
+internal fun PendingMutation.request() = mutationRequest(itemId, operation, title, completed)
+
+internal fun pendingMutation(itemId: String, operation: String, title: String? = null, completed: Boolean? = null,
+                             clientMutationId: String, sequence: Long = 0): PendingMutation {
+    require(clientMutationId.isNotBlank()) { "Mutation identity must not be blank" }
+    return PendingMutation(sequence, itemId, operation, title, completed, clientMutationId,
+        mutationRequest(itemId, operation, title, completed).hash())
+}
+
+/** Compact JSON with literal UTF-8 text, recursive key ordering, and integer model values. */
+internal fun canonicalJson(value: Any?): String = when (value) {
+    null -> "null"
+    is String -> buildString {
+        append('"')
+        value.forEach { character ->
+            append(when (character) {
+                '"' -> "\\\""
+                '\\' -> "\\\\"
+                '\b' -> "\\b"
+                '\u000c' -> "\\f"
+                '\n' -> "\\n"
+                '\r' -> "\\r"
+                '\t' -> "\\t"
+                else -> if (character.code < 0x20) "\\u" + character.code.toString(16).padStart(4, '0')
+                        else character.toString()
+            })
+        }
+        append('"')
+    }
+    is Boolean, is Int, is Long -> value.toString()
+    is List<*> -> value.joinToString(",", "[", "]") { canonicalJson(it) }
+    is Map<*, *> -> value.keys.map { it as String }.sortedWith { left, right ->
+        compareCodePoints(left, right)
+    }.joinToString(",", "{", "}") { canonicalJson(it) + ":" + canonicalJson(value[it]) }
+    else -> error("Unsupported mutation JSON value: ${value.javaClass.simpleName}")
+}
+
+private fun compareCodePoints(left: String, right: String): Int {
+    var a = 0
+    var b = 0
+    while (a < left.length && b < right.length) {
+        val first = left.codePointAt(a)
+        val second = right.codePointAt(b)
+        if (first != second) return first.compareTo(second)
+        a += Character.charCount(first)
+        b += Character.charCount(second)
+    }
+    return (left.length - a).compareTo(right.length - b)
+}
diff --git a/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt b/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
index 68f9264..7182f59 100644
--- a/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
+++ b/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
@@ -177,7 +177,9 @@ class ItemStoreTest {
     fun frozenM05NewSessionDrainsFourStoredPayloadsInOrder(): Unit = runBlocking {
         val dao = FakeItemDao()
         var localTime = 1700000300000L
-        val store = ItemStore(dao, nextId = { "device-001" }, now = { localTime.also { localTime += 1000 } })
+        var mutationId = 0
+        val store = ItemStore(dao, nextId = { "device-001" }, now = { localTime.also { localTime += 1000 } },
+            nextMutationId = { "m05-${++mutationId}" })
         val remote = FakeRemote().apply { time = 1700000300000L }
         val seeds = remote.rows.values.toList()
         ItemSync(store, remote, now = { 1700000300000L }).synchronize()
@@ -189,10 +191,10 @@ class ItemStoreTest {
         store.delete("remote-002")
 
         val pending = listOf(
-            PendingMutation(1, "device-001", "CREATE", "Gamma", false),
-            PendingMutation(2, "remote-001", "RENAME", "Queued edit", null),
-            PendingMutation(3, "remote-001", "COMPLETE", null, true),
-            PendingMutation(4, "remote-002", "DELETE", null, null),
+            pendingMutation("device-001", "CREATE", "Gamma", false, "m05-1", 1),
+            pendingMutation("remote-001", "RENAME", "Queued edit", null, "m05-2", 2),
+            pendingMutation("remote-001", "COMPLETE", null, true, "m05-3", 3),
+            pendingMutation("remote-002", "DELETE", null, null, "m05-4", 4),
         )
         val local = listOf(
             Item("remote-001", "Queued edit", true, 1, 1700000302000L),
@@ -220,11 +222,113 @@ class ItemStoreTest {
         assertEquals(canonical, reopened.items())
         assertEquals(canonical, remote.rows.values.sortedBy(Item::id))
         assertTrue(reopened.pendingMutations().isEmpty())
+        assertEquals(pending.map { it.clientMutationId to it.payloadHash },
+            reopened.acknowledgedMutations().map { it.clientMutationId to it.payloadHash })
         assertEquals(1700000304000L, remote.time)
         assertEquals(listOf("GET", "POST device-001", "PATCH remote-001 title=Queued edit completed=null",
             "PATCH remote-001 title=null completed=true", "DELETE remote-002", "GET"), remote.calls)
     }
 
+    @Test
+    fun m06CanonicalHashUsesActualPayloadAndStableUtf8Ordering() {
+        val create = pendingMutation("crash-001", "CREATE", "Crash safe", false, "m06-create-001")
+        assertEquals("09d1c3d2db9864ea761ce798ad13b1ee10faba85a8500b552467408d8245d2ba", create.payloadHash)
+        assertEquals("{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"crash-001\",\"title\":\"Crash safe\"}}",
+            create.request().canonical())
+        assertEquals(create.payloadHash, create.copy(clientMutationId = "another-identity").request().hash())
+        assertEquals("3578ce895fdfea0354ac912266ee46823b075efb9a4f54f07333fe6f101bda73",
+            create.copy(title = "Different payload").request().hash())
+        val delete = pendingMutation("crash-001", "DELETE", clientMutationId = "m06-delete-001")
+        assertEquals("{\"method\":\"DELETE\",\"path\":\"/items/crash-001\",\"payload\":null}", delete.request().canonical())
+        assertEquals("443d17b7938e7859a5b26a7d68fc6b5d4a6f9f1006294bdb97439de3f5610535", delete.payloadHash)
+        assertEquals("{\"method\":\"PATCH\",\"path\":\"/items/é\",\"payload\":{\"a\":{\"b\":false,\"z\":null},\"title\":\"한글 🐈\\n\\\"\\\\\",\"z\":[1,true]}}",
+            MutationRequest("PATCH", "/items/é", mapOf("z" to listOf(1, true), "title" to "한글 🐈\n\"\\",
+                "a" to mapOf("z" to null, "b" to false))).canonical())
+        assertEquals("{\"\uE000\":1,\"𐀀\":2}", canonicalJson(mapOf("𐀀" to 2, "\uE000" to 1)))
+    }
+
+    @Test
+    fun frozenM06LostAcknowledgmentReusesStoredIdentityAndRecordsOriginalReply(): Unit = runBlocking {
+        val dao = FakeItemDao()
+        val store = ItemStore(dao, nextId = { "crash-001" }, now = { 1700000400000L },
+            nextMutationId = { "m06-create-001" })
+        val remote = FakeRemote().apply { rows.clear(); time = 1700000400000L; dropNextCreateResponse = true }
+        store.create("Crash safe")
+        val before = store.pendingMutations()
+        val sync = ItemSync(store, remote)
+        assertThrows(IOException::class.java) { runBlocking { sync.synchronize() } }
+        assertEquals(before, store.pendingMutations())
+        assertTrue(store.acknowledgedMutations().isEmpty())
+        assertEquals(1, remote.applied)
+        val original = remote.responses.getValue("m06-create-001").second
+        val reopened = ItemStore(dao, nextMutationId = { error("Replay must not mint a new identity") })
+        ItemSync(reopened, remote, now = { 1700000400000L }).synchronize()
+        assertEquals(listOf(Item("crash-001", "Crash safe", false, 1, 1700000400000L)), reopened.items())
+        assertTrue(reopened.pendingMutations().isEmpty())
+        assertEquals(listOf(AcknowledgedMutation("m06-create-001", before.single().payloadHash,
+            original.statusCode, original.responseBody)), reopened.acknowledgedMutations())
+        assertEquals(listOf(before.single(), before.single()), remote.mutations)
+        assertEquals(listOf("POST crash-001", "POST crash-001", "GET"), remote.calls)
+        assertEquals(1, remote.applied)
+        assertEquals(1, remote.duplicates)
+        assertEquals(1700000401000L, remote.time)
+    }
+
+    @Test
+    fun m06ReceiptWriteFailureLeavesIntentForTheExactReplay(): Unit = runBlocking {
+        val dao = FakeItemDao().apply { rejectAcknowledgment = true }
+        val store = ItemStore(dao, nextId = { "crash-001" }, now = { 1700000400000L },
+            nextMutationId = { "m06-create-001" })
+        val remote = FakeRemote().apply { rows.clear(); time = 1700000400000L }
+        store.create("Crash safe")
+        val pending = store.pendingMutations()
+        val local = store.items()
+        val sync = ItemSync(store, remote)
+        assertThrows(IOException::class.java) { runBlocking { sync.synchronize() } }
+        assertEquals(pending, store.pendingMutations())
+        assertEquals(local, store.items())
+        assertTrue(store.acknowledgedMutations().isEmpty())
+        assertEquals(listOf("ack"), dao.acknowledgmentWrites) // Dequeue was never reached.
+        dao.rejectAcknowledgment = false
+        sync.synchronize()
+        assertTrue(store.pendingMutations().isEmpty())
+        assertEquals(1, store.acknowledgedMutations().size)
+        assertEquals(listOf("ack", "ack", "delete"), dao.acknowledgmentWrites)
+        assertEquals(1, remote.applied)
+        assertEquals(1, remote.duplicates)
+        // Native trigger tests, not this fake, prove rollback after the receipt has been inserted.
+    }
+
+    @Test
+    fun m06IdentityCollisionIsDurableAndNeverResentByANewSession(): Unit = runBlocking {
+        val remote = FakeRemote().apply { rows.clear(); time = 1700000400000L }
+        val original = pendingMutation("crash-001", "CREATE", "Crash safe", false, "m06-create-001")
+        remote.send(original)
+        val dao = FakeItemDao()
+        val store = ItemStore(dao, nextId = { "crash-001" }, now = { 1700000400000L },
+            nextMutationId = { "m06-create-001" })
+        store.create("Different payload")
+        val sync = ItemSync(store, remote)
+        assertThrows(MutationIdentityConflict::class.java) { runBlocking { sync.synchronize() } }
+        val pending = store.pendingMutations().single()
+        assertEquals("identity_conflict", pending.terminalError)
+        assertEquals("3578ce895fdfea0354ac912266ee46823b075efb9a4f54f07333fe6f101bda73", pending.payloadHash)
+        assertEquals(2, remote.calls.size)
+        assertEquals(1, remote.identityConflicts)
+        assertTrue(store.acknowledgedMutations().isEmpty())
+        val reopened = ItemStore(dao)
+        val resumed = ItemSync(reopened, remote)
+        resumed.readSavedStatus()
+        assertEquals(SyncStatus(SyncState.ERROR, null, "identity_conflict"), resumed.status.value)
+        assertThrows(MutationIdentityConflict::class.java) { runBlocking { resumed.synchronize() } }
+        assertEquals(listOf(pending), reopened.pendingMutations())
+        assertEquals(2, remote.calls.size) // No resend or final GET, including after session recreation.
+        assertEquals(listOf(Item("crash-001", "Crash safe", false, 1, 1700000400000L)), remote.rows.values.toList())
+        assertEquals("Different payload", reopened.items().single().title)
+        assertEquals(1, remote.applied)
+        assertEquals(1700000401000L, remote.time)
+    }
+
     private class FakeRemote : ItemRemote {
         val rows = linkedMapOf(
             "remote-001" to Item("remote-001", "Alpha", false, 1, 1700000000000L),
@@ -236,6 +340,12 @@ class ItemStoreTest {
         var beforeGet: () -> Unit = {}
         var beforeMutation: () -> Unit = {}
         var time = 1700000100000L
+        var dropNextCreateResponse = false
+        var applied = 0
+        var duplicates = 0
+        var identityConflicts = 0
+        val mutations = mutableListOf<PendingMutation>()
+        val responses = linkedMapOf<String, Pair<String, MutationResult>>()
         private fun tick() = time.also { time += 1000 }
         override suspend fun items(): List<Item> {
             beforeGet()
@@ -247,26 +357,64 @@ class ItemStoreTest {
             }
             return rows.values.sortedBy(Item::id)
         }
-        override suspend fun create(id: String, title: String, completed: Boolean): Item {
+        override suspend fun send(mutation: PendingMutation): MutationResult {
             beforeMutation()
-            calls += "POST $id"
-            check(id !in rows)
-            return Item(id, title, completed, 1, tick()).also { rows[id] = it }
+            calls += when (mutation.operation) {
+                "CREATE" -> "POST ${mutation.itemId}"
+                "DELETE" -> "DELETE ${mutation.itemId}"
+                else -> "PATCH ${mutation.itemId} title=${mutation.title} completed=${mutation.completed}"
+            }
+            mutations += mutation
+            val request = mutation.request()
+            if (request.hash() != mutation.payloadHash) throw IOException("payload_hash_mismatch")
+            responses[mutation.clientMutationId]?.let { original ->
+                if (original.first != request.canonical()) {
+                    identityConflicts++
+                    throw MutationIdentityConflict()
+                }
+                duplicates++
+                return original.second
+            }
+            val result = when (mutation.operation) {
+                "CREATE" -> {
+                    check(mutation.itemId !in rows)
+                    val item = Item(mutation.itemId, requireNotNull(mutation.title), requireNotNull(mutation.completed), 1, tick())
+                    rows[item.id] = item
+                    itemResponse(201, item)
+                }
+                "RENAME", "COMPLETE" -> itemResponse(200, patchRow(mutation.itemId, mutation.title, mutation.completed))
+                "DELETE" -> {
+                    check(rows.remove(mutation.itemId) != null)
+                    tick()
+                    MutationResult(200, canonicalJson(mapOf("deletedId" to mutation.itemId)))
+                }
+                else -> error("Unknown test operation")
+            }
+            applied++
+            responses[mutation.clientMutationId] = request.canonical() to result
+            if (dropNextCreateResponse && mutation.operation == "CREATE") {
+                dropNextCreateResponse = false
+                throw IOException("Lost response")
+            }
+            return result
         }
-        override suspend fun patch(id: String, title: String?, completed: Boolean?): Item {
+
+        // Other-client helper for the unchanged M04 host scenario, outside the local queue.
+        fun patch(id: String, title: String?, completed: Boolean?): Item {
             beforeMutation()
             calls += "PATCH $id title=$title completed=$completed"
+            return patchRow(id, title, completed)
+        }
+
+        private fun patchRow(id: String, title: String?, completed: Boolean?): Item {
             val item = rows.getValue(id)
             return item.copy(title = title ?: item.title, completed = completed ?: item.completed,
                 version = item.version + 1, updatedAt = tick()).also { rows[id] = it }
         }
-        override suspend fun delete(id: String): String {
-            beforeMutation()
-            calls += "DELETE $id"
-            check(rows.remove(id) != null)
-            tick()
-            return id
-        }
+
+        private fun itemResponse(status: Int, item: Item) = MutationResult(status, canonicalJson(mapOf("item" to mapOf(
+            "id" to item.id, "title" to item.title, "completed" to item.completed, "version" to item.version, "updatedAt" to item.updatedAt,
+        ))))
     }
 
     private class FakeItemDao : ItemDao {
@@ -274,6 +422,9 @@ class ItemStoreTest {
         private var lastRefresh: Long? = null
         private val pending = linkedMapOf<Long, PendingMutation>()
         private var nextSequence = 1L
+        private val acknowledged = linkedMapOf<String, AcknowledgedMutation>()
+        var rejectAcknowledgment = false
+        val acknowledgmentWrites = mutableListOf<String>()
 
         override suspend fun readPending(): List<PendingMutation> = pending.values.sortedBy { it.sequence }
 
@@ -283,7 +434,29 @@ class ItemStoreTest {
             pending[sequence] = mutation.copy(sequence = sequence)
         }
 
-        override suspend fun deletePending(sequence: Long): Int = if (pending.remove(sequence) == null) 0 else 1
+        override suspend fun deletePending(sequence: Long, clientMutationId: String, payloadHash: String): Int {
+            acknowledgmentWrites += "delete"
+            val current = pending[sequence] ?: return 0
+            if (current.clientMutationId != clientMutationId || current.payloadHash != payloadHash || current.terminalError != null) return 0
+            pending.remove(sequence)
+            return 1
+        }
+
+        override suspend fun insertAcknowledged(result: AcknowledgedMutation) {
+            acknowledgmentWrites += "ack"
+            if (rejectAcknowledgment) throw IOException("Receipt rejected")
+            check(result.clientMutationId !in acknowledged)
+            acknowledged[result.clientMutationId] = result
+        }
+
+        override suspend fun readAcknowledged(): List<AcknowledgedMutation> = acknowledged.values.sortedBy { it.clientMutationId }
+
+        override suspend fun markIdentityConflict(sequence: Long, clientMutationId: String, payloadHash: String): Int {
+            val current = pending[sequence] ?: return 0
+            if (current.clientMutationId != clientMutationId || current.payloadHash != payloadHash) return 0
+            pending[sequence] = current.copy(terminalError = "identity_conflict")
+            return 1
+        }
 
         override suspend fun lastSuccessfulRefreshAt(): Long? = lastRefresh
 
diff --git a/verification/M06.md b/verification/M06.md
index aee63cf..dd23f9e 100644
--- a/verification/M06.md
+++ b/verification/M06.md
@@ -2,7 +2,8 @@
 
 START `9b66fe83f24808ade59d9313a460a3c76652e52c`; branch `track/android-kotlin`.
 SPEC_REVISION `61280dd86ce88b6e431f408241c0998a275960aa`.
-Initial attempt1, repair0/2. Baseline accepted; final candidate not yet verified.
+Initial attempt1, repair0/2 at failure. Baseline accepted; fixed candidate failed final
+JUnit (16/17 pass); external replay NOT_RUN. Repair1/2 authorized, not yet applied.
 
 Evidence (`E`): `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/android-kotlin/M06`.
 Exact arguments/environment/exits are in `E/commands.jsonl` and each named JSON/log.
@@ -35,3 +36,34 @@ Root independently accepted the native/PID/UI/HTTP evidence in main
 `threads/evidence/phase-1/android-kotlin/M06/main-baseline-audit.json`.
 `E/baseline-cleanup-audit.json` also checks all39 frozen source files and both APKs.
 Final Android verification belongs to main; no owner-side final suite will run.
+
+## Original fixed candidate checkpoint — failed
+
+The original ten changed product/test/schema files are preserved without modification
+from `E/fixed-built-01/changed-source`; all42 source hashes, both APKs and all10
+changed-source copies matched the frozen manifest before this checkpoint.
+Manifest SHA256 `ad4734ae0224f1e97d0d0cfd9e158bd311304157209c72ec408da595c22eb89d`;
+pre-checkpoint HEAD `01400f3504dde68656487d1e72db0997e810a117`.
+Frozen app SHA256 `0857d61276825965652bb1d1c12a348eff84a0f33688b0201e434ab7a66fa79a`;
+frozen test APK SHA256 `599901c547b8611e277c8f90b6516649fa758cfd3f5803914b8cc437fa2f4be7`.
+Existing host evidence is 11PASS/0FAIL (`E/fixed-built-01/host-results.xml`); no rerun.
+
+Main's actual Android suite executed17:16PASS/1FAIL. All11 Room methods and the
+original5 UI methods passed. Only the new
+`m06ActualHttpCollisionStaysVisibleAndTerminalAfterRoomReopen` failed at
+`ItemUiTest.kt:543`: the semantics text was
+`Refresh failed: identity_conflict. Saved items retained; remote outcome unconfirmed.`
+The failure occurred before the reopen/no-retry assertions; that portion is
+UNVERIFIED. External lost-ack replay is NOT_RUN after suite failure.
+
+Root failure record:
+`/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M06/main-android-failure-01.json`.
+Unmodified raw evidence is alongside it in `main-android-01`; raw `junit.stdout`
+SHA256 `e30614781ce3a0b18a84b21f4714e0d4a03c3e7baa0c8f64a886d2fdaafba92f` proves
+17 executed and one failure. The original driver's `junit: NOT_RUN` field is only
+set to success on completion; it does not mean the suite was unexecuted.
+
+Ledger: initial attempt1 failed; repair1/2 is authorized for only the two new
+`identity_conflict` matchers and a test APK rebuild. No second repair is started.
+This checkpoint contains the original failing test bytes, not a repaired candidate.
+Main owns the subsequent full17 and external replay acceptance runs.


