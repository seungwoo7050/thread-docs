## `feat: schedule durable sync with bounded persistent work`

diff --git a/TRACK.md b/TRACK.md
index a46252a..97234bc 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -243,7 +243,7 @@ The fixture12 baseline results and test APK were reused unchanged. Main confirme
 frozen execution bytes and actual cleanup. Attempt1, repair0/2; only this reporting and
 `verification/M09.md` changed after runtime. Final history/tag audit remains main-owned.
 
-## M10 boundary — baseline accepted
+## M10 boundary — official runtime passed
 
 Main accepted the unchanged M09 app's foreground-only limitation after two preserved
 harness failures. The repaired baseline used an explicitly labelled native preseed, then
@@ -251,8 +251,34 @@ ordinary offline startup, HOME and same-UID SIGKILL. The original queued identit
 durable while the package stayed unstopped and absent through ten seconds online, with
 no WorkManager database, registered job or application HTTP request. This is not production
 creation or scheduler acceptance. `verification/M10.md` retains A1/A2 failures, the accepted
-A3 evidence and the user's subsequent repair/verification authorization. M10 implementation
-and both final OS cases remain outstanding; no M10 completion or tag is claimed.
+A3 evidence and the user's subsequent repair/verification authorization.
+
+WorkManager 2.9.1 now registers one persistent unique CONNECTED work chain after a committed
+edit and during ordinary startup recovery. Room schema6 stores its cycle identity, actual
+HTTP reservations and terminal state. Reservation and dispatch marking share a transaction;
+at most three automatic HTTP attempts are allowed per cycle, including after process loss.
+Retries use exponential10/20-second backoff. An accepted response atomically commits its
+canonical Item, original receipt and dequeue without a GET. Existing immutable envelopes,
+own-ACK successor preparation and terminal conflict behavior remain intact. A scheduling
+failure after a successful save is reported separately and does not turn it into a failed edit.
+
+The debug build's clock/identity determinism requires an explicit verification file; release
+behavior uses ordinary clocks and identities. Final creation uses the real UI and committed
+queue, with no seed, instrumentation or app entrypoint after HOME/same-UID SIGKILL and before
+the OS creates the replacement process through SystemJobService.
+
+Main passed A6 (503/503/201, applied1, canonical version1, pending0) and B1 (503x3,
+durable DEFERRED/pending1, no fourth request after later OS eligibility and ordinary UI).
+Both retain one WorkSpec and the original mutation identity; main read37 native snapshots.
+Host28 and fixture14 passed. Main also passed all24 native regressions and the actual M08
+same-process lifecycle case with automatic scheduling enabled and real radios offline:
+six snapshots prove no draft write before Save and one queued rename afterward, with no HTTP.
+
+A4's numeric-job-ID assumption and A5's premature connectivity readiness failure are preserved.
+Their harness-only repairs keep the original deadlines, current native job binding and nonforced
+OS commands. Attempt5/cumulative repair4, actual A6/B1 usage, follows the explicit unlimited
+necessary-repair authorization; the product HTTP limit remains3. Main verified frozen bytes
+and actual cleanup. Only reporting changed after runtime; final history/tag audit remains main-owned.
 
 ## Pinned build
 
@@ -260,6 +286,7 @@ and both final OS cases remain outstanding; no M10 completion or tag is claimed.
 - Android Gradle Plugin 8.6.1, Kotlin 1.9.24, Compose compiler 1.5.14
 - Compose BOM 2024.06.00, Activity Compose 1.9.0
 - Room runtime/KT extensions/compiler 2.6.1, Kotlin kapt 1.9.24
+- WorkManager runtime/KT extensions 2.9.1
 - OkHttp 4.12.0; Python 3 standard library for the local fixture
 - Compile SDK 35, target SDK 34, minimum SDK 26
 - Required verification device: API 34, Pixel 6, ARM64 Google APIs image, serial `emulator-5554`
