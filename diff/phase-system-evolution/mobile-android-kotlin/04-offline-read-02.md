## `test(android): add frozen M04 offline reconciliation coverage`

diff --git a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt
index 943b47e..fc09a83 100644
--- a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt
+++ b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt
@@ -31,6 +31,7 @@ import androidx.test.platform.app.InstrumentationRegistry
 import androidx.compose.ui.state.ToggleableState
 import kotlinx.coroutines.runBlocking
 import okhttp3.OkHttpClient
+import okhttp3.MediaType.Companion.toMediaType
 import okhttp3.Request
 import okhttp3.RequestBody.Companion.toRequestBody
 import org.json.JSONObject
@@ -40,6 +41,12 @@ import org.junit.Test
 import org.junit.runner.RunWith
 import org.junit.rules.ExternalResource
 import org.junit.rules.RuleChain
+import java.io.IOException
+import java.util.concurrent.CountDownLatch
+import java.util.concurrent.TimeUnit
+import java.util.concurrent.atomic.AtomicBoolean
+import java.util.concurrent.atomic.AtomicInteger
+import java.util.concurrent.atomic.AtomicLong
 
 @RunWith(AndroidJUnit4::class)
 class ItemUiTest {
@@ -281,9 +288,202 @@ class ItemUiTest {
         }
     }
 
