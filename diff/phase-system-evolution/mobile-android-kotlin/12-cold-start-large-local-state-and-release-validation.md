# M15 — Cold Start, Large Local State와 Release Validation

## `feat: bound M15 release reads and virtualize item pages`

diff --git a/TRACK.md b/TRACK.md
index abb8c7d..60f1683 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -302,6 +302,30 @@ retained, not rerun. Initial sandbox denials and root observation-parser diagnos
 recorded in `verification/M14.md`. Attempt1, repair0, fixed-case usage1/3. Root confirmed the
 unchanged app/tested source bytes and actual cleanup; final commit/tag audit remains root-owned.
 
+## M15 boundary — phase-1 release paging
+
+The release baseline preserved M14's query/UI behavior and reproduced two unbounded
+2000-row reads (4000 cumulative Items). The screen now requests 25-row pages through a
+transactional scalar count and `LIMIT/OFFSET`, preserving SQLite rowid order. The page
+offset is published only after a successful read. A `LazyColumn` renders visible rows;
+the scrollable editor/status header is height-bounded so rows remain reachable. Schema6,
+committed CRUD, mutation identity, synchronization and automatic-work behavior are unchanged.
+
+Root verified the actual nondebuggable release and same-signature audit instrumentation.
+The isolated31-row native regression passed without an Activity. Final ordinary offline
+startup in PID19933 materialized two pages of25 (cumulative50), with peak7 composed rows.
+Seventy-nine actual Next presses reached Item2000; one additional Next reached the new
+local Item during rename/complete/create/delete. The final native audit found2000 Items,
+1999 unchanged canonical rows and4 original pending envelopes; automatic HTTP attempts0.
+All86 ordinary-window Item queries were bounded. The observed first-usable time3.497718s
+includes external UI sampling and is not a timing SLA or benchmark sweep.
+
+Host33, focused controller4+1 and the actual release native/UI cases passed. Earlier
+M01–M10/M14 Android guarantees retain their original evidence; their full device suites
+were not rerun on this release. Attempt1, repair0, full2000 usage2/3 (baseline and final).
+Preparation failures are retained. Root confirmed frozen bytes, native data and cleanup;
+history/tag audit remains root-owned. See `verification/phase-1/M15.md` for the exact evidence.
+
 ## Pinned build
 
 - JDK 17, Gradle 8.7 (wrapper distribution SHA-256 pinned)
diff --git a/app/build.gradle.kts b/app/build.gradle.kts
index 89bde97..0fdf26a 100644
--- a/app/build.gradle.kts
+++ b/app/build.gradle.kts
@@ -4,9 +4,14 @@ plugins {
     id("org.jetbrains.kotlin.kapt")
 }
 