diff --git a/app/build.gradle.kts b/app/build.gradle.kts
index e172ae5..89bde97 100644
--- a/app/build.gradle.kts
+++ b/app/build.gradle.kts
@@ -39,6 +39,7 @@ dependencies {
     implementation("androidx.room:room-runtime:2.6.1")
     implementation("androidx.room:room-ktx:2.6.1")
     implementation("com.squareup.okhttp3:okhttp:4.12.0")
+    implementation("androidx.work:work-runtime-ktx:2.9.1")
     kapt("androidx.room:room-compiler:2.6.1")
 
     testImplementation("junit:junit:4.13.2")
diff --git a/app/schemas/com.mobilesystemsevolution.kotlin.ItemDatabase/6.json b/app/schemas/com.mobilesystemsevolution.kotlin.ItemDatabase/6.json
new file mode 100644
index 0000000..de63ca6
--- /dev/null
+++ b/app/schemas/com.mobilesystemsevolution.kotlin.ItemDatabase/6.json
@@ -0,0 +1,273 @@
+{
+  "formatVersion": 1,
+  "database": {
+    "version": 6,
+    "identityHash": "80ec301667bc66507ce1e1f23eefd5ce",
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
+      },
+      {
+        "tableName": "automatic_sync",
+        "createSql": "CREATE TABLE IF NOT EXISTS `${TABLE_NAME}` (`id` INTEGER NOT NULL, `workId` TEXT NOT NULL, `httpAttempts` INTEGER NOT NULL, `state` TEXT NOT NULL, PRIMARY KEY(`id`))",
+        "fields": [
+          {
+            "fieldPath": "id",
+            "columnName": "id",
+            "affinity": "INTEGER",
+            "notNull": true
+          },
+          {
+            "fieldPath": "workId",
+            "columnName": "workId",
+            "affinity": "TEXT",
+            "notNull": true
+          },
+          {
+            "fieldPath": "httpAttempts",
+            "columnName": "httpAttempts",
+            "affinity": "INTEGER",
+            "notNull": true
+          },
+          {
+            "fieldPath": "state",
+            "columnName": "state",
+            "affinity": "TEXT",
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
+      "INSERT OR REPLACE INTO room_master_table (id,identity_hash) VALUES(42, '80ec301667bc66507ce1e1f23eefd5ce')"
+    ]
+  }
+}
\ No newline at end of file
diff --git a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemDatabaseTest.kt b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemDatabaseTest.kt
index 21ae91a..74b969b 100644
--- a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemDatabaseTest.kt
+++ b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemDatabaseTest.kt
@@ -71,7 +71,7 @@ class ItemDatabaseTest {
         }
         withDatabase { reopened ->
             assertEquals(expected, ItemStore(reopened.items()).items())
-            assertEquals(5, reopened.openHelper.readableDatabase.version)
+            assertEquals(6, reopened.openHelper.readableDatabase.version)
         }
     }
 
@@ -151,7 +151,7 @@ class ItemDatabaseTest {
         withDatabase { migrated ->
             val store = ItemStore(migrated.items())
             assertEquals(seeds, store.items())
-            assertEquals(5, migrated.openHelper.readableDatabase.version)
+            assertEquals(6, migrated.openHelper.readableDatabase.version)
             assertNull(store.lastSuccessfulRefreshAt())
             store.replaceWithCanonical(seeds, 1700000200000L)
         }
@@ -193,7 +193,7 @@ class ItemDatabaseTest {
         }
         withDatabase { reopened ->
             val store = ItemStore(reopened.items())
-            assertEquals(5, reopened.openHelper.readableDatabase.version)
+            assertEquals(6, reopened.openHelper.readableDatabase.version)
             assertEquals(expectedRows, store.items())
             assertEquals(expectedPending, store.pendingMutations())
             assertEquals(1700000300000L, store.lastSuccessfulRefreshAt())
@@ -293,7 +293,7 @@ class ItemDatabaseTest {
         }
         withDatabase { migrated ->
             val store = ItemStore(migrated.items())
-            assertEquals(5, migrated.openHelper.readableDatabase.version)
+            assertEquals(6, migrated.openHelper.readableDatabase.version)
             assertEquals(rows, store.items())
             identifiedPending = store.pendingMutations()
             assertEquals(expectedPayload, identifiedPending.map { listOf(it.sequence, it.itemId, it.operation, it.title, it.completed) })
@@ -405,7 +405,7 @@ class ItemDatabaseTest {
             withDatabase { database ->
                 val store = ItemStore(database.items(), nextId = { "new-001" }, now = { 1700000401000L },
                     nextMutationId = { "m06-after-migration" })
-                assertEquals(5, database.openHelper.readableDatabase.version)
+                assertEquals(6, database.openHelper.readableDatabase.version)
                 val queued = store.pendingMutations()
                 assertEquals(if (hasPending) legacyPayloads else emptyList<List<Any?>>(),
                     queued.map { listOf(it.sequence, it.itemId, it.operation, it.title, it.completed) })
@@ -625,7 +625,7 @@ class ItemDatabaseTest {
         withDatabase { migrated ->
             val store = ItemStore(migrated.items(), nextId = { "new-001" }, now = { 1700000501000L },
                 nextMutationId = { "new-after-migration" })
-            assertEquals(5, migrated.openHelper.readableDatabase.version)
+            assertEquals(6, migrated.openHelper.readableDatabase.version)
             assertEquals(expected, store.pendingMutations())
             assertEquals(listOf(receipt), store.acknowledgedMutations())
             assertEquals(listOf(row), store.items())
@@ -649,6 +649,120 @@ class ItemDatabaseTest {
         }
     }
 
+    @Test
+    fun m10ReservationAndDispatchRollbackTogetherAndCeilingSurvivesReopen(): Unit = runBlocking {
+        var original: PendingMutation? = null
+        withDatabase { database ->
+            val store = ItemStore(database.items(), nextId = { "work-001" }, nextMutationId = { "m10-create-001" })
+            store.create("Background item")
+            original = store.pendingMutations().single()
+            store.ensureAutomaticCycle("native-cycle")
+            val sql = database.openHelper.writableDatabase
+            sql.execSQL("CREATE TRIGGER reject_reservation BEFORE INSERT ON automatic_sync " +
+                "WHEN NEW.httpAttempts > 0 BEGIN SELECT RAISE(ABORT, 'reservation rejected'); END")
+            assertThrows(android.database.sqlite.SQLiteConstraintException::class.java) {
+                runBlocking { store.reserveAutomaticRequest("native-cycle") }
+            }
+            assertEquals(listOf(original), store.pendingMutations())
+            assertEquals(AutomaticCycle(workId = "native-cycle"), store.automaticCycle())
+            sql.execSQL("DROP TRIGGER reject_reservation")
+        }
+        repeat(3) { attempt ->
+            withDatabase { database ->
+                val store = ItemStore(database.items())
+                assertEquals(requireNotNull(original).copy(dispatched = true), store.reserveAutomaticRequest("native-cycle"))
+                assertEquals(attempt + 1, store.automaticCycle()?.httpAttempts)
+            }
+        }
+        withDatabase { database ->
+            val store = ItemStore(database.items())
+            assertNull(store.reserveAutomaticRequest("native-cycle"))
+            assertEquals(AutomaticCycle(workId = "native-cycle", httpAttempts = 3, state = "DEFERRED"), store.automaticCycle())
+            assertEquals(store.automaticCycle(), store.ensureAutomaticCycle("must-not-replace"))
+            assertEquals(listOf(requireNotNull(original).copy(dispatched = true)), store.pendingMutations())
+            val remote = object : ItemRemote {
+                override suspend fun items(): List<Item> = error("Deferred startup performed GET")
+                override suspend fun send(mutation: PendingMutation): MutationResult = error("Deferred startup sent HTTP")
+            }
+            val sync = ItemSync(store, remote)
+            sync.recoverPending()
+            assertTrue(sync.status.value.error.orEmpty().contains("Automatic sync deferred"))
+            android.util.Log.i("M10Reservation", "native rollback PASS; reopen reservations=3; deferred startup=0HTTP")
+        }
+    }
+
+    @Test
+    fun m10CanonicalAckReceiptAndDequeueAreAtomicWithoutRefresh(): Unit = runBlocking {
+        val canonical = Item("work-001", "Background item", false, 1, 1700000700000L)
+        val result = MutationResult(201, "{\"item\":{\"id\":\"work-001\",\"title\":\"Background item\",\"completed\":false,\"version\":1,\"updatedAt\":1700000700000}}", canonical)
+        withDatabase { database ->
+            val store = ItemStore(database.items(), nextId = { canonical.id }, now = { 1800000000000L },
+                nextMutationId = { "m10-create-001" })
+            store.create(canonical.title)
+            store.ensureAutomaticCycle("native-ack")
+            val mutation = requireNotNull(store.reserveAutomaticRequest("native-ack"))
+            val local = store.items()
+            val sql = database.openHelper.writableDatabase
+            sql.execSQL("CREATE TRIGGER reject_canonical BEFORE INSERT ON items " +
+                "WHEN NEW.version = 1 BEGIN SELECT RAISE(ABORT, 'canonical rejected'); END")
+            assertThrows(android.database.sqlite.SQLiteConstraintException::class.java) {
+                runBlocking { store.acknowledge(mutation, result, applyCanonical = true) }
+            }
+            assertEquals(local, store.items())
+            assertEquals(listOf(mutation), store.pendingMutations())
+            assertTrue(store.acknowledgedMutations().isEmpty())
+            assertEquals(1, store.automaticCycle()?.httpAttempts)
+            sql.execSQL("DROP TRIGGER reject_canonical")
+            store.acknowledge(mutation, result, applyCanonical = true)
+            assertEquals("COMPLETE", store.settleAutomaticCycle("native-ack"))
+        }
+        withDatabase { database ->
+            val store = ItemStore(database.items())
+            assertEquals(listOf(canonical), store.items())
+            assertTrue(store.pendingMutations().isEmpty())
+            assertEquals(result.responseBody, store.acknowledgedMutations().single().responseBody)
+            assertNull(store.lastSuccessfulRefreshAt()) // No invented successful GET timestamp.
+            assertEquals(AutomaticCycle(workId = "native-ack", httpAttempts = 1, state = "COMPLETE"), store.automaticCycle())
+        }
+    }
+
+    @Test
+    fun m10V5MigrationPreservesEnvelopeReceiptsAndAllocatorWithoutInventingCycle(): Unit = runBlocking {
+        val path = context.getDatabasePath(name)
+        requireNotNull(path.parentFile).mkdirs()
+        val mutation = pendingMutation("work-001", "CREATE", "Background item", false, "m10-create-001", 7).copy(dispatched = true)
+        SQLiteDatabase.openOrCreateDatabase(path, null).use { legacy ->
+            legacy.execSQL("CREATE TABLE items (id TEXT NOT NULL PRIMARY KEY, title TEXT NOT NULL, completed INTEGER NOT NULL, version INTEGER NOT NULL, updatedAt INTEGER NOT NULL)")
+            legacy.execSQL("INSERT INTO items VALUES('work-001','Background item',0,0,1700000700000)")
+            legacy.execSQL("CREATE TABLE pending_mutations (sequence INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL, itemId TEXT NOT NULL, operation TEXT NOT NULL, title TEXT, completed INTEGER, clientMutationId TEXT NOT NULL, payloadHash TEXT NOT NULL, terminalError TEXT, baseVersion INTEGER, dispatched INTEGER NOT NULL DEFAULT 0)")
+            legacy.execSQL("INSERT INTO pending_mutations VALUES(7,'work-001','CREATE','Background item',0,'m10-create-001',?,NULL,NULL,1)", arrayOf(mutation.payloadHash))
+            legacy.execSQL("UPDATE sqlite_sequence SET seq=10 WHERE name='pending_mutations'")
+            legacy.execSQL("CREATE TABLE sync_metadata (id INTEGER NOT NULL PRIMARY KEY, lastSuccessfulRefreshAt INTEGER NOT NULL)")
+            legacy.execSQL("INSERT INTO sync_metadata VALUES(1,1700000000000)")
+            legacy.execSQL("CREATE TABLE acknowledged_mutations (clientMutationId TEXT NOT NULL PRIMARY KEY, payloadHash TEXT NOT NULL, statusCode INTEGER NOT NULL, responseBody TEXT NOT NULL)")
+            legacy.execSQL("INSERT INTO acknowledged_mutations VALUES('prior-receipt','prior-hash',200,'prior-body')")
+            legacy.execSQL("CREATE TABLE tombstones (id TEXT NOT NULL PRIMARY KEY, version INTEGER NOT NULL, updatedAt INTEGER NOT NULL, deleted INTEGER NOT NULL)")
+            legacy.execSQL("INSERT INTO tombstones VALUES('gone',2,1700000001000,1)")
+            legacy.version = 5
+        }
+        withDatabase { database ->
+            val store = ItemStore(database.items(), nextId = { "new-001" }, nextMutationId = { "new-after-m10" })
+            assertEquals(6, database.openHelper.readableDatabase.version)
+            assertNull(store.automaticCycle())
+            assertEquals(listOf(mutation), store.pendingMutations())
+            assertEquals(listOf(Item("work-001", "Background item", false, 0, 1700000700000L)), store.items())
+            assertEquals(listOf(AcknowledgedMutation("prior-receipt", "prior-hash", 200, "prior-body")), store.acknowledgedMutations())
+            assertEquals(listOf(Tombstone("gone", 2, 1700000001000L)), store.tombstones())
+            assertEquals(1700000000000L, store.lastSuccessfulRefreshAt())
+            store.create("After migration")
+            assertEquals(11L, store.pendingMutations().last().sequence)
+        }
+        withDatabase { database ->
+            assertEquals(mutation, database.items().readPending().first())
+            assertNull(database.items().readAutomaticCycle())
+        }
+    }
+
     @Test
     fun unknownSchemaVersionRejectsOpenWithoutDeletingExistingData() {
         runBlocking { withDatabase { it.items().insert(fixture.toEntity()) } }
diff --git a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt
index b635dc5..55f4ea3 100644
--- a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt
+++ b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt
@@ -28,6 +28,7 @@ import androidx.compose.ui.test.performTextReplacement
 import androidx.compose.ui.test.printToString
 import androidx.test.ext.junit.runners.AndroidJUnit4
 import androidx.test.platform.app.InstrumentationRegistry
+import androidx.work.WorkManager
 import androidx.compose.ui.state.ToggleableState
 import kotlinx.coroutines.runBlocking
 import okhttp3.OkHttpClient
@@ -57,8 +58,15 @@ class ItemUiTest {
     val rules: RuleChain = RuleChain.outerRule(object : ExternalResource() {
         override fun before() {
             val context = InstrumentationRegistry.getInstrumentation().targetContext
+            WorkManager.getInstance(context).cancelAllWork().result.get()
             context.deleteDatabase(ItemDatabase.NAME)
             context.deleteDatabase("m02-ui-error.db")
+            context.deleteDatabase("m10-ui-schedule-error.db")
+        }
+
+        override fun after() {
+            val context = InstrumentationRegistry.getInstrumentation().targetContext
+            WorkManager.getInstance(context).cancelAllWork().result.get()
         }
     }).around(compose)
 
@@ -181,6 +189,31 @@ class ItemUiTest {
         }
     }
 
+    @Test
+    fun m10SchedulingFailureKeepsTheSavedRowAndClearsItsSubmittedDraft() {
+        val context = InstrumentationRegistry.getInstrumentation().targetContext
+        val database = ItemDatabase.open(context, "m10-ui-schedule-error.db")
+        val store = ItemStore(database.items(), nextId = { "schedule-001" },
+            nextMutationId = { "schedule-failure-001" },
+            afterCommit = { throw IOException("Scheduler unavailable") })
+        try {
+            val sync = ItemSync(store, HttpItemRemote())
+            compose.runOnUiThread {
+                compose.activity.setContent { MaterialTheme { ItemScreen(store, sync) } }
+            }
+            awaitCrudText("Items (0)")
+            compose.onNodeWithTag("item-title-input").performTextInput("Saved despite scheduler")
+            compose.onNodeWithText("Add").performClick()
+            awaitCrudText("Saved despite scheduler")
+            compose.onNodeWithTag("storage-error").assertDoesNotExist()
+            compose.onNodeWithTag("sync-error").assertTextContains("scheduling failed", substring = true)
+            assertEquals("Saved despite scheduler", runBlocking { store.items().single().title })
+            assertEquals(1, runBlocking { store.pendingMutations().size })
+        } finally {
+            database.close()
+        }
+    }
+
     @Test
     fun frozenM03SequenceUsesHttpRoomAndTwoIndependentInstances() {
         val context = InstrumentationRegistry.getInstrumentation().targetContext
diff --git a/app/src/debug/AndroidManifest.xml b/app/src/debug/AndroidManifest.xml
new file mode 100644
index 0000000..6b8ee26
--- /dev/null
+++ b/app/src/debug/AndroidManifest.xml
@@ -0,0 +1,4 @@
+<?xml version="1.0" encoding="utf-8"?>
+<manifest xmlns:android="http://schemas.android.com/apk/res/android" xmlns:tools="http://schemas.android.com/tools">
+    <application android:name=".VerificationApplication" tools:replace="android:name" />
+</manifest>
diff --git a/app/src/debug/java/com/mobilesystemsevolution/kotlin/VerificationApplication.kt b/app/src/debug/java/com/mobilesystemsevolution/kotlin/VerificationApplication.kt
new file mode 100644
index 0000000..dd6b3d9
--- /dev/null
+++ b/app/src/debug/java/com/mobilesystemsevolution/kotlin/VerificationApplication.kt
@@ -0,0 +1,31 @@
+package com.mobilesystemsevolution.kotlin
+
+import android.util.Log
+import androidx.work.Clock
+import androidx.work.Configuration
+import androidx.work.RunnableScheduler
+import java.io.File
+
+/** Explicit M10 verification inputs; absent clock file leaves ordinary debug behavior unchanged. */
+class VerificationApplication : ItemApplication() {
+    private val clockFile: File get() = File(filesDir, "m10-clock")
+
+    override val workManagerConfiguration: Configuration
+        get() {
+            if (!clockFile.exists()) return super.workManagerConfiguration
+            return Configuration.Builder()
+                .setClock(Clock { clockFile.readText().trim().toLong() })
+                .setRunnableScheduler(object : RunnableScheduler {
+                    override fun scheduleWithDelay(delayInMillis: Long, runnable: Runnable) = Unit
+                    override fun cancel(runnable: Runnable) = Unit
+                })
+                .setMinimumLoggingLevel(Log.DEBUG)
+                .build()
+        }
+
+    override fun createStore(dao: ItemDao): ItemStore {
+        if (!clockFile.exists()) return super.createStore(dao)
+        return ItemStore(dao, nextId = { "work-001" }, now = { 1_700_000_700_000L },
+            nextMutationId = { "m10-create-001" }, afterCommit = ::scheduleCommitted)
+    }
+}
diff --git a/app/src/main/AndroidManifest.xml b/app/src/main/AndroidManifest.xml
index 3874211..dce5764 100644
--- a/app/src/main/AndroidManifest.xml
+++ b/app/src/main/AndroidManifest.xml
@@ -1,12 +1,20 @@
 <?xml version="1.0" encoding="utf-8"?>
-<manifest xmlns:android="http://schemas.android.com/apk/res/android">
+<manifest xmlns:android="http://schemas.android.com/apk/res/android" xmlns:tools="http://schemas.android.com/tools">
     <uses-permission android:name="android.permission.INTERNET" />
     <application
+        android:name=".ItemApplication"
         android:allowBackup="false"
         android:label="Offline Item Tracker"
         android:networkSecurityConfig="@xml/network_security_config"
         android:supportsRtl="true"
         android:theme="@android:style/Theme.Material.Light.NoActionBar">
+        <provider
+            android:name="androidx.startup.InitializationProvider"
+            android:authorities="${applicationId}.androidx-startup"
+            android:exported="false"
+            tools:node="merge">
+            <meta-data android:name="androidx.work.WorkManagerInitializer" tools:node="remove" />
+        </provider>
         <activity android:name=".MainActivity" android:exported="true">
             <intent-filter>
                 <action android:name="android.intent.action.MAIN" />
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/AutomaticItemSync.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/AutomaticItemSync.kt
new file mode 100644
index 0000000..2258a48
--- /dev/null
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/AutomaticItemSync.kt
@@ -0,0 +1,32 @@
+package com.mobilesystemsevolution.kotlin
+
+import kotlinx.coroutines.CancellationException
+import kotlinx.coroutines.sync.withLock
+
+enum class AutomaticResult { COMPLETE, RETRY, DEFERRED }
+
+/** One reserved HTTP request per invocation; WorkManager owns delay and execution. */
+class AutomaticItemSync(private val store: ItemStore, private val remote: ItemRemote) {
+    suspend fun runOnce(workId: String): AutomaticResult = itemSyncMutex.withLock {
+        val mutation = store.reserveAutomaticRequest(workId)
+        if (mutation != null) {
+            try {
+                store.acknowledge(mutation, remote.send(mutation), applyCanonical = true)
+            } catch (cancelled: CancellationException) {
+                throw cancelled
+            } catch (conflict: MutationVersionConflict) {
+                store.recordVersionConflict(mutation, conflict)
+            } catch (collision: MutationIdentityConflict) {
+                store.markIdentityConflict(mutation)
+            } catch (_: Exception) {
+                // The original payload and reserved attempt remain durable, including
+                // remote-success/local-ACK failures. A later run may replay only that envelope.
+            }
+        }
+        when (store.settleAutomaticCycle(workId)) {
+            "COMPLETE" -> AutomaticResult.COMPLETE
+            "DEFERRED" -> AutomaticResult.DEFERRED
+            else -> AutomaticResult.RETRY
+        }
+    }
+}
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemApplication.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemApplication.kt
new file mode 100644
index 0000000..da1ed20
--- /dev/null
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemApplication.kt
@@ -0,0 +1,16 @@
+package com.mobilesystemsevolution.kotlin
+
+import android.app.Application
+import androidx.work.Configuration
+
+/** Also supplies WorkManager when Android starts SystemJobService without an Activity. */
+open class ItemApplication : Application(), Configuration.Provider {
+    override val workManagerConfiguration: Configuration
+        get() = Configuration.Builder().build()
+
+    open fun createStore(dao: ItemDao): ItemStore = ItemStore(dao, afterCommit = ::scheduleCommitted)
+
+    suspend fun scheduleCommitted(store: ItemStore) {
+        ItemWorkScheduler(this, store).schedule()
+    }
+}
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemDatabase.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemDatabase.kt
index 9931cbe..3be4c71 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemDatabase.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemDatabase.kt
@@ -65,6 +65,19 @@ data class AcknowledgedMutation(
     val responseBody: String,
 )
 
+@Entity(tableName = "automatic_sync")
+data class AutomaticCycle(
+    @PrimaryKey val id: Int = 1,
+    val workId: String,
+    val httpAttempts: Int = 0,
+    val state: String = "ACTIVE",
+)
+
+// Match the foreground queue: version conflicts may be passed, identity conflicts
+// stop the ordered drain before any later mutation is dispatched.
+private fun List<PendingMutation>.nextOrdinaryMutation(): PendingMutation? =
+    firstOrNull { it.terminalError != "version_conflict" && it.terminalError != "base_version_unknown" }
+
 @Dao
 interface ItemDao {
     // Preserve the M01 insertion order without adding a future metadata column.
@@ -144,7 +157,8 @@ interface ItemDao {
     }
 
     @Transaction
-    suspend fun acknowledge(sequence: Long, result: AcknowledgedMutation, item: ItemEntity? = null) {
+    suspend fun acknowledge(sequence: Long, result: AcknowledgedMutation, item: ItemEntity? = null,
+                            applyCanonical: Boolean = false) {
         val mutation = checkNotNull(readPendingBySequence(sequence)) { "Pending change no longer exists" }
         check(mutation.clientMutationId == result.clientMutationId && mutation.payloadHash == result.payloadHash)
         check(item == null || item.id == mutation.itemId)
@@ -154,8 +168,13 @@ interface ItemDao {
         }
         if (item != null) {
             // An old receipt must not replace newer local fields or resurrect a deleted Item.
-            advanceObservedVersion(item.id, item.version)
             val successor = readPending().firstOrNull { it.itemId == item.id && it.sequence > sequence }
+            val current = readItem(item.id)
+            if (applyCanonical && successor == null && current != null && current.version <= item.version) {
+                upsertCanonical(item)
+            } else {
+                advanceObservedVersion(item.id, item.version)
+            }
             if (successor != null && !successor.dispatched && successor.terminalError == null && successor.operation != "CREATE") {
                 val prepared = successor.withBaseVersion(item.version)
                 check(updatePreparedMutation(successor.sequence, successor.clientMutationId, successor.payloadHash,
@@ -231,6 +250,55 @@ interface ItemDao {
         insertPending(mutation.withBaseVersion(baseVersion))
     }
 
+    @Query("SELECT * FROM automatic_sync WHERE id = 1")
+    suspend fun readAutomaticCycle(): AutomaticCycle?
+
+    @Insert(onConflict = OnConflictStrategy.REPLACE)
+    suspend fun saveAutomaticCycle(cycle: AutomaticCycle)
+
+    @Transaction
+    suspend fun ensureAutomaticCycle(workId: String): AutomaticCycle? {
+        val next = readPending().nextOrdinaryMutation() ?: return null
+        if (next.terminalError != null) return null
+        val previous = readAutomaticCycle()
+        if (previous != null && previous.state != "COMPLETE") return previous
+        return AutomaticCycle(workId = workId).also { saveAutomaticCycle(it) }
+    }
+
+    @Transaction
+    suspend fun reserveAutomaticRequest(workId: String): PendingMutation? {
+        val cycle = readAutomaticCycle() ?: return null
+        if (cycle.workId != workId || cycle.state != "ACTIVE") return null
+        val queued = readPending().nextOrdinaryMutation()
+        if (queued == null || queued.terminalError != null) {
+            saveAutomaticCycle(cycle.copy(state = "COMPLETE"))
+            return null
+        }
+        if (cycle.httpAttempts >= 3) {
+            saveAutomaticCycle(cycle.copy(state = "DEFERRED"))
+            return null
+        }
+        val mutation = prepareDispatch(queued.sequence)
+        // Reservation and immutable dispatch marker commit before entering HTTP.
+        // A process loss between reservation and reply never refunds this allowance.
+        saveAutomaticCycle(cycle.copy(httpAttempts = cycle.httpAttempts + 1))
+        return mutation
+    }
+
+    @Transaction
+    suspend fun settleAutomaticCycle(workId: String): String {
+        val cycle = readAutomaticCycle() ?: return "COMPLETE"
+        if (cycle.workId != workId) return "COMPLETE"
+        val next = readPending().nextOrdinaryMutation()
+        val state = when {
+            next == null || next.terminalError != null -> "COMPLETE"
+            cycle.httpAttempts >= 3 -> "DEFERRED"
+            else -> cycle.state
+        }
+        saveAutomaticCycle(cycle.copy(state = state))
+        return state
+    }
+
     @Transaction
     suspend fun replaceAll(items: List<ItemEntity>, refreshedAt: Long? = null,
                            tombstones: List<Tombstone> = emptyList()): List<ItemEntity> {
@@ -251,8 +319,8 @@ interface ItemDao {
     }
 }
 
-@Database(entities = [ItemEntity::class, SyncMetadata::class, PendingMutation::class, AcknowledgedMutation::class, Tombstone::class],
-    version = 5, exportSchema = true)
+@Database(entities = [ItemEntity::class, SyncMetadata::class, PendingMutation::class, AcknowledgedMutation::class,
+    Tombstone::class, AutomaticCycle::class], version = 6, exportSchema = true)
 abstract class ItemDatabase : RoomDatabase() {
     abstract fun items(): ItemDao
 
@@ -326,9 +394,16 @@ abstract class ItemDatabase : RoomDatabase() {
             }
         }
 
+        private val MIGRATION_5_6 = object : Migration(5, 6) {
+            override fun migrate(db: SupportSQLiteDatabase) {
+                db.execSQL("CREATE TABLE automatic_sync (`id` INTEGER NOT NULL, `workId` TEXT NOT NULL, " +
+                    "`httpAttempts` INTEGER NOT NULL, `state` TEXT NOT NULL, PRIMARY KEY(`id`))")
+            }
+        }
+
         fun open(context: Context, name: String = NAME): ItemDatabase =
             Room.databaseBuilder(context.applicationContext, ItemDatabase::class.java, name)
-                .addMigrations(MIGRATION_1_2, MIGRATION_2_3, MIGRATION_3_4, MIGRATION_4_5)
+                .addMigrations(MIGRATION_1_2, MIGRATION_2_3, MIGRATION_3_4, MIGRATION_4_5, MIGRATION_5_6)
                 // Preserve existing Items; reject unknown schemas instead of erasing data.
                 .build()
     }
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemStore.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemStore.kt
index 7d158f7..acd9acd 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemStore.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemStore.kt
@@ -1,6 +1,7 @@
 package com.mobilesystemsevolution.kotlin
 
 import java.util.UUID
+import kotlinx.coroutines.CancellationException
 
 data class Item(
     val id: String,
@@ -16,7 +17,11 @@ class ItemStore(
     private val nextId: () -> String = { UUID.randomUUID().toString() },
     private val now: () -> Long = System::currentTimeMillis,
     private val nextMutationId: () -> String = { UUID.randomUUID().toString() },
+    private val afterCommit: suspend (ItemStore) -> Unit = {},
 ) {
+    var schedulingError: String? = null
+        private set
+
     suspend fun items(): List<Item> = dao.readAll().map(ItemEntity::toDomain)
 
     suspend fun lastSuccessfulRefreshAt(): Long? = dao.lastSuccessfulRefreshAt()
@@ -29,15 +34,23 @@ class ItemStore(
 
     suspend fun prepareDispatch(sequence: Long): PendingMutation = dao.prepareDispatch(sequence)
 
-    suspend fun acknowledge(mutation: PendingMutation, result: MutationResult) {
+    suspend fun acknowledge(mutation: PendingMutation, result: MutationResult, applyCanonical: Boolean = false) {
         val item = if (mutation.operation == "DELETE") null else checkNotNull(result.item) {
             "Acknowledgment has no validated canonical Item"
         }
         check(item == null || item.id == mutation.itemId)
         dao.acknowledge(mutation.sequence, AcknowledgedMutation(mutation.clientMutationId,
-            mutation.payloadHash, result.statusCode, result.responseBody), item?.toEntity())
+            mutation.payloadHash, result.statusCode, result.responseBody), item?.toEntity(), applyCanonical)
     }
 
+    suspend fun automaticCycle(): AutomaticCycle? = dao.readAutomaticCycle()
+
+    suspend fun ensureAutomaticCycle(workId: String): AutomaticCycle? = dao.ensureAutomaticCycle(workId)
+
+    suspend fun reserveAutomaticRequest(workId: String): PendingMutation? = dao.reserveAutomaticRequest(workId)
+
+    suspend fun settleAutomaticCycle(workId: String): String = dao.settleAutomaticCycle(workId)
+
     suspend fun markIdentityConflict(mutation: PendingMutation) {
         check(dao.markIdentityConflict(mutation.sequence, mutation.clientMutationId, mutation.payloadHash) == 1) {
             "Pending identity conflict no longer exists"
@@ -57,21 +70,38 @@ class ItemStore(
         val validTitle = title.trim().also { require(it.isNotEmpty()) }
         val item = Item(nextId(), validTitle, false, 0, now()).toEntity()
         dao.createLocal(item, pendingMutation(item.id, "CREATE", item.title, item.completed, nextMutationId()))
+        scheduleCommitted()
     }
 
     suspend fun rename(id: String, title: String) {
         val validTitle = title.trim().also { require(it.isNotEmpty()) }
         dao.renameLocal(id, validTitle, now(), pendingMutation(id, "RENAME", title = validTitle,
             clientMutationId = nextMutationId()))
+        scheduleCommitted()
     }
 
     suspend fun setCompleted(id: String, completed: Boolean) {
         dao.setCompletedLocal(id, completed, now(), pendingMutation(id, "COMPLETE", completed = completed,
             clientMutationId = nextMutationId()))
+        scheduleCommitted()
     }
 
     suspend fun delete(id: String) {
         dao.deleteLocal(id, pendingMutation(id, "DELETE", clientMutationId = nextMutationId()))
+        scheduleCommitted()
+    }
+
+    private suspend fun scheduleCommitted() {
+        try {
+            afterCommit(this)
+            schedulingError = null
+        } catch (cancelled: CancellationException) {
+            throw cancelled
+        } catch (_: Exception) {
+            // SQL has already committed. A scheduling failure must not present the
+            // saved edit as an unsuccessful write or leave its submitted draft intact.
+            schedulingError = "Automatic sync scheduling failed. Saved change retained."
+        }
     }
 }
 
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemSync.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemSync.kt
index 515a498..ca91f94 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemSync.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemSync.kt
@@ -3,6 +3,10 @@ package com.mobilesystemsevolution.kotlin
 import kotlinx.coroutines.CancellationException
 import kotlinx.coroutines.flow.MutableStateFlow
 import kotlinx.coroutines.flow.asStateFlow
+import kotlinx.coroutines.sync.Mutex
+import kotlinx.coroutines.sync.withLock
+
+internal val itemSyncMutex = Mutex()
 
 interface ItemRemote {
     suspend fun items(): List<Item>
@@ -25,6 +29,7 @@ class ItemSync(
     private val store: ItemStore,
     private val remote: ItemRemote,
     private val now: () -> Long = System::currentTimeMillis,
+    private val schedulePending: (suspend () -> Unit)? = null,
 ) {
     private val mutableStatus = MutableStateFlow(SyncStatus())
     val status = mutableStatus.asStateFlow()
@@ -32,9 +37,17 @@ class ItemSync(
     // A new session is stale until refreshed, even when a saved timestamp exists.
     suspend fun readSavedStatus() {
         mutableStatus.value = mutableStatus.value.copy(lastSuccessfulRefreshAt = store.lastSuccessfulRefreshAt())
-        store.pendingMutations().firstOrNull { it.terminalError != null && it.terminalError != "version_conflict" }?.let {
+        val pending = store.pendingMutations()
+        if (pending.isNotEmpty()) store.schedulingError?.let {
+            mutableStatus.value = mutableStatus.value.copy(state = SyncState.ERROR, error = it)
+        }
+        pending.firstOrNull { it.terminalError != null && it.terminalError != "version_conflict" }?.let {
             mutableStatus.value = mutableStatus.value.copy(state = SyncState.ERROR, error = it.terminalError)
         }
+        if (store.automaticCycle()?.state == "DEFERRED") {
+            mutableStatus.value = mutableStatus.value.copy(state = SyncState.ERROR,
+                error = "Automatic sync deferred after 3 HTTP attempts. Saved changes retained.")
+        }
     }
 
     fun markLocalChange() {
@@ -43,10 +56,27 @@ class ItemSync(
 
     /** Startup resumes durable work once; empty or terminal-only queues need no request. */
     suspend fun recoverPending() {
-        if (store.pendingMutations().any { it.terminalError == null }) synchronize()
+        if (store.automaticCycle()?.state == "DEFERRED") {
+            readSavedStatus()
+            return
+        }
+        if (store.pendingMutations().any { it.terminalError == null }) {
+            if (schedulePending == null) {
+                synchronize()
+            } else {
+                try {
+                    schedulePending.invoke()
+                } catch (cancelled: CancellationException) {
+                    throw cancelled
+                } catch (_: Exception) {
+                    mutableStatus.value = mutableStatus.value.copy(state = SyncState.ERROR,
+                        error = "Automatic sync scheduling failed. Saved changes retained.")
+                }
+            }
+        }
     }
 
-    suspend fun synchronize() {
+    suspend fun synchronize() = itemSyncMutex.withLock {
         try {
             mutableStatus.value = SyncStatus(SyncState.REFRESHING, store.lastSuccessfulRefreshAt())
             for (queued in store.pendingMutations()) {
@@ -73,6 +103,7 @@ class ItemSync(
             val refreshedAt = now()
             // Commit the canonical list and its timestamp together, then use the Room readback.
             store.replaceWithCanonical(canonical.items, refreshedAt, canonical.tombstones)
+            store.automaticCycle()?.let { store.settleAutomaticCycle(it.workId) }
             mutableStatus.value = SyncStatus(SyncState.FRESH, refreshedAt)
         } catch (cancelled: CancellationException) {
             mutableStatus.value = mutableStatus.value.copy(state = SyncState.STALE, error = null)
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemWorkScheduler.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemWorkScheduler.kt
new file mode 100644
index 0000000..349b072
--- /dev/null
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemWorkScheduler.kt
@@ -0,0 +1,68 @@
+package com.mobilesystemsevolution.kotlin
+
+import android.content.Context
+import android.util.Log
+import androidx.work.BackoffPolicy
+import androidx.work.Constraints
+import androidx.work.CoroutineWorker
+import androidx.work.ExistingWorkPolicy
+import androidx.work.NetworkType
+import androidx.work.OneTimeWorkRequestBuilder
+import androidx.work.WorkManager
+import androidx.work.WorkerParameters
+import java.util.UUID
+import java.util.concurrent.TimeUnit
+import kotlinx.coroutines.CancellationException
+import kotlinx.coroutines.Dispatchers
+import kotlinx.coroutines.sync.Mutex
+import kotlinx.coroutines.sync.withLock
+import kotlinx.coroutines.withContext
+
+class ItemWorkScheduler(context: Context, private val store: ItemStore) {
+    private val context = context.applicationContext
+
+    suspend fun schedule() = withContext(Dispatchers.IO) {
+        enqueueMutex.withLock {
+            val cycle = store.ensureAutomaticCycle(UUID.randomUUID().toString()) ?: return@withLock
+            if (cycle.state != "ACTIVE") return@withLock
+            val manager = WorkManager.getInstance(context)
+            val id = UUID.fromString(cycle.workId)
+            if (manager.getWorkInfoById(id).get() != null) return@withLock
+            val request = OneTimeWorkRequestBuilder<ItemSyncWorker>()
+                .setId(id)
+                .setConstraints(Constraints.Builder().setRequiredNetworkType(NetworkType.CONNECTED).build())
+                .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 10, TimeUnit.SECONDS)
+                .build()
+            // Repeated requests retain this persisted ID. A genuinely new cycle joins
+            // the same unique chain even if its completed predecessor is still finishing.
+            manager.enqueueUniqueWork(UNIQUE_WORK, ExistingWorkPolicy.APPEND_OR_REPLACE, request).result.get()
+        }
+    }
+
+    companion object {
+        const val UNIQUE_WORK = "item-automatic-sync"
+        private val enqueueMutex = Mutex()
+    }
+}
+
+class ItemSyncWorker(context: Context, parameters: WorkerParameters) : CoroutineWorker(context, parameters) {
+    override suspend fun doWork(): Result {
+        val database = ItemDatabase.open(applicationContext)
+        return try {
+            val result = AutomaticItemSync(ItemStore(database.items()), HttpItemRemote()).runOnce(id.toString())
+            Log.i("ItemSyncWorker", "work=$id runAttempt=$runAttemptCount result=$result")
+            when (result) {
+                AutomaticResult.COMPLETE -> Result.success()
+                AutomaticResult.RETRY -> Result.retry()
+                AutomaticResult.DEFERRED -> Result.failure()
+            }
+        } catch (cancelled: CancellationException) {
+            throw cancelled
+        } catch (error: Exception) {
+            Log.e("ItemSyncWorker", "Work could not read or commit its durable state", error)
+            Result.retry()
+        } finally {
+            database.close()
+        }
+    }
+}
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
index 0eddf27..c55ccf6 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
@@ -45,8 +45,9 @@ class MainActivity : ComponentActivity() {
     override fun onCreate(savedInstanceState: Bundle?) {
         super.onCreate(savedInstanceState)
         database = ItemDatabase.open(applicationContext)
-        val store = ItemStore(database.items())
-        val sync = ItemSync(store, HttpItemRemote())
+        val app = application as ItemApplication
+        val store = app.createStore(database.items())
+        val sync = ItemSync(store, HttpItemRemote(), schedulePending = { app.scheduleCommitted(store) })
         setContent { MaterialTheme { ItemScreen(store, sync) } }
     }
 
diff --git a/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt b/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
index fa93fdc..9c372c5 100644
--- a/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
+++ b/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
@@ -671,6 +671,198 @@ class ItemStoreTest {
         assertEquals(SyncStatus(SyncState.ERROR, null, "Forced offline"), sync.status.value)
     }
 
+    @Test
+    fun m10AutomaticCycleCountsHttpReservationsAndDefersWithoutFourthRequest(): Unit = runBlocking {
+        for (succeeds in listOf(true, false)) {
+            val dao = FakeItemDao()
+            val store = ItemStore(dao, nextId = { "work-001" }, now = { 1700000700000L },
+                nextMutationId = { "m10-create-001" })
+            store.create("Background item")
+            val original = store.pendingMutations().single().copy(dispatched = true)
+            val fake = FakeRemote().apply { rows.clear(); time = 1700000700000L }
+            val requests = mutableListOf<PendingMutation>()
+            val remote = object : ItemRemote by fake {
+                override suspend fun send(mutation: PendingMutation): MutationResult {
+                    requests += mutation
+                    if (!succeeds || requests.size < 3) throw IOException("Sync HTTP 503")
+                    return fake.send(mutation)
+                }
+            }
+            val cycle = requireNotNull(store.ensureAutomaticCycle("cycle-1"))
+            assertEquals(cycle, store.ensureAutomaticCycle("not-a-new-cycle"))
+            assertEquals(AutomaticResult.RETRY, AutomaticItemSync(store, remote).runOnce(cycle.workId))
+            assertEquals(1, store.automaticCycle()?.httpAttempts)
+            assertEquals(AutomaticResult.RETRY, AutomaticItemSync(ItemStore(dao), remote).runOnce(cycle.workId))
+            assertEquals(2, store.automaticCycle()?.httpAttempts)
+            val expected = if (succeeds) AutomaticResult.COMPLETE else AutomaticResult.DEFERRED
+            assertEquals(expected, AutomaticItemSync(ItemStore(dao), remote).runOnce(cycle.workId))
+            assertEquals(3, store.automaticCycle()?.httpAttempts)
+            assertEquals(List(3) { original }, requests)
+            assertFalse(fake.calls.contains("GET"))
+            if (succeeds) {
+                assertEquals(listOf(Item("work-001", "Background item", false, 1, 1700000700000L)), store.items())
+                assertTrue(store.pendingMutations().isEmpty())
+                assertEquals(1, fake.applied)
+                assertEquals("COMPLETE", store.automaticCycle()?.state)
+            } else {
+                val reopened = ItemStore(dao)
+                val startup = ItemSync(reopened, remote, schedulePending = { error("Deferred work must not enqueue") })
+                startup.recoverPending()
+                assertTrue(startup.status.value.error.orEmpty().contains("Automatic sync deferred"))
+                assertEquals("cycle-1", reopened.ensureAutomaticCycle("replacement")?.workId)
+                assertEquals(AutomaticResult.DEFERRED, AutomaticItemSync(reopened, remote).runOnce(cycle.workId))
+                assertEquals(3, requests.size)
+                assertEquals(listOf(original), reopened.pendingMutations())
+                assertEquals(0, fake.applied)
+            }
+        }
+    }
+
+    @Test
+    fun m10ReservedButUnknownOutcomesAreNotRefundedAfterReopen(): Unit = runBlocking {
+        val dao = FakeItemDao()
+        val store = ItemStore(dao, nextId = { "work-001" }, nextMutationId = { "m10-create-001" })
+        store.create("Background item")
+        store.ensureAutomaticCycle("cycle-unknown")
+        val original = store.pendingMutations().single().copy(dispatched = true)
+        repeat(3) { assertEquals(original, ItemStore(dao).reserveAutomaticRequest("cycle-unknown")) }
+        val remote = FakeRemote()
+        assertEquals(AutomaticResult.DEFERRED, AutomaticItemSync(ItemStore(dao), remote).runOnce("cycle-unknown"))
+        assertTrue(remote.calls.isEmpty())
+        assertEquals(3, store.automaticCycle()?.httpAttempts)
+        assertEquals(listOf(original), store.pendingMutations())
+    }
+
+    @Test
+    fun m10LostAckReplaysTheOriginalIdentityAndCommitsCanonicalWithoutGet(): Unit = runBlocking {
+        val dao = FakeItemDao()
+        val store = ItemStore(dao, nextId = { "work-001" }, now = { 1700000700000L },
+            nextMutationId = { "m10-create-001" })
+        store.create("Background item")
+        store.ensureAutomaticCycle("cycle-ack")
+        val remote = FakeRemote().apply { rows.clear(); time = 1700000700000L; dropNextCreateResponse = true }
+        assertEquals(AutomaticResult.RETRY, AutomaticItemSync(store, remote).runOnce("cycle-ack"))
+        val retained = store.pendingMutations().single()
+        assertEquals(AutomaticResult.COMPLETE, AutomaticItemSync(ItemStore(dao), remote).runOnce("cycle-ack"))
+        assertEquals(listOf(retained, retained), remote.mutations)
+        assertEquals(listOf("POST work-001", "POST work-001"), remote.calls)
+        assertEquals(1, remote.applied)
+        assertEquals(1, remote.duplicates)
+        assertEquals(remote.rows.values.toList(), store.items())
+        assertEquals(remote.responses.getValue(retained.clientMutationId).second.responseBody,
+            store.acknowledgedMutations().single().responseBody)
+    }
+
+    @Test
+    fun m10AutomaticAckKeepsLaterLocalFieldsAndPreparesOnlyItsOwnSuccessor(): Unit = runBlocking {
+        val dao = FakeItemDao()
+        var identity = 0
+        val store = ItemStore(dao, nextId = { "work-001" }, now = { 1700000700000L },
+            nextMutationId = { "successor-${++identity}" })
+        store.create("Background item")
+        store.rename("work-001", "Later local title")
+        store.ensureAutomaticCycle("cycle-successor")
+        val remote = FakeRemote().apply { rows.clear(); time = 1700000700000L }
+        assertEquals(AutomaticResult.RETRY, AutomaticItemSync(store, remote).runOnce("cycle-successor"))
+        assertEquals("Later local title", store.items().single().title)
+        assertEquals(1L, store.pendingMutations().single().baseVersion)
+        assertEquals(AutomaticResult.COMPLETE, AutomaticItemSync(store, remote).runOnce("cycle-successor"))
+        assertEquals(Item("work-001", "Later local title", false, 2, 1700000701000L), store.items().single())
+        assertFalse(remote.calls.contains("GET"))
+    }
+
+    @Test
+    fun m10CommittedEditsScheduleAfterPersistenceAndStartupDelegatesWithoutHttp(): Unit = runBlocking {
+        val dao = FakeItemDao()
+        var scheduled = 0
+        val store = ItemStore(dao, nextId = { "work-001" }, afterCommit = {
+            assertEquals(1, it.items().size)
+            assertEquals(1, it.pendingMutations().size)
+            scheduled++
+        })
+        store.create("Background item")
+        assertEquals(1, scheduled)
+        assertThrows(IllegalStateException::class.java) { runBlocking { store.create("Duplicate") } }
+        assertEquals(1, scheduled)
+        val remote = FakeRemote()
+        ItemSync(store, remote, schedulePending = { scheduled++ }).recoverPending()
+        assertEquals(2, scheduled)
+        assertTrue(remote.calls.isEmpty())
+    }
+
+    @Test
+    fun m10SchedulingFailureDoesNotTurnACommittedEditIntoASaveFailure(): Unit = runBlocking {
+        val dao = FakeItemDao()
+        var fails = true
+        var scheduled = 0
+        val store = ItemStore(dao, nextId = { "work-001" }, afterCommit = {
+            scheduled++
+            assertEquals(1, it.items().size)
+            if (fails) throw IOException("Scheduler storage unavailable")
+        })
+        ItemEditor(title = "Background item").submit(store)
+        assertEquals("Background item", store.items().single().title)
+        assertEquals(1, store.pendingMutations().size)
+        val remote = FakeRemote()
+        val sync = ItemSync(store, remote)
+        sync.readSavedStatus()
+        assertTrue(sync.status.value.error.orEmpty().contains("scheduling failed"))
+        fails = false
+        store.rename("work-001", "Committed rename")
+        assertEquals(2, scheduled)
+        assertEquals(null, store.schedulingError)
+        assertEquals(2, store.pendingMutations().size)
+        val startup = ItemSync(store, remote, schedulePending = { throw IOException("Scheduler unavailable") })
+        startup.recoverPending()
+        assertTrue(startup.status.value.error.orEmpty().contains("Saved changes retained"))
+        assertEquals("Committed rename", store.items().single().title)
+        assertTrue(remote.calls.isEmpty())
+    }
+
+    @Test
+    fun m10IdentityCollisionStopsBeforeLaterIntentsAndOrdinaryStartupCannotBypassIt(): Unit = runBlocking {
+        val dao = FakeItemDao()
+        var index = 0
+        val store = ItemStore(dao, nextId = { "local-${index + 1}" },
+            nextMutationId = { "identity-${++index}" })
+        store.create("First local")
+        store.create("Later local")
+        val remote = FakeRemote().apply { rows.clear() }
+        remote.send(pendingMutation("other-item", "CREATE", "Other payload", false, "identity-1"))
+        val first = store.pendingMutations().first()
+        store.ensureAutomaticCycle("cycle-identity")
+        assertEquals(AutomaticResult.COMPLETE, AutomaticItemSync(store, remote).runOnce("cycle-identity"))
+        assertEquals(first.copy(dispatched = true, terminalError = "identity_conflict"), store.pendingMutations().first())
+        assertFalse(store.pendingMutations().last().dispatched)
+        val requests = remote.calls.toList()
+        val reopened = ItemStore(dao)
+        assertEquals(null, reopened.ensureAutomaticCycle("must-not-skip-identity"))
+        assertEquals(AutomaticResult.COMPLETE, AutomaticItemSync(reopened, remote).runOnce("cycle-identity"))
+        ItemSync(reopened, remote, schedulePending = {
+            assertEquals(null, reopened.ensureAutomaticCycle("startup-must-not-skip"))
+        }).recoverPending()
+        assertEquals(requests, remote.calls)
+        assertEquals(1, store.automaticCycle()?.httpAttempts)
+        assertEquals(2, store.pendingMutations().size)
+    }
+
+    @Test
+    fun m10TerminalConflictIsRetainedWithoutAutomaticResend(): Unit = runBlocking {
+        val dao = FakeItemDao()
+        val remote = conflictRemote()
+        val store = ItemStore(dao, nextMutationId = { "automatic-conflict" })
+        store.replaceWithCanonical(remote.rows.values.toList())
+        store.rename("conflict-001", "Local attempt")
+        remote.patch("conflict-001", "Remote winner", null)
+        store.ensureAutomaticCycle("cycle-conflict")
+        assertEquals(AutomaticResult.COMPLETE, AutomaticItemSync(store, remote).runOnce("cycle-conflict"))
+        assertEquals("version_conflict", store.pendingMutations().single().terminalError)
+        assertEquals("Remote winner", store.items().single().title)
+        val requests = remote.calls.toList()
+        assertEquals(AutomaticResult.COMPLETE, AutomaticItemSync(ItemStore(dao), remote).runOnce("cycle-conflict"))
+        assertEquals(requests, remote.calls)
+    }
+
     private fun conflictRemote() = FakeRemote().apply {
         rows.clear()
         rows["conflict-001"] = Item("conflict-001", "Initial", false, 1, 1700000000000L)
@@ -810,6 +1002,11 @@ class ItemStoreTest {
         private val tombstones = linkedMapOf<String, Tombstone>()
         var rejectAcknowledgment = false
         val acknowledgmentWrites = mutableListOf<String>()
+        private var automaticCycle: AutomaticCycle? = null
+
+        override suspend fun readAutomaticCycle(): AutomaticCycle? = automaticCycle
+
+        override suspend fun saveAutomaticCycle(cycle: AutomaticCycle) { automaticCycle = cycle }
 
         override suspend fun readPending(): List<PendingMutation> = pending.values.sortedBy { it.sequence }
 
diff --git a/fixture/server.py b/fixture/server.py
index 14c0fd3..1abab59 100644
--- a/fixture/server.py
+++ b/fixture/server.py
@@ -76,6 +76,7 @@ class Fixture(ThreadingHTTPServer):
         self.m09_case = None
         self.hold_identity = None
         self.held = None
+        self.m10_case = None
 
     def tick(self):
         timestamp = self.next_timestamp
@@ -142,6 +143,9 @@ class Fixture(ThreadingHTTPServer):
     def death_state(self):
         return dict(self.conflict_state(), case=self.m09_case, barrier=self.hold_state())
 
+    def work_state(self):
+        return dict(self.conflict_state(), case=self.m10_case, getRequests=self.get_requests)
+
 
 class Handler(BaseHTTPRequestHandler):
     def respond(self, status, payload, delivery="delivered"):
@@ -284,15 +288,31 @@ class Handler(BaseHTTPRequestHandler):
             self.respond(200, self.server.conflict_state())
         elif self.path == "/__m09":
             self.respond(200, self.server.death_state())
+        elif self.path == "/__m10":
+            self.respond(200, self.server.work_state())
         else:
             self.respond(404, {"error": "not found"})
 
     @serialized_response
     def do_POST(self):
-        if self.path in ("/__reset", "/__m06/reset", "/__m07/reset", "/__m09/reset"):
+        if self.path in ("/__reset", "/__m06/reset", "/__m07/reset", "/__m09/reset", "/__m10/reset"):
             if self.server.held is not None and self.server.held["outcome"] == "held":
                 self.respond(409, {"error": "held_response_must_finish_before_reset"})
                 return
+        if self.path == "/__m10/reset":
+            try:
+                data = self.payload()
+                if set(data) != {"case"} or data["case"] not in ("A", "B"):
+                    raise ValueError("Expected case A or B")
+            except (ValueError, TypeError) as error:
+                self.respond(400, {"error": str(error)})
+                return
+            self.server.reset()
+            self.server.items = {}
+            self.server.next_timestamp = 1700000700000
+            self.server.m10_case = data["case"]
+            self.respond(200, self.server.work_state())
+            return
         if self.path == "/__m09/reset":
             try:
                 data = self.payload()
@@ -410,6 +430,9 @@ class Handler(BaseHTTPRequestHandler):
             return
         if self.replay_if_known():
             return
+        if self.server.m10_case == "B" or (self.server.m10_case == "A" and self.server.mutation_requests <= 2):
+            self.respond(503, {"error": "temporary failure"})
+            return
         if data["id"] in self.server.items:
             self.respond(400, {"error": "Expected a new nonempty id"})
             return
diff --git a/fixture/test_server.py b/fixture/test_server.py
index aa0433d..daa250e 100644
--- a/fixture/test_server.py
+++ b/fixture/test_server.py
@@ -39,6 +39,32 @@ def mutation_wire(method, path, payload, identity):
 
 
 class FixtureContractTest(unittest.TestCase):
+    def test_m10_fixed_a503503201_keeps_identity_and_applies_once(self):
+        payload = dict(id="work-001", title="Background item", completed=False)
+        wire = mutation_wire("POST", "/items", payload, "m10-create-001")
+        self.assertEqual("b096e8b4ea45527ced6766b4aaf04fe9309e73b19437d84aa73bd2ccadb17359", wire["payloadHash"])
+        with m06_fixture() as request:
+            request("POST", "/__m10/reset", {"case": "A"})
+            self.assertEqual([503, 503, 201], [request("POST", "/items", wire)[0] for _ in range(3)])
+            state = request("GET", "/__m10")[1]
+            self.assertEqual((1, 0, 3, 0), (state["applied"], state["duplicates"], state["mutationRequests"], state["getRequests"]))
+            self.assertEqual([dict(payload, version=1, updatedAt=1700000700000)], state["items"])
+            self.assertEqual([wire] * 3, [entry["request"] for entry in state["requests"]])
+            self.assertEqual(201, request("POST", "/items", wire)[0])
+            replay = request("GET", "/__m10")[1]
+            self.assertEqual((1, 1), (replay["applied"], replay["duplicates"]))
+
+    def test_m10_b_never_masks_an_erroneous_fourth_client_request(self):
+        wire = mutation_wire("POST", "/items", dict(id="work-001", title="Background item", completed=False), "m10-create-001")
+        with m06_fixture() as request:
+            request("POST", "/__m10/reset", {"case": "B"})
+            self.assertEqual([503] * 4, [request("POST", "/items", wire)[0] for _ in range(4)])
+            state = request("GET", "/__m10")[1]
+            self.assertEqual(([], 0, 4, 0, 1700000700000),
+                             (state["items"], state["applied"], state["mutationRequests"], state["getRequests"], state["nextTimestamp"]))
+            self.assertEqual([wire] * 4, [entry["request"] for entry in state["requests"]])
+            self.assertEqual(400, request("POST", "/__m10/reset", {"case": "other"})[0])
+
     def test_m09_pending_create_fixed_contract(self):
         payload = dict(id="death-001", title="Recovered create", completed=False)
         wire = mutation_wire("POST", "/items", payload, "m09-create-001")
diff --git a/verification/M10-work-inputs.json b/verification/M10-work-inputs.json
new file mode 100644
index 0000000..b10ff81
--- /dev/null
+++ b/verification/M10-work-inputs.json
@@ -0,0 +1,31 @@
+{
+  "thread": "M10",
+  "profile": "phase-1",
+  "specRevision": "61280dd86ce88b6e431f408241c0998a275960aa",
+  "itemId": "work-001",
+  "title": "Background item",
+  "completed": false,
+  "clientMutationId": "m10-create-001",
+  "payloadHash": "b096e8b4ea45527ced6766b4aaf04fe9309e73b19437d84aa73bd2ccadb17359",
+  "localTimestamp": 1700000700000,
+  "firstSuccessTimestamp": 1700000700000,
+  "schemaVersion": 6,
+  "workManagerVersion": "2.9.1",
+  "workDatabaseVersion": 20,
+  "uniqueWorkName": "item-automatic-sync",
+  "workClass": "com.mobilesystemsevolution.kotlin.ItemSyncWorker",
+  "clockFile": "files/m10-clock",
+  "initialClock": "device epoch seconds plus 2 seconds, in milliseconds; independent of remote clock",
+  "clockAdvancesMs": [10000, 20000],
+  "terminalEligibilityAdvanceMs": 40000,
+  "httpCeiling": 3,
+  "cases": {"A": [503, 503, 201], "B": [503, 503, 503]},
+  "harness": {
+    "adbTimeoutSeconds": 45,
+    "uiWaitSeconds": 30,
+    "networkWaitSeconds": 30,
+    "preSeedTeardownWaitSeconds": 30,
+    "lossWaitSeconds": 15,
+    "workStateWaitSeconds": 30
+  }
+}
diff --git a/verification/M10.md b/verification/M10.md
index b2b42ca..a59c753 100644
--- a/verification/M10.md
+++ b/verification/M10.md
@@ -1,10 +1,11 @@
-# M10 verification ledger — baseline accepted
+# M10 verification ledger — official runtime passed
 
 - Profile: `phase-1`; Thread: `M10`; branch: `track/android-kotlin`.
 - SPEC_REVISION: `61280dd86ce88b6e431f408241c0998a275960aa`.
 - START: `6c3b11fb9aa6ecdcfea1312e083183f83ef524d5` (verified M09).
-- Current attempt3 / cumulative repair2. Actual fixed-case usage A3 / B0, never reset.
-- Status: unchanged-M09 limitation accepted by root; M10 product and final A/B acceptance outstanding.
+- Final attempt5 / cumulative repair4. Actual fixed-case usage A6 / B1, never reset.
+- Baseline support commit: `23cf17ef1374b98571e7e52838cd0a47caf54b1c`.
+- Status: root accepted the M09 limitation, final M10 A/B, actual M08 lifecycle and native24 regressions; final commit history/tag audit remains root-owned.
 
 Evidence root `E`:
 `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/android-kotlin/M10/`.
@@ -20,8 +21,17 @@ Each run keeps its exact argv, exits, timestamps, raw outputs, hashes and cleanu
 | Baseline A2 | Seed write/readback45056 bytes PASS; FAIL at cmd036 because unrelated queryability User0 was counted.44 adb commands,13.710s; no HOME/SIGKILL | `repair1-baseline-android-A2/` |
 | Repair2 | Target-package User0 parser only; actual raw dump plus malformed/duplicate/missing-state coverage, host3 PASS | `repair2-host01.*`, `repair2-frozen-01/manifest.json` |
 | Baseline A3 | LIMITATION_REPRODUCED;83 adb commands,24.387s; root audited193 raw artifacts and4 native DB/WAL snapshots | `repair2-baseline-android-A3/` |
+| Implementation checks | Initial Gradle/fixture launches failed at sandbox socket access; identical approved runs passed host28 plus both APK builds (40.155s) and fixture14 (6.274s). No device warmup | `fixed-host-build01.*`, `fixed-fixture-tests01.*`, `fixed-host-build02.*`, `fixed-fixture-tests02.*` |
+| Final A4 | FAIL after first actual OS HTTP503: harness incorrectly required the numeric OS job ID to remain0;108 adb,24.603s;9 native DB/WAL snapshots | `fixed-android-A4/` |
+| Repair3 | Bind current SystemIdInfo while preserving UUID/generation/unique chain; focused host4 PASS,15 negative cases. No product/APK change | `repair3-host01.*`, `repair3-frozen-01/manifest.json` |
+| Final A5 | FAIL with0 HTTP: active/CONNECTED network lacked VALIDATED, so immediate nonforced trigger returned22;101 adb,21.805s;6 native snapshots | `repair3-android-A5/` |
+| Repair4 | Observe actual job eligibility or natural execution inside the existing30s attempt deadline; at most one nonforced trigger. One focused host check PASS (6 sequences, deadline,4 parser negatives); no product/APK change | `repair4-host01.*`, `repair4-frozen-01/manifest.json` |
+| Final A6 | PASS;152 adb,25.871s;18 native snapshots; PID9932→absent→10320;503/503/201, applied1/pending0 | `repair4-android-A6/` |
+| Final B1 | PASS;163 adb,33.055s;19 native snapshots; PID10694→absent→11077;503x3, DEFERRED/pending1, no fourth HTTP | `repair4-android-B1/` |
+| M08 regression | PASS;37 adb,10.758s; actual lifecycle/recreation in one process,6 native snapshots,0 HTTP | `repair4-m08-android01/` |
+| Native regressions |24/24 PASS,0 skipped;48 exact JUnit start/pass events,38.415s JUnit;25 adb,40.310s wrapper | `repair4-native-android01/` |
 
-A1 and A2 remain charged and failed. The earlier budget stops are retained, not erased.
+A1, A2, A4 and A5 remain charged and failed. The earlier budget stops are retained, not erased.
 The user's [repair authorization](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/REPAIR-AUTHORIZATION-2026-08-29.md)
 permits necessary bounded repairs and verification for fixable issues while preserving all
 counts and the product's max3 HTTP ceiling. [Repair2 dispatch](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/resume-04/repair2-dispatch.json)
@@ -50,5 +60,50 @@ All54 START files were unchanged during baseline execution; only two verificatio
 files were added. The same app `3f4b3fc39b1d9d6906fb162aa608e770debbe05f74f0de06b3aaded17d1ef909`,
 test reference `16a5e3967d68e6790431e831f08ded7b1294172174b64a3c979d6883390a35b0`,
 and seed `58325cf3cadaa82af1f5d31e8623d0a1dc7d5eecb7dfeee3b34e561f99122e00` were reused.
-No owner device run, product build or final OS test is claimed. Root separately
-authorized minimal M10 implementation after accepting this baseline.
+This baseline used no owner device run or product build and did not claim final OS acceptance.
+Root separately authorized minimal M10 implementation after accepting it.
+
+## Frozen implementation and final acceptance
+
+Schema6 adds only the durable automatic-sync cycle and HTTP reservation state. Real committed
+edits/startup register unique CONNECTED WorkManager2.9.1 work; each actual automatic request
+reserves its attempt transactionally before HTTP. The ceiling remains3 across process loss.
+Canonical ACK/receipt/dequeue commits atomically without a GET, retaining M06/M07 identity,
+base and terminal-conflict rules. A failed post-commit schedule retains successful-edit semantics.
+The optional fixed clock/identities are debug-only; the release path is unchanged by that support.
+
+Both final cases create work-001 through the actual production UI, repeat ordinary registration
+without replacing the unique chain, then prove HOME/same-UID SIGKILL with stopped=false and
+an OS-created replacement process through SystemJobService. No app-entry helper runs across
+that loss/recovery boundary. Native mappings may allocate new numeric OS jobs for the same
+WorkSpec UUID/generation; exact current service/constraints remain bound. The retry clock
+advances10/20 seconds, without force or a higher HTTP budget.
+
+A6 applies the original m10-create-001/hash once after503/503/201. Its final row is exactly
+work-001 / Background item / false / version1 /1700000700000; the saved201 receipt and empty
+queue commit with COMPLETE/httpAttempts3. B1 preserves its original envelope and version0
+row in DEFERRED/httpAttempts3 after503x3. Later OS eligibility and ordinary UI add no fourth
+request. Neither automatic case performs a GET. Root directly read all37 native snapshots.
+
+The retained native24 selection includes real HTTP ordered mutation/ACK regressions and
+same-process Room reopen coverage; it does not claim reruns of the old full external M05/M06/M09
+scenarios. Separate M08 evidence uses real offline radios with scheduling enabled: PID11597,
+Application145325337, Activity37549152→5054629, restored draft/selection, first5 snapshots
+unchanged, then exactly one base1 rename and ACTIVE/httpAttempts0 after one Save. It is
+same-process Activity restoration, not process-kill draft durability.
+
+Root's [runtime verification and linked raw audits](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M10/main-runtime-verification.json)
+bind all65 frozen files and both APKs. The [final cleanup](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M10/main-final-cleanup/result.json)
+confirms owned fixture47230 exited0 and is absent, port18080 free, app absent and network0/1/1.
+All runtime was root-owned. No source, test, fixture, input, build or device action followed
+the final PASS; only this ledger and TRACK metadata changed before the implementation commit.
+
+Final A/B manifest: `repair4-frozen-01/manifest.json`, SHA256
+`98453301465eda78219b8d8ad56336fa20b17e5b04b946616ae494175526db62`.
+Native24/M08 manifest: `repair4-regressions-01/manifest.json`, SHA256
+`612407f2610cd18941c06a9d029cf6bb5c99315cf4733a4072481f3b2347dd06`.
+Their `rootCommands` contain the exact frozen invocations; all earlier manifests/raw remain.
+App SHA256 `20a3793542f23e63037badeb3aa4d81e678dcd623b14207b3fc3b08333d85c01`;
+test SHA256 `2b13132f41dd893d177673ee736b7a9945215f88899e9fb92d2b79504ada2557`.
+Build-source bindings and host28/fixture14 evidence remain at `fixed-built-01/manifest.json`;
+unchanged successful builds/tests were not repeated during either later harness repair.
diff --git a/verification/activity_state.py b/verification/activity_state.py
index d7fe5d3..8ce9fd6 100644
--- a/verification/activity_state.py
+++ b/verification/activity_state.py
@@ -117,7 +117,7 @@ class ActivityStateScenario(OfflineQueueScenario):
                         shutil.copyfile(source, local / source.name)
                 with sqlite3.connect(local / DB_NAME) as database:
                     assert database.execute("PRAGMA integrity_check").fetchone()[0] == "ok"
-                    assert database.execute("PRAGMA user_version").fetchone()[0] == 5
+                    assert database.execute("PRAGMA user_version").fetchone()[0] == self.args.schema_version
                     def rows(table, order):
                         fields = [row[1] for row in database.execute(f"PRAGMA table_info({table})")]
                         return [dict(zip(fields, row)) for row in database.execute(f"SELECT * FROM {table} ORDER BY {order}")]
@@ -128,6 +128,16 @@ class ActivityStateScenario(OfflineQueueScenario):
                     assert rows("acknowledged_mutations", "clientMutationId") == point["acknowledged"] == []
                     assert rows("tombstones", "id") == point["tombstones"] == []
                     assert rows("sync_metadata", "id") == []
+                    if self.args.schema_version == 6:
+                        assert [row[1] for row in database.execute("PRAGMA table_info(automatic_sync)")] == ["id", "workId", "httpAttempts", "state"]
+                        cycles = rows("automatic_sync", "id")
+                        if stage == "submitted":
+                            cycle, = cycles
+                            assert cycle["id"] == 1 and cycle["httpAttempts"] == 0 and cycle["state"] == "ACTIVE"
+                            assert re.fullmatch(r"[0-9a-f]{8}(?:-[0-9a-f]{4}){3}-[0-9a-f]{12}", cycle["workId"])
+                        else:
+                            assert cycles == []
+                        self.result.setdefault("automaticCycles", {})[stage] = cycles
             if stage != "submitted":
                 assert items == [seed] and pending == [], (stage, items, pending)
             else:
@@ -163,7 +173,11 @@ class ActivityStateScenario(OfflineQueueScenario):
         self.adb("logcat", "-c")
         self.fixture_offset = self.args.fixture_log.stat().st_size
         self.result["fixtureLogStartOffset"] = self.fixture_offset
+        if self.args.offline:
+            self.go_offline()
         code, text = self.instrument()
+        if self.args.offline:
+            self.result["offlineAfterLifecycle"] = self.wait_network(False)
         self.adb("logcat", "-d", "-v", "threadtime")
         self.result["pidAfterJUnit"] = self.adb("shell", "pidof", PACKAGE, allow_failure=True)
         native = self.export_native()
@@ -189,6 +203,8 @@ def main():
     parser.add_argument("--adb", type=Path, default=Path(os.environ.get("ANDROID_HOME", "")) / "platform-tools/adb")
     parser.add_argument("--fixture-log", type=Path, required=True)
     parser.add_argument("--output", type=Path, required=True)
+    parser.add_argument("--schema-version", type=int, choices=(5, 6), default=5)
+    parser.add_argument("--offline", action="store_true", help="Keep the actual network unmet through the lifecycle case")
     args = parser.parse_args()
     scenario = ActivityStateScenario(args)
     started = time.monotonic()
diff --git a/verification/background_work.py b/verification/background_work.py
new file mode 100644
index 0000000..2c693ef
--- /dev/null
+++ b/verification/background_work.py
@@ -0,0 +1,491 @@
+#!/usr/bin/env python3
+"""Root-owned M10 A/B: real UI commit, persistent OS work, SIGKILL and bounded HTTP.
+
+No test instrumentation, DB seed, direct Worker, or app entrypoint is used between
+loss and OS dispatch. Only the debug clock file is advanced; native databases are
+copied read-only. A shell 'Running job' response never substitutes for completion.
+"""
+import argparse
+import hashlib
+import json
+from pathlib import Path
+import re
+import shlex
+import shutil
+import sqlite3
+import subprocess
+import tempfile
+import time
+import uuid
+
+from background_baseline import BackgroundBaseline, package_stopped, registered_jobs, resumed_activity_lines
+from offline_queue_restart import OfflineQueueScenario
+from process_restart import DB_NAME, PACKAGE, SERIAL
+from version_conflict_restart import VersionConflictScenario
+
+INPUTS_PATH = Path(__file__).with_name('M10-work-inputs.json')
+INPUTS = json.loads(INPUTS_PATH.read_text())
+WORK_DB = 'androidx.work.workdb'
+SERVICE = 'androidx.work.impl.background.systemjob.SystemJobService'
+
+
+def sha(data):
+    return hashlib.sha256(data).hexdigest()
+
+
+def job_block(text, work_id, job_id):
+    headers = registered_jobs(text)
+    assert len(headers) == 1, ('Expected one unique registered job', headers)
+    header = headers[0]
+    assert re.match(rf'JOB #\S+/{job_id}:\s+', header), header
+    assert f'{PACKAGE}/{SERVICE}' in header, header
+    lines = text.splitlines()
+    start, = [n for n, line in enumerate(lines) if line.strip() == header]
+    indent = len(lines[start]) - len(lines[start].lstrip())
+    block = []
+    for line in lines[start + 1:]:
+        if line.strip() and len(line) - len(line.lstrip()) <= indent:
+            break
+        block.append(line.strip())
+    content = '\n'.join(block)
+    # SystemIdInfo is the exact native WorkSpec -> OS-job mapping. If dumpsys
+    # renders the extras, also reject an inconsistent ID or generation there.
+    if 'EXTRA_WORK_SPEC_ID=' in content:
+        assert f'EXTRA_WORK_SPEC_ID={work_id}' in content, content
+        assert 'EXTRA_WORK_SPEC_GENERATION=0' in content, content
+    assert re.search(r'^Required constraints:.*\bCONNECTIVITY\b', content, re.M), content
+    return dict(header=header, lines=block)
+
+
+def job_eligibility(block):
+    constraints = {}
+    for label in ('Satisfied', 'Unsatisfied'):
+        line, = [line for line in block['lines'] if line.startswith(label + ' constraints:')]
+        value = re.sub(r' \[0x[0-9a-f]+\]$', '', line.split(':', 1)[1].strip())
+        assert re.fullmatch(r'(?:[A-Z_]+(?: [A-Z_]+)*)?', value), line
+        constraints[label] = set(value.split())
+    line, = [line for line in block['lines'] if line.startswith('Ready:')]
+    names = ('job', 'user', '!restricted', '!pending', '!active', '!backingup', 'comp')
+    pattern = r'Ready: (?:true|false) \(' + ' '.join(re.escape(name) + r'=(true|false)' for name in names) + r'\)'
+    match = re.fullmatch(pattern, line)
+    assert match, line
+    flags = {name: value == 'true' for name, value in zip(names, match.groups())}
+    already_queued = not flags['!pending'] or not flags['!active']
+    connected = 'CONNECTIVITY' in constraints['Satisfied'] and 'CONNECTIVITY' not in constraints['Unsatisfied']
+    # Nonforced run may bypass only the already-tested timing delay here;
+    # actual connectivity and the remaining execution conditions must be ready.
+    eligible = connected and constraints['Unsatisfied'] <= {'TIMING_DELAY'} and all(
+        flags[name] for name in names if name != 'job')
+    return dict(eligible=eligible, alreadyQueued=already_queued, connectivitySatisfied=connected,
+                unsatisfied=sorted(constraints['Unsatisfied']), executionFlags=flags)
+
+
+class BackgroundWork(OfflineQueueScenario):
+    pre_seed_teardown = VersionConflictScenario.pre_seed_teardown
+    wait_background = BackgroundBaseline.wait_background
+    wait_absent = BackgroundBaseline.wait_absent
+
+    def __init__(self, args):
+        super().__init__(args)
+        self.invocations, self.native_count = [], 0
+        self.result.update(scenario='M10', case=args.case, expectation='persistent-work',
+                           harnessSha256=sha(Path(__file__).read_bytes()),
+                           inputsSha256=sha(INPUTS_PATH.read_bytes()),
+                           fixtureLog=str(args.fixture_log.resolve()), testApkUsed=False,
+                           queueSeeded=False, instrumentationRuns=0)
+
+    def adb(self, *arguments, binary=False, allow_failure=False, stdin=None):
+        command = [str(self.args.adb), '-s', SERIAL, *arguments]
+        self.command_count += 1
+        self.invocations.append(arguments)
+        prefix = self.output / f'command-{self.command_count:03d}'
+        if stdin is not None:
+            prefix.with_suffix('.stdin').write_bytes(stdin)
+        with (self.output / 'commands.txt').open('a') as log:
+            log.write(f'{self.command_count:03d} {shlex.join(command)}\n')
+        started = time.monotonic()
+        result = subprocess.run(command, input=stdin, capture_output=True,
+                                timeout=INPUTS['harness']['adbTimeoutSeconds'])
+        prefix.with_suffix('.stdout').write_bytes(result.stdout)
+        prefix.with_suffix('.stderr').write_bytes(result.stderr)
+        self.last_command = dict(number=self.command_count, command=command, arguments=arguments,
+                                 startedMonotonic=started, endedMonotonic=time.monotonic(),
+                                 exit=result.returncode, stdoutBytes=len(result.stdout),
+                                 stdoutSha256=sha(result.stdout), stderrBytes=len(result.stderr),
+                                 stderrSha256=sha(result.stderr),
+                                 stdinBytes=0 if stdin is None else len(stdin),
+                                 stdinSha256=None if stdin is None else sha(stdin))
+        with (self.output / 'command-records.jsonl').open('a') as log:
+            log.write(json.dumps(self.last_command) + '\n')
+        with (self.output / 'commands.txt').open('a') as log:
+            log.write(f'    exit={result.returncode}\n')
+        if not allow_failure:
+            assert result.returncode == 0, (command, result.returncode, result.stderr)
+        return result.stdout if binary else result.stdout.decode('utf-8', errors='replace').strip()
+
+    def set_clock(self, value):
+        data = f'{value}\n'.encode('ascii')
+        path = INPUTS['clockFile']
+        output = self.adb('shell', '-T', 'run-as', PACKAGE, 'tee', path + '.next', stdin=data, binary=True)
+        assert output == data and self.last_command['stderrBytes'] == 0
+        self.adb('shell', 'run-as', PACKAGE, 'mv', path + '.next', path)
+        assert self.adb('shell', '-T', '-n', 'run-as', PACKAGE, 'cat', path, binary=True) == data
+        self.clock = value
+        self.result.setdefault('clockChanges', []).append(dict(value=value, lastCommand=self.command_count))
+
+    def native(self, kind, stage):
+        directory, name, version = ('databases', DB_NAME, INPUTS['schemaVersion']) if kind == 'items' else (
+            'no_backup', WORK_DB, INPUTS['workDatabaseVersion'])
+        self.native_count += 1
+        folder = self.output / f'native-{self.native_count:02d}-{kind}-{stage}'
+        folder.mkdir()
+        names = self.adb('shell', 'run-as', PACKAGE, 'ls', directory).splitlines()
+        assert name in names, (directory, names)
+        for suffix in ('', '-wal'):
+            filename = name + suffix
+            if filename in names:
+                (folder / filename).write_bytes(self.adb('exec-out', 'run-as', PACKAGE, 'cat',
+                                                         directory + '/' + filename, binary=True))
+        with tempfile.TemporaryDirectory(prefix='mse-m10-native-') as temporary:
+            local = Path(temporary)
+            for file in folder.iterdir():
+                shutil.copyfile(file, local / file.name)
+            with sqlite3.connect(local / name) as database:
+                assert database.execute('PRAGMA integrity_check').fetchone()[0] == 'ok'
+                assert database.execute('PRAGMA user_version').fetchone()[0] == version
+                tables = ('items', 'pending_mutations', 'acknowledged_mutations', 'tombstones',
+                          'sync_metadata', 'automatic_sync', 'sqlite_sequence') if kind == 'items' else (
+                    'WorkSpec', 'WorkName', 'SystemIdInfo', 'Dependency', 'Preference')
+                extracted = {}
+                for table in tables:
+                    fields = [row[1] for row in database.execute(f'PRAGMA table_info({table})')]
+                    assert fields, table
+                    extracted[table] = [dict(zip(fields, [dict(hex=x.hex()) if isinstance(x, bytes) else x for x in row]))
+                                        for row in database.execute(f'SELECT * FROM {table} ORDER BY 1')]
+                if kind == 'items':
+                    schema = json.loads((Path(__file__).parents[1] / 'app/schemas/com.mobilesystemsevolution.kotlin.ItemDatabase/6.json').read_text())['database']
+                    for table in schema['entities']:
+                        fields = [row[1] for row in database.execute(f"PRAGMA table_info({table['tableName']})")]
+                        assert fields == [field['columnName'] for field in table['fields']], fields
+        record = dict(kind=kind, stage=stage, schemaVersion=version, tables=extracted,
+                      lastCommand=self.command_count,
+                      raw={p.name: sha(p.read_bytes()) for p in folder.iterdir()})
+        (folder / 'state.json').write_text(json.dumps(record, indent=2) + '\n')
+        self.result.setdefault('nativeSnapshots', []).append(str(folder / 'state.json'))
+        return extracted
+
+    def expected_pending(self, dispatched):
+        return dict(sequence=1, itemId=INPUTS['itemId'], operation='CREATE', title=INPUTS['title'], completed=0,
+                    clientMutationId=INPUTS['clientMutationId'], payloadHash=INPUTS['payloadHash'],
+                    terminalError=None, baseVersion=None, dispatched=int(dispatched))
+
+    def assert_items(self, tables, attempt, terminal=False):
+        success = terminal and self.args.case == 'A'
+        item = dict(id=INPUTS['itemId'], title=INPUTS['title'], completed=0,
+                    version=int(success), updatedAt=INPUTS['firstSuccessTimestamp'])
+        assert tables['items'] == [item], tables
+        assert tables['pending_mutations'] == ([] if success else [self.expected_pending(attempt > 0)]), tables
+        assert tables['tombstones'] == tables['sync_metadata'] == []
+        assert tables['sqlite_sequence'] == [dict(name='pending_mutations', seq=1)]
+        state = ('COMPLETE' if success else 'DEFERRED') if terminal else 'ACTIVE'
+        assert tables['automatic_sync'] == [dict(id=1, workId=self.work_id, httpAttempts=attempt, state=state)], tables
+        receipts = tables['acknowledged_mutations']
+        if success:
+            receipt, = receipts
+            assert (receipt['clientMutationId'], receipt['payloadHash'], receipt['statusCode']) == (
+                INPUTS['clientMutationId'], INPUTS['payloadHash'], 201)
+            assert json.loads(receipt['responseBody']) == dict(item={**item, 'completed': False})
+        else:
+            assert receipts == []
+
+    def work(self, stage):
+        tables = self.native('work', stage)
+        spec, = tables['WorkSpec']
+        assert spec['worker_class_name'] == INPUTS['workClass']
+        assert spec['required_network_type'] == 1 and spec['backoff_policy'] == 0
+        assert spec['backoff_delay_duration'] == 10000 and spec['generation'] == 0
+        assert spec['interval_duration'] == spec['initial_delay'] == spec['run_in_foreground'] == 0
+        uuid.UUID(spec['id'])
+        assert tables['WorkName'] == [dict(name=INPUTS['uniqueWorkName'], work_spec_id=spec['id'])]
+        assert tables['Dependency'] == []
+        if hasattr(self, 'work_id'):
+            assert spec['id'] == self.work_id, 'WorkSpec replaced during the same cycle'
+        return tables, spec
+
+    def bind_job(self, tables, label):
+        mapping, = tables['SystemIdInfo']
+        assert mapping['work_spec_id'] == self.work_id and mapping['generation'] == 0
+        # WorkManager may assign a new OS job ID when rescheduling this same
+        # WorkSpec. Bind the current native mapping, not the prior numeric ID.
+        self.job_id = mapping['system_id']
+        deadline = time.monotonic() + INPUTS['harness']['workStateWaitSeconds']
+        while True:
+            text = self.adb('shell', 'dumpsys', 'jobscheduler', PACKAGE)
+            if registered_jobs(text):
+                block = job_block(text, self.work_id, self.job_id)
+                self.result.setdefault('osRegistrations', []).append(dict(label=label, jobId=self.job_id,
+                    workId=self.work_id, namespace=None, command=self.command_count, **block))
+                return
+            assert time.monotonic() < deadline, 'WorkSpec has no corresponding OS job'
+            time.sleep(0.2)
+
+    def remote(self, attempts):
+        state = self.http('/__m10')
+        assert state['case'] == self.args.case and state['getRequests'] == 0
+        assert state['mutationRequests'] == attempts and state['duplicates'] == 0, state
+        expected = INPUTS['cases'][self.args.case][:attempts]
+        assert [entry['status'] for entry in state['requests']] == expected
+        wire = dict(id=INPUTS['itemId'], title=INPUTS['title'], completed=False,
+                    clientMutationId=INPUTS['clientMutationId'], payloadHash=INPUTS['payloadHash'])
+        assert all(entry['request'] == wire and entry['actualPayloadHash'] == INPUTS['payloadHash']
+                   and entry['method'] == 'POST' and entry['path'] == '/items' for entry in state['requests'])
+        success = attempts == 3 and self.args.case == 'A'
+        assert state['applied'] == int(success) and state['identityConflicts'] == state['versionConflicts'] == 0
+        assert state['nextTimestamp'] == INPUTS['firstSuccessTimestamp'] + (1000 if success else 0)
+        assert state['items'] == ([dict(id=INPUTS['itemId'], title=INPUTS['title'], completed=False,
+                                      version=1, updatedAt=INPUTS['firstSuccessTimestamp'])] if success else [])
+        return state
+
+    def run_job(self, expectation):
+        stdout = self.adb('shell', 'cmd', 'jobscheduler', 'run', '-u', '0', PACKAGE, str(self.job_id), allow_failure=True)
+        stderr = (self.output / f'command-{self.command_count:03d}.stderr').read_text().strip()
+        record = dict(self.last_command, expectation=expectation, stdout=stdout, stderr=stderr)
+        self.result.setdefault('schedulerInvocations', []).append(record)
+        if expectation == 'offline':
+            assert record['exit'] != 0 and stderr == f'Job {self.job_id} in package {PACKAGE} / user 0 has functional constraints but --force not specified', record
+        elif expectation == 'terminal-absent':
+            assert record['exit'] != 0 and stderr == f'Could not find job {self.job_id} in package {PACKAGE} / user 0', record
+        else:
+            assert record['exit'] == 0 and stdout == 'Running job' and stderr == '', record
+
+    def await_attempt(self, attempt, dispatch_when_ready=False):
+        deadline = time.monotonic() + INPUTS['harness']['workStateWaitSeconds']
+        terminal = attempt == 3
+        desired = (2 if self.args.case == 'A' else 3) if terminal else 0
+        while True:
+            tables, spec = self.work(f'attempt-{attempt}-observe')
+            state = self.http('/__m10')
+            assert state['mutationRequests'] <= attempt and state['getRequests'] == 0, state
+            if spec['state'] == desired and spec['run_attempt_count'] == attempt and state['mutationRequests'] == attempt:
+                self.assert_items(self.native('items', f'attempt-{attempt}'), attempt, terminal=terminal)
+                self.remote(attempt)
+                self.result.setdefault('attempts', []).append(dict(http=attempt, workState=spec['state'],
+                    workRunAttempt=spec['run_attempt_count'], clock=self.clock,
+                    lastEnqueueTime=spec['last_enqueue_time'], workId=self.work_id))
+                return tables, spec
+            if dispatch_when_ready:
+                observation = dict(attempt=attempt, workId=self.work_id, command=self.command_count,
+                                   nativeState=spec['state'], nativeRunAttempt=spec['run_attempt_count'],
+                                   http=state['mutationRequests'])
+                if state['mutationRequests'] == attempt or spec['state'] == 1 or spec['run_attempt_count'] == attempt:
+                    dispatch_when_ready = False
+                    observation['naturalExecutionObserved'] = True
+                else:
+                    assert spec['state'] == 0 and spec['run_attempt_count'] == attempt - 1
+                    mapping, = tables['SystemIdInfo']
+                    assert mapping['work_spec_id'] == self.work_id and mapping['generation'] == 0
+                    self.job_id = mapping['system_id']
+                    text = self.adb('shell', 'dumpsys', 'jobscheduler', PACKAGE)
+                    observation.update(jobId=self.job_id, namespace=None, command=self.command_count)
+                    # The OS may start and finish while its dump is being read.
+                    # A real request prevents another trigger of that attempt.
+                    current = self.http('/__m10')
+                    assert current['mutationRequests'] in (attempt - 1, attempt) and current['getRequests'] == 0
+                    if current['mutationRequests'] == attempt:
+                        dispatch_when_ready = False
+                        observation.update(naturalExecutionObserved=True, http=attempt)
+                    else:
+                        block = job_block(text, self.work_id, self.job_id)
+                        readiness = job_eligibility(block)
+                        observation.update(**readiness, **block)
+                        if readiness['alreadyQueued']:
+                            dispatch_when_ready = False
+                        elif readiness['eligible']:
+                            dispatch_when_ready = False
+                            self.run_job('eligible')
+                self.result.setdefault('eligibilityObservations', []).append(observation)
+            assert time.monotonic() < deadline, (attempt, spec, state)
+            time.sleep(0.2)
+
+    def activity_id(self):
+        text = self.adb('shell', 'dumpsys', 'activity', 'activities')
+        records = {re.search(r'ActivityRecord\{([^ ]+)', line).group(1)
+                   for line in resumed_activity_lines(text) if PACKAGE in line}
+        assert len(records) == 1, records
+        return records.pop()
+
+    def run(self):
+        self.result['initialNetwork'] = self.wait_network(True)
+        self.adb('shell', 'am', 'force-stop', PACKAGE)
+        self.pre_seed_teardown('before-install')
+        self.adb('install', '-r', str(self.args.apk.resolve()))
+        assert self.adb('shell', 'pm', 'clear', PACKAGE) == 'Success'
+        self.pre_seed_teardown('after-clear')
+        assert registered_jobs(self.adb('shell', 'dumpsys', 'jobscheduler', PACKAGE)) == []
+        self.http('/__m10/reset', dict(case=self.args.case))
+        self.fixture_offset = self.args.fixture_log.stat().st_size
+        self.result['fixtureLogStartOffset'] = self.fixture_offset
+        self.remote(0)
+        self.go_offline()
+        self.adb('shell', 'input', 'keyevent', 'KEYCODE_WAKEUP')
+        self.adb('shell', 'wm', 'dismiss-keyguard')
+        self.adb('logcat', '-c')
+        self.adb('shell', 'run-as', PACKAGE, 'mkdir', '-p', 'files')
+        now = self.adb('shell', 'date', '+%s')
+        assert now.isdigit()
+        self.set_clock((int(now) + 2) * 1000)
+        self.adb('shell', 'am', 'start', '-W', '-n', f'{PACKAGE}/.MainActivity')
+        self.completed_ui([])
+        self.tap(**{'class': 'android.widget.EditText'})
+        self.text(INPUTS['title'])
+        self.tap(text='Add')
+        self.completed_ui([INPUTS['title']])
+        self.wait_text('Pending changes: 1')
+        tables, spec = self.work('registered')
+        self.work_id = spec['id']
+        assert spec['state'] == spec['run_attempt_count'] == 0
+        self.assert_items(self.native('items', 'real-ui-committed'), 0)
+        self.bind_job(tables, 'real-ui-committed')
+        self.remote(0)
+        first_activity = self.activity_id()
+        # Standard launchMode + CLEAR_TOP creates a new Activity. Its ordinary
+        # startup recovery requests the same work again, before the loss boundary.
+        self.adb('shell', 'am', 'start', '-W', '--activity-clear-top', '-n', f'{PACKAGE}/.MainActivity')
+        self.completed_ui([INPUTS['title']])
+        second_activity = self.activity_id()
+        assert first_activity != second_activity
+        repeated, spec = self.work('repeated-registration')
+        assert repeated == tables, 'Repeated ordinary startup changed the unique chain'
+        self.assert_items(self.native('items', 'repeated-registration'), 0)
+        self.bind_job(repeated, 'repeated-registration')
+        self.result['repeatedScheduling'] = dict(firstActivity=first_activity, secondActivity=second_activity,
+                                                workId=self.work_id, activeChains=1)
+
+        self.adb('shell', 'input', 'keyevent', 'KEYCODE_HOME')
+        self.wait_background()
+        uid = self.adb('shell', 'run-as', PACKAGE, 'id', '-u')
+        pid = self.adb('shell', 'pidof', PACKAGE)
+        assert uid.isdigit() and pid.isdigit() and int(pid) > 1
+        status = self.adb('shell', 'run-as', PACKAGE, 'cat', f'/proc/{pid}/status')
+        match = re.search(r'^Uid:\s+(\d+)\s+(\d+)\s+(\d+)\s+(\d+)\s*$', status, re.M)
+        assert match and all(x == uid for x in match.groups())
+        assert self.adb('shell', 'pidof', PACKAGE) == pid
+        assert not package_stopped(self.adb('shell', 'dumpsys', 'package', PACKAGE))
+        self.result.update(pidBefore=int(pid), targetUid=int(uid), lossCommand=self.command_count + 1,
+                           lossMethod='same-UID SIGKILL after HOME', killRequestedMonotonic=time.monotonic())
+        self.adb('shell', 'run-as', PACKAGE, 'kill', '-9', pid)
+        self.result['absenceMonotonic'] = self.wait_absent()
+        assert not package_stopped(self.adb('shell', 'dumpsys', 'package', PACKAGE))
+        killed, spec = self.work('killed-offline')
+        assert killed == repeated
+        self.assert_items(self.native('items', 'killed-offline'), 0)
+        self.bind_job(killed, 'killed-offline')
+        self.result['offlineAfterLoss'] = self.wait_network(False)
+        self.run_job('offline')
+        assert not self.adb('shell', 'pidof', PACKAGE, allow_failure=True)
+        self.remote(0)
+
+        self.result['onlineForOsDispatch'] = self.restore_network()
+        tables, spec = self.await_attempt(1, dispatch_when_ready=True)
+        replacement = self.adb('shell', 'pidof', PACKAGE)
+        assert replacement.isdigit() and replacement != pid
+        self.result.update(pidAfter=int(replacement), osRecoveryObservedCommand=self.command_count)
+        assert not package_stopped(self.adb('shell', 'dumpsys', 'package', PACKAGE))
+        resumed = resumed_activity_lines(self.adb('shell', 'dumpsys', 'activity', 'activities'))
+        assert resumed and all(PACKAGE not in line for line in resumed), resumed
+        for attempt, delay in ((2, 10000), (3, 20000)):
+            self.bind_job(tables, f'before-attempt-{attempt}')
+            due = spec['last_enqueue_time'] + delay
+            assert due > self.clock and spec['last_enqueue_time'] == self.clock
+            self.set_clock(due)
+            tables, spec = self.await_attempt(attempt, dispatch_when_ready=True)
+        assert self.adb('shell', 'pidof', PACKAGE) == replacement
+        self.result['terminalNative'] = self.native('items', 'terminal')
+        self.assert_items(self.result['terminalNative'], 3, terminal=True)
+
+        logcat = self.adb('logcat', '-d', '-v', 'threadtime')
+        starts = [line for line in logcat.splitlines() if f'Start proc {replacement}:' in line and PACKAGE in line]
+        assert len(starts) == 1 and 'for service' in starts[0] and SERVICE in starts[0], starts
+        dispatches = [line for line in logcat.splitlines()
+                      if re.search(rf'^\S+\s+\S+\s+{replacement}\s+', line)
+                      and 'WM-SystemJobService' in line and 'onStartJob' in line and self.work_id in line]
+        assert dispatches, 'No SystemJobService onStartJob in the replacement process'
+        actual_runs = [line for line in logcat.splitlines() if 'ItemSyncWorker' in line and f'work={self.work_id}' in line]
+        assert [int(re.search(r'runAttempt=(\d+)', line).group(1)) for line in actual_runs] == [0, 1, 2], actual_runs
+        self.result['osDispatchProof'] = dict(processStart=starts[0], jobService=dispatches, workerRuns=actual_runs)
+        self.result['unattendedEndCommand'] = self.command_count
+
+        if self.args.case == 'B':
+            deadline = time.monotonic() + INPUTS['harness']['workStateWaitSeconds']
+            while registered_jobs(self.adb('shell', 'dumpsys', 'jobscheduler', PACKAGE)):
+                assert time.monotonic() < deadline, 'Terminal job was not removed'
+                time.sleep(0.2)
+            self.set_clock(self.clock + INPUTS['terminalEligibilityAdvanceMs'])
+            self.run_job('terminal-absent')
+            self.remote(3)
+            before_ui = self.native('items', 'terminal-eligible')
+            self.assert_items(before_ui, 3, terminal=True)
+            self.adb('shell', 'am', 'start', '-W', '-n', f'{PACKAGE}/.MainActivity')
+            self.completed_ui([INPUTS['title']])
+            self.wait_text('Pending changes: 1')
+            self.wait_text('Stale local data · sync error')
+            self.wait_for(lambda tree: any('Automatic sync deferred after 3 HTTP attempts' in n.get('text', '')
+                                          for n in tree.iter('node')), 'durable deferred state')
+            assert self.native('items', 'ordinary-ui-deferred') == before_ui
+            terminal_work, terminal_spec = self.work('ordinary-ui-deferred')
+            assert terminal_spec['state'] == 3 and terminal_spec['run_attempt_count'] == 3
+            assert registered_jobs(self.adb('shell', 'dumpsys', 'jobscheduler', PACKAGE)) == []
+            self.remote(3)
+            self.result['ordinaryUiAfterTerminal'] = True
+
+        window = self.invocations[self.result['lossCommand']:self.result['unattendedEndCommand']]
+        forbidden = [args for args in window if args[:3] in (
+            ('shell', 'am', 'start'), ('shell', 'am', 'instrument'), ('shell', 'am', 'force-stop'),
+            ('shell', 'pm', 'clear')) or args[:1] in (('install',), ('exec-in',))]
+        assert forbidden == [], forbidden
+        data = self.args.fixture_log.read_bytes()
+        events = [json.loads(line) for line in data[self.fixture_offset:].decode().splitlines() if line.startswith('{')]
+        requests = [e for e in events if e.get('path') == '/items' or e.get('path', '').startswith('/items/')]
+        assert len(requests) == 3 and [e['status'] for e in requests] == INPUTS['cases'][self.args.case]
+        self.result.update(status='PASS', finalRemote=self.remote(3), fixtureLogEndOffset=len(data),
+                           actualHttpRequests=3, pending=0 if self.args.case == 'A' else 1,
+                           productionCreationProved=True, schedulerRegistrationProved=True,
+                           noAppEntrypointBetweenLossAndOsRecovery=True)
+
+
+def main():
+    parser = argparse.ArgumentParser(description=__doc__)
+    parser.add_argument('--case', choices=('A', 'B'), required=True)
+    parser.add_argument('--apk', type=Path, required=True)
+    parser.add_argument('--test-apk', type=Path, required=True, help='Hash reference only; not installed or invoked')
+    parser.add_argument('--adb', type=Path, required=True)
+    parser.add_argument('--fixture-log', type=Path, required=True)
+    parser.add_argument('--output', type=Path, required=True)
+    args = parser.parse_args()
+    args.expect = 'persistent-work'
+    args.schema_version = INPUTS['schemaVersion']
+    scenario = BackgroundWork(args)
+    started = time.monotonic()
+    try:
+        scenario.run()
+    except Exception as error:
+        scenario.result.update(status='FAIL', error=repr(error))
+        raise
+    finally:
+        try:
+            scenario.adb('shell', 'am', 'force-stop', PACKAGE)
+            absent = not scenario.adb('shell', 'pidof', PACKAGE, allow_failure=True)
+            network = scenario.restore_network()
+            scenario.result['cleanup'] = dict(appAbsent=absent, network=network)
+            assert absent
+        except Exception as error:
+            scenario.result.update(status='FAIL', cleanupError=repr(error))
+            raise
+        finally:
+            scenario.result.update(adbCommands=scenario.command_count, elapsedSeconds=round(time.monotonic() - started, 3))
+            (scenario.output / 'result.json').write_text(json.dumps(scenario.result, indent=2) + '\n')
+            print(json.dumps(scenario.result, indent=2), flush=True)
+
+
+if __name__ == '__main__':
+    main()