-    private fun fixtureControl(method: String, path: String): JSONObject {
+    @Test
+    fun frozenM04OfflineHttpFailureAndReconnectUseVisibleRoomState() {
+        fixtureControl("POST", "__reset")
+        val database = ItemDatabase.open(InstrumentationRegistry.getInstrumentation().targetContext)
+        val transport = ControlledTransport()
+        val time = AtomicLong(1700000200000L)
+        val clockReads = AtomicInteger(0)
+        val store = ItemStore(database.items(), now = { 1700000200000L })
+        val sync = ItemSync(store, HttpItemRemote(client = transport.client),
+            now = { clockReads.incrementAndGet(); time.get() })
+        try {
+            assertEquals(emptyList<Item>(), runBlocking { store.items() })
+            compose.runOnUiThread {
+                compose.activity.setContent { MaterialTheme { ItemScreen(store, sync) } }
+            }
+            awaitText("Items (0)")
+            assertM04Status(sync, SyncStatus())
+            assertEquals(0, transport.attempts.get())
+
+            clickSync()
+            assertM04Status(sync, SyncStatus(SyncState.FRESH, 1700000200000L))
+            assertM04Rows(store, m04Seeds)
+            recordM04("initial-success", store, sync, transport)
+
+            transport.offline.set(true)
+            clickSync()
+            assertM04Status(sync, SyncStatus(SyncState.ERROR, 1700000200000L, "Forced offline"))
+            assertM04Rows(store, m04Seeds)
+            assertEquals(1, fixtureControl("GET", "__control").getInt("getRequests"))
+            assertEquals(1, transport.delivered.get())
+            recordM04("offline-before-response", store, sync, transport)
+
+            transport.offline.set(false)
+            val failureControl = fixtureControl("POST", "__control", JSONObject().put("getFailures", 1))
+            assertEquals(0, failureControl.getInt("delayMs"))
+            clickSync()
+            assertM04Status(sync, SyncStatus(SyncState.ERROR, 1700000200000L, "Sync HTTP 503"))
+            assertM04Rows(store, m04Seeds)
+            assertEquals(1700000200000L, runBlocking { store.lastSuccessfulRefreshAt() })
+            assertEquals(2, fixtureControl("GET", "__control").getInt("getRequests"))
+            assertEquals(1, clockReads.get())
+            recordM04("http-503", store, sync, transport)
+
+            transport.offline.set(true)
+            fixtureControl("POST", "__control", JSONObject().put("nextTimestamp", 1700000201000L))
+            val changed = fixtureControl("PATCH", "items/remote-001", JSONObject().put("title", "Remote revised"))
+                .getJSONObject("item")
+            assertEquals(2L, changed.getLong("version"))
+            assertEquals(1700000201000L, changed.getLong("updatedAt"))
+            assertM04Rows(store, m04Seeds) // The remote client changed, not this local database.
+
+            transport.offline.set(false)
+            fixtureControl("POST", "__control", JSONObject().put("getFailures", 0))
+            time.set(1700000202000L)
+            clickSync()
+            val expected = listOf(m04Seeds[0].copy(title = "Remote revised", version = 2,
+                updatedAt = 1700000201000L), m04Seeds[1])
+            assertM04Status(sync, SyncStatus(SyncState.FRESH, 1700000202000L))
+            assertM04Rows(store, expected)
+            compose.onNodeWithText("Alpha").assertDoesNotExist()
+            val remoteRows = fixtureControl("GET", "__state").getJSONArray("items")
+            assertEquals(expected, List(remoteRows.length()) { index ->
+                val row = remoteRows.getJSONObject(index)
+                Item(row.getString("id"), row.getString("title"), row.getBoolean("completed"),
+                    row.getLong("version"), row.getLong("updatedAt"))
+            })
+            val control = fixtureControl("GET", "__control")
+            assertEquals(3, control.getInt("getRequests"))
+            assertEquals(0, control.getInt("getFailures"))
+            assertEquals(4, transport.attempts.get())
+            assertEquals(3, transport.delivered.get())
+            assertEquals(2, clockReads.get())
+            recordM04("reconciled", store, sync, transport)
+        } finally {
+            transport.offline.set(false)
+            fixtureControl("POST", "__control", JSONObject().put("getFailures", 0))
+            android.util.Log.i("M04Cleanup", "transportOffline=${transport.offline.get()}; GET failures reset; OS connectivity unchanged")
+            database.close()
+        }
+    }
+
+    @Test
+    fun savedRowsAndTimeAreVisibleBeforeAnExplicitHttpRefreshCompletes() {
+        fixtureControl("POST", "__reset")
+        val database = ItemDatabase.open(InstrumentationRegistry.getInstrumentation().targetContext)
+        val release = CountDownLatch(1)
+        val transport = ControlledTransport(release)
+        val store = ItemStore(database.items(), now = { 1700000200000L })
+        val sync = ItemSync(store, HttpItemRemote(client = transport.client), now = { 1700000202000L })
+        try {
+            // Supporting case: native persistent cache is the only input to the new screen.
+            runBlocking { store.replaceWithCanonical(m04Seeds, 1700000200000L) }
+            compose.runOnUiThread {
+                compose.activity.setContent { MaterialTheme { ItemScreen(store, sync) } }
+            }
+            awaitText("Items (2)")
+            assertM04Status(sync, SyncStatus(SyncState.STALE, 1700000200000L))
+            assertM04Rows(store, m04Seeds)
+            assertEquals(0, transport.attempts.get())
+            assertEquals(0, fixtureControl("GET", "__control").getInt("getRequests"))
+
+            clickSync()
+            compose.waitUntil(5_000) { transport.entered.count == 0L }
+            awaitText("Stale local data · refreshing…")
+            compose.onNodeWithTag("sync-status").assertIsDisplayed()
+                .assertTextEquals("Stale local data · refreshing…")
+            compose.onNodeWithTag("last-successful-refresh").assertIsDisplayed()
+                .assertTextEquals("Last successful refresh: 1700000200000")
+            assertM04Rows(store, m04Seeds)
+            assertEquals(SyncStatus(SyncState.REFRESHING, 1700000200000L), sync.status.value)
+            assertEquals(0, transport.delivered.get())
+            assertEquals(0, fixtureControl("GET", "__control").getInt("getRequests"))
+            recordM04("local-rows-while-http-pending", store, sync, transport)
+
+            release.countDown()
+            assertM04Status(sync, SyncStatus(SyncState.FRESH, 1700000202000L))
+            assertM04Rows(store, m04Seeds)
+            assertEquals(1, transport.delivered.get())
+            assertEquals(1, fixtureControl("GET", "__control").getInt("getRequests"))
+        } finally {
+            release.countDown()
+            transport.offline.set(false)
+            android.util.Log.i("M04Cleanup", "transportOffline=${transport.offline.get()}; test gate released; OS connectivity unchanged")
+            database.close()
+        }
+    }
+
+    private val m04Seeds = listOf(
+        Item("remote-001", "Alpha", false, 1, 1700000000000L),
+        Item("remote-002", "Beta", false, 1, 1700000000000L),
+    )
+
+    private fun clickSync() {
+        compose.onNodeWithText("Sync").performScrollTo().assertIsDisplayed().assertIsEnabled().performClick()
+    }
+
+    private fun assertM04Status(sync: ItemSync, expected: SyncStatus) {
+        compose.waitUntil(5_000) {
+            sync.status.value == expected && compose.onNodeWithText("Sync").fetchSemanticsNode()
+                .config.getOrNull(SemanticsProperties.Disabled) == null
+        }
+        compose.onNodeWithTag("sync-status").assertIsDisplayed().assertTextEquals(when (expected.state) {
+            SyncState.STALE -> "Stale local data"
+            SyncState.REFRESHING -> "Stale local data · refreshing…"
+            SyncState.FRESH -> "Fresh local data"
+            SyncState.ERROR -> "Stale local data · sync error"
+        })
+        compose.onNodeWithTag("last-successful-refresh").assertIsDisplayed()
+            .assertTextEquals("Last successful refresh: ${expected.lastSuccessfulRefreshAt ?: "never"}")
+        val error = expected.error
+        if (error == null) compose.onNodeWithTag("sync-error").assertDoesNotExist()
+        else compose.onNodeWithTag("sync-error").assertIsDisplayed().assertTextContains(error)
+        compose.onNodeWithTag("storage-error").assertDoesNotExist()
+    }
+
+    private fun assertM04Rows(store: ItemStore, expected: List<Item>) {
+        assertEquals(expected, runBlocking { store.items() })
+        compose.onNodeWithTag("item-count").assertTextEquals("Items (2)")
+        compose.onAllNodes(rowMatcher).assertCountEquals(2)
+        expected.forEach {
+            compose.onNodeWithTag("item-row-${it.id}").assertIsDisplayed()
+            compose.onNodeWithText(it.title).assertIsDisplayed()
+            compose.onNodeWithContentDescription("Completed: ${it.title}").assertIsOff()
+        }
+    }
+
+    private fun recordM04(phase: String, store: ItemStore, sync: ItemSync, transport: ControlledTransport) {
+        android.util.Log.i("M04State", "$phase: status=${sync.status.value}; rows=${runBlocking { store.items() }}; " +
+            "persistedSuccess=${runBlocking { store.lastSuccessfulRefreshAt() }}; " +
+            "attempts=${transport.attempts.get()}; delivered=${transport.delivered.get()}\n${compose.onRoot().printToString()}")
+    }
+
+    // Instrumentation-only deterministic input. Online calls still use actual OkHttp HTTP.
+    private class ControlledTransport(private val release: CountDownLatch? = null) {
+        val offline = AtomicBoolean(false)
+        val attempts = AtomicInteger(0)
+        val delivered = AtomicInteger(0)
+        val entered = CountDownLatch(1)
+        val client = OkHttpClient.Builder().retryOnConnectionFailure(false).followRedirects(false)
+            .callTimeout(10, TimeUnit.SECONDS).addInterceptor { chain ->
+                attempts.incrementAndGet()
+                if (offline.get()) throw IOException("Forced offline")
+                if (release != null) {
+                    entered.countDown()
+                    check(release.await(10, TimeUnit.SECONDS)) { "M04 test gate was not released" }
+                }
+                delivered.incrementAndGet()
+                chain.proceed(chain.request())
+            }.build()
+    }
+
+    private fun fixtureControl(method: String, path: String, payload: JSONObject? = null): JSONObject {
         val request = Request.Builder().url(HttpItemRemote.FIXTURE_URL + path)
-            .method(method, if (method == "POST") byteArrayOf().toRequestBody() else null).build()
+            .header("Connection", "close")
+            .method(method, if (method == "GET") null else
+                (payload?.toString() ?: "").toRequestBody("application/json".toMediaType())).build()
         return OkHttpClient.Builder().retryOnConnectionFailure(false).build().newCall(request).execute().use {
             assertEquals(200, it.code)
             val body = requireNotNull(it.body).string()
@@ -291,4 +491,5 @@ class ItemUiTest {
             JSONObject(body)
         }
     }
+
 }
diff --git a/fixture/server.py b/fixture/server.py
index 75d05ce..888aa93 100644
--- a/fixture/server.py
+++ b/fixture/server.py
@@ -1,5 +1,5 @@
 #!/usr/bin/env python3
-"""M03's local, in-memory HTTP fixture. No delay, injected failures, or external services."""
+"""Local M03/M04 HTTP fixture: in-memory canonical Items, fixed zero delay, test controls."""
 
 import argparse
 from http.server import BaseHTTPRequestHandler, HTTPServer
@@ -20,6 +20,8 @@ class Fixture(HTTPServer):
                                version=1, updatedAt=1700000000000),
         }
         self.next_timestamp = 1700000100000