+// The release audit has no Activity rule. Ordinary debug test selection is unchanged.
+val verificationTestBuildType = providers.gradleProperty("verificationTestBuildType").orElse("debug").get()
+require(verificationTestBuildType in setOf("debug", "release"))
+
 android {
     namespace = "com.mobilesystemsevolution.kotlin"
     compileSdk = 35
+    testBuildType = verificationTestBuildType
 
     defaultConfig {
         applicationId = "com.mobilesystemsevolution.kotlin"
@@ -14,7 +19,23 @@ android {
         targetSdk = 34
         versionCode = 1
         versionName = "0.1"
-        testInstrumentationRunner = "com.mobilesystemsevolution.kotlin.M05ScenarioInstrumentation"
+        testInstrumentationRunner = if (verificationTestBuildType == "release") {
+            "androidx.test.runner.AndroidJUnitRunner"
+        } else {
+            "com.mobilesystemsevolution.kotlin.M05ScenarioInstrumentation"
+        }
+    }
+
+    buildTypes.getByName("release") {
+        isDebuggable = false
+        // This local verification artifact uses the same test key as its target instrumentation.
+        signingConfig = signingConfigs.getByName("debug")
+    }
+    if (verificationTestBuildType == "release") {
+        sourceSets.getByName("androidTest") {
+            java.setSrcDirs(listOf("src/releaseAudit/java"))
+            assets.setSrcDirs(listOf("src/releaseAudit/assets"))
+        }
     }
 
     buildFeatures { compose = true }
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemDatabase.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemDatabase.kt
index 3be4c71..287b43d 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemDatabase.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemDatabase.kt
@@ -78,12 +78,34 @@ data class AutomaticCycle(
 private fun List<PendingMutation>.nextOrdinaryMutation(): PendingMutation? =
     firstOrNull { it.terminalError != "version_conflict" && it.terminalError != "base_version_unknown" }
 
+internal const val ITEM_READ_ALL_SQL = "SELECT * FROM items ORDER BY rowid"
+internal const val ITEM_READ_PAGE_SQL = "SELECT * FROM items ORDER BY rowid LIMIT :limit OFFSET :offset"
+internal const val ITEM_COUNT_SQL = "SELECT COUNT(*) FROM items"
+
+data class ItemEntityPage(val items: List<ItemEntity>, val totalCount: Int, val offset: Int)
+
 @Dao
 interface ItemDao {
     // Preserve the M01 insertion order without adding a future metadata column.
-    @Query("SELECT * FROM items ORDER BY rowid")
+    @Query(ITEM_READ_ALL_SQL)
     suspend fun readAll(): List<ItemEntity>
 
+    @Query(ITEM_COUNT_SQL)
+    suspend fun countItems(): Int
+
+    @Query(ITEM_READ_PAGE_SQL)
+    suspend fun readPageRows(limit: Int, offset: Int): List<ItemEntity>
+
+    @Transaction
+    suspend fun readPage(limit: Int, offset: Int): ItemEntityPage {
+        require(limit in 1..50 && offset >= 0)
+        val count = countItems()
+        // Deleting the final row on a page moves to the preceding nonempty page.
+        val lastPage = if (count == 0) 0 else ((count - 1) / limit) * limit
+        val boundedOffset = offset.coerceAtMost(lastPage)
+        return ItemEntityPage(readPageRows(limit, boundedOffset), count, boundedOffset)
+    }
+
     @Query("SELECT * FROM items WHERE id = :id")
     suspend fun readItem(id: String): ItemEntity?
 
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemStore.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemStore.kt
index acd9acd..f4d4e0c 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemStore.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemStore.kt
@@ -11,6 +11,13 @@ data class Item(
     val updatedAt: Long,
 )
 
+internal const val ITEM_PAGE_SIZE = 25
+
+data class ItemPage(val items: List<Item>, val totalCount: Int, val offset: Int, val limit: Int) {
+    val hasPrevious: Boolean get() = offset > 0
+    val hasNext: Boolean get() = offset + items.size < totalCount
+}
+
 /** UI reads committed local rows; local creates stay at version zero until synchronized. */
 class ItemStore(
     private val dao: ItemDao,
@@ -22,7 +29,32 @@ class ItemStore(
     var schedulingError: String? = null
         private set
 
-    suspend fun items(): List<Item> = dao.readAll().map(ItemEntity::toDomain)
+    suspend fun items(): List<Item> {
+        val started = System.nanoTime()
+        val entities = dao.readAll()
+        // Observe actual returned Room entities before mapping. Android logcat supplies
+        // the real PID; every startup call is counted, without changing the query or rows.
+        println("M15_ITEM_READ " + canonicalJson(mapOf(
+            "sql" to ITEM_READ_ALL_SQL, "limit" to null, "returnedRows" to entities.size,
+            "startedNanos" to started, "finishedNanos" to System.nanoTime(),
+            "boundary" to "Room entities before domain mapping",
+        )))
+        return entities.map(ItemEntity::toDomain)
+    }
+
+    suspend fun page(offset: Int = 0, limit: Int = ITEM_PAGE_SIZE): ItemPage {
+        val started = System.nanoTime()
+        val page = dao.readPage(limit, offset)
+        println("M15_ITEM_COUNT " + canonicalJson(mapOf(
+            "sql" to ITEM_COUNT_SQL, "scalarResult" to page.totalCount, "itemRowsMaterialized" to 0,
+        )))
+        println("M15_ITEM_READ " + canonicalJson(mapOf(
+            "sql" to ITEM_READ_PAGE_SQL, "limit" to limit, "offset" to page.offset,
+            "returnedRows" to page.items.size, "startedNanos" to started,
+            "finishedNanos" to System.nanoTime(), "boundary" to "Room entities before domain mapping",
+        )))
+        return ItemPage(page.items.map(ItemEntity::toDomain), page.totalCount, page.offset, limit)
+    }
 
     suspend fun lastSuccessfulRefreshAt(): Long? = dao.lastSuccessfulRefreshAt()
 
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
index c55ccf6..5a948ac 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
@@ -4,13 +4,18 @@ import android.os.Bundle
 import androidx.activity.ComponentActivity
 import androidx.activity.compose.setContent
 import androidx.compose.foundation.layout.Arrangement
+import androidx.compose.foundation.layout.BoxWithConstraints
 import androidx.compose.foundation.layout.Column
 import androidx.compose.foundation.layout.Row
 import androidx.compose.foundation.layout.fillMaxSize
 import androidx.compose.foundation.layout.fillMaxWidth
+import androidx.compose.foundation.layout.heightIn
 import androidx.compose.foundation.layout.padding
 import androidx.compose.foundation.rememberScrollState
 import androidx.compose.foundation.verticalScroll
+import androidx.compose.foundation.lazy.LazyColumn
+import androidx.compose.foundation.lazy.items
+import androidx.compose.foundation.lazy.rememberLazyListState
 import androidx.compose.material3.Button
 import androidx.compose.material3.Checkbox
 import androidx.compose.material3.MaterialTheme
@@ -19,6 +24,7 @@ import androidx.compose.material3.Text
 import androidx.compose.material3.TextButton
 import androidx.compose.runtime.Composable
 import androidx.compose.runtime.collectAsState
+import androidx.compose.runtime.DisposableEffect
 import androidx.compose.runtime.LaunchedEffect
 import androidx.compose.runtime.getValue
 import androidx.compose.runtime.mutableStateOf
@@ -60,6 +66,8 @@ class MainActivity : ComponentActivity() {
 @Composable
 internal fun ItemScreen(store: ItemStore, sync: ItemSync? = null) {
     var items by remember(store) { mutableStateOf<List<Item>?>(null) }
+    var totalCount by remember(store) { mutableStateOf(0) }
+    var pageOffset by remember(store) { mutableStateOf(0) }
     var pendingCount by remember(store) { mutableStateOf(0) }
     var conflictCount by remember(store) { mutableStateOf(0) }
     var busy by remember(store) { mutableStateOf(true) }
@@ -76,9 +84,11 @@ internal fun ItemScreen(store: ItemStore, sync: ItemSync? = null) {
     var editor by rememberSaveable(stateSaver = ItemEditor.Saver) { mutableStateOf(ItemEditor()) }
     val scope = rememberCoroutineScope()
     val focus = LocalFocusManager.current
+    val listState = rememberLazyListState()
     val rows = items.orEmpty().toRows()
 
-    suspend fun accessStorage(action: suspend () -> Unit = {}, after: () -> Unit = {}) {
+    suspend fun accessStorage(action: suspend () -> Unit = {}, after: () -> Unit = {},
+                              requestedOffset: Int = pageOffset) {
         try {
             try {
                 action()
@@ -87,7 +97,11 @@ internal fun ItemScreen(store: ItemStore, sync: ItemSync? = null) {
                 val pending = store.pendingMutations()
                 pendingCount = pending.size
                 conflictCount = pending.count { it.terminalError == "version_conflict" }
-                items = store.items()
+                val page = store.page(requestedOffset)
+                if (pageOffset != page.offset) listState.scrollToItem(0)
+                items = page.items
+                totalCount = page.totalCount
+                pageOffset = page.offset
             }
             sync?.readSavedStatus()
             storageError = null
@@ -104,13 +118,14 @@ internal fun ItemScreen(store: ItemStore, sync: ItemSync? = null) {
         }
     }
 
-    fun change(action: suspend () -> Unit, after: () -> Unit = {}, isSync: Boolean = false) {
+    fun change(action: suspend () -> Unit, after: () -> Unit = {}, isSync: Boolean = false,
+               requestedOffset: Int = pageOffset) {
         if (busy) return
         // Release focus before busy disables and removes the controls' focus targets.
         focus.clearFocus()
         busy = true
         syncing = isSync
-        scope.launch { accessStorage(action, after) }
+        scope.launch { accessStorage(action, after, requestedOffset) }
     }
 
     LaunchedEffect(store, sync) {
@@ -121,120 +136,149 @@ internal fun ItemScreen(store: ItemStore, sync: ItemSync? = null) {
         }
     }
 
-    Column(
-        modifier = Modifier.fillMaxSize().verticalScroll(rememberScrollState()).padding(16.dp),
-        verticalArrangement = Arrangement.spacedBy(8.dp),
-    ) {
-        Text("Offline Item Tracker", style = MaterialTheme.typography.headlineSmall)
-        OutlinedTextField(
-            value = editor.title,
-            onValueChange = { editor = editor.copy(title = it) },
-            label = { Text("Item title") },
-            singleLine = true,
-            enabled = !busy && items != null,
-            modifier = Modifier.fillMaxWidth().testTag("item-title-input"),
-        )
-        Row {
-            Button(
-                enabled = !busy && items != null && editor.title.isNotBlank(),
-                onClick = {
-                    val submittedEditor = editor
-                    change(
-                        action = {
-                            submittedEditor.submit(store)
-                            sync?.markLocalChange()
-                        },
-                        after = { editor = ItemEditor() },
-                    )
-                },
-            ) { Text(if (editor.itemId == null) "Add" else "Save") }
-            if (editor.itemId != null) {
-                TextButton(enabled = !busy, onClick = {
-                    editor = ItemEditor()
-                    focus.clearFocus()
-                }) { Text("Cancel") }
-            }
-            if (sync != null) {
-                TextButton(
+    BoxWithConstraints(Modifier.fillMaxSize().padding(16.dp)) {
+        val headerMaxHeight = maxHeight * 2 / 3
+        Column(
+            modifier = Modifier.fillMaxSize(),
+            verticalArrangement = Arrangement.spacedBy(8.dp),
+        ) {
+            // Keep rows reachable even when the editor/status needs to scroll in a short window.
+            Column(Modifier.fillMaxWidth().heightIn(max = headerMaxHeight).verticalScroll(rememberScrollState()),
+                verticalArrangement = Arrangement.spacedBy(8.dp)) {
+                Text("Offline Item Tracker", style = MaterialTheme.typography.headlineSmall)
+                OutlinedTextField(
+                    value = editor.title,
+                    onValueChange = { editor = editor.copy(title = it) },
+                    label = { Text("Item title") },
+                    singleLine = true,
                     enabled = !busy && items != null,
-                    onClick = { change(action = { sync.synchronize() }, isSync = true) },
-                ) { Text("Sync") }
-            }
-        }
-        if (busy && !syncing) Text(when {
-            items == null -> "Loading local items…"
-            else -> "Saving locally…"
-        })
-        syncStatus?.let { status ->
-            Text(when (status.state) {
-                SyncState.STALE -> "Stale local data"
-                SyncState.REFRESHING -> "Stale local data · refreshing…"
-                SyncState.FRESH -> "Fresh local data"
-                SyncState.ERROR -> "Stale local data · sync error"
-            }, modifier = Modifier.testTag("sync-status"))
-            Text("Last successful refresh: ${status.lastSuccessfulRefreshAt ?: "never"}",
-                modifier = Modifier.testTag("last-successful-refresh"))
-            status.error?.let {
-                Text("Refresh failed: $it. Saved items retained; remote outcome unconfirmed.",
-                    modifier = Modifier.testTag("sync-error"))
-            }
-        }
-        storageError?.let { message ->
-            Text(message, modifier = Modifier.testTag("storage-error"))
-            TextButton(enabled = !busy, onClick = { change(action = {}) }) { Text("Reload") }
-        }
-        if (items != null) {
-            Text("Pending changes: $pendingCount", modifier = Modifier.testTag("pending-count"))
-            if (conflictCount > 0) {
-                Text("CONFLICT: remote value kept; local attempt retained.", modifier = Modifier.testTag("conflict-notice"))
-            }
-            Text("Items (${rows.size})", modifier = Modifier.testTag("item-count"))
-            if (rows.isEmpty()) Text("No items")
-        }
-        rows.forEach { row ->
-            Column(Modifier.fillMaxWidth().testTag(row.tag)) {
-                Text(row.title, style = MaterialTheme.typography.titleMedium)
-                Row(verticalAlignment = Alignment.CenterVertically) {
-                    Checkbox(
-                        checked = row.completed,
-                        enabled = !busy,
-                        onCheckedChange = {
-                            change(action = {
-                                store.setCompleted(row.id, it)
-                                sync?.markLocalChange()
-                            })
-                        },
-                        modifier = Modifier.semantics {
-                            contentDescription = "Completed: ${row.title}"
-                        },
-                    )
-                    Text(if (row.completed) "Completed" else "Incomplete")
-                    TextButton(
-                        enabled = !busy,
-                        onClick = {
-                            editor = ItemEditor(row.id, row.title)
-                        },
-                        modifier = Modifier.semantics {
-                            contentDescription = "Edit ${row.title}"
-                        },
-                    ) { Text("Edit") }
-                    TextButton(
-                        enabled = !busy,
+                    modifier = Modifier.fillMaxWidth().testTag("item-title-input"),
+                )
+                Row {
+                    Button(
+                        enabled = !busy && items != null && editor.title.isNotBlank(),
                         onClick = {
+                            val submittedEditor = editor
                             change(
                                 action = {
-                                    store.delete(row.id)
+                                    submittedEditor.submit(store)
                                     sync?.markLocalChange()
                                 },
-                                after = {
-                                    if (editor.itemId == row.id) editor = ItemEditor()
-                                },
+                                after = { editor = ItemEditor() },
                             )
                         },
-                        modifier = Modifier.semantics {
-                            contentDescription = "Delete ${row.title}"
-                        },
-                    ) { Text("Delete") }
+                    ) { Text(if (editor.itemId == null) "Add" else "Save") }
+                    if (editor.itemId != null) {
+                        TextButton(enabled = !busy, onClick = {
+                            editor = ItemEditor()
+                            focus.clearFocus()
+                        }) { Text("Cancel") }
+                    }
+                    if (sync != null) {
+                        TextButton(
+                            enabled = !busy && items != null,
+                            onClick = { change(action = { sync.synchronize() }, isSync = true) },
+                        ) { Text("Sync") }
+                    }
+                }
+                if (busy && !syncing) Text(when {
+                    items == null -> "Loading local items…"
+                    else -> "Saving locally…"
+                })
+                syncStatus?.let { status ->
+                    Text(when (status.state) {
+                        SyncState.STALE -> "Stale local data"
+                        SyncState.REFRESHING -> "Stale local data · refreshing…"
+                        SyncState.FRESH -> "Fresh local data"
+                        SyncState.ERROR -> "Stale local data · sync error"
+                    }, modifier = Modifier.testTag("sync-status"))
+                    Text("Last successful refresh: ${status.lastSuccessfulRefreshAt ?: "never"}",
+                        modifier = Modifier.testTag("last-successful-refresh"))
+                    status.error?.let {
+                        Text("Refresh failed: $it. Saved items retained; remote outcome unconfirmed.",
+                            modifier = Modifier.testTag("sync-error"))
+                    }
+                }
+                storageError?.let { message ->
+                    Text(message, modifier = Modifier.testTag("storage-error"))
+                    TextButton(enabled = !busy, onClick = { change(action = {}) }) { Text("Reload") }
+                }
+                if (items != null) {
+                    Text("Pending changes: $pendingCount", modifier = Modifier.testTag("pending-count"))
+                    if (conflictCount > 0) {
+                        Text("CONFLICT: remote value kept; local attempt retained.", modifier = Modifier.testTag("conflict-notice"))
+                    }
+                    Text("Items ($totalCount)", modifier = Modifier.testTag("item-count"))
+                    if (rows.isEmpty()) Text("No items")
+                    if (totalCount > ITEM_PAGE_SIZE) {
+                        Row(Modifier.fillMaxWidth(), horizontalArrangement = Arrangement.SpaceBetween,
+                            verticalAlignment = Alignment.CenterVertically) {
+                            TextButton(enabled = !busy && pageOffset > 0, onClick = {
+                                change(action = {}, requestedOffset = (pageOffset - ITEM_PAGE_SIZE).coerceAtLeast(0))
+                            }) { Text("Previous") }
+                            Text("Page ${pageOffset / ITEM_PAGE_SIZE + 1} / ${maxOf(1, (totalCount + ITEM_PAGE_SIZE - 1) / ITEM_PAGE_SIZE)}",
+                                modifier = Modifier.testTag("item-page"))
+                            TextButton(enabled = !busy && pageOffset + rows.size < totalCount, onClick = {
+                                change(action = {}, requestedOffset = pageOffset + ITEM_PAGE_SIZE)
+                            }) { Text("Next") }
+                        }
+                    }
+                }
+            }
+            LazyColumn(Modifier.fillMaxWidth().weight(1f).testTag("item-list"), state = listState,
+                verticalArrangement = Arrangement.spacedBy(8.dp)) {
+                items(rows, key = { it.id }) { row ->
+                    DisposableEffect(row.id) {
+                        println("M15_ROW_RENDER " + canonicalJson(mapOf("event" to "enter", "id" to row.id)))
+                        onDispose {
+                            println("M15_ROW_RENDER " + canonicalJson(mapOf("event" to "leave", "id" to row.id)))
+                        }
+                    }
+                    Column(Modifier.fillMaxWidth().testTag(row.tag)) {
+                        Text(row.title, style = MaterialTheme.typography.titleMedium)
+                        Row(verticalAlignment = Alignment.CenterVertically) {
+                            Checkbox(
+                                checked = row.completed,
+                                enabled = !busy,
+                                onCheckedChange = {
+                                    change(action = {
+                                        store.setCompleted(row.id, it)
+                                        sync?.markLocalChange()
+                                    })
+                                },
+                                modifier = Modifier.semantics {
+                                    contentDescription = "Completed: ${row.title}"
+                                },
+                            )
+                            Text(if (row.completed) "Completed" else "Incomplete")
+                            TextButton(
+                                enabled = !busy,
+                                onClick = {
+                                    editor = ItemEditor(row.id, row.title)
+                                },
+                                modifier = Modifier.semantics {
+                                    contentDescription = "Edit ${row.title}"
+                                },
+                            ) { Text("Edit") }
+                            TextButton(
+                                enabled = !busy,
+                                onClick = {
+                                    change(
+                                        action = {
+                                            store.delete(row.id)
+                                            sync?.markLocalChange()
+                                        },
+                                        after = {
+                                            if (editor.itemId == row.id) editor = ItemEditor()
+                                        },
+                                    )
+                                },
+                                modifier = Modifier.semantics {
+                                    contentDescription = "Delete ${row.title}"
+                                },
+                            ) { Text("Delete") }
+                        }
+                    }
                 }
             }
         }
diff --git a/app/src/main/res/xml/network_security_config.xml b/app/src/main/res/xml/network_security_config.xml
index 70d48a3..7702390 100644
--- a/app/src/main/res/xml/network_security_config.xml
+++ b/app/src/main/res/xml/network_security_config.xml
@@ -2,6 +2,6 @@
 <network-security-config>
     <base-config cleartextTrafficPermitted="false" />
     <domain-config cleartextTrafficPermitted="true">
-        <domain>10.0.2.2</domain>
+        <domain includeSubdomains="false">10.0.2.2</domain>
     </domain-config>
 </network-security-config>
diff --git a/app/src/releaseAudit/assets/M15-inputs.json b/app/src/releaseAudit/assets/M15-inputs.json
new file mode 100644
index 0000000..a925495
--- /dev/null
+++ b/app/src/releaseAudit/assets/M15-inputs.json
@@ -0,0 +1,41 @@
+{
+  "thread": "M15",
+  "profile": "phase-1",
+  "specRevision": "61280dd86ce88b6e431f408241c0998a275960aa",
+  "start": "a9ca3a773fce81c5130adeccfa95cf0ffb439ae9",
+  "api": 34,
+  "schemaVersion": 6,
+  "dataset": {
+    "count": 2000,
+    "idPrefix": "large-",
+    "titlePrefix": "Item ",
+    "decimalDigits": 4,
+    "completed": false,
+    "version": 1,
+    "updatedAt": 1700000000000
+  },
+  "probeCount": 2,
+  "initialCumulativeItemRowsMaximum": 50,
+  "observationPrefix": "M15_ITEM_READ ",
+  "unchangedSql": "SELECT * FROM items ORDER BY rowid",
+  "exportDirectory": "m15-release-01",
+  "finalOfflineCrud": {
+    "existingId": "large-2000",
+    "renamedTitle": "Item 2000 edited",
+    "completed": true,
+    "createdTitle": "Large local",
+    "createIdentityAndClock": "ordinary production UUID and current clock",
+    "deleteCreatedItem": true,
+    "finalItemCount": 2000,
+    "finalPendingCount": 4
+  },
+  "harness": {
+    "adbTimeoutSeconds": 45,
+    "preSeedTeardownWaitSeconds": 30,
+    "networkWaitSeconds": 30,
+    "instrumentationTimeoutSeconds": 180,
+    "initialUiWaitSeconds": 120,
+    "pollSeconds": 0.2,
+    "timingMeaning": "one observed ordinary cold-launch-to-first-usable-list time; functional timeout is not an SLA"
+  }
+}
diff --git a/app/src/releaseAudit/java/com/mobilesystemsevolution/kotlin/M15StorageTest.kt b/app/src/releaseAudit/java/com/mobilesystemsevolution/kotlin/M15StorageTest.kt
new file mode 100644
index 0000000..2e8218a
--- /dev/null
+++ b/app/src/releaseAudit/java/com/mobilesystemsevolution/kotlin/M15StorageTest.kt
@@ -0,0 +1,351 @@
+package com.mobilesystemsevolution.kotlin
+
+import android.app.Activity
+import android.app.Application
+import android.content.pm.ApplicationInfo
+import android.content.pm.PackageManager
+import android.database.sqlite.SQLiteDatabase
+import android.os.Build
+import android.os.Bundle
+import android.os.Process
+import android.os.SystemClock
+import androidx.room.withTransaction
+import androidx.test.ext.junit.runners.AndroidJUnit4
+import androidx.test.platform.app.InstrumentationRegistry
+import androidx.test.runner.lifecycle.ActivityLifecycleMonitorRegistry
+import androidx.test.runner.lifecycle.Stage
+import java.io.File
+import java.security.MessageDigest
+import java.util.Locale
+import java.util.UUID
+import java.util.concurrent.atomic.AtomicInteger
+import kotlinx.coroutines.runBlocking
+import org.json.JSONObject
+import org.junit.After
+import org.junit.Assert.assertEquals
+import org.junit.Assert.assertFalse
+import org.junit.Assert.assertTrue
+import org.junit.Before
+import org.junit.Test
+import org.junit.runner.RunWith
+
+/** Same-signature target instrumentation only: no Activity rule or production seed hook. */
+@RunWith(AndroidJUnit4::class)
+class M15StorageTest {
+    private val instrumentation = InstrumentationRegistry.getInstrumentation()
+    private val context = instrumentation.targetContext
+    private val application = context.applicationContext as Application
+    private val inputs = JSONObject(instrumentation.context.assets.open("M15-inputs.json").bufferedReader().use { it.readText() })
+    private val createdActivities = AtomicInteger()
+    private var observing = false
+    private val observer = object : Application.ActivityLifecycleCallbacks {
+        override fun onActivityCreated(activity: Activity, state: Bundle?) { createdActivities.incrementAndGet() }
+        override fun onActivityStarted(activity: Activity) = Unit
+        override fun onActivityResumed(activity: Activity) = Unit
+        override fun onActivityPaused(activity: Activity) = Unit
+        override fun onActivityStopped(activity: Activity) = Unit
+        override fun onActivitySaveInstanceState(activity: Activity, state: Bundle) = Unit
+        override fun onActivityDestroyed(activity: Activity) = Unit
+    }
+
+    @Before
+    fun verifyReleaseAndObserveNoActivity() {
+        assertEquals(inputs.getInt("api"), Build.VERSION.SDK_INT)
+        assertEquals(ItemApplication::class.java.name, application.javaClass.name)
+        assertFalse(context.applicationInfo.flags and ApplicationInfo.FLAG_DEBUGGABLE != 0)
+        assertFalse(context.packageManager.getApplicationInfo(context.packageName, 0).flags and ApplicationInfo.FLAG_DEBUGGABLE != 0)
+        assertEquals(signers(context.packageName), signers(instrumentation.context.packageName))
+        assertFalse(File(context.filesDir, "m10-clock.txt").exists())
+        application.registerActivityLifecycleCallbacks(observer)
+        observing = true
+        assertNoActivity()
+    }
+
+    @After
+    fun removeObserverAndVerifyNoActivity() {
+        try {
+            assertNoActivity()
+            assertEquals(0, createdActivities.get())
+        } finally {
+            if (observing) application.unregisterActivityLifecycleCallbacks(observer)
+        }
+    }
+
+    @Test
+    fun probeReleaseSeedAndAuditWithoutActivity(): Unit = runBlocking {
+        val count = inputs.getInt("probeCount")
+        assertEquals(2, count)
+        seed(count, observeSmallRead = true)
+        export("probe-seeded", count)
+        val database = ItemDatabase.open(context)
+        try {
+            database.withTransaction { database.items().deleteAll() }
+        } finally {
+            database.close()
+        }
+        export("probe-empty", 0)
+    }
+
+    @Test
+    fun seedFixedCanonicalDatasetWithoutActivity(): Unit = runBlocking {
+        val count = inputs.getJSONObject("dataset").getInt("count")
+        assertEquals(2000, count)
+        seed(count, observeSmallRead = false)
+        export("seeded", count)
+    }
+
+    @Test
+    fun auditCanonicalDatasetWithoutActivity() {
+        // This new target-instrumentation process is outside the ordinary launch window.
+        // It never constructs MainActivity, ItemScreen, or a production ItemStore.
+        export("audited", inputs.getJSONObject("dataset").getInt("count"))
+    }
+
+    @Test
+    fun auditOfflineCrudDatasetWithoutActivity() {
+        assertNoActivity()
+        val fixed = inputs.getJSONObject("finalOfflineCrud")
+        val source = context.getDatabasePath(ItemDatabase.NAME)
+        val state = SQLiteDatabase.openDatabase(source.path, null, SQLiteDatabase.OPEN_READONLY).use { database ->
+            assertEquals(inputs.getInt("schemaVersion"), database.version)
+            assertEquals(fixed.getInt("finalItemCount"), scalar(database, "SELECT COUNT(*) FROM items"))
+            assertEquals(1999, scalar(database, "SELECT COUNT(*) FROM items WHERE rowid<2000 " +
+                "AND completed=0 AND version=1 AND updatedAt=1700000000000 " +
+                "AND id='large-' || printf('%04d',rowid) AND title='Item ' || printf('%04d',rowid)"))
+            val last = checkNotNull(edge(database, "DESC"))
+            assertEquals(fixed.getString("existingId"), last["id"])
+            assertEquals(fixed.getString("renamedTitle"), last["title"])
+            assertEquals(true, last["completed"])
+            assertEquals(1L, last["version"])
+            assertTrue(last["updatedAt"] as Long > 1700000000000L)
+            val pending = readPending(database)
+            assertEquals(fixed.getInt("finalPendingCount"), pending.size)
+            assertEquals(listOf("RENAME", "COMPLETE", "CREATE", "DELETE"), pending.map { it.operation })
+            assertEquals(listOf(1L, 2L, 3L, 4L), pending.map { it.sequence })
+            assertEquals(listOf(1L, 1L, null, 0L), pending.map { it.baseVersion })
+            assertEquals(listOf(fixed.getString("existingId"), fixed.getString("existingId"),
+                pending[2].itemId, pending[2].itemId), pending.map { it.itemId })
+            assertEquals(fixed.getString("renamedTitle"), pending[0].title)
+            assertEquals(listOf(null, true, false, null), pending.map { it.completed })
+            assertEquals(fixed.getString("createdTitle"), pending[2].title)
+            assertEquals(null, pending[1].title)
+            assertEquals(null, pending[3].title)
+            assertEquals(4, pending.map { it.clientMutationId }.toSet().size)
+            assertEquals(pending[2].itemId, UUID.fromString(pending[2].itemId).toString())
+            pending.forEach {
+                assertEquals(it.clientMutationId, UUID.fromString(it.clientMutationId).toString())
+                assertEquals(null, it.terminalError)
+                assertFalse(it.dispatched)
+                assertEquals(it.request().hash(), it.payloadHash)
+            }
+            val counts = EMPTY_TABLES.associateWith { scalar(database, "SELECT COUNT(*) FROM $it") }
+            assertEquals(mapOf("pending_mutations" to 4, "acknowledged_mutations" to 0,
+                "tombstones" to 0, "automatic_sync" to 1, "sync_metadata" to 0), counts)
+            val cycle = database.rawQuery("SELECT id,workId,httpAttempts,state FROM automatic_sync", null).use {
+                check(it.moveToFirst())
+                assertEquals(1, it.getInt(0))
+                assertEquals(0, it.getInt(2))
+                assertEquals("ACTIVE", it.getString(3))
+                mapOf("id" to it.getInt(0), "workId" to it.getString(1),
+                    "httpAttempts" to it.getInt(2), "state" to it.getString(3))
+            }
+            mapOf("schemaVersion" to database.version, "itemCount" to fixed.getInt("finalItemCount"),
+                "first" to edge(database, "ASC"), "last" to last, "tableCounts" to counts,
+                "automaticCycle" to cycle, "pending" to pending.map { mapOf(
+                    "sequence" to it.sequence, "itemId" to it.itemId, "operation" to it.operation,
+                    "title" to it.title, "completed" to it.completed, "clientMutationId" to it.clientMutationId,
+                    "payloadHash" to it.payloadHash, "terminalError" to it.terminalError,
+                    "baseVersion" to it.baseVersion, "dispatched" to it.dispatched,
+                ) })
+        }
+        exportState("after-crud", state)
+    }
+
+    @Test
+    fun smallNativePagesPreserveOrderClampAndAtomicCrud(): Unit = runBlocking {
+        val name = "m15-small-pages.db"
+        context.deleteDatabase(name)
+        val database = ItemDatabase.open(context, name)
+        try {
+            database.withTransaction {
+                for (index in 1..31) database.items().insert(
+                    ItemEntity("small-$index", "Item $index", false, 1, 1700000000000L))
+            }
+            var mutationIndex = 0
+            val store = ItemStore(database.items(), nextId = { "small-local" }, now = { 1700000001000L },
+                nextMutationId = { "small-mutation-${++mutationIndex}" })
+            val first = store.page()
+            val refreshed = store.page()
+            assertEquals((1..25).map { "small-$it" }, first.items.map(Item::id))
+            assertEquals(first, refreshed)
+            assertEquals(50, first.items.size + refreshed.items.size)
+            assertEquals(31, first.totalCount)
+            assertEquals((26..31).map { "small-$it" }, store.page(25).items.map(Item::id))
+            store.rename("small-31", "Edited")
+            store.setCompleted("small-31", true)
+            store.create("Local")
+            assertEquals("small-local", store.page(25).items.last().id)
+            store.delete("small-local")
+            assertEquals(Item("small-31", "Edited", true, 1, 1700000001000L), store.page(25).items.last())
+            val pending = store.pendingMutations()
+            assertEquals(listOf("RENAME", "COMPLETE", "CREATE", "DELETE"), pending.map { it.operation })
+            assertEquals(listOf(1L, 1L, null, 0L), pending.map { it.baseVersion })
+            assertTrue(pending.all { !it.dispatched && it.request().hash() == it.payloadHash })
+            for (index in 26..31) store.delete("small-$index")
+            assertEquals(0, store.page(25).offset)
+            assertEquals(25, store.page().totalCount)
+            println("M15_NATIVE_PAGING " + canonicalJson(mapOf("status" to "PASS", "pid" to Process.myPid(),
+                "fixtureRows" to 31, "initialCumulativeRows" to 50, "fourIntentsVerified" to true)))
+        } finally {
+            database.close()
+            context.deleteDatabase(name)
+        }
+    }
+
+    private suspend fun seed(count: Int, observeSmallRead: Boolean) {
+        val database = ItemDatabase.open(context)
+        try {
+            // Opening the real Room schema initializes/migrates it normally; no raw substitute DB.
+            val sql = database.openHelper.writableDatabase
+            assertEquals(inputs.getInt("schemaVersion"), sql.version)
+            sql.query("SELECT COUNT(*) FROM items").use { rows ->
+                assertTrue(rows.moveToFirst())
+                assertEquals(0, rows.getInt(0))
+            }
+            for (table in EMPTY_TABLES) {
+                sql.query("SELECT COUNT(*) FROM $table").use { rows ->
+                    assertTrue(rows.moveToFirst())
+                    assertEquals(0, rows.getInt(0))
+                }
+            }
+            val fixed = inputs.getJSONObject("dataset")
+            database.withTransaction {
+                // Generate one record at a time inside the setup transaction, never an
+                // in-memory 2000-Item array handed to production state after startup.
+                for (index in 1..count) {
+                    val digits = String.format(Locale.ROOT, "%0${fixed.getInt("decimalDigits")}d", index)
+                    database.items().insert(ItemEntity(
+                        fixed.getString("idPrefix") + digits, fixed.getString("titlePrefix") + digits,
+                        fixed.getBoolean("completed"), fixed.getLong("version"), fixed.getLong("updatedAt"),
+                    ))
+                }
+            }
+            if (observeSmallRead) {
+                // Only the two-row helper smoke calls the observed production read. The
+                // full seed never calls it and never participates in cold-start timing.
+                val rows = ItemStore(database.items()).items()
+                assertEquals(2, rows.size)
+                assertEquals("large-0001", rows.first().id)
+                assertEquals("large-0002", rows.last().id)
+            }
+        } finally {
+            database.close()
+        }
+    }
+
+    private fun export(stage: String, count: Int) {
+        assertNoActivity()
+        assertEquals(0, createdActivities.get())
+        val source = context.getDatabasePath(ItemDatabase.NAME)
+        val state = SQLiteDatabase.openDatabase(source.path, null, SQLiteDatabase.OPEN_READONLY).use { database ->
+            assertEquals(inputs.getInt("schemaVersion"), database.version)
+            database.rawQuery("PRAGMA integrity_check", null).use { rows ->
+                assertTrue(rows.moveToFirst())
+                assertEquals("ok", rows.getString(0))
+            }
+            assertEquals(count, scalar(database, "SELECT COUNT(*) FROM items"))
+            val empty = EMPTY_TABLES.associateWith { table ->
+                scalar(database, "SELECT COUNT(*) FROM $table").also { assertEquals(0, it) }
+            }
+            val fixed = inputs.getJSONObject("dataset")
+            // Scalar validation only. Root independently opens every exported native row.
+            assertEquals(count, scalar(database, "SELECT COUNT(*) FROM items WHERE " +
+                "completed=0 AND version=1 AND updatedAt=${fixed.getLong("updatedAt")} " +
+                "AND id='large-' || printf('%04d',rowid) AND title='Item ' || printf('%04d',rowid)"))
+            mapOf("schemaVersion" to database.version, "itemCount" to count,
+                "first" to edge(database, "ASC"), "last" to edge(database, "DESC"), "emptyTables" to empty)
+        }
+        exportState(stage, state)
+    }
+
+    private fun exportState(stage: String, state: Map<String, Any?>) {
+        assertNoActivity()
+        assertEquals(0, createdActivities.get())
+        val source = context.getDatabasePath(ItemDatabase.NAME)
+        // All Room/SQLite handles owned by this test are closed before copying DB/WAL.
+        val parent = File(checkNotNull(context.getExternalFilesDir(null)), inputs.getString("exportDirectory"))
+        val output = File(parent, stage)
+        check(!output.exists()) { "Refusing to overwrite $output" }
+        check(output.mkdirs())
+        val files = mutableListOf<Map<String, Any>>()
+        for (suffix in listOf("", "-wal", "-shm")) {
+            val from = File(source.path + suffix)
+            if (from.exists()) {
+                val copy = File(output, from.name)
+                from.copyTo(copy, overwrite = false)
+                files += mapOf("name" to copy.name, "bytes" to copy.length(), "sha256" to digest(copy.readBytes()))
+            }
+        }
+        val result = mapOf(
+            "status" to "PASS", "stage" to stage, "pid" to Process.myPid(), "uid" to Process.myUid(),
+            "elapsedRealtimeNanos" to SystemClock.elapsedRealtimeNanos(), "application" to application.javaClass.name,
+            "targetPackage" to context.packageName, "testPackage" to instrumentation.context.packageName,
+            "targetSourceDir" to context.applicationInfo.sourceDir,
+            "targetDebuggable" to (context.applicationInfo.flags and ApplicationInfo.FLAG_DEBUGGABLE != 0),
+            "targetSignerSha256" to signers(context.packageName),
+            "testSignerSha256" to signers(instrumentation.context.packageName),
+            "observedCreatedActivities" to createdActivities.get(), "activeActivities" to activeActivities(),
+            "setupKind" to "same-signature release-target instrumentation; Application/providers initialized; no Activity launch",
+            "roomAndAuditHandlesClosedBeforeCopy" to true, "database" to state, "files" to files,
+        )
+        File(output, "result.json").writeText(canonicalJson(result) + "\n")
+        println("M15_STORAGE_AUDIT " + canonicalJson(mapOf("stage" to stage, "pid" to Process.myPid(), "output" to output.path)))
+    }
+
+    private fun readPending(database: SQLiteDatabase): List<PendingMutation> = database.rawQuery(
+        "SELECT sequence,itemId,operation,title,completed,clientMutationId,payloadHash,terminalError,baseVersion,dispatched " +
+            "FROM pending_mutations ORDER BY sequence", null).use { rows ->
+        buildList {
+            while (rows.moveToNext()) add(PendingMutation(rows.getLong(0), rows.getString(1), rows.getString(2),
+                if (rows.isNull(3)) null else rows.getString(3), if (rows.isNull(4)) null else rows.getInt(4) != 0,
+                rows.getString(5), rows.getString(6), if (rows.isNull(7)) null else rows.getString(7),
+                if (rows.isNull(8)) null else rows.getLong(8), rows.getInt(9) != 0))
+        }
+    }
+
+    private fun scalar(database: SQLiteDatabase, sql: String): Int = database.rawQuery(sql, null).use {
+        check(it.moveToFirst())
+        it.getInt(0)
+    }
+
+    private fun edge(database: SQLiteDatabase, order: String): Map<String, Any>? =
+        database.rawQuery("SELECT id,title,completed,version,updatedAt FROM items ORDER BY rowid $order LIMIT 1", null).use {
+            if (!it.moveToFirst()) null else mapOf("id" to it.getString(0), "title" to it.getString(1),
+                "completed" to (it.getInt(2) != 0), "version" to it.getLong(3), "updatedAt" to it.getLong(4))
+        }
+
+    private fun signers(packageName: String): List<String> = checkNotNull(context.packageManager
+        .getPackageInfo(packageName, PackageManager.GET_SIGNING_CERTIFICATES).signingInfo)
+        .apkContentsSigners.map { digest(it.toByteArray()) }.sorted().also { assertTrue(it.isNotEmpty()) }
+
+    private fun activeActivities(): List<String> {
+        var active = emptyList<String>()
+        instrumentation.runOnMainSync {
+            val monitor = ActivityLifecycleMonitorRegistry.getInstance()
+            active = Stage.values().flatMap { stage ->
+                monitor.getActivitiesInStage(stage).map { stage.name + ":" + it.javaClass.name }
+            }
+        }
+        return active
+    }
+
+    private fun assertNoActivity() { assertEquals(emptyList<String>(), activeActivities()) }
+
+    private fun digest(bytes: ByteArray): String = MessageDigest.getInstance("SHA-256").digest(bytes)
+        .joinToString("") { "%02x".format(it) }
+
+    companion object {
+        private val EMPTY_TABLES = listOf("pending_mutations", "acknowledged_mutations", "tombstones",
+            "automatic_sync", "sync_metadata")
+    }
+}
diff --git a/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt b/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
index cbf8acc..fd880f1 100644
--- a/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
+++ b/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
@@ -928,6 +928,97 @@ class ItemStoreTest {
         assertEquals(remote.rows.values.sortedBy(Item::id), store.items())
     }
 
+    @Test
+    fun m15SmallReadObservationCountsReturnedEntitiesWithoutChangingRows(): Unit = runBlocking {
+        val dao = FakeItemDao()
+        val rows = listOf(ItemEntity("small-1", "First", false, 1, 1700000000000L),
+            ItemEntity("small-2", "Second", true, 2, 1700000001000L))
+        rows.forEach { dao.insert(it) }
+        val captured = java.io.ByteArrayOutputStream()
+        val previous = System.out
+        try {
+            System.setOut(java.io.PrintStream(captured, true, "UTF-8"))
+            val store = ItemStore(dao)
+            assertEquals(rows.map(ItemEntity::toDomain), store.items())
+            assertEquals(rows.map(ItemEntity::toDomain), store.items())
+        } finally {
+            System.setOut(previous)
+        }
+        val events = captured.toString("UTF-8").lineSequence().filter { it.isNotEmpty() }.toList()
+        assertEquals(2, events.size)
+        events.forEach {
+            assertTrue(it.startsWith("M15_ITEM_READ {"))
+            assertTrue(it.contains("\"sql\":\"SELECT * FROM items ORDER BY rowid\""))
+            assertTrue(it.contains("\"limit\":null"))
+            assertTrue(it.contains("\"returnedRows\":2"))
+            assertTrue(it.contains("\"boundary\":\"Room entities before domain mapping\""))
+        }
+    }
+
+    @Test
+    fun m15TwoStartupPageReadsMaterializeAtMostFiftySmallFixtureRows(): Unit = runBlocking {
+        val dao = FakeItemDao()
+        for (index in 1..31) dao.insert(ItemEntity("small-$index", "Item $index", false, 1, 1700000000000L))
+        val store = ItemStore(dao)
+        val first = store.page()
+        val refreshed = store.page()
+        assertEquals((1..25).map { "small-$it" }, first.items.map(Item::id))
+        assertEquals(first, refreshed)
+        assertEquals(31, first.totalCount)
+        assertEquals(50, first.items.size + refreshed.items.size)
+        assertEquals(listOf(25 to 0, 25 to 0), dao.pageReads)
+        assertEquals(0, dao.fullReads)
+        assertFalse(first.hasPrevious)
+        assertTrue(first.hasNext)
+        assertThrows(IllegalArgumentException::class.java) { runBlocking { store.page(limit = 51) } }
+        assertThrows(IllegalArgumentException::class.java) { runBlocking { store.page(offset = -1) } }
+    }
+
+    @Test
+    fun m15SmallLastPagePreservesCrudAndOriginalDurableEnvelopes(): Unit = runBlocking {
+        val dao = FakeItemDao()
+        for (index in 1..31) dao.insert(ItemEntity("small-$index", "Item $index", false, 1, 1700000000000L))
+        var identity = 0
+        val store = ItemStore(dao, nextId = { "small-local" }, now = { 1700000001000L },
+            nextMutationId = { "small-intent-${++identity}" })
+        val last = store.page(25)
+        assertEquals((26..31).map { "small-$it" }, last.items.map(Item::id))
+        assertTrue(last.hasPrevious)
+        assertFalse(last.hasNext)
+        store.rename("small-31", "Item 31 edited")
+        store.setCompleted("small-31", true)
+        store.create("Local")
+        assertEquals("small-local", store.page(25).items.last().id)
+        store.delete("small-local")
+        val edited = store.page(25)
+        assertEquals(31, edited.totalCount)
+        assertEquals(Item("small-31", "Item 31 edited", true, 1, 1700000001000L), edited.items.last())
+        val pending = store.pendingMutations()
+        assertEquals(listOf("RENAME", "COMPLETE", "CREATE", "DELETE"), pending.map { it.operation })
+        assertEquals(listOf(1L, 1L, null, 0L), pending.map { it.baseVersion })
+        assertEquals(4, pending.map { it.clientMutationId }.toSet().size)
+        assertTrue(pending.all { !it.dispatched && it.terminalError == null && it.request().hash() == it.payloadHash })
+        assertEquals(0, dao.fullReads)
+    }
+
+    @Test
+    fun m15PageClampsAfterLastRowDeletionAndEmptyStorage(): Unit = runBlocking {
+        val dao = FakeItemDao()
+        for (index in 1..26) dao.insert(ItemEntity("small-$index", "Item $index", false, 1, 1700000000000L))
+        val store = ItemStore(dao)
+        assertEquals(listOf("small-26"), store.page(25).items.map(Item::id))
+        store.delete("small-26")
+        val previous = store.page(25)
+        assertEquals(0, previous.offset)
+        assertEquals(25, previous.totalCount)
+        assertFalse(previous.hasNext)
+        dao.deleteAll()
+        val empty = store.page(25)
+        assertEquals(0, empty.offset)
+        assertEquals(0, empty.totalCount)
+        assertTrue(empty.items.isEmpty())
+    }
+
     private fun conflictRemote() = FakeRemote().apply {
         rows.clear()
         rows["conflict-001"] = Item("conflict-001", "Initial", false, 1, 1700000000000L)
@@ -1068,6 +1159,8 @@ class ItemStoreTest {
         var rejectAcknowledgment = false
         val acknowledgmentWrites = mutableListOf<String>()
         private var automaticCycle: AutomaticCycle? = null
+        var fullReads = 0
+        val pageReads = mutableListOf<Pair<Int, Int>>()
 
         override suspend fun readAutomaticCycle(): AutomaticCycle? = automaticCycle
 
@@ -1159,7 +1252,17 @@ class ItemStoreTest {
             lastRefresh = metadata.lastSuccessfulRefreshAt
         }
 
-        override suspend fun readAll(): List<ItemEntity> = rows.values.toList()
+        override suspend fun readAll(): List<ItemEntity> {
+            fullReads++
+            return rows.values.toList()
+        }
+
+        override suspend fun countItems(): Int = rows.size
+
+        override suspend fun readPageRows(limit: Int, offset: Int): List<ItemEntity> {
+            pageReads += limit to offset
+            return rows.values.drop(offset).take(limit)
+        }
 
         override suspend fun insert(item: ItemEntity) {
             check(item.id !in rows) { "Duplicate primary key" }
diff --git a/verification/M15-inputs.json b/verification/M15-inputs.json
new file mode 100644
index 0000000..a925495
--- /dev/null
+++ b/verification/M15-inputs.json
@@ -0,0 +1,41 @@
+{
+  "thread": "M15",
+  "profile": "phase-1",
+  "specRevision": "61280dd86ce88b6e431f408241c0998a275960aa",
+  "start": "a9ca3a773fce81c5130adeccfa95cf0ffb439ae9",
+  "api": 34,
+  "schemaVersion": 6,
+  "dataset": {
+    "count": 2000,
+    "idPrefix": "large-",
+    "titlePrefix": "Item ",
+    "decimalDigits": 4,
+    "completed": false,
+    "version": 1,
+    "updatedAt": 1700000000000
+  },
+  "probeCount": 2,
+  "initialCumulativeItemRowsMaximum": 50,
+  "observationPrefix": "M15_ITEM_READ ",
+  "unchangedSql": "SELECT * FROM items ORDER BY rowid",
+  "exportDirectory": "m15-release-01",
+  "finalOfflineCrud": {
+    "existingId": "large-2000",
+    "renamedTitle": "Item 2000 edited",
+    "completed": true,
+    "createdTitle": "Large local",
+    "createIdentityAndClock": "ordinary production UUID and current clock",
+    "deleteCreatedItem": true,
+    "finalItemCount": 2000,
+    "finalPendingCount": 4
+  },
+  "harness": {
+    "adbTimeoutSeconds": 45,
+    "preSeedTeardownWaitSeconds": 30,
+    "networkWaitSeconds": 30,
+    "instrumentationTimeoutSeconds": 180,
+    "initialUiWaitSeconds": 120,
+    "pollSeconds": 0.2,
+    "timingMeaning": "one observed ordinary cold-launch-to-first-usable-list time; functional timeout is not an SLA"
+  }
+}
diff --git a/verification/large_release.py b/verification/large_release.py
new file mode 100644
index 0000000..651ce66
--- /dev/null
+++ b/verification/large_release.py
@@ -0,0 +1,339 @@
+#!/usr/bin/env python3
+"""Root-only M15 release helper smoke and one ordinary offline cold-start baseline.
+
+The two-row smoke has no Activity launch. Full seeding and post-run read-only audit
+use target instrumentation outside the measurement window; they are never a
+replacement for the single normal MainActivity launch. No fixture is required.
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
+import tarfile
+import tempfile
+import time
+
+from activity_state import ActivityStateScenario
+from offline_queue_restart import OfflineQueueScenario
+from process_restart import DB_NAME, PACKAGE, SERIAL, Scenario
+
+INPUTS_PATH = Path(__file__).with_name('M15-inputs.json')
+INPUTS = json.loads(INPUTS_PATH.read_text())
+LIMITS = INPUTS['harness']
+RUNNER = f'{PACKAGE}.test/androidx.test.runner.AndroidJUnitRunner'
+TEST_CLASS = f'{PACKAGE}.M15StorageTest'
+EXTERNAL = f'/sdcard/Android/data/{PACKAGE}/files/{INPUTS["exportDirectory"]}'
+EMPTY_TABLES = ('pending_mutations', 'acknowledged_mutations', 'tombstones', 'automatic_sync', 'sync_metadata')
+
+
+def sha(data):
+    return hashlib.sha256(data).hexdigest()
+
+
+def item(index):
+    fixed = INPUTS['dataset']
+    digits = f'{index:0{fixed["decimalDigits"]}d}'
+    return dict(id=fixed['idPrefix'] + digits, title=fixed['titlePrefix'] + digits,
+                completed=int(fixed['completed']), version=fixed['version'], updatedAt=fixed['updatedAt'])
+
+
+def observations(raw, pid):
+    events = []
+    for line in raw.splitlines():
+        if INPUTS['observationPrefix'] not in line:
+            continue
+        match = re.fullmatch(r'\d\d-\d\d\s+\d\d:\d\d:\d\d\.\d+\s+(\d+)\s+(\d+)\s+'
+                             r'[VDIWEF]\s+System\.out\s*:\s*M15_ITEM_READ (\{.*\})', line)
+        assert match, ('Malformed materialization record', line)
+        assert int(match[1]) == pid, ('Unexpected process in isolated observation window', line)
+        event = json.loads(match[3])
+        assert set(event) == {'sql', 'limit', 'returnedRows', 'startedNanos', 'finishedNanos', 'boundary'}, event
+        assert event['sql'] == INPUTS['unchangedSql'] and event['limit'] is None, event
+        assert type(event['returnedRows']) is int and event['returnedRows'] >= 0, event
+        assert type(event['startedNanos']) is int and type(event['finishedNanos']) is int, event
+        assert 0 < event['startedNanos'] <= event['finishedNanos'], event
+        assert event['boundary'] == 'Room entities before domain mapping', event
+        events.append(dict(event, pid=pid, tid=int(match[2]), rawLine=line))
+    assert events, 'No actual returned-entity observation for the launched process'
+    return events
+
+
+def validate_native(folder, count):
+    """Read a disposable native DB/WAL copy; never checkpoint the raw exported files."""
+    report = json.loads((folder / 'result.json').read_text())
+    assert report['status'] == 'PASS' and report['targetPackage'] == PACKAGE
+    assert report['testPackage'] == PACKAGE + '.test'
+    assert report['application'] == PACKAGE + '.ItemApplication'
+    assert report['targetDebuggable'] is False
+    assert report['targetSignerSha256'] == report['testSignerSha256'] and len(report['targetSignerSha256']) == 1
+    assert report['observedCreatedActivities'] == 0 and report['activeActivities'] == []
+    assert report['roomAndAuditHandlesClosedBeforeCopy'] is True
+    assert type(report['pid']) is int and report['pid'] > 0
+    names = []
+    for record in report['files']:
+        assert record['name'] in (DB_NAME, DB_NAME + '-wal', DB_NAME + '-shm')
+        path = folder / record['name']
+        assert path.stat().st_size == record['bytes'] and sha(path.read_bytes()) == record['sha256']
+        names.append(record['name'])
+    assert DB_NAME in names and len(names) == len(set(names))
+    with tempfile.TemporaryDirectory(prefix='mse-m15-native-') as temporary:
+        local = Path(temporary)
+        for suffix in ('', '-wal'):
+            source = folder / (DB_NAME + suffix)
+            if source.exists():
+                shutil.copyfile(source, local / source.name)
+        with sqlite3.connect(local / DB_NAME) as database:
+            assert database.execute('PRAGMA integrity_check').fetchone() == ('ok',)
+            assert database.execute('PRAGMA user_version').fetchone() == (INPUTS['schemaVersion'],)
+            columns = [row[1] for row in database.execute('PRAGMA table_info(items)')]
+            assert columns == ['id', 'title', 'completed', 'version', 'updatedAt']
+            rows = [dict(zip(columns, row)) for row in database.execute('SELECT * FROM items ORDER BY rowid')]
+            assert len(rows) == count
+            assert all(row == item(index) for index, row in enumerate(rows, 1)), 'Canonical dataset changed'
+            empty = {table: database.execute(f'SELECT COUNT(*) FROM {table}').fetchone()[0] for table in EMPTY_TABLES}
+            assert all(value == 0 for value in empty.values()), empty
+            assert database.execute('SELECT * FROM sqlite_sequence').fetchall() == []
+    expected = dict(schemaVersion=INPUTS['schemaVersion'], itemCount=count,
+                    first=None if count == 0 else dict(item(1), completed=False),
+                    last=None if count == 0 else dict(item(count), completed=False), emptyTables=empty)
+    assert report['database'] == expected, report['database']
+    return dict(helper=report, rowCount=count,
+                actualRowsSha256=sha(json.dumps(rows, sort_keys=True, separators=(',', ':')).encode()),
+                emptyTables=empty)
+
+
+class LargeRelease(Scenario):
+    # Previously verified radio observation and attached-task teardown, unchanged.
+    network_state = OfflineQueueScenario.network_state
+    wait_network = OfflineQueueScenario.wait_network
+    go_offline = OfflineQueueScenario.go_offline
+    restore_network = OfflineQueueScenario.restore_network
+    teardown_gate = ActivityStateScenario.teardown_gate
+
+    def __init__(self, args):
+        self.args = args
+        self.output = args.output.resolve()
+        self.output.mkdir(parents=True, exist_ok=False)
+        self.command_count = self.dump_count = 0
+        self.network_changed = False
+        self.result = dict(scenario='M15', phase=args.phase, profile='phase-1', serial=SERIAL,
+                           package=PACKAGE, status='RUNNING', ordinaryLaunches=0,
+                           instrumentationRuns=[], full2000Invocation=args.phase == 'baseline',
+                           harnessSha256=sha(Path(__file__).read_bytes()),
+                           inputsSha256=sha(INPUTS_PATH.read_bytes()),
+                           apkSha256=sha(args.apk.read_bytes()), testApkSha256=sha(args.test_apk.read_bytes()))
+
+    def adb(self, *arguments, binary=False, allow_failure=False, timeout=None):
+        command = [str(self.args.adb), '-s', SERIAL, *arguments]
+        self.command_count += 1
+        prefix = self.output / f'command-{self.command_count:03d}'
+        with (self.output / 'commands.txt').open('a') as ledger:
+            ledger.write(f'{self.command_count:03d} {shlex.join(command)}\n')
+        started = time.monotonic()
+        timed_out = False
+        with prefix.with_suffix('.stdout').open('xb') as stdout, prefix.with_suffix('.stderr').open('xb') as stderr:
+            process = subprocess.Popen(command, stdout=stdout, stderr=stderr)
+            try:
+                code = process.wait(timeout=timeout or LIMITS['adbTimeoutSeconds'])
+            except subprocess.TimeoutExpired:
+                timed_out = True
+                process.kill()
+                code = process.wait()
+        raw = prefix.with_suffix('.stdout').read_bytes()
+        error = prefix.with_suffix('.stderr').read_bytes()
+        self.last_command = dict(number=self.command_count, command=command, hostPid=process.pid,
+                                 startedMonotonic=started, endedMonotonic=time.monotonic(),
+                                 exit=code, timedOut=timed_out, timeoutSeconds=timeout or LIMITS['adbTimeoutSeconds'],
+                                 stdoutBytes=len(raw), stdoutSha256=sha(raw), stderrBytes=len(error), stderrSha256=sha(error))
+        with (self.output / 'command-records.jsonl').open('a') as ledger:
+            ledger.write(json.dumps(self.last_command) + '\n')
+        with (self.output / 'commands.txt').open('a') as ledger:
+            ledger.write(f'    exit={code} timeout={timed_out}\n')
+        assert not timed_out, ('Command timed out; no retry', self.last_command)
+        if not allow_failure:
+            assert code == 0, (command, code, error)
+        return raw if binary else raw.decode('utf-8', errors='strict').strip()
+
+    def installed(self):
+        records = []
+        for package, artifact in ((PACKAGE, self.args.apk), (PACKAGE + '.test', self.args.test_apk)):
+            paths = self.adb('shell', 'pm', 'path', package).splitlines()
+            assert len(paths) == 1 and paths[0].startswith('package:'), paths
+            path = paths[0][len('package:'):]
+            checksum = self.adb('shell', 'sha256sum', path).split()
+            assert len(checksum) == 2 and checksum[0] == sha(artifact.read_bytes()) and checksum[1] == path
+            records.append(dict(package=package, path=path, sha256=checksum[0]))
+        # Native helper separately reads actual PackageManager debuggable flags and certificates.
+        self.adb('shell', 'dumpsys', 'package', PACKAGE)
+        registration = self.adb('shell', 'pm', 'list', 'instrumentation').splitlines()
+        expected = f'instrumentation:{RUNNER} (target={PACKAGE})'
+        assert registration.count(expected) == 1, registration
+        self.result['installedArtifacts'] = records
+
+    def instrument(self, method):
+        output = self.adb('shell', 'am', 'instrument', '-w', '-r', '-e', 'class', f'{TEST_CLASS}#{method}',
+                          RUNNER, timeout=LIMITS['instrumentationTimeoutSeconds'])
+        record = dict(self.last_command, method=method, ordinaryActivityLaunch=False)
+        self.result['instrumentationRuns'].append(record)
+        assert re.findall(r'INSTRUMENTATION_STATUS_CODE:\s*(-?\d+)', output) == ['1', '0'], output
+        assert 'OK (1 test)' in output and re.search(r'INSTRUMENTATION_CODE:\s*-1\s*$', output), output
+        assert 'FAILURES!!!' not in output and 'INSTRUMENTATION_FAILED' not in output
+
+    def export(self, stage, count):
+        self.adb('exec-out', 'tar', '-cf', '-', '-C', EXTERNAL, stage, binary=True)
+        archive = self.output / f'command-{self.command_count:03d}.stdout'
+        native = self.output / 'native'
+        native.mkdir(exist_ok=True)
+        assert not (native / stage).exists()
+        with tarfile.open(archive) as source:
+            for member in source.getmembers():
+                path = Path(member.name)
+                assert not path.is_absolute() and '..' not in path.parts and path.parts[0] == stage
+                assert member.isdir() or member.isfile(), member.name
+            source.extractall(native, filter='data')
+        audit = validate_native(native / stage, count)
+        self.result.setdefault('native', {})[stage] = audit
+        return audit
+
+    def first_usable_ui(self, started):
+        deadline = started + LIMITS['initialUiWaitSeconds']
+        while True:
+            tree = self.hierarchy()
+            ready = (self.matching(tree, text='Item 0001')
+                     and self.matching(tree, text='Items (2000)')
+                     and self.matching(tree, text='Pending changes: 0')
+                     and self.matching(tree, **{'class': 'android.widget.EditText', 'enabled': 'true', 'text': ''})
+                     and self.matching(tree, **{'content-desc': 'Completed: Item 0001', 'enabled': 'true', 'checked': 'false'})
+                     and self.matching(tree, **{'content-desc': 'Edit Item 0001', 'enabled': 'true'})
+                     and self.matching(tree, **{'content-desc': 'Delete Item 0001', 'enabled': 'true'}))
+            observed = time.monotonic()
+            assert observed < deadline, 'First usable list not observed within frozen functional deadline'
+            if ready:
+                return dict(observedAtMonotonic=observed, elapsedSeconds=observed - started,
+                            xml=f'ui-{self.dump_count:02d}.xml', criteria='Item0001 visible with enabled edit/delete/completion/editor; count2000 pending0')
+            time.sleep(LIMITS['pollSeconds'])
+
+    def run(self):
+        self.result['initialNetwork'] = self.wait_network(True)
+        if self.args.phase == 'probe':
+            self.adb('install', '-r', str(self.args.apk.resolve()))
+            self.adb('install', '-r', str(self.args.test_apk.resolve()))
+        else:
+            assert self.args.probe_result is not None, 'Baseline requires the same-byte two-row helper smoke'
+            proof = json.loads(self.args.probe_result.read_text())
+            for field in ('apkSha256', 'testApkSha256', 'harnessSha256', 'inputsSha256'):
+                assert proof[field] == self.result[field], (field, 'Probe did not test these bytes')
+            assert proof['phase'] == 'probe' and proof['status'] == 'PASS' and proof['ordinaryLaunches'] == 0
+            assert proof['cleanup']['appAbsent'] and proof['cleanup']['network']['activeDefaultNetwork'].isdigit()
+            self.result['probeProof'] = dict(path=str(self.args.probe_result.resolve()), sha256=sha(self.args.probe_result.read_bytes()))
+        self.installed()
+        self.adb('shell', 'am', 'force-stop', PACKAGE)
+        self.teardown_gate()
+        assert self.adb('shell', 'pm', 'clear', PACKAGE) == 'Success'
+        self.teardown_gate()
+        self.go_offline()
+        self.adb('logcat', '-c')
+        if self.args.phase == 'probe':
+            self.instrument('probeReleaseSeedAndAuditWithoutActivity')
+            seeded = self.export('probe-seeded', INPUTS['probeCount'])
+            empty = self.export('probe-empty', 0)
+            assert seeded['helper']['pid'] == empty['helper']['pid']
+            raw = self.adb('logcat', '-b', 'all', '-v', 'threadtime', '-d')
+            observed = observations(raw, seeded['helper']['pid'])
+            assert [event['returnedRows'] for event in observed] == [2]
+            self.result.update(status='PASS', probeRows=2, queryObservations=observed,
+                               observed='Release helper seed/read/export/empty audit only; no Activity launch or cold-start timing')
+            return
+
+        self.instrument('seedFixedCanonicalDatasetWithoutActivity')
+        seeded = self.export('seeded', INPUTS['dataset']['count'])
+        self.result['seedInstrumentationExited'] = True
+        self.result['pidAfterSeedExit'] = self.adb('shell', 'pidof', PACKAGE, allow_failure=True)
+        assert self.result['pidAfterSeedExit'] in ('', str(seeded['helper']['pid']))
+        self.adb('shell', 'am', 'force-stop', PACKAGE)
+        self.teardown_gate()
+        self.result['processAbsentBeforeOrdinaryLaunch'] = True
+        self.result['offlineBeforeOrdinaryLaunch'] = self.wait_network(False)
+        self.adb('logcat', '-c')
+        started = time.monotonic()
+        self.result['ordinaryLaunches'] += 1
+        launch = self.adb('shell', 'am', 'start', '-W', '-n', f'{PACKAGE}/.MainActivity',
+                          timeout=LIMITS['initialUiWaitSeconds'])
+        self.result['ordinaryLaunchCommand'] = dict(self.last_command)
+        assert re.search(r'^Status: ok\s*$', launch, re.M), launch
+        pid = self.adb('shell', 'pidof', PACKAGE)
+        assert pid.isdigit() and int(pid) != seeded['helper']['pid'], pid
+        self.result['coldPid'] = int(pid)
+        self.result['firstUsableList'] = self.first_usable_ui(started)
+        raw = self.adb('logcat', '-b', 'all', '-v', 'threadtime', '-d')
+        self.result['initialObservationLogCommand'] = dict(self.last_command)
+        events = observations(raw, int(pid))
+        assert all(event['returnedRows'] == INPUTS['dataset']['count'] for event in events), events
+        total = sum(event['returnedRows'] for event in events)
+        assert total > INPUTS['initialCumulativeItemRowsMaximum'], ('No unbounded limitation observed', events)
+        self.result.update(queryObservations=events, initialCumulativeRows=total,
+                           initialObservationWindowClosedMonotonic=time.monotonic(),
+                           reach2000='NOT_RUN on baseline limitation branch', offlineCrud='NOT_RUN on baseline limitation branch',
+                           virtualization='Existing eager rows.forEach retained; no new runtime rendering claim')
+        self.adb('exec-out', 'screencap', '-p', binary=True)
+        self.result['initialScreenshotCommand'] = dict(self.last_command)
+        self.result['offlineAfterMeasurement'] = self.wait_network(False)
+        # Target instrumentation may create a different process here; the measured
+        # ordinary cold-start window is already closed, and no Activity is launched.
+        self.instrument('auditCanonicalDatasetWithoutActivity')
+        audited = self.export('audited', INPUTS['dataset']['count'])
+        assert audited['actualRowsSha256'] == seeded['actualRowsSha256']
+        self.result.update(status='LIMITATION_REPRODUCED', auditOutsideMeasurement=True,
+                           observed='Ordinary nondebuggable release launch materialized unbounded Room rows before mapping')
+
+    def cleanup(self):
+        if self.result['status'] == 'FAIL':
+            self.adb('logcat', '-b', 'all', '-v', 'threadtime', '-d')
+            self.result['failureLogCommand'] = dict(self.last_command)
+            self.adb('shell', 'dumpsys', 'activity', 'activities')
+        self.adb('shell', 'am', 'force-stop', PACKAGE)
+        self.teardown_gate()
+        if self.network_changed:
+            self.restore_network()
+        absent = not self.adb('shell', 'pidof', PACKAGE, allow_failure=True)
+        network = self.wait_network(True)
+        self.result['cleanup'] = dict(appAbsent=absent, network=network, fixtureUsed=False)
+        assert absent
+
+
+def main():
+    parser = argparse.ArgumentParser(description=__doc__)
+    parser.add_argument('--phase', choices=('probe', 'baseline'), required=True)
+    parser.add_argument('--apk', type=Path, required=True)
+    parser.add_argument('--test-apk', type=Path, required=True)
+    parser.add_argument('--adb', type=Path, required=True)
+    parser.add_argument('--output', type=Path, required=True)
+    parser.add_argument('--probe-result', type=Path)
+    args = parser.parse_args()
+    scenario = LargeRelease(args)
+    try:
+        scenario.run()
+    except Exception as error:
+        scenario.result.update(status='FAIL', error=repr(error))
+        raise
+    finally:
+        try:
+            scenario.cleanup()
+        except Exception as error:
+            scenario.result.update(status='FAIL', cleanupError=repr(error))
+            raise
+        finally:
+            scenario.result['adbCommands'] = scenario.command_count
+            (scenario.output / 'result.json').write_text(json.dumps(scenario.result, indent=2) + '\n')
+            records = [dict(path=str(path.relative_to(scenario.output)), bytes=path.stat().st_size,
+                            sha256=sha(path.read_bytes())) for path in sorted(scenario.output.rglob('*')) if path.is_file()]
+            (scenario.output / 'artifacts.json').write_text(json.dumps(records, indent=2) + '\n')
+
+
+if __name__ == '__main__':
+    main()
diff --git a/verification/large_release_final.py b/verification/large_release_final.py
new file mode 100644
index 0000000..7db2247
--- /dev/null
+++ b/verification/large_release_final.py
@@ -0,0 +1,476 @@
+#!/usr/bin/env python3
+"""Root-only M15 final release run: one cold launch, ordinary paging and offline CRUD.
+
+The frozen baseline helper is reused for transport, seed and cleanup. No target
+instrumentation runs between the ordinary launch and the end of the UI window.
+The separate small-native phase uses 31 rows and never launches an Activity.
+"""
+import argparse
+import json
+from pathlib import Path
+import re
+import shutil
+import sqlite3
+import tarfile
+import tempfile
+import time
+import uuid
+import xml.etree.ElementTree as ET
+
+from large_release import (DB_NAME, EMPTY_TABLES, EXTERNAL, INPUTS, INPUTS_PATH, LIMITS,
+                           PACKAGE, LargeRelease, item, sha)
+
+PAGE_SIZE = 25
+PAGE_SQL = 'SELECT * FROM items ORDER BY rowid LIMIT :limit OFFSET :offset'
+COUNT_SQL = 'SELECT COUNT(*) FROM items'
+UI_ACTION_SECONDS = 30  # Existing Scenario.wait_for bound, not a cold-start SLA.
+
+
+def log_events(raw, prefix, pid):
+    events = []
+    for line in raw.splitlines():
+        if prefix + ' ' not in line:
+            continue
+        match = re.fullmatch(r'\d\d-\d\d\s+\d\d:\d\d:\d\d\.\d+\s+(\d+)\s+(\d+)\s+'
+                             r'[VDIWEF]\s+System\.out\s*:\s*' + re.escape(prefix) + r' (\{.*\})', line)
+        assert match, ('Malformed observation', line)
+        assert int(match[1]) == pid, ('Unexpected observation process', line)
+        events.append(dict(value=json.loads(match[3]), pid=pid, tid=int(match[2]), rawLine=line))
+    return events
+
+
+def bounded_observations(raw, pid, initial=False):
+    reads = log_events(raw, 'M15_ITEM_READ', pid)
+    assert reads, 'No actual returned-entity observation'
+    for record in reads:
+        event = record['value']
+        assert set(event) == {'sql', 'limit', 'offset', 'returnedRows', 'startedNanos', 'finishedNanos', 'boundary'}
+        assert event['sql'] == PAGE_SQL and event['limit'] == PAGE_SIZE, event
+        assert type(event['offset']) is int and event['offset'] >= 0 and event['offset'] % PAGE_SIZE == 0, event
+        assert type(event['returnedRows']) is int and 0 <= event['returnedRows'] <= PAGE_SIZE, event
+        assert type(event['startedNanos']) is int and type(event['finishedNanos']) is int, event
+        assert 0 < event['startedNanos'] <= event['finishedNanos'], event
+        assert event['boundary'] == 'Room entities before domain mapping', event
+    counts = log_events(raw, 'M15_ITEM_COUNT', pid)
+    for record in counts:
+        event = record['value']
+        assert set(event) == {'sql', 'scalarResult', 'itemRowsMaterialized'}
+        assert event['sql'] == COUNT_SQL and event['itemRowsMaterialized'] == 0, event
+        assert type(event['scalarResult']) is int and event['scalarResult'] >= 0, event
+    total = sum(event['value']['returnedRows'] for event in reads)
+    if initial:
+        # Both existing startup access points must be present, not just a chosen first read.
+        assert len(reads) == len(counts) == 2, (reads, counts)
+        assert all(event['value']['offset'] == 0 and event['value']['returnedRows'] == PAGE_SIZE for event in reads)
+        assert all(event['value']['scalarResult'] == INPUTS['dataset']['count'] for event in counts)
+        assert total <= INPUTS['initialCumulativeItemRowsMaximum'], total
+    return dict(reads=reads, scalarCounts=counts, cumulativeItemRows=total)
+
+
+def initial_render_observations(raw, pid):
+    events = log_events(raw, 'M15_ROW_RENDER', pid)
+    assert events, 'No actual LazyColumn row composition observations'
+    active, entered, peak = set(), set(), 0
+    for record in events:
+        value = record['value']
+        assert set(value) == {'event', 'id'} and value['event'] in ('enter', 'leave'), value
+        assert re.fullmatch(r'large-\d{4}', value['id']), value
+        assert 1 <= int(value['id'][-4:]) <= PAGE_SIZE, value
+        if value['event'] == 'enter':
+            assert value['id'] not in active, ('Duplicate active row', value)
+            active.add(value['id'])
+            entered.add(value['id'])
+            peak = max(peak, len(active))
+        else:
+            assert value['id'] in active, ('Disposed row was not observed', value)
+            active.remove(value['id'])
+    assert 'large-0001' in entered and 0 < peak <= PAGE_SIZE < INPUTS['dataset']['count']
+    return dict(events=events, enteredIds=sorted(entered), activeIds=sorted(active), peakComposedRows=peak,
+                interpretation='Actual LazyColumn composition on the first bounded page; not all 2000 rows rendered')
+
+
+def require_small_native_proof(proof, current):
+    for field in ('apkSha256', 'testApkSha256', 'harnessSha256', 'sharedHarnessSha256', 'inputsSha256'):
+        assert proof[field] == current[field], (field, 'Native helper did not test these frozen bytes')
+    assert proof['phase'] == 'small-native' and proof['status'] == 'PASS'
+    assert proof['ordinaryLaunches'] == 0 and proof['full2000Invocation'] is False and proof['fixtureRows'] == 31
+    assert proof['cleanup']['appAbsent'] and proof['cleanup']['network']['activeDefaultNetwork'].isdigit()
+
+
+def pending_requests(pending):
+    """Independently reconstruct the four frozen wire envelopes from native columns."""
+    fixed = INPUTS['finalOfflineCrud']
+    assert len(pending) == 4
+    assert [row['sequence'] for row in pending] == [1, 2, 3, 4]
+    assert [row['operation'] for row in pending] == ['RENAME', 'COMPLETE', 'CREATE', 'DELETE']
+    assert [row['baseVersion'] for row in pending] == [1, 1, None, 0]
+    local_id = pending[2]['itemId']
+    assert str(uuid.UUID(local_id)) == local_id and uuid.UUID(local_id).version == 4
+    assert [row['itemId'] for row in pending] == [fixed['existingId'], fixed['existingId'], local_id, local_id]
+    assert [row['title'] for row in pending] == [fixed['renamedTitle'], None, fixed['createdTitle'], None]
+    assert [row['completed'] for row in pending] == [None, 1, 0, None]
+    assert len({row['clientMutationId'] for row in pending}) == 4
+    payloads = [dict(title=fixed['renamedTitle'], baseVersion=1), dict(completed=True, baseVersion=1),
+                dict(id=local_id, title=fixed['createdTitle'], completed=False), dict(baseVersion=0)]
+    requests = []
+    for row, method, payload in zip(pending, ('PATCH', 'PATCH', 'POST', 'DELETE'), payloads):
+        assert row['dispatched'] == 0 and row['terminalError'] is None
+        assert str(uuid.UUID(row['clientMutationId'])) == row['clientMutationId']
+        assert uuid.UUID(row['clientMutationId']).version == 4
+        request = dict(method=method, path='/items' if method == 'POST' else '/items/' + row['itemId'], payload=payload)
+        canonical = json.dumps(request, sort_keys=True, separators=(',', ':'), ensure_ascii=False)
+        assert row['payloadHash'] == sha(canonical.encode()), ('Native immutable hash mismatch', row)
+        requests.append(dict(clientMutationId=row['clientMutationId'], payloadHash=row['payloadHash'], canonical=canonical))
+    return requests
+
+
+def validate_crud_native(folder):
+    """Distinct post-CRUD check. The canonical/empty baseline predicate stays unchanged."""
+    report = json.loads((folder / 'result.json').read_text())
+    assert report['status'] == 'PASS' and report['stage'] == 'after-crud'
+    assert report['targetPackage'] == PACKAGE and report['testPackage'] == PACKAGE + '.test'
+    assert report['application'] == PACKAGE + '.ItemApplication' and report['targetDebuggable'] is False
+    assert report['targetSignerSha256'] == report['testSignerSha256'] and len(report['targetSignerSha256']) == 1
+    assert report['observedCreatedActivities'] == 0 and report['activeActivities'] == []
+    assert report['roomAndAuditHandlesClosedBeforeCopy'] is True and type(report['pid']) is int and report['pid'] > 0
+    names = []
+    for record in report['files']:
+        assert record['name'] in (DB_NAME, DB_NAME + '-wal', DB_NAME + '-shm')
+        path = folder / record['name']
+        assert path.stat().st_size == record['bytes'] and sha(path.read_bytes()) == record['sha256']
+        names.append(record['name'])
+    assert DB_NAME in names and len(names) == len(set(names))
+    with tempfile.TemporaryDirectory(prefix='mse-m15-crud-native-') as temporary:
+        local = Path(temporary)
+        for suffix in ('', '-wal'):
+            source = folder / (DB_NAME + suffix)
+            if source.exists():
+                shutil.copyfile(source, local / source.name)
+        with sqlite3.connect(local / DB_NAME) as database:
+            assert database.execute('PRAGMA integrity_check').fetchone() == ('ok',)
+            assert database.execute('PRAGMA user_version').fetchone() == (INPUTS['schemaVersion'],)
+            columns = [row[1] for row in database.execute('PRAGMA table_info(items)')]
+            assert columns == ['id', 'title', 'completed', 'version', 'updatedAt']
+            rows = [dict(zip(columns, row)) for row in database.execute('SELECT * FROM items ORDER BY rowid')]
+            assert len(rows) == INPUTS['finalOfflineCrud']['finalItemCount'] == 2000
+            assert all(row == item(index) for index, row in enumerate(rows[:-1], 1)), 'Unedited canonical rows changed'
+            last = rows[-1]
+            assert last == dict(item(2000), title=INPUTS['finalOfflineCrud']['renamedTitle'], completed=1,
+                                updatedAt=last['updatedAt']) and last['updatedAt'] > item(2000)['updatedAt']
+            pending_columns = [row[1] for row in database.execute('PRAGMA table_info(pending_mutations)')]
+            pending = [dict(zip(pending_columns, row)) for row in database.execute('SELECT * FROM pending_mutations ORDER BY sequence')]
+            requests = pending_requests(pending)
+            counts = {table: database.execute(f'SELECT COUNT(*) FROM {table}').fetchone()[0] for table in EMPTY_TABLES}
+            assert counts == dict(pending_mutations=4, acknowledged_mutations=0, tombstones=0, automatic_sync=1, sync_metadata=0)
+            cycles = database.execute('SELECT id,workId,httpAttempts,state FROM automatic_sync').fetchall()
+            assert len(cycles) == 1 and cycles[0][0] == 1 and cycles[0][2:] == (0, 'ACTIVE'), cycles
+            cycle = dict(zip(('id', 'workId', 'httpAttempts', 'state'), cycles[0]))
+            assert str(uuid.UUID(cycle['workId'])) == cycle['workId']
+            allocator = database.execute('SELECT * FROM sqlite_sequence').fetchall()
+            assert allocator == [('pending_mutations', 4)], allocator
+    boolean_pending = [dict(row, completed=None if row['completed'] is None else bool(row['completed']),
+                            dispatched=bool(row['dispatched'])) for row in pending]
+    expected = dict(schemaVersion=INPUTS['schemaVersion'], itemCount=2000,
+                    first=dict(rows[0], completed=False), last=dict(last, completed=True),
+                    tableCounts=counts, automaticCycle=cycle, pending=boolean_pending)
+    assert report['database'] == expected, report['database']
+    return dict(helper=report, rowCount=len(rows), actualRowsSha256=sha(json.dumps(rows, sort_keys=True, separators=(',', ':')).encode()),
+                pending=pending, requests=requests, tableCounts=counts, automaticCycle=cycle, allocator=allocator)
+
+
+def bounds(node):
+    match = re.fullmatch(r'\[(-?\d+),(-?\d+)\]\[(-?\d+),(-?\d+)\]', node.get('bounds', ''))
+    assert match, ('Malformed UI bounds', node.attrib)
+    value = tuple(map(int, match.groups()))
+    assert value[0] <= value[2] and value[1] <= value[3], value
+    return value
+
+
+def visible_nodes(tree, **attributes):
+    selected = LargeRelease.matching(tree, **attributes)
+    return [node for node in selected if (lambda b: b[0] < b[2] and b[1] < b[3])(bounds(node))]
+
+
+def scroll_container(tree, rows):
+    containers = []
+    for node in tree.iter('node'):
+        if node.get('scrollable') != 'true':
+            continue
+        children = list(node.iter('node'))
+        has_rows = any(child.get('content-desc', '').startswith('Completed: ') for child in children)
+        has_header = any(child.get('class') == 'android.widget.EditText'
+                         or child.get('text', '').startswith(('Pending changes:', 'Page ', 'Items (')) for child in children)
+        if (has_rows and not has_header) if rows else (has_header and not has_rows):
+            containers.append(node)
+    assert len(containers) == 1, ('Expected one list/header scroll container', rows, len(containers))
+    assert bounds(containers[0])[3] - bounds(containers[0])[1] > 24
+    return containers[0]
+
+
+class FinalLargeRelease(LargeRelease):
+    def __init__(self, args):
+        super().__init__(args)
+        self.result.update(full2000Invocation=args.phase == 'final', harnessSha256=sha(Path(__file__).read_bytes()),
+                           sharedHarnessSha256=sha(Path(__file__).with_name('large_release.py').read_bytes()), nextActions=[])
+
+    def same_pid(self, stage):
+        pid = self.adb('shell', 'pidof', PACKAGE)
+        assert pid == str(self.result['coldPid']), ('Ordinary process changed during UI window', stage, pid)
+        self.result.setdefault('ordinaryPidChecks', []).append(dict(stage=stage, pid=int(pid), command=dict(self.last_command)))
+
+    def tap_visible(self, tree, **attributes):
+        nodes = visible_nodes(tree, **attributes)
+        assert len(nodes) == 1, ('Expected one visible enabled control', attributes, len(nodes))
+        assert nodes[0].get('enabled') == 'true', nodes[0].attrib
+        left, top, right, bottom = bounds(nodes[0])
+        self.adb('shell', 'input', 'tap', str((left + right) // 2), str((top + bottom) // 2))
+        return dict(self.last_command)
+
+    def swipe(self, tree, rows, forward):
+        node = scroll_container(tree, rows)
+        left, top, right, bottom = bounds(node)
+        start, end = (bottom - 12, top + 12) if forward else (top + 12, bottom - 12)
+        self.adb('shell', 'input', 'swipe', str((left + right) // 2), str(start),
+                 str((left + right) // 2), str(end), '300')
+
+    def reveal_header(self, tree, forward, **attributes):
+        deadline = time.monotonic() + UI_ACTION_SECONDS
+        while not visible_nodes(tree, **attributes):
+            assert time.monotonic() < deadline, ('Header control not reachable', attributes)
+            before = ET.tostring(scroll_container(tree, False))
+            self.swipe(tree, rows=False, forward=forward)
+            tree = self.hierarchy()
+            assert ET.tostring(scroll_container(tree, False)) != before, ('Header scroll made no progress', attributes)
+        return tree
+
+    @staticmethod
+    def row_ready(tree, title, completed):
+        return bool(visible_nodes(tree, text=title)
+                    and visible_nodes(tree, **{'content-desc': 'Completed: ' + title, 'checked': str(completed).lower(), 'enabled': 'true'})
+                    and visible_nodes(tree, **{'content-desc': 'Edit ' + title, 'enabled': 'true'})
+                    and visible_nodes(tree, **{'content-desc': 'Delete ' + title, 'enabled': 'true'}))
+
+    def next_page(self, tree, page, total, pending, first_title, stage):
+        tree = self.reveal_header(tree, forward=True, text='Next', enabled='true')
+        command = self.tap_visible(tree, text='Next', enabled='true')
+        label = f'Page {page} / {(total + PAGE_SIZE - 1) // PAGE_SIZE}'
+        tree = self.wait_for(lambda ui: bool(visible_nodes(ui, text=label)
+                                           and visible_nodes(ui, text=f'Items ({total})')
+                                           and visible_nodes(ui, text=f'Pending changes: {pending}')
+                                           and self.row_ready(ui, first_title, False)), label)
+        self.result['nextActions'].append(dict(stage=stage, page=page, command=command,
+                                               firstTitle=first_title, xml=f'ui-{self.dump_count:02d}.xml'))
+        return tree
+
+    def reveal_row(self, tree, title, completed):
+        deadline = time.monotonic() + UI_ACTION_SECONDS
+        gestures = 0
+        while not self.row_ready(tree, title, completed):
+            assert time.monotonic() < deadline and gestures < PAGE_SIZE, ('Row not reached in ordinary page', title)
+            before = ET.tostring(scroll_container(tree, True))
+            self.swipe(tree, rows=True, forward=True)
+            gestures += 1
+            tree = self.hierarchy()
+            assert ET.tostring(scroll_container(tree, True)) != before, ('List scroll made no progress', title)
+        return tree
+
+    def title_input(self, tree, value, previous):
+        tree = self.reveal_header(tree, forward=False, **{'class': 'android.widget.EditText', 'enabled': 'true'})
+        nodes = visible_nodes(tree, **{'class': 'android.widget.EditText', 'enabled': 'true'})
+        assert len(nodes) == 1 and nodes[0].get('text') == previous, (previous, [node.attrib for node in nodes])
+        self.tap_visible(tree, **{'class': 'android.widget.EditText', 'enabled': 'true'})
+        if previous:
+            self.adb('shell', 'input', 'keyevent', 'KEYCODE_MOVE_END')
+            self.adb('shell', 'input', 'keyevent', *(['KEYCODE_DEL'] * len(previous)))
+        self.text(value)
+        return self.wait_for(lambda ui: bool(visible_nodes(ui, **{'class': 'android.widget.EditText', 'text': value})), 'typed title')
+
+    def saved(self, pending, total):
+        return self.wait_for(lambda ui: bool(visible_nodes(ui, text=f'Pending changes: {pending}')
+                                           and visible_nodes(ui, text=f'Items ({total})')
+                                           and visible_nodes(ui, **{'class': 'android.widget.EditText', 'enabled': 'true', 'text': ''})),
+                             f'{pending} committed intents and {total} items')
+
+    def record_crud(self, stage, command):
+        self.same_pid(stage)
+        self.result.setdefault('crudActions', []).append(dict(stage=stage, command=command,
+                                                             completedXml=f'ui-{self.dump_count:02d}.xml'))
+
+    def offline_crud(self, tree):
+        fixed = INPUTS['finalOfflineCrud']
+        self.tap_visible(tree, **{'content-desc': 'Edit Item 2000', 'enabled': 'true'})
+        tree = self.wait_for(lambda ui: bool(visible_nodes(ui, text='Save', enabled='true')), 'selected Item 2000')
+        tree = self.title_input(tree, fixed['renamedTitle'], 'Item 2000')
+        command = self.tap_visible(tree, text='Save', enabled='true')
+        tree = self.saved(1, 2000)
+        tree = self.reveal_row(tree, fixed['renamedTitle'], False)
+        self.record_crud('rename', command)
+        command = self.tap_visible(tree, **{'content-desc': 'Completed: ' + fixed['renamedTitle'], 'enabled': 'true'})
+        tree = self.saved(2, 2000)
+        tree = self.reveal_row(tree, fixed['renamedTitle'], True)
+        self.record_crud('complete', command)
+        tree = self.title_input(tree, fixed['createdTitle'], '')
+        command = self.tap_visible(tree, text='Add', enabled='true')
+        tree = self.saved(3, 2001)
+        self.record_crud('create', command)
+        tree = self.next_page(tree, 81, 2001, 3, fixed['createdTitle'], 'CRUD reach newly created item')
+        command = self.tap_visible(tree, **{'content-desc': 'Delete ' + fixed['createdTitle'], 'enabled': 'true'})
+        tree = self.saved(4, 2000)
+        tree = self.wait_for(lambda ui: bool(visible_nodes(ui, text='Page 80 / 80')
+                                           and self.row_ready(ui, 'Item 1976', False)), 'clamped last page after delete')
+        assert not visible_nodes(tree, text=fixed['createdTitle'])
+        self.record_crud('delete', command)
+        self.result['offlineCrud'] = dict(status='PASS_UI', finalItems=2000, finalPending=4, ordinaryActions=4,
+                                         generatedIdentity='production UUID, independently checked after measurement')
+        return tree
+
+    def export_crud(self):
+        stage = 'after-crud'
+        self.adb('exec-out', 'tar', '-cf', '-', '-C', EXTERNAL, stage, binary=True)
+        archive = self.output / f'command-{self.command_count:03d}.stdout'
+        native = self.output / 'native'
+        assert not (native / stage).exists()
+        with tarfile.open(archive) as source:
+            for member in source.getmembers():
+                path = Path(member.name)
+                assert not path.is_absolute() and '..' not in path.parts and path.parts[0] == stage
+                assert member.isdir() or member.isfile(), member.name
+            source.extractall(native, filter='data')
+        result = validate_crud_native(native / stage)
+        self.result.setdefault('native', {})[stage] = result
+        return result
+
+    def run(self):
+        self.result['initialNetwork'] = self.wait_network(True)
+        if self.args.phase == 'small-native':
+            self.adb('install', '-r', str(self.args.apk.resolve()))
+            self.adb('install', '-r', str(self.args.test_apk.resolve()))
+            self.installed()
+            self.adb('shell', 'am', 'force-stop', PACKAGE)
+            self.teardown_gate()
+            self.go_offline()
+            self.adb('logcat', '-c')
+            self.instrument('smallNativePagesPreserveOrderClampAndAtomicCrud')
+            raw = self.adb('logcat', '-b', 'all', '-v', 'threadtime', '-d')
+            matches = [line for line in raw.splitlines() if 'M15_NATIVE_PAGING ' in line]
+            assert len(matches) == 1, matches
+            value = json.loads(matches[0].split('M15_NATIVE_PAGING ', 1)[1])
+            assert type(value['pid']) is int and value['pid'] > 0
+            assert value == dict(status='PASS', pid=value['pid'], fixtureRows=31,
+                                 initialCumulativeRows=50, fourIntentsVerified=True), value
+            native_events = log_events(raw, 'M15_NATIVE_PAGING', value['pid'])
+            assert len(native_events) == 1
+            self.result.update(status='PASS', fixtureRows=31, nativeObservation=native_events[0],
+                               observed='Native Room page/order/clamp/atomic CRUD; no Activity or full2000 launch')
+            return
+        assert self.args.baseline_result is not None
+        assert self.args.small_native_result is not None, 'Final requires the same-byte isolated native page regression'
+        require_small_native_proof(json.loads(self.args.small_native_result.read_text()), self.result)
+        self.result['smallNativeProof'] = dict(path=str(self.args.small_native_result.resolve()),
+                                              sha256=sha(self.args.small_native_result.read_bytes()))
+        proof = json.loads(self.args.baseline_result.read_text())
+        assert proof['phase'] == 'baseline' and proof['status'] == 'LIMITATION_REPRODUCED'
+        assert proof['inputsSha256'] == sha(INPUTS_PATH.read_bytes()) and proof['ordinaryLaunches'] == 1
+        self.result['retainedBaselineProof'] = dict(path=str(self.args.baseline_result.resolve()),
+                                                   sha256=sha(self.args.baseline_result.read_bytes()),
+                                                   meaning='Accepted previous app limitation; not a same-byte final runtime probe')
+        self.installed()
+        self.adb('shell', 'am', 'force-stop', PACKAGE)
+        self.teardown_gate()
+        assert self.adb('shell', 'pm', 'clear', PACKAGE) == 'Success'
+        self.teardown_gate()
+        self.go_offline()
+        self.adb('logcat', '-c')
+        self.instrument('seedFixedCanonicalDatasetWithoutActivity')
+        seeded = self.export('seeded', INPUTS['dataset']['count'])
+        self.result['seedInstrumentationExited'] = True
+        self.result['pidAfterSeedExit'] = self.adb('shell', 'pidof', PACKAGE, allow_failure=True)
+        assert self.result['pidAfterSeedExit'] in ('', str(seeded['helper']['pid']))
+        self.adb('shell', 'am', 'force-stop', PACKAGE)
+        self.teardown_gate()
+        self.result['processAbsentBeforeOrdinaryLaunch'] = True
+        self.result['offlineBeforeOrdinaryLaunch'] = self.wait_network(False)
+        self.adb('logcat', '-c')
+        started = time.monotonic()
+        self.result['ordinaryLaunches'] += 1
+        launch = self.adb('shell', 'am', 'start', '-W', '-n', f'{PACKAGE}/.MainActivity', timeout=LIMITS['initialUiWaitSeconds'])
+        self.result['ordinaryLaunchCommand'] = dict(self.last_command)
+        assert re.search(r'^Status: ok\s*$', launch, re.M), launch
+        pid = self.adb('shell', 'pidof', PACKAGE)
+        assert pid.isdigit() and int(pid) != seeded['helper']['pid']
+        self.result['coldPid'] = int(pid)
+        self.result['firstUsableList'] = self.first_usable_ui(started)
+        tree = ET.parse(self.output / self.result['firstUsableList']['xml']).getroot()
+        assert visible_nodes(tree, text='Page 1 / 80') and not visible_nodes(tree, text='Item 2000')
+        list_bounds = bounds(scroll_container(tree, True))
+        raw = self.adb('logcat', '-b', 'all', '-v', 'threadtime', '-d')
+        self.result['initialObservationLogCommand'] = dict(self.last_command)
+        observed = bounded_observations(raw, int(pid), initial=True)
+        self.result.update(queryObservations=observed, initialCumulativeRows=observed['cumulativeItemRows'],
+                           virtualization=dict(initial_render_observations(raw, int(pid)), visibleListBounds=list_bounds),
+                           initialObservationWindowClosedMonotonic=time.monotonic())
+        self.adb('exec-out', 'screencap', '-p', binary=True)
+        self.result['initialScreenshotCommand'] = dict(self.last_command)
+        self.adb('shell', 'dumpsys', 'activity', 'activities')
+        self.result['ordinaryInitialActivitiesCommand'] = dict(self.last_command)
+        self.same_pid('initial observation complete')
+        for page in range(2, 81):
+            tree = self.next_page(tree, page, 2000, 0, item((page - 1) * PAGE_SIZE + 1)['title'], 'reach Item 2000')
+            if page % 20 == 0:
+                self.same_pid(f'page {page}')
+        tree = self.reveal_row(tree, 'Item 2000', False)
+        self.adb('exec-out', 'screencap', '-p', binary=True)
+        self.result['reach2000'] = dict(status='PASS_UI', nextActions=79, xml=f'ui-{self.dump_count:02d}.xml',
+                                       screenshotCommand=dict(self.last_command))
+        self.same_pid('Item 2000 reached')
+        self.offline_crud(tree)
+        assert len(self.result['nextActions']) == 80
+        self.adb('exec-out', 'screencap', '-p', binary=True)
+        self.result['afterCrudScreenshotCommand'] = dict(self.last_command)
+        self.result['offlineAfterMeasurement'] = self.wait_network(False)
+        self.adb('logcat', '-b', 'all', '-v', 'threadtime', '-d')
+        self.result['ordinaryWindowFinalLogCommand'] = dict(self.last_command)
+        self.adb('shell', 'dumpsys', 'activity', 'activities')
+        self.result['ordinaryFinalActivitiesCommand'] = dict(self.last_command)
+        self.adb('shell', 'dumpsys', 'jobscheduler', PACKAGE)
+        self.result['ordinaryFinalJobsCommand'] = dict(self.last_command)
+        self.same_pid('ordinary UI window complete')
+        self.result['ordinaryUiWindowClosedMonotonic'] = time.monotonic()
+        self.instrument('auditOfflineCrudDatasetWithoutActivity')
+        audited = self.export_crud()
+        assert audited['helper']['targetSignerSha256'] == seeded['helper']['targetSignerSha256']
+        self.result.update(status='PASS', auditOutsideMeasurement=True, ordinaryPagingNextActions=80,
+                           observed='One nondebuggable release process; cumulative initial50; actual virtualized paging to2000; four durable offline intents')
+
+
+def main():
+    parser = argparse.ArgumentParser(description=__doc__)
+    parser.add_argument('--phase', choices=('final', 'small-native'), required=True)
+    parser.add_argument('--apk', type=Path, required=True)
+    parser.add_argument('--test-apk', type=Path, required=True)
+    parser.add_argument('--adb', type=Path, required=True)
+    parser.add_argument('--output', type=Path, required=True)
+    parser.add_argument('--baseline-result', type=Path)
+    parser.add_argument('--small-native-result', type=Path)
+    scenario = FinalLargeRelease(parser.parse_args())
+    try:
+        scenario.run()
+    except Exception as error:
+        scenario.result.update(status='FAIL', error=repr(error))
+        raise
+    finally:
+        try:
+            scenario.cleanup()
+        except Exception as error:
+            scenario.result.update(status='FAIL', cleanupError=repr(error))
+            raise
+        finally:
+            scenario.result['adbCommands'] = scenario.command_count
+            (scenario.output / 'result.json').write_text(json.dumps(scenario.result, indent=2) + '\n')
+            artifacts = [dict(path=str(path.relative_to(scenario.output)), bytes=path.stat().st_size, sha256=sha(path.read_bytes()))
+                         for path in sorted(scenario.output.rglob('*')) if path.is_file()]
+            (scenario.output / 'artifacts.json').write_text(json.dumps(artifacts, indent=2) + '\n')
+
+
+if __name__ == '__main__':
+    main()
diff --git a/verification/phase-1/M15.md b/verification/phase-1/M15.md
new file mode 100644
index 0000000..7d525a0
--- /dev/null
+++ b/verification/phase-1/M15.md
@@ -0,0 +1,84 @@
+# M15 verification ledger — phase-1 release paging
+
+- Thread: `M15`; profile: `phase-1`; branch: `track/android-kotlin`.
+- SPEC_REVISION: `61280dd86ce88b6e431f408241c0998a275960aa`.
+- START: `a9ca3a773fce81c5130adeccfa95cf0ffb439ae9` (verified M14).
+- Attempt1 / repair0; actual full2000 usage **2/3**. Both full cases and both native helpers were root-run; no owner device execution or warmup.
+- Status: root accepted final runtime. This commit contains the frozen eleven candidate files and only TRACK/this ledger as subsequent metadata. Final history/tag audit remains root-owned.
+
+Evidence root `E` is `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/android-kotlin/M15/`.
+Exact argv, environment, durations, exits and raw output remain in `E/commands.jsonl` and each named record.
+
+## Recorded executions
+
+| Check | Actual result | Evidence under E |
+| --- | --- | --- |
+| Release preparation | Build01 sandbox socket denial; build02 targeted host1 PASS then nullable signing-info helper compile failure; build03 fatal release lint; build04 PASS13.274s | `support-build-01`…`04.*`, `support-prebuild-01`…`03/` |
+| Baseline support | Host parser2 PASS; initial optimized-resource filename lookup failed, corrected inspection PASS; primary-source download DNS denial then approved success | `harness-host-01.*`, `artifact-inspection-01/02.*`, `network-config-source-01/02.*` |
+| Root two-row helper | PASS3.961s,43 adb, two native snapshots, no Activity or full-case charge | `helper-probe-01/`, `helper-probe-01.*` |
+| Root full2000 baseline | Limitation reproduced; exit0,19.594s,70 adb, two native snapshots | `baseline-android-01/`, `baseline-android-01.*` |
+| Fixed host/release build | Host33 PASS,0 skipped; app/test build PASS27.630s, once | `fixed-build-01.*`, `fixed-prebuild-01/`, `fixed-built-01/` |
+| Final controller | Focused synthetic parser/action4 plus proof-binding1 PASS; no Android | `final-harness-host-01.*`, `final-harness-setup-host-01.*` |
+| Root isolated31-row native | Actual JUnit1 PASS; wrapper0,3.656s,38 adb; no Activity/full-case charge | `fixed-small-native-01/`, `fixed-small-native-01.*` |
+| Root final2000 release | PASS; wrapper0,226.722s,356 adb,817 raw files, two native DB/WAL snapshots | `fixed-release-android-02/`, `fixed-release-android-02.*` |
+
+The explicit `includeSubdomains="false"` matches the existing Android parser default;
+lint remained enabled and the allowed host was not broadened. All five preparation
+failures remain preserved; none is a hidden device retry or a reset of the case counter.
+The [root budget audit](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M15/main-budget-audit.json)
+enumerates exactly two full cases and two zero-Activity helpers.
+
+## Baseline and final observations
+
+Baseline release kept the original query and eager UI behavior. After closed seed
+instrumentation and actual PID absence, one ordinary offline launch in PID16885 returned
+2000 Room entities twice: cumulative4000 before domain mapping. Canonical rows and empty
+queue were unchanged. Its observed first-usable time was14.964019s; end traversal/CRUD
+were explicitly NOT_RUN on that limitation branch. This is retained baseline evidence,
+not a final-release measurement.
+
+Final release was actually nondebuggable, with the audit APK signed by the same local key.
+Seed/audit instrumentation initializes ItemApplication/providers without starting an Activity;
+its owned Room/SQLite handles close before export and exit.
+Seed PID19863 exited, root proved process absence, and ordinary MainActivity launched
+PID19933. Both initial SQL reads had LIMIT25/OFFSET0 and returned25 entities each;
+cumulative50 includes both startup access points. Scalar COUNT returned2000 separately
+and materialized no Item entities. Initial LazyColumn composition peaked at7 rows.
+The single first-usable observation was3.497717916s including external UI sampling;
+no timing SLA, tuning sweep or statistical speedup is claimed.
+
+The same ordinary process handled79 real Next presses and list scrolling to Item2000.
+Rename to `Item 2000 edited`, completion=true, create `Large local`, and delete that created
+Item used the existing UI/store path. One extra Next reached the new Item on page81;
+deletion clamped back to page80. All86 ordinary-window Item queries remained bounded.
+Only after that UI window closed did readonly audit PID22408 inspect/export native storage.
+Root independently reopened both DB/WAL snapshots: exactly2000 Items,1999 canonical rows
+unchanged, edited Item2000 still version1, and four ordered original UUID/hash intents
+with bases1/1/null/0. All are undispatched/nonterminal. Automatic work remains enabled,
+with one ACTIVE cycle and0 HTTP reservations; no scheduler suppression or schema change.
+
+[Root final acceptance](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M15/main-final02-acceptance.json)
+has SHA256 `d0ba1894bc0eab8b01e8495c86dc6b96f387651c2910f81fc99ac9f3576af67b`.
+It links the baseline and [isolated31-row acceptance](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M15/main-small31-acceptance.json),
+and records root's raw SQL/identity/hash/PID/UI checks. Cleanup confirms app absent,
+network0/1/1 with active network145, and no fixture used.
+
+## Frozen scope and retained regressions
+
+[Final manifest](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/android-kotlin/M15/fixed-frozen-01/manifest.json),
+SHA256 `26d06d863e507dd862128642589201c4d665afb8e7b4746e755761d6f48d2b18`,
+binds75 pre-metadata files,11 source copies,619 retained support files and the executed argv.
+Release app SHA256 `4fed2c8b1f0e09a7d7a189eea4c3d0c2340695b2b3b364874c31b4f708072f03`;
+audit APK SHA256 `9e5adecbc6396ae9941e5aebb2b95bc2f1007081922fd587d260c7bf0fcc8a84`.
+Root's [compiled-artifact preflight](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M15/main-fixed-build-preflight.json)
+checked38 compiled inputs, actual manifest/signatures/DEX, all five public-void helper
+methods, generated transactional Room paging and host33 XML. No duplicate inspection/build ran.
+
+Host33 includes the existing queue/identity/conflict/editor/startup/scheduler/cancellation
+regressions plus focused paging checks. Actual final-release Android validation was the
+isolated31-row Room regression and the one fixed2000 ordinary UI/CRUD case with seed/audit
+instrumentation outside its window. Earlier M01–M10/M14 Android and fixture results are
+retained historical evidence, **not rerun on this release**; no new process-loss, lifecycle,
+background retry or cancellation acceptance is inferred from this M15 run.
+After root PASS only TRACK and this ledger changed. No product/test/fixture/input/build,
+device, tag, push or phase-2 action followed the frozen runtime.
