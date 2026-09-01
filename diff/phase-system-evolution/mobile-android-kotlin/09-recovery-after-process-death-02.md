## `fix(M09): recover durable pending work on startup`

diff --git a/TRACK.md b/TRACK.md
index 128abc3..78dd05f 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -221,6 +221,28 @@ verified frozen execution bytes and cleanup. Attempt1, repair0/2; only reporting
 after runtime verification. Commands and evidence are in `verification/M08.md`; final
 history/tag audit remains main-owned.
 
+## M09 boundary — official runtime passed
+
+Ordinary startup reads committed local rows/status, then resumes nonterminal pending work
+once through the existing foreground drain. Empty/all-terminal queues make no HTTP request.
+The original saved identity/hash/base and acknowledgment transaction are reused; failures
+retain the intent without a retry loop. M08 editor restoration is unchanged. There is no
+scheduler, background work, new storage schema or protocol change.
+
+Main reproduced M08's missing startup recovery on both fixed actual-process cases, then
+passed the frozen M09 A/B cases: one queued create applied once, and one committed rename
+replayed with its original base1 identity after death while its response was held open and
+headerless. Both ordinary cold starts reached pending0 without Sync or data injection.
+Main read all nine native SQLite/WAL checkpoints and actual PID/HTTP/UI evidence.
+
+Host20/20 and the one app build passed. Main also passed the unchanged M08 lifecycle case
+(six native checkpoints, restored draft, no HTTP) and the selected seven native regressions:
+all six UI methods plus the real-HTTP successor-ACK Room-reopen case. The other13 Room
+methods retain their prior M08 evidence; no full native suite or old external-case rerun is claimed.
+The fixture12 baseline results and test APK were reused unchanged. Main confirmed all
+frozen execution bytes and actual cleanup. Attempt1, repair0/2; only this reporting and
+`verification/M09.md` changed after runtime. Final history/tag audit remains main-owned.
+
 ## Pinned build
 
 - JDK 17, Gradle 8.7 (wrapper distribution SHA-256 pinned)
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemSync.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemSync.kt
index 12e9f67..515a498 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemSync.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/ItemSync.kt
@@ -20,7 +20,7 @@ data class SyncStatus(
     val error: String? = null,
 )
 