+        self.get_failures = 0
+        self.get_requests = 0
 
     def tick(self):
         timestamp = self.next_timestamp
@@ -29,6 +31,10 @@ class Fixture(HTTPServer):
     def sorted_items(self):
         return [self.items[key] for key in sorted(self.items)]
 
+    def controls(self):
+        return dict(getFailures=self.get_failures, getRequests=self.get_requests,
+                    delayMs=0, nextTimestamp=self.next_timestamp)
+
 
 class Handler(BaseHTTPRequestHandler):
     def respond(self, status, payload):
@@ -59,10 +65,17 @@ class Handler(BaseHTTPRequestHandler):
 
     def do_GET(self):
         if self.path == "/items":
-            self.respond(200, {"items": self.server.sorted_items()})
+            self.server.get_requests += 1
+            if self.server.get_failures:
+                self.server.get_failures -= 1
+                self.respond(503, {"error": "temporary failure"})
+            else:
+                self.respond(200, {"items": self.server.sorted_items()})
         elif self.path == "/__state":
             self.respond(200, {"items": self.server.sorted_items(),
                                "nextTimestamp": self.server.next_timestamp})
+        elif self.path == "/__control":
+            self.respond(200, self.server.controls())
         else:
             self.respond(404, {"error": "not found"})
 
