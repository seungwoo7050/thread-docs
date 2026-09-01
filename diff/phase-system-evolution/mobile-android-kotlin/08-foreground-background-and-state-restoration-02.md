## `feat(M08): retain editor state across Activity recreation`

diff --git a/TRACK.md b/TRACK.md
index 61b9d73..128abc3 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -203,6 +203,24 @@ cleanup. Historical M05/M06 external scripts are not claimed rerun on schema 5.
 repair0/2; only M07 reporting changed after verification. `verification/M07.md` links the
 commands, hashes and main results. Final commit audit/tagging remains main-owned.
 
+## M08 boundary
+
+The editor keeps only its selected Item ID and draft title in Compose saved state. Activity
+recreation restores that pair; Room remains the authority for committed Items and pending
+mutations. Save still delegates to the existing create/rename path, and only a successful
+write clears the editor. Sync status collection follows the existing Activity lifecycle
+while STARTED. No schema, sync protocol, dependency or background work was added.
+
+The unchanged M07 baseline reproduced draft/selection loss after one actual recreation,
+with the same process and unchanged storage. Host17/17 and the app build passed; the baseline
+instrumentation APK was reused unchanged. Main's one restored lifecycle case and all 20
+existing Android regressions passed with zero skips. Six native snapshots prove no writes
+before Save and exactly one base1 rename afterward; the lifecycle case made no HTTP request.
+This proves same-process Activity restoration, not process-kill draft durability. Main
+verified frozen execution bytes and cleanup. Attempt1, repair0/2; only reporting changed
+after runtime verification. Commands and evidence are in `verification/M08.md`; final
+history/tag audit remains main-owned.
+
 ## Pinned build
 
 - JDK 17, Gradle 8.7 (wrapper distribution SHA-256 pinned)
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemEditor.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemEditor.kt
new file mode 100644
index 0000000..e782d59
--- /dev/null
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemEditor.kt
@@ -0,0 +1,17 @@
+package com.mobilesystemsevolution.kotlin
+
+import androidx.compose.runtime.saveable.listSaver
+
+/** Selection and draft text are UI state; only submit changes committed Items. */
+internal data class ItemEditor(val itemId: String? = null, val title: String = "") {
+    suspend fun submit(store: ItemStore) {
+        if (itemId == null) store.create(title) else store.rename(itemId, title)
+    }
+
+    companion object {
+        val Saver = listSaver<ItemEditor, String?>(
+            save = { listOf(it.itemId, it.title) },
+            restore = { ItemEditor(itemId = it[0], title = checkNotNull(it[1])) },
+        )
+    }
+}
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
index c5c3ace..3cb4ae8 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
@@ -24,14 +24,18 @@ import androidx.compose.runtime.getValue
 import androidx.compose.runtime.mutableStateOf
 import androidx.compose.runtime.remember
 import androidx.compose.runtime.rememberCoroutineScope
+import androidx.compose.runtime.saveable.rememberSaveable
 import androidx.compose.runtime.setValue
 import androidx.compose.ui.Alignment
 import androidx.compose.ui.Modifier
 import androidx.compose.ui.platform.LocalFocusManager
+import androidx.compose.ui.platform.LocalLifecycleOwner
 import androidx.compose.ui.platform.testTag
 import androidx.compose.ui.semantics.contentDescription
 import androidx.compose.ui.semantics.semantics
 import androidx.compose.ui.unit.dp
+import androidx.lifecycle.Lifecycle
+import androidx.lifecycle.flowWithLifecycle
 import kotlinx.coroutines.CancellationException
 import kotlinx.coroutines.launch
 