-/** One explicit foreground call drains the durable payloads in their original order. */
+/** One foreground call drains the durable payloads in their original order. */
 class ItemSync(
     private val store: ItemStore,
     private val remote: ItemRemote,
@@ -29,7 +29,7 @@ class ItemSync(
     private val mutableStatus = MutableStateFlow(SyncStatus())
     val status = mutableStatus.asStateFlow()
 
-    // A new session is stale until explicitly refreshed, even when a saved timestamp exists.
+    // A new session is stale until refreshed, even when a saved timestamp exists.
     suspend fun readSavedStatus() {
         mutableStatus.value = mutableStatus.value.copy(lastSuccessfulRefreshAt = store.lastSuccessfulRefreshAt())
         store.pendingMutations().firstOrNull { it.terminalError != null && it.terminalError != "version_conflict" }?.let {
@@ -41,6 +41,11 @@ class ItemSync(
         mutableStatus.value = mutableStatus.value.copy(state = SyncState.STALE, error = null)
     }
 
+    /** Startup resumes durable work once; empty or terminal-only queues need no request. */
+    suspend fun recoverPending() {
+        if (store.pendingMutations().any { it.terminalError == null }) synchronize()
+    }
+
     suspend fun synchronize() {
         try {
             mutableStatus.value = SyncStatus(SyncState.REFRESHING, store.lastSuccessfulRefreshAt())
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
index 3cb4ae8..0eddf27 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
@@ -112,7 +112,13 @@ internal fun ItemScreen(store: ItemStore, sync: ItemSync? = null) {
         scope.launch { accessStorage(action, after) }
     }
 
-    LaunchedEffect(store) { accessStorage() }
+    LaunchedEffect(store, sync) {
+        // Populate committed local rows and status before any startup network work.
+        accessStorage()
+        if (sync != null && storageError == null) {
+            change(action = { sync.recoverPending() }, isSync = true)
+        }
+    }
 
     Column(
         modifier = Modifier.fillMaxSize().verticalScroll(rememberScrollState()).padding(16.dp),
diff --git a/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt b/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
index 02e9829..fa93fdc 100644
--- a/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
+++ b/app/src/test/java/com/mobilesystemsevolution/kotlin/ItemStoreTest.kt
@@ -571,6 +571,106 @@ class ItemStoreTest {
             "m08-rename-001", 1, baseVersion = 1)), store.pendingMutations())
     }
 
+    @Test
+    fun m09StartupRecoversFixedCreateAndLostPatchAcknowledgment(): Unit = runBlocking {
+        for (case in listOf("A", "B")) {
+            val dao = FakeItemDao()
+            val identity = if (case == "A") "m09-create-001" else "m09-update-001"
+            val store = ItemStore(dao, nextId = { "death-001" }, now = { 1700000600000L },
+                nextMutationId = { identity })
+            val remote = FakeRemote().apply { rows.clear(); time = 1700000600000L }
+            if (case == "A") {
+                store.create("Recovered create")
+            } else {
+                val seed = Item("death-001", "Initial", false, 1, 1700000000000L)
+                remote.rows[seed.id] = seed
+                store.replaceWithCanonical(listOf(seed))
+                store.rename(seed.id, "Recovered update")
+                remote.dropNextPatchResponse = true
+                assertThrows(IOException::class.java) { runBlocking { ItemSync(store, remote).synchronize() } }
+                assertEquals(1, remote.applied)
+                assertEquals(0, remote.duplicates)
+            }
+            val pending = store.pendingMutations().single()
+            val local = store.items()
+            assertEquals(if (case == "A")
+                "4af45629e87641c772fbdd173149b3b1a8779ef74699af2b1768d2599a0a3bd4" else
+                "53cba8ff4fb268be35bb6c26b1a61903143e7237b78f3065996c7469c7ccda12", pending.payloadHash)
+            assertTrue(store.acknowledgedMutations().isEmpty())
+
+            // This host check recreates the store/sync; the frozen Android case proves actual death.
+            val reopened = ItemStore(dao, nextMutationId = { error("Recovery must not mint an identity") })
+            val sync = ItemSync(reopened, remote, now = { 1700000600000L })
+            sync.readSavedStatus()
+            assertEquals(local, reopened.items())
+            assertEquals(listOf(pending), reopened.pendingMutations())
+            sync.recoverPending()
+
+            val result = remote.responses.getValue(identity).second
+            assertEquals(listOf(requireNotNull(result.item)), reopened.items())
+            assertTrue(reopened.pendingMutations().isEmpty())
+            assertEquals(listOf(AcknowledgedMutation(identity, pending.payloadHash,
+                result.statusCode, result.responseBody)), reopened.acknowledgedMutations())
+            assertEquals(List(if (case == "A") 1 else 2) { pending.copy(dispatched = true) }, remote.mutations)
+            assertEquals(1, remote.applied)
+            assertEquals(if (case == "A") 0 else 1, remote.duplicates)
+            assertEquals(SyncStatus(SyncState.FRESH, 1700000600000L), sync.status.value)
+            val calls = remote.calls.toList()
+            sync.recoverPending()
+            assertEquals(calls, remote.calls) // An empty queue does not trigger another refresh.
+        }
+    }
+
+    @Test
+    fun m09StartupSkipsEmptyAndAllTerminalQueues() = runBlocking {
+        for (terminal in listOf(null, "identity_conflict", "version_conflict", "base_version_unknown")) {
+            val dao = FakeItemDao()
+            val seed = Item("death-001", "Saved local", false, 1, 1700000000000L)
+            dao.insert(seed.toEntity())
+            if (terminal != null) {
+                dao.insertPending(pendingMutation(seed.id, "RENAME", "Retained attempt",
+                    clientMutationId = "m09-terminal-001", sequence = 1, baseVersion = 1)
+                    .copy(dispatched = true, terminalError = terminal))
+            }
+            val store = ItemStore(dao)
+            val pending = store.pendingMutations()
+            val remote = FakeRemote()
+            val sync = ItemSync(store, remote)
+            sync.readSavedStatus()
+            val status = sync.status.value
+
+            sync.recoverPending()
+
+            assertEquals(listOf(seed), store.items())
+            assertEquals(pending, store.pendingMutations())
+            assertEquals(status, sync.status.value)
+            assertTrue(remote.calls.isEmpty())
+            assertTrue(store.acknowledgedMutations().isEmpty())
+        }
+    }
+
+    @Test
+    fun m09StartupTransportFailureDoesNotRetryOrDiscardItsIntent(): Unit = runBlocking {
+        val store = ItemStore(FakeItemDao(), nextId = { "death-001" }, now = { 1700000600000L },
+            nextMutationId = { "m09-create-001" })
+        store.create("Recovered create")
+        val local = store.items()
+        val pending = store.pendingMutations().single()
+        var attempts = 0
+        val remote = FakeRemote().apply {
+            beforeMutation = { attempts++; throw IOException("Forced offline") }
+        }
+        val sync = ItemSync(store, remote)
+
+        assertThrows(IOException::class.java) { runBlocking { sync.recoverPending() } }
+
+        assertEquals(1, attempts)
+        assertEquals(listOf(pending.copy(dispatched = true)), store.pendingMutations())
+        assertEquals(local, store.items())
+        assertTrue(store.acknowledgedMutations().isEmpty())
+        assertEquals(SyncStatus(SyncState.ERROR, null, "Forced offline"), sync.status.value)
+    }
+
     private fun conflictRemote() = FakeRemote().apply {
         rows.clear()
         rows["conflict-001"] = Item("conflict-001", "Initial", false, 1, 1700000000000L)
diff --git a/verification/M09.md b/verification/M09.md
index 6e35210..59f6e51 100644
--- a/verification/M09.md
+++ b/verification/M09.md
@@ -1,7 +1,7 @@
 # M09 — phase-1 process-death startup recovery
 
-Status: unchanged-M08 limitation baseline independently accepted. Product implementation
-and final M09 verification are pending; this is not a completion claim.
+Status: main's official M09 runtime and cleanup passed. Final commit/history/tag audit
+remains main-owned. The independently accepted unchanged-M08 baseline is retained below.
 START `8f39fb5cb565d04035c3623da56b8326cb03171d`; SPEC_REVISION
 `61280dd86ce88b6e431f408241c0998a275960aa`; attempt1, repair0/2.
 
@@ -45,5 +45,61 @@ unchanged. [Cleanup](</Users/woopinbell/Desktop/working/workflow/mobile-systems-
 owned fixture90586 stopped with SIGINT and exited0; direct checks proved PID absent/18080
 free, app absent, network0/1/1 active131. Main released the lease. No owner device run.
 
-This commit preserves baseline support/evidence only. TRACK remains at M08; implementation
-requires main's separate authorization after its Git/history audit.
+Baseline-support commit `d36e668ae2af3009c66336152dde19b72fa0c687` preserved only those
+support/evidence files. Main audited its exact six paths/bytes/history before authorizing
+the following implementation.
+
+## Candidate preparation
+
+Startup now loads committed rows/status first, then invokes the existing foreground drain
+once when nonterminal pending work exists. Empty/all-terminal queues send no HTTP. The
+existing focus/busy/error path is reused; no retry loop, scheduler, schema, identity/base
+change, startup suppression flag or editor change was added.
+
+[One host/build command](../../evidence/phase-1/android-kotlin/M09/fixed-build01.json)
+ran all20 host methods (zero failures/errors/skips; XML0.112 s) and assembled the app:
+exit0, wrapper19.997 s, 45 tasks/13 executed/32 up-to-date. All original17 host methods
+remain byte-identical; three added cases cover fixed A/B recovery, terminal/empty queues,
+and transport failure without retry. These host cases use store recreation, not actual
+process death. Fixture12 baseline results and all21 native methods/test APK are retained
+unchanged; no fixture/native rerun or test APK rebuild was performed by the owner.
+
+[Static read-only check](../../evidence/phase-1/android-kotlin/M09/fixed-static-checks.json)
+verified prebuild source bytes, current app API/DEX, reused test APK and old method bytes.
+[Final manifest](../../evidence/phase-1/android-kotlin/M09/fixed-built-01/manifest.json)
+freezes source/app/host artifacts and root-only commands: unchanged M09 A/B with recovered
+expectation, unchanged actual M08 lifecycle, then all six ItemUiTest methods plus the
+existing real-HTTP M05/M06 successor-ACK Room-reopen method. No claim is made that the
+other prior external scenarios or all14 Room methods were rerun for this candidate.
+
+## Official final verification
+
+[Main runtime record](</Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M09/main-runtime-verification.json>)
+links five independently checked subaudits. Main executed the frozen commands once, without
+repairs or capped cases, and confirmed all54 source hashes and both APKs unchanged.
+
+| Actual process case | PID boundary | Result |
+| --- | --- | --- |
+| A (26.758 s) | 28632 → absent → 28918 | Original m09-create-001 applied once, duplicate0; canonical v1, pending0 and stored receipt. |
+| B (36.199 s) | 29143 → absent → 29483 | Original m09-update-001/base1/hash replayed; applied1/duplicate1, canonical v2, pending0 and original receipt. |
+
+The [process audit](</Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M09/main-process-recovery-audit.json>)
+read all nine SQLite/WAL snapshots and 150 adb commands. B's actual fresh open/headerless
+observation at145648.299881208 preceded confirmed absence at145648.428225875; the response
+was dropped only after absence, within the unchanged10-second client/30-second barrier.
+Both cold starts used ordinary MainActivity with no manual Sync or injection after death.
+Parked controller termination remains expected termination, not a JUnit pass.
+
+The [M08 regression](</Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M09/main-lifecycle-regression-audit.json>)
+passed in PID29765/Application88250467 with Activity187410066→98133114: one background cycle,
+one recreation, selected draft restored, six native snapshots, no HTTP and no write until
+one explicit Save. This is same-process Activity evidence, not a process-kill draft claim.
+The [selected native regression](</Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M09/main-native-regression-audit.json>)
+passed all seven named methods with exact start1/pass0 codes and OK(7), zero skips:
+JUnit32.512 s/wrapper33.55 s. Its HTTP successor replay is a same-process Room reopen
+(applied4/duplicate1/requests5); other13 native database methods were not rerun.
+
+[Cleanup](</Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M09/main-final-cleanup.json>):
+owned fixture4455 received SIGINT and actually exited0; direct checks proved PID absent,
+port18080 free, app absent and network0/1/1 active133. No owner device run or additional
+build/test occurred. Only TRACK.md and this ledger changed after the successful frozen run.