@@ -72,6 +85,22 @@ class Handler(BaseHTTPRequestHandler):
             self.respond(200, {"items": self.server.sorted_items(),
                                "nextTimestamp": self.server.next_timestamp})
             return
+        if self.path == "/__control":
+            try:
+                data = self.payload()
+                if not data or not set(data) <= {"getFailures", "nextTimestamp"}:
+                    raise ValueError("Expected getFailures and/or nextTimestamp")
+                if any(type(value) is not int or value < 0 for value in data.values()):
+                    raise ValueError("Expected nonnegative integers")
+            except (ValueError, TypeError) as error:
+                self.respond(400, {"error": str(error)})
+                return
+            if "getFailures" in data:
+                self.server.get_failures = data["getFailures"]
+            if "nextTimestamp" in data:
+                self.server.next_timestamp = data["nextTimestamp"]
+            self.respond(200, self.server.controls())
+            return
         if self.path != "/items":
             self.respond(404, {"error": "not found"})
             return
@@ -127,7 +156,7 @@ if __name__ == "__main__":
     parser.add_argument("--port", type=int, default=18080)
     args = parser.parse_args()
     with Fixture(("127.0.0.1", args.port)) as fixture:
-        print(f"M03 fixture listening on http://127.0.0.1:{args.port}; delay=0; failures=none", flush=True)
+        print(f"M03/M04 fixture listening on http://127.0.0.1:{args.port}; delay=0; getFailures=0", flush=True)
         try:
             fixture.serve_forever()
         except KeyboardInterrupt:
diff --git a/fixture/test_server.py b/fixture/test_server.py
index 40e7df3..d546f29 100644
--- a/fixture/test_server.py
+++ b/fixture/test_server.py
@@ -2,11 +2,56 @@ import json
 from threading import Thread
 import unittest
 from urllib.request import Request, urlopen
