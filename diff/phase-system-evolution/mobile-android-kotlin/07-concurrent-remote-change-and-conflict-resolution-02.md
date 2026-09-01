## `feat(M07): preserve stale mutation conflicts and acknowledged bases`

diff --git a/TRACK.md b/TRACK.md
index 0637c66..61b9d73 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -174,6 +174,35 @@ already-created Item after process death. The single M06 ledger, exact inputs an
 references are in `verification/M06.md`; initial attempt1, repair0/2. Main owns final Android
 verification. Mutation identity and atomic acknowledgment work is now scoped to M06 only.
 
+## M07 boundary — official runtime passed
+
+Schema 5 stores each update/delete's observed `baseVersion` in its hashed envelope.
+Local edits and that envelope commit together. Dispatch marks the saved envelope before
+HTTP; a failed response never permits changing its identity, payload, base, or hash.
+
+An accepted own predecessor atomically records its original receipt, dequeues, advances
+only the Item's known version, and prepares its immediate never-dispatched successor.
+Preparation validates the old hash and uses that ACK's version, never an external GET.
+Later optimistic fields and newer canonical data are not replaced by an older receipt.
+
+Version rejection atomically preserves the original intent as `version_conflict` and
+accepts its canonical Item/tombstone. Ordinary drains exclude it; the screen retains a
+CONFLICT notice. A fresh explicit edit gets a new identity at the observed canonical version.
+Identity collisions retain M06's terminal behavior. Migration preserves old envelopes and
+receipts without guessing a missing base from optimistic rows: old unversioned updates/
+deletes remain blocked as `base_version_unknown`; old CREATE identities remain replayable.
+
+The frozen actual-process baseline is independently accepted. Host checks passed 16/16;
+main's official Android suite passed 20/20 with zero skips, including all six unchanged UI
+tests. Native HTTP regressions proved M05's exact four accepts with bases none/1/2/1 and
+identical lost PATCH acknowledgment replay after a same-process Room reopen (applied4,
+duplicate1). The separate M07 A/B/C run passed all three actual process-loss boundaries,
+retained the original conflicts without ordinary resends, and accepted A's fresh base2 edit.
+Main audited 17 native snapshots, raw HTTP/PID/UI evidence, unchanged execution bytes and
+cleanup. Historical M05/M06 external scripts are not claimed rerun on schema 5. Attempt1,
+repair0/2; only M07 reporting changed after verification. `verification/M07.md` links the
+commands, hashes and main results. Final commit audit/tagging remains main-owned.
+
 ## Pinned build
 
 - JDK 17, Gradle 8.7 (wrapper distribution SHA-256 pinned)