@@ -60,9 +64,15 @@ internal fun ItemScreen(store: ItemStore, sync: ItemSync? = null) {
     var busy by remember(store) { mutableStateOf(true) }
     var storageError by remember(store) { mutableStateOf<String?>(null) }
     var syncing by remember(store) { mutableStateOf(false) }
-    val syncStatus = sync?.status?.collectAsState()?.value
-    var title by remember { mutableStateOf("") }
-    var editingId by remember { mutableStateOf<String?>(null) }
+    val lifecycle = LocalLifecycleOwner.current.lifecycle
+    val syncStatus = sync?.let { itemSync ->
+        val activeStatus = remember(itemSync, lifecycle) {
+            itemSync.status.flowWithLifecycle(lifecycle, Lifecycle.State.STARTED)
+        }
+        activeStatus.collectAsState(initial = itemSync.status.value).value
+    }
+    // Only the editor pair is saved UI state; committed rows and intents remain in Room.
+    var editor by rememberSaveable(stateSaver = ItemEditor.Saver) { mutableStateOf(ItemEditor()) }
     val scope = rememberCoroutineScope()
     val focus = LocalFocusManager.current
     val rows = items.orEmpty().toRows()
@@ -110,8 +120,8 @@ internal fun ItemScreen(store: ItemStore, sync: ItemSync? = null) {
     ) {
         Text("Offline Item Tracker", style = MaterialTheme.typography.headlineSmall)
         OutlinedTextField(
-            value = title,
-            onValueChange = { title = it },
+            value = editor.title,
+            onValueChange = { editor = editor.copy(title = it) },
             label = { Text("Item title") },
             singleLine = true,
             enabled = !busy && items != null,
@@ -119,25 +129,21 @@ internal fun ItemScreen(store: ItemStore, sync: ItemSync? = null) {
         )
         Row {
             Button(
-                enabled = !busy && items != null && title.isNotBlank(),
+                enabled = !busy && items != null && editor.title.isNotBlank(),
                 onClick = {
-                    val id = editingId
+                    val submittedEditor = editor
                     change(
                         action = {
-                            if (id == null) store.create(title) else store.rename(id, title)
+                            submittedEditor.submit(store)
                             sync?.markLocalChange()
                         },
-                        after = {
-                            title = ""
-                            editingId = null
-                        },
+                        after = { editor = ItemEditor() },
                     )
                 },
-            ) { Text(if (editingId == null) "Add" else "Save") }
-            if (editingId != null) {
+            ) { Text(if (editor.itemId == null) "Add" else "Save") }
+            if (editor.itemId != null) {
                 TextButton(enabled = !busy, onClick = {
-                    title = ""
-                    editingId = null
+                    editor = ItemEditor()
                     focus.clearFocus()
                 }) { Text("Cancel") }
             }
@@ -199,8 +205,7 @@ internal fun ItemScreen(store: ItemStore, sync: ItemSync? = null) {
                     TextButton(
                         enabled = !busy,
                         onClick = {
-                            editingId = row.id
-                            title = row.title
+                            editor = ItemEditor(row.id, row.title)
                         },
                         modifier = Modifier.semantics {
                             contentDescription = "Edit ${row.title}"
@@ -215,10 +220,7 @@ internal fun ItemScreen(store: ItemStore, sync: ItemSync? = null) {
                                     sync?.markLocalChange()
                                 },
                                 after = {
-                                    if (editingId == row.id) {
-                                        editingId = null
-                                        title = ""
-                                    }
+                                    if (editor.itemId == row.id) editor = ItemEditor()
                                 },
                             )
                         },
diff --git a/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt b/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
index 5ad9e65..02e9829 100644
--- a/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
+++ b/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
@@ -1,5 +1,6 @@
 package com.mobilesystemsevolution.kotlin
 
+import androidx.compose.runtime.saveable.SaverScope
 import org.junit.Assert.assertEquals
 import org.junit.Assert.assertFalse
 import org.junit.Assert.assertTrue
@@ -542,6 +543,34 @@ class ItemStoreTest {
         assertEquals(0, remote.applied)
     }
 
+    @Test
+    fun frozenM08EditorRestorationDoesNotCommitUntilExplicitSubmit() = runBlocking {
+        val seed = Item("ui-001", "Saved title", false, 1, 1700000000000L)
+        val dao = FakeItemDao()
+        dao.insert(seed.toEntity())
+        val savedAt = 1700000600000L
+        val store = ItemStore(dao, nextId = { "ui-002" }, now = { savedAt },
+            nextMutationId = { "m08-rename-001" })
+        val row = store.items().toRows().single()
+        val selected = ItemEditor(itemId = row.id, title = row.title)
+        val edited = selected.copy(title = "Unsubmitted draft")
+        val saverScope = object : SaverScope {
+            override fun canBeSaved(value: Any): Boolean = value is String
+        }
+        val saved = with(ItemEditor.Saver) { checkNotNull(saverScope.save(edited)) }
+        val restored = checkNotNull(ItemEditor.Saver.restore(saved))
+
+        assertEquals(edited, restored)
+        assertEquals(listOf(seed), store.items())
+        assertTrue(store.pendingMutations().isEmpty())
+
+        restored.submit(store)
+
+        assertEquals(listOf(seed.copy(title = "Unsubmitted draft", updatedAt = savedAt)), store.items())
+        assertEquals(listOf(pendingMutation("ui-001", "RENAME", "Unsubmitted draft", null,
+            "m08-rename-001", 1, baseVersion = 1)), store.pendingMutations())
+    }
+
     private fun conflictRemote() = FakeRemote().apply {
         rows.clear()
         rows["conflict-001"] = Item("conflict-001", "Initial", false, 1, 1700000000000L)
diff --git a/verification/M08.md b/verification/M08.md
index 970048c..34a9aa6 100644
--- a/verification/M08.md
+++ b/verification/M08.md
@@ -1,6 +1,7 @@
 # M08 — phase-1 Activity/editor state
 
-Status: unchanged-M07 baseline independently accepted; implementation/final verification pending.
+Status: baseline accepted; host/build and main's final lifecycle/regression verification
+passed. Final history/tag audit remains main-owned.
 START `d01bcd480f9a6beb0fae759a7c63a093c47b3296`; SPEC_REVISION
 `61280dd86ce88b6e431f408241c0998a275960aa`; attempt1, repair0/2.
 
@@ -32,3 +33,51 @@ fixture log offset262→262. Save was left unrun after the reproduced loss, not
 and lsof checks prove PID absent/18080 free. App absent; network0/1/1, active123. Lease released.
 Main independently accepted [the raw baseline](</Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M08/main-baseline-audit.json>).
 Final runtime verification remains main-owned; no further owner device run is authorized.
+
+## Fixed candidate
+
+Baseline support commit: `7a5e731546f3b7b07f98dda82b7bd7e4db040e6b`.
+The product change is the selected-ID/draft `ItemEditor.Saver` used by `rememberSaveable`
+and STARTED `flowWithLifecycle` collection using the existing dependency. Save delegates
+to the existing ItemStore create/rename path; successful-write clearing and failed-write
+draft retention stay intact. No Item/schema/queue/HTTP change or process-kill draft claim.
+
+[One fixed build](../../evidence/phase-1/android-kotlin/M08/fixed-build01.json) ran
+`:app:testDebugUnitTest :app:assembleDebug`, exit0 in 20.660 s. Host17/17 passed with zero
+failures/errors/skips, including all 16 prior cases and the actual editor Saver round trip:
+no Item/intent change before one explicit submit, then the exact base1 rename. No fixture
+tests or test-APK build were repeated. [Static checks](../../evidence/phase-1/android-kotlin/M08/fixed-static-checks.json)
+confirm unchanged native tests, runner, frozen inputs/harness, fixture, schemas and dependency
+bytes. Public test-callable MainActivity/ItemScreen entrypoints remain compatible; compiler
+synthetic access helpers are not claimed byte-identical.
+
+[Candidate manifest](../../evidence/phase-1/android-kotlin/M08/fixed-built-01/manifest.json)
+freezes all 51 source files, the five changed candidate copies, the new app APK
+`05b81312…a5dea`, and the reused baseline test APK `d59f428e…f683`. It links the original
+three baseline support copies and host XML/build/static evidence without duplicating them.
+Main ran the frozen restored M08 case exactly once, then only ItemDatabaseTest and
+ItemUiTest (14+6 existing methods). The no-HTTP window is limited to the M08 case; later
+HTTP regressions used the same owned fixture. Attempt1, repair0/2; no failure or retry.
+
+## Main final verification
+
+[Lifecycle audit](</Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M08/main-lifecycle-audit.json>):
+one JUnit case PASS in 8.286 s; harness 10.252 s/27 adb commands. One background cycle kept
+the Activity; actual recreation changed Activity187410066→169656174 with unchanged
+PID24268/Application88250467. The selected draft survived both boundaries. First five native
+SQLite/WAL snapshots retained the exact seed and pending0. One explicit Save produced the
+same Item with title Unsubmitted draft and one unsent RENAME/base1; its timestamp was inside
+the observed Save clock bounds. No HTTP request (fixture offset262→262). Final Activity
+destruction is JUnit teardown, not another recreation or process-death acceptance case.
+
+[Native audit](</Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M08/main-native-audit.json>):
+all 20 named prior Android methods PASS in 31.835 s, zero failures/skips. Root checked raw
+JUnit/logcat, including M05's four accepted bases and M06/M07 ACK/conflict/Room regressions.
+Those Room reopens are not new external process-death runs. All 51 frozen source files and
+both APKs still matched before final reporting; only TRACK.md and this ledger changed after.
+
+[Cleanup audit](</Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M08/main-cleanup-audit.json>):
+owned fixture61536 stopped with SIGINT, exit0; direct ps/lsof prove absent/18080 free. App
+absent and network0/1/1 active125 were verified by main. No owner final device run or rebuild.
+Raw fixture/lifecycle/native wrapper commands and exits remain in the command ledger;
+main's audit files link its separate cleanup and raw evidence without duplicate copies.