+from urllib.error import HTTPError
 
 from server import Fixture
 
 
 class FixtureContractTest(unittest.TestCase):
+    def test_frozen_m04_failure_and_remote_change_controls(self):
+        with Fixture(("127.0.0.1", 0)) as server:
+            thread = Thread(target=server.serve_forever, daemon=True)
+            thread.start()
+            try:
+                base = f"http://127.0.0.1:{server.server_port}"
+
+                def request(method, path, payload=None):
+                    data = None if payload is None else json.dumps(payload).encode()
+                    try:
+                        response = urlopen(Request(base + path, data=data, method=method,
+                                                   headers={"Content-Type": "application/json"}), timeout=5)
+                    except HTTPError as error:
+                        response = error
+                    with response:
+                        return response.status, json.load(response)
+
+                seeds = [
+                    dict(id="remote-001", title="Alpha", completed=False, version=1, updatedAt=1700000000000),
+                    dict(id="remote-002", title="Beta", completed=False, version=1, updatedAt=1700000000000),
+                ]
+                self.assertEqual((200, {"items": seeds}), request("GET", "/items"))
+                self.assertEqual((200, dict(getFailures=1, getRequests=1, delayMs=0,
+                                            nextTimestamp=1700000100000)),
+                                 request("POST", "/__control", {"getFailures": 1}))
+                self.assertEqual((503, {"error": "temporary failure"}), request("GET", "/items"))
+                self.assertEqual((200, dict(items=seeds, nextTimestamp=1700000100000)),
+                                 request("GET", "/__state"))
+                self.assertEqual((200, dict(getFailures=0, getRequests=2, delayMs=0,
+                                            nextTimestamp=1700000201000)),
+                                 request("POST", "/__control", {"nextTimestamp": 1700000201000}))
+                revised = dict(seeds[0], title="Remote revised", version=2, updatedAt=1700000201000)
+                self.assertEqual((200, {"item": revised}), request("PATCH", "/items/remote-001",
+                                                                 {"title": "Remote revised"}))
+                self.assertEqual((200, dict(getFailures=0, getRequests=2, delayMs=0,
+                                            nextTimestamp=1700000202000)),
+                                 request("POST", "/__control", {"getFailures": 0}))
+                self.assertEqual((200, {"items": [revised, seeds[1]]}), request("GET", "/items"))
+                self.assertEqual((200, dict(getFailures=0, getRequests=3, delayMs=0,
+                                            nextTimestamp=1700000202000)), request("GET", "/__control"))
+            finally:
+                server.shutdown()
+                thread.join(timeout=5)
+
     def test_frozen_m03_exact_http_contract(self):
         with Fixture(("127.0.0.1", 0)) as server:
             thread = Thread(target=server.serve_forever, daemon=True)