diff --git a/app/schemas/com.mobilesystemsevolution.kotlin.ItemDatabase/5.json b/app/schemas/com.mobilesystemsevolution.kotlin.ItemDatabase/5.json
new file mode 100644
index 0000000..84e1dea
--- /dev/null
+++ b/app/schemas/com.mobilesystemsevolution.kotlin.ItemDatabase/5.json
@@ -0,0 +1,235 @@
+{
+  "formatVersion": 1,
+  "database": {
+    "version": 5,
+    "identityHash": "86c098a60cee5588a80b162a50237f08",
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
+        "createSql": "CREATE TABLE IF NOT EXISTS `${TABLE_NAME}` (`sequence` INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL, `itemId` TEXT NOT NULL, `operation` TEXT NOT NULL, `title` TEXT, `completed` INTEGER, `clientMutationId` TEXT NOT NULL, `payloadHash` TEXT NOT NULL, `terminalError` TEXT, `baseVersion` INTEGER, `dispatched` INTEGER NOT NULL DEFAULT 0)",
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
+          },
+          {
+            "fieldPath": "baseVersion",
+            "columnName": "baseVersion",
+            "affinity": "INTEGER",
+            "notNull": false
+          },
+          {
+            "fieldPath": "dispatched",
+            "columnName": "dispatched",
+            "affinity": "INTEGER",
+            "notNull": true,
+            "defaultValue": "0"
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
+      },
+      {
+        "tableName": "tombstones",
+        "createSql": "CREATE TABLE IF NOT EXISTS `${TABLE_NAME}` (`id` TEXT NOT NULL, `version` INTEGER NOT NULL, `updatedAt` INTEGER NOT NULL, `deleted` INTEGER NOT NULL, PRIMARY KEY(`id`))",
+        "fields": [
+          {
+            "fieldPath": "id",
+            "columnName": "id",
+            "affinity": "TEXT",
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
+          },
+          {
+            "fieldPath": "deleted",
+            "columnName": "deleted",
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
+      "INSERT OR REPLACE INTO room_master_table (id,identity_hash) VALUES(42, '86c098a60cee5588a80b162a50237f08')"
+    ]
+  }
+}
\ No newline at end of file
diff --git a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemDatabaseTest.kt b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemDatabaseTest.kt
index 43d9792..21ae91a 100644
--- a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemDatabaseTest.kt
+++ b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemDatabaseTest.kt
@@ -3,7 +3,14 @@ package com.mobilesystemsevolution.kotlin
 import android.database.sqlite.SQLiteDatabase
 import androidx.test.ext.junit.runners.AndroidJUnit4
 import androidx.test.platform.app.InstrumentationRegistry
+import java.io.IOException
+import java.util.concurrent.TimeUnit
 import kotlinx.coroutines.runBlocking
+import okhttp3.MediaType.Companion.toMediaType
+import okhttp3.OkHttpClient
+import okhttp3.Request
+import okhttp3.RequestBody.Companion.toRequestBody
+import org.json.JSONObject
 import org.junit.Assert.assertEquals
 import org.junit.Assert.assertThrows
 import org.junit.Assert.assertNull
@@ -17,6 +24,8 @@ class ItemDatabaseTest {
     private val context = InstrumentationRegistry.getInstrumentation().targetContext
     private val name = "m02-room-test.db"
     private val fixture = Item("item-001", "Alpha edited", true, 0, 1700000006000L)
+    private val fixtureClient = OkHttpClient.Builder().retryOnConnectionFailure(false).followRedirects(false)
+        .callTimeout(10, TimeUnit.SECONDS).build()
 
     @Before
     fun emptyTestDatabase() {
@@ -32,6 +41,15 @@ class ItemDatabaseTest {
         }
     }
 
+    private fun fixtureRequest(method: String, path: String, body: JSONObject? = null): JSONObject {
+        val request = Request.Builder().url(HttpItemRemote.FIXTURE_URL + path).header("Connection", "close")
+            .method(method, body?.toString()?.toRequestBody("application/json".toMediaType())).build()
+        return fixtureClient.newCall(request).execute().use {
+            assertEquals(200, it.code)
+            JSONObject(requireNotNull(it.body).string())
+        }
+    }
+
     @Test
     fun frozenCrudSurvivesDatabaseReopen() = runBlocking {
         val expected = listOf(Item("item-001", "Alpha edited", true, 0, 1700000005000L))
@@ -53,7 +71,7 @@ class ItemDatabaseTest {
         }
         withDatabase { reopened ->
             assertEquals(expected, ItemStore(reopened.items()).items())
-            assertEquals(4, reopened.openHelper.readableDatabase.version)
+            assertEquals(5, reopened.openHelper.readableDatabase.version)
         }
     }
 
@@ -133,7 +151,7 @@ class ItemDatabaseTest {
         withDatabase { migrated ->
             val store = ItemStore(migrated.items())
             assertEquals(seeds, store.items())
-            assertEquals(4, migrated.openHelper.readableDatabase.version)
+            assertEquals(5, migrated.openHelper.readableDatabase.version)
             assertNull(store.lastSuccessfulRefreshAt())
             store.replaceWithCanonical(seeds, 1700000200000L)
         }
@@ -141,7 +159,7 @@ class ItemDatabaseTest {
             val store = ItemStore(reopened.items())
             assertEquals(seeds, store.items())
             assertEquals(1700000200000L, store.lastSuccessfulRefreshAt())
-            android.util.Log.i("M04Migration", "schema=4; rows=${store.items()}; lastSuccess=${store.lastSuccessfulRefreshAt()}")
+            android.util.Log.i("M04Migration", "schema=5; rows=${store.items()}; lastSuccess=${store.lastSuccessfulRefreshAt()}")
         }
     }
 
@@ -153,9 +171,9 @@ class ItemDatabaseTest {
         )
         val expectedPending = listOf(
             pendingMutation("device-001", "CREATE", "Gamma", false, "m05-1", 1),
-            pendingMutation("remote-001", "RENAME", "Queued edit", null, "m05-2", 2),
-            pendingMutation("remote-001", "COMPLETE", null, true, "m05-3", 3),
-            pendingMutation("remote-002", "DELETE", null, null, "m05-4", 4),
+            pendingMutation("remote-001", "RENAME", "Queued edit", null, "m05-2", 2, baseVersion = 1),
+            pendingMutation("remote-001", "COMPLETE", null, true, "m05-3", 3, baseVersion = 1),
+            pendingMutation("remote-002", "DELETE", null, null, "m05-4", 4, baseVersion = 1),
         )
         withDatabase { database ->
             var time = 1700000300000L
@@ -175,11 +193,39 @@ class ItemDatabaseTest {
         }
         withDatabase { reopened ->
             val store = ItemStore(reopened.items())
-            assertEquals(4, reopened.openHelper.readableDatabase.version)
+            assertEquals(5, reopened.openHelper.readableDatabase.version)
             assertEquals(expectedRows, store.items())
             assertEquals(expectedPending, store.pendingMutations())
             assertEquals(1700000300000L, store.lastSuccessfulRefreshAt())
             android.util.Log.i("M05Queue", "reopenedRows=${store.items()}; pending=${store.pendingMutations()}")
+            fixtureRequest("POST", "__reset", JSONObject())
+            fixtureRequest("POST", "__control", JSONObject().put("nextTimestamp", 1700000300000L))
+            ItemSync(store, HttpItemRemote(), now = { 1700000300000L }).synchronize()
+            val canonical = listOf(
+                Item("device-001", "Gamma", false, 1, 1700000300000L),
+                Item("remote-001", "Queued edit", true, 3, 1700000302000L),
+            )
+            assertEquals(canonical, store.items())
+            assertTrue(store.pendingMutations().isEmpty())
+            assertEquals(4, store.acknowledgedMutations().size)
+            val state = fixtureRequest("GET", "__m07")
+            assertEquals(4, state.getInt("applied"))
+            assertEquals(4, state.getInt("mutationRequests"))
+            assertEquals(0, state.getInt("duplicates"))
+            assertEquals(0, state.getInt("versionConflicts"))
+            assertEquals(1700000304000L, state.getLong("nextTimestamp"))
+            val requests = state.getJSONArray("requests")
+            assertEquals(listOf("POST", "PATCH", "PATCH", "DELETE"),
+                List(requests.length()) { requests.getJSONObject(it).getString("method") })
+            assertEquals(listOf(null, 1L, 2L, 1L), List(requests.length()) {
+                val body = requests.getJSONObject(it).getJSONObject("request")
+                if (body.has("baseVersion")) body.getLong("baseVersion") else null
+            })
+            assertEquals(listOf(201, 200, 200, 200),
+                List(requests.length()) { requests.getJSONObject(it).getInt("status") })
+            assertEquals(expectedPending[2].withBaseVersion(2).payloadHash,
+                store.acknowledgedMutations().single { it.clientMutationId == "m05-3" }.payloadHash)
+            android.util.Log.i("M07M05", "four real HTTP accepts; bases=null,1,2,1; rows=${store.items()}; pending=0")
         }
     }
 
@@ -247,7 +293,7 @@ class ItemDatabaseTest {
         }
         withDatabase { migrated ->
             val store = ItemStore(migrated.items())
-            assertEquals(4, migrated.openHelper.readableDatabase.version)
+            assertEquals(5, migrated.openHelper.readableDatabase.version)
             assertEquals(rows, store.items())
             identifiedPending = store.pendingMutations()
             assertEquals(expectedPayload, identifiedPending.map { listOf(it.sequence, it.itemId, it.operation, it.title, it.completed) })
@@ -260,26 +306,31 @@ class ItemDatabaseTest {
             assertEquals(rows, store.items())
             assertEquals(identifiedPending, store.pendingMutations())
             assertEquals(1700000200000L, store.lastSuccessfulRefreshAt())
-            android.util.Log.i("M05Migration", "schema=4; rows=${store.items()}; pending=${store.pendingMutations()}")
+            android.util.Log.i("M05Migration", "schema=5; rows=${store.items()}; pending=${store.pendingMutations()}")
         }
     }
 
     @Test
     fun m06AcknowledgmentAndExactDequeueRollbackTogetherAndSurviveReopen(): Unit = runBlocking {
         val result = MutationResult(201,
-            "{\"item\":{\"id\":\"crash-001\",\"title\":\"Crash safe\",\"completed\":false,\"version\":1,\"updatedAt\":1700000400000}}")
+            "{\"item\":{\"id\":\"crash-001\",\"title\":\"Crash safe\",\"completed\":false,\"version\":1,\"updatedAt\":1700000400000}}",
+            Item("crash-001", "Crash safe", false, 1, 1700000400000L))
         var pending = emptyList<PendingMutation>()
         var local = emptyList<Item>()
+        var prepared = emptyList<PendingMutation>()
+        var observed = emptyList<Item>()
         withDatabase { database ->
             val identities = listOf("m06-create-001", "m06-rename-001").iterator()
             val store = ItemStore(database.items(), nextId = { "crash-001" }, now = { 1700000400000L },
                 nextMutationId = { identities.next() })
             store.create("Crash safe")
             store.rename("crash-001", "Later local edit")
+            store.prepareDispatch(store.pendingMutations().first().sequence)
             pending = store.pendingMutations()
             local = store.items()
             val sql = database.openHelper.writableDatabase
-            for ((table, event) in listOf("acknowledged_mutations" to "INSERT", "pending_mutations" to "DELETE")) {
+            for ((table, event) in listOf("acknowledged_mutations" to "INSERT", "pending_mutations" to "DELETE",
+                "items" to "UPDATE", "pending_mutations" to "UPDATE")) {
                 sql.execSQL("CREATE TRIGGER reject_ack BEFORE $event ON $table " +
                     "BEGIN SELECT RAISE(ABORT, 'Rejected acknowledgment transaction'); END")
                 assertThrows(android.database.sqlite.SQLiteConstraintException::class.java) {
@@ -296,14 +347,26 @@ class ItemDatabaseTest {
             }
             assertEquals(pending, store.pendingMutations())
             assertTrue(store.acknowledgedMutations().isEmpty())
+            sql.execSQL("UPDATE pending_mutations SET payloadHash = ? WHERE sequence = ?",
+                arrayOf("0".repeat(64), pending.last().sequence))
+            assertThrows(IllegalStateException::class.java) {
+                runBlocking { store.acknowledge(pending.first(), result) }
+            }
+            assertEquals(listOf(pending.first(), pending.last().copy(payloadHash = "0".repeat(64))), store.pendingMutations())
+            assertEquals(local, store.items())
+            assertTrue(store.acknowledgedMutations().isEmpty())
+            sql.execSQL("UPDATE pending_mutations SET payloadHash = ? WHERE sequence = ?",
+                arrayOf(pending.last().payloadHash, pending.last().sequence))
             store.acknowledge(pending.first(), result)
-            assertEquals(pending.drop(1), store.pendingMutations())
-            assertEquals(local, store.items()) // Recording the old CREATE reply cannot erase the later edit.
+            prepared = listOf(pending.last().withBaseVersion(1))
+            observed = local.map { it.copy(version = 1) }
+            assertEquals(prepared, store.pendingMutations())
+            assertEquals(observed, store.items()) // Only observed version advances; later fields/time survive.
         }
         withDatabase { reopened ->
             val store = ItemStore(reopened.items())
-            assertEquals(pending.drop(1), store.pendingMutations())
-            assertEquals(local, store.items())
+            assertEquals(prepared, store.pendingMutations())
+            assertEquals(observed, store.items())
             assertEquals(listOf(AcknowledgedMutation("m06-create-001", pending.first().payloadHash,
                 result.statusCode, result.responseBody)), store.acknowledgedMutations())
             android.util.Log.i("M06Atomicity", "reopened pending=${store.pendingMutations()}; acknowledged=${store.acknowledgedMutations()}")
@@ -342,7 +405,7 @@ class ItemDatabaseTest {
             withDatabase { database ->
                 val store = ItemStore(database.items(), nextId = { "new-001" }, now = { 1700000401000L },
                     nextMutationId = { "m06-after-migration" })
-                assertEquals(4, database.openHelper.readableDatabase.version)
+                assertEquals(5, database.openHelper.readableDatabase.version)
                 val queued = store.pendingMutations()
                 assertEquals(if (hasPending) legacyPayloads else emptyList<List<Any?>>(),
                     queued.map { listOf(it.sequence, it.itemId, it.operation, it.title, it.completed) })
@@ -350,7 +413,9 @@ class ItemDatabaseTest {
                 queued.forEach {
                     assertEquals(it.clientMutationId, java.util.UUID.fromString(it.clientMutationId).toString())
                     assertEquals(it.request().hash(), it.payloadHash)
-                    assertNull(it.terminalError)
+                    assertNull(it.baseVersion)
+                    assertTrue(it.dispatched)
+                    assertEquals(if (it.operation == "CREATE") null else "base_version_unknown", it.terminalError)
                 }
                 assertEquals(listOf(Item("device-001", "Newer local title", true, 0, 1700000400000L)), store.items())
                 assertEquals(1700000300000L, store.lastSuccessfulRefreshAt())
@@ -369,6 +434,221 @@ class ItemDatabaseTest {
         }
     }
 
+    @Test
+    fun m07ConflictCanonicalWriteRollsBackWithItsEnvelopeAndSurvivesReopen(): Unit = runBlocking {
+        val seed = Item("conflict-001", "Initial", false, 1, 1700000000000L)
+        val winner = seed.copy(title = "Remote winner", version = 2, updatedAt = 1700000500000L)
+        val deleted = Tombstone(seed.id, 2, 1700000500000L)
+        for ((case, identity) in listOf("A" to "m07-update-001", "B" to "m07-delete-001", "C" to "m07-deleted-001")) {
+            context.deleteDatabase(name)
+            val canonical = if (case == "C") emptyList() else listOf(winner)
+            val tombstones = if (case == "C") listOf(deleted) else emptyList()
+            val conflict = MutationVersionConflict(canonical.singleOrNull(), tombstones.singleOrNull())
+            var preserved: PendingMutation? = null
+            withDatabase { database ->
+                val store = ItemStore(database.items(), now = { 1700000500000L }, nextMutationId = { identity })
+                store.replaceWithCanonical(listOf(seed), 1700000500000L)
+                if (case == "B") store.delete(seed.id) else store.rename(seed.id, "Local attempt")
+                val original = store.pendingMutations().single()
+                assertEquals(1L, original.baseVersion)
+                assertEquals(if (case == "B") "dcdc12c2281f2619b08cf53b7fb3aebf5c1d9fa3e0273b2857868ad75c08c736"
+                    else "3a20776167a2f7bcd285fce59e9b6ce1a855ad169b067726ce2016102d1fcd8d", original.payloadHash)
+                val sending = store.prepareDispatch(original.sequence)
+                assertEquals(original.copy(dispatched = true), sending)
+                val optimistic = store.items()
+                val table = if (case == "C") "tombstones" else "items"
+                val sql = database.openHelper.writableDatabase
+                sql.execSQL("CREATE TRIGGER reject_conflict BEFORE INSERT ON $table " +
+                    "BEGIN SELECT RAISE(ABORT, 'Rejected canonical conflict'); END")
+                assertThrows(android.database.sqlite.SQLiteConstraintException::class.java) {
+                    runBlocking { store.recordVersionConflict(sending, conflict) }
+                }
+                assertEquals(listOf(sending), store.pendingMutations())
+                assertEquals(optimistic, store.items())
+                assertTrue(store.tombstones().isEmpty())
+                assertTrue(store.acknowledgedMutations().isEmpty())
+                assertEquals(1700000500000L, store.lastSuccessfulRefreshAt())
+                sql.execSQL("DROP TRIGGER reject_conflict")
+                store.recordVersionConflict(sending, conflict)
+                preserved = sending.copy(terminalError = "version_conflict")
+                assertEquals(listOf(preserved), store.pendingMutations())
+                assertEquals(canonical, store.items())
+                assertEquals(tombstones, store.tombstones())
+            }
+            withDatabase { reopened ->
+                val store = ItemStore(reopened.items())
+                assertEquals(listOf(preserved), store.pendingMutations())
+                assertEquals(canonical, store.items())
+                assertEquals(tombstones, store.tombstones())
+                var reads = 0
+                val remote = object : ItemRemote {
+                    override suspend fun items(): List<Item> { reads++; return canonical }
+                    override suspend fun snapshot() = CanonicalSnapshot(items(), tombstones)
+                    override suspend fun send(mutation: PendingMutation): MutationResult = error("Conflict was resent")
+                }
+                val sync = ItemSync(store, remote, now = { 1700000501000L })
+                sync.readSavedStatus()
+                sync.synchronize()
+                sync.readSavedStatus()
+                assertEquals(SyncState.FRESH, sync.status.value.state)
+                assertEquals(1, reads)
+                assertEquals(listOf(preserved), store.pendingMutations())
+                assertEquals(canonical, store.items())
+                assertEquals(tombstones, store.tombstones())
+                assertTrue(store.acknowledgedMutations().isEmpty())
+                // An optional-tombstone GET must not erase previously accepted deletion evidence.
+                store.replaceWithCanonical(canonical, 1700000501000L)
+                assertEquals(tombstones, store.tombstones())
+                android.util.Log.i("M07Conflict", "$case reopened canonical=${store.items()}; tombstones=${store.tombstones()}; original=${store.pendingMutations()}")
+            }
+        }
+    }
+
+    @Test
+    fun m07RealHttpLostSuccessorAckReplaysPreparedEnvelopeAfterRoomReopen(): Unit = runBlocking {
+        fixtureRequest("POST", "__reset", JSONObject())
+        fixtureRequest("POST", "__control", JSONObject().put("nextTimestamp", 1700000300000L))
+        var afterFailure = emptyList<PendingMutation>()
+        var optimistic = emptyList<Item>()
+        var droppedBody: String? = null
+        var atResponse: PendingMutation? = null
+        withDatabase { database ->
+            var time = 1700000300000L
+            var identity = 0
+            val store = ItemStore(database.items(), nextId = { "device-001" }, now = { time.also { time += 1000 } },
+                nextMutationId = { "m07-replay-${++identity}" })
+            ItemSync(store, HttpItemRemote(), now = { 1700000300000L }).synchronize()
+            store.create("Gamma")
+            store.rename("remote-001", "Queued edit")
+            store.setCompleted("remote-001", true)
+            store.delete("remote-002")
+            val original = store.pendingMutations()
+            var patches = 0
+            val dropping = fixtureClient.newBuilder().addInterceptor { chain ->
+                val response = chain.proceed(chain.request())
+                if (chain.request().method == "PATCH" && ++patches == 2) {
+                    // The fixture has really committed. Drop only the completion ACK before ItemSync receives it.
+                    atResponse = runBlocking { store.pendingMutations().first() }
+                    assertEquals(200, response.code)
+                    droppedBody = requireNotNull(response.body).string()
+                    response.close()
+                    throw IOException("M07 completion acknowledgment dropped after acceptance")
+                }
+                response
+            }.build()
+            val sync = ItemSync(store, HttpItemRemote(client = dropping), now = { 1700000300000L })
+            val error = assertThrows(IOException::class.java) { runBlocking { sync.synchronize() } }
+            assertEquals("M07 completion acknowledgment dropped after acceptance", error.message)
+            afterFailure = store.pendingMutations()
+            assertEquals(listOf(original[2].withBaseVersion(2).copy(dispatched = true), original[3]), afterFailure)
+            assertEquals(afterFailure.first(), atResponse)
+            assertEquals(2, store.acknowledgedMutations().size)
+            optimistic = listOf(Item("remote-001", "Queued edit", true, 2, 1700000302000L),
+                Item("device-001", "Gamma", false, 1, 1700000300000L))
+            assertEquals(optimistic, store.items())
+            val state = fixtureRequest("GET", "__m07")
+            assertEquals(3, state.getInt("applied"))
+            assertEquals(3, state.getInt("mutationRequests"))
+            assertEquals(0, state.getInt("duplicates"))
+            assertEquals(1700000303000L, state.getLong("nextTimestamp"))
+            assertEquals(3L, JSONObject(requireNotNull(droppedBody)).getJSONObject("item").getLong("version"))
+        }
+        // This is a same-process Room reopen. The external M07 harness separately proves actual process loss.
+        withDatabase { reopened ->
+            val store = ItemStore(reopened.items(), nextMutationId = { error("Replay minted an identity") })
+            assertEquals(afterFailure, store.pendingMutations())
+            assertEquals(optimistic, store.items())
+            ItemSync(store, HttpItemRemote(), now = { 1700000300000L }).synchronize()
+            assertTrue(store.pendingMutations().isEmpty())
+            assertEquals(listOf(Item("device-001", "Gamma", false, 1, 1700000300000L),
+                Item("remote-001", "Queued edit", true, 3, 1700000302000L)), store.items())
+            val receipt = store.acknowledgedMutations().single { it.clientMutationId == "m07-replay-3" }
+            assertEquals(4, store.acknowledgedMutations().size)
+            assertEquals(requireNotNull(droppedBody), receipt.responseBody)
+            assertEquals(afterFailure.first().payloadHash, receipt.payloadHash)
+            val state = fixtureRequest("GET", "__m07")
+            assertEquals(4, state.getInt("applied"))
+            assertEquals(5, state.getInt("mutationRequests"))
+            assertEquals(1, state.getInt("duplicates"))
+            assertEquals(0, state.getInt("versionConflicts"))
+            assertEquals(1700000304000L, state.getLong("nextTimestamp"))
+            val requests = state.getJSONArray("requests")
+            val first = requests.getJSONObject(2)
+            val replay = requests.getJSONObject(3)
+            assertEquals("PATCH", first.getString("method"))
+            assertEquals("PATCH", replay.getString("method"))
+            assertEquals(first.getString("canonical"), replay.getString("canonical"))
+            assertEquals(first.getJSONObject("request").toString(), replay.getJSONObject("request").toString())
+            assertEquals(2L, replay.getJSONObject("request").getLong("baseVersion"))
+            assertEquals("replayed", replay.getString("delivery"))
+            android.util.Log.i("M07Replay", "same-process Room reopen; applied=4 duplicate=1; originalReceipt=${receipt.responseBody}; pending=0")
+        }
+    }
+
+    @Test
+    fun m07V4MigrationPreservesUnknownEnvelopesReceiptsAndAllocator(): Unit = runBlocking {
+        val original = listOf(
+            pendingMutation("remote-001", "RENAME", "Original attempt", null, "legacy-rename", 2),
+            pendingMutation("remote-001", "DELETE", null, null, "legacy-delete", 4),
+            pendingMutation("crash-001", "CREATE", "Crash safe", false, "m06-create-001", 6),
+            pendingMutation("remote-001", "COMPLETE", null, true, "legacy-collision", 8).copy(terminalError = "identity_conflict"),
+        )
+        val receipt = AcknowledgedMutation("older-ack", "a".repeat(64), 200, "{\"deletedId\":\"older-removed\"}")
+        val expected = original.map { it.copy(dispatched = true,
+            terminalError = it.terminalError ?: if (it.operation == "CREATE") null else "base_version_unknown") }
+        val row = Item("remote-001", "Newer optimistic value", true, 9, 1700000500000L)
+        val path = context.getDatabasePath(name)
+        requireNotNull(path.parentFile).mkdirs()
+        SQLiteDatabase.openOrCreateDatabase(path, null).use { legacy ->
+            legacy.execSQL("CREATE TABLE items (id TEXT NOT NULL PRIMARY KEY, title TEXT NOT NULL, completed INTEGER NOT NULL, " +
+                "version INTEGER NOT NULL, updatedAt INTEGER NOT NULL)")
+            legacy.execSQL("INSERT INTO items VALUES(?,?,?,?,?)", arrayOf(row.id, row.title, 1, row.version, row.updatedAt))
+            legacy.execSQL("CREATE TABLE sync_metadata (id INTEGER NOT NULL PRIMARY KEY, lastSuccessfulRefreshAt INTEGER NOT NULL)")
+            legacy.execSQL("INSERT INTO sync_metadata VALUES(1,1700000300000)")
+            legacy.execSQL("CREATE TABLE pending_mutations (sequence INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL, " +
+                "itemId TEXT NOT NULL, operation TEXT NOT NULL, title TEXT, completed INTEGER, " +
+                "clientMutationId TEXT NOT NULL, payloadHash TEXT NOT NULL, terminalError TEXT)")
+            original.forEach {
+                legacy.execSQL("INSERT INTO pending_mutations VALUES(?,?,?,?,?,?,?,?)", arrayOf(it.sequence,
+                    it.itemId, it.operation, it.title, it.completed?.let { value -> if (value) 1 else 0 },
+                    it.clientMutationId, it.payloadHash, it.terminalError))
+            }
+            legacy.execSQL("INSERT INTO pending_mutations VALUES(12,'removed','DELETE',NULL,NULL,'removed','old-hash',NULL)")
+            legacy.execSQL("DELETE FROM pending_mutations WHERE sequence=12")
+            legacy.execSQL("CREATE TABLE acknowledged_mutations (clientMutationId TEXT NOT NULL PRIMARY KEY, " +
+                "payloadHash TEXT NOT NULL, statusCode INTEGER NOT NULL, responseBody TEXT NOT NULL)")
+            legacy.execSQL("INSERT INTO acknowledged_mutations VALUES(?,?,?,?)",
+                arrayOf(receipt.clientMutationId, receipt.payloadHash, receipt.statusCode, receipt.responseBody))
+            legacy.version = 4
+        }
+        var afterCreate = emptyList<PendingMutation>()
+        withDatabase { migrated ->
+            val store = ItemStore(migrated.items(), nextId = { "new-001" }, now = { 1700000501000L },
+                nextMutationId = { "new-after-migration" })
+            assertEquals(5, migrated.openHelper.readableDatabase.version)
+            assertEquals(expected, store.pendingMutations())
+            assertEquals(listOf(receipt), store.acknowledgedMutations())
+            assertEquals(listOf(row), store.items())
+            assertTrue(store.tombstones().isEmpty())
+            assertEquals(1700000300000L, store.lastSuccessfulRefreshAt())
+            assertThrows(IllegalStateException::class.java) { runBlocking { store.prepareDispatch(2) } }
+            assertEquals(expected[2], store.prepareDispatch(6)) // Legacy CREATE remains replayable without a guessed base.
+            store.create("After migration")
+            afterCreate = store.pendingMutations()
+            assertEquals(expected, afterCreate.dropLast(1))
+            assertEquals(pendingMutation("new-001", "CREATE", "After migration", false,
+                "new-after-migration", 13), afterCreate.last())
+        }
+        withDatabase { reopened ->
+            val store = ItemStore(reopened.items())
+            assertEquals(afterCreate, store.pendingMutations())
+            assertEquals(listOf(receipt), store.acknowledgedMutations())
+            assertEquals(row, store.items().first())
+            assertEquals(1700000300000L, store.lastSuccessfulRefreshAt())
+            android.util.Log.i("M07Migration", "no inferred old base; original identities/hashes/receipts retained; allocator=13")
+        }
+    }
+
     @Test
     fun unknownSchemaVersionRejectsOpenWithoutDeletingExistingData() {
         runBlocking { withDatabase { it.items().insert(fixture.toEntity()) } }
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/HttpItemRemote.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/HttpItemRemote.kt
index 9e04375..48352b7 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/HttpItemRemote.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/HttpItemRemote.kt
@@ -17,9 +17,14 @@ class HttpItemRemote(
 ) : ItemRemote {
     private val base = baseUrl.toHttpUrl()
 
-    override suspend fun items(): List<Item> {
-        val rows = JSONObject(request("GET", "/items", null, 200).responseBody).getJSONArray("items")
-        return List(rows.length()) { rows.getJSONObject(it).toItem() }
+    override suspend fun items(): List<Item> = snapshot().items
+
+    override suspend fun snapshot(): CanonicalSnapshot {
+        val response = JSONObject(request("GET", "/items", null, 200).responseBody)
+        val rows = response.getJSONArray("items")
+        val tombstones = if (response.has("tombstones")) response.getJSONArray("tombstones") else null
+        return CanonicalSnapshot(List(rows.length()) { rows.getJSONObject(it).toItem() },
+            if (tombstones == null) emptyList() else List(tombstones.length()) { tombstones.getJSONObject(it).toTombstone() })
     }
 
     override suspend fun send(mutation: PendingMutation): MutationResult {
@@ -27,12 +32,17 @@ class HttpItemRemote(
         val body = JSONObject()
         stored.payload?.forEach { (key, value) -> body.put(key, value) }
         body.put("clientMutationId", mutation.clientMutationId).put("payloadHash", mutation.payloadHash)
-        val result = request(stored.method, stored.path, body, if (stored.method == "POST") 201 else 200)
+        val result = try {
+            request(stored.method, stored.path, body, if (stored.method == "POST") 201 else 200)
+        } catch (conflict: MutationVersionConflict) {
+            if (conflict.itemId != mutation.itemId) throw IOException("Unexpected conflict Item identity")
+            throw conflict
+        }
         val response = JSONObject(result.responseBody)
-        val responseId = if (stored.method == "DELETE") response.getString("deletedId")
-                         else response.getJSONObject("item").toItem().id
+        val item = if (stored.method == "DELETE") null else response.getJSONObject("item").toItem()
+        val responseId = item?.id ?: response.getString("deletedId")
         if (responseId != mutation.itemId) throw IOException("Unexpected acknowledged Item identity")
-        return result
+        return result.copy(item = item)
     }
 
     private suspend fun request(method: String, path: String, body: JSONObject?, status: Int): MutationResult =
@@ -46,8 +56,13 @@ class HttpItemRemote(
                 .build()
             client.newCall(request).execute().use { response ->
                 val raw = response.body?.string() ?: throw IOException("Empty sync response")
-                if (response.code == 409 && JSONObject(raw).optString("error") == "identity_conflict") {
-                    throw MutationIdentityConflict()
+                if (response.code == 409) {
+                    val conflict = JSONObject(raw)
+                    when (conflict.optString("error")) {
+                        "identity_conflict" -> throw MutationIdentityConflict()
+                        "version_conflict" -> throw MutationVersionConflict(
+                            conflict.optJSONObject("item")?.toItem(), conflict.optJSONObject("tombstone")?.toTombstone())
+                    }
                 }
                 if (response.code != status) throw IOException("Sync HTTP ${response.code}")
                 MutationResult(response.code, raw)
@@ -59,6 +74,10 @@ class HttpItemRemote(
         getLong("version"), getLong("updatedAt"),
     )
 
+    private fun JSONObject.toTombstone() = Tombstone(
+        getString("id"), getLong("version"), getLong("updatedAt"), getBoolean("deleted"),
+    ).also { require(it.deleted) { "Expected a deleted tombstone" } }
+
     companion object {
         const val FIXTURE_URL = "http://10.0.2.2:18080/"
         private val CLIENT = OkHttpClient.Builder()
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemDatabase.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemDatabase.kt
index 582497c..9931cbe 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemDatabase.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemDatabase.kt
@@ -1,6 +1,7 @@
 package com.mobilesystemsevolution.kotlin
 
 import android.content.Context
+import androidx.room.ColumnInfo
 import androidx.room.Dao
 import androidx.room.Database
 import androidx.room.Entity
@@ -44,6 +45,16 @@ data class PendingMutation(
     val clientMutationId: String,
     val payloadHash: String,
     val terminalError: String? = null,
+    val baseVersion: Long? = null,
+    @ColumnInfo(defaultValue = "0") val dispatched: Boolean = false,
+)
+
+@Entity(tableName = "tombstones")
+data class Tombstone(
+    @PrimaryKey val id: String,
+    val version: Long,
+    val updatedAt: Long,
+    val deleted: Boolean = true,
 )
 
 @Entity(tableName = "acknowledged_mutations")
@@ -60,9 +71,18 @@ interface ItemDao {
     @Query("SELECT * FROM items ORDER BY rowid")
     suspend fun readAll(): List<ItemEntity>
 
+    @Query("SELECT * FROM items WHERE id = :id")
+    suspend fun readItem(id: String): ItemEntity?
+
     @Insert
     suspend fun insert(item: ItemEntity)
 
+    @Insert(onConflict = OnConflictStrategy.REPLACE)
+    suspend fun upsertCanonical(item: ItemEntity)
+
+    @Query("UPDATE items SET version = max(version, :version) WHERE id = :id")
+    suspend fun advanceObservedVersion(id: String, version: Long): Int
+
     @Query("UPDATE items SET title = :title, updatedAt = :updatedAt WHERE id = :id")
     suspend fun rename(id: String, title: String, updatedAt: Long): Int
 
@@ -84,6 +104,9 @@ interface ItemDao {
     @Query("SELECT * FROM pending_mutations ORDER BY sequence")
     suspend fun readPending(): List<PendingMutation>
 
+    @Query("SELECT * FROM pending_mutations WHERE sequence = :sequence")
+    suspend fun readPendingBySequence(sequence: Long): PendingMutation?
+
     @Insert
     suspend fun insertPending(mutation: PendingMutation)
 
@@ -97,19 +120,90 @@ interface ItemDao {
     @Query("SELECT * FROM acknowledged_mutations ORDER BY clientMutationId")
     suspend fun readAcknowledged(): List<AcknowledgedMutation>
 
+    @Query("UPDATE pending_mutations SET dispatched = 1 WHERE sequence = :sequence " +
+        "AND clientMutationId = :clientMutationId AND payloadHash = :payloadHash AND terminalError IS NULL")
+    suspend fun markDispatched(sequence: Long, clientMutationId: String, payloadHash: String): Int
+
+    @Query("UPDATE pending_mutations SET baseVersion = :baseVersion, payloadHash = :payloadHash " +
+        "WHERE sequence = :sequence AND clientMutationId = :clientMutationId AND payloadHash = :previousHash " +
+        "AND dispatched = 0 AND terminalError IS NULL")
+    suspend fun updatePreparedMutation(sequence: Long, clientMutationId: String, previousHash: String,
+                                       baseVersion: Long, payloadHash: String): Int
+
+    @Transaction
+    suspend fun prepareDispatch(sequence: Long): PendingMutation {
+        val mutation = checkNotNull(readPendingBySequence(sequence)) { "Pending change no longer exists" }
+        check(mutation.terminalError == null) { "Terminal mutation cannot be dispatched" }
+        check(mutation.operation == "CREATE" || mutation.baseVersion != null) { "Mutation has no recorded base version" }
+        check(mutation.request().hash() == mutation.payloadHash) { "Stored mutation hash does not match its payload" }
+        if (!mutation.dispatched) {
+            check(markDispatched(sequence, mutation.clientMutationId, mutation.payloadHash) == 1)
+        }
+        // Persist before entering HTTP. An unsuccessful response never authorizes a new envelope.
+        return mutation.copy(dispatched = true)
+    }
+
     @Transaction
-    suspend fun acknowledge(sequence: Long, result: AcknowledgedMutation) {
-        // Record the original success without overwriting later queued local Item edits.
+    suspend fun acknowledge(sequence: Long, result: AcknowledgedMutation, item: ItemEntity? = null) {
+        val mutation = checkNotNull(readPendingBySequence(sequence)) { "Pending change no longer exists" }
+        check(mutation.clientMutationId == result.clientMutationId && mutation.payloadHash == result.payloadHash)
+        check(item == null || item.id == mutation.itemId)
         insertAcknowledged(result)
         check(deletePending(sequence, result.clientMutationId, result.payloadHash) == 1) {
             "Pending change no longer matches acknowledgment"
         }
+        if (item != null) {
+            // An old receipt must not replace newer local fields or resurrect a deleted Item.
+            advanceObservedVersion(item.id, item.version)
+            val successor = readPending().firstOrNull { it.itemId == item.id && it.sequence > sequence }
+            if (successor != null && !successor.dispatched && successor.terminalError == null && successor.operation != "CREATE") {
+                val prepared = successor.withBaseVersion(item.version)
+                check(updatePreparedMutation(successor.sequence, successor.clientMutationId, successor.payloadHash,
+                    item.version, prepared.payloadHash) == 1)
+            }
+        }
     }
 
     @Query("UPDATE pending_mutations SET terminalError = 'identity_conflict' " +
         "WHERE sequence = :sequence AND clientMutationId = :clientMutationId AND payloadHash = :payloadHash")
     suspend fun markIdentityConflict(sequence: Long, clientMutationId: String, payloadHash: String): Int
 
+    @Query("UPDATE pending_mutations SET terminalError = 'version_conflict' " +
+        "WHERE sequence = :sequence AND clientMutationId = :clientMutationId AND payloadHash = :payloadHash " +
+        "AND dispatched = 1 AND terminalError IS NULL")
+    suspend fun markVersionConflict(sequence: Long, clientMutationId: String, payloadHash: String): Int
+
+    @Query("SELECT * FROM tombstones ORDER BY id")
+    suspend fun readTombstones(): List<Tombstone>
+
+    @Query("SELECT * FROM tombstones WHERE id = :id")
+    suspend fun readTombstone(id: String): Tombstone?
+
+    @Insert(onConflict = OnConflictStrategy.REPLACE)
+    suspend fun insertTombstone(tombstone: Tombstone)
+
+    @Query("DELETE FROM tombstones WHERE id = :id")
+    suspend fun deleteTombstone(id: String): Int
+
+    @Transaction
+    suspend fun recordVersionConflict(mutation: PendingMutation, item: ItemEntity?, tombstone: Tombstone?) {
+        check((item == null) xor (tombstone == null))
+        val id = item?.id ?: checkNotNull(tombstone).id
+        check(id == mutation.itemId)
+        check(markVersionConflict(mutation.sequence, mutation.clientMutationId, mutation.payloadHash) == 1)
+        val knownVersion = maxOf(readItem(id)?.version ?: -1, readTombstone(id)?.version ?: -1)
+        val version = item?.version ?: checkNotNull(tombstone).version
+        if (version >= knownVersion) {
+            if (item != null) {
+                upsertCanonical(item)
+                deleteTombstone(id)
+            } else {
+                delete(id)
+                insertTombstone(checkNotNull(tombstone))
+            }
+        }
+    }
+
     @Transaction
     suspend fun createLocal(item: ItemEntity, mutation: PendingMutation) {
         insert(item)
@@ -118,34 +212,47 @@ interface ItemDao {
 
     @Transaction
     suspend fun renameLocal(id: String, title: String, updatedAt: Long, mutation: PendingMutation) {
+        val baseVersion = checkNotNull(readItem(id)) { "Item no longer exists" }.version
         check(rename(id, title, updatedAt) == 1) { "Item no longer exists" }
-        insertPending(mutation)
+        insertPending(mutation.withBaseVersion(baseVersion))
     }
 
     @Transaction
     suspend fun setCompletedLocal(id: String, completed: Boolean, updatedAt: Long, mutation: PendingMutation) {
+        val baseVersion = checkNotNull(readItem(id)) { "Item no longer exists" }.version
         check(setCompleted(id, completed, updatedAt) == 1) { "Item no longer exists" }
-        insertPending(mutation)
+        insertPending(mutation.withBaseVersion(baseVersion))
     }
 
     @Transaction
     suspend fun deleteLocal(id: String, mutation: PendingMutation) {
+        val baseVersion = checkNotNull(readItem(id)) { "Item no longer exists" }.version
         check(delete(id) == 1) { "Item no longer exists" }
-        insertPending(mutation)
+        insertPending(mutation.withBaseVersion(baseVersion))
     }
 
     @Transaction
-    suspend fun replaceAll(items: List<ItemEntity>, refreshedAt: Long? = null): List<ItemEntity> {
+    suspend fun replaceAll(items: List<ItemEntity>, refreshedAt: Long? = null,
+                           tombstones: List<Tombstone> = emptyList()): List<ItemEntity> {
+        tombstones.forEach {
+            if (it.version >= (readTombstone(it.id)?.version ?: -1)) insertTombstone(it)
+        }
         deleteAll()
-        items.forEach { insert(it) }
+        items.forEach { item ->
+            val tombstone = readTombstone(item.id)
+            if (tombstone == null || item.version > tombstone.version) {
+                insert(item)
+                deleteTombstone(item.id)
+            }
+        }
         if (refreshedAt != null) saveSyncMetadata(SyncMetadata(lastSuccessfulRefreshAt = refreshedAt))
         // A failed write/read rolls back both canonical rows and their successful-refresh time.
         return readAll()
     }
 }
 
-@Database(entities = [ItemEntity::class, SyncMetadata::class, PendingMutation::class, AcknowledgedMutation::class],
-    version = 4, exportSchema = true)
+@Database(entities = [ItemEntity::class, SyncMetadata::class, PendingMutation::class, AcknowledgedMutation::class, Tombstone::class],
+    version = 5, exportSchema = true)
 abstract class ItemDatabase : RoomDatabase() {
     abstract fun items(): ItemDao
 
@@ -205,9 +312,23 @@ abstract class ItemDatabase : RoomDatabase() {
             }
         }
 
+        private val MIGRATION_4_5 = object : Migration(4, 5) {
+            override fun migrate(db: SupportSQLiteDatabase) {
+                db.execSQL("ALTER TABLE pending_mutations ADD COLUMN baseVersion INTEGER")
+                db.execSQL("ALTER TABLE pending_mutations ADD COLUMN dispatched INTEGER NOT NULL DEFAULT 0")
+                // M06 did not record either the original base or whether HTTP was entered.
+                // Preserve its envelopes; do not infer versions from optimistic Item rows.
+                db.execSQL("UPDATE pending_mutations SET dispatched = 1")
+                db.execSQL("UPDATE pending_mutations SET terminalError = 'base_version_unknown' " +
+                    "WHERE operation != 'CREATE' AND terminalError IS NULL")
+                db.execSQL("CREATE TABLE tombstones (`id` TEXT NOT NULL, `version` INTEGER NOT NULL, " +
+                    "`updatedAt` INTEGER NOT NULL, `deleted` INTEGER NOT NULL, PRIMARY KEY(`id`))")
+            }
+        }
+
         fun open(context: Context, name: String = NAME): ItemDatabase =
             Room.databaseBuilder(context.applicationContext, ItemDatabase::class.java, name)
-                .addMigrations(MIGRATION_1_2, MIGRATION_2_3, MIGRATION_3_4)
+                .addMigrations(MIGRATION_1_2, MIGRATION_2_3, MIGRATION_3_4, MIGRATION_4_5)
                 // Preserve existing Items; reject unknown schemas instead of erasing data.
                 .build()
     }
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemStore.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemStore.kt
index fbde32e..7d158f7 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemStore.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemStore.kt
@@ -25,9 +25,17 @@ class ItemStore(
 
     suspend fun acknowledgedMutations(): List<AcknowledgedMutation> = dao.readAcknowledged()
 
+    suspend fun tombstones(): List<Tombstone> = dao.readTombstones()
+
+    suspend fun prepareDispatch(sequence: Long): PendingMutation = dao.prepareDispatch(sequence)
+
     suspend fun acknowledge(mutation: PendingMutation, result: MutationResult) {
+        val item = if (mutation.operation == "DELETE") null else checkNotNull(result.item) {
+            "Acknowledgment has no validated canonical Item"
+        }
+        check(item == null || item.id == mutation.itemId)
         dao.acknowledge(mutation.sequence, AcknowledgedMutation(mutation.clientMutationId,
-            mutation.payloadHash, result.statusCode, result.responseBody))
+            mutation.payloadHash, result.statusCode, result.responseBody), item?.toEntity())
     }
 
     suspend fun markIdentityConflict(mutation: PendingMutation) {
@@ -36,8 +44,14 @@ class ItemStore(
         }
     }
 
-    suspend fun replaceWithCanonical(items: List<Item>, refreshedAt: Long? = null): List<Item> =
-        dao.replaceAll(items.map(Item::toEntity), refreshedAt).map(ItemEntity::toDomain)
+    suspend fun recordVersionConflict(mutation: PendingMutation, conflict: MutationVersionConflict) {
+        check(conflict.itemId == mutation.itemId)
+        dao.recordVersionConflict(mutation, conflict.item?.toEntity(), conflict.tombstone)
+    }
+
+    suspend fun replaceWithCanonical(items: List<Item>, refreshedAt: Long? = null,
+                                     tombstones: List<Tombstone> = emptyList()): List<Item> =
+        dao.replaceAll(items.map(Item::toEntity), refreshedAt, tombstones).map(ItemEntity::toDomain)
 
     suspend fun create(title: String) {
         val validTitle = title.trim().also { require(it.isNotEmpty()) }
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemSync.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemSync.kt
index 0a7bdcb..12e9f67 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemSync.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemSync.kt
@@ -6,9 +6,12 @@ import kotlinx.coroutines.flow.asStateFlow
 
 interface ItemRemote {
     suspend fun items(): List<Item>
+    suspend fun snapshot(): CanonicalSnapshot = CanonicalSnapshot(items())
     suspend fun send(mutation: PendingMutation): MutationResult
 }
 
+data class CanonicalSnapshot(val items: List<Item>, val tombstones: List<Tombstone> = emptyList())
+
 enum class SyncState { STALE, REFRESHING, FRESH, ERROR }
 
 data class SyncStatus(
@@ -29,7 +32,7 @@ class ItemSync(
     // A new session is stale until explicitly refreshed, even when a saved timestamp exists.
     suspend fun readSavedStatus() {
         mutableStatus.value = mutableStatus.value.copy(lastSuccessfulRefreshAt = store.lastSuccessfulRefreshAt())
-        store.pendingMutations().firstOrNull { it.terminalError != null }?.let {
+        store.pendingMutations().firstOrNull { it.terminalError != null && it.terminalError != "version_conflict" }?.let {
             mutableStatus.value = mutableStatus.value.copy(state = SyncState.ERROR, error = it.terminalError)
         }
     }
@@ -41,10 +44,19 @@ class ItemSync(
     suspend fun synchronize() {
         try {
             mutableStatus.value = SyncStatus(SyncState.REFRESHING, store.lastSuccessfulRefreshAt())
-            for (mutation in store.pendingMutations()) {
-                if (mutation.terminalError != null) throw MutationIdentityConflict()
+            for (queued in store.pendingMutations()) {
+                when (queued.terminalError) {
+                    "version_conflict", "base_version_unknown" -> continue
+                    null -> Unit
+                    else -> throw MutationIdentityConflict()
+                }
+                // An earlier ACK may have prepared this never-dispatched successor in Room.
+                val mutation = store.prepareDispatch(queued.sequence)
                 val result = try {
                     remote.send(mutation)
+                } catch (conflict: MutationVersionConflict) {
+                    store.recordVersionConflict(mutation, conflict)
+                    continue
                 } catch (collision: MutationIdentityConflict) {
                     store.markIdentityConflict(mutation)
                     throw collision
@@ -52,10 +64,10 @@ class ItemSync(
                 store.acknowledge(mutation, result)
             }
 
-            val canonical = remote.items()
+            val canonical = remote.snapshot()
             val refreshedAt = now()
             // Commit the canonical list and its timestamp together, then use the Room readback.
-            store.replaceWithCanonical(canonical, refreshedAt)
+            store.replaceWithCanonical(canonical.items, refreshedAt, canonical.tombstones)
             mutableStatus.value = SyncStatus(SyncState.FRESH, refreshedAt)
         } catch (cancelled: CancellationException) {
             mutableStatus.value = mutableStatus.value.copy(state = SyncState.STALE, error = null)
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
index 4dacc39..c5c3ace 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
@@ -56,6 +56,7 @@ class MainActivity : ComponentActivity() {
 internal fun ItemScreen(store: ItemStore, sync: ItemSync? = null) {
     var items by remember(store) { mutableStateOf<List<Item>?>(null) }
     var pendingCount by remember(store) { mutableStateOf(0) }
+    var conflictCount by remember(store) { mutableStateOf(0) }
     var busy by remember(store) { mutableStateOf(true) }
     var storageError by remember(store) { mutableStateOf<String?>(null) }
     var syncing by remember(store) { mutableStateOf(false) }
@@ -71,10 +72,12 @@ internal fun ItemScreen(store: ItemStore, sync: ItemSync? = null) {
             try {
                 action()
             } finally {
-                // A failed refresh may follow accepted uploads; show the remaining stored work.
-                pendingCount = store.pendingMutations().size
+                // A conflict may commit canonical data before a later refresh fails.
+                val pending = store.pendingMutations()
+                pendingCount = pending.size
+                conflictCount = pending.count { it.terminalError == "version_conflict" }
+                items = store.items()
             }
-            items = store.items()
             sync?.readSavedStatus()
             storageError = null
             after()
@@ -169,6 +172,9 @@ internal fun ItemScreen(store: ItemStore, sync: ItemSync? = null) {
         }
         if (items != null) {
             Text("Pending changes: $pendingCount", modifier = Modifier.testTag("pending-count"))
+            if (conflictCount > 0) {
+                Text("CONFLICT: remote value kept; local attempt retained.", modifier = Modifier.testTag("conflict-notice"))
+            }
             Text("Items (${rows.size})", modifier = Modifier.testTag("item-count"))
             if (rows.isEmpty()) Text("No items")
         }
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/MutationIdentity.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/MutationIdentity.kt
index 1ca46dd..b023dc5 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/MutationIdentity.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/MutationIdentity.kt
@@ -4,17 +4,27 @@ import java.io.IOException
 import java.security.MessageDigest
 import okhttp3.HttpUrl
 
-data class MutationResult(val statusCode: Int, val responseBody: String)
+data class MutationResult(val statusCode: Int, val responseBody: String, val item: Item? = null)
 
 class MutationIdentityConflict : IOException("identity_conflict")
 
+class MutationVersionConflict(val item: Item?, val tombstone: Tombstone?) : IOException("version_conflict") {
+    init {
+        require((item == null) xor (tombstone == null)) { "Expected one canonical conflict value" }
+        require(tombstone?.deleted != false) { "Expected a deleted tombstone" }
+    }
+    val itemId: String get() = item?.id ?: requireNotNull(tombstone).id
+}
+
 internal data class MutationRequest(val method: String, val path: String, val payload: Map<String, Any?>?) {
     fun canonical(): String = canonicalJson(mapOf("method" to method, "path" to path, "payload" to payload))
     fun hash(): String = MessageDigest.getInstance("SHA-256").digest(canonical().toByteArray(Charsets.UTF_8))
         .joinToString("") { (it.toInt() and 0xff).toString(16).padStart(2, '0') }
 }
 
-internal fun mutationRequest(itemId: String, operation: String, title: String?, completed: Boolean?): MutationRequest {
+internal fun mutationRequest(itemId: String, operation: String, title: String?, completed: Boolean?,
+                             baseVersion: Long? = null): MutationRequest {
+    require(baseVersion == null || (operation != "CREATE" && baseVersion >= 0))
     val method = when (operation) {
         "CREATE" -> "POST"
         "RENAME", "COMPLETE" -> "PATCH"
@@ -24,22 +34,30 @@ internal fun mutationRequest(itemId: String, operation: String, title: String?,
     // Use the same encoded path segment rules as the actual HTTP request.
     val path = HttpUrl.Builder().scheme("http").host("localhost").addPathSegment("items")
         .apply { if (operation != "CREATE") addPathSegment(itemId) }.build().encodedPath
-    val payload = when (operation) {
+    val payload: Map<String, Any?>? = when (operation) {
         "CREATE" -> mapOf("id" to itemId, "title" to requireNotNull(title), "completed" to requireNotNull(completed))
         "RENAME" -> mapOf("title" to requireNotNull(title))
         "COMPLETE" -> mapOf("completed" to requireNotNull(completed))
-        else -> null // DELETE hashes JSON null; its wire body contains only identity metadata.
+        else -> null // Preserve the original M06 envelope when no base was recorded.
     }
-    return MutationRequest(method, path, payload)
+    return MutationRequest(method, path,
+        if (baseVersion == null) payload else payload.orEmpty() + ("baseVersion" to baseVersion))
 }
 
-internal fun PendingMutation.request() = mutationRequest(itemId, operation, title, completed)
+internal fun PendingMutation.request() = mutationRequest(itemId, operation, title, completed, baseVersion)
+
+internal fun PendingMutation.withBaseVersion(version: Long): PendingMutation {
+    check(!dispatched && terminalError == null) { "A dispatched or terminal envelope cannot change" }
+    check(request().hash() == payloadHash) { "Stored mutation hash does not match its payload" }
+    return copy(baseVersion = version,
+        payloadHash = mutationRequest(itemId, operation, title, completed, version).hash())
+}
 
 internal fun pendingMutation(itemId: String, operation: String, title: String? = null, completed: Boolean? = null,
-                             clientMutationId: String, sequence: Long = 0): PendingMutation {
+                             clientMutationId: String, sequence: Long = 0, baseVersion: Long? = null): PendingMutation {
     require(clientMutationId.isNotBlank()) { "Mutation identity must not be blank" }
     return PendingMutation(sequence, itemId, operation, title, completed, clientMutationId,
-        mutationRequest(itemId, operation, title, completed).hash())
+        mutationRequest(itemId, operation, title, completed, baseVersion).hash(), baseVersion = baseVersion)
 }
 
 /** Compact JSON with literal UTF-8 text, recursive key ordering, and integer model values. */
diff --git a/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt b/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
index 7182f59..5ad9e65 100644
--- a/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
+++ b/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
@@ -192,9 +192,9 @@ class ItemStoreTest {
 
         val pending = listOf(
             pendingMutation("device-001", "CREATE", "Gamma", false, "m05-1", 1),
-            pendingMutation("remote-001", "RENAME", "Queued edit", null, "m05-2", 2),
-            pendingMutation("remote-001", "COMPLETE", null, true, "m05-3", 3),
-            pendingMutation("remote-002", "DELETE", null, null, "m05-4", 4),
+            pendingMutation("remote-001", "RENAME", "Queued edit", null, "m05-2", 2, baseVersion = 1),
+            pendingMutation("remote-001", "COMPLETE", null, true, "m05-3", 3, baseVersion = 1),
+            pendingMutation("remote-002", "DELETE", null, null, "m05-4", 4, baseVersion = 1),
         )
         val local = listOf(
             Item("remote-001", "Queued edit", true, 1, 1700000302000L),
@@ -210,7 +210,7 @@ class ItemStoreTest {
         assertEquals(pending, reopened.pendingMutations())
         assertThrows(IOException::class.java) { runBlocking { sync.synchronize() } }
         assertEquals(local, reopened.items())
-        assertEquals(pending, reopened.pendingMutations())
+        assertEquals(listOf(pending.first().copy(dispatched = true)) + pending.drop(1), reopened.pendingMutations())
         assertEquals(listOf("GET"), remote.calls)
 
         remote.offline = false
@@ -222,8 +222,13 @@ class ItemStoreTest {
         assertEquals(canonical, reopened.items())
         assertEquals(canonical, remote.rows.values.sortedBy(Item::id))
         assertTrue(reopened.pendingMutations().isEmpty())
-        assertEquals(pending.map { it.clientMutationId to it.payloadHash },
+        val accepted = pending.toMutableList().apply {
+            this[2] = pendingMutation("remote-001", "COMPLETE", null, true, "m05-3", 3, baseVersion = 2)
+        }
+        assertEquals(accepted.map { it.clientMutationId to it.payloadHash },
             reopened.acknowledgedMutations().map { it.clientMutationId to it.payloadHash })
+        assertEquals(listOf(null, 1L, 2L, 1L), remote.mutations.map { it.baseVersion })
+        assertEquals(accepted.map { it.copy(dispatched = true) }, remote.mutations)
         assertEquals(1700000304000L, remote.time)
         assertEquals(listOf("GET", "POST device-001", "PATCH remote-001 title=Queued edit completed=null",
             "PATCH remote-001 title=null completed=true", "DELETE remote-002", "GET"), remote.calls)
@@ -257,7 +262,8 @@ class ItemStoreTest {
         val before = store.pendingMutations()
         val sync = ItemSync(store, remote)
         assertThrows(IOException::class.java) { runBlocking { sync.synchronize() } }
-        assertEquals(before, store.pendingMutations())
+        val dispatched = before.map { it.copy(dispatched = true) }
+        assertEquals(dispatched, store.pendingMutations())
         assertTrue(store.acknowledgedMutations().isEmpty())
         assertEquals(1, remote.applied)
         val original = remote.responses.getValue("m06-create-001").second
@@ -267,7 +273,7 @@ class ItemStoreTest {
         assertTrue(reopened.pendingMutations().isEmpty())
         assertEquals(listOf(AcknowledgedMutation("m06-create-001", before.single().payloadHash,
             original.statusCode, original.responseBody)), reopened.acknowledgedMutations())
-        assertEquals(listOf(before.single(), before.single()), remote.mutations)
+        assertEquals(listOf(dispatched.single(), dispatched.single()), remote.mutations)
         assertEquals(listOf("POST crash-001", "POST crash-001", "GET"), remote.calls)
         assertEquals(1, remote.applied)
         assertEquals(1, remote.duplicates)
@@ -285,7 +291,7 @@ class ItemStoreTest {
         val local = store.items()
         val sync = ItemSync(store, remote)
         assertThrows(IOException::class.java) { runBlocking { sync.synchronize() } }
-        assertEquals(pending, store.pendingMutations())
+        assertEquals(pending.map { it.copy(dispatched = true) }, store.pendingMutations())
         assertEquals(local, store.items())
         assertTrue(store.acknowledgedMutations().isEmpty())
         assertEquals(listOf("ack"), dao.acknowledgmentWrites) // Dequeue was never reached.
@@ -329,6 +335,219 @@ class ItemStoreTest {
         assertEquals(1700000401000L, remote.time)
     }
 
+    @Test
+    fun m07FixedConflictsRetainTheirEnvelopesAndOnlyFreshExplicitEditIsSent() = runBlocking {
+        for ((case, operation, identity) in listOf(
+            Triple("A", "RENAME", "m07-update-001"),
+            Triple("B", "DELETE", "m07-delete-001"),
+            Triple("C", "RENAME", "m07-deleted-001"),
+        )) {
+            val dao = FakeItemDao()
+            val remote = conflictRemote()
+            val store = ItemStore(dao, now = { 1700000500000L }, nextMutationId = { identity })
+            val sync = ItemSync(store, remote, now = { 1700000500000L })
+            sync.synchronize()
+            remote.offline = true
+            if (operation == "RENAME") store.rename("conflict-001", "Local attempt")
+            else store.delete("conflict-001")
+            val original = pendingMutation("conflict-001", operation,
+                title = if (operation == "RENAME") "Local attempt" else null,
+                clientMutationId = identity, sequence = 1, baseVersion = 1)
+            assertEquals(listOf(original), store.pendingMutations())
+            assertTrue(remote.mutations.isEmpty())
+            if (case == "C") remote.deleteRemote("conflict-001")
+            else remote.patch("conflict-001", "Remote winner", null)
+            remote.offline = false
+            sync.synchronize()
+
+            val conflict = original.copy(dispatched = true, terminalError = "version_conflict")
+            val winner = Item("conflict-001", "Remote winner", false, 2, 1700000500000L)
+            val tombstone = Tombstone("conflict-001", 2, 1700000500000L)
+            val canonicalItems = if (case == "C") emptyList() else listOf(winner)
+            val canonicalTombstones = if (case == "C") listOf(tombstone) else emptyList()
+            assertEquals(canonicalItems, store.items())
+            assertEquals(canonicalTombstones, store.tombstones())
+            assertEquals(listOf(conflict), store.pendingMutations())
+            assertTrue(store.acknowledgedMutations().isEmpty())
+            assertEquals(listOf(original.copy(dispatched = true)), remote.mutations)
+            assertEquals(0, remote.applied)
+            assertEquals(1, remote.versionConflicts)
+
+            val reopened = ItemStore(dao, now = { 1700000500000L }, nextMutationId = { "m07-explicit-001" })
+            val resumed = ItemSync(reopened, remote, now = { 1700000500000L })
+            resumed.readSavedStatus()
+            assertEquals(listOf(conflict), reopened.pendingMutations())
+            assertEquals(canonicalItems, reopened.items())
+            assertEquals(canonicalTombstones, reopened.tombstones())
+            resumed.synchronize()
+            assertEquals(SyncState.FRESH, resumed.status.value.state)
+            assertEquals(listOf(conflict), reopened.pendingMutations())
+            assertEquals(1, remote.mutations.size) // Ordinary drains may GET but never resend the conflict.
+
+            if (case == "A") {
+                reopened.rename("conflict-001", "Explicit edit")
+                val explicit = pendingMutation("conflict-001", "RENAME", "Explicit edit",
+                    clientMutationId = "m07-explicit-001", sequence = 2, baseVersion = 2)
+                assertEquals(listOf(conflict, explicit), reopened.pendingMutations())
+                resumed.synchronize()
+                val edited = winner.copy(title = "Explicit edit", version = 3, updatedAt = 1700000501000L)
+                assertEquals(listOf(edited), reopened.items())
+                assertTrue(reopened.tombstones().isEmpty())
+                assertEquals(listOf(conflict), reopened.pendingMutations())
+                assertEquals(listOf(original.copy(dispatched = true), explicit.copy(dispatched = true)), remote.mutations)
+                val result = remote.responses.getValue(explicit.clientMutationId).second
+                assertEquals(listOf(AcknowledgedMutation(explicit.clientMutationId, explicit.payloadHash,
+                    result.statusCode, result.responseBody)), reopened.acknowledgedMutations())
+                assertEquals(1, remote.applied)
+                assertEquals(1700000502000L, remote.time)
+            } else {
+                assertEquals(1700000501000L, remote.time)
+            }
+            assertEquals(1, remote.versionConflicts)
+            assertEquals(0, remote.duplicates)
+            assertEquals(0, remote.identityConflicts)
+        }
+    }
+
+    @Test
+    fun m07OwnAcknowledgmentPreparesSuccessorAcrossNewStoresAndLostPatchReplyReplaysExactly(): Unit = runBlocking {
+        val dao = FakeItemDao()
+        val ids = listOf("m07-own-001", "m07-own-002").iterator()
+        var time = 1700000500000L
+        val store = ItemStore(dao, now = { time.also { time += 1000 } }, nextMutationId = { ids.next() })
+        val remote = conflictRemote()
+        store.replaceWithCanonical(remote.rows.values.toList())
+        store.rename("conflict-001", "Queued edit")
+        store.setCompleted("conflict-001", true)
+        val queued = store.pendingMutations()
+        val optimistic = store.items().single()
+        assertEquals(listOf(1L, 1L), queued.map { it.baseVersion })
+
+        val first = requireNotNull(store.prepareDispatch(queued.first().sequence))
+        val firstResult = remote.send(first)
+        store.acknowledge(first, firstResult)
+        val successor = pendingMutation("conflict-001", "COMPLETE", completed = true,
+            clientMutationId = queued[1].clientMutationId, sequence = queued[1].sequence, baseVersion = 2)
+        assertEquals(listOf(successor), store.pendingMutations())
+        assertFalse(successor.payloadHash == queued[1].payloadHash)
+        assertEquals(listOf(optimistic.copy(version = 2)), store.items())
+
+        val reopened = ItemStore(dao, nextMutationId = { error("Replay must keep the existing identity") })
+        assertEquals(listOf(successor), reopened.pendingMutations())
+        remote.dropNextPatchResponse = true
+        assertThrows(IOException::class.java) { runBlocking { ItemSync(reopened, remote).synchronize() } }
+        val sent = successor.copy(dispatched = true)
+        assertEquals(listOf(sent), reopened.pendingMutations())
+        assertEquals(listOf(optimistic.copy(version = 2)), reopened.items())
+        val successorResult = remote.responses.getValue(sent.clientMutationId).second
+        assertEquals(2, remote.applied)
+
+        val afterLoss = ItemStore(dao, nextMutationId = { error("Replay must keep the existing identity") })
+        assertEquals(listOf(sent), afterLoss.pendingMutations())
+        ItemSync(afterLoss, remote, now = { 1700000501000L }).synchronize()
+        assertTrue(afterLoss.pendingMutations().isEmpty())
+        assertEquals(listOf(requireNotNull(successorResult.item)), afterLoss.items())
+        assertEquals(listOf(first, sent, sent), remote.mutations)
+        assertEquals(listOf(
+            AcknowledgedMutation(first.clientMutationId, first.payloadHash, firstResult.statusCode, firstResult.responseBody),
+            AcknowledgedMutation(sent.clientMutationId, sent.payloadHash, successorResult.statusCode, successorResult.responseBody),
+        ), afterLoss.acknowledgedMutations())
+        assertEquals(2, remote.applied)
+        assertEquals(1, remote.duplicates)
+        assertEquals(0, remote.versionConflicts)
+        assertEquals(1700000502000L, remote.time)
+    }
+
+    @Test
+    fun m07ExternalVersionThreeDoesNotRebaseSuccessorOrGetOverwrittenByOlderAcknowledgment() = runBlocking {
+        val dao = FakeItemDao()
+        val ids = listOf("m07-own-001", "m07-own-002").iterator()
+        val store = ItemStore(dao, now = { 1700000500000L }, nextMutationId = { ids.next() })
+        val remote = conflictRemote()
+        store.replaceWithCanonical(remote.rows.values.toList())
+        store.rename("conflict-001", "Queued edit")
+        store.setCompleted("conflict-001", true)
+        val queued = store.pendingMutations()
+        val first = requireNotNull(store.prepareDispatch(queued.first().sequence))
+        val olderAcknowledgment = remote.send(first)
+        val external = remote.patch("conflict-001", "External v3", null)
+        store.replaceWithCanonical(listOf(external))
+        store.acknowledge(first, olderAcknowledgment)
+
+        val prepared = pendingMutation("conflict-001", "COMPLETE", completed = true,
+            clientMutationId = queued[1].clientMutationId, sequence = queued[1].sequence, baseVersion = 2)
+        assertEquals(listOf(external), store.items())
+        assertEquals(listOf(prepared), store.pendingMutations())
+        val reopened = ItemStore(dao)
+        ItemSync(reopened, remote).synchronize()
+        assertEquals(listOf(prepared.copy(dispatched = true, terminalError = "version_conflict")), reopened.pendingMutations())
+        assertEquals(listOf(external), reopened.items())
+        assertEquals(listOf(1L, 2L), remote.mutations.map { it.baseVersion })
+        assertEquals(1, remote.applied)
+        assertEquals(1, remote.versionConflicts)
+        assertEquals(1700000502000L, remote.time)
+    }
+
+    @Test
+    fun m07BaseOnlyIdentityCollisionRemainsTerminalBeforeVersionChecks(): Unit = runBlocking {
+        val remote = conflictRemote()
+        val original = pendingMutation("conflict-001", "RENAME", "Local attempt",
+            clientMutationId = "m07-update-001", baseVersion = 1)
+        val accepted = remote.send(original)
+        val dao = FakeItemDao()
+        val store = ItemStore(dao, now = { 1700000500000L }, nextMutationId = { original.clientMutationId })
+        store.replaceWithCanonical(listOf(requireNotNull(accepted.item)))
+        store.rename("conflict-001", "Local attempt")
+        val collision = store.pendingMutations().single()
+        assertEquals(2L, collision.baseVersion)
+        assertEquals(original.title, collision.title)
+        assertFalse(original.payloadHash == collision.payloadHash)
+        assertThrows(MutationIdentityConflict::class.java) { runBlocking { ItemSync(store, remote).synchronize() } }
+        val terminal = collision.copy(dispatched = true, terminalError = "identity_conflict")
+        assertEquals(listOf(terminal), store.pendingMutations())
+        assertTrue(store.acknowledgedMutations().isEmpty())
+        val reopened = ItemStore(dao)
+        assertThrows(MutationIdentityConflict::class.java) { runBlocking { ItemSync(reopened, remote).synchronize() } }
+        assertEquals(listOf(terminal), reopened.pendingMutations())
+        assertEquals(2, remote.mutations.size)
+        assertEquals(1, remote.applied)
+        assertEquals(1, remote.identityConflicts)
+        assertEquals(0, remote.versionConflicts)
+        assertEquals(0, remote.duplicates)
+        assertEquals(1700000501000L, remote.time)
+    }
+
+    @Test
+    fun m07UnknownLegacyBaseRetainsItsOriginalEnvelopeAndIsExcludedFromOrdinaryDrains() = runBlocking {
+        val dao = FakeItemDao()
+        val legacy = pendingMutation("remote-001", "RENAME", "Legacy local edit",
+            clientMutationId = "legacy-update-001", sequence = 4)
+            .copy(dispatched = true, terminalError = "base_version_unknown")
+        dao.insert(ItemEntity("remote-001", "Legacy local edit", false, 1, 1700000500000L))
+        dao.insertPending(legacy)
+        val remote = FakeRemote()
+        val store = ItemStore(dao)
+        val sync = ItemSync(store, remote)
+        sync.readSavedStatus()
+        assertEquals("base_version_unknown", sync.status.value.error)
+        sync.synchronize()
+        assertEquals(listOf(legacy), store.pendingMutations())
+        assertEquals(legacy.request().hash(), legacy.payloadHash)
+        assertTrue(store.acknowledgedMutations().isEmpty())
+        assertTrue(remote.mutations.isEmpty())
+        val reopened = ItemStore(dao)
+        ItemSync(reopened, remote).synchronize()
+        assertEquals(listOf(legacy), reopened.pendingMutations())
+        assertEquals(listOf("GET", "GET"), remote.calls)
+        assertEquals(0, remote.applied)
+    }
+
+    private fun conflictRemote() = FakeRemote().apply {
+        rows.clear()
+        rows["conflict-001"] = Item("conflict-001", "Initial", false, 1, 1700000000000L)
+        time = 1700000500000L
+    }
+
     private class FakeRemote : ItemRemote {
         val rows = linkedMapOf(
             "remote-001" to Item("remote-001", "Alpha", false, 1, 1700000000000L),
@@ -341,13 +560,18 @@ class ItemStoreTest {
         var beforeMutation: () -> Unit = {}
         var time = 1700000100000L
         var dropNextCreateResponse = false
+        var dropNextPatchResponse = false
         var applied = 0
         var duplicates = 0
         var identityConflicts = 0
+        var versionConflicts = 0
         val mutations = mutableListOf<PendingMutation>()
         val responses = linkedMapOf<String, Pair<String, MutationResult>>()
+        private val conflicts = linkedMapOf<String, Pair<String, MutationVersionConflict>>()
+        private val tombstones = linkedMapOf<String, Tombstone>()
         private fun tick() = time.also { time += 1000 }
-        override suspend fun items(): List<Item> {
+        override suspend fun items(): List<Item> = snapshot().items
+        override suspend fun snapshot(): CanonicalSnapshot {
             beforeGet()
             if (offline) throw IOException("Forced offline")
             calls += "GET"
@@ -355,7 +579,7 @@ class ItemStoreTest {
                 getFailures--
                 throw IOException("Sync HTTP 503")
             }
-            return rows.values.sortedBy(Item::id)
+            return CanonicalSnapshot(rows.values.sortedBy(Item::id), tombstones.values.sortedBy(Tombstone::id))
         }
         override suspend fun send(mutation: PendingMutation): MutationResult {
             beforeMutation()
@@ -375,6 +599,23 @@ class ItemStoreTest {
                 duplicates++
                 return original.second
             }
+            conflicts[mutation.clientMutationId]?.let { original ->
+                if (original.first != request.canonical()) {
+                    identityConflicts++
+                    throw MutationIdentityConflict()
+                }
+                duplicates++
+                throw original.second
+            }
+            if (mutation.operation != "CREATE" && mutation.baseVersion != null) {
+                val currentVersion = rows[mutation.itemId]?.version ?: tombstones[mutation.itemId]?.version
+                if (currentVersion != null && currentVersion != mutation.baseVersion) {
+                    versionConflicts++
+                    val conflict = MutationVersionConflict(rows[mutation.itemId], tombstones[mutation.itemId])
+                    conflicts[mutation.clientMutationId] = request.canonical() to conflict
+                    throw conflict
+                }
+            }
             val result = when (mutation.operation) {
                 "CREATE" -> {
                     check(mutation.itemId !in rows)
@@ -384,8 +625,7 @@ class ItemStoreTest {
                 }
                 "RENAME", "COMPLETE" -> itemResponse(200, patchRow(mutation.itemId, mutation.title, mutation.completed))
                 "DELETE" -> {
-                    check(rows.remove(mutation.itemId) != null)
-                    tick()
+                    deleteRow(mutation.itemId)
                     MutationResult(200, canonicalJson(mapOf("deletedId" to mutation.itemId)))
                 }
                 else -> error("Unknown test operation")
@@ -396,6 +636,10 @@ class ItemStoreTest {
                 dropNextCreateResponse = false
                 throw IOException("Lost response")
             }
+            if (dropNextPatchResponse && mutation.operation in listOf("RENAME", "COMPLETE")) {
+                dropNextPatchResponse = false
+                throw IOException("Lost response")
+            }
             return result
         }
 
@@ -406,6 +650,17 @@ class ItemStoreTest {
             return patchRow(id, title, completed)
         }
 
+        fun deleteRemote(id: String) {
+            beforeMutation()
+            calls += "DELETE $id"
+            deleteRow(id)
+        }
+
+        private fun deleteRow(id: String) {
+            val item = requireNotNull(rows.remove(id))
+            tombstones[id] = Tombstone(id, item.version + 1, tick())
+        }
+
         private fun patchRow(id: String, title: String?, completed: Boolean?): Item {
             val item = rows.getValue(id)
             return item.copy(title = title ?: item.title, completed = completed ?: item.completed,
@@ -414,7 +669,7 @@ class ItemStoreTest {
 
         private fun itemResponse(status: Int, item: Item) = MutationResult(status, canonicalJson(mapOf("item" to mapOf(
             "id" to item.id, "title" to item.title, "completed" to item.completed, "version" to item.version, "updatedAt" to item.updatedAt,
-        ))))
+        ))), item)
     }
 
     private class FakeItemDao : ItemDao {
@@ -423,11 +678,38 @@ class ItemStoreTest {
         private val pending = linkedMapOf<Long, PendingMutation>()
         private var nextSequence = 1L
         private val acknowledged = linkedMapOf<String, AcknowledgedMutation>()
+        private val tombstones = linkedMapOf<String, Tombstone>()
         var rejectAcknowledgment = false
         val acknowledgmentWrites = mutableListOf<String>()
 
         override suspend fun readPending(): List<PendingMutation> = pending.values.sortedBy { it.sequence }
 
+        override suspend fun readItem(id: String): ItemEntity? = rows[id]
+
+        override suspend fun readPendingBySequence(sequence: Long): PendingMutation? = pending[sequence]
+
+        override suspend fun markDispatched(sequence: Long, clientMutationId: String, payloadHash: String): Int {
+            val current = pending[sequence] ?: return 0
+            if (current.clientMutationId != clientMutationId || current.payloadHash != payloadHash || current.terminalError != null) return 0
+            pending[sequence] = current.copy(dispatched = true)
+            return 1
+        }
+
+        override suspend fun updatePreparedMutation(sequence: Long, clientMutationId: String, previousHash: String,
+                                                    baseVersion: Long, payloadHash: String): Int {
+            val current = pending[sequence] ?: return 0
+            if (current.clientMutationId != clientMutationId || current.payloadHash != previousHash ||
+                current.dispatched || current.terminalError != null) return 0
+            pending[sequence] = current.copy(baseVersion = baseVersion, payloadHash = payloadHash)
+            return 1
+        }
+
+        override suspend fun advanceObservedVersion(id: String, version: Long): Int {
+            val current = rows[id] ?: return 0
+            rows[id] = current.copy(version = maxOf(current.version, version))
+            return 1
+        }
+
         override suspend fun insertPending(mutation: PendingMutation) {
             val sequence = if (mutation.sequence == 0L) nextSequence++ else mutation.sequence
             check(sequence !in pending)
@@ -458,6 +740,28 @@ class ItemStoreTest {
             return 1
         }
 
+        override suspend fun markVersionConflict(sequence: Long, clientMutationId: String, payloadHash: String): Int {
+            val current = pending[sequence] ?: return 0
+            if (current.clientMutationId != clientMutationId || current.payloadHash != payloadHash ||
+                !current.dispatched || current.terminalError != null) return 0
+            pending[sequence] = current.copy(terminalError = "version_conflict")
+            return 1
+        }
+
+        override suspend fun readTombstones(): List<Tombstone> = tombstones.values.sortedBy(Tombstone::id)
+
+        override suspend fun readTombstone(id: String): Tombstone? = tombstones[id]
+
+        override suspend fun insertTombstone(tombstone: Tombstone) {
+            tombstones[tombstone.id] = tombstone
+        }
+
+        override suspend fun deleteTombstone(id: String): Int = if (tombstones.remove(id) == null) 0 else 1
+
+        override suspend fun upsertCanonical(item: ItemEntity) {
+            rows[item.id] = item
+        }
+
         override suspend fun lastSuccessfulRefreshAt(): Long? = lastRefresh
 
         override suspend fun saveSyncMetadata(metadata: SyncMetadata) {
diff --git a/verification/M07.md b/verification/M07.md
index 64808e8..55a846e 100644
--- a/verification/M07.md
+++ b/verification/M07.md
@@ -1,6 +1,6 @@
 # M07 — phase-1 version conflicts
 
-Status: baseline independently accepted; implementation/final verification pending.
+Status: baseline independently accepted; host/build and main official runtime PASS; commit audit pending.
 START `425a0fbc05b7802c8e788f52aba59a4274d7a725` (verified phase-1 M06).
 SPEC_REVISION `61280dd86ce88b6e431f408241c0998a275960aa`; attempt 1, repair 0/2.
 
@@ -12,7 +12,7 @@ Evidence root: `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/a
 44 files, five support changes/copies, unchanged M06 app `0857d612…66fa79a`, and
 test-only launcher APK `03df92d8…a178d30`. No product or existing JUnit scenario changed.
 
-[Recorded commands](../../evidence/phase-1/android-kotlin/M07/commands.jsonl): eight invocations;
+[Recorded commands](../../evidence/phase-1/android-kotlin/M07/commands.jsonl): eight baseline invocations;
 fixture 10/10 PASS (five old tests unchanged), runner-only build PASS, static checks PASS.
 Port checks returned expected exit 1 with no listener; other recorded commands exited 0.
 [One actual baseline](../../evidence/phase-1/android-kotlin/M07/baseline-android-01/result.json):
@@ -30,3 +30,54 @@ network 0/1/1 with active network 115. All frozen bytes unchanged.
 Main independently accepted [raw evidence](</Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M07/main-baseline-audit.json>)
 and [cleanup](</Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M07/main-baseline-cleanup.json>).
 Final Android execution remains main-owned; no owner final device run is authorized.
+
+## Initial candidate
+
+Baseline support commit: `f8d0e6c` (linear from START, current revision/profile trailers).
+Schema 5 adds saved base/dispatch metadata and tombstones. Conflict canonical writes and
+original intent retention are atomic; ordinary drains exclude conflicts. ACK/dequeue,
+observed-version advance and preparation of the immediate unsent successor are one
+transaction. Old envelopes are never rebased from external state or guessed during migration.
+
+[fixed-build01](../../evidence/phase-1/android-kotlin/M07/fixed-build01.json) PASS:
+`testDebugUnitTest assembleDebug assembleDebugAndroidTest`; host 16/16, zero failures,
+errors or skips. No fixed-build failure or retry. Compilation of the 20 Android methods
+was not counted as runtime evidence. Existing six UI scenarios, all fixture/input bytes and historical
+M05/M06 helpers remain unchanged. The fixture's earlier 10/10 checks were not rerun.
+
+The frozen handoff separates the 20-method suite from actual-process M07 A/B/C.
+The real-HTTP native M05 regression requires exactly four accepts with bases none/1/2/1.
+The lost-ACK PATCH regression drops the response after real server acceptance, then reopens
+Room in the same process and checks identical replay/original receipt and applied/duplicate
+counts. It does not substitute for, or claim a rerun of, the historical external M06 scenario.
+
+[Final freeze](../../evidence/phase-1/android-kotlin/M07/fixed-built-01/manifest.json) records
+all 46 source hashes, 11 changed-file copies, host XML, and exact main-owned fixture,
+20-method JUnit and M07 A/B/C commands. App SHA-256 `2ffa468609860d494ce2d58e38cf44aa1e75cb113474f4ef1e15ca055926a13d`;
+test SHA-256 `fb99cb3fee49bd33b10c7052a978681e6a2407908abd50055310378c28c14a7c`.
+All execution bytes remained unchanged through main's official run. The only post-run
+documentation exceptions are `TRACK.md` and this report; no test/build/device rerun occurred.
+
+## Main official verification
+
+[Native audit](</Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M07/main-native-audit.json>):
+20/20 Android methods PASS, zero failures/skips (JUnit 33.737 s), including the unchanged
+six UI tests. Real HTTP M05 accepted exactly POST/PATCH/PATCH/DELETE with bases none/1/2/1.
+The same-process Room-reopen lost-ACK regression retained the prepared PATCH identity,
+base and hash, replayed the original receipt, and finished with applied4/duplicate1.
+Native transaction rollback and schema migration checks also passed.
+
+[Actual-process audit](</Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M07/main-runtime-audit.json>):
+one A/B/C run PASS, 339 adb commands, 17 native SQLite/WAL snapshots and 873 raw files.
+A: PID 16628 → absent → 17136 (138 commands); B: 17693 → absent → 18121 (96);
+C: 18435 → absent → 18942 (105). All three received the exact 409 canonical Item/tombstone,
+retained the original conflict across actual process loss, and sent it zero times during
+ordinary drains. A's fresh explicit edit used a new identity/base2 and reached
+`Explicit edit`/false/v3/1700000501000. Expected parked-controller terminations are separate
+from the passing JUnit suite. Main checked all 46 source hashes and both APKs unchanged.
+Historical M05/M06 external scripts were not rerun on schema 5.
+
+[Cleanup audit](</Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M07/main-cleanup-audit.json>):
+owned fixture 18619 exited 0; direct process/port checks prove PID absent and
+18080 free. App PID is absent; network restored to 0/1/1. Attempt1, repair0/2; no failed
+official M07 run or repair. Main owns the final history/byte audit and progress tag.