diff --git a/verification/M04-inputs.json b/verification/M04-inputs.json
new file mode 100644
index 0000000..925c9dc
--- /dev/null
+++ b/verification/M04-inputs.json
@@ -0,0 +1,30 @@
+{
+  "thread": "M04",
+  "specRevision": "ed7baa246ee947c6778e0f84751c9b91cec7abfe",
+  "start": "f3591d527b40bd00e2a885c4a53d2508bae209fd",
+  "fixturePort": 18080,
+  "delayMs": 0,
+  "seeds": [
+    {"id":"remote-001","title":"Alpha","completed":false,"version":1,"updatedAt":1700000000000},
+    {"id":"remote-002","title":"Beta","completed":false,"version":1,"updatedAt":1700000000000}
+  ],
+  "sequence": [
+    "Reset fixture and local storage; explicit refresh succeeds at 1700000200000",
+    "Set test client transport offline; one explicit refresh throws IOException before chain.proceed and receives no HTTP response",
+    "Restore test client transport; POST /__control {getFailures:1}; one explicit GET /items returns 503 {error:temporary failure}",
+    "Set client transport offline; POST /__control {nextTimestamp:1700000201000}; another client PATCH /items/remote-001 {title:Remote revised}",
+    "Restore transport; POST /__control {getFailures:0}; explicit refresh succeeds at 1700000202000"
+  ],
+  "expectedFinal": [
+    {"id":"remote-001","title":"Remote revised","completed":false,"version":2,"updatedAt":1700000201000},
+    {"id":"remote-002","title":"Beta","completed":false,"version":1,"updatedAt":1700000000000}
+  ],
+  "baseline": "Unchanged M03 production and HTTP fixture; run only the initial successful refresh and forced-offline failure prefix. Assert saved rows survive, generic sync error exists, but sync-status and last-successful-refresh nodes are absent. M03 has no refresh-clock input.",
+  "supportingCases": {
+    "localFirstAndroid": "Two seed rows and success time 1700000200000 already persisted; construct a new ItemSync and screen. Before any explicit refresh, show saved rows, stale status and prior time with zero HTTP calls. For one explicit refresh, a test interceptor waits on a latch before real HTTP; assert saved rows/status/time while pending, release latch once, then require fresh with unchanged canonical rows at 1700000202000. No fixture response delay is changed.",
+    "migration": "Real SQLite v1 items schema with the two seed rows upgrades to Room v2 without changing any Item field; successful-refresh time initially absent, then a canonical transaction at 1700000200000 persists it across close/open.",
+    "rollback": "Existing item-001/Alpha edited/true/0/1700000006000 with last-success 1700000200000; canonical replacement containing duplicate remote-001 primary keys at 1700000202000 must roll back both rows and last-success time.",
+    "hostTransitions": "Use the same frozen seeds, clock and failure sequence; capture stale -> refreshing -> fresh -> refreshing -> error -> refreshing -> error -> refreshing -> fresh. Neither failure advances 1700000200000. A new sync instance loads that persisted time as stale.",
+    "priorRegression": "Run unchanged M01/M03 mutation inputs, unknown-schema=99 preservation and canonical external process restart. Update only expected current schema version in the existing read-only capture helper when v2 is introduced; do not weaken Item equality or death-boundary rules."
+  }
+}
diff --git a/verification/canonical_restart.py b/verification/canonical_restart.py
index 5246774..67fd576 100644
--- a/verification/canonical_restart.py
+++ b/verification/canonical_restart.py
@@ -58,6 +58,7 @@ if __name__ == "__main__":
     parser = argparse.ArgumentParser(description=__doc__)
     parser.add_argument("--apk", type=Path, required=True, help="Built APK identity only; never installed here")
     parser.add_argument("--output", type=Path, required=True)
+    parser.add_argument("--schema-version", type=int, default=1)
     parser.add_argument("--adb", type=Path, default=Path(os.environ.get("ANDROID_HOME", "")) / "platform-tools/adb")
     args = parser.parse_args()
     args.expect = "canonical"
diff --git a/verification/process_restart.py b/verification/process_restart.py
index 8ffa2da..250f5d2 100644
--- a/verification/process_restart.py
+++ b/verification/process_restart.py
@@ -123,8 +123,9 @@ class Scenario:
                 if columns != ["id", "title", "completed", "version", "updatedAt"]:
                     raise AssertionError(f"Unexpected schema fields: {columns}")
                 version = database.execute("PRAGMA user_version").fetchone()[0]
-                if version != 1:
-                    raise AssertionError(f"Expected initial schema 1, got {version}")
+                expected_version = getattr(self.args, "schema_version", 1)
+                if version != expected_version:
+                    raise AssertionError(f"Expected schema {expected_version}, got {version}")
                 rows = [dict(zip(columns, row)) for row in database.execute(
                     "SELECT id, title, completed, version, updatedAt FROM items ORDER BY rowid"
                 )]
@@ -209,6 +210,7 @@ def main():
     parser.add_argument("--expect", choices=("memory-loss", "durable"), required=True)
     parser.add_argument("--apk", type=Path, required=True)
     parser.add_argument("--output", type=Path, required=True)
+    parser.add_argument("--schema-version", type=int, default=1)
     parser.add_argument("--adb", type=Path, default=Path(os.environ.get("ANDROID_HOME", "")) / "platform-tools/adb")
     args = parser.parse_args()
     scenario = Scenario(args)


